# roboshop-infra-provisioning

> Ansible-based infrastructure provisioning and configuration management for the RoboShop e-commerce platform on AWS.

---

## Overview

This repository contains the Ansible automation used to provision, configure, and manage all infrastructure components of the **RoboShop** platform. It handles end-to-end environment bootstrap — from EC2 instance creation and Route 53 DNS registration, through service-level configuration of every application tier.

The playbooks in this repo are intended to be run sequentially. Each playbook is numbered to reflect its execution order and service dependency chain. Roles consumed by these playbooks are maintained separately in [`roboshop-ansible-collection`](https://github.com/rahul-paladugu/Ansible-Roboshop-Roles).

---

## Architecture

RoboShop is a microservices-based e-commerce application. The infrastructure is composed of the following services, each running on a dedicated EC2 instance within a private subnet, exposed via internal Route 53 DNS records.

| # | Service    | Runtime      | Role                          |
|---|------------|--------------|-------------------------------|
| 0 | Infra Init | AWS / Ansible| EC2 provisioning + R53 setup  |
| 1 | MongoDB    | MongoDB       | Product catalogue data store  |
| 2 | Catalogue  | Node.js       | Product catalogue API         |
| 3 | Redis      | Redis         | Session / cart cache          |
| 4 | User       | Node.js       | User accounts & auth          |
| 5 | Cart       | Node.js       | Shopping cart service         |
| 6 | MySQL      | MySQL         | Order & shipping data store   |
| 7 | Shipping   | Java (Maven)  | Shipping cost calculation     |
| 8 | RabbitMQ   | RabbitMQ      | Async message broker          |
| 9 | Payment    | Python        | Payment processing service    |
| 10| Dispatch   | Go            | Order dispatch service        |
| 11| Frontend   | Nginx         | Web reverse proxy / UI host   |

All backend services communicate over private DNS (`<service>.rscloudservices.icu`). The frontend is the only service exposed with a public IP, registered to the apex domain.

---

## Repository Structure

```
.
├── 0-instances.yaml       # Provision EC2 instances + Route 53 DNS records
├── 1-mongodb.yaml         # Install and configure MongoDB
├── 2-catalogue.yaml       # Deploy Catalogue service (Node.js)
├── 3-redis.yaml           # Install and configure Redis
├── 4-user.yaml            # Deploy User service (Node.js)
├── 5-cart.yaml            # Deploy Cart service (Node.js)
├── 6-mysql.yaml           # Install and configure MySQL + seed schema
├── 7-shipping.yaml        # Deploy Shipping service (Java)
├── 8-rabbitmq.yaml        # Install and configure RabbitMQ
├── 9-payment.yaml         # Deploy Payment service (Python)
├── 10-dispatch.yaml       # Deploy Dispatch service (Go)
├── 11-frontend.yaml       # Configure Nginx frontend + reverse proxy
├── inventory.ini          # Static inventory (post-provisioning)
├── nginx.conf             # Nginx reverse proxy configuration
├── mongo.repo             # MongoDB yum repository definition
├── rabbitmq.repo          # RabbitMQ yum repository definition
├── *.service              # Systemd unit files for each application service
└── readme-shipping        # Notes specific to shipping service setup
```

---

## Prerequisites

### Control Node Requirements

- Ansible `>= 2.14`
- Python `>= 3.9`
- `boto3` and `botocore` Python libraries (installed automatically by `0-instances.yaml`)
- AWS credentials configured (`~/.aws/credentials` or IAM instance role)
- SSH access to target EC2 instances (key pair configured)

### AWS Requirements

- An existing **Security Group** permitting required inter-service ports
- A **Route 53 Hosted Zone** for your domain (private or public)
- An **AMI** compatible with RHEL/CentOS (dnf-based package management)
- IAM permissions: `ec2:RunInstances`, `ec2:DescribeInstances`, `route53:ChangeResourceRecordSets`

### Update Before Running

Before executing any playbook, review and update the following variables in `0-instances.yaml`:

```yaml
instance_type: t3.micro             # Adjust based on environment tier
image_id: ami-0220d79f3f480ecf5     # Verify AMI is available in your region
security_group_id: sg-XXXXXXXXXX    # Replace with your SG ID
hosted_zone: your-domain.com        # Your Route 53 hosted zone
hosted_zone_id: ZXXXXXXXXXX         # Your hosted zone ID
```

---

## Usage

### Step 1 — Provision Infrastructure

Run the instance provisioning playbook from the control node. This creates all EC2 instances and registers DNS records in Route 53.

```bash
ansible-playbook 0-instances.yaml
```

This playbook will:
- Spin up 11 EC2 instances (one per service), tagged with `Environment: Dev`
- Register private IP DNS A-records for all backend services
- Register a public IP DNS A-record at the apex domain for the frontend
- Dump the full EC2 output to `/home/ec2-user/ec2_output.json` for reference

> **Note:** Wait for all instances to reach a running state before proceeding. The playbook does not include a deliberate wait step — verify in the AWS Console or via `aws ec2 describe-instances` if needed.

### Step 2 — Update Inventory

Once instances are running, update `inventory.ini` with the actual private IP addresses or DNS names assigned. Example format:

```ini
[mongodb]
mongodb.rscloudservices.icu

[catalogue]
catalogue.rscloudservices.icu

[redis]
redis.rscloudservices.icu

# ... repeat for each service group
```

### Step 3 — Configure Services

Run each service playbook **in order**. Services have hard dependencies on upstream components being healthy before they can start.

```bash
ansible-playbook -i inventory.ini 1-mongodb.yaml
ansible-playbook -i inventory.ini 2-catalogue.yaml
ansible-playbook -i inventory.ini 3-redis.yaml
ansible-playbook -i inventory.ini 4-user.yaml
ansible-playbook -i inventory.ini 5-cart.yaml
ansible-playbook -i inventory.ini 6-mysql.yaml
ansible-playbook -i inventory.ini 7-shipping.yaml
ansible-playbook -i inventory.ini 8-rabbitmq.yaml
ansible-playbook -i inventory.ini 9-payment.yaml
ansible-playbook -i inventory.ini 10-dispatch.yaml
ansible-playbook -i inventory.ini 11-frontend.yaml
```

### Full Stack Deployment (Single Command)

To run the complete stack in one shot after instance provisioning:

```bash
for i in $(seq 1 11); do
  playbook=$(ls ${i}-*.yaml)
  echo "==> Running: $playbook"
  ansible-playbook -i inventory.ini "$playbook"
done
```

---

## Service DNS Convention

All services register under the pattern:

```
<service-name>.<hosted-zone>
```

Example (using `rscloudservices.icu`):

| Service   | DNS Record                          |
|-----------|-------------------------------------|
| MongoDB   | `mongodb.rscloudservices.icu`       |
| Catalogue | `catalogue.rscloudservices.icu`     |
| Redis     | `redis.rscloudservices.icu`         |
| Frontend  | `rscloudservices.icu` (apex record) |

TTL is set to `1` second during provisioning to allow rapid DNS propagation during environment setup. Adjust this before promoting to higher environments.

---

## Systemd Service Management

Each application service is managed via a systemd unit file. These are deployed automatically by the respective playbook. To manage services manually post-deployment:

```bash
# Check service status
systemctl status catalogue

# Restart a service
systemctl restart payment

# View service logs
journalctl -u shipping -f
```

---

## Related Repositories

| Repository | Purpose |
|---|---|
| [`roboshop-ansible-collection`](https://github.com/rahul-paladugu/Ansible-Roboshop-Roles) | Reusable Ansible roles consumed by playbooks in this repo |

---

## Environment Tags

All EC2 instances provisioned by this repo are tagged with `Environment: Dev` by default. For higher environments, update the tag value in `0-instances.yaml` before provisioning:

```yaml
tags:
  Name: "{{ item }}"
  Environment: Staging   # or Production
```

---

## Known Limitations

- **No idempotency guard on Route 53 records** — re-running `0-instances.yaml` will overwrite existing DNS records with new instance IPs. This is intentional for environment resets.
- **Static inventory** — `inventory.ini` must be updated manually after provisioning. A dynamic inventory plugin (e.g., `amazon.aws.aws_ec2`) is recommended for team environments.
- **No secrets management** — credentials and connection strings are currently passed as variables. Integrate with AWS Secrets Manager or HashiCorp Vault before promoting to production.
- **Single AZ deployment** — all instances are provisioned in a single Availability Zone. Multi-AZ support is out of scope for this configuration.

---

## Contributing

1. Branch off `main` using the convention `feature/<short-description>` or `fix/<short-description>`
2. Test playbooks against the Dev environment before raising a PR
3. Ensure playbooks remain idempotent — running them twice should produce no unintended changes
4. Update this README if you add new services or change the execution order

---

## Maintainer

**Rahul Paladugu** — Infrastructure & Platform Engineering

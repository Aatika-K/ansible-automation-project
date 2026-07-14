# VM Monitor - Ansible Automation Project

![Ansible](https://img.shields.io/badge/Ansible-2.18.6-red?style=flat&logo=ansible)
![AWS](https://img.shields.io/badge/AWS-EC2-orange?style=flat&logo=amazon-aws)
![Python](https://img.shields.io/badge/Python-3.x-blue?style=flat&logo=python)
![License](https://img.shields.io/badge/License-MIT-green)

## Project Overview

This project demonstrates a **real-world DevOps automation use case** using Ansible to monitor node metrics (CPU usage, RAM usage, and Disk availability) across multiple virtual machines and automatically send a consolidated report via email.

### Use Case

Imagine managing 500+ VMs in a company where you need to regularly report node metrics to your manager. Manually checking each VM and compiling a report is tedious and error-prone. This project automates the entire workflow:

1. Fetch details of all target VMs dynamically (using AWS dynamic inventory)
2. Collect node metrics (CPU, RAM, Disk usage) from all VMs
3. Consolidate the data into a nicely formatted HTML report
4. Send the report automatically via email

<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/8893aa7a-2358-4a08-8e37-a26f8588e8b8" />

---

## Architecture

- **Ansible Master Node** – A single EC2 instance from which all Ansible commands/playbooks are executed (Ansible is agentless, so no installation is needed on target servers).
- **Target Servers** – Multiple EC2 instances (tagged with `environment: dev`) whose metrics need to be collected.
- **Dynamic Inventory** – Automatically fetches IP addresses of all target EC2 instances based on tags, instead of manually maintaining a static inventory file.
- **Email Reporting** – Uses Ansible's `mail` module with a Jinja2 HTML template to send a well-formatted report.

---

## Prerequisites

- AWS Account with permissions to create EC2 instances
- Basic understanding of Ansible, EC2, and Linux
- Gmail (or any SMTP provider) account with an **App Password** generated
- Key pair (`.pem` file) for SSH access to EC2 instances

### Ports Required

- **22** (SSH)
- **587** (SMTP)

---

## Setup Steps

### 1. Create Ansible Master Server

- Ubuntu 24.04 LTS
- Name: `Ansible-Master`
- Instance type: `t2.medium`
- Security group with ports `22` and `587` open
- Storage `20GB`

<img width="1277" height="112" alt="image" src="https://github.com/user-attachments/assets/19964590-3dd9-4b92-9af9-7f8e13e74a7b" />

### 2. Create Target Servers

- Create multiple EC2 instances (e.g., 10 VMs) tagged with `Environment: dev`
- Instance type: `t2.small`
- Same key pair as used for connecting
- Storage `15GB`

<img width="663" height="371" alt="image" src="https://github.com/user-attachments/assets/423014c7-2a86-41ff-91cb-40f92786c95a" />

Here you can see all instances have same name, we will change names by using script, which is given below.

### 3. Install Ansible on Master Node

Change hostname so it is easy for you to understand which machine you are working on 
sudo hostnamectl set-hostname Ansible-Master

```bash
# Step 1: Update the System
sudo apt update && sudo apt upgrade -y

# Step 2: Add the Ansible PPA
# Ansible provides an official maintained PPA (for latest versions):
sudo add-apt-repository --yes --update ppa:ansible/ansible

# Step 3: Install Ansible
sudo apt install ansible -y

# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install

```
### 4. Configure AWS CLI on Ansible Master

```bash
aws configure
```

Provide:
- Access Key
- Secret Access Key
- Region (e.g., `us-east-1`)

<img width="245" height="322" alt="image" src="https://github.com/user-attachments/assets/2aea6cb9-593b-4761-955c-7ab18fa2e398" />

<img width="1102" height="167" alt="image" src="https://github.com/user-attachments/assets/5118144b-97e9-495b-8f5b-b6d7dc0b8114" />

<img width="978" height="407" alt="image" src="https://github.com/user-attachments/assets/51ccde95-71e3-486f-8787-e34d6bb85357" />


### 5. Tag & Rename Instances

Use the `tag.sh` script to name instances dynamically (`web01`, `web02`, etc.) based on their tags using AWS CLI.

```bash
sudo nano tag.sh
```

```bash
#!/bin/bash

# Fetch instance IDs that match Environment=dev and Role=web
instance_ids=$(aws ec2 describe-instances \
  --filters "Name=tag:Environment,Values=dev" "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].InstanceId' \
  --output text)

# Sort instance IDs deterministically
sorted_ids=($(echo "$instance_ids" | tr '\t' '\n' | sort))

# Rename instances sequentially
counter=1
for id in "${sorted_ids[@]}"; do
  name="web-$(printf "%02d" $counter)"
  echo "Tagging $id as $name"
  aws ec2 create-tags --resources "$id" \
    --tags Key=Name,Value="$name"
  ((counter++))
done
```

```bash
sudo chmod +x tag.sh
./tag.sh
```

<img width="326" height="172" alt="image" src="https://github.com/user-attachments/assets/9a3ba604-7da8-411a-87de-08c75b603030" />

<img width="701" height="337" alt="image" src="https://github.com/user-attachments/assets/b493f208-bd45-43fa-88fb-a6fac85f5381" />

### 6. Setup Dynamic Inventory

Create `inventory/aws_ec2.yaml`:

```bash
mkdir inventory
nano inventory/aws_ec2.yaml
```

```yaml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
filters:
  tag:Environment: dev
  instance-state-name: running
compose:
  ansible_host: public_ip_address
keyed_groups:
  - key: tags.Name
    prefix: name
  - key: tags.Environment
    prefix: env     

```

```bash
# Step 1: Install venv module if not already present
sudo apt install python3-venv -y

# Step 2: Create a virtual environment
python3 -m venv ansible-env

# Step 3: Activate it
source ansible-env/bin/activate

# Step 4: Install required Python packages
pip install boto3 botocore docker

ansible-galaxy collection install amazon.aws

# Verify Inventory
ansible-inventory -i inventory/aws_ec2.yaml --graph
```

<img width="516" height="613" alt="image" src="https://github.com/user-attachments/assets/31d03aaf-40d3-47f9-9511-1ae09d6faaaa" />

### 7. Setup SSH Key-Based Authentication

Generate SSH key pair on Ansible master:

```bash
ssh-keygen -t rsa -b 4096 -C "Ansible-Master"
```

### 7. Copy Pub Key

nano copy-public-key.sh

```bash
#!/bin/bash

# Define vars
PEM_FILE="ansible-key.pem"
PUB_KEY=$(cat ~/.ssh/id_rsa.pub)
USER="ubuntu"  # or ec2-user
INVENTORY_FILE="inventory/aws_ec2.yaml"

# Extract hostnames/IPs from dynamic inventory
HOSTS=$(ansible-inventory -i $INVENTORY_FILE --list | jq -r '._meta.hostvars | keys[]')

for HOST in $HOSTS; do
  echo "Injecting key into $HOST"
  ssh -o StrictHostKeyChecking=no -i $PEM_FILE $USER@$HOST "
    mkdir -p ~/.ssh && \
    echo \"$PUB_KEY\" >> ~/.ssh/authorized_keys && \
    chmod 700 ~/.ssh && \
    chmod 600 ~/.ssh/authorized_keys
  "
done
```
### 8. Ansible Configuration File (`ansible.cfg`)

nano ansible.cfg
```ini
[defaults]
inventory = ./inventory/aws_ec2.yaml
host_key_checking = False

[ssh_connection]
ssh_args = -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null

```


Copy the public key to all target servers using the `copy_public_key.sh` script.

```bash
sudo chmod +x copy_public_key.sh
./copy_public_key.sh
```
<img width="910" height="465" alt="image" src="https://github.com/user-attachments/assets/4a9490a4-e933-4977-9dc1-970ae4b78b1e" />

<img width="620" height="410" alt="image" src="https://github.com/user-attachments/assets/7314c51e-d56f-4f0f-941f-e976885e758b" />

---

## Project Structure

```
vm-monitor/
│
├── ansible.cfg
├── inventory/
│   └── aws_ec2.yaml
├── group_vars/
│   └── all.yaml
├── templates/
│   └── report_email_animated.html.j2
├── playbooks/
│   ├── collect_metrics.yaml
│   ├── send_report.yaml
│   └── playbook.yaml
├── tag.sh
└── copy_public_key.sh
```

---

## Variables (`group_vars/all.yaml`)

```yaml
smtp_server: smtp.gmail.com
smtp_port: 587
email_user: your-email@gmail.com
email_password: your-app-password
alert_recipient: recipient-email@gmail.com
```

> **Note:** Use an **App Password**, not your actual Gmail password. This ensures your real credentials remain secure even if the app password is exposed.

---

## Playbooks

### `collect_metrics.yaml`

- Installs required packages (`sysstat` for Debian-based, `sysstat` via `yum` for RedHat-based systems)
- Collects:
  - **CPU Usage** via `mpstat`
  - **Memory Usage** via `free`
  - **Disk Usage** via `df`
- Stores results using `set_fact` into a `vm_metrics` dictionary per host

### `send_report.yaml`

- Aggregates metrics from all hosts
- Formats timestamp and subject line dynamically
- Sends an HTML email report using the `mail` module and Jinja2 template

### `playbook.yaml`

- Master playbook that imports both `collect_metrics.yaml` and `send_report.yaml`

```yaml
- import_playbook: playbooks/collect_metrics.yaml
- import_playbook: playbooks/send_report.yaml
```

---

## Execution

Run the complete automation with a single command:

```bash
ansible-playbook playbooks/playbook.yaml
```
<img width="950" height="277" alt="image" src="https://github.com/user-attachments/assets/e279f0f7-62ae-4341-ab51-deea3f5cb75d" />

<img width="1507" height="647" alt="image" src="https://github.com/user-attachments/assets/0f0c8dc1-bf7b-48dc-82d7-cc0a2b3a9e66" />


This will:

1. Connect to all target VMs
2. Collect CPU, Memory, and Disk metrics
3. Generate a consolidated HTML report
4. Send the report via email to the configured recipient

---

## Sample Output

The email report includes:

- DNS/IP addresses of all monitored VMs
- CPU usage (%)
- Memory usage (%)
- Disk usage (%)
- Average usage across all VMs
- Timestamp of report generation

---

## Key Learnings

- **Agentless architecture** of Ansible (no need to install anything on target nodes)
- **Dynamic Inventory** for managing large, scalable infrastructure
- **Ansible Modules** used: `package`, `yum`, `shell`, `set_fact`, `mail`
- **Jinja2 Templating** for HTML email generation
- **Ansible Vault / App Passwords** for secure credential handling
- **Automation scripts** (Bash) integrated with Ansible for tagging instances and SSH key distribution

---

## Notes

- Make sure security groups allow ports `22` and `587`
- Ensure the same key pair is used across Ansible master and target servers for SSH access
- Modify `group_vars/all.yaml` with your own SMTP and email details before running

---

## Resources

All scripts and playbooks used in this project are available in this repository.

---

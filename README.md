# VM Monitor - Ansible Automation Project

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
- Instance type: `t2.medium`
- Security group with ports `22` and `587` open

### 2. Create Target Servers

- Create multiple EC2 instances (e.g., 10 VMs) tagged with `environment: dev`
- Instance type: `t2.small`
- Same key pair as used for connecting

### 3. Install Ansible on Master Node

```bash
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
ansible --version
```

### 4. Tag & Rename Instances

Use the `tag.sh` script to name instances dynamically (`web01`, `web02`, etc.) based on their tags using AWS CLI.

```bash
sudo chmod +x tag.sh
./tag.sh
```

### 5. Configure AWS CLI on Ansible Master

```bash
aws configure
```

Provide:
- Access Key
- Secret Access Key
- Region (e.g., `ap-south-1`)

### 6. Setup Dynamic Inventory

Create `inventory/aws_ec2.yaml`:

```yaml
plugin: amazon.aws.aws_ec2
regions:
  - ap-south-1
filters:
  tag:environment: dev
  instance-state-name: running
compose:
  ansible_host: public_ip_address
groups:
  env_dev: true
```

Verify inventory:

```bash
ansible-inventory -i inventory/aws_ec2.yaml --graph
```

### 7. Setup SSH Key-Based Authentication

Generate SSH key pair on Ansible master:

```bash
ssh-keygen -t rsa -b 4096 -C "ansible-master"
```

Copy the public key to all target servers using the `copy_public_key.sh` script.

```bash
sudo chmod +x copy_public_key.sh
./copy_public_key.sh
```

### 8. Ansible Configuration File (`ansible.cfg`)

```ini
[defaults]
inventory = inventory/aws_ec2.yaml
host_key_checking = False

[ssh_connection]
ssh_args = -o StrictHostKeyChecking=no
```

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

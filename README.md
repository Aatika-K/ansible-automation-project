# VM Monitor - Ansible Automation Project

![Ansible](https://img.shields.io/badge/Ansible-2.18.6-red?style=flat&logo=ansible)
![AWS](https://img.shields.io/badge/AWS-EC2-orange?style=flat&logo=amazon-aws)
![Python](https://img.shields.io/badge/Python-3.x-blue?style=flat&logo=python)
![License](https://img.shields.io/badge/License-MIT-green)

## Table of Contents

- [Project Overview](#-project-overview)
- [Use Case](#-use-case)
- [Architecture](#️-architecture)
- [Prerequisites](#️-prerequisites)
- [Project Structure](#-project-structure)
- [Step-by-Step Implementation](#-step-by-step-implementation)
  - [1. Create Ansible Master Server](#1-create-ansible-master-server)
  - [2. Create Target Servers](#2-create-target-servers)
  - [3. Install Ansible on Master Node](#3-install-ansible-on-master-node)
  - [4. Tag & Rename Instances](#4-tag--rename-instances)
  - [5. Setup Dynamic Inventory](#5-setup-dynamic-inventory)
  - [6. Ansible Configuration File](#7-ansible-configuration-file)
  - [7. Setup SSH Key-Based Authentication](#6-setup-ssh-key-based-authentication)
  - [8. Create Project Variables](#8-create-project-variables)
  - [9. Create Playbooks](#9-create-playbooks)
  - [10. Create Email Template](#10-create-email-template)
- [Execution](#️-execution)
- [Key Learnings](#-key-learnings)
- [Important Notes](#-important-notes)
- [Resources](#-resources)
  

## Project Overview

This project demonstrates a **real-world DevOps automation use case** using Ansible to monitor node metrics (CPU usage, RAM usage, and Disk availability) across multiple virtual machines and automatically send a consolidated report via email.

Managing hundreds of VMs manually to check node metrics and report them is extremely hectic and not feasible. This project sets up complete automation using Ansible that:

- Automatically fetches details of all target virtual machines
- Collects node metrics like CPU usage, RAM usage, and disk space availability
- Consolidates everything into a nicely formatted HTML report
- Sends that report automatically via email to the manager or any required recipient
- Bonus: You can setup a cronjob to automatically send reports on specified time

## Use Case

**Scenario:** You are working in a company and are responsible for managing 100+ VMs. You are required to send node metrics (CPU usage, RAM usage, disk space availability) of all these VMs to your manager over email at a specific time — for example, at 3:00 PM daily.

Doing this manually for 100+ machines is not practical. Using Ansible, we can write scripts to automatically:

1. Fetch details of all VMs dynamically
2. Collect their node metrics (CPU, RAM, Disk usage)
3. Consolidate the information into a proper format
4. Send it via email automatically

The final email report includes:
- DNS names / IP addresses of all machines
- CPU usage percentage
- Memory usage percentage
- Disk usage / availability percentage
- Average usage across all machines

<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/8893aa7a-2358-4a08-8e37-a26f8588e8b8" />

---

## Architecture

- **Ansible Master Node** – A single EC2 instance from which all Ansible commands/playbooks are executed. Ansible is **agentless**, meaning we only need to install Ansible on this master node — no installation is required on target servers.
- **Target Servers** – Multiple EC2 instances (tagged with `Environment: dev`) whose metrics need to be collected.
- **Dynamic Inventory** – Automatically fetches IP addresses of all target EC2 instances based on tags, instead of manually maintaining a static inventory file (essential when managing large numbers of servers).
- **Email Reporting** – Uses Ansible's `mail` module along with a Jinja2 HTML template to send a well-formatted, animated report over email.

---

## Prerequisites

- AWS Account with permissions to create EC2 instances
- Basic understanding of Ansible, EC2, and Linux
- Gmail (or any SMTP provider) account with an **App Password** generated
- Key pair (`.pem` file) for SSH access to EC2 instances
- MobaXterm / any SSH client to connect to instances

### Required Ports

| Port | Purpose |
|------|---------|
| 22   | SSH access to instances |
| 587  | SMTP for sending email |

Make sure both ports are open in your EC2 Security Group.
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
│   └── report_email_animated_html.j2
├── playbooks/
│   ├── collect_metrics.yaml
│   ├── send_report.yaml
│   └── playbook.yaml
├── tag.sh
└── copy_public_key.sh
```

### File Descriptions

| File/Folder | Purpose |
|-------------|---------|
| `ansible.cfg` | Ansible configuration file — defines inventory path, disables host key checking |
| `inventory/aws_ec2.yaml` | Dynamic inventory file that fetches EC2 instance IPs automatically using AWS plugin |
| `group_vars/all.yaml` | Stores external variables like SMTP server, email credentials, recipient email |
| `templates/report_email_animated_html.j2` | Jinja2 HTML template used to format the email report |
| `playbooks/collect_metrics.yaml` | Playbook to collect CPU, RAM, and Disk metrics from target VMs |
| `playbooks/send_report.yaml` | Playbook to consolidate metrics and send the report via email |
| `playbooks/playbook.yaml` | Master playbook that imports both collect_metrics and send_report playbooks |
| `tag.sh` | Shell script to rename/tag EC2 instances dynamically (web01, web02, etc.) |
| `copy_public_key.sh` | Shell script to copy Ansible master's public SSH key to all target servers |


## Step-by-Step Implementation

### 1. Create Ansible Master Server

Create an EC2 instance with the following configuration:

- **OS:** Ubuntu 24.04 LTS
- **Instance Type:** `t2.medium`
- **Key Pair:** Use existing or create new (RSA format, `.pem`)
- **Security Group:** Open ports `22` and `587`
- **Storage:** 20 GB

This will be the machine from where we execute all Ansible commands.

<img width="1277" height="112" alt="image" src="https://github.com/user-attachments/assets/19964590-3dd9-4b92-9af9-7f8e13e74a7b" />

### 2. Create Target Servers

Create multiple EC2 instances (e.g., 10 VMs) that will be monitored:

- **Tag:** `environment: dev` (used later for dynamic inventory filtering)
- **OS:** Ubuntu 24.04 LTS
- **Instance Type:** `t2.small`
- **Key Pair:** Same key pair as used for the Ansible master (required for SSH access)
- **Security Group:** Port `22` open
- **Storage:** 15 GB each

> **Note:** All instances will initially be named the same (e.g., "web"). We will rename them uniquely (web01, web02, etc.) using a script in a later step.

<img width="663" height="371" alt="image" src="https://github.com/user-attachments/assets/423014c7-2a86-41ff-91cb-40f92786c95a" />

### 3. Install Ansible on Master Node

Connect to the Ansible master node via SSH (e.g., using MobaXterm) and run:

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

# Verify installation
ansible --version
```

### 4. Tag & Rename Instances

Since all target instances are named the same by default, we use a shell script (`tag.sh`) to rename them uniquely based on their tags using the AWS CLI.

**Install AWS CLI first:**

```bash
# Install AWS CLI
sudo apt install awscli -y
```

**Configure AWS credentials:**

1. Go to AWS Console → Profile → Security Credentials
2. Create a new Access Key (download the CSV if needed)

<img width="245" height="322" alt="image" src="https://github.com/user-attachments/assets/2aea6cb9-593b-4761-955c-7ab18fa2e398" />

<img width="1102" height="167" alt="image" src="https://github.com/user-attachments/assets/5118144b-97e9-495b-8f5b-b6d7dc0b8114" />

<img width="978" height="407" alt="image" src="https://github.com/user-attachments/assets/51ccde95-71e3-486f-8787-e34d6bb85357" />

3. Run the configure command on the Ansible master:

```bash
aws configure
```

Provide:
- AWS Access Key ID
- AWS Secret Access Key
- Default region (e.g., `ap-south-1`)
- Output format (press enter for default)


**Create and run the tagging script:**

```bash
vi tag.sh
# Paste the tag script content (you can find script in vm-monitor folder, same repo)
sudo chmod +x tag.sh
./tag.sh
```

This script filters instances by:
- Tag: `Environment=dev`
- State: `running`

And renames them sequentially as `web01`, `web02`, `web03`, etc.

Verify in AWS Console → EC2 → Instances that the naming has changed.

<img width="326" height="172" alt="image" src="https://github.com/user-attachments/assets/9a3ba604-7da8-411a-87de-08c75b603030" />

<img width="701" height="337" alt="image" src="https://github.com/user-attachments/assets/b493f208-bd45-43fa-88fb-a6fac85f5381" />

---

### 5. Configure AWS CLI on Ansible Master

Already covered in Step 4 — ensures the Ansible master can communicate with AWS to fetch instance details dynamically.

---

### 6. Setup Dynamic Inventory

Since manually maintaining IP addresses for 100+ machines is not feasible, we use a **dynamic inventory** file that automatically fetches IP addresses of all tagged EC2 instances.

**Create the inventory directory and file:**

```bash
mkdir inventory
vi inventory/aws_ec2.yaml

# Paste the inventory/aws_ec2.yaml file content (you can find this file in vm-monitor folder, same repo)

```

**File Explanation:**

| Key | Purpose |
|-----|---------|
| `plugin` | Specifies we're using the dynamic inventory plugin |
| `regions` | AWS region where instances are located |
| `filters` | Filters instances by tag (`Environment: dev`) and state (`running`) |
| `compose` | Defines `ansible_host` as the instance's public IP address |
| `groups` | Creates logical groups (by environment tag and by instance name) for easier targeting |

**Create a Python virtual environment for Ansible:**

```bash
sudo apt install python3-venv -y
python3 -m venv ansible-env
source ansible-env/bin/activate
```

**Install required Python dependencies:**

```bash
pip install boto3 botocore docker
```

**Verify the dynamic inventory:**

```bash
ansible-galaxy collection install amazon.aws
ansible-inventory -i inventory/aws_ec2.yaml --graph
```

This should display all EC2 instances grouped by tags and individual instance names.

<img width="516" height="613" alt="image" src="https://github.com/user-attachments/assets/31d03aaf-40d3-47f9-9511-1ae09d6faaaa" />

---

---

### 8. Ansible Configuration File

Create `ansible.cfg` in the project root to disable host key checking (since we don't want interactive `yes/no` prompts when connecting to 100+ servers):

```ini
[defaults]
inventory = inventory/aws_ec2.yaml
host_key_checking = False

[ssh_connection]
ssh_args = -o StrictHostKeyChecking=no
```

---

### 7. Setup SSH Key-Based Authentication

Generate an SSH key pair on the Ansible master node:

```bash
ssh-keygen -t rsa -b 4096 -C "ansible-master"
```

Press enter through all prompts (no passphrase needed).

**Copy the private key (used to create target instances) to the Ansible master:**

```bash
vi ansible-key.pem
# Paste the private key content and save
sudo chmod 400 ansible-key.pem
```

**Create and run the script to copy the public key to all target servers:** (you can find script in vm-monitor folder, same repo)

```bash
vi copy_public_key.sh
# Paste the script content
sudo chmod +x copy_public_key.sh
./copy_public_key.sh
```

This script:
- Uses the `.pem` file to connect to each target server
- Copies the Ansible master's public key to each server's `authorized_keys`
- Sets correct permissions on `.ssh` folder and `authorized_keys` file (Linux is strict about SSH key permissions)

**Test the connection:**

```bash
ssh ubuntu@<target-server-ip>
```

If this connects without a password prompt, the setup is successful.

<img width="910" height="465" alt="image" src="https://github.com/user-attachments/assets/4a9490a4-e933-4977-9dc1-970ae4b78b1e" />

<img width="620" height="410" alt="image" src="https://github.com/user-attachments/assets/7314c51e-d56f-4f0f-941f-e976885e758b" />

---

### 9. Create Project Variables

Create the `group_vars` folder and `all.yaml` file to store external/reusable variables:

```bash
mkdir group_vars
vi group_vars/all.yaml
```

```yaml
smtp_server: smtp.gmail.com
smtp_port: 587
email_user: your-email@gmail.com
email_password: your-app-password
alert_recipient: recipient-email@gmail.com
```

**⚠️ Important — Generating an App Password:**

Never use your actual Gmail password. Instead, generate an **App Password**:

1. Go to [Google Account Security](https://myaccount.google.com/security)
2. Enable 2-Step Verification (if not already enabled)
3. Navigate to **App Passwords**
4. Create a new app password (name it e.g., "VM Monitor")
5. Copy the generated password and use it as `email_password` (don't remove spaces from app password)

This ensures that even if the password is exposed, it can only be used with the specific application (Ansible) and not to fully access your Gmail account.

---

### 10. Create Playbooks 

#### `playbooks/collect_metrics.yaml`  (you can find this file in vm-monitor folder, same repo)

This playbook collects CPU, RAM, and Disk metrics from all target VMs.

**Key concepts used:**

| Directive | Purpose |
|-----------|---------|
| `hosts: env_dev` | Targets all VMs in the `env_dev` group |
| `become: true` | Executes commands with elevated (sudo) permissions |
| `gather_facts: true` | Collects system information about remote hosts |

**Tasks performed:**

1. **Install `sysstat` package** (provides `mpstat` command for CPU stats)
   - Uses `apt` module for Debian/Ubuntu family (`when: ansible_os_family == "Debian"`)
   - Uses `yum` module for RedHat/CentOS family (`when: ansible_os_family == "RedHat"`)

2. **Collect CPU Usage:**

   ```bash
   mpstat 1 1 | awk '...'
   ```

   Fetches `%idle` value and calculates usage as `100 - idle`, stored in a registered variable.

3. **Collect Memory Usage:**

   ```bash
   free | awk '...'
   ```

   Extracts memory usage percentage and stores it in a registered variable.

4. **Collect Disk Usage:**

   ```bash
   df | awk '...'
   ```

   Extracts disk usage percentage and stores it in a registered variable.

5. **Consolidate metrics using `set_fact`:**

   ```yaml
        vm_metrics:
          hostname: "{{ inventory_hostname }}"
          cpu: "{{ cpu_usage.stdout | float | round(2) }}"
          mem: "{{ mem_usage.stdout | float | round(2) }}"
          disk: "{{ disk_usage.stdout | float | round(2) }}"
   ```

#### `playbooks/send_report.yaml`

This playbook consolidates metrics from all hosts and sends the report via email.

**Key concepts used:**

| Directive | Purpose |
|-----------|---------|
| `hosts: localhost` | Runs on the Ansible master itself (email sending doesn't need remote execution) |
| `vars.collected_metrics` | Aggregates `vm_metrics` from all hosts using `hostvars`, `dict2items`, `selectattr`, and `map` filters |
| `timestamp` | Uses Ansible's built-in `ansible_date_time` for dynamic date/time in report |

**Main task:**

```yaml
    - name: Send animated HTML report via email
      mail:
        host: "{{ smtp_server }}"
        port: "{{ smtp_port }}"
        username: "{{ email_user }}"
        password: "{{ email_pass }}"
        to: "{{ alert_recipient }}"
        subject: "{{ subject_line }}"
        body: "{{ lookup('template', 'templates/report_email_animated.html.j2') }}"
        subtype: html
```

#### `playbooks/playbook.yaml`

Master playbook that imports both playbooks in sequence:

```yaml
- import_playbook: collect_metrics.yaml
- import_playbook: send_report.yaml
```

---

### 11. Create Email Template

Create the HTML/Jinja2 template used to format the report email:

```bash
mkdir templates
vi templates/report_email_animated_html.j2 (you can find this file in vm-monitor folder, same repo)
```

This file contains HTML and CSS to render a well-structured, animated report with:
- A table listing all VM hostnames/IPs
- CPU usage percentage per VM
- Memory usage percentage per VM
- Disk usage percentage per VM
- Average usage statistics across all VMs

---

## Execution

Once all files and configurations are in place, run the master playbook:

```bash
ansible-playbook playbooks/playbook.yaml
```

**What happens during execution:**

1. Ansible connects to all target VMs (fetched dynamically via inventory)
2. Installs required monitoring packages (`sysstat`) if not already present
3. Collects CPU, Memory, and Disk usage from each VM
4. Aggregates all collected metrics on the Ansible master (localhost)
5. Generates a formatted HTML report using the Jinja2 template
6. Sends the report via email to the configured recipient

Execution time depends on the number of VMs (approx. a few minutes for 10 VMs).

Upon successful execution, the recipient receives an email containing:

| Field | Description |
|-------|--------------|
| DNS/IP Address | Identifies each monitored VM |
| CPU Usage | Percentage of CPU utilized |
| Memory Usage | Percentage of RAM utilized |
| Disk Usage | Percentage of disk space utilized |
| Average Usage | Average CPU/Memory/Disk usage across all VMs |
| Timestamp | Date and time the report was generated |


<img width="950" height="277" alt="image" src="https://github.com/user-attachments/assets/e279f0f7-62ae-4341-ab51-deea3f5cb75d" />

<img width="1507" height="647" alt="image" src="https://github.com/user-attachments/assets/0f0c8dc1-bf7b-48dc-82d7-cc0a2b3a9e66" />

---


## Key Learnings

- **Agentless architecture** of Ansible — no need to install anything on target nodes, only SSH access is required
- **Dynamic Inventory** using the `amazon.aws.aws_ec2` plugin for managing large, scalable, cloud-based infrastructure
- **Tag-based filtering** of AWS resources for targeted automation
- **Ansible Modules** used: `package`, `apt`, `yum`, `shell`, `set_fact`, `mail`
- **Jinja2 Templating** for generating dynamic, styled HTML email reports
- **Variable management** using `group_vars` for reusable, external configuration
- **App Passwords** for secure email credential handling instead of actual account passwords
- **Bash scripting** integrated with Ansible for tagging EC2 instances and automating SSH key distribution
- **SSH key-based authentication** setup for passwordless, prompt-free connections across multiple servers
- Filtering command output using Linux tools (`awk`, `mpstat`, `free`, `df`) to extract precise metrics

---

## Important Notes

- Ensure security groups allow inbound traffic on ports `22` (SSH) and `587` (SMTP)
- Use the **same key pair** across the Ansible master and target servers for SSH access to work correctly
- Never hardcode your actual email password — always use an **App Password**
- For production use, consider storing sensitive variables (like `email_password`) using **Ansible Vault** instead of plain text in `group_vars/all.yaml`
- Modify `group_vars/all.yaml` with your own SMTP server details, email credentials, and recipient email before running the playbook
- This project currently targets a `dev` environment tag — modify the tag filter in `inventory/aws_ec2.yaml` if using different environment tags (e.g., `staging`, `prod`)

---

## Resources

All scripts, playbooks, and configuration files used in this project are available in this repository:

- `tag.sh` — Script to tag/rename EC2 instances
- `copy_public_key.sh` — Script to distribute SSH public keys
- `inventory/aws_ec2.yaml` — Dynamic inventory configuration
- `group_vars/all.yaml` — Project variables
- `playbooks/collect_metrics.yaml` — Metrics collection playbook
- `playbooks/send_report.yaml` — Email reporting playbook
- `playbooks/playbook.yaml` — Master playbook
- `templates/report_email_animated_html.j2` — HTML email template

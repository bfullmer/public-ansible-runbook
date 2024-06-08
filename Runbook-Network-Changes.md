# 🚀 Chief AI Agent's Ultimate Runbook Script Setup and Usage Guide

Welcome to the setup and usage guide for the Chief AI Agent's Runbook Script! This guide will help you manage network configurations, including backing up, updating IP addresses, and handling reboots. It supports dry runs for testing without making actual changes. Let's get started! 🎉

## Summary

The runbook script is designed to automate the process of managing network configurations on your servers. It includes features such as:

- Backing up existing network configuration files.
- Updating IP addresses based on a provided CSV file.
- Scheduling IP address changes and reboots.
- Running in agent mode to periodically check and apply updates.
- Logging all actions for easy troubleshooting.

The guide will walk you through the prerequisites, setting up variables, running the script, enabling agent mode, and troubleshooting common issues. Whether you're performing a dry run or applying real changes, this script aims to simplify network management tasks.

## 📋 Table of Contents

1. [Prerequisites](#prerequisites)
2. [Variables](#variables)
3. [Steps to Run the Script](#steps-to-run-the-script)
4. [Agent Mode Setup](#agent-mode-setup)
5. [Example YAML Configuration](#example-yaml-configuration)
6. [Troubleshooting](#troubleshooting)
7. [FAQ](#faq)
8. [Contact](#contact)

---

**Prerequisites**

Before running the script, make sure you have the following:

- Ansible Vault password for decrypting the inventory file.
- Required system tools (`atd`, `awk`, `sed`, `nmcli`) installed and configured.

### Installing Prerequisites

**Install `atd` Service**:
```sh
sudo apt-get install at -y   # For Debian-based systems
sudo yum install at -y       # For Red Hat-based systems
sudo systemctl start atd
sudo systemctl enable atd
```

**Install `awk`, `sed`, and `nmcli`**:
```sh
sudo apt-get install gawk sed network-manager -y  # For Debian-based systems
sudo yum install gawk sed NetworkManager -y       # For Red Hat-based systems
```

---

**Variables**

The script uses several variables, which need to be set before running. These include the ticket number, user, vault password, inventory file path, CSV file path, IP change time, interface, reboot options, timezone, log file path, dry run flag, agent mode flag, agent interval, and the inventory array.

---

**Steps to Run the Script**

1. **Clone the Repository**:
   ```sh
   git clone https://github.com/your-repo/runbook-script.git
   cd runbook-script
   ```

2. **Set Variables**:
   Edit the runbook YAML file to set the necessary variables.

3. **Run the Script**:
   Execute the runbook script using a YAML interpreter.
   ```sh
   ansible-playbook runbook.yml
   ```

4. **Perform a Dry Run**:
   Modify the YAML configuration to perform a dry run by setting `dry_run` to `true`.

5. **Check the Logs**:
   Verify the logs to ensure the script executed correctly.
   ```sh
   tail -f /home/admin/{{ ticket_number }}/runbook.log
   ```

---

**Agent Mode Setup**

Enable agent mode to run the script automatically at specified intervals.

1. **Enable Agent Mode**:
   Set `agent_mode` to `true` in the YAML file and specify the `agent_interval`.

2. **Create Agent Script**:
   Create a script `runbook_agent.sh` to run the runbook in a loop.

3. **Set Up Service**:
   Create a systemd service to run the agent script.

4. **Enable and Start the Service**:
   ```sh
   sudo systemctl enable runbook_agent.service
   sudo systemctl start runbook_agent.service
   ```

---

**Example YAML Configuration**

Here's an example configuration for the runbook YAML file:

```yaml
runbook:
  variables:
    ticket_number: "240607-12345"
    user: "admin"
    vault_password: "your-vault-password"
    inventory_file: "/path/to/encrypted-inventory-file"
    csv_file: "/home/admin/240607-12345/ip_changes.csv"
    ip_change_time: "2024-06-07 15:00"
    interface: "Ethernet0"
    reboot_option: "scheduled"
    reboot_time: "2024-06-07 16:00"
    timezone: "UTC"
    log_file: "/home/admin/240607-12345/runbook.log"
    dry_run: false
    agent_mode: true
    agent_interval: 3600
    inventory: []
  steps:
    - step: "Initialize"
      description: "Start the runbook process"
      actions:
        - action: "Log start"
          command: "echo 'Runbook started at $(date)' >> {{ log_file }}"
        - action: "Set ticket number"
          input: "{{ ticket_number }}"
        - action: "Set user"
          input: "{{ user }}"
        - action: "Prompt for Ansible Vault password"
          input: "{{ vault_password }}"
          prompt: "Please enter the Ansible Vault password: "
    - step: "Ensure atd is running"
      description: "Ensure the atd service is running"
      actions:
        - action: "Start atd"
          command: "systemctl start atd && echo 'atd started' >> {{ log_file }} || echo 'Failed to start atd' >> {{ log_file }}"
        - action: "Enable atd"
          command: "systemctl enable atd && echo 'atd enabled' >> {{ log_file }} || echo 'Failed to enable atd' >> {{ log_file }}"
    - step: "Ensure Path Exists"
      description: "Check if the directory for the ticket number exists"
      actions:
        - action: "Check directory"
          command: |
            if [[ ! -d "/home/admin/{{ ticket_number }}" ]]; then
              echo "Directory /home/admin/{{ ticket_number }} does not exist. Creating it." >> {{ log_file }}
              mkdir -p /home/admin/{{ ticket_number }} && echo 'Directory created' >> {{ log_file }} || echo 'Failed to create directory' >> {{ log_file }}
            else
              echo 'Directory already exists' >> {{ log_file }}
            fi
    - step: "Decrypt Inventory"
      description: "Decrypt the inventory file using Ansible Vault"
      actions:
        - action: "Run Ansible Vault"
          command: "if [[ {{ dry_run }} = false ]]; then ansible-vault decrypt {{ inventory_file }} --ask-vault-pass && echo 'Inventory decrypted' >> {{ log_file }} || echo 'Failed to decrypt inventory' >> {{ log_file }}; else echo 'Dry run: Skipping decryption' >> {{ log_file }}; fi"
    - step: "Verify Ticket"
      description: "Verify the ticket number"
      actions:
        - action: "Check ticket number"
          input: "{{ ticket_number }}"
          command: |
            if [[ ! $ticket_number =~ ^[0-9]{6}-[0-9]{5}$ ]]; then
              echo "Invalid ticket number format. Exiting." >> {{ log_file }}
              exit 1
            else
              echo 'Ticket number verified' >> {{ log_file }}
            fi
    - step: "Backup Configuration"
      description: "Back up the Ethernet configuration file"
      actions:
        - action: "Backup ifcfg"
          command: "if [[ {{ dry_run }} = false ]]; then cp /etc/sysconfig/network-scripts/ifcfg-{{ interface }} /home/admin/{{ ticket_number }}/ifcfg-{{ interface }}.orig && echo 'Configuration backed up' >> {{ log_file }} || echo 'Backup failed. Exiting.' >> {{ log_file }}; exit 1; else echo 'Dry run: Skipping backup' >> {{ log_file }}; fi"
    - step: "Load Inventory"
      description: "Load IP address mappings from CSV file"
      actions:
        - action: "Load CSV"
          script: |
            if [[ ! -f {{ csv_file }} ]]; then
              echo "CSV file not found. Exiting." >> {{ log_file }}
              exit 1
            else
              echo 'CSV file found' >> {{ log_file }}
            fi
            inventory=()
            while IFS=, read -r old_ip new_ip; do
              inventory+=("$old_ip $new_ip")
              echo "Loaded mapping: $old_ip -> $new_ip" >> {{ log_file }}
            done < {{ csv_file }}
            echo "Inventory loaded with ${#inventory[@]} entries" >> {{ log_file }}
    - step: "Schedule IP Address Change"
      description: "Schedule the IP address change"
      when: "{{ ip_change_time != '' }}"
      actions:
        - action: "Schedule with at"
          command: |
            for mapping in "${inventory[@]}"; do
              old_ip=$(echo $mapping | awk '{print $1}')
              new_ip=$(echo $mapping | awk '{print $2}')
              if [[ {{ dry_run }} = false ]]; then
                echo "sed -i 's/$old_ip/$new_ip/g' /etc/sysconfig/network-scripts/ifcfg-{{ interface }} && nmcli con reload {{ interface }} && echo 'IP address updated at scheduled time' >> {{ log_file }} || echo 'Failed to update IP address at scheduled time' >> {{ log_file }}" | at -t $(TZ={{ timezone }} date -d '{{ ip_change_time }}' +%Y

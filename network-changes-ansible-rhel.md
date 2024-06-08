# Network Changes Code Runbook with Ansible for RHEL 7 and RHEL 8 Systems

## Purpose
A concise guide to execute network changes using Ansible on RHEL 7 and RHEL 8 systems with IP changes from a CSV file.

## Prerequisites
1. AWS CLI installed and configured.
2. Necessary IAM permissions.
3. Ansible installed.
4. Inventory file and CSV file with IP changes.
5. Backup of current network settings.
6. Access to RHEL 7 or RHEL 8 systems.

## Steps

### 1. Preparation
#### a. Backup Configuration
```bash
# Export current security groups
aws ec2 describe-security-groups --output json > /var/backups/security-groups-backup.json

# Export current route tables
aws ec2 describe-route-tables --output json > /var/backups/route-tables-backup.json
```

#### b. Notification
Inform stakeholders of the planned changes via email or communication tools.

### 2. Implement Changes
#### a. Update Ansible Inventory
Create a playbook to update the inventory with new IPs from the CSV file.

**CSV File (ip_changes.csv) Format:**
```
host1,new_ip1
host2,new_ip2
```

**Ansible Playbook (update_inventory.yml):**
```yaml
---
- name: Update Ansible inventory with new IP addresses
  hosts: localhost
  vars:
    csv_file: "/path/to/ip_changes.csv"
    inventory_file: "/path/to/inventory"

  tasks:
    - name: Read CSV file
      read_csv:
        path: "{{ csv_file }}"
      register: ip_changes

    - name: Update inventory
      lineinfile:
        path: "{{ inventory_file }}"
        regexp: "^{{ item.host }}"
        line: "{{ item.host }} ansible_host={{ item.new_ip }}"
      loop: "{{ ip_changes.list }}"
```

Run the playbook to update the inventory:
```bash
ansible-playbook update_inventory.yml
```

#### b. Apply Network Changes with Ansible
Create a playbook to modify security groups and route tables.

**Ansible Playbook (network_changes.yml):**
```yaml
---
- name: Apply network changes
  hosts: all
  become: yes
  tasks:
    - name: Update Security Group to allow HTTP traffic
      shell: |
        aws ec2 authorize-security-group-ingress \
          --group-id sg-12345678 \
          --protocol tcp \
          --port 80 \
          --cidr 0.0.0.0/0
      when: ansible_distribution == "RedHat" and ansible_distribution_major_version in ["7", "8"]

    - name: Update Route Table with new route
      shell: |
        aws ec2 create-route \
          --route-table-id rtb-12345678 \
          --destination-cidr-block 10.0.2.0/24 \
          --gateway-id igw-12345678
      when: ansible_distribution == "RedHat" and ansible_distribution_major_version in ["7", "8"]
```

Run the playbook:
```bash
ansible-playbook -i /path/to/inventory network_changes.yml
```

### 3. Verification
#### a. Connectivity Tests
```bash
# Example: Check connectivity to a specific instance
ping <instance-public-ip>
```

#### b. Security Validation
```bash
# Check security group rules
aws ec2 describe-security-groups --group-ids sg-12345678

# Check route table routes
aws ec2 describe-route-tables --route-table-ids rtb-12345678
```

### 4. Rollback (if needed)
#### a. Revert Changes
**Ansible Playbook (rollback_changes.yml):**
```yaml
---
- name: Revert network changes
  hosts: all
  become: yes
  tasks:
    - name: Remove added security group rule
      shell: |
        aws ec2 revoke-security-group-ingress \
          --group-id sg-12345678 \
          --protocol tcp \
          --port 80 \
          --cidr 0.0.0.0/0
      when: ansible_distribution == "RedHat" and ansible_distribution_major_version in ["7", "8"]

    - name: Remove added route
      shell: |
        aws ec2 delete-route \
          --route-table-id rtb-12345678 \
          --destination-cidr-block 10.0.2.0/24
      when: ansible_distribution == "RedHat" and ansible_distribution_major_version in ["7", "8"]
```

Run the playbook:
```bash
ansible-playbook -i /path/to/inventory rollback_changes.yml
```

#### b. Retest
Repeat connectivity and security tests to ensure the network is restored to its previous state.

### 5. Documentation
#### a. Update Records
Document the changes and results in your configuration management system or a simple log file.

```bash
echo "Network changes implemented on $(date)" >> /var/log/network-changes-log.txt
echo "Details: Added HTTP rule to sg-12345678, Updated route for rtb-12345678" >> /var/log/network-changes-log.txt
```

---

Chief AI Agent

# **Ansible Service Restart & Validation Automation**

This project automates **restarting** and **validating** services on remote servers using Ansible. While examples use **Tomcat** and **Nginx**, the framework is **fully adaptable** for any service (PostgreSQL, Redis, custom apps, etc.).

## **1. Overview**

This automation provides:

* Remote restart of one or more services
* Validation to ensure services are running and reachable
* Flexible configuration for **cloud (AWS EC2)**, **on-premises servers**, **containers**, or **hybrid environments**


## **2. Prerequisites**

### Control Node Requirements

* **Ansible 2.13+** (or latest stable)
* **Python 3.x**
* SSH access to target servers

### Managed Host Requirements

* SSH enabled
* Python 3 installed
* Services managed by `systemd` or a supported init system


## **3. Installation**

### Install Ansible

**Amazon Linux / RHEL:**

    ```bash
    sudo yum update -y
    sudo amazon-linux-extras enable ansible2
    sudo yum install ansible -y
    ```

**Ubuntu / Debian:**

    ```bash
    sudo apt update
    sudo apt install ansible -y
    ```

Verify installation:

    ```bash
    ansible --version
    ```

## **4. Project Structure**

A typical setup:

    ```
    .
    |-- ansible.cfg                # Ansible configuration
    |-- README.md                  # This guide
    |-- site.yml                   # Main orchestration playbook
    |
    |-- inventories
    |   |-- aws_hosts.yml          # Example AWS inventory (YAML)
    |   `-- production
    |       |-- hosts.ini          # Production inventory (INI)
    |       `-- group_vars
    |           `-- all.yml        # Global variables
    |
    |-- playbooks
    |   |-- restart.yml            # Restart services
    |   |-- rollback.yml           # Placeholder rollback logic
    |   `-- status.yml             # Health/status checks
    |
    `-- roles
        |-- nginx
        |   |-- defaults/main.yml
        |   |-- handlers/main.yml
        |   `-- tasks/main.yml
        |
        `-- tomcat
            |-- defaults/main.yml
            |-- handlers/main.yml
            `-- tasks/main.yml
    ```

## **5. Configuration**

### 5.1 Inventory Setup

Define servers and groupings in **inventories**:

**Example (AWS):**

    ```yaml
    all:
    children:
        nginx_servers:
        hosts:
            nginx1:
            ansible_host: 13.60.104.189
            ansible_user: ubuntu
            ansible_ssh_private_key_file: ~/.ssh/nginx-key.pem
        tomcat_servers:
        hosts:
            tomcat1:
            ansible_host: 13.53.165.246
            ansible_user: ec2-user
            ansible_ssh_private_key_file: ~/.ssh/tomcat-key.pem
    ```

**Example (Production INI):**

    ```ini
    [tomcat]
    tomcat1.example.com

    [nginx]
    nginx1.example.com
    ```

**Group Variables (inventories/production/group\_vars/all.yml):**

    ```yaml
    ansible_user: deploy_user
    ansible_become: true
    service_restart_retries: 5
    service_restart_delay: 5
    tomcat_port: 8080
    nginx_port: 80
    ```

**Adaptation Options:**

    * Add new groups (e.g., `database`, `cache`)
    * Use different SSH users and keys per host
    * Integrate with dynamic inventories (AWS, GCP, Kubernetes)


### 5.2 SSH Key Permissions

    ```bash
    chmod 600 ~/.ssh/nginx-key.pem
    chmod 600 ~/.ssh/tomcat-key.pem
    ```

### 5.3 Service Roles

Each role defines:

1. **Restart logic**
2. **Validation logic**

**Example: Tomcat (`roles/tomcat/tasks/main.yml`)**

    ```yaml
    - name: Restart Tomcat
    systemd:
        name: tomcat
        state: restarted

    - name: Verify Tomcat is serving traffic
    uri:
        url: http://localhost:8080
        status_code: 200
    ```

**Example: Nginx (`roles/nginx/tasks/main.yml`)**

    ```yaml
    - name: Restart Nginx
    systemd:
        name: nginx
        state: restarted

    - name: Verify Nginx is reachable
    uri:
        url: http://localhost
        status_code: 200
    ```

**To adapt for other services:**

    * Replace `systemd` name with your service
    * Modify validation (TCP port checks, database queries, API health checks)

### 5.4 Main Playbook (`site.yml`)

    ```yaml
    - name: Restart and Validate Tomcat
    hosts: tomcat_servers
    roles:
        - tomcat

    - name: Restart and Validate Nginx
    hosts: nginx_servers
    roles:
        - nginx
    ```

## **6. Running the Automation**

### 6.1 Test Connectivity

    ```bash
    ansible -i inventories/aws_hosts.yml all -m ping
    ```

**Expected output:**

    ```
    nginx1 | SUCCESS => {"ping": "pong"}
    tomcat1 | SUCCESS => {"ping": "pong"}
    ```

### 6.2 Restart Services

    ```bash
    ansible-playbook -i inventories/aws_hosts.yml playbooks/restart.yml
    ```

### 6.3 Check Service Status

    ```bash
    ansible-playbook -i inventories/aws_hosts.yml playbooks/status.yml
    ```

### 6.4 Run Full Automation

    ```bash
    ansible-playbook -i inventories/aws_hosts.yml site.yml
    ```

This will:

* Restart and validate all defined services
* Output a summary of successes/failures


### 6.5 Rollback (Placeholder)

    ```bash
    ansible-playbook -i inventories/aws_hosts.yml playbooks/rollback.yml
    ```

    *(Extend with real rollback logic: previous deployments, DB restores, container reversion.)*


## **7. Adapting for Your Environment**

1. **Add new roles** for additional services:

   ```bash
   ansible-galaxy init roles/<service-name>
   ```
2. **Update `site.yml`** to include new services.
3. **Customize validation** per service:

   * HTTP or TCP checks
   * Database queries
   * Custom health scripts


## **8. Troubleshooting**

* **404 on HTTP services:** Ensure the service has a deployed app or default page.
* **Python missing on managed hosts:**

  ```bash
  sudo yum install python3 -y   # RHEL/Amazon Linux
  sudo apt install python3 -y   # Ubuntu
  ```
* **Permission issues:** Ensure SSH user has `sudo` privileges.
* **Connection errors:** Verify SSH keys, firewall rules, and host reachability.


## **9. Security & Cleanup Before Sharing**

* Remove unused roles:

  ```bash
  rm -rf roles/<unused-role>
  ```
* Exclude sensitive files:

  ```bash
  echo "*.pem" >> .gitignore
  ```
* Confirm no credentials or secrets are committed.

## **10. How to Extend**

* Use dynamic inventories (cloud, containers)
* Implement rolling restarts for zero downtime
* Integrate with CI/CD pipelines
* Add monitoring hooks for proactive alerting

## **11. Architecture Overview**

    ```
    Control Node (Ansible)
        |
        |--- SSH
        |
    Managed Hosts
    ├── Application Servers (Tomcat, etc.)
    └── Web Servers (Nginx, etc.)
    ```

This architecture scales with additional service groups (databases, caches) and supports hybrid/cloud-native deployments.
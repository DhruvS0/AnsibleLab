# Ansible Lab Project
## Overview
This project sets up a multi-server environment using Ansible. It includes three primary roles: *Web Server*, *Database Server*, and *Monitoring Server*. The goal of this lab is to demonstrate intermediate-level Ansible skills, including role-based configuration, dynamic inventories, and monitoring.

## Architecture 
The lab environment consists of:
- Web Server (web01)
    - Role: Hosts web applications and services.
    - Software: Nginx
- Database Server (db01)
    - Role: Hosts databases.
    - Software: MongoDB
- Monitoring Server (monitoring)
    - Role: Monitors the web and database servers.
    - Software: Prometheus, Grafana

## Diagram
```lua
        +-----------------+      +----------------- +      +-----------------+
        |    web01        |      |       db01       |      |    monitoring   |
        | (Web Server)    |      | (Database Server)|      |   (Monitoring)  |
        |                 |      |                  |      |                 |
        | +-------------+ |      | +-------------+  |      | +-------------+ |
        | |   Nginx       | |    | |    MySQL    |  |      | | Prometheus  | |
        | | (Web App)   | |      | |    (DB)     |  |      | | (Grafana)   | |
        | +-------------+ |      | +-------------+  |      | +-------------+ |
        +-----------------+      +------------------+      +-----------------+

                                Ansible Control Node
                                +------------------+
                                |     Ansible      |
                                |     Playbooks    |
                                |     Roles        |
                                |     Inventory    |
                                +------------------+

```
## Project Structure
```css
ansible-lab/
  ├── inventory.ini
  ├── site.yml
  └── roles/
      ├── web/
      │   ├── tasks/
      │   │   └── main.yml
      │   └── handlers/
      │       └── main.yml
      ├── db/
      │   └── tasks/
      │       └── main.yml
      |
      └── monitoring/
          ├── tasks/
          │   └── main.yml
          └── handlers/
              └── main.yml
```
## Setting Up Ubuntu VMs for the Ansible Lab

This section describes how to set up the Ubuntu VMs for the Ansible lab environment. We will set up one control node and three managed nodes: web, db, and monitoring. All VMs will have a common user **ansible** with the password **ansible**.

### 1. Create the VMs

Create four Ubuntu Vms using your preferred virtualization software (e.g., VirtualBox, VMware, etc.).

Name the VMs as follows:
- Control Node: **ansible-control**
- Web Server: **web01**
- Database Server: **db01**
- Monitoring Server: **monitoring**

### 2. Set Up the Control Node
- **Install Ubuntu:** Install Ubuntu on the control node and update the packages and install net-tools.
```sh
sudo apt update 
sudo apt upgrade -y
sudo apt install net-tools #installing net-tools package
```
- **Install Ansible:**  Install Ansible on the control node.
```sh
sudo apt update
sudo apt install ansible
```
- **Install 'sshpass':**  Install sshpass to allow password-based SSH connections.
```sh
sudo apt-get update
sudo apt-get install sshpass
```
- **Generate SSH Key:** Generate an SSH key pair if you haven't already.
```sh
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

### 3. Set Up Managed Nodes
For each managed node (web01, db01, and monitoring):
- **Install Ubuntu:** Install Ubuntu on the managed nodes.
- **Create a User:** Create a user named ansible with the password ansible.
```sh
sudo adduser ansible
sudo usermod -aG sudo ansible
```
- **Configure SSH for the *ansible* User:**
    - Switch to the *ansible* user.
    ```sh
    su -ansible
    ```
    - Create an .ssh directory and set permissions:
    ```sh
    mkdir ~/.ssh
    chmod 700 ~/.ssh
    ```
    - Add the control node's public key to the *authorized_keys* file:
    ```sh
    echo "your_public_key" >> ~/.ssh/authorized_keys
    chmod 600 ~/.ssh/authorized_keys

    # Note: Replace your_public_key with the content of your control node's public key (usually found in ~/.ssh/id_rsa.pub).
    ```
- **Install Required Packages:** Ensure that essential packages are installed.
```sh
sudo apt update
sudo apt install python3 sudo
```

### 4. Configure SSH Key-Based Authentication.
On your control node, copy the public key to each of the managed nodes:

```sh
ssh-copy-id ansible@your-web_server-ip #web01
ssh-copy-id ansible@your-database_server-ip #db01
ssh-copy-id ansible@your-monitoring_server-ip #monitoring
```

### 5. Test SSH Connectivity
Test the SSH connectivity from the control node to each managed node:

```sh
ssh ansible@your-web_server-ip #web01
ssh ansible@your-database_server-ip #db01
ssh ansible@your-monitoring_server-ip #monitoring
```
**Note:** You should be able to log in to each managed node without being prompted for a password.

### 6. Configure Ansible Inventory
Create an inventory file named inventory.ini on the control node:
```ini
[web]
web01 ansible_host=your-web_server-ip #web01

[database]
db01 ansible_host=your-database_server-ip #db01

[monitoring]
monitoring ansible_host=your-monitoring_server-ip #monitoring

[all:vars]
ansible_ssh_user=ansible
ansible_ssh_pass=ansible
```
### 7. Test Ansible Connectivity
Test the connection to all the hosts using Ansible:
```sh
ansible all -i inventory.ini -m ping --ask-pass
```
By following these steps, you will set up a control node and three managed nodes (web, db, monitoring) with a common user ansible. This setup will allow you to use Ansible to automate the configuration and management of your lab environment.

## Roles for Ansible Playbook

### General Role Structure
```css
roles/
  myrole/
    tasks/
      main.yml
    handlers/
      main.yml
    templates/
      ...
    files/
      ...
    vars/
      main.yml
    defaults/
      main.yml
    meta/
      main.yml
```
### Role for Web Server (web01)
**roles/web/tasks/main.yml**
```yaml
---
- name: Install Apache
  apt: 
    name: apache2
    state: present
  notify: Restart Apache

- name: Start and enable Apache
  systemd:
    name: apache2
    state: started
    enabled: yes
```
**roles/web/handlers/main.yml**
```yaml
---
- name: Restart Apache
  systemd: 
    name: apache2
    state: restarted
```
### Role for Database Server (db01)
**roles/db/tasks/main.yml**
```yaml
---
- name: Ensure no other apt processes are running
  shell: |
    while fuser /var/lib/dpkg/lock-frontend >/dev/null 2>&1; do
      echo "Waiting for other apt processes to exit..."
      sleep 2
    done

- name: Install MySQL
  apt:
    name: mysql-server
    state: present

- name: Start and enable MySQL
  systemd:
    name: mysql
    state: started
    enabled: yes
```
### Role for Monitoring Server (monitoring)
**roles/monitoring/tasks/main.yml**
```yaml
---
- name: Install Prometheus
  apt:
    name: prometheus
    state: present

- name: Start and enable Prometheus
  systemd:
    name: prometheus
    state: started
    enabled: yes

- name: Add Grafana APT key
  apt_key:
    url: https://packages.grafana.com/gpg.key
    state: present

- name: Add Grafana APT repository
  apt_repository:
    repo: 'deb https://packages.grafana.com/oss/deb stable main'
    state: present

- name: Update APT package cache
  apt:
    update_cache: yes

- name: Install Grafana
  apt:
    name: grafana
    state: present

- name: Start and enable Grafana
  systemd:
    name: grafana-server
    state: started
    enabled: yes
```

## Playbook for the Ansible task
The site.yml playbook is the central file for orchestrating the configuration of your servers. It leverages the roles defined in the roles directory to apply configurations to different types of servers.

```yaml
---
- name: Configure web servers
  hosts: web
  become: yes
  roles:
    - web

- name: Configure database servers
  hosts: db
  become: yes
  roles:
    - db

- name: Configure monitoring servers
  hosts: monitoring
  become: yes
  roles:
    - monitoring
```
## Running the Playbook
To execute the playbook and apply configurations to your servers, use the following command:
```sh
ansible-playbook -i inventory.ini site.yml --ask-pass --ask-become-pass
```

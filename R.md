# Ansible Playbooks for Nginx, Application, and Database Provisioning

This document contains Ansible playbooks for setting up a full-stack web application, including Nginx, Node.js, and MongoDB. The playbooks are modular and follow best practices for easier maintenance and scalability.

---

## **1. Ansible Playbook: Install and Configure Nginx**

### **Filename**: `install_nginx.yml`

#### **Playbook:**
```yaml
---
- name: Install and configure Nginx
  hosts: web
  gather_facts: yes
  become: yes

  tasks:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: yes  # Refreshes the package list to get the latest updates

    - name: Upgrade all packages
      ansible.builtin.apt:
        upgrade: dist  # Upgrades all installed packages to their latest versions

    - name: Install Nginx
      ansible.builtin.apt:
        name: nginx
        state: present  # Ensures that Nginx is installed

    - name: Ensure Nginx is running and enabled on boot
      ansible.builtin.systemd:
        name: nginx
        state: started  # Starts Nginx service
        enabled: yes  # Enables Nginx to start on boot

    - name: Remove default Nginx site
      ansible.builtin.file:
        path: /etc/nginx/sites-enabled/default
        state: absent  # Removes the default site to prevent conflicts
      notify: Restart Nginx

    - name: Create Nginx reverse proxy configuration
      ansible.builtin.copy:
        dest: /etc/nginx/sites-available/app
        content: |
          server {
              listen 80;
              server_name _;

              location / {
                  proxy_pass http://localhost:3000/;  # Redirects traffic to the application running on port 3000
              }
          }
      notify: Restart Nginx

    - name: Enable the new Nginx configuration
      ansible.builtin.file:
        src: /etc/nginx/sites-available/app
        dest: /etc/nginx/sites-enabled/app
        state: link  # Creates a symbolic link to enable the site
      notify: Restart Nginx

  handlers:
    - name: Restart Nginx
      ansible.builtin.service:
        name: nginx
        state: restarted  # Restarts Nginx to apply changes
```

### **How to Run the Playbook**
```bash
ansible-playbook install_nginx.yml
```

### **Verification**
```bash
ansible web -m shell -a "systemctl status nginx"
```

![Nginx Running](<images/active_nginx.png>)

---

# Ansible Playbook: Provision App VM and Run Application

This playbook installs **Node.js**, clones an application from GitHub, installs dependencies, seeds the database, and starts the application on target nodes belonging to the `web` group. It also ensures that traffic on **port 3000** is allowed.

### **Filename**: `prov_app_with_npm_start.yml`

## **Playbook Structure**
```yaml
---
- name: Install app dependencies and run app
  hosts: web
  gather_facts: yes
  become: yes  # Grants admin access (equivalent to sudo)

  tasks:
    - name: Install Node.js 20 and npm
      ansible.builtin.shell: |
        curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
        apt-get install -y nodejs  # Installs Node.js 20
      args:
        executable: /bin/bash

    - name: Install npm
      ansible.builtin.apt:
        name: npm
        state: present  # Ensures npm is installed

    - name: Clone the app from GitHub
      ansible.builtin.git:
        repo: "https://github.com/marmari9/tech501-sparta-app.git"
        dest: /home/ubuntu/tech501-sparta-app
        version: main  # Clones the main branch
        force: yes  # Overwrites any existing content

    - name: Install app dependencies without running seed script
      ansible.builtin.command:
        cmd: npm install --ignore-scripts  # Installs dependencies without executing scripts
        chdir: /home/ubuntu/tech501-sparta-app

    - name: Manually seed the database
      ansible.builtin.command:
        cmd: node seeds/seed.js  # Seeds the database
        chdir: /home/ubuntu/tech501-sparta-app

    - name: Start the app with npm
      ansible.builtin.command:
        cmd: npm start  # Starts the application
        chdir: /home/ubuntu/tech501-sparta-app

    - name: Allow traffic on port 3000
      ansible.builtin.iptables:
        chain: INPUT
        protocol: tcp
        destination_port: 3000
        jump: ACCEPT  # Ensures traffic can reach the application
```

## **Explanation of Commands**

### **1️. Play Definition**
```yaml
- name: Install app dependencies and run app
  hosts: web
  gather_facts: yes
  become: yes
```
- **`name`** → Descriptive name for the play.
- **`hosts: web`** → Runs the play on all machines in the `web` inventory group.
- **`gather_facts: yes`** → Collects system information (useful for conditionals in playbooks).
- **`become: yes`** → Grants admin privileges (`sudo`) for all tasks.

### **2️. Installing Node.js and npm**
```yaml
    - name: Install Node.js 20 and npm
      ansible.builtin.shell: |
        curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
        apt-get install -y nodejs
      args:
        executable: /bin/bash
```
- **Downloads Node.js 20 setup script** and installs Node.js.
- **Uses `sudo -E`** to retain the environment variables.

### **3️. Installing npm**
```yaml
    - name: Install npm
      ansible.builtin.apt:
        name: npm
        state: present
```
- **Ensures npm is installed** alongside Node.js.

### **4️. Cloning the Application Repository**
```yaml
    - name: Clone the app from GitHub
      ansible.builtin.git:
        repo: "https://github.com/marmari9/tech501-sparta-app.git"
        dest: /home/ubuntu/tech501-sparta-app
        version: main
        force: yes
```
- **`repo`** → Specifies the GitHub repository to clone.
- **`dest`** → Defines where the repository should be cloned.
- **`force: yes`** → Overwrites existing files if the repo already exists.

### **5️. Installing Dependencies Without Running Scripts**
```yaml
    - name: Install app dependencies without running seed script
      ansible.builtin.command:
        cmd: npm install --ignore-scripts
        chdir: /home/ubuntu/tech501-sparta-app
```
- **Runs `npm install`** but skips executing any package scripts.

### **6️. Seeding the Database**
```yaml
    - name: Manually seed the database
      ansible.builtin.command:
        cmd: node seeds/seed.js
        chdir: /home/ubuntu/tech501-sparta-app
```
- **Runs the seed script manually** to populate the database.

### **7️. Starting the Application**
```yaml
    - name: Start the app with npm
      ansible.builtin.command:
        cmd: npm start
        chdir: /home/ubuntu/tech501-sparta-app
```
- **Executes `npm start`** to launch the application.

### **8️. Allowing Traffic on Port 3000**
```yaml
    - name: Allow traffic on port 3000
      ansible.builtin.iptables:
        chain: INPUT
        protocol: tcp
        destination_port: 3000
        jump: ACCEPT
```
- **Ensures incoming traffic on port 3000 is accepted** so the app can be accessed.

## **How to Run the Playbook**
Execute the playbook from the **Ansible controller**:
```bash
ansible-playbook prov_app_with_npm_start.yml
```

## **Verification**
After running the playbook, confirm that the app is running:
```bash
ansible web -m shell -a "curl http://localhost:3000"
```
If successful, the application should be accessible on **port 3000**.

![App Running on Port 3000](<images/app_page_working_on_3000.png>)

---

# Ansible Playbook: Install and Configure MongoDB

This playbook installs and configures **MongoDB** on target nodes belonging to the `db` group. It ensures that the system is updated before installing MongoDB, modifies the configuration to allow remote access, and starts the database service.

### **Filename**: `install_mongodb.yml`

## **Playbook Structure**
```yaml
---
- name: Install and configure MongoDB
  hosts: db
  gather_facts: yes
  become: yes  # Grants admin access (equivalent to sudo)

  tasks:
    - name: Import MongoDB public key
      ansible.builtin.apt_key:
        url: https://www.mongodb.org/static/pgp/server-7.0.asc
        state: present  # Ensures the MongoDB key is added for secure installation

    - name: Add MongoDB repository
      ansible.builtin.apt_repository:
        repo: "deb [ arch=amd64 ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse"
        state: present  # Adds the MongoDB repo to the system
        filename: mongodb-org

    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: yes  # Refreshes package sources

    - name: Install MongoDB
      ansible.builtin.apt:
        name: mongodb-org  # Installs the MongoDB package
        state: present  # Ensures MongoDB is installed

    - name: Allow connections from any IP
      ansible.builtin.replace:
        path: /etc/mongod.conf
        regexp: 'bindIp: 127.0.0.1'
        replace: 'bindIp: 0.0.0.0'  # Enables remote access to MongoDB

    - name: Restart and enable MongoDB
      ansible.builtin.service:
        name: mongod  # Manages the MongoDB service
        state: restarted  # Ensures MongoDB is running
        enabled: yes  # Enables MongoDB to start on boot
```

## **Explanation of Commands**

### **1️. Play Definition**
```yaml
- name: Install and configure MongoDB
  hosts: db
  gather_facts: yes
  become: yes
```
- **`name`** → Descriptive name for the play.
- **`hosts: db`** → Runs the play on all machines in the `db` inventory group.
- **`gather_facts: yes`** → Collects system information (useful for conditionals in playbooks).
- **`become: yes`** → Grants admin privileges (`sudo`) for all tasks.

### **2️. Importing MongoDB Public Key**
```yaml
    - name: Import MongoDB public key
      ansible.builtin.apt_key:
        url: https://www.mongodb.org/static/pgp/server-7.0.asc
        state: present
```
- **`apt_key`** → Adds MongoDB's public key to verify the package's authenticity.
- **`state: present`** → Ensures the key is added before package installation.

### **3️. Adding the MongoDB Repository**
```yaml
    - name: Add MongoDB repository
      ansible.builtin.apt_repository:
        repo: "deb [ arch=amd64 ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse"
        state: present
        filename: mongodb-org
```
- **`apt_repository`** → Adds MongoDB's official repository to package sources.
- **`state: present`** → Ensures the repository is active for installation.

### **4️. Installing MongoDB**
```yaml
    - name: Install MongoDB
      ansible.builtin.apt:
        name: mongodb-org
        state: present
```
- **`name: mongodb-org`** → Installs MongoDB from the added repository.
- **`state: present`** → Ensures MongoDB is installed and not removed.

### **5️. Enabling Remote Access**
```yaml
    - name: Allow connections from any IP
      ansible.builtin.replace:
        path: /etc/mongod.conf
        regexp: 'bindIp: 127.0.0.1'
        replace: 'bindIp: 0.0.0.0'
```
- **`replace`** → Modifies MongoDB’s configuration to allow external connections.
- **`bindIp: 0.0.0.0`** → Allows MongoDB to accept connections from any IP address.

### **6️. Restarting and Enabling MongoDB**
```yaml
    - name: Restart and enable MongoDB
      ansible.builtin.service:
        name: mongod
        state: restarted
        enabled: yes
```
- **`service`** → Ensures MongoDB is running and starts on boot.
- **`state: restarted`** → Restarts MongoDB to apply configuration changes.
- **`enabled: yes`** → Enables MongoDB to start automatically on reboot.

## **How to Run the Playbook**
Execute the playbook from the **Ansible controller**:
```bash
ansible-playbook install_mongodb.yml
```

## **Verification**
After running the playbook, confirm that MongoDB is installed and running:
```bash
ansible db -m shell -a "systemctl status mongod"
```
If MongoDB is running, you should see output indicating that the service is **active**.

![MongoDB Running](<images/verify_mongodb_install.png>)



---

## **Ansible Playbook: Importing Multiple Playbooks**
This playbook imports separate playbooks for **database** and **application** provisioning, ensuring modular deployment.

### **Filename**: `deploy_app_and_db.yml`

### **Playbook Structure**
```yaml
---
- import_playbook: prov-db.yml  # Imports the database provisioning playbook
- import_playbook: prov_app_with_pm2.yml  # Imports the app provisioning playbook
```

### **Why Use Importing?**
- **Keeps roles separate** → The **database** and **application** provisioning are handled independently.
- **Enhances reusability** → You can run individual playbooks separately when required.
- **Improves debugging** → Errors in one playbook won’t necessarily affect others.

---

## **Application Web Page and Posts Page Running on Port 3000**
Once provisioning is complete, the application should be accessible via **port 3000**.

### **Verification**
![Application Running](<images/app_page_working_on_3000.png>)

![Posts Page Running](<images/posts_page_working_with_ansible.png>)

---

## **Ansible Playbook: Setting Up a Reverse Proxy with Nginx**
To configure **Nginx** as a reverse proxy for the application, modify the **install_nginx.yml** playbook.

### **Reverse Proxy Configuration in `install_nginx.yml`**
```yaml
    - name: Create Nginx reverse proxy configuration
      ansible.builtin.copy:
        dest: /etc/nginx/sites-available/app
        content: |
          server {
              listen 80;
              server_name _;

              location / {
                  proxy_pass http://localhost:3000/;  # Directs traffic to app running on port 3000
              }
          }
      notify: Restart Nginx

    - name: Enable the new Nginx configuration
      ansible.builtin.file:
        src: /etc/nginx/sites-available/app
        dest: /etc/nginx/sites-enabled/app
        state: link  # Creates a symbolic link to activate the new config
      notify: Restart Nginx
```

### **Verification**
After running the playbook, verify the proxy setup:
![Reverse Proxy Working](<images/working_with_reverse_proxy.png>)

![Posts Page Through Proxy](<images/posts_working_with_reverse_proxy.png>)

---

## **Using Parent-Child Hosts in Ansible Inventory**
To organize infrastructure, define a **parent host group** with child nodes for the **app** and **database** servers.

### **Example Inventory File (`hosts.ini`)**
```ini
[test:children]
app
db
```

### **Verifying Parent Group with Ansible Ping**
```bash
ansible test -m ping
```

### **Verification Output**
![Parent Host Ping](<images/test_parent_ping.png>)

---

This structured approach to **updates, modular provisioning, reverse proxy setup, and inventory management** ensures a scalable and maintainable Ansible deployment.
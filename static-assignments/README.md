# 🧩 Ansible Refactoring & Static Assignments (Imports and Roles)

This project demonstrates refactoring Ansible playbooks for better structure, scalability, and reusability using imports and roles. It also includes Jenkins automation enhancements for artifact management.

---
<details>
<summary>📘 Table of Contents (click to expand)</summary>

- [🧩 Ansible Refactoring \& Static Assignments (Imports and Roles)](#-ansible-refactoring--static-assignments-imports-and-roles)
- [🚀 Project Overview](#-project-overview)
- [🚀 Project Architecture](#-project-architecture)
  - [](#)
  - [🧱 Step 1 – Jenkins Job Enhancement](#-step-1--jenkins-job-enhancement)
    - [🛠️ Setup](#️-setup)
  - [⚙️ Step 2 – Refactor Ansible Code Using Imports](#️-step-2--refactor-ansible-code-using-imports)
    - [🎯 Objective](#-objective)
    - [🧩 Steps](#-steps)
  - [🧹 Step 3 – Delete Wireshark from Dev Servers](#-step-3--delete-wireshark-from-dev-servers)
  - [🏗️ Step 4 – Configure UAT Webservers Using a Role](#️-step-4--configure-uat-webservers-using-a-role)
    - [🔧 Steps](#-steps-1)
  - [📘 Step 5 – Reference the Role](#-step-5--reference-the-role)
  - [🧪 Step 6 – Test and Verify](#-step-6--test-and-verify)
  - [📂 Recommended Repository Structure](#-recommended-repository-structure)
  - [🧭 Key Takeaways](#-key-takeaways)

</details>

---

# 🚀 Project Overview

As infrastructure grows, a single monolithic playbook becomes hard to manage. Refactoring allows us to:

- Organize playbooks into logical units
- Reuse existing configuration easily
- Maintain clear structure for multiple environments (Dev, UAT, Prod)
- Improve team collaboration and code readability

This phase focuses on:

1. Enhancing the Jenkins automation job
2. Refactoring existing Ansible playbooks
3. Introducing the import_playbook concept
4. Implementing roles for modular automation
5. Testing a new “Webserver” role for UAT deployment

---

# 🚀 Project Architecture

![alt image](/static-assignments/images/39.png)
---

## 🧱 Step 1 – Jenkins Job Enhancement

To manage build artifacts more efficiently, we’ll create a new Jenkins job and directory to handle them.

### 🛠️ Setup

**1. Create artifact storage:**
```bash
sudo mkdir /home/ubuntu/ansible-config-artifact
sudo chmod -R 0777 /home/ubuntu/ansible-config-artifact
```

![alt image](/static-assignments/images/1.png)
![alt image](/static-assignments/images/2.png)

**2. In Jenkins:**

- Go to Manage Jenkins → Manage Plugins
- Search for Copy Artifact Plugin and install it (no restart required).

![alt image](/static-assignments/images/3.png)
![alt image](/static-assignments/images/4.png)
![alt image](/static-assignments/images/5.png)
![alt image](/static-assignments/images/6.png)

**3. Create a new Freestyle Project called save_artifacts.**

- Configure it to trigger after the ansible project completes.
- Add a Build Step → Copy artifacts from another project
  - Source project: ansible
  - Target directory: `/home/ubuntu/ansible-config-artifact`

![alt image](/static-assignments/images/7.png)
![alt image](/static-assignments/images/8.png)
![alt image](/static-assignments/images/9.png)
![alt image](/static-assignments/images/10.png)
![alt image](/static-assignments/images/11.png)

**4. Test your setup by committing a small change (e.g., in README.md) to your repo.**

✅ You should see new build artifacts copied into /home/ubuntu/ansible-config-artifact.

![alt image](/static-assignments/images/12.png)
![alt image](/static-assignments/images/13.png)
![alt image](/static-assignments/images/14.png)
![alt image](/static-assignments/images/15.png)

---

## ⚙️ Step 2 – Refactor Ansible Code Using Imports
### 🎯 Objective

Simplify your playbooks and make them modular by splitting them into smaller logical files and importing them as needed.

### 🧩 Steps

**1. Pull latest code from the main branch and create a new branch:**
```bash
git checkout -b refactor
```
![alt image](/static-assignments/images/16b.png)

**2. Create a new entry-point playbook:**
```bash
cd playbooks
touch site.yml
```
![alt image](/static-assignments/images/17.png)

**3. Create a folder to hold static assignments:**
```bash
mkdir ../static-assignments
```
![alt image](/static-assignments/images/18.png)

**4. Move `common.yml` into `static-assignments/`.**
![alt image](/static-assignments/images/19.png)

**5. Update `site.yml` to import the playbook:**
```yml
---
- import_playbook: ../static-assignments/common.yml
```

---

## 🧹 Step 3 – Delete Wireshark from Dev Servers

We’ll add a new playbook to remove wireshark from dev environments.

**Create `static-assignments/common-del.yml`:**

```yml
---
- name: update web and nfs servers
  hosts: webservers, nfs
  remote_user: ec2-user
  become: yes
  tasks:
    - name: delete wireshark
      yum:
        name: wireshark
        state: removed

- name: update LB and db servers
  hosts: lb, db
  remote_user: ubuntu
  become: yes
  tasks:
    - name: delete wireshark
      apt:
        name: wireshark-qt
        state: absent
        autoremove: yes
        purge: yes
        autoclean: yes
```
![alt image](/static-assignments/images/20.png)

**Update your `site.yml`:**
```yml
---
- import_playbook: ../static-assignments/common-del.yml
```
![alt image](/static-assignments/images/21.png)

**Run it against the dev inventory:**
```bash
ansible-playbook -i inventory/dev.ini playbooks/site.yml
```
![alt image](/static-assignments/images/22.png)
![alt image](/static-assignments/images/23.png)
![alt image](/static-assignments/images/24.png)
![alt image](/static-assignments/images/25.png)


**Verify deletion:**
```bash
wireshark --version
```
![alt image](/static-assignments/images/26.png)
![alt image](/static-assignments/images/27.png)

✅ wireshark should be removed from all dev servers.

---

## 🏗️ Step 4 – Configure UAT Webservers Using a Role

Roles make your configurations reusable and modular.

### 🔧 Steps

**1. Launch two new EC2 instances (RHEL 8) and name them:**

- Web1-UAT
- Web2-UAT

![alt image](/static-assignments/images/28.png)

**2. Create a `roles/` directory and initialize the webserver role:**
```bash
mkdir roles
cd roles

ansible-galaxy init webserver
#  or create manually 
mkdir -p webserver/{defaults,handlers,meta,tasks,templates}
```
![alt image](/static-assignments/images/29.png)
![alt image](/static-assignments/images/30.png)
![alt image](/static-assignments/images/31.png)

**3. Update your inventory `inventory/uat.ini` with your UAT servers’ public IPs.**
![alt image](/static-assignments/images/32.png)

**4. In /etc/ansible/ansible.cfg, uncomment and update:**
```arduino
roles_path = /home/ubuntu/ansible-config-mgt/roles
```
![alt image](/static-assignments/images/33a.png)
![alt image](/static-assignments/images/33.png)

**5. Edit `roles/webserver/tasks/main.yml`:**
```yml
---
- name: install apache
  become: true
  yum:
    name: httpd
    state: present

- name: install git
  become: true
  yum:
    name: git
    state: present

- name: clone tooling website
  become: true
  git:
    repo: https://github.com/princemaxi/tooling.git
    dest: /var/www/html
    force: yes

- name: copy html content
  become: true
  command: cp -r /var/www/html/html/ /var/www

- name: start apache
  become: true
  service:
    name: httpd
    state: started

- name: remove extra html directory
  become: true
  file:
    path: /var/www/html/html/
    state: absent
```
![alt image](/static-assignments/images/34.png)

---

## 📘 Step 5 – Reference the Role

**Create `static-assignments/uat-webservers.yml`:**
```yml
---
- hosts: uat-webservers
  roles:
    - webserver
```
![alt image](/static-assignments/images/35a.png)
![alt image](/static-assignments/images/35b.png)

**Then update `site.yml`:**
```ynl
---
- import_playbook: ../static-assignments/common.yml
- import_playbook: ../static-assignments/uat-webservers.yml
```
![alt image](/static-assignments/images/36.png)

---

## 🧪 Step 6 – Test and Verify

**1. Commit and push your changes:**
```bash
git add .
git commit -m "Refactored Ansible code and added webserver role"
git push origin refactor
```

**2. Create a pull request and merge into `main`.**

**3. Confirm Jenkins triggers both jobs and copies files into:**
```arduino
/home/ubuntu/ansible-config-mgt/
```

**4. Run the playbook against UAT servers:**
```bash
ansible-playbook -i inventory/uat.ini playbooks/site.yml
```
![alt image](/static-assignments/images/37.png)

**5. Verify access:**
```pgsql
http://<Web1-UAT-Public-IP>/index.php
http://<Web2-UAT-Public-IP>/index.php
```
![alt image](/static-assignments/images/38.png)

You should see the Tooling Website homepage 🎉

---

## 📂 Recommended Repository Structure
```arduino
ansible-config-mgt/
│
├── playbooks/
│   ├── site.yml
│
├── static-assignments/
│   ├── common.yml
│   ├── common-del.yml
│   └── uat-webservers.yml
│
├── inventory/
│   ├── dev.yml
│   └── uat.yml
│
├── roles/
│   └── webserver/
│       ├── tasks/main.yml
│       ├── handlers/
│       └── templates/
│
├── images
└── README.md
```

---

## 🧭 Key Takeaways

✅ Modular and reusable playbooks with import_playbook

✅ Cleaner Jenkins pipeline with controlled artifact storage

✅ Introduction of roles for scalable configuration

✅ Separation of environments (Dev, UAT, Prod)

✅ Easier maintenance and collaboration
# Ansible Configuration Management (Automating Projects 7–10)

<details>
<summary>Table of Contents</summary>

- [Ansible Configuration Management (Automating Projects 7–10)](#ansible-configuration-management-automating-projects-710)
  - [Ansible Client as a Jump Server (Bastion Host)](#ansible-client-as-a-jump-server-bastion-host)
  - [Tasks Breakdown](#tasks-breakdown)
    - [1. Install and Configure Ansible Client](#1-install-and-configure-ansible-client)
    - [2. Create a Simple Ansible Playbook](#2-create-a-simple-ansible-playbook)
- [Architectural Diagram](#architectural-diagram)
- [Automation Steps](#automation-steps)
  - [Why this pattern?](#why-this-pattern)
  - [Step 1: Install \& Configure Ansible on an EC2 Jenkins (Jenkins-Ansible) Jump Server](#step-1-install--configure-ansible-on-an-ec2-jenkins-jenkins-ansible-jump-server)
    - [1 — Update the EC2 Name tag (optional but helpful)](#1--update-the-ec2-name-tag-optional-but-helpful)
    - [2 — Create GitHub repo ansible-config-mgt](#2--create-github-repo-ansible-config-mgt)
    - [3 — Install Ansible on Jenkins-Ansible (control node)](#3--install-ansible-on-jenkins-ansible-control-node)
    - [4 — Configure Jenkins build job to archive ansible-config-mgt repo](#4--configure-jenkins-build-job-to-archive-ansible-config-mgt-repo)
  - [Step 2 — Prepare Your Development Environment Using Visual Studio Code](#step-2--prepare-your-development-environment-using-visual-studio-code)
    - [Why This Step Matters](#why-this-step-matters)
    - [Install Visual Studio Code](#install-visual-studio-code)
    - [💡 Tip: Install recommended extensions:](#-tip-install-recommended-extensions)
    - [Configure GitHub Integration in VS Code](#configure-github-integration-in-vs-code)
    - [Clone Repo on Jenkins-Ansible Instance](#clone-repo-on-jenkins-ansible-instance)
  - [Step 3 – Begin Ansible Development](#step-3--begin-ansible-development)
  - [Step 4 – Setting Up an Ansible Inventory](#step-4--setting-up-an-ansible-inventory)
  - [Step 5 – Create a Common Playbook](#step-5--create-a-common-playbook)
  - [Step 6 – Update Git with the Latest Code](#step-6--update-git-with-the-latest-code)
  - [Step 7 – Run the First Ansible Test](#step-7--run-the-first-ansible-test)
- [🔧 Troubleshooting Guide – Ansible Configuration Management Project](#-troubleshooting-guide--ansible-configuration-management-project)
    - [✅ Summary](#-summary)

</details>

In Projects 7 to 10, we manually performed several repetitive DevOps tasks such as provisioning virtual servers, installing and configuring software, and deploying web applications. While these steps built our foundational knowledge, they also highlighted how time-consuming and error-prone manual operations can be.

This is where Ansible Configuration Management comes in. Ansible allows us to automate these tasks using simple declarative code written in YAML, ensuring consistency, repeatability, and efficiency across environments. With Ansible, you define what the system should look like, and Ansible takes care of how to achieve it.

By automating these projects with Ansible, we not only eliminate manual overhead but also embrace the real power of DevOps: automation, reproducibility, and scalability.

## Ansible Client as a Jump Server (Bastion Host)

Before diving into playbooks, let’s discuss the architecture.

A Jump Server (also called a Bastion Host) is a secure, intermediary server that acts as a bridge between external users and the internal infrastructure. In a production-grade setup:

- Web servers and databases are placed in private subnets for security.
- These internal servers cannot be accessed directly from the internet.
- Instead, engineers connect to a Jump Server that is allowed inbound access (e.g., via SSH).
- From the Jump Server, they can securely connect to internal servers.

This architecture significantly reduces the attack surface because external access is funneled through a single controlled point, improving monitoring and security compliance.

In our setup, we configure the Ansible Client to act as the Jump Server(Bastion). That means:

- We’ll install Ansible on the Jump Server.
- The Jump Server will have SSH access to other servers (web, database, etc.).
- From this central point, Ansible can push configurations and run playbooks across all target servers.

## Tasks Breakdown
### 1. Install and Configure Ansible Client

The first step is to set up Ansible on the Jump Server. Once installed, the server will serve as the control node from which all automation is executed.

Key points:

- Ansible requires only Python and SSH access to manage remote hosts.
- No agent needs to be installed on the target servers (unlike Puppet or Chef).
- Configuration is defined in an inventory file listing the managed hosts.

### 2. Create a Simple Ansible Playbook

An Ansible playbook is a YAML file that describes the desired configuration of target servers. Playbooks define tasks such as:

- Installing packages (e.g., Nginx, Apache, MySQL).
- Copying configuration files.
- Starting and enabling services.

Instead of manually logging into each server and running commands, a playbook allows you to declare the final state you want, and Ansible enforces it automatically.

Example (simplified):
```yaml
- name: Configure Web Server
  hosts: webservers
  become: yes
  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present
    - name: Start Nginx service
      service:
        name: nginx
        state: started
        enabled: yes
```

This playbook ensures all servers in the webservers group have Nginx installed, running, and enabled on startup — without logging into each one.

---

# Architectural Diagram
![alt image](/images/45.png)

---

# Automation Steps

This section turns the short checklist into a reproducible, secure workflow: 1) prepare the Jenkins instance as an Ansible control/jump host, 2) wire it to GitHub, and 3) configure Jenkins to archive your Ansible repo on every change to main. I include explicit commands, configuration notes, and troubleshooting tips.

## Why this pattern?

We make Jenkins also our Ansible control node / jump server so:

- Jenkins can react automatically to Git commits (CI trigger → artifact snapshot).
- Ansible runs from a single hardened point that has SSH access to private hosts (bastion pattern).
- You keep infrastructure-as-code (Git) and automated execution (Jenkins + Ansible) closely integrated.

## Step 1: Install & Configure Ansible on an EC2 Jenkins (Jenkins-Ansible) Jump Server

### 1 — Update the EC2 Name tag (optional but helpful)

Why: Tags make instances easy to identify in the console or automation.

> AWS Console: select instance → Tags → edit Name → Jenkins-Ansible

AWS CLI:
```bash
# replace i-0123456789abcdef0 with your instance id
aws ec2 create-tags \
  --resources i-0123456789abcdef0 \
  --tags Key=Name,Value=Jenkins-Ansible \
  --region eu-west-2
```

![alt image](/images/1.png)
![alt image](/images/2.png)
![alt image](/images/3.png)

### 2 — Create GitHub repo ansible-config-mgt

> Quick UI: GitHub → New → Repository name ansible-config-mgt → Private/Public as you prefer → Create.

CLI (gh):
```bash
gh repo create <your-github-username>/ansible-config-mgt --public --confirm
```

![alt image](/images/4.png)
![alt image](/images/5.png)

Then clone the repo on the Jenkins-Ansible instance:
```
git clone git@github.com:princemaxi/ansible-config-mgt.git
cd ansible-config-mgt
```

### 3 — Install Ansible on Jenkins-Ansible (control node)

**Option A — Recommended:** isolated Python venv (reproducible)

This avoids system package conflicts and is future-proof.

```bash
# ensure essentials
sudo apt update
sudo apt install -y python3-pip python3-venv git

# create and activate venv
python3 -m venv ~/ansible-venv
source ~/ansible-venv/bin/activate

# upgrade pip inside venv and install ansible-core (pick stable core version)
pip install --upgrade pip
pip install "ansible-core==2.13.13"    # or 2.15.x / 2.16.x depending on compatibility
# optionally install community bundle if you need collections shipped with the meta package:
# pip install "ansible==6.7.0"
```
Put the venv activation into the Jenkins job build step (or wrap playbook runs in a script that activates the venv).

**Option B — Quick & simple (system apt)**

Not recommended for modern Ubuntu versions due to older package versions:
```bash
sudo apt update
sudo apt install -y ansible git
ansible --version
```

![alt image](/images/6.png)
![alt image](/images/7.png)
![alt image](/images/8.png)

> Note: apt Ansible on older distros can be outdated; use the venv approach for predictable behavior (and to avoid the PEP-668/system pip issues on newer Ubuntu releases)

### 4 — Configure Jenkins build job to archive ansible-config-mgt repo

- **Why this step matters**

    In a DevOps pipeline:

  - Traceability: we want a historical record of every configuration change (build artifacts are immutable).

  - Repeatability: archived repo snapshots can be replayed later (audit, rollback, debugging).

  - Automation: webhooks eliminate manual builds — every push to main triggers Jenkins automatically.

- **Create a Jenkins Freestyle project**

  - Log into your Jenkins dashboard → New Item.

  - Enter project name: `ansible` → choose Freestyle project → click OK.

  - Inside project config:

    - Description: CI job to archive ansible-config-mgt repo

    - Check Discard old builds (optional) to keep artifacts manageable.y
    - 
    ![alt image](/images/10.png)

- **Connect GitHub repo created earlier (ansible-config-mgt)**
    **SCM setup**

    - Under Source Code Management, choose Git.

    - Repository URL: 
        ```
        https://github.com/<your-username>/ansible-config-mgt
        ```
    - Credentials:

        HTTPS: add a GitHub personal access token in Jenkins Credentials store.
    - Branch specifier:
        ```
        */main
        ```
    ![alt image](/images/11.png)
    ![alt image](/images/12.png)

- **Configure GitHub Webhook**

    **On the GitHub repo:**

  1. Go to Settings → Webhooks → Add Webhook.

  2. Payload URL:
        ```
        http://<jenkins-ansible-server-public-IP>:8080/github-webhook/
        ```
        (replace with your Elastic IP or DNS of Jenkins-Ansible server)

  3. Content type: application/json

  4. Choose Just the push event.

  5. Save.

    ![alt image](/images/13.png)
    ![alt image](/images/14.png)

    **On Jenkins:**

  - In project config → Build Triggers → select:

      **✅ GitHub hook trigger for GITScm polling**

      This ensures Jenkins will build whenever there’s a push to main.

- **Configure Post-Build Archiving**

    Now, we want Jenkins to save the repo snapshot after every build:

    1. Scroll to Post-build Actions → select Archive the artifacts.

    2. Files to archive:
        ```
        **/*
        ```
    ![alt image](/images/15.png)

        - `**/*` means all files recursively in the workspace.

        - You can narrow it down later if you want to archive only playbooks or inventory files.

    Artifacts will be stored here per build:
    ```swift
    /var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/
    ```

- **Test the Setup**

1. Make a change in your repo:
    ```bash
    echo "Test CI $(date)" >> README.md
    git add README.md
    git commit -m "ci: test webhook build"
    git push origin main
    ```
    ![alt image](/images/16.png)    
2. GitHub webhook will fire → Jenkins job runs.

3. Check build log in Jenkins dashboard.
    ![alt image](/images/17.png)

4. Verify artifacts:

    ```bash
    ls /var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/
    ```

    You should see the full repo snapshot.
![alt image](/images/18.png)

✅ At this point: every Git push to `main` in `ansible-config-mgt` → automatically triggers Jenkins → repo snapshot is archived as a build artifact.

---

## Step 2 — Prepare Your Development Environment Using Visual Studio Code
### Why This Step Matters

In a DevOps workflow, your IDE (Integrated Development Environment) is more than just a code editor — it’s your productivity hub. With VS Code:

- You can write and debug playbooks with YAML linting and IntelliSense.

- Source control integration ensures changes sync seamlessly with GitHub.

- Extensions allow for remote development directly on your Jenkins-Ansible server or any EC2 instance.

By preparing VS Code properly, you create a streamlined feedback loop: write → commit → push → CI job runs automatically in Jenkins.

### Install Visual Studio Code

On your local machine (Windows/Mac/Linux):

- Download VS Code from the official website: https://code.visualstudio.com

- Install with default options.

### 💡 Tip: Install recommended extensions:

- YAML (for syntax highlighting & linting of Ansible playbooks).

- Ansible (by Red Hat — provides snippets, docs, and linting).

- GitHub Pull Requests and Issues (for seamless collaboration).

### Configure GitHub Integration in VS Code

To collaborate effectively, connect VS Code to your GitHub repo.

- Authenticate GitHub in VS Code

  - Open VS Code → View → Command Palette → search for GitHub: Sign in.

  - Sign in with your GitHub account (Personal Access Token or browser-based OAuth).

- Clone Repo Directly in VS Code (Optional)

  - Open Source Control tab → “Clone Repository” → paste repo URL.

  - Choose a local folder, VS Code auto-opens the repo.

This ensures your local IDE is always in sync with GitHub.

### Clone Repo on Jenkins-Ansible Instance

Since Jenkins will actually run Ansible playbooks, we also need the repo available on the EC2 server. SSH into your Jenkins-Ansible instance and run:
```bash
git clone <ansible-config-mgt repo link>
```

Expected output:
```bash
Cloning into 'ansible-config-mgt'...
remote: Enumerating objects: 5, done.
remote: Counting objects: 100% (5/5), done.
Unpacking objects: 100% (5/5), done.
```

Now verify:
```bash
cd ansible-config-mgt
ls
```
![alt image](/images/20.png)

You should see your project files (e.g., README.md).

✅ At this stage:

- You have VS Code installed locally with GitHub integration.
- Repo is cloned both on your local machine (for coding) and Jenkins-Ansible EC2 (for execution).

## Step 3 – Begin Ansible Development

After setting up Ansible and integrating Jenkins with GitHub, the next step is to structure your Ansible project for scalability, collaboration, and maintainability. In DevOps, establishing a clean directory structure early ensures that teams can extend and automate infrastructure management without confusion.

In this step, we will:

1. Create a new branch for feature development.

2. Organize directories for playbooks and inventory.

3. Write the first Ansible playbook (common.yaml).

4. Set up environment-specific inventory files.

**1. Create a New Branch for Feature Development**

Working on the main branch is risky, as it may cause broken automation pipelines if unstable code is merged. Instead, we follow Git best practices: create feature branches for each task or module.

On your local machine or Jenkins-Ansible EC2 instance, run:
```bash
# Navigate to your project directory
cd ansible-config-mgt

# Create and switch to a new branch
git checkout -b dev-setup
```
![alt image](/images/21.png)

> - git checkout -b dev-setup creates a new branch and immediately checks it out.
> - This ensures isolation of work until it’s tested and ready for merge into main.

**2. Organize Your Project Structure**

A well-structured project keeps playbooks modular and hosts organized for multiple environments.

Run the following to create directories:
```bash
# Create directories
mkdir playbooks inventory
```

> - playbooks → stores YAML playbooks (e.g., configuration automation, package installation).
> - inventory → defines groups of hosts across environments (dev, staging, uat, prod).

Your structure now looks like this:
```bash
ansible-config-mgt/
├── inventory/
├── playbooks/
├── README.md
```

**3. Create Your First Playbook: common.yaml**

Playbooks are the heart of Ansible. Each playbook describes tasks that run against specified hosts. Let’s create a common playbook that will define baseline configurations (packages, users, etc.) across all environments.

Inside the playbooks directory, create the file:
```bash
nano playbooks/common.yaml
```
![alt image](/images/22.png)

**4. Create Environment-Specific Inventory Files**

Ansible inventories allow you to define host groups (dev, staging, uat, prod). Using .ini style makes grouping intuitive.

Inside the inventory directory, create four files:
```bash
cd inventory
nano dev.ini
nano staging.ini
nano uat.ini
nano prod.ini
```
![alt image](/images/23.png)

✅ Result:

- We now have a Git-controlled project with a scalable structure.

- Environments are isolated via inventory files.

- `common.yaml` serves as your baseline for consistent infrastructure configuration.

---

## Step 4 – Setting Up an Ansible Inventory

An Ansible inventory is the foundation of infrastructure automation. It tells Ansible which servers to target and how to connect to them. For example, you might want to run a task only on your web servers but not on your load balancer or database. With inventory groups, this becomes easy and scalable.

By default, Ansible uses SSH (TCP/22) to connect to remote servers. Since your Jenkins-Ansible instance will be the control node, we’ll configure SSH agent forwarding so Jenkins-Ansible can reach target servers securely.

**1. Enable SSH Agent and Add Your Key**

Start the SSH agent and load your private key (the same one used when launching your EC2 servers):
```bash
# Start ssh-agent
eval `ssh-agent -s`

# Add your private key (replace with your path)
ssh-add ~/.ssh/my-aws-key.pem
```

Confirm the key has been added:
```bash
ssh-add -l
```
![alt image](/images/24.png)

Expected output: fingerprint and name of your key (e.g., `2048 SHA256:xyz my-aws-key.pem`).

**2. SSH into Jenkins-Ansible with Agent Forwarding**

Agent forwarding (-A) allows your Jenkins-Ansible host to connect to other servers without re-entering the key:
```bash
ssh -A ubuntu@<jenkins-ansible-public-ip>
```
![alt image](/images/25.png)
> 🔑 Note:
> - On Ubuntu servers (e.g., load balancer, db), the user is ubuntu.
> - On RHEL-based servers (web, nfs), the user is ec2-user.

**3. Update Inventory File**

Inside your project repo:
```bash
cd ansible-config-mgt/inventory
nano dev.ini
```
![alt image](/images/26a.png)

Paste the following (replace `<private-ip>` values with your actual EC2 internal IPs):
```ini
[nfs]
<nfs server private ip address> ansible_ssh_user=ec2-user

[webservers]
<web server1 private ip address> ansible_ssh_user=ec2-user
<web server2 private ip address> ansible_ssh_user=ec2-user
<web server3 private ip address> ansible_ssh_user=ec2-user

[db]
<database private ip address> ansible_ssh_user=ec2-user

[lb]
<load balancer private ip address> ansible_ssh_user=ubuntu
```
![alt image](/images/26b.png)

- [nfs], [webservers], [db], [lb] are groups you can call in playbooks.
- ansible_ssh_user tells Ansible which user account to use for SSH login.

✅ At this point: Your dev environment is structured, and Ansible knows how to connect to each server group.

## Step 5 – Create a Common Playbook

The next step is to write your first reusable playbook: common.yaml. The goal is to install/update a package (wireshark) across both RHEL 8 and Ubuntu servers, using their respective package managers.

**1. Create the Playbook File**

Inside the playbooks directory:
```bash
cd ../playbooks
nano common.yaml
```
![alt image](/images/27.png)

Paste the following:
```yaml
---
- name: update web, and nfs servers
  hosts: webservers, nfs
  become: yes
  tasks:
    - name: ensure wireshark is at the latest version
      yum:
        name: wireshark
        state: latest

- name: update LB and db server
  hosts: lb, db
  become: yes
  tasks:
    - name: Update apt repo
      apt:
        update_cache: yes

    - name: ensure wireshark is at the latest version
      apt:
        name: wireshark
        state: latest
```

**🔎 Explanation**

- `hosts: webservers, nfs` → targets RHEL hosts.

- `yum` module → package manager for RHEL/CentOS.

- `hosts: lb, db` → targets Ubuntu load balancer.

- `apt` module → package manager for Debian/Ubuntu.

- `become: yes` → escalates privileges (sudo).

    This ensures idempotency: if Wireshark is already installed, Ansible will skip it.

**2. Extend the Playbook (Optional Tasks)**

You can add more common configuration tasks like creating directories, changing timezones, or running scripts. Example:
```yaml
    - name: Create a directory and file
      file:
        path: /opt/app/config.txt
        state: touch

    - name: Change timezone to UTC
      command: timedatectl set-timezone UTC

    - name: Run a shell script
      shell: /home/ubuntu/scripts/deploy.sh
```
![alt image](/images/27b.png)


✅ Result:

- Wireshark is installed/updated on all servers.

- RHEL servers use yum, Ubuntu servers uses apt.

- Infrastructure is now consistent and repeatable, forming the baseline for future automation.

**⚡ DevOps Insight:**

This approach highlights a key Ansible principle: separating inventory from playbooks. The same common.yaml can later run against staging or prod inventories without code duplication — ensuring scalability and environment parity.

---

## Step 6 – Update Git with the Latest Code

With our playbooks and inventory in place, the next step is to push changes to GitHub and let Jenkins handle Continuous Integration (CI). This ensures every update is version-controlled, peer-reviewed, and automatically archived.

**1. Stage and Commit Code**

From your project directory:
```bash
# Check changes
git status  

# Stage specific files (or use . to add all)
git add playbooks/common.yaml inventory/dev.ini  

# Commit changes
git commit -m "Added common playbook and dev inventory"
```

![alt image](/images/28.png)
![alt image](/images/29.png)

**2. Push to Feature Branch**

If you’re working on a feature branch (recommended):
```bash
git push origin dev-setup
```
![alt image](/images/31.png)

**3. Create a Pull Request (PR)**

- Go to GitHub → open your repo → compare & create pull request from feature-ansible-setup into main.

- Add context in the PR description (e.g., “Initial playbook and inventory setup”).

![alt image](/images/30.png)
![alt image](/images/32.png)
![alt image](/images/35.png)

**4. Peer Review**

As in real-world teams, code reviews are crucial. Another developer (or you wearing the reviewer’s hat) reviews the PR for:

- Code quality (proper YAML formatting).

- Correct inventory host definitions.

- Clear commit messages.

If all looks good, approve and merge into main.

![alt image](/images/36.png)
![alt image](/images/37.png)
![alt image](/images/38.png)

**5. Sync Local Repo with Main**

After merging:
```
# Switch back to main
git checkout main  

# Pull latest merged changes
git pull origin main
```
![alt image](/images/39.png)

**6. Jenkins CI Automation**

Once merged, Jenkins (via the configured webhook) triggers a build automatically.

- Jenkins checks out the updated repo.

![alt image](/images/40.png)
![alt image](/images/41.png)

- Build artifacts are stored in:
```bash
/var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/
```

**✅ At this point:**
Your code is safely version-controlled, peer-reviewed, and integrated into Jenkins CI. This process prevents misconfigurations and maintains a single source of truth for automation code.

## Step 7 – Run the First Ansible Test

Now it’s time to validate automation end-to-end by running the common.yaml playbook against dev servers.

**1. Connect with VSCode Remote SSH**

Using the Remote SSH plugin in VSCode, connect to your jenkins-ansible EC2 instance. This allows you to edit code locally in VSCode while executing commands directly on the server.

**2. Execute the Playbook**

Navigate into your repo and run the playbook:
```bash
cd ~/ansible-config-mgt
ansible-playbook -i inventory/dev.ini playbooks/common.yaml
```
![alt image](/images/42.png)
![alt image](/images/44.png)

> - `-i inventory/dev.ini` → tells Ansible which environment inventory to use.
> - `playbooks/common.yaml` → executes the common playbook tasks.

**3. Verify Installation**

On each target server (web, db, nfs, lb), confirm Wireshark was installed:
```bash
# Check if Wireshark exists in PATH
which wireshark  

# Or check version
wireshark --version
```
![alt image](/images/43.png)

Expected output → version number of Wireshark.

**✅ At this point:**

- Jenkins has archived your updated automation code.

- Ansible has successfully executed your first automated configuration across multiple servers.

- Manual installation/configuration is officially replaced with infrastructure automation at scale.

**⚡ DevOps Insight:**

This workflow reflects a GitOps model—all infrastructure changes go through Git, are reviewed, and applied automatically through CI/CD pipelines. It ensures traceability, repeatability, and scalability.

---

# 🔧 Troubleshooting Guide – Ansible Configuration Management Project

While setting up Ansible and integrating it with Jenkins, you may encounter some common issues. Below is a troubleshooting guide based on real-world errors faced during this project.

**1. ansible --version shows old version (2.9.x)**

Problem:

When you install Ansible via apt install ansible, Ubuntu’s default package repository may install an outdated version (e.g., 2.9.x).

Fix:

Remove the system-installed package and install the latest version with pip3:
```
# Remove old version
sudo apt remove -y ansible

# Upgrade pip
pip3 install --upgrade pip

# Install latest stable Ansible for your user
pip3 install ansible --user
```

Add the local bin directory to PATH if not already:

```
echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

✅ Now, ansible --version should show core 2.13+.

**2. no such option: --break-system-packages**

Problem:
When running:
```
pip3 install ansible --user --break-system-packages
```

You may see:
```
no such option: --break-system-packages
```

Cause:

The --break-system-packages flag exists only in newer versions of pip (>=23.0). If your pip version is old (20.x), it won’t recognize the flag.

Fix:

Upgrade pip:
```
pip3 install --upgrade pip --user
```

Then retry without --break-system-packages if not supported:
```
pip3 install ansible --user
```

**3. Virtual Environment Error – ensurepip is not available**

Problem:

When creating a virtual environment:
```
python3 -m venv ~/ansible-venv
```

You may see:
```
The virtual environment was not created successfully because ensurepip is not available
```

Cause:

The package python3-venv is not installed on your system.

Fix:

Install the missing dependency:
```
sudo apt install -y python3.8-venv
```

Recreate the venv:
```
python3 -m venv ~/ansible-venv
source ~/ansible-venv/bin/activate
pip install ansible==2.13.13
```

**4. -bash: /usr/bin/ansible: No such file or directory**

Problem:

After removing the old system Ansible package, running ansible throws:
```
-bash: /usr/bin/ansible: No such file or directory
```

Cause:

The binary was removed, and the new pip-based Ansible is installed in $HOME/.local/bin.

Fix:

Add the correct bin path:
```
echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

**5. Jenkins Webhook Not Triggering Builds**

Problem:

You configured a GitHub webhook but Jenkins builds are not triggered.

Checklist to Fix:

Ensure GitHub webhook URL is correct:
```
http://<jenkins-public-ip>:8080/github-webhook/
```

- Check Jenkins GitHub plugin is installed.

- Verify Jenkins security settings allow anonymous webhook POSTs.

- Ensure your Jenkins instance has a static Elastic IP so webhook doesn’t break after restart.

**6. SSH Issues with Ansible Inventory**

Problem:

Ansible cannot connect to remote hosts and throws errors like:
```
Permission denied (publickey).
```

Fix:

Use ssh-agent to load your private key:
```
eval `ssh-agent -s`
ssh-add <path-to-private-key>
```

Confirm key is loaded:
```
ssh-add -l
```

Use `ssh -A ubuntu@<public-ip>` to enable agent forwarding.

Ensure inventory file has the correct SSH user (ubuntu for Ubuntu, ec2-user for RHEL).

**7. Playbook Syntax Errors (mapping values are not allowed here)**

Problem:

YAML is whitespace-sensitive. Even a wrong indentation can break playbooks.

Fix:

Validate your playbook:
```
ansible-playbook playbooks/common.yml --syntax-check
```

Use consistent 2-space indentation in YAML files.

**8. Wireshark Package Not Found on RHEL**

Problem:

Running the playbook for RHEL servers may fail with:
```
No package wireshark available
```

Fix:

Enable EPEL repository first:
```
- name: enable EPEL repo
  yum:
    name: epel-release
    state: present
```

Then install Wireshark.

### ✅ Summary

By addressing these troubleshooting steps:

- Package versions → resolved via pip3 installs.

- Virtual environment issues → fixed by installing python3-venv.

- PATH issues → fixed by exporting $HOME/.local/bin.

- Jenkins webhooks & SSH keys → resolved by proper configuration.

- Playbook errors → solved with YAML validation.

With this guide, engineers can quickly diagnose and fix common blockers when setting up Ansible automation in a CI/CD pipeline.

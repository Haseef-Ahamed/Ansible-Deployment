# Simplified Ansible Deployment Plan for Students

## Overview
This plan helps 10 students build an Ansible project to automate deploying a web app (Nginx + Flask) and MySQL database. It’s split into 6 phases, with clear tasks for each sub-team. Total time: 10-14 hours.

---

## Phase 1: Understand the Big Picture (1 Hour)
**Why**: Know what we’re building to avoid mistakes.  
**Who**: All 10 students.  
**Tasks**:  
1. **Discuss Goal** (30 min):  
   - Automate deploying:  
     - **Web servers**: Nginx + Flask app.  
     - **DB server**: MySQL with schema.  
     - Use Ansible Vault for passwords.  
     - GitHub Actions for auto-deploy.  
   - Form sub-teams:  
     - Sub-team 1: 4 students (roles).  
     - Sub-team 2: 3 students (CI/CD).  
     - Sub-team 3: 3 students (testing).  
2. **Set Up Tools** (30 min):  
   - Install: `sudo pip install ansible`, Git.  
   - Ensure SSH access to servers (2 web, 1 DB).  
   - Tools: VS Code, terminal, GitHub account.  

**Output**: Team knows plan, tools ready, sub-teams assigned.  

---

## Phase 2: Set Up Project and Inventory (1.5 Hours)
**Why**: Organize files and list servers.  
**Who**: Sub-team 1 (4 students: 2 for structure, 2 for inventory).  
**Tasks**:  
1. **Create Project Folder** (30 min, 2 students):  
   - Run:  
     ```bash
     mkdir -p ansible-deployment/{inventory,playbooks,group_vars,roles/{web,backend,db}/{tasks,templates,handlers,vars,defaults}}
     ```
   - Push to GitHub: `git init`, `git add .`, `git commit -m "Initial structure"`, `git push`.  
2. **Create Inventory** (1 hour, 2 students):  
   - File: `inventory/hosts.ini`  
     ```
     [web]
     web1 ansible_host=192.168.1.10 ansible_user=ubuntu
     web2 ansible_host=192.168.1.11 ansible_user=ubuntu

     [db]
     db1 ansible_host=192.168.1.20 ansible_user=ubuntu

     [dev:children]
     web
     db

     [staging:children]
     web
     db
     ```  
   - Test: `ansible-inventory -i inventory/hosts.ini --list`.  
   - Test servers: `ansible all -i inventory/hosts.ini -m ping`.  
   - Commit to GitHub.  

**Output**: Project folder created, `hosts.ini` lists servers, servers reachable.  

---

## Phase 3: Build Ansible Roles (4-5 Hours)
**Why**: Roles automate Nginx, Flask, and MySQL setup.  
**Who**: Sub-team 1 (4 students: 1 per role + 1 coordinator).  
**Tasks**:  
1. **Scaffold Roles** (30 min, Coordinator):  
   - Run: `ansible-galaxy init roles/web`, repeat for `backend`, `db`.  
2. **Web Role** (1.5 hours, 1 student):  
   - File: `roles/web/tasks/main.yml`  
     ```yaml
     ---
     - name: Install Nginx
       apt:
         name: nginx
         state: present
         update_cache: yes
       become: yes

     - name: Deploy frontend
       git:
         repo: "{{ frontend_repo }}"
         dest: /var/www/html
         version: main
       become: yes

     - name: Copy Nginx config
       template:
         src: nginx.conf.j2
         dest: /etc/nginx/sites-available/default
       become: yes
       notify: restart nginx
     ```  
   - File: `roles/web/templates/nginx.conf.j2`  
     ```
     server {
         listen {{ ansible_default_ipv4.address }}:80;
         server_name _;
         root /var/www/html;
         index index.html;
     }
     ```  
   - File: `roles/web/defaults/main.yml`  
     ```yaml
     frontend_repo: https://github.com/your/frontend.git
     ```  
3. **Backend Role** (1.5 hours, 1 student):  
   - File: `roles/backend/tasks/main.yml`  
     ```yaml
     ---
     - name: Install Python and pip
       apt:
         name: "{{ item }}"
         state: present
       loop:
         - python3
         - python3-pip
       become: yes

     - name: Install Flask
       pip:
         name: flask
       become: yes

     - name: Deploy Flask app
       git:
         repo: "{{ backend_repo }}"
         dest: "{{ flask_app_dir }}"
         version: main
       become: yes

     - name: Copy Flask config
       template:
         src: app.conf.j2
         dest: "{{ flask_app_dir }}/app.conf"
       notify: restart backend
     ```  
   - File: `roles/backend/templates/app.conf.j2`  
     ```plaintext
     DEBUG = {{ debug_mode }}
     DATABASE_URI = mysql://{{ db_user }}:{{ db_app_password }}@{{ groups['db'][0] }}/{{ db_name }}
     ```  
   - File: `roles/backend/defaults/main.yml`  
     ```yaml
     backend_repo: https://github.com/your/backend.git
     flask_app_dir: /opt/backend
     debug_mode: false
     ```  
4. **DB Role** (1.5 hours, 1 student):  
   - File: `roles/db/tasks/main.yml`  
     ```yaml
     ---
     - name: Install MySQL
       apt:
         name: mysql-server
         state: present
       become: yes

     - name: Set MySQL root password
       mysql_user:
         name: root
         password: "{{ db_root_password }}"
         host: localhost
       become: yes
       no_log: true

     - name: Create app database
       mysql_db:
         name: "{{ db_name }}"
         state: present
       become: yes

     - name: Copy MySQL config
       template:
         src: my.cnf.j2
         dest: /etc/mysql/my.cnf
       become: yes
       notify: restart mysql
     ```  
   - File: `roles/db/templates/my.cnf.j2`  
     ```plaintext
     [mysqld]
     bind-address = {{ db_bind_address }}
     ```  
   - File: `roles/db/defaults/main.yml`  
     ```yaml
     db_name: myapp
     db_user: appuser
     db_bind_address: 0.0.0.0
     ```  
5. **Main Playbook** (30 min, Coordinator):  
   - File: `playbooks/deploy.yml`  
     ```yaml
     ---
     - hosts: db
       roles:
         - db

     - hosts: web
       roles:
         - backend
         - web
     ```  
   - Commit all to GitHub.  

**Output**: Roles for `web`, `backend`, `db` with tasks, templates, vars. Playbook ready.  

---

## Phase 4: Secure Secrets and Add Handlers (1.5 Hours)
**Why**: Protect passwords, restart services on changes.  
**Who**: Sub-team 1 (4 students: 2 for secrets, 2 for handlers).  
**Tasks**:  
1. **Ansible Vault** (45 min, 2 students):  
   - Run: `ansible-vault create group_vars/db.yml` (set password).  
   - Content:  
     ```yaml
     db_root_password: supersecret
     db_app_password: appsecret
     ```  
   - Update `roles/db/tasks/main.yml` to use `db_app_password`.  
   - Commit (encrypted).  
2. **Handlers** (45 min, 2 students):  
   - File: `roles/web/handlers/main.yml`  
     ```yaml
     ---
     - name: restart nginx
       systemd:
         name: nginx
         state: restarted
       become: yes
     ```  
   - File: `roles/backend/handlers/main.yml`  
     ```yaml
     ---
     - name: restart backend
       systemd:
         name: backend
         state: restarted
       become: yes
     ```  
   - File: `roles/db/handlers/main.yml`  
     ```yaml
     ---
     - name: restart mysql
       systemd:
         name: mysql
         state: restarted
       become: yes
     ```  
   - Commit to GitHub.  

**Output**: Secrets in `db.yml`, handlers for service restarts.  

---

## Phase 5: GitHub Actions for CI/CD (2 Hours)
**Why**: Auto-deploy to dev (push) and staging (PR merge).  
**Who**: Sub-team 2 (3 students: 2 for workflows, 1 for secrets).  
**Tasks**:  
1. **GitHub Secrets** (30 min, 1 student):  
   - In GitHub > Settings > Secrets and variables > Actions:  
     - `ANSIBLE_VAULT_PASS`: Vault password.  
     - `SSH_PRIVATE_KEY`: SSH key for servers.  
     - `INVENTORY_DEV`: Content of `hosts.ini`.  
2. **Dev Workflow** (45 min, 1 student):  
   - File: `.github/workflows/deploy-dev.yml`  
     ```yaml
     name: Deploy to Dev
     on:
       push:
         branches: [ main ]
     jobs:
       deploy:
         runs-on: ubuntu-latest
         steps:
         - uses: actions/checkout@v4
         - name: Install Ansible
           run: pip install ansible
         - name: Set up SSH
           run: |
             mkdir -p ~/.ssh
             echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
             chmod 600 ~/.ssh/id_rsa
         - name: Deploy
           env:
             ANSIBLE_VAULT_PASSWORD: ${{ secrets.ANSIBLE_VAULT_PASS }}
           run: |
             echo "${{ secrets.INVENTORY_DEV }}" > inventory/dev.ini
             ansible-playbook -i inventory/dev.ini playbooks/deploy.yml --limit dev --vault-id @<(echo $ANSIBLE_VAULT_PASSWORD)
     ```  
3. **Staging Workflow** (45 min, 1 student):  
   - File: `.github/workflows/deploy-staging.yml`  
     ```yaml
     name: Deploy to Staging
     on:
       pull_request:
         types: [closed]
       if: github.event.pull_request.merged == true
     jobs:
       deploy:
         runs-on: ubuntu-latest
         steps:
         - uses: actions/checkout@v4
         - name: Install Ansible
           run: pip install ansible
         - name: Set up SSH
           run: |
             mkdir -p ~/.ssh
             echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
             chmod 600 ~/.ssh/id_rsa
         - name: Deploy
           env:
             ANSIBLE_VAULT_PASSWORD: ${{ secrets.ANSIBLE_VAULT_PASS }}
           run: |
             echo "${{ secrets.INVENTORY_DEV }}" > inventory/staging.ini
             ansible-playbook -i inventory/staging.ini playbooks/deploy.yml --limit staging --vault-id @<(echo $ANSIBLE_VAULT_PASSWORD)
     ```  
   - Commit workflows.  

**Output**: CI/CD for dev and staging.  

---

## Phase 6: Test and Validate (1-2 Hours)
**Why**: Ensure everything works.  
**Who**: Sub-team 3 (3 students: 2 testers, 1 coordinator).  
**Tasks**:  
1. **Dry-Run** (30 min, 1 tester):  
   - Run: `ansible-playbook -i inventory/hosts.ini playbooks/deploy.yml --check --diff --ask-vault-pass`.  
2. **Live Test** (1 hour, 1 tester):  
   - Run: `ansible-playbook -i inventory/hosts.ini playbooks/deploy.yml --limit dev --ask-vault-pass`.  
   - Check:  
     - Nginx: `curl http://web1`.  
     - Flask: `curl http://web1:5000`.  
     - MySQL: `mysql -u appuser -p -h db1 myapp`.  
3. **Test CI/CD** (30 min, Coordinator):  
   - Push to `main` (dev deploy).  
   - Merge PR (staging deploy).  
   - Check GitHub Actions logs.  

**Output**: Deployment works, services run, CI/CD triggers.  

---

## Team Roles
- **Sub-team 1 (4 students)**: Build roles, playbook, secrets, handlers.  
- **Sub-team 2 (3 students)**: Set up CI/CD workflows, secrets.  
- **Sub-team 3 (3 students)**: Test and validate.  

## Tips
- Talk daily (Discord/Slack).  
- Commit often with clear messages.  
- Test small changes with `--check`.  
- Ask for help if stuck (share errors).  

## Timeline
- **Day 1**: Phases 1-2.  
- **Day 2**: Phase 3.  
- **Day 3**: Phases 4-5.  
- **Day 4**: Phase 6.
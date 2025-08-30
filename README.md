

# **Infrastructure Automation with Ansible: Web + Database**

Automate a multi-server web application using **Ansible**. Sets up web servers (Nginx), database servers (PostgreSQL), firewalls, backups, and reporting—ensuring deployments are fast, safe, and repeatable.

---

## **Project Structure & Example Files**

```
ansible_project/
├── inventories/
│   └── hosts.yml
├── playbooks/
│   ├── main.yml
│   ├── user_setup.yml
│   ├── webservers.yml
│   └── dbservers.yml
├── roles/
│   ├── webserver/
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── install_nginx.yml
│   │   │   ├── deploy_app.yml
│   │   │   └── cron_backup.yml
│   │   ├── handlers/
│   │   │   └── main.yml
│   │   └── templates/
│   │       ├── nginx.conf.j2
│   │       └── index.html.j2
│   └── dbserver/
│       ├── tasks/
│       │   ├── main.yml
│       │   ├── install_postgres.yml
│       │   ├── create_db.yml
│       │   └── firewall.yml
│       └── handlers/
│           └── main.yml
├── group_vars/
│   ├── webservers.yml
│   └── dbservers.yml
└── reports/
```

---

### **1. Inventory** (`inventories/hosts.yml`)

```yaml
all:
  children:
    webservers:
      hosts:
        web1.example.com:
        web2.example.com:
    dbservers:
      hosts:
        db1.example.com:
        db2.example.com:
```

---

### **2. Group Variables**

**`group_vars/webservers.yml`**

```yaml
nginx_port: 80
backup_dir: /var/backups
app_index_template: "index.html.j2"
cron_time: "0 2 * * *"
```

**`group_vars/dbservers.yml`**

```yaml
postgres_db: company_db
postgres_user: company_user
postgres_password: secure_password
db_port: 5432
```

---

### **3. User Setup Playbook** (`playbooks/user_setup.yml`)

```yaml
- hosts: all
  become: yes
  tasks:
    - name: Create devops user
      user:
        name: devops
        state: present
        shell: /bin/bash

    - name: Grant passwordless sudo
      copy:
        dest: /etc/sudoers.d/devops
        content: "devops ALL=(ALL) NOPASSWD:ALL"
        mode: '0440'
```

---

### **4. Main Playbook** (`playbooks/main.yml`)

```yaml
- import_playbook: user_setup.yml
- import_playbook: webservers.yml
- import_playbook: dbservers.yml
```

---

### **5. Webserver Role Tasks**

**`roles/webserver/tasks/main.yml`**

```yaml
- import_tasks: install_nginx.yml
- import_tasks: deploy_app.yml
- import_tasks: cron_backup.yml
```

**`roles/webserver/tasks/install_nginx.yml`**

```yaml
- name: Remove Apache if installed
  apt:
    name: apache2
    state: absent
  become: yes

- name: Install Nginx
  apt:
    name: nginx
    state: present
  become: yes

- name: Configure Nginx
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/sites-available/default
  notify: restart nginx
  become: yes

- name: Ensure Nginx is running and enabled
  service:
    name: nginx
    state: started
    enabled: yes
  become: yes

- name: Open firewall port 80
  ufw:
    rule: allow
    port: "{{ nginx_port }}"
    proto: tcp
  become: yes
```

**`roles/webserver/tasks/deploy_app.yml`**

```yaml
- name: Deploy index.html
  template:
    src: "{{ app_index_template }}"
    dest: /var/www/html/index.html
  become: yes
```

**`roles/webserver/tasks/cron_backup.yml`**

```yaml
- name: Create backup directory
  file:
    path: "{{ backup_dir }}"
    state: directory
    mode: '0755'
  become: yes

- name: Setup daily backup cron
  cron:
    name: "Daily web backup"
    minute: "0"
    hour: "2"
    job: "tar czf {{ backup_dir }}/html_backup_$(date +\\%F).tar.gz /var/www/html"
  become: yes
```

**`roles/webserver/handlers/main.yml`**

```yaml
- name: restart nginx
  service:
    name: nginx
    state: restarted
  become: yes
```

---

### **6. DBserver Role Tasks**

**`roles/dbserver/tasks/main.yml`**

```yaml
- import_tasks: install_postgres.yml
- import_tasks: create_db.yml
- import_tasks: firewall.yml
```

**`roles/dbserver/tasks/install_postgres.yml`**

```yaml
- name: Install PostgreSQL
  apt:
    name: postgresql
    state: present
  become: yes

- name: Ensure PostgreSQL service is running
  service:
    name: postgresql
    state: started
    enabled: yes
  become: yes
```

**`roles/dbserver/tasks/create_db.yml`**

```yaml
- name: Create database
  postgresql_db:
    name: "{{ postgres_db }}"
    state: present
  become: yes
  become_user: postgres

- name: Create user
  postgresql_user:
    name: "{{ postgres_user }}"
    password: "{{ postgres_password }}"
    priv: "{{ postgres_db }}:ALL"
    state: present
  become: yes
  become_user: postgres
```

**`roles/dbserver/tasks/firewall.yml`**

```yaml
- name: Open PostgreSQL port
  ufw:
    rule: allow
    port: "{{ db_port }}"
    proto: tcp
  become: yes
```

**`roles/dbserver/handlers/main.yml`**

```yaml
- name: restart postgresql
  service:
    name: postgresql
    state: restarted
  become: yes
```

---

### **7. Templates**

**`roles/webserver/templates/nginx.conf.j2`**

```nginx
server {
    listen {{ nginx_port }};
    server_name _;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

**`roles/webserver/templates/index.html.j2`**

```html
<html>
<head>
    <title>Welcome to Ansible Webserver</title>
</head>
<body>
    <h1>Deployment Successful!</h1>
    <p>This page was deployed using Ansible.</p>
</body>
</html>
```

---

### **8. Quick Start / Usage Hint**

```bash
# Test connection
ansible -i inventories/hosts.yml all -m ping

# Run main playbook
ansible-playbook -i inventories/hosts.yml playbooks/main.yml \
  2> /tmp/ansible_errors.log | tee /tmp/ansible_report.log
```

* Check `/tmp/ansible_report.log` for disk usage.
* Check `/tmp/ansible_errors.log` for failed tasks.
* Idempotent playbooks: safe to run multiple times.

---



Do you want me to add those as well?

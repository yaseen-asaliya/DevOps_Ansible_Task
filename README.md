# DevOps_Ansible_Task

## Install and configure zabbix(server and agent on vm’s cluster) using Ansible
* Install ansible on your local vm 
```
# yum install epel-release
# yum install ansible
```


### Configure ssh keys to connect to remote machines
- Generate ssh key in `/root/.ssh`
```
# ssh-keygen
```
- Copy key to the destination servers
```
# cd /root/.ssh
# ssh-copy-id -i id_rsa.pub root@10.0.2.4
# ssh-copy-id -i id_rsa.pub root@10.0.2.5
# ssh-copy-id -i id_rsa.pub root@10.0.2.6
```
- To conncetion to the remote machines using ssh use this format `username@hostIp`, and here is some examples based on our remote machines
```
# ssh root@10.0.2.5
# ssh root@10.0.2.6
```
> All subsequent steps will generate inside `ansible_task` directory.





### Configure ansible hosts and host group(local,zabbix-server, zabbix-agent)
* Create file in `ansible_task/inventory.txt`
```
# vi ansible_task/inventory.txt
```
* List all the hosts as a groubs inside `inventory.txt`
```
[local]
controller ansible_host=10.0.2.4

[zabbix-server]
zabbix_server_1 ansible_host=10.0.2.5

[zabbix-agent]
zabbix_agent_1 ansible_host=10.0.2.6
```

 


### Create apach server role & configure port 80 and SElinux
- Create role structure
```
# ansible-galaxy init apache-server
```
- Write apache server configuration in `ansible_task/roles/apache-server/tasks/main.yml`
```
---
# tasks file for apache-server & open port 80 and SELinux to make sure that is accessible

- name: Disable SELINUX
  lineinfile:
    dest: /etc/sysconfig/selinux
    regexp: '^SELINUX=enforcing'
    line: 'SELINUX=disabled'
    
# Check port 80 & selinux
- name: Open port 80 in firewall
  firewalld:
    state: enabled
    port: 80/tcp
    permanent: true
    
- name: Reboot system
  reboot:

- name: Install Apache server
  yum:
    name: httpd
    state: present

- name: Start Apache service
  service:
    name: httpd
    state: started
    enabled: true
```





### Create zabbix configrations role
- Create role structure
```
# ansible-galaxy init zabbix-configrations
```
- Write zabbix configrations in `ansible_task/roles/zabbix-configrations/tasks/main.yml`
```
---
# tasks file for apache-server

- name: Download and install EPEL repository
  shell: wget http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm && rpm -Uvh epel-release-latest-7*.rpm

- name: Install epel-release package
  yum:
    name: epel-release
    state: present

- name: Install remi-release package
  yum:
    name: http://rpms.remirepo.net/enterprise/remi-release-7.rpm
    state: present

- name: Disable remi-php54 repository
  command: yum-config-manager --disable remi-php54

- name: Enable remi-php72 repository
  command: yum-config-manager --enable remi-php72

- name: Install PHP packages
  yum:
    name:
      - php
      - php-pear
      - php-cgi
      - php-common
      - php-mbstring
      - php-snmp
      - php-gd
      - php-pecl-mysql
      - php-xml
      - php-mysql
      - php-gettext
      - php-bcmath
    state: present

- name: Uncomment date.timezone in /etc/php.ini
  lineinfile:
    dest: /etc/php.ini
    regexp: '^;date.timezone ='
    line: 'date.timezone = Australia/Sydney'
    state: present

- name: Install MySQL-python
  yum:
    name: MySQL-python
    state: present

- name: Install MariaDB
  yum:
    name: mariadb-server
    enablerepo: remi
    state: present

- name: Start MariaDB service
  service:
    name: mariadb
    state: started
    enabled: true

- name: Run mysql_secure_installation
  expect:
    command: mysql_secure_installation
    responses:
      "Enter current password for root (enter for none):": "{{ root_pass }}\n"
      "Set root password? [Y/n]": "y\n"
      "New password:": "{{ new_root_pass }}\n"
      "Re-enter new password:": "{{ new_root_pass }}\n"
      "Remove anonymous users? [Y/n]": "y\n"
      "Disallow root login remotely? [Y/n]": "y\n"
      "Remove test database and access to it? [Y/n]": "y\n"
      "Reload privilege tables now? [Y/n]": "y\n"

- name: Log in to MariaDB as root user
  mysql_user:
    name: root
    password: "{{ root_pass }}"
    host: localhost
    login_user: root
    login_password: "{{ root_pass }}"
  become: true
  become_user: root

- name: Create fosslinuxzabbix database
  mysql_db:
    name: fosslinuxzabbix
    state: present
    encoding: utf8
    collation: utf8_bin
    login_user: root
    login_password: "{{ root_pass }}"
  become: true
  become_user: root

- name: Create zabbixuser user and grant privileges on fosslinuxzabbix database
  mysql_user:
    name: zabbixuser
    password: "{{ root_pass }}"
    priv: 'fosslinuxzabbix.*:ALL'
    state: present
    login_user: root
    login_password: "{{ root_pass }}"
  become: yes
  become_user: root

- name: Flush privileges
  command: mysql -u root -p"{{ root_pass }}" -e "FLUSH PRIVILEGES"

- name: Install Zabbix repository
  become: true
  command: rpm -Uvh https://repo.zabbix.com/zabbix/4.0/rhel/7/x86_64/zabbix-release-4.0-1.el7.noarch.rpm

```
* Set variables inside `ansible_task/roles/zabbix-configrations/vars/main.yml` to use them in tasks
```
---
# vars file for zabbix-configrations

root_pass: "osboxes.org"
new_root_pass: "osboxes.org"
zabbix_user_pass: "osboxes.org"
```


### Create zabbix server role
- Create role structure
```
# ansible-galaxy init zabbix-server
```
- Write zabbix server configrations in `ansible_task/roles/zabbix-server/tasks/main.yml`
```
---
# tasks file for zabbix-server

- name: Install the Zabbix server
  yum:
    name: "{{ item }}"
    state: present
  with_items:
    - zabbix-server-mysql
    - zabbix-web-mysql

- name: Start Zabbix server service
  service:
    name: zabbix-server
    state: started
    enabled: true

- name: Uncomment timezone line
  lineinfile:
    path: /etc/httpd/conf.d/zabbix.conf
    regexp: '^# php_value date.timezone*'
    line: 'php_value date.timezone Australia/Sydney'

- name: Restart httpd
  systemd:
    name: httpd
    state: restarted

- name: Import database
  shell: zcat /usr/share/doc/zabbix-server-mysql-4.0.44/create.sql.gz | mysql -u zabbixuser -posboxes.org fosslinuxzabbix

- name: Modify zabbix_server.conf file
  lineinfile:
    path: /etc/zabbix/zabbix_server.conf
    regexp: '^{{ item.name }}='
    line: '{{ item.name }}={{ item.value }}'
  with_items:
    - { name: DBHost, value: localhost }
    - { name: DBName, value: fosslinuxzabbix }
    - { name: DBUser, value: zabbixuser }
    - { name: DBPassword, value: "{{ database_pass }}" }

- name: Restart zabbix-server service
  service:
    name: zabbix-server
    state: restarted

- name: Enable zabbix-server service
  service:
    name: zabbix-server
    enabled: yes

- name: Add HTTP service to firewall
  firewalld:
    service: http
    permanent: yes
    state: enabled

- name: Add HTTPS service to firewall
  firewalld:
    service: https
    permanent: yes
    state: enabled

- name: Reload firewall
  systemd:
    name: firewalld
    state: reloaded

- name: Add Zabbix port to firewall
  firewalld:
    port: 10051/tcp
    permanent: yes
    state: enabled
  register: result
- name: Add Zabbix port to firewall
  firewalld:
    port: 10050/tcp
    permanent: yes
    state: enabled
  when: result.changed

- name: Restart Apache service
  service:
    name: httpd
    state: restarted

```
* Set variables inside `ansible_task/roles/zabbix-server/vars/main.yml` to use them in tasks
```
---
# vars file for zabbix-server

database_pass: osboxes.org
```


### Create zabbix agent role
- Create role structure
```
# ansible-galaxy init zabbix-agent
```
- Write zabbix configrations in `ansible_task/roles/zabbix-agent/tasks/main.yml
```
---
# vars file for zabbix-agent[root@ansiblecontroller zabbix-agent]# cat tasks/*
---
# tasks file for zabbix-agent

- name: Install Zabbix Agent
  yum:
    name: zabbix-agent
    state: present

- name: Modify agent configration file
  lineinfile:
    path: /etc/zabbix/zabbix_agentd.conf
    regexp: '^{{ item.name }}='
    line: '{{ item.name }}={{ item.value }}'
  with_items:
    - { name: Server, value: 10.0.2.6 }
    - { name: ServerActive, value: 10.0.2.6 }
    - { name: Hostname, value: target2 }


- name: Restart Zabbix agent service
  service:
    name: zabbix-agent
    state: restarted

- name: Enable Zabbix agent service
  service:
    name: zabbix-agent
    enabled: yes

```


### Create playbook inside `ansible_task` to create local yum repositorys for all hosts & set roles for local server, zabbix server, and zabbix agent
```
---
# Create local yum repositorys on all hosts & configure zabbix server and agent

# create repos in all hosts

- hosts: all
  tasks:
  - name: Create local yum repository directory
    file:
      path: /var/www/html/localrepo
      state: directory

  - name: Download Zabbix repository package
    get_url:
      url: https://repo.zabbix.com/zabbix/4.4/rhel/7/x86_64/
      dest: /var/www/html/localrepo

  - name: Create repodata for local yum repository
    shell: createrepo /var/www/html/localrepo

  - name: Create yum repo configration file
    blockinfile:
      path: /etc/yum.repos.d/localrepo.repo
      block: |
        [localrepo]
        name=Apache
        baseurl=file:/var/www/html/localrepo
        enabled=1
        gpgcheck=0
      create: yes


# Configure zabbix server and agent

- name: Install Apache server on local machines
  hosts: local
  roles:
    - roles/apache-server
    - roles/zabbix-configrations

- name: Configer Zabbix-server
  hosts: zabbix-server
  roles:
    - roles/apache-server
    - roles/zabbix-configrations
    - roles/zabbix-server

- name: Configer Zabbix-agent
  hosts: zabbix-agent
  roles:
    - roles/apache-server
    - roles/zabbix-configrations
    - roles/zabbix-agent
```
* execute playbook to apply all changes
```
# ansible-playbook playbook.yml -i inventory.txt
```

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
controller ansible_host=10.0.2.4 ansible_ssh_pass=osboxes.org

[zabbix-server]
zabbix_server_1 ansible_host=10.0.2.5 ansible_ssh_pass=osboxes.org

[zabbix-agent]
zabbix_agent_1 ansible_host=10.0.2.6 ansible_ssh_pass=osboxes.org
```



### Create apach server role & configure port 80 and SElinux
- Create role structure
```
# ansible-galaxy init apache-server
```
- Write apache server configuration in `~/ansible_task/roles/apache-server/tasks/main.yml`
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

* Create playbook inside `ansible_task` to create local yum repositorys for all hosts & set roles for local server, zabbix server, and zabbix agent
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

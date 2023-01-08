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



### Create apach server role
- Create role structure
```
# ansible-galaxy init apache-server
```
- Write apache server configuration in `~/ansible_task/roles/apache-server/tasks/main.yml`
```
---
# tasks file for apache-server

- name: Disable SELINUX
  lineinfile:
    dest: /etc/sysconfig/selinux
    regexp: '^SELINUX=enforcing'
    line: 'SELINUX=disabled'
    
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

# Check port 80 & selinux
- name: Open port 80 in firewall
  firewalld:
    state: enabled
    port: 80/tcp
    permanent: true
```





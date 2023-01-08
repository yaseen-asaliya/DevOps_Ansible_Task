# DevOps_Ansible_Task

## Install and configure zabbix(server and agent on vm’s cluster) using Ansible:
* Install ansible on your local vm 
```
# yum install epel-release
# yum install ansible
```


### Configure ssh keys to connect to remote machines
- generate ssh key in `/root/.ssh`
```
ssh-keygen
```
- copy key to the destination servers
```
# cd /root/.ssh
# ssh-copy-id -i id_rsa.pub root@10.0.2.5
# ssh-copy-id -i id_rsa.pub root@10.0.2.6
```
- to test ssh conncetion use one of these
```
# ssh root@10.0.2.5
# ssh root@10.0.2.6
```


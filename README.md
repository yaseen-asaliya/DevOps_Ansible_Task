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
- To conncetion to the remote machines using ssh use this format `username@hostnumber`, and here is some examples based on our remote machines
```
# ssh root@10.0.2.5
# ssh root@10.0.2.6
```


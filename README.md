# docker EE:

## Installation on centos digital ocean droplet:
- ssh-key => generate public and private keys
- ssh -i ~/.ssh/id_rsa root@165.227.37.133
- [root@node1 ~]# yum update -y
- [root@node1 ~]# yum install -y wget
- [root@node1 ~]# export DOCKERURL="https://storebits.docker.com/ee/trial/sub-xxxx-fcfa-xxx-8fef-xxx" (copy from docker hub)
- [root@node1 ~]# wget $DOCKERURL/centos/docker-ee.repo
- root@node1 ~]# cp docker-ee.repo /etc/yum
- [root@node1 ~]# sudo -E sh -c 'echo "$DOCKERURL/centos" > /etc/yum/vars/dockerurl'
- [root@node1 ~]# yum -y install docker-ee docker-ee-cli containerd.io
- [root@node1 ~]# systemctl status docker
● docker.service - Docker Application Container Engine <br>
   Loaded: loaded (/usr/lib/systemd/system/docker.service; disabled; vendor preset: disabled) <br>
   Active: inactive (dead) <br>
     Docs: https://docs.docker.com <br>
- [root@node1 ~]# systemctl start docker
## Activating License
- Either by manually uploading license file to ther server and activating it or login and activate the license
<br> On local machine download the license file and upload it to the server
- $ scp -i /Users/xxx/.ssh/id_rsa docker_subscription.lic root@165.227.37.133:/root
<br>  On the server:
<br>  ===
- [root@node1 ~]# docker engine activate --license docker_subscription.lic 
<br> License: Quantity: 10 Nodes	Expiration date: 2020-01-04	License is currently active
<br> Successfully activated engine license on existing enterprise engine.



# docker EE:

## Installation on centos digital ocean droplet:
- ssh-key => generate public and private keys
- ssh -i ~/.ssh/id_rsa root@165.227.37.133
- [root@node1 ~]# yum update -y
- [root@node1 ~]# yum install -y wget
- [root@node1 ~]# export DOCKERURL="https://storebits.docker.com/ee/trial/sub-xxxx-fcfa-xxx-8fef-xxx" (copy from docker hub)
- [root@node1 ~]# wget $DOCKERURL/centos/docker-ee.repo
- root@node1 ~]# cp docker-ee.repo /etc/yum.repo.d
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

## UCP - Universal control Plane
https://docs.docker.com/ee/ucp/ucp-architecture/
https://docs.docker.com/v17.09/datacenter/ucp/2.1/guides/admin/install/system-requirements/
```
[root@node1 ~]# docker container run --rm -it --name ucp \
> -v /var/run/docker.sock:/var/run/docker.sock \
> docker/ucp:3.1.4 install --host-address 165.227.37.133 \
> --force-minimums
```
- You will get docker URL, username and password in the output of above cmd. You can login and upload the license file to continue.
- Min reqmt for UCP
-- 8 GB RAM for Mgr Node, 4GB for worker nodes
-- 5 GB free space for /var partition for mgr nodes, 500 MB for worker nodes
* Access Control
  * Subject - individual user, org, team
  * Role - is a set of permitted API operations that you can assign to a specific subject and collection (No access, Read Only, Restricted Control, Full Control)
  * Collection - Swarm resources viz. Nodes, Containers, Services, Networks, Volumes, Secrets, Application Configs



## Storage Driver
http://100daysofdevops.com/21-days-of-docker-day-13-docker-storage-part-2/2/
<br> [root@node1 overlay2]# docker system info | grep -i storage
<br> Storage Driver: overlay2
<br> [root@node1 overlay2]# pwd
<br> /var/lib/docker/overlay2
<br> [root@node1 overlay2]# ls -l
<br>  total 0
<br> brw-------. 1 root root 253, 1 Dec 18 22:17 backingFsBlockDev
<br> drwx------. 2 root root      6 Dec 18 16:38 l


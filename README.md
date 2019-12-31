# docker EE:

## Installation on centos digital ocean droplet:
- ssh-key => generate public and private keys
- ssh -i ~/.ssh/id_rsa root@165.227.37.133
- [root@node1 ~]# yum update -y
- [root@node1 ~]# yum install -y wget
- [root@node1 ~]# export DOCKERURL="https://storebits.docker.com/ee/trial/sub-xxxx-fcfa-xxx-8fef-xxx" (copy from docker hub)
- [root@node1 ~]# wget $DOCKERURL/centos/docker-ee.repo
- root@node1 ~]# cp docker-ee.repo /etc/yum.repos.d/
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
https://success.docker.com/article/docker-enterprise-best-practices
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

# DTR - Docker Trusted Registry
- https://docs.docker.com/ee/dtr/architecture/
- containerized application that runs on top of UCP
- 16 GB of RAM for nodes running DTR
- 2 vCPUs for node running DTR
- 10 GB of free space
- To install DTR, all nodes must be a worker node managed by UCP. Must have fixed hostname.
```
docker run -it --rm docker/dtr install \
  --ucp-node node2 \
  --ucp-username admin \
  --ucp-url https://165.227.37.133 \
  --ucp-insecure-tls
 ```
 - To uninstall
 ```
 docker run -it --rm docker/dtr destroy --ucp-insecure-tls
 ```
 <br> Enter ucp URL, username, password
 
 Also on manager node:
 ```
 [root@node1 ~]# docker exec -e ETCDCTL_API=3 ucp-kv etcdctl --endpoints https://127.0.0.1:2379 get /orca/v1/config2/dtr/ --prefix --keys-only 2>/dev/null
/orca/v1/config2/dtr/138.197.164.234

[root@node1 ~]# docker exec -e ETCDCTL_API=3 ucp-kv etcdctl --endpoints https://127.0.0.1:2379 del /orca/v1/config2/dtr/138.197.164.234 2>/dev/null
1
[root@node1 ~]# docker exec -e ETCDCTL_API=3 ucp-kv etcdctl --endpoints https://127.0.0.1:2379 get /orca/v1/config2/dtr/ --prefix --keys-only 2>/dev/null
[root@node1 ~]# 
```
## Configuring Trusted CA  & pushing images to DTR
- wget https://138.197.164.234/ca --no-check-certificate
- cp ca /etc/pki/ca-trust/source/anchors/example.com.crt
- update-ca-trust
- add entry in /etc/hosts
- systemctl restart docker
- docker login example.com
- docker pull busybox
- docker tag busybox:latest example.com/admin/webserver:v1
- docker push example.com/admin/webserver:v1

## DTR Backup
- doesnt cause downtime
- doesnt backup images stored in your registry or users and orgs (done by UCP bkp)
- https://docs.docker.com/ee/admin/backup/back-up-dtr/
```
docker container run --log-driver none -i --rm docker/dtr backup \
--ucp-url https://165.227.37.133:443 -ucp-insecure-tls --ucp-username admin 
--ucp-password xxx > backup.tar
```

To backup images get the DTR volume :
cd /var/lib/docker/volumes/<br>
tar -czvf dtr-registry-backup.tar.gz dtr-reistry-REPLICAID

## Limiting CPU for containers
- --cpus=1 (if u have 2 CPUs, this guarantees at most 1 CPU. You can also provide 0.5 as a value)
- --cpuset-cpus (limits the no. of CPU cores a container can use. Takes comma separated list or hyphen seperated range of CPUs a container can use. 1st container is numbered 0. 0-2 means usage of 1, 2nd and 3rd CPU. 1,3 means 1st and 2nd CPU)
- By default, a container can use as much resource as the host's kernel scheduler allows. On Linuz on OOM, linux starts killing process to free up memory.
- Limit (-m, --memory) = hard limit. Reservation (--memory-reservation) = soft limit. When there is a low memory, the reservation tries to bring container's memory consumption at or below the reservation limit.
- docker container run -dt --name c1 -m 500m --memory-reservation 250m busybox sh | free -m

## Swarm Mutual TLS (MTLS)
- On a worker node and manager node
<br> [root@node2 ~]# cd /var/lib/docker/swarm/certificates
<br> [root@node2 certificates]# ls
<br> swarm-node.crt  swarm-node.key  swarm-root-ca.crt
- On worker docker swarm leave => the certificates are gone | On master do "docker node rm ID" | docker node ls -> no wrk node
- docker swarm ca --rotate (You will see that .crt and .key are rotated on all nodes)
- You can check md5sum swarm-node.crt
- if you run "docker swarm join-token SWMTKN-1-OLD 165.227.37.133:2377" => Error response from daemon: remote CA does not match fingerprint. Expected: xxxx"
  - This is because join token is associated w/ the CA and it's rotated now. So we need new join token "docker swarm join-token worker" and use this to join a node on worker.
  - By default each node in the swarm renews its certificate every 3 months. 
    - Override by "docker swarm --update --cert-expiry 2160h0m0s
  

# UCP Client Bundles
- is a group of certificates downloadable directly from UCP
- Depending upon the permission associated with the user, you can now execute docker swarm commands from your remote machine that take effect on remote cluster. Like u can create a new svc in UCP from ur laptop or login to remote container from ur laptop without SSH via API
- Download client bundle locally
- Local machine:
  - docker run -dt --name myubuntu ubuntu
  - docker cp "ucp-bundle-admin.zip" myubuntu:/tmp
  - docker exec -it myubunti bash
 ```
 /# mkdir ucp
 /# cp /tmp/ucp-*.zip ucp
 /# cd ucp && unzip *.zip
 /# cat env.sh (See that it has export DOCKER_HOST=tcp://ucp-url etc)
 /# eval "$(<env.sh)" 
 /# echo DOCKER_HOST
 /# curl -sSL https://get.docker.com | sh
 /#...
 /# docker info -> shows docker ee on remote machine where UCP is installed
 /# docker service create --name mydemoservice nginx => created on UCP machine
 ```
 
 ## Docker Content Trust
 - gives ability to verify boh the integrity & the publisher of all the data received from registry over any channel
 - docker trust inspect IMAGE => gives signers
 - export DOCKER_CONTENT_TRUST=1
 - https://docs.docker.com/engine/security/trust/content_trust/
 
 ## Docker group
 - useradd newuser | su - newuser | docker ps => error 
 <br> "Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get http://%2Fvar%2Frun%2Fdocker.sock/v1.40/containers/json: dial unix /var/run/docker.sock: connect: permission denied <br>
 - vi /etc/group => docker:x:993: newuser OR 
 - sudo usermod -aG docker newuser
 
 ## Linux capabilities
 
 - https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities
 ```
docker run -dt —name cap01 —cap-add LINUX_IMMUTABLE amazonlinux
docker exec -it cap01 bash
# yum what provides chattr
# yum -y install e2fsprogs
# touch test.txt
# chattr +i test.txt
```
 ## Priviledged Containers
 - Docker containers are not allowed to access any devices on the host
- Hence docker container can’t, by default, run use cases like docker containers inside a docker container
- Privileged containers can access all the devices on the host as well as have configs in AppArmor or SELinux to allow the container nearly all the same access to the host as processes running outside containers on the host
 - docker run -dt —privileged amzonlinux bash


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

<br> The new l (lowercase L) directory contains shortened layer identifiers as symbolic links. 
<br> These identifiers are used to avoid hitting the page size limitation on arguments to the mount command.
```
$ ls -l /var/lib/docker/overlay2/l
total 0
[node1] (local) root@192.168.0.18 ~
$ docker pull alpine
Using default tag: latest
latest: Pulling from library/alpine
e6b0cf9c0882: Pull complete 
Digest: sha256:2171658620155679240babee0a7714f6509fae66898db422ad803b951257db78
Status: Downloaded newer image for alpine:latest
docker.io/library/alpine:latest
[node1] (local) root@192.168.0.18 ~
$ ls -l /var/lib/docker/overlay2/l
total 0
lrwxrwxrwx    1 root     root            72 Dec 31 17:11 YJHCGPQJLO6Q65JWH7XUBTI2OX -> ../f90df637b413292da87ac997ec07912778e7a84f6d8c0a091b9d37461f7b1ae7/diff
[node1] (local) root@192.168.0.18 ~
$ 
```

```
$ cd /var/lib/docker/overlay2/f90df637b413292da87ac997ec07912778e7a84f6d8c0a091b9d37461f7b1ae7[node1] (local) root@192.168.0.18 /var/lib/docker/overlay2/f90df637b413292da87ac997ec07912778e7a84f6d8c0a091b9d37461f7b1ae7
$ ls -l
total 4
drwxr-xr-x   19 root     root           199 Dec 31 17:11 diff
-rw-r--r--    1 root     root            26 Dec 31 17:11 link
[node1] (local) root@192.168.0.18 /var/lib/docker/overlay2/f90df637b413292da87ac997ec07912778e7a84f6d8c0a091b9d37461f7b1ae7
$ cat link
YJHCGPQJLO6Q65JWH7XUBTI2OX[node1] (local) root@192.168.0.18 /var/lib/docker/overlay2/f90df637b413292da87ac997ec07912778e7a84f6d8c0a091b9d37461f7b1ae7
$ cd diff/
[node1] (local) root@192.168.0.18 /var/lib/docker/overlay2/f90df637b413292da87ac997ec07912778e7a84f6d8c0a091b9d37461f7b1ae7/diff
$ ls
bin    etc    lib    mnt    proc   run    srv    tmp    var
dev    home   media  opt    root   sbin   sys    usr
```

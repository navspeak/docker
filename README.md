# Syllabus:
- Orchestration (25%)
- Image Creation, Management, and Registry (20%)
- Installation and Configuration (15%)
- Networking (15%)
- Security (15%)
- Storage and Volumes (10%)

https://medium.com/bb-tutorials-and-thoughts/250-practice-questions-for-the-dca-exam-84f3b9e8f5ce
http://www.adelzaalouk.me/2017/docker-musings/
## using jq with Docker
- docker inspect <container_name> | jq .\[0\] | jq keys
- docker inspect <container_name> | jq .\[0\] | jq .DockerVersion
   - "1.12.3"
- (2375 for unencrypted dockerd and 2376 for encrypted)[https://docs.docker.com/engine/reference/commandline/dockerd/]
- 2377 - TCP - Cluster mgmt communication
- 4789 - UDP - overlay network traffic
- 7946 - TCP/UDP - communication between nodes

## Orchestration
- $ docker swarm init --advertise-addr 192.168.0.13 --listen-addr 192.168.0.13
```
Swarm initialized: current node (u9u6h80ju0l3obpn5fzung612) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-35unrlnskdez6708m0vplmqm4o8exakk6k6sz01d1upilxr428-3wgalezvqv4uy8lnmihepsn61 192.168.0.13:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
- docker swarm join-token manager => to get manager token. Worker token replace manager w/worker
- docker swarm join --token SWMTKN-1-CLUSTERID-MGRTOKEN IP:2377
- docker swarm join --token SWMTKN-1-CLUSTERID-MWRKTOKEN IP:2377
- On a node where swarm isn't initilised
  - $ docker network ls => bridge, host, none (null driver)
  - after it joins the swarm as manager or worker => docker_gwbridge (bridge) and ingress(overlay)
```
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
f38edc93a8a9        bridge              bridge              local
d193ef924940        docker_gwbridge     bridge              local
b4befbbd8887        host                host                local
jjrm003op8so        ingress             overlay             swarm
dec0198d34a8        none                null                local
```
- docker node ls => AVAILABILITY, MANAGER STATUS (LEADER)
- docker service create --replicas 3 --name xyz busybox sleep 5m
- docker service ls
```
$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
j93ejjiifs3b        xyz                 replicated          2/2                 busybox:latest    
```
- docker service inspect xyz --pretty
- docker service ps xyz
```
$ docker service ps xyz
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE                ERROR               PORTS
uosjx6wvudkp        xyz.1               busybox:latest      node2               Running             Running about a minute ago                       
lora1ydrcod2        xyz.2               busybox:latest      node1               Running             Running about a minute ago         
```
- on the nodes go and run "docker ps"
- docker service scale SERVICE1=N1 SERVICE2=N2 or docker service update 
- docker service update --image busybox:musl xyz
- docker service update --update-parallelism 2  xyz
- docker service rollback xyz
- https://medium.com/better-programming/rollout-and-rollback-in-docker-swarm-7f19e2fe2cd1
```
$ docker service create --name whoami \
--replicas 2 \
--update-failure-action rollback \  #continue or pause. Default scheduler pauses if 1 of the tasks returns failure
--update-delay 10s \
--update-monitor 10s \
--publish 8000:8000 \
lucj/whoami:1.0
```
- (placement constraints)[https://success.docker.com/article/using-contraints-and-labels-to-control-the-placement-of-containers]
- docker node update --label-add region=east node1
```
docker service create \
    --name nginx-east-no-devel \
    --constraint node.labels.region==east \
    --constraint node.labels.type!=devel \
    nginx
```
- (placement-prefs)[https://docs.docker.com/engine/swarm/services/#placement-constraints]
- docker service create --placement-pref 'spread=node.label.zone'
```
$ ls -l /var/lib/docker/swarm
total 8
drwxr-xr-x    2 root     root            75 Jan 13 13:25 certificates
-rw-------    1 root     root           216 Jan 13 13:25 docker-state.json
drwx------    4 root     root            55 Jan 13 13:25 raft
-rw-------    1 root     root            68 Jan 13 13:25 state.json
drwxr-xr-x    2 root     root            22 Jan 13 13:25 worker
```
- docker swarm update --autolock=true or docker swarm init --autolock => encrypts encryption key
- docker swarm unlock-key
```
 docker swarm unlock-key --rotate
Successfully rotated manager unlock key.

To unlock a swarm manager after it restarts, run the `docker swarm unlock`
command and provide the following key:

    SWMKEY-1-QGyrIy6yYq2F46QVuUL95XLXWjPfvBNZxRvAvTxG85E

Please remember to store this key in a password manager, since without it you
will not be able to restart the manager.
```
# Images:
## Dockerfile
https://docs.docker.com/develop/develop-images/dockerfile_best-practices/
```
docker build -t myimage:latest -<<EOF
FROM busybox
COPY somefile.txt .
RUN cat /somefile.txt
EOF

# build an image using the current directory as context, and a Dockerfile passed through stdin
docker build -t myimage:latest -f- . <<EOF
FROM busybox
COPY somefile.txt .
RUN cat /somefile.txt
EOF

```
- Multistage build 
- leveraging caching
- Only the instructions RUN, COPY, ADD create layers. Other instructions create temporary intermediate images, and do not increase the size of the build.
- Stop at a specific build stage : docker build --target builder -t alexellis2/href-counter:latest .
- https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#minimize-the-number-of-layers
- docker port to find port mapping. EXPOSE, -p and -P
- LABEL vs MAINTAINER (deprecated) -> docker inspect to view image labels.
- Links vs userdefined networks (https://stackoverflow.com/questions/41768157/how-to-link-container-in-docker)
```
$ export today=Wednesday
[node1] (local) root@192.168.0.23 ~
$ docker run -e "deep=purple" -e today --rm alpine env
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
e6b0cf9c0882: Pull complete 
Digest: sha256:2171658620155679240babee0a7714f6509fae66898db422ad803b951257db78
Status: Downloaded newer image for alpine:latest
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=8179e75053af
deep=purple
today=Wednesday
HOME=/root
```
- https://docs.docker.com/engine/reference/run/#healthcheck --format '{{json .State.Health.Status}}' with docker inspect or also docker ps
- docker attach vs docker exec
# Storage and Volumes
- --volume => originally for standalone containers --mount for services in swarm mode. Now you can use --mount for both
- https://docs.docker.com/storage/bind-mounts/#start-a-container-with-a-bind-mount
- -v \<pathOnHost\>:\<onCont\>:<ro[, consistent, delegated, cached, z]>
- --mount type: bind, volume, tmfs | source
- If you use -v or --volume to bind-mount a file or directory that does not yet exist on the Docker host, -v creates the endpoint for you. It is always created as a directory.
- If you use --mount to bind-mount a file or directory that does not yet exist on the Docker host, Docker does not automatically create it for you, but generates an error.
- If you bind-mount into a non-empty directory on the container, the directory’s existing contents are obscured by the bind mount
- What happens when you do this?
```
   $ docker run -d -it  --name broken-container 
  -v /tmp:/usr  nginx:latest
   or
   $ docker run -d -it --name broken-container --mount type=bind,source=/tmp,target=/usr  nginx:latest
docker: Error response from daemon: oci runtime error: container_linux.go:262:
starting container process caused "exec: \"nginx\": executable file not found in $PATH".
```
- https://docs.docker.com/storage/volumes/
- --mount type=bind,source="$(pwd)"/target,target=/app,readonly 
- -v "$(pwd)"/target:/app:ro
- Bind propagation defaults to rprivate for bind mounts. 
- Volumes use rprivate bind propagation, and bind propagation is not configurable for volumes
- Docker Desktop for Mac uses osxfs to propagate directories and files shared from macOS to the Linux VM. 
- bind vs volume vs tmps
- Backup a container's volume using volume-from:
```
$ docker run --rm --volumes-from dbstore -v $(pwd):/backup ubuntu tar cvf /backup/backup.tar /dbdata
$ docker run -v /dbdata --name dbstore2 ubuntu /bin/bash
$ docker run --rm --volumes-from dbstore2 -v $(pwd):/backup ubuntu bash -c "cd /dbdata && tar xvf /backup/backup.tar --strip 1"
```
- -rm removes anonymous volumes not names ones
- docker volume prune => to remove all unused volumes
- --tmfs (standalone container) vs --mount
- docker ps -s
   - size = writeable layer | virtual size = writeable layer + RO image (overestimates)
   - total size of 'n' containers using same image = nSize + (virtual size - size)
- Container space constituents : 
   - Disk space used for log files if you use the json-file logging driver. 
   - Volumes and bind mounts used by the container.
   - Disk space used for the container’s configuration files, which are typically small.
   - Memory written to disk (if swapping is enabled).
   - Checkpoints, if you’re using the experimental checkpoint/restore feature.
- overlay2, aufs, and overlay => file level rather than the block level. This uses memory more efficiently, but the container’s writable layer may grow quite large in write-heavy workloads.
- devicemapper, btrfs, and zfs => Block-level storage drivers. perform better for write-heavy workloads (though not as well as Docker volumes).
- For lots of small writes or containers with many layers or deep filesystems, overlay may perform better than overlay2, but consumes more inodes, which can lead to inode exhaustion.
- btrfs and zfs require a lot of memory.
- zfs is a good choice for high-density workloads such as PaaS.
- Relationship between dockerfile and docker-compose.yml : https://www.techrepublic.com/article/what-is-the-difference-between-dockerfile-and-docker-compose-yml-files/

## DOCKERFILE
- https://docs.docker.com/engine/reference/builder
- Releases : Stable & experimental
- Only ARG & directives can be before FROM
```
ARG  CODE_VERSION=latest
FROM base:${CODE_VERSION}
CMD  /code/run-app

FROM extras:${CODE_VERSION}
CMD  /code/run-extras
```
- https://docs.docker.com/engine/reference/builder/#automatic-platform-args-in-the-global-scope
- Environment variables defined using the ENV instruction always override an ARG instruction of the same name.
- https://docs.docker.com/engine/reference/builder/#predefined-args 
- Predefined args are not cached unless specified in the dockerfile using ARG. Read about it.
- RUN has two formats (RUN <cmd> => shell form and RUN ["executable", "param1", "param2"] (exec form))
```
  RUN /bin/bash -c 'source $HOME/.bashrc; echo $HOME'
  RUN ["/bin/bash", "-c", "echo hello"]
```
- COPY vs ADD (copy + URL or even extract tar file into destination)
- use curl or wget to download .tar from remote location instead of ADD. Use ADD for uncompressing local file only
- Use COPY over ADD when possible
- HEALTHCHECK --interval 5s CMD ping google.com (default --timeout and --interval 30s, --start-period 0, --retries 3) => docker ps and also docker container inspect
 - The main purpose of a CMD is to provide defaults for an executing container. These defaults can include an executable, or they can omit the executable, in which case you must specify an ENTRYPOINT instruction as well. - CMD - JSON Array w/o shell
- CMD ["executable","param1","param2"] (exec form, this is the preferred form)
- CMD ["param1","param2"] (as default parameters to ENTRYPOINT)
- CMD command param1 param2 (shell form) https://docs.docker.com/engine/reference/builder/#entrypoint
- CMD = Overwrite, ENTRYPOINT = doesn't override (except --entrypoint) but appends
- shell form doesn't catch signal from docker stop
- like if CMD ["sh"] and during run if u say "ping -c 10 google.com", it will run ping 10 times and exit
```
   FROM busybox
   ENTRYPOINT ["/bin/ping"]
   
   $ docker build . -t myimage
   $ docker run -dt --name c1 myimage sh => /bin/ping sh (appended)
   
```
- https://www.ctl.io/developers/blog/post/dockerfile-entrypoint-vs-cmd/
- https://www.ctl.io/developers/blog/post/dockerfile-add-vs-copy/
- COPY vs ADD (copy + URL or even extract tar file into destination)
- use curl or wget to download .tar from remote location instead of ADD. Use ADD for uncompressing local file only
- Use COPY over ADD when possible
- HEALTHCHECK --interval 5s CMD ping google.com (default --timeout and --interval 30s, --start-period 0, --retries 3) => docker ps and also docker container inspect 
- docker tag | 
- docker search nginix --limit 5
- docker search --filter "is-official=true" --filter "stars=3" busybox
- docker search --format "{{.Name}}: {{.StarCount}}" nginx
- docker search --format "table {{.Name}}\t{{.IsAutomated}}\t{{.IsOfficial}}" nginx
- docker container commit container1 modified-image-based-on-container1 (while container is being committed, its process is  paused)
## Flattening a docker container

- docker container run --name myubuntu ubuntu
- docker export myubuntu > myubuntu.tar
- cat myubuntu.tar | docker import - myubuntuimage:latest => one layer - less size
- export & import => containers 
- save and load => images

## Docker Networking
- https://docs.docker.com/network/overlay/#encrypt-traffic-on-an-overlay-network
- [Physically hosted application](https://www.youtube.com/watch?v=PpyPa92r44s&t=1820s) [Virtual Applications] (https://www.youtube.com/watch?v=PpyPa92r44s&t=1820s)


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

## HRM
https://success.docker.com/article/ucp-service-discovery-swarm

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
- docker ps -s => size (writeable layer) & Virtual Size (writeable layer + Readonly layer)
- https://docs.docker.com/storage/storagedriver/#container-size-on-disk

## Overlay2 and Overlay
http://100daysofdevops.com/21-days-of-docker-day-13-docker-storage-part-2/2/
<br>https://docs.docker.com/storage/storagedriver/overlayfs-driver/
<br> Read upperDir, lowerDir, copy_up, whiteout file

-  OverlayFS supports page cache sharing. Multiple containers accessing the same file share a single page cache entry for that file. This makes the overlay and overlay2 drivers efficient with memory and a good option for high-density use cases such as PaaS.
- https://docs.docker.com/storage/storagedriver/overlayfs-driver/#overlayfs-and-docker-performance
- The OverlayFS copy_up operation is faster than the same operation with AUFS, because AUFS supports more layers than OverlayFS and it is possible to incur far larger latencies if searching through many AUFS layers. overlay2 supports multiple layers as well, but mitigates any performance hit with caching.
- touch workaround for yum and open https://docs.docker.com/storage/storagedriver/overlayfs-driver/#limitations-on-overlayfs-compatibility
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

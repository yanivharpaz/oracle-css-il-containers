
# CSS-IL RAC Workshop

### Based on the Oracle RAC container documentation

* https://github.com/oracle/docker-images/tree/main/OracleDatabase/RAC/OracleDNSServer

Provision a Linux server (Oracle Linux 7.9) with 2 CPUs and 16GB of RAM, 256GB of disk space


### install docker
```

sudo yum update -y ; sudo yum update -y 
sudo yum install -y docker-engine
sudo service docker start
sudo systemctl enable docker

sudo docker run hello-world

sudo groupadd docker
sudo usermod -aG docker $USER

```

### clone the Oracle docker-images repository

```
sudo yum install -y nfs-utils git wget curl jq

mkdir ~/dev
cd ~/dev

git clone https://github.com/oracle/docker-images.git

```

### it is recommended to provision a Windows VM for the the download and copy of the Oracle RDBMS binaries (with WinSCP)

### Download and copy the Oracle RDBMS binaries from:   
https://www.oracle.com/technetwork/database/enterprise-edition/downloads/index.html

* download the 21c binaries
* Oracle Database 21c Grid Infrastructure (21.3) for Linux x86-64

#### Oracle Database 21c (21.3) for Linux x86-64


### Prepare linux server
## disable SELINUX
```
sudo yum update -y 
sudo yum -y install nfs-utils

sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
sudo yum install -y epel-release 
# apt-transport-https conntrack 
sudo yum install -y git mc ncdu zsh htop vim gcc wget jq


# sudo yum -y groupinstall "Development Tools"
sudo yum install -y openssl-devel 
# bzip2-devel libffi-devel xz-devel

sudo yum install -y bind-utils mlocated yum-utils createrepo bin-utils openssh-clients perl parted

cat /etc/selinux/config
sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
sudo sed -i 's/SELINUX=permissive/SELINUX=disabled/' /etc/selinux/config
cat /etc/selinux/config

sestatus

# reboot if sestatus is not disabled

```


### DNS Server
```

sudo service docker restart

cd ~/dev/docker-images/OracleDatabase/RAC/OracleDNSServer/dockerfiles/
./buildContainerImage.sh latest

docker images

docker network create --driver=bridge --subnet=172.16.1.0/24 rac_pub1_nw

docker run -d  --name racdns \
 --hostname rac-dns  \
 --dns-search="example.com" \
 --cap-add=SYS_ADMIN  \
 --network  rac_pub1_nw \
 --ip 172.16.1.25 \
 --sysctl net.ipv6.conf.all.disable_ipv6=1 \
 --env SETUP_DNS_CONFIG_FILES="setup_true" \
 --env DOMAIN_NAME="example.com" \
 --env RAC_NODE_NAME_PREFIX="racnode" \
 oracle/rac-dnsserver:latest

 docker logs -f racdns

```
### Storage -> NFS (make sure SELinux is disabled through "sestatus" command)  
```
cd ~/dev/docker-images/OracleDatabase/RAC/OracleRACStorageServer/dockerfiles/
./buildDockerImage.sh -v 19.3.0

docker images

docker network create --driver=bridge --subnet=192.168.17.0/24 rac_priv1_nw

sudo yum -y install nfs-utils

export ORACLE_DBNAME=ORCLCDB
docker run -d -t --hostname racnode-storage \
--dns-search=example.com  --cap-add SYS_ADMIN --cap-add AUDIT_WRITE \
--volume /docker_volumes/asm_vol/$ORACLE_DBNAME:/oradata --init \
--network=rac_priv1_nw --ip=192.168.17.25 --tmpfs=/run  \
--volume /sys/fs/cgroup:/sys/fs/cgroup:ro \
--name racnode-storage oracle/rac-storage-server:19.3.0

docker logs -f racnode-storage

```

### Create a docker volume  

```

docker volume create --driver local \
--opt type=nfs \
--opt   o=addr=192.168.17.25,rw,bg,hard,tcp,vers=3,timeo=600,rsize=32768,wsize=32768,actimeo=0 \
--opt device=192.168.17.25:/oradata \
racstorage

```

### update sysctl configuration
```
# sudo vim /etc/sysctl.conf

sudo su
cat >> /etc/sysctl.conf << EOF
fs.aio-max-nr = 1048576
fs.file-max = 6815744
net.core.rmem_max = 4194304
net.core.rmem_default = 262144
net.core.wmem_max = 1048576
net.core.wmem_default = 262144
net.core.rmem_default = 262144
EOF

sudo sysctl -a
sudo sysctl -p  

exit

``` 

### Make sure you copied the Oracle RDBMS binaries to the linux server  
download from here:  
https://www.oracle.com/technetwork/database/enterprise-edition/downloads/index.html


```
[opc@rac-test-1]$ pwd
/home/opc/dev/docker-images/OracleDatabase/RAC/OracleRealApplicationClusters/dockerfiles/21.3.0

[opc@rac-test-1]$ ls -lh *.zip
-rw-r--r-- 1 opc opc 2.9G Nov  8 18:03 LINUX.X64_213000_db_home.zip
-rw-r--r-- 1 opc opc 2.3G Nov  9 14:39 LINUX.X64_213000_grid_home.zip

```

### Download the software from the storage container

```
cd ~/dev/docker-images/OracleDatabase/RAC/OracleRealApplicationClusters/dockerfiles/21.3.0
wget https://stgvscodepub.blob.core.windows.net/yhpub/LINUX.X64_213000_db_home.zip
wget https://stgvscodepub.blob.core.windows.net/yhpub/LINUX.X64_213000_grid_home.zip
cd ..

```


### Build the Oracle RDBMS binaries image
```
# deal with the error: [FATAL] [INS-40937] The following hostnames are invalid:buildkitsandbox

cd ~/dev/docker-images/OracleDatabase/RAC/OracleRealApplicationClusters/dockerfiles

export DOCKER_BUILDKIT=0

time ./buildContainerImage.sh -v 21.3.0 -o '--build-arg  BASE_OL_IMAGE=oraclelinux:7 --build-arg SLIMMING=true|false'

```  


## network
```
docker network create --driver=bridge --subnet=172.16.1.0/24 rac_pub1_nw
docker network create --driver=bridge --subnet=192.168.17.0/24 rac_priv1_nw

```

## secrets
```
sudo su
mkdir /opt/.secrets/
openssl rand -out /opt/.secrets/pwd.key -hex 64
exit

```

### set a common password
```
sudo su

cat > /opt/.secrets/common_os_pwdfile << EOF
P@ssw0rd123#@
EOF

openssl enc -aes-256-cbc -salt -in /opt/.secrets/common_os_pwdfile -out /opt/.secrets/common_os_pwdfile.enc -pass file:/opt/.secrets/pwd.key
rm -f /opt/.secrets/common_os_pwdfile

export COMMON_OS_PWD_FILE="P@ssw0rd123#@"

exit
export COMMON_OS_PWD_FILE="P@ssw0rd123#@"


```

### create a RAC hosts file
```
sudo su
mkdir /opt/containers/
touch /opt/containers/rac_host_file
chmod 777 /opt/containers/rac_host_file
exit

```

### create the first node container

```
docker create -t -i \
  --hostname racnode1 \
  --volume /boot:/boot:ro \
  --volume /dev/shm \
  --tmpfs /dev/shm:rw,exec,size=4G \
  --volume /opt/containers/rac_host_file:/etc/hosts  \
  --volume /opt/.secrets:/run/secrets:ro \
  --dns=172.16.1.25 \
  --dns-search=example.com \
  --privileged=false \
  --volume racstorage:/oradata \
  --cap-add=SYS_NICE \
  --cap-add=SYS_RESOURCE \
  --cap-add=NET_ADMIN \
  -e DNS_SERVERS="172.16.1.25" \
  -e NODE_VIP=172.16.1.160  \
  -e VIP_HOSTNAME=racnode1-vip  \
  -e PRIV_IP=192.168.17.150  \
  -e PRIV_HOSTNAME=racnode1-priv \
  -e PUBLIC_IP=172.16.1.150 \
  -e PUBLIC_HOSTNAME=racnode1  \
  -e SCAN_NAME=racnode-scan \
  -e OP_TYPE=INSTALL \
  -e DOMAIN=example.com \
  -e ASM_DISCOVERY_DIR=/oradata \
  -e ASM_DEVICE_LIST=/oradata/asm_disk01.img,/oradata/asm_disk02.img,/oradata/asm_disk03.img,/oradata/asm_disk04.img,/oradata/asm_disk05.img  \
  -e CMAN_HOSTNAME=racnode-cman1 \
  -e CMAN_IP=172.16.1.15 \
  -e COMMON_OS_PWD_FILE=common_os_pwdfile.enc \
  -e PWD_KEY=pwd.key \
  --restart=always \
  --tmpfs=/run -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
  --ulimit rtprio=99  \
  --name racnode1 \
  oracle/database-rac:21.3.0

```

### prepare the network for the first node

```
docker network disconnect bridge racnode1
docker network connect rac_pub1_nw --ip 172.16.1.150 racnode1
docker network connect rac_priv1_nw --ip 192.168.17.150  racnode1

```

### start the first node
```
docker start racnode1

```

```
docker start racnode1
docker exec -it racnode1 bash

cat >>  /tmp/resetFailedunits.sh << EOL
 #!/bin/bash
          failed_unit_file_name=reset_failed_units.txt
          current_time=$(date "+Y.%m.%d-%H.%M.%S")
          passed_unit_file_name=passed_units.txtnew_failed_unit_file_name=$file_name.$current_time
          systemctl_state=$(systemctl status | awk '/State:/{ print $0 }' | grep -v 'awk /State:/' | awk '{ print $2 }')
          if [ "${systemctl_state}" != "running" ]; then
            systemctl reset-failed
            touch /tmp/$new_failed_unit_file_name
          else
            touch /tmp/$passed_unit_file_name
          fi
EOL

chmod a+x /tmp/resetFailedunits.sh
/tmp/resetFailedunits.sh
echo "* * * * * sleep 00; /tmp/resetFailedunits.sh" >> mycron
echo "* * * * * sleep 15; /tmp/resetFailedunits.sh" >> mycron
echo "* * * * * sleep 30; /tmp/resetFailedunits.sh" >> mycron
echo "* * * * * sleep 45; /tmp/resetFailedunits.sh" >> mycron
crontab mycron
crontab -l
sleep 5
systemctl stop rhnsd
systemctl start rhnsd
systemctl status

tail -f /tmp/orod.log

```

Thank you for reading.  
  

You can find me on https://linktr.ee/yanivharpaz

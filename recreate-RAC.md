# Recreate RAC env

### remove docker containers

```

docker stop racnode2
docker stop racnode1
docker stop racnode-cman
docker stop racnode-storage
docker stop racdns

docker rm -f racnode2 racnode1 racnode-storage racnode-cman racdns
sudo rm -fr /docker_volumes/asm_vol

sudo rm -fr /opt/containers/rac_host_file
sudo rm -fr /opt/.secrets

docker network ls
docker network rm rac_pub1_nw
docker network rm rac_priv1_nw

docker volume ls
docker volume rm racstorage

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

### storage NFS

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

### docker volume

```

docker volume create --driver local \
--opt type=nfs \
--opt   o=addr=192.168.17.25,rw,bg,hard,tcp,vers=3,timeo=600,rsize=32768,wsize=32768,actimeo=0 \
--opt device=192.168.17.25:/oradata \
racstorage


docker volume create rman_backup_volume

```

### connection manager
```

docker network create --driver=bridge --subnet=172.16.1.0/24 rac_pub1_nw

/usr/bin/docker run -d --hostname racnode-cman1 --dns-search=example.com \
--network=rac_pub1_nw --ip=172.16.1.15 \
-e DOMAIN=example.com -e PUBLIC_IP=172.16.1.15 \
-e PUBLIC_HOSTNAME=racnode-cman1 -e SCAN_NAME=racnode-scan \
-e SCAN_IP=172.16.1.70 --privileged=false \
-p 1523:1521 --name racnode-cman oracle/client-cman:21.3.0

docker logs -f racnode-cman

```


### networks
```

docker network create --driver=bridge --subnet=172.16.1.0/24 rac_pub1_nw
docker network create --driver=bridge --subnet=192.168.17.0/24 rac_priv1_nw


```

### secrets
```

sudo su
mkdir /opt/.secrets/
openssl rand -out /opt/.secrets/pwd.key -hex 64

cat > /opt/.secrets/common_os_pwdfile << EOF
Passw0rd123#a
EOF

openssl enc -aes-256-cbc -salt -in /opt/.secrets/common_os_pwdfile -out /opt/.secrets/common_os_pwdfile.enc -pass file:/opt/.secrets/pwd.key
rm -f /opt/.secrets/common_os_pwdfile

# for root
export COMMON_OS_PWD_FILE="Passw0rd123#a"

exit
# for the basic user
export COMMON_OS_PWD_FILE="Passw0rd123#a"

```


### RAC host file 
```

sudo su
mkdir /opt/containers/
touch /opt/containers/rac_host_file
chmod 777 /opt/containers/rac_host_file
exit

```

### create docker network

```

docker network create --driver=bridge --subnet=172.16.1.0/24 rac_pub1_nw
docker network create --driver=bridge --subnet=192.168.17.0/24 rac_priv1_nw

```

### create node 1
####  --volume rman_backup_volume:/u01/backup:rw \
#### 
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
  --volume rman_backup_volume:/u01/backup:rw \
  --cap-add=SYS_NICE \
  --cap-add=SYS_RESOURCE \
  --cap-add=NET_ADMIN \
  --publish 1521:1521 \
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
  --cpu-rt-runtime=95000 --ulimit rtprio=99  \
  --ulimit rtprio=99  \
  --name racnode1 \
  oracle/database-rac:21.3.0

docker network disconnect bridge racnode1
docker network connect rac_pub1_nw --ip 172.16.1.150 racnode1
docker network connect rac_priv1_nw --ip 192.168.17.150  racnode1

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

sleep 2

tail -f /tmp/orod.log

```

### node 2
```

docker create -t -i \
  --hostname racnode2 \
  --volume /dev/shm \
  --tmpfs /dev/shm:rw,exec,size=4G  \
  --volume /boot:/boot:ro \
  --dns-search=example.com  \
  --volume /opt/containers/rac_host_file:/etc/hosts \
  --volume /opt/.secrets:/run/secrets:ro \
  --dns=172.16.1.25 \
  --dns-search=example.com \
  --privileged=false \
  --volume racstorage:/oradata \
  --volume rman_backup_volume:/u01/backup:rw \
  --cap-add=SYS_NICE \
  --cap-add=SYS_RESOURCE \
  --cap-add=NET_ADMIN \
  --publish 1522:1521 \
  -e DNS_SERVERS="172.16.1.25" \
  -e EXISTING_CLS_NODES=racnode1 \
  -e NODE_VIP=172.16.1.161  \
  -e VIP_HOSTNAME=racnode2-vip  \
  -e PRIV_IP=192.168.17.151  \
  -e PRIV_HOSTNAME=racnode2-priv \
  -e PUBLIC_IP=172.16.1.151  \
  -e PUBLIC_HOSTNAME=racnode2  \
  -e DOMAIN=example.com \
  -e SCAN_NAME=racnode-scan \
  -e ASM_DISCOVERY_DIR=/oradata \
  -e ASM_DEVICE_LIST=/oradata/asm_disk01.img,/oradata/asm_disk02.img,/oradata/asm_disk03.img,/oradata/asm_disk04.img,/oradata/asm_disk05.img \
  -e ORACLE_SID=ORCLCDB \
  -e OP_TYPE=ADDNODE \
  -e COMMON_OS_PWD_FILE=common_os_pwdfile.enc \
  -e PWD_KEY=pwd.key \
  --restart=always \
  --tmpfs=/run -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
  --cpu-rt-runtime=95000 --ulimit rtprio=99  \
  --ulimit rtprio=99  \
  --name racnode2 \
  oracle/database-rac:21.3.0


docker network disconnect bridge racnode2
docker network connect rac_pub1_nw --ip 172.16.1.151 racnode2
docker network connect rac_priv1_nw --ip 192.168.17.151 racnode2

docker start racnode2
docker exec -it racnode2 bash

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

sleep 2

tail -f /tmp/orod.log

```




# Start RAC commands

This document contains the commands to start the RAC.
---

## DNS
```
docker start racdns
sleep 2
docker logs racdns

```

## Storage
```
docker start racnode-storage
sleep 5
docker logs racnode-storage

```

## Connection manager
```
docker start racnode-cman
sleep 15
docker logs racnode-cman

```

## Node 1 network
```
docker network disconnect bridge racnode1
docker network connect rac_pub1_nw --ip 172.16.1.150 racnode1
docker network connect rac_priv1_nw --ip 192.168.17.150  racnode1

```

## Node 1
```
docker start racnode1
docker exec -it racnode1 bash

sleep 5
systemctl stop rhnsd
systemctl start rhnsd
systemctl status

sleep 2

tail -f /tmp/orod.log

```

## Check RAC Status // Cluster and databases
```
docker exec -it racnode1 bash

su - grid
crsctl check cluster -all
exit

su - oracle
export ORACLE_SID=ORCLCDB1
export ORACLE_HOME=/u01/app/oracle/product/21.3.0/dbhome_1
srvctl status database -d ORCLCDB

exit
exit

```

## Node 2 network
```
docker network disconnect bridge racnode2
docker network connect rac_pub1_nw --ip 172.16.1.151 racnode2
docker network connect rac_priv1_nw --ip 192.168.17.151 racnode2

```

## Node 2
```
docker start racnode2
docker exec -it racnode2 bash

sleep 5
systemctl stop rhnsd
systemctl start rhnsd
systemctl status

sleep 2

tail -f /tmp/orod.log

```

## Open database
```

docker exec -it racnode1 bash

su - oracle
export ORACLE_SID=ORCLCDB1
export ORACLE_HOME=/u01/app/oracle/product/21.3.0/dbhome_1
srvctl status database -d ORCLCDB

sqlplus / as sysdba
show pdbs

alter pluggable database all open;

show pdbs

exit
exit
exit

```

## Connect from outside the container
```
sql USERNAME/PASSWORD@localhost:1523/orclpdb

```


## Stop RAC
```
docker stop racnode2
docker stop racnode1
docker stop racnode-cman
docker stop racnode-storage
docker stop racdns

docker ps

```

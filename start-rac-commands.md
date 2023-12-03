
# Start RAC commands  

* This document contains the commands to start the RAC.  
* The VM host has 5 containers: DNS, Storage, Connection manager, Node 1 and Node 2.
* The containers are started in the following order: DNS, Storage, Connection manager, Node 1 and Node 2.
* The containers are started with a delay between them to allow the previous container to start.
  
* The commands are executed in the VM host.


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

#### Example of valid output (should take up to 10 minutes to get this output)
```
[root@racnode1 rac-work-dir]# su - grid
Last login: Sun Dec  3 14:10:29 UTC 2023 on pts/2
[grid@racnode1 ~]$ crsctl check cluster -all
**************************************************************
racnode1:
CRS-4537: Cluster Ready Services is online
CRS-4529: Cluster Synchronization Services is online
CRS-4533: Event Manager is online
**************************************************************
racnode2:
CRS-4537: Cluster Ready Services is online
CRS-4529: Cluster Synchronization Services is online
CRS-4533: Event Manager is online
**************************************************************
[grid@racnode1 ~]$ exit
logout
[root@racnode1 rac-work-dir]#
[root@racnode1 rac-work-dir]# su - oracle
Last login: Sun Dec  3 14:10:29 UTC 2023 on pts/2
[oracle@racnode1 ~]$ export ORACLE_SID=ORCLCDB1
[oracle@racnode1 ~]$ export ORACLE_HOME=/u01/app/oracle/product/21.3.0/dbhome_1
[oracle@racnode1 ~]$ srvctl status database -d ORCLCDB
Instance ORCLCDB1 is running on node racnode1
Instance ORCLCDB2 is running on node racnode2
[oracle@racnode1 ~]$
[oracle@racnode1 ~]$ exit
logout
[root@racnode1 rac-work-dir]# exit
exit
[opc@rac-test-1 ~]$

```


## Open database
#### only after you validate that both nodes are up and running
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

#### Example of valid output
```
[root@racnode1 rac-work-dir]# su - oracle
Last login: Sun Dec  3 14:10:38 UTC 2023 on pts/2
[oracle@racnode1 ~]$ export ORACLE_SID=ORCLCDB1
[oracle@racnode1 ~]$ export ORACLE_HOME=/u01/app/oracle/product/21.3.0/dbhome_1
[oracle@racnode1 ~]$ srvctl status database -d ORCLCDB
Instance ORCLCDB1 is running on node racnode1
Instance ORCLCDB2 is running on node racnode2
[oracle@racnode1 ~]$
[oracle@racnode1 ~]$ sqlplus / as sysdba

SQL*Plus: Release 21.0.0.0.0 - Production on Sun Dec 3 14:12:53 2023
Version 21.3.0.0.0

Copyright (c) 1982, 2021, Oracle.  All rights reserved.


Connected to:
Oracle Database 21c Enterprise Edition Release 21.0.0.0.0 - Production
Version 21.3.0.0.0

SQL>
    CON_ID CON_NAME			  OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
	 2 PDB$SEED			  READ ONLY  NO
	 3 ORCLPDB			  MOUNTED
SQL> SQL>
Pluggable database altered.

SQL> SQL>
    CON_ID CON_NAME			  OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
	 2 PDB$SEED			  READ ONLY  NO
	 3 ORCLPDB			  READ WRITE NO
SQL> SQL> Disconnected from Oracle Database 21c Enterprise Edition Release 21.0.0.0.0 - Production
Version 21.3.0.0.0

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

---
Written by Yaniv Harpaz, 2023  
https://linktr.ee/yanivharpaz

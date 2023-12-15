# Backup and restore RAC
---


### Backup

```

#------------------------
# backup on node 1
#------------------------

docker exec -it racnode1 bash
su - oracle
export ORACLE_SID=ORCLCDB1
export ORACLE_HOME=/u01/app/oracle/product/21.3.0/dbhome_1
srvctl status database -d ORCLCDB

rman target /
# for pluggable database backup do this:
# backup pluggable database orclpdb;
backup database;

exit

```

### Restore on node 1  

Currently, the restore is done into /home/oracle/backups inside node 1 container. This is a temporary location. The restore will be done into the same location as the backup.



```
#------------------------
# restore
#------------------------
# shutdown databases on both 1 + 2
# shutdown node 2
docker exec -it racnode2 bash
su - oracle
export ORACLE_SID=ORCLCDB2
export ORACLE_HOME=/u01/app/oracle/product/21.3.0/dbhome_1
srvctl status database -d ORCLCDB

sqlplus / as sysdba
shu immediate

exit
exit

# shutdown node 1 (and have it in mount mode)

docker exec -it racnode1 bash
su - oracle
export ORACLE_SID=ORCLCDB1
export ORACLE_HOME=/u01/app/oracle/product/21.3.0/dbhome_1
srvctl status database -d ORCLCDB

sqlplus / as sysdba
shu immediate
startup mount

exit

# begin restore on node 1
rman target /
# for pluggable database restore do this:
# restore pluggable database orclpdb;
restore database;
recover database;

exit

sqlplus / as sysdba
alter database open;

show pdbs

alter session set container=orclpdb;
recover database;
alter database open;

exit
exit
exit

# start node 2 
docker exec -it racnode2 bash
su - oracle
export ORACLE_SID=ORCLCDB2
export ORACLE_HOME=/u01/app/oracle/product/21.3.0/dbhome_1
srvctl status database -d ORCLCDB

sqlplus / as sysdba
startup
alter pluggable database all open;

show pdbs

exit
exit
exit 


```



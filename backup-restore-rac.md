# Backup and restore RAC


```

#------------------------
# backup from node 1
#------------------------

docker exec -it racnode1 bash
su - oracle
export ORACLE_SID=ORCLCDB1
export ORACLE_HOME=/u01/app/oracle/product/21.3.0/dbhome_1
srvctl status database -d ORCLCDB

rman target /
backup pluggable database orclpdb;

exit

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
restore pluggable database orclpdb;
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



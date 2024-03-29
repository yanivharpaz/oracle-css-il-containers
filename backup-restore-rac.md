# Backup and restore RAC
---

### initial setup
```

mkdir /oradata/rman_backup

rman target /

backup database;

backup database plus archivelog;

```
<!--
RUN {
    ALLOCATE CHANNEL ch1 TYPE DISK FORMAT '/oradata/rman_backup/backup_ch1_%U';

    BACKUP DATABASE;
    validate database include current controlfile;

}
 

RUN {
    ALLOCATE CHANNEL c1 DEVICE TYPE DISK 
    CONNECT 'c##bck/Password1@(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=racnode1)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=ORCLCDB)))';

    ALLOCATE CHANNEL c2 DEVICE TYPE DISK 
    CONNECT 'c##bck/Password1@(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=racnode2)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=ORCLCDB)))';

    ALLOCATE CHANNEL ch1 TYPE DISK FORMAT '/oradata/rman_backup/backup_ch1_%U';

    BACKUP DATABASE PLUS ARCHIVELOG;

    validate database include current controlfile;

}


RUN {
    ALLOCATE CHANNEL c1 DEVICE TYPE DISK 
    CONNECT 'c##bck/Password1@(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=racnode1)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=ORCLCDB)))';

    ALLOCATE CHANNEL c2 DEVICE TYPE DISK 
    CONNECT 'c##bck/Password1@(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=racnode2)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=ORCLCDB)))';

    crosscheck archivelog all;

}

-->


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

backup database plus archivelog;

exit 

```
<!--
RUN {
    ALLOCATE CHANNEL c1 DEVICE TYPE DISK 
    CONNECT 'c##bck/Password1@(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=racnode1)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=ORCLCDB)))';

    ALLOCATE CHANNEL c2 DEVICE TYPE DISK 
    CONNECT 'c##bck/Password1@(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=racnode2)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=ORCLCDB)))';

    crosscheck archivelog all;
    backup database plus archivelog;
    validate database include current controlfile;

}
-->

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

### **************************************************************
### now it's the time to open the wallet 
### administer key management set keystore open identified by "XXXXXXX";
### **************************************************************

exit

# begin restore on node 1
rman target /
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

### Example of restore to a point in time

```

rman target /

shutdown immediate;
startup mount;


run {

    set until time "to_date('2024-02-16:07:46:00', 'YYYY-MM-DD:HH24:MI:SS')";
    restore database;
    recover database;
}


alter database open resetlogs;

```

<!--
    ALLOCATE CHANNEL c1 DEVICE TYPE DISK 
    CONNECT 'c##bck/Password1@(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=racnode1)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=ORCLCDB)))';

    ALLOCATE CHANNEL c2 DEVICE TYPE DISK 
    CONNECT 'c##bck/Password1@(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=racnode2)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=ORCLCDB)))';

-->

### backup after resetlogs  
please make sure you perform a new backup after the restore, because the old backups after resetlogs are unusable

```
rman target /


RUN {

    crosscheck archivelog all;
    backup database plus archivelog;
    validate database include current controlfile;

}


exit

```
<!--
    ALLOCATE CHANNEL c1 DEVICE TYPE DISK 
    CONNECT 'c##bck/Password1@(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=racnode1)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=ORCLCDB)))';

    ALLOCATE CHANNEL c2 DEVICE TYPE DISK 
    CONNECT 'c##bck/Password1@(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=racnode2)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=ORCLCDB)))';

    -->
    
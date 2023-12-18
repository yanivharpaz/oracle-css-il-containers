# Backup and restore RAC
---

### initial setup
```
rman target /

CONFIGURE DEVICE TYPE DISK PARALLELISM 3;


RUN {
    ALLOCATE CHANNEL ch1 TYPE DISK FORMAT '/home/oracle/backup/chn1/backup_ch1_%U';
    ALLOCATE CHANNEL ch2 TYPE DISK FORMAT '/home/oracle/backup/chn2/backup_ch2_%U';
    ALLOCATE CHANNEL ch3 TYPE DISK FORMAT '/home/oracle/backup/chn3/backup_ch3_%U';

    BACKUP DATABASE;
}


RUN {
    ALLOCATE CHANNEL ch1 TYPE DISK FORMAT '/home/oracle/backup/chn1/backup_ch1_%U';
    ALLOCATE CHANNEL ch2 TYPE DISK FORMAT '/home/oracle/backup/chn2/backup_ch2_%U';
    ALLOCATE CHANNEL ch3 TYPE DISK FORMAT '/home/oracle/backup/chn3/backup_ch3_%U';

    BACKUP DATABASE PLUS ARCHIVELOG;
}

```

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
CONFIGURE DEVICE TYPE DISK PARALLELISM 3;

# for pluggable database backup do this:
backup pluggable database orclpdb;
backup database plus archivelog;

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

### **************************************************************
### now it's the time to open the wallet 
### administer key management set keystore open identified by "XXXXXXX";
### **************************************************************

exit

# begin restore on node 1
rman target /
# for pluggable database restore do this:
# restore pluggable database orclpdb;
restore database;
recover database;
# restore pluggable database orclpdb;
# restore pluggable database orclpdb;

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
    set until time "to_date('2023-12-18:12:02:00', 'YYYY-MM-DD:HH24:MI:SS')";
    restore database;
    recover database;
}


alter database open resetlogs;

```


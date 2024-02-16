# Create table & data in the pluggable database

```

alter pluggable database all open;
alter pluggable database orclpdb save state;

alter pluggable database all open;
alter pluggable database all save state;



alter session set container=orclpdb;
select tablespace_name,file_name from dba_data_files;

drop tablespace my_data_tbs including contents and datafiles;
create tablespace my_data_tbs datafile '+DATA/ORCLCDB/my_data_tbs01.dbf' size 10m;

drop table system.my_data_tb purge;
create table system.my_data_tb (
    my_id          number primary key
    ,first_name    varchar(100)
    ,last_name     varchar(100)
    ,insert_date   date
    )
tablespace my_data_tbs ;

insert into system.my_data_tb (my_id, first_name, last_name, insert_date) values (1,  'Scott',  'Tiger'  , sysdate);
insert into system.my_data_tb (my_id, first_name, last_name, insert_date) values (2,  'Jim'   , 'Brown'  , sysdate);
insert into system.my_data_tb (my_id, first_name, last_name, insert_date) values (3,  'Peyton', 'Manning', sysdate);

commit;

select count(*) from system.my_data_tb;



restore database from tag='TAG20231215T185837';
restore database from tag='TAG20231215T185829';

TAG20231215T185829
TAG20231215T185837

restore database;
ALTER SESSION SET NLS_DATE_FORMAT = 'DD-MON-RR HH24:MI:SS';
RECOVER DATABASE UNTIL TIME '16-DEC-23 15:55:00';

alter database open resetlogs;

```
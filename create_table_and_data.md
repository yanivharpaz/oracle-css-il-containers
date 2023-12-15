# Create table & data in the pluggable database

```

alter session set container=orclpdb;

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

```
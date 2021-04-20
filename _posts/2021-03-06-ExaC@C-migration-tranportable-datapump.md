---
title: ExaC@C Migration 2 - Transportable Datapump
date: 2021-03-05
comments: true
categories: [Exadata, Oracle]
tags: [Exadata, ExaC@C, Oracle Cloud]
typora-copy-images-to: ../assets/img



---

Transporting data is much faster than performing either an export/import or unload/load of the same data. It is faster because, for user-defined tablespaces, the data files containing all of the actual data are  copied to the target location, and you use Data Pump to transfer only  the metadata of the database objects to the new database.

You can transport data at any of the following levels:

- Database

  You can use the full transportable export/import feature to move an entire database to a different database instance.                           

- Tablespaces

  You can use the transportable tablespaces feature to move a set of tablespaces between databases.                           

- Tables, partitions, and subpartitions

  You can use the transportable tables feature to move a set of tables, partitions, and subpartitions between databases.                         

  

It is good for:

1. DB (11gr2 and above) upgrade to 19c. 
2. non-CDB (11gr2 and above) to PDB.

Limitations:

[General Limitations on Transporting Data](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/transporting-data.html#GUID-28800719-6CB9-4A71-95DD-4B61AA603173)

1. The source and the target databases must use compatible database character sets.
2. In a non-CDB, you cannot transport a tablespace to a target database that contains a tablespace of the same name. 

Notes:

1. You still have to encrypt those transported tablespaces once migration is done. 

---

Create or delete a pluggable database is so easy on ExaCC. 

```sql
dbaascli pdb create --pdbname targetdb --dbname TARGETCDB
dbaascli pdb delete --pdbname targetdb --dbname TARGETCDB

dbaascli pdb connect_string --pdbname targetdb
dbaascli pdb connect_info --pdbname targetdb
```

Check the newly created pluggable database on ExaCC.

~~~shell
[oracle@exacc-db01 ~]$ dbi pdbsize

    CON_ID NAME 		OPEN_MODE    Size(GB) RECOVERY GUID
---------- -------------------- ---------- ---------- -------- --------------------------------
	 2 PDB$SEED		READ ONLY	 2.73 ENABLED  B50214B4214037B1E053C40DD10A4E69
	 3 PDB1 		  READ WRITE	 4.34 ENABLED  BC1D3D315F0BABC7E053120D0E0AB35C
	 4 PDB2		    READ WRITE	25.91 ENABLED  BF519DFD96A53B2EE053120D0E0AB2E5
	 5 PDB3 		  READ WRITE	63.14 ENABLED  BBA20EE802A86377E053120D0E0A1DDF
	 6 PDB4		    READ WRITE	44.06 ENABLED  BF51D27EFA0B701CE053120D0E0A5354
	 7 targetdb   READ WRITE	 4.34 ENABLED  BFD26282F071FAC0E053120D0E0A299A

~~~



Export source database metadata.

~~~shell
create directory SOURCEDMP as '/staging';
expdp userid=\"/ as sysdba\" directory=SOURCEDMP full=y transportable=always version=12.0 dumpfile=source_full_transportable.dmp logfile=source_full_transportable.log reuse_dumpfiles=y exclude=table_statistics,index_statistics
~~~

You will see

~~~shell
Estimate in progress using BLOCKS method...
ORA-39123: Data Pump transportable tablespace job aborted
ORA-39185: The transportable tablespace failure list is

ORA-29335: tablespace 'DATA1' is not read only
ORA-29335: tablespace 'DATA2' is not read only
ORA-29335: tablespace 'USERS' is not read only
Job "SYS"."SYS_EXPORT_FULL_01" stopped due to fatal error at Mon Apr 12 11:37:46 2021 elapsed 0 00:00:12

~~~

Make sure the tablespaces are read only before export. (source database outage starts)

~~~sql
alter tablespace DATA1 read only;
alter tablespace DATA2 read only;
alter tablespace USERS read only;
select tablespace_name , status from dba_tablespaces;
~~~

Copy data files

~~~
cp /oradata/data01.dbf  /staging/
cp /oradata/data02.dbf  /staging/
cp /oradata/user01.dbf  /staging/

~~~

Switch back the tablespaces to read write. (source database outage ends)

~~~
alter tablespace DATA1 read write;
alter tablespace DATA2 read write;
alter tablespace USERS read write;
select tablespace_name , status from dba_tablespaces;
~~~

Pre-steps on target database. 

~~~sql
alter session set container=targetdb;
create directory SOURCEDMP as '/staging';
alter tablespace users rename to users_ts; --> sourcedb has same users tablespace which is transportable.
CREATE TEMPORARY TABLESPACE TEMP2; --> You might need to create particular temporary tablespace.
alter system set global_names=false; --> Exadata database by default setting is true, which affects views/materialized views with database links importing
~~~

Copy data files into target ASM.

~~~shell
sudo su - grid
asmcmd cp /staging/data01.dbf +datac2/TARGET_CDB_yyz18j/BFD26282F071FAC0E053120D0E0A299A/datafile
asmcmd cp /staging/data02.dbf +datac2/TARGET_CDB_yyz18j/BFD26282F071FAC0E053120D0E0A299A/datafile
asmcmd cp /staging/users01.dbf +datac2/TARGET_CDB_yyz18j/BFD26282F071FAC0E053120D0E0A299A/datafile

asmcmd ls -lt +datac2/TSTSS19_yyz18j/BFD26282F071FAC0E053120D0E0A299A/datafile/

~~~

Import metadata into target database.

~~~shell
impdp \"sys/xxxxx@//exacc-db-.yourdomain.com:1521/targetdb.yourdomain.com as sysdba\" \
full=y directory=SOURCEDMP dumpfile=sourcedb_full_transportable.dmp exclude=table_statistics,index_statistics exclude=schema:\"IN \(\'APEX_040200\',\'SYSTEM\',\'SYS\'\)\" \
transport_datafiles='+datac2/TARGET_CDB_yyz18j/BFD26282F071FAC0E053120D0E0A299A/datafile/data01.dbf', \
'+datac2/TARGET_CDB_yyz18j/BFD26282F071FAC0E053120D0E0A299A/datafile/data02.dbf', \
'+datac2/TARGET_CDB_yyz18j/BFD26282F071FAC0E053120D0E0A299A/datafile/users01.dbf' \
 logfile=import_sourcedb.log

##You could exclude specific schemas or tables or tablespaces. 
~~~

Now the migrateion is complete. You will see the transported tablespaces are not encrypted. And you have to encrypt them manually.

~~~sql
SQL> alter session set container=targetdb;

Session altered.

SQL> select tablespace_name,encrypted from dba_tablespaces;

TABLESPACE_NAME 	       ENC
------------------------------ ---
SYSTEM			       YES
SYSAUX			       YES
UNDOTBS1		       YES
PDBTEMP 		       YES
TEMP2			         YES
DATA1			         NO
DATA2	             NO
USERS			         NO

8 rows selected.

SQL> set linesize 200 pagesize 0
SQL> select 'alter database datafile '||chr(39)||df.name||chr(39)||' encrypt;' COMMAND from v$tablespace ts, v$datafile df where ts.ts#=df.ts# and ts.con_id=df.con_id and (ts.name not in ('SYSTEM','SYSAUX') and ts.name not in (select value from gv$parameter where name='undo_tablespace'));

alter database datafile '+datac2/TARGET_CDB_yyz18j/BFD26282F071FAC0E053120D0E0A299A/datafile/data01.dbf' encrypt;
alter database datafile '+datac2/TARGET_CDB_yyz18j/BFD26282F071FAC0E053120D0E0A299A/datafile/data02.dbf' encrypt;
alter database datafile '+datac2/TARGET_CDB_yyz18j/BFD26282F071FAC0E053120D0E0A299A/datafile/users01.dbf' encrypt;

SQL> alter pluggable database targetdb close immediate instances=all;

Pluggable database altered.

SQL> alter database datafile '+datac2/TARGET_CDB_yyz18j/BFD26282F071FAC0E053120D0E0A299A/datafile/data01.dbf' encrypt;

Database altered.
SQL> alter database datafile '+datac2/TARGET_CDB_yyz18j/BFD26282F071FAC0E053120D0E0A299A/datafile/data02.dbf' encrypt;

Database altered.
SQL> alter database datafile '+datac2/TARGET_CDB_yyz18j/BFD26282F071FAC0E053120D0E0A299A/datafile/users01.dbf' encrypt;

Database altered.

SQL> alter pluggable database targetdb open instances=all;

Pluggable database altered.

SQL> select tablespace_name,encrypted from dba_tablespaces;

TABLESPACE_NAME 	       ENC
------------------------------ ---
SYSTEM			       YES
SYSAUX			       YES
UNDOTBS1		       YES
PDBTEMP 		       YES
TEMP2			         YES
DATA1			         YES
DATA2              YES
USERS			         YES

8 rows selected.
~~~


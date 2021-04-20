---

title: ExaC@C Migration 3 - Pdb Remote Clone
date: 2021-03-07
comments: true
categories: [Exadata, Oracle]
tags: [Exadata, ExaC@C, Oracle Cloud]
typora-copy-images-to: ../assets/img



---

Cloning a Remote PDB or Non-CDB is so easy, just a few steps. 

~~~sql
--on non-cdb source database:
CREATE USER remote_clone_user IDENTIFIED BY xxxx;
GRANT CREATE SESSION, CREATE PLUGGABLE DATABASE TO remote_clone_user;

--on cdb target ExaCC database:
alter session set global_names=false; 

CREATE DATABASE LINK sourcedb connect to remote_clone_user identified by xxxx using '(DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = sourcedb.yourdomain.com)(PORT = 1521)) (CONNECT_DATA = (SERVER = DEDICATED) (SID = SOURCEDB)))';

select * from dual@sourcedb;

set time on timing on
create pluggable database sourcedb_clone FROM sourcedb@sourcedb KEYSTORE identified by "xxxxxx" parallel 8;

~~~

OCI How to HOT Clone a remote PDB with TDE from one DBS to another DBS with DB link (Doc ID 2623702.1)

If **source** database is **Active  Data Guard**, you will encounter this error. In Active dataguard with database open read only and Apply process running the datafiles would be in standby Fuzzy status. Refer to this document.

Create Pluggable Database From ADG errors out with ORA-600 [3647] (Doc ID 2072550.1)

~~~
2021-03-07T22:01:59.387673-05:00
Incident 22141 created, dump file: /u02/app/oracle/diag/rdbms/XXX/XXX/incident/incdir_22141/XXX_ora_102427_i22141.trc
ORA-00600: internal error code, arguments: [3647], [978], [16400], [], [], [], [], [], [], [], [], []

~~~



Disable redo apply on ADG first before running 'create pluggable database statement'.

~~~sql
SQL> Alter database recover managed  standby database cancel ;

Database altered.
~~~



Post steps after pdb hot clone is done.

~~~
alter pluggable database sourcedb_clone open;
select name,cause,type,message,status from PDB_PLUG_IN_VIOLATIONS where type = 'ERROR' and status <> 'RESOLVED';
~~~

If you are cloning non-cdb database to pdb having same version with different patch level. You will see below messages. 

PDB plugged in is a non-CDB, requires noncdb_to_pdb.sql be run.
DBRU bundle patch 201020 (DATABASE OCT 2020 RELEASE UPDATE 12.2.0.1.201020): Installed in the CDB but not in the PDB.

~~~shell
cd $ORACLE_HOME/OPatch
datapatch
~~~

If you are cloning non-cdb database to pdb having higher version. You will see below messages.

PDB plugged in is a non-CDB, requires noncdb_to_pdb.sql be run.
PDB's version does not match CDB's version: PDB's version 12.2.0.1.0. CDB's version 19.0.0.0.0.

~~~shell
dbupgrade -l /home/oracle/sourcedb_clone -c "sourcedb_clone"
~~~

~~~sql
alter session set container=sourcedb_clone;
set time on timing on
@?/rdbms/admin/noncdb_to_pdb

~~~

~~~sql
alter pluggable database sourcedb_clone close immediate;
alter pluggable database sourcedb_clone open instances=all;
SQL> select name,cause,type,message,status from PDB_PLUG_IN_VIOLATIONS where type = 'ERROR' and status <> 'RESOLVED';

no rows selected
~~~



If TDE is not enabled on source database. We will have to run below SQLs.

~~~sql
SQL> SELECT wrl_parameter, status, wallet_type FROM v$encryption_wallet;

WRL_PARAMETER
--------------------------------------------------------------------------------
STATUS                         WALLET_TYPE
------------------------------ --------------------
/var/opt/oracle/dbaas_acfs/xxxx/tde_wallet/
OPEN                           AUTOLOGIN

SQL> alter session set container=sourcedb_clone;

Session altered.

SQL> SELECT wrl_parameter, status, wallet_type FROM v$encryption_wallet;

WRL_PARAMETER
--------------------------------------------------------------------------------
STATUS                         WALLET_TYPE
------------------------------ --------------------

OPEN_NO_MASTER_KEY             AUTOLOGIN

SQL> ADMINISTER KEY MANAGEMENT SET KEY FORCE KEYSTORE IDENTIFIED BY xxxxx with backup;

keystore altered.
SQL> SELECT wrl_parameter, status, wallet_type FROM v$encryption_wallet;

WRL_PARAMETER
--------------------------------------------------------------------------------
STATUS                         WALLET_TYPE
------------------------------ --------------------

OPEN                           AUTOLOGIN

SQL> SELECT wrl_parameter, status, wallet_type FROM gv$encryption_wallet;

WRL_PARAMETER
--------------------------------------------------------------------------------
STATUS                         WALLET_TYPE
------------------------------ --------------------

OPEN                           AUTOLOGIN


OPEN                           AUTOLOGIN

~~~



Also we have to encrypt tablespaces.

~~~sql
SQL> select tablespace_name,encrypted from dba_tablespaces;

TABLESPACE_NAME                ENC
------------------------------ ---
SYSTEM                         NO
SYSAUX                         NO
UNDOTBS1                       NO
TEMP                           NO
APPDATA                        NO
APPDATA_IND                    NO
USERS                          NO

7 rows selected.

SQL> select name from v$datafile;

NAME
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+DATAC2/XXXXXX/BC712788B74C2BACE053140D0E0A08A1/DATAFILE/system.2857.1065735251
+DATAC2/XXXXXX/BC712788B74C2BACE053140D0E0A08A1/DATAFILE/sysaux.2858.1065735251
+DATAC2/XXXXXX/BC712788B74C2BACE053140D0E0A08A1/DATAFILE/undotbs1.2860.1065735251
+DATAC2/XXXXXX/BC712788B74C2BACE053140D0E0A08A1/DATAFILE/appdata.2861.1065735251
+DATAC2/XXXXXX/BC712788B74C2BACE053140D0E0A08A1/DATAFILE/appdata_ind.2859.1065735251
+DATAC2/XXXXXX/BC712788B74C2BACE053140D0E0A08A1/DATAFILE/users.2856.1065735251

6 rows selected.

SQL> alter pluggable database sourcedb_clone close instances=all;

Pluggable database altered.

SQL> SQL> SQL> 
SQL> 
SQL> alter database datafile '+DATAC2/XXXXXX/BC712788B74C2BACE053140D0E0A08A1/DATAFILE/users.2856.1065735251' encrypt;

Database altered.
SQL> alter database datafile '+DATAC2/XXXXXX/BC712788B74C2BACE053140D0E0A08A1/DATAFILE/appdata.2861.1065735251' encrypt;

Database altered.

SQL> alter database datafile '+DATAC2/XXXXXX/BC712788B74C2BACE053140D0E0A08A1/DATAFILE/appdata_ind.2859.1065735251' encrypt;

Database altered.

10 mins

SQL> alter pluggable database sourcedb_clone open instances=all;

Pluggable database altered.

SQL> SELECT wrl_parameter, status, wallet_type FROM v$encryption_wallet;

WRL_PARAMETER
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
STATUS                         WALLET_TYPE
------------------------------ --------------------

OPEN                           AUTOLOGIN


SQL> select tablespace_name,encrypted from dba_tablespaces;

TABLESPACE_NAME                ENC
------------------------------ ---
SYSTEM                         NO
SYSAUX                         NO
UNDOTBS1                       NO
TEMP                           NO
APPDATA                        YES
APPDATA_IND                    YES
USERS                          YES

 rows selected.

~~~

At this moment, we have done pdb hot clone successfully. 

---

There is another easy way just one command to clone pdb on ExaCC by using dbaascli. 

~~~shell
[oracle@exacc-db01 ~]$ dbaascli pdb remote_clone --pdbname PDB1 --source_db SCDB_yyz1xc --source_db_scan exacc-db-scan.yourdomain.com
DBAAS CLI version 20.1.3.5.0
Executing command pdb remote_clone --pdbname PDB1 --source_db SCDB_yyz1xc --source_db_scan exacc-db-.yourdomain.com
Executing: pdb remote_clone
dbcore: WARNING: query returned no output
Please enter the password for SYS user for CDB SCDB_yyz1xc:
Please enter SYS password:

Please confirm password:

TNS_DIR is /var/opt/oracle/dbaas_acfs/log/SCDB/pdb/17889
INFO : pdb remote_clone operation success


2021-04-13T16:13:52.141354-04:00
create pluggable database SCDB_PDB1 from PDB1@SCDB_yyz1xc keystore identified by *
.....
2021-04-13T16:33:51.006286-04:00
freeing rdom 4
Completed: create pluggable database SCDB_PDB1 from PDB1@SCDB_yyz1xc keystore identified by *
~~~


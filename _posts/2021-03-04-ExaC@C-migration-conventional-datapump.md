---
title: ExaC@C Migration 1 - Conventional Datapump
date: 2021-03-04
comments: true
categories: [Exadata, Oracle]
tags: [Exadata, ExaC@C, Oracle Cloud]
typora-copy-images-to: ../assets/img


---

### <font color=pink>Conventional  Datapump should be always the first migration method for the small non-critical databases. </font>

You can use this method regardless of the endian format and database character set of the source database. You can also use Data Pump to migrate data between different versions of Oracle Database. This method is simple to implement, provides the broadest cross-platform support and enables you to physically re-organize your target database; however, the time and resources required for export and import may rule out this approach for situations with large databases or limited timeframes.



## References

- [Upgrade Your Database To 19c With Export-Import Datapump](http://dbaparadise.com/2020/08/upgrade-your-database-to-19c-with-export-import-datapump/)

>**5 Reasons I like this method** ( totally agree :smile: )
>
>1. My favorite reason for this method is, you start with a brand new database, usually on a brand new server, and you are not upgrading from previous versions. Brand new database, brand new data dictionary, less bugs and less problems.
>2. My second favorite reason is that the rollback process is very easy and very quick. You just point the application and the users back to the original database and you are back in business. No need to restore from previous backup, as the source database is intact.
>3. You can setup everything ahead of time, with the only thing remaining to bring over the users and the data.
>4. You can test the upgrade process many times, and fine tune it, until you get the desired results. Also you can test the upgraded database ahead of time, after your test upgrade runs.
>5. You can time the upgrade as well, so you know exactly how long your upgrade takes, and how long the outage will be.

The good thing for me is this method is bypassing the steps.

1. TDE setup. 
2. DB (11gr2 and above) upgrade to 19c. 
3. non-CDB (11gr2 and above) to PDB.
4. Character set change. 

---

Create or delete a pluggable database is so easy on ExaCC. 

```sql
dbaascli pdb create --pdbname targetdb --dbname TARGETCDB
dbaascli pdb delete --pdbname targetdb --dbname TARGETCDB

dbaascli pdb connect_string --pdbname targetdb
dbaascli pdb connect_info --pdbname targetdb
```

Capture the invalid objects on the source database (on premise). Will compare it with target database (ExaCC) later after migration.

```sql
 select owner,count(*) from dba_invalid_objects group by owner order by 1;
```

Capture the components status. 

```sql
select comp_name,status from dba_registry;
```

Capture the tablespaces except **system, sysaux, undotbs, users, temp**

```sql
select tablespace_name from dba_tablespaces where tablespace_name not in ('SYSTEM','SYSAUX','USERS','TEMP','UNDOTBS');
```



To make a nice and clean import into ExaCC non-CDB or PDB, We need to find out which specific schemas or users to migrate except those predefined user accounts provided by Oracle database and installed components ( for example if you installed Oracle Text, Oracle Multimediam Oracle Application Express, etc).  I have collected Oracle predefined user accounts in below query. Just run it we will get those specific schemas or users for migration. 

```sql
alter session set nls_date_format='yyyy-mm-dd HH24:mi:ss';
col username for a30
col PROFILE for a30
set linesize 200 pagesize 1000
select username,created,default_tablespace,profile from dba_users where
username not in (select username FROM DBA_USERS_WITH_DEFPWD)
and default_tablespace not in ('SYSAUX','SYSTEM')
and username not in ('ANONYMOUS',
'CTXSYS',
'DBSNMP',
'EXFSYS',
'LBACSYS',
'MDSYS',
'MGMT_VIEW',
'OLAPSYS',
'ORDDATA',
'OWBSYS',
'ORDPLUGINS',
'ORDSYS',
'OUTLN',
'SI_INFORMTN_SCHEMA',
'SYS',
'SYSMAN',
'SYSTEM',
'WK_TEST',
'WKSYS',
'WKPROXY',
'WMSYS',
'XDB',
'APEX_PUBLIC_USER',
'DIP',
'FLOWS_040100',
'FLOWS_030000',
'SCOTT',
'FLOWS_FILES',
'MDDATA',
'ORACLE_OCM',
'SPATIAL_CSW_ADMIN_USR',
'SPATIAL_WFS_ADMIN_USR',
'XS$NULL',
'APPQOSSYS',
'APEX_030200',
'APEX_200200',
'APEX_INSTANCE_ADMIN_USER',
'APEX_LISTENER',
'APEX_REST_PUBLIC_USER',
'ORDS_METADATA',
'ORDS_PUBLIC_USER',
'TSMSYS',
'AUDSYS',
'GSMUSER',
'SYSBACKUP',
'SYSDG',
'SYSKM') order by 1;
```

Double check the oracle features whether they are in use or not.

```sql
alter session set nls_date_format='yyyy-mm-dd HH24:mi:ss';
set pages 999 linesize 200
col c1 heading 'feature' format a45
col c2 heading 'times|used' format 999,999
col c3 heading 'first|used'
col c4 heading 'used|now'

select
   name c1,version,
   detected_usages c2,
   first_usage_date c3,
   currently_used c4
from dba_feature_usage_statistics
where upper(name) like upper('Application Express') 
or upper(name) like upper('%Java%')
or upper(name) like upper('%Workspace%')
or upper(name) like upper('%Multimedia%')
or upper(name) like upper('%XML%')
or upper(name) like upper('%Text%')
or upper(name) like upper('%XDK%')
or upper(name) like upper('%XDB%')
or upper(name) like upper('%Catalog%')
or upper(name) like upper('%OLAP%')
or upper(name) like upper('%Spatial%')
or upper(name) like upper('%Expression%')
or upper(name) like upper('%Rules%');

select owner, count(*) from all_objects
where object_type like '%JAVA%' group by owner;

select
   name c1,version,
   detected_usages c2,
   first_usage_date c3,
   currently_used c4
from
   dba_feature_usage_statistics
where
   first_usage_date is not null order by name;
```



--------------------------------------
Generate pre-create objects SQLs

1.  **sourcedb_tablespaces.sql (remove existing pdb tablespaces system, sysaux, undotbs, users, temp)**
2.  **sourcedb_users.sql ( replace temp with pdbtemp )**
3.  **sourcedb_profiles.sql**
4.  **sourcedb_roles.sql**
5.  **sourcedb_directories.sql**
6.  **Sourcedb_dblinks.sql**
7.  **sourcedb_public_synonyms.sql**

```sql
SET LONG 9999999 LONGCHUNKSIZE 9999999 PAGESIZE 0 LINESIZE 9999 FEEDBACK OFF VERIFY OFF TRIMSPOOL ON TIMING OFF HEADING OFF

BEGIN
   DBMS_METADATA.set_transform_param (DBMS_METADATA.session_transform, 'SQLTERMINATOR', true);
   DBMS_METADATA.set_transform_param (DBMS_METADATA.session_transform, 'PRETTY', true);
END;
/

spool sourcedb_tablespaces.sql
select 'create tablespace '||tablespace_name||';' from dba_tablespaces where tablespace_name not in ('SYSTEM','SYSAUX','USERS','TEMP','UNDOTBS');
spool off

spool sourcedb_users.sql
SELECT dbms_metadata.get_ddl('USER',U.username) from dba_users U where U.username not in (select username FROM DBA_USERS_WITH_DEFPWD) and default_tablespace not in ('SYSAUX','SYSTEM') and U.username not in ('Appliction Schema list');
spool off

spool sourcedb_profiles.sql
  SELECT DISTINCT 'CREATE PROFILE ' || profile || ' LIMIT CPU_PER_CALL DEFAULT;'
  FROM dba_profiles
  WHERE PROFILE <> 'DEFAULT';
  SELECT      'ALTER PROFILE '
           || profile
           || ' LIMIT '
           || resource_name
           || ' '
           || LIMIT
           || ';'
  FROM     dba_profiles
  ORDER BY 1;
spool off

spool sourcedb_system_grant.sql
SELECT DBMS_METADATA.GET_GRANTED_DDL('SYSTEM_GRANT',U.username) from dba_users U where U.username not in (select username FROM DBA_USERS_WITH_DEFPWD) and default_tablespace not in ('SYSAUX','SYSTEM') and U.username not in ('Appliction Schema list');
spool off

spool sourcedb_role_grant.sql
SELECT DBMS_METADATA.GET_GRANTED_DDL('ROLE_GRANT',U.username) from dba_users U where U.username not in (select username FROM DBA_USERS_WITH_DEFPWD) and default_tablespace not in ('SYSAUX','SYSTEM') and U.username not in ('Appliction Schema list');
spool off

spool sourcedb_object_grant.sql
SELECT DBMS_METADATA.GET_GRANTED_DDL('OBJECT_GRANT',U.username) from dba_users U where U.username not in (select username FROM DBA_USERS_WITH_DEFPWD) and default_tablespace not in ('SYSAUX','SYSTEM') and U.username not in ('Appliction Schema list');
spool off

spool sourcedb_roles.sql
select dbms_metadata.get_ddl('ROLE',role) from dba_roles where role not in ('SPATIAL_CSW_ADMIN','CSW_USR_ROLE','MGMT_USER','XDBWEBSERVICES','ORACLE_JAVA_DEV','ORDS_ADMINISTRATOR_ROLE','ORDS_RUNTIME_ROLE') and ORACLE_MAINTAINED != 'Y';

SELECT DBMS_METADATA.GET_GRANTED_DDL('SYSTEM_GRANT',role) from dba_roles where role not in ('SPATIAL_CSW_ADMIN','CSW_USR_ROLE','MGMT_USER','XDBWEBSERVICES','ORACLE_JAVA_DEV','ORDS_ADMINISTRATOR_ROLE','ORDS_RUNTIME_ROLE') and ORACLE_MAINTAINED != 'Y';

SELECT DBMS_METADATA.GET_GRANTED_DDL('ROLE_GRANT',role) from dba_roles where role not in ('SPATIAL_CSW_ADMIN','CSW_USR_ROLE','MGMT_USER','XDBWEBSERVICES','ORACLE_JAVA_DEV','ORDS_ADMINISTRATOR_ROLE','ORDS_RUNTIME_ROLE') and ORACLE_MAINTAINED != 'Y';

SELECT DBMS_METADATA.GET_GRANTED_DDL('OBJECT_GRANT',role) from dba_roles where role not in ('SPATIAL_CSW_ADMIN','CSW_USR_ROLE','MGMT_USER','XDBWEBSERVICES','ORACLE_JAVA_DEV','ORDS_ADMINISTRATOR_ROLE','ORDS_RUNTIME_ROLE') and ORACLE_MAINTAINED != 'Y';
spool off

spool sourcedb_dblinks.sql
select dbms_metadata.get_ddl('DB_LINK',DB_LINK,OWNER) from dba_db_links;
spool off

spool sourcedb_directories.sql
select dbms_metadata.get_ddl('DIRECTORY',DIRECTORY_NAME) from dba_directories;
spool off

spool sourcedb_public_synonyms.sql
select dbms_metadata.get_ddl('SYNONYM',synonym_name, owner) from dba_synonyms where owner='PUBLIC' and table_owner not in (select username FROM DBA_USERS_WITH_DEFPWD)
and table_owner not in ('Appliction Schema list');
spool off
```

------------------------------------------------
export the schemas on prem

```shell
#!/bin/sh

if [ $# -ne 2 ]; then
echo "";
echo "#################";
echo "usage - expdp_schema.sh SOURCEDB parfile";
echo "#################";
exit 1;
fi

ORACLE_SID=$1
ORACLE_HOME=`/usr/local/bin/dbhome ${ORACLE_SID}`
LD_LIBRARY_PATH=$ORACLE_HOME/lib
LD_LIBRARY_PATH_64=$LD_LIBRARY_PATH:/usr/shlib:/usr/lib/X11:/lib:/usr/lib
ORACLE_BASE=/u01/app/oracle
export ORACLE_SID ORACLE_HOME LD_LIBRARY_PATH LD_LIBRARY_PATH_64 ORACLE_BASE
LOGFILE=${ORACLE_SID}_db_expdp.log;
EXPFILE=${ORACLE_SID}_expdp_%U.dmp;

if [ ! -d /backup/${ORACLE_SID} ]; then
mkdir /backup/${ORACLE_SID}
fi
${ORACLE_HOME}/bin/sqlplus -S "/ as sysdba" <<EOF
create or replace directory ${ORACLE_SID}DMP as '/backup/${ORACLE_SID}';
EOF

flashbackuptime=`date +'%Y-%m-%d %H:%M:%S'`

${ORACLE_HOME}/bin/expdp userid=\"/ as sysdba\" dumpfile=${EXPFILE} logfile=${LOGFILE} directory=${ORACLE_SID}DMP FLASHBACK_TIME=\"TO_TIMESTAMP\(\'$flashbackuptime\', \'YYYY-MM-DD HH24:MI:SS\'\)\"  parfile=$2 compression=all REUSE_DUMPFILES=y
```

```shell
cat sourcedb_para.txt

EXCLUDE=STATISTICS
PARALLEL=4
SCHEMAS=schema_a, schema_b....
```

```shell
expdp_schema.sh sourcedb sourcedb_para.txt
```

-------------------------------------------------



Run sql in following orders on ExaCC

1.  **sourcedb_tablespaces.sql (remove existing pdb tablespaces system, sysaux, undotbs, users, temp)**
2.  **sourcedb_profiles.sql**
3.  **sourcedb_roles.sql**
4.  **sourcedb_users.sql ( replace temp with pdbtemp )**
6.  **sourcedb_directories.sql **

Run import the schemas on exacc

```sql
create directory sourcedb as '/backup/sourcedb';
impdp \"sys@sourcedb as sysdba\" directory=sourcedb dumpfile=sourcedb_expdp_%U.dmp parallel=4 logfile=impdp.log
```

7. **sourcedb_public_synonyms.sql**

----------------------------------------

recompile invalid objects.

```sql
@?rdbms/admin/utlrp
```

Compare the invalid objects between on prem and exacc

```sql
select owner,count(*) from dba_invalid_objects group by owner order by 1;
```

Compare the components status between on prem and exacc

```sql
select comp_name,status from dba_registry;
```

```sql
SQL> create database link sourcedb_prem connect to oradba identified by xxxxxxx using 'sourcedb_PREM';

Database link created.
```

Use below query to get the result for newly increased invalid objects compared with on prem.

```sql
select owner,count(*) from dba_invalid_objects group by owner minus select owner,count(*) from dba_invalid_objects@sourcedb_PREM  group by owner;
```


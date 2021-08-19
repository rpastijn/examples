# Setup LOB storage on S3 storage #

This example will show step-by-step how to setup the Oracle database for storage of LOB table contents on remote Amazon S3 storage using the DBMS_DBFS_HS package.

## Prerequisites ##

- Amazon account with S3 usage rights
   - Generated Access Key ID and Secret Access Key
- Oracle database 19c and up

## 1 - Install OSB Cloud Module ##

For the setup we need an Oracle wallet that contains the Amazon Access Key ID and Secret Access Key. We also need the library (```libosbws.so```) for connecting to the S3 storage. We could leverage any existing library and/or manually generate the Oracle wallet but it is easier to have Oracle generate the required items. This also means we will have the latest version and we can update the items without updating our Oracle Home.

### Download the Oracle Backup Cloud Module Setup ###

- Log-in as the user owning your oracle installation
	- In this example, this is the ```oracle``` user. 
- Navigate to https://www.oracle.com/database/technologies/secure-backup-s3.html and download the install package
	- For this example, the download location is ```/home/oracle```
	- If you did not download the item on the Oracle Database server, upload it now from the download location to the Oracle Database server.
- Unzip the install package

```
[oracle@dba ~]$ <copy>unzip osbws_installer.zip</copy>
Archive:  osbws_installer.zip
  inflating: osbws_install.jar
  inflating: osbws_readme.txt
```

### Create object needed for install ###

You can either use command line options for the install of the package or use a configuration file. In this example, we will use a configuration file (```/home/oracle/aws_osbws_setup.conf```):

```
<copy>
echo "" > /home/oracle/aws_osbws_setup.conf
echo "-awsEndPoint s3.amazonaws.com" >> /home/oracle/aws_osbws_setup.conf
echo "-useHttps" >> /home/oracle/aws_osbws_setup.conf
echo "-walletDir /home/oracle/aws_walletdir" >> /home/oracle/aws_osbws_setup.conf
echo "-configFile /home/oracle/aws_configfile.conf" >> /home/oracle/aws_osbws_setup.conf
echo "-libDir /home/oracle/aws_libdir" >> /home/oracle/aws_osbws_setup.conf
echo "-libPlatform linux64" >> /home/oracle/aws_osbws_setup.conf
</copy>
```

We also need to add your personal AWS Access Key and AWS Secret Key to the file. Please replace the below values with your personal values before executing:

```
$ <copy>echo "-AWSID (your AWS Acces Key)" >> /home/oracle/aws_osbws_setup.conf</copy>
```
```
$ <copy>echo "-AWSKey (your AWS Secret Key)" >> /home/oracle/aws_osbws_setup.conf</copy>
```

> **Note**: Amazon AWS allows 2 ways to access the policy; using the generated AWS Keys and through a user with IAM  policies. In this example we are using the (less secure) AWS Keys. This means that anyone who gets access to the keys will have the same security level as the user who generated the keys. It is more secure to use IAM policies. Make sure you secure your keys by deleting the setup file after generating the Oracle wallet.

We also need to create the library location for our required libosbws.so:

```
$ <copy>mkdir /home/oracle/aws_libdir</copy>
```

### Execute the installation of the module ###

Now we can install the Oracle Cloud Backup Module:

```
[oracle@dba ~]$ <copy>java -jar osbws_install.jar -argFile /home/oracle/aws_osbws_setup.conf</copy>
Oracle Secure Backup Web Service Install Tool, build 12.2.0.1.0DBBKPCSBP_2018-06-12
AWS credentials are valid.
Oracle Secure Backup Web Service wallet created in directory /home/oracle/aws_walletdir.
Oracle Secure Backup Web Service initialization file /home/oracle/aws_configfile.conf created.
Downloading Oracle Secure Backup Web Service Software Library from file osbws_linux64.zip.
Download complete.
```

### Optional check that installation was correct ###

The setup we just did was officially to have RMAN backup the Oracle database to Amazon S3 storage. We can test the installation by doing a small backup (for example of the controlfile) to the Amazon storage. 

> **WARNING**
> Only execute the following step if your current RMAN backup is not using SBT as channel device.

First we start RMAN and connect to the target database:

```
[oracle@dba ~]$ <copy>rman target /</copy>

Recovery Manager: Release 19.0.0.0.0 - Production on Wed Aug 18 09:21:53 2021
Version 19.10.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

connected to target database: DB19 (DBID=1532791018)
```

Second, we configure the channel SBT to make use of our recently installed and configured cloud destination:

```
RMAN> <copy>configure channel device type sbt
      parms='SBT_LIBRARY=libosbws.so, SBT_PARMS=(OSB_WS_PFILE=/home/oracle/aws_configfile.conf)';</copy>

using target database control file instead of recovery catalog
new RMAN configuration parameters:
CONFIGURE CHANNEL DEVICE TYPE 'SBT_TAPE' PARMS  'SBT_LIBRARY=libosbws.so, SBT_PARMS=(OSB_WS_PFILE=/home/oracle/aws_configfile.conf)';
new RMAN configuration parameters are successfully stored
```

Now we can start a small test backup to see if the setup is succesful:

```
RMAN> <copy>backup device type sbt current controlfile;</copy>

Starting backup at 18-AUG-21
allocated channel: ORA_SBT_TAPE_1
channel ORA_SBT_TAPE_1: SID=41 device type=SBT_TAPE
channel ORA_SBT_TAPE_1: Oracle Secure Backup Web Services Library VER=19.0.0.1
channel ORA_SBT_TAPE_1: starting full datafile backup set
channel ORA_SBT_TAPE_1: specifying datafile(s) in backup set
including current control file in backup set
channel ORA_SBT_TAPE_1: starting piece 1 at 18-AUG-21
channel ORA_SBT_TAPE_1: finished piece 1 at 18-AUG-21
piece handle=0306qcbs_3_1_1 tag=TAG20210818T092516 comment=API Version 2.0,MMS Version 19.0.0.1
channel ORA_SBT_TAPE_1: backup set complete, elapsed time: 00:00:15
Finished backup at 18-AUG-21

Starting Control File and SPFILE Autobackup at 18-AUG-21
piece handle=c-1532791018-20210818-00 comment=API Version 2.0,MMS Version 19.0.0.1
Finished Control File and SPFILE Autobackup at 18-AUG-21

RMAN>
```

When you login to your Amazon S3 storage, you will see that the library has created a new bucket (directory) and has backed-up your controlfile to the S3 storage.

This means we can now continue setting up the Oracle environment.

## 2 - Setup Oracle DBFS Hybrid Storage on S3 ##

For this example, we will use the original example from the documentation but split it out into sections to explain what is happening. The original example can be found in the Oracle documentation: https://docs.oracle.com/en/database/oracle/oracle-database/21/adlob/using-hierarchical-store.html#GUID-EE3C969D-C6DE-49B5-8694-1AF22E143E4D

### Create user and grant permissions ###



### Create the tablespace we will use for the S3 Store ###

Login to the database as user who has rights to create a new tablespace. In this example, the tablespace is called S3_TABLESPACE and can be created like this:

```
SQL> <copy>create tablespace s3_tablespace datafile size 100M;</copy>

Tablespace created.
```

### Create the S3 store ###

In the below example

- v_tbsname is the tablespace used for the store,
- v_tblname is the table holding all the store entities,
- v_cachesize is the space used by the store to cache content in the tablespace,
- v_lob_cache_quota is the fraction of v_cachesize allocated to level-1 cache and
- v_opt_tarball_size is minimum amount of content that is accumulated in level-2 cache before being stored in AmazonS3

```
<copy>
declare 
  v_store_name       varchar2(32)  := 's3_store'; 
  v_tblname          varchar2(30)  := 's3_table'; 
  v_tbsname          varchar2(30)  := 's3_tablespace'; 
  v_lob_cache_quota  number        := 0.8 ; 
  v_opt_tarball_size number        := 5242880; 
  v_cachesize        number        := 5*v_opt_tarball_size ; 
begin 
 
  dbms_dbfs_hs.createStore(
    store_name           => v_store_name,  
    store_type           => dbms_dbfs_hs.STORETYPE_AMAZONS3,
    tbl_name             => v_tblname, 
	tbs_name             => v_tbsname, 
	cache_size           => v_cachesize,
    lob_cache_quota      => v_lob_cache_quota, 
	optimal_tarball_size => v_opt_tarball_size) ; 
	
end;
/
</copy>

PL/SQL procedure successfully completed.
```

We can check the creation of the store by querying the USER_DBFS_HS view:

```
SQL> <copy>select * from user_dbfs_hs;</copy>

STORENAME
--------------------------------------------------------------------------------
s3_store
```

### Add required properties to the S3 store ###

> **Note**: The bucket name on Amazon cannot contain any special characters and needs to be in lower case.
> **Note**: The bucket name needs to be unique across all tenancies (not just your tenancy). So I put in a random generator to be sure.

```
<copy>
declare 

  v_store_name       varchar2(32)  := 's3_store'; 
  v_sbt_loc          varchar2(200) := '/home/oracle/aws_libdir/libosbws.so';
  v_aws_wallet_loc   varchar2(200) := 'location=file:/home/oracle/aws_walletdir CREDENTIAL_ALIAS=aws_aws';
  v_aws_endpoint     varchar2(200) := 's3.amazonaws.com';
  v_aws_bucket       varchar2(200) := 'oracles3'||dbms_random.string('l',8);
  v_opt_tarball_size number        := 5242880;

begin	

  dbms_dbfs_hs.setstoreproperty(
    store_name     => v_store_name,
    property_name  => dbms_dbfs_hs.PROPNAME_SBTLIBRARY,
    property_value => v_sbt_loc );
	
  dbms_dbfs_hs.setstoreproperty(
    store_name     => v_store_name,
    property_name  => dbms_dbfs_hs.PROPNAME_S3HOST,
    property_value => v_aws_endpoint); 

  dbms_dbfs_hs.setstoreproperty(
    store_name     => v_store_name,
    property_name  => dbms_dbfs_hs.PROPNAME_BUCKET,
    property_value => v_aws_bucket ) ; 

  dbms_output.put_line('Your generated bucket is: '||v_aws_bucket);
 
  dbms_dbfs_hs.setstoreproperty(
    store_name     => v_store_name,
    property_name  => dbms_dbfs_hs.PROPNAME_WALLET,
    property_value => v_aws_wallet_loc);
	
  dbms_dbfs_hs.setstoreproperty(
    store_name     => v_store_name,
    property_name  => dbms_dbfs_hs.PROPNAME_LICENSEID,
    property_value => dbms_random.random );

  dbms_dbfs_hs.setstoreproperty(
    store_name     => v_store_name,
    property_name  => dbms_dbfs_hs.PROPNAME_COMPRESSLEVEL,
    property_value => 'NONE'); 

end;
/
</copy>

PL/SQL procedure successfully completed.
```

If your database is behind a firewall, you can optionally execute the following statement:

```
<copy>
declare 
  v_store_name       varchar2(32)  := 's3_store';
  v_proxy            varchar2(200) := '<http://www-proxy.mycompany.com:80/>';
begin
   dbms_dbfs_hs.setstoreproperty(
    store_name     => v_store_name,
    property_name  => dbms_dbfs_hs.PROPNAME_HTTPPROXY,
    propert_value  => v_proxy );
end;
/
</copy>

PL/SQL procedure successfully completed.
```

### Create the bucket for the S3 store ###

We can now create the bucket on the S3 store. This can be done manually in AWS but we can also do it using SQL. It is also a good test to see if the setup was correct as this is the first command that actually connects to AWS.

```
<copy>
declare
 
  v_store_name        varchar2(32)  := 's3_store'; 

begin	
 
  dbms_dbfs_hs.createbucket(
    store_name => v_store_name ) ;

end;
/
</copy>

PL/SQL procedure successfully completed.
```

## Register and mount the S3 store ##

```
<copy>
declare
 
  v_store_name       varchar2(32)  := 's3_store';
  v_provider_name    varchar2(32)  := 's3_provider';
  v_store_mount      varchar2(32)  := 's3_mount'; 

begin	

  dbms_dbfs_content.registerstore(
    store_name       => v_store_name,
    provider_name    => v_provider_name,
    provider_package => 'dbms_dbfs_hs') ; 
 
  dbms_dbfs_content.mountstore(
    store_name     => v_store_name,
    store_mount    => v_store_mount) ; 
end ; 
/ 
</copy>

PL/SQL procedure successfully completed.
```

We have now succesfully setup a Hybrid Storage store on Amazon S3.

## Query the created table by the setup ##

In the beginning we set the table name for the store to be 's3_table'. We can now describe and query the table:

```
<copy>
SQL> <copy>desc S3_TABLE</copy>
 Name                                      Null?    Type
 ----------------------------------------- -------- ----------------------------
 VOLID                                     NOT NULL NUMBER
 CSNAP#                                    NOT NULL NUMBER
 LSNAP#                                             NUMBER
 PATHNAME                                  NOT NULL VARCHAR2(1024)
 ITEM                                      NOT NULL VARCHAR2(256)
 PATHTYPE                                  NOT NULL NUMBER(38)
 FILEDATA                                           BLOB
 POSIX_NLINK                                        NUMBER(38)
 POSIX_MODE                                         NUMBER(38)
 POSIX_UID                                          NUMBER(38)
 POSIX_GID                                          NUMBER(38)
 STD_ACCESS_TIME                           NOT NULL TIMESTAMP(6)
 STD_ACL                                            VARCHAR2(1024)
 STD_CHANGE_TIME                           NOT NULL TIMESTAMP(6)
 STD_CONTENT_TYPE                                   VARCHAR2(1024)
 STD_CREATION_TIME                         NOT NULL TIMESTAMP(6)
 STD_DELETED                               NOT NULL NUMBER(38)
 STD_GUID                                  NOT NULL NUMBER(38)
 STD_MODIFICATION_TIME                     NOT NULL TIMESTAMP(6)
 STD_OWNER                                          VARCHAR2(32)
 STD_PARENT_GUID                           NOT NULL NUMBER(38)
 STD_REFERENT                                       VARCHAR2(1024)
 OPT_HASH_TYPE                                      VARCHAR2(32)
 OPT_HASH_VALUE                                     VARCHAR2(128)
 OPT_LOCK_COUNT                                     NUMBER(38)
 OPT_LOCK_DATA                                      VARCHAR2(128)
 OPT_LOCK_STATUS                                    NUMBER(38)
```

or query the table

```
SQL> <copy>select pathname, item from s3_table;</copy>

PATHNAME            ITEM
------------------- ----------------------
/                   ROOT
/.sfs               .sfs
/.sfs/attributes    attributes
/.sfs/tools         tools
/.sfs/snapshots     snapshots
/.sfs/RECYCLE       RECYCLE
/.sfs/content       content

7 rows selected.
```

## Move data into the table and store ##

### Create new files on the storage ###

```
<copy>
declare
  v_store_name  varchar(32)   := 's3_store';
  v_mycontent   blob; 
  v_path        varchar2(256); 
  v_prop1       dbms_dbfs_content_properties_t ; 

begin 

  for x in 1.10 loop
  
    v_path := '/file'|| to_char(x) ; 

    dbms_dbfs_content.createFile( 
      store_name   => v_store_name,
      path         => v_path,
      properties   => v_prop1, 
      content      => v_mycontent) ;
 
  end loop ; 

  commit ; 
  
end;
/
</copy>

PL/SQL procedure successfully completed.
```

At this moment, the files are not yet physically created on the remote storage. They are buffered in the cache setup for the store and will be written when we push the files to the store.

### Add some text to the empty files ###

```
<copy>
declare
  v_store_name  varchar2(32)  := 's3_store'; 
  v_path        varchar2(256); 
  v_mycontent   blob          := empty_blob() ; 
  v_buffer      varchar2(1050) ; 
  v_item_type   integer ; 
  v_properties  dbms_dbfs_content_properties_t ; 

begin 

  v_buffer := 'Oracle provides an integrated management solution for managing Oracle database with '||
              'a unique top-down application management approach. With new self-managing '          ||
              'capabilities, Oracle eliminates time-consuming, error-prone administrative '     ||
              'tasks, so database administrators can focus on strategic business objectives '    ||
              'instead of performance and availability fire drills. Oracle Management Packs for '  ||
              'Database provide signifiCant cost and time-saving capabilities for managing Oracle '   ||
              'Databases. Independent studies demonstrate that Oracle Database is 40 percent easier ' ||
              'to manage over DB2 and 38 percent over SQL Server.'; 
 
  v_mycontent := utl_raw.cast_to_raw(v_buffer);

  for x in 1..10 loop

    v_path := '/file'|| to_char(x) ;
 
    dbms_dbfs_content.putpath(
      store_name  => v_store_name,
      path        => v_path, 
      properties  => v_properties, 
      content     => v_mycontent,
      item_type   => v_item_type);

  end loop;

  commit;

end;
/
</copy>
```

## Flushing the cache to the S3 store ##

To force the system to flush the cache to the S3 object store, you can execute the following command:

```
<copy>
declare
  v_store_name  varchar2(32)  := 's3_store'; 
begin
  dbms_dbfs_hs.flushCache(
    store_name => v_store_name) ;

  dbms_dbfs_hs.storePush(
    store_name => v_store_name) ;
 
  commit ; 

end;
/
</copy>

PL/SQL procedure successfully completed.
```

After a while, a directory structure will show up in your Amazon S3 storage. You cannot see the individual files like we named then (file1, file2 etc). Instead, there is an RMAN kind of structure that holds the information. 

```
file_chunk/812118129/DB19/lob/2021-08-19/__DB19_1_C9EAF57CC2EC6118E053AD00000A16E8_1/zY2KdHVaIDdC/0000000001
file_chunk/812118129/DB19/lob/2021-08-19/__DB19_1_C9EAF57CC2EC6118E053AD00000A16E8_1/zY2KdHVaIDdC/metadata.xml
sbt_catalog/__DB19_1_C9EAF57CC2EC6118E053AD00000A16E8_1/metadata.xml
```

We can see the files in our table that contains the BLOB:

```
SQL> <copy>select pathname, item, pathtype from s3_table </copy>

PATHNAME            ITEM
------------------- -------------------------
/                   ROOT
/.sfs               .sfs
/.sfs/attributes    attributes
/.sfs/tools         tools
/.sfs/snapshots     snapshots
/.sfs/RECYCLE       RECYCLE
/.sfs/content       content
/file1              file1
/file2              file2
/file3              file3
/file4              file4
/file5              file5
/file6              file6
/file7              file7
/file8              file8
/file9              file9
/file10             file10

17 rows selected.
```

End of Lab

### Work Arounds to get this to work ###

- Set the opt_tarball_size to 5242880
	- If you dont do this, the following error
	- ```KBHS-01015: value of parameter OSB_WS_CHUNK_SIZE is out of range (5242880 - 
524288000) ```
- set the cache size to 5x the optimal Tarball size
	- I changed this from 50x just to keep the same sizes
	- Did not specifically receive an error
- Change the Oracle Package DBMS_DBFS_HS in the following way (bug 33244837)
	- Line 44: PROPVAL_MAXBF_S3 CONSTANT NUMBER := 524288000; --(change from 3154728)
	- Line 45: PROPVAL_CHUNKSIZE CONSTANT VARCHAR2(50) := '5242880'; --(change from  '1048576')
	- Recompile the package
	- If you don't do this, you will get 
```ORA-20130: Illegal Parameter Value, MAX_BACKUPFILE_SIZE(3154728) SHOULD BE MORE THAN 2 x OTS(5242880) ```

## Acknowledgements ##

- Initial version: 19-AUG-2021 - Robert Pastijn, Oracle Database Product Development (PTS EMEA)
 
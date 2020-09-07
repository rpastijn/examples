# Application Containers #

## Overview ##

Short demo for application containers in Multitenant

## Requirements ##

- 19c CDB created
- sys access to the environment
- Logged in as Oracle user

## Create an Application Root ##

In this lab we will go through some examples regarding Application Containers. We will create an application container, create applications and move those applications to other containers.

Creating an Application Root is similar to creating a normal PDB, just with an extra parameter. The source of the Application Root can be an existing database or the SEED database on CDB level.

First, we set the correct environment

````
$ <copy>. oraenv</copy>
````
````
ORACLE_SID = [CDB1] ? <copy>CDB1</copy>

The Oracle base remains unchanged with value /u01/oracle
````

Now we can create a new application container by logging into SQLPlus:

````
$ <copy>sqlplus / as sysdba</copy>

SQL*Plus: Release 12.2.0.1.0 
Production on Tue Sep 19 14:18:40 2017
...
````

And execute the create statement:

````
SQL> <copy>create pluggable database APP_ROOT as APPLICATION CONTAINER admin user admin identified by admin;</copy>

Pluggable database created.
````

After creating it, we need to open the database to use it:

````
SQL> <copy>alter pluggable database APP_ROOT open;</copy>

Pluggable database altered.
````

> **Attention**: If this example is executed in an environment where Transparent Database Encryption (TDE) is active, you need to execute the following statement in every pluggable database (either APP_ROOT or APP_PDBs) before you can use it:
````
SQL> administer key management set key using tag '<PDB_NAME>' 
     force keystore identified by <your wallet password> 
     with backup using 'create_<PDB_NAME>';
````
> Example:
````
> SQL> <copy>administer key management set key using tag 'APP_ROOT' 
     force keystore identified by <your wallet password> 
     with backup using 'create_APP_ROOT';</copy>
````
> **Execute this statement inside every new Application root and every application PDB**

## Creating an application ##

In order to create an application we need to be connected to the Application Root container. When an Application is created, you need to give it a version number and a name. All statements that are executed after the initial 'BEGIN' clause of the application are captured and can be replayed in the target PDB.

````
SQL> <copy>alter session set container=APP_ROOT;</copy>

Session altered.
````

We can now define out first application with version 1.0:

````
SQL> <copy>alter pluggable database application APP01 begin install '1.0';</copy>

Pluggable database altered.
````

In this example, all hard-coded PDB names have been removed and replaced by the variables from `sys_context` so that when the statement is executed inside a new PDB, it automatically adopts to this new PDB name.

One of the things we can do now is create a new user and create some objects for this user.

````
SQL> <copy>create user app_test identified by app_test;</copy>

User created.
````
````
SQL> <copy>alter user APP_TEST quota unlimited on SYSTEM;</copy>

User altered.
````
````
SQL> <copy>create table app_test.mytable (id number);</copy>

Table created.
````
````
SQL> <copy>insert into app_test.mytable values (1);</copy>

1 row created.
````
````
SQL> <copy>commit;</copy>

Commit complete.
````

Usually a lot more statements would follow, comparable to an application install script. But for now we simulate that one table and one user is all we need in our application.

We need to finalize the application before we can push it into a application container:

````
SQL> <copy>alter pluggable database application APP01 end install;</copy>

Pluggable database altered.
````

The actual statements that were recorded are visible in the view DBA\_APP\_STATEMENTS

````
SQL> <copy>select app_statement from dba_app_statements where app_name='APP01' order by statement_id;</copy>

APP_STATEMENT
--------------------------------------------------------------------------------
SYS
BEGIN DBMS_APPLICATION_INFO.SET_MODULE('sqlplus@mydb (TNS V1-V3)', ''); END;
alter pluggable database application APP01 begin install '1.0'
create user app_test identified by values *
alter user APP_TEST quota unlimited on SYSTEM
create table app_test.mytable (id number)
insert into app_test.mytable values (1)
commit
alter pluggable database application APP01 end install

9 rows selected.
````

As you can see, each statement that we have executed will be recorded. Any statements that lead to an error (because of a typo or because of another error) are discarded. This way the APPLICATION is basically the install script you would normally run for a new installation at a new customer.

The Application can be installed in an Application PDB.

## Create application PDB ##

An Application PDB is nothing more than a regular PDB. The source could either be the regular CDB$SEED, an existing application PDB or a special Application Seed PDB. For this example we will use the regular CDB$SEED pluggable database.

The only difference between a regular PDB and an Application PDB Is the location where it is plugged in. Regular PDBs are plugged into the CDB$ROOT, Application PDBs are plugged into the Application Root Container. So, in order to create a new Application PDB, we need to connect to the Application Root Container and execute a normal Create Pluggable Database statement.

````
SQL> <copy>alter session set container=APP_ROOT;</copy>

Session altered.
````
````
SQL> <copy>create pluggable database APP_PDB1 admin user admin identified by admin;</copy>

Pluggable database created.
````
````
SQL> <copy>alter pluggable database APP_PDB1 open;</copy>

Pluggable database altered.
````

After the creation, we log into the PDB to execute some commands:

````
SQL> <copy>alter session set container=APP_PDB1;</copy>

Session altered.
````

> **Attention**: If this example is executed in an environment where Transparent Database Encryption (TDE) is active, you need to execute the following statement in every pluggable database (either APP_ROOT or APP_PDBs) before you can use it:
````
SQL> administer key management set key using tag '<PDB_NAME>' 
     force keystore identified by <your wallet password> 
     with backup using 'create_<PDB_NAME>';
````
> Example:
````
> SQL> <copy>administer key management set key using tag 'APP_PDB1' 
     force keystore identified by <your wallet password> 
     with backup using 'create_APP_PDB1';</copy>
````
> **Execute this statement inside every new Application root and every application PDB**

The Application Root works basically as a regular container database. This means that if we query the databases available in this container, it will only show us the Application PDBs and not the remaining PDBs in the regular CDB$ROOT.

````
SQL> <copy>show pdbs</copy>

CON_ID     CON_NAME                       OPEN MODE  RESTRICTED 
---------- ------------------------------ ---------- ---------- 
3          APP_ROOT                       READ WRITE NO 
4          APP_PDB1                       READ WRITE NO
````

Installing, upgrading or patching an application in an Application PDB is basically running the statements that have been captured during the initial INSTALL command in the Application Root. The running of the statements is called 'Syncing' to a particular version of the application. If no version has been specified during the SYNC process, the system will run all commands up to the latest version of the Application.

First, we can check if the user that we will be creating does not already exist in the new database. For example:

````
SQL> <copy>select username from dba_users where username='APP_TEST';</copy>

no rows selected
````

We can now install the application into the PDB:

````
SQL> <copy>alter pluggable database application APP01 SYNC;</copy>

Pluggable database altered.
````

We can now check to see if the user has been created and whether or not our data has been inserted.

````
SQL> <copy>select * from APP_TEST.MYTABLE;</copy>

        ID 
---------- 
         1 
````

## Patching a application ##

Patching means changing the application in a non-destructive way. Basically, do anything that would not result in data loss. For example, we can add a new table to the application, add a column to an existing table or add data into the existing tables. Dropping a table would not be allowed as this would mean data loss. 

Here is an example:

````
SQL> <copy>alter session set container=APP_ROOT;</copy>

Session altered.
````
````
SQL> <copy>alter pluggable database application APP01 begin patch 1.1;</copy>

Pluggable database altered.
````

We can now start making changes:

````
SQL> <copy>create table app_test.mytable2 (id number);</copy>

Table created.
````

or

````
SQL> <copy>alter table app_test.mytable add DESCRIPTION varchar2(100);</copy>

Table altered.
````

or

````
SQL> <copy>insert into app_test.mytable values (2,'Two');</copy>

1 row created.
````

````
SQL> <copy>commit;</copy>

Commit complete.
````

After all this, we finish the creation of the patch so that we can roll-out the patch:

````
SQL> <copy>alter pluggable database application APP01 end patch;</copy>

Pluggable database altered.
````

If we connect to the Application PDB, no changes are forwarded yet. We could automate this process but by default it is a manual action to sync the Application in the Application PDB with the one in the Application Root.

````
SQL> <copy>alter session set container=APP_PDB1;</copy>

Session altered.
````

````
SQL> <copy>desc app_test.mytable</copy>

Name       Null?    Type 
---------- -------- --------------- 
ID                  NUMBER 
````
````
SQL> <copy>desc app_test.mytable2</copy>

ERROR: ORA-04043: object app_test.mytable2 does not exist
````

To push the changes to the application PDB, we need to sync it with the application root:

````
SQL> <copy>alter pluggable database application APP01 sync;</copy>

Pluggable database altered.
````
````
SQL> <copy>desc app_test.mytable</copy>

Name        Null?    Type 
----------- -------- ---------------------------- 
ID                   NUMBER 
DESCRIPTION          VARCHAR2(100)
````
````
SQL> <copy>desc app_test.mytable2</copy>

Name        Null?    Type 
----------- -------- ---------------------------- 
ID                   NUMBER
````
````
SQL> <copy>select * from app_test.mytable;</copy>
ID   DESCRIPTION 
---- ------------ 
   1 
   2 Two 

2 rows selected.
````

Upgrading an application works in a similar way but requires more resources. The commands to create the list of changes and the steps to Sync are the samen. 

## Shared tables ##

In the previous labs, tables were created locally in each PDB and data was stored locally in each PDB as well. Inside Application Containers we can do more than simple creating a single table per PDB. For example; if each Application PDB needs a ZIP-code translation table, it would good to have the data stored once but accessible in each PDB.

Sharing table (meta) data between de Application Root and Application PDBs can only be done if the schema owner of the tables is a 'common' user (common user = single user available in all PDBs).

Now we can start to create our patch for our application: 

````
SQL> <copy>alter pluggable database application APP01 begin patch 2;</copy>

Pluggable database altered.
````

We are going to create a new tablespace for the common user as an example:

````
SQL> <copy>create tablespace users datafile size 100M;</copy>

Tablespace created.
````

We can now create the new common user:

````
SQL> <copy>create user app_common identified by app_common default tablespace users container=all; </copy>

User created.
````
````
SQL> <copy>alter user app_common quota unlimited on users;</copy>

User altered.
````

````
SQL> <copy>grant connect, resource to app_common;</copy>

Grant succeeded.
````

We can now also create 3 tables, each with a different sharing clause:

````
SQL> <copy>create table app_common.T_DATA sharing=data (id number);</copy>

Table created.
````
````
SQL> <copy>create table app_common.T_METADATA sharing=METADATA (id number);</copy>

Table created.
````
````
SQL> <copy>create table app_common.T_EXTENDED sharing=EXTENDED DATA (id number);</copy>

Table created.
````

After this, we can finalize the patching procedure.

````
SQL> <copy>alter pluggable database application APP01 end patch;</copy>

Pluggable database altered.
````

To see what happens with the data, we will now insert some values in each table:

````
SQL> <copy>begin
       insert into app_common.T_DATA values (1);
       insert into app_common.T_EXTENDED values (1);
       insert into app_common.T_METADATA values (1);
       commit;
     end;
     /</copy>

PL/SQL procedure successfully completed.
````

Now we can patch the application in the Application PDB and see the effect of our actions.

````
SQL> <copy>alter session set container=APP_PDB1;</copy>

Session altered.
````
````
SQL> <copy>alter pluggable database application APP01 sync;</copy>

Pluggable database altered.
````

Execute the following statements to see the result:

````
SQL> <copy>select * from app_common.T_DATA;</copy>

        ID 
---------- 
         1 
````
````
SQL> <copy>select * from app_common.T_EXTENDED;</copy>

        ID 
---------- 
         1 
````
````
SQL> <copy>select * from app_common.T_METADATA;</copy>

no rows selected
````

Here is the effect of inserting data into the shared tables in the application PDB:

````
SQL> <copy>insert into app_common.T_DATA values (2);</copy>

insert into app_common.T_DATA values (2);
ERROR at line 1: 
ORA-65097: DML into a data link table is outside an application action 
````
````
SQL> <copy>insert into app_common.T_EXTENDED values (2);</copy>

1 row created.
````
````
SQL> <copy>insert into app_common.T_METADATA values (2);</copy>

1 row created.
````
````
SQL> <copy>commit;></copy>

Commit complete.
````

As a last step, try to delete the record with ID=1 from the tables 

````
SQL> <copy>delete app_common.T_DATA where id=1; </copy>

delete app_common.T_DATA where id=1 
ERROR at line 1: 
ORA-65097: DML into a data link table is outside an application action

0 rows deleted.
````
````
SQL> delete app_common.T_EXTENDED where id=1;

0 rows deleted.
````
````
SQL> delete app_common.T_METADATA where id=1;

0 rows deleted.
````

In some tables we inserted a record with ID 2. So let us see how much of this information is available in the APP_ROOT

````
SQL> <copy>alter session set container=APP_ROOT;</copy>

Session altered.
```` 

````
SQL> <copy>select * from app_common.T_DATA;</copy>

        ID 
----------
         1
````
 
````
SQL> <copy>select * from app_common.T_EXTENDED;</copy>

        ID 
----------
         1
````
 
````
SQL> <copy>select * from app_common.T_METADATA;</copy>

        ID 
----------
         1
````

To summarize, the SHARING clause has the following options:

<table>
<tr><th>SHARING clause</th><th>DML IN</th><th>Result</th></tr>
<tr><td>DATA</td><td>APP_ROOT</td><td>Data visible in APP_PDB, cannot change data from APP PDB</th></tr>
<tr><td></td><td>APP_PDB</td><td>Not allowed, read only table</th></tr>
<tr><td>EXTENDED DATA</td><td>APP_ROOT</td><td>Data from APP_ROOT visible in APP_PDB read-only</th></tr>
<tr><td></td><td>APP_PDB</td><td>All actions allowed on local data, read-only for data from APP_ROOT</th></tr>
<tr><td>METADATA</td><td>APP_ROOT</td><td>Data from APP_ROOT not visible in APP_PDB</th></tr>
<tr><td></td><td>APP_PDB</td><td>Acts like local table</th></tr>
</table>


## Acknowledgements ##

- **Author** - Robert Pastijn, Database Product Management, PTS EMEA - March 2017
- **Added TDE lines** - Robert Pastijn, Database Product Management, PTS EMEA - April 2020 


<!---USERNAME---!>

<!---## Container queries ##

We can query across containers. For example, If we have an application for multiple stores where each store has their own PDB, we can very easily find out how many orders each store has received.

To build a small demo environment, we are going to build an Application Root called STORE_MASTER with stores (read PDBs) in various countries in the world. We are going to start with stores in Amsterdam, Paris and London.

````
 RUN THE SCRIPT 02_SETUP_STORES.SH FROM THE ~/HOL/LAB05 DIRECTORY
The result is the following
 Application Root called STORE_MASTER
o 3x Application PDB called NL, FR and UK
 Application called STORE_APP
o 1x DATA shared table called STORE_LOCATION
o 1x EXTENDED DATA shared table called STORE_PRODUCTS
o 1x METADATA shared table called STORE_SALES
 Each PDB serves a country identified by a country code.
 The actual locations for all stores are in the STORE_LOCATION table.
 All products available generic and local are in the STORE_PRODUCTS table
 All sales is in the STORE_SALES, a very simple table with store_id, sales_id, product_id and amount
We can now query the tables in the containers while connected to the Application Root.
 CONNECT TO THE APPLICATION ROOT'S USER STORE_ADMIN SQL> connect store_admin/store_admin@//localhost:1521/STORE_MASTER; Connected.
43
 QUERY ALL STORE_SALES TABLES SQL> select * from CONTAINERS(STORE_SALES); STORE_ID SALE_ID PROD_ID AMOUNT CON_ID ---------- ---------- ---------- ---------- ---------- 1 1001 1001 1 4 1 1002 1002 1 4 2 1003 1003 5 4 3 1002 1003 5 4 3 1003 1003 5 4 7 7001 1002 2 6 7 7002 1002 1 6 7 7003 1002 1 6 8 8001 1003 2 6 8 8002 1002 1 6 9 9001 1003 2 6 4 4001 1001 1 5 4 4002 1002 4 5 5 4003 1001 3 5 14 rows selected.
 QUERY ALL SALES (PRODUCT * PRICE * AMOUNT) AND TOTAL THE NUMBER PER STORE SQL> select store_id, sum(amount*price) from CONTAINERS(STORE_SALES) a, CONTAINERS(STORE_PRODUCTS) b where a.con_id = b.con_id and a.prod_id = b.prod_id group by store_id; STORE_ID SUM(AMOUNT*PRICE) ---------- ----------------- 4 17.91 5 29.85 3 69.9 7 7.96 9 13.98 2 34.95 8 15.97 1 11.94 8 rows selected.
We can also insert a value into the stores table. Assume the store with ID=4 wants to sell a new product called 'Computer' for a price of 200. The insert of this record can be done while connected to either the PDB itself or from the Application Root.
For example; if we want to add a new product, we could simply insert them in the STORE_MASTER version of the STORE_PRODUCTS table. But what happens if the store decides to sell something that cannot be available in any other country. Because we have setup the table STORE_PRODUCTS in SHARING=EXTENDED DATA we can insert a
44
new record both at the Application Root level (so all stores can see the new record) or locally in the PDB called FR so that the record is only available in the local PDB.
While querying the v$pdbs table we have discovered that the store with ID=4 is part of the PDB called FR which has CON_ID=5.
 INSERT A NEW (LOCAL) PRODUCT INTO THE STORE_PRODUCTS TABLE FOR STORE WITH ID=4 SQL> insert into containers(STORE_PRODUCTS) (con_id, prod_id, description, price) values (5 ,9901, 'Computer', 200); 1 row created. SQL> commit; Commit complete.
The same way, we could add lines from a centralized account into the various other tables, for example the STORE_SALES table if the sale was done through a world wide internet system.
If we would not have specified the CON_ID in the insert statement, all connected Application PDBs would have executed the statement and the article would have been inserted in all tables of the PDBs.

```` -->

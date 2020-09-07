# Troubleshooting Richard and Vincenze #

Steps to change the spfile in a DBaaS environment

## Requirements ##

- Access to the DBaaS system as oracle user
- Error should like a 'failed to start' because of parameter issues

## Find spfile location ##

We need to change a file that is not editable with a regular editor, the so-called spfile. We can translate this spfile (not readable) into a pfile (readable) or init.ora file. For this we need the location of the spfile which is listed in the alert log of the database.

First, set the $ORACLE_BASE parameter by running the oraenv script. This will make it easier for us to locate the diag directory we need:

````
$ <copy>. oraenv</copy>
ORACLE_SID = [CDB] ?
The Oracle base has been set to /u01/app/oracle
````

Now we can find the location of the SPFILE:

````
$ <copy>grep -R spfile $ORACLE_BASE/diag/rdbms/*/*/trace/alert*</copy>

Using parameter settings in server-side spfile /u01/app/oracle/product/19.0.0/dbhome_1/dbs/spfileCDB.ora
````

Please note the location of the spfile. It could start with `+DATA/xxxxx` or like the above example `/u01/app/oracle/product/19.0.0/dbhome_1/dbs/spfileCDB.ora`

## Generate pfile from spfile ##

Now we can generate the pfile or init.ora from the command line. We do this through the SQLPlus command line:

````
$ <copy>sqlplus / as sysdba</copy>
````

Generate the pfile in a location and with a filename you know. In this example I use the `/tmp` directory with a filename of `robert.ora`. But you can use any directory and filename as long as you remember it.

````
SQL> <copy>create pfile='/tmp/robert.ora' from spfile='</copy>your_spfile_location';

File created.
````

## Change value that causes issue ##

Now use your favourite (and available) editor to edit the new pfile and update the values. This can be done directly from SQLPlus:

````
SQL> <copy>!vi /tmp/robert.ora</copy>
````

In our example, we needed to change the `pga_aggregate_limit` to at least twice the amount of the value of the `pga_aggregate_target`. So the resulting values could look like this:

````
SQL> <copy>show parameter pga</copy>

NAME                          TYPE         VALUE
----------------------------- ------------ -----------
pga_aggregate_limit           big integer  15G
pga_aggregate_target          big integer  7680M
````

Save the file and exit the editor. 

## Test the change ##

After saving the (temporary) pfile, you can test the file to make sure that the new values work (before we change back into the spfile again):

````
SQL> <copy>startup pfile='/tmp/robert.ora'</copy>
````

If the database starts, you can continue with the next steps. If there is an error, shutdown the database and edit the .ora file again until you are able to start the database without an error.

## Generate new correct spfile ##

After the database has started succesfully, we can re-generate the spfile with the correct value in our new spfile. Again, we will use the SQLPlus prompt for this:

For a non-ASM system, use the following:

````
SQL> <copy>create spfile from pfile='/tmp/robert.ora';</copy>

File created.
````

For an ASM based system, use the following:

````
SQL> <copy>create spfile='+DATA' from pfile='/tmp/robert.ora';</copy>

File created.
````
## Test the new spfile ##

To test the new spfile, shutdown the database and restart it again without the `pfile=` clause. This automatically triggers the use of the spfile.

````
SQL> <copy>shutdown immediate;</copy>

Database closed.
Database dismounted.
ORACLE instance shut down.
````

Now we should be able to start the database without any parameters:

````
SQL> <copy>startup</copy>

ORACLE instance started.

Total System Global Area 1.6106E+10 bytes
Fixed Size                  9154008 bytes
Variable Size            2583691264 bytes
Database Buffers         1.3489E+10 bytes
Redo Buffers               24399872 bytes
Database mounted.
Database opened.
````

## Acknowledgements ##

- **Author**, Robert Pastijn, Oracle Database Product Development



$ORACLE_BASE

exit

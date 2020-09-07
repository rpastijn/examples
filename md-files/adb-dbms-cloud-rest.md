# Accessing OCI REST API from PL/SQL / APEX in ADB #

Oracle Autonomous Database now has the DBMS_CLOUD package enhanced to send REST-API requests to the Oracle database. Combining this new functionality with the existing JSON functionality allows us to leverage PL/SQL to manage and Oracle OCI environment.

https://docs.oracle.com/en/cloud/paas/autonomous-data-warehouse-cloud/user/dbms-cloud-rest.html#GUID-19B1639E-68E2-45BB-802C-817ABD0DBE88

## Prerequisites ##

This functionality can be used in both the Oracle paid and Oracle Free ADB environment.

The following needs to be available:

- Running Autonomous Database environment
  - At the time of writing, there was an issue with ADB 18c. ADB 19c worked

## Create PEM key (public and private) ##

Example from GIT BASH on Windows:

````
$ <copy>openssl genrsa -out oci_api_key.pem 2048</copy>

Generating RSA private key, 2048 bit long modulus (2 primes)
..................+++++
......................+++++
e is 65537 (0x010001)
````
````
$ <copy>openssl rsa -pubout -in oci_api_key.pem -out oci_api_key_public.pem</copy>

writing RSA key
````

Gitbash or OpenSSL https://wiki.openssl.org/index.php/Binaries

## OCI setup ##

The following steps are needed to setup the OCI environment through the console to accept requests through the REST API interface.

### Create API key entry ###

In OCI, you do not authenticate using your password but using the private key we just generated in the previous section. You need to enable OCI to authenticate you using this key. We do this in the OCI users setting under the API Keys menu.

- Click 'Add Public Key'
- Paste the generated public key (in the example the contents of the oci_api_key_public.pem file)
- Click on 'Add'

If all was correct, a fingerprint was generated for your key. We need this fingerprint so copy-paste it in a notepad or similar.

### Gather information from OCI tenancy to manage ###

The following information needs to be gathered:

- Tenancy name `<tenancy_name>`
- Tenancy OCID `ocid1.tenancy.oc1..aaaaaaaa3fcdss2...kwpfbloh54cz7azsyjqgbl5qujnga`
- OCI user username `<username>`
- OCI user OCID `ocid1.user.oc1..aaaaaaaaw4....kxr6tkvqmflgmfihfigrnkp6dmkj7q`

The following information should already be available after the previous section:

- API KeyFingerprint:  `5d:6d:3f:3....9:ec:69:df:f5`

## Optional APEX workspace ##

The actual code to execute will run from a database environment. For this, you would need to setup SQL*Developer access or SQLPLus access to the Autonomous Database. In this demo, a APEX environment will be used so nothing needs to be downloaded to the clients laptop but all (SQL) commands can be accessed and used from any SQL prompt.

### Create a new workspace in the APEX ADB ###

When the ADB creation has finished, do the following:

- Navigate to the Developer console and start the ADB APEX environment
- Login as ADMIN user and create a new Workspace
	- In this example, the workspace will be called OCI
- Log out of the ADMIN environment and login to the OCI workspace

## Setup OCI credentials ##

### Grant execute on DBMS_CLOUD to new OCI workspace/user ###

- Through the ADB console, start the SQL Developer Web
	- or use your preferred way to connect to the Autonomous Database
- Login as the ADMIN user
- Grant the required rights to the (APEX) user (our case 'OCI')

````
SQL> <copy>GRANT EXECUTE ON DBMS_CLOUD TO OCI;</copy>
````

When using APEX for the remaining example, you can now close the SQLDeveloper Web as we do not need it anymore. 

### Create CLOUD credentials under users schema ###

Navigate back to your (APEX) environment (logged in as user who will execute the commands, in our case OCI) and open a SQL Commands screen for executing SQL statements.

Enter the following PL/SQL block, replacing the values in there by the values you gathered.

````
<copy>begin
  DBMS_CLOUD.CREATE_CREDENTIAL ( credential_name => 'OCI_CRED',
                                  user_ocid       => 'ocid1.user.oc1..aaaaaaaaw4tkvowdatrjqlnijwmztykxr6tkddmflgmfihfigrnkp6dmkj7q',
                                  tenancy_ocid    => 'ocid1.tenancy.oc1..aaaaaaxx3fcdss2mnuqwijt4yrmkz4wkwpfbloh54cz7azsyjqgbl5quaaga',
                                  fingerprint     => '5d:6d:3f:3c:97:0d:d8:b3:f5:ef:5f:e9:ec:69:df:f5',
                                  private_key     => 'MIIEowIBAAK.....yqql3jC2e+DmvOYjru1kurK0iQB8MJr6z2j1Jx4NTKfZZTuY2tf4e8Ai8Yf3VrAk5mU1ehRQa8rirujoDuOECyzZqBCXaP.....Ltu9KPRvdq3X/'||
                                                     'z/MaDI4j7Yu0j5X0sa.....GIGHJewso3dZ8QyI/A7c7TDebXjK2BSJSxbh+AnCs92UpKu6GJpo9SSeM3n0sIg5iQ.....YWTCqE4KPr9aWSUD7kej9teoTRYEmUpPtD'||
                                                     'JvqywZv8MTVTiqc94oG.....RpSTXb92LIHiJVZuXNkqWgEb8H+dC+dY8Rfrryi3Uejtosjo9bA+BjBdIRm6Y56ojNEGdacbXOdvfwIDAQABAoIBADEtrag4C+ANmBAY'||
                                                     'jGojQwEm4jYofvpQcTIpVuvSGAoU5OTyzEm2hJ6K5i/T/Yn0nk+mGY7xJP6zMcSd0cnjHaSUkpB8zjDynidFdpiCgnJHs8eqj5PHeve2+jLnZr489i6yC1W5mp66++g2'||
                                                     ...
                                                     '1qXfVwKBgEXa/hP0cBiTJcxZeqUvyMH3LR7t6Zx9U4uReNP0JsENntbk6Kjz8pgI2aPfQ39V2eZzBCO8S6lJbdtGV6HNCLTNMNtFfjilq8EVu5RcoQXhZGOiasz8opVD'||
                                                     'KIiNUocZ1Yb8mEqV6bHJnZMpRibvoQ3m4eBV2+GMhnqqg0yGuWhy');
end;</copy>
````

The goal is to have your private pem key inside this statement. Please strip the values `-----BEGIN RSA PRIVATE KEY-----` and 
`-----END RSA PRIVATE KEY-----` from the private .pem file. 

> **Be aware:** if your source file does not start/ends with exactly the above lines, you do not have a PEM key but a RSA key.

Execute the PL/SQL command, you should get the following result:

    Statement processed.
    
    1.17 seconds

## Initial test to retrieve information ##

Full REST API information can be found here:

https://docs.cloud.oracle.com/en-us/iaas/api/

For this example we will list all Autonomous databases in the root compartment. The REST API for this can be found here: 

https://docs.cloud.oracle.com/en-us/iaas/api/#/en/database/20160918/AutonomousDatabase/ListAutonomousDatabases

````
<copy>declare
  v_response     DBMS_CLOUD_TYPES.resp;
  v_result       clob;
  v_url          varchar2(200);
  v_region       varchar2(200) := 'eu-frankfurt-1';
  v_compartment  varchar2(200) := '<your tenancy OCID or compartment OCID>';
  v_credential   varchar2(200) := 'OCI_CRED';
begin
  v_url := 'https://'||v_region||'/20160918/autonomousDatabases/?compartmentId='||v_compartment;
  v_response := DBMS_CLOUD.SEND_REQUEST( 
       credential_name => v_credential,
       uri => v_url,
       method => DBMS_CLOUD.METHOD_GET);
  v_result := DBMS_CLOUD.GET_RESPONSE_TEXT( resp => v_response );
  dbms_output.put_line(v_result);
end;</copy>
````

In this initial example, I used the tenancy as compartmentID. Unless you have databases running in the root compartment, you will get no output like:

````
[]

Statement processed.
1.08 seconds
````

If you change the compartmentId into the compartment where you created this ADB/APEX environment (and make sure you have the correct region), you can get more information. Example:

````
[{"additionalDatabaseStatus":null,"autonomousContainerDatabaseId":null,"compartmentId":"ocid1.compartment.oc1..aaaaaaaaja5hmjbi6wxhyfsbk4ysztgzig5uvx7zljgwnenywjex4xmkcolq","connectionStrings":
    (etc etc)
    000Z","timeReclamationOfFreeAutonomousDatabase":null,"usedDataStorageSizeInTBs":1,"whitelistedIps":null}]
````

### Create a function ###

Using the JSON_TABLE command, we can process the JSON output and display it in a relational way. The easiest way is if the output is returned from a FUNCTION that we can use in a select statement. We will use the same example (using your own compartment details) as before:

````
<copy>create function F_MY_ADB return clob is
  v_response     DBMS_CLOUD_TYPES.resp;
  v_result       clob;
  v_url          varchar2(200);
  v_region       varchar2(200) := 'eu-frankfurt-1';
  v_compartment  varchar2(200) := '<your tenancy OCID or compartment OCID>';;
  v_credential   varchar2(200) := 'OCI_CRED';
begin
  v_url := 'https://database.'||v_region||'.oraclecloud.com/20160918/autonomousDatabases/?compartmentId='||v_compartment;
  v_response := DBMS_CLOUD.SEND_REQUEST( 
       credential_name => v_credential,
       uri => v_url,
       method => DBMS_CLOUD.METHOD_GET);
  v_result := DBMS_CLOUD.GET_RESPONSE_TEXT( resp => v_response );
  return(v_result);
end;</copy>
````

After you get `Function created.` you can execute the following query:

````
SQL> <copy>select F_MY_ADB from dual;</copy>
````

It should give you the same response as the initial PL/SQL block.

## Retrieving and scaling ADB through DBMS_CLOUD ##

### Retrieve ADB information ###

With the previous example function, we can retrieve information from our OCI environment and display this in a relational way using the JSON_TABLE function:

````
SQL> <copy>select * from JSON_TABLE( F_MY_ADB,
    '$[*]' columns (id, displayName, cpuCoreCount, isAutoScalingEnabled));</copy>

ID                                                       DISPLAYNAME  CPUCORECOUNT  ISAUTOSCALINGENABLED
-------------------------------------------------------  -----------  ------------  --------------------
ocid1.autonomousdatabase.oc1.eu-frankfurt-1.abtheljrxx1  OML	      1             true
ocid1.autonomousdatabase.oc1.eu-frankfurt-1.abtheljsxx2  UPV11        1             true
ocid1.autonomousdatabase.oc1.eu-frankfurt-1.abtheljsxx3  GGTARGET     1             false
ocid1.autonomousdatabase.oc1.eu-frankfurt-1.abtheljqxx4  ARCOS        2             true
ocid1.autonomousdatabase.oc1.eu-frankfurt-1.abtheljtxx5  ARCOS12      1             false
ocid1.autonomousdatabase.oc1.eu-frankfurt-1.abtheljfxx6  ARCOS11      1             true
ocid1.autonomousdatabase.oc1.eu-frankfurt-1.abtheljhxx7  TESTDB       1             false
````

### Change scaling and OCPU for ADB ###

According to the REST manual, scaling needs a PUT request (https://docs.cloud.oracle.com/en-us/iaas/api/#/en/database/20160918/AutonomousDatabase/UpdateAutonomousDatabase). We can use the following code to change the OCPUs of an ADB instance:

````

<copy>create or replace procedure scale_ADB (p_OCID varchar2, 
                                       p_OCPU in number, 
                                       p_autoscale varchar2) is

  v_response     DBMS_CLOUD_TYPES.resp;
  v_result       clob;
  v_body         clob;
  v_url          varchar2(200);
  v_region       varchar2(200) := 'eu-frankfurt-1';
  v_compartment  varchar2(200) := '<your tenancy OCID or compartment OCID>';
  v_credential   varchar2(200) := 'OCI_CRED';


  
begin

  v_url := 'https://database.'||v_region||'.oraclecloud.com/20160918/autonomousDatabases/'||p_OCID;

  v_body := JSON_OBJECT('cpuCoreCount'         VALUE to_char(p_OCPU),
                        'isAutoScalingEnabled' VALUE p_autoscale);

  v_response := DBMS_CLOUD.send_request( credential_name => v_credential,
                                         uri             => v_url,
                                         method          => DBMS_CLOUD.METHOD_PUT,
                                         body            => UTL_RAW.cast_to_raw(v_body)
                                       );
                                         
end;
/</copy>
````

After creating the procedure, you can scale the ADB instances with a call to the procedure:

````
SQL> <copy>begin
       scale_adb( p_OCID      => '<OCID of the DB instance>,
                  p_OCPU      => 4, 
                  p_autoscale => 'false');
     end;
/</copy>

procedure executed.
````
## Acknowledgements ##

- **Initial version** - Robert Pastijn, Oracle Product Development, 29-APR-2020
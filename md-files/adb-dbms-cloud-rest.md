# Accessing OCI REST API from PL/SQL / APEX in ADB #

Oracle Autonomous Database now has the DBMS_CLOUD package enhanced to send REST-API requests to the Oracle database. Combining this new functionality with the existing JSON functionality allows us to leverage PL/SQL to manage and Oracle OCI environment.

https://docs.oracle.com/en/cloud/paas/autonomous-data-warehouse-cloud/user/dbms-cloud-rest.html#GUID-19B1639E-68E2-45BB-802C-817ABD0DBE88

## Prerequisites ##

This functionality can be used in both the Oracle paid and Oracle Free ADB environment.

The following needs to be available:

- Running Autonomous Database environment

## OCI Setup option 1 : API Key

When using an API key, we create an option for the database to execute commands as if the user would execute them on the console. The following steps are needed to setup the OCI environment through the console to accept requests through the REST API interface with the same rights as a user:

### Create API key entry ###

In OCI, you do not authenticate in REST API calls using your (console) password but using a private key. You need to enable OCI to authenticate you using this key. We do this in the OCI users setting under the API Keys menu.

- Navigate to the user who will execute the API calls
  - Click right-top of the console on the user icon next tot the globe
  - Select the username
- Select 'API Key' in the menu under Resources
- Click 'Add API Key'
- Either upload an existing public key or select Generate API Key Pair
  - Make sure you download the private key when you generate a new one
- Click on 'Add'

If all was correct, Configuration File Preview will be generated. Copy the preview and save it on a notepad (or similar) as we need the information in a next step.

Example:

````
[DEFAULT]
user=ocid1.user.oc1..aaaaxxxxtxfkn7zrmoors......5bocukvpwbmq7xxxyz
fingerprint=3e:b0:4b:b3:15:f2:14:32:8e:15:d4:aa:7b:26:3a:72
tenancy=ocid1.tenancy.oc1..aaaaaaaafjlfjjg73442jfngtt7fo27tuejsfqbybzrq
region=eu-amsterdam-1
key_file=<path to your private keyfile> # TODO
````

### Optional: Grant execute on DBMS_CLOUD to database user 

If you are not using the ADMIN user in the Autonomous Database, the user needs to be given permission to execute comments using the DBMS_CLOUD package.

- Through the ADB console, navigate to Database Actions
- Start a new SQL window (left top icon)
	- or use your preferred way to connect to the Autonomous Database
- You will be logged in as the ADMIN user automatically
- Grant the required rights to the user (our case 'OCI')

````
SQL> <copy>GRANT EXECUTE ON DBMS_CLOUD TO OCI;</copy>
````

### Create CLOUD credentials under the users schema ###

Login to the SQL window in Database Actions (if not logged in already). 

- Through the ADB console, navigate to Database Actions
- Start a new SQL window (left top icon)
	- or use your preferred way to connect to the Autonomous Database
- You will be logged in as the ADMIN user automatically

> If you are logged in as ADMIN user but would like to execute the DBMS\_CLOUD commands as another user (in our case the OCI user), choose Logout from the menu currently labelled as ADMIN. Then, log in with the username and password of the user who will execute the DBMS_CLOUD commands.

Copy and paste the following PL/SQL block in your SQL web page. 

````
<copy>begin
  DBMS_CLOUD.CREATE_CREDENTIAL ( credential_name  => 'OCI_CRED',
                                  user_ocid       => '[user_ocid]',
                                  tenancy_ocid    => '[tenancy_ocid]',
                                  fingerprint     => '[fingerprint]',
                                  private_key     => '[private_key]' 
                                );
end;
</copy>
````

Replace the user_ocid, tenancy_ocid and fingerprint values for the values in the Configuration Preview file you saved earlier. The private_key value should contain all characters from your (downloaded) PEM private key file used when creating the API key.

Example private_key entry: 

````
private_key    => 'MIIEowIBAAK.....yqql3jC2e+DmvOYjru1kurK0iQB8MJr6z2j1Jx4NTKfZZTuY2tf4e8Ai8Yf3VrAk5mU1ehRQa8rirujoDuOECyzZqBCXaP.....Ltu9KPRvdq3X/'||
                  'z/MaDI4j7Yu0j5X0sa.....GIGHJewso3dZ8QyI/A7c7TDebXjK2BSJSxbh+AnCs92UpKu6GJpo9SSeM3n0sIg5iQ.....YWTCqE4KPr9aWSUD7kej9teoTRYEmUpPtD'||
                  'JvqywZv8MTVTiqc94oG.....RpSTXb92LIHiJVZuXNkqWgEb8H+dC+dY8Rfrryi3Uejtosjo9bA+BjBdIRm6Y56ojNEGdacbXOdvfwIDAQABAoIBADEtrag4C+ANmBAY'||
                  'jGojQwEm4jYofvpQcTIpVuvSGAoU5OTyzEm2hJ6K5i/T/Yn0nk+mGY7xJP6zMcSd0cnjHaSUkpB8zjDynidFdpiCgnJHs8eqj5PHeve2+jLnZr489i6yC1W5mp66++g2'||
                  ...
                  '1qXfVwKBgEXa/hP0cBiTJcxZeqUvyMH3LR7t6Zx9U4uReNP0JsENntbk6Kjz8pgI2aPfQ39V2eZzBCO8S6lJbdtGV6HNCLTNMNtFfjilq8EVu5RcoQXhZGOiasz8opVD'||
                  'KIiNUocZ1Yb8mEqV6bHJnZMpRibvoQ3m4eBV2+GMhnqqg0yGuWhy');
````

The goal is to have your private pem key inside this statement. Please strip the values `-----BEGIN RSA PRIVATE KEY-----` and 
`-----END RSA PRIVATE KEY-----` from the private .pem file. 

> **Be aware:** if your source file does not start/ends with exactly the above lines (BEGIN RSA KEY), you do not have a correct PEM key.

Execute the PL/SQL command, you should get the following result:

    Statement processed.
    
    1.17 seconds

After this step, you have created a credential to send REST API calls to the Oracle OCI cloud. You can skip the next chapter 'OPTION 2' and continue to the Initial test to retrieve information.

## OCI Setup option 2 : Resource Principal

When using Resource Principal, we grant the database directly the right to execute commands in the OCI Cloud instead of going through a user. This makes it easier to only grant those rights the database needs.

Before we can use the DBMS_CLOUD, some setup steps need to be executed

### Create Dynamic Group ###

Granting rights in OCI can only be done to Groups. Because a database is not an OCI user, we need to create something called a Dynamic Group that contains the database as a resource.

Navigate to the OCI console as a user with administration privileges

- Identity and Security
- Dynamic Groups
	- If the option Dynamic Groups is not visible, choose 'Domains' first and select the Default domain. After this, the option for dynamic groups should be visible.
- Add a new Dynamic Group
- Fill in a suitable name and description
	- We need the Dynamic Group name in the next section
- Add the following rule

````
resource.id = '[your autonomous database ocid]'
````

Make sure to replace _[your autonomous database ocid]_ with the ocid of your autonomous database. After this, save the Dynamic Group.
  
### Create Policy ###

Now that we have correlated the Autonomous Database to a Dynamic Group, we can write the policy to grant this group the rights to execute commands.

- Navigate to Identity & Security in the Menu
- Choose the Policies option
- Create a new policy and give it a suitable name and description
- Choose the compartment in which the autonomous database is created
- Select 'Show Manual Editor' by switching the switch
- Copy and paste the following new policy line

````
allow dynamic-group [dynamic group name] to manage autonomous-databases in compartment [your compartment]
````

In this example, we grant the right to manage all autonomous databases in the compartment to the Autonomous Database Instance. In production, add or remove rights in the policy file as needed according to the documentation on policy options.

After saving the file, we are ready to go to the database for the final step.

### Grant right to use Resource Principal ###

Before any user can use the rights through the dynamic group, the ADMIN user needs to enable Dynamics Groups and grant usage to the individual database users. For this, start a SQL window in the Database Actions through your console

- Navigate to the console of your Autonomous Database
- Choose Database Actions
- Choose SQL (left top) to start a SQL web window

Once logged in, execute the following command to enable Resource Principal usage in your database:

```
EXEC DBMS_CLOUD_ADMIN.ENABLE_RESOURCE_PRINCIPAL();
```

If another user than the ADMIN user will execute the commands, an additional command needs to be executed to grant the right to the user:

```
EXEC DBMS_CLOUD_ADMIN.ENABLE_RESOURCE_PRINCIPAL(username => '[your db user]');
```

After executing the first (and optionally the second) command, you are ready to start sending REST API calls to the OCI cloud.

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
  v_compartment  varchar2(200) := '[your tenancy OCID or compartment OCID]';
  v_credential   varchar2(200) := 'OCI_CRED'; -- or OCI$RESOURCE_PRINCIPAL
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
  v_credential   varchar2(200) := 'OCI_CRED'; -- or OCI$RESOURCE_PRINCIPAL
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
  v_credential   varchar2(200) := 'OCI_CRED'; -- or OCI$RESOURCE_PRINCIPAL


  
begin

  v_url := 'https://database.'||v_region||'.oraclecloud.com/20160918/autonomousDatabases/'||p_OCID;

  v_body := JSON_OBJECT('computeCount'         VALUE to_char(p_OCPU),
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
- **Updated with OCI_RESOURCE_PRINCIPAL** Robert Pastijn, 20-MAR-2023
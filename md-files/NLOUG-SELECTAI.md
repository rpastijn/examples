#DEMO NLOUG Select AI

## Prepare before the session

```
GRANT EXECUTE on DBMS_CLOUD_AI to DEMO;
grant execute on DBMS_CLOUD to your_schema_name;
```

-- Allow access to OpenAI host (OCI enabled by default)

```
BEGIN  
    DBMS_NETWORK_ACL_ADMIN.APPEND_HOST_ACE(
         host => 'api.openai.com',
         ace  => xs$ace_type(privilege_list => xs$name_list('http'),
                             principal_name => 'DEMO',
                             principal_type => xs_acl.ptype_db)
   );
END;
/
```

## Beginning of demonstration

-- Start SQLcl

```
<copy>
cd ~/iCloud/NLOUG
sql /nolog
</copy>
```

-- Enable Cloudconfig

```
<copy>
set cloudconfig Wallet_NLOUG.zip
</copy>
```

-- Show available TNS entries

```
<copy>
show tns
</copy>
```

-- Connect to the instance

```
<copy>
connect demo@NLOUG_low
</copy>
```

-- Create an OPENAI credential

```
<copy>
begin
  DBMS_CLOUD.CREATE_CREDENTIAL(
    credential_name => 'OPENAI_CREDENTIAL',
    username        => '<username>',
    password        => '<OpenAI Project Key>');
end;
/
</copy>
```

-- Create an OCI credential

```
<copy>
begin
  DBMS_CLOUD.CREATE_CREDENTIAL (
      credential_name => 'OCI_CREDENTIAL',
      user_ocid       => '<user_ocid>',
      tenancy_ocid    => '<tenancy_ocid>',
      fingerprint     => '<fingerprint>',
      private_key     => '<private key>');

end;
/
</copy>
```

-- Check the created credential

```
<copy>
select credential_name
from user_credentials;
</copy>
```

-- Create an OCI profile

```
<copy>
begin
  DBMS_CLOUD_AI.CREATE_PROFILE (
     profile_name => 'OCI_DEMO',
     attributes   => '{"provider": "oci",
                       "credential_name": "OCI_CREDENTIAL",
                       "object_list": [{"owner" : "DEMO"},
                                       {"name": "customers"},
                                       {"name": "products"},
                                       {"name": "orders"},
                                       {"name": "order_lines"}],
                       "region" : "eu-frankfurt-1"
                        }');
end;
/
</copy>
```

-- Create an OPENAI profile

```
<copy>
begin
  DBMS_CLOUD_AI.CREATE_PROFILE (
     profile_name => 'OAI_DEMO',
     attributes   => '{"provider": "openai",
                       "credential_name": "OPENAI_CREDENTIAL",
                       "object_list": [{"owner": "DEMO"}, 
                                       {"name": "customers"},
                                       {"name": "products"},
                                       {"name": "orders"},
                                       {"name": "order_lines"}]
                        }');
end;
/
</copy>
```
-- Show the created profiles

```
<copy>
select profile_name, status
from user_cloud_ai_profiles;
</copy>
```

-- Show some attributes

```
<copy>
select profile_name, attribute_name, attribute_value
from user_cloud_ai_profile_attributes
where profile_name='OCI_DEMO';
</copy>
```

-- Set the profile

```
<copy>
exec dbms_cloud_ai.set_profile('OCI_DEMO');
</copy>
```

-- Do the test

```
<copy>
select ai how many customers are in the system;
select ai how many orders were shipped last week;
select ai how many orders were only partially shipped;
select ai how many orders took more than a week between ordering and shipping;
select ai and what is the average time between shipment and order;
</copy>
```

-- Using showsql, explainsql

```
<copy>
select ai showsql how many orders took more than a week between ordering and shipping;
</copy>
```

```
<copy>
select ai explainsql how many orders took more than a week between ordering and shipping;
</copy>
```

-- Using Narrate

```
<copy>
select ai narrate how many orders took more than a week between ordering and shipping;
</copy>
```

-- Using Narrate with different language

```
<copy>
select ai narrate how many orders took more than a week between ordering and shipping in French;
</copy>
```

```
<copy>
select ai narrate Hoeveel orders hadden meer dan een week nodig tussen besteldatum en verzenddatum, antwoord in het Frans;
</copy>
```

-- Using chat with options

```
<copy>
select ai chat how many orders took more than a week between ordering and shipping, give 3 reasons why;
</copy>
```

```
<copy>
select ai chat how many orders took more than a week between ordering and shipping with 3 reasons in the form of a haiku in Dutch;
</copy>
```

### Difference between LLM

-- Start with the OCI version

```
<copy>
exec DBMS_CLOUD_AI.SET_PROFILE('OCI_DEMO');
</copy>
```

-- Execute query

```
<copy>
select ai what is our best selling product;
select ai showsql what is our best selling product;
</copy>
```

-- Switch to OpenAI

```
<copy>
exec DBMS_CLOUD_AI.SET_PROFILE('OAI_DEMO');
</copy>
```

-- Execute query

```
<copy>
select ai what is our best selling product;
select ai showsql what is our best selling product;
</copy>
```

### Using comments

```
<copy>
select ai how much money is still owed;
select ai showsql how much money is still owed;

select ai how many customers have open payments;
select ai showsql how many customers have open payments;

select ai how many customers have payment status open;
select ai showsql how many customers have payment status open;

select ai how many customers have payment status OPEN;
select ai showsql how many customers have payment status OPEN;
</copy>
```

-- Add comment

```
<copy>
comment on column orders.payment_status is 'If the value is OPEN, item is unpaid and still needs to be paid.';
</copy>
```

-- New profile with comments enabled

```
<copy>
begin
  DBMS_CLOUD_AI.CREATE_PROFILE (
     profile_name => 'OCI_DEMO_COMMENTS',
     attributes   => '{"provider": "oci",
                       "credential_name": "OCI_CREDENTIAL",
                       "object_list": [{"owner" : "DEMO"},
                                       {"name": "customers"},
                                       {"name": "products"},
                                       {"name": "orders"},
                                       {"name": "order_lines"}],
                       "region": "eu-frankfurt-1",
                       "comments": "true"
                        }');
end;
/
</copy>
```

-- Enable the new profile

```
<copy>
exec dbms_cloud_ai.set_profile('OCI_DEMO_COMMENTS');
</copy>
```
-- Run the queries again

```
<copy>
select ai how much money is still owed;
select ai showsql how much money is still owed;

select ai how many customers have open payments;
select ai showsql how many customers have open payments;

select ai how many customers have payment status open;
select ai showsql how many customers have payment status open;

select ai how many customers have payment status OPEN;
select ai showsql how many customers have payment status OPEN;
</copy>
```

### Interact with data

-- Switch to OpenAI

```
<copy>
exec DBMS_CLOUD_AI.SET_PROFILE('OAI_DEMO_COMMENTS');
</copy>
```

-- Ask a question

```
<copy>
select ai how many customers live in Kansas;
select ai and how many of them have unpaid items;
select ai showsql and how many of them have unpaid items;
</copy>
```

-- Create profiles with conversation

```
<copy>
begin
  DBMS_CLOUD_AI.CREATE_PROFILE (
     profile_name => 'OCI_DEMO_CONVERSATION',
     attributes   => '{"provider": "oci",
                       "credential_name": "OCI_CREDENTIAL",
                       "object_list": [{"owner" : "DEMO"},
                                       {"name": "customers"},
                                       {"name": "products"},
                                       {"name": "orders"},
                                       {"name": "order_lines"}],
                       "region": "eu-frankfurt-1",
                       "comments": "true",
                       "conversation": "true"
                        }');
end;
/

begin
  DBMS_CLOUD_AI.CREATE_PROFILE (
     profile_name => 'OAI_DEMO_CONVERSATION',
     attributes   => '{"provider": "oci",
                       "credential_name": "OCI_CREDENTIAL",
                       "object_list": [{"owner" : "DEMO"},
                                       {"name": "customers"},
                                       {"name": "products"},
                                       {"name": "orders"},
                                       {"name": "order_lines"}],
                       "comments": "true",
                       "conversation": "true"
                        }');
end;
/
</copy>
```

-- Switch to the new profile with OAI

```
<copy>
exec DBMS_CLOUD_AI.SET_PROFILE('OAI_DEMO_CONVERSATION');
</copy>
```

-- Try query again

```
<copy>
select ai how many customers live in Kansas;
select ai and how many of them have unpaid items;
select ai showsql and how many of them have unpaid items;
</copy>
```
-- Switch to the new profile with OCI

```
<copy>
exec DBMS_CLOUD_AI.SET_PROFILE('OCI_DEMO_CONVERSATION');
</copy>
```

-- Try query again

```
<copy>
select ai how many customers live in Kansas;
select ai and how many of them have unpaid items;
select ai showsql and how many of them have unpaid items;
</copy>
```

### Using PL/SQL instead of SQL

-- Enable serveroutput

```
<copy>
set serveroutput on
</copy>
```

-- Execute code

```
<copy>
begin
  dbms_output.put_line( DBMS_CLOUD_AI.GENERATE (
                          prompt       => 'How many customers do we have',
                          profile_name => 'OCI_DEMO',
                          action       => 'showsql'));
end;
/
</copy>
```
-- Doing tasks

```
<copy>
select ai showsql which customers have open payments, how much and how long are they outstanding;
</copy>
```

-- Original outcome

```
<copy>
select c."NAME" AS Customer_Name, 
       o."ORDER_DATE" AS Order_Date, 
       SYSTIMESTAMP - o."ORDER_DATE" AS Days_Outstanding, 
       ol."NUMBER_OF_ITEMS" * p."PRICE" AS Total_Amount 
FROM "DEMO"."CUSTOMERS" c
  JOIN "DEMO"."ORDERS" o ON c."CUSTOMER_ID" = o."CUSTOMER_ID"
  JOIN "DEMO"."ORDER_LINES" ol ON o."ORDER_ID" = ol."ORDER_ID"
  JOIN "DEMO"."PRODUCTS" p ON ol."PRODUCT_ID" = p."PRODUCT_ID"
WHERE o."PAYMENT_STATUS" = 'OPEN');   
</copy>
```

-- Add task and task_lines

```
<copy>
create or replace view outstanding_payments as
SELECT JSON_OBJECT(*) as MYJSON from (
  select 'Create email that will persuade the customer to pay their outstanding payments' as task,
         '1. Strongly encourage the customer to do this but only if the days outstanding is more than 100. '||
         '2. Specify the consequences when they will not comply. '||
         '3. Threaten them in a nice way that next orders will not be accepted but only if the days outstanding is more than 350' as task_rules,
         c."NAME" AS Customer_Name, 
         o."ORDER_DATE" AS Order_Date, 
         SYSTIMESTAMP - o."ORDER_DATE" AS Days_Outstanding, 
         ol."NUMBER_OF_ITEMS" * p."PRICE" AS Total_Amount
  FROM "DEMO"."CUSTOMERS" c
  JOIN "DEMO"."ORDERS" o ON c."CUSTOMER_ID" = o."CUSTOMER_ID"
  JOIN "DEMO"."ORDER_LINES" ol ON o."ORDER_ID" = ol."ORDER_ID"
  JOIN "DEMO"."PRODUCTS" p ON ol."PRODUCT_ID" = p."PRODUCT_ID"
  WHERE o."PAYMENT_STATUS" = 'OPEN');    
</copy>
```

-- Use this in a script

```
<copy>
declare
  cursor c_customers is 
       SELECT MYJSON from OUTSTANDING_PAYMENTS where rownum < 4;
  v_result clob;
begin
  for x in c_customers loop
    v_result :=  DBMS_CLOUD_AI.GENERATE (
                      prompt => x.myjson,
                      profile_name => 'OCI_DEMO',
                      action => 'chat');
    dbms_output.put_line(v_result);
  end loop;
end;
/
</copy>
```
### Cleanup

```
<copy>
declare
  cursor c_profiles is select profile_name from user_cloud_ai_profiles;
  cursor c_credentials is select credential_name from user_credentials;
  cursor c_views is select view_name from user_views;
  cursor c_plsql is select distinct name, type from user_source
                      where type in ('FUNCTION','PROCEDURE');
begin
  for x in c_profiles loop
    dbms_cloud_ai.drop_profile(x.profile_name);
  end loop;
  
  for x in c_credentials loop
    dbms_cloud.drop_credential(x.credential_name);
  end loop;
  
  for x in c_views loop
    execute immediate ('drop view '||x.view_name);
  end loop;

  for x in c_plsql loop
    execute immediate ('drop '||x.type||' '||x.name);
  end loop;

end;
/
</copy>
```
  
  

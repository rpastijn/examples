# Setup email in DBaaS #

How to setup email in Autonomous and non autonomous DBaaS

## Introduction ##

Following two links describe how to setup  UTL_SMTP and APEX_MAIL into Autonomous DB. This lab describes on how to do the same in DBaaS (only needs a wallet extra)..

- https://blogs.oracle.com/datawarehousing/how-to-send-an-email-using-utl_smtp-in-autonomous-database
- https://blogs.oracle.com/apex/sending-email-from-your-oracle-apex-app-on-autonomous-database

## Setup the OCI part of things ##

### Generate user details for STM sending ###

Userid = ocid1.user.oc1..aaa<etcetcetc>@ocid1.tenancy.oc1..aaaaaaaafj37<etcetcetc>.eo.com
Password = <generated_password>

Please keep in mind that the authorized sender is region specific. This example was setup for Frankfurt.

### Add approved sender for UTL_SMTP.MAIL ####

created sendmail@oraclepts.nl in the OCI console

## Setup the database part of things ##

### Find root certificate for the SMTP server ###

To find root CA

````
# <copy>openssl s_client -crlf -quiet -connect smtp.email.eu-frankfurt-1.oci.oraclecloud.com:587 -starttls smtp</copy>
````

For Oracle they are currently Digicert CA:

````
<copy>https://global-root-ca.chain-demos.digicert.com/info/index.html</copy>
````

````
<copy>-----BEGIN CERTIFICATE-----
MIIDrzCCApegAwIBAgIQCDvgVpBCRrGhdWrJWZHHSjANBgkqhkiG9w0BAQUFADBh
MQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYDVQQLExB3
d3cuZGlnaWNlcnQuY29tMSAwHgYDVQQDExdEaWdpQ2VydCBHbG9iYWwgUm9vdCBD
QTAeFw0wNjExMTAwMDAwMDBaFw0zMTExMTAwMDAwMDBaMGExCzAJBgNVBAYTAlVT
MRUwEwYDVQQKEwxEaWdpQ2VydCBJbmMxGTAXBgNVBAsTEHd3dy5kaWdpY2VydC5j
b20xIDAeBgNVBAMTF0RpZ2lDZXJ0IEdsb2JhbCBSb290IENBMIIBIjANBgkqhkiG
9w0BAQEFAAOCAQ8AMIIBCgKCAQEA4jvhEXLeqKTTo1eqUKKPC3eQyaKl7hLOllsB
CSDMAZOnTjC3U/dDxGkAV53ijSLdhwZAAIEJzs4bg7/fzTtxRuLWZscFs3YnFo97
nh6Vfe63SKMI2tavegw5BmV/Sl0fvBf4q77uKNd0f3p4mVmFaG5cIzJLv07A6Fpt
43C/dxC//AH2hdmoRBBYMql1GNXRor5H4idq9Joz+EkIYIvUX7Q6hL+hqkpMfT7P
T19sdl6gSzeRntwi5m3OFBqOasv+zbMUZBfHWymeMr/y7vrTC0LUq7dBMtoM1O/4
gdW7jVg/tRvoSSiicNoxBN33shbyTApOB6jtSj1etX+jkMOvJwIDAQABo2MwYTAO
BgNVHQ8BAf8EBAMCAYYwDwYDVR0TAQH/BAUwAwEB/zAdBgNVHQ4EFgQUA95QNVbR
TLtm8KPiGxvDl7I90VUwHwYDVR0jBBgwFoAUA95QNVbRTLtm8KPiGxvDl7I90VUw
DQYJKoZIhvcNAQEFBQADggEBAMucN6pIExIK+t1EnE9SsPTfrgT1eXkIoyQY/Esr
hMAtudXH/vTBH1jLuG2cenTnmCmrEbXjcKChzUyImZOMkXDiqw8cvpOp/2PV5Adg
06O/nVsJ8dWO41P0jmP6P6fbtGbfYmbW0W5BjfIttep3Sp+dWOIrWcBAI+0tKIJF
PnlUkiaY4IBIqDfv8NZ5YBberOgOzW6sRBc4L0na4UU+Krk2U886UAb3LujEV0ls
YSEY1QSteDwsOoBrp+uvFRTp2InBuThs4pFsiv9kuXclVzDAGySj4dzp30d8tbQk
CAUw7C29C79Fv1C5qfPrmAESrciIxpg0X40KPMbp1ZWVbd4=
-----END CERTIFICATE-----</copy>
````

Save this certificate into a file (mine is called digicert.pem).

### Create a wallet ###

````
$ <copy>orapki wallet create -wallet email_wallet -pwd <new_password></copy>
Oracle PKI Tool Release 19.0.0.0.0 - Production
Version 19.4.0.0.0
Copyright (c) 2004, 2019, Oracle and/or its affiliates. All rights reserved.

Operation is successfully completed.
````

Add certificate to the wallet

````
$ <copy>orapki wallet add -wallet email_wallet -cert digicert.pem -trusted_cert  -pwd <previous_password></copy>
Oracle PKI Tool Release 19.0.0.0.0 - Production
Version 19.4.0.0.0
Copyright (c) 2004, 2019, Oracle and/or its affiliates. All rights reserved.

Operation is successfully completed.
````

## Setup database ##

### Setup ACL for email server ###

````
<copy>begin
  -- Allow SMTP access for user ADMIN
  dbms_network_acl_admin.append_host_ace(
    host =>'smtp.email.eu-frankfurt-1.oci.oraclecloud.com',
    lower_port => 587,
    upper_port => 587,
    ace => xs$ace_type(
      privilege_list => xs$name_list('SMTP'),
    principal_name => 'SYS',
    principal_type => xs_acl.ptype_db));
end;
/</copy> 
````

### Setup test procedure for sending mail ###

````
SQL> <copy>CREATE OR REPLACE PROCEDURE SEND_MAIL (
  msg_to varchar2,
  msg_subject varchar2,
  msg_text varchar2 ) 
IS
 
  mail_conn utl_smtp.connection;
  username varchar2(1000):= 'ocid1.user.oc1..aaaaaaaapkxconfncwlw3ep6fsh737tfkn7zrmoors5bocukvpwbmq7w6zba@ocid1.tenancy.oc1..aaaaaaaafj37mytx22oquorcznlfuh77cd45int7tt7fo27tuejsfqbybzrq.eo.comocid1.user.oc1.username';
  passwd varchar2(50):= '9wn3G(3kN}v8c8!Vp[dk';
  msg_from varchar2(50) := 'sendmail@oraclepts.nl';
  mailhost VARCHAR2(50) := 'smtp.email.eu-frankfurt-1.oci.oraclecloud.com';
  wallet_path varchar2(100) := 'file:/home/oracle/email_wallet';
  wallet_password varchar2(100) := 'WElcome__123';
 
BEGIN
  mail_conn := UTL_smtp.open_connection( host => mailhost, port => 587, wallet_path => wallet_path, wallet_password => wallet_password );
  utl_smtp.starttls(mail_conn);
   
  UTL_SMTP.AUTH(mail_conn, username, passwd, schemes => 'PLAIN');
   
  utl_smtp.mail(mail_conn, msg_from);
  utl_smtp.rcpt(mail_conn, msg_to);
   
  UTL_smtp.open_data(mail_conn);
  
  UTL_SMTP.write_data(mail_conn, 'Date: ' || TO_CHAR(SYSDATE, 'DD-MON-YYYY HH24:MI:SS') || UTL_TCP.crlf);
  UTL_SMTP.write_data(mail_conn, 'To: ' || msg_to || UTL_TCP.crlf);
  UTL_SMTP.write_data(mail_conn, 'From: ' || msg_from || UTL_TCP.crlf);
  UTL_SMTP.write_data(mail_conn, 'Subject: ' || msg_subject || UTL_TCP.crlf);
  UTL_SMTP.write_data(mail_conn, 'Reply-To: ' || msg_to || UTL_TCP.crlf || UTL_TCP.crlf);
  UTL_SMTP.write_data(mail_conn, msg_text || UTL_TCP.crlf || UTL_TCP.crlf);
   
  UTL_smtp.close_data(mail_conn);
  UTL_smtp.quit(mail_conn);
 
EXCEPTION
  WHEN UTL_smtp.transient_error OR UTL_smtp.permanent_error THEN
    UTL_smtp.quit(mail_conn);
    dbms_output.put_line(sqlerrm);
  WHEN OTHERS THEN
    UTL_smtp.quit(mail_conn);
    dbms_output.put_line(sqlerrm);
END;
/</copy>
````

## Send initial email ##

````
SQL> <copy>execute send_mail('robert.pastijn@oracle.com','MySubject','My content');</copy>
````

## Acknowledgements ##

- **Author** - Robert Pastijn, Database Product Management, PTS EMEA - April 2020 


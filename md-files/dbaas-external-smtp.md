# Using external SMTP with Oracle PaaS #

## Description ## 

Due to security reasons, Oracle has decided to close port 25 for all outgoing (email) traffic to the public internet. This means that sending email from an Oracle Cloud VM to a third party using port 25, the internet standard for a listening mail server, is not possible.

Please note that this example was written for OCI Classic environments but is also applicable to Oracle OCI environments.

MOS note 2079333.1 describes the possible work-arounds for this situation:

- Use a REST-API provided by Oracle which can be used to send email messages
- Use another port than 25 for outgoing email traffic
- Use an external Mail Transport Agent (MTA) through another port than 25

The REST-API solution provided works for command-line (cURL) and other REST-capable scripts and is also available for any Oracle Application Express (APEX) setups on the Cloud instances. Downside of the REST-API solution is that the sender (from: field) cannot be changed and defaults to the OPC domain user email address that was used during authentication in the call to the REST-API.

Using another port than port 25 for outgoing traffic is also not feasible unless the recipient of the mail messages is aware of the port-change. This means that you can only send email to systems that are under your control or who have made arrangements to receive on different ports. Sending regular messages to recipients hosted on systems outside of your control is therefore not possible.

This document describes the last option offered in the MOS note, using an external mail transport agent (MTA) through another port than 25. The local MTA can be configured to forward/relay local messages to another MTA (in this example Gmail) in the same way a local email program would contact and send messages. The port used (in our example port 587) is not blocked in the Oracle Cloud.

## Requirements ##

- A valid Gmail username (emailaddress) and password
	- If you do not have a valid Gmail username and password, please register at gmail.com
- An application password (in case 2-factor authentication is used)
    - https://security.google.com/settings/security/apppasswords

## Install sendmail and required tools ##

Log in as opc user and become root user:

````
# <copy>sudo su -</copy>
````
    
Install the required packages

````
# <copy>yum install sendmail sendmail-cf cyrus-sasl-plain cyrus-sasl-md5</copy>
````

Create required directories

````
# <copy>mkdir /etc/mail/auth /etc/mail/certs</copy>
````

Change the properties for the directories

````
# <copy>chmod 700 /etc/mail/auth /etc/mail/certs</copy>
````

Create authentication file for 3rd party relay

````
# <copy>vi /etc/mail/auth/gmail-auth</copy>
````

Paste the following content into this file. Change the values as required:

````
<copy>AuthInfo:smtp.gmail.com "U:smmsp" "I:your_gmail_address"
"P:your_password" "M:PLAIN"
AuthInfo:smtp.gmail.com:587 "U:smmsp" "I:your_gmail_address"
"P:your_password" "M:PLAIN"</copy>
````

Please note that Gmail usually uses a 2-factor authentication setup. This means you cannot use your regular Gmail password in this setup. Please create an application password using the following URL: https://security.google.com/settings/security/apppasswords

Hash the authentication file for sendmail

````
# <copy>cd /etc/mail/auth</copy>
````

````
# <copy>makemap -r hash gmail-auth < gmail-auth</copy>
````
````
# <copy>chmod 600 gmail-auth*</copy>
````

Next, since gmail uses TLS/SSL to connect, we need to generate some key files for sendmail:

````
# <copy>cd /etc/mail/certs</copy>
````
````
# <copy>openssl req -nodes -new -x509 -keyout sendmail.pem -out sendmail.pem -days 3650</copy>
````

Please enter the required details when asked.

After this, we can copy our CA bundle of certifications to the mail certification directory.

````
# <copy>cp /etc/pki/tls/certs/ca-bundle.crt /etc/mail/certs</copy>
````
 
Now change the sendmail.mc file to include the relay information. Open the sendmail.mc and locate the first MAILER entry

````
# <copy>vi /etc/mail/sendmail.mc</copy>
````

(in vi, you can search using the '/' command eg. `/MAILER <enter>`)
My initial entry was at line 174; Original entry looks like this:

````
dnl MASQUERADE_DOMAIN(localhost.localdomain)dnl
dnl MASQUERADE_DOMAIN(mydomainalias.com)dnl
dnl MASQUERADE_DOMAIN(mydomain.lan)dnl
MAILER(smtp)dnl
MAILER(procmail)dnl
````

Insert the following lines above the MAILER entry:

````
<copy># Adding config for gmail #
define(`SMART_HOST',`smtp.gmail.com')dnl
define(`RELAY_MAILER_ARGS', `TCP $h 587')dnl
define(`ESMTP_MAILER_ARGS', `TCP $h 587')dnl
define(`CERT_DIR', `/etc/mail/certs')
define(`confCACERT_PATH', `CERT_DIR')
define(`confCACERT', `CERT_DIR/ca-bundle.crt')
define(`confCRL', `CERT_DIR/ca-bundle.crt')
define(`confSERVER_CERT', `CERT_DIR/sendmail.pem')
define(`confSERVER_KEY', `CERT_DIR/sendmail.pem')
define(`confCLIENT_CERT', `CERT_DIR/sendmail.pem')
define(`confCLIENT_KEY', `CERT_DIR/sendmail.pem')
define(`confAUTH_OPTIONS', `A p')dnl
TRUST_AUTH_MECH(`EXTERNAL DIGEST-MD5 CRAM-MD5 LOGIN PLAIN')dnl
define(`confAUTH_MECHANISMS', `EXTERNAL GSSAPI DIGEST-MD5 CRAM-MD5 LOGIN PLAIN')dnl
FEATURE(`authinfo',`hash -o /etc/mail/authinfo/gmail-auth.db')dnl
# End config for gmail #</copy>
````
 
The result should therefore look like this:

````
dnl MASQUERADE_DOMAIN(localhost)dnl
dnl MASQUERADE_DOMAIN(localhost.localdomain)dnl
dnl MASQUERADE_DOMAIN(mydomainalias.com)dnl
dnl MASQUERADE_DOMAIN(mydomain.lan)dnl

# Adding config for gmail #
define(`SMART_HOST',`smtp.gmail.com')dnl
define(`RELAY_MAILER_ARGS', `TCP $h 587')dnl
define(`ESMTP_MAILER_ARGS', `TCP $h 587')dnl

define(`CERT_DIR', `/etc/mail/certs')
define(`confCACERT_PATH', `CERT_DIR')
define(`confCACERT', `CERT_DIR/ca-bundle.crt')
define(`confCRL', `CERT_DIR/ca-bundle.crt')
define(`confSERVER_CERT', `CERT_DIR/sendmail.pem')
define(`confSERVER_KEY', `CERT_DIR/sendmail.pem')
define(`confCLIENT_CERT', `CERT_DIR/sendmail.pem')
define(`confCLIENT_KEY', `CERT_DIR/sendmail.pem')

define(`confAUTH_OPTIONS', `A p')dnl
TRUST_AUTH_MECH(`EXTERNAL DIGEST-MD5 CRAM-MD5 LOGIN PLAIN')dnl
define(`confAUTH_MECHANISMS', `EXTERNAL GSSAPI DIGEST-MD5 CRAM-MD5 LOGIN PLAIN')dnl
FEATURE(`authinfo',`hash -o /etc/mail/authinfo/gmail-auth.db')dnl
# End config for gmail #

MAILER(smtp)dnl
MAILER(procmail)dnl
dnl MAILER(cyrusv2)dnl
````

Save the file and rebuild sendmails config:

````
# <copy>make -C /etc/mail</copy>
````

Restart sendmail and the sendmail authentication service to read the new setup.

````
# <copy>service sendmail reload</copy>
````

````
# <copy>service saslauthd reload</copy>
````

Make sure the two services will start after the next reboot:

````
# <copy>chkconfig sendmail on</copy>
````
````
# <copy>chkconfig saslauthd on</copy>
````

## Testing ##

You can test sendmail by sending a testmail using the ‘mail’ command:

````
# <copy>echo  "Cloud Test" | mail -s "Mail from my Oracle Cloud domain" <recipient-email></copy>
````

You can trace the mail by tailing the /var/log/maillog file

````
# <copy>tail -f /var/log/maillog</copy>
````

The result should be similar to this:

````
Aug 31 07:39:24 rpastijn sendmail[2118]: u7VBdN0m002118: from=<opc@rpastijn.compute-usoracle80245.oraclecloud.internal>, size=615, class=0, nrcpts=1, msgid=<201608311139.u7VBdNbl002117@rpastijn.compute-usoracle80245.oraclecloud.internal>, proto=ESMTP, daemon=MTA, relay=localhost [127.0.0.1]
Aug 31 07:39:24 rpastijn sendmail[2117]: u7VBdNbl002117: to=robert.pastijn@oracle.com, ctladdr=opc (500/500), delay=00:00:01, xdelay=00:00:01, mailer=relay, pri=30267, relay=[127.0.0.1] [127.0.0.1], dsn=2.0.0, stat=Sent (u7VBdN0m002118 Message accepted for delivery)
Aug 31 07:39:27 rpastijn sendmail[2120]: STARTTLS=client, relay=smtp.gmail.com, version=TLSv1/SSLv3, verify=OK, cipher=ECDHE-RSA-AES128-GCM-SHA256, bits=128/128
Aug 31 07:39:31 rpastijn sendmail[2120]: u7VBdN0m002118: to=<robert.pastijn@oracle.com>, ctladdr=<opc@rpastijn.compute-usoracle80245.oraclecloud.internal> (500/500), delay=00:00:08, xdelay=00:00:07, mailer=relay, pri=120615, relay=smtp.gmail.com [74.125.200.108], dsn=2.0.0, stat=Sent (OK 1472643570 n69sm64610909pfa.77 - gsmtp)
````

In this example, the sender of the email message is the user of the OS where the mail command was executed, followed by the domain name of the local server. When a message is sent using the rfc(2)822 standard, this will be overruled. To test sending with another sender, the following can be used:

````
# <copy>echo "Cloud test" | mail -s "Test" -S from=someuser@domain.com <recipient></copy>
````

## Acknowledgements ##

- **Author**, Robert Pastijn, Oracle Database Product Development, AUG-2016
- **Contributor**, Valentin Tabacaru, Oracle Database Product Developement, AUG-2016
- **Adopted for .md**, Robert Pastijn, Oracle Database Product Development, 04-MAY-2020

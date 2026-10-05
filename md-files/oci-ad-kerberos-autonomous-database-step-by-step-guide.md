# OCI Windows Active Directory Kerberos Authentication to Autonomous Database

Customer implementation guide for a disposable test environment.

## 1. Objective

This guide builds the following test path:

1. A Windows user signs in to an Active Directory domain.
2. Active Directory issues the user a Kerberos ticket-granting ticket (TGT).
3. SQL\*Plus uses the Kerberos cache to request an Autonomous Database service ticket.
4. SQL\*Plus connects to an Autonomous Database through a private endpoint.
5. Autonomous Database maps the Kerberos principal to a database user without a database password in the SQL\*Plus command.

The implementation uses two independent security layers:

- TLS protects the Oracle Net connection to the private endpoint.
- Kerberos authenticates the Windows user to the database.

The selected database transport is TLS without mutual TLS (mTLS). This is still an encrypted `TCPS` connection; it does not mean an unencrypted connection. The ADB private endpoint remains the network restriction.

This guide configures direct external-user mapping. It does not configure Centrally Managed Users with Active Directory (CMU-AD).

## 2. Important implementation decisions

### 2.1 The ADB must exist before the SPN is created

The ADB GUID and `KINSTANCE` do not exist before the Autonomous Database is created. Do not create the Oracle service principal in Active Directory during the initial AD build.

The required order is:

1. Create the ADB with a private endpoint.
2. Connect to the ADB as `ADMIN` through the private endpoint using TLS.
3. Retrieve the ADB GUID and `KINSTANCE` with SQL queries.
4. Create the AD SPN and keytab using those exact values.
5. Upload the Kerberos files and enable Kerberos in the ADB.

### 2.2 The Kerberos service values are retrieved, not guessed

The default ADB Kerberos service component is the database GUID returned by:

```sql
<copy>SELECT GUID FROM V$PDBS;</copy>
```

The Kerberos instance component is returned by:

```sql
<copy>
SELECT json_value(cloud_identity, '$.PUBLIC_DOMAIN_NAME') AS KINSTANCE
FROM   V$PDBS;</copy>
```

The resulting service principal is:

```text
<ADB_GUID>/<KINSTANCE>@<KERBEROS_REALM>
```

Do not substitute any of the following values for `ADB_GUID`:

- the OCI Autonomous Database OCID;
- the database display name;
- the database name;
- the private endpoint IP address;
- the private endpoint FQDN.

Oracle documents that `KINSTANCE` can differ from the private-endpoint FQDN. Record both values exactly, including case.

### 2.3 TLS without mTLS is required for this guide

When provisioning the ADB private endpoint, leave **Require mutual TLS (mTLS) authentication** cleared. The database must show that mTLS is not required and that TLS connections are allowed.

Use the private-endpoint TLS connection string from the ADB **Database connection** page. Do not reuse an mTLS connection string or assume that port `1522` is always correct. ADB supplies the correct TLS port and service name; TLS can use port `1521` or `1522`.

The Oracle Client home must not set `WALLET_LOCATION` for this TLS/no-wallet path. The database wallet is not used for the Oracle Net connection in this procedure.

### 2.4 The 19.3 package is only the base installation

The downloadable full Windows client package `WINDOWS.X64_193000_client_home.zip` is a 19.3 base home. Do not use that unpatched home for the final test.

Download the applicable Windows Oracle Database Client patch from Oracle Support and patch the home to version 19.14 or later. Oracle documents that Windows OCI clients support TLS without a wallet from 19.14 onward. The observed 19.3 client behavior is therefore treated as an unsupported baseline for this selected TLS/non-mTLS path.

The final prerequisite is:

```text
Oracle Database Client for Windows: 19.14.0.0.0 or later
```

The client must also contain the full-client Kerberos utilities used by this guide, including `sqlplus.exe`, `okinit.exe`, and `oklist.exe`.

## 3. Reference values

Replace every angle-bracket value before implementation. Keep generated secrets in an approved secret store and out of this document.

| Item | Example | Customer value |
|---|---|---|
| OCI region | `eu-frankfurt-1` | `<REGION>` |
| Tenancy home region | `us-phoenix-1` | `<TENANCY_HOME_REGION>` |
| OCI profile | `ORACLEPARTNERSAS` | `<OCI_PROFILE>` |
| Test compartment | `KERBEROS-ADB-LAB` | `<LAB_COMPARTMENT>` |
| VCN CIDR | `10.20.0.0/16` | `<VCN_CIDR>` |
| Windows subnet | `10.20.1.0/24` | `<WINDOWS_SUBNET_CIDR>` |
| ADB subnet | `10.20.2.0/24` | `<ADB_SUBNET_CIDR>` |
| AD server | `WIN-AD` / `10.20.1.10` | `<AD_HOST>` / `<AD_PRIVATE_IP>` |
| AD server FQDN | `WIN-AD.corp.example.com` | `<AD_FQDN>` |
| Windows client | `WIN-Client` / `10.20.1.20` | `<CLIENT_HOST>` / `<CLIENT_PRIVATE_IP>` |
| AD DNS domain | `corp.example.com` | `<AD_DOMAIN>` |
| NetBIOS domain | `CORP` | `<NETBIOS_DOMAIN>` |
| Kerberos realm | `CORP.EXAMPLE.COM` | `<KERBEROS_REALM>` |
| Test user | `CORP\alice` | `<AD_TEST_USER>` |
| Oracle service account | `CORP\ora_adb` | `<AD_SERVICE_ACCOUNT>` |
| ADB display name | `KERBEROS-ADB` | `<ADB_DISPLAY_NAME>` |
| ADB OCID | `ocid1.autonomousdatabase...` | `<ADB_OCID>` |
| ADB GUID | Retrieved from `V$PDBS` | `<ADB_GUID>` |
| ADB `KINSTANCE` | Retrieved from `V$PDBS` | `<KINSTANCE>` |
| ADB private endpoint FQDN | Retrieved from OCI | `<ADB_PRIVATE_FQDN>` |
| ADB private endpoint IP | Retrieved from OCI | `<ADB_PRIVATE_IP>` |
| ADB TLS port | Retrieved from OCI | `<ADB_TLS_PORT>` |
| ADB TLS service | Retrieved from OCI | `<ADB_TLS_SERVICE>` |
| TLS TNS alias | `ADB_TLS` | `<ADB_TLS_ALIAS>` |
| Oracle home | `C:\oracle\client19home` | `<ORACLE_HOME>` |
| MIT Kerberos configuration | `C:\oracle\krb.conf` | `<KRB5_CONF>` |
| Kerberos cache | `FILE:C:\Temp\krb5cc` | `<KRB5_CCACHE>` |

## 4. Prerequisites

### OCI permissions

The operator needs permission to:

- create or use the test compartment;
- create VCNs, subnets, route tables, gateways, security lists, and NSGs;
- create Windows Compute instances and retrieve their initial credentials;
- create and manage the Autonomous Database and private endpoint;
- generate or view ADB connection information;
- write temporary objects to a private Object Storage bucket;
- create or use the ADB credential required to read the Kerberos input files.

### Software

| Component | Requirement |
|---|---|
| OCI CLI | Current supported version on the administration workstation |
| AD server | Windows Server 2022 or an approved supported Windows Server release |
| Windows client | Windows Server 2022 or an approved supported Windows client release |
| Oracle client | Full Windows client 19.3 base package patched to 19.14 or later. This guide was written with client version 19.32 |
| ADB | Autonomous Database with Kerberos external-authentication support and private endpoint |
| Object Storage | Private bucket for `krb.conf` and `v5srvtab` during enablement |

### Network prerequisites

- WIN-AD and WIN-Client must be able to communicate over the AD DS and Kerberos ports.
- WIN-Client must use WIN-AD as its DNS server after AD DS is promoted.
- WIN-Client must resolve the ADB private endpoint FQDN to the private endpoint IP.
- WIN-Client must reach the ADB private TLS port.
- WIN-AD and WIN-Client must use synchronized time.
- The private endpoint must not be reachable only through public DNS or a public endpoint.

## 5. OCI network

Create one VCN with separate Windows and ADB subnets. The example uses public IPs only for temporary administration. Use private subnets and a bastion, VPN, or equivalent controlled access path for production.

### 5.1 Required traffic

| Source | Destination | Protocol/port | Purpose |
|---|---|---|---|
| Approved admin CIDRs | WIN-AD and WIN-Client | TCP 3389 | Temporary RDP administration |
| WIN-Client subnet | WIN-AD | TCP/UDP 53 | DNS |
| WIN-Client subnet | WIN-AD | TCP/UDP 88 | Kerberos |
| WIN-Client subnet | WIN-AD | TCP/UDP 464 | Kerberos password and ticket operations |
| WIN-Client subnet | WIN-AD | TCP 389 | LDAP and domain operations |
| WIN-Client subnet | WIN-AD | TCP 445 | SMB and domain operations |
| WIN-Client subnet | WIN-AD | TCP 135 and dynamic RPC | AD management and domain join |
| WIN-AD and WIN-Client | Approved time source | UDP 123 | Time synchronization |
| WIN-Client subnet | ADB private endpoint | TCP `<ADB_TLS_PORT>` | TLS Oracle Net connection |
| WIN-AD and WIN-Client | OCI services as required | TCP 443 | OCI agent and package retrieval |

Use stateful rules. Restrict RDP to the approved administrator source CIDRs. Do not use an all-protocol rule outside the disposable test environment.

### 5.2 OCI CLI examples

Run OCI CLI commands from the administration workstation. Every command that contacts OCI should have a bounded timeout.

```bash
<copy>timeout 10 oci iam compartment create \
  --profile <OCI_PROFILE> \
  --region <TENANCY_HOME_REGION> \
  --compartment-id <PARENT_COMPARTMENT_OCID> \
  --name KERBEROS-ADB-LAB \
  --description 'AD Kerberos and Autonomous Database test environment'

timeout 10 oci network vcn create \
  --profile <OCI_PROFILE> \
  --region <REGION> \
  --compartment-id <LAB_COMPARTMENT_OCID> \
  --cidr-block 10.20.0.0/16 \
  --display-name KERBEROS-ADB-VCN \
  --dns-label corp

timeout 10 oci network subnet create \
  --profile <OCI_PROFILE> \
  --region <REGION> \
  --compartment-id <LAB_COMPARTMENT_OCID> \
  --vcn-id <VCN_OCID> \
  --cidr-block 10.20.1.0/24 \
  --display-name WINDOWS-SUBNET \
  --route-table-id <WINDOWS_ROUTE_TABLE_OCID> \
  --security-list-ids '["<WINDOWS_SECURITY_LIST_OCID>"]' \
  --prohibit-public-ip-on-vnic false

timeout 10 oci network subnet create \
  --profile <OCI_PROFILE> \
  --region <REGION> \
  --compartment-id <LAB_COMPARTMENT_OCID> \
  --vcn-id <VCN_OCID> \
  --cidr-block 10.20.2.0/24 \
  --display-name ADB-SUBNET \
  --route-table-id <ADB_ROUTE_TABLE_OCID> \
  --security-list-ids '["<ADB_SECURITY_LIST_OCID>"]' \
  --prohibit-public-ip-on-vnic true<copy>
```

## 6. Active Directory server

### 6.1 Create and prepare WIN-AD

Launch a Windows Server VM with a reserved private IP.

```bash
<copy>timeout 10 oci compute instance launch \
  --profile <OCI_PROFILE> \
  --region <REGION> \
  --availability-domain <AVAILABILITY_DOMAIN> \
  --compartment-id <LAB_COMPARTMENT_OCID> \
  --shape VM.Standard.E5.Flex \
  --shape-config '{"ocpus":2,"memoryInGBs":16}' \
  --display-name WIN-AD \
  --hostname-label WIN-AD \
  --image-id <WINDOWS_SERVER_IMAGE_OCID> \
  --subnet-id <WINDOWS_SUBNET_OCID> \
  --private-ip 10.20.1.10 \
  --assign-public-ip true \
  --boot-volume-size-in-gbs 64 \
  --instance-options '{"areLegacyImdsEndpointsDisabled":true}'</copy>
```

1. Retrieve the one-time Windows credential with `oci compute instance get-windows-initial-creds`.
2. Connect to WIN-AD using the approved administration method.
3. Rename the computer to `WIN-AD` and restart.
4. Confirm that the private IP remains reserved.
5. Before DNS is installed, use the OCI resolver or DHCP-provided resolver for updates.
6. Keep Windows Firewall enabled.

### 6.2 Install AD DS and promote the forest

Run in an elevated PowerShell session on WIN-AD:

```powershell
<copy>
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

Install-ADDSForest `
  -DomainName 'corp.example.com' `
  -DomainNetbiosName 'CORP' `
  -InstallDNS `
  -Force
</copy>
```

After the restart, verify the domain controller:

```powershell
<copy>
Get-ComputerInfo | Select-Object CsName, WindowsProductName, CsDomain
Get-ADDomain | Select-Object DNSRoot, NetBIOSName, DomainMode
Get-ADDomainController -Discover -DomainName 'corp.example.com'
Resolve-DnsName -Name 'WIN-AD.corp.example.com'
dcdiag /test:dns /v
w32tm /query /status
</copy>
```

Expected results:

- the computer name is `WIN-AD`;
- the DNS root is `corp.example.com`;
- WIN-AD is returned as the domain controller;
- the WIN-AD A record resolves to the reserved private IP;
- DNS diagnostics pass;
- the time source is valid.

### 6.3 Configure DNS, time, and test accounts

Run on WIN-AD. Replace the forwarder with the customer-approved resolver if required.

```powershell
<copy>Add-DnsServerForwarder -IPAddress 169.254.169.254 -PassThru
Add-DnsServerResourceRecordA `
  -ZoneName 'corp.example.com' `
  -Name 'WIN-AD' `
  -IPv4Address '10.20.1.10' `
  -TimeToLive 01:00:00

New-ADOrganizationalUnit `
  -Name 'Kerberos Lab' `
  -Path 'DC=corp,DC=example,DC=com'

New-ADUser `
  -Name 'Alice Example' `
  -SamAccountName 'alice' `
  -UserPrincipalName 'alice@corp.example.com' `
  -Path 'OU=Kerberos Lab,DC=corp,DC=example,DC=com' `
  -AccountPassword (Read-Host 'Alice password' -AsSecureString) `
  -Enabled $true

New-ADUser `
  -Name 'Oracle ADB Service' `
  -SamAccountName 'ora_adb' `
  -UserPrincipalName 'ora_adb@corp.example.com' `
  -Path 'OU=Kerberos Lab,DC=corp,DC=example,DC=com' `
  -AccountPassword (Read-Host 'Service account password' -AsSecureString) `
  -Enabled $true `
  -PasswordNeverExpires $true

w32tm /config /manualpeerlist:'<NTP_SERVER>' /syncfromflags:manual /update
Restart-Service W32Time
w32tm /resync
</copy>
```

> The IP address for the NTP server in OCI is 169.254.169.254 (same as the DNS)

`PasswordNeverExpires` is acceptable only for this disposable test. Use managed service-account controls and rotation in production.

Do not create the Oracle SPN or keytab yet. The required ADB values are not available until section 7 is complete.

## 7. Windows client

### 7.1 Create and join WIN-Client

Launch the Windows client with a reserved private IP:

```bash
<copy>
timeout 10 oci compute instance launch \
  --profile <OCI_PROFILE> \
  --region <REGION> \
  --availability-domain <AVAILABILITY_DOMAIN> \
  --compartment-id <LAB_COMPARTMENT_OCID> \
  --shape VM.Standard.E5.Flex \
  --shape-config '{"ocpus":2,"memoryInGBs":16}' \
  --display-name WIN-Client \
  --hostname-label WIN-Client \
  --image-id <WINDOWS_SERVER_IMAGE_OCID> \
  --subnet-id <WINDOWS_SUBNET_OCID> \
  --private-ip 10.20.1.20 \
  --assign-public-ip true \
  --boot-volume-size-in-gbs 64 \
  --instance-options '{"areLegacyImdsEndpointsDisabled":true}'
</copy>
```

On WIN-Client (only if the initial name of the instance was not WIN-Client), run PowerShell as Administrator:

```powershell
<copy>
Rename-Computer -NewName 'WIN-Client' -Restart
</copy>
```

After the restart, set the DNS server to WIN-AD. Use the actual interface index or alias returned by `Get-NetAdapter`.

```powershell
<copy>
Get-NetAdapter | Where-Object Status -eq 'Up'

Set-DnsClientServerAddress `
  -InterfaceAlias 'Ethernet' `
  -ServerAddresses '<AD_PRIVATE_IP>'

Clear-DnsClientCache
Resolve-DnsName '<AD_DOMAIN>'
Resolve-DnsName '<AD_FQDN>'
</copy>
```

Join the domain and restart:

```powershell
<copy>
Add-Computer `
  -DomainName '<AD_DOMAIN>' `
  -Credential (Get-Credential '<NETBIOS_DOMAIN>\Administrator') `
  -Restart
</copy>
```

Sign in interactively as the domain test user. Do not use a local Windows account for the final Kerberos test.

### 7.2 Install the full 19.3 base client

Obtain the full client ZIP from the approved software source. The example below uses a placeholder URL; do not store a PAR URL or password in this guide.

Run in an elevated PowerShell session on WIN-Client:

```powershell
<copy>New-Item -ItemType Directory -Force -Path 'C:\Temp','C:\oracle' | Out-Null

Invoke-WebRequest `
  -Uri '<FULL_19_3_CLIENT_PACKAGE_URL>' `
  -OutFile 'C:\Temp\WINDOWS.X64_193000_client_home.zip'

Expand-Archive `
  -Path 'C:\Temp\WINDOWS.X64_193000_client_home.zip' `
  -DestinationPath 'C:\oracle\client19home' `
  -Force

Test-Path 'C:\oracle\client19home\bin\sqlplus.exe'
Test-Path 'C:\oracle\client19home\bin\okinit.exe'
Test-Path 'C:\oracle\client19home\bin\oklist.exe'
</copy>
``` 

> 19.3 client URL I used was https://objectstorage.eu-amsterdam-1.oraclecloud.com/p/JDDgmK1s20OBKTUxyhtLK5p1LF-puL8B7oTMEn1ZYCoSxwUMrppHiZ_YQnCVnjlQ/n/oraclepartnersas/b/LDAP-AI-BUCKET/o/WINDOWS.X64_193000_client_home.zip

All three checks must return `True`. Do not continue with an Instant Client package or a package that lacks the full-client Kerberos utilities.

### 7.3 Patch the client to 19.14 or later

In order to be able to patch the client, it needs to have an Oracle Inventory. Login as opc on the WIN-Client system and run the `setup.bat` file from the unzipped Oracle Home location

- Install the client on the system
- Set the Oracle Base directory to c:\oracle

Download the applicable Windows 19.14-or-later client patch and the compatible OPatch version from Oracle Support. Patch identifiers vary by platform and patch bundle; use the README shipped with the selected patch.

```powershell
<copy>
Invoke-WebRequest `
-Uri '<19.14 or above patch URL>' `
-OutFile 'C:\Temp\Windows_Client_Patch.zip'
</copy>
```

> URL for me is: https://objectstorage.eu-amsterdam-1.oraclecloud.com/p/qzy72yW9HhU97MsZXeeB7WVul-sZ05GM9S0HoHeFw4bpSPLvTeLPZYCWN2e3cA4g/n/oraclepartnersas/b/LDAP-AI-BUCKET/o/p39418910_190000_MSWIN-x86-64.zip

```powershell
<copy>
Invoke-WebRequest `
-Uri '<OPatch for Windows URL>' `
-OutFile 'C:\Temp\Windows_OPatch.zip'
</copy>
```

> URL for me is: https://objectstorage.eu-amsterdam-1.oraclecloud.com/p/cdcXJHPxGADt25xb5Pdo8pGcCfZ_dkkG_Y_O0HMunyJ5Yfwc02dw_RamvdElwmEE/n/oraclepartnersas/b/LDAP-AI-BUCKET/o/p6880880_190000_MSWIN-x86-64.zip
>

```powershell
<copy>
Expand-Archive `
-Path 'C:\Temp\Windows_OPatch.zip' `
-DestinationPath 'C:\oracle\client19home' `
-Force
</copy>
```

Before applying the patch:

```powershell
<copy>
$env:ORACLE_HOME = 'C:\oracle\client19home'
$env:PATH = "$env:ORACLE_HOME\bin;$env:ORACLE_HOME\OPatch;$env:PATH"

& "$env:ORACLE_HOME\OPatch\opatch.bat" version
& "$env:ORACLE_HOME\OPatch\opatch.bat" lsinventory
</copy>
```

Stop all processes using the Oracle home, including SQL\*Plus, before patching. Apply it from an elevated PowerShell session:

```powershell
<copy>
Expand-Archive `
  -Path 'C:\Temp\Windows_Client_Patch.zip' `
  -DestinationPath 'C:\oracle\client19cPatch' `
  -Force

Set-Location 'C:\oracle\client19cPatch\39418910'

& "$env:ORACLE_HOME\OPatch\opatch.bat" `
  prereq CheckConflictAgainstOHWithDetail `
  -phBaseDir (Get-Location)

& "$env:ORACLE_HOME\OPatch\opatch.bat" apply
& "$env:ORACLE_HOME\OPatch\opatch.bat" lsinventory
& "$env:ORACLE_HOME\bin\sqlplus.exe" -V
</copy>
```

The SQL\*Plus version must report 19.14 or later. If it still reports 19.3, stop and correct the patching process before continuing.

This patch level is required for the selected Windows OCI TLS/no-wallet path. The patch level also removes the ambiguity observed when the 19.3 base home was used for the Kerberos test.

## 8. Create the private-endpoint Autonomous Database

### 8.1 Provision the ADB

In the OCI Console:

1. Open **Oracle AI Database**, **Autonomous AI Database**, and select **Create Autonomous AI Database**.
2. Select the test compartment and the same region as the VCN.
3. Select the required workload, compute, and storage settings.
4. Select **Private endpoint access only**.
5. Select the VCN and ADB subnet created in section 5.
6. Attach the ADB NSG or security list that allows TCP `<ADB_TLS_PORT>` from the Windows subnet.
7. Leave **Require mutual TLS (mTLS) authentication** cleared. TLS must be allowed.
8. Create the database and wait until its lifecycle state is `AVAILABLE`.

The database must show a private endpoint and must not require mTLS. The private endpoint applies to both TLS and mTLS network access; this guide deliberately uses TLS without mTLS.

Retrieve the OCI-level details after the database is available:

```bash
<copy>
timeout 10 oci db autonomous-database get \
  --profile <OCI_PROFILE> \
  --region <REGION> \
  --autonomous-database-id <ADB_OCID> \
  --query 'data.{id:id,displayName:"display-name",dbName:"db-name",state:"lifecycle-state",privateFqdn:"private-endpoint",privateIp:"private-endpoint-ip"}' \
  --output table
</copy>
```

Record the following in the secure implementation record:

- ADB OCID;
- ADB display name and database name;
- lifecycle state;
- private endpoint FQDN;
- private endpoint IP;
- private TLS port and TLS service name from the Database connection page.

### 8.2 Obtain the private TLS connection information

In the ADB Console:

1. Open **Database connection**.
2. Select **TLS authentication**, not mTLS authentication.
3. Select **Private endpoint** as the access type.
4. Select the required service, such as `HIGH`, `MEDIUM`, `LOW`, `TP`, or `TPURGENT`.
5. Copy the complete private TLS connection string.

The connection string is the authoritative source for the private host, port, service name, and TLS settings. Do not construct these values from the database display name.

Create a temporary `tnsnames.ora` entry on the host that will perform the initial ADMIN connection. Use the exact values from the OCI TLS connection string:

```text
<copy>
<ADB_TLS_ALIAS> =
  (description=
    (retry_count=20)
    (retry_delay=3)
    (address=(protocol=tcps)(port=<ADB_TLS_PORT>)(host=<ADB_PRIVATE_FQDN>))
    (connect_data=(service_name=<ADB_TLS_SERVICE>))
    (security=(ssl_server_dn_match=yes))
  )
</copy>
```

For TLS without a wallet, do not add `WALLET_LOCATION` to `sqlnet.ora`.

### 7.3 Retrieve the ADB GUID and `KINSTANCE`

Connect as `ADMIN` over the private TLS alias. This is the first point at which the database Kerberos values can be obtained. Only the SPN and keytab must wait for the values returned below.

```powershell
<copy>
$env:ORACLE_HOME = 'C:\oracle\client19home'
New-Item -ItemType Directory -Force -Path 'C:\oracle\adb_tls' | Out-Null

# Save the TLS entry from section 7.2 as:
# C:\oracle\adb_tls\tnsnames.ora

$env:TNS_ADMIN = 'C:\oracle\adb_tls'
$env:PATH = "$env:ORACLE_HOME\bin;$env:PATH"

& "$env:ORACLE_HOME\bin\sqlplus.exe" 'ADMIN@<ADB_TLS_ALIAS>'
</copy>
```

Enter the ADMIN password at the prompt. Run:

```sql
<copy>
SET HEADING ON
SET PAGESIZE 100

SELECT GUID
FROM   V$PDBS;

SELECT json_value(cloud_identity, '$.PUBLIC_DOMAIN_NAME') AS KINSTANCE
FROM   V$PDBS;

EXIT;
</copy>
```

Record the returned values exactly:

```text
ADB_GUID=<value returned by SELECT GUID FROM V$PDBS>
KINSTANCE=<value returned by PUBLIC_DOMAIN_NAME query>
```

Important:

- `ADB_GUID` is not the OCI Database OCID.
- `ADB_GUID` is case-sensitive.
- `KINSTANCE` is not necessarily the private endpoint FQDN.
- The private endpoint FQDN is used for Oracle Net connectivity; `KINSTANCE` is used in the Kerberos service principal.

### 8.4 Create the AD service principal and keytab

Return to WIN-AD only after section 7.3 has returned the values.

The exact principal is:

```text
<ADB_GUID>/<KINSTANCE>@<KERBEROS_REALM>
```

Run in an elevated Command Prompt or PowerShell session on WIN-AD. Use a secure password prompt; do not place the service-account password in the command history.

```powershell
<copy>
ktpass `
  -princ '<ADB_GUID>/<KINSTANCE>@<KERBEROS_REALM>' `
  -mapuser '<NETBIOS_DOMAIN>\<AD_SERVICE_ACCOUNT>' `
  -crypto ALL `
  -ptype KRB5_NT_PRINCIPAL `
  -pass '*' `
  -out 'C:\oracle\v5srvtab'

setspn -Q '<ADB_GUID>/<KINSTANCE>'
setspn -L '<NETBIOS_DOMAIN>\<AD_SERVICE_ACCOUNT>'
certutil -hashfile 'C:\oracle\v5srvtab' SHA256
</copy>
```

Expected results:

- `ktpass` creates `C:\oracle\v5srvtab`;
- `setspn -Q` returns exactly one owning account;
- `setspn -L` shows the exact `<ADB_GUID>/<KINSTANCE>` SPN;
- the SHA-256 hash is recorded only in the secure implementation record if needed.

If `setspn -Q` returns multiple accounts, stop and resolve the duplicate before continuing.

### 8.5 Create the Kerberos configuration

Create the following file with the customer values. It is later copied to WIN-Client and uploaded to Object Storage for ADB enablement.

`krb.conf`:

```ini
<copy>
[libdefaults]
    default_realm = <KERBEROS_REALM>
    dns_lookup_realm = false
    dns_lookup_kdc = false
    rdns = false
    ticket_lifetime = 24h
    forwardable = true

[realms]
    <KERBEROS_REALM> = {
        kdc = <AD_FQDN>:88
        admin_server = <AD_FQDN>:749
    }

[domain_realm]
    .<AD_DOMAIN> = <KERBEROS_REALM>
    <AD_DOMAIN> = <KERBEROS_REALM>
</copy>
```

The SQL\*Plus client needs the KDC entry, DNS resolution, and port 88. The `admin_server` entry is retained for standard MIT Kerberos configuration completeness.

### 8.6 Upload Kerberos inputs and enable ADB Kerberos

Place `krb.conf` and `v5srvtab` in a private Object Storage bucket. Use a short-lived PAR or an ADB credential with the minimum required access. Do not place a permanent public URL in the guide.

```bash
<copy>
timeout 10 oci os object put \
  --profile <OCI_PROFILE> \
  --region <REGION> \
  --namespace-name <OBJECT_STORAGE_NAMESPACE> \
  --bucket-name <PRIVATE_BUCKET> \
  --name krb.conf \
  --file ./krb.conf \
  --force

timeout 10 oci os object put \
  --profile <OCI_PROFILE> \
  --region <REGION> \
  --namespace-name <OBJECT_STORAGE_NAMESPACE> \
  --bucket-name <PRIVATE_BUCKET> \
  --name v5srvtab \
  --file ./v5srvtab \
  --force
</copy>
```

Using the private TLS ADMIN connection, run:

```sql
<copy>
ALTER DATABASE PROPERTY SET ROUTE_OUTBOUND_CONNECTIONS = 'PRIVATE_ENDPOINT';

BEGIN
  DBMS_CLOUD_ADMIN.ENABLE_EXTERNAL_AUTHENTICATION(
    type   => 'KERBEROS',
    params => JSON_OBJECT(
      'location_uri' VALUE '<PRIVATE_KERBEROS_INPUTS_URI>',
      'credential_name' VALUE '<ADB_OBJECT_STORAGE_CREDENTIAL>'
    )
  );
END;
/

SELECT property_name, property_value
FROM   database_properties
WHERE  property_name IN ('KERBEROS_DIRECTORY', 'ROUTE_OUTBOUND_CONNECTIONS');

SELECT SYS_CONTEXT('USERENV', 'KERBEROS_SERVICE_NAME') AS KERBEROS_SERVICE_NAME
FROM   DUAL;
</copy>
```

If the location is a PAR, omit `credential_name` when the ADB procedure permits it. Follow the current ADB procedure for the selected Object Storage authentication method.

Expected results:

- Kerberos enablement completes successfully;
- `KERBEROS_DIRECTORY` is non-null;
- `ROUTE_OUTBOUND_CONNECTIONS` is `PRIVATE_ENDPOINT`;
- the Kerberos service name matches the selected default GUID or the explicitly configured custom service name.

After successful import and validation, delete `krb.conf`, `v5srvtab`, and the temporary PAR or object unless the test requires a documented repeat run.

### 8.7 Map the database user

Use a dedicated database user for a real application. Keeping the  password-based ADMIN recovery available until the test is complete. Create a new user for testing the Kerberos setup

```text
<copy>
SQL> grant dba, unlimited tablespace to alice identified by <Temp_Password_Alice>;
</copy>
```

First inspect the existing mapping:

```sql
<copy>
SELECT username, authentication_type, external_name
FROM   dba_users
WHERE  username = '<ADB_DATABASE_USER>';
</copy>
```

Map the exact AD principal returned by the Kerberos environment:

```sql
<copy>
ALTER USER <ADB_DATABASE_USER>
  IDENTIFIED EXTERNALLY AS '<AD_USER_PRINCIPAL>';

SELECT username, authentication_type, external_name
FROM   dba_users
WHERE  username = '<ADB_DATABASE_USER>';
</copy>
```

For example, the external name may be `alice@CORP.EXAMPLE.COM`, but use the exact principal and case required by the implementation. Do not assume that the display name or UPN is interchangeable with the Kerberos principal.

## 9. Kerberos client configuration

### 9.1 Create and inspect the FILE cache

Sign in interactively as the domain user on WIN-Client. Run:

```powershell
</copy>
whoami
klist

& 'C:\oracle\client19home\jdk\bin\kinit.exe' `
  -k `
  -t 'C:\oracle\v5srvtab' `
  '<ADB_GUID>/<KINSTANCE>@<KERBEROS_REALM>'
</copy>
```

Expected results:

- `whoami` returns the expected domain user;
- the cache contains a TGT for the user realm principal;

## 10. Configure Oracle Net for Kerberos over TLS without mTLS

### 10.1 Create the client directories

```powershell
<copy>
New-Item -ItemType Directory -Force `
  -Path 'C:\oracle\adkerb_native','C:\oracle\krb5' | Out-Null
</copy>
```

Copy the private TLS `tnsnames.ora` entry obtained in section 7.2 to:

```text
<copy>
C:\oracle\adkerb_native\tnsnames.ora
</copy>
```

Do not copy an mTLS wallet configuration into this directory.

### 10.2 Create `sqlnet.ora`

Save this file as `C:\oracle\adkerb_native\sqlnet.ora`:

```ini
<copy>
SQLNET.AUTHENTICATION_SERVICES = (KERBEROS5)
SQLNET.AUTHENTICATION_KERBEROS5_SERVICE = <ADB_GUID>
SQLNET.KERBEROS5_CC_NAME = MSLSA:
SQLNET.KERBEROS5_CONF = C:\oracle\krb.conf
SQLNET.FALLBACK_AUTHENTICATION = FALSE
SSL_SERVER_DN_MATCH = yes
</copy>
```

Do not add `WALLET_LOCATION`. If another `sqlnet.ora` is being inherited through the active Oracle home or `TNS_ADMIN`, remove or comment out its `WALLET_LOCATION` for this test.

| Parameter | Purpose |
|---|---|
| `SSL_SERVER_DN_MATCH` | Validates the server identity in the TLS connection descriptor |
| `SQLNET.AUTHENTICATION_SERVICES` | Selects Kerberos 5 database authentication |
| `SQLNET.AUTHENTICATION_KERBEROS5_SERVICE` | Uses the ADB GUID as the Kerberos service component |
| `SQLNET.KERBEROS5_CONF` | Points to the MIT Kerberos configuration |
| `SQLNET.FALLBACK_AUTHENTICATION` | Prevents an accidental password fallback |

### 10.3 Verify the private TLS path and patched client

```powershell
</copy>
$env:ORACLE_HOME = 'C:\oracle\client19home'
$env:TNS_ADMIN = 'C:\oracle\adkerb_native'
$env:PATH = "$env:ORACLE_HOME\bin;$env:PATH"

Test-Path "$env:ORACLE_HOME\bin\sqlplus.exe"
Test-Path "$env:ORACLE_HOME\bin\okinit.exe"
Test-Path "$env:TNS_ADMIN\tnsnames.ora"
Test-Path "$env:TNS_ADMIN\sqlnet.ora"
Test-NetConnection '<ADB_PRIVATE_FQDN>' -Port <ADB_TLS_PORT>
& "$env:ORACLE_HOME\bin\sqlplus.exe" -V
</copy>
```

The TCP test must succeed and SQL\*Plus must report 19.14 or later. A failed TCP test is an OCI route, NSG/security-list, DNS, or Windows Firewall problem; it is not yet a Kerberos problem.

## 11. Final SQL\*Plus Kerberos test

The final test must run in the same interactive domain-user context that owns the FILE cache.

```powershell
<copy>
$env:ORACLE_HOME = 'C:\oracle\client19home'
$env:TNS_ADMIN = 'C:\oracle\adkerb_native'
$env:PATH = "$env:ORACLE_HOME\bin;$env:PATH"

whoami

& 'C:\oracle\client19home\jdk\bin\kinit.exe' `
  -k `
  -t 'C:\oracle\v5srvtab' `
  -c 'FILE:C:\Temp\krb5cc' `
  '<ADB_GUID>/<KINSTANCE>@<KERBEROS_REALM>'
  
& 'C:\oracle\client19home\jdk\bin\klist.exe' -c 'FILE:C:\Temp\krb5cc'

& "$env:ORACLE_HOME\bin\sqlplus.exe" -L /@ADB
</copy>
```

At the SQL\*Plus prompt:

```sql
<copy>
SET HEADING ON
SET PAGESIZE 50

SELECT USER FROM DUAL;

SELECT SYS_CONTEXT('USERENV', 'AUTHENTICATION_METHOD') AS AUTHENTICATION_METHOD
FROM   DUAL;

SELECT SYS_CONTEXT('USERENV', 'KERBEROS_SERVICE_NAME') AS KERBEROS_SERVICE_NAME
FROM   DUAL;

EXIT;
</copy>
```

Expected result:

- `USER` is the mapped database user;
- `AUTHENTICATION_METHOD` is `KERBEROS`;
- the reported service name corresponds to the configured ADB Kerberos service.

The SQL\*Plus command contains no database username or password. The transport is the private ADB TLS connection, and the database authentication is Kerberos.

## 12. End-to-end validation checklist

Run checks in order. Troubleshoot the first failing layer before continuing.

| Layer | Check | Expected result |
|---|---|---|
| OCI | ADB lifecycle state | `AVAILABLE` |
| OCI | ADB network access | Private endpoint access only; TLS allowed |
| DNS | `Resolve-DnsName <AD_FQDN>` | WIN-AD private IP |
| DNS | `Resolve-DnsName <ADB_PRIVATE_FQDN>` | ADB private endpoint IP |
| Domain | `Get-ComputerInfo` and `whoami` | WIN-Client is domain joined; interactive domain user is active |
| AD health | `dcdiag /test:dns /v` | DNS tests pass |
| Time | `w32tm /query /status` | Synchronized source and current time |
| TGT | `klist` | User TGT exists and is not expired |
| Service ticket | `kvno <ADB_GUID>/<KINSTANCE>` | ADB service ticket obtained |
| SPN | `setspn -Q <ADB_GUID>/<KINSTANCE>` | Exactly one owner |
| Keytab | `kinit -k -t v5srvtab <ADB_GUID>/<KINSTANCE>@<KERBEROS_REALM>` | Service ticket obtained |
| Client | `sqlplus -V` | Oracle Client 19.14 or later |
| Client tools | `Test-Path ...\okinit.exe` | `True` |
| TLS network | `Test-NetConnection <ADB_PRIVATE_FQDN> -Port <ADB_TLS_PORT>` | `TcpTestSucceeded=True` |
| Oracle Net | `Test-Path tnsnames.ora` and `sqlnet.ora` | Both `True` |
| Database mapping | Query `DBA_USERS` as ADMIN | `EXTERNAL` and expected external name |
| SQL\*Plus | `sqlplus /@<ADB_TLS_ALIAS>` | Session opens without a password |
| Authentication | `SYS_CONTEXT('USERENV','AUTHENTICATION_METHOD')` | `KERBEROS` |

## 13. Troubleshooting

### ADB GUID or `KINSTANCE` is unavailable

Cause: the ADB has not been created, is not `AVAILABLE`, or the ADMIN connection is not working.

Action:

1. Verify the ADB lifecycle state.
2. Verify that WIN-Client can resolve and reach the private endpoint.
3. Verify that the ADB allows TLS/non-mTLS.
4. Connect as ADMIN through the private TLS alias.
5. Run the two `V$PDBS` queries in section 7.3.

### Private endpoint resolves to a public address

Cause: WIN-Client is using a public resolver, the AD forwarder is incorrect, or the wrong ADB hostname is being used.

Action: set WIN-Client DNS to WIN-AD, configure the AD DNS forwarder, flush the cache, and verify the exact private endpoint FQDN from OCI.

### TCP connection to ADB fails

Cause: private route, NSG/security list, Windows Firewall, DNS, or incorrect TLS port.

Action: use the exact private TLS connection information from OCI, verify the ADB NSG/security list, and test the specified `<ADB_TLS_PORT>`.

### TLS connection fails after the 19.3 install

Cause: the 19.3 base client is being used for a Windows OCI TLS/no-wallet connection, or `WALLET_LOCATION` is still active.

Action:

1. Confirm that `sqlplus -V` reports 19.14 or later.
2. Confirm the Windows patch was applied to the same `ORACLE_HOME` used by SQL\*Plus.
3. Remove or comment out `WALLET_LOCATION`.
4. Use the TLS, not mTLS, private connection string.

### Duplicate SPN

Cause: the service principal is attached to more than one AD account.

Action: identify the intended service account, remove the duplicate through directory change control, and rerun `setspn -Q`.

### No TGT or service ticket

Cause: local Windows logon, expired ticket, incorrect DNS, unavailable KDC, clock skew, or wrong realm.

Action: sign in interactively with the domain user, run `klist`, verify WIN-AD DNS and port 88, synchronize time, and recreate the FILE cache with `ms2mit.exe`.

### `ORA-12638` or `ORA-12641`

Cause: Oracle cannot initialize or read the selected Kerberos provider/cache.

Action: verify the patched full client, `SQLNET.KERBEROS5_CONF`, `SQLNET.KERBEROS5_CONF_MIT`, `SQLNET.KERBEROS5_CC_NAME`, MIT tools, and the user context. Test from the same interactive session that owns the cache.

### `ORA-01017` after a successful service ticket

Cause: the database external name does not exactly match the Kerberos principal, or password fallback occurred.

Action: keep `SQLNET.FALLBACK_AUTHENTICATION=FALSE`, inspect `DBA_USERS`, compare case and realm, and rerun the test.

### SQL\*Plus prompts for a database password

Cause: Kerberos is not selected, the cache is not readable, the wrong Oracle home is active, or the connection uses the wrong alias.

Action: verify `sqlnet.ora`, `TNS_ADMIN`, `ORACLE_HOME`, the active FILE cache, `kvno`, and the patched SQL\*Plus version.

## 14. Security and production requirements

This guide intentionally describes a test environment. Before production use:

- use private subnets and controlled administration access;
- use at least two domain controllers with monitored replication;
- use minimum OCI security rules and NSG rules;
- use a managed service-account lifecycle and rotate its secret;
- restrict the Object Storage bucket and delete `v5srvtab` after import;
- do not map `ADMIN` for application use;
- protect the Kerberos cache with NTFS ACLs;
- keep passwords, keytabs, PARs, private keys, and wallet material out of source control and command history;
- retain `SQLNET.FALLBACK_AUTHENTICATION=FALSE` for validation;
- use current Oracle Client and Windows patch levels approved for the target environment.

## 15. Final implementation record

Store these values outside this reusable guide:

- OCI OCIDs and private IPs;
- ADB private endpoint FQDN and TLS port;
- ADB GUID;
- `KINSTANCE`;
- complete Kerberos service principal;
- SPN owner;
- keytab hash if required;
- database external-user mapping;
- Oracle Client and patch versions;
- validation timestamp and command results.

Never store the following in this document:

- AD passwords;
- ADB passwords;
- service-account passwords;
- wallet passwords;
- PAR URLs;
- keytab contents;
- Kerberos ticket contents.

## 16. References

- [Oracle: Configure Kerberos Authentication with Autonomous AI Database](https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/mdw-configure-kerberos-authentication-autonomous-database.html)
- [Oracle: Configure Network Access with Private Endpoints](https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/private-endpoints-autonomous.html)
- [Oracle: Allow TLS or require only mTLS authentication](https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/support-tls-mtls-authentication.html)
- [Oracle: Download database connection information](https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/connect-download-wallet.html)
- [Oracle: Connect SQL\*Plus without a wallet](https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/connect-sqlplus-tls.html)
- [Oracle: Prepare OCI connections using TLS authentication](https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/connect-prepare-oci-tls.html)
- [Oracle: OCI TLS connections](https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/connect-oci-tls.html)
- [Oracle: Connect to an Oracle Database Server Authenticated by Kerberos](https://docs.oracle.com/en/database/oracle/oracle-database/19/dbseg/connecting-oracle-database-server-authenticated-kerberos.html)
- [Oracle: Oracle Database 19c Kerberos configuration parameters](https://docs.oracle.com/en/database/oracle/oracle-database/19/dbseg/oracle-database-parameters-used-kerberos-configuration.html)
- [Oracle: Perform external user authentication tasks on the client](https://docs.oracle.com/en/database/oracle/oracle-database/19/ntqrf/performing-external-user-authentication-tasks-on-the-oracle-client-computer.html)
- [Microsoft: Install Active Directory Domain Services](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/deploy/install-active-directory-domain-services--level-100-)
- [Microsoft: Windows Time Service tools and settings](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings)
- [Microsoft: `ktpass` command](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/ktpass)
- [Microsoft: `setspn` command](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn)

Document date: 2026-10-05

# Install xRDP on Oracle Linux 7 # 

The following I created after a request from my friend and colleague Valentin Tabacaru..

## Disclaimer ##
The following is intended to outline our general product direction. It is intended for information purposes only, and may not be incorporated into any contract. It is not a commitment to deliver any material, code, or functionality, and should not be relied upon in making purchasing decisions. The development, release, and timing of any features or functionality described for Oracle’s products remains at the sole discretion of Oracle.

## Requirements ##

- Running Oracle Linux 7 image on OCI
- Logged in as user `opc`

## Install GUI environment ##

In a default Linux 7 image on OCI, there is no desktop installed. So before we can do anything with VNC or xRDP, we need to install a GUI in the image:

````
$ <copy>sudo -s</copy>
````

````
<copy>${ORACLE_HOME}/bin/sqlplus -s '/ as sysdba' << EOF
 alter user sys identified by "${NEWPASS}" container=all;
EOF</copy>
````

Next we can have the YUM package manager install the files for the GUI:

````
# <copy>yum -y groupinstall "Server with GUI"</copy>

Loaded plugins: langpacks, ulninfo
Resolving Dependencies
--> Running transaction check

(removed long list)
  libreport-python.x86_64 0:2.1.11-53.0.1.el7
  libreport-web.x86_64 0:2.1.11-53.0.1.el7
  python-firewall.noarch 0:0.6.3-8.0.1.el7

Complete!
````

## Install required packages from repo ##

In the default Linux 7, the required repositories are available and activated so we can immediately start the installation of the required packages, including the xRDP package:

````
# <copy>yum -y install xrdp tigervnc-server terminus-fonts terminus-fonts-console cabextract</copy>

Loaded plugins: langpacks, ulninfo
Resolving Dependencies

<removed long list)
Installed:
  cabextract.x86_64 0:1.5-1.el7
  terminus-fonts.noarch 0:4.38-3.el7
  terminus-fonts-console.noarch 0:4.38-3.el7
  tigervnc-server.x86_64 0:1.8.0-19.0.1.el7
  xrdp.x86_64 1:0.9.5-1.el7

Dependency Installed:
  xorgxrdp.x86_64 0:0.2.11-1.0.1.el7

Complete!
````

## Download and install additional packages ##

Sometimes the fonts used through VNC and xRDP are distorted and difficult to read. This is because some developers have used Windows compatible fonts in their applications. The SourceForge project MSCoreFonts has a package with recent fonts that can be used on our image:

```
# <copy>yum -y localinstall https://downloads.sourceforge.net/project/mscorefonts2/rpms/msttcore-fonts-installer-2.6-1.noarch.rpm</copy>

Loaded plugins: langpacks, ulninfo
msttcore-fonts-installer-2.6-1.noarch.rpm                |  29 kB     00:00
Examining /var/tmp/yum-root-H66ZIW/msttcore-fonts-installer-2.6-1.noarch.rpm: msttcore-fonts-installer-2.6-1.noarch
Marking /var/tmp/yum-root-H66ZIW/msttcore-fonts-installer-2.6-1.noarch.rpm to be installed
Resolving Dependencies
--> Running transaction check
---> Package msttcore-fonts-installer.noarch 0:2.6-1 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package              Arch   Version
                                   Repository                              Size
================================================================================
Installing:
 msttcore-fonts-installer
                      noarch 2.6-1 /msttcore-fonts-installer-2.6-1.noarch  24 k

Transaction Summary
================================================================================
Install  1 Package

(removed long list)
  Verifying  : msttcore-fonts-installer-2.6-1.noarch                        1/1

Installed:
  msttcore-fonts-installer.noarch 0:2.6-1

Complete!
````

We can now configure system and the xRDP package !

## Configure the system ##

### Configure xRDP ###

First, we change the xrdp.ini file to make the remote desktop resolution compatible with the users. If you fail to change this setting, you might end-up with a black screen or a screen that is too small to work:

````
# <copy>vi /etc/xrdp/xrdp.ini</copy>
````

Do the following in this file:

- Under `[Globals]` change `max_bpp` to 128
- Add `<copy>use_compression=yes<copy>` below the `max_bpp` 

Save the xrdp.ini file and exit the editor.

To inform the system that xRDP should start after every reboot, we can enable it through systemctl:

````
# <copy>systemctl enable xrdp</copy>

Created symlink from /etc/systemd/system/multi-user.target.wants/xrdp.service to /usr/lib/systemd/system/xrdp.service.
````

### Configure the firewall ###

Execute the following command to enable the firewall to accept requests from port 3389:

````
# <copy>firewall-cmd --permanent --add-port=3389/tcp</copy>
success
````

Reload the firewall for the changes to take effect

````
# <copy>firewall-cmd --reload</copy>
success
````

### Configure SELinux ###

The default image for Oracle Linux 7 on OCI has SELinux enabled:

````
# <copy>cat /etc/selinux/config</copy>

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=enforcing
# SELINUXTYPE= can take one of three values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
````

If you want to keep SELinux enabled, you need to change the settings so SELinux sees xRDP as legitimate:

````
# <copy>chcon --type=bin_t /usr/sbin/xrdp</copy>
````
````
# <copy>chcon --type=bin_t /usr/sbin/xrdp-sesman</copy>
````

Be aware that the above commands only enable xRDP functionalty. The Desktop GUI install will automatically start other programs as well (Location services, AVC) which trigger SELinux. Also other application like FireFox will trigger SELinux as well and cause a pop-up on the users desktop.

### Optional changes ###

#### Auto updates through PackageKit ####

By default, the system checks regularly if there are any updates for the packages you use. To prevent the automatic updates or the pop-ups on the screen, remove PackageKit from the system:

````
# <copy>yum -y remove PackageKit*</copy>

Loaded plugins: langpacks, ulninfo
Resolving Dependencies
--> Running transaction check
---> Package PackageKit.x86_64 0:1.1.10-2.0.1.el7 will be erased

================================================================================
 Package                        Arch     Version            Repository     Size
================================================================================
Removing:
 PackageKit                     x86_64   1.1.10-2.0.1.el7   @ol7_latest   2.6 M
 PackageKit-command-not-found   x86_64   1.1.10-2.0.1.el7   @ol7_latest    44 k
 PackageKit-glib                x86_64   1.1.10-2.0.1.el7   @ol7_latest   484 k
 PackageKit-gstreamer-plugin    x86_64   1.1.10-2.0.1.el7   @ol7_latest    20 k
 PackageKit-gtk3-module         x86_64   1.1.10-2.0.1.el7   @ol7_latest    22 k
 PackageKit-yum                 x86_64   1.1.10-2.0.1.el7   @ol7_latest   301 k
Removing for dependencies:
 gnome-initial-setup            x86_64   3.28.0-2.el7       @ol7_latest   2.5 M
 gnome-packagekit               x86_64   3.28.0-1.el7       @ol7_latest   0.0
 gnome-packagekit-common        x86_64   3.28.0-1.el7       @ol7_latest   6.4 M
 gnome-packagekit-installer     x86_64   3.28.0-1.el7       @ol7_latest   202 k
 gnome-packagekit-updater       x86_64   3.28.0-1.el7       @ol7_latest   194 k
 gnome-software                 x86_64   3.28.2-3.el7       @ol7_latest   8.4 M

(Removed long list)
Dependency Removed:
  gnome-initial-setup.x86_64 0:3.28.0-2.el7
  gnome-packagekit.x86_64 0:3.28.0-1.el7
  gnome-packagekit-common.x86_64 0:3.28.0-1.el7
  gnome-packagekit-installer.x86_64 0:3.28.0-1.el7
  gnome-packagekit-updater.x86_64 0:3.28.0-1.el7
  gnome-software.x86_64 0:3.28.2-3.el7

Complete!
````



### Start xRDP ###

After everything has been configured, we can start xRDP:

````
# <copy>systemctl start xrdp</copy>
````

## Test xRDP for the first time ##

### Set password for the connecting user ###

The xRDP session authenticates against the operating system. Since the OL7 installations on OCI only use public/private keys for authentication, we have never set a password for any user.

In the actual environment, first create a user with the required rights and groups. In this example, I will use the `root` user (but this should be avoided in real setups).

Change a password for a user:

````
# <copy>passwd root</copy>

Changing password for user root.
New password:
Retype new password:
````

### Connect to xRDP ###

Now you can connect to your remote system using the IP address for that system and the username/password combination you have just setup

## Acknowledgements ##

- **Author** - Robert Pastijn, Database Product Management, PTS EMEA - April 2020


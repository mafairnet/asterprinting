Aloha IVR-AVP
===================
Inbound Voice Recording Automatic Verification Platform

![Aloha Dashboard](http://maf.mx/astricon/2017/images/aloha_interface.png)

About
-----------
#### Objective: Validate TelCo Services

Sometimes the telephony companies make mistakes when they are routing your calls to your Asterisk PBX. Sometimes calls are routed to another number or you have Toll-Free number not routing as they supposed to be routed.

The Aloha platform helps you to check automatically that configuration hourly, daily, weekly, monthly or yearly and it's very simple to use.

When you program a test you can add one or more numbers to check. The platform will dial the numbers and create a recording of the call. This recording will be compared to your existing IVR audios of you PBX or other PBX that are stored in a database.

![Audio FingerPrinting](http://www.maf.mx/astricon/2017/images/spectrogram_peaks.png)

The method to compare the recording and the IVR audios database is called Audio Fingerprinting (We use the Dejavu Audio Fingerprinting Python Library). The audios area compared and if the confidence (number of point matching between audio fingerprintings) is bigger you have a perfect match between audios.

More info about this project and how to deal with TelCo at http://www.maf.mx/astricon/2017/.
Also you can watch the next video (https://www.youtube.com/watch?v=M9Nkh4mp2fI&t=1052s) where Chris from Crosstalk solutions talks about the Aloha project (minute 7:00).

Prerequisites
-----------
- CentOS 6.7-x86_64-minimal ISO (Oreka)
- CentOS 7-x86_64-minimal ISO (Aloha/Dejavu)
- Dejavu (Python AudioFingerPrinting Library)
- Oreka OpenSource
- Python 2.6
- MySQL Server (5.1 or later)

Installation
-----------
This system needs 3 separate servers to work.
1. Free PBX
2. Oreka OpenSource
3. Aloha Server (with Dejavu (Audio Fingerprinting) Python Module)

#### System Architecture Diagram
![Aloha System Architecture](http://maf.mx/astricon/2017/images/aloha_system_architecture.png)

### Oreka OpenSource Server
Before you start you need two network interfaces on the Oreka Server. One will be to access the server and manage it. The other one will be the interface where we are going to receive all the VoIP traffic through a PortMirroring Configuration (AKA Promisc). Also the minimun requirements for the server are 2 cores in the processor and 2GB of RAM. 

1. Install CentOS 6 on a physical server or virtual server.
2. Disable SELinux (SELINUX=disabled)
```
yum install nano -y
nano /etc/sysconfig/selinux
reboot
```
3. Install Setup Tools and configure network devices to start at boot.
```
yum -y update
yum install setuptool system-config-network* system-config-firewall* system-config-securitylevel-tui system-config-keyboard ntsysv -y
setup
```
4. Install OrekaAudio Open Source with the next steps:
```
yum install -y git
cd /opt
git clone https://github.com/mafairnet/aloha.git
cd aloha/oreka/
tar xvf orkaudio-1.7-844-os-x86_64.centos6-installer.sh.tar
tar xvf orkweb-1.7-838-x64-os-linux-installer.sh.tar
chmod +x orkaudio-1.7-844.x86_64-x-x86_64.centos6-installer.sh
chmod +x orkweb-1.7-838-x64-os-linux-installer.sh
./orkaudio-1.7-844.x86_64-x-x86_64.centos6-installer.sh
```
5. Complete orkaudio installation proccess. On the installation process you must consider:
   1. Select the option i to continue installation process.
   2. Indicate that license file willl be installed later with 'l'.
6. Install MySQL Server.
```
yum install mysql-server -y
chkconfig mysqld on
service mysqld start
mysql_secure_installation
```
7. Log into the MySQL Server and create an empty DB called oreka.
```
create database oreka;
```
8. Install OrkWeb Open Source with the installer:
```
./orkweb-1.7-838-x64-os-linux-installer.sh
```
9. Edit OrkWeb web.xml
```
nano /opt/tomcat7/webapps/orkweb/WEB-INF/web.xml
```
10. Edit the ConfigDirectory param-value set "/opt/oreka/"
```
<param-name>ConfigDirectory</param-name>
<param-value>/opt/oreka/</param-value>
```
11. Edit OrkTrack web.xml
```
nano /opt/tomcat7/webapps/orktrack/WEB-INF/web.xml
```
12. Edit the ConfigDirectory param-value set "/opt/oreka/"
```
<param-name>ConfigDirectory</param-name>
<param-value>/opt/oreka/</param-value>
```
13. Copy config files to /opt/oreka
```
mkdir /opt/oreka
cd /opt/aloha/oreka/opt/
cp * /opt/oreka/
```
14. Modify the /opt/oreka/database.hbm.xml file and add your DB credentials.
```
<property name="hibernate.connection.url">jdbc:mysql://localhost/oreka</property>
<property name="hibernate.connection.password">PASSWORD</property>
<property name="hibernate.connection.username">USER</property>
```
15. Reconfigure iptables for access the OrkWeb Server
```
iptables -A INPUT -m state --state NEW -m tcp -p tcp --dport 8080 -j ACCEPT -m comment --comment "Tomcat Server port"
service iptables save
service iptables restart
iptables -F
```
16. Edit the configuration of the OrkAudio file "/etc/orkaudio/config.xml" and set de values of the next configurations.
```
<AudioOutputPath>/opt/tomcat7/webapps/ROOT</AudioOutputPath>
<StorageAudioFormat>pcmwav</StorageAudioFormat>
<!--In this case the network interface receiving all tha through port mirroring is eth1-->
<Devices>eth1</Devices>
```
17. Configure OrkAudio start at boot and start OrkAudio Service
```
chkconfig orkaudio on
service orkaudio start
```
18. Login to MySQL and add a field to the oreka.orktape table.
```
alter table oreka.orktape `aloha_processed` binary(1) DEFAULT '0';
```
19. Go to OrkWeb at http://SERVERIP:8080/orkweb/ and test calls are being recorded.
### Aloha Server
1. Install CentOS 7 Minimal on a physical server or virtual server.
2. Disable SELinux (SELINUX=disabled)
```
yum install nano -y
nano /etc/sysconfig/selinux
reboot
```
3. Install Setup Tools and configure network devices to start at boot.
```
yum -y update
nmtui
```
4. Install HTTPD, PHP and MySQL.
```
yum install httpd mariadb-server php php-mysql -y
systemctl enable httpd.service
systemctl enable mariadb.service
systemctl start mariadb.service
systemctl start httpd.service
mysql_secure_installation
```
5. Install the dejavu audiofingerprinting library and dependencies
```
yum install epel-release -y
yum groupinstall -y "development tools"
yum install gcc gcc-c++ kernel-devel python-devel libxslt-devel libffi-devel openssl-devel portaudio-devel zlib-devel bzip2-devel ncurses-devel sqlite-devel readline-devel tk-devel gdbm-devel db4-devel libpcap-devel xz-devel expat-devel numpy scipy python-matplotlib ffmpeg portaudio-devel python-pip wget MySQL-python tkinter  -y
pip install --upgrade pip
pip install PyAudio
pip install pydub
pip install PyDejavu
```
6. Clone the repository to your custom PBX and copy the aloha scripts and webapp.
```
yum install -y git
cd /opt
git clone https://github.com/mafairnet/aloha.git
cd /opt/aloha/webapp/
yes | cp -rf * /var/www/html/
cd /opt/aloha/aloha_dejavu_scripts/
mkdir /opt/dejavu/
cp /opt/aloha/aloha_dejavu_scripts/dejavu.cnf /opt/dejavu/
mkdir /opt/aloha/bin/
cp * /opt/aloha/bin/
cd /opt/aloha/bin/
chmod +x aloha_worker.py
chmod +x aloha_dejavu_fingerprinting.py
chmod +x aloha_call_audiofingerprinting.py
```
7. Login to MySQL and create the aloha database and dejavu db.
```
CREATE DATABASE IF NOT EXISTS dejavu;
CREATE DATABASE IF NOT EXISTS aloha;
```
8. Run the DB installation script.
```
mysql -u db_user -p aloha < /opt/aloha/database/aloha-db-install.sql
```
9. Edit the Dejavu configuration file "/opt/dejavu/dejavu.cnf" to set your DB Credentials:
```
{
    "database": {
        "host": "127.0.0.1",
        "user": "user",
        "passwd": "password", 
        "db": "dejavu"
    }
}
```
10. Edit the Aloha Call Generator Script "aloha_worker.py" to set your DB Credentials:
```
fileServer = "http://OREKASERVERIP:8080/"
db = MySQLdb.connect("OREKASERVERIP","DBUSER","PASSWORD","oreka" )
```
11. Edit the Aloha Frequency Check Script "aloha_call_audiofingerprinting.py" to set your DB Credentials:
```
fileServer = "http://OREKASERVERIP:8080/"
db = MySQLdb.connect("ALOHASERVERIP","DBUSER","PASSWORD","aloha" )
db = MySQLdb.connect("OREKASERVERIP","DBUSER","PASSWORD","oreka" )
```
12. Edit the DB config file for te Aloha WebApp
```
$GLOBALS['aloha_db_server']   = '0.0.0.0';
$GLOBALS['aloha_db_username'] = 'db_user';
$GLOBALS['aloha_db_password'] = 'db_pass';
$GLOBALS['aloha_db_name']    = 'aloha';

$GLOBALS['dejavu_db_server']   = '0.0.0.0';
$GLOBALS['dejavu_db_username'] = 'db_user';
$GLOBALS['dejavu_db_password'] = 'db_pass';
$GLOBALS['dejavu_db_name']    = 'dejavu';

$GLOBALS['aloha_recorder_db_server']   = '0.0.0.0';
$GLOBALS['aloha_recorder_db_username'] = 'db_user';
$GLOBALS['aloha_recorder_db_password'] = 'db_pass';
$GLOBALS['aloha_recorder_db_name']    = 'oreka';
```
12. Add your scripts to the crontab table with "crontab -e" and add the following lines:
```
* * * * * /opt/aloha/bin/aloha_worker.py
* * * * * /opt/aloha/bin/aloha_dejavu_fingerprinting.py
* * * * * /opt/aloha/bin/aloha_call_audiofingerprinting.py
```
13. Allow HTTP access to the servers
```
firewall-cmd --zone=public --add-port=80/tcp
```
14. Go to the Aloha Dashboard and check that the platform show data on the catalogs like "Status" at http//ALOHASERVERIP/ the default user is "admin" and the password is "pass".
### PBX Server
1. Install on other server AsteriskNow Distro with FreePBX13. You can download it from http://downloads.asterisk.org/pub/telephony/asterisk-now/AsteriskNow-1013-current-64.iso
2. Edit the /etc/asterisk/extensions_custom.conf file and add the next lines:
```
[wait-ivr]
exten => wait,1,Answer()
exten => wait,n,Wait(300)
exten => wait,n,Hangup()

[macro-aloha]
exten => s,1,NoOp(to-customer)
exten => s,n,Set(CALLERID(num)=9999)
exten => s,n,Set(CALLERID(name)=Aloha)
exten => s,n,Set(TIMEOUT(absolute)=30)
exten => s,n,Answer()
exten => s,n,Playback(/var/lib/asterisk/sounds/en/mrwhite)
```
3. Create a SIP trunk between your PBX and another PBX (This other PBX wil handle the outbound calls). The SIP Settings will be:
```
disallow=all
allow=alaw&ulaw&gsm
username=myuser-sip
type=friend
secret=PASSWORD
host=OUTBOUNDPBXIP
context=from-internal
trunk=yes
requirecalltoken=no
qualify=yes
```
5. Create the SIP Trunk also at your Outbound PBX to the Custom PBX.
```
disallow=all
allow=alaw&ulaw&gsm
username=myuser-sip
type=friend
secret=PASSWORD
host=CUSTOMPBXIP
context=from-internal
trunk=yes
requirecalltoken=no
qualify=yes
```
4. Clone the repository to your custom PBX and copy the aloha scripts
```
yum install -y git MySQL-python
cd /opt
git clone https://github.com/mafairnet/aloha.git
cd /opt/aloha/pbx_scripts/
mkdir /opt/aloha/bin/
cp * /opt/aloha/bin/
cd /opt/aloha/bin/
chmod +x aloha_call_generator.py
chmod +x aloha_frecuency_check.py
```
5. Edit the Aloha Call Generator Script "aloha_call_generator.py" to set your DB Credentials:
```
db = MySQLdb.connect("ALOHASERVERIP","aloha_user","PASSWORD","aloha" )
```
6. Edit the Aloha Frequency Check Script "aloha_frecuency_check.py" to set your DB Credentials:
```
db = MySQLdb.connect("ALOHASERVERIP","aloha_user","PASSWORD","aloha" )
```
7. Add your scripts to the crontab table with "crontab -e" and add the following lines:
```
* * * * * /opt/aloha/bin/aloha_call_generator.py
* * * * * /opt/aloha/bin/aloha_frecuency_check.py
```


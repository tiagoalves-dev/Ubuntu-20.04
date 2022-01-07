## Create an Active Directory Infrastructure with Samba4 on Ubuntu
### Table of contents

[I. Introducton](#modau)

[II. Getting Start](#batdau)
- [1. Step 1: Initial Configuration for Samba4](#step1)
- [2. Step 2: Install Required Packages for Samba4 AD DC](#step2)
- [3. Step 3: Provision Samba AD DC for Your Domain](#step3)

[III. Summary](#Tongket)

===========================

<a name="Modau"></a>
## I. Introduction

<a name="batdau"></a>
## II. Getting Start:

<a name="step1"></a>
## Step 1: Initial Configuration for Samba4

1. Open machine /etc/fstab file and assure that your partitions file system has ACLs enabled as illustrated 
Usually, common modern Linux file systems such as ext3, ext4, xfs or btrfs support and have ACLs enabled by default. If that’s not the case with your file system just open /etc/fstab file for editing and add acl string at the end of third column and reboot the machine in order to apply changes.

2. Assign a static IP to your server. Ubuntu Server uses netplan for network management. Your network configuration will look similar to this:
$ sudo vim /etc/netplan/00-installer
``` sh
network:
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
      - 192.168.30.241/24
      gateway4: 192.168.30.2
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
        search: []
  version: 2
```
- Apply the network config

> $ sudo netplan apply

- Check if time synchronization with an Internet server is working

> $ timedatectl

> $ sudo timedatectl set-timezone Asia/Ho_Chi_Minh

- Update the apt cache
>$ sudo apt update

3. Setup your machine hostname with a descriptive name, such as adc1 used in this example, by editing /etc/hostname file or by issuing.

> $ sudo vi /etc/hostname

``` sh
dc1.htu.local
```

> $ sudo vi /etc/hosts

``` sh
192.168.30.241 dc1.htu.local dc1
```

> $ sudo reboot

A reboot is necessary after you’ve changed your machine name in order to apply changes.

======
<a name="step2"></a>
## Step 2: Install Required Packages for Samba4 AD DC

4. In order to transform your server into an Active Directory Domain Controller, install Samba and all the required packages on your machine by issuing the below command with root privileges in a console.
$ sudo apt-get install samba krb5-user krb5-config winbind smbclient     #libpam-winbind libnss-winbind

``` sh
Kerberos Realm: HTU.LOCAL
Kerberos servers for your realm: dc1.htu.local
Administrative server for your Kerberos realm: dc1.htu.local
```

5. While the installation is running a series of questions will be asked by the installer in order to configure the domain controller.

On the first screen you will need to add a name for Kerberos default REALM in uppercase. Enter the name you will be using for your domain in uppercase and hit Enter to continue..

6. Next, enter the hostname of Kerberos server for your domain. Use the same name as for your domain, with lowercases this time and hit Enter to continue.
7. Finally, specify the hostname for the administrative server of your Kerberos realm. Use the same as your domain and hit Enter to finish the installation.

<a name="step3"></a>
## Step 3: Provision Samba AD DC for Your Domain

8. Next, rename or remove samba original configuration. This step is absolutely required before provisioning Samba AD because at the provision time Samba will create a new configuration file from scratch and will throw up some errors in case it finds an old smb.conf file.

> $ sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.original

9. Now, start the domain provisioning interactively by issuing the below command with root privileges and accept the default options that Samba provides you.

> $ sudo samba-tool domain provision --use-rfc2307 --interactive

``` sh
Realm [HTU.LOCAL]:
Domain [HTU]:
Server Role (dc, member, standalone) [dc]:
DNS backend (SAMBA_INTERNAL, BIND9_FLATFILE, BIND9_DLZ, NONE) [SAMBA_INTERNAL]:
DNS forwarder IP address (write 'none' to disable forwarding) [127.0.0.53]:  8.8.8.8,8.8.4.4
Administrator password:
Retype password:
```

10. Finally, rename or remove Kerberos main configuration file from /etc directory and replace it using a symlink with Samba newly generated Kerberos file located in /var/lib/samba/private path by issuing the below commands:

> $ sudo mv /etc/krb5.conf /etc/krb5.conf.original

> $ sudo cp /var/lib/samba/private/krb5.conf /etc/

11. Stop and disable the samba services and the dns resolver service

> $ sudo systemctl disable --now smbd nmbd winbind systemd-resolved

12. Unmask the SAMBA AD service

> $ sudo systemctl unmask samba-ad-dc

13. Enable Samba Active Directory Domain Controller daemons.

> $ sudo systemctl enable --now samba-ad-dc

14. At this moment Samba should be fully operational at your premises. The highest domain level Samba is emulating should be Windows AD DC 2008 R2.

It can be verified with the help of samba-tool utility.

> $ sudo samba-tool domain level show

``` sh
Domain and forest function level for domain 'DC=htu,DC=local'

Forest function level: (Windows) 2008 R2
Domain function level: (Windows) 2008 R2
Lowest function level of a DC: (Windows) 2008 R2
```

15. In order for DNS resolution to work locally, you need to open end edit network interface settings and point the DNS resolution by modifying dns-nameservers statement to the IP Address of your Domain Controller (use 127.0.0.1 for local DNS resolution) and dns-search statement to point to your realm.

Recreate the dns nameserver file

> $ sudo rm -f /etc/resolv.conf && sudo vi /etc/resolv.conf

``` sh
nameserver 127.0.0.1
domain htu.local
```

> $ sudo cat /etc/network/interfaces

> $ sudo cat /etc/resolv.conf

When finished, reboot your server and take a look at your resolver file to make sure it points back to the right DNS name servers.

16. Finally, test the DNS resolver by issuing queries and pings against some AD DC crucial records, as in the below excerpt. Replace the domain name accordingly.

> $ ping -c3 htu.local         #Domain Name

> $ ping -c3 dc1.htu.local   #FQDN

> $ ping -c3 dc1               #Host

Run following few queries against Samba Active Directory Domain Controller..

> $ host -t A htu.local

> $ host -t A dc1.htu.local

> $ host -t SRV _kerberos._udp.htu.local  # UDP Kerberos SRV record

> $ host -t SRV _ldap._tcp.htu.local # TCP LDAP SRV record

17. Also, verify Kerberos authentication by requesting a ticket for the domain administrator account and list the cached ticket. Write the domain name portion with uppercase.

> $ kinit administrator@HTU.LOCAL

> $ klist

That’s all! Now you have a fully operational AD Domain Controller installed in your network and you can start integrate Windows or Linux machines into Samba AD.

On the next series we’ll cover other Samba AD topics, such as how to manage you’re the domain controller from Samba command line, how to integrate Windows 10 into the domain name and manage Samba AD remotely using RSAT and other important topics.

<a name="tongket"></a>
## III. Summary:

**Watch Video here:** 

- [How to Install and configure Samba on Ubuntu 20.04 Part 1:  Public folder](https://youtu.be/2o5zgA8ml38)
- [How to configure and Install Samba on Ubuntu 20.04 Part2: Private Share](https://youtu.be/6s9ZEp3xS94)

Contact me:
- Email: manhhungbl@gmail.com
- Skype: spyerx3
- Youtube Channel: youtube.com/howtoused

Thank you very much!

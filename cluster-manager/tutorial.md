# Gluu Cluster Manager Tutorial
## Overview
This tutorial will demonstrate the creation of a two-node Gluu Server cluster with Gluu Cluster Manager at a high level. For more technical information, see https://gluu.org/docs/cm/.

## Gluu Cluster Manager
Gluu Cluster Manager (GCM) is web-based, free open source software used to create and manage clustered Gluu Server. It also allows you to monitor various system and LDAP status with graphics. Download the latest version from the Github repository: https://github.com/GluuFederation/cluster-mgr.

## Prerequisites
In this tutorial, you will create a two-node Gluu Server cluster using Virtual Box VMs on Ubuntu 16. To do so, four machines are needed, shown in the table below. My VM hosts are as follows:

|Name | FQN Hostname  | IP Address    | OS Type   | minimum RAM | Usage|
|---- | -----         | ----          | ----      | ------      | ---- |
| CM  | cm.mygluu.org |198.168.56.150 | Ubuntu 16 | 1GB         | GCM will be installed on this machine. Manages nodes via SSH. |
| LB | lb.mygluu.org  |192.168.56.200 | Ubuntu 16 | 1GB         | This machine will be used as a load balancer |
| GS1| gs1.mygluu.org |192.168.56.201 | Ubuntu 16 | 4GB         | Gluu Server Node 1 |
| GS2| gs2.mygluu.org |192.168.56.202 | Ubuntu 16 | 4GB         | Gluu Server Node 2 |

!!! note  
    The concepts demonstrated in this tutorial will work on other virtual or physical machines and different [supported operating systems](https://www.gluu.org/docs/ce/installation-guide/#supported-operating-systems), but exact settings and commands may vary.
    
Each machine has the following entries in the `/etc/hosts` file:

```
192.168.56.150 cm.mygluu.org
192.168.56.200 lb.mygluu.org
192.168.56.201 gs1.mygluu.org
192.168.56.202 gs2.mygluu.org
```

After setup, the structure looks like this:

**image placeholder**

For this tutorial, there is no firewall to protect the servers. For ports, see https://gluu.org/docs/cm/installation/#ports .
### Prepare VMs

To get the VMs , follow these steps:

- Created one fresh Virtual Box Machine with 1GB RAM
- Installed Ubuntu 16.04 LTS.
- Change Adapter 1’s network type to `Host-only adapter` to allow communication between the desktop and each server via a private network. 

**image placeholder**

- Clone this VM four times, labeled according to the image below. Use Linked Clone to disk save space:

**image placeholder**

- Change the RAM size for GS1 and GS2 to 4GB.
- Since the VMs need internet access and are configured to `Host-only network`, I need my host machine to NAT these VMs. To do so, execute these commands on the host machine (not the VMs):

```
  root@debian:~# sysctl -w net.ipv4.ip_forward=1
  root@debian:~# sudo iptables -t nat -A POSTROUTING -o wlo1 -j MASQUERADE
```

  In this case, my host machine is reaching the internet via my WiFi adapter, `wlo1`. Substitute this with your own. I booted up each machine and write the following to `/etc/network/interfaces`
  
```
auto enp0s3
iface enp0s3 inet static
    address 192.168.56.150
    netmask 255.255.255.0
    network 192.168.56.0
    broadcast 192.168.56.255
    gateway 192.168.56.1
    dns-nameservers 192.168.1.1
```

- Change the address for each machine accordingly, as well as the hostname:

```
echo “cm” > /etc/hostname
```

- Finally, reboot the machine to enable the new settings.

## Create User foo 

Create a user named foo on all servers. For example, log in to LB as root and create the user:

```
root@lb:~# adduser foo
Adding user `foo' ...
Adding new group `foo' (1001) ...
Adding new user `foo' (1001) with group `foo' ...
Creating home directory `/home/foo' ...
Copying files from `/etc/skel' ...
Enter new UNIX password: 
Retype new UNIX password: 
passwd: password updated successfully
Changing the user information for foo
Enter the new value, or press ENTER for the default
	Full Name []: 
	Room Number []: 
	Work Phone []: 
	Home Phone []: 
	Other []: 
Is the information correct? [Y/n] y
```

### Generate SSH Public Key on CM 
GCM will manage all servers via SSH without a password. So we need an SSH public key created on CM and distributed to LB, GCS1 and GCS2.
Log in to CM as foo and create an SSH public key:

```
foo@cm:~$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/home/foo/.ssh/id_rsa): 
Created directory '/home/foo/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again: 
Your identification has been saved in /home/foo/.ssh/id_rsa.
Your public key has been saved in /home/foo/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:qDi+YD/tN5MU8xudNyAxi6MRsnCEl3nEvrSebQhDt+0 foo@cm
The key's randomart image is:
+---[RSA 2048]----+
|  o.=.           |
| o * +   o       |
|  + = . . +      |
|   o = * o .     |
|  . o O S o o    |
|   + * o o o o   |
|..o * * . o . .  |
|.o.o = E .       |
|  ooo.o o        |
+----[SHA256]-----+
```

If a passphrase is provided, GCM will ask for it before establishing SSH to the servers. 
The new user, foo, needs to be able to modify GCM. Give it passwordless `sudo` authorization by logging in to GCM as root and execute the following command:  

`root@cm:~# echo "foo ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers`

### Copy CM’s Public Key to Other Servers
Log in to CM as root and copy the public key to  the other servers:

```
foo@cm:~$ scp /home/foo/.ssh/id_rsa.pub foo@lb.mygluu.org:/tmp
foo@cm:~$ scp /home/foo/.ssh/id_rsa.pub foo@gs1.mygluu.org:/tmp
foo@cm:~$ scp /home/foo/.ssh/id_rsa.pub foo@gs2.mygluu.org:/tmp
```

### Add CM’s Public Key to authorized_keys on Other Servers

Add CM’s public key to /root/.ssh/authorized_keys on LB, GS1 and GS2. For example, log in to LB as root and:

```
root@lb:~# mkdir -p /root/.ssh/
root@lb:~# chmod 700 /root/.ssh/
root@lb:~# cat /tmp/id_rsa.pub >> /root/.ssh/authorized_keys 
```

Test on CM if foo can log in to GS1 via SSH without a password:

```
foo@cm:~$ ssh root@gs1.mygluu.org
Welcome to Ubuntu 16.04.5 LTS (GNU/Linux 4.4.0-131-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

121 packages can be updated.
90 updates are security updates.
```

## Install Gluu Cluster Manager

When finished preparing the servers, it’s time to install GCM. 

1. Log in to CM as root and install dependencies with:

    ```
    root@cm:~# apt-get update
    root@cm:~# apt-get install python-pip python-dev libffi-dev libssl-dev python-ldap redis-server default-jre
    root@cm:~# pip install --upgrade setuptools influxdb psutil
    ```
    
1. Install GCM:

    ```
    root@cm:~# pip install clustermgr
    ```
    
1. It’s not recommended to run GCM as root, so log in as `foo` and start GCM:

    ```
    foo@cm:~$ clustermgr-cli start
    ```
    
  - To later stop GCM, run this command:
    ```
    $ clustermgr-cli stop
    ```
1. For security reasons, GCM will only listen on localhost:5000. To be able to use GCM, we need to set up SSH port forwarding from the host computer. Execute the following command on the host machine:

    ```
    mbaser@debian:~$ ssh -L 5000:localhost:5000 foo@cm.mygluu.org
    foo@cm.mygluu.org's password: 
    ```
    
GCM will display errors most of the time. If you have something unexpected, please see log files under `/home/foo/.clustermgr/logs`

### Sign Up Cluster Manager

On your desktop computer, open a browser (Firefox or Chrome) and navigate to http://localhost:5000
The first step is signup GCM. Signup will be performed only once. Choose a username and a password. I chose `gcmuser`

**image placeholder**

On subsequent visits to GCM, you will use this username and password. In case you forget username or password, you can delete `/home/foo/.clustermgr/auth.ini` file. No other settings/configurations will be damaged by deleting this file.

### Configure Application

Second step is to choose the clustering type. Since we are going to create a new cluster with two nodes, I choose `Create a New Gluu Server Cluster`. After hitting this button, we will be directed to Application Settings. On this page we need to enter these values:

**Replication Manager Password**: This password will be used GCM to set replication between the OpenDJ servers and mostly you won’t use this password yourself. Choose a strong password.
**Load Balancer Hostname**: In our case it is lb.mygluu.org
**Load Balancer IP Address**: In our case it is 192.168.56.200
We also checked **Add IP Addresses and hostnames to /etc/hosts file on each server**

**image placeholder**

Since we are going to use LDAP Cache, we left this option checked. If you prefer redis cache, please unchecked it. In that case, you should configure caches on **Cache Management** menu after LDAP replication is configured.
Next step is to Add Primary Server.

**image placeholder**

Add a second server as follows

**image placeholder**

Your Dashboard should look similar to the following figure. At any time you can add a new Gluu Server node to your cluster so that you can enlarge the cluster horizontally.

**image placeholder**

The Dashboard will also display the health status of major Gluu components (LDAP, oxAuth and Identity) after installation.
We can install Gluu Servers by clicking Install Gluu button. First we should install primary server. When click, it will be asked some trivial info for creating ssl certificates, license agreements and Gluu Server Components to be installed. Click Submit button, after over-viewing setup properties, hit **Install Gluu Server** button.
Gluu Server will be installed and configured so that it can work in cluster:

**image placeholder**

Note that hostname will be set to lb.mygluu.org, all nodes hostname will be set to load balancer’s hostname, since the world will interact only with load balancer. After installation of all servers is finished, all status should be green in Service Liveness Status column on Dashboard.

### Install Nginx

The Nginx server will be installed on load balancer (LB). To install Nginx, choose `Cluster > Install Nginx` from the left menu.

**image placeholder**

After installation of Nginx finished, hit Go to LDAP Replication button

### Enable Replication

If you hit the green button after nginx, you will jump to replication page. Alternatively you can reach this page by Replication > LDAP from the left menu. Hit Deploy All button. This will configure ldap replication on all servers and will make configurations on each server so that they become a node of cluster. After configuration of replication finished, GCM will display status of replication:

**image placeholder**

On the page, if you can see Entires matches across the server and all is zero on M.C. column, then replication is good. 
This is the final step for creating Gluu Server Cluster. You can test if your cluster is working. For this I follow these steps:
1. Stop one of the Gluu Servers. I choose to stop GS2, so login as root and:  
`# /etc/init.d/gluu-server-3.1.4 stop`
1. Login to Gluu UI via LB (lb.mygluu.org) with browser and add a test person: testuser

  **image placeholder**

Since GS2 is not working while you are adding person, this person is created on GS1
1. Start Gluu Server on GS2:  
`# root@lb:~# /etc/init.d/gluu-server-3.1.4 start`  
1. After a few seconds, stop Gluu Server on GS1:  
`# /etc/init.d/gluu-server-3.1.4 stop` 
1. Then login Gluu UI with testuser via LB (lb.mygluu.org). If login is successful you are lucky.

**image placeholder

Since GS1 is not working, you are logged in GS2 which was not working while you are creating testuser. So cluster is working successfully.
Before going to optional configurations please start Gluu Server on GS1:  
`# root@lb:~# /etc/init.d/gluu-server-3.1.4 start`

### Monitoring (optional)

GCM can be used for monitoring your servers in the cluster. To setup monitoring, click `Monitoring > Install Monitoring` from the left menu. In the upcoming page, hit `Install Monitoring` button. You may get the following error, don’t worry it won’t break things:

```
An error occurred while executing pip install pyDes: /usr/local/lib/python2.7/dist-packages/pip/_vendor/requests/__init__.py:83: RequestsDependencyWarning: Old version of cryptography ([1, 2, 3]) may cause slowdown. warnings.warn(warning, RequestsDependencyWarning) DEPRECATION: Python 2.7 will reach the end of its life on January 1st, 2020. Please upgrade your Python as Python 2.7 won't be maintained after that date. A future version of pip will drop support for Python 2.7.
```

After finishing first step, click **Configure Local Server** button. After 5 minutes of completion monitoring setup, you will be able to see the following in Monitoring dashboard.

**image placeholder**

### Cache Management (Optional)
Although LDAP CacheManagement is easy to configure and works well, it makes high load on LDAP replication. Therefore many admins prefer redis cache management. To configure redis cache management, go Setting on the left menu and unchecked Use LDAP Cache. After this you will see Cache Management on the left menu. Click on it and hit Setup Cache button. Complete 3 steps.

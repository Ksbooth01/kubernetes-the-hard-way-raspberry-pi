# 01 - Infrastructure Provisioning-RASPIAN

Kubernetes can be installed just about anywhere Linux runs on. It could be either physical or virtual machines. In this lab we are going to focus on [Raspian(Buster) on Raspberry Pi](https://www.raspberrypi.org).

This lab will walk you through provisioning the Raspberry Pi software required for running a H/A Kubernetes cluster. 

## DISCLAIMER
The steps in this tutorial are "AS IS" without any warranties and support.
I am not responsible for any misconfiguration or damages to the Raspberry Pi equipment involved on this tutorial.


# OS Configuration

* First, we need to download (https://www.raspberrypi.org/software/operating-systems/#raspberry-pi-os-32-bit) they 32-bit option.) This will get installed on a total of 5 Raspberry Pis will be configured. Here are their names and IP addresses:
* Next, you'll need to download imager software to load the SD Micro cards with the image. Raspberry Pi offers a free too for this [here](https://www.raspberrypi.org/downloads/)

## Flashing the SD Micro
* If you're using the Raspberry Pi Imager pick `Choose OS` then scroll down and choose `Use Custom` 
* Navigate to the place where you downloaded your ubuntu Pi image  
* Insert your MicroSD card into you PC and `Choose SD Card`
* Then pick `Write`
* When its finished transferring to the card, remove it and insert it into your Raspberry Pi, and power it up.

## Initial login 
* On first boot you will be presented with the initial login prompt. use **`ubuntu`** for the login name and **`ubuntu`** as the password.
* You will be asked to change the password on first login. Change it!

### Change the name of the computer
In the table is the names and IP addresses of the servers configured in this setup.  These will be referenced throughout.

|  Hostname    | IP address-EXT| IP address-INT|            
|:------------:|:-------------:|:-------------:|              
| master-1     | 192.168.1.20  | 172.16.0.20   |
| master-2     | 192.168.1.30  | 172.16.0.30   |
| node-1       | 192.168.1.31  | 172.16.0.31   |
| node-2       | 192.168.1.32  | 172.16.0.32   |
| loadbalancer | 192.168.1.40  | 172.16.0.40   | 

To change the default server name of ubuntu use the following command.
**`sudoedit /etc/hostname`**


### Setting the root password
* Type **`sudo su`** this will put you at root `#`
* type **`passwd`**. Once the password for root has been changed enter **`exit`**
### (IP address setup - Public and Private)
The IP address configuration for ubuntu is contained in a file called /etc/netplan/50-cloud-init.yaml.   
There's a bit of typing required here. The ubuntu image does not have wifi preconfigured so this it what the file needs to look like to get it to work 
**Notes:** 
    * spelling counts.  spacing - spelling - all the syntax. 
    * Your IPs need to be static. 
    * On the wired (Eth0) network I had no DHCP, so I enter addresses 
    * On the wifi (wlan0) network there is DHCP so I created reseverations. Everyones wifi is different so I'm not going into that.
    * 172.16.0.10 is just my workstation address for SSHing into the PIs.

**`
sudoedit /etc/netplan/50-cloud-init.yaml
`**

```
# This file is generated from information provided by the datasource.  Changes
# to it will not persist across an instance reboot.  To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
  version: 2
  wifis:
    wlan0:
      optional: true
      access-points:
        "YOUR-SSID-NAME":
          password: "YOUR-NETWORK-PASSWORD"
      dhcp4: true    
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 172.16.0.##/24
 #     gateway4: 172.16.0.10
 #     nameservers:
 #       addresses: [8.8.8.8,1.1.1.1]
      optional: true
```
**Notes:** 
    * YOUR-SSID-NAME  is the name of your wifi network. 
    * The Quotes around YOUR-SSID-NAME are necessary. 
    * Mine wouldn't work with the 5G so I had to use the lower speed Wifi.
* Test start the wireless network using your modifications 
    **`sudo netplan generate `**
* This should run without errors, if not fix your typos.
### Permit root to ssh (optional)
* If you want to console in for the remaining part you can do this.
Leaving root allowed to ssh should not be left on - make sure you turn this off at the end of setup. 
**`sudo nano /etc/ssh/sshd_config`**
* Find the line:
  `#PermitRootLogin prohibit-password` and change it to 
  `PermitRootLogin yes` Save the file
  ### Patching
  **`sudo apt-get update -y`**
  **`sudo apt-get upgrade -y`**
  on completion **`reboot`** 

## Changing the default user from "Ubuntu" (Optional) 

#### Assumptions:
* A brand new raspberry pi
* You want to change the default username `ubuntu` to `kubeadmin` (or someother name to your liking)
* You want to adapt also the main group from `ubuntu` to `kubeadmin`
* You want other things to work out like sudo and auto-login
 

#### Step 1: make the user change
* SSH in as **`root`** with your root password. You are now alone in the system, and changes to `ubuntu` will not be met with `usermod: user ubuntu is currently used by process 2104.` Check with $ `ps -u ubuntu` to see an empty list.
* Very carefully, key by key, type **`usermod -l kubeadmin ubuntu`** . This will change your username in the `/etc/passwd` file.
* To check,type **`tail /etc/passwd`** and verify the last line `kubeadmin:1000:...` The 1000 is the UID and it is now yours.
* Try **`su kubeadmin`** just to be sure. Do nothing. Just `exit` again to root. It should work. Now you need to adjust the group and a `$HOME` folder.
#### Step 2: make the group change
* Type **`groupmod -n kubeadmin ubuntu`** . This will change the ubuntu group name. 
* Check it with $ **`tail /etc/group`** and you will see the last line the new name associated with GID 1000.
* Just to clarify, type **`ls -la /home/ubuntu`** and you will see that the ubuntu HOME now belongs to you, kubeadmin.
#### Step 3: lets adopt the new home.
* Move to **`cd /home`** to make it easier. Type $ **`ls -la`** and see `ubuntu`, onwer `kubeadmin` group `kubeadmin`
* Carefully type $ **`mv ubuntu kubeadmin`**. You now need to associate this change with your new user.
* Carefully type $ **`usermod -d /home/kubeadmin kubeadmin`**. This will change your home directory. To confirm **`tail /etc/passwd`** and look at the sixth field (separated by `:`).

#### ..Just the commands
```
usermod -l kubeadmin ubuntu
groupmod -n kubeadmin ubuntu
cd /home
mv ubuntu kubeadmin
usermod -d /home/kubeadmin kubeadmin
```
## Enter Hostnames & IP Addresses to the hosts file

In my experience, whether you use a direct connection versus Wi-Fi really depends on connection speed. The one thing that **IS CRITICAL** is that you assign IP Addresses to each server. for my machines is set up DHCP reservations.  (Yes. even home WI-FI routers allow you to set up DHCP reservations in their configuration) 

```
sudo sh -c cat >> /etc/hosts << EOF
192.168.1.20       master-1
192.168.1.30       master-2
192.168.1.31       worker-1
192.168.1.32       worker-2
192.168.1.40       loadbalancer

EOF

```

> Remember to run these steps on `master-1`, `master-2`, `worker-1`, `worker-2`, and `loadbalancer`


Next [Setting up a Certificate Authority](04-certificate-authority.md)

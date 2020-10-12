# 01 - Infrastructure Provisioning

Kubernetes can be installed just about anywhere Linux runs on. It could be either physical or virtual machines. In this lab we are going to focus on [Ubuntu on Raspberry Pi](https://ubuntu.com/download/raspberry-pi).

This lab will walk you through provisioning the Raspberry Pi software required for running a H/A Kubernetes cluster. 

# DISCLAIMER
The steps in this tutorial are "AS IS" without any warranties and support.
I am not responsible for any misconfiguration or damages to the Raspberry Pi equipment involved on this tutorial.


# OS Configuration

The OS for each Raspberry Pi is [Ubuntu 20.04](https://ubuntu.com/download/raspberry-pi) I went with the 64-bit option and not the 32-bit option.  Though it's possible you can get Kubernetes to run on 32-bit, I couldn't get any of the new editions to really run properly.  Also, it's not officially supported.

download the image and install it on  
A total of 5 Raspberry Pis will be configured. Here are their names and IP addresses:

| Hostname    | IP address    |             
|:-----------:|:-------------:|              
| controller0 | 192.168.1.20  |             
| controller1 | 192.168.1.40  |
|             |               |
| worker1     | 192.168.1.21  |
| worker2     | 192.168.1.22  |
| loadbalancer| 192.168.1.30  |


## Flashing the SD Micro

## Initial login 
* On first boot you will be presented with the initial login prompt. use **`ubuntu`** for the login name and **`ubuntu`** as the password.
* You will be asked to change the password on first login. Change it!
### Setting the root password
* Type **`sudo su`**  then type **`passwd`**. Once the password for root has been changed enter **`exit`**
### (Optional - Setting up WiFi)
Hard wired is definately the better option for setting up a Kubernetes cluster, but MY Ethernet is not near my office, and I make enough changes to the physical that hard wired wasn't that great of an option, beside I have 5G it's not really that big a deal.
The image does not hae wifi configured so this it was the file needs to look like to get it to work **Note:** spelling counts.
**```
sudoedit /etc/netplan/50-clout-init.yaml
```**
# This file is generated from information provided by
# the datasource.  Changes to it will not persist across an instance.
# To disable cloud-init's network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
    version: 2
    ethernets:
        eth0:
            optional: true
            dhcp4: true
    # add wifi setup information here ...
    wifis:
        wlan0:
            optional: true
            access-points:
                "YOUR-SSID-NAME":
                    password: "YOUR-NETWORK-PASSWORD"
            dhcp4: true
```
**Note:** YOUR-SSID-NAME  is the neme of you wifi network. the Quotes are necessary. (mine wouldn't work with the 5G so I had to use the regular one)
* Test start the wireless network using your modifications 
    **`systemctl start netplan`**
    This Should run without errors, if not fix your typos.
### Permit root to ssh
Leaving root allowed to ssh shoudl not be left on - make sure you turn this off at the end of setup. 
**`sudo nano /etc/ssh/sshd_config`**
* Find the line:
  `#PermitRootLogin prohibit-password` and change it to 
  `PermitRootLogin yes` Save the file on exit and **`reboot`** 

## Changing the default user from "Ubuntu" (Optional) 

#### Assumptions:
* A brand new raspberry pi
* You want to change the default username `ubuntu` to `kubeadmin` (or someother name to your liking)
* You want to adapt also the main group from `ubuntu` to `kubeadmin`
* You want other things to work out like sudo and auto-login
 

#### Step 1: make the user change
* SSH in as **`root`** with your root password. You are now alone in the system, and changes to `ubuntu` will not be met with `usermod: user pi is currently used by process 2104.` Check with $ `ps -u ubuntu` to see an empty list.
* Very carefully, key by key, type **`usermod -l kubeadmin ubuntu`** . This will change your username in the `/etc/passwd` file. To check,type **`tail /etc/passwd`** and verify the last line `kubeadmin:1000:...` The 1000 is the UID and it is now yours.
* Try **`su kubeadmin`** just to be sure. Do nothing. Just `exit` again to root. It should work. Now you need to adjust the group and a `$HOME` folder.
#### Step 2: make the group change
* Type **`groupmod -n kubeadmin ubuntu`** . This will change the pi group name. Check it with $ **`tail /etc/group`** and you will see the last line the new name associated with GID 1000.
* Just to clarify, type **`ls -la /home/ubuntu`** and you will see that the ubuntu HOME now belongs to you, kubeadmin.
#### Step 3: lets adopt the new home.
* Move to **`cd /home`** to make it easier. Type $ **`ls -la`** and see `ubuntu`, onwer `kubeadmin` group `kubeadmin`
* Carefully type $ **`mv ubuntu kubeadmin`**. You now need to associate this change with your new user.
* Carefully type $ **`usermod -d /home/kubeadmin kubeadmin`**. This will change your home directory. To confirm **`tail /etc/passwd`** and look at the sixth field (separated by `:`).
* Reboot with ```reboot```
#### Step 4: some adjusts after the fact.
* Login as your new user `kubeadmin` in the graphical interface.
* Open a terminal.
*Change your password*
  * Type passwd to change the password of ```kubeadmin``` to something other than *raspberry*
  * Type $ ```sudo su - ``` and you will be asked your password.

* If you want back the ALT+F1 autologin, find and edit the file:
    * $ ```sudo nano /etc/systemd/system/autologin@.service``` and change the line
    * ```#ExecStart=-/sbin/agetty --autologin kubeadmin --noclear %I $TERM```

*While we're here now's a good time the chage the hostname*
  *  $ ```sudo raspi-config``` pick ```2 Network Options```, then pick ```N1 Hostname``` Enter the name of the host you are configuring then ```Ok``` and ```Finish```
* Once everything has been successfullly completed  $ ```sudo reboot```


### Disable the SWAP file :
 Some people seem concerned that frequent writes to the MicroSD card will make it fail quickly. I find that if its turned on, things break. In a effort to reduce troubleshooting time, I turn it off.

First turn off all existing swapfiles.
```
sudo swapoff -a
```
then comment out the swapfile components in the /etc/fstab file
```
nano /etc/fstab
```
put a # sign in fron of the following line like so
```
# use  dphys-swapfile swap[on|off]  for that
```
disable swapfile from systemd 
```
sudo dphys-swapfile swapoff && \
sudo dphys-swapfile uninstall && \
sudo systemctl disable dphys-swapfile
```
and finally...
```
sudo systemctl disable dphys-swapfile.service
```
reboot the server and test to see if it works
```
sudo swapon show
```
expected output...
```
swapon: cannot open show: No such file or directory
```

## Hostnames to IP Addresses

In my experience, whether you use a direct connection versus Wi-Fi really depends on connection speed. The one thing that **IS CRITICAL** is that you assign IP Addresses to each server. for my machines is set up DHCP reservations.  (Yes. even home WI-FI routers allow you to set up DHCP reservations in their configuration) 

```
sudo sh -c "cat >>/etc/hosts <<EOF
192.168.1.20       controller0
192.168.1.40       controller1
192.168.1.21       worker1
192.168.1.22       worker2

192.168.1.30       loadbalancer

EOF

```

> Remember to run these steps on `controller0`, `controller1`, `worker1`, `worker2`, and `loadbalancer`

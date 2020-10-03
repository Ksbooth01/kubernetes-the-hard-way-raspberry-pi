# Infrastructure Provisioning

Kubernetes can be installed just about anywhere Linux runs on. It could be either physical or virtual machines. In this lab we are going to focus on [RASPBIAN](https://www.raspberrypi.org/downloads/raspbian/).

This lab will walk you through provisioning the Raspberry Pi software required for running a H/A Kubernetes cluster. 

# DISCLAIMER
The steps in this tutorial are "AS IS" without any warranties and support.
I am not responsible for any misconfiguration or damages to the Raspberry Pi equipment involved on this tutorial.


# OS Configuration

The OS for each Raspberry Pi is [RASPBIAN BUSTER 2020-02-14](https://downloads.raspberrypi.org/raspbian/images/raspbian-2020-02-14/2020-02-13-raspbian-buster.zip) It's the desktop version. You can go with the LITE edition, but it was easier for me to just go with imaging the card with the desktop version.

A total of 5 Raspberry Pis will be configured. Here are their names and IP addresses:

| Hostname    | IP address    |             
|:-----------:|:-------------:|              
| control0    | 192.168.1.20  |             
| control1    | 192.168.1.40  |
|             |               |
| node1       | 192.168.1.21  |
| node2       | 192.168.1.22  |
| loadbalancer| 192.168.1.30  |


## Changing the default user from "Pi" (Optional) 
(props to DrBeco for posting the original)
#### Assumptions:
* A brand new raspberry pi
* You want to change the default username pi to kubeadmin (or someother name to your liking)
* You want to adapt also the main group from pi to kubeadmin
* You want other things to work out like sudo and auto-login
 
#### Step 1: stop user pi from running before the change.
* Boot you Pi, go to RPI configurations and
    * allow SSH,
    * disallow auto-login
    * hit ok
* Press ALT+F1 to go to the first tty
* Escalate to root with ```sudo su -```
* Edit ```$nano /etc/systemd/system/autologin@.service```
    * Find and comment (#) the line
        * ``` #ExecStart=-/sbin/agetty --autologin pi --noclear %I $TERM ```
        you can uncomment it later if you want console autologin, but then don't forget to change the user ```pi``` to your new username ```kubeadmin```
    * Create a new root password with ```passwd```. **(DON'T FORGET IT)**
    * Type ```reboot```
#### Step 2: make the user change
* If you see the graphical login prompt, you are good. Do not login. Instead, press ALT+F1 
* After ALT+F1, you should see a ```login``` question (and not an autologin).
* Login as ```root``` with your root password. Now you are alone in the system, and changes to ```pi``` will not be met with ```usermod: user pi is currently used by process 2104.``` Check with ```ps -u pi``` to see an empty list.
* Very carefully, key by key, type ```usermod -l kubeadmin pi``` . This will change your username, from ```/etc/passwd``` file, but things are not ready yet. Anyway, check with ```tail /etc/passwd``` and see the last line ```kubeadmin:1000:...``` The 1000 is the UID and it is now yours.
* Try ```su kubeadmin``` just to be sure. Do nothing. Just ```exit``` again to root. It should work. Now you need to adjust the group and a ```$HOME``` folder.
#### Step 3: make the group change
* Type, again carefully, groupmod -n mypie pi . This will change the pi group name. Check it with tail /etc/group and you will see the last line the new name associated with GID 1000.
* Just to clarify, type ls -la /home/pi and you will see that the pi HOME now belongs to you, mypie.
#### Step 4: lets adopt the new home.
* Move to ```cd /home``` to make it easier. Type ```ls -la``` and see ```pi```, onwer ```kubeadmin``` group ```kubeadmin```
* Type carefully: ```mv pi kubeadmin```. You now need to associate this change with your new user.
* Type carefully: ```usermod -d /home/kubeadmin kubeadmin```. This will change your home directory. To confirm ```tail /etc/passwd``` and look at the sixth field (separated by ```:```).
* Reboot with ```reboot```
#### Step 5: some adjusts after the fact.
* Login as your new user ```kubeadmin``` in the graphical interface.
* Open a terminal.
*Change your password*
  * Type passwd to change the password of ```kubeadmin``` to something other than *raspberry*
  * Type ```sudo su - ``` and you will be asked your password.

* If you want back the ALT+F1 autologin, find and edit the file:
    * $ ```sudo nano /etc/systemd/system/autologin@.service and change the line```
    * ```#ExecStart=-/sbin/agetty --autologin kubeadmin --noclear %I $TERM```

*While we're now's a good time the chage the hostname*
  * ```sudo raspi-config``` pick 2 Network Options, then pick ```N1 Hostname``` Enter the name of the host you are configuring then ```Ok```

#### Step 6: reboot
* Type, carefully, ```reboot```


### SWAP (optional):
I add 1GB of swap space.
Some people seem concerned that frequent writes to the MicroSD card will make it fail quickly. I therefore decided to set the swappiness such that it only uses the swap as a last resort.

```
sudo swapoff -a
```
```
/etc/fstab
```
put a # sign in fron of the following line like so
```
# use  dphys-swapfile swap[on|off]  for that
```

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

sudo show
## Hostnames to IP Addresses

In my exprience, connectivity between each server would be highly improved if connected directly to LAN network instead of Wi-Fi.

```
sudo sh -c "cat >>/etc/hosts <<EOF
192.168.1.20       control0
192.168.1.40       control1
192.168.1.21       node1
192.168.1.22       node2

192.168.1.30       loadbalancer

EOF
"
```

> Remember to run these steps on `control0`, `control1`, `node1`, `node2`, and `loadbalancer`

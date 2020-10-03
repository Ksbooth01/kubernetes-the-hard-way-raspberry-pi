# Infrastructure Provisioning

Kubernetes can be installed just about anywhere Linux runs on. It could be either physical or virtual machines. In this lab we are going to focus on [RASPBIAN](https://www.raspberrypi.org/downloads/raspbian/).

This lab will walk you through provisioning the Raspberry Pi software required for running a H/A Kubernetes cluster. 

# DISCLAIMER
The steps in this tutorial are "AS IS" without any warranties and support.
I am not responsible for any misconfiguration or damages to the Raspberry Pi equipment involved on this tutorial.


# OS Configuration

The OS for each Raspberry Pi is [RASPBIAN BUSTER 2020-08-20](https://downloads.raspberrypi.org/raspbian_lite/images/raspbian_lite-2017-03-03/2017-03-02-raspbian-jessie-lite.zip) This is a small image with a nice size OS perfect for small servers.

A total of 5 Raspberry Pis will be configured. Here are their names and IP addresses:

| Hostname   | IP address    |              | Hostname   | IP address    |
|:----------:|:-------------:|              |:----------:|:-------------:| 
| control0   | 192.168.1.20  |              | node1      | 192.168.1.21  |
| control1   | 192.168.1.40  |              | node2      | 192.168.1.22  |

| controller2   | 10.0.1.92     |


## Memory and Swap

This GPU configuration has worked well for previous server setups I've done.

```
sudo sh -c 'echo "gpu_mem=16" >> /boot/config.txt'
```

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

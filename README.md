![Image of Yaktocat](kubernetes_logo.png) ![Image of Yaktocat](raspberry_pi_logo.png)

# Kubernetes the Hard Way Raspberry Pi

This tutorial is inspired by the [Kelsey Hightower's Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) tutorial, but targeted to the Raspberry Pi. This guide is not for people looking for a fully automated command to bring up a Kubernetes cluster. If that's you then check out [Google Container Engine](https://cloud.google.com/container-engine), or the [Getting Started Guides](http://kubernetes.io/docs/getting-started-guides/).

This tutorial is optimized for learning, which means taking the long route to help people understand each task required to bootstrap a Kubernetes cluster. This tutorial can be completed on the following platforms:

* [Raspberry Pi 3 Model B](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/)
* [Raspberry Pi 4](https://www.raspberrypi.org/products/raspberry-pi-4-model-b/)

> The results of this tutorial should not be viewed as production ready, and may receive limited support from the community, but don't let that prevent you from learning!

# DISCLAIMER
The steps in this tutorial are "AS IS" without any warranties and support.
I'm not responsible for any misconfiguration or damages of the Raspberry Pi equipment involved on this tutorial.



## Target Audience

The target audience for this tutorial is someone who wants to get some hands on skills with the Kubernetes cluster and wants to understand how everything fits together. `If you're planning to support a production Kubernetes cluster, I highly recommend following Kelsey's tutorial instead`. After completing this tutorial I encourage you to automate away the manual steps presented in this guide.

* This tutorial is for educational purposes only. There is much more configuration required for a production ready cluster.

## Cluster Details

* Kubernetes 1.19.2 (ARM)
* Docker 1.18.3 (ARM)
* etcd 3.1.0-rc.1 (Built from source for ARM)
* [CNI Based Networking](https://github.com/containernetworking/cni)
* Secure communication between all components (etcd, control plane, workers)
* Default Service Account and Secrets


### What's Missing

The resulting cluster will be missing the following items:

* [Cluster add-ons](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons)
* [Logging](http://kubernetes.io/docs/user-guide/logging)
* [No Cloud Provider Integration]q(http://kubernetes.io/docs/getting-started-guides/)
* Setup Kubernetes API Server Frontend Load Balancer

## Hardware and OS version

This tutorial assumes you have access to one of the following:

* 5 Raspberry Pi 3/4 Model B
* 2020-08-20-raspbian-buster or newer

## Labs
* Prerequisites
* Installing Client tools
* [Infrastructure Provisioning](docs/01-infrastructure.md)
* 
* [Provisioning the CA Generating TLS Certificates](docs/04-certificate-authority.md)
* [Generating Kubernetes Configuration Files for Authorization](docs/05-kubernetes-configuration-files.md)
* [Generating the Data Encryption Config and Key](docs/06-data-encryption-keys.md)
* [Bootstrapping an H/A etcd cluster](docs/07-bootstrapping-etcd.md)
* [Bootstrapping the Kubernetes Control Plane](docs/08-bootstrapping-kubernetes-controllers.md)
* [Provision a Network Load Balancer](docs/08b-kubernetes-loadbalancer.md)
* [Bootstrapping the Kubernetes Worker Nodes](docs/09-bootstrapping-kubernetes-workers.md)
* [Configuring kubectl for Remote Access](docs/10-configuring-kubectl.md)
* [Provisioning Pod Network Routes](docs/11-pod-network-routes.md)
* [Deploying the Cluster DNS Add-on](docs/12-dns-addon.md)
* [Smoke Test](docs/13-smoke-test.md)
* [Cleaning Up](docs/14-cleanup.md)

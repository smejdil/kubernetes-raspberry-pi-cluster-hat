# Introduction - Setting Up the Cluster
This is the first attempt after some time when I tried to set up such a cluster and I started learning [Kubernetes](https://kubernetes.io/). One of the reasons I wanted to do this is because I have four [Raspberry Pi Zero](https://www.raspberrypi.org/products/raspberry-pi-zero/) lying around and a new (at the time) [Raspberry Pi 4](https://www.raspberrypi.org/products/raspberry-pi-4-model-b/). After experimenting for a bit with them individually, I decided to put them to work together. I am also getting acquainted with [Zabbix](https://www.zabbix.com) [monitoring K8S](https://www.zabbix.com/cz/integrations/kubernetes) and need somewhere to test it.


Some research brought my attention to [ClusterHAT](https://clusterhat.com/) which simplified all the messing around with USB gadgets to make the Pi Zeros believe they’re connected to a network using USB. Having tested it for a while, I decided to give it a go and install a Kubernetes cluster. 
## Hardware

The specific hardware I used for this exercise is:

* Raspberry Pi 4
* Raspberry Pi Zero
* Cluster HAT v2.5
* 5 x micro SD cards (32 GB for master and 32 GB for workers)

For instruction on how to set up the hardware, see [their website](https://clusterctrl.com/setup-assembly). Bought [here](https://rpishop.cz/clustery/2973-cluster-hat-kit.html).

## Operating System

Download the images for each Pi: controller CNAT and each pi (p1 .. p4) in this case I only used p1 and p2 as I only have two pi zeros. As of this writing, these are the files available to download from [clusterctrl downloads](https://clusterctrl.com/setup-software) I chose the lite version (no Desktop or GUI) and CNAT to use the internal NAT network.

* [Controller] 2022-04-04-2-bullseye-ClusterCTRL-arm64-lite-CNAT.img
* [Worker 1] 2022-04-04-2-bullseye-ClusterCTRL-arm64-lite-p1.img
* [Worker 2] 2022-04-04-2-bullseye-ClusterCTRL-arm64-lite-p2.img
* [Worker 3] 2022-04-04-2-bullseye-ClusterCTRL-arm64-lite-p3.img
* [Worker 4] 2022-04-04-2-bullseye-ClusterCTRL-arm64-lite-p4.img


## Preparing the Controller

If you're using Wi-Fi, it needs setting up before booting to make it easy to connect totally headless.

### Setting Up Wi-Fi on the Controller 

Mount the microSD card and in /boot partition modify the file `/boot/wpa_supplicant.conf`

```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=CZ

network={
 ssid="XYZ"
 psk="abcdef.zyx.1234"
}
```

Create an empty file to allow SSH service to start with the system.

```shell
touch /boot/ssh
```

Setup password for user pi.

```shell
echo 'mypassword' | openssl passwd -6 -stdin

vi /boot/userconf.txt
pi:encrypted-password
```

### Understand the Networking Model

Host name      |   External IP |    Internal IP | Role   |
---------------|---------------|----------------|--------|
rpi-k8s-master | 192.168.5.150 | 172.19.181.150 | master |
p1             |      NAT      | 172.19.181.1   | worker |
p2             |      NAT      | 172.19.181.2   | worker |
p3             |      NAT      | 172.19.181.3   | worker |
p4             |      NAT      | 172.19.181.4   | worker |

Diagram 

### Upgrade the System

It's always a good start with a fresh up to date system. Something particular of Raspbian is that you should use `full-upgrade` instead of just `upgrade` as it could be the case that it doesn't download all the dependencies of the new packages, kind of weird. 

```shell
$ sudo aptitude update
$ sudo aptitude safe-upgrade
```

### Change the hostname 

In my case, I wanted to identify the master node from the rest, I updated to `rpi-k8s-master` 

```shell
$ sudo hostnamectl set-hostname rpi-k8s-master
```

### Set Up and Verify Connectivity 

Once all the base hardware is up and running, it'll be much easier if ssh config file is configured to connect to the pi zeros.

Create a config file if it's not already there.

```shell
vi ~/.ssh/config
```

```
Host *
  ServerAliveInterval 180
  ServerAliveCountMax 2
  IdentitiesOnly=yes
  IdentityFile ~/.ssh/local_ed25519

Host p1
    Hostname 172.19.181.1
    User pi
Host p2
    Hostname 172.19.181.2
    User pi
Host p2
    Hostname 172.19.181.3
    User pi
Host p2
    Hostname 172.19.181.4
    User pi
```

Add the pi zeros IPs to the local hosts file.

```shell
cat | sudo tee -a /etc/hosts << HERE
172.19.181.1    p1
172.19.181.2    p2
172.19.181.3    p3
172.19.181.4    p4
HERE
```

### Start ClusterHat nodes
```shell
clusterhat init
clusterhat on p1
clusterhat on p2
clusterhat on p3
clusterhat on p4
```

### Generate SSH Keys and Copy to the Pi Zeros

```shell
ssh-keygen -t ed25519 -b 4096 -N "" -f ~/.ssh/local_ed25519 -C "local key ClusterHAT"

ssh-copy-id -i .ssh/local_ed25519.pub p1

ssh-copy-id -i .ssh/local_ed25519.pub p2

ssh-copy-id -i .ssh/local_ed25519.pub p3

ssh-copy-id -i .ssh/local_ed25519.pub p4
```
### Install the Client Tools

The Client Tools (on the Pi 3) this can be installed optionally on your local environment as well. 

Get the CloudFlare's PKI toolkit

```shell
sudo apt install golang-cfssl
```

Verify it’s working 
```shell
$ cfssl version
Version: 1.2.0
Revision: dev
Runtime: go1.15.9
```

Install tmux

```shell
sudo apt install tmux
```

### Working Directories

The following directories are going to be used to store the files produced by the command outputs during this process.

```
~/pki
~/certs
~/config
~/bin
~/plugins
```

Create the directories, positioned in $HOME directory.

```shell
mkdir pki certs config bin plugins
```

### Enable Cgroups Memory
On each worker node append `cgroup_enable=memory cgroup_memory=1` to */boot/cmdline.txt*. This needs to be run as root.
```
sudo su
echo -n ' cgroup_enable=memory cgroup_memory=1' | tee -a /boot/cmdline.txt
```

Then restart the node:
```
sudo shutdown -r 0
```

Cgroups memory needs to be turned on, or in step 6 [Test Worker Nodes](https://github.com/abelperezok/kubernetes-raspberry-pi-cluster-hat/blob/master/06-Worker-Nodes.md#test-worker-nodes) your node status may all come up as `NotReady`.

After running step 6 on master:

```
kubectl get nodes --kubeconfig config/admin.kubeconfig
```

the statuses were all `NotReady`.  Running

```
journalctl -fu kubelet
```
On *p1*, showed an error:

> Failed to start ContainerManager system validation failed - Following Cgroup subsystem not mounted: [memory]  

Turns out that the memory cgroup is disabled by default since it adds some [additional memory overhead](https://github.com/raspberrypi/linux/issues/1950).

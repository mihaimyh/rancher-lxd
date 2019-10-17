# rancher-lxd

Provisioning rancher inside lxd containers

Steps:

## 1. LXD

```bash
sudo apt-get purge lxd lxd-client

sudo snap install lxd

echo "root:1000000:65536" | sudo tee -a /etc/subuid /etc/subgid

sudo usermod -a -G lxd $USER

newgrp lxd

sudo lxd init --auto

lxc network set lxdbr0 ipv6.address none
```

> If you got an error when trying to execute any lxc/lxd commands be sure you add /snap/bin in your path:
>
> echo "export PATH="/snap/bin:$PATH"" >> ~/.bashrc

### 1.1 Production setup

See <https://linuxcontainers.org/lxd/docs/master/production-setup>

#### /etc/security/limits.conf

```bash
soft nofile 1048576
hard nofile 1048576
root soft nofile 1048576
root hard nofile 1048576
soft memlock unlimited
hard memlock unlimited
```

#### /etc/sysctl.conf

```bash
fs.inotify.max_queued_events 1048576
fs.inotify.max_user_instances 1048576
fs.inotify.max_user_watches 1048576
vm.max_map_count 262144
kernel.dmesg_restrict 1
net.ipv4.neigh.default.gc_thresh3 8192
net.ipv6.neigh.default.gc_thresh3 8192
kernel.keys.maxkeys 2000
```

#### Network Bandwidth Tweaking

If you have at least 1GbE NIC on your lxd host with a lot of local activity (container - container connections, or host - container connections), or you have 1GbE or better internet connection on your lxd host it worth play with txqueuelen. These settings work even better with 10GbE NIC.

##### Server Changes

###### txqueuelen

You need to change txqueuelen of your real NIC to 10000 (not sure about the best possible value for you), and change and change lxdbr0 interface txqueuelen to 10000.

In Debian-based distros you can change txqueuelen permanently in /etc/network/interfaces
You can add for ex.: `up ip link set eth0 txqueuelen 10000` to your interface configuration to set txqueuelen value on boot.
You could set it txqueuelen temporary (for test purpose) with `ifconfig <interface> txqueuelen 10000`

###### /etc/sysctl.conf

You also need to increase `net.core.netdev_max_backlog` value.

You can add `net.core.netdev_max_backlog = 182757` to /etc/sysctl.conf to set it permanently (after reboot).

You set netdev_max_backlog temporary (for test purpose) with echo 182757 > /proc/sys/net/core/netdev_max_backlog

> Note: You can find this value too high, most people prefer set netdev_max_backlog = net.ipv4.tcp_mem min. value.
>
> For example I use this values `net.ipv4.tcp_mem = 182757 243679 365514`

##### Containers changes

You also need to change txqueuelen value for all you ethernet interfaces in containers.
In Debian-based distros you can change txqueuelen permanently in /etc/network/interfaces
You can add for ex.: `up ip link set eth0 txqueuelen 10000` to your interface configuration to set txqueuelen value on boot.

### 1.2 Create main directory storage pool

`lxc storage create lxd-main dir source=~/lxd`

### 1.3. Create profiles

See yamls [yamls](yamls\3_profile.yaml)

## 2. Docker

### 2.1 UNINSTALL OLD VERSIONS

Older versions of Docker were called docker, docker.io , or docker-engine. If these are installed, uninstall them:

`sudo apt-get remove docker docker-engine docker.io containerd runc`

### 2.2 SET UP THE REPOSITORY

#### 2.2.1 Update the `apt` package index

`sudo apt-get update`

#### 2.2.2 Install packages to allow apt to use a repository over HTTPS

```bash
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
```

#### 2.2.3 Add Docker’s official GPG key

`curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`

#### 2.2.4 Add the Docker APT repository to your system

`sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"`

### 2.3 INSTALL DOCKER CE

#### 2.3.1 Update the `apt` package index

`sudo apt-get update`

#### 2.3.2 List the available versions in the Docker repository

`apt list -a docker-ce`

#### 2.3.3 Install a supported Rancher Docker version

`sudo apt install docker-ce=5:18.09.9~3-0~ubuntu-bionic docker-ce-cli=5:18.09.9~3-0~ubuntu-bionic containerd.io`

#### 2.3.4 Prevent the Docker package from being automatically updated, mark it as held back

`sudo apt-mark hold docker-ce`

### 2.4 MANAGE DOCKER AS NON-ROOT USER

#### 2.4.1 Create the docker group if not exists already

`sudo groupadd docker`

#### 2.4.2 Add your user to the docker group

```bash
sudo usermod -aG docker $USER
newgrp docker
```

## 3. Rancher

First of all we need to install few CLI tools required for this installation:

### kubectl

`sudo snap install kubectl --classic`

### rke

Please check github for an updated version:

```bash
wget https://github.com/rancher/rke/releases/download/v0.3.1/rke_linux-amd64
sudo mv rke_linux-amd64 /usr/local/bin/rke
sudo chmod +x /usr/local/bin/rke
```

### Helm

`sudo snap install helm --classic`

### 3.1 Create nodes and load balancer

We need to create 3 worker nodes and 1 load balancer for RKE install.

#### 3.1.1 Worker nodes

```bash
lxc launch ubuntu:18.04 k-worker1 -p k8s-worker
lxc launch ubuntu:18.04 k-worker2 -p k8s-worker
lxc launch ubuntu:18.04 k-worker3 -p k8s-worker
lxc launch ubuntu:18.04 proxy -p k8s-master
```

#### 3.1.2 Load balancer

We can use the following nginx config to spin-up a load-balancer, be sure to modify <IP_NODE_> with the ips of the master and worker nodes:

```bash
worker_processes 4;
worker_rlimit_nofile 40000;

events {
    worker_connections 8192;
}

stream {
    upstream rancher_servers_http {
        least_conn;
        server <IP_NODE_1>:80 max_fails=3 fail_timeout=5s;
        server <IP_NODE_2>:80 max_fails=3 fail_timeout=5s;
        server <IP_NODE_3>:80 max_fails=3 fail_timeout=5s;
    }
    server {
        listen 80;
        proxy_pass rancher_servers_http;
    }

    upstream rancher_servers_https {
        least_conn;
        server <IP_NODE_1>:443 max_fails=3 fail_timeout=5s;
        server <IP_NODE_2>:443 max_fails=3 fail_timeout=5s;
        server <IP_NODE_3>:443 max_fails=3 fail_timeout=5s;
    }
    server {
        listen 443;
        proxy_pass rancher_servers_https;
    }
}
```

Save the edited Example NGINX config as `/etc/nginx.conf` and run the following command to launch the NGINX container:

```bash
docker run -d --restart=unless-stopped \
  -p 80:80 -p 443:443 \
  -v /etc/nginx.conf:/etc/nginx/nginx.conf \
  nginx:1.14
```

### 3.2 Install Kubernetes with RKE

#### 3.2.1 Create the rancher-cluster.yml file

Using the sample below create the rancher-cluster.yml file. Replace the IP Addresses in the nodes list with the IP address or DNS names of the 3 nodes you created.

```bash
nodes:
  - address: 10.63.218.161
 #   internal_address: 172.16.22.12
    user: ubuntu
    role: [controlplane,worker,etcd]
  - address: 10.63.218.138
 #   internal_address: 172.16.32.37
    user: ubuntu
    role: [controlplane,worker,etcd]
  - address: 10.63.218.103
 #   internal_address: 172.16.42.73
    user: ubuntu
    role: [controlplane,worker,etcd]

services:
  etcd:
    snapshot: true
    creation: 6h
    retention: 24h
```

#### 3.2.2 Run RKE

`rke up --config ./rancher-cluster.yml`
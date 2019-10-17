# rancher-lxd

Provisioning rancher inside lxd containers

Steps:

**1. Install LXD:**

```bash
sudo apt-get remove lxd lxd-client

sudo snap install lxd

echo "root:1000000:65536" | sudo tee -a /etc/subuid /etc/subgid

sudo usermod -a -G lxd $USER

newgrp lxd

lxc network set lxdbr0 ipv6.address none

sudo lxd init --auto
```

**2. Production setup:**

See <https://linuxcontainers.org/lxd/docs/master/production-setup>

**/etc/security/limits.conf**

* soft nofile 1048576
* hard nofile 1048576
* root soft nofile 1048576
* root hard nofile 1048576
* soft memlock unlimited
* hard memlock unlimited

**/etc/sysctl.conf**

* fs.inotify.max_queued_events 1048576
* fs.inotify.max_user_instances 1048576
* fs.inotify.max_user_watches 1048576
* vm.max_map_count 262144
* kernel.dmesg_restrict 1
* net.ipv4.neigh.default.gc_thresh3 8192
* net.ipv6.neigh.default.gc_thresh3 8192
* kernel.keys.maxkeys 2000

**Network Bandwidth Tweaking**

If you have at least 1GbE NIC on your lxd host with a lot of local activity (container - container connections, or host - container connections), or you have 1GbE or better internet connection on your lxd host it worth play with txqueuelen. These settings work even better with 10GbE NIC.

**Server Changes
txqueuelen**

You need to change txqueuelen of your real NIC to 10000 (not sure about the best possible value for you), and change and change lxdbr0 interface txqueuelen to 10000.
In Debian-based distros you can change txqueuelen permanently in /etc/network/interfaces
You can add for ex.: up ip link set eth0 txqueuelen 10000 to your interface configuration to set txqueuelen value on boot.
You could set it txqueuelen temporary (for test purpose) with ifconfig <interface> txqueuelen 10000

**/etc/sysctl.conf**

You also need to increase net.core.netdev_max_backlog value.
You can add net.core.netdev_max_backlog = 182757 to /etc/sysctl.conf to set it permanently (after reboot) You set netdev_max_backlog temporary (for test purpose) with echo 182757 > /proc/sys/net/core/netdev_max_backlog Note: You can find this value too high, most people prefer set netdev_max_backlog = net.ipv4.tcp_mem min. value. For example I use this values net.ipv4.tcp_mem = 182757 243679 365514

**Containers changes**

You also need to change txqueuelen value for all you ethernet interfaces in containers.
In Debian-based distros you can change txqueuelen permanently in /etc/network/interfaces
You can add for ex.: up ip link set eth0 txqueuelen 10000 to your interface configuration to set txqueuelen value on boot.

**3. Create profiles:**

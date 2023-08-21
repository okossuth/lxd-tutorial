<h1>Building an LXD 3-node Cluster on DigitalOcean with a ZFS storage based pool and
  FAN networking enabled.</h1>

<p>In this example i'm using a Ubuntu 22.04 vainilla image with a 20GB extra volume disk attached to each droplet to be used for a ZFS storage pool. This ZFS pool will be named "local" and can be later expanded online by adding more physical volumes to the nodes. The 3 nodes are <b>orion</b>, <b>rigel</b> and <b>vega</b>, the first one used as the leader node in the cluster and the others as secondary nodes.The LXD version used is 5.15.</p>

A FAN network is also configured for the potential containers that will run in the LXD cluster. if you dont know what is a FAN network please check here: https://wiki.ubuntu.com/FanNetworking

First we create 3 droplets on DigitalOcean's control panel, as this is the minimum number for quorum in the cluster and each with a 20GB unformatted volume attached, all in the same region and using same VPC. 

Next we install LXD and ZFS-utils on all droplets:

```
root@orion:~# apt-get update
root@orion:~# snap remove lxd
root@orion:~# snap install lxd
root@orion:~# apt install zfsutils-linux 
```

The first node orion will be the cluster's leader node so let's get the IP of the internal network interface, in my case eth1:
```
root@orion:~# ip addr show eth1 | grep "inet "
    inet 10.136.143.77/16 brd 10.136.255.255 scope global eth1
root@orion:~#
```
Knowing that the IP is 10.136.143.77/16, then the subnet is 10.136.0.0/16 which we will need to configure the FAN network subnet with lxd init.

Confirm the device name of the attached volume, should be /dev/sda:
```
root@orion:~# lsblk
NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0     7:0    0  73.9M  1 loop /snap/core22/858
loop1     7:1    0  63.3M  1 loop /snap/core20/1828
loop2     7:2    0  49.8M  1 loop /snap/snapd/18357
loop3     7:3    0 173.5M  1 loop /snap/lxd/25112
sda       8:0    0    20G  0 disk 
vda     252:0    0    50G  0 disk 
├─vda1  252:1    0  49.9G  0 part /
├─vda14 252:14   0     4M  0 part 
└─vda15 252:15   0   106M  0 part /boot/efi
vdb     252:16   0   490K  1 disk 
root@orion:~#
```

Configure LXD service on orion node as root:

```
root@orion:~# lxd init
Would you like to use LXD clustering? (yes/no) [default=no]: yes
What IP address or DNS name should be used to reach this server? [default=159.89.86.39]: 10.136.143.77
Are you joining an existing cluster? (yes/no) [default=no]: no
What member name should be used to identify this server in the cluster? [default=orion]: orion
Do you want to configure a new local storage pool? (yes/no) [default=yes]: yes
Name of the storage backend to use (btrfs, dir, lvm, zfs) [default=zfs]: zfs
Create a new ZFS pool? (yes/no) [default=yes]: yes
Would you like to use an existing empty block device (e.g. a disk or partition)? (yes/no) [default=no]: yes
Path to the existing block device: /dev/sda
Do you want to configure a new remote storage pool? (yes/no) [default=no]: no
Would you like to connect to a MAAS server? (yes/no) [default=no]: no
Would you like to configure LXD to use an existing bridge or host interface? (yes/no) [default=no]: no
Would you like to create a new Fan overlay network? (yes/no) [default=yes]: yes
What subnet should be used as the Fan underlay? [default=auto]: 10.136.0.0/16
Would you like stale cached images to be updated automatically? (yes/no) [default=yes]: yes
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]: no
root@orion:~#
```
Let's check the status of the ZFS storage based pool for LXD:
```
root@orion:~# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
local  19.5G   618K  19.5G        -         -     0%     0%  1.00x    ONLINE  -
root@orion:~# zpool status
  pool: local
 state: ONLINE
config:

	NAME        STATE     READ WRITE CKSUM
	local       ONLINE       0     0     0
	  sda       ONLINE       0     0     0

errors: No known data errors
root@orion:~# 
```

Now check the status of the lxdfan0 network interface for the FAN network:
```
root@orion:~# ip addr show lxdfan0
4: lxdfan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default qlen 1000
    link/ether 00:16:3e:35:96:ef brd ff:ff:ff:ff:ff:ff
    inet 240.143.77.1/8 scope global lxdfan0
       valid_lft forever preferred_lft forever
root@orion:~# 
```
Finally, let's check the status of the LXD cluster so far:
```
root@orion:~# lxc cluster list
+--------+----------------------------+-----------------+--------------+----------------+-------------+--------+-------------------+
|  NAME  |            URL             |      ROLES      | ARCHITECTURE | FAILURE DOMAIN | DESCRIPTION | STATE  |      MESSAGE      |
+--------+----------------------------+-----------------+--------------+----------------+-------------+--------+-------------------+
| orion | https://10.136.143.77:8443 | database-leader | x86_64       | default        |             | ONLINE | Fully operational |
|        |                            | database        |              |                |             |        |                   |
+--------+----------------------------+-----------------+--------------+----------------+-------------+--------+-------------------+
root@orion:~# 
```
That’s it, you have a cluster (of one system) with FAN networking and ZFS storage configured. Now, let’s join the other two nodes. First let's add rigel node to the cluster, so on orion execute:
```
root@orion:~# lxc cluster add rigel
Member rigel join token:
eyJzZXJ2ZXJfbmFtZSI6Im9za2FyMiIsImZpbmdlcnByaW50IjoiYTZmNTBjZDQ5YTI1NzM3NWM4YjljNDA2OTMzNjVkYWJhNTgyZDFkYmZlYzMyNzA0YzlhYzM3ZDAwYjdkN2FiMCIsImFkZHJlc3NlcyI6WyIxMC4xMzYuMTQzLjc3Ojg0NDMiXSwic2VjcmV0IjoiYmI4Njg4ZmU0YjE1M2U5OTFmMzY3Yzc5MTQ4NTRhNTJkZWZmZjcwODk1MjQ5OWUxMDU3YzIyOTcyY2Q3MmM2ZCIsImV4cGlyZXNfYXQiOiIyMDIzLTA4LTE0VDE2OjEyOjA0LjM4NTAwNTA1NVoifQ==
root@orion:~# 
```

Then on rigel node verify that you have the volume /dev/sda and get the IP of the internal network interface. Then execute:
```
root@rigel:~# lxd init
Would you like to use LXD clustering? (yes/no) [default=no]: yes
What IP address or DNS name should be used to reach this server? [default=204.48.29.25]: 10.136.192.9
Are you joining an existing cluster? (yes/no) [default=no]: yes
Do you have a join token? (yes/no/[token]) [default=no]: yes
Please provide join token: eyJzZXJ2ZXJfbmFtZSI6Im9za2FyMiIsImZpbmdlcnByaW50IjoiYTZmNTBjZDQ5YTI1NzM3NWM4YjljNDA2OTMzNjVkYWJhNTgyZDFkYmZlYzMyNzA0YzlhYzM3ZDAwYjdkN2FiMCIsImFkZHJlc3NlcyI6WyIxMC4xMzYuMTQzLjc3Ojg0NDMiXSwic2VjcmV0IjoiYmI4Njg4ZmU0YjE1M2U5OTFmMzY3Yzc5MTQ4NTRhNTJkZWZmZjcwODk1MjQ5OWUxMDU3YzIyOTcyY2Q3MmM2ZCIsImV4cGlyZXNfYXQiOiIyMDIzLTA4LTE0VDE2OjEyOjA0LjM4NTAwNTA1NVoifQ==
All existing data is lost when joining a cluster, continue? (yes/no) [default=no] yes
Choose "zfs.pool_name" property for storage pool "local": local
Choose "source" property for storage pool "local": /dev/sda
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]: no
root@rigel:~#
```
Verify the configured ZFS storage:
```
root@rigel:~# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
local  19.5G   618K  19.5G        -         -     0%     0%  1.00x    ONLINE  -
root@rigel:~# zpool status
  pool: local
 state: ONLINE
config:

	NAME        STATE     READ WRITE CKSUM
	local       ONLINE       0     0     0
	  sda       ONLINE       0     0     0

errors: No known data errors
```
And the lxdfan0 network interface for the FAN networking:
```
root@rigel:~# ip addr show lxdfan0
4: lxdfan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default qlen 1000
    link/ether 00:16:3e:a8:2e:ad brd ff:ff:ff:ff:ff:ff
    inet 240.192.9.1/8 scope global lxdfan0
       valid_lft forever preferred_lft forever
root@rigel:~# 
```
The cluster should have rigel as node member added:
```
root@rigel:~# lxc cluster list
To start your first container, try: lxc launch ubuntu:22.04
Or for a virtual machine: lxc launch ubuntu:22.04 --vm

+--------+----------------------------+------------------+--------------+----------------+-------------+--------+-------------------+
|  NAME  |            URL             |      ROLES       | ARCHITECTURE | FAILURE DOMAIN | DESCRIPTION | STATE  |      MESSAGE      |
+--------+----------------------------+------------------+--------------+----------------+-------------+--------+-------------------+
| orion | https://10.136.143.77:8443 | database-leader  | x86_64       | default        |             | ONLINE | Fully operational |
|        |                            | database         |              |                |             |        |                   |
+--------+----------------------------+------------------+--------------+----------------+-------------+--------+-------------------+
| rigel | https://10.136.192.9:8443  | database-standby | x86_64       | default        |             | ONLINE | Fully operational |
+--------+----------------------------+------------------+--------------+----------------+-------------+--------+-------------------+
root@rigel:~#
```
We repeat the same process again for vega, we add the node to the cluster on orion:
```
root@orion:~# lxc cluster add vega
Member vega join token:
eyJzZXJ2ZXJfbmFtZSI6Im9za2FyMyIsImZpbmdlcnByaW50IjoiYTZmNTBjZDQ5YTI1NzM3NWM4YjljNDA2OTMzNjVkYWJhNTgyZDFkYmZlYzMyNzA0YzlhYzM3ZDAwYjdkN2FiMCIsImFkZHJlc3NlcyI6WyIxMC4xMzYuMTQzLjc3Ojg0NDMiLCIxMC4xMzYuMTkyLjk6ODQ0MyJdLCJzZWNyZXQiOiJlY2JiOWMxNzUxODU0NGZjMjUzOGZhMzZlOTdkMWQ3NWNmMDY5ZmRmZTU0YjYzYjUxZDNlOTM3ZmY5YThiYWI0IiwiZXhwaXJlc19hdCI6IjIwMjMtMDgtMTRUMTY6MjI6MDkuMzE1MDcxNjgyWiJ9
root@orion:~#
```
Then on vega node verify that you have the volume /dev/sda and get the IP of the internal network interface. Then execute:
```
root@vega:~# lxd init
Would you like to use LXD clustering? (yes/no) [default=no]: yes
What IP address or DNS name should be used to reach this server? [default=167.99.3.126]: 10.136.0.92
Are you joining an existing cluster? (yes/no) [default=no]: yes
Do you have a join token? (yes/no/[token]) [default=no]: yes
Please provide join token: eyJzZXJ2ZXJfbmFtZSI6Im9za2FyMyIsImZpbmdlcnByaW50IjoiYTZmNTBjZDQ5YTI1NzM3NWM4YjljNDA2OTMzNjVkYWJhNTgyZDFkYmZlYzMyNzA0YzlhYzM3ZDAwYjdkN2FiMCIsImFkZHJlc3NlcyI6WyIxMC4xMzYuMTQzLjc3Ojg0NDMiLCIxMC4xMzYuMTkyLjk6ODQ0MyJdLCJzZWNyZXQiOiJlY2JiOWMxNzUxODU0NGZjMjUzOGZhMzZlOTdkMWQ3NWNmMDY5ZmRmZTU0YjYzYjUxZDNlOTM3ZmY5YThiYWI0IiwiZXhwaXJlc19hdCI6IjIwMjMtMDgtMTRUMTY6MjI6MDkuMzE1MDcxNjgyWiJ9
All existing data is lost when joining a cluster, continue? (yes/no) [default=no] yes
Choose "source" property for storage pool "local": /dev/sda
Choose "zfs.pool_name" property for storage pool "local": local
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]: no
root@vega:~#
```
Verify the configured ZFS storage:
```
root@vega:~# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
local  19.5G   675K  19.5G        -         -     0%     0%  1.00x    ONLINE  -
root@vega:~# zpool status
  pool: local
 state: ONLINE
config:

	NAME        STATE     READ WRITE CKSUM
	local       ONLINE       0     0     0
	  sda       ONLINE       0     0     0

errors: No known data errors
root@vega:~#
```
And the lxdfan0 network device for the FAN networking:
```
root@vega:~# ip addr show lxdfan0
4: lxdfan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default qlen 1000
    link/ether 00:16:3e:0d:ca:29 brd ff:ff:ff:ff:ff:ff
    inet 240.0.92.1/8 scope global lxdfan0
       valid_lft forever preferred_lft forever
root@vega:~#
```
The cluster should have vega as node member added too:
```
root@vega:~# lxc cluster list
To start your first container, try: lxc launch ubuntu:22.04
Or for a virtual machine: lxc launch ubuntu:22.04 --vm

+--------+----------------------------+-----------------+--------------+----------------+-------------+--------+-------------------+
|  NAME  |            URL             |      ROLES      | ARCHITECTURE | FAILURE DOMAIN | DESCRIPTION | STATE  |      MESSAGE      |
+--------+----------------------------+-----------------+--------------+----------------+-------------+--------+-------------------+
| orion | https://10.136.143.77:8443 | database-leader | x86_64       | default        |             | ONLINE | Fully operational |
|        |                            | database        |              |                |             |        |                   |
+--------+----------------------------+-----------------+--------------+----------------+-------------+--------+-------------------+
| rigel | https://10.136.192.9:8443  | database        | x86_64       | default        |             | ONLINE | Fully operational |
+--------+----------------------------+-----------------+--------------+----------------+-------------+--------+-------------------+
| vega  | https://10.136.0.92:8443   | database        | x86_64       | default        |             | ONLINE | Fully operational |
+--------+----------------------------+-----------------+--------------+----------------+-------------+--------+-------------------+
root@vega:~#
```

Now the cluster is initially configured with 3 nodes, one as the primary and the other two as secondary. We can now start creating containers, let's create one based on ubuntu 22.04 on orion:

```
root@orion:~# lxc launch ubuntu:22.04
Creating the instance
Instance name is: internal-lobster          
Starting internal-lobster
root@orion:~#
```

Status of the container:
```
root@orion:~# lxc list
+------------------+---------+-----------------------+------+-----------+-----------+----------+
|       NAME       |  STATE  |         IPV4          | IPV6 |   TYPE    | SNAPSHOTS | LOCATION |
+------------------+---------+-----------------------+------+-----------+-----------+----------+
| internal-lobster | RUNNING | 240.143.77.250 (eth0) |      | CONTAINER | 0         | orion    |
+------------------+---------+-----------------------+------+-----------+-----------+----------+
root@orion:~#
```

We can also enter into the container:
```
root@orion:~# lxc exec internal-lobster bash
root@internal-lobster:~# w
 13:36:14 up 4 min,  0 users,  load average: 0.04, 0.33, 0.20
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
root@internal-lobster:~# 
```

Let's create two more containers,one on rigel and another on vega:
```
root@rigel:~# lxc launch ubuntu:22.04
Creating the instance
Instance name is: active-magpie             
Starting active-magpie
root@rigel:~# 

root@vega:~# lxc launch ubuntu:22.04
Creating the instance
Instance name is: civil-porpoise          
Starting civil-porpoise
root@vega:~# 
```
Now check the status of all containers running in the cluster:
```
root@orion:~# lxc list
+------------------+---------+-----------------------+------+-----------+-----------+----------+
|       NAME       |  STATE  |         IPV4          | IPV6 |   TYPE    | SNAPSHOTS | LOCATION |
+------------------+---------+-----------------------+------+-----------+-----------+----------+
| active-magpie    | RUNNING | 240.192.9.97 (eth0)   |      | CONTAINER | 0         | rigel    |
+------------------+---------+-----------------------+------+-----------+-----------+----------+
| civil-porpoise   | RUNNING | 240.0.92.222 (eth0)   |      | CONTAINER | 0         | vega     |
+------------------+---------+-----------------------+------+-----------+-----------+----------+
| internal-lobster | RUNNING | 240.143.77.250 (eth0) |      | CONTAINER | 0         | orion    |
+------------------+---------+-----------------------+------+-----------+-----------+----------+
root@orion:~#
```
<h3> Creating snapshots of containers</h3>

To create a snapshot of a runing container on orion node:
```
root@orion:~# lxc snapshot internal-lobster internal-lobster-snap20230814
root@orion:~#
```
You can list the snapshot also:
```
root@orion:~# lxc info internal-lobster --resources
Name: internal-lobster
Status: RUNNING
Type: container
Architecture: x86_64
Location: orion
PID: 13077
Created: 2023/08/14 13:31 UTC
Last Used: 2023/08/14 14:16 UTC

Resources:
  Processes: 77
  Disk usage:
    root: 898.06MiB
  CPU usage:
    CPU usage (in seconds): 192
  Memory usage:
    Memory (current): 604.89MiB
  Network usage:
    lo:
      Type: loopback
      State: UP
      MTU: 65536
      Bytes received: 6.48kB
      Bytes sent: 6.48kB
      Packets received: 52
      Packets sent: 52
      IP addresses:
        inet:  127.0.0.1/8 (local)
        inet6: ::1/128 (local)
    eth0:
      Type: broadcast
      State: UP
      Host interface: veth25afb2ea
      MAC address: 00:16:3e:ef:d3:41
      MTU: 1450
      Bytes received: 63.85MB
      Bytes sent: 755.46kB
      Packets received: 16828
      Packets sent: 9028
      IP addresses:
        inet:  240.143.77.250/8 (global)
        inet6: fe80::216:3eff:feef:d341/64 (link)

Snapshots:
+-------------------------------+----------------------+------------+----------+
|             NAME              |       TAKEN AT       | EXPIRES AT | STATEFUL |
+-------------------------------+----------------------+------------+----------+
| internal-lobster-snap20230814 | 2023/08/14 18:14 UTC |            | NO       |
+-------------------------------+----------------------+------------+----------+
root@orion:~#
```
You can also list the snapshot using the normal ZFS commands:
```
root@orion:~# zfs list -t snapshot
NAME                                                                                     USED  AVAIL     REFER  MOUNTPOINT
local/containers/internal-lobster@snapshot-internal-lobster-snap20230814                 479K      -      898M  -
local/images/862f797cfcbc5969ce41e908994d61319ad93b141eaa78cd87c1c43986d95f03@readonly     0B      -      642M  -
root@orion:~#
```
<h3>Expand the local ZFS pool online</h3>

For this we need to add first another volume to the node from DigitalOcean's control panel. For example i attached a new 10GB volume to the vega node and the device /dev/sdb should appear:
```
root@vega:~# lsblk | grep sd
sda       8:0    0    20G  0 disk 
├─sda1    8:1    0    20G  0 part 
└─sda9    8:9    0     8M  0 part 
sdb       8:16   0    10G  0 disk 
root@vega:~# 
```

Then we can expand the local zfs pool:
```
root@vega:~# zpool add local /dev/sdb
```
And we can check now the status of the zpool expanded with the new device and the new size:
```
root@vega:~# zpool status
  pool: local
 state: ONLINE
config:

	NAME        STATE     READ WRITE CKSUM
	local       ONLINE       0     0     0
	  sda       ONLINE       0     0     0
	  sdb       ONLINE       0     0     0

errors: No known data errors
root@vega:~# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
local    29G  1.52G  27.5G        -         -     0%     5%  1.00x    ONLINE  -
root@vega:~#
```

<h3>Adding custom storage volume to containers</h3>

We can add a custom storage volume to a particular container in case you want to separate files. This custom storage volume is created as a ZFS filesystem dataset at the host level and it can be later snapshotted and restored using ZFS commands. For example let's create one named "volume1" on rigel:

```
root@rigel:~# lxc list
+------------------+---------+---------------------+------+-----------+-----------+----------+
|       NAME       |  STATE  |        IPV4         | IPV6 |   TYPE    | SNAPSHOTS | LOCATION |
+------------------+---------+---------------------+------+-----------+-----------+----------+
| active-magpie    | RUNNING | 240.192.9.97 (eth0) |      | CONTAINER | 0         | rigel   |
+------------------+---------+---------------------+------+-----------+-----------+----------+
| civil-porpoise   | RUNNING | 240.0.92.222 (eth0) |      | CONTAINER | 0         | vega   |
+------------------+---------+---------------------+------+-----------+-----------+----------+
| internal-lobster | RUNNING | 240.143.77.250 (eth0) |    | CONTAINER | 1         | orion   |
+------------------+---------+---------------------+------+-----------+-----------+----------+
root@rigel:~# 

root@rigel:~# lxc storage list
+-------+--------+-------------+---------+---------+
| NAME  | DRIVER | DESCRIPTION | USED BY |  STATE  |
+-------+--------+-------------+---------+---------+
| local | zfs    |             | 8       | CREATED |
+-------+--------+-------------+---------+---------+
root@rigel:~# lxc storage volume create local volume1
Storage volume volume1 created
root@rigel:~#
```

Now we attach the created "volume1" custom storage volume to the container active-magpie and configure it to be mounted under /testvol in the container:
```
root@rigel:~# lxc storage volume attach local volume1 active-magpie /testvol

root@rigel:~# lxc exec active-magpie bash
root@active-magpie:~# cd /
root@active-magpie:/# ls
bin   dev  home  lib32  libx32  mnt  proc  run   snap  sys      tmp  var
boot  etc  lib   lib64  media   opt  root  sbin  srv   testvol  usr
root@active-magpie:/# cd testvol/
root@active-magpie:/testvol# ls
root@active-magpie:/testvol# echo "0010101010" >  hello.txt
root@active-magpie:/testvol# ls -l
total 1
-rw-r--r-- 1 root root 11 Aug 15 18:14 hello.txt
root@active-magpie:/testvol# 
```
We can also see the related ZFS filesystem dataset created at the host level:
```
root@rigel:~# zfs list | egrep "NAME|custom"  | egrep "NAME|volume1"
NAME                                                                                    USED  AVAIL     REFER  MOUNTPOINT
local/custom/default_volume1                                                             24K  17.5G       24K  legacy
root@rigel:~# 
```

<h3>Setting quotas and limits to running containers</h3>

You can set CPU/Memory usage limits to containers and also set disk space quotas to the custom storage volumes. 
For example setting Memory limits for container running on rigel:
```
root@rigel:~# free -m
               total        used        free      shared  buff/cache   available
Mem:            1975        1297          83           4         595         482
Swap:              0           0           0
root@rigel:~# lxc exec active-magpie bash
root@active-magpie:~# free -m
               total        used        free      shared  buff/cache   available
Mem:            1975          78        1828           0          68        1897
Swap:              0           0           0
root@active-magpie:~# exit
exit
root@rigel:~# lxc config set active-magpie limits.memory 1GiB
root@rigel:~# lxc exec active-magpie bash
root@active-magpie:~# free -m
               total        used        free      shared  buff/cache   available
Mem:            1024          78         876           0          68         945
Swap:              0           0           0
root@active-magpie:~#
```
Setting CPU limits for container running on rigel to use only one core and up to 50% of it. If you install the stress program in the container after configuring the limits and execute stress -c 1 you will see that the CPU usage by the program will hover around 50% and it will never reach 100%.
```
root@rigel:~# lscpu | grep "CPU(s)" | head -1
CPU(s):                          2
root@active-magpie:~# lscpu | grep "CPU(s)" | head -1
CPU(s):                          2
root@active-magpie:~#
root@rigel:~# lxc config set active-magpie limits.cpu 1
root@rigel:~# lxc config set active-magpie limits.cpu.allowance 50ms/100ms
root@rigel:~#
root@rigel:~# lxc exec active-magpie bash
root@active-magpie:~# lscpu | grep "CPU(s)" | head -1
CPU(s):                          1
root@active-magpie:~#
root@active-magpie:~# stress -c 1 &
[1] 3299
root@active-magpie:~#
root@active-magpie:~# top -b | grep -A6 top
top - 14:20:21 up 2 days, 42 min,  0 users,  load average: 0.26, 0.23, 0.12
Tasks:  25 total,   2 running,  23 sleeping,   0 stopped,   0 zombie
%Cpu(s): 50.5 us,  0.3 sy,  0.0 ni, 49.2 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   1024.0 total,    798.2 free,     76.2 used,    149.6 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.    947.8 avail Mem 

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
   3300 root      20   0    3704    112      0 R  49.8   0.0   1:06.45 stress
```

Setting disk quota to custom storage volume used by running container on rigel:
```
root@rigel:~# lxc exec active-magpie bash
root@active-magpie:~# df -h | egrep  "Size|testvol"
Filesystem                      Size  Used Avail Use% Mounted on
local/custom/default_volume1   1000M  477M  524M  48% /testvol
root@active-magpie:~# exit
root@rigel:~#
root@rigel:~# lxc storage volume set local custom/volume1 size=750MB
root@rigel:~# lxc exec active-magpie bash
root@active-magpie:~# df -h | egrep  "Size|testvol"
Filesystem                      Size  Used Avail Use% Mounted on
local/custom/default_volume1    716M  477M  239M  67% /testvol
root@active-magpie:~# cd /testvol
root@active-magpie:/testvol# dd if=/dev/random of=filename bs=250M count=1
dd: error writing 'filename': Disk quota exceeded
1+0 records in
0+0 records out
249823232 bytes (250 MB, 238 MiB) copied, 6.68017 s, 37.4 MB/s
root@active-magpie:/testvol# df -h | egrep  "Size|testvol"
Filesystem                      Size  Used Avail Use% Mounted on
local/custom/default_volume1    716M  716M     0 100% /testvol
root@active-magpie:/testvol# ls -lh filename
-rw-r--r-- 1 root root 239M Aug 16 14:37 filename
root@active-magpie:/testvol#
```

Keep in mind that by default, LXD enables compression when creating a ZFS pool, so custom storage volumes will inherit this setting as enabled by default. Custom storage volumes related to a pool can have different compression algorithms. Enabling compression means that files could occupy less space on the filesystem, so sometimes df -h will not show expected values like in ext4 filesystems.

```
root@rigel:~# zfs get compression local/custom/default_volume1
NAME                          PROPERTY     VALUE           SOURCE
local/custom/default_volume1  compression  on              local
root@rigel:~# zfs get compressratio local/custom/default_volume1
NAME                          PROPERTY       VALUE  SOURCE
local/custom/default_volume1  compressratio  1.34x  -
root@rigel:~#
```

<h3>DNS configuration</h3>

Now lets configure DNS resolution so we can get the IP of each container if needed. Execute these steps on each node of the cluster. 

Get the IP used by the running forkdns daemon on orion:
```
root@orion:~# ps -ef  |grep forkdns | grep fan
lxd         9949    9586  0 13:01 ?        00:00:00 /snap/lxd/current/bin/lxd forkdns 240.143.77.1:1053 lxd lxdfan0
root@orion:~#
```
Then execute:
```
root@orion:~# resolvectl dns lxdfan0 240.143.77.1
root@orion:~# resolvectl domain lxdfan0 '~lxd'
root@orion:~# resolvectl status lxdfan0
Link 4 (lxdfan0)
    Current Scopes: DNS
         Protocols: -DefaultRoute +LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 240.143.77.1
       DNS Servers: 240.143.77.1
        DNS Domain: ~lxd
root@orion:~# 
```
Repeat the same process on the other nodes using the respective IPs used by their forkdns daemons.

To make these DNS changes persistent after reboots on all nodes:

Create the file /etc/systemd/system/lxd-dns-lxdfan0.service with the right IP used by the forkdns daemon:

```
[Unit]
Description=LXD per-link DNS configuration for lxdfan0
BindsTo=sys-subsystem-net-devices-lxdfan0.device
After=sys-subsystem-net-devices-lxdfan0.device

[Service]
Type=oneshot
ExecStart=/usr/bin/resolvectl dns lxdfan0 240.143.77.1 
ExecStart=/usr/bin/resolvectl domain lxdfan0 '~lxd'
ExecStopPost=/usr/bin/resolvectl revert lxdfan0
RemainAfterExit=yes

[Install]
WantedBy=sys-subsystem-net-devices-lxdfan0.device
```
Then execute:

```
root@orion:~# systemctl daemon-reload
root@orion:~# systemctl start lxd-dns-lxdfan0.service
root@orion:~# systemctl status lxd-dns-lxdfan0.service
● lxd-dns-lxdfan0.service - LXD per-link DNS configuration for lxdfan0
     Loaded: loaded (/etc/systemd/system/lxd-dns-lxdfan0.service; disabled; vendor preset: enabled)
     Active: active (exited) since Mon 2023-08-14 13:55:55 UTC; 9s ago
    Process: 12782 ExecStart=/usr/bin/resolvectl dns lxdfan0 240.143.77.1 (code=exited, status=0/SUCCESS)
    Process: 12783 ExecStart=/usr/bin/resolvectl domain lxdfan0 ~lxd (code=exited, status=0/SUCCESS)
   Main PID: 12783 (code=exited, status=0/SUCCESS)
        CPU: 13ms
root@orion:~# systemctl enable lxd-dns-lxdfan0.service
Created symlink /etc/systemd/system/sys-subsystem-net-devices-lxdfan0.device.wants/lxd-dns-lxdfan0.service → /etc/systemd/system/lxd-dns-lxdfan0.service.
Unit /etc/systemd/system/lxd-dns-lxdfan0.service is added as a dependency to a non-existent unit sys-subsystem-net-devices-lxdfan0.device.
root@orion:~#
```
Repeat the same steps in the other nodes to make the changes persistent across reboots. Before doing this execute on each secondary node these commands to start forkdns daemon and get the IP that it uses:
```
root@rigel:~# systemctl reload snap.lxd.daemon
root@rigel:~# ps -ef  |grep forkdns | grep lxdfan0
lxd        12583   12416  0 14:02 ?        00:00:00 /snap/lxd/current/bin/lxd forkdns 240.192.9.1:1053 lxd lxdfan0
root@rigel:~#

root@vega:~# systemctl reload snap.lxd.daemon
root@vega:~# ps -ef | grep forkdns | grep lxdfan0
lxd        12489   12323  0 14:02 ?        00:00:00 /snap/lxd/current/bin/lxd forkdns 240.0.92.1:1053 lxd lxdfan0
root@vega:~#
```
Now after creating /etc/systemd/system/lxd-dns-lxdfan0.service on both nodes and executing the appropiate commands, verify that you can resolve the hostnames of the running containers, for example from vega:
```
root@vega:~# dig active-magpie.lxd +short
240.192.9.97
root@vega:~# dig internal-lobster.lxd +short
240.143.77.250
root@vega:~# dig civil-porpoise.lxd +short
240.0.92.222
root@vega:~# 
```
We can confirm the IPs are accurate based on lxc list:
```
root@vega:~# lxc list
+------------------+---------+-----------------------+------+-----------+-----------+----------+
|       NAME       |  STATE  |         IPV4          | IPV6 |   TYPE    | SNAPSHOTS | LOCATION |
+------------------+---------+-----------------------+------+-----------+-----------+----------+
| active-magpie    | RUNNING | 240.192.9.97 (eth0)   |      | CONTAINER | 0         | rigel    |
+------------------+---------+-----------------------+------+-----------+-----------+----------+
| civil-porpoise   | RUNNING | 240.0.92.222 (eth0)   |      | CONTAINER | 0         | vega     |
+------------------+---------+-----------------------+------+-----------+-----------+----------+
| internal-lobster | RUNNING | 240.143.77.250 (eth0) |      | CONTAINER | 0         | orion    |
+------------------+---------+-----------------------+------+-----------+-----------+----------+
root@vega:~#
```

<h3>Migrating containers between nodes</h3>
	
Lets migrate the container internal-lobster running on orion to vega:
```
root@orion:~# lxc list
+------------------+---------+-----------------------+------+-----------+-----------+----------+
|       NAME       |  STATE  |         IPV4          | IPV6 |   TYPE    | SNAPSHOTS | LOCATION |
+------------------+---------+-----------------------+------+-----------+-----------+----------+
| active-magpie    | RUNNING | 240.192.9.97 (eth0)   |      | CONTAINER | 0         | rigel    |
+------------------+---------+-----------------------+------+-----------+-----------+----------+
| civil-porpoise   | RUNNING | 240.0.92.222 (eth0)   |      | CONTAINER | 0         | vega     |
+------------------+---------+-----------------------+------+-----------+-----------+----------+
| internal-lobster | RUNNING | 240.143.77.250 (eth0) |      | CONTAINER | 0         | orion    |
+------------------+---------+-----------------------+------+-----------+-----------+----------+
root@orion:~# lxc stop internal-lobster
root@orion:~# lxc move internal-lobster --target=vega
root@orion:~# lxc start internal-lobster
root@orion:~# lxc list
+------------------+---------+---------------------+------+-----------+-----------+----------+
|       NAME       |  STATE  |        IPV4         | IPV6 |   TYPE    | SNAPSHOTS | LOCATION |
+------------------+---------+---------------------+------+-----------+-----------+----------+
| active-magpie    | RUNNING | 240.192.9.97 (eth0) |      | CONTAINER | 0         | rigel    |
+------------------+---------+---------------------+------+-----------+-----------+----------+
| civil-porpoise   | RUNNING | 240.0.92.222 (eth0) |      | CONTAINER | 0         | vega     |
+------------------+---------+---------------------+------+-----------+-----------+----------+
| internal-lobster | RUNNING | 240.0.92.250 (eth0) |      | CONTAINER | 0         | vega     |
+------------------+---------+---------------------+------+-----------+-----------+----------+
root@orion:~#
```
You can see from above that the container internal-lobster is now running on vega.

<h3> Running a basic web page using nginx/php-fpm in one of the Containers</h3> 	

First lets install haproxy on vega node which will receive any http request and redirect it to the correct container:
```
root@vega:~# apt install haproxy
```

Now edit /etc/haproxy/haproxy.cfg, use the forkdns IP for the resolver mydns nameserver line that haproxy would use:
```
root@vega:~# vi /etc/haproxy/haproxy.cfg 
root@vega:~# cat /etc/haproxy/haproxy.cfg
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

	# Default SSL material locations
	ca-base /etc/ssl/certs
	crt-base /etc/ssl/private

	# See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
	log	global
	mode	http
	option	httplog
	option	dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
	errorfile 400 /etc/haproxy/errors/400.http
	errorfile 403 /etc/haproxy/errors/403.http
	errorfile 408 /etc/haproxy/errors/408.http
	errorfile 500 /etc/haproxy/errors/500.http
	errorfile 502 /etc/haproxy/errors/502.http
	errorfile 503 /etc/haproxy/errors/503.http
	errorfile 504 /etc/haproxy/errors/504.http

resolvers mydns
    nameserver dns1 240.0.92.1:53
    hold valid 1h
    accepted_payload_size 8192 # allow larger DNS payloads

frontend http-in    

    bind *:80
    mode http

    default_backend internal-lobster_backend

backend internal-lobster_backend
  mode http
  server internal-lobster_server1 internal-lobster.lxd:80 check
```

Restart haproxy on vega node:
```
root@vega:~# systemctl restart haproxy
root@vega:~#
```
	
Now lets install nginx/php-fpm on the internal-lobster container running on vega:
```
root@vega:~# lxc exec  internal-lobster bash
root@internal-lobster:~# apt-get update
root@internal-lobster:~# apt install nginx
root@internal-lobster:~# apt install php8.1-fpm 
```

Now inside the internal-lobster container execute:
```
root@internal-lobster:/var/www/html# echo "<?php" > /var/www/html/index.php 
root@internal-lobster:/var/www/html# echo "phpinfo();" >> /var/www/html/index.php
```
Edit /etc/nginx/sites-enabled/default on internal-lobster container:
```
server {
        listen 80 default_server;
        listen [::]:80 default_server;

        root /var/www/html;

        index index.html index.htm index.nginx-debian.html index.php;

        server_name _;

        location / {
                try_files $uri $uri/ =404;
        }

        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/run/php/php8.1-fpm.sock;
        }

}
```

Restart nginx on internal-lobster container:
```
root@internal-lobster:~# systemctl restart nginx
root@internal-lobster:~#
```

On vega node get the IP address of the external network interface:
```
root@vega:~# ip addr show eth0 | grep global | head -1
    inet 167.99.3.126/20 brd 167.99.15.255 scope global eth0
root@vega:~# 
```
On your local machine add this line to your /etc/hosts file based on the IP from above:
```
167.99.3.126 internal-lobster.com
```
Now try accessing the URL with your browser or CURL, you should see a PHP info page and get a 200 HTTP status code:
```
$ curl -vI http://internal-lobster.com/index.php
*   Trying 167.99.3.126:80...
* Connected to internal-lobster.com (167.99.3.126) port 80 (#0)
> HEAD /index.php HTTP/1.1
> Host: internal-lobster.com
> User-Agent: curl/7.29.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
< server: nginx/1.18.0 (Ubuntu)
server: nginx/1.18.0 (Ubuntu)
< date: Mon, 14 Aug 2023 15:46:39 GMT
date: Mon, 14 Aug 2023 15:46:39 GMT
< content-type: text/html; charset=UTF-8
content-type: text/html; charset=UTF-8

< 
* Connection #0 to host internal-lobster.com left intact
```
This confirms that the request was redirected by haproxy to the right container, received by nginx and processed by php-fpm. 

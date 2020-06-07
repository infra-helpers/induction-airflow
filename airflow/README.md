Apache Airflow Setup on Proxmox LXC Containers
==============================================

[![Vršič Pass, Soča, Slovenia, by Miha Rekar on Unsplash](../img/miha-rekar-VfwwaXyg63Y-unsplash-small.jpg)](https://unsplash.com/photos/VfwwaXyg63Y)

# Overview
That
[README](https://github.com/infra-helpers/induction-airflow/tree/master/airflow/README.md)
is part of a [broader tutorial about
workflow engines](https://github.com/infra-helpers/induction-airflow),
and gives details on how to setup Apache Airflow on a LXC container of
a Proxmox host, secured by an SSH gateway, also acting as an Nginx-based
reverse proxy.

For the installation of the Proxmox host and LXC containers themselves,
refer to the
[dedicated tutorial on GitHub](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/proxmox/README.md),
itself a full tutorial on Kubernetes (k8s). Only a summary is given here,
focusing on Apache Airflow.

# Table of Content (ToC)
- [Apache Airflow Setup on Proxmox LXC Containers](#apache-airflow-setup-on-proxmox-lxc-containers)
- [Overview](#overview)
- [Table of Content (ToC)](#table-of-content--toc-)
- [References](#references)
  * [Installation](#installation)
  * [LXC Containers on Proxmox](#lxc-containers-on-proxmox)
  * [Apache Airflow](#apache-airflow)
- [Host preparation](#host-preparation)
  * [Get the latest CentOS templates](#get-the-latest-centos-templates)
  * [Kernel modules](#kernel-modules)
    + [Overlay module](#overlay-module)
    + [`nf_conntrack`](#-nf-conntrack-)
- [SSH gateway and reverse proxy](#ssh-gateway-and-reverse-proxy)
- [Airflow node](#airflow-node)
  * [PostgreSQL database server](#postgresql-database-server)
  * [Docker](#docker)
  * [SystemD services for Airflow](#systemd-services-for-airflow)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

# References

## Installation
* PostgreSQL database:
 + https://www.digitalocean.com/community/tutorials/how-to-install-postgresql-on-ubuntu-20-04-quickstart:
 + https://medium.com/@Newt_Tan/apache-airflow-installation-based-on-postgresql-database-26549b154d8
* Airflow on Ubuntu:
  + https://airflow.apache.org/docs/stable/installation.html
  + https://www.corbettanalytics.com/installing-airflow-on-ubuntu-18/
  + https://www.ryanmerlin.com/2019/07/apache-airflow-installation-on-ubuntu-18-04-18-10/
  + https://github.com/apache/airflow/tree/master/scripts/systemd

## LXC Containers on Proxmox
* [Setup of a Proxmox host](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/proxmox/README.md)
* [Setup of LXC containers on Proxmox](https://github.com/cloud-helpers/kubernetes-hard-way-bare-metal/blob/master/lxc/README.md)
* [Setup of an Elasticsearch (ES) cluster on Proxmox](https://github.com/infra-helpers/induction-monitoring/tree/master/elasticsearch/README.md)

## Apache Airflow
* [Apache Airflow](https://airflow.apache.org/)
  + [Airflow - Quick Start](https://airflow.apache.org/docs/stable/start.html)
  + [Airflow - Installation](https://airflow.apache.org/docs/stable/installation.html)
  + [Airflow - Tutorial](https://airflow.apache.org/docs/stable/tutorial.html)
  + [Airflow - How-to Guides](https://airflow.apache.org/docs/stable/howto/index.html)
* [Data Engineering — Basics of Apache Airflow — Build Your First Pipeline](https://towardsdatascience.com/data-engineering-basics-of-apache-airflow-build-your-first-pipeline-eefecb7f1bb9),
  June 2019,
  [Nicholas Leong](https://towardsdatascience.com/@nickmydata)


# Host preparation
In that section, it is assumed that we are logged on the Proxmox host
as `root`.

The following parameters are used in the remaining of the guide, and may be
adapted according to your configuration:
* IP of the routing gateway on the host (typically ends with `.254`:
  `HST_GTW_IP`
* (Potentially virtual) MAC address of the Airflow gateway: `GTW_MAC`
* IP address of the Airflow gateway: `GTW_IP`
* VM ID of that Airflow gateway: `103`
* For the LXC container-based setup, the private IP addresses
  are summarized as below:

| VM ID | Private IP  |  Host name (full)    | Short name  |
| ----- | ----------- | -------------------- | ----------- |
|  104  | 10.30.2.4   | proxy8.example.com   | proxy8      |
|  200  | 10.30.2.200 | arfl-int.example.com | arfl-int    |

* Extract of the host network configuration:
```bash
root@proxmox:~$ cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

# The loopback network interface
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual

auto eno2
iface eno2 inet manual

auto bond0
iface bond0 inet manual
        bond-slaves eno1 eno2
        bond-miimon 100
        bond-mode active-backup

# vmbr0: Bridging. Make sure to use only MAC adresses that were assigned to you.
auto vmbr0
iface vmbr0 inet static
        address ${HST_IP}
        netmask 255.255.255.0
        gateway ${HST_GTW_IP}
        bridge_ports bond0
        bridge_stp off
        bridge_fd 0

auto vmbr2
iface vmbr2 inet static
        address 10.30.2.2
        netmask 255.255.255.0
        bridge-ports none
        bridge-stp off
        bridge-fd 0
        post-up echo 1 > /proc/sys/net/ipv4/ip_forward
        post-up iptables -t nat -A POSTROUTING -s '10.30.2.0/24' -o vmbr0 -j MASQUERADE
        post-down iptables -t nat -D POSTROUTING -s '10.30.2.0/24' -o vmbr0 -j MASQUERADE

root@proxmox:~$ cat /etc/systemd/network/50-default.network
# This file sets the IP configuration of the primary (public) network device.
# You can also see this as "OSI Layer 3" config.
# It was created by the OVH installer, please be careful with modifications.
# Documentation: man systemd.network or https://www.freedesktop.org/software/systemd/man/systemd.network.html

[Match]
Name=vmbr0

[Network]
Description=network interface on public network, with default route
DHCP=no
Address=${HST_IP}/24
Gateway=${HST_GTW_IP}
IPv6AcceptRA=no
NTP=ntp.ovh.net
DNS=127.0.0.1
DNS=8.8.8.8

[Address]
Address=${HST_IPv6}

[Route]
Destination=2001:0000:0000:34ff:ff:ff:ff:ff
Scope=link

root@proxmox:~$ cat /etc/systemd/network/50-public-interface.link
# This file configures the relation between network device and device name.
# You can also see this as "OSI Layer 2" config.
# It was created by the OVH installer, please be careful with modifications.
# Documentation: man systemd.link or https://www.freedesktop.org/software/systemd/man/systemd.link.html

[Match]
Name=vmbr0

[Link]
Description=network interface on public network, with default route
MACAddressPolicy=persistent
NamePolicy=kernel database onboard slot path mac
#Name=eth0	# name under which this interface is known under OVH rescue system
#Name=eno1	# name under which this interface is probably known by systemd
```

* The maximal virtual memory needs to be increased on the host:
```bash
$ sysctl -w vm.max_map_count=262144
$ cat >> /etc/sysctl.conf << _EOF

###########################
# Elasticsearch in VM
vm.max_map_count = 262144

_EOF
```

## Get the latest CentOS templates
* List
 [available templates supported by Proxmox](http://download.proxmox.com/images/system/):
```bash
root@proxmox:~$ pveam update
root@proxmox:~$ pveam available
```

* Download locally a remotely available template:
```bash 
root@proxmox:~$ pveam download local centos-8-default_20191016_amd64.tar.xz
root@proxmox:~$ pveam list local
root@proxmox:~$ # Which should give the same result as:
root@proxmox:~$ ls -lFh /var/lib/vz/template/cache
```

* (not needed here) Download the latest template from the
  [Linux containers site](https://us.images.linuxcontainers.org/images/centos/7/amd64/default/)
  (change the date and time-stamp according to the time you download that
  template):
```bash
root@proxmox:~$ wget https://us.images.linuxcontainers.org/images/centos/8/amd64/default/20200411_07:08/rootfs.tar.xz -O /vz/template/cache/centos-8-default_20200411_amd64.tar.xz
```

## Kernel modules

### Overlay module
```bash
root@proxmox:~$ modprobe overlay && \
  cat > /etc/modules-load.d/docker-overlay.conf << _EOF
overlay
_EOF
```

### `nf_conntrack`
* The `hashsize` parameter should be set to at least 32,768.
  We can set it tp 65,536. But the Proxmox firewall resets the value every
  so often to 16384. The following shows how that parameter may be set
  when the module is loaded:
```bash
root@proxmox:~$ modprobe nf_conntrack hashsize=65536 && \
  cat > /etc/modules-load.d/nf_conntrack.conf << _EOF
options nf_conntrack hashsize=65536
_EOF
root@proxmox:~$ # echo "65536" > /sys/module/nf_conntrack/parameters/hashsize
root@proxmox:~$ cat /sys/module/nf_conntrack/parameters/hashsize
65536
```

* However, as the Proxmox (PVE) firewall resets that value periodically,
  a work around (to make that change permanent) is to alter
  the PVE firewall script:
```bash
root@proxmox:~$ sed -i -e 's|my $hashsize = int($max/4);|my $hashsize = $max;|g' /usr/share/perl5/PVE/Firewall.pm
root@proxmox:~$ systemctl restart pve-firewall.service
root@proxmox:~$ cat /sys/module/nf_conntrack/parameters/hashsize
65536
```

# SSH gateway and reverse proxy
* The goal is both to set up an SSH gateway and to create an end-point
  for Airflow (https://airflow.example.com)

* All the traffic from the internet to Airflow is then forced
  through the gateway (SSH) and/or the reverse proxy (HTTP/HTTPS)

* The IP address of the web end-point is the gateway's one (`GTW_IP`)

* Add a few DNS `A` records in the `example.com` domain:
  + New record: `arfl-int.example.com`
    - Target: `10.30.2.200`
  + New record: `kibana.example.com`
    - Target: `GTW_IP` (public IP of `proxy8.example.com`, needed for SSL)

* Create the LXC container for the SSH gateway:
```bash
root@proxmox:~$ pct create 104 local:vztmpl/centos-8-default_20191016_amd64.tar.xz --arch amd64 --cores 1 --hostname proxy8.example.com --memory 16134 --swap 32268 --net0 name=eth0,bridge=vmbr0,firewall=1,gw=${HST_GTW_IP},hwaddr=${GTW_MAC},ip=${GTW_IP}/32,type=veth --net1 name=eth1,bridge=vmbr2,ip=10.30.2.4/24,type=veth --onboot 1 --ostype centos
root@proxmox:~$ pct resize 104 rootfs 10G
root@proxmox:~$ ls -laFh /var/lib/vz/images/104/vm-104-disk-0.raw
-rw-r----- 1 root root 10G Apr 12 00:00 /var/lib/vz/images/104/vm-104-disk-0.raw
root@proxmox:~$ cat /etc/pve/lxc/104.conf 
arch: amd64
cores: 2
hostname: proxy8.example.com
memory: 16134
net0: name=eth0,bridge=vmbr0,firewall=1,gw=${HST_GTW_IP},hwaddr=${GTW_MAC},ip=${GTW_IP}/32,type=veth
net1: name=eth1,bridge=vmbr2,hwaddr=<some-mac-addr>,ip=10.30.2.4/24,type=veth
onboot: 1
ostype: centos
rootfs: local:104/vm-104-disk-0.raw,size=10G
swap: 32268
```

* As of April 2020, the LXC templates for (Fedora and) CentOS 8
  do not come with NetworkManager (more specifically, `nmcli` for the tool
  and `NeetworkManager-tui` for the RPM package).
  And to install it, the network needs to be set up, manually first.
  Once NetworkManager has been installed, the network is then setup
  automatically at startup times.

* Manual setup of the network for CentOS 8:
```bash
root@proxmox:~$ pct start 104 && pct enter 104
[root@proxy8]# mkdir ~/bin && cat > ~/bin/netup.sh << _EOF
#!/bin/sh

ip addr add ${HST_GTW_IP}/5 dev eth0
ip link set eth0 up
ip route add default via ${GTW_IP} dev eth0

ip addr add 10.30.2.4/24 dev eth1
ip link set eth1 up

_EOF
[root@proxy8]# chmod 755 ~/bin/netup.sh
[root@proxy8]# ~/bin/netup.sh
[root@proxy8]# dnf -y upgrade
[root@proxy8]# dnf -y install epel-release
[root@proxy8]# dnf -y install NetworkManager-tui
[root@proxy8]# systemctl start NetworkManager.service \
	&& systemctl status NetworkManager.service \
	&& systemctl enable NetworkManager.service
[root@proxy8]# nmcli con # to check the name of the connection
[root@proxy8]# nmcli con up "System eth0"
[root@proxy8]# exit
```

* Complement the installation on the SSH gateway/reverse proxy container.
  For security reason, it may be a good idea to change the SSH port
  from `22` to, say `7022`:
```bash
root@proxmox:~$ pct enter 104
[root@proxy8]# dnf -y install hostname rpmconf dnf-utils wget curl net-tools tar
[root@proxy8]# hostnamectl set-hostname proxy8.example.com
[root@proxy8]# dnf -y install htop less screen bzip2 dos2unix man man-pages
[root@proxy8]# dnf -y install sudo whois ftp rsync vim git-all patch mutt
[root@proxy8]# dnf -y install java-11-openjdk-headless
[root@proxy8]# dnf -y install nginx python3-pip
[root@proxy8]# pip-3 install certbot-nginx
[root@proxy8]# rpmconf -a
[root@proxy8]# ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
[root@proxy8]# setenforce 0
[root@proxy8]# dnf -y install openssh-server
root@proxy8# sed -i -e 's/#Port 22/Port 7022/g' /etc/ssh/sshd_config
[root@proxy8]# systemctl start sshd.service \
	&& systemctl status sshd.service \
	&& systemctl enable sshd.service
[root@proxy8]# mkdir ~/.ssh && chmod 700 ~/.ssh
[root@proxy8]# cat > ~/.ssh/authorized_keys << _EOF
ssh-rsa AAAA<Add-Your-own-SSH-public-key>BLAgU first.last@example.com
_EOF
[root@proxy8]# chmod 600 ~/.ssh/authorized_keys
[root@proxy8]# passwd -d root
[root@proxy8]# rpm --import http://wiki.psychotic.ninja/RPM-GPG-KEY-psychotic
[root@proxy8]# rpm -ivh http://packages.psychotic.ninja/7/base/x86_64/RPMS/keychain-2.8.0-3.el7.psychotic.noarch.rpm
```

* SSL certificates.
  If `certbot` is not available in `/usr/local/bin` (from the installation
  by `pip-3` above), it can get installed thanks to
  https://certbot.eff.org/lets-encrypt/centosrhel8-nginx.html
```bash
[root@proxy8]# /usr/local/bin/certbot --nginx # and then certbot renew
```

* Some convenient additional setup:
```bash
[root@proxy8]# cat > ~/.screenrc << _EOF
hardstatus alwayslastline "%{.kW}%-w%{.B}%n %t%{-}%{=b kw}%?%+w%? %=%c %d/%m/%Y" #B&W & date&time
startup_message off
defscrollback 1024
_EOF
[root@proxy8]# cat > /etc/hosts << _EOF
# Local VM
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

# ES cluster
10.30.2.200     arfl-int.example.com     arfl-int

_EOF
```

* A few handy aliases:
```bash
root@proxy8:~# cat >> ~/.bashrc << _EOF

# Source aliases
if [ -f ~/.bash_aliases ]
then
        . ~/.bash_aliases
fi

_EOF
root@proxy8:~$ cat ~/.bash_aliases << _EOF
# User specific aliases and functions
alias dir='ls -laFh --color'
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'

_EOF
root@proxy8:~# . ~/.bashrc
root@proxy8:~# exit
```

* Configure Nginx as a reverse proxy for Airflow:
```bash
root@proxmox:~$ pct enter 104
[root@proxy8]# cat > /etc/nginx/conf.d/tiairflow.conf << _EOF
server {
  server_name airflow.example.com;
  access_log  /var/log/nginx/tiairflow.access.log;

  auth_basic "Restricted Access";
  auth_basic_user_file /etc/nginx/.airflow-user;

  location / {
      proxy_set_header  Host \$host;
      proxy_set_header  X-Real-IP \$remote_addr;
      proxy_set_header  X-Forwarded-For \$proxy_add_x_forwarded_for;
      proxy_set_header  X-Forwarded-Proto \$scheme;

      # Fix the "It appears that your reverse proxy set up is broken" error.
      proxy_pass         http://10.30.2.200:8080;
      proxy_read_timeout 90;

      proxy_redirect     http://10.30.2.200 https://\$host;
  }
}

_EOF
[root@proxy8]# htpasswd -c /etc/nginx/.airflow-user <airflow-user>
New password: 
Re-type new password: 
Adding password for user <airflow-user>>
[root@proxy8]# /usr/local/bin/certbot --nginx
[root@proxy8]# nginx -t
[root@proxy8]# nginx -s reload
[root@proxy8]# exit
```

# Airflow node
* Create the LXC container:
```bash
root@proxmox:~$ pct create 200 local:vztmpl/centos-8-default_20191016_amd64.tar.xz --arch amd64 --cores 2 --hostname arfl-int.example.com --memory 16134 --swap 32268 --net0 name=eth0,bridge=vmbr2,gw=10.30.2.2,ip=10.30.2.200/24,type=veth --onboot 1 --ostype centos
root@proxmox:~$ pct resize 200 rootfs 50G
root@proxmox:~$ ls -laFh /var/lib/vz/images/200/vm-200-disk-0.raw
-rw-r----- 1 root root 50G Dec 19 22:27 /var/lib/vz/images/200/vm-200-disk-0.raw
root@proxmox:~$ cat /etc/pve/lxc/200.conf
arch: amd64
cores: 2
hostname: arfl-int.example.com
memory: 16134
net0: name=eth0,bridge=vmbr2,gw=10.30.2.2,hwaddr=1A:EC:7F:9E:90:34,ip=10.30.2.200/24,type=veth
onboot: 1
ostype: centos
rootfs: local:200/vm-200-disk-0.raw,size=50G
swap: 32268
```

* Manual setup of the network:
```bash
root@proxmox:~$ pct start 200 && pct enter 200
[root@arfl-int]# mkdir ~/bin && cat > ~/bin/netup.sh << _EOF
#!/bin/sh

ip addr add 10.30.2.200/24 dev eth0
ip link set eth0 up
ip route add default via 10.30.2.2 dev eth0

_EOF
[root@arfl-int]# chmod 755 ~/bin/netup.sh
[root@arfl-int]# ~/bin/netup.sh # may not be needed
[root@arfl-int]# dnf -y upgrade
[root@arfl-int]# dnf -y install epel-release
[root@arfl-int]# dnf -y install NetworkManager-tui
[root@arfl-int]# systemctl start NetworkManager.service \
	&& systemctl status NetworkManager.service \
	&& systemctl enable NetworkManager.service
[root@arfl-int]# nmcli con # to check the name of the connection
[root@arfl-int]# nmcli con up "System eth0"
[root@arfl-int]# exit
```

* Complement the installation:
```bash
root@proxmox:~$ pct enter 200
[root@arfl-int]# dnf -y install hostname rpmconf dnf-utils wget curl net-tools tar
[root@arfl-int]# hostnamectl set-hostname arfl-int.example.com
[root@arfl-int]# dnf -y install htop less screen bzip2 dos2unix man man-pages
[root@arfl-int]# dnf -y install sudo ftp rsync vim git-all patch mutt
[root@arfl-int]# dnf -y install autoconf libtool make gcc gcc-c++ m4
[root@arfl-int]# dnf -y install python2-devel python3-devel
[root@arfl-int]# dnf -y install postgresql
[root@arfl-int]# dnf -y install python3-pip
[root@arfl-int]# rpmconf -a
[root@arfl-int]# ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
[root@arfl-int]# setenforce 0
[root@arfl-int]# dnf -y install openssh-server
[root@arfl-int]# systemctl start sshd.service \
	&& systemctl status sshd.service \
	&& systemctl enable sshd.service
[root@arfl-int]# mkdir ~/.ssh && chmod 700 ~/.ssh
[root@arfl-int]# cat > ~/.ssh/authorized_keys << _EOF
ssh-rsa AAAA<Add-Your-own-SSH-public-key>BLAgU first.last@example.com
_EOF
[root@arfl-int]# chmod 600 ~/.ssh/authorized_keys
[root@arfl-int]# passwd -d root
[root@arfl-int]# rpm --import http://wiki.psychotic.ninja/RPM-GPG-KEY-psychotic
[root@arfl-int]# rpm -ivh http://packages.psychotic.ninja/7/base/x86_64/RPMS/keychain-2.8.0-3.el7.psychotic.noarch.rpm
[root@arfl-int]# cat > ~/.screenrc << _EOF
hardstatus alwayslastline "%{.kW}%-w%{.B}%n %t%{-}%{=b kw}%?%+w%? %=%c %d/%m/%Y" #B&W & date&time
startup_message off
defscrollback 1024
_EOF
[root@arfl-int]# exit
```

* Create and setup the `airflow` user:
```bash
[root@arfl-int]# adduser -m airflow
[root@arfl-int]# cp -a ~/.ssh ~/.bashrc ~/.bash_aliases ~airflow/ && \
  sudo chown -R airflow.airflow ~airflow/
```

* Add an entry on the SSH client (_e.g._, on a laptop):
```bash
user@laptop$ cat >> ~/.ssh/config << _EOF
# Airflow
Host proxy8
  HostName proxy8.example.com
  Port 7022
  ForwardAgent yes
Host tiafl
  HostName arfl-int.example.com
  ProxyCommand ssh -W %h:22 root@proxy8
_EOF
```

## PostgreSQL database server
* Reference: https://www.tecmint.com/install-postgressql-and-pgadmin-in-centos-8/

* Install PostgreSQL server and set it up:
```bash
[root@arfl-int]# dnf -y install postgresql-server
[root@arfl-int ~]# /usr/bin/postgresql-setup --initdb
 * Initializing database in '/var/lib/pgsql/data'
 * Initialized, logs are in /var/lib/pgsql/initdb_postgresql.log
[root@arfl-int]# systemctl start postgresql && systemctl enable postgresql && systemctl status postgresql
[root@arfl-int ~]# passwd postgres
Changing password for user postgres.
New password: 
Retype new password: 
passwd: all authentication tokens updated successfully.
[root@arfl-int]# su - postgres
[postgres@arfl-int]$ psql -c "ALTER USER postgres WITH PASSWORD 'securep';"
ALTER ROLE
[postgres@arfl-int ~]$ exit
logout
```

## Docker
* Reference: https://linuxconfig.org/how-to-install-docker-in-rhel-8

* On the Proxomox host, stop the Airflow container, loose LXC restrictions
  and restart the container:
```bash
root@proxmox:~$ pct stop 200
root@proxmox:~$ cat >> /etc/pve/lxc/200.conf << _EOF
lxc.apparmor.profile: unconfined
lxc.cgroup.devices.allow: a
lxc.cap.drop: 
_EOF
root@proxmox:~$ pct start 200 && pct enter 200
```

* Install Docker:
```bash
[root@arfl-int]# dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
[root@arfl-int]# dnf install -y https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.2.6-3.3.el7.x86_64.rpm
[root@arfl-int]# dnf install docker-ce --nobest -y
[root@arfl-int]# systemctl start docker && \
  systemctl enable docker && systemctl status docker
[root@arfl-int]# usermod -aG docker airflow
```

* Install `docker-compose`:
```bash
[root@arfl-int]# curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o docker-compose
[root@arfl-int]# chmod +x /usr/local/bin/docker-compose
[root@arfl-int]# docker-compose version
docker-compose version 1.25.5, build 8a1c60f6
docker-py version: 4.1.0
CPython version: 3.7.5
OpenSSL version: OpenSSL 1.1.0l  10 Sep 2019
```

* Per user installation for docker-compose:
```bash
[airflow@arfl-int]# pip3 install docker-compose --user
```

* Install Redis:
```bash
[root@arfl-int]# dnf -y install redis
[root@arfl-int]# systemctl start redis && systemctl enable redis && \
  systemctl status redis
[root@arfl-int]# python3 -m pip install -U celery[redis]
```

* Create the airflow user and database on PostgreSQL:
```bash
[root@arfl-int]# psql -h localhost -U postgres -d postgres -c "create database airflow;"
CREATE DATABASE
[root@arfl-int]# psql -h localhost -U postgres -d postgres \
  -c "create user airflow with encrypted password '<airflow-pass>'; \
  grant all privileges on database airflow to airflow;"
GRANT
[root@arfl-int]# echo "localhost:5432:airflow:airflow:<airflow-pass>" > ~/.pgpass && chmod 600 ~/.pgpass
[root@arfl-int ~]# psql -h localhost -U airflow -d airflow
airflow=> \q
```

* Install a few Python 3 packages as the `root` user:
```bash
[root@arfl-int]# python3 -m pip install -U pip
[root@arfl-int]# python3 -m pip install -U wheel
```

* Install a few Python 3 packages as the `airflow` user:
```bash
[root@arfl-int]# su - airflow
[airflow@arfl-int]# python3 -m pip install --user -U pip
[airflow@arfl-int]# python3 -m pip install --user -U wheel
[airflow@arfl-int]# python3 -m pip install -U pyjq
[airflow@arfl-int]# python3 -m pip install -U awsume
[airflow@arfl-int]# python3 -m pip install -U setuptools tox pytest twine sphinx
[airflow@arfl-int]# python3 -m pip install -U cx_oracle
[airflow@arfl-int]# python3 -m pip install -U elasticsearch
[airflow@arfl-int]# python3 -m pip install -U apache-airflow
[airflow@arfl-int]# python3 -m pip install -U typing_extensions pyamqp
[airflow@arfl-int]# python3 -m pip install -U apache-airflow[postgres,celery,redis,pyamqp,sqlalchemy,elasticsearch]
```

* Test the PostgreSQL connection:
```bash
airflow@airflow$ psql -h localhost -U airflow -d airflow -c "select 1 as test"
Password for user airflow: 
 test 
------
    1
(1 row)
```

* Setup Airflow (SQLite3) database a first time (which will create
  `~/airflow/airflow.cfg`):
```bash
airflow@arfl-int:~# airflow initdb
```

* Alter the Airflow setup for PostgreSQL:
```bash
airflow@arfl-int:~# sed -i -E 's|sqlite:////home/airflow/airflow/airflow.db|postgresql+psycopg2://airflow:<ask-admin>@localhost/airflow|g' ~/airflow/airflow.cfg
airflow@arfl-int:~# sed -i -E 's|rbac = False|rbac = True|g' ~/airflow/airflow.cfg
```

* Setup Airflow PostgreSQL database a first time:
```bash
airflow@arfl-int:~# airflow initdb
```

* Check the tables on the PostgreSQL Airflow database:
```bash
airflow@airflow$ psql -h localhost -U airflow -d airflow -c "\dt"
```
* List of relations
  Schema |             Name              | Type  |  Owner
 ------- | ----------------------------- | ----- | -------
  public | alembic_version               | table | airflow
  public | chart                         | table | airflow
  public | connection                    | table | airflow
  public | dag                           | table | airflow
  ...    | ... | ... |
  public | users                         | table | airflow
  public | variable                      | table | airflow
  public | xcom                          | table | airflow

* Create an admin user
  (the authentication is done by proxy8, so, not needed here):
```bash
airflow@airflow$ airflow create_user --username admin --role Admin \
  --email firstname.lastname@example.com \
  --firstname FirstName --lastname LastName \
  --password <ask-admin>
```

## SystemD services for Airflow
* Download the SystemD service specification files and install them
  in `/etc/systemd/`.
  Source: https://github.com/apache/airflow/tree/master/scripts/systemd
```bash
root@airflow$ cd /usr/lib/systemd/system
root@airflow$ wget https://github.com/apache/airflow/raw/master/scripts/systemd/airflow-webserver.service
root@airflow$ wget https://github.com/apache/airflow/raw/master/scripts/systemd/airflow-flower.service
root@airflow$ wget https://github.com/apache/airflow/raw/master/scripts/systemd/airflow-scheduler.service
root@airflow$ wget https://github.com/apache/airflow/raw/master/scripts/systemd/airflow-worker.service
root@airflow$ cd /etc/sysconfig
root@airflow$ wget https://github.com/apache/airflow/raw/master/scripts/systemd/airflow
```

* Directory for the Airflow PID files (may not work, as `/run` is dynamically
  created):
```bash
root@airflow$ mkdir -p /rub/airflow && sudo chown airflow.airflow /run/airflow
```

* Airflow SystemD environment file:
```bash
root@airflow$ sed -i -E 's|# AIRFLOW_HOME=|AIRFLOW_HOME=/home/airflow/airflow|g' /etc/sysconfig/airflow
```

* Airflow web server service:
```bash
root@airflow$ sed -i -E 's|ExecStart=/bin/airflow|ExecStart=/home/airflow/.local/bin/airflow|g' /usr/lib/systemd/system/airflow-webserver.service
root@airflow$ sed -i -E 's|mysql.service ||g' /usr/lib/systemd/system/airflow-webserver.service
# Add the following two lines just after Group=airflow
root@airflow$ cat >> /usr/lib/systemd/system/airflow-webserver.service << _EOF
RuntimeDirectory=airflow
RuntimeDirectoryMode=0775
_EOF
root@airflow$ systemctl enable airflow-webserver.service && sudo systemctl start airflow-webserver.service && systemctl status airflow-webserver.service
```

* Airflow scheduler service:
```bash
airflow@airflow$ sudo sed -i -E 's|ExecStart=/bin/airflow|ExecStart=/home/airflow/.local/bin/airflow|g' /etc/systemd/system/airflow-scheduler.service
airflow@airflow$ sudo sed -i -E 's|mysql.service ||g' /etc/systemd/system/airflow-scheduler.service
airflow@airflow$ sudo systemctl enable airflow-scheduler.service && sudo systemctl start airflow-scheduler.service && systemctl status airflow-scheduler.service
```

* Airflow worker service:
```bash
airflow@airflow$ sudo sed -i -E 's|ExecStart=/bin/airflow celery worker|ExecStart=/home/airflow/.local/bin/airflow worker|g' /etc/systemd/system/airflow-worker.service
airflow@airflow$ sed -i -E 's|mysql.service ||g' /etc/systemd/system/airflow-worker.service
airflow@airflow$ sudo systemctl enable airflow-worker.service && sudo systemctl start airflow-worker.service && systemctl status airflow-worker.service
```

* Airflow flower service:
```bash
airflow@airflow$ sudo sed -i -E 's|ExecStart=/bin/airflow celery flower|ExecStart=/home/airflow/.local/bin/airflow flower|g' /etc/systemd/system/airflow-flower.service
airflow@airflow$ sudo sed -i -E 's|mysql.service ||g' /etc/systemd/system/airflow-flower.service
airflow@airflow$ sudo systemctl enable airflow-flower.service && sudo systemctl start airflow-flower.service && systemctl status airflow-flower.service
```

* Setup an alias to check the status of Airflow services:
```bash
[root@arfl-int]# alias getstatusairflow='for svc in airflow-flower airflow-worker airflow-webserver airflow-scheduler; do systemctl status $svc; done'
```


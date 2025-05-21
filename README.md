# Apache CloudStack Installataion on Ubuntu

## Contributors

* Fayza Nirwasita (2106635700)
* Adrien Ardra Ramadhan (2106731485)
* Sahrif Fatih Asad Masyhur (2206063014)
* Lavly Rantissa Zunnuraina Rusdi (2206830624)

---

## Table of Contents

### 1. [Introduction](#introduction)
### 2. [Initial Environment Configuration](#initial-environment-configuration)
### 3. [Network Configuration](#network-configuration)
### 4. [SSH Configuration](#ssh-configuration)
### 5. [Cloudstack Installation](#cloudstack-installation)
### 6. [Configure CloudStack Host with KVM Hypervisor](#configure-cloudstack-host-with-kvm-hypervisor)
### 7. [Launch CloudStack Management Server](#launch-cloudstack-management-server)

---

## Introduction
Apache CloudStack is an open-source Infrastructure-as-a-Service (IaaS) cloud computing platform designed to deploy and manage large networks of virtual machines. It abstracts physical infrastructure resources such as compute, storage, and networking, and enables cloud administrators to provision and orchestrate them via APIs or a web-based interface.

CloudStack supports multiple hypervisors including KVM, VMware, and XenServer, and enables configuration of virtualized datacenters using logical constructs such as Zones, Pods, Clusters, and Hosts. It includes features like virtual machine templates, elastic IP management, system-wide resource monitoring, and multi-tenant user management. In this guide, we use KVM as the hypervisor and NFS as shared storage for primary and secondary storage.

This single-node deployment includes the CloudStack Management Server, KVM compute node, and both primary and secondary storage services running on the same host.

---

## Initial Environment Configuration

### IP Addressing Scheme

```
Network Address : 192.168.68.0/24
Host IP address : 192.168.68.106/24
Gateway : 192.168.68.1
Management IP :
System IP :
Public IP :
```

### Installing Tools

```
apt update -y
apt upgrade -y
apt install htop lynx duf bridge-utils -y
```

## Network Configuration

Netplan was used to define a bridge interface cloudbr0 which connects the physical NIC to virtual interfaces used by VMs.
```bash
cd /etc/netplan
sudo nano /etc/netplan/01-netcfg.yaml
```

Edit file to change the IP address:
```
# This is the network config written by 'subiquity'
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0:
      dhcp4: false
      dhcp6: false
      optional: true
  bridges:
    cloudbr0:
      addresses: [192.168.68.106/24]
      routes:
        - to: default
          via: 192.168.68.1
      nameservers:
        addresses: [1.1.1.1,8.8.8.8]
      interfaces: [enp1s0]
      dhcp4: false
      dhcp6: false
      parameters:
        stp: false
        forward-delay: 0
```

Apply the changes:

```bash
netplan generate
netplan apply
```

## SSH Configuration

### Allow Root Access via SSH

```
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config
#restart ssh service
service ssh restart
#or
systemctl restart sshd.service
```

### Verify SSH Configuration
```
nano /etc/ssh/sshd_config
```

### Set Timezone
```
timedatectl set-timezone Asia/Jakarta
```

## Cloudstack Installation

### Importing cloudstack repositories key

```
sudo -i
mkdir -p /etc/apt/keyrings 
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null
echo deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.18 / > /etc/apt/sources.list.d/cloudstack.list
```

### Verify repository configruation

```
nano /etc/apt/sources.list.d/cloudstack.list
```

### Installing Cloudstack Management and Mysql Server

```
apt-get update -y
apt-get install cloudstack-management mysql-server
```

### Configure mysql

Edit MySQL Configuration File:
```
sudo -e /etc/mysql/mysql.conf.d/mysqld.cnf
```

```
server-id = 1
sql-mode="STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION,ERROR_FOR_DIVISION_BY_ZERO,NO_ZERO_DATE,NO_ZERO_IN_DATE,NO_ENGINE_SUBSTITUTION"
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
max_connections=1000
log-bin=mysql-bin
binlog-format = 'ROW'
```

```
systemctl restart mysql
systemctl status mysql
```

### Initialize CloudStack Schema and User

```
cloudstack-setup-databases cloud:cloud@localhost --deploy-as=root:Pa$$w0rd -i 192.168.104.24
```

### Configure Primary and Secondary Storage

```
apt-get install nfs-kernel-server quota
echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a
```

```
sed -i -e 's/^RPCMOUNTDOPTS="--manage-gids"$/RPCMOUNTDOPTS="-p 892 --manage-gids"/g' /etc/default/nfs-kernel-server
sed -i -e 's/^STATDOPTS=$/STATDOPTS="--port 662 --outgoing-port 2020"/g' /etc/default/nfs-common
echo "NEED_STATD=yes" >> /etc/default/nfs-common
sed -i -e 's/^RPCRQUOTADOPTS=$/RPCRQUOTADOPTS="-p 875"/g' /etc/default/quota
service nfs-kernel-server restart
```

## Configure Cloudstack Host with KVM Hypervisor

```bash
apt install qemu-kvm cloudstack-agent -y
```

### Configure KVM Virtualization Management

```
sed -i.bak 's/^\(LIBVIRTD_ARGS=\).*/\1"--listen"/' /etc/default/libvirtd
```

```
echo 'listen_tls=0' >> /etc/libvirt/libvirtd.conf
echo 'listen_tcp=1' >> /etc/libvirt/libvirtd.conf
echo 'tcp_port = "16509"' >> /etc/libvirt/libvirtd.conf
echo 'mdns_adv = 0' >> /etc/libvirt/libvirtd.conf
echo 'auth_tcp = "none"' >> /etc/libvirt/libvirtd.conf
```

```
systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd
```

```
echo "net.bridge.bridge-nf-call-arptables = 0" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 0" >> /etc/sysctl.conf
sysctl -p
```

### Generate Unique Host ID

```
apt-get install uuid -y
UUID=$(uuid)
echo host_uuid = \"$UUID\" >> /etc/libvirt/libvirtd.conf
systemctl restart libvirtd
```

### Configure IPtables Firewall

```
NETWORK=192.168.101.0/24
sudo -e /etc/iptables/rules.v4
iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 111 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 111 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 2049 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 32803 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 32769 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 892 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 875 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 662 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8250 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8080 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8443 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 9090 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 16514 -j ACCEPT
#iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 3128 -j ACCEPT
#iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 3128 -j ACCEPT
apt-get install iptables-persistent
```

### Disable apparmour on libvirtd

```
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

### Importing cloudstack repositories key

```
sudo -i
mkdir -p /etc/apt/keyrings
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null
echo deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.20 / > /etc/apt/sources.list.d/cloudstack.list

```

### Configuration to Support Docker Services
```
#on certain hosts where you may be running docker and other services, 
#you may need to add the following in /etc/sysctl.conf
#and then run sysctl -p: --> /etc/sysctl.conf

echo "net.bridge.bridge-nf-call-arptables = 0" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 0" >> /etc/sysctl.conf
sysctl -p
```


### Launch Cloudstack Management Server

```
cloudstack-setup-management
systemctl status cloudstack-management
tail -f /var/log/cloudstack/management/management-server.log #if you want to troubleshoot 
#wait until all services (components) running successfully
```

### Open Cloudstack Dashboard

```
http://192.168.68.106:8080
```

### You should see the cloudstack dashboard

![image](https://github.com/user-attachments/assets/1fc841f5-dcb3-4e2a-b8b0-1881cf7372fc)
---
Login Credentials (Default)
Username: admin
Password: password
*Recommended to change the default password for security
---

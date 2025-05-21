# Apache CloudStack Installataion on Ubuntu

## Contributors

* Fayza Nirwasita (2106635700)
* Adrien Ardra Ramadhan (2106731485)
* Sharif Fatih Asad Masyhur (2206063014)
* Lavly Rantissa Zunnuraina Rusdi (2206830624)

---

## Table of Contents

- [Introduction](#introduction)
- [Initial Environment Configuration](#initial-environment-configuration)
  - [IP Addressing Scheme](#ip-addressing-scheme)
  - [Installing Tools](#installing-tools)
- [Network Configuration](#network-configuration)
- [SSH Configuration](#ssh-configuration)
  - [Allow Root Access via SSH](#allow-root-access-via-ssh)
  - [Verify SSH Configuration](#verify-ssh-configuration)
  - [Set Timezone](#set-timezone)
- [Cloudstack Installation](#cloudstack-installation)
  - [Importing CloudStack Repositories Key](#importing-cloudstack-repositories-key)
  - [Verify Repository Configuration](#verify-repository-configuration)
  - [Installing CloudStack Management and MySQL Server](#installing-cloudstack-management-and-mysql-server)
  - [Configure MySQL](#configure-mysql)
  - [Initialize CloudStack Schema and User](#initialize-cloudstack-schema-and-user)
  - [Configure Primary and Secondary Storage](#configure-primary-and-secondary-storage)
- [Configure CloudStack Host with KVM Hypervisor](#configure-cloudstack-host-with-kvm-hypervisor)
  - [Configure KVM Virtualization Management](#configure-kvm-virtualization-management)
  - [Generate Unique Host ID](#generate-unique-host-id)
  - [Configure IPtables Firewall](#configure-iptables-firewall)
  - [Disable Apparmour on Libvirtd](#disable-apparmour-on-libvirtd)
- [Launch CloudStack Management Server](#launch-cloudstack-management-server)
  - [Open CloudStack Dashboard](#open-cloudstack-dashboard)
- [Add a Zone, Pod, and Cluster](#add-a-zone-pod-and-cluster)
  - [Zone](#zone)
  - [Pod](#pod)
  - [Cluster](#cluster)
- [Add Primary and Secondary Storage](#add-primary-and-secondary-storage)
  - [Primary Storage](#primary-storage)
  - [Secondary Storage](#secondary-storage)
- [Add Hypervisor Hosts](#add-hypervisor-hosts)
  - [Supported Hypervisors](#supported-hypervisors)
  - [Prerequisites](#prerequisites)
  - [Steps to Add a Host](#steps-to-add-a-host)
- [Configure Network Access for the VM](#configure-network-access-for-the-vm)
  - [Upload Ubuntu 20.04 Template](#upload-ubuntu-2004-template)
  - [Launch the VM](#launch-the-vm)
  - [Assign Public IP](#assign-public-ip)
  - [Enable Internet Access](#enable-internet-access)
- [Enable SSH Access](#enable-ssh-access)
- [Enable HTTP/HTTPS Access](#enable-httphttps-access)

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
Management IP address (Host) : 192.168.68.106/24
Gateway : 192.168.68.1
System IP : 192.168.68.120
Public IP : 192.168.68.120 - 192.168.68.140
```
> **Note**: The IP address structure here aligns with CloudStack's requirement for having separate network ranges or IP pools for management traffic, system VMs, and public access. Although we use a simplified setup, in a production-grade environment, network isolation using VLANs or SDN controllers is recommended.

### Installing Tools

```
apt update -y
apt upgrade -y
apt install htop lynx duf bridge-utils -y
```

> These tools help monitor system utilization during and after deployment. `htop` displays real-time CPU/memory load, `lynx` is a CLI browser useful for checking service availability without GUI, and `duf` provides an overview of disk usage.

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

> CloudStack uses bridge networking to connect VMs directly to the external physical network. This allows seamless Layer 2 connectivity and eliminates the need for NAT translation on the hypervisor.

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

> Enabling SSH access for root allows remote administration of the host, especially useful when physical access is unavailable. However, in production, it is highly recommended to disable root login or use key-based authentication with unprivileged users.

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

> ShapeBlue provides upstream builds of Apache CloudStack. Adding their repository enables installation of the latest stable versions aligned with the official release.

### Verify Repository Configuration

```
nano /etc/apt/sources.list.d/cloudstack.list
```

### Installing Cloudstack Management and Mysql Server

```
apt-get update -y
apt-get install cloudstack-management mysql-server
```

> The CloudStack management server acts as the brain of the cloud, handling API requests, scheduling VM placement, and interacting with the database backend. MySQL is used to store all metadata, configurations, and runtime state.

### Configure MySQL

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
cloudstack-setup-databases cloud:cloud@localhost --deploy-as=root:Pa$$w0rd -i 192.168.68.106
```

> This command initializes the CloudStack schema, creates required tables, and provisions a privileged database user that will be used by the management server.

### Configure Primary and Secondary Storage

```
apt-get install nfs-kernel-server quota
echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a
```

> CloudStack distinguishes between primary storage (used by VMs for root and data volumes) and secondary storage (used for templates, ISO images, and VM snapshots). We use NFS because it is simple to set up and supported out-of-the-box.

```
sed -i -e 's/^RPCMOUNTDOPTS="--manage-gids"$/RPCMOUNTDOPTS="-p 892 --manage-gids"/g' /etc/default/nfs-kernel-server
sed -i -e 's/^STATDOPTS=$/STATDOPTS="--port 662 --outgoing-port 2020"/g' /etc/default/nfs-common
echo "NEED_STATD=yes" >> /etc/default/nfs-common
sed -i -e 's/^RPCRQUOTADOPTS=$/RPCRQUOTADOPTS="-p 875"/g' /etc/default/quota
service nfs-kernel-server restart
```

> Port pinning here ensures compatibility with firewalls and consistent behavior across system reboots.

---

## Configure Cloudstack Host with KVM Hypervisor

KVM (Kernel-based Virtual Machine) is a native Linux hypervisor. Libvirt is the management layer, and `cloudstack-agent` serves as the communication bridge between CloudStack and libvirt.

```bash
apt install qemu-kvm cloudstack-agent -y
```

### Configure KVM Virtualization Management

By default, libvirt listens only on UNIX sockets for local clients. We reconfigure it to listen over TCP so that CloudStack's management server can communicate with KVM remotely, even though in our case they reside on the same machine.

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

Restart Libvirt:
```
systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd
```

Configuration to Support Docker Services:
```
echo "net.bridge.bridge-nf-call-arptables = 0" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 0" >> /etc/sysctl.conf
sysctl -p
```

> These sysctl flags prevent the Linux bridge from filtering packets with iptables, which can otherwise interfere with VM communication or container networking.

### Generate Unique Host ID

```
apt-get install uuid -y
UUID=$(uuid)
echo host_uuid = \"$UUID\" >> /etc/libvirt/libvirtd.conf
systemctl restart libvirtd
```

> `host_uuid` uniquely identifies each host to the CloudStack management system. This is required for accurate tracking and VM placement.

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
```

> CloudStack and its underlying infrastructure components require multiple open ports for communication between services like libvirt, NFS, and the management server. Below is a breakdown of each rule:
>
> * **TCP/UDP 111**: Used by the portmapper (`rpcbind`) service to coordinate RPC-based services such as NFS.
> * **TCP 2049**: NFS service port where clients read/write to exported directories.
> * **TCP 32803 & UDP 32769**: Associated with the `nfsd` and mountd daemons for mounting and file handle access.
> * **TCP 892, 875, 662**: These are fixed ports for `rpc.mountd`, `rpc.rquotad`, and `rpc.statd` respectively to support quota and locking over NFS.
> * **TCP 8250**: Typically used by CloudStack agent services for internal communication.
> * **TCP 8080**: CloudStack Management Server UI and API.
> * **TCP 8443 & 9090**: Used for secured API access and internal monitoring.
> * **TCP 16514**: Libvirt remote management port used by CloudStack to control VM lifecycle operations on KVM hosts.
>   CloudStack uses a wide range of ports for NFS, libvirt, web UI, and various internal services. Explicitly allowing these ports ensures no communication is blocked, especially in production with a strict firewall policy.


Make the rules persistent:
```
apt-get install iptables-persistent
```

### Disable apparmour on libvirtd

```
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

> AppArmor is a mandatory access control system which may block libvirt and KVM from functioning correctly in dynamic cloud environments. Disabling it for relevant services avoids potential runtime errors.

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

![image](https://github.com/user-attachments/assets/25820e74-d131-4246-bf65-ad730e891838)

```
Login Credentials (Default)
- Username: admin
- Password: password
*Recommended to change the default password for security
```

## Add a Zone, Pod, and Cluster

Once logged in to the CloudStack Dashboard:

1. Go to `Infrastructure` > `Zones` > `Add Zone`
2. Choose `Advanced` or `Basic Networking`, depends on network setup
3. Follow the guided wizard to complete the setup:

---
> **Note**: Make sure to complete it step by step from Zone, Pod, to Cluster. Make sure each setup is configured correctly. Recommended to plan out the setup such as IPs, subnets, VLANs, before go through these steps.

### Zone

A **Zone** represents a single physical data center. It includes:
- One or more **Pods**
- Shared **Secondary Storage** (for templates, ISOs, and snapshots)

Add details:
- Zone name (e.g. KELOMPOK2_ZONE)
- Public and guest IP address ranges, with subnets
- DNS servers (internal and external)
- Network type (Basic or Advanced)

![image](https://github.com/user-attachments/assets/4bc549c8-c94c-4177-a571-f90819c2eafe)
![image](https://github.com/user-attachments/assets/42b89800-5edc-40a4-b843-c1dc01092fe3)

---

### Pod

A **Pod** resides within a zone and typically represents a single rack or switch. Each Pod includes:
- One or more **Clusters**
- Its own subnet (Layer 2 broadcast domain)
- Access to **Primary Storage**

Add details:
- Pod name
- IP address range
- Netmask
- Gateway

![image](https://github.com/user-attachments/assets/f2a88f84-62cb-4a7a-893d-3ea98bae7655)
![image](https://github.com/user-attachments/assets/1e98fb3f-bea1-4990-b1a1-52831b7ec083)

---

### Cluster

A **Cluster** is a group of hosts (physical servers) that use the same hypervisor (e.g., KVM, XenServer, or VMware), where your virtual machines (VMs) will be deployed. Each cluster shares:
- Primary Storage
- Network configurations

Add details:
- Cluster name (e.g. KELOMPOK2_CLUSTER)
- Hypervisor type (e.g., KVM, XenServer, or VMware)
- Add physical hosts (including credentials and IPs)

![image](https://github.com/user-attachments/assets/c53f6aa1-a4b6-4587-8b95-32ee0d01c44e)
![image](https://github.com/user-attachments/assets/15a7c726-2440-4145-9e41-9222a6cc2c94)

---

## Add Primary and Secondary Storage

### Primary Storage

- To stores VM root volumes, data disks, and snapshots.
- Configured per **Cluster**.
- **Types**:
  - NFS (Network File System)
  - iSCSI
  - Shared local storage
- **Steps**:
  1. Go to `Infrastructure` > `Primary Storage` > `Add Primary Storage`
  2. Select the **zone**, **pod**, and **cluster** that configured previously
  3. Enter a name and the storage path (e.g., `nfs://192.168.1.100/export/PRIMARY`)
  4. Choose the protocol (NFS, iSCSI, etc.)
  5. Save and wait for CloudStack to mount and verify the storage

![image](https://github.com/user-attachments/assets/c0a40966-b623-4a6b-9daf-d8dc5ad6a488)
![image](https://github.com/user-attachments/assets/8f2995df-5315-4cb2-8590-82a03484e3de)

### Secondary Storage

- To Stores templates, ISO images, and system VM snapshots.
- Configured per **Zone**.
- **Steps**:
  1. Go to `Infrastructure` > `Secondary Storage` > `Add Secondary Storage`
  2. Enter the NFS path or object store details (e.g., `nfs://192.168.1.100/export/SECONDARY`)
  3. Assign to the desired zone
  4. Save and verify connectivity

![image](https://github.com/user-attachments/assets/e6123cd7-af1c-402b-8745-d92fa3af46d7)
![image](https://github.com/user-attachments/assets/f596858d-1f32-4aa8-b06e-2e0aa3f96f5a)

---

## Add Hypervisor Hosts

Hypervisors (physical servers) are where your virtual machines will run. Hosts must be added to the **Cluster** that already configured previously and running.

### Supported Hypervisors

- KVM (Linux)
- XenServer / XCP-ng
- VMware vSphere/ESXi

### Prerequisites

- Hypervisors must be installed and properly networked
- Ensure they can communicate with:
  - Management server
  - Storage servers
- SSH access and appropriate credentials must be available

### Add a Host

1. Navigate to `Infrastructure` > `Clusters` > `View Hosts` > `Add Host`
2. Select:
   - **Hypervisor Type** (e.g., KVM)
   - **Host IP Address**
   - **Username** and **Password** (usually root credentials)
   - **Cluster** and **Pod**
3. Click **OK**
4. Wait for CloudStack to verify the host and connect it to the cluster

---

## Configure Network Access for the VM

### Upload Ubuntu 20.04 Template

1. Navigate to `Templates` > `Register Template`
2. Use the following settings:
   - **Name**: `Ubuntu-20.04`
   - **URL**:  
     ```
     https://releases.ubuntu.com/focal/ubuntu-20.04.6-live-server-amd64.iso
     ```
   - **OS Type**: Ubuntu (64-bit)
   - **Hypervisor**: KVM
   - **Format**: QCOW2
   - **Zone**: `KELOMPOK2-ZONE`
   - **Public**: ✅
3. Click **OK** and wait until the status shows `Ready`

### Launch the VM

1. Go to `Instances` > `Add Instance`
2. Follow the wizard:
   - **Zone**: `KELOMPOK2-ZONE`
   - **Template**: `Ubuntu-20.04`
   - **Compute Offering**: Medium Instance (1 vCPU, 1GB RAM)
   - **Disk Offering**: Use default
   - **Network**: `KELOMPOK2-NETWORK` (Isolated with Source NAT)
   - **Name**: e.g., `ubuntu-web01`
3. Click **Launch VM**

**Example:**
- **Private IP**: `10.1.1.226`
- **Guest Network**: `KELOMPOK2-NETWORK`
- **Public IP**: (assigned in next step)

### Assign Public IP

1. Go to `Network` > `KELOMPOK2-NETWORK`
2. Under **Public IP Addresses**, click `Acquire New IP`
3. Example public IP: `192.168.68.126`
4. Click the IP, then choose `Enable Static NAT` 

### Enable Internet Access

1. Go to `Network` > `KELOMPOK2-NETWORK` > `Egress Rules`
2. Add the following rule:
   - **Protocol**: All  
   - **CIDR**: `0.0.0.0/0`  
   - **Action**: Allow  
**Verify from inside the VM:**
```bash
ping 8.8.8.8
```
3. Optionally, you can try step 4. if internet access still doesn't work on running instances.
4. Go to `Service offerings` > `Network Offerings`, then try to assign the following network offerings into your created zones.
![image](https://github.com/user-attachments/assets/52b7e433-8837-4cd7-9210-437dff03abc8)
![image](https://github.com/user-attachments/assets/ff2a6cd7-3c00-48c8-bf18-3b7eae362089)

---

## Enable SSH Access

1. **Go to the Firewall tab** under IP `192.168.68.126` (in `Network` section under `Public IP addresses`).
2. **Add a Port Forwarding Rule**:
   - **Protocol**: TCP  
   - **Public Port**: 22  
   - **Private Port**: 22  
   - **VM**: `ubuntu-web01` (`10.1.1.226`)
  
![image](https://github.com/user-attachments/assets/c29df4c5-b6c1-4c14-ae6b-475c59be10ce)
![image](https://github.com/user-attachments/assets/1f8a2017-a1cb-4861-845e-8983746437a2)

3. **SSH into the VM**:
   ```bash
   ssh ubuntu@192.168.68.126
    ```

---

## Enable HTTP/HTTPS Access

1. SSH into the VM
Make sure you have SSH into the VM first. (Refer to the previous SSH step.)

2. Install Apache
Update package list dan install Apache:

```bash
sudo apt update
sudo apt install apache2 -y
sudo systemctl enable apache2
sudo systemctl start apache2
sudo systemctl status apache2
```

3. Add Firewall Rules for the Public IP

#### HTTP (Port 80)
- **Protocol:** TCP  
- **Public Port:** 80  
- **Private Port:** 80  

#### HTTPS (Port 443)
- **Protocol:** TCP  
- **Public Port:** 443  
- **Private Port:** 443
### Open in browser

```
http://192.168.68.126
```

---

### References:
#### [1] L. Reynolds, “Ubuntu Server: Connect to Wi-Fi from command line,” LinuxConfig, Apr. 06, 2020. Available: https://linuxconfig.org/ubuntu-20-04-connect-to-wifi-from-command-line (accessed May 02, 2025).
#### [2] BCruz, szovatilevente, “ACPI error after installing Ubuntu 22.04,” Linux.org, Jun. 27, 2022. Available: https://www.linux.org/threads/acpi-error-after-installing-ubuntu-22-04.40993/?__cf_chl_tk=B0f_GmnIzHN8XBzOtLSWePXf3j1vksVK.ekiMOCARt8-1746608196-1.0.1.1-R84f2whyGFqS.2J3VdjeMS1VA8F8jbK_dCJKJzYkS1I (accessed May 02, 2025).
#### [3] AhmadRifqi86, “cloudstack-install-and-configure/cloudstack-install at main · AhmadRifqi86/cloudstack-install-and-configure,” GitHub, 2024. Available: https://github.com/AhmadRifqi86/cloudstack-install-and-configure/tree/main/cloudstack-install (accessed May 03, 2025).
#### [4] maradens, “apachecloudstack/README.md at main · maradens/apachecloudstack,” GitHub, 2023. Available: https://github.com/maradens/apachecloudstack/blob/main/README.md (accessed May 03, 2025).
#### [5] Yan Maraden, “Continue with installation (Apache Cloudstack 4.18) - Shorter Version,” YouTube, Apr. 30, 2024. Available: https://www.youtube.com/watch?v=8ZZeU3vbbl4 (accessed May 10, 2025).
#### [6] Yan Maraden, “Register ISO and Add Instance (Apache Cloustack 4.18),” YouTube, Apr. 29, 2024. Available: https://www.youtube.com/watch?v=y_S3x3tJvCg (accessed May 10, 2025).
#### [7] ShapeBlue, “Public IP Address | How To CloudStack,” YouTube, May 24, 2021. Available: https://www.youtube.com/watch?v=oBJjqYg1Mpk (accessed May 13, 2025).
#### [8] ShapeBlue, “Apache CloudStack Installation and Configuration Guide,” YouTube, Apr. 15, 2024. Available: https://www.youtube.com/watch?v=DlJg3LYvIIs (accessed May 13, 2025).

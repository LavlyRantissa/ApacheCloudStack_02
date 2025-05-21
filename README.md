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


# RKE-Setup

This repository is a collected of files documenting my install of different aspects of establishing a Rancher-managed set of cluster.

### Services VM
This VM serves as the DHCP server, DNS server, and loadbalancer for the highly-available cluster hosting the Rancher server. The steps for this setup are detailed in **ServicesInstall.md**

### Rancher Cluster Setup
While a single node Rancher server install in a single Docker command, the HA deployment is bit more involved. The steps for this setup are detailed in **RancherClusterSetup.md**

### Rancher Provisioned RKE Cluster
**RKEClusterSetup.md** details the sets to establishing a set of nodes for Rancher to provision into a cluster. As the Rancher server itself is hosted on a similar setup, this document shares a lot of the details with the previous.

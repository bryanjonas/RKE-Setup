# RKE Cluster Setup

The purpose of this document is to set up the machines for Rancher to provision into a cluster. Much like establishing the cluster to host Rancher itself, this will consist of building a node template with several things installed and then cloning that template several times in order to make all the nodes. For an HA cluster, I will create nine nodes: three control planes, three workers and three etcd nodes. 

While you can get away with co-locating some of these node roles together, taking this approach will expose you to a fairly complete setup in terms of communication between nodes in your cluster. *If you can dodge a wrench, you can dodge a ball.*

## Create Node Template

Once again I will use Ubuntu 16.04 for my nodes because of it's confirmed support with RKE. Since my cluster will be used for little more than testing, I will use the following guide for hardware provisioning:

| Node Type | CPUs | RAM | SSD |
|-----------|------|-----|-----|
| Control Plane | 4 | 8 GB | 32 GB |
| ETCD | 4 | 4 GB | 32 GB |
| Worker | 4 | 8 GB | 32 GB |

### Install Docker
```{bash}
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
   
sudo apt-get update

#RKE doesn't support the latest version as of writing
sudo apt-get install docker-ce=5:19.03.14~3-0~ubuntu-xenial \
    docker-ce-cli=5:19.03.14~3-0~ubuntu-xenial \
    containerd.io

#Need to add user to docker group
sudo /usr/sbin/usermod -aG docker route
```

### Clone Nodes and Copy SSH Keys

Once I've cloned this node into three and put the new MAC addresses in the DHCP configuration on the Services VM. I will generate a SSH key on **rancher-1** and
copy that over to the other two nodes to facilitate communication.

```{bash}
ssh-keygen

ssh-copy-id route@iterate-through-node-ips
```

## Changes to Services VM

To support this new cluster, you will need to make several changes to the configuration files on the Services VM.

### DHCP Reservations

Go into **/etc/dhcp/dhcpd.conf** and add records for each of the new machines.

### DNS Records

In addition to add *A* records for these new machines to **/etc/bind/db.rancher.lan**, we will want to add records to support the load balancing the external communications with the Kubernetes API. Internal load balancing is taken care of with specially deployed pods in the cluster. The first step in this process is to add a DNS record for the cluster:

```{bash}
rke1.rancher.lan     IN  A   10.10.1.1
```

### HAProxy Services

After you add this new DNS record, you will have to make some modifications in **/etc/haproxy/haproxy.cfg** to support the load balancing. You need to make modifications to the preexisting services listening on port 443 in order to route based on the domain coming into the proxy.

```{bash}
frontend https_fe #Renamed from rancher_https_fe
    bind *:443
    #Define hosts
    acl host host_rancher hdr(host) -i rancher.rancher.lan
    acl host host_rke1 hdr(host) -i rke1.rancher.lan
    
    #Define backends
    use_backend rancher_https_be if host_rancher
    use_backend rke1_https_be if host_rke1
    
    default_backend rancher_https_be
    mode tcp
    option tcplog
    
backend rancher_https_be
    balance source
    mode tcp
    server rancher-1 10.10.1.10:443 check
    server rancher-2 10.10.1.11:443 check
    server rancher-3 10.10.1.12:442 check

backend rke1_https_be
    balance source
    mode tcp
    server rke1-1 10.10.1.20:6443 check
    server rke1-2 10.10.1.21:6443 check
    server rke1-3 10.10.1.22:6443 check
```

This should be all you need to get your cluster going!

# RKE-Setup
Rancher and RKE Cluster Setup

## Node Setup:
As I only have a single server at home to play with, I stood up several VMs in Proxmox to play with Rancher and RKE. To each VM I gave 4 cores, 8 GB of RAM, and 100 GB of storage. The RKE website doesn't give a minimum hardware configuration so I assumed I would be safe with this setup. THe documentation does mention that most of their testing has been on nodes with Ubuntu 16.04 so I installed the server version of that on each node. 

The actual mechanics of setting up the nodes involved me installing the pre-reqs on a single node and then cloning that four times so save myself some time. The required software includes: Docker, RKE, and kubectl.

### Docker:
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
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

### RKE:
Visit the link below to get the latest release of the RKE client: 

https://github.com/rancher/rke/releases/tag/v1.0.14

### Kubectl:
```{bash}
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x ./kubectl

sudo mv ./kubectl /usr/local/bin/kubectl
```
Once these nodes were cloned the hostnames need to be changed (otherwise they'll all be the same after the cloning) and DHCP reservations need to be assigned. 

**Future Addition:** It's probably best to set up a network segment for these cluster in a similar manner as the OKD4 cluster I set up. 

### SSH Key Exchanges:
The RKE cluster communicates via SSH so we need to create a SSH key on the origination node (my term for whoever is going to host the Rancher control UI). So on that node we need to run these commands:

```{bash}
ssh-keygen
```

You can leave all the defaults on this key generation process. Following this, you need to install this key as an acceptable login key for whatever user is going to be your non-root docker user. If your key is named the default (~/.ssh/id_rsa) you can use the command below.

```{bash}
ssh-copy-id username@other-node-ip
```

## Starting the Rancher Control Containers:

It's very easy to set up the Rancher Control UI by running this command on origination node. 

```{bash}
sudo docker run --privileged -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher
```

Once these containers are up you can visit the interface at https://orignating-node-ip and follow the instructions there.

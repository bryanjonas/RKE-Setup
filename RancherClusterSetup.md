# Rancher Cluster Setup

In order to set up a highly-available install of Rancher to manager our RKE cluster, I'm going to set up one node and clone it into three seperate nodes.
Besides the usual OS setup, the only thing that needs to be installed on the node before cloning is Docker.

## Install Docker
```{bash}
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -

sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/debian \
   $(lsb_release -cs) \
   stable"
   
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

Once I've cloned this node into three and put the new MAC addresses in the DHCP configuration on the Services VM. I will generate a SSH key on **rancher-1** and
copy that over to the other two nodes to facilitate communication.

```{bash}
ssh-keygen

ssh-copy-id route@10.10.1.11
ssh-copy-id route@10.10.1.12
```

## Start Rancher

Rancher is very easy to start using the command provided on the Rancher website:

```{bash}
sudo docker run --privileged -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher
```

Now you can navigate to **https://10.10.1.10** on your service node (which is why it is the only VM I put a GUI on) and start to get your cluster set up. 

**Reminder:** Got back an add the HTTP and HTTPS service to HAProxy on the Services VM and you can instead visit **https://10.10.1.1** or even the 
IP for interface on your home network.

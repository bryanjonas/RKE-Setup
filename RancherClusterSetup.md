# Rancher Cluster Setup

## Create Node Template

In order to set up a highly-available install of Rancher, you have to establish a RKE cluster (the hard way). This is actually pretty straightforward if you make a node template (or whatever other automation you want to use) and clone it three times.

### Install Docker
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
### Install RKE
Visit the link below to get the latest release of the RKE client: 

https://github.com/rancher/rke/releases/tag/v1.0.14

```{bash}
wget https://github.com/rancher/rke/releases/download/v1.0.14/rke_linux-amd64

chmod +x rke_linux-amd64

mv rke_linux-amd64 /usr/local/bin/rke
```

### Install Kubectl:
```{bash}
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x ./kubectl

sudo mv ./kubectl /usr/local/bin/kubectl
```

### Clone Nodes and Copy SSH Keys

Once I've cloned this node into three and put the new MAC addresses in the DHCP configuration on the Services VM. I will generate a SSH key on **rancher-1** and
copy that over to the other two nodes to facilitate communication.

```{bash}
ssh-keygen

ssh-copy-id route@10.10.1.11
ssh-copy-id route@10.10.1.12
```

### Create RKE Cluster

It's possible to compose a config file by hand for RKE, the the **rke** utility can help you if you run this command on the **rancher-1** node:

```{bash}
rke config
```

Once that's all filled out, you can raise the cluster with:

```{bash}
rke up

#Add the following line to the ~/.bashrc
export KUBECONFIG=~/kube_config_cluster.yml
```

### Installing Helm

The easiest way to install things on RKE cluster is using a Helm chart (if one exists). To install the Rancher server on the RKE cluster, you must first install Helm. **Note:** You'll see mentions of *Tiller* if you look up Helm-stuff. If you are using Helm 3.0+, Tiller is no longer required.

```{bash}
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3

chmod +x get_helm.sh

./get_helm.sh
```

### Install Rancher
https://rancher.com/docs/rancher/v2.x/en/installation/install-rancher-on-k8s/

```{bash}
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest

kubectl create namespace cattle-system
```

Before we can install Rancher, we have to install cert-manager. I'm going to probably use Rancher's default certificates for now so I'll install cert-manager using these commands:

```{bash}
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.4/cert-manager.crds.yaml

kubectl create namespace cert-manager

helm repo add jetstack https://charts.jetstack.io

helm repo update

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.0.4
```

How we can finally install Rancher:

```{bash}
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.local
```


## Start Rancher

Rancher is very easy to start using the command provided on the Rancher website:

```{bash}
sudo docker run --privileged -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher
```

Now you can navigate to **https://10.10.1.10** on your service node (which is why it is the only VM I put a GUI on) and start to get your cluster set up. 

**Reminder:** Got back an add the HTTP and HTTPS service to HAProxy on the Services VM and you can instead visit **https://10.10.1.1** or even the 
IP for interface on your home network.

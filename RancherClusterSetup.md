# Rancher Cluster Setup

## Create Node Template

In order to set up a highly-available install of Rancher, you have to establish a RKE cluster (the hard way). This is actually pretty straightforward if you make a node template (or whatever other automation you want to use) and clone it three times. After trying this with Debian 10 and running into DNS resolution issues in the pods, I decided to go with what was recommended by Rancher: Ubuntu 16.04.

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
### Install RKE
Visit the link below to get the latest release of the RKE client: 

https://github.com/rancher/rke/releases/tag/v1.0.14

```{bash}
wget https://github.com/rancher/rke/releases/download/v1.0.14/rke_linux-amd64

chmod +x rke_linux-amd64

sudo mv rke_linux-amd64 /usr/local/bin/rke
```

### Install Kubectl:
```{bash}
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x ./kubectl

sudo mv ./kubectl /usr/local/bin/kubectl
```

### Installing Helm

The easiest way to install things on RKE cluster is using a Helm chart (if one exists). To install the Rancher server on the RKE cluster, you must first install Helm. **Note:** You'll see mentions of *Tiller* if you look up Helm-stuff. If you are using Helm 3.0+, Tiller is no longer required.

```{bash}
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
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

## Install Rancher
https://rancher.com/docs/rancher/v2.x/en/installation/install-rancher-on-k8s/

```{bash}
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable

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
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.rancher.lan
```

While you are waiting for the deployment to finish, go back an add the HTTP and HTTPS service to HAProxy on the Services VM. Once everything is complete, you can visit **https://rancher.rancher.lan** to start using Rancher. 

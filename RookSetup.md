# Rook/Ceph Setup

Before you can do any real work on your shiny new cluster, you have to have some persistent storage for configuration or other state-ful tasks. 
I decided to utilize Rook because of it's integration with Ceph with a well-tested distributed storage solution. Rook offers your cluster access 
to Ceph's filesystem, block storage, or object storage backends. While object storage is the hotness for cloud stuff, I chose block storage due to 
the user-friendliness of it's filesystem interaction.

## Preparing Nodes

The easiest way to prepare disk space for usage by Rook is to mount a second drive to the VMs. Ceph wants a completely blank slate so once you mount the drive,
you are already ready to go. I added a 64 GB drive to each of my worker nodes at */dev/sdb** (the exact **sdX** doesn't matter but having a standard place is nice
for the next step).

## Installing Rook

### Install the Repository

Rook has numerous custom resource definitions that need to be installed so it's best to allow Helm to do this for you. Rancher has an easy way to install a Helm chart
in the "Cluster Explorer" interface. Once there you click the "Apps" button and go to the "Chart Repositories" tab. You'll want to create a new repository and use
this link from the Rook documentation:

```
https://charts.rook.io/release
```

Once you add a name (like "Rook") you should be all set.

### Project and Namespace

You'll want to create a project and/or namespace for your Rook install. I created a new project called "Rook" and a new namespace in that project called "Rook-Ceph". 
This is most easily accomplished in the Cluster Manager interface of Rancher. I believe you can also just create a namespace under the default project but the namespace
still needs to be called "rook-ceph" unless you want to make configuration edits elsewhere.

### Install the Rook Operator

The next step is to create all the Rook CRDs by deploying the Rook app in Cluster Explorer. This should be pretty painless as long as you point the install to the
"rook-ceph" namespace.

### Deploying the Cluster

I pulled the **cluster.yaml** from the Rook GitHub repo here: https://github.com/rook/rook/tree/master/cluster/examples/kubernetes/ceph

I didn't make a lot of changes except to the node section of the *yaml* to ensure that only the second hard drives on my worker nodes were provisioned as OSDs:

```{bash}
...
  storage:
    useAllNodes: false
    useAllDevices: false
  ...
  nodes:
  - names: "rke1-7"
    devices:
    - name: "sdb"
  - names: "rke1-8"
    devices:
    - name: "sdb"
  - names: "rke1-9"
    devices:
    - name: "sdb"
    #-name: "/dev/disk/by-id/XXXXX-XXXX-XXXXX"
```

**Note:** I added a line in the configuration above to show how to define your disk by the UUID. If I were using real hard drives for OSDs, I would probably use this
method of naming the disks as it's much less prone to issue in the future should hard drives be physically switched around if you have a failure to mount.

Once you've made your changes, you can use kubectl (inside Rancher or outside) to create this deployment:

```{bash}
kubectl create -f cluster.yaml
```

**Note**: If you utilize kubectl outside of Rancher, you need to sort out your kubeconfig file.




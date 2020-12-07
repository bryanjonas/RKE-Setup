# Services VM

Taking the lessons learned from setting up an OKD4 cluster, I really liked the flexibility and segmentation 
offered by establishing a seperate LAN for the cluster machines. In order to enable this, I made a "services" VM
that hosted the DNS server, DHCP server, and served as the gateway for communication outside of the cluster.

As I will be utilizing Debian for the cluster machines, I wanted to use Debian 10 for the service VM (not that it's
strictly necessary at all). The VM has 4 cores, 8GB of RAM, and a 32 GB harddrive. It's got two Proxmox virtual bridges
attached as network devices: one connected to the internet (**vmbr0**) and the other for internal communication (**vmbr1**).

### Firewalld

I got to use **firewalld** on CentOS during my OKD4 set up and it seemed to make things a lot easier than the **iptables** 
utility which is standard in Debian. Instead of wrestling with **iptables**, I decided to put **firewalld** on my services VM
and set up the internal and external firewall zones. I renamed my connections to their device name for easy tracking.

```{bash}
sudo apt install firewalld

sudo nmcli connection modify ens18 connection.zone external
sudo nmcli connection modify ens19 connection.zone internal

#Check settings
sudo firewall-cmd --get-active-zones
```

In order to allow clients on the internal network to communicate with outside (to grab images and other tasks), we turn on masquerading on 
both interfaces and check to ensure that IP forwarding is turned on.
```{bash}
sudo firewall-cmd --zone=external --add-masquerade --permanent
sudo firewall-cmd --zone=internal --add-masquerade --permanent

#Should print '1' to screen
cat /proc/sys/net/ipv4/ip_forwarding
```

### DNS Server
This next thing to install is *bind* as the DNS server for the cluster internal communication. 

```{bash}
sudo apt install -y bind9
```

We need to do into **/etc/default/bind9** and change an **options** line to **OPTIONS="-u bind -4"** to set in IPv4-only mode.

The following lines should be added inside the existing **options** block inside **/etc/bind/named.conf.options**:

```{bash}
verison "not currently available";

recursion no;

querylog yes;

allow-transfer { none; };
```
Next you need to add a zone to **/etc/bind/named.conf.local** by adding this block:

```{bash}
zone "rancher.local" {
  type master;
  file "/etc/bind/db.rancher.local";
  allow-query { any; };
```

Of course we then need to add the configuration for that zone but creating a file at **/etc/bind/db.rancher.local** with these contents:

```{bash}
$TTL  86400
$ORIGIN rancher.local.

; Key characteristics of zone
@ IN  SOA services.rancher.local.  services.rancher.local. (
          1
          604800
          86400
          2419200
          86400 )

; Name servers
@ IN  NS  services.rancher.local.

; A records
; Records for Rancher Cluster
rancher-1 IN  A 10.10.1.10
rancher-2 IN  A 10.10.1.11
rancher-2 IN  A 10.10.1.12

; Records for RKE Cluster 1
rke1-1  IN  A 10.10.1.20
rke1-2  IN  A 10.10.1.21
rke1-3  IN  A 10.10.1.22
rke1-4  IN  A 10.10.1.23
rke1-5  IN  A 10.10.1.24
```

####

```{bash}
sudo firewall-cmd --permanent --zone=internal --add-port=53/udp
sudo firewall-cmd --reload
sudo systemctl enabled named
sudo systemctl start named
```

*Remember to update the ens18 (external) to use 127.0.0.1 for DNS.*

### DHCP Server
We need to provide a DHCP server for the internal network clients. Be sure that the MAC addresses in the configuration file match those of your
clients. 

```{bash}
sudo dnf install -y dhcp-server 
sudo cp dhcpd.conf /etc/dhcp/dhcpd.conf
sudo firewall-cmd --permanent --zone=internal --add-service=dhcp
sudo firewall-cmd --reload
sudo systemctl enable dhcpd
sudo systemctl starrt dhcpd
```

### Loadbalancer
We are going to use HAProxy as the load balancer between the control planes and between the compute nodes. The services VM will act as the load balancer and redirect requests to the various APIs to the proper nodes.

We need to install 
```{bash}
sudo dnf install haproxy -y
sudo cp haproxy.cfg /etc/haproxy/haproxy.cfg

sudo setsebool -P haproxy_connect_any 1
sudo systemctl enable haproxy
sudo systemctl start haproxy
sudo systemctl status haproxy

sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=22623/tcp
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

      

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
listen-on port 53 { 127.0.0.1; 10.10.1.1; };

allow-query { localhost; 10.10.1.0/24; };

recursion yes;

forwarders {
       192.168.1.53;
       192.168.1.54;
};
```
Next you need to add a zone to **/etc/bind/named.conf.local** by adding this block:

```{bash}
zone "rancher.local" {
  type master;
  file "/etc/bind/db.rancher.local";
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
We have to open the firewall to allow traffic to the DNS server and start the DNS server service.

```{bash}
sudo firewall-cmd --permanent --zone=internal --add-port=53/udp
sudo firewall-cmd --reload
sudo systemctl enabled bind9
sudo systemctl start bind9
```

*Remember to update the ens18 (external) to use 127.0.0.1 for DNS.*

### DHCP Server
We need to provide a DHCP server for the internal network clients. Be sure that the MAC addresses in the configuration file match those of your
clients. 

```{bash}
sudo apt install -y isc-dhcp-server 
```

The first thing you need to do is to establish what interface to listen on for the DHCP server. You can so this by setting the correct interface (ens19 in my case) in **/etc/default/isc-dhcp-server** as well as uncommenting the lines pointing to the DHCP configuration and PID for IPv4.

Next some changes need to be made **/etc/dhcp/dhcpd.conf**:

```{bash}
option domain-name "rancher.local";
option domain-name-servers services.rancher.local;

default-lease-time 600;
max-lease-time 7200;

ddns-update-style none;

authoritative;

subnet 10.10.1.0 netmask 255.255.255.0 {
option routers 10.10.1.1;
option subnet-mask 255.255.255.0;
option domain-name "rancher.local";
option domain-name-servers 10.10.1.1;
range 10.10.1.100 10.10.1.199;
}

#Add these for all your clients
host rancher-1 {
 hardware ethernet 00:00:00:00:00:00;
 fixed-address 10.10.1.10;
}
```
Now we have to add the DHCP service to the firewall and start the DHCP service.

```{bash}
sudo firewall-cmd --permanent --zone=internal --add-service=dhcp
sudo firewall-cmd --reload
sudo systemctl enable dhcpd
sudo systemctl starrt dhcpd
```

### Loadbalancer
We are going to use HAProxy as the load balancer between the control planes and between the compute nodes. The services VM will act as the load balancer and redirect requests to the various APIs to the proper nodes.

```{bash}
sudo apt install haproxy
```

Replace the contents of **/etc/haproxy/haproxy.cfg** with what is below:

```{bash}
global
    maxconn     20000
    log         /dev/log local0 info
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    user        haproxy
    group       haproxy
    daemon

    stats socket /var/lib/haproxy/stats

defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          300s
    timeout server          300s
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 20000

listen stats
    bind :9000
    mode http
    stats enable
    stats uri /

#Define services as needed
frontend rancher_http_fe
    bind :80
    default_backend rancher_http_be
    mode tcp
    option tcplog

backend rancher_http_be
    balance source
    mode tcp
    server rancher-1 10.10.1.10 check
    server rancher-2 10.10.1.11 check
    server rancher-3 10.10.1.12 check
```

Finally open the firewall and start the service:

```{bash}
sudo firewall-cmd --permanent --add-port=9000/tcp
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
sudo systemctl enable haproxy
sudo systemctl start haproxy
```

This should be all of the services necessary for this VM.
      

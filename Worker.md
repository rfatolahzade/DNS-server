# Setting up Worker Node as a DNS Server

This guide outlines the steps to configure a worker node to act as a DNS server, functioning as a slave server that replicates DNS zone data from a master server. 
By setting up BIND (Berkeley Internet Name Domain) on the worker node, you'll ensure efficient DNS resolution within your network.
Ensure time synchronization between master and worker nodes:
```bash
# Check current timezone and set it if necessary
timedatectl list-timezones | grep Asia/Teh
timedatectl set-timezone Asia/Tehran

# Install and enable NTP service for time synchronization
yum install chrony -y
systemctl enable --now chronyd
```
make sure on Worker node (CentOS) bind packege installed:
```bash
[root@centos-20 ~]# yum search server | grep -i "domain name system"
bind.x86_64 : The Berkeley Internet Name Domain (BIND) DNS (Domain Name System)

now search it in local:
rpm -qa | grep bind
Result:
bind-libs-lite-9.11.4-26.P2.el7_9.15.x86_64
bind-utils-9.11.4-26.P2.el7_9.15.x86_64
bind-license-9.11.4-26.P2.el7_9.15.noarch
bind-libs-9.11.4-26.P2.el7_9.15.x86_64
bind-export-libs-9.11.4-26.P2.el7.x86_64
```
Install BIND DNS server and configure it to act as a worker:

```bash
# Install BIND DNS server
yum install bind -y
# This packege will add:
# bind-9.11.4-26.P2.el7_9.15.x86_64
```

Configure BIND to listen on specific interfaces and allow queries
Edit `/etc/named.conf` and add the following lines:
```bash
options {
    listen-on port 53 { 192.168.165.20; }; # Change to the appropriate interface IP any for all
    allow-query { 192.168.165.0/24; };     # Adjust IP range as needed any for all
};
```
Define zone for transfer from master server
```bash
zone "finezone.local" {
    type slave;
    masters { 192.168.165.10; }; # Change to master DNS server IP
    file "db.finezone.local";
};
``` 
Restart BIND service
```bash
systemctl restart named
[root@centos-20 ~]# cat /var/log/messages | grep transfer
Apr 11 20:40:04 centos-20 kernel: pci 0000:00:00.0: Limiting direct PCI/PCI transfers
Apr 11 22:01:41 centos-20 named[5245]: transfer of 'finezone.local/IN' from 192.168.165.10#53: connected using 192.168.165.20#53312
Apr 11 22:01:41 centos-20 named[5245]: zone finezone.local/IN: transferred serial 10
Apr 11 22:01:41 centos-20 named[5245]: transfer of 'finezone.local/IN' from 192.168.165.10#53: Transfer status: success
Apr 11 22:01:41 centos-20 named[5245]: transfer of 'finezone.local/IN' from 192.168.165.10#53: Transfer completed: 1 messages, 8 records, 235 bytes, 0.008 secs (29375 bytes/sec)

#ZoneTransfering
[root@centos-20 ~]# ls /var/named/
data  db.finezone.local  dynamic  named.ca  named.empty  named.localhost  named.loopback  slaves
```
Let test it:

On Worker node:
```bash
[root@centos-20 ~]# dig @localhost mail.finezone.local

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> @localhost mail.finezone.local
; (2 servers found)
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 4236
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;mail.finezone.local.           IN      A

;; ANSWER SECTION:
mail.finezone.local.    600     IN      A       192.168.165.70

;; AUTHORITY SECTION:
finezone.local.         600     IN      NS      ubuntuserver.finezone.local.

;; ADDITIONAL SECTION:
ubuntuserver.finezone.local. 600 IN     A       192.168.165.10

;; Query time: 2 msec
;; SERVER: ::1#53(::1)
;; WHEN: Thu Apr 11 21:22:20 +0330 2024
;; MSG SIZE  rcvd: 107
```
On Client node:
```bash
#sudo systemctl stop firewalld in future repo Im going to define iptable role for it.
root@worker-30:~# dig @192.168.165.20 mail.finezone.local

; <<>> DiG 9.18.18-0ubuntu0.22.04.2-Ubuntu <<>> @192.168.165.20 mail.finezone.local
; (1 server found)
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 18393
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: c71349025bd35f40f6474f3e6619065c2e02f0dd7a10f35e (good)
;; QUESTION SECTION:
;mail.finezone.local.           IN      A

;; ANSWER SECTION:
mail.finezone.local.    600     IN      A       192.168.165.70

;; AUTHORITY SECTION:
finezone.local.         600     IN      NS      ubuntuserver.finezone.local.

;; ADDITIONAL SECTION:
ubuntuserver.finezone.local. 600 IN     A       192.168.165.10

;; Query time: 12 msec
;; SERVER: 192.168.165.20#53(192.168.165.20) (UDP)
;; WHEN: Fri Apr 12 13:31:01 +0330 2024
;; MSG SIZE  rcvd: 135

```
Also you can check master node too.
If you define these two nodes as nameserver you can have name reolotion and name reservation (based on which node is down) live.
on ubuntu client:
```bash
nano /etc/netplan/00-installer-config.yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [192.168.165.30/24]
      routes:
          - to: default
            via: 192.168.165.1
      nameservers:
        addresses: [192.168.165.20, 192.168.165.10]
  version: 2

```
To ensure that systemd-resolved uses the DNS settings specified in Netplan and updates /etc/resolv.conf accordingly, you need to disable its DNSStubListener feature.
Here's how you can do it via edit the systemd-resolved configuration file:
```bash
 nano /etc/systemd/resolved.conf
#Add or modify the following line to disable DNSStubListener:
DNSStubListener=no
```
save and close then run:
```bash
systemctl restart systemd-resolved
 cat /etc/resolv.conf 
# operation for /etc/resolv.conf.

nameserver 192.168.165.20
nameserver 192.168.165.10
search .
```
Ensure Failover Capability by configuring both master and worker nodes as DNS servers on client nodes, you ensure failover capability. If one of the DNS servers goes down, client nodes will automatically switch to the other available DNS server for uninterrupted DNS resolution.

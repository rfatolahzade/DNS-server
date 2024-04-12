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

#### TSIG (Transaction Signature)
Also referred to as Secret Key Transaction Authentication, ensures that DNS packets originate from an authorized sender by using shared secret keys and one-way hashing to add a cryptographic signature to the DNS packets.
##### Setup on Master Node:
Generate a TSIG key on the Master node using the following command:
```bash
tsig-keygen -a HMAC-MD5 dnskey >  /etc/bind/named.conf.tsig

```
Include the generated TSIG key in the BIND configuration file /etc/bind/named.conf by adding the following line:

```bash
 include "/etc/bind/named.conf.tsig";
```
Allow zone transfer to the Worker node by adding the following configuration to /etc/bind/named.conf.local:
Add `allow-transfer { key "dnskey"; };` :
```bash
zone "finezone.local" {
    type master;
    file "/etc/bind/db.finezone.local";
    allow-transfer { key "dnskey"; };
};
```
Modify the zone file /etc/bind/db.finezone.local to include necessary DNS records such as NS records and A records.
```bash
root@ubuntuserver-10:/etc/bind# cat /etc/bind/db.finezone.local
$TTL 600
@       IN      SOA     finezone.local.   root.finezone.local. (
        11      ; Serial Number
        1200    ; Refresh Interval
        300     ; Retry Interval
        86400   ; Expiry
        7200 )  ; Minimum Cache TTL

@       IN      NS      ubuntuserver.finezone.local.
@       IN      NS      centos.finezone.local.
ubuntuserver.finezone.local.    IN      A       192.168.165.10
centos.finezone.local.    IN      A       192.168.165.20
www     IN      A       192.168.165.60
mail    IN      A       192.168.165.70
dash    IN      A       192.168.165.80
blog    IN      A       192.168.165.90
print   IN      A       192.168.165.100
```
Restart the BIND service to apply the changes:
```bash
systemctl restart bind9
rndc reload
```
##### Configuration on Worker Node
Restart named service to force it zone transfering,Check for zone transfer logs in /var/log/messages to verify if zone transfer is failing.
If zone transfer fails, add the generated TSIG key to the BIND configuration file /etc/named.conf on the Worker node:
```bash
[root@centos ~]# grep transfer  /var/log/messages 
Apr 12 14:18:30 centos named[2225]: transfer of 'finezone.local/IN' from 192.168.165.10#53: connected using 192.168.165.20#38307
Apr 12 14:18:30 centos named[2225]: zone finezone.local/IN: transferred serial 10
Apr 12 14:18:30 centos named[2225]: transfer of 'finezone.local/IN' from 192.168.165.10#53: Transfer status: success
Apr 12 14:18:30 centos named[2225]: transfer of 'finezone.local/IN' from 192.168.165.10#53: Transfer completed: 1 messages, 8 records, 235 bytes, 0.012 secs (19583 bytes/sec)
Apr 12 14:53:47 centos named[2225]: transfer of 'finezone.local/IN' from 192.168.165.10#53: failed to connect: connection refused
Apr 12 14:53:47 centos named[2225]: transfer of 'finezone.local/IN' from 192.168.165.10#53: Transfer status: connection refused
Apr 12 14:53:47 centos named[2225]: transfer of 'finezone.local/IN' from 192.168.165.10#53: Transfer completed: 0 messages, 0 records, 0 bytes, 0.008 secs (0 bytes/sec)
Apr 12 15:00:41 centos named[2225]: transfer of 'finezone.local/IN' from 192.168.165.10#53: failed to connect: connection refused
Apr 12 15:00:41 centos named[2225]: transfer of 'finezone.local/IN' from 192.168.165.10#53: Transfer status: connection refused
Apr 12 15:00:41 centos named[2225]: transfer of 'finezone.local/IN' from 192.168.165.10#53: Transfer completed: 0 messages, 0 records, 0 bytes, 0.008 secs (0 bytes/sec)
Apr 12 16:55:09 centos named[2225]: transfer of 'finezone.local/IN' from 192.168.165.10#53: failed to connect: connection refused
Apr 12 16:55:09 centos named[2225]: transfer of 'finezone.local/IN' from 192.168.165.10#53: Transfer status: connection refused
Apr 12 16:55:09 centos named[2225]: transfer of 'finezone.local/IN' from 192.168.165.10#53: Transfer completed: 0 messages, 0 records, 0 bytes, 0.006 secs (0 bytes/sec)
Apr 12 16:56:03 centos named[2225]: zone finezone.local/IN: refresh: skipping zone transfer as master 192.168.165.10#53 (source 0.0.0.0#0) is unreachable (cached)
```
As you can see zone transfering failed.
To fix: paste generated sig file to `/etc/named.conf ` on Worker node:
```bash
key "dnskey" {
        algorithm hmac-md5;
        secret "XklbnFYLTvw4kxHrQkfcmA==";
};

server 192.168.165.10 {
        keys { dnskey; };
};

```
Restart the BIND service on the Worker node to apply the configuration changes:
```bash
systemctl restart named
rndc reload
```
Check the logs in /var/log/messages to ensure successful zone transfer.
```bash
[root@centos ~]# cat /var/log/messages | grep transfer
..
 zone finezone.local/IN: transferred serial 11: TSIG 'dnskey'
..
```
Test the DNS resolution for a specific record, for example, the print record:
```bash
[root@centos ~]# dig @localhost print.finezone.local

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> @localhost print.finezone.local
; (2 servers found)
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 37466
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;print.finezone.local.          IN      A

;; ANSWER SECTION:
print.finezone.local.   600     IN      A       192.168.165.100

;; AUTHORITY SECTION:
finezone.local.         600     IN      NS      ubuntuserver.finezone.local.
finezone.local.         600     IN      NS      centos.finezone.local.

;; ADDITIONAL SECTION:
centos.finezone.local.  600     IN      A       192.168.165.20
ubuntuserver.finezone.local. 600 IN     A       192.168.165.10

;; Query time: 7 msec
;; SERVER: ::1#53(::1)
;; WHEN: Fri Apr 12 16:40:04 +0330 2024
;; MSG SIZE  rcvd: 145
```
Verify the DNS resolution results to ensure the desired IP address is returned.

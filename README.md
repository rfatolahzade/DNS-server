# DNS Server Setup Guide
This guide provides step-by-step instructions for setting up and configuring a DNS Server on a Linux machine using BIND.

### Introduction

Setting up a DNS (Domain Name System) server allows you to translate domain names into IP addresses and vice versa, facilitating network communication. This guide will walk you through the process of installing and configuring a DNS server using BIND (Berkeley Internet Name Domain) on a Linux machine.

### Prerequisites
Before proceeding, ensure you have the following:

- A Linux machine (Ubuntu, Debian, CentOS, etc.)
- Root or sudo access to the machine
- Basic knowledge of Linux command-line interface

#### Understanding Name Resolution on Linux

In Linux systems, the order of priority for name resolution is defined in the /etc/nsswitch.conf file. Let's explore:
```bash
# /etc/nsswitch.conf
#
# Example configuration of GNU Name Service Switch functionality.
# If you have the `glibc-doc-reference' and `info' packages installed, try:
# `info libc "Name Service Switch"' for information about this file.

passwd:         files systemd
group:          files systemd
shadow:         files
gshadow:        files

hosts:          files dns
networks:       files

protocols:      db files
services:       db files
ethers:         db files
rpc:            db files

netgroup:       nis
```
In the `nsswitch.conf` file, the line `hosts: files dns` specifies the order of lookup for hostname resolution. It means that the system will first check the `/etc/hosts` file (`files`) and then query the DNS server (`dns`). You can modify this order by editing the `nsswitch.conf` file.

By default, DNS operates on port 53 for name resolution. To troubleshoot name resolutions, you can use commands like `dig`, `nslookup`, and `host`, which provide information about both IPv4 and IPv6 addresses.

When you run the `dig` command without any options, it queries the DNS server configured on your system:
```bash
root@ubuntuserver-10:~# dig google.com

; <<>> DiG 9.18.18-0ubuntu0.22.04.2-Ubuntu <<>> google.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 24843
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;google.com.                    IN      A

;; ANSWER SECTION:
google.com.             60      IN      A       216.239.38.120

;; Query time: 51 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
;; WHEN: Sat Mar 30 09:18:42 UTC 2024
;; MSG SIZE  rcvd: 55

```
The output includes details such as the DNS server used for resolution (`SERVER`), response time (`Query time`), and the type of query performed (`QUESTION SECTION`).

To change the DNS server, you can edit the `/etc/resolv.conf` file. For example:
```bash
nano /etc/resolv.conf
```
Add the desired DNS server(s) using the nameserver directive:
```bash
nameserver 1.1.1.1
```
Then, save the file and run dig again to verify the changes.
```bash
root@ubuntuserver-10:~# dig google.com

; <<>> DiG 9.18.18-0ubuntu0.22.04.2-Ubuntu <<>> google.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 19498
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;google.com.                    IN      A

;; ANSWER SECTION:
google.com.             60      IN      A       216.239.38.120

;; Query time: 24 msec
;; SERVER: 1.1.1.1#53(1.1.1.1) (UDP)
;; WHEN: Sat Mar 30 09:25:20 UTC 2024
;; MSG SIZE  rcvd: 44
```
By observing the `SERVER` field in the `dig` output, you can confirm which DNS server responded to the query. Additionally, the `Query time `indicates the time taken for the DNS resolution process.
The `QUESTION SECTION` specifies the type of record being queried (e.g., `IN A` for IPv4 address resolution). Other record types include `AAAA` for IPv6, `MX` for mail exchange, `CNAME` for canonical name, etc.
Remember to use appropriate options with `dig` to query specific record types or filter the output as needed.

#### Resource Records Overview

Resource Records (RRs) are essential components of DNS (Domain Name System) that provide various types of information associated with domain names. Here's a brief overview of commonly used resource record types:
| Type | Description |
| --- | --- |
| A (Address)  Record |  Maps a Fully Qualified Domain Name (FQDN) to an IPv4 address |
| AAAA (IPv6 Address)  Record |  Maps an FQDN to an IPv6 address |
| PTR (Pointer)  Record |  Maps an IPv4 or IPv6 address to an FQDN, commonly used in reverse DNS lookups |
| CAA (Certification Authority Authorization)  Record |  Specifies which certificate authorities are allowed to issue certificates for a domain |
| CERT (Certificate)  Record |  Stores certificates associated with a domain |
| CNAME (Canonical Name)  Record |  Alias of one domain name to another, allowing multiple domain names to resolve to the same IP address |
| DNSKEY (DNS Key)  Record |  Contains public encryption keys for DNSSEC (Domain Name System Security Extensions) |
| DS (Delegation Signer)  Record |  Contains a hash of a DNSKEY record, used to authenticate child zones |
| HTTPS (HTTP Secure)  Record |  Specifies the location of HTTPS service for a domain |
| LOC (Location)  Record |  Stores geographic location information about a domain |
| MX (Mail Exchange)  Record |  Specifies the mail server responsible for receiving email messages on behalf of a domain |
| NAPTR (Naming Authority Pointer)  Record |  Specifies rules for rewriting domain names for applications such as SIP (Session Initiation Protocol) and ENUM (Telephone Number Mapping) |
| NS (Name Server)  Record |  Specifies authoritative name servers for a domain |
| PTR (Pointer)  Record |  Maps an IPv4 or IPv6 address to an FQDN, commonly used in reverse DNS lookups |
| SMIMEA (S/MIME Certificate Association)  Record |  Specifies certificates associated with S/MIME email encryption |
| SRV (Service)  Record |  Specifies the location of services such as SIP or XMPP (Extensible Messaging and Presence Protocol) |
| SSHFP (SSH Fingerprint)  Record |  Stores fingerprints of SSH (Secure Shell) public keys |
| SVCB (Service Binding)  Record |  Describes parameters and preferences for network services |
| TLSA (TLS Authentication)  Record |  Associates a TLS certificate or public key with a domain name |
| TXT (Text)  Record |  Contains human-readable text, commonly used for DNS-based authentication like SPF (Sender Policy Framework) and DKIM (DomainKeys Identified Mail) |
| URI (Uniform Resource Identifier)  Record |  Specifies a resource available via URI |


Each resource record belongs to a specific class. The most commonly used class is the Internet class (IN).
Examples:
  - TXT Sample: Specifies SPF records for email authentication:
```bash
  v=spf1 include:_spf.firebasemail.com ~all
```
 - MX Sample: Specifies the mail exchange server for a domain:
```bash
 alt4.aspmx.l.google.com
```
 - CNAME Sample: Aliases one domain name to another:
```bash
 app-name.ondigitalocean.app
```

#### Additional Options for dig

To query specific record types or filter the output:

  - To query for AAAA records:

```bash
dig -t AAAA google.com
```

 - To display only the answer section:

```bash
dig -t AAAA +noall +answer google.com
```

 - To specify a DNS server to query:

```bash
dig @8.8.8.8 -t AAAA +noall +answer google.com
```

### Configuration
To set up a BIND DNS server on Ubuntu as a Caching DNS Server, follow these steps:

##### 1. Check Installed Packages:

```bash 
root@ubuntuserver-10:~# dpkg -l | grep bind9
ii  bind9-dnsutils                        1:9.18.18-0ubuntu0.22.04.2              amd64        Clients provided with BIND 9
ii  bind9-host                            1:9.18.18-0ubuntu0.22.04.2              amd64        DNS Lookup Utility
ii  bind9-libs:amd64                      1:9.18.18-0ubuntu0.22.04.2              amd64        Shared Libraries used by BIND 9
```
These packages are for the client-side utilities. To install BIND 9 on Ubuntu, execute:

```bash
apt install bind9 -y
```
Checkout again:
```bash
root@ubuntuserver-10:~# dpkg -l | grep bind9
ii  bind9                                  1:9.18.18-0ubuntu0.22.04.2              amd64        Internet Domain Name Server
ii  bind9-dnsutils                         1:9.18.18-0ubuntu0.22.04.2              amd64        Clients provided with BIND 9
ii  bind9-host                             1:9.18.18-0ubuntu0.22.04.2              amd64        DNS Lookup Utility
ii  bind9-libs:amd64                       1:9.18.18-0ubuntu0.22.04.2              amd64        Shared Libraries used by BIND 9
ii  bind9-utils                            1:9.18.18-0ubuntu0.22.04.2              amd64        Utilities for BIND 9

```
The first line is our desire package.
###### 2. Verify BIND Service:
Check the status of the BIND 9 service:

```bash
root@ubuntuserver-10:~# systemctl status bind9
```
###### 3. Configuration Files:
The main configuration files for BIND 9 are located in `/etc/bind`. Here's an example of the options configuration file:
```bash
root@ubuntuserver-10:/etc/bind# cat named.conf.options 
options {
        directory "/var/cache/bind";

        // DNSSEC validation setting
        dnssec-validation auto;

        // Listen on IPv6 interfaces
        listen-on-v6 { any; };
};

```
##### 4. Configuring as a Caching DNS Server:
To configure Ubuntu server as a caching DNS server, edit the `/etc/bind/named.conf.options` file then add `recursion yes;` to config file to enable caching. 
The directive` recursion yes`; in the BIND configuration file (`named.conf.options`) enables recursive DNS resolution on the DNS server.
When recursive DNS resolution is enabled, the DNS server will perform queries on behalf of clients to resolve domain names recursively. This means that if the DNS server receives a query for a domain name it doesn't have in its cache, it will query other DNS servers on the internet to resolve the domain name.
In simpler terms, enabling recursion allows the DNS server to act as a resolver for client queries, providing them with complete answers even if it requires querying other DNS servers. This is essential for DNS servers that serve client devices on a network, as it allows them to resolve domain names beyond their local zone.
Restarting BIND9 Service to apply changes:
```bash
systemctl restart bind9
```
If you want to force a reload of the service,The `including` files in `/etc/bind/named.conf`, you can use rndc reload:
```bash
rndc reload 
# Output: server reload successful
```
The `rndc reload` command is used to reload the configuration of the BIND name server without interrupting its operation. It stands for "Remote Name Daemon Control" and is part of the BIND suite of tools.
When you make changes to the BIND configuration files, such as named.conf or its included files, you need to reload the configuration to apply those changes. However, restarting the BIND service (systemctl restart bind9) would momentarily interrupt the service, potentially causing downtime for DNS resolution.This is where rndc reload comes in handy. 

For further debugging, you can check the service status using:
```bash
 journalctl -xeu bind9
# or
 systemctl status bind9;
```
You can also monitor the listening port using either `netstat` or `ss`:
```bash
netstat -plntu | grep named
# or
ss -plntu | grep /named
```

`resolvectl` Status Output:
```bash
root@ubuntuserver-10:/etc/bind# resolvectl status

Global
       Protocols: -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
resolv.conf mode: stub

Link 2 (enp0s3)
    Current Scopes: DNS
         Protocols: +DefaultRoute +LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 4.2.2.4
       DNS Servers: 4.2.2.4

Link 3 (docker0)
Current Scopes: none
     Protocols: -DefaultRoute +LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported

```
Explanation:

   - Global: Indicates settings applicable across all interfaces.
   - Link 2 (enp0s3): DNS configuration specific to the "enp0s3" network interface.
   - Link 3 (docker0): DNS configuration for the "docker0" network interface.


The DNS server in Link 2 (enp0s3) originates from the netplan configuration:
```bash
root@ubuntuserver-10:/etc/bind# cat /etc/netplan/00-installer-config.yaml 
# This is the network config written by 'subiquity'
network:
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [192.168.165.10/24]
      routes:
          - to: default
            via: 192.168.165.1
      nameservers:
        addresses: [4.2.2.4]
  version: 2
```
For test at first Im going to comment nameserver in netplan config:
```bash
#      nameservers:
#        addresses: [4.2.2.4]
```
After applying :
```bash
netplan generate
netplan apply
```
Execute again:

```bash
root@ubuntuserver-10:/etc/bind# resolvectl status

Global
       Protocols: -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
resolv.conf mode: stub

Link 2 (enp0s3)
Current Scopes: none
     Protocols: -DefaultRoute +LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported

```
To adjust `global` DNS settings:
```bash

nano /etc/systemd/resolved.conf
```
Add your desired DNS server:
```bash
DNS=4.2.2.4
```
Then restart the service:
```bash
systemctl restart systemd-resolved
```
Updated resolvectl Status:
```bash
root@ubuntuserver-10:~# resolvectl status
Global
         Protocols: -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
  resolv.conf mode: stub
Current DNS Server: 4.2.2.4
       DNS Servers: 4.2.2.4

Link 2 (enp0s3)
Current Scopes: none
     Protocols: -DefaultRoute +LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
```
The Global DNS Server: Updated to 4.2.2.4 as configured.
To test our Caching DNS Server, execute the following command:
```bash
dig @localhost yahoo.com
```
Output:
```bash
; <<>> DiG 9.18.18-0ubuntu0.22.04.2-Ubuntu <<>> @localhost yahoo.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 37527
;; flags: qr rd ra; QUERY: 1, ANSWER: 6, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: bea19b791138cda101000000660aac6b7c0dc2bfa54cbdc1 (good)
;; QUESTION SECTION:
;yahoo.com.                     IN      A

;; ANSWER SECTION:
yahoo.com.              1800    IN      A       98.137.11.163
yahoo.com.              1800    IN      A       74.6.143.25
yahoo.com.              1800    IN      A       74.6.231.20
yahoo.com.              1800    IN      A       74.6.143.26
yahoo.com.              1800    IN      A       74.6.231.21
yahoo.com.              1800    IN      A       98.137.11.164

;; Query time: 983 msec
;; SERVER: 127.0.0.1#53(localhost) (UDP)
;; WHEN: Mon Apr 01 12:45:31 UTC 2024
;; MSG SIZE  rcvd: 162

```
Analysis:
   - Query Time: Initially took `983 msec`
   - Server: Resolved via `127.0.0.1#53(localhost)`.
   - Received: 6 IP addresses for `"yahoo.com"`.

Subsequent Test:
Upon running the command again:
```bash
root@ubuntuserver-10:~# dig @localhost yahoo.com
```
Updated Output:
```bash
; <<>> DiG 9.18.18-0ubuntu0.22.04.2-Ubuntu <<>> @localhost yahoo.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 10569
;; flags: qr rd ra; QUERY: 1, ANSWER: 6, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: cf13ee186986cb8501000000660aac6d50b2ce04dc7f1eae (good)
;; QUESTION SECTION:
;yahoo.com.                     IN      A

;; ANSWER SECTION:
yahoo.com.              1798    IN      A       74.6.231.21
yahoo.com.              1798    IN      A       74.6.143.25
yahoo.com.              1798    IN      A       74.6.143.26
yahoo.com.              1798    IN      A       74.6.231.20
yahoo.com.              1798    IN      A       98.137.11.164
yahoo.com.              1798    IN      A       98.137.11.163

;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(localhost) (UDP)
;; WHEN: Mon Apr 01 12:45:33 UTC 2024
;; MSG SIZE  rcvd: 162
```
Analysis:
   - Query Time: Reduced to` 0 msec` upon subsequent query.
   - Server: Response still via` 127.0.0.1#53(localhost)`.
   - Received: Same 6 IP addresses for `"yahoo.com"`.

The `0 msec` query time indicates successful caching of the DNS response, confirming the effectiveness of our settings.


#### Setting Up a Master DNS Zone
In this guide, I'll configure a single Master DNS server in Static mode for the finezone.local domain with a single worker.

##### 1.Configure  `named.conf.local`
Open the BIND configuration file for local configurations:
```bash
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";

zone "finezone.local" {
             type master;
             file "/etc/bind/db.finezone.local";
};

```
##### 2.Create the Zone Database
Before creating the zone database, let's understand some concepts:
to define zones we need SOA record and SOA records contains:
   - Serial No (Integer) 
   - Refresh Interval (seconds) master send soa query to worker
   - Retry Interval  (seconds) if worker couldnt reach master
   - Expire (seconds) if worker cant reach master drop worker node
   - Cache TTL (seconds) - for client cache 
   - SOA TTL - for worker/secondery
   - NS record stands for 'nameserver,' and the nameserver record indicates which DNS server is authoritative for that domain(i.e. which server contains the actual DNS records)
   - SOA Record Contains critical information about the zone, including the refresh, retry, and expiry intervals.

Create the zone database file:

```bash
root@ubuntuserver-10:/etc/bind# nano /etc/bind/db.finezone.local
$TTL 600
@       IN      SOA     finezone.local.   root.finezone.local. (
        10      ;serial No
        1200    ;refresh Interval
        300     ;retry Interval
        86400   ;Expiration
        7200 )  ;cache TTL
@       IN      NS      ubuntuserver.finezone.local.
ubuntuserver.finezone.local.    IN      A 192.168.165.10
www.finezone.local.     IN      A       192.168.165.60
mail.finezone.local.    IN      A       192.168.165.70
dash.finezone.local.    IN      A       192.168.165.80
blog.finezone.local.    IN      A       192.168.165.90

```
In my configuration, I've specified the hostname `ubuntuserver.finezone.local` as the nameserver `(NS)` for the `finezone.local` domain. This means that when clients query for records within the `finezone.local` domain, they will be directed to the DNS server identified by the hostname `ubuntuserver.finezone.local`.
In the configuration, `ubuntuserver.finezone.local` is set as the nameserver `(NS)` for the `finezone.local` domain. This directs queries within the domain to the DNS server identified by `ubuntuserver.finezone.local.`

##### 3.Verification
After configuration, verify the DNS server's functionality using tools like  `dig` or `nslookup`:
```bash
[root@centos-20 ~]# dig @192.168.165.10  mail.finezone.local

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> @192.168.165.10 mail.finezone.local
; (1 server found)
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 38967
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;mail.finezone.local.           IN      A

;; ANSWER SECTION:
mail.finezone.local.    600     IN      A       192.168.165.70

;; Query time: 7 msec
;; SERVER: 192.168.165.10#53(192.168.165.10)
;; WHEN: Thu Apr 11 18:43:30 +0430 2024
;; MSG SIZE  rcvd: 64

```
Your DNS zone for `finezone.local` is now configured with a `master` DNS server. Verify the DNS resolution to ensure correct functionality.

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

#Configuring as a Caching DNS Server:

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


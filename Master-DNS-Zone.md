
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
        10      ; Serial Number
        1200    ; Refresh Interval
        300     ; Retry Interval
        86400   ; Expiry
        7200 )  ; Minimum Cache TTL

@       IN      NS      ubuntuserver.finezone.local.
ubuntuserver.finezone.local.    IN      A       192.168.165.10
www     IN      A       192.168.165.60
mail    IN      A       192.168.165.70
dash    IN      A       192.168.165.80
blog    IN      A       192.168.165.90

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

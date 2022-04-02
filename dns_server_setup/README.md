
# DNS Server Setup | Pre-requisite to create a Key Distribution Center

In this lab, we will go through how to setup an Internal DNS server using the BIND9 nameserver 
software in Ubuntu 18.04.

# Primary Goal: 
To have a central way to manage your internal hostnames and private IP addresses using FQDN,
thus increasing the managability of configuration files. 

# Prerequisites:
- A data center named, nys3
- 2x DNS servers in this Datacenter nys3 which will act as our name servers: ns1 and ns2. ns1 will be the master/primary nameserver and ns2 will the slave/secondary nameserver.
- 2x hsots in the same Datacenter: host1 and host2
- We assumed that all of our servers and hosts are connected to a project named "example.com". But we dont need to purchase this domain since our DNS system will be entirely private.


| **Datacenter** | nyc3 | nyc3.example.com |  |  |
|---|---|---|---|---|
| **Host** | **Role** | **Private FQDN** | **Private IP Address** | **Subnet** |
| ns1 | Primary/master DNS Server | ns1.nyc3.example.com | 10.128.10.11 | 10.128.0.0/16 |
| ns2 | Secondary/Salve DNS Server | ns2.nyc3.example.com | 10.128.20.12 | 10.128.0.0/16 |
| host1 | Generic Host/Client in the Data Center | host1.nyc3.example.com | 10.128.100.101 | 10.128.0.0/16 |
| host2 | Generic Host.Client in the Data Center | host2.nyc3.example.com | 10.128.100.102 | 10.128.0.0/16 |



## Installation
### A. Lab Environment Setup
- Download the [Vagrant file](./Vagrantfile) at your local machine
```bash
> vagrant status 
> vagrant up
> vagarnt status    # 4x Vagarnt server would be running
                    # 2x Name Servers
                    # 2x Clients/Hosts
```

### B. Install BIND9 DNS Software in all Name Servers: ns1 and ns2

#### B1. At Name Server 1: ns1.nyc1.example.com 
```bash
  # At local machine
  > vagrant ssh ns1.nyc1.example.com

  # Inside NameServer1 - Vagrant
  > sudo apt update
  > sudo apt install bind9 bind9utils bind9-doc

  # Setting up Bind to IPv4 Mode
  > sudo vim /etc/default/bind9
  ----
  ...
  OPTIONS="-u bind -4"

  ----

  # Restart the BIND9 Daemon and check the status
  > sudo systemctl restart bind9 
  > sudo systemctl enable bind9
  > sudo systemctl status bind9

  # Let your firewall open up access to BIND9
  > sudo ufw allow Bind9 
```

#### B2. At Name Server 2: ns2.nyc2.example.com
```bash
  # At local machine
  > vagrant ssh ns1.nyc1.example.com

  # Inside NameServer1 - Vagrant
  > sudo apt update
  > sudo apt install bind9 bind9utils bind9-doc

  # Setting up Bind to IPv4 Mode
  > sudo vim /etc/default/bind9
  ----
  ...
  OPTIONS="-u bind -4"

  ----

  # Restart the BIND9 Daemon and check the status
  > sudo systemctl restart bind9 
  > sudo systemctl enable bind9
  > sudo systemctl status bind9

  # Let your firewall open up access to BIND9
  > sudo ufw allow Bind9 
```
    
## Configuring the Local Files of Name Servers
Lookup for the following files:
- /etc/bind/named.conf.options: Here we will setup a trusted ACL with the hosts in the subnet 10.128.0.0/16. This configuration specifies that only our own servers (4x in case) will be able to query our DNS, not the others.
- /etc/bind/named.conf.local: Configuring the DNS Zones
    - Forward Zone: FQDN/hostname->ip resolution.
        - Could be Master DNS/Slave DNS.
        - Requires a zone file with path (path, only if Master): [/etc/bind/zones/db.nyc3.example.com](../Name_Server_1_(Master)_Setup/db.nyc3.example.com) 
    - Reserve Zone: ip->FQDN resolution 
        - Could be Master/Slave DNS.
        - Requires a zone file with path (path, only if Master): [/etc/bind/zones/db.10.128](../Name_Server_1_(Master)_Setup/db.10.128) 

### Configuring Name Server 1: ns1 
```bash
# Check the files to be configured for Name Server setup
> cat /etc/bind/named.conf  # this is the primary configuration file for BIND DNS Name Server
---
// This is the primary configuration file for the BIND DNS server named.
//
// Please read /usr/share/doc/bind9/README.Debian.gz for information on the 
// structure of BIND configuration files in Debian, *BEFORE* you customize 
// this configuration file.
//
// If you are just adding zones, please do that in /etc/bind/named.conf.local

include "/etc/bind/named.conf.options"; 
include "/etc/bind/named.conf.local";
include "/etc/bind/named.conf.default-zones";
--

# Setup the Truested ACL" Only the servers who can get through our private DNS server in the subnet
> sudo vim /etc/bind/named.conf.options 
---
acl "trusted" {
        10.128.10.11;   # ns1
        10.128.20.12;   # ns2
        10.128.100.101; # host1
        10.128.100.102; # host2
};

options {
        directory "/var/cache/bind";

        recursion yes;                          # enables recursive quries
        allow-recursion { trusted; };           # allows recursive queries from "trusted" clients
        listen-on { 10.128.10.11; };            # ns1 private IP address - listen on private network only
        allow-transfer { none; };               # disable zone transfers by default


        // If there is a firewall between you and nameservers you want
        // to talk to, you may need to fix the firewall to allow multiple
        // ports to talk.  See http://www.kb.cert.org/vuls/id/800113

        // If your ISP provided one or more IP addresses for stable
        // nameservers, you probably want to use them as forwarders.
        // Uncomment the following block, and insert the addresses replacing
        // the all-0's placeholder.

        forwarders {
                8.8.8.8;
                8.8.4.4;
        };

        //========================================================================
        // If BIND logs error messages about the root key being expired,
        // you will need to update your keys.  See https://www.isc.org/bind-keys
        //========================================================================
        dnssec-validation auto;

        auth-nxdomain no;    # conform to RFC1035
        listen-on-v6 { any; };
};

---





```


# Setup Gateway

## Update the OS system repos cache

```bash
# SSH into the gateway VM
vagrant ssh gw01

# Update local repo cache
sudo dnf update -y
sudo dnf install curl wget git unzip tree bash-completion -y
```

## Enable Linux packet routing
```bash
# edit sysctl configs
vim /etc/sysctl.d/99-lab-router.conf

# Add the line
net.ipv4.ip_forward = 1

# Apply changes
sudo sysctl --system
sysctl net.ipv4.ip_forward
```

## Setup DNS for the Lab

```bash
sudo dnf install -y dnsmasq dnsutils

# Create the DNS Config file
vim /etc/dnsmasq.d/elastic-lab.conf

# Add the following lines

domain-needed
bogus-priv
no-resolv
server=1.1.1.1
server=8.8.8.8
server=9.9.9.9

local=/elastic.lab.test/
domain=elastic.lab.test
expand-hosts

listen-address=127.0.0.1,192.168.10.1,192.168.20.1,192.168.30.1
bind-dynamic

address=/gw01.elastic.lab.test/10.10.30.1

# VLAN 10 - Elasticsearch data tier
host-record=es01.elastic.lab.test,192.168.10.11
host-record=es02.elastic.lab.test,192.168.10.12
host-record=es03.elastic.lab.test,192.168.10.13

# VLAN 20 - Logstashn and App Server
host-record=ingest01.elastic.lab.test,192.168.20.21
host-record=app01.elastic.lab.test,192.168.20.22

# VLAN 30 - Admin VM and Kibana, APM, Fleet agents 
host-record=obs01.elastic.lab.test,192.168.30.31
host-record=ops01.elastic.lab.test,192.168.30.32

```
---
title:  "Server optimisation for nginx"
date: 2017-07-21T07:35:01+02:00
draft: false
tags: ["sysctl", "nginx", "linux"]
---

# Optimisation to use nginx node

### Use the full range of ports.
```config
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_max_syn_backlog=4096
net.core.netdev_max_backlog=4096
net.core.somaxconn=1024


net.ipv4.tcp_keepalive_time=300
net.core.somaxconn=250000
net.ipv4.tcp_max_syn_backlog=2500
net.core.netdev_max_backlog=2500
net.ipv4.tcp_tw_reuse = 1
```
### Bash resource initialisation
```bash
echo "10152 65535" > /proc/sys/net/ipv4/ip_local_port_range
sysctl -w fs.file-max=128000
sysctl -w net.ipv4.tcp_keepalive_time=300
sysctl -w net.core.somaxconn=250000
sysctl -w net.ipv4.tcp_max_syn_backlog=2500
sysctl -w net.core.netdev_max_backlog=2500

ulimit -n 10240 
```
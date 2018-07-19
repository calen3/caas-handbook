# iptables white list

### 

> 若在部署机器iptables采用了INPUT链的白名单机制，请执行以下iptables操作，允许各个服务的访问。

### 所有节点白名单

```bash
# ssh
iptables -I INPUT -p tcp --dport 22 -j ACCEPT
# dnsmasq
iptables -I INPUT -p udp --dport 53 -j ACCEPT
iptables -I INPUT -p tcp --dport 53 -j ACCEPT
# ntp
iptables -I INPUT -p udp --dport 123 -j ACCEPT
iptables -I INPUT -p tcp --dport 123 -j ACCEPT
# vxlan
iptables -I INPUT -p udp --dport 4789 -j ACCEPT
# prometheus node_exporter
iptables -I INPUT -p tcp --dport 9100 -j ACCEPT
```

### 存储机（storage）iptables白名单

```bash
# nfsd mountd rpcbind
iptables -I INPUT -p tcp --dport 111 -j ACCEPT
iptables -I INPUT -p udp --dport 111 -j ACCEPT
iptables -I INPUT -p tcp --dport 2049 -j ACCEPT
iptables -I INPUT -p udp --dport 2049 -j ACCEPT
iptables -I INPUT -p tcp --dport 20048:20050 -j ACCEPT
iptables -I INPUT -p udp --dport 20048:20050 -j ACCEPT
# convert2nfs
iptables -I INPUT -p tcp --dport 8080 -j ACCEPT
# rsync 
iptables -I INPUT -p tcp --dport 873 -j ACCEPT
iptables -I INPUT -p udp --dport 873 -j ACCEPT
# ldap
iptables -I INPUT -p tcp --dport 3389 -j ACCEPT
iptables -I INPUT -p tcp --dport 389 -j ACCEPT
# mysql 
iptables -I INPUT -p tcp --dport 3306 -j ACCEPT
# harbor
iptables -I INPUT -p tcp --dport 80 -j ACCEPT
# keepalived
iptables -I INPUT -p vrrp -j ACCEPT
```

### LoadBalance节点iptables白名单

```bash
# sync with haproxy.cfg
iptables -I INPUT -p tcp --dport 9000 -j ACCEPT
iptables -I INPUT -p tcp --dport 8443 -j ACCEPT
iptables -I INPUT -p tcp --dport 80 -j ACCEPT
iptables -I INPUT -p tcp --dport 443 -j ACCEPT
iptables -I INPUT -p tcp --dport 1936 -j ACCEPT
iptables -I INPUT -p tcp --dport 30000:32767 -j ACCEPT
iptables -I INPUT -p vrrp -j ACCEPT
```

### Openshift Master节点白名单

```bash
iptables -I INPUT -p tcp --dport 80 -j ACCEPT
iptables -I INPUT -p tcp --dport 443 -j ACCEPT
iptables -I INPUT -p tcp --dport 1936 -j ACCEPT
iptables -I INPUT -p tcp --dport 8443 -j ACCEPT
iptables -I INPUT -p tcp --dport 8053 -j ACCEPT
iptables -I INPUT -p udp --dport 8053 -j ACCEPT
iptables -I INPUT -p tcp --dport 10250 -j ACCEPT
iptables -I INPUT -p tcp --dport 8080 -j ACCEPT
iptables -I INPUT -p tcp --dport 9125 -j ACCEPT
iptables -I INPUT -p tcp --dport 30000:32767 -j ACCEPT
```

### Openshift Node节点端口

```bash
iptables -I INPUT -p tcp --dport 9125 -j ACCEPT
iptables -I INPUT -p tcp --dport 30000:32767 -j ACCEPT
```


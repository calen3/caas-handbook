# haproxy,keepalived,rsync

haproxy keepalived rsync 安装

## 安装步骤

> 生成haproxy配置模版

```bash
cat > haproxy.cfg << EOF

global
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

defaults
    mode             http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    #option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

listen mysql
    bind $CAAS_VIP_MYSQL_LDAP:3306
    mode tcp 
    server mysql01 {{ master }}:3306 check port 3306  
    server mysql02 {{ slave }}:3306 check port 3306 backup

listen ldap
    bind :389
    mode tcp
    server ldap1 {{ master }}:3389 check port 3389 
    server ldap2 {{ slave }}:3389 check port 3389 backup

EOF
```

> 安装并配置haproxy

```bash
cat > haproxy.yaml << EOF
---
- hosts: storages
  vars: 
    master: "{{ groups.storages[0] }}"
    slave: "{{ groups.storages[1] }}"
  tasks:
    - name: set ip_nonlocal_bind
      shell: echo "net.ipv4.ip_nonlocal_bind=1" >> /etc/sysctl.conf && sysctl -p
    - name: install haproxy 
      yum: name=haproxy state=installed
    - name: copy haprxoy config 
      template: src=./haproxy.cfg dest=/etc/haproxy/haproxy.cfg force=true
    - name: enable and start haproxy
      service: name=haproxy state=started enabled=true

EOF

ansible-playbook -i ./ansible_hosts --ssh-common-args "-o StrictHostKeyChecking=no" ./haproxy.yaml
```

> 安装并配置KeepAlived && rsync

```bash
cat > keepalived-rsync.yaml << EOF

---
- hosts: storages
  tasks:
    - name: copy install script 
      copy: src=../caas/nfs/keepalive-rsync/ dest=/opt/ force=true mode=0755
    - name: install keepalived
      yum: 
        name: "{{ item }}"
        state: present
      with_items:
        - keepalived
        - rsync
    - name: clear /etc/keepalived/keepalived.conf
      shell: echo '! Configuration File for keepalived' > /etc/keepalived/keepalived.conf

- hosts: "{{ groups.storages[0] }}"
  tasks:
    - name: config haproxy vip
      shell: /opt/keepalive-ha.sh haproxy master {{ groups.storages[0] }} {{ groups.storages[1] }} $CAAS_VIP_MYSQL_LDAP
    - name: install nfs vip
      shell: /opt/keepalive-ha.sh nfs master {{ groups.storages[0] }} {{ groups.storages[1] }} $CAAS_VIP_NFS
    - name: install harbor vip
      shell: /opt/keepalive-ha.sh harbor master {{ groups.storages[0] }} {{ groups.storages[1] }} $CAAS_VIP_HARBOR
    - name: install rsync
      shell: /opt/rsync-ha.sh master {{ groups.storages[0] }} {{ groups.storages[1] }} $CAAS_VIP_HARBOR /nfs /caas_data/harbor_data

- hosts: "{{ groups.storages[1] }}"
  tasks:
    - name: config haproxy vip
      shell: /opt/keepalive-ha.sh haproxy backup {{ groups.storages[0] }} {{ groups.storages[1] }} $CAAS_VIP_MYSQL_LDAP
    - name: install nfs vip
      shell: /opt/keepalive-ha.sh nfs backup {{ groups.storages[0] }} {{ groups.storages[1] }} $CAAS_VIP_NFS
    - name: install harbor vip
      shell: /opt/keepalive-ha.sh harbor backup {{ groups.storages[0] }} {{ groups.storages[1] }} $CAAS_VIP_HARBOR
    - name: install rsync
      shell: /opt/rsync-ha.sh backup {{ groups.storages[0] }} {{ groups.storages[1] }} $CAAS_VIP_HARBOR /nfs /caas_data/harbor_data

EOF

ansible-playbook -i ./ansible_hosts --ssh-common-args "-o StrictHostKeyChecking=no" ./keepalived-rsync.yaml
```

## 验证 {#验证}


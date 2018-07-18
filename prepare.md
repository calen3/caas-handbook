# prepare

安装caas 平台提前需要准备的工具

## 安装离线包

### 配置离线安装包

在CAAS\_HOST\_MASTER1 上 ，通过下面命令找一块分区，分区大小必须 &gt; 50G， 若没有大于50G的分区，请联系客户或相关人员，增加盘或者划分分区。（分区方法可参照下一节docker-&gt;docker配置-&gt;分区示例）

**注意：**此处若新建分区，请务必将分区挂载写入/etc/fstab文件，防止出现主机重启后离线安装包目录未挂载的情况

> 请执行以下命令 获得&gt;50G 的分区目录，若**未找到**，请在找相关人员完成分区后，**再次**执行以下命令

```bash
validdata=$(df -m |sed 1d |sort -rn -k2 |awk '{if($2>50000) print $6}'  | head -1)
echo $validdata
if [ "$validdata" == "" ]; then
   echo "没有找到>50G的分区，无法执行后续的操作，请联系相关人员 ..."
else
   echo "找到>50G的分区目录: $validdata"
   offlinedata=$validdata
   echo "export offlinedata=$validdata" >> ~/.bashrc
   mkdir -p $offlinedata
fi
```

> 将caas-offline.tar.gz离线包上传到到 CAAS\_MASTER1 机器的目录$offlinedata下

解压离线文件

```bash
tar zxvf $offlinedata/caas-offline.tar.gz -C $offlinedata
```

```bash
cd $offlinedata/caas-offline/cent7.2

# 启动 http 服务，搭建caas的yum源

nohup python -m SimpleHTTPServer 38888 &

# 配置iptables 规则，其实能访问
iptables -I INPUT -p tcp  --dport 38888 -j ACCEPT
```

## master1配置ansible

> 配置本地yum 源

```bash
cd /etc/yum.repos.d
mkdir back
mv *.repo back/

cat > caas.repo << EOF

[caas]
name=caas-offline-local
failovermethod=priority
baseurl=http://$CAAS_HOST_MASTER1:38888/
#mirrorlist=http://mirrorlist.centos.org/?release=&arch=&repo=os
gpgcheck=0
#gpgkey=http://mirrors.haihangyun.com/centos/RPM-GPG-KEY-CentOS-7
EOF

cp caas.repo /tmp/caas.repo
```

> 配置ansible 配置文件

```bash
cd $offlinedata/caas-offline
mkdir -p install
cd  install

env |grep CAAS_HOST_MASTER |awk -F '=' '{if ($2!="") { split(tolower($1),arrays, "_"); print $2" hostname="arrays[3]}}' > /tmp/masters
env |grep CAAS_HOST_NODE |awk -F '=' '{if ($2!="") { split(tolower($1),arrays, "_"); print $2" hostname="arrays[3]}}' > /tmp/nodes
env |grep CAAS_HOST_LB |awk -F '=' '{if ($2!="") { split(tolower($1),arrays, "_"); print $2" hostname="arrays[3]}}' > /tmp/lbs
env |grep CAAS_HOST_STORAGE |awk -F '=' '{if ($2!="") { split(tolower($1),arrays, "_"); print $2" hostname="arrays[3]}}' > /tmp/storage

cat > ansible_hosts <<EOF
[dockers:children]
masters
nodes
storages
lbs
[masters]
$(cat /tmp/masters)
[nodes]
$(cat /tmp/nodes)
[lbs]
$(cat /tmp/lbs)
[storages]
$(cat /tmp/storage)
EOF
```

> 创建hosts 附加文件

```bash
env |grep CAAS_HOST_MASTER |awk -F '=' '{if ($2!="") { split(tolower($1),arrays, "_"); print $2" "arrays[3]}}' > extra_hosts
env |grep CAAS_HOST_NODE |awk -F '=' '{if ($2!="") { split(tolower($1),arrays, "_"); print $2" "arrays[3]}}' >> extra_hosts
env |grep CAAS_HOST_LB |awk -F '=' '{if ($2!="") { split(tolower($1),arrays, "_"); print $2" "arrays[3]}}' >> extra_hosts
env |grep CAAS_HOST_STORAGE |awk -F '=' '{if ($2!="") { split(tolower($1),arrays, "_"); print $2" "arrays[3]}}' >> extra_hosts
harbor_host="`env|grep CAAS_VIP_HARBOR|awk -F= '{print $2}'` `env|grep CAAS_DOMAIN_HARBOR |awk -F= '{print $2}'`"
ldap_host="`env|grep CAAS_VIP_MYSQL_LDAP|awk -F= '{print $2}'` `env|grep CAAS_DOMAIN_LDAP |awk -F= '{print $2}'`"
grep "$harbor_host" ./extra_hosts || echo $harbor_host >> ./extra_hosts
grep "$ldap_host" ./extra_hosts || echo $ldap_host >> ./extra_hosts
echo "$CAAS_VIP_LOADBALANCE $CAAS_DOMAIN_OS_CONSOLE" >> ./extra_hosts
```

> 查看文件

```bash
#检查ansible_hosts文件是否正确
cat ansible_hosts
#检查extra_hosts文件是否正确
cat extra_hosts
```

> 安装ansible

```bash
yum install ansible -y
```

## 免密登录

> 在CAAS\_HOST\_MASTER1上执行如下命令,生成当前主机的密钥

```bash
if [ ! -f ~/.ssh/id_rsa.pub ]; then
    ssh-keygen -t rsa -b 1024 -C "root"
fi
```

> copy 本机密钥到所有主机上，下面提供两种方式：

**方式1：（当所有主机密码相同时，可使用此方法，否则使用方式2）**

> 设置主机密码

```text
SSH_PASSWD=(此处输入主机密码)
```

```bash
cat > ssh_key_copy.yaml << EOF
---
- hosts: all
  vars:
    ansible_ssh_user: root
    ansible_ssh_pass: $SSH_PASSWD
  tasks:
    - name: copy ssh key to all host
      authorized_key: user=root key="{{ lookup('file', '/root/.ssh/id_rsa.pub') }}"
EOF

ansible-playbook -i ./hosts --ssh-common-args "-o StrictHostKeyChecking=no" ./ssh_key_copy.yaml
```

**方式2：（此方法中，需手动输入所有主机的密码）**

```bash
hosts=$(env |grep CAAS_HOST_ |awk -F '=' '{print $2}')

for h in $hosts; do
    ssh-copy-id root@$h
done
```

## Selinux Check

> 检查所有master和node节点的selinux, **当下面命令运行出现错误时，请手动重启所有master和node节点, 之后再次运行该命令，确保该修改成功**

```bash
cat > selinux-check.yaml << EOF

---
- hosts: masters,nodes
  tasks:
    - name: enable selinux Persist
      shell: echo "SELINUX=enforcing" > /etc/selinux/config
    - name: enable selinux Persist
      shell: echo "SELINUXTYPE=targeted" >> /etc/selinux/config
    - name: check selinux
      shell: setenforce 1
      register: result
      failed_when: "result.rc != 0 or 'SELinux is disabled' in result.stderr"

EOF

ansible-playbook -i ./ansible_hosts --ssh-common-args "-o StrictHostKeyChecking=no" ./selinux-check.yaml
```

**若上边操作重启了master1节点，请在master1上执行以下命令，否则不必执行**

```bash
cd $offlinedata/caas-offline/cent7.2

# 重新启动 http 服务，搭建caas的yum源

nohup python -m SimpleHTTPServer 38888 &
iptables -I INPUT -p tcp --dport 38888 -j ACCEPT

# 进入caas安装目录
cd $offlinedata/caas-offline/install
```

## 主机环境准备

> 生成prepare配置文件， 进行环境的一些准备工作

```bash
cat > prepare.yaml << EOF

---
- hosts: all
  tasks:
    - name: bak prepare yum repo - dir create
      file: path=/etc/yum.repos.d/caas_bak state=directory
    - name: bak prepare yum repo - bak repo
      shell: mv -f /etc/yum.repos.d/*.repo /etc/yum.repos.d/caas_bak/
    - name: copy caas repo to all host
      copy: src=/tmp/caas.repo dest=/etc/yum.repos.d/ force=true
    - name: yum update
      shell: yum clean all && yum makecache

    - name: base packages install
      yum: 
        name: "{{ item }}"
        state: present
      with_items:
        - wget 
        - git 
        - net-tools 
        - bind-utils 
        - yum-utils 
        - iptables-services 
        - bridge-utils 
        - bash-completion 
        - kexec-tools 
        - sos 
        - psacct
        - PyYAML
        - python-ipaddress

    - name: copy caas host resolve to all hosts
      copy: src=./extra_hosts dest=/tmp/extra_hosts force=true
    - name: add extra host
      shell: cat /tmp/extra_hosts >> /etc/hosts

    - name: set host names
      shell: hostnamectl set-hostname {{ hostname }}

    - name: disable firewalld
      service: name=firewalld state=stopped enabled=no

- hosts: storages
  tasks:
    - name: Disable selinux
      shell: setenforce 0
      register: result
      failed_when: "result.rc != 0 and 'SELinux is disabled' not in result.stderr"
    - name: Disable selinux Persist
      shell: echo "SELINUX=disabled" > /etc/selinux/config
    - name: Disable selinux Persist
      shell: echo "SELINUXTYPE=targeted" >> /etc/selinux/config

- hosts: masters 
  tasks:
    - name: install java 
      yum: name=java state=present

EOF

ansible-playbook -i ./ansible_hosts --ssh-common-args "-o StrictHostKeyChecking=no" ./prepare.yaml
```


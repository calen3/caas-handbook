# prepare

安装caas 平台提前需要准备的工具

### 免密登录

> 在CAAS\_HOST\_MASTER1上执行如下命令,对所有主机免密登录

```text
hosts=$(env |grep CAAS_HOST_ |awk -F '=' '{print $2}')

if [ ! -f ~/.ssh/id_rsa.pub ]; then
ssh-keygen -t rsa -b 1024 -C "root"
fi
```

> copy 秘钥

```text
for h in $hosts; do
ssh-copy-id root@$h
done
```

### 安 装离线包

#### 配置离线安装包

> 在 CAAS\_HOST\_MASTER1 上 ，找一块分区，分区大小必须 &gt; 50G， 若没有大于50G的分区，请联系客户或相关人员，增加盘或者划分分区
>
> 分区查看命令 （Avail\)

```text
df -h


Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root   36G  2.3G   34G   7% /
devtmpfs                 7.8G     0  7.8G   0% /dev
tmpfs                    7.8G     0  7.8G   0% /dev/shm
tmpfs                    7.8G  8.6M  7.8G   1% /run
tmpfs                    7.8G     0  7.8G   0% /sys/fs/cgroup
/dev/sda1                497M  124M  373M  25% /boot
tmpfs                    1.6G     0  1.6G   0% /run/user/0
/dev/sdc1                100G   33M  100G   1% /deploydata
```

> 请执行以下命令 获得&gt;50G 的分区目录

```text
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




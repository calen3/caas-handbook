# nfs

nfs 安装

## 安装步骤

> 生成nfs配置文件

```bash
cat > config.py << EOF
role = {
    "identity": "{{ nfs_role }}", 
    "port": "8080",
    "master_ip": "$CAAS_HOST_STORAGE1",
    "slave_ip": "$CAAS_HOST_STORAGE2",
}

EOF
```

> 安装nfs

```bash
cat > nfs.yaml << EOF

---
- hosts: storages
  tasks:
    - name: install the required yums for nfs
      yum:
        name: "{{ item }}"
        state: present
      with_items:
        - nfs-utils
        - rpcbind
        - python-devel
        - lvm2
        - unzip 

    - name: set the port for nfs
      shell: echo  "MOUNTD_PORT=20048" >> /etc/sysconfig/nfs && echo  "STATD_PORT=20050" >> /etc/sysconfig/nfs && echo "options lockd nlm_tcpport=20049" >> /etc/modprobe.d/lockd.conf && echo "options lockd nlm_udpport=20049" >> /etc/modprobe.d/lockd.conf

    - name: start servce nfs and rpcbind
      systemd:
        name: "{{ item }}"
        state: started
        enabled: True
      with_items:
        - nfs
        - rpcbind

    - name: create the directory /nfs
      file:
        path: /nfs
        state: directory

    - name: unarchive the nfsenv and convert2nfs to /opt/
      unarchive:
        src: "{{ item }}"
        dest: /opt/
      with_items:
        - "../caas/nfs/convert2nfs.zip"
        - "../caas/nfs/rsync.tar.gz" 

- hosts: "{{ groups.storages[0] }}"
  vars: 
    nfs_role: master
  tasks:
    - template: 
        src: config.py
        dest: /opt/convert2nfs/convert2nfs/controllers/config.py

- hosts: "{{ groups.storages[1] }}"
  vars: 
    nfs_role: slave 
  tasks:
    - template: 
        src: config.py
        dest: /opt/convert2nfs/convert2nfs/controllers/config.py

- hosts: storages
  tasks: 
    - name: install the python interface for nfs
      shell: source /opt/rsync2nfs/bin/activate && cd /opt/convert2nfs && chmod +x bin/* &&  python setup.py install 
    - name: start the nfs interface 
      shell: source /opt/rsync2nfs/bin/activate && cd /opt/convert2nfs &&  uwsgi -d /var/log/convert2nfs.log --http-socket :8080 --venv /opt/rsync2nfs --pecan config.py
EOF

ansible-playbook -i ./ansible_hosts --ssh-common-args "-o StrictHostKeyChecking=no" ./nfs.yaml
```

## 验证


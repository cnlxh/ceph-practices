| 编号  | 系统         | 角色                    | IP             | 主机名              | Ceph版本 |
| --- | ---------- | --------------------- | -------------- | ---------------- | ------ |
| 1   | Centos 8.5 | bootstrap，mon，mgr，osd | 192.168.30.130 | host1.xiaohui.cn | Quincy |
| 2   | Centos 8.5 | mon，mgr，osd，rgw，mds   | 192.168.30.131 | host2.xiaohui.cn | Quincy |
| 3   | Centos 8.5 | mon，mgr，rgw，mds       | 192.168.30.132 | host3.xiaohui.cn | Quincy |

作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. QQ：939958092

3. 邮箱：[xiaohui_li@foxmail.com](mailto:xiaohui_li@foxmail.com)

# 先决条件准备

除非另有说明，不然先决条件需在每个参与节点准备

## 准备域名解析

```bash
cat >> /etc/hosts <<EOF
192.168.30.130 host1.xiaohui.cn host1
192.168.30.131 host2.xiaohui.cn host2
192.168.30.132 host3.xiaohui.cn host3
EOF
```

## 依赖包

1. Python 3
2. Systemd
3. Podman 3
4. Chrony
5. LVM2

```bash
yum install podman chrony lvm2 systemd python3 -y
reboot
```

## 为ceph准备仓库

```ini
cat > /etc/yum.repos.d/cephadm.repo << eof
[cephadm]
name=cephadm lixiaohui write
baseurl=http://mirrors.aliyun.com/centos/8-stream/storage/x86_64/ceph-quincy
enabled=1
gpgcheck=0
eof
```

安装cephadm

```bash
yum install cephadm -y
```

# 引导新集群

本操作将创建 Ceph 集群的第一个"Mon守护程序”，后期Ceph 会随着集群的增长自动部署Mon守护程序，Ceph 会在集群收缩时自动缩减Mon守护进程。此自动增长和收缩的顺利执行取决于正确的子网配置。如果集群中的所有 ceph Mon守护程序都位于同一子网中，则无需手动管理 ceph Mon守护程序。 将根据需要自动向子网添加多个Mon，因为新主机已添加到群集

要将 Ceph 集群配置为在单个主机上运行，请在引导时使用--single-host-defaults标志

```bash
[root@host1 ~]# cephadm bootstrap --mon-ip 192.168.30.130 --allow-fqdn-hostname
Creating directory /etc/ceph for ceph.conf
Verifying podman|docker is present...
Verifying lvm2 is present...
Verifying time synchronization is in place...
Unit chronyd.service is enabled and running
Repeating the final host check...
podman (/usr/bin/podman) version 4.2.0 is present
systemctl is present
lvcreate is present
Unit chronyd.service is enabled and running
Host looks OK
Cluster fsid: 40db1f1a-3333-11ed-b7f6-000c29759605
Verifying IP 192.168.30.130 port 3300 ...
Verifying IP 192.168.30.130 port 6789 ...
Mon IP `192.168.30.130` is in CIDR network `192.168.30.0/24`
Mon IP `192.168.30.130` is in CIDR network `192.168.30.0/24`
Internal network (--cluster-network) has not been provided, OSD replication will default to the public_network
Pulling container image quay.io/ceph/ceph:v17...
Ceph version: ceph version 17.2.3 (dff484dfc9e19a9819f375586300b3b79d80034d) quincy (stable)
Extracting ceph user uid/gid from container image...
Creating initial keys...
Creating initial monmap...
Creating mon...
firewalld ready
Enabling firewalld service ceph-mon in current zone...
Waiting for mon to start...
Waiting for mon...
mon is available
Assimilating anything we can from ceph.conf...
Generating new minimal ceph.conf...
Restarting the monitor...
Setting mon public_network to 192.168.30.0/24
Wrote config to /etc/ceph/ceph.conf
Wrote keyring to /etc/ceph/ceph.client.admin.keyring
Creating mgr...
Verifying port 9283 ...
firewalld ready
Enabling firewalld service ceph in current zone...
firewalld ready
Enabling firewalld port 9283/tcp in current zone...
Waiting for mgr to start...
Waiting for mgr...
mgr not available, waiting (1/15)...
mgr not available, waiting (2/15)...
mgr is available
Enabling cephadm module...
Waiting for the mgr to restart...
Waiting for mgr epoch 5...
mgr epoch 5 is available
Setting orchestrator backend to cephadm...
Generating ssh key...
Wrote public SSH key to /etc/ceph/ceph.pub
Adding key to root@localhost authorized_keys...
Adding host host1.xiaohui.cn...
Deploying mon service with default placement...
Deploying mgr service with default placement...
Deploying crash service with default placement...
Deploying prometheus service with default placement...
Deploying grafana service with default placement...
Deploying node-exporter service with default placement...
Deploying alertmanager service with default placement...
Enabling the dashboard module...
Waiting for the mgr to restart...
Waiting for mgr epoch 9...
mgr epoch 9 is available
Generating a dashboard self-signed certificate...
Creating initial admin user...
Fetching dashboard port number...
firewalld ready
Enabling firewalld port 8443/tcp in current zone...
Ceph Dashboard is now available at:

             URL: https://mta60.brightcolors.net:8443/
            User: admin
        Password: c44pz18shw

Enabling client.admin keyring and conf on hosts with "admin" label
Saving cluster configuration to /var/lib/ceph/40db1f1a-3333-11ed-b7f6-000c29759605/config directory
Enabling autotune for osd_memory_target
You can access the Ceph CLI as following in case of multi-cluster or non-default config:

        sudo /usr/sbin/cephadm shell --fsid 40db1f1a-3333-11ed-b7f6-000c29759605 -c /etc/ceph/ceph.conf -k /etc/ceph/ceph.client.admin.keyring

Or, if you are only running a single cluster on this host:

        sudo /usr/sbin/cephadm shell 

Please consider enabling telemetry to help improve Ceph:

        ceph telemetry on

For more information see:

        https://docs.ceph.com/docs/master/mgr/telemetry/

Bootstrap complete.
```

上述命令将：

1. 为本地主机上的新群集创建Mon和Mgr守护程序。

2. 为 Ceph 集群生成新的 SSH 密钥，并将其添加到root用户的文件: /root/.ssh/authorized_keys

3. 将公钥的副本写入: /etc/ceph/ceph.pub

4. 将最小配置文件写入/etc/ceph/ceph.conf，需要此文件才能与新群集进行通信

5. 将管理（特权 client.admin） 密钥的副本写入: /etc/ceph/ceph.client.admin.keyring

6. 将_admin标签添加到引导主机，默认情况下，任何具有此标签的主机都将（也）获得/etc/ceph/ceph.conf和/etc/ceph/ceph.client.admin.keyring的副本

目前我们需要继续完成集群，需要先扩展集群，因为我们只有一个mon节点，没有osd等其他节点，集群状态为HEALTH_WARN无法开始工作

```bash
[root@host1 ~]# cephadm shell -- ceph -s
Inferring fsid a13e967e-07fb-11ed-920e-000c29759605
Inferring config /var/lib/ceph/a13e967e-07fb-11ed-920e-000c29759605/mon.host1.xiaohui.cn/config
Using ceph image with id 'e5af760fa1c1' and tag 'v17' created on 2022-06-23 19:49:45 +0000 UTC
quay.io/ceph/ceph@sha256:d3f3e1b59a304a280a3a81641ca730982da141dad41e942631e4c5d88711a66b
  cluster:
    id:     a13e967e-07fb-11ed-920e-000c29759605
    health: HEALTH_WARN
            OSD count 0 < osd_pool_default_size 3

  services:
    mon: 1 daemons, quorum host1.xiaohui.cn (age 16m)
    mgr: host1.xiaohui.cn.oslofw(active, since 14m)
    osd: 0 osds: 0 up, 0 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:
```

所有的ceph命令都需要执行cephadm shell进入容器之后才可以执行，不然会报告没有此命令，而且记得进入之后创建的所有文件在退出后会全部丢失，也可以用cephadm shell --mount挂载本地文件进去

# 添加存储节点

显示在集群中所有的存储设备

```bash
[ceph: root@host1 /]# ceph orch device ls
HOST              PATH          TYPE  DEVICE ID                                   SIZE  AVAILABLE  REFRESHED  REJECT REASONS
host1.xiaohui.cn  /dev/nvme0n2  ssd   VMware_Virtual_NVMe_Disk_VMware_NVME_0000   214G  Yes        15m ago
host1.xiaohui.cn  /dev/nvme0n3  ssd   VMware_Virtual_NVMe_Disk_VMware_NVME_0000   214G  Yes        15m ago
host1.xiaohui.cn  /dev/nvme0n4  ssd   VMware_Virtual_NVMe_Disk_VMware_NVME_0000   214G  Yes        15m ago
```

添加单独的设备到集群

```bash
[ceph: root@host1 /]# ceph orch daemon add osd host1.xiaohui.cn:/dev/nvme0n2
Created osd(s) 0 on host 'host1.xiaohui.cn'
[ceph: root@host1 /]# ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME       STATUS  REWEIGHT  PRI-AFF
-1         0.19530  root default
-3         0.19530      host host1
 0    ssd  0.19530          osd.0       up   1.00000  1.00000
[ceph: root@host1 /]# ceph osd ls
0
```

将集群中所有未使用的存储设备添加进来

```bash
[ceph: root@host1 /]# ceph orch apply osd --all-available-devices
Scheduled osd.all-available-devices update...
[ceph: root@host1 /]# ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME       STATUS  REWEIGHT  PRI-AFF
-1         0.19530  root default
-3         0.19530      host host1
 0    ssd  0.19530          osd.0       up   1.00000  1.00000
 1               0  osd.1             down   1.00000  1.00000
 2               0  osd.2             down   1.00000  1.00000
[ceph: root@host1 /]# ceph osd ls
0
1
2
```

# 添加主机到集群

## 先决条件

需要注意，新主机必须满足此文章的先决条件

## 添加密钥

添加密钥到目标主机以实现免密

```bash
ceph cephadm get-pub-key > ceph.pub
ssh-copy-id -f -i ceph.pub root@host1
ssh-copy-id -f -i ceph.pub root@host2
ssh-copy-id -f -i ceph.pub root@host3
```

## 添加主机到集群

在此过程中，将会自动扩展mon和mgr节点，以及自动应用其未使用的osd设备，可使用ceph orch host rm删除主机，--offline --force参数可以用于已经脱机的删除场景，可以用--labels=my_label1,my_label2的方式添加标签

```bash
[ceph: root@host1 /]# ceph orch host add host2.xiaohui.cn 192.168.30.131
Added host 'host2.xiaohui.cn' with addr '192.168.30.131'
[ceph: root@host1 /]# ceph orch host add host3.xiaohui.cn 192.168.30.132
Added host 'host3.xiaohui.cn' with addr '192.168.30.132'
[ceph: root@host1 /]# ceph orch host ls
HOST              ADDR            LABELS  STATUS
host1.xiaohui.cn  192.168.30.130  _admin
host2.xiaohui.cn  192.168.30.131
host3.xiaohui.cn  192.168.30.132
3 hosts in cluster
```

## 验证服务的自动扩展

可以看到mon、mgr、osd等已经自动做了扩展

```bash
[ceph: root@host1 /]# ceph orch ps
NAME                         HOST              PORTS        STATUS          REFRESHED   AGE  MEM USE  MEM LIM  VERSION  IMAGE ID      CONTAINER ID  
alertmanager.host1           host1.xiaohui.cn  *:9093,9094  running (39s)      4s ago   35m    18.4M        -           ba2b418f427c  d768d12ac07d  
crash.host1                  host1.xiaohui.cn               running (34m)      4s ago   34m    6958k        -  17.2.3   0912465dcea5  8e04b9cfeb5d  
crash.host2                  host2.xiaohui.cn               running (5m)       6s ago    5m    7159k        -  17.2.3   0912465dcea5  ad46d42eeff3  
crash.host3                  host3.xiaohui.cn               running (2m)       6s ago    2m    7151k        -  17.2.3   0912465dcea5  6184ee3db390  
grafana.host1                host1.xiaohui.cn  *:3000       running (33m)      4s ago   34m    59.4M        -  8.3.5    dad864ee21e9  727f92c7e6cb  
mgr.host1.xiaohui.cn.fhdtnc  host1.xiaohui.cn  *:9283       running (36m)      4s ago   36m     492M        -  17.2.3   0912465dcea5  966c28044b0b  
mgr.host2.amijwr             host2.xiaohui.cn  *:8443,9283  running (5m)       6s ago    5m     425M        -  17.2.3   0912465dcea5  b1ac73668b88  
mon.host1.xiaohui.cn         host1.xiaohui.cn               running (36m)      4s ago   36m    66.3M    2048M  17.2.3   0912465dcea5  7e6093c6a960  
mon.host2                    host2.xiaohui.cn               running (5m)       6s ago    5m    49.6M    2048M  17.2.3   0912465dcea5  d7aa98a44dda  
mon.host3                    host3.xiaohui.cn               running (2m)       6s ago    2m    36.9M    2048M  17.2.3   0912465dcea5  ef1e71e9fd5f  
node-exporter.host1          host1.xiaohui.cn  *:9100       running (33m)      4s ago   33m    24.3M        -           1dbe0e931976  c10f9505b6c2  
node-exporter.host2          host2.xiaohui.cn  *:9100       running (5m)       6s ago    5m    21.9M        -           1dbe0e931976  71d499d48ff7  
node-exporter.host3          host3.xiaohui.cn  *:9100       running (2m)       6s ago    2m    24.7M        -           1dbe0e931976  3ccfbb9629fc  
osd.0                        host1.xiaohui.cn               running (30m)      4s ago   30m    57.4M    4096M  17.2.3   0912465dcea5  eb521dccbf3c  
osd.1                        host1.xiaohui.cn               running (30m)      4s ago   30m    52.4M    4096M  17.2.3   0912465dcea5  35ca2f77e0d5  
osd.2                        host1.xiaohui.cn               running (30m)      4s ago   30m    53.9M    4096M  17.2.3   0912465dcea5  235e1724229e  
osd.3                        host1.xiaohui.cn               running (30m)      4s ago   30m    54.0M    4096M  17.2.3   0912465dcea5  fb1334dc0614  
osd.4                        host2.xiaohui.cn               running (5m)       6s ago    5m    46.1M    4096M  17.2.3   0912465dcea5  251f3d750824  
osd.5                        host2.xiaohui.cn               running (5m)       6s ago    5m    40.8M    4096M  17.2.3   0912465dcea5  32cba9ef4fa1  
osd.6                        host2.xiaohui.cn               running (4m)       6s ago    4m    53.0M    4096M  17.2.3   0912465dcea5  f376f33eba32  
osd.7                        host2.xiaohui.cn               running (4m)       6s ago    4m    50.3M    4096M  17.2.3   0912465dcea5  a9a7cf523e36  
osd.8                        host3.xiaohui.cn               running (60s)      6s ago   60s    39.1M    4096M  17.2.3   0912465dcea5  290727b476e2  
osd.9                        host3.xiaohui.cn               running (55s)      6s ago   55s    38.6M    4096M  17.2.3   0912465dcea5  3b6ce2e1897d  
osd.10                       host3.xiaohui.cn               running (100s)     6s ago  100s    41.6M    4096M  17.2.3   0912465dcea5  d5ea0d3d3a62  
osd.11                       host3.xiaohui.cn               running (95s)      6s ago   95s    43.2M    4096M  17.2.3   0912465dcea5  5176e0ca0d1c  
prometheus.host1             host1.xiaohui.cn  *:9095       running (34s)      4s ago   33m    53.7M        -           514e6a882f6e  f9144007d39f   
```

## 分配服务到主机

```bash
ceph orch apply rgw host2.xiaohui.cn
Scheduled rgw.host2.xiaohui.cn update...
ceph orch apply mds host2.xiaohui.cn
Scheduled mds.host2.xiaohui.cn update...
ceph orch apply mds host3.xiaohui.cn
Scheduled mds.host3.xiaohui.cn update...
ceph orch apply rgw host3.xiaohui.cn
Scheduled rgw.host3.xiaohui.cn update...
```

## 验证服务分配效果

```bash
[ceph: root@host1 /]# ceph orch ps
NAME                               HOST              PORTS        STATUS         REFRESHED  AGE  MEM USE  MEM LIM  VERSION  IMAGE ID      CONTAINER ID  
alertmanager.host1                 host1.xiaohui.cn  *:9093,9094  running (9m)      2m ago  43m    28.4M        -           ba2b418f427c  d768d12ac07d  
crash.host1                        host1.xiaohui.cn               running (43m)     2m ago  43m    6958k        -  17.2.3   0912465dcea5  8e04b9cfeb5d  
crash.host2                        host2.xiaohui.cn               running (14m)     2m ago  14m    7142k        -  17.2.3   0912465dcea5  ad46d42eeff3  
crash.host3                        host3.xiaohui.cn               running (10m)     3m ago  10m    7151k        -  17.2.3   0912465dcea5  6184ee3db390  
grafana.host1                      host1.xiaohui.cn  *:3000       running (41m)     2m ago  42m    63.7M        -  8.3.5    dad864ee21e9  727f92c7e6cb  
mds.host2.xiaohui.cn.host1.gjicjk  host1.xiaohui.cn               running (2m)      2m ago   2m    23.5M        -  17.2.3   0912465dcea5  885cebd2ffcb  
mds.host2.xiaohui.cn.host2.hqqzet  host2.xiaohui.cn               running (2m)      2m ago   2m    12.2M        -  17.2.3   0912465dcea5  0986b9c7d8c4  
mds.host3.xiaohui.cn.host1.hktjak  host1.xiaohui.cn               running (2m)      2m ago   2m    12.6M        -  17.2.3   0912465dcea5  8cc1a09fb0f5  
mds.host3.xiaohui.cn.host2.fmeopq  host2.xiaohui.cn               running (2m)      2m ago   2m    13.4M        -  17.2.3   0912465dcea5  0c48cee66f6a  
mgr.host1.xiaohui.cn.fhdtnc        host1.xiaohui.cn  *:9283       running (44m)     2m ago  44m     501M        -  17.2.3   0912465dcea5  966c28044b0b  
mgr.host2.amijwr                   host2.xiaohui.cn  *:8443,9283  running (14m)     2m ago  14m     427M        -  17.2.3   0912465dcea5  b1ac73668b88  
mon.host1.xiaohui.cn               host1.xiaohui.cn               running (44m)     2m ago  44m    72.2M    2048M  17.2.3   0912465dcea5  7e6093c6a960  
mon.host2                          host2.xiaohui.cn               running (14m)     2m ago  14m    57.9M    2048M  17.2.3   0912465dcea5  d7aa98a44dda  
mon.host3                          host3.xiaohui.cn               running (10m)     3m ago  10m    38.6M    2048M  17.2.3   0912465dcea5  ef1e71e9fd5f  
node-exporter.host1                host1.xiaohui.cn  *:9100       running (42m)     2m ago  42m    27.7M        -           1dbe0e931976  c10f9505b6c2  
node-exporter.host2                host2.xiaohui.cn  *:9100       running (14m)     2m ago  14m    26.5M        -           1dbe0e931976  71d499d48ff7  
node-exporter.host3                host3.xiaohui.cn  *:9100       running (10m)     3m ago  10m    32.4M        -           1dbe0e931976  3ccfbb9629fc  
osd.0                              host1.xiaohui.cn               running (39m)     2m ago  39m    57.3M    4096M  17.2.3   0912465dcea5  eb521dccbf3c  
osd.1                              host1.xiaohui.cn               running (38m)     2m ago  38m    56.5M    4096M  17.2.3   0912465dcea5  35ca2f77e0d5  
osd.2                              host1.xiaohui.cn               running (38m)     2m ago  38m    54.3M    4096M  17.2.3   0912465dcea5  235e1724229e  
osd.3                              host1.xiaohui.cn               running (38m)     2m ago  38m    55.9M    4096M  17.2.3   0912465dcea5  fb1334dc0614  
osd.4                              host2.xiaohui.cn               running (13m)     2m ago  13m    48.7M    4096M  17.2.3   0912465dcea5  251f3d750824  
osd.5                              host2.xiaohui.cn               running (13m)     2m ago  13m    44.1M    4096M  17.2.3   0912465dcea5  32cba9ef4fa1  
osd.6                              host2.xiaohui.cn               running (13m)     2m ago  13m    59.5M    4096M  17.2.3   0912465dcea5  f376f33eba32  
osd.7                              host2.xiaohui.cn               running (13m)     2m ago  13m    54.6M    4096M  17.2.3   0912465dcea5  a9a7cf523e36  
osd.8                              host3.xiaohui.cn               running (9m)      3m ago   9m    42.2M    4096M  17.2.3   0912465dcea5  290727b476e2  
osd.9                              host3.xiaohui.cn               running (9m)      3m ago   9m    41.5M    4096M  17.2.3   0912465dcea5  3b6ce2e1897d  
osd.10                             host3.xiaohui.cn               running (10m)     3m ago  10m    44.6M    4096M  17.2.3   0912465dcea5  d5ea0d3d3a62  
osd.11                             host3.xiaohui.cn               running (10m)     3m ago  10m    46.8M    4096M  17.2.3   0912465dcea5  5176e0ca0d1c  
prometheus.host1                   host1.xiaohui.cn  *:9095       running (9m)      2m ago  41m    88.5M        -           514e6a882f6e  f9144007d39f  
rgw.host2.xiaohui.cn.host1.qvhrqr  host1.xiaohui.cn  *:80         running (2m)      2m ago   2m    86.0M        -  17.2.3   0912465dcea5  a0f2ef05120c  
rgw.host2.xiaohui.cn.host2.qmppew  host2.xiaohui.cn  *:80         running (2m)      2m ago   2m    85.6M        -  17.2.3   0912465dcea5  65aa6403eec8  
```

## 给机器打标签

除_admin的标签之外，其他的都是由管理员自定的随意字符串，这里我打了一个lixiaohuilabel的标签给host2

```bash
[ceph: root@host1 /]# ceph orch host label add host2.xiaohui.cn lixiaohuilabel 
Added label lixiaohuilabel to host host2.xiaohui.cn
[ceph: root@host1 /]# ceph orch host ls 
HOST              ADDR            LABELS          STATUS  
host1.xiaohui.cn  192.168.30.130  _admin                  
host2.xiaohui.cn  192.168.30.131  lixiaohuilabel          
host3.xiaohui.cn  192.168.30.132                          
3 hosts in cluster
```

# 设置管理节点

在host2还不是管理节点的情况下，执行管理命令如下

```bash
[ceph: root@host2 /]# ceph orch ls
2022-09-13T09:22:57.979+0000 7fb6e0b37700 -1 auth: unable to find a keyring on /etc/ceph/ceph.client.admin.keyring,/etc/ceph/ceph.keyring,/etc/ceph/keyring,/etc/ceph/keyring.bin: (2) No such file or directory
2022-09-13T09:22:57.979+0000 7fb6e0b37700 -1 AuthRegistry(0x7fb6dc063870) no keyring found at /etc/ceph/ceph.client.admin.keyring,/etc/ceph/ceph.keyring,/etc/ceph/keyring,/etc/ceph/keyring.bin, disabling cephx
2022-09-13T09:22:57.979+0000 7fb6e0b37700 -1 auth: unable to find a keyring on /etc/ceph/ceph.client.admin.keyring,/etc/ceph/ceph.keyring,/etc/ceph/keyring,/etc/ceph/keyring.bin: (2) No such file or directory
2022-09-13T09:22:57.979+0000 7fb6e0b37700 -1 AuthRegistry(0x7fb6e0b35e90) no keyring found at /etc/ceph/ceph.client.admin.keyring,/etc/ceph/ceph.keyring,/etc/ceph/keyring,/etc/ceph/keyring.bin, disabling cephx
2022-09-13T09:22:57.982+0000 7fb6da59c700 -1 monclient(hunting): handle_auth_bad_method server allowed_methods [2] but i only support [1]
2022-09-13T09:22:57.982+0000 7fb6dad9d700 -1 monclient(hunting): handle_auth_bad_method server allowed_methods [2] but i only support [1]
2022-09-13T09:22:57.982+0000 7fb6d9d9b700 -1 monclient(hunting): handle_auth_bad_method server allowed_methods [2] but i only support [1]
2022-09-13T09:22:57.982+0000 7fb6e0b37700 -1 monclient: authenticate NOTE: no keyring found; disabled cephx authentication
[errno 13] RADOS permission denied (error connecting to the cluster)
```

执行以下步骤配置admin节点：

1. 将_admin标签分配给节点

2. 将管理密钥复制到管理节点

3. 将ceph.conf文件复制到管理节点

```bash
[ceph: root@host1 /]# ceph orch host label add host2.xiaohui.cn _admin
[ceph: root@host1 /]# exit
[root@host1 ~]# scp /etc/ceph/ceph.client.admin.keyring /etc/ceph/ceph.client.admin.keyring root@host2:/etc/ceph
root@host2's password: 
ceph.client.admin.keyring                                                                                               100%  151   195.0KB/s   00:00    
ceph.client.admin.keyring                                                                                               100%  151   306.8KB/s   00:00    
```

再次执行管理命令如下

```bash
[ceph: root@host2 /]# ceph orch ps

NAME                               HOST              PORTS        STATUS         REFRESHED  AGE  MEM USE  MEM LIM  VERSION  IMAGE ID      CONTAINER ID  
alertmanager.host1                 host1.xiaohui.cn  *:9093,9094  running (6m)      4m ago  23m    18.3M        -           ba2b418f427c  754261bc879a  
crash.host1                        host1.xiaohui.cn               running (23m)     4m ago  23m    6953k        -  17.2.3   0912465dcea5  4f13168985cb  
crash.host2                        host2.xiaohui.cn               running (10m)     3m ago  10m    7151k        -  17.2.3   0912465dcea5  32825531e13a  
crash.host3                        host3.xiaohui.cn               running (7m)      5m ago   7m    7168k        -  17.2.3   0912465dcea5  e45c06e02210  
grafana.host1                      host1.xiaohui.cn  *:3000       running (20m)     4m ago  22m    61.0M        -  8.3.5    dad864ee21e9  6b66653aa85d  
mds.host3.xiaohui.cn.host1.gzwohl  host1.xiaohui.cn               running (9m)      4m ago   9m    15.5M        -  17.2.3   0912465dcea5  797b59445605  
mds.host3.xiaohui.cn.host2.lytooj  host2.xiaohui.cn               running (9m)      3m ago   9m    15.7M        -  17.2.3   0912465dcea5  f29fb005b1da  
mgr.host1.xiaohui.cn.jerjkg        host1.xiaohui.cn  *:9283       running (24m)     4m ago  24m     472M        -  17.2.3   0912465dcea5  e66a2a5f6233  
mgr.host2.avchoo                   host2.xiaohui.cn  *:8443,9283  running (10m)     3m ago  10m     426M        -  17.2.3   0912465dcea5  d6fc181f0220  
mon.host1.xiaohui.cn               host1.xiaohui.cn               running (24m)     4m ago  24m    52.8M    2048M  17.2.3   0912465dcea5  db58f6dd791e  
mon.host2                          host2.xiaohui.cn               running (9m)      3m ago   9m    37.4M    2048M  17.2.3   0912465dcea5  9ab94043efbe  
mon.host3                          host3.xiaohui.cn               running (7m)      5m ago   7m    35.7M    2048M  17.2.3   0912465dcea5  e4ef9d49053f  
node-exporter.host1                host1.xiaohui.cn  *:9100       running (22m)     4m ago  22m    27.0M        -           1dbe0e931976  0e269d122c98  
node-exporter.host2                host2.xiaohui.cn  *:9100       running (9m)      3m ago   9m    33.1M        -           1dbe0e931976  61e3c6238976  
node-exporter.host3                host3.xiaohui.cn  *:9100       running (7m)      5m ago   7m    29.2M        -           1dbe0e931976  b97cdfd34a2f  
prometheus.host1                   host1.xiaohui.cn  *:9095       running (6m)      4m ago  20m    56.8M        -           514e6a882f6e  622d04c7cd65  
rgw.host2.xiaohui.cn.host1.lycyay  host1.xiaohui.cn  *:80         running (4m)      4m ago   9m    18.4M        -  17.2.3   0912465dcea5  0b8848fb7d8c  
rgw.host2.xiaohui.cn.host2.izkkpp  host2.xiaohui.cn  *:80         running (4m)      3m ago   9m    18.4M        -  17.2.3   0912465dcea5  49eea547dadd  
rgw.host3.xiaohui.cn.host1.mwhlcr  host1.xiaohui.cn  *:80         running (7m)      4m ago   7m    17.1M        -  17.2.3   0912465dcea5  7e8919783e59  
rgw.host3.xiaohui.cn.host2.vtaepb  host2.xiaohui.cn  *:80         running (7m)      3m ago   7m    18.6M        -  17.2.3   0912465dcea5  2e3681cd82c0  
```

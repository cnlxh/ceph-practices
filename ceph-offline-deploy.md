```textfile
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. QQ：939958092

3. 邮箱：xiaohui_li@foxmail.com

```

| 角色     | IP            | 平台              | 备注       |
| ------ | ------------- | --------------- | -------- |
| harbor | 172.16.50.200 | CentOS 8 Stream |          |
| ceph   | 172.16.50.201 | CentOS 8 Stream | 3个500G硬盘 |

# 准备hosts

两个机器都需要

```bash
cat >> /etc/hosts <<EOF
172.16.50.200 registry.xiaohui.cn
172.16.50.201 ceph.xiaohui.cn
EOF
```

# 部署harbor

## 生成root证书信息

```bash
openssl genrsa -out /etc/pki/tls/private/selfsignroot.key 4096
openssl req -x509 -new -nodes -sha512 -days 3650 -subj "/C=CN/ST=Shanghai/L=Shanghai/O=Company/OU=SH/CN=Root" \
-key /etc/pki/tls/private/selfsignroot.key \
-out /etc/pki/ca-trust/source/anchors/selfsignroot.crt
```

## 生成服务器私钥以及证书请求文件

```bash
openssl genrsa -out /etc/pki/tls/private/registry.key 4096
openssl req -sha512 -new \
-subj "/C=CN/ST=Shanghai/L=Shanghai/O=Company/OU=SH/CN=xiaohui.cn" \
-key /etc/pki/tls/private/registry.key \
-out registry.csr
```

## 生成openssl cnf扩展文件

```bash
cat > certs.cnf << EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = registry.xiaohui.cn
EOF
```

## 签发证书

```bash
openssl x509 -req -in registry.csr \
-CA /etc/pki/ca-trust/source/anchors/selfsignroot.crt \
-CAkey /etc/pki/tls/private/selfsignroot.key -CAcreateserial \
-out /etc/pki/tls/certs/registry.crt \
-days 3650 -extensions v3_req -extfile certs.cnf
```

## 信任根证书

```bash
update-ca-trust
```

分发root证书给ceph

```bash
scp /etc/pki/ca-trust/source/anchors/selfsignroot.crt root@172.16.50.201:/etc/pki/ca-trust/source/anchors/selfsignroot.crt
```

## 部署Harbor仓库

```bash
yum install -y yum-utils
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo 
yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y 
systemctl enable docker --now
```

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["http://hub-mirror.c.163.com"]
}
EOF
curl -L "https://github.com/docker/compose/releases/download/v2.11.2/docker-compose-linux-x86_64" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
sudo systemctl daemon-reload
sudo systemctl restart docker
```

```bash
wget https://github.com/goharbor/harbor/releases/download/v2.6.1/harbor-offline-installer-v2.6.1.tgz
tar xf harbor-offline-installer-v2.6.1.tgz -C /usr/local/bin
cd /usr/local/bin/harbor
```

在harbor.yml中，修改以下参数，定义了网址、证书、密码

```bash
cp harbor.yml.tmpl harbor.yml
vim harbor.yml
hostname: registry.xiaohui.cn
  certificate: /etc/pki/tls/certs/registry.crt
  private_key: /etc/pki/tls/private/registry.key
harbor_admin_password: admin
```

```bash
./prepare
./install.sh
```

## 生成服务文件

```bash
cat > /etc/systemd/system/harbor.service <<EOF
[Unit]
Description=Harbor
After=docker.service systemd-networkd.service systemd-resolved.service
Requires=docker.service
Documentation=http://github.com/vmware/harbor
[Service]
Type=simple
Restart=on-failure
RestartSec=5
ExecStart=/usr/local/bin/docker-compose -f /usr/local/bin/harbor/docker-compose.yml up
ExecStop=/usr/local/bin/docker-compose -f /usr/local/bin/harbor/docker-compose.yml down
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
systemctl enable harbor --now
```

# 镜像同步准备

先下载必备镜像

```bash
for image in `grep .*IMAGE.*=.*: /usr/sbin/cephadm | cut -d "'" -f 2`;do docker pull $image;done
```

重命名镜像

```bash
docker tag quay.io/ceph/ceph:v17 registry.xiaohui.cn/library/ceph/ceph:v17
docker tag quay.io/ceph/ceph-grafana:8.3.5 registry.xiaohui.cn/library/ceph/ceph-registry.xiaohui.cn/library:8.3.5
docker tag quay.io/prometheus/prometheus:v2.33.4 registry.xiaohui.cn/library/prometheus/prometheus:v2.33.4
docker tag quay.io/ceph/keepalived:2.1.5 registry.xiaohui.cn/library/ceph/keepalived:2.1.5
docker tag quay.io/ceph/haproxy:2.3 registry.xiaohui.cn/library/ceph/haproxy:2.3
docker tag quay.io/prometheus/node-exporter:v1.3.1 registry.xiaohui.cn/library/prometheus/node-exporter:v1.3.1
docker tag docker.io/grafana/loki:2.4.0 registry.xiaohui.cn/library/loki:2.4.0
docker tag docker.io/grafana/promtail:2.4.0 registry.xiaohui.cn/library/promtail:2.4.0
docker tag quay.io/prometheus/alertmanager:v0.23.0 registry.xiaohui.cn/library/prometheus/alertmanager:v0.23.0
docker tag docker.io/maxwo/snmp-notifier:v1.2.1 registry.xiaohui.cn/library/snmp-notifier:v1.2.1
```

登录仓库

```bash
docker login registry.xiaohui.cn -u admin -p admin
```

将镜像推送到私有仓库

```bash
docker push registry.xiaohui.cn/library/ceph/ceph:v17
docker push registry.xiaohui.cn/library/ceph/ceph-registry.xiaohui.cn/library:8.3.5
docker push registry.xiaohui.cn/library/prometheus/prometheus:v2.33.4
docker push registry.xiaohui.cn/library/ceph/keepalived:2.1.5
docker push registry.xiaohui.cn/library/ceph/haproxy:2.3
docker push registry.xiaohui.cn/library/prometheus/node-exporter:v1.3.1
docker push registry.xiaohui.cn/library/loki:2.4.0
docker push registry.xiaohui.cn/library/promtail:2.4.0
docker push registry.xiaohui.cn/library/prometheus/alertmanager:v0.23.0
docker push registry.xiaohui.cn/library/snmp-notifier:v1.2.1
```

# Ceph安装

## 先决条件

如果这个也需要离线

1. 在仓库文件中加上keepcache=1

2. 从/var/cache/dnf中去find rpm后缀的文件

3. 用createrepo命令生成元数据

4. 将元数据和rpm一起放到内网http服务器

```bash
yum install podman chrony lvm2 systemd python3 -y
reboot
```

## 为ceph准备仓库

如果放在了内网服务器，baseurl用内网的

```ini
cat > /etc/yum.repos.d/cephadm.repo << eof
[cephadm]
name=cephadm lixiaohui write
baseurl=http://mirrors.aliyun.com/centos/8-stream/storage/x86_64/ceph-quincy
enabled=1
gpgcheck=0
eof
```

## 安装cephadm

```bash
yum install cephadm -y
update-ca-trust
```

## 修改cephadm

默认情况下，cephadm从外网下载，我们修改镜像指向我们的harbor

```bash
sed -i 's/^DEFAULT_IMAGE =.*/DEFAULT_IMAGE = "registry.xiaohui.cn\/library\/ceph\/ceph:v17"/' /usr/sbin/cephadm
sed -i 's/^DEFAULT_PROMETHEUS_IMAGE =.*/DEFAULT_PROMETHEUS_IMAGE = "registry.xiaohui.cn\/library\/prometheus\/prometheus:v2.33"/' /usr/sbin/cephadm
sed -i 's/^DEFAULT_LOKI_IMAGE =.*/DEFAULT_LOKI_IMAGE = "registry.xiaohui.cn\/library\/grafana\/loki:2.4.0"/' /usr/sbin/cephadm
sed -i 's/^DEFAULT_PROMTAIL_IMAGE =.*/DEFAULT_PROMTAIL_IMAGE = "registry.xiaohui.cn\/library\/grafana\/promtail:2.4.0"/' /usr/sbin/cephadm
sed -i 's/^DEFAULT_NODE_EXPORTER_IMAGE =.*/DEFAULT_NODE_EXPORTER_IMAGE = "registry.xiaohui.cn\/library\/prometheus\/node-exporter:v1.3.1"/' /usr/sbin/cephadm
sed -i 's/^DEFAULT_ALERT_MANAGER_IMAGE =.*/DEFAULT_ALERT_MANAGER_IMAGE = "registry.xiaohui.cn\/library\/prometheus\/alertmanager:v0.23.0"/' /usr/sbin/cephadm
sed -i 's/^DEFAULT_GRAFANA_IMAGE =.*/DEFAULT_GRAFANA_IMAGE = "registry.xiaohui.cn\/library\/ceph\/ceph-grafana:8.3.5"/' /usr/sbin/cephadm
sed -i 's/^DEFAULT_HAPROXY_IMAGE =.*/DEFAULT_HAPROXY_IMAGE = "registry.xiaohui.cn\/library\/ceph\/haproxy:2.3"/' /usr/sbin/cephadm
sed -i 's/^DEFAULT_KEEPALIVED_IMAGE =.*/DEFAULT_KEEPALIVED_IMAGE = "registry.xiaohui.cn\/library\/ceph\/keepalived:2.1.5"/' /usr/sbin/cephadm
sed -i 's/^DEFAULT_SNMP_GATEWAY_IMAGE =.*/DEFAULT_SNMP_GATEWAY_IMAGE = "registry.xiaohui.cn\/library\/maxwo\/snmp-notifier:v1.2.1"/' /usr/sbin/cephadm
```

## 下载镜像

```bash
podman pull registry.xiaohui.cn/library/ceph/ceph:v17
podman pull registry.xiaohui.cn/library/ceph/ceph-registry.xiaohui.cn/library:8.3.5
podman pull registry.xiaohui.cn/library/prometheus/prometheus:v2.33.4
podman pull registry.xiaohui.cn/library/ceph/keepalived:2.1.5
podman pull registry.xiaohui.cn/library/ceph/haproxy:2.3
podman pull registry.xiaohui.cn/library/prometheus/node-exporter:v1.3.1
podman pull registry.xiaohui.cn/library/loki:2.4.0
podman pull registry.xiaohui.cn/library/promtail:2.4.0
podman pull registry.xiaohui.cn/library/prometheus/alertmanager:v0.23.0
podman pull registry.xiaohui.cn/library/snmp-notifier:v1.2.1


podman tag registry.xiaohui.cn/library/ceph/ceph:v17 quay.io/ceph/ceph:v17
podman tag registry.xiaohui.cn/library/ceph/ceph-registry.xiaohui.cn/library:8.3.5 quay.io/ceph/ceph-grafana:8.3.5
podman tag registry.xiaohui.cn/library/prometheus/prometheus:v2.33.4 quay.io/prometheus/prometheus:v2.33.4
podman tag registry.xiaohui.cn/library/ceph/keepalived:2.1.5 quay.io/ceph/keepalived:2.1.5
podman tag registry.xiaohui.cn/library/ceph/haproxy:2.3 quay.io/ceph/haproxy:2.3
podman tag registry.xiaohui.cn/library/prometheus/node-exporter:v1.3.1 quay.io/prometheus/node-exporter:v1.3.1
podman tag registry.xiaohui.cn/library/loki:2.4.0 docker.io/grafana/loki:2.4.0
podman tag registry.xiaohui.cn/library/promtail:2.4.0 docker.io/grafana/promtail:2.4.0
podman tag registry.xiaohui.cn/library/prometheus/alertmanager:v0.23.0 quay.io/prometheus/alertmanager:v0.23.0
podman tag registry.xiaohui.cn/library/snmp-notifier:v1.2.1 docker.io/maxwo/snmp-notifier:v1.2.1
```

## 部署ceph

```bash
 cephadm bootstrap --mon-ip 172.16.50.201 --allow-fqdn-hostname --single-host-defaults
```

## 查询集群状态

```bash
cephadm shell
ceph -s
  cluster:
    id:     09bcd510-4976-11ed-aa7b-005056942509
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum ceph.xiaohui.cn (age 7m)
    mgr: ceph.xiaohui.cn.jzxkxe(active, since 4m), standbys: ceph.rkqbfy
    osd: 3 osds: 3 up (since 81s), 3 in (since 2m)

  data:
    pools:   1 pools, 1 pgs
    objects: 2 objects, 449 KiB
    usage:   61 MiB used, 1.5 TiB / 1.5 TiB avail
    pgs:     1 active+clean

  progress:
    Updating alertmanager deployment (+1 -> 1) (0s)
      [............................]
```

## 增加OSD

```BAS
ceph orch apply osd --all-available-devices
```

## 查询Ceph进程列表

```bash
ceph orch ls
NAME                       PORTS        RUNNING  REFRESHED  AGE  PLACEMENT
alertmanager               ?:9093,9094      1/1  7s ago     6m   count:1
crash                                       1/1  7s ago     7m   *
grafana                    ?:3000           1/1  7s ago     7m   count:1
mgr                                         2/2  7s ago     7m   count:2
mon                                         1/5  7s ago     7m   count:5
node-exporter              ?:9100           1/1  7s ago     7m   *
osd.all-available-devices                     3  7s ago     12s  *
prometheus                 ?:9095           1/1  7s ago     7m   count:1
```

## 查询Ceph进程状态

```bash
ceph orch ps 
NAME                        HOST             PORTS        STATUS         REFRESHED  AGE  MEM USE  MEM LIM  VERSION  IMAGE ID      CONTAINER ID
alertmanager.ceph           ceph.xiaohui.cn  *:9093,9094  running (54s)    19s ago  82s    13.7M        -           ba2b418f427c  210748e20a43
crash.ceph                  ceph.xiaohui.cn               running (6m)     19s ago   6m    6995k        -  17.2.4   652a1437d122  2d4268b7565a
grafana.ceph                ceph.xiaohui.cn  *:3000       running (50s)    19s ago  75s    49.2M        -  8.3.5    dad864ee21e9  e6971c744c1a
mgr.ceph.rkqbfy             ceph.xiaohui.cn  *:8443       running (6m)     19s ago   6m     421M        -  17.2.4   652a1437d122  e67f19e15fe8
mgr.ceph.xiaohui.cn.jzxkxe  ceph.xiaohui.cn  *:9283       running (8m)     19s ago   8m     468M        -  17.2.4   652a1437d122  ab98c499af8b
mon.ceph.xiaohui.cn         ceph.xiaohui.cn               running (8m)     19s ago   8m    51.9M    2048M  17.2.4   652a1437d122  176b7634b006
node-exporter.ceph          ceph.xiaohui.cn  *:9100       running (91s)    19s ago   5m    18.5M        -           1dbe0e931976  3556d92d755b
osd.0                       ceph.xiaohui.cn               running (3m)     19s ago   3m    68.4M    4096M  17.2.4   652a1437d122  087173d0fc27
osd.1                       ceph.xiaohui.cn               running (3m)     19s ago   3m    70.6M    4096M  17.2.4   652a1437d122  6429f6f3ae70
osd.2                       ceph.xiaohui.cn               running (2m)     19s ago   2m    64.3M    4096M  17.2.4   652a1437d122  287030daade5
prometheus.ceph             ceph.xiaohui.cn  *:9095       running (68s)    19s ago  68s    37.8M        -           514e6a882f6e  5bda216eaa8a
```

一切正常

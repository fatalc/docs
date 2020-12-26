
# 二进制安装 calico v3

calico v3 官方所有教程中均推荐使用 docker 方式运行，使用 calicoctl 配合 docker 运行会帮你提供好运行依赖和自动配置等。而如果使用二进制方式运行 calico 则需要手动安装依赖和配置各个组件。

> It automatically pre-initializes the etcd database (which the other installation methods do not).

**对于calico 集群，需要在每个节点均安装一套calico node。所有集群节点均链接到一个etcd集群，进行集群数据同步。**

calico node 容器主要提供以下组件的安装运行,本地安装则需要手动安装配置这些组件:

- calicoctl,calico 命令行工具。
- felix,calico node daemon。
- confd,管理calico BGP 配置文件。
- bird,用于 BGP 节点互联 BGP mesh。

此外，calico node还依赖于：

- etcd v3,用于提供calico集群的数据源。
- net-tools,用于提供 arp 命令。
- conntrack,用于 Netfilter 连接追踪。
- iptables,用于管理 iptable 规则等。
- procps,提供 ps 命令。
- kmod,管理内核模块。

centos 上可以运行以下命令安装上述依赖：

```sh
yum install -y conntrack net-tools iptables procps kmod
```

calico-libnetwork-plugin 目前对 ip v6 的支持在高版本内核下会有问题，该 issue 在 calico/issues/2191
需要禁用所有 ipv6 ：

```sh
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.lo.disable_ipv6=1
```

## calicoctl 安装

```sh
# 从 ctl 镜像中 copy 出来，也可以从 git release 下载。
CALICO_CTL_IMAGE=calico/ctl:v3.12.0
docker pull ${CALICO_CTL_IMAGE}
docker create --name calico-ctl-create ${CALICO_CTL_IMAGE}
sudo docker cp calico-ctl-create:/calicoctl /usr/local/bin/calicoctl
docker rm calico-ctl-create

ETCD_ENPOINTS="http://192.168.2.31:2379"
# 这里配置calicoctl连接数据源，根据实际情况变化
sudo mkdir -p /etc/calico
sudo sh -c "cat > /etc/calico/calicoctl.cfg" << EOF
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  datastoreType: etcdv3
  etcdEndpoints: ${ETCD_ENPOINTS}
EOF
```

## calico-node 安装

官方对于该操作的文档：[Binary install without package manager](https://docs.projectcalico.org/getting-started/bare-metal/installation/binary)

calico-node 包含了运行所需的各种依赖文件，可以从里面copy到主机上。这些依赖项目在[linux-dependencies](https://github.cqom/projectcalico/node#linux-dependencies)中描述。

二进制文件下载:

```sh
CALICO_NODE_IMAGE=calico/node:v3.12.0
docker pull ${CALICO_NODE_IMAGE}
docker create --name calico-node-create  ${CALICO_NODE_IMAGE}
# calico-node(felix confd)
sudo docker cp calico-node-create:/bin/calico-node /usr/local/bin/calico-node
# felix,felix 里面所需要的环境变量与calico node 重叠但是名称不同，所以直接使用脚本方式。详见：https://github.com/projectcalico/node/blob/release-v3.12/filesystem/etc/service/available/felix/run
sudo docker cp calico-node-create:/etc/service/available/felix/run /usr/local/bin/calico-felix
# bird，用于节点互联的组件，使用由confd生成的配置文件。
sudo docker cp calico-node-create:/usr/bin/bird /usr/local/bin/bird
# confd configurations，confd 的模板等，confd 从这些模板动态生成 bird 等所需的配置文件。
sudo docker cp calico-node-create:/etc/calico/confd /etc/calico/confd
docker rm calico-node-create
```

集中配置 calico 环境变量:

```sh
# 主要配置一个变量，其他变量去 calico.env 里面修改。
ETCD_ENPOINTS="http://192.168.2.31:2379"
sudo sh -c "cat > /etc/calico/calico.env" << EOF
# all support env,default values are referenced: https://docs.projectcalico.org/reference/node/configuration
NODENAME=$(hostname)
NO_DEFAULT_POOLS=false
IP=""
IP6=""
IP_AUTODETECTION_METHOD=first-found
IP6_AUTODETECTION_METHOD=first-found
DISABLE_NODE_IP_CHECK=false
AS=
CALICO_DISABLE_FILE_LOGGING=false
CALICO_ROUTER_ID=""
DATASTORE_TYPE=etcdv3
WAIT_FOR_DATASTORE=false
CALICO_NETWORKING_BACKEND=bird
CALICO_IPV4POOL_CIDR=172.16.0.0/16
CALICO_IPV6POOL_CIDR=""
CALICO_IPV4POOL_BLOCK_SIZE=26
CALICO_IPV6POOL_BLOCK_SIZE=122
CALICO_IPV4POOL_IPIP=Always
CALICO_IPV4POOL_VXLAN=Never
CALICO_IPV4POOL_NAT_OUTGOING=true
CALICO_IPV6POOL_NAT_OUTGOING=false
CALICO_IPV4POOL_NODE_SELECTOR="all()"
CALICO_IPV6POOL_NODE_SELECTOR="all()"
CALICO_STARTUP_LOGLEVEL=ERROR
CLUSTER_TYPE=""
ETCD_ENDPOINTS=${ETCD_ENPOINTS}
ETCD_DISCOVERY_SRV=""
ETCD_KEY_FILE=""
ETCD_CERT_FILE=""
ETCD_CA_CERT_FILE=""
CALICO_MANAGE_CNI=false
FELIX_LOGSEVERITYSCREEN=INFO
EOF
```

### 安装 calico-felix service

```sh
sudo sh -c "cat > /etc/systemd/system/calico-felix.service" << EOF
[Unit]
Description=Calico Felix agent
After=syslog.target network.target

[Service]
User=root
EnvironmentFile=/etc/calico/calico.env
ExecStartPre=/usr/bin/mkdir -p /var/run/calico
ExecStartPre=/usr/local/bin/calico-node -startup
ExecStart=/usr/local/bin/calico-felix
KillMode=process
Restart=on-failure
LimitNOFILE=32000

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable calico-felix
sudo systemctl start calico-felix
```

### 安装 calico-confd service

```sh
sudo sh -c "cat > /etc/systemd/system/calico-confd.service" << EOF
[Unit]
Description=Calico confd
After=syslog.target network.target

[Service]
User=root
EnvironmentFile=/etc/calico/calico.env
ExecStartPre=/usr/bin/mkdir -p /var/run/calico
ExecStart=/usr/local/bin/calico-node -confd
KillMode=process
Restart=on-failure
LimitNOFILE=32000

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable calico-confd
sudo systemctl start calico-confd
```

### 安装 bird service

```sh
sudo sh -c "cat > /etc/systemd/system/bird.service" << EOF
[Unit]
Description=BIRD internet routing daemon
After=syslog.target network.target

[Service]
User=root
EnvironmentFile=/etc/calico/calico.env
ExecStartPre=/usr/bin/mkdir -p /var/run/calico
ExecStart=/usr/local/bin/bird -R -s /var/run/calico/bird.ctl -d -c /etc/calico/confd/config/bird.cfg
KillMode=process
Restart=on-failure
LimitNOFILE=32000

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable bird
sudo systemctl start bird
```

### calico libnetwork-plugin 安装

```sh
CALICO_LIBNETWORK_PLUGIN_IMAGE=internal-registry.ghostcloud.cn/calico/libnetwork-plugin:v2.6

docker pull ${CALICO_LIBNETWORK_PLUGIN_IMAGE}
docker create --name calico-libnetwork-plugin-create ${CALICO_LIBNETWORK_PLUGIN_IMAGE}
sudo docker cp calico-libnetwork-plugin-create:/libnetwork-plugin /usr/local/bin/calico-libnetwork-plugin
docker rm calico-libnetwork-plugin-create

sudo sh -c "cat > /etc/systemd/system/calico-libnetwork-plugin.service" << EOF
[Unit]
Description=Calico libnetwork plugin
After=syslog.target network.target calico-felix.service
Requires=calico-felix.service

[Service]
User=root
EnvironmentFile=/etc/calico/calico.env
ExecStartPre=/usr/bin/mkdir -p /var/run/calico
ExecStart=/usr/local/bin/calico-libnetwork-plugin
KillMode=process
Restart=on-failure
LimitNOFILE=32000

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable calico-libnetwork-plugin
sudo systemctl start calico-libnetwork-plugin
```

docker 创建网络：

docker 需要配置连接至etcd成为集群。

修改 `/usr/lib/systemd/system/docker.service` (centos下，不同系统路径和配置可能不同)，
在 `ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock ` 行增加 ` --cluster-store=etcd://192.168.2.21:2379`

这里必须指定 subnet ，该subnet 需要是 ippool 中的地址或子集。

```sh
docker network create --driver calico --ipam-driver calico-ipam --subnet 192.168.0.0/16 cali_net
```

## 后续配置

配置全部允许的 calico network policy, 否则在默认规则下所有环境不能互通。

```sh
sudo sh -c "cat > /etc/calico/global-network-policy-allow-all.yaml" << EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-all
spec:
  selector: all()
  ingress:
  - action: Allow
  egress:
  - action: Allow
EOF
sudo calicoctl apply -f /etc/calico/global-network-policy-allow-all.yaml
```

增加ip池

```yaml
sudo sh -c "cat > /etc/calico/newben-ippool.yaml" << EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: newben-pool
spec:
  cidr: 172.168.0.0/16
  nodeSelector: all()
EOF
sudo calicoctl apply -f /etc/calico/newben-ippool.yaml
```

## 附录

附上调试时的script：

install.sh

```sh
#!/usr/bin/env sh

# 在这之前需要设置 docker.service 增加连接至etcd相关配置， 例如： --cluster-store=etcd://192.168.2.21:2379
ETCD_ENPOINTS="http://192.168.2.xx:2379"
# calcioctl 从 ctl 镜像中 copy 出来，也可以从 git release 下载。
CALICO_CTL_IMAGE=calico/ctl:v3.12.0
CALICO_NODE_IMAGE=calico/node:v3.12.0

yum install -y conntrack net-tools iptables procps kmod

sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.lo.disable_ipv6=1

docker pull ${CALICO_CTL_IMAGE}
docker create --name calico-ctl-create ${CALICO_CTL_IMAGE}
sudo docker cp calico-ctl-create:/calicoctl /usr/local/bin/calicoctl
docker rm calico-ctl-create

# calicoctl 配置
sudo mkdir -p /etc/calico
sudo sh -c "cat > /etc/calico/calicoctl.cfg" << EOF
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  datastoreType: etcdv3
  etcdEndpoints: ${ETCD_ENPOINTS}
EOF

docker pull ${CALICO_NODE_IMAGE}
docker create --name calico-node-create  ${CALICO_NODE_IMAGE}
# felix
sudo docker cp calico-node-create:/bin/calico-node /usr/local/bin/calico-node
sudo docker cp calico-node-create:/etc/service/available/felix/run /usr/local/bin/calico-felix
# bird
sudo docker cp calico-node-create:/usr/bin/bird /usr/local/bin/bird
# confd
sudo docker cp calico-node-create:/etc/calico/confd /etc/calico/confd
docker rm calico-node-create

sudo sh -c "cat > /etc/calico/calico.env" << EOF
# all support env,default values are referenced: https://docs.projectcalico.org/reference/node/configuration
NODENAME=$(hostname)
NO_DEFAULT_POOLS=false
IP=""
IP6=""
IP_AUTODETECTION_METHOD=first-found
IP6_AUTODETECTION_METHOD=first-found
DISABLE_NODE_IP_CHECK=false
AS=
CALICO_DISABLE_FILE_LOGGING=false
CALICO_ROUTER_ID=""
DATASTORE_TYPE=etcdv3
WAIT_FOR_DATASTORE=false
CALICO_NETWORKING_BACKEND=bird
CALICO_IPV4POOL_CIDR=192.168.0.0/16
CALICO_IPV6POOL_CIDR=""
CALICO_IPV4POOL_BLOCK_SIZE=26
CALICO_IPV6POOL_BLOCK_SIZE=122
CALICO_IPV4POOL_IPIP=Always
CALICO_IPV4POOL_VXLAN=Never
CALICO_IPV4POOL_NAT_OUTGOING=true
CALICO_IPV6POOL_NAT_OUTGOING=false
CALICO_IPV4POOL_NODE_SELECTOR="all()"
CALICO_IPV6POOL_NODE_SELECTOR="all()"
CALICO_STARTUP_LOGLEVEL=ERROR
CLUSTER_TYPE=""
ETCD_ENDPOINTS=${ETCD_ENPOINTS}
ETCD_DISCOVERY_SRV=""
ETCD_KEY_FILE=""
ETCD_CERT_FILE=""
ETCD_CA_CERT_FILE=""
CALICO_MANAGE_CNI=false
FELIX_LOGSEVERITYSCREEN=INFO
EOF

# felix,reference: https://github.com/projectcalico/node/blob/master/filesystem/etc/service/available/felix/run
sudo sh -c "cat > /etc/systemd/system/calico-felix.service" << EOF
[Unit]
Description=Calico Felix agent
After=syslog.target network.target

[Service]
User=root
EnvironmentFile=/etc/calico/calico.env
ExecStartPre=/usr/bin/mkdir -p /var/run/calico
ExecStartPre=/usr/local/bin/calico-node -startup
ExecStart=/usr/local/bin/calico-felix
KillMode=process
Restart=on-failure
LimitNOFILE=32000

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable calico-felix
sudo systemctl start calico-felix

# confd,reference: https://github.com/projectcalico/node/blob/master/filesystem/etc/service/available/confd/run
sudo sh -c "cat > /etc/systemd/system/calico-confd.service" << EOF
[Unit]
Description=Calico confd
After=syslog.target network.target

[Service]
User=root
EnvironmentFile=/etc/calico/calico.env
ExecStartPre=/usr/bin/mkdir -p /var/run/calico
ExecStart=/usr/local/bin/calico-node -confd
KillMode=process
Restart=on-failure
LimitNOFILE=32000

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable calico-confd
sudo systemctl start calico-confd

# bird,reference: https://github.com/projectcalico/node/blob/master/filesystem/etc/service/available/bird/run
sudo sh -c "cat > /etc/systemd/system/bird.service" << EOF
[Unit]
Description=BIRD internet routing daemon
After=syslog.target network.target

[Service]
User=root
EnvironmentFile=/etc/calico/calico.env
ExecStartPre=/usr/bin/mkdir -p /var/run/calico
ExecStart=/usr/local/bin/bird -R -s /var/run/calico/bird.ctl -d -c /etc/calico/confd/config/bird.cfg
KillMode=process
Restart=on-failure
LimitNOFILE=32000

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable bird
sudo systemctl start bird

# libnetwork-plugin

CALICO_LIBNETWORK_PLUGIN_IMAGE=internal-registry.ghostcloud.cn/calico/libnetwork-plugin:v2.6

docker pull ${CALICO_LIBNETWORK_PLUGIN_IMAGE}
docker create --name calico-libnetwork-plugin-create ${CALICO_LIBNETWORK_PLUGIN_IMAGE}
sudo docker cp calico-libnetwork-plugin-create:/libnetwork-plugin /usr/local/bin/calico-libnetwork-plugin
docker rm calico-libnetwork-plugin-create

cat > /etc/systemd/system/calico-libnetwork-plugin.service << EOF
[Unit]
Description=Calico libnetwork plugin
After=syslog.target network.target calico-felix.service
Requires=calico-felix.service

[Service]
User=root
EnvironmentFile=/etc/calico/calico.env
ExecStartPre=/usr/bin/mkdir -p /var/run/calico
ExecStart=/usr/local/bin/calico-libnetwork-plugin
KillMode=process
Restart=on-failure
LimitNOFILE=32000

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable calico-libnetwork-plugin
sudo systemctl start calico-libnetwork-plugin

cat > /etc/calico/global-network-policy-allow-all.yaml << EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-all
spec:
  selector: all()
  ingress:
  - action: Allow
  egress:
  - action: Allow
EOF
calicoctl apply -f /etc/calico/global-network-policy-allow-all.yaml

# 在这之前请确保设置了 docker.service 增加连接至etcd相关配置， 例如： --cluster-store=etcd://192.168.2.21:2379
docker network create --driver calico --ipam-driver calico-ipam --subnet 172.16.0.0/16 cali_net

# ip-bind
# 增加子网 192.168.19.0/24 的 ippool
cat > /etc/calico/bindip-ippool.yaml << EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: bind-pool
spec:
  cidr: 192.168.19.0/24
  nodeSelector: all()
  natOutgoing: true
EOF
calicoctl apply -f /etc/calico/bindip-ippool.yaml
docker network create --driver calico --ipam-driver calico-ipam --subnet 192.168.19.0/24 fix_net
```

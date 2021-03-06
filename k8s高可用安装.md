# 一，环境准备

| 角色   | ip             | 主机名 | 内存/CPU/磁盘 | 来源                 | 网络接入             | 安装方式                  | 镜像名称                        | 账户                      | 密码   |
| ------ | -------------- | ------ | ------------- | -------------------- | ------------------------- | ------------------------------- | ------ | ------ | ------ |
| master01 | 192.168.31.209 | master01 | 2U/4G/40G | Vmware安装 | 桥接模式 | Win32DiskImager制作启动盘 | CentOS-8.3.2011-x86_64-dvd1.iso | root | 123456 |
| master02 | 192.168.31.214 | master01 | 2U/4G/40G     | Vmware安装           | 桥接模式       | Vmware 创建虚拟机         | CentOS-8.3.2011-x86_64-dvd1.iso | root | 123456 |
| worker node | 192.168.31.240 | node | 2U/4G/40G | dell笔记本安装centos | wifi无线 | Win32DiskImager制作启动盘 | CentOS-8.3.2011-x86_64-dvd1.iso | root | 123456 |

```
cat /etc/redhat-release 
CentOS Linux release 8.3.2011
```
软件包:
链接：https://pan.baidu.com/s/12x4fKEN5onswPfDvGFnoUw 
提取码：lebb 

# 二，安装前操作

```
# 修改主机名 重启机器
hostnamectl set-hostname <hostname>
cat >> /etc/hosts << EOF
192.168.31.240 node
192.168.31.209 master01
192.168.31.214 master02
EOF

# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld
# 关闭selinux
setenforce 0  # 临时
sed -i 's/enforcing/disabled/' /etc/selinux/config  # 永久
# 关闭交换分区
swapoff -a  # 临时
sed -ri 's/.*swap.*/#&/' /etc/fstab    # 永久

# 将桥接的IPv4流量传递到iptables的链
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system  # 生效

# 时间同步
rpm -ivh http://mirrors.wlnmp.com/centos/wlnmp-release-centos.noarch.rpm
yum install wntp -y
ntpdate ntp1.aliyun.com
```

# 三，安装docker

(1) 下载安装包

```
https://download.docker.com/linux/static/stable/x86_64/docker-19.03.9.tgz
```

(2) 解压二进制包

```
tar zxvf docker-19.03.9.tgz
mv docker/* /usr/bin
```

(3)  systemd 管理 docker

```
cat > /usr/lib/systemd/system/docker.service << EOF
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target
[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s
[Install]
WantedBy=multi-user.target
EOF
```

(4)  创建配置文件

```
mkdir /etc/docker
cat > /etc/docker/daemon.json << EOF
{
"registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
EOF
```

(5) 启动并设置开机启动

```
systemctl daemon-reload
systemctl start docker
systemctl enable docker
systemctl status docker
```

# 四，签发etcd证书

```
cfssl 是一个开源的证书管理工具，使用 json 文件生成证书，相比 openssl 更方便使用。
找任意一台服务器操作，这里用 Master 节点。
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
```

#### 生成etcd证书

（1）自签证书颁发机构（CA）

```
mkdir -p ~/TLS/{etcd,k8s}
cd TLS/etcd
cat > ca-config.json<< EOF
{
    "signing":{
        "default":{
            "expiry":"87600h"
        },
        "profiles":{
            "www":{
                "expiry":"87600h",
                "usages":[
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
EOF
cat > ca-csr.json<< EOF
{
    "CN":"etcd CA",
    "key":{
        "algo":"rsa",
        "size":2048
    },
    "names":[
        {
            "C":"CN",
            "L":"ShenZhen",
            "ST":"ShenZhen"
        }
    ]
}
EOF
```

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
ls *pem
ca-key.pem ca.pem
```

（2）使用自签 CA 签发 Etcd HTTPS 证书

```
cat > server-csr.json<< EOF
{
    "CN":"etcd",
    "hosts":[
        "192.168.31.240",
        "192.168.31.209",
        "192.168.31.214"
    ],
    "key":{
        "algo":"rsa",
        "size":2048
    },
    "names":[
        {
            "C":"CN",
            "L":"ShenZhen",
            "ST":"ShenZhen"
        }
    ]
}
EOF
```

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server
ls server*pem
server-key.pem server.pem
```

# 五，部署Etcd集群

(1) 下载安装包

```
 从 Github 下载二进制文件
下载地址：https://github.com/etcd-io/etcd/releases/download/v3.4.9/etcd-v3.4.9-linux-amd64.tar.gz
```

(2) 部署Etcd集群 [当前操作在master节点]

```
mkdir /opt/etcd/{bin,cfg,ssl} -p
tar zxvf etcd-v3.4.9-linux-amd64.tar.gz
mv etcd-v3.4.9-linux-amd64/{etcd,etcdctl} /opt/etcd/bin/

cat > /opt/etcd/cfg/etcd.conf << EOF
#[Member]
ETCD_NAME="etcd-1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.31.209:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.31.209:2379"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.31.209:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.31.209:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.31.209:2380,etcd-2=https://192.168.31.214:2380,etcd-3=https://192.168.31.240:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```

字段说明

```
ETCD_NAME：节点名称，集群中唯一
ETCD_DATA_DIR：数据目录
ETCD_LISTEN_PEER_URLS：集群通信监听地址
ETCD_LISTEN_CLIENT_URLS：客户端访问监听地址
ETCD_INITIAL_ADVERTISE_PEER_URLS：集群通告地址
ETCD_ADVERTISE_CLIENT_URLS：客户端通告地址
ETCD_INITIAL_CLUSTER：集群节点地址
ETCD_INITIAL_CLUSTER_TOKEN：集群 Token
ETCD_INITIAL_CLUSTER_STATE：加入集群的当前状态，new 是新集群，existing 表示加入
已有集群
```

（3）systemd 管理 etcd

```
cat > /usr/lib/systemd/system/etcd.service << EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
[Service]
Type=notify
EnvironmentFile=/opt/etcd/cfg/etcd.conf
ExecStart=/opt/etcd/bin/etcd \
--cert-file=/opt/etcd/ssl/server.pem \
--key-file=/opt/etcd/ssl/server-key.pem \
--peer-cert-file=/opt/etcd/ssl/server.pem \
--peer-key-file=/opt/etcd/ssl/server-key.pem \
--trusted-ca-file=/opt/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/opt/etcd/ssl/ca.pem \
--logger=zap
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF
```

（4）拷贝刚才生成的证书

```
cp ~/TLS/etcd/ca*pem ~/TLS/etcd/server*pem /opt/etcd/ssl/
```

（5）分发配置到node01 和 node02

```
# 分发至node01
scp -r /opt/etcd/ root@192.168.31.2214:/opt/
scp /usr/lib/systemd/system/etcd.service root@192.168.31.2214:/usr/lib/systemd/system/
# 分发至node02
scp -r /opt/etcd/ root@192.168.31.240:/opt/
scp /usr/lib/systemd/system/etcd.service root@192.168.31.240:/usr/lib/systemd/system/
```

（6）修改分发后的集群配置

```
# -------node01---------
#[Member]
ETCD_NAME="etcd-2"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.31.214:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.31.214:2379"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.31.214:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.31.214:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.31.209:2380,etcd-2=https://192.168.31.214:2380,etcd-3=https://192.168.31.240:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
# -------node02---------
#[Member]
ETCD_NAME="etcd-3"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.31.240:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.31.240:2379"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.31.240:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.31.240:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.31.209:2380,etcd-2=https://192.168.31.214:2380,etcd-3=https://192.168.31.240:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```

（7）启动并设置开机启动

```
systemctl daemon-reload
systemctl start etcd
systemctl enable etcd
systemctl status etcd

 etcd.service - Etcd Server
   Loaded: loaded (/usr/lib/systemd/system/etcd.service; enabled; vendor preset: disabled)
   Active: active (running) (thawing) since Sat 2021-03-13 14:57:44 CST; 13s ago
 Main PID: 7363 (etcd)
    Tasks: 13 (limit: 47203)
   Memory: 34.5M
   CGroup: /system.slice/etcd.service
           └─7363 /opt/etcd/bin/etcd --cert-file=/opt/etcd/ssl/server.pem --key-file=/opt/etcd/ssl/server-key.pem --peer-cert-file=/opt/etcd/ssl/server.pem --peer-key-file=/opt/etcd/ssl/server-key.pem --trusted-ca-file=/opt/etcd/ssl/ca.pem --peer-trusted-ca-file=/opt>
```

（8）查看集群状态

```
ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://192.168.31.240:2379,https://192.168.31.209:2379,https://192.168.31.214:2379" endpoint health

https://192.168.31.240:2379 is healthy: successfully committed proposal: took = 12.95498ms
https://192.168.31.209:2379 is healthy: successfully committed proposal: took = 29.649939ms
https://192.168.31.214:2379 is healthy: successfully committed proposal: took = 31.383899ms

如果输出上面信息，就说明集群部署成功。如果有问题第一步先看日志：
/var/log/message 或 journalctl -u etcd
```



# 五，创建master组件配置

### 部署kube-apiserver

（1）生成 kube-apiserver 证书

```
cd ~/TLS/k8s
cat > ca-config.json<< EOF
{
    "signing":{
        "default":{
            "expiry":"87600h"
        },
        "profiles":{
            "kubernetes":{
                "expiry":"87600h",
                "usages":[
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
EOF

cat > ca-csr.json<< EOF
{
    "CN":"kubernetes",
    "key":{
        "algo":"rsa",
        "size":2048
    },
    "names":[
        {
            "C":"CN",
            "L":"ShenZhen",
            "ST":"ShenZhen",
            "O":"k8s",
            "OU":"System"
        }
    ]
}
EOF
```

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
ls *pem
ca-key.pem ca.pem
```

```
# 使用自签 CA 签发 kube-apiserver HTTPS 证书
cat > server-csr.json<< EOF
{
    "CN":"kubernetes",
    "hosts":[
        "10.0.0.1",
        "127.0.0.1",
        "192.168.31.240",
        "192.168.31.209",
        "192.168.31.214",
        "kubernetes",
        "kubernetes.default",
        "kubernetes.default.svc",
        "kubernetes.default.svc.cluster",
        "kubernetes.default.svc.cluster.local"
    ],
    "key":{
        "algo":"rsa",
        "size":2048
    },
    "names":[
        {
            "C":"CN",
            "L":"ShenZhen",
            "ST":"ShenZhen",
            "O":"k8s",
            "OU":"System"
        }
    ]
}
EOF
```

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server
ls server*pem
server-key.pem server.pem
```

（2）下载安装包

```
https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.18.md#v1183
注：打开链接你会发现里面有很多包，下载一个 server 包就够了，包含了 Master 和
Worker Node 二进制文件
```

（3） 解压二进制包

```
mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs}
tar zxvf kubernetes-server-linux-amd64.tar.gz
cd kubernetes/server/bin
cp kube-apiserver kube-scheduler kube-controller-manager /opt/kubernetes/bin
cp kubectl /usr/bin/
```

（4） 部署 kube-apiserver

```
master01
cat > /opt/kubernetes/cfg/kube-apiserver.conf << EOF
KUBE_APISERVER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--etcd-servers=https://192.168.31.209:2379,https://192.168.31.214:2379,https://192.168.31.240:2379 \\
--bind-address=192.168.31.209 \\
--secure-port=6443 \\
--advertise-address=192.168.31.209 \\
--allow-privileged=true \\
--service-cluster-ip-range=10.0.0.0/24 \\
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \\
--authorization-mode=RBAC,Node \\
--enable-bootstrap-token-auth=true \\
--token-auth-file=/opt/kubernetes/cfg/token.csv \\
--service-node-port-range=30000-32767 \\
--kubelet-client-certificate=/opt/kubernetes/ssl/server.pem \\
--kubelet-client-key=/opt/kubernetes/ssl/server-key.pem \\
--tls-cert-file=/opt/kubernetes/ssl/server.pem \\
--tls-private-key-file=/opt/kubernetes/ssl/server-key.pem \\
--client-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--etcd-cafile=/opt/etcd/ssl/ca.pem \\
--etcd-certfile=/opt/etcd/ssl/server.pem \\
--etcd-keyfile=/opt/etcd/ssl/server-key.pem \\
--audit-log-maxage=30 \\
--audit-log-maxbackup=3 \\
--audit-log-maxsize=100 \\
--audit-log-path=/opt/kubernetes/logs/k8s-audit.log"
EOF

```

```
注：上面两个\ \ 第一个是转义符，第二个是换行符，使用转义符是为了使用 EOF 保留换
行符。
–logtostderr：启用日志
—v：日志等级
–log-dir：日志目录
–etcd-servers：etcd 集群地址
–bind-address：监听地址
–secure-port：https 安全端口
–advertise-address：集群通告地址
–allow-privileged：启用授权
–service-cluster-ip-range：Service 虚拟 IP 地址段
–enable-admission-plugins：准入控制模块
–authorization-mode：认证授权，启用 RBAC 授权和节点自管理
–enable-bootstrap-token-auth：启用 TLS bootstrap 机制
–token-auth-file：bootstrap token 文件
–service-node-port-range：Service nodeport 类型默认分配端口范围
–kubelet-client-xxx：apiserver 访问 kubelet 客户端证书
–tls-xxx-file：apiserver https 证书
–etcd-xxxfile：连接 Etcd 集群证书
–audit-log-xxx：审计日志
```

（5） 拷贝刚才生成的证书

```
cp -rf ~/TLS/k8s/ca*pem ~/TLS/k8s/server*pem /opt/kubernetes/ssl/
scp -r ~/TLS/k8s/ca*pem ~/TLS/k8s/server*pem 
```

（6） 启用 TLS Bootstrapping 机制
![image](https://github.com/Mountains-and-rivers/k8s-install-by-binary/blob/main/image/TLS%20bootstraping%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B.jpg)

```
 TLS Bootstraping：Master apiserver 启用 TLS 认证后，Node 节点 kubelet 和 kube-
proxy 要与 kube-apiserver 进行通信，必须使用 CA 签发的有效证书才可以，当 Node
节点很多时，这种客户端证书颁发需要大量工作，同样也会增加集群扩展复杂度。为了
简化流程，Kubernetes 引入了 TLS bootstraping 机制来自动颁发客户端证书，kubelet
会以一个低权限用户自动向 apiserver 申请证书，kubelet 的证书由 apiserver 动态签署。
所以强烈建议在 Node 上使用这种方式，目前主要用于 kubelet，kube-proxy 还是由我
们统一颁发一个证书。

cat > /opt/kubernetes/cfg/token.csv << EOF
378c331fdff5dfa16d0b8b475f831b24,kubelet-bootstrap,10001,"system:node-bootstrapper"
EOF
```

格式：token，用户名，UID，用户组
token 也可自行生成替换：

```
head -c 16 /dev/urandom | od -An -t x | tr -d ' '
```

（7） systemd 管理 apiserver

```
 cat > /usr/lib/systemd/system/kube-apiserver.service << EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-apiserver.conf
ExecStart=/opt/kubernetes/bin/kube-apiserver \$KUBE_APISERVER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```

#### kube-controller-manager

（1）创建配置文件

```
 cat > /opt/kubernetes/cfg/kube-controller-manager.conf << EOF
KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--leader-elect=true \\
--master=127.0.0.1:8080 \\
--bind-address=127.0.0.1 \\
--allocate-node-cidrs=true \\
--cluster-cidr=10.244.0.0/16 \\
--service-cluster-ip-range=10.0.0.0/24 \\
--cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \\
--cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--root-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--experimental-cluster-signing-duration=87600h0m0s"
EOF
```

```
–master：通过本地非安全本地端口 8080 连接 apiserver。
–leader-elect：当该组件启动多个时，自动选举（HA）
–cluster-signing-cert-file/–cluster-signing-key-file：自动为 kubelet 颁发证书
的 CA，与 apiserver 保持一致
```

（2）systemd 管理 controller-manager

```
cat > /usr/lib/systemd/system/kube-controller-manager.service << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-controller-manager.conf
ExecStart=/opt/kubernetes/bin/kube-controller-manager \$KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```

（3）启动 controller-manager

```
systemctl daemon-reload
systemctl start kube-controller-manager
systemctl enable kube-controller-manager
systemctl status kube-controller-manager
```

####  kube-scheduler

 （1）创建配置文件

```
cat > /opt/kubernetes/cfg/kube-scheduler.conf << EOF
KUBE_SCHEDULER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--leader-elect \
--master=127.0.0.1:8080 \
--bind-address=127.0.0.1"
EOF
```

```
–master：通过本地非安全本地端口 8080 连接 apiserver。
–leader-elect：当该组件启动多个时，自动选举（HA）
```

 （2）systemd 管理 scheduler

```
cat > /usr/lib/systemd/system/kube-scheduler.service << EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-scheduler.conf
ExecStart=/opt/kubernetes/bin/kube-scheduler \$KUBE_SCHEDULER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```



# 六， 创建 Worker组件配置

####  kubelet

（1） 创建工作目录并拷贝二进制文件

```
cd /root
cd kubernetes/server/bin
cp -rf kubelet kube-proxy /opt/kubernetes/bin # 本地拷贝
```

（2） 创建kubelet启动配置文件

```
cat > /opt/kubernetes/cfg/kubelet.conf << EOF
KUBELET_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--hostname-override=master01 \\
--network-plugin=cni \\
--kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \\
--bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \\
--config=/opt/kubernetes/cfg/kubelet-config.yml \\
--cert-dir=/opt/kubernetes/ssl \\
--pod-infra-container-image=lizhenliang/pause-amd64:3.0"
EOF
```

```
–hostname-override：显示名称，集群中唯一
–network-plugin：启用 CNI
–kubeconfig：空路径，会自动生成，后面用于连接 apiserver
–bootstrap-kubeconfig：首次启动向 apiserver 申请证书
–config：配置参数文件
–cert-dir：kubelet 证书生成目录
–pod-infra-container-image：管理 Pod 网络容器的镜像
```

（3）  配置参数文件

```
cat > /opt/kubernetes/cfg/kubelet-config.yml << EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
cgroupDriver: cgroupfs
clusterDNS:
- 10.0.0.2
clusterDomain: cluster.local
failSwapOn: false
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /opt/kubernetes/ssl/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
evictionHard:
imagefs.available: 15%
memory.available: 100Mi
nodefs.available: 10%
nodefs.inodesFree: 5%
maxOpenFiles: 1000000
maxPods: 110
EOF
```

（4） 生成 bootstrap.kubeconfig 文件

```
KUBE_APISERVER="https://192.168.31.209:6443" #apiserver IP:PORT
TOKEN="378c331fdff5dfa16d0b8b475f831b24" #与 token.csv 里保持一致
# 生成 kubelet bootstrap kubeconfig 配置文件
kubectl config set-cluster kubernetes \
--certificate-authority=/opt/kubernetes/ssl/ca.pem \
--embed-certs=true \
--server=${KUBE_APISERVER} \
--kubeconfig=bootstrap.kubeconfig
kubectl config set-credentials "kubelet-bootstrap" \
--token=${TOKEN} \
--kubeconfig=bootstrap.kubeconfig
kubectl config set-context default \
--cluster=kubernetes \
--user="kubelet-bootstrap" \
--kubeconfig=bootstrap.kubeconfig
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
```

```
拷贝到配置文件路径：
cp -rf bootstrap.kubeconfig /opt/kubernetes/cfg
```

（5）  systemd 管理 kubelet

```
cat > /usr/lib/systemd/system/kubelet.service << EOF
[Unit]
Description=Kubernetes Kubelet
After=docker.service
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kubelet.conf
ExecStart=/opt/kubernetes/bin/kubelet \$KUBELET_OPTS
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF
```

####  kube-proxy

（1）创建配置文件

```
cat > /opt/kubernetes/cfg/kube-proxy.conf << EOF
KUBE_PROXY_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--config=/opt/kubernetes/cfg/kube-proxy-config.yml"
EOF
```

（2）配置参数文件

```
cat > /opt/kubernetes/cfg/kube-proxy-config.yml << EOF
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
metricsBindAddress: 0.0.0.0:10249
clientConnection:
  kubeconfig: /opt/kubernetes/cfg/kube-proxy.kubeconfig
hostnameOverride: master01
clusterCIDR: 10.0.0.0/24
EOF
```

（3）生成 kube-proxy.kubeconfig 文件

生成 kube-proxy 证书

```
# 切换工作目录
cd ~/TLS/k8s
# 创建证书请求文件
cat > kube-proxy-csr.json<< EOF
{
    "CN":"system:kube-proxy",
    "hosts":[],
    "key":{
        "algo":"rsa",
        "size":2048
    },
    "names":[
        {
            "C":"CN",
            "L":"ShenZhen",
            "ST":"ShenZhen",
            "O":"k8s",
            "OU":"System"
        }
    ]
}
EOF
# 生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
ls kube-proxy*pem
kube-proxy-key.pem kube-proxy.pem
```

生成 kubeconfig 文件：

```
KUBE_APISERVER="https://192.168.31.209:6443"
kubectl config set-cluster kubernetes \
--certificate-authority=/opt/kubernetes/ssl/ca.pem \
--embed-certs=true \
--server=${KUBE_APISERVER} \
--kubeconfig=kube-proxy.kubeconfig
kubectl config set-credentials kube-proxy \
--client-certificate=./kube-proxy.pem \
--client-key=./kube-proxy-key.pem \
--embed-certs=true \
--kubeconfig=kube-proxy.kubeconfig
kubectl config set-context default \
--cluster=kubernetes \
--user=kube-proxy \
--kubeconfig=kube-proxy.kubeconfig
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

```
拷贝到配置文件指定路径：
cp -rf kube-proxy.kubeconfig /opt/kubernetes/cfg/
```

（4） systemd 管理 kube-proxy

```
cat > /usr/lib/systemd/system/kube-proxy.service << EOF
[Unit]
Description=Kubernetes Proxy
After=network.target
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-proxy.conf
ExecStart=/opt/kubernetes/bin/kube-proxy \$KUBE_PROXY_OPTS
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF
```

# CNI

（1） 下载安装包

```
先准备好 CNI 二进制文件：
下载地址：
cd /root
wget https://github.com/containernetworking/plugins/releases/download/v0.8.6/cni-plugins-linux-amd64-v0.8.6.tgz
解压二进制包并移动到默认工作目录
```

```
mkdir -p /opt/cni/bin
tar zxvf cni-plugins-linux-amd64-v0.8.6.tgz -C /opt/cni/bin
```

```
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
sed -i -r "s#quay.io/coreos/flannel:.*-rc2#lizhenliang/flannel:v0.12.0-amd64#g" kube-flannel.yml
```

（7） 授权 apiserver 访问 kubelet

```
cat > apiserver-to-kubelet-rbac.yaml<< EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
       - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
      - pods/log
    verbs:
      - "*"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

（8） 拷贝已部署好的 Node 相关文件到新节点

```
scp -r /opt/kubernetes root@192.168.31.214:/opt
scp -r /opt/cni/ root@192.168.31.214:/opt
scp /usr/lib/systemd/system/kube* root@192.168.31.214:/usr/lib/systemd/system
scp /usr/bin/kubectl root@192.168.31.214:/usr/bin

scp -r /opt/kubernetes root@192.168.31.240:/opt
scp -r /opt/cni/ root@192.168.31.240:/opt
scp /usr/lib/systemd/system/kube* root@192.168.31.240:/usr/lib/systemd/system
scp /usr/bin/kubectl root@192.168.31.240:/usr/bin
```

（9） master节点启动并设置开机启动

```
systemctl daemon-reload
systemctl start kube-apiserver
systemctl start kube-controller-manager
systemctl start kube-scheduler
systemctl start kubelet
systemctl start kube-proxy
systemctl enable kube-apiserver
systemctl enable kube-controller-manager
systemctl enable kube-scheduler
systemctl enable kubelet
systemctl enable kube-proxy
```

（10） node启动并设置开机启动

```
systemctl daemon-reload
systemctl start kube-apiserver
systemctl start kube-controller-manager
systemctl start kube-scheduler
systemctl start kubelet
systemctl start kube-proxy
systemctl enable kubelet
systemctl enable kube-proxy
```



（11） 查看集群状态

```
kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"} 
```



# 启动后操作

（1）授权 kubelet-bootstrap 用户允许请求证书

```
 kubectl create clusterrolebinding kubelet-bootstrap \
--clusterrole=system:node-bootstrapper \
--user=kubelet-bootstrap
```
（2） 部署 CNI 网络
```
kubectl apply -f kube-flannel.yml
kubectl get pods -n kube-system
kubectl get node
```
（3） 删除 kubelet 证书和 kubeconfig 文件重新生成
```
rm -rf /opt/kubernetes/cfg/kubelet.kubeconfig
rm -rf /opt/kubernetes/ssl/kubelet*
# 修改主机名
vi /opt/kubernetes/cfg/kubelet.conf
--hostname-override=node01 #修改为对应的主机名
vi /opt/kubernetes/cfg/kube-proxy-config.yml
hostnameOverride:node01 #修改为对应的主机名
```
（4） 授权 apiserver 访问 kubelet

```
kubectl apply -f apiserver-to-kubelet-rbac.yaml
```

(5） 在 Master 上批准新 Node kubelet 证书申请

```
kubectl get csr
NAME                                                   AGE     SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-8wh7AEAj8bGhaeI_K_pNRWCFVTM65eRrXtbEgCxbenc   4m33s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
node-csr-alnfvT1q97XCiVC6TILz8frakVi_av_f9yCSwWk5xKo   42m     kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
node-csr-s7PYu_ZCaDocKvq4mWLaawvswObcEQPo4ON6KhkOrwo   6m14s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
# master01 && master02 && node
kubectl certificate approve node-csr-8wh7AEAj8bGhaeI_K_pNRWCFVTM65eRrXtbEgCxbenc
kubectl certificate approve node-csr-alnfvT1q97XCiVC6TILz8frakVi_av_f9yCSwWk5xKo
kubectl certificate approve node-csr-s7PYu_ZCaDocKvq4mWLaawvswObcEQPo4ON6KhkOrwo

如果kubectl get node 看不到node，把master01的配置删除，重新批准证书申请
```

(6） 查看 Node 状态

```
kubectl get node
NAME       STATUS   ROLES    AGE    VERSION
master01   Ready    <none>   46s    v1.18.3
master02   Ready    <none>   4m1s   v1.18.3
node       Ready    <none>   4m2s   v1.18.3
```
# 验证

(1） 查看 etcd中的pods数据

```
ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://192.168.31.240:2379,https://192.168.31.209:2379,https://192.168.31.214:2379" get / --prefix --keys-only

ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://192.168.31.240:2379,https://192.168.31.209:2379,https://192.168.31.214:2379" get /registry/services/specs/default/nginx --prefix

```

(2） 验证

```
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get pod -owide -n kube-system
kubectl get pod,svc
NAME                        READY   STATUS              RESTARTS   AGE
pod/nginx-f89759699-2xf2v   1/1     Running   0          19m

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP        65m
service/nginx        NodePort    10.0.0.50    <none>        80:31091/TCP   21s
```
![image](https://github.com/Mountains-and-rivers/k8s-install-by-binary/blob/main/image/%E8%AE%BF%E9%97%AE%E9%AA%8C%E8%AF%81.png)

# 高可用

（1）安装keepalived 和 haproxy

```
yum install -y conntrack-tools libseccomp libtool-ltdl
yum install -y keepalived
yum install -y haproxy
```

(2) 配置keepalived

主

```
cat > /etc/keepalived/keepalived.conf << EOF
! Configuration File for keepalived
global_defs {

   notification_email {

     acassen@firewall.loc

     failover@firewall.loc

     sysadmin@firewall.loc

   }

   notification_email_from Alexandre.Cassen@firewall.loc

   smtp_server 192.168.31.209

   smtp_connect_timeout 30

   router_id lb01

}

vrrp_instance VI_1 {

    state MASTER

    interface ens160

    virtual_router_id 55

    priority 150

    advert_int 1

    authentication {

        auth_type PASS

        auth_pass 123456

    }

    virtual_ipaddress {

        192.168.31.158/24 dev ens160 label ens160:1

    }
}
EOF
```

备

```
cat > /etc/keepalived/keepalived.conf << EOF
! Configuration File for keepalived

global_defs {

   notification_email {

     acassen@firewall.loc

     failover@firewall.loc

     sysadmin@firewall.loc

   }

   notification_email_from Alexandre.Cassen@firewall.loc

   smtp_server 192.168.31.214

   smtp_connect_timeout 30

   router_id lb02

}

vrrp_instance VI_1 {

    state BACKUP

    interface ens160

    virtual_router_id 55

    priority 100

    advert_int 1

    authentication {

        auth_type PASS

        auth_pass 123456

    }

    virtual_ipaddress {

        192.168.31.158 dev ens160 label ens160:1

    }
}
EOF
```

(3) 启动keepalived

```
systemctl start keepalived.service
systemctl enable keepalived.service
systemctl status keepalived.service
```

(4) 配置haproxy

```
cat > /etc/haproxy/haproxy.cfg << EOF
global
    log /dev/log    local0
    log /dev/log    local1 notice
    chroot /var/lib/haproxy
    stats socket /var/run/haproxy-admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    nbproc 1

defaults
    log     global
    timeout connect 5000
    timeout client  10m
    timeout server  10m

listen  admin_stats
    bind 0.0.0.0:10080
    mode http
    log 127.0.0.1 local0 err
    stats refresh 30s
    stats uri /status
    stats realm welcome login\ Haproxy
    stats auth admin:123456
    stats hide-version
    stats admin if TRUE

listen kube-master
    bind 0.0.0.0:8443
    mode tcp
    option tcplog
    balance roundrobin
    server 192.168.31.214 192.168.31.214:6443 check inter 2000 fall 2 rise 2 weight 1
    server 192.168.31.209 192.168.31.209:6443 check inter 2000 fall 2 rise 2 weight 1
EOF
```

(5) 启动haproxy

```
systemctl enable haproxy
systemctl start haproxy
systemctl status haproxy
```
(6) 验证

```
curl -k https://192.168.31.158:8443/version
```
# 集群访问测试
(1) 访问集群
```

# 获取与之交互的集群名称
export CLUSTER_NAME=$(kubectl config view -o jsonpath='{"Cluster name\tServer\n"}{range .clusters[*]}{.name}{"\t"}{.cluster.server}{"\n"}{end}' --kubeconfig=bootstrap.kubeconfig | grep kubernetes| awk -F' ' '{print $1}')
# 获取TOKEN
TOKEN=$(kubectl get secrets -o jsonpath="{.items[?(@.metadata.annotations['kubernetes\.io/service-account\.name']=='default')].data.token}"|base64 -d)
# 获取集群地址
APISERVER=$(kubectl config view -o jsonpath='{"Cluster name\tServer\n"}{range .clusters[*]}{.name}{"\t"}{.cluster.server}{"\n"}{end}' --kubeconfig=bootstrap.kubeconfig | grep kubernetes| awk -F' ' '{print $2}')
# 使用令牌访问API
curl -X GET $APISERVER/api --header "Authorization: Bearer $TOKEN" --insecure

{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "192.168.31.209:6443"
    }
  ]
}

```

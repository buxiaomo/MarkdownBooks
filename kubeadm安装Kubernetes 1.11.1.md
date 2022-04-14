# kubeadm安装Kubernetes 1.11.1

作者QQ：95112082
## 环境

| 服务 | 版本 |
| --- | --- |
| system | ubuntu 16.04 |
| Docker | 17.03  |
| Kubernetes | 1.11.1  |
## 前期准备
### 代理设置
#### 安装sslocal、privoxy
```
apt-get update
apt-get upgrade -y
apt install shadowsocks privoxy -y
apt-get autoremove -y
```
#### 创建service文件
```
cat > /lib/systemd/system/sslocal.service << EOF
[Unit]
Description=Shadowsocks Client
After=network.target

[Service]
Type=simple
Environment="SS_ADDR=xxx.xxx.xxx"
Environment="SS_PROT=xxx"
Environment="SS_PASS=xxx"
Environment="SS_MODE=aes-256-cfb"
ExecStart=/usr/bin/sslocal -s \${SS_ADDR} -p \${SS_PROT} -k \${SS_PASS} -m \${SS_MODE} -b 127.0.0.1
ExecStop=/bin/kill -SIGTERM
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF
```

```
# 此文件请在CentOS下创建
cat > /etc/systemd/system/privoxy.service << EOF
[Unit]
Description=Privacy enhancing HTTP Proxy

[Service]
Environment=PIDFILE=/var/run/privoxy.pid
Environment=OWNER=privoxy
Environment=CONFIGFILE=/etc/privoxy/config
Type=forking
PIDFile=/var/run/privoxy.pid
ExecStart=/usr/sbin/privoxy --pidfile \$PIDFILE --user \$OWNER \$CONFIGFILE
ExecStopPost=/bin/rm -f $PIDFILE
SuccessExitStatus=15

[Install]
WantedBy=multi-user.target
EOF
```

#### 配置privoxy
```
echo "forward-socks5t   /               127.0.0.1:1080 ." >> /etc/privoxy/config
sed -i "s/listen-address.*/listen-address 127.0.0.1:8118/g" /etc/privoxy/config
echo "forward         192.168.*.*/     .
forward            10.*.*.*/     .
forward           127.*.*.*/     ." >> /etc/privoxy/config
```

#### 启动代理
```
systemctl daemon-reload
systemctl restart sslocal.service
systemctl enable sslocal.service
systemctl restart privoxy.service
systemctl enable privoxy.service
```

## 安装Docker
```
apt-get update
apt-get install -y apt-transport-https ca-certificates curl software-properties-common
curl --proxy http://127.0.0.1:8118 -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository "deb https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") $(lsb_release -cs) stable"
apt-get -o Acquire::http::proxy="http://127.0.0.1:8118" update 
VERSION=17.03.3
apt-get install -y docker-ce=$(apt-cache madison docker-ce | awk "/${VERSION}/{print \$3}")
```

### 配置Docker加速器
```
cat > /etc/docker/daemon.json << EOF
{
    "builder": {
        "gc": {
            "defaultKeepStorage": "20GB",
            "enabled": true
        }
    },
    "data-root": "/var/lib/docker",
    "debug": false,
    "default-ulimits": {
        "core": {
            "Hard": -1,
            "Name": "core",
            "Soft": -1
        },
        "nofile": {
            "Hard": 65535,
            "Name": "nofile",
            "Soft": 65535
        },
        "nproc": {
            "Hard": 65535,
            "Name": "nproc",
            "Soft": 65535
        }
    },
    "exec-opts": [
        "native.cgroupdriver=systemd"
    ],
    "experimental": false,
    "features": {
        "buildkit": true
    },
    "icc": false,
    "insecure-registries": [
        "0.0.0.0/0"
    ],
    "log-driver": "json-file",
    "log-opts": {
        "max-file": "5",
        "max-size": "100m"
    },
    "max-concurrent-downloads": 20,
    "max-concurrent-uploads": 10,
    "registry-mirrors": [
        "https://i3jtbyvy.mirror.aliyuncs.com"
    ],
    "storage-driver": "overlay2",
    "storage-opts": [
        "overlay2.override_kernel_check=true"
    ],
    "userland-proxy": false
}
EOF
```

### 创建分区
```
systemctl stop docker
rm -rf /var/lib/docker/*
echo "/dev/sdb /var/lib/docker xfs defaults 0 0" >> /etc/fstab
mount -a
systemctl start docker
```

### 配置Docker代理
```
mkdir -p /etc/systemd/system/docker.service.d
cat > /etc/systemd/system/docker.service.d/http-proxy.conf <<EOF
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:8118"
EOF

cat > /etc/systemd/system/docker.service.d/https-proxy.conf <<EOF
[Service]
Environment="HTTPS_PROXY=http://127.0.0.1:8118"
EOF

systemctl daemon-reload
systemctl restart docker
```

### 启动Docker
```
systemctl enable docker.service
```

### 测试Docker代理是否成功
拉取下来则成功，否则失败

```
systemctl show --property=Environment docker
# docker pull k8s.gcr.io/pause:3.1
```

## 安装Kubeadm
```
apt-get update && apt-get install -y apt-transport-https curl
curl --proxy http://127.0.0.1:8118 -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get -o Acquire::http::proxy="http://127.0.0.1:8118" update
# 最新版本安装
apt-get -o Acquire::http::proxy="http://127.0.0.1:8118" install -y kubelet kubeadm kubectl

# 指定版本安装
VERSION=1.11.3
apt-get -o Acquire::http::proxy="http://127.0.0.1:8118" install -y kubeadm=$(apt-cache  madison kubeadm | awk "/${VERSION}/{print \$3}") kubelet=$(apt-cache  madison kubeadm | awk "/${VERSION}/{print \$3}") kubectl=$(apt-cache  madison kubectl | awk "/${VERSION}/{print \$3}")
```

## 初始化集群
```
systemctl disable ufw
systemctl stop ufw
swapoff -a
systemctl enable kubelet
systemctl start kubelet
# 下载镜像，高版本才有此命令
kubeadm config images pull
# 初始化集群，修改--apiserver-advertise-address为主机IP地址
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=172.17.0.11 | tee -a kubeadm.log


# 如果出现报错之后执行下面的命令再次初始化即可（原因是kubeadm使用的代理访问本地的API-Server导致无法访问，取消代理之后重新初始化即可）
unset http_proxy https_proxy
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=172.17.0.11 --ignore-preflight-errors=all | tee -a kubeadm.log
```
## 后续操作
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
## 创建网络
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
## 创建Dashboard
### 部署Dashboard Service
```
wget https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml -O kubernetes-dashboard.yaml
```
将最后部分的端口暴露修改如下

```
# ------------------- Dashboard Service ------------------- #

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  type: NodePort
  ports:
    - name: dashboard-tls
      port: 443
      targetPort: 8443
      nodePort: 30000
      protocol: TCP
  selector:
    k8s-app: kubernetes-dashboard
```

然后执行 `kubectl create -f kubernetes-dashboard.yaml` 即可

### 创建Dashboard Admin  账户
```
kubectl create sa dashboard-admin -n kube-system
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
# 查看登陆Token
kubectl describe secret -n kube-system $(kubectl get secrets -n kube-system | awk '/dashboard-admin/{print $1}') | awk '/^token/{print $2}'
```

### 访问Dashboard
```
https://172.17.0.11:30000
```

## 配置命令提示
```
echo "source <(kubectl completion bash)" >> ~/.bashrc
echo "source <(kubeadm completion bash)" >> ~/.bashrc
```


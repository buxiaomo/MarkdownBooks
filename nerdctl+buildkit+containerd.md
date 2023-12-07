# nerdctl + buildkit + containerd 构建镜像

本文档安装软件均为二进制部署

## containerd

部署 containerd

```
apt-get install git iptables -y

# runc
wget https://github.com/opencontainers/runc/releases/download/v1.1.10/runc.amd64 -O /usr/local/bin/runc
chmod +x /usr/local/bin/runc

# cni
wget https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz -O /tmp/cni-plugins-linux-amd64-v1.4.0.tgz
mkdir -p /opt/cni/bin
tar -zxf /tmp/cni-plugins-linux-amd64-v1.4.0.tgz  -C /opt/cni/bin/

# containerd
wget https://github.com/containerd/containerd/releases/download/v1.7.10/containerd-1.7.10-linux-amd64.tar.gz -O /tmp/containerd-1.7.10-linux-amd64.tar.gz
tar -zxf /tmp/containerd-1.7.10-linux-amd64.tar.gz --strip-components=1 -C /usr/local/bin
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -O /etc/systemd/system/containerd.service
mkdir -p /etc/containerd/
touch /etc/containerd/config.toml
containerd config dump
sed -i 's|root = "/var/lib/containerd"|root = "/data/containerd"|g' /etc/containerd/config.toml
sed -i 's|sandbox_image = "registry.k8s.io/pause:3.8"|sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.8"|g' /etc/containerd/config.toml
systemctl daemon-reload
systemctl restart containerd.service
```

## buildkit

安装 buildkit

```
wget https://github.com/moby/buildkit/releases/download/v0.12.4/buildkit-v0.12.4.linux-amd64.tar.gz -O /tmp/buildkit-v0.12.4.linux-amd64.tar.gz
tar -zxf /tmp/buildkit-v0.12.4.linux-amd64.tar.gz --strip-components=1 -C /usr/local/bin
wget https://raw.githubusercontent.com/moby/buildkit/master/examples/systemd/system/buildkit.service -O /etc/systemd/system/buildkit.service
wget https://raw.githubusercontent.com/moby/buildkit/master/examples/systemd/system/buildkit.socket -O /etc/systemd/system/buildkit.socket
systemctl daemon-reload
systemctl restart buildkit.service
```

## nerdctl

安装 nerdctl

```
wget https://github.com/containerd/nerdctl/releases/download/v1.4.0/nerdctl-1.4.0-linux-amd64.tar.gz -O /tmp/nerdctl-1.4.0-linux-amd64.tar.gz
tar -zxf /tmp/nerdctl-1.4.0-linux-amd64.tar.gz -C /usr/local/bin
```

## 支持异构

```
nerdctl run --privileged --rm tonistiigi/binfmt --install all
ls -1 /proc/sys/fs/binfmt_misc/qemu*
```

## 构建镜像

```
nerdctl build --platform=amd64,arm64 --output type=image,name=buxiaomo/kubeasy:latest,push=true .
```
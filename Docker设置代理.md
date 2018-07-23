# CentOS  
## 安装相关服务
```
yum install epel-release ca-certificates -y
yum install python-pip wget vim pcre-devel zlib-devel autoconf gcc make -y
pip install shadowsocks
```

## 配置 sslocal
```
cat > /etc/systemd/system/sslocal.service << EOF
[Unit]
Description=Shadowsocks Client
After=network.target

[Service]
Type=simple
Environment="SS_ADDR=xxx.xxx.xxx.xxx"
Environment="SS_PROT=xxx"
Environment="SS_PASS=xxx.xx.xxx"
Environment="SS_MODE=aes-256-cfb"
ExecStart=/usr/bin/sslocal -s \${SS_ADDR} -p \${SS_PROT} -k \${SS_PASS} -m \${SS_MODE} -b 127.0.0.1
ExecStop=/bin/kill -SIGTERM
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF
```
## 安装 proxychains
```
wget https://jaist.dl.sourceforge.net/project/proxychains-ng/proxychains-ng-4.12.tar.xz -O /usr/local/src/proxychains-ng-4.12.tar.xz
tar -Jvxf /usr/local/src/proxychains-ng-4.12.tar.xz -C /usr/local/src
./configure
make && make install
cat > /etc/proxychains.conf << EOF
strict_chain
proxy_dns
tcp_read_time_out 15000
tcp_connect_time_out 8000
[ProxyList]
socks5     127.0.0.1 1080
EOF
```

## 安装 privoxy
```
useradd privoxy
wget https://jaist.dl.sourceforge.net/project/ijbswa/Sources/3.0.24%20%28stable%29/privoxy-3.0.24-stable-src.tar.gz -O /usr/local/src/privoxy-3.0.24-stable-src.tar.gz
tar -zvxf /usr/local/src/privoxy-3.0.24-stable-src.tar.gz -C /usr/local/src
cd /usr/local/src/privoxy-3.0.24-stable
autoheader && autoconf
./configure
make && make install
echo "forward-socks5t   /               127.0.0.1:1080 ." >> /usr/local/etc/privoxy/config
cat > /etc/systemd/system/privacy.service << EOF
[Unit]
Description=Privacy enhancing HTTP Proxy

[Service]
Environment=PIDFILE=/var/run/privoxy.pid
Environment=OWNER=privoxy
Environment=CONFIGFILE=/usr/local/etc/privoxy/config
Type=forking
PIDFile=/var/run/privoxy.pid
ExecStart=/usr/local/sbin/privoxy --pidfile $PIDFILE --user $OWNER $CONFIGFILE
ExecStopPost=/bin/rm -f $PIDFILE
SuccessExitStatus=15

[Install]
WantedBy=multi-user.target
EOF
```
## 重载配置文件
```
systemctl daemon-reload
systemctl restart sslocal.service
systemctl enable sslocal.service
systemctl restart privoxy.service
systemctl enable privoxy.service
```

## 使用
```
export http_proxy=http://localhost:8118;export https_proxy='http://localhost:8118'
curl -v www.google.com
unset http_proxy https_proxy
proxychains4 curl -v www.google.com
```

## Docker配置代理
```
mkdir -p /etc/systemd/system/docker.service.d
cat > /etc/systemd/system/docker.service.d/http-proxy.conf <<EOF
[Service]
Environment="HTTP_PROXY=http://localhost:8118"
EOF

cat > /etc/systemd/system/docker.service.d/https-proxy.conf <<EOF
[Service]
Environment="HTTP_PROXY=http://localhost:8118"
EOF

systemctl daemon-reload
systemctl restart docker
```

## 测试Docker代理是否正常
```
docker pull k8s.gcr.io/pause-amd64:3.1
```

# Ubuntu
## 安装相关服务
```
apt install shadowsocks proxychains privoxy -y
```

## 配置proxychains
```
sed -i "s/socks4.*127.0.0.1.*9050/socks5     127.0.0.1 1080/g" /etc/proxychains.conf
```

## 配置privoxy
```
echo "forward-socks5t   /               127.0.0.1:1080 ." >> /etc/privoxy/config
```

## 配置sslocal
```
cat > /etc/systemd/system/sslocal.service << EOF
[Unit]
Description=Shadowsocks Client
After=network.target

[Service]
Type=simple
Environment="SS_ADDR=xxx.xxx.xxx.xxx"
Environment="SS_PROT=xxx"
Environment="SS_PASS=xxx.xx.xxx"
Environment="SS_MODE=aes-256-cfb"
ExecStart=/usr/bin/sslocal -s \${SS_ADDR} -p \${SS_PROT} -k \${SS_PASS} -m \${SS_MODE} -b 127.0.0.1
ExecStop=/bin/kill -SIGTERM
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF
```
## 重载配置文件
```
systemctl daemon-reload
systemctl restart sslocal.service
systemctl enable sslocal.service
systemctl restart privoxy.service
systemctl enable privoxy.service
```
## 使用
```
export http_proxy=http://localhost:8118;export https_proxy='http://localhost:8118'
curl -v www.google.com
unset http_proxy https_proxy
proxychains curl -v www.google.com
```

## Docker配置代理
```
mkdir -p /etc/systemd/system/docker.service.d
cat > /etc/systemd/system/docker.service.d/http-proxy.conf <<EOF
[Service]
Environment="HTTP_PROXY=http://localhost:8118"
EOF

cat > /etc/systemd/system/docker.service.d/https-proxy.conf <<EOF
[Service]
Environment="HTTP_PROXY=http://localhost:8118"
EOF

systemctl daemon-reload
systemctl restart docker
```

## 测试Docker代理是否正常
```
docker pull k8s.gcr.io/pause-amd64:3.1
```
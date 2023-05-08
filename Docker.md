# 二进制安装Docker
```
wget https://download.docker.com/linux/static/stable/x86_64/docker-23.0.5.tgz -O /usr/local/src/docker-23.0.5.tgz

tar -zvxf /usr/local/src/docker-23.0.5.tgz --strip-components=1 /usr/local/bin

cat > /etc/systemd/system/docker.service<<EOF
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Slice=podruntime.slice
ExecStartPre=/usr/bin/env iptables -P FORWARD ACCEPT
ExecStart=/usr/local/bin/dockerd
ExecReload=/bin/kill -s HUP \$MAINPID
TimeoutSec=0
RestartSec=2
Restart=always
StartLimitBurst=3
StartLimitInterval=60s
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl restart docker.service
systemctl enable docker.service
```
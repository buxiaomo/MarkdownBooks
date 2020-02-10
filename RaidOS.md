# 软Raid Nas系统
## 目录规范

| type  | local  |  nfs | smb | 备注|
|---|---|---|---|---|
|  docker | yes  | no  |no|docker数据目录|
|  appdata | yes  |  no |no|docker容器数据目录|
|  vmware | no  |  yes | no |虚拟化数据数据目录|
|  isos | no  |  yes | no |系统镜像|
|  movies | no  |  no | yes | 电影|
|  teleplay | no  |  no | yes | 电视剧|
|  users | no  |  no | yes | 授权用户家目录|

## 安装系统
系统版本: `Ubuntu 16.04.6 LTS`

步骤: 略

## 配置网络

### 设置主机名

```
root@RaidOS:~# hostnamectl set-hostname RaidOS
root@RaidOS:~# echo RaidOS > /etc/hostname
```

### 单网卡
```
root@RaidOS:~# cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# 修改的部分
auto enp1s0
iface enp1s0 inet static
address 172.16.1.10
netmask 255.255.0.0
gateway 172.16.1.1
dns-nameserver 172.16.1.1
```

### 多网卡Bond(Moode=6)
#### 加载Bonding模块
```
echo "bonding" >> /etc/modules
```

#### 配置网络
```
root@RaidOS:~# cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto enp1s0
iface enp1s0 inet manual
bond-master bond0

auto enp4s0f0
iface enp4s0f0 inet manual
bond-master bond0

auto enp4s0f1
iface enp4s0f1 inet manual
bond-master bond0

auto bond0
iface bond0 inet static
address 172.16.1.10
netmask 255.255.0.0
network 172.16.0.0
broadcast 172.16.255.255
gateway 172.16.1.1
dns-nameservers 172.16.1.1
bond-mode 6
bond-miimon 100
bond-lacp-rate 1
bond-slaves enp1s0 enp4s0f0 enp4s0f1
```

## 创建Raid 5

### 磁盘分区

```

root@RaidOS:~# printf 'n\n\n\n\n\n\nw\n' | fdisk /dev/sda
root@RaidOS:~# printf 'n\n\n\n\n\n\nw\n' | fdisk /dev/sdb
root@RaidOS:~# printf 'n\n\n\n\n\n\nw\n' | fdisk /dev/sdc
root@RaidOS:~# printf 'n\n\n\n\n\n\nw\n' | fdisk /dev/sdd
root@RaidOS:~# lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda       8:0    0  3.7T  0 disk
└─sda1    8:1    0  3.7T  0 part
sdb       8:16   0  3.7T  0 disk
└─sdb1    8:17   0  3.7T  0 part
sdc       8:32   0  3.7T  0 disk
└─sdc1    8:33   0  3.7T  0 part
sdd       8:48   0  3.7T  0 disk
└─sdd1    8:49   0  3.7T  0 part
sde       8:64   1 14.3G  0 disk
└─sde1    8:65   1 14.3G  0 part  /
```

### 安装软件包

```
root@RaidOS:~# apt-get install mdadm -y
```

### 创建磁盘整列

```
root@RaidOS:~# mdadm --create /dev/md5 --level=5 --raid-devices=4 /dev/sda1 /dev/sdb1 /dev/sdc1 /dev/sdd1
```
### 查看整列状态

建议等待状态为 `clean` 在进行下一步.

```
root@RaidOS:~# mdadm --detail /dev/md5
/dev/md5:
        Version : 1.2
  Creation Time : Sun Feb  9 02:18:39 2020
     Raid Level : raid5
     Array Size : 11720658432 (11177.69 GiB 12001.95 GB)
  Used Dev Size : 3906886144 (3725.90 GiB 4000.65 GB)
   Raid Devices : 4
  Total Devices : 4
    Persistence : Superblock is persistent

  Intent Bitmap : Internal

    Update Time : Tue Feb 11 00:12:46 2020
          State : clean
 Active Devices : 4
Working Devices : 4
 Failed Devices : 0
  Spare Devices : 0

         Layout : left-symmetric
     Chunk Size : 512K

           Name : RaidOS:5  (local to host RaidOS)
           UUID : 68f92c96:5f2498b9:448b214a:ed64ef81
         Events : 4870

    Number   Major   Minor   RaidDevice State
       0       8        1        0      active sync   /dev/sda1
       1       8       17        1      active sync   /dev/sdb1
       2       8       33        2      active sync   /dev/sdc1
       4       8       49        3      active sync   /dev/sdd1
```

查看详细进度

```
root@RaidOS:~# cat /proc/mdstat

Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10]
md0 : active raid5 sde[4] sdd[2] sdc[1] sdb[0]
      31432704 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/3] [UUU_]
      [==>..................]  recovery = 11.2% (1174268/10477568) finish=1.3min speed=117426K/sec

unused devices: <none>
```

### 保存整列配置
```
root@RaidOS:~# echo "DEVICE /dev/sda1 /dev/sdb1 /dev/sdc1 /dev/sdd1" >> /etc/mdadm/mdadm.conf
root@RaidOS:~# mdadm -Ds >> /etc/mdadm.conf
root@RaidOS:~# update-initramfs -u
```

## 创建挂载点和文件系统

`UUID` 可通过 `blkid` 查看

```
root@RaidOS:~# mkfs.ext4 /dev/md5
root@RaidOS:~# mkdir /mnt/user
root@RaidOS:~# echo "UUID=9e2081d3-8ca2-4c86-b748-df2a8644d56d /mnt/user ext4 defaults,noatime,nodiratime,nofail 0 0" >> /etc/fstab
root@RaidOS:~# mount -a
root@RaidOS:~# df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            7.7G     0  7.7G   0% /dev
tmpfs           1.6G   19M  1.6G   2% /run
/dev/sde1        14G  2.7G   11G  21% /
tmpfs           7.7G     0  7.7G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           7.7G     0  7.7G   0% /sys/fs/cgroup
/dev/md5         11T  728G  9.6T   7% /mnt/user
```

## 配置共享服务

### SMB

#### 安装

```
root@RaidOS:~# apt-get install samba -
```

#### 配置

```
root@RaidOS:~# cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
root@RaidOS:~# cat /etc/samba/smb.conf
[global]
	workgroup = WORKGROUP
	netbios name = RaidOS
	disable netbios = no
	log file = /var/log/samba/%m.log
	server string = Media server
	max log size = 1000
	security = USER
	hide dot files = yes
	os level = 100
	preferred master = yes
	map to guest = Bad User
	null passwords = Yes
	map archive = No
	map hidden = No
	map system = No
	map readonly = Yes
	create mask = 0664
	directory mask = 0755
	passdb backend = smbpasswd
	encrypt passwords = Yes
	null passwords = Yes
	local master = yes
	syslog only = yes
	syslog = 0
	include = /etc/samba/%U.smb.conf

# [Home]
# 	comment = Home Directories
# 	browseable = yes
# 	read only = yes
# 	directory mask = 0755
# 	create mask = 0644
# 	writable = yes
# 	valid users = %S

[Movies]
	path = /mnt/user/movies
	comment = 电影
	browseable = yes
	public = yes
	writeable = yes
	oplocks = yes

[Teleplay]
	path = /mnt/user/teleplay
	comment = 电视剧
	browseable = yes
	public = yes
	writeable = yes
	oplocks = yes

[Book]
	path = /mnt/user/book
	comment = 电子书
	browseable = yes
	public = yes
	writeable = yes
	oplocks = yes

[ISO]
	path = /mnt/user/isos
	comment = 系统镜像
	browseable = yes
	public = yes
	writeable = yes
	oplocks = yes

;[print$]
;   comment = Printer Drivers
;   path = /var/lib/samba/printers
;   browseable = yes
;   read only = yes
;   guest ok = no
;   write list = root, @lpadmin
```

#### 创建共享目录

```
root@RaidOS:~# mkdir -p /mnt/user/{appdata,book,docker,isos,movies,teleplay,users,vmware}
root@RaidOS:~# chown -R nobody.nogroup /mnt/user/{appdata,book,isos,movies,teleplay,users,vmware}
root@RaidOS:~# chmod -R 755 /mnt/user/{appdata,book,isos,movies,teleplay,users,vmware}
```

#### 启动服务

```
root@RaidOS:~# systemctl restart smbd.service
root@RaidOS:~# systemctl enable smbd.service
```

### NFS

#### 安装

```
root@RaidOS:~# apt-get install nfs-kernel-server -y
```

#### 配置

```
root@RaidOS:~# cat /etc/exports
# /etc/exports: the access control list for filesystems which may be exported
#		to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
#
"/mnt/user/isos" -async,no_subtree_check,fsid=100 *(sec=sys,rw,insecure,anongid=65534,anonuid=65534,all_squash)
"/mnt/user/kubernetes" -async,no_subtree_check,fsid=103 *(sec=sys,rw,insecure,anongid=65534,anonuid=65534,all_squash)
"/mnt/user/vmware" -async,no_subtree_check,fsid=101 *(sec=sys,rw,insecure,anongid=65534,anonuid=65534,all_squash)
```

#### 启动服务

```
root@RaidOS:~# systemctl restart nfs-server.service
root@RaidOS:~# systemctl enable nfs-server.service
```

## 安装配置Docker

```
root@RaidOS:~# curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
root@RaidOS:~# cat /etc/docker/daemon.json
{
  "data-root": "/mnt/user/docker",
  "registry-mirrors": [
    "https://i3jtbyvy.mirror.aliyuncs.com"
  ],
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "500m",
    "max-file": "2"
  }
}
```

## 测试

### IO测试

#### 读

```
root@RaidOS:~# hdparm -tT --direct /dev/md5
```

#### 写

```
root@RaidOS:~# dd if=/dev/zero of=/mnt/user/test.iso bs=1024M count=1 conv=fdatasync
```

## 参考文献

* [RAID详解](https://www.dwhd.org/20150522_103540.html)
* [Samba服务](https://www.jianshu.com/p/650dda31b62e)

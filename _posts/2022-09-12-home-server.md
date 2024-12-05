# 家庭服务器

## 起因
为了不再忍受Esxi突然断电导致的，直通硬盘无法加载而无法启动虚拟机等问题，本人决定将HPE Proliant MicroServer Gen8重装Proxmox VE（下称PVE）系统。

更新：加了一台成品NAS，双轨运行。

## 参考手册
官方提供了比较详尽的手册，涉及安装过程、维护操作，必读。

## 存储
重建zfs过程中发现硬盘挂掉了一个， 运行
> sudo zpool scrub [pool]

也无法修复，于是参考了其他人的容灾做法，决定将原本的zfs mirror的故障硬盘换成新的，并使用FreeFileSync同步两盘数据。

## 重建过程注意事项
1. PVE硬盘直通：
> qm set [vmid] -scsi[n] [dev]<br>
> vmid为虚拟机id、n为该虚拟机不重复的scsi设备编号、dev为设备文件如/dev/disk/by-id/xxx
2. 将二磁盘直通虚拟机，格式化新的硬盘，最新的Kubuntu支持NTFS，于是就用了NTFS。
   ~~用KDE分区管理工具可挂载NTFS，~~ fstab参数（重启生效）：
> UUID=XXXX /MOUNT/POINT ntfs3 uid=1000,gid=1000,dmask=022,fmask=133 0 0 #（ntfs3为官方ntfs驱动，最后一个pass选项设为0避免因掉盘造成系统无法启动）
3. ~~自带的kdenetwork-filesharing可设置samba分享。如果samba未安装，在dolphin点击后也可以在apt安装日志看到利用packagekit安装samba的信息。~~
   配置/etc/samba/smb.conf
```text
[sync]
   path = /MOUNT/POINT
   browsable = yes
   guest ok = no
   read only = yes
[path]
   path = /MOUNT/POINT
   browsable = yes
   guest ok = no
   write list = username
   create mask = 0644
   directory mask = 0755
```
4. 安装FreeFileSync进行同步（或者用成品NAS同步），~~Kubuntu需要安装libgtk2.0-0，~~ apt安装日志会在/var/log/apt/history.log(或zless /var/log/apt/history.log.1.gz)。
5. FreeFileSync建议定时同步，Kubuntu（KDE）可在Discover（snap）安装KCron，设置好后还需要再运行crontab -e在执行的命令前加
> DISPLAY=:0<br>
> :0为在桌面终端运行的echo $DISPLAY<br>
> 效果如<br>
> 0 0 * * * DISPLAY=:0 /home/lwq/FreeFileSync/FreeFileSync /home/lwq/BatchRun.ffs_batch

6. OneDrive备份 https://rclone.org/install/
> sudo usermod -aG docker $(id -un)<br>
> newgrp docker<br>
> docker pull rclone/rclone:latest
> mkdir -p ~/config/dir<br>
> docker run --rm -it --volume ~/config/dir:/config/rclone --user $(id -u):$(id -g) rclone/rclone config<br>
> mkdir -p ~/data/dir<br>
> docker run -it --volume ~/config/dir:/config/rclone --volume ~/data/dir:/data --user $(id -u):$(id -g) rclone/rclone copy lwq: /data --log-level INFO

同样可以加到cron中执行
> docker container start [rclone container id]

iCloud用户可以用icloudpd https://github.com/boredazfcuk/docker-icloudpd

## 网络

以下仅供参考，本人已转RouterOS。

### VLAN

为了实现部分虚拟机隔离到DMZ，需要用到VLAN，同时配合OpenWrt实现。

OpenWrt VLAN配置：

![VLAN](https://github.com/lvv9/lvv9.github.io/blob/master/pic/vlan.png?raw=true)

以下PVE修改需要重启生效，或安装ifupdown2才能在页面上apply。

修改IP地址:/etc/network/interfaces

PVE网络接口：
```text
auto lo
iface lo inet loopback

iface eno1 inet manual

iface eno1.1 inet manual

auto vmbr1
iface vmbr1 inet static
address 10.10.10.2
netmask 255.255.255.0
gateway 10.10.10.1
bridge_ports eno1.1
bridge_stp off
bridge_fd 0

auto vmbr0
iface vmbr0 inet manual
bridge_ports eno1
bridge_stp off
bridge_fd 0
```
或许是打开方式不对，新增eno2无法直接配置静态的IP使用。

DMZ虚拟机的网络配置中使用vmbr0 tag=3。

### IP
按照前面方式划分了两个网络，只使用传统的IPv4 NAT配置起来比较简单：
1. 新增类似于lan的接口，关联eth0.3设备
2. 网络设置为不同于lan的网络如192.168.2.1/24
3. 加入到lan区的netfilter规则（临时加入，建议建一个新的，见下）

但IPv6就复杂多了，互联网上对这个的讨论都比较粗浅，而且网络环境各不相同。

本文目前所处环境（可提前生效wan、wan6配置试得环境）：
分配了IPv6前缀（IPv6-PD），但分配的前缀是64位的，也就是说只能分配在一个子网。

配置/etc/config/network：
```text
config globals 'globals'
	option ula_prefix 'fd99:ca91:0a5d::/48'

config interface 'wan'
	option device 'eth0.2'
	option proto 'pppoe'
	option username 'xxx'
	option password 'yyy'
	option ipv6 '1'
	option delegate '0'

config interface 'wan6'
	option proto 'dhcpv6'
	option device '@wan'
	option reqaddress 'try'
	option reqprefix 'auto'
	option delegate '1'

config interface 'lan'
	option device 'br-lan'
	option proto 'static'
	option ipaddr '192.168.1.1'
	option netmask '255.255.255.0'
	option ip6assign '64'
	option delegate '0'

config interface 'dmz'
	option device 'eth0.3'
	option proto 'static'
	option ipaddr '192.168.2.1'
	option netmask '255.255.255.0'
	option ip6assign '64'
	option ip6weight '1'
	option delegate '0'
```
- ULA类似于IPv4的局域网地址
- wan接口ipv6选项配置为auto时会自动生成一个接口，'1'时使用自定义的（wan6即@wan）
- 由于ISP分配的的GUA前缀是64位的，因此这里无法配置不同网络的GUA，因而这里使用dhcp配置（/etc/config/dhcp）的server（SLAAC only）、relay中继模式分别对两个不同的接口分配IP，中继模式跨路由器传递IP相关的管理信息。
  但是relay后始终不能把分配的IP加到路由（ https://github.com/openwrt/openwrt/issues/11599 ），就把ip6weight选项加上让前缀在dmz接口分配
- 后面无意中发现无线客户端接入上游可以正常relay
```text
config dhcp 'wan'
	option interface 'wan'
	option ignore '1'

config dhcp 'wan6'
	option interface 'wan6'
	option ignore '1'
	option master '1'
	option ra 'relay'
	option dhcpv6 'relay'
	option ndp 'relay'

config dhcp 'lan'
	option interface 'lan'
	option start '100'
	option limit '150'
	option leasetime '12h'
	option dhcpv4 'server'
	option dhcpv6 'relay'
	option ra 'relay'
	option netmask '255.255.255.0'
	option ndp 'relay'

config dhcp 'dmz'
	option interface 'dmz'
	option start '100'
	option limit '150'
	option leasetime '12h'
	option ra 'server'
	option dhcpv6 'disabled'
	option ndp 'disabled'
	list ra_flags 'none'
```
配置生效后dmz接口得到一个ULA地址、一个从前缀计算出来的GUA，
与dmz接口连接的设备得到ULA、GUA。
lan接口得到一个ULA。

### netfilter
为了深入了解防火墙，特地找本教材学习了一下，ISBN：9787302278863。

### DDNS
这里选用dynv6.com的DDNS服务：界面简洁、支持子域、API丰富（支持AAAA记录）、免费。
> crontab -e 定时执行以下命令更新<br>
> curl "https://ipv6.dynv6.com/api/update?ipv6=auto&token=...&zone=..."

### WireGuard
https://wiki.debian.org/WireGuard

### MC801A
MC801A是中兴的一款5G CPE，在存在下级路由的情况下需要手动设置APN来设置桥模式，除了名称其它参数保持与自动一样：
https://community.three.co.uk/t5/Broadband/ZTE-MC801A-B12-update-Bridge-mode-broken/td-p/5860/highlight/true/page/3

## Hyper-V
更新：
自MicroServer Gen8入手以来已经过去了接近10年，硬件也变得有些过时。在最近的检测中发现待机功率接近50W，于是进行了全面的升级。
鉴于MicroServer或者类似的服务器价格上也“升级”了许多，且当下桌面级机器也普遍过剩，于是选用了某个NUC——带两个2280 M.2接口、一个2.5 SATA。
在新机器试用Hyper-V，以下是一些注意事项。

### 安装
为了快速地安装Win，可能需要跳过网络连接、OneDrive登录等。
跳过网络连接可能导致安装完成后没有网卡驱动，可以先准备好网卡驱动。

### 电源
由于Headless桌面PC不像之前的服务器那样，可以进行方便的Web管理，而用远程桌面进行操作，但是Win 11的电源策略会导致睡眠后无法连接，可以把睡眠时间改成永不。

### 更新
防止服务被更新中断，设置了更新策略为通知下载，通过组策略编辑器修改。

### 硬盘直通
Hyper-V也支持硬盘直通，这次将硬盘格式化为了exFAT。
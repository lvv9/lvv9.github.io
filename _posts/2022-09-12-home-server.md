# 家庭服务器

## 起因
为了不再忍受Esxi突然断电导致的，直通硬盘无法加载而无法启动虚拟机等问题，本人决定将HPE Proliant MicroServer Gen8重装Proxmox VE（下称PVE）系统。

## 参考手册
官方提供了比较详尽的手册，涉及安装过程、维护操作，必读。

## 存储
重建zfs过程中发现硬盘挂掉了一个， 运行
> sudo zpool scrub [pool]

也无法修复，于是参考了其他人的容灾做法，决定将原本的zfs mirror的故障硬盘换成新的，并使用FreeFileSync同步两盘数据。

## 重建过程注意事项
1. PVE硬盘直通：
> qm set [vmid] -scsi[n] [dev]<br>
> vmid为虚拟机id、n为不重复的scsi设备编号、dev为设备文件如/dev/sdb
2. 将二磁盘直通虚拟机，格式化新的硬盘，最新的Kubuntu支持NTFS，于是就用了NTFS
   用KDE分区管理工具可挂载NTFS，参数（重启生效）：
> uid=1000,gid=1000,dmask=022,fmask=133
> 自带的kdenetwork-filesharing可设置samba分享
3. 安装FreeFileSync进行同步，Kubuntu需要安装libgtk2.0-0，安装后日志会在/var/log/apt/history.log
4. FreeFileSync建议定时同步，Kubuntu（KDE）可在Discover（snap）安装KCron，设置好后还需要再运行crontab -e在执行的命令前加
> DISPLAY=:0<br>
> :0为在桌面终端运行的echo $DISPLAY<br>
> 效果如<br>
> 0 0 * * * DISPLAY=:0 /home/lwq/FreeFileSync/FreeFileSync /home/lwq/BatchRun.ffs_batch

5. OneDrive备份 https://rclone.org/install/
> sudo usermod -aG docker $(id -un)<br>
> newgrp docker<br>
> docker pull rclone/rclone:latest
> mkdir -p ~/config/dir<br>
> docker run --rm -it --volume ~/config/dir:/config/rclone --user $(id -u):$(id -g) rclone/rclone config<br>
> mkdir -p ~/data/dir<br>
> docker run -it --volume ~/config/dir:/config/rclone --volume ~/data/dir:/data --user $(id -u):$(id -g) rclone/rclone copy lwq: /data --log-level INFO

同样可以加到cron中执行
> docker container start [rclone]

6. 网络

以下修改需要重启生效，或安装ifupdown2才能在页面上apply。

修改ip地址:/etc/network/interfaces

为了实现部分虚拟机隔离到DMZ，需要用到VLAN，同时配合OpenWrt实现。

OpenWrt VLAN配置：

![VLAN](https://github.com/lvv9/lvv9.github.io/blob/master/pic/vlan.png?raw=true)

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
或许是打开方式不对，新增eno2无法直接使用。

DMZ虚拟机使用vmbr0 tag=3。
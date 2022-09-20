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
3. 安装FreeFileSync进行同步，Kubuntu需要安装libgtk2.0-0，安装后日志会在/var/log/apt/history.log
4. FreeFileSync建议定时同步，Kubuntu（KDE）可在Discover（snap）安装KCron，设置好后还需要再运行crontab -e在执行的命令前加
> DISPLAY=:0<br>
> :0为在桌面终端运行的echo $DISPLAY<br>
> 效果如<br>
> 0 0 * * * DISPLAY=:0 /home/lwq/FreeFileSync/FreeFileSync /home/lwq/BatchRun.ffs_batch
# 数码玩具
最近剁手了一些大件——电视、NAS、NUC，也配了一些小件、软件，期望能整出一些花来。

## DisplayLink
严格来说这不是一个玩具。因为单位配的笔记本显示输出口只有一个，不能再接两个显示器实现多屏（三屏）异显。
DisplayLink扩展坞基本上就是一个USB 3.0显卡，是一种便宜又不太便宜的产品——便宜的显卡、不太便宜的扩展坞。这里本人选用的是海备思的。

## 无线投屏器（发射+接收）
在NUC服务器上安装系统时最需要的，因服务器通常是放在犄角旮旯的位置，遇到一些问题用HDMI线连接显示器解决时就变得非常不方便。
无线投屏器（免驱），用无线信道代替线缆，帮助解决这个问题。

## 显卡直通/vGPU
这个没玩。
本来打算将虚拟机显示输出（VirtIO）到显示器或者电视上的，奈何串流（Sunshine+Moonlight）支持的是Nvidia、AMD、Intel的编解码，直通、虚拟化又太复杂。
而且vGPU一般是面向企业用户。

## NoMachine
可在多平台上运行的“远程桌面”，用在虚拟机里。Win直接用远程桌面。

## Sunshine+Moonlight+Virtual Display Driver
Sunshine编码并通过网络发送数据，Moonlight接收网络数据并解码，Virtual Display Driver虚拟多一块显示器，让Moonlight画出虚拟的屏幕，软件实现DisplayLink。
由于本人的Steam机器是AMD APU+Nvidia 40系显卡并在UEFI设置了混合，使用最新发布的Sunshine会有Bug（ https://github.com/LizardByte/Sunshine/pull/3002 ）。
可以用预发布的版本。
---
layout: post
title: '懒人教程: 躺在床上使用台式机'
date: 2022-03-12
author: Calvin Wang
cover: '/assets/img/202203/workstations.png'
tags: windows10 wsl2 dev-env
---

> 用 Docker on Win10 WSL2 台式机来扩展 MBP 性能. 演示环境以Spark, Zeppelin 为例.

# 演示
![demo](/assets/img/202203/workstation_example.gif)

# 背景
## 为什么要躺在床上用台式机?
1. 懒! 能躺着绝不坐着!
2. 穷! 目前在使用的是 15 年 MBP, 性能有点堪忧. 家中有一台性能过剩的台式机, 用来配合 MBP 很合适.

## 为什么不直接装linux/双系统?
已经安装了 win10 和 ubuntu20 的双系统.
但是家里的另外一位时不时的就要使用台式机. 看到是 ubuntu会造成不便.
为了家庭和谐还是用 win10 来扩展性能最稳妥.

# 简单结构
![structure](/assets/img/202203/workstations.png)

# 环境搭建
## 配置静态 IP:
1. 获取MAC 地址: 在 powershell 中执行`ipconfig /all`, 记录以太网卡的物理地址.
![mac address](/assets/img/202203/mac-address.png)

2. 在路由器中配置静态 IP, 以华为AX3 为例. (不同的路由器可能有不同的配置方式.)
![route config](/assets/img/202203/route_config.png)

# windows 配置
1. Install WLS2:
在 PowerShell 依次执行下列命令:
```sh
wsl –install # install  wsl
wsl --install -d ubuntu # install ubuntu
wsl -l -v # 查看 WSL 版本
#  NAME                   STATE           VERSION
#* Ubuntu-20.04           Running         2
wsl --set-version Ubuntu-20.04 2  # 版本设置为 2
wsl --set-default Ubuntu-20.04
```
2. Install docker
  * 下载: https://desktop.docker.com/win/main/amd64/Docker%20Desktop%20Installer.exe
  * 双击安装完成

3. 配置 Docker: 略

4. 配置代理:
由于每次重启 wsl 内的机器 ip 都会变换. 可以设置一个脚本在登录以后自动设置代理
* 新建代理配置脚本
```sh
# https://github.com/759803573/etc/blob/master/windows/base/create_portforward_after_login/route_ssh_to_wsl.ps1
wsl.exe sudo /etc/init.d/ssh start   # 启动 SSH
$wsl_ip = (wsl hostname -I).trim()         # 获取 WSL 主机 IP
Write-Host "WSL Machine IP: ""$wsl_ip"""   
netsh interface portproxy add v4tov4 listenport=2022 connectport=2022 connectaddress=$wsl_ip # 配置代理.
```

* 设置自动任务:
通过 windows 系统自带的`任务计划程序`, 来设置脚本当用户登录后自动启动.
已导出计划任务: [click here to download](https://github.com/759803573/etc/blob/master/windows/base/create_portforward_after_login/create%20ssh%20connection%20after%20login.xml)

![scheduler 1](/assets/img/202203/scheduler1.png)
![scheduler 2](/assets/img/202203/scheduler2.png)

# Macbook 配置
主要是 VS Code 的配置.
1. 安装 Extension: remote-ssh
2. Add New Host:
![vscode add host](/assets/img/202203/vs-code-config.png)

# 后续
* 设置一下远程开机:
  由于台式机主板支持的 WOL 不太好, 不能很方便的做到网络环境.
  后续计划配置智能插座. 通过主板的通电自动开机来实现远程开机.

* 关机通过的远程桌面
  
* 天气这么好, 出去玩吧!
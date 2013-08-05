---
layout: post 
title: 在windows7/8上建立WIFI热点
summary: 在windows7/8操作系统上建立WIFI热点
tags: win8,hostednetwork
---

{{ page.title }}
================

* Win+X 以管理员身份打开命令行
* 运行 ``` netsh wlan set hostednetwork mode=allow ssid=ssid_name   key=your_password ```
* 打开网络和共享中心，更改适配器设置
* 选择当前的网络连接，右键-属性
* 选择共享面板，选中"允许其他网络用户通过此计算机的Internet连接来连接",并选择新建的WIFI热点
* 命令行运行 ``` netsh wlan start hostednetwork ```
* 建立完成，可以连接了

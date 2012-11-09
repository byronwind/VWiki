---
layout: post 
title: 使用tcpdump在Android平台抓包
summary: 使用tcpdump在Android平台抓包
tags: tcpdump android
---

{{ page.title }}
================

利用tcpdump可以方便的在android上进行网络数据包的抓取，抓取之后，可以利用wireshark在PC上分析数据文件。

操作步骤：

* root android设备
* 下载tcpdump，利用adb push到 /system/bin路径下 ``` adb push tcpdump /system/bin ```
* 修改tcpdump的可执行权限 ``` chmod 6755 /system/bin/tcpdump ```
* 抓包 
    ``` adb shell su
         tcpdump  -i any -p -s 0 -w /sdcard/capture.pcap
    ```
* 导出日志 ``` adb pull /sdcard/capture.pcap ```
* wireshark 分析

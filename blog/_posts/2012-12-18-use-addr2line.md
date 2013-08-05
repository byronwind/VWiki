---
layout: post 
title: 使用addr2line查看崩溃信息
summary: Android中使用addr2line工具查看native崩溃信息
tags: ndk,addr2line
---

{{ page.title }}
================

addr2line可将指定的虚拟地址(或在可重定位的目标文件中相对节的偏移)转换为在源码中的位置。注意要有调试信息才可。
    
使用：

    ```$addr2line -e a.out 80483C4```


参数：
    -a    在显示函数名或文件行号前显示地址-b bfdname
    -b    指定二进制文件格式
    -C    解析C++符号为用户级的名称,可指定解析样式
    -e filename 指定二进制文件
    -f    同时显示函数名称
    -s    仅显示文件的基本名，而不是完整路径
    -i    展开内联函数
    -j    读取相对于指定节的偏移而不是绝对地址
    -p    每个位置都在一行显示

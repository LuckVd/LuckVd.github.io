---
title: offsec-Lab-Sar
date: 2023-10-09 20:06:42
categories: 
- 靶场WriteUP
tags:
- 靶场WriteUP
- OffSec
cover: ../images/offsec-Lab-Sumo/cover.png
---

## 知识点



## 正文

主页如下

<img src="../images/offsec-Lab-Sar/image-20231009200836846.png" alt="image-20231009200836846" style="zoom:50%;" />

路径扫描扫到robots.txt

![image-20231009201105087](../images/offsec-Lab-Sar/image-20231009201105087.png)

![image-20231009200925621](../images/offsec-Lab-Sar/image-20231009200925621.png)

然后发现存在sar2HTML 。

`sar2HTML` 是一个用于将 Linux 系统性能监控数据（通过 `sar` 命令收集的）转换为可视化的HTML报告的工具。它提供了一个直观的界面，以便用户可以更容易地分析和理解系统的性能表现。

搜索发现存在exp，[sar2html 3.2.1 - 'plot' Remote Code Execution](https://www.exploit-db.com/exploits/49344)

直接执行poc

![image-20231009201131829](../images/offsec-Lab-Sar/image-20231009201131829.png)



![image-20231009201950273](../images/offsec-Lab-Sar/image-20231009201950273.png)

但是这里尝试后发现没办法反弹shell



### 利用php shell反弹shell

在这了发现php可以正常上传，那么可以这里上传，通过刚才的命令行得到上传shenll 的路径。

![image-20231009203538192](../images/offsec-Lab-Sar/image-20231009203538192.png)

看到路径为,http://ip/sar2HTML/sarDATA/uPLOAD/php-reverse-shell.php

![image-20231009203643700](../images/offsec-Lab-Sar/image-20231009203643700.png)

在本地监听，访问该shell后反弹成功。

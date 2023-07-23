---
title: offsec-Lab Sumo
date: 2023-07-23 20:03:28
categories: 
- 靶场WriteUP
tags:
- 靶场WriteUP
- OffSec
cover: ../images/offsec-Lab-Sumo/cover.png
---



首先进行端口扫描

![image-20230723200436532](../images/offsec-Lab-Sumo/image-20230723200436532.png)

路径扫描

`feroxbuster -u http://192.168.181.87/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt`  

![image-20230723200517738](../images/offsec-Lab-Sumo/image-20230723200517738.png)

没扫到东西。然后试试用nuclei扫描。

![image-20230723200631301](../images/offsec-Lab-Sumo/image-20230723200631301.png)

扫到一个EXP： https://github.com/opsxcq/exploit-CVE-2014-6271，试了一下发现可以直接利用

![image-20230723200759961](../images/offsec-Lab-Sumo/image-20230723200759961.png)

尝试反弹shell，反弹成功。

![image-20230723201045105](../images/offsec-Lab-Sumo/image-20230723201045105.png)

直接拿到用户flag

![image-20230723202105328](../images/offsec-Lab-Sumo/image-20230723202105328.png)

## 提权

SUID、定时任务，sudo都没有找到利用方法。看内核时Linux3.2.0，搜索是否存在内核提权。

![image-20230723203927961](../images/offsec-Lab-Sumo/image-20230723203927961.png

![image-20230723210555307](../images/offsec-Lab-Sumo/image-20230723210555307.png)

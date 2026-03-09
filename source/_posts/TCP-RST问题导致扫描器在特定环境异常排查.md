---
title: TCP RST问题导致扫描器在特定环境异常排查
date: 2026-03-09 21:47:12
tags:
  - 扫描器
---
## 问题描述
最近扫描器一个poc在扫描时，遇到了一个诡异的现象，扫描器(linux环境)使用向目标服务器发送的Https POST 请求包总是抛出 `ConnectionResetError`。然而，在win中同样的环境(相同版本的python和requests)包下，却能正常收到 `HTTP/1.1 404 Not Found` 响应。

```python

import requests
requests.post('https://ip:port/events', 
              json={"id": -1}, verify=False)
# 抛出 requests.exceptions.ConnectionError: ('Connection aborted.', ConnectionResetError(104, 'Connection reset by peer'))

```

## 抓包跟踪
```
1  client -> server [SYN] Seq=0
2  server -> client [SYN, ACK] Seq=0 Ack=1
3  client -> server [ACK] Seq=1 Ack=1
4  client -> server [PSH, ACK] Seq=1 Ack=1 Len=517   (HTTP请求)
5  server -> client [ACK] Seq=1 Ack=518
6  server -> client [PSH, ACK] Seq=1 Ack=518 Len=1230 (HTTP响应开始)
... 中间多个数据包 ...
13 client -> server [ACK] Seq=734 Ack=1700
14 client -> server [FIN, ACK] Seq=734 Ack=1700       (客户端主动FIN)
15 server -> client [PSH, ACK] Seq=1700 Ack=735 Len=31 (服务器发31字节数据)
16 client -> server [RST] Seq=735                      (客户端RST)
17 server -> client [FIN, ACK] Seq=1731 Ack=735
18 client -> server [RST] Seq=735

```
从时序上看，现象是这样的：

- 客户端已经开始关闭连接，发送了 `FIN`
    
- 服务端随后又发来了一段 31 字节的数据
    
- 客户端紧接着回了 `RST`
    

继续解析那 31 字节数据，发现它是一个 **TLS Alert** 记录；如果内容类型是 `0x15`，那基本可以确认它属于 TLS Alert。结合上下文，它**很可能**是 `close_notify`，也就是 TLS 层在做正常关闭时发送的告警。OpenSSL 文档明确说明，TLS 关闭流程中通常会发送并接收 `close_notify`；推荐的完整关闭方式是继续读到 EOF/`SSL_ERROR_ZERO_RETURN`，以确保连接末尾的应用数据和关闭通知都被处理到。

##  为是么是RST？

TCP 是**全双工**协议，两个方向是独立关闭的。  
一端发送 `FIN`，只表示：

> **我这一侧以后不再发送数据了**

它**不表示**：

> **对方也不能继续发送数据了**

所以，单从抓包上看到：

- 客户端先发 `FIN`
    
- 服务端后面还有数据发来

虽然 TCP 允许半关闭，但某些实现可以采用更偏“半双工”的关闭策略。对于这种实现，如果应用已经调用了 `CLOSE`，而此时连接里还有尚未读取的数据，或者在 `CLOSE` 之后又收到了新数据，那么 TCP 实现**应该**发送 `RST`，用来明确表明有数据丢失。

这表明：**客户端已经进入关闭流程，而服务端/对端 TLS 层在连接尾部仍有数据要发送；Linux 上这一异常收尾最终表现成了 RST。**


## RST 更像是谁发的：应用还是内核？

为了确认，我们编写了测试脚本，使用不同方式关闭 socket：

- 默认 `close()`
    
- 设置 `SO_LINGER` 为 `{1,0}`（强制 RST）
    
- 设置 `SO_LINGER` 为 `{1,5}`（等待5秒）
    
- 先 `shutdown()` 再 `close()`
    

结果（抓包）显示：**无论哪种关闭方式，只要服务器在 FIN 后发送了数据，客户端都会触发 RST**，且 RST 的序列号与服务器期望的 Ack 一致。
**这表明RST 大概率由客户端主机上的内核 / 底层网络栈根据当前 socket 状态触发，而不是 Python 应用主动构造。这更像是 TLS/HTTP/TCP 收尾阶段的兼容性问题，而不是简单的“HTTP 请求失败”或“服务端完全无响应”。**


## 为什么win下没有报错？

Windows 与 Linux 在连接关闭细节上的实现差异，可能导致同一台服务端、同一段 Python 代码，在两个平台上表现不同。Linux 这里更容易把异常收尾暴露成 reset，而 Windows 则没有表现为相同错误。


## 后记
> 扫描器应对异常服务端更鲁棒

真实环境里，扫描器/探测器确实不能假设：

- 对端 HTTP 一定规范
    
- chunked 一定完整
    
- TLS 一定优雅关闭
    
- 连接一定正常四次挥手
    

所以：

- 捕获异常
    
- 尽早读取关键响应片段
    
- 不把“连接末尾异常”直接等价成“业务检测失败”


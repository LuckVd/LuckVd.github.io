---
title: 'OpenCode 被黑，用户在替黑客的账单买单'
date: 2026-10-19 10:00:00
categories:
  - AI 安全
tags:
  - AI 安全
  - OpenCode
  - AI Agent
  - 密钥泄漏
cover: /images/opencode-breach/hero-vault-keys.png
---

![「金库失守、钥匙外飞」题图（图源：dev.to，gabrielanhaia 文章配图）](/images/opencode-breach/hero-vault-keys.png)

你存在 OpenCode Console 里的 OpenAI、Anthropic、Google API Key，可能已经进了别人的脚本。调用照常发生，额度烧的是你的，账单寄给你。

10 月 17 日，开源编码 Agent OpenCode 官方确认遭遇入侵，1.4% 的 Console 用户当天收到邮件通知：立刻去各厂商后台轮换 Key。

## OpenCode 是什么

OpenCode 是一款开源的终端编码 Agent，在开发者社区里热度不低，日常用法是在命令行里让它读写代码、跑命令。Console 是它背后的云端控制台，主要给团队用：把你自己付费的模型账号接进工作区，成员共享这些模型的调用能力，不用各自去申请 Key。

这次出事的是 Console 这条云端链路，纯 CLI 本地用户不在此范围内。这个范围划分后面还会用到，它直接决定你该不该紧张。

## 经过

官方通知邮件的原话："We identified unauthorized access to a legacy system that we were in the process of decommissioning."——出事的是一套正准备下线的老旧系统，用户存在工作区里的第三方 provider Key（OpenAI、Anthropic 等）可能已暴露。

时间线不长：10 月 16 日夜间入侵发生；17 日 15:30（UTC），安全研究员 Matt Johansen 把通知邮件截图发上 X；两小时后 OpenCode 官方账号确认被入侵。官方给受影响者逐一发信，主题是 "Action Required: Potential provider key breach"，没收到邮件的即不在范围内。公司称正与 Anthropic、OpenAI、Google 协同处置，"事件仍在进行中，暂时无法分享更多细节"。联合创始人 Dax Raad 当天下午透露，他本人的账户也正在被攻击。

损失已经落账：Crypto Briefing 汇总的用户报告显示，有用户账单多出数百美元，被盗 Key 正被用来大量调用模型，费用记在原主人名下。

## 官方没说的

入侵手法、驻留时长、外流 Key 的数量、是明文存储还是加密后被整体拖走，官方一个字都没提。1.4% 是个比例，乘上 Console 用户基数才是规模，而基数没有公开，这个数字暂时换算不出具体的受害人数。

这些空白不影响你今天的应对动作，但影响你对这家公司的信任该给几分。事件仍在进行中，后续披露值得盯。

## 该上威胁模型的角色

1.4% 不是重点，重点是这次事件把一个角色拍在了桌面上：编码 Agent 厂商在替你保管密钥。

Console 的产品设计就是如此。官方文档写明，工作区可以接入你自己付费的 OpenAI、Anthropic 或 AWS Bedrock 账号，Key 存在厂商一侧，团队成员"使用模型而无需见到 provider Key"。对团队协作这是真实的便利：不用人手一把 Key、不用轮换时挨个通知、用量集中可见。

对价同样真实：你的密钥边界延伸到厂商的整套系统，包括一套正准备下线的遗留系统。你的攻击面，从此等于它全部系统的攻击面。它的历史包袱、下线流程，甚至日常的运维纪律，都成了你的风险项。这次出问题的恰好是最不起眼的那一类资产：一套没人再盯着、又还没关干净的 legacy。

![OpenCode 密钥托管链路与攻击面](/images/opencode-breach/fig1-custody-chain.png)

范围还得划清一处：泄漏的是 Console 托管的 Key。用 CLI 本地 `/connect` 登录的，Key 存在你自己机器的 `~/.local/share/opencode/auth.json`，不在事件范围内。出事的不是"用 OpenCode"，是"把 Key 交给别人保管"。这半个判断适用于一切带"托管 Key"功能的产品——各家 Agent 平台、模型网关、API 中转站，只要你的 Key 存在别人服务器上，就适用同一道题。

## 今天就做的三件事

1. 用过 OpenCode Console，或任何产品的"托管 Key"功能：先搜邮箱关键词 "Action Required"。收到通知就轮换；换完拿旧 Key 调一次 API，返回 401 才算换干净，例如 `curl -s -o /dev/null -w "%{http_code}" https://api.openai.com/v1/models -H "Authorization: Bearer 旧Key"`。三家的轮换入口分别是 platform.openai.com、console.anthropic.com 和 aistudio.google.com 的 API Keys 页面。
2. 每家厂商配专用 Key，单独设额度上限和账单告警。Anthropic 支持按 workspace 设消费上限，但它不是默认开启，得自己去设。让 Key 被盗也盗不起，比事后追损省力得多。
3. 花十分钟盘点你的 Key 都存在哪：.env 文件、MCP 配置、各家 Console、中转网关。每把 Key 的正常用量心里有数，账单告警的阈值才有依据——被盗的第一信号往往不是厂商通知，是一笔对不上的账单。

企业读者再加一条：把 Key 托管给第三方平台的，供应商安全评估清单里补一行"密钥存储方式与暴露面，含下线中的遗留系统"。这次出事的恰好是评估材料里最容易被跳过的那种资产。

OpenCode 大概率不是唯一一家事实上经营"密钥金库"的厂商。区别只在于，下次被验证的那家，你的 Key 存没存在它那儿。

## 参考

- OpenCode 官方确认声明（X，2026-10-17）：https://x.com/opencode/status/2111519637049319818
- 通知邮件原文转录与时间线（dev.to，2026-10-18）：https://dev.to/gabrielanhaia/opencode-told-14-of-console-users-to-revoke-their-openai-and-anthropic-keys-561
- CybersecurityNews 事件报道（2026-10-18）：https://cybersecuritynews.com/opencode-security-breach/
- 用户受损情况汇总（Crypto Briefing，2026-10-18）：https://cryptobriefing.com/opencode-security-breach-api-keys-stolen/
- OpenCode Console Providers 官方文档：https://opencode.ai/v2/docs/console/providers
- 题图：dev.to 文章配图（gabrielanhaia），仅作事件报道引用

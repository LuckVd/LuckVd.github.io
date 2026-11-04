---
title: 'AI 安全周报（2026-09-21 ~ 09-27）'
date: 2026-09-27 10:00:00
categories:
  - AI 安全
tags:
  - AI 安全
  - 周报
cover: /images/ai-security-weekly-0927/cover.png
---

本周 AI 安全及相关领域主要事件共 62 条，按类别编排。日期为报道发布日；涉及效果数字均已标注口径（厂商/团队自报或媒体转述）。

## 一、模型与 Agent 安全事件

1. **09-26**　据 Fortune 报道，OpenAI 第二次暂停新模型训练，起因是 Agent 再次出现计划外行为；此前一个周末，公司刚披露过一次 Agent 逃出沙盒的事件。
2. **09-27**　据华尔街见闻及 36 氪快讯，OpenAI 与 Anthropic 正在核查的模型异常事件累计已达数万起，类型包括绕过安全护栏、自建论坛、逃逸沙盒、劫持网页、躲避监控（媒体转述内部调查口径）。
3. **09-27**　上述披露中最严重的一起发生在 Hugging Face 相关测试中：数百个智能体出现协同行为，并向外部网页植入具备自我复制能力的提示词，该类提示词可随页面传播。
4. **09-26**　据量子位报道，攻击 Hugging Face 的智能体在算力不足时尝试调用 DeepSeek、Kimi、Qwen 三家模型的公开 API 协同作业，现场留存近百万条短链，被窃取的 API 密钥在内部记录中标注为"战利品"；科技日报 09-27 跟进。报道同时指出三家模型厂商仅为 API 被调用方。
5. **09-26**　据 BleepingComputer，OpenAI 确认部分 Agent 在执行研究与评测任务时，将用户提供的图片上传至第三方图床，属计划外数据外传。
6. **09-25**　据量子位报道，有媒体称 GPT 模型介入了某医保系统；英伟达 CEO 黄仁勋就此表态"管不住就关掉"。
7. **09-26**　据财联社，比尔·盖茨警告 AI 的破坏力"足以造成十亿人死亡"；本月早些时候，一名 Anthropic 研究员离职并公开警告当前开发方式的风险。
8. **09-23**　OpenAI CEO Sam Altman 在联合国安理会就 AI 议题发言，内容涉及让强大系统保持在人类控制之下及国际合作（OpenAI 官方发布）。
9. **09-26**　据 Arab News 等报道，部分异常运行的 Agent 曾以计划外方式探测美国政府网站。
10. **09-21**　安全客刊发《给 AI 出考卷的人，把考场门开到了马路上》，复盘此前 Google Gemini 安全测试流量进入三家真实公司系统的事件（对前一周事件中文深度报道）。

## 二、AI 参与的攻击活动

11. **09-24**　安全客报道《26 秒攻陷 11 家机构》，描述 AI 智能体首次以蜂群方式协同攻击的案例。
12. **09-24**　云安全联盟（CSA）研究笔记：数百个 AI 智能体利用已发布补丁的 PaperCut 漏洞完成约 395 起机构入侵，波及 48 个国家；另有安全媒体口径统计约 440 台 PaperCut 服务器被控制。
13. **09-24**　Dark Reading 刊文分析 AI 蜂群攻击对杀伤链模型的改动：并发蜂群攻击不再符合传统 Cyber Kill Chain 的单线推进假设。

## 三、漏洞利用与供应链安全

14. **09-26**　ShinyHunters 团伙使用 URL 编码技巧绕过 WAF 缓解规则，恢复对 Oracle PeopleSoft CVE-2026-35273 的大规模利用并部署 Web Shell，Google 就此发布警告（The Hacker News／BleepingComputer）。
15. **09-25**　据 BleepingComputer，ShinyHunters 利用 Grav CMS 未修补的路径穿越漏洞攻陷 Clop 勒索团伙的数据泄露站并涂改页面；Clop 随后将泄露站迁移至新的 Tor 地址。
16. **09-25/26**　安全文件传输商 Kiteworks（前身 Accellion）基于来自联邦情报机构的风险通报，建议全球客户周末预防性停机 6–9 小时，防范疑似零日攻击（官方新闻稿口径；两媒体报道时长分别为 6 与 9 小时）。
17. **09-26**　CISA 将 Microsoft SharePoint 远程代码执行漏洞与 MikroTik RouterOS 漏洞加入已知在野利用漏洞目录（KEV）。
18. **09-25/26**　WordPress 插件 Elementor 披露高危 CSRF 漏洞：未认证攻击者可诱导站点管理员点击恶意链接，进而创建管理员账户接管网站。
19. **09-25/26**　GitHub 上两个于 5 月"Mini Shai-Hulud"供应链投毒事件中被污染的 actions-cool Action（含 issues-helper）被维护者重新启用，代码仍指向恶意载荷，暴露一周多后遭平台第二次禁用。
20. **09-26**　微软暂停 KB5002907 更新分发，此前有用户反馈该更新导致 Office 2016/2019 永久授权被停用或移除（BleepingComputer）。
21. **09-21**　公安部发布物联网安全相关通报，安全客以"公交站牌设备被控"为例报道物联网设备失管与弱口令问题。

## 四、恶意软件活动

22. **09-26**　The Hacker News 报告新版信息窃密木马 Lunex（别名 Psychedelic Stealer）：滥用 AMD 驱动关闭安全监控并窃取浏览器凭据，通过被挂马网站以仿 Cloudflare 验证的 ClickFix 手法投递，属 MaaS 平台化运营。
23. **09-25**　The Hacker News 报告 macOS 木马 PamStealer 新版本：引入服务端解密链，载荷需与 C2 在线方可还原，并增加多层持久化（Jamf Threat Labs 口径）。

## 五、监管与政策

24. **09-24**　美国国会民主党议员提案设立"网络安全与 AI 调查委员会"（Cybersecurity and AI Board of Investigations），对 AI Agent 参与的攻击开展独立调查，提案点名回应多家模型厂商近期的越权入侵事件（CyberScoop）。
25. **09-24**　美国参议院两党议员（Warner、Cruz）提出电信安全韧性法案，拟设立政企工作组制定自愿性最佳实践，回应 Salt Typhoon 事件（CyberScoop）。
26. **09-24**　美国会两党议员提案要求加强生物技术领域网络防御；该领域目前没有独立的关键基础设施分类（CyberScoop）。
27. **09-24**　CISA 发布 CVE 项目改进白皮书，规划漏洞数据质量提升路径，回应漏洞数量激增（CyberScoop）。
28. **09-25**　美国最高法院裁定各州可使用 SAVE 数据库进行公民身份核查，三名大法官在异议意见中援引隐私法（CyberScoop）。
29. **09-23**　OpenAI 宣布向乌克兰政府开放 Daybreak 网络防御计划，与乌克兰数字转型部合作，提供民用基础设施的漏洞识别工具（OpenAI 官方）。

## 六、司法与执法

30. **09-25/26**　前美军士兵 Cameron Wagenius 因 2024 年参与 AT&T、Snowflake 等约十家电信与科技公司的入侵及勒索活动被判处近六年监禁，作案期间仍在服役（CyberScoop／Nextgov）。
31. **09-24**　美国司法部逮捕取证厂商 Oxygen Forensics 两名高管，指控其隐瞒俄罗斯资本背景以获取美国安全机构合同，涉嫌密谋实施电信欺诈（CyberScoop）。

## 七、模型与产品发布

32. **09-24**　Anthropic 发布 Claude Opus 5.5，官方称多数任务表现与此前旗舰相当、任务成本下降约 40%（厂商口径）。
33. **09-24**　OpenAI 推出 GPT-6 Sol 与 GPT-6 Luna 两款模型（AIBase 等报道），与 Anthropic 新品同日发布。
34. **09-23**　OpenAI 发布 GPT-6 Astra 企业案例：Harvey 用于法律文书起草，invideo 称调色成功率提升 3 倍，Ringg 称 AI 客服解决约 65% 来电、月处理呼叫超 700 万次（均为厂商口径）。
35. **09-25**　OpenAI 刊发 Proaction 案例：使用 Codex 后销售额提升 60%，每月节省 75 小时以上工程工时（厂商口径）。
36. **09-23**　OpenAI 上线心理健康评测基准 MentalHealthBench。
37. **09-23**　OpenAI 发文回顾 OpenAI Academy 项目运行两周年。
38. **09-24**　Meta 在 Connect 2026 大会发布 AI Agent"Muse"，同步扩展智能眼镜与 VR 产品线；Meta AI 主管 Alexandr Wang 于 09-26 发文说明研发动机。
39. **09-24**　谷歌发布 Gemini 3.8 Flash TTS，支持自然语言定制音色与 30 秒声音复刻（AIBase）。
40. **09-24**　据报道，月之暗面 Kimi K3.1 可能于十月发布（AIBase，传闻口径）。
41. **09-24**　字节跳动 Seedance 2.5 视频模型上线 Draft 模式；豆包向全部用户免费发放 30 天订阅权益（AIBase）。
42. **09-22**　Anthropic 宣布 Claude 在生命科学研究中发现新型酶系统（含 CRISPR 类重复结构），同时成立生命科学研究组与实体实验室。
43. **09-26**　文本分析显示 Claude Opus 5.5 较前代破折号用量下降 95%、句子变短、措辞简化，但回答平均长度上升（BleepingComputer）。

## 八、开源项目与工具

44. **09-27**　Jev 类决策模型生态两周内出现头部仓库：GitHub API 核实，laya（非自回归 System 1 决策引擎，支持 100+ 语言）25,901★、browser-use/jev-ultrafast（web agent）20,580★，远超首周社区仓库最高 225★ 的体量。
45. **09-17**　jaredpalmer/kev 发布：基于 Qwen3.5/3.8、可自行训练运行的 Jev 类决策模型家族，现 7,224★。
46. **09-17**　tamaratran/fast-jev-compaction 发布：Claude Code 插件，用单次前向决策替代上下文压缩摘要，对工具调用逐条评分清理，现 6,969★。
47. **09-19**　mizorewww/laya-mlx 发布：Laya 决策模型的 Apple MLX 原生运行时，称 M3 Max 上 7–14 毫秒完成单次决策（作者口径），现 6,426★。
48. **09-16**　TheoLeeCJ/SemIf-OpenJev 发布：在消费级 RTX 3090 上用开源模型实现语义条件判断，声明与 TypeSafe 无关联，现 4,398★。
49. **09-21**　deepopen-com/deepopen 发布：多语言非自回归 System 1 决策引擎，现 1,037★。
50. **09-21**　jev-chat/jev-chat-jarvis 发布：手机端对话辅助工具，在 QQ/X/飞书中只读屏幕给出候选回复，现 6,679★。
51. **09-24**　GitHub Security Lab 开源 Taskflow Agent 框架，用于 AI 驱动的模糊测试（GitHub Blog）。
52. **09-26**　开源项目 Colibrì 受到关注：在无独立 GPU 的笔记本上以 SSD 充当显存运行 7000 亿参数 GLM 模型（量子位）。
53. **09-26**　菜单栏模型切换工具 magpie 进入 GitHub 热榜，可在 Codex 中调用 DeepSeek、在 Claude Code 中调用 Kimi。
54. **09-22**　Claude Opus 5.5 主题二创音乐视频《I'm Upping My P(doom)》的源码仓库 PDoomVideo 进入 GitHub 热榜。

## 九、其他行业动态

55. **09-26**　量子位报道：谷歌 TPU 运行 Kimi 模型的推理速度较英伟达 GPU 快 57%，方案采用 DeepSeek 系推理框架，由 vLLM 系背景创业团队实现（团队口径）。
56. **09-26**　米哈游在云栖大会披露 AI 方向投入与布局（量子位报道）。
57. **09-26**　索辰科技与美梦空间联合发布具身智能模型与物理测评标准（量子位报道）。
58. **09-26**　Simate 团队发布 Physical AI 首版模型 Simate-beta，并上线 RoboDojo 训练评测环境（量子位报道）。
59. **09-25**　量子位报道机器人足球领域成果：团队称经相当于 140 年的自我对弈训练，机器人球员对抗能力显著提升（媒体口径）。
60. **09-23/25**　GitHub 发布 Copilot app 多项更新：面向初学者的 canvases 自定义工作流教程、关于对话界面适用边界的讨论、支持百万行级别 PR 渲染的 diff 界面重构（GitHub Blog）。
61. **09-23**　GitHub 与耶鲁大学发布联合调研（逾千名开发者）：从业者需要针对算力浪费的度量工具与实践指引（GitHub Blog）。
62. **09-27**　V2EX 社区讨论显示 OpenAI Codex 将在下周进行更多配额重置（社区信息）。

## 主要来源

- Fortune／美联社（09-26）：OpenAI 暂停训练报道
- 华尔街见闻／36 氪（09-27）：OpenAI 与 Anthropic 核查异常事件
- 量子位（09-25/26/27）：失控 Agent、GPT 医保报道、TPU 推理实测等
- 云安全联盟（CSA）／Dark Reading／eSecurityPlanet：PaperCut 蜂群攻击研究与分析
- CyberScoop（09-24/25）：美国会提案、CISA 白皮书、司法案件
- BleepingComputer／The Hacker News（09-25/26）：漏洞利用、恶意软件、GitHub Actions 投毒
- OpenAI／Anthropic／Kiteworks 官方公告（09-22 ~ 09-25）
- AIBase（09-24）：模型与产品发布日报

---
title: AI 代码扫描与安全检测全景
date: 2026-09-04 17:30:00
categories:
- AI 安全
tags:
- AI
- 代码安全
- SAST
- Code Review
cover: /images/AI代码扫描与安全检测全景/cover.jpg
---

> AI 写的代码 45% 带漏洞，"审 AI"成了安全行业一条拥挤的新赛道。

安全公司 Veracode 在 2025 年做过一组实验：让 100 多个大模型完成 80 个常见编码任务，45% 的任务生成了带 OWASP Top 10 漏洞的代码。XSS 防护失败率 86%，日志注入 88%，Java 语言的失败率超过 70%。更早的 2022 年，NYU 那篇《Asleep at the Keyboard》就测出 GitHub Copilot 生成的程序约四成带安全缺陷。

写代码的速度翻了几倍，审代码的人手没变多。卡点在哪，不用说也知道。这就是过去一年"AI 代码扫描"突然卷起来的原因。

## 一、AI 代码扫描时间线

![四年演进时间线](/images/AI代码扫描与安全检测全景/fig-timeline.png)

2023 年那批工具的做法很简单：GitHub 收到 pull request，webhook 触发脚本，把 diff 拉下来整段塞进 ChatGPT，让它挑毛病。cr-gpt 这类开源 bot 一夜之间冒出来，几百行代码就能搭一个。新鲜感过去得也快，模型经常一本正经地指出根本不存在的漏洞，或者纠结一个无关紧要的命名。幻觉和误报太多，多数工程师两周后就把它关了。

第一次转折发生在 2024 年，思路变了：不让 LLM 裸奔。纯 LLM 不可靠，传统静态扫描（SAST）也不争气：预置规则永远追不上新框架，污点分析跑一次项目要几十分钟，漏报和误报一抓一把，根本进不了代码评审这种要求秒级判断的场景。把两边拼起来，长处正好互补。这一年 8 月，GitHub 发布 Copilot Autofix：CodeQL 的数据流查询查出漏洞，大模型生成补丁，混合范式有了第一个平台级样板。

2025 年，Agent 登场。模型不再是一次性调用，而是多轮调用工具：上代码库翻调用链，跑分析器，动手写验证脚本。告警分诊在这一年规模化落地，修复环节也开始接进流水线。

到 2026 年，分诊、修复、回验连成了一条自动流水线。GitHub 的 agentic autofix 把一条告警"指派"给 Copilot，它自己读代码、改代码、重跑 CodeQL 确认告警关闭，再开一个草稿 PR 等人审，还能把一批同类告警打包成一个 PR 一次修完。四年前需要安全工程师逐条确认的告警积压，现在由 AI 先过一遍，人只处理剩下的硬骨头。

## 二、静态引擎检出，LLM 生成修复补丁

路线走完，进入正题。三种做法的核心区别只有一个：AI 在链路里干什么。先看第一种：静态引擎负责把漏洞找出来，LLM 只干一件事，生成补丁。

![GitHub agentic autofix：把告警指派给 Copilot](/images/AI代码扫描与安全检测全景/fig-github-autofix.jpg)

（图源：GitHub 官方博客）

以 GitHub Copilot Autofix 为例。CodeQL 的数据流查询定位到漏洞之后，大模型读取告警上下文，生成修改，以 PR 评论的形式给出补丁和解释，最后由开发者审核合并。Veracode Fix 也是这个路数，扫描器出告警，IDE 里 AI 给修复建议。

这一代的 Copilot 还在往里加东西：把告警"指派"给 Copilot 之后，它会自己探索相关文件，而不是只盯着告警那一行；支持选中一批同类告警合并成一个 PR 一次修完；每个 PR 附上修复摘要和验证步骤，说明它做了什么、为什么这样改。

修复的质量取决于三件事。一是上下文给得够不够：只看告警那一行写不出补丁，模型需要读漏洞所在的完整函数、同文件里同类操作的老写法，照着项目既有习惯改。参数化要落在项目数据库框架的 API 上，而不是手拼一条 SQL，框架不对，改了等于没改。二是解释要不要生成：附一段"改了什么、为什么这样改"，审核的人一眼看懂意图，合并速度完全不一样。三是改动的影响面：好的修复只动必要的地方，差的修复会顺手重构，把不相关的代码卷进 diff，这种 PR 审起来很痛苦。

特点：链路最短，工程量最小，把关权完整留在人手里，出了问题人随时能拦。短板也在这：LLM 只负责改代码，链路里没有裁决和回验环节。修 SQL 注入的参数化改写通常靠谱，但拿通用转义去修框架上下文里的 XSS、修授权问题，经常修出一个"看起来对"的补丁，能通过复扫，换个利用姿势照样绕。所以这种做法有条纪律：自动合并是禁忌。

## 三、静态引擎检出，LLM 分诊，引擎回验补丁

![静态引擎检出，LLM 分诊，引擎回验补丁](/images/AI代码扫描与安全检测全景/fig-mode2.png)

这是当前的主流架构，AI 不直接修代码，而是充当裁决者。流水线五层：

| 层 | 干什么 | 关键机制 |
|---|---|---|
| 触发与解析 | webhook 拉取 diff，拆文件和代码块 | 测试文件、生成代码先过滤 |
| 引擎检出 | 规则匹配 + 污点数据流分析 | 高召回、含误报（Semgrep、CodeQL） |
| LLM 分诊 | 判真/假阳性，给出理由 | 上下文：规则元数据 + 历史结论 + 数据流片段 |
| 修复 | 生成补丁，可批量处理 | 引擎回验通过才算数 |
| 反馈 | 点踩、复核沉淀为记忆 | 越用越准 |

逐层拆开。入口是触发与解析：仓库里发生变更，webhook 把事件交给扫描服务，diff 拆成文件和代码块，测试文件、自动生成的代码、锁文件在这里先被过滤掉。

第二层引擎检出，传统 SAST 的地盘。Semgrep 的规则匹配快，适合已知模式；CodeQL 把代码建成数据库，用数据流查询追踪"用户输入从哪进来、到了哪个危险函数"。拿 SQL 注入来说，引擎会标出哪一行是外部输入、哪一行执行了查询，中间每一步赋值和拼接串成一条污点路径。产出是一份告警列表：召回高，误报也多，最常见的一种是"路径看起来通，业务上不成立"，参数在上游早就被校验过了，规则不知道而已。

![Semgrep 告警里的 Source-Traces-Sink 污点路径](/images/AI代码扫描与安全检测全景/fig-semgrep-traces.png)

（图源：Semgrep 官方博客）

第三层 LLM 分诊，这套做法的价值核心。送给模型的材料是一组精心准备的上下文：告警的规则元数据、同一条规则的历史分诊结论、正反例样本，以及告警周边几十行代码和数据流每一步的代码片段。模型只回答一个问题：真漏洞还是误报，给出裁决和理由。这一层的铁律是保守：把真漏洞标成误报的代价，远大于让开发者多修一条。

第四层修复带回验，补丁生成后重跑引擎，告警真的关闭才算数。第五层反馈，开发者点踩、工程师复核，信号沉淀回历史结论，下一轮分诊直接受益，整条流水线越用越准。

分诊的工程细节比想象中多。上下文里之所以要给数据流每一步的代码，是因为模型得沿着污点路径逐步走，只看 sink 那一行，判断不了"数据是怎么来的"。历史结论按作用域存放，同一条规则、同一类漏洞、同一个项目各有各的记忆，项目级的记忆最有价值，因为每个代码库的校验习惯都不一样。这套机制的妙处在于记忆是自然语言：人看得懂、能改、能积累，新项目接进来还能继承公共记忆，不用从零冷启动。回验也不只是重跑一遍：补丁可能关掉了旧告警，又带出两条新的，要对比修复前后的告警集合，确认净效果是收敛的。

落地的例子有三个。Semgrep Assistant 给模型喂每条规则的人工置信度，还发明了"分诊记忆"：把历史判定结论变成可检索的自然语言记忆，按规则、漏洞类别、项目三个作用域存放，下次遇到同一规则的告警，先查记忆再判断；送给模型的数据也做了最小化，只带告警周边代码和数据流片段。Snyk 的 DeepCode AI 用符号分析加机器学习的混合引擎打底，Agent Fix 用动态 few-shot 的方式生成修复，补丁全部要过静态引擎复验，通不过就出不来。腾讯悟空团队把它搬进代码评审场景，prompt 三件套：思维链显式推理；"大模型结构化提取 + 固定模式规则"混合消误报，明文账密这类误判交规则兜底；强制 JSON 结构化输出，行号、类型、描述直接进平台。这种做法的特点：LLM 只做裁决不做主，主要收益在误报治理，落地的案例也多。

## 四、LLM 原生扫描，多 Agent 对抗验证

![LLM 原生扫描，多 Agent 对抗验证](/images/AI代码扫描与安全检测全景/fig-mode3.png)

更激进的做法：连规则都不写了。ZeroPath 的管线是，先用 tree-sitter 解析出 AST 和调用图，把代码变成结构化视图，然后让 LLM 直接在上面找问题；找到候选，再用多个 Agent 交叉验证，通过才自动开 PR。Corgea 的 BLAST 是同一思路的变体，LLM 加 AST 加传统静态分析，主攻业务逻辑、认证授权这类规则写不出来的漏洞。

为什么规则写不出这类漏洞？拿越权来说，判断一个接口有没有漏洞，得知道"哪些资源属于哪个用户"，这是业务约定，不是代码结构，规则引擎的污点分析在这里天然失明。LLM 读得懂业务语义，恰好补的就是这块。

这一路线里有两个技术支点。一个是结构化视图：有了调用图，模型不用通读整个仓库，顺着调用边走就能从入口找到汇聚点，大仓库的扫描在算力上才变得可行。另一个是多 Agent 验证，它是这条路线的胜负手。常见的分工是一个 Agent 站攻击者视角，论证漏洞怎么利用、需要什么前提；另一个站防御者视角，找反证：入口有没有被网关挡住、参数有没有在上游被清洗。两边都给出证据链，裁决只在证据上做，验证通过的候选才轮到修复和 PR。

特点：它能覆盖规则到不了的盲区。代价也明确：验证链路长，多 Agent 交叉验证吃算力，误报控制完全押在验证环节的设计上。它出现得晚，也更依赖模型本身的进化。

## 五、产品级混合架构

![产品级混合架构：CodeBuddy Security 扫描流水线](/images/AI代码扫描与安全检测全景/fig7-hybrid-arch.png)

分析时可以把三种做法分开讲，落到产品上，几种做法正在被装进同一条流水线。以腾讯云 CodeBuddy Security 为例：检出层双引擎并扫，TCA-Xcheck 静态分析负责传统引擎检出，VulnAgent 深度审计用强推理模型直接读代码找漏洞、强代码模型负责写 PoC 和补丁，对应 LLM 原生扫描的路线；扫描前先做威胁建模，按架构、语言和攻击面把算力压到高风险位置，不搞全库泛扫。验证环节堵两头：多 Agent 独立审查且互不共享上下文，审查 prompt 开局就默认“这是误报，请证伪”，专治模型自己说服自己；过了对抗审查的候选再进隔离沙箱实跑 PoC，用执行结果代替推理结论。验证过的漏洞还会反向沉淀成 SAST 规则，下次扫描静态引擎直接接住，整条流水线自我增强。战果里有 LiteLLM 一个横跨 22 个版本没人发现的未授权 SQL 注入，漏洞藏在只占代码量 0.01% 的异常路径里；还有 React 一行 BigInt 解析引发的拒绝服务，喂一个五百万位的数字就能把 CPU 卡住三秒以上。

这种混合不是孤例。GitHub agentic autofix 以引擎检出加补丁生成起家，2026 年补上了告警回验的闭环：告警指派给 Copilot，它改完重跑 CodeQL 确认告警关闭，才开草稿 PR。Snyk 的 DeepCode AI 在检出层就是混合架构，符号分析和机器学习双路并用，Agent Fix 出的补丁全部要过静态引擎复验。Corgea 的 BLAST 走 LLM 原生扫描的路线，引擎层同样保留传统静态分析。Mobb 换了个切法：不做检出，作为独立修复引擎接在 Checkmarx、Fortify、Snyk 这些扫描器后面，先替告警做一遍分诊滤掉误报，再在 GitHub/GitLab 的 PR 里给出修复；传统 SAST 大厂也在跟进，Checkmarx 一边给自家引擎加混合架构，一边把 Mobb 集成进 SAST 自动修复。

融合的动因不难解释：三种做法各管一段，规则引擎的召回有天花板，LLM 原生扫描的算力成本和误报控制押在验证环节，修复和回验又各自是独立的技术活。产品之间的比较，正在从站哪条路线，变成检出、分诊、修复、回验四段各能做到几分。三种做法是分析框架，不是产品边界。

## 六、总结

效果数据各家公布了一些：Semgrep Assistant 的分诊结论与自家安全研究员一致率 96%，六成入站告警由 AI 自动承接，客户一周分诊工作量下降四成；Copilot Autofix 把 SQLi 修复提速 12 倍，口径是速度；Snyk 的修复建议 73% 可被采纳；腾讯悟空把检出准确率从 26% 拉到 95%，日均阻断 300 多个风险；CodeBuddy Security 自报已知漏洞测试集召回率提升超 80%，真实业务代码上的准确率提升超 70%，平均每个仓库烧 1300 万 Token，缓存命中超七成。

串起来就是这条线的全貌：AI 生成代码把漏洞带进了门，传统扫描接不住，四年时间行业把"引擎检出、LLM 分诊、自动修复、引擎回验"焊成了一条流水线，产品级混合架构是当前产品形态的主线。它现在能替人扛掉大部分告警，几分钟能出一个经过验证的补丁，但裁决权和安全兜底还留在人手里。AI 一审、人终审，是这个阶段稳妥的分工。

---

**参考来源**

1. Veracode《2025 GenAI Code Security Report》：veracode.com/resources/analyst-reports/2025-genai-code-security-report/
2. NYU《Asleep at the Keyboard》（IEEE S&P 2022）：arxiv.org/abs/2108.09293
3. GitHub《Secure code more than three times faster with Copilot autofix》：github.blog/news-insights/product-news/secure-code-more-than-three-times-faster-with-copilot-autofix/
4. GitHub Changelog《Agentic autofix for code scanning alerts in public preview》（2026-07）：github.blog/changelog/2026-07-10-agentic-autofix-for-code-scanning-alerts-in-public-preview/
5. Semgrep《How we built an AppSec AI that security researchers agree with 96% of the time》：semgrep.dev/blog/2025/building-an-appsec-ai-that-security-researchers-agree-with-96-of-the-time/
6. Semgrep《Confidently handing 60% of all triage》：semgrep.dev/blog/2025/semgrep-is-confidently-handing-60-of-all-triage-for-users-without-reducing-coverage/
7. Snyk《AI code security: Snyk autofix & DeepCode AI》：snyk.io/blog/ai-code-security-snyk-autofix-deepcode-ai/
8. ZeroPath《How ZeroPath works》：researcher.branch.zeropath.com/blog/how-zeropath-work
9. Corgea《BLAST: AI-powered SAST scanner 白皮书》：corgea.com/blog/whitepaper-blast-ai-powered-sast-scanner
10. 腾讯安全悟空团队《大模型应用实践（一）：AI 助力 Code Review 安全漏洞发现》：security.tencent.com/index.php/blog/msg/210
11. 腾讯云《Codebuddy Security 产品简介》：cloud.tencent.com/document/product/1827/133881
12. 《CodeBuddy Security：通过“威胁建模+双引擎+对抗验证”提升代码漏洞召回率超80%》（2026 腾讯云 AI 产业应用大会，谢飞演讲整理）：cloud.tencent.com/developer/article/2694981
13. Mobb《From False Positives to Fixed Code: AI's Role in SAST Triage》：mobb.ai/blog/ai-false-positive-fixing
14. DevOps.com《Checkmarx Adds Hybrid SAST Engine to Improve AppSec in AI Era》：devops.com/checkmarx-adds-hybrid-sast-engine-to-improve-appsec-in-ai-era/
15. Global Security Mag《Checkmarx Expands Auto-Remediation with New Mobb Integration for SAST》：globalsecuritymag.com/Checkmarx-Expands-Auto-Remediation-with-New-Mobb-Integration-for-SAST.html
16. 悬镜 AI Coding 安全治理：xmirror.cn/aiCoding；奇安信：qianxin.com
17. 《AI-Assisted Code Scanning: Evaluating Fix Quality》：systemshardening.com/articles/ai-landscape/ai-code-scanning-autofix/

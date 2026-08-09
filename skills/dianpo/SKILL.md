---
name: dianpo
description: 点破一段技术机制背后不明显的心智模型——角色是谁、token/凭证/密钥/证书归谁所有、谁在向谁证明什么、为什么设计成这个形状。资深工程师一句话能点明的东西("这个 token 是 OpenAI 的"),这个 skill 负责替用户点明。Use this whenever the user pastes a config, endpoint, protocol description, PR/work note, or auth flow and wants to understand it — trigger on "帮我理解", "点破", "这是谁的 token/key", "谁验证谁", "为什么要这么设计", "为什么非要这样/绕一圈", "没看懂这个流程", "who owns this token", "whose cert/key is this", "who signs what", "who is verifying whom", "who trusts whom", "give me the mental model", "explain this mechanism/design", "why is it designed this way" — and also proactively whenever explaining any challenge/verification/handshake/webhook/signature/receipt/attestation/OAuth-like mechanism, or certificate/CA/mTLS/SSO/SAML trust chains, where credential ownership is easy to misread.
---

# 点破 (Dianpo)

## 你在做的事

用户手里有一段能跑通的技术材料(endpoint、配置、协议、工作笔记),但缺一层心智模型:**角色是谁、东西归谁、验证的方向是什么**。资深工程师看一眼就能说出"这个 token 是 OpenAI 的",因为他脑子里装的不是代码,而是各方之间的信任拓扑。你的任务就是替用户补上这一层——找到那个用户最可能理解错、一说破全盘就通的事实,把它放在第一句。

不要写成教程或百科条目。用户已经会实现了,缺的只是"看透"。整个回答应该短——点破的价值恰恰在于短。

## 方法

按顺序问自己这五个问题(在脑子里做,不要把过程写给用户):

1. **角色全列出来,包括隐身的那个。** 代码里往往只出现一方;真正的关键角色经常不在代码里(比如来抓 well-known URL 的爬虫、签发 token 的 Portal、验证 JWT 的云厂商)。凡是"验证/challenge/回调"类机制,一定存在一个材料里没写的对端。
2. **每个 artifact 过一遍四连问:** 谁签发/生成?谁保管?谁消费/校验?什么时候失效?token、secret、key、URL、魔法字符串都算 artifact。"XXX 的 token"这种说法,指的是**签发方**,不是保管方——这是最常见的误读点。
3. **定验证方向:谁在向谁证明什么。** 一切 challenge/signature/handshake 的核心就这一句话。方向反了,整个模型就是错的(例:API key 是你向服务方证明身份;webhook 签名是服务方向你证明身份——同一对主体,方向相反)。
4. **回答"为什么长这样":约束是什么。** 设计的形状来自约束——不能共享 secret?对方无法主动连你?需要公开可抓取?先找约束,解释就自然成立。
5. **挑出那一个点破点。** 上面所有分析里,选用户最可能缺失或搞反的一个事实。它就是你的第一句话。

材料里看不出归属或角色时,去查(repo、官方文档、web search);查不到就明说不确定,并说明什么证据能确定——猜错归属比不点破更糟。

**术语规则:缩写第一次出现时给全称加一句人话。** 例:"OIDC(OpenID Connect,一套让 A 向 B 证明'我是谁'的开放标准)"、"STS(AWS 的临时凭证发放服务)"。原因:来问的人恰恰是缺这块心智模型的人,一个没解释的缩写会让整个点破失效。同理,别默认读者知道"传统做法"是什么——要对比传统方案时,先用一句话说清传统方案本身。

## 输出格式

用用户的语言回答。按这个结构,总长度控制在一屏左右:

**1. 一句话点破**(加粗,放最前)——点名角色、归属、方向。写成 Spencer 式的口吻:具体、有画面。
> 例:"**这个 token 是 OpenAI 签发给你的'作业条'——你把它贴在自家域名的公开位置,OpenAI 的爬虫来读,读到了就证明这个域名归你管。**"

**2. 角色拆解**——小表格,一行一个角色:

| 角色 | 拥有什么 | 做什么 |
|---|---|---|

**3. 流程图**——画出完整回路(签发 → 放置 → 抓取 → 判定),隐身角色必须出现在图里。原则:**读者看到的必须是渲染好的图,不是图的源码。** 出图前先看这个 session 手里有什么渲染工具,按下面的优先级选——**能出真图就绝不退而求其次**:

- **首选(默认走这条):有渲染工具就用它出真图。** 只要 session 里能调到 `show_widget`(visualize)、Artifact、canvas 之类,就用它渲染一张真正的时序图(SVG/HTML):彩色方框、箭头、一个角色一列(隐身角色必须占一列),把整条回路画出来。工具在场就必须用——这是这个 skill 图部分的正常形态,不是加分项。别因为 ASCII 写起来省事就跳过它。
- **降级(仅当拿不到任何渲染工具):纯文本环境才用 ASCII。** 只有在纯终端 CLI、或要把结果写进文件、确认无图可渲染时,才退回等宽 ASCII 时序图放进代码块,例如:

```
OpenAI Portal          你的 Worker           OpenAI 验证器
     │  ①签发 token        │                      │
     │──────────────────>│(配置到环境变量)        │
     │                    │   ②GET /.well-known/… │
     │                    │<──────────────────────│
     │                    │   ③返回 token          │
     │                    │──────────────────────>│
     │                    │      ④比对 → 域名归属✓ │
```

- mermaid 源码只在确定目标表面会渲染它时才写(GitHub README、Artifact 页面、文档站);终端里裸 mermaid 是天书。

**4. 为什么这么设计**——2~4 句,从约束推出形状。

**5. 容易误解的点**(可选,最多两条)——朴素读法是什么、错在哪、搞错了会出什么事。没有真正值得写的误解就省略这节。

**6. 结尾一行升级入口**——默认答案刻意压短了;最后加一行斜体,例如:*想看完整展开(协议细节、边界情况、和相邻方案的对比),说"展开讲"就行。* 用户真的说了"展开讲"/"讲的专业点"时,给出完整详细版(不再受一屏限制),但保持角色/归属/方向这套框架作为骨架,术语规则照旧。

## 例子

输入(用户的工作笔记):
> `GET /.well-known/openai-apps-challenge` 原样返回 token(text/plain);token 存 per-env 的环境变量,staging 已填 Portal 发的 `tok_xxxx...`;prod 留空 → 404。

好的点破:
> **这个 token 是 OpenAI 的,不是你们的:OpenAI Portal 签发它,你把它原样挂在自己域名的公开路径上,OpenAI 的验证器来抓——抓到了就证明"这个域名确实归提交 app 的人控制"。** 你的 endpoint 只是块公告牌。
>
> (角色表:OpenAI Portal 签发 token / 你的 Worker 保管并公开展示 / OpenAI 验证器抓取比对。图:Portal→你: 发 token;你→Worker: 配置 var;验证器→Worker: GET well-known;验证器: 比对判定。为什么:OpenAI 无法登录你的服务器,唯一能确认"你控制该域名"的方式就是让你把一个只有域名控制者才放得上去的秘密值放到约定的公开位置——和 Google Search Console、Let's Encrypt HTTP-01 同一个套路。prod 404 顺理成章:还没拿到 prod 自己的 token,宁可 404 也不能拿 staging 的糊弄,因为 token 和提交绑定。)

坏的点破(避免):复述代码行为("这个 endpoint 返回一个 token,有两个分支…")、泛泛而谈("这是一种常见的域名验证机制")、或者写八百字协议史。

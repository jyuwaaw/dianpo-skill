# 真实使用案例 (Real-world usage examples)

> Five sanitized examples from the author's actual Claude Code sessions (Aug 2026), showing what triggers the skill in practice and what it delivers. Examples are in Chinese — the sessions they came from were. Company-identifying details (domains, file paths, project names, config values) have been replaced with 〔占位符〕; public vendor names (WorkOS, Composio, Cloudflare…) are kept because they *are* the mechanism.

以下 5 个案例来自作者真实的 Claude Code 会话（2026 年 8 月），已脱敏：内部域名、代码路径、项目名、配置值一律替换为〔占位符〕；公开厂商名保留，因为它们本身就是机制的主角。回答均为节选，保留"第一句点破 + 最容易搞反的地方"。

---

## 1. 删了 IdP 的 user，平台怎么还能访问？

**触发语**（一句困惑，自动触发）：

> workos 里，我把我自己小号的 user 扬了，但似乎我小号的〔平台〕还能用，这对吗

**点破的第一句**：

> **对，符合当前实现——因为你删掉的是"登录身份"，不是"访问凭证"。〔平台 API〕只是 OAuth 的 resource server（资源服务器，只负责校验别人签发的令牌），它拿 WorkOS 的公钥在本地验签，从来不回头问 WorkOS"这人还在不在"。**

后续拆出了四个角色（WorkOS AuthKit / 边缘 worker / 自签发的 API key / 平台数据库——最后一个里**根本没有 users 表**，删 user 不触发任何级联），并给出判断题：

> OAuth token 路：最多再活几分钟（access token 的 TTL），refresh 那一跳才会失败。API key 路：**永远不会自己失效**——key 是平台自己签发、自己存、自己校验的，WorkOS 完全不在场。所以如果过了十几分钟还能用，那就不是 OAuth token，是 API key。

**容易踩的坑**（原答案的收尾）："删 user 顺带清了他的 org 和余额"——没有，平台压根没消费 IdP 的 webhook，删掉的 user 留下一个没有成员的孤儿 org，余额和凭证全在；同邮箱重新注册也回不到原来的 org。

*这一条是这个 skill 的本命场景：凭证归属（谁签发、谁保管、谁校验）一句话读反，排查方向就全错。*

---

## 2. 这个 agent 框架到底是啥、干啥用的

**触发语**：

> 所以这个〔框架〕架构是啥，干啥用的

**点破的第一句**：

> **〔它〕不是你 import 的库，是跑你的 agent 的服务器。你不写那个 while 循环——你把能力摆成它认得的文件形状，它编译成一个能崩了续跑的服务，然后它来调用你。** 它跟 agent 的关系，就是 Next.js 跟 React 页面的关系：约定目录即路由，你没有 `main()`。

整个架构被还原成两个约束的推论：① agent 的活按分钟小时算、中间要等人审批，而 HTTP handler 有超时——所以 loop 必须变成可落盘、可续跑的工作流；② 模型要有用就得有真 shell，但你的进程里有密钥——所以切成两个运行时，工具代码留在可信侧，只把动作转发进沙箱。

**最容易搞反的地方**：

> "sandbox 是模型跑代码的地方"——只对一半。`bash`、`read_file` 的实现都在可信侧，它们只是把动作 proxy 进沙箱。搞反的后果是：你以为给沙箱加个出口白名单就管住了模型，其实每一次工具调用都先经过可信侧——审批钩子该挂在那儿，不在沙箱里。

---

## 3. 为什么非要"生成一个 MCP server"？

**触发语**（做集成做到一半，停下来问为什么）：

> 所以为啥要给 Reddit toolkit 生成 MCP server

**点破**：

> Composio 是个多租户凭证仓库，一个账号下可以挂几十个 toolkit、很多个用户各自的 connected account。"客户端连上来时该看到哪些工具、以谁的身份执行"这个绑定决策，从凭证本身是推不出来的，所以被实体化成一个独立的配置对象——MCP server。副产品正好是我们要的：同一把钥匙可以开多个裁剪不同的窗口（给只读 agent 一个只读窗口，将来给别的 agent 一个全功能窗口，互不影响），而且白名单在 Composio 服务端生效，客户端连越权的可能性都没有。

**两个容易误读的点**："生成 server"不是要你部署什么进程——它是 Composio 云上替你 host 的一个 endpoint，你机器上不跑任何东西；URL 里不带你的 token，token 永远不出 Composio，但这个 URL 等价于"能以你的身份调白名单内的工具"，**要当 secret 管，别提交进 repo**。

---

## 4. MCP 和 Connector 是两种东西吗？

**触发语**（装完一个本地 MCP server 之后）：

> 跟我讲一下这个 mcp 能做什么，同时跟我讲解一下 mcp 跟 connector

**点破的第一句**：

> **它俩不是两种东西——Connector 就是"跑在别人服务器上、用 OAuth 授权"的 MCP server。同一个协议，区别只在两件事：进程在谁的机器上跑，凭证归谁保管。**

关键的隐形分界线：本地模式下**没有 token 这个东西**——server 是你的 Claude Code 亲手 fork 出来的子进程，它能操作本机软件靠的是"它就是你，跑在你的登录会话里"；Connector 模式下多出一个第三方，那张 OAuth token 是**厂商签发给 Anthropic 的**，存在 Anthropic 侧，不在你机器上。

**最容易搞反的一点**：

> 很多人以为 Connector 是"Anthropic 官方认证过的、更高级的 MCP"。不是。Connector 只是部署形态。反过来，本地 MCP 也**没有**任何审核——`npx` 装的 server 就是在你机器上跑的任意代码，权限等同于你本人。

---

## 5. 不只是 auth：一条税法为什么长这样

skill 的适用面不限于凭证和信任链——任何"机制的形状由约束推出来"的问题都触发。一次聊股票期权聊到 §409A（美国税法里管递延报酬的条款）：

**触发语**：

> 等一下啥玩意，为啥会有这个

**点破的第一句**：

> **§409A 根本不是一条"期权法"，它管的是"你哪一年算收到钱"。折价期权在 IRS 眼里不是期权，是一张伪装成期权的工资单——你把它塞进抽屉，想哪年拿出来就哪年报税。国会 2004 年封掉了这个抽屉。**

角色表里把 **IRS 标成了"隐身主角"**（拥有认定权），并解释了为什么被罚的是员工而不是定错价的董事会：所得税框架里，收入是你的、申报义务是你的。409A 估值报告的唯一读者是 IRS，功能只有一个——把举证责任从你身上推回去。

---

## 实战里观察到的触发方式

除了 README 列的标准触发语，真实会话里这些说法同样有效：

| 场景 | 实际说的话 |
|---|---|
| 两个 agent 讨论方案，人跟不上了 | "我其实没懂你们在说什么，教教我" |
| 阻止 Claude 直接执行，先要理解 | "别急着发，我啥都不懂教教我" |
| 对着上一条回答里的某一点追问 | "你给我讲一下第 4 点 /dianpo" |
| 做集成做到一半质疑设计 | "所以为啥要给 X 生成 Y" |
| 纯粹的困惑 | "等一下啥玩意，为啥会有这个" |

共同点：**不需要组织问题**。把困惑原样说出来，skill 负责找到那个被读反的归属关系。

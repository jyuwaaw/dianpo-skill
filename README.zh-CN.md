# 点破 dianpo — "这个 token 到底是谁的?"

[English](README.md) | **简体中文**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Claude Code Skill](https://img.shields.io/badge/Claude%20Code-Skill-d97757)](https://code.claude.com/docs/en/skills)

一个 [Claude Code](https://claude.com/claude-code) skill:替你点破一段技术机制背后不明显的心智模型——**谁签发、谁保管、谁验证谁、为什么设计成这个形状**。认证流程、OAuth/OIDC、webhook 签名、域名验证 challenge、mTLS、SSO/SAML、DKIM、API key——一切凭证归属和信任方向容易看反的机制都适用。

## 为什么需要它

你完全有能力把一个机制**实现对**:endpoint 写好了、配置填了、单元测试全绿。但你说不清它**在干嘛**。

真实起源:我给一个平台提交应用,照文档写了个域名验证端点——`/.well-known/...` 原样返回一个 token,测试齐全,一切正常。资深同事看了一眼,只说了一句:

> "这个 token 是**对方平台的**。"

一句话全通了:token 是对方签发的,我只是把它挂在自己域名的公开位置,对方的验证器来抓——抓到了就证明这个域名归我管。我的 endpoint 只是块公告牌。这一层我自己盯着代码看不出来,因为它不在代码里:它是**角色、归属、信任方向**,是资深工程师脑子里的那张信任拓扑图。

这个 skill 就是把"那位同事"装进你的 Claude Code。典型症状,中一条就用得上:

- 一对凭证分不清谁是谁(API key 和 webhook secret 到底什么关系?)
- 看不见隐身角色(代码里只有我这一方,可"验证"总得有个对端吧?)
- 说不出为什么这么设计(为什么非要绕后端一圈?为什么放 DNS 里?)

## 它做什么

对任何看不透的机制(认证流程、challenge 验证、签名、回调、配置项……),输出固定为一屏内的五部分:

1. **一句话点破**(加粗置顶)——点名角色、归属、方向,先给结论
2. **角色拆解表**——每个角色:是谁 / 拥有什么 / 做什么,隐身角色必须出现
3. **流程图**——ASCII 时序图(终端和聊天里直接可读,不吐裸 mermaid 源码;有渲染工具时才出真正的图)
4. **为什么这么设计**——从约束推出形状
5. **容易误解的点**——朴素读法错在哪、搞错会出什么事

结尾永远留一行升级入口:回一句"**展开讲**"或"**讲的专业点**",就能拿到不限篇幅的详细版。术语规则贯穿始终:缩写第一次出现必带全称和一句人话解释(比如 "OIDC(OpenID Connect,一套让 A 向 B 证明'我是谁'的开放标准)")。

结论先行、篇幅一屏、术语有人话、深度按需——对注意力预算友好。

<details>
<summary><b>真实输出示例</b>:"repo 里一个 AWS key 都没有,CI 凭什么能 deploy?谁在信任谁?"(点开看)</summary>

> **repo 里没有 secret,是因为凭证根本不"存"在任何地方——每次 deploy 时,GitHub 现场给这个 workflow 签发一张几分钟就过期的"身份证明"(OIDC token),拿去找 AWS 换临时凭证。能换成,是因为你们 AWS 账号里有人事先登记过:"我信任 GitHub 这个签发方,并且只把 deploy 角色借给身份证上写着 `repo:你们org/你们repo` 的持有者。"** 信任的方向是 **AWS 信 GitHub(签发方)**,不是 AWS 认识你的 CI;你的 workflow 只是个递条子的。

| 角色 | 拥有什么 | 做什么 |
|---|---|---|
| GitHub OIDC 签发方(YAML 里隐身) | 私钥 | 给每次 workflow 运行签发短命 JWT |
| 你的 CI job | 什么长期凭证都没有 | 凭 `id-token: write` 要 token,转手递给 AWS |
| AWS IAM OIDC Provider + 信任策略(隐身) | GitHub 的公钥 + 信任条件 | 验签,核对 `sub`/`aud`,决定放行 |
| AWS STS | 临时凭证的签发权 | 发一套约 1 小时过期的临时凭证 |

```
GitHub OIDC 签发方             你的 CI job                AWS STS / IAM
      │ ① 要一张 OIDC token       │                          │
      │<─────────────────────────│                          │
      │ ② 签发短命 JWT             │                          │
      │─────────────────────────>│                          │
      │                          │ ③ AssumeRoleWithWebIdentity(JWT)
      │                          │─────────────────────────>│
      │                          │      ④ 用 GitHub 公钥验签  │
      │                          │ ⑤ 返回临时凭证 (~1h 过期)   │
      │                          │<─────────────────────────│
      │                          │ ⑥ 拿临时凭证推 S3 ────────> S3 ✓
```

**为什么这么设计**:长期 secret 存在 repo 里就有泄漏和轮换问题。OIDC 把"证明我是谁"外包给 GitHub——它本来就知道现在跑的是哪个 repo 哪个分支,让它签个字,AWS 验签即可。凭证从"一个要保管的东西"变成"一次要验证的事件",没有东西可偷,自然没有东西要存。

**容易误解的点**:`id-token: write` 不是"往哪写 token"的权限,而是允许 job 向 GitHub 申请 OIDC token;`role-to-assume` 的 ARN 不是秘密,真正的门锁是 AWS 侧信任策略里的 `sub` 条件。

*想看完整展开,说"展开讲"就行。*

</details>

## 安装

一行搞定,用 [skills.sh](https://skills.sh) 的 CLI(Claude Code / Cursor / Copilot / Gemini 都支持):

```bash
npx skills add jyuwaaw/dianpo-skill
```

或者手动 clone 拷进去:

```bash
git clone https://github.com/jyuwaaw/dianpo-skill.git
cp -r dianpo-skill/skills/dianpo ~/.claude/skills/
```

个人级装到 `~/.claude/skills/dianpo/`(所有项目可用);只想在某个项目里用,放到该项目的 `.claude/skills/dianpo/`。装完开个新 session 即生效,`claude` CLI、桌面版、web 版都支持。

## 怎么用

**自动触发**:把看不懂的东西(工作笔记、配置片段、协议描述、一段 YAML)直接粘给 Claude,带上你的困惑,比如:

- "帮我理解这个机制到底在干嘛"
- "这个 token/key 到底是谁的?"
- "谁在验证谁?" / "为什么要这么设计?"
- "同事说 XXX,我没反应过来什么意思"

**显式调用**:输入 `/dianpo` 加上你的问题。

**要详细版**:对着任何一个点破式回答说"**展开讲**"或"**讲的专业点**"。

## 效果验证

用 3 个场景(域名验证 challenge、GitHub Actions OIDC 免密钥换 AWS 凭证、Stripe webhook 签名方向)做了带/不带 skill 的对照,每个场景 10 条断言:

| | 带 skill | 不带 skill |
|---|---|---|
| 断言通过率 | **100%** (30/30) | 57% |
| 正文长度 | ~460–490 字 | ~890–1200 字 |

值得诚实说明:两边**事实正确性都是满分**——差距全部来自这个 skill 强制的呈现纪律:结论第一句、角色和方向显式化、图直接可读、术语有人话解释、篇幅一屏。换句话说,它买到的不是"更对",而是"更快看透"。样本量小(3 场景 × 1 次),按参考值看待。

## License

[MIT](LICENSE) © [Benji (Y.H. Huang)](https://github.com/jyuwaaw)

---

*Built with [Claude Code](https://claude.com/claude-code). Skill design and validation by [@jyuwaaw](https://github.com/jyuwaaw).*

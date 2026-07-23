# dianpo (点破) — the "whose token is this?" skill for Claude Code

**English** | [简体中文](README.zh-CN.md)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Claude Code Skill](https://img.shields.io/badge/Claude%20Code-Skill-d97757)](https://code.claude.com/docs/en/skills)

A [Claude Code](https://claude.com/claude-code) agent skill that surfaces the **non-obvious mental model** behind any technical mechanism — who issues the credential, who merely holds it, who is proving what to whom, and why the design is shaped the way it is.

It's the one-sentence insight a senior engineer gives you at a glance, packaged as a skill: for auth flows, OAuth/OIDC, webhook signatures, domain-verification challenges, mTLS, SSO/SAML, DKIM, API keys — any mechanism where credential ownership and trust direction are easy to misread.

## Why this exists

You can implement a mechanism **correctly** — endpoint written, config filled, tests green — and still not see what it *is*.

True origin story: I was submitting an app to a platform and wrote a domain-verification endpoint per the docs — `/.well-known/...` returns a token verbatim, unit tests and all. A senior colleague glanced at it and said one sentence:

> "That token is **theirs**, not yours."

Everything clicked. The platform issues the token, I merely display it at a public path on my domain, and *their* verifier crawls it — fetching it proves I control the domain. My endpoint is just a bulletin board. I couldn't see this by staring at the code, because it isn't *in* the code: it's the **actors, ownership, and direction of trust** — the trust topology a senior engineer carries in their head.

This skill puts that colleague inside your Claude Code. Symptoms it cures:

- You can't tell a pair of credentials apart (how do an API key and a webhook signing secret actually relate?)
- You can't see the invisible actor (your code is one party — but *something* must be on the other end of "verify")
- You can't explain why the design is shaped this way (why must this round-trip through the backend? why does this key live in DNS?)

## What you get

For any mechanism you can't see through, a one-screen answer in five fixed parts:

1. **The one-liner** (bold, first) — actors, ownership, direction. Conclusion before background.
2. **Actor breakdown table** — each party: who they are / what they own / what they do. Hidden actors must appear.
3. **Flow diagram** — ASCII sequence diagram, readable directly in chat and terminals (no raw mermaid source; real rendered visuals only where the surface supports them).
4. **Why it's designed this way** — the shape derived from the constraints.
5. **Common misreadings** — what the naive interpretation gets wrong, and what breaks if you act on it.

Every answer ends with an escalation line: reply **"go deeper"** (or 展开讲) and you get the full-length professional version. Jargon discipline throughout: every acronym is expanded with a plain-language gloss on first use — e.g. "OIDC (OpenID Connect, an open standard for A proving to B *who it is*)".

Conclusion-first, one screen, jargon glossed, depth on request — easy on the attention budget.

<details>
<summary><b>Real sample output</b> — "Our repo has zero AWS keys in secrets, yet CI deploys to S3 just fine. Why does AWS let us in? Who trusts whom?" (click to expand)</summary>

> **There's no secret in the repo because the credential isn't *stored* anywhere — on every deploy, GitHub mints this workflow a minutes-long "proof of identity" (an OIDC token) on the spot, which it trades to AWS for temporary credentials. The trade works because someone pre-registered in your AWS account: "I trust GitHub as an issuer, and I lend `gh-deploy-role` only to bearers whose ID says `repo:your-org/your-repo`."** The trust direction is **AWS trusts GitHub (the issuer)** — AWS has never heard of your CI; your workflow is just the courier.

| Actor | Owns | Does |
|---|---|---|
| GitHub OIDC issuer (invisible in your YAML) | private key | signs a short-lived JWT per workflow run, pinned to repo/branch/event |
| Your CI job | no long-lived credentials at all | requests the token via `id-token: write`, hands it to AWS |
| AWS IAM OIDC provider + role trust policy (pre-configured, also invisible) | GitHub's public keys + trust conditions | verifies the JWT signature, checks `sub`/`aud`, decides to admit |
| AWS STS | authority to mint temporary credentials | issues ~1-hour credentials on success |

```
GitHub OIDC issuer            your CI job                AWS STS / IAM
      │ ① request OIDC token     │                          │
      │<─────────────────────────│                          │
      │ ② sign short-lived JWT   │                          │
      │─────────────────────────>│                          │
      │                          │ ③ AssumeRoleWithWebIdentity(JWT)
      │                          │─────────────────────────>│
      │                          │   ④ verify with GitHub's │
      │                          │     public keys, check sub/aud
      │                          │ ⑤ temp credentials (~1h) │
      │                          │<─────────────────────────│
      │                          │ ⑥ push to S3 ───────────> S3 ✓
```

**Why it's designed this way**: a long-lived secret in repo settings is a leak-and-rotation liability. OIDC outsources "proving who I am" to GitHub — it already knows exactly which repo and branch is running — so it signs a statement and AWS verifies it. The credential stops being *a thing you store* and becomes *an event you verify*: nothing to steal, nothing to keep.

**Common misreadings**: `id-token: write` isn't permission to "write a token somewhere" — it lets the job *request* an OIDC token. And the `role-to-assume` ARN isn't a secret; the real lock is the `sub` condition in the AWS-side trust policy.

*Want the full breakdown (JWT claims, tightening the trust policy, item-by-item comparison with access keys)? Just say "go deeper".*

</details>

## Install

```bash
git clone https://github.com/jyuwaaw/dianpo-skill.git
cp -r dianpo-skill/dianpo ~/.claude/skills/
```

`~/.claude/skills/dianpo/` makes it available in every project; for a single project, use `<project>/.claude/skills/dianpo/` instead. Works in Claude Code CLI, desktop, and web — new sessions pick it up automatically.

> The skill's instructions are written in Chinese (its native audience), but Claude reads them natively and **answers in whatever language you ask in** — English questions get English answers, diagrams and all.

## Usage

**Automatic triggering** — paste the thing you can't see through (a work note, a config snippet, a protocol description, a chunk of YAML) plus your confusion:

- "whose token/cert/key even is this?"
- "who is verifying whom here?" / "who trusts whom?"
- "why is it designed this way?" / "give me the mental model"
- "my colleague said X and I don't get what they meant"
- 中文同样触发:"帮我理解"、"这是谁的 token"、"谁验证谁"、"为什么要这么设计"

**Explicit invocation** — type `/dianpo` followed by your question.

**Escalation** — reply "go deeper" / "讲的专业点" to any answer for the unabridged version.

## Does it actually help?

Benchmarked against baseline Claude (no skill) on 3 scenarios — a domain-verification challenge, GitHub Actions OIDC → AWS, and Stripe webhook signature direction — with 10 assertions each:

| | with skill | without skill |
|---|---|---|
| assertions passed | **100%** (30/30) | 57% |
| prose length | ~1 screen | ~2–3 screens |

Honest caveat: **both sides got the facts right**. The entire gap is the discipline this skill enforces — conclusion first, actors and direction made explicit, diagrams readable as text, jargon glossed, one-screen length. It doesn't buy you "more correct"; it buys you "see through it faster". Small sample (3 scenarios × 1 run); treat as directional.

## Related

- [Claude Code skills documentation](https://code.claude.com/docs/en/skills) — how agent skills work
- Adjacent but different: thinking-framework packs (first principles, OODA, etc.) teach you *methods of thinking*; dianpo answers *this specific mechanism's* ownership and trust questions for you.

## License

[MIT](LICENSE) © [Benji (Y.H. Huang)](https://github.com/jyuwaaw)

---

*Built with [Claude Code](https://claude.com/claude-code). Skill design and validation by [@jyuwaaw](https://github.com/jyuwaaw).*

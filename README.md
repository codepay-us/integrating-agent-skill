# integrating-codepay-payments

An [Agent Skill](https://agentskills.io/specification) that guides an AI coding agent
through integrating **CodePay** (a.k.a. Wisehub / PayCloud) credit-card payment
terminals into a POS / cash-register app — in **any language or platform**
(Flutter/Dart, Kotlin, C#/.NET, TypeScript, Java…).

It is *architecture-aware*: instead of dumping boilerplate, it makes the agent
1. **refresh the CodePay protocol from the live docs** (so it never goes stale),
2. **pick the right integration topology** (on-terminal Intent / external-POS local / cloud),
3. **adapt to the host codebase's conventions** (its layering, state, settings, receipts), and
4. **implement sale / void / refund / tip** with correct recovery, idempotency, and money handling.

> ⚠️ Not affiliated with CodePay. The bundled protocol notes are a **dated snapshot** —
> the skill instructs the agent to verify against the official docs/SDK on every use.

## What's inside

| File | Purpose |
|---|---|
| `SKILL.md` | The playbook — triggers, the "don't invent the wire contract" rule, the topology decision, architecture-adaptation steps, recovery/amount/tip rules, red flags. |
| `codepay-protocol-reference.md` | The ECR Hub protocol snapshot — message envelope, topics, `biz_data` fields, response codes, tip models, PCI, RSA2, and links to the official docs + SDKs (the real source of truth). |

## Install

### Claude Code (CLI) — recommended
Clone it so the folder name **is** the skill name, into your personal skills dir:

```bash
# macOS / Linux
git clone <THIS_REPO_URL> ~/.claude/skills/integrating-codepay-payments

# Windows (PowerShell)
git clone <THIS_REPO_URL> $env:USERPROFILE\.claude\skills\integrating-codepay-payments
```

Or per-project: clone into `<project>/.claude/skills/integrating-codepay-payments`.

Then in any session, either just describe the task ("integrate CodePay card payments")
and Claude auto-discovers it via the `description`, or invoke it explicitly with
`/integrating-codepay-payments`.

**Update later:** `git -C ~/.claude/skills/integrating-codepay-payments pull`

### Claude.ai / Claude Desktop (Skills) & Claude Agent SDK
Zip the folder (or this whole repo) and upload it as a Skill, or point the Agent SDK's
skills loader at the folder.

### Other agents (Cursor, GitHub Copilot CLI, Gemini CLI, custom agents)
There may be no native skill auto-loader, but the content is plain Markdown — include
`SKILL.md` + `codepay-protocol-reference.md` as a rules file / system-prompt context /
knowledge doc. The only Claude-specific bits are the `WebFetch` tool name (map it to
your platform's web/browse tool) and description-based auto-triggering (invoke manually
instead). The CodePay knowledge and the integration process transfer unchanged.

## Keeping it current

CodePay updates its docs/SDKs over time. **Step 0** of the skill makes the agent fetch
the live docs and reconcile *before* trusting the bundled snapshot — so day-to-day usage
stays correct even if this repo lags. When the agent (or you) spot a drift, update
`codepay-protocol-reference.md` (and its snapshot date) and commit, so everyone benefits
on their next `git pull`.

## Contributing

PRs welcome — especially refreshing the protocol snapshot when CodePay changes a field,
enum, or response code. Keep the `SNAPSHOT` date in `codepay-protocol-reference.md`
current with any change.

## How it was built

Authored with the `writing-skills` TDD methodology: a baseline agent without the skill
**invents a wrong CodePay wire contract**; with the skill it uses the real ECR Hub
protocol, picks the right topology, and handles recovery/amounts/tips correctly.

## License

MIT — see [`LICENSE`](LICENSE).

# ⚡ power-automate-flows-skill

> 🇮🇹 [Leggi in italiano](README.it.md)

An [Agent Skill](https://agentskills.io/specification) (`SKILL.md` format) that turns **Claude** — and any other skill-aware coding agent — into a hands-on operator for **Power Automate cloud flows on Dataverse**. 🤖

## 💡 The idea

The whole point: **use Claude to create and edit Power Automate flows from the CLI/API instead of clicking around the maker portal.** 🖱️❌

Claude pulls the live flow definition, edits it as JSON, deploys by patching `clientdata`, and verifies the change by triggering a *real* run and reading its raw outputs — no UI, no guessing. It also covers document generation (Word/Excel → PDF) and the connector/identity gotchas that eat the most time. 🕳️

🌱 **This skill grows as it gets used.** Every new gotcha, error code, or workaround discovered in real work gets folded back in. So it's intentionally a living document — **pull requests with improvements are very welcome!** 🙏

## 📦 What's inside

One skill, `power-automate-flows`, with the reference docs split by functionality:

| File | What it covers |
|---|---|
| 📄 [`SKILL.md`](SKILL.md) | Entry point: overview, golden rules, the deploy+test loop, and an index into the reference files. |
| 🚀 [`deploy-and-test.md`](deploy-and-test.md) | The high-frequency loop pull → edit → deploy → trigger → verify; `az` tokens, `clientdata` PATCH, the **trigger re-registration trap** (the #1 time-waster), reading run outputs via `outputsLink`. |
| 📝 [`document-generation.md`](document-generation.md) | Document generation: connector matrix (Word/Excel/OneDrive/PDF), Word content controls keyed by `w:id`, variable-length list workaround, Excel→PDF and email-attachment traps. |
| 🔐 [`auth-environments.md`](auth-environments.md) | Auth & identities: `pac`/`az` drift, which identity can reach which drive (the OneDrive trap), connection references & binding, error → cause map. |

> ✅ The skill is **generic**: no org ids, GUIDs, tenants, or customer names are hardcoded — only placeholders (`<org>`, `<WF>`, `<ENV>`, `<expected>`). Drop it onto any environment.

## 🛠️ Installation

The skill goes in a folder named `power-automate-flows` inside your agent's skills directory. The most portable way is to clone the repo straight into that location. 📥

### 🟣 Claude Code

Personal skill (available across all projects):

```bash
git clone git@github.com:MeStakes/power-automate-flows-skill.git ~/.claude/skills/power-automate-flows
```

Or for the current project only: clone into `.claude/skills/power-automate-flows`. Claude Code auto-discovers it; invoke it with the `Skill` tool. ✨

### 🟢 Codex CLI

User-level skill:

```bash
git clone git@github.com:MeStakes/power-automate-flows-skill.git ~/.codex/skills/power-automate-flows
```

Or per project: `.agents/skills/power-automate-flows`. Codex auto-detects new skills; if it doesn't show up, restart Codex to force a rescan. 🔄

### 🔵 GitHub Copilot CLI

Personal skill (all projects):

```bash
git clone git@github.com:MeStakes/power-automate-flows-skill.git ~/.copilot/skills/power-automate-flows
```

Or per repo: clone into `.github/skills/power-automate-flows`. Skills are auto-discovered via the `*/SKILL.md` convention; invoke it with the `skill` tool.

> ℹ️ On gh CLI ≥ 2.90.0 there's also `gh skill install <org/repo>`, but the `git clone` above is the most reliable method and identical across all platforms.

## ♻️ Updating

```bash
cd <skills-dir>/power-automate-flows && git pull
```

## 📋 Usage prerequisites

`pac` (Power Platform CLI), `az` (Azure CLI, for Graph / Dataverse / Flow API tokens), and Python (urllib). Every operation is doable headless. 💻

## 🤝 Contributing

Found a new connector quirk, a cryptic error code, or a better workaround? Open a **pull request** — this skill improves with every real-world flow it touches. 🚀

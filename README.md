# Qubes OS Skills for AI Agents

Community-driven [SKILL.md](https://www.agensi.io/learn/agent-skills-open-standard) collection for **Qubes OS**.

AI agents (Claude Code, Codex, Cursor, OpenCode, Hermes, Gemini CLI, and 60+ more) can load these skills to correctly operate and manage Qubes OS — avoiding common pitfalls that come from treating Qubes like a regular Linux system.

## Why?

Qubes OS is architecturally different from any other operating system. Its security-by-compartmentalization model introduces concepts that AI agents consistently get wrong:

- AppVM / TemplateVM / StandaloneVM lifecycle and persistence
- qrexec RPC protocol and dom0 policy system
- ProxyVM networking and firewall chain architecture
- Xen-based isolation vs traditional process isolation

Generic Linux knowledge is not enough. These skills encode hard-won experience so agents don't make the same mistakes twice.

## Available Skills

| Skill | Description |
|-------|-------------|
| **qubes-vm-admin** | VM lifecycle, bind-dirs persistence, template management, dom0 operations |
| *More coming — contributions welcome!* | |

## Installation

Most AI coding agents support the SKILL.md standard. Install a single skill:

```bash
# Claude Code / Codex CLI
skill install iasds/qubes-os-skills/skills/qubes-basics

# Or copy the skill directory manually
cp -r skills/qubes-basics .claude/skills/  # Claude Code
cp -r skills/qubes-basics .codex/skills/   # Codex CLI
cp -r skills/qubes-basics .cursor/rules/   # Cursor
```

## Contributing

**We want your skill.** If you've figured out a Qubes OS workflow that an AI agent keeps getting wrong — or you want to share battle-tested automation — open a PR.

### Skill format

Each skill is a **directory** containing a `SKILL.md` file. Directory name = skill name. Use a `qubes-` prefix.

```
your-skill-name/
└── SKILL.md
```

`SKILL.md` must have YAML frontmatter (`---` delimited metadata):

```yaml
---
name: your-skill-name
description: "One-line description"
triggers:
  - trigger phrase 1
  - trigger phrase 2
---
```

### SKILL.md template

<details>
<summary>📄 Click to expand</summary>

```markdown
---
name: your-skill-name
description: "Short description of what this skill does"
triggers:
  - Qubes trigger keywords
  - Questions that should load this skill
  - Use case description
---

# Skill Title

## Background

Why is this skill needed? What problem do agents commonly get wrong?

## Core Concepts

- Key point 1
- Key point 2

## Steps

### Step 1: Description

```bash
command --option value
```

### Step 2: Description

Explanation and commands.

## Pitfalls

**⚠️ Pitfall name**

Symptoms, root cause, fix.

## Verification

How to confirm the operation succeeded:

```bash
verification-command
```

## References

- Official Qubes docs links
- Related issues or forum threads

## Notes

Author, experience summary, etc.
```

</details>

### Directory layout

Skills live under `skills/`. Multi-file skills (references, scripts, templates) go inside the skill directory:

```
skills/
├── qubes-basics/
│   └── SKILL.md
├── qubes-vm-admin/
│   ├── SKILL.md
│   └── references/
│       └── advanced-patterns.md
└── your-name/
    └── your-skill/
        └── SKILL.md
```

### Submission steps

1. Fork this repo
2. Branch: `git checkout -b skill/your-skill-name`
3. Create your skill directory and `SKILL.md`
4. Commit & push: `git commit -m "Add skill: your-skill-name" && git push`
5. Open a Pull Request with title `[Skill] your-skill-name`

### Review criteria

| Item | Requirement |
|------|-------------|
| Frontmatter | name, description, triggers all present |
| Triggers | Accurate keywords, agent can hit them |
| Commands | Runnable, reproducible |
| Pitfalls | At least one real gotcha documented |
| No PII | No real IPs/domains/secrets — use `<placeholder>` |
| No sensitive content | No subscription links, proxy nodes, crack tools |

### What makes a good skill?

- **Trigger-heavy frontmatter** — makes it easy for agents to match
- **Pitfalls section** — the most valuable part, where agents mess up
- **Verification steps** — how to confirm it worked
- **Real commands** — copy-pasteable, not theory
- **Experience** — not just official docs, but hard-won lessons

### What to submit

Anything Qubes OS related that agents need to know:

- Custom qrexec services + dom0 policy
- bind-dirs persistence gotchas
- Template → AppVM sync after updates
- nftables special behavior on Qubes
- PCI / GPU passthrough steps
- Backup & restore best practices
- DispVM customization
- QSB tracking
- Anything you spent an afternoon debugging

### What NOT to submit

- Plain doc copy (use [qubes-os-ai-knowledge-base](https://github.com/iasds/qubes-os-ai-knowledge-base) instead)
- Subscription links, proxy nodes, crack tools
- PII (real names, emails, IPs, keys)

### Attribution

Contributors are credited in the skill's `SKILL.md` Notes section and the repo's contributor list.

## License

GNU General Public License v2.0 — same as Qubes OS.

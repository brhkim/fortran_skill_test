# Fortran Skill Demonstration

# fortran — an Agent Skill for modern Fortran

A self-contained [Agent Skill](https://code.claude.com/docs/en/skills) that steers AI coding assistants toward **modern Fortran** (2018/2023 idiom, fpm tooling, verified compiler flags) instead of the legacy FORTRAN 77 style they tend to produce by default.

## ⚠️ No warranties — read this first

This skill was assembled by AI agents doing structured web research, with a separate AI review pass for citation completeness. **The author has never written a line of Fortran and doesn't plan to start.** It is offered as a *friendly push in the right direction* and a demonstration of how skills can be built — not as vetted expert guidance. Expect to iterate: if you actually write Fortran, please correct it, extend it, and treat every claim as checkable (each one carries the source URL and access date for exactly that reason).

## What's inside

```
fortran/
├── SKILL.md                          # Entry point: core rules, decision tree, compile-verify loop
├── references/
│   ├── modern-idiom.md               # Modern-defaults ruleset, arrays, floats, strings, OOP, gotchas
│   ├── toolchain.md                  # fpm, stdlib, compilers, strictness/debug flags
│   ├── legacy-and-interop.md         # Maintaining F77, incremental modernization, C interop
│   └── standards.md                  # F77→F2023 timeline, F2023 delta, free standard PDFs
└── README.md                         # This file (ignored by skill loaders)
```

Every substantive claim in the reference files cites its exact source URL with an access date, and each file opens with a metadata block (creation date, verified-as-of date, and a staleness note telling future maintainers what drifts fastest and where to re-check).

## Installation

### Claude Code (CLI / IDE)

Copy this folder into your skills directory:

```bash
# Per-project:
mkdir -p .claude/skills
cp -r fortran .claude/skills/fortran

# Or globally, for all your projects:
mkdir -p ~/.claude/skills
cp -r fortran ~/.claude/skills/fortran
```

Claude Code auto-discovers it on the next session. Invoke explicitly with `/fortran`, or just start asking Fortran questions — the skill description triggers it automatically.

### Claude.ai (web)

1. Zip this folder (the zip must contain `SKILL.md` at the top level of the `fortran/` folder).
2. Go to **Settings → Capabilities → Skills** and upload the zip.
3. Requires an account with skills/code-execution enabled. Claude will use the skill automatically when a conversation involves Fortran.

### Claude Desktop

Same as Claude.ai — skills uploaded to your account are available in the desktop app as well. Alternatively, check **Settings → Capabilities** in the app for a local skill upload option.

### OpenAI Codex

Codex supports the SKILL.md format natively (since December 2025), so this skill works as-is:

```bash
# Per-repository:
mkdir -p .agents/skills
cp -r fortran .agents/skills/fortran

# Or user-level, for all your projects:
mkdir -p ~/.agents/skills
cp -r fortran ~/.agents/skills/fortran
```

Invoke explicitly with `$fortran` (or browse via `/skills`), or let Codex trigger it automatically from the description. (Paths per OpenAI's skills docs, https://developers.openai.com/codex/skills, accessed 2026-07-26.)

Tool interfaces change quickly — if these steps have drifted, check your tool's current docs for "skills" or "agent instructions."

## How this was made (verbatim prompts)

For transparency, these are the actual prompts used to create this skill, in order, in a Claude Code session (Fable 5). Everything else — research, drafting, citations, review — was done by the agents.

> I'm curious: Are there any online documentation resources for the Fortran language? I'm wondering if it's possible to write a good Skill for fortran to support coding assistants

> Great, can you go ahead and sketch a plan to do so with some research sleuthing first to scope?

> Great, I'd like you to make this completely standalone. Make sure all references are thoroughly and carefully reviewed, cited exactly of the resources they came from online across websites. I'd like you to have several agents put this together, and then a separate pass for consistency/accuracy and citation completeness. This is for a colleague to demonstrate how skills work. Don't forget to include date of creation, and other flags that help make it maintainable by agents using this in the future

> Great, in addition to this, I'll plan to upload it to GitHub. Can you draft me a simple README to instruct them on how to install it in their favorite tool (lets cover Claude.ai, Claude Desktop, Claude Code, and Codex) and also give a quick note on exactly my prompts to do this? Verbatim for their visibility. Make no warranties; I've never touched Fortran and never intend to, so this is purely to give a simple sense of direction that should be iterated on! Friendly push in the right direction, in other words.

>

The build itself used four parallel research/drafting agents (one per reference file, each fetching sources live), an orchestrator-written SKILL.md, and a separate top-tier review agent auditing consistency, accuracy, and citation completeness across all files.

## Maintenance

- **Maintainer:** Brian Heseung Kim ([@brhkim](https://github.com/brhkim)) — a non-Fortran-user; issues/PRs from actual Fortran folks very welcome.
- **Created:** 2026-07-26. Sources verified as of that date.
- Fastest-drifting content: compiler versions and Fortran 2023 conformance status (see the staleness notes at the top of each reference file for exactly which pages to re-check).
- If you regenerate or update a reference file, update its verified-as-of date and keep the citation convention: every claim gets a URL and access date.

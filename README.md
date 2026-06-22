# Chrome Extensions (Manifest V3) Skill

> _Faithful, task-routed reference for building, scaffolding, and debugging Manifest V3 Chrome, Chromium, and Edge browser extensions._

Part of **[Agent Kaizen](https://github.com/DirectorGunner/agent-kaizen)** — Agent Kaizen is designed to support reliable AI agent workflows, context engineering, and AI systems engineering in VS Code, Codex, and Claude Code. Build reusable agent skills, reduce unnecessary context loading, add validation loops, and apply Spec → Verifier → Environment scaffolding to new and existing projects.

This repository is the `chrome-extensions` skill: a reusable, trigger-rich task handbook that an AI coding agent (OpenAI Codex, Claude Code) loads on demand when a task matches its triggers.

## What this skill covers

This skill covers Manifest V3 extension development end to end: the `manifest.json` format (`manifest_version`, action/background, content_scripts, permissions, `host_permissions`, side_panel, CSP keys), background **service workers**, content scripts, popups, options pages, side panels, omnibox, and context menus. It routes the full surface of `chrome.*` / `browser.*` APIs — `action`, `tabs`, `scripting`, `storage`, `runtime`, `alarms`, `permissions`, `declarativeNetRequest`, `webRequest`, `identity`, `sidePanel`, `commands`, `cookies`, `notifications`, and `devtools` — plus message passing, native messaging, and loading unpacked extensions. It also handles MV2-to-MV3 migration, treating Manifest V2 only as labeled migration context. It is not for Firefox/Safari-specific WebExtension behavior, ordinary web-page automation, or Chrome Web Store listing and publishing beyond manifest keys.

## What's inside

- `SKILL.md` — frontmatter (`name` + trigger-rich `description`) and a lean body.
- `references/` — right-sized topic files the agent loads only when relevant (plus `INDEX.md` and `topics.json`).
- `GOTCHA.md` — known pitfalls and edge cases, such as the event-driven service worker that terminates when idle, requiring state to persist to `chrome.storage` with listeners registered at the top level.

## Use it

This skill is one git repo inside the Agent Kaizen **skills store**. The store nests two folders on purpose: the outer **`SKILLS\`** is a VS Code project for building and maintaining skills (its own workspace + tooling), and the inner lowercase **`skills\`** holds every skill as its own repo. That split lets a project pull skills two ways — the **whole `skills\` folder at once** (loads everything — **not recommended**) or **one skill at a time** (recommended: load only what a task needs and stay under Claude Code's skill-listing budget).

Paths below use **`%DEVROOT%`** — the `DEVROOT` environment variable pointing at the folder that contains `SKILLS\`. Set it once by running **`SetDevRoot.cmd`** in the SKILLS repo root (no admin); a new shell then resolves `%DEVROOT%` automatically.

Link **just this skill** (recommended) — a Windows directory junction, no admin:

```cmd
mklink /J .agents\skills\chrome-extensions  "%DEVROOT%\SKILLS\skills\chrome-extensions"
mklink /J .claude\skills\chrome-extensions  "%DEVROOT%\SKILLS\skills\chrome-extensions"
```

Or link the **whole store** at once (loads every skill — not recommended outside a skills-dev project):

```cmd
mklink /J .agents\skills  "%DEVROOT%\SKILLS\skills"
mklink /J .claude\skills  "%DEVROOT%\SKILLS\skills"
```

Remove a link (the store copy is untouched):

```cmd
rmdir .agents\skills\chrome-extensions
rmdir .claude\skills\chrome-extensions
```

The agent (OpenAI Codex, Claude Code) then auto-loads this skill whenever a task matches its triggers. Built and validated with **[skill-drafting](https://github.com/DirectorGunner/AI-skill-drafting)** to the Agent Kaizen gold standard.

## License

Licensed under **AGPL-3.0**, matching the [Agent Kaizen](https://github.com/DirectorGunner/agent-kaizen) project.

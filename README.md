# tauri-pty Skill

Agent skill for building terminal emulators in Tauri 2 + React.

## What's Inside

```
tauri-pty/
├── SKILL.md                   ← Entry point
└── references/
    ├── rust-backend.md        ← Cargo setup, PTY manager, Tauri commands
    ├── react-frontend.md      ← React components, hooks, TypeScript
    ├── xterm-integration.md   ← Addons, themes, resize, WebGL
    ├── multi-tab.md           ← Tab management, session lifecycle
    └── best-practices.md      ← Performance, cross-platform, common bugs
```

## Installation

Each agent has its own directory convention for injecting context. Copy the skill files to the right place for your agent.

### Claude Code
```
.claude/skills/tauri-pty/       ← project-level (commit to repo)
~/.claude/skills/tauri-pty/     ← global (all projects)
```
Claude Code auto-discovers skills from these directories. No config needed.

### Claude.ai
Upload `tauri-pty.skill` via **Settings → Customize → Skills → Upload Skill**.

### Cursor
```
.cursor/skills/tauri-pty
```

### Windsurf (Cascade)

```
.windsurf/skills/tauri-pty 
```

### GitHub Copilot / VS Code
```
.github/skills/tauri-pty ← always-on repo instructions
or
.copilot/skills/tauri-pty                  
```

### Any Other Agent (common universal approach)
```
.agent/skills/tauri-pty                
```

## Stack

- **Tauri 2** + **Rust** backend
- **tauri-plugin-pty** or manual **portable_pty**
- **React** + **TypeScript** frontend
- **xterm.js** renderer

# tauri-pty Skill

A Claude skill for building terminal emulators in Tauri 2 + React.

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

1. Download `tauri-pty.skill`
2. Go to **Claude.ai → Settings → Skills → Upload Skill**
3. Done — Claude will use this skill automatically when you're working on a Tauri terminal project

## Stack

- **Tauri 2** + **Rust** backend
- **tauri-plugin-pty** or manual **portable_pty**
- **React** + **TypeScript** frontend
- **xterm.js** renderer

## License

MIT

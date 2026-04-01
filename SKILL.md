---
name: tauri-pty
description: >
  Expert guide for building terminal emulators and PTY (pseudo-terminal) integrations in Tauri 2
  applications with a React frontend. Use this skill whenever the user mentions: building a terminal
  in Tauri, using portable_pty, tauri-plugin-pty, xterm.js in a Tauri app, spawning shell processes
  in Tauri, streaming PTY output, wiring keyboard input to a shell, or building anything that behaves
  like a terminal tab or terminal emulator inside a desktop app. Also trigger for: "how do I run
  commands in Tauri", "Tauri shell integration", "interactive CLI in Tauri", or "embed a terminal in
  my Tauri app". This skill covers the full vertical stack — Cargo.toml setup, Rust plugin
  registration, React component wiring, xterm theming, resize handling, multi-tab management,
  cross-platform shell detection, and production best practices. Always use this skill in full when
  the user is building or debugging a Tauri terminal — don't rely on memory alone.
---

# Tauri PTY Skill

A complete vertical guide for embedding a fully functional terminal emulator inside a Tauri 2 +
React desktop application using `tauri-plugin-pty` and `xterm.js`.

---

## Quick Decision: Which Approach?

| Situation | Recommended Approach |
|---|---|
| Standard terminal tab/panel | `tauri-plugin-pty` (Option A — this skill) |
| Need deep PTY lifecycle control | Manual `portable_pty` (Option B — see `references/rust-backend.md`) |
| Multiple terminal tabs | `tauri-plugin-pty` + React state (covered here) |
| PTY inside a Tauri Webview only | Option A |
| Streaming huge volumes of data | Consider Tauri Channels — see `references/best-practices.md` |

**This skill defaults to Option A (`tauri-plugin-pty`).** It is the least boilerplate, most
maintainable path for the vast majority of Tauri terminal use cases.

---

## Reference Files — Read These As Needed

| File | When to Read It |
|---|---|
| `references/rust-backend.md` | Cargo setup, plugin registration, manual `portable_pty` deep dive |
| `references/react-frontend.md` | Full React component code, hooks, TypeScript types |
| `references/xterm-integration.md` | xterm.js addons, themes, fonts, key bindings, Unicode |
| `references/multi-tab.md` | Tab management, session persistence, splitting panes |
| `references/best-practices.md` | Performance, security, cross-platform quirks, common bugs |

---

## Skill Workflow

When a user asks to build a Tauri terminal, follow this sequence:

1. **Read `references/rust-backend.md`** — set up Cargo.toml and register the plugin
2. **Read `references/react-frontend.md`** — scaffold the Terminal React component
3. **Read `references/xterm-integration.md`** — wire addons, apply theme, handle resize
4. If multi-tab is needed → **read `references/multi-tab.md`**
5. Before finalizing → **read `references/best-practices.md`** for gotchas

Always produce working, copy-pasteable code. Never produce pseudocode unless the user asks for it.

---

## Minimal Working Example (Quick Reference)

### Cargo.toml
```toml
[dependencies]
tauri = { version = "2", features = [] }
tauri-plugin-pty = "0.1"
```

### src-tauri/src/lib.rs
```rust
pub fn run() {
    tauri::Builder::default()
        .plugin(tauri_plugin_pty::init())
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### Frontend install
```bash
npm install tauri-pty @xterm/xterm @xterm/addon-fit @xterm/addon-web-links
```

### Terminal.tsx (minimal)
```tsx
import { useEffect, useRef } from "react";
import { Terminal } from "@xterm/xterm";
import { FitAddon } from "@xterm/addon-fit";
import { spawn } from "tauri-pty";
import "@xterm/xterm/css/xterm.css";

export default function TerminalView() {
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!ref.current) return;
    const term = new Terminal({ cursorBlink: true });
    const fit = new FitAddon();
    term.loadAddon(fit);
    term.open(ref.current);
    fit.fit();

    const shell = navigator.userAgent.includes("Windows") ? "powershell.exe" : "bash";
    const pty = spawn(shell, [], {
      cols: term.cols,
      rows: term.rows,
      env: { TERM: "xterm-256color" },
    });

    pty.onData((data) => term.write(data));
    term.onData((data) => pty.write(data));
    term.onResize(({ cols, rows }) => pty.resize(cols, rows));

    const onResize = () => fit.fit();
    window.addEventListener("resize", onResize);
    return () => {
      window.removeEventListener("resize", onResize);
      term.dispose();
    };
  }, []);

  return <div ref={ref} style={{ width: "100%", height: "100%", background: "#1e1e1e" }} />;
}
```

For anything beyond this minimal example, read the relevant reference files listed above.

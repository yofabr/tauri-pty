# React Frontend Reference

## Table of Contents
1. [Dependencies](#dependencies)
2. [Basic Terminal Component](#basic-component)
3. [Full Production Terminal Component](#production-component)
4. [Custom Hook: useTerminal](#use-terminal-hook)
5. [TypeScript Types](#typescript-types)
6. [Using with Option B (Manual Commands)](#option-b-frontend)
7. [Shell Selector Component](#shell-selector)
8. [Keyboard Shortcuts](#keyboard-shortcuts)
9. [Copy/Paste Handling](#copy-paste)

---

## 1. Dependencies {#dependencies}

```bash
# Core
npm install tauri-pty @xterm/xterm @xterm/addon-fit

# Recommended addons
npm install @xterm/addon-web-links     # Clickable URLs in terminal output
npm install @xterm/addon-search        # Ctrl+F search in terminal
npm install @xterm/addon-unicode11     # Better Unicode/emoji support
npm install @xterm/addon-image         # Inline image support (iTerm2 protocol)

# For Option B (manual Tauri commands)
npm install @tauri-apps/api
```

---

## 2. Basic Terminal Component {#basic-component}

Minimal, working component. Good starting point.

```tsx
// src/components/Terminal.tsx
import { useEffect, useRef } from "react";
import { Terminal } from "@xterm/xterm";
import { FitAddon } from "@xterm/addon-fit";
import { spawn } from "tauri-pty";
import "@xterm/xterm/css/xterm.css";

export default function TerminalView() {
  const containerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!containerRef.current) return;

    const term = new Terminal({
      cursorBlink: true,
      fontSize: 14,
      fontFamily: '"Cascadia Code", "Fira Code", monospace',
    });

    const fitAddon = new FitAddon();
    term.loadAddon(fitAddon);
    term.open(containerRef.current);
    fitAddon.fit();

    const shell = navigator.userAgent.includes("Windows")
      ? "powershell.exe"
      : "bash";

    const pty = spawn(shell, [], {
      cols: term.cols,
      rows: term.rows,
      env: { TERM: "xterm-256color" },
    });

    pty.onData((data) => term.write(data));
    term.onData((data) => pty.write(data));
    term.onResize(({ cols, rows }) => pty.resize(cols, rows));

    const handleResize = () => fitAddon.fit();
    window.addEventListener("resize", handleResize);

    return () => {
      window.removeEventListener("resize", handleResize);
      term.dispose();
    };
  }, []);

  return (
    <div
      ref={containerRef}
      style={{ width: "100%", height: "100%", background: "#1e1e1e", padding: 4 }}
    />
  );
}
```

---

## 3. Full Production Terminal Component {#production-component}

This component includes:
- Proper cleanup and ref guards
- ResizeObserver (more reliable than window resize)
- Shell detection passed as prop
- onExit callback
- Overlay for "session ended" state
- Accessible container attributes

```tsx
// src/components/Terminal.tsx
import { useEffect, useRef, useState, useCallback } from "react";
import { Terminal } from "@xterm/xterm";
import { FitAddon } from "@xterm/addon-fit";
import { WebLinksAddon } from "@xterm/addon-web-links";
import { Unicode11Addon } from "@xterm/addon-unicode11";
import { spawn } from "tauri-pty";
import "@xterm/xterm/css/xterm.css";

export interface TerminalProps {
  shell?: string;
  initialCols?: number;
  initialRows?: number;
  theme?: Partial<ITheme>;
  fontSize?: number;
  fontFamily?: string;
  onExit?: (code: number) => void;
  className?: string;
  style?: React.CSSProperties;
}

interface ITheme {
  background: string;
  foreground: string;
  cursor: string;
  cursorAccent: string;
  selectionBackground: string;
  black: string; white: string;
  red: string; green: string; yellow: string; blue: string;
  magenta: string; cyan: string;
  brightBlack: string; brightWhite: string;
  brightRed: string; brightGreen: string;
  brightYellow: string; brightBlue: string;
  brightMagenta: string; brightCyan: string;
}

// VS Code Dark+ theme
export const vscodeDarkTheme: Partial<ITheme> = {
  background: "#1e1e1e",
  foreground: "#d4d4d4",
  cursor: "#aeafad",
  selectionBackground: "#264f78",
  black: "#000000", white: "#d4d4d4",
  red: "#f44747", green: "#608b4e",
  yellow: "#dcdcaa", blue: "#569cd6",
  magenta: "#c678dd", cyan: "#56b6c2",
  brightBlack: "#808080", brightWhite: "#ffffff",
  brightRed: "#f44747", brightGreen: "#89d185",
  brightYellow: "#dcdcaa", brightBlue: "#4fc1ff",
  brightMagenta: "#c678dd", brightCyan: "#9cdcfe",
};

export const detectShell = (): string => {
  const ua = navigator.userAgent;
  if (ua.includes("Windows")) return "powershell.exe";
  if (ua.includes("Mac")) return "/bin/zsh";
  return "/bin/bash";
};

export default function TerminalView({
  shell,
  theme = vscodeDarkTheme,
  fontSize = 14,
  fontFamily = '"Cascadia Code", "Fira Code", Menlo, monospace',
  onExit,
  className,
  style,
}: TerminalProps) {
  const containerRef = useRef<HTMLDivElement>(null);
  const termRef = useRef<Terminal | null>(null);
  const fitAddonRef = useRef<FitAddon | null>(null);
  const [exited, setExited] = useState(false);

  const handleResize = useCallback(() => {
    fitAddonRef.current?.fit();
  }, []);

  useEffect(() => {
    if (!containerRef.current) return;

    // --- Init xterm ---
    const term = new Terminal({
      cursorBlink: true,
      cursorStyle: "block",
      fontSize,
      fontFamily,
      theme: theme as ITheme,
      allowProposedApi: true,        // Required for some addons
      scrollback: 10000,             // Lines of scrollback history
      convertEol: true,              // Convert \n to \r\n automatically
      windowsMode: navigator.userAgent.includes("Windows"),
    });

    const fitAddon = new FitAddon();
    const webLinksAddon = new WebLinksAddon();
    const unicodeAddon = new Unicode11Addon();

    term.loadAddon(fitAddon);
    term.loadAddon(webLinksAddon);
    term.loadAddon(unicodeAddon);
    term.unicode.activeVersion = "11"; // Use Unicode 11

    term.open(containerRef.current);

    // Must fit AFTER open() so dimensions are known
    requestAnimationFrame(() => fitAddon.fit());

    termRef.current = term;
    fitAddonRef.current = fitAddon;

    // --- Spawn PTY ---
    const resolvedShell = shell ?? detectShell();

    const pty = spawn(resolvedShell, [], {
      cols: term.cols,
      rows: term.rows,
      env: {
        TERM: "xterm-256color",
        COLORTERM: "truecolor",
        LANG: "en_US.UTF-8",
      },
    });

    // PTY → display
    pty.onData((data) => {
      if (!term.element) return; // Guard: component may be unmounting
      term.write(data);
    });

    // PTY exit
    pty.onExit(({ exitCode }) => {
      setExited(true);
      onExit?.(exitCode);
      term.write(`\r\n\x1b[31m[Process exited with code ${exitCode}]\x1b[0m\r\n`);
    });

    // Keyboard → PTY
    term.onData((data) => pty.write(data));

    // Resize
    term.onResize(({ cols, rows }) => pty.resize(cols, rows));

    // Use ResizeObserver on the container — more reliable than window resize
    const resizeObserver = new ResizeObserver(() => {
      requestAnimationFrame(() => fitAddon.fit());
    });
    resizeObserver.observe(containerRef.current);

    return () => {
      resizeObserver.disconnect();
      term.dispose();
      termRef.current = null;
      fitAddonRef.current = null;
    };
  }, []); // Empty deps — PTY spawns once per mount

  // Expose resize externally (e.g., when a tab panel becomes visible)
  useEffect(() => {
    fitAddonRef.current?.fit();
  });

  return (
    <div
      className={className}
      style={{
        width: "100%",
        height: "100%",
        backgroundColor: (theme as ITheme)?.background ?? "#1e1e1e",
        position: "relative",
        ...style,
      }}
      role="terminal"
      aria-label="Terminal"
    >
      <div ref={containerRef} style={{ width: "100%", height: "100%" }} />

      {exited && (
        <div
          style={{
            position: "absolute",
            bottom: 8,
            right: 12,
            color: "#888",
            fontSize: 12,
            pointerEvents: "none",
          }}
        >
          Process exited — press Enter to start a new session
        </div>
      )}
    </div>
  );
}
```

---

## 4. Custom Hook: useTerminal {#use-terminal-hook}

Extract all terminal logic into a reusable hook for cleaner components:

```tsx
// src/hooks/useTerminal.ts
import { useEffect, useRef, useCallback } from "react";
import { Terminal } from "@xterm/xterm";
import { FitAddon } from "@xterm/addon-fit";
import { spawn } from "tauri-pty";

interface UseTerminalOptions {
  shell?: string;
  onExit?: (code: number) => void;
}

export function useTerminal(
  containerRef: React.RefObject<HTMLDivElement>,
  options: UseTerminalOptions = {}
) {
  const termRef = useRef<Terminal | null>(null);
  const fitAddonRef = useRef<FitAddon | null>(null);

  const fitTerminal = useCallback(() => {
    fitAddonRef.current?.fit();
  }, []);

  useEffect(() => {
    if (!containerRef.current) return;

    const term = new Terminal({
      cursorBlink: true,
      fontSize: 14,
      fontFamily: "monospace",
      theme: { background: "#1e1e1e", foreground: "#d4d4d4" },
      scrollback: 5000,
    });

    const fit = new FitAddon();
    term.loadAddon(fit);
    term.open(containerRef.current);
    requestAnimationFrame(() => fit.fit());

    termRef.current = term;
    fitAddonRef.current = fit;

    const shell = options.shell ?? (navigator.userAgent.includes("Windows") ? "powershell.exe" : "bash");
    const pty = spawn(shell, [], {
      cols: term.cols,
      rows: term.rows,
      env: { TERM: "xterm-256color" },
    });

    pty.onData((d) => term.write(d));
    pty.onExit(({ exitCode }) => options.onExit?.(exitCode));
    term.onData((d) => pty.write(d));
    term.onResize(({ cols, rows }) => pty.resize(cols, rows));

    const ro = new ResizeObserver(() => requestAnimationFrame(() => fit.fit()));
    ro.observe(containerRef.current);

    return () => {
      ro.disconnect();
      term.dispose();
    };
  }, []);

  return { term: termRef, fitTerminal };
}

// Usage:
// const containerRef = useRef<HTMLDivElement>(null);
// const { fitTerminal } = useTerminal(containerRef, { shell: "/bin/zsh" });
// <div ref={containerRef} style={{ width: "100%", height: "100%" }} />
```

---

## 5. TypeScript Types {#typescript-types}

```ts
// src/types/terminal.ts

export interface TerminalSession {
  id: string;                // crypto.randomUUID()
  title: string;             // Tab label
  shell: string;             // Shell path
  cwd?: string;              // Working directory
  isActive: boolean;
  exitCode?: number;
}

export interface PtySpawnOptions {
  cols: number;
  rows: number;
  cwd?: string;
  env?: Record<string, string>;
}

export type ShellType = "bash" | "zsh" | "fish" | "powershell" | "pwsh" | "cmd";
```

---

## 6. Using with Option B (Manual Tauri Commands) {#option-b-frontend}

When using manual `portable_pty` + Tauri commands instead of `tauri-plugin-pty`:

```tsx
import { invoke } from "@tauri-apps/api/core";
import { listen } from "@tauri-apps/api/event";

// Spawn a PTY session
const sessionId = crypto.randomUUID();

await invoke("spawn_pty", {
  sessionId,
  shell: null,    // null = auto-detect on Rust side
  cols: term.cols,
  rows: term.rows,
});

// Listen for output (per-session event name)
const unlisten = await listen<string>(`pty-output:${sessionId}`, (event) => {
  term.write(event.payload);
});

// Listen for exit
const unlistenExit = await listen(`pty-exit:${sessionId}`, () => {
  term.write("\r\n\x1b[31m[Session ended]\x1b[0m");
});

// Send input
term.onData((data) => {
  invoke("write_pty", { sessionId, data });
});

// Resize
term.onResize(({ cols, rows }) => {
  invoke("resize_pty", { sessionId, cols, rows });
});

// Cleanup
return () => {
  unlisten();
  unlistenExit();
  invoke("kill_pty", { sessionId });
  term.dispose();
};
```

---

## 7. Shell Selector Component {#shell-selector}

```tsx
// src/components/ShellSelector.tsx
import { useState } from "react";

const SHELLS = [
  { label: "bash", value: "/bin/bash" },
  { label: "zsh", value: "/bin/zsh" },
  { label: "fish", value: "/usr/bin/fish" },
  { label: "PowerShell", value: "pwsh" },
];

interface Props {
  onSelect: (shell: string) => void;
}

export default function ShellSelector({ onSelect }: Props) {
  const [selected, setSelected] = useState(SHELLS[0].value);

  return (
    <select
      value={selected}
      onChange={(e) => {
        setSelected(e.target.value);
        onSelect(e.target.value);
      }}
      style={{ background: "#2d2d2d", color: "#d4d4d4", border: "none", padding: "2px 8px" }}
    >
      {SHELLS.map((s) => (
        <option key={s.value} value={s.value}>{s.label}</option>
      ))}
    </select>
  );
}
```

---

## 8. Keyboard Shortcuts {#keyboard-shortcuts}

```tsx
// Prevent Tauri window shortcuts from intercepting terminal keys
// Add to your Terminal component's useEffect:

term.attachCustomKeyEventHandler((event) => {
  // Allow Ctrl+C / Ctrl+V to pass through to terminal (not browser)
  if (event.ctrlKey && (event.key === "c" || event.key === "v")) {
    return true; // Let xterm handle it
  }
  // Block browser default for F5 (refresh) inside terminal
  if (event.key === "F5") {
    event.preventDefault();
    return false;
  }
  return true; // Pass all other keys through
});
```

---

## 9. Copy/Paste Handling {#copy-paste}

```tsx
// xterm.js handles copy automatically on selection (rightMouseButton mode)
// For explicit Ctrl+Shift+C / Ctrl+Shift+V:

term.attachCustomKeyEventHandler((event) => {
  if (event.ctrlKey && event.shiftKey) {
    if (event.key === "C") {
      const selection = term.getSelection();
      if (selection) navigator.clipboard.writeText(selection);
      return false;
    }
    if (event.key === "V") {
      navigator.clipboard.readText().then((text) => {
        pty.write(text);
      });
      return false;
    }
  }
  return true;
});
```

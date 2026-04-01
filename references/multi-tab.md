# Multi-Tab Terminal Management

## Table of Contents
1. [Architecture Overview](#architecture)
2. [Tab State with Zustand](#zustand-state)
3. [Tab State with useReducer (no deps)](#reducer-state)
4. [TerminalManager Component (Full Implementation)](#terminal-manager)
5. [Individual Tab Component](#tab-component)
6. [Tab Bar Component](#tab-bar)
7. [Hiding vs Unmounting Tabs (CRITICAL)](#hide-vs-unmount)
8. [Fit on Tab Switch](#fit-on-switch)
9. [Renaming Tabs](#renaming-tabs)
10. [Drag to Reorder Tabs](#drag-reorder)

---

## 1. Architecture Overview {#architecture}

```
TerminalManager
├── TabBar               ← tab buttons, add/close/rename
└── TabPanels
    ├── TabPanel (id=1)  ← display: block (active)
    │   └── TerminalView
    ├── TabPanel (id=2)  ← display: none (inactive — NOT unmounted)
    │   └── TerminalView
    └── TabPanel (id=3)  ← display: none (inactive)
        └── TerminalView
```

**Key principle**: Inactive tabs are hidden with `display: none`, NOT unmounted.
Unmounting destroys the PTY session. See section 7.

---

## 2. Tab State with Zustand {#zustand-state}

```bash
npm install zustand
```

```ts
// src/stores/terminalStore.ts
import { create } from "zustand";

export interface TabSession {
  id: string;
  title: string;
  shell?: string;
  isActive: boolean;
}

interface TerminalStore {
  tabs: TabSession[];
  activeId: string | null;

  addTab: (shell?: string) => void;
  closeTab: (id: string) => void;
  setActive: (id: string) => void;
  renameTab: (id: string, title: string) => void;
}

export const useTerminalStore = create<TerminalStore>((set, get) => ({
  tabs: [],
  activeId: null,

  addTab: (shell) => {
    const id = crypto.randomUUID();
    const newTab: TabSession = {
      id,
      title: `Terminal ${get().tabs.length + 1}`,
      shell,
      isActive: true,
    };
    set((s) => ({
      tabs: [...s.tabs, newTab],
      activeId: id,
    }));
  },

  closeTab: (id) => {
    const { tabs, activeId } = get();
    const idx = tabs.findIndex((t) => t.id === id);
    const remaining = tabs.filter((t) => t.id !== id);

    // If closing active tab, activate the nearest remaining tab
    let newActiveId = activeId;
    if (activeId === id) {
      newActiveId = remaining[Math.min(idx, remaining.length - 1)]?.id ?? null;
    }

    set({ tabs: remaining, activeId: newActiveId });
  },

  setActive: (id) => set({ activeId: id }),

  renameTab: (id, title) =>
    set((s) => ({
      tabs: s.tabs.map((t) => (t.id === id ? { ...t, title } : t)),
    })),
}));
```

---

## 3. Tab State with useReducer (no deps) {#reducer-state}

If you prefer not to add Zustand:

```tsx
// src/hooks/useTabManager.ts
import { useReducer, useCallback } from "react";

export interface TabSession {
  id: string;
  title: string;
  shell?: string;
}

interface TabState {
  tabs: TabSession[];
  activeId: string | null;
}

type TabAction =
  | { type: "ADD"; shell?: string }
  | { type: "CLOSE"; id: string }
  | { type: "SET_ACTIVE"; id: string }
  | { type: "RENAME"; id: string; title: string };

function tabReducer(state: TabState, action: TabAction): TabState {
  switch (action.type) {
    case "ADD": {
      const id = crypto.randomUUID();
      return {
        tabs: [...state.tabs, { id, title: `Terminal ${state.tabs.length + 1}`, shell: action.shell }],
        activeId: id,
      };
    }
    case "CLOSE": {
      const idx = state.tabs.findIndex((t) => t.id === action.id);
      const remaining = state.tabs.filter((t) => t.id !== action.id);
      const newActive =
        state.activeId === action.id
          ? remaining[Math.min(idx, remaining.length - 1)]?.id ?? null
          : state.activeId;
      return { tabs: remaining, activeId: newActive };
    }
    case "SET_ACTIVE":
      return { ...state, activeId: action.id };
    case "RENAME":
      return {
        ...state,
        tabs: state.tabs.map((t) => (t.id === action.id ? { ...t, title: action.title } : t)),
      };
    default:
      return state;
  }
}

export function useTabManager(initialShell?: string) {
  const [state, dispatch] = useReducer(tabReducer, { tabs: [], activeId: null }, (init) => {
    const id = crypto.randomUUID();
    return {
      tabs: [{ id, title: "Terminal 1", shell: initialShell }],
      activeId: id,
    };
  });

  return {
    ...state,
    addTab: useCallback((shell?: string) => dispatch({ type: "ADD", shell }), []),
    closeTab: useCallback((id: string) => dispatch({ type: "CLOSE", id }), []),
    setActive: useCallback((id: string) => dispatch({ type: "SET_ACTIVE", id }), []),
    renameTab: useCallback((id: string, title: string) => dispatch({ type: "RENAME", id, title }), []),
  };
}
```

---

## 4. TerminalManager Component (Full Implementation) {#terminal-manager}

```tsx
// src/components/TerminalManager.tsx
import { useTabManager } from "../hooks/useTabManager";
import TabBar from "./TabBar";
import TerminalTab from "./TerminalTab";

export default function TerminalManager() {
  const { tabs, activeId, addTab, closeTab, setActive, renameTab } = useTabManager();

  return (
    <div style={{ display: "flex", flexDirection: "column", height: "100vh", background: "#1e1e1e" }}>
      <TabBar
        tabs={tabs}
        activeId={activeId}
        onAdd={() => addTab()}
        onClose={closeTab}
        onSelect={setActive}
        onRename={renameTab}
      />

      <div style={{ flex: 1, position: "relative", overflow: "hidden" }}>
        {tabs.map((tab) => (
          <div
            key={tab.id}
            style={{
              position: "absolute",
              inset: 0,
              // HIDE, not unmount — preserves PTY session
              display: tab.id === activeId ? "block" : "none",
            }}
          >
            <TerminalTab
              sessionId={tab.id}
              shell={tab.shell}
              isActive={tab.id === activeId}
              onExit={(code) => console.log(`Tab ${tab.id} exited with code ${code}`)}
            />
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## 5. Individual Tab Component {#tab-component}

```tsx
// src/components/TerminalTab.tsx
import { useEffect, useRef } from "react";
import { Terminal } from "@xterm/xterm";
import { FitAddon } from "@xterm/addon-fit";
import { spawn } from "tauri-pty";
import "@xterm/xterm/css/xterm.css";

interface Props {
  sessionId: string;
  shell?: string;
  isActive: boolean;
  onExit?: (code: number) => void;
}

export default function TerminalTab({ sessionId, shell, isActive, onExit }: Props) {
  const containerRef = useRef<HTMLDivElement>(null);
  const fitAddonRef = useRef<FitAddon | null>(null);

  useEffect(() => {
    if (!containerRef.current) return;

    const term = new Terminal({
      cursorBlink: true,
      fontSize: 14,
      theme: { background: "#1e1e1e", foreground: "#d4d4d4" },
    });

    const fit = new FitAddon();
    term.loadAddon(fit);
    term.open(containerRef.current);
    fitAddonRef.current = fit;

    requestAnimationFrame(() => fit.fit());

    const resolvedShell = shell ?? (navigator.userAgent.includes("Windows") ? "powershell.exe" : "bash");
    const pty = spawn(resolvedShell, [], {
      cols: term.cols,
      rows: term.rows,
      env: { TERM: "xterm-256color", COLORTERM: "truecolor" },
    });

    pty.onData((d) => term.write(d));
    pty.onExit(({ exitCode }) => onExit?.(exitCode));
    term.onData((d) => pty.write(d));
    term.onResize(({ cols, rows }) => pty.resize(cols, rows));

    const ro = new ResizeObserver(() => requestAnimationFrame(() => fit.fit()));
    ro.observe(containerRef.current);

    return () => {
      ro.disconnect();
      term.dispose();
    };
  }, [sessionId]); // Re-mount if sessionId changes (i.e., restart session)

  // Re-fit when this tab becomes active (display: none → block)
  useEffect(() => {
    if (isActive) {
      requestAnimationFrame(() => fitAddonRef.current?.fit());
    }
  }, [isActive]);

  return (
    <div ref={containerRef} style={{ width: "100%", height: "100%" }} />
  );
}
```

---

## 6. Tab Bar Component {#tab-bar}

```tsx
// src/components/TabBar.tsx
import { useState } from "react";
import type { TabSession } from "../hooks/useTabManager";

interface Props {
  tabs: TabSession[];
  activeId: string | null;
  onAdd: () => void;
  onClose: (id: string) => void;
  onSelect: (id: string) => void;
  onRename: (id: string, title: string) => void;
}

export default function TabBar({ tabs, activeId, onAdd, onClose, onSelect, onRename }: Props) {
  const [editingId, setEditingId] = useState<string | null>(null);
  const [editValue, setEditValue] = useState("");

  const startEdit = (id: string, currentTitle: string) => {
    setEditingId(id);
    setEditValue(currentTitle);
  };

  const commitEdit = () => {
    if (editingId && editValue.trim()) {
      onRename(editingId, editValue.trim());
    }
    setEditingId(null);
  };

  return (
    <div style={{
      display: "flex",
      alignItems: "center",
      background: "#2d2d2d",
      borderBottom: "1px solid #3a3a3a",
      height: 36,
      overflowX: "auto",
      flexShrink: 0,
    }}>
      {tabs.map((tab) => (
        <div
          key={tab.id}
          onClick={() => onSelect(tab.id)}
          style={{
            display: "flex",
            alignItems: "center",
            gap: 6,
            padding: "0 12px",
            height: "100%",
            cursor: "pointer",
            background: tab.id === activeId ? "#1e1e1e" : "transparent",
            color: tab.id === activeId ? "#d4d4d4" : "#888",
            borderRight: "1px solid #3a3a3a",
            whiteSpace: "nowrap",
            minWidth: 100,
            userSelect: "none",
          }}
        >
          {editingId === tab.id ? (
            <input
              autoFocus
              value={editValue}
              onChange={(e) => setEditValue(e.target.value)}
              onBlur={commitEdit}
              onKeyDown={(e) => {
                if (e.key === "Enter") commitEdit();
                if (e.key === "Escape") setEditingId(null);
                e.stopPropagation(); // Don't send to PTY
              }}
              style={{
                background: "transparent",
                border: "none",
                color: "inherit",
                fontSize: 13,
                outline: "1px solid #569cd6",
                width: 80,
              }}
            />
          ) : (
            <span
              onDoubleClick={() => startEdit(tab.id, tab.title)}
              style={{ fontSize: 13 }}
            >
              {tab.title}
            </span>
          )}

          <button
            onClick={(e) => {
              e.stopPropagation();
              onClose(tab.id);
            }}
            style={{
              background: "transparent",
              border: "none",
              color: "#666",
              cursor: "pointer",
              fontSize: 14,
              lineHeight: 1,
              padding: "0 2px",
              marginLeft: 4,
            }}
          >
            ×
          </button>
        </div>
      ))}

      <button
        onClick={onAdd}
        style={{
          background: "transparent",
          border: "none",
          color: "#888",
          cursor: "pointer",
          fontSize: 18,
          padding: "0 12px",
          height: "100%",
        }}
      >
        +
      </button>
    </div>
  );
}
```

---

## 7. Hiding vs Unmounting Tabs (CRITICAL) {#hide-vs-unmount}

| Behavior | Result |
|---|---|
| `display: none` (CORRECT) | PTY stays alive, shell keeps running, history preserved |
| Unmounting component (WRONG) | PTY is destroyed, shell killed, history lost |
| `visibility: hidden` | Also fine — but xterm fit() may compute wrong dimensions |

**Use `display: none` / `display: block` toggling ONLY. Never conditionally render `<TerminalTab>`
based on which tab is active.**

```tsx
// CORRECT
<div style={{ display: isActive ? "block" : "none" }}>
  <TerminalTab ... />
</div>

// WRONG — unmounts the component when tab is inactive
{isActive && <TerminalTab ... />}
```

---

## 8. Fit on Tab Switch {#fit-on-switch}

When a tab panel transitions from `display: none` → `display: block`, the terminal dimensions
inside it were measured as 0. The FitAddon won't fire automatically since there was no DOM change
— the element was just hidden.

Fix: call `fit.fit()` explicitly whenever `isActive` becomes true:

```tsx
useEffect(() => {
  if (isActive) {
    // Two rAFs: first lets display:none → block commit, second measures correct dimensions
    requestAnimationFrame(() => {
      requestAnimationFrame(() => {
        fitAddonRef.current?.fit();
      });
    });
  }
}, [isActive]);
```

---

## 9. Renaming Tabs {#renaming-tabs}

Double-click on a tab title to rename it. See TabBar component in section 6. Key details:
- `e.stopPropagation()` on the input's `onKeyDown` prevents keys from reaching the PTY
- `onBlur` commits the edit when the input loses focus
- Empty titles are rejected — keep the old title

---

## 10. Drag to Reorder Tabs {#drag-reorder}

```tsx
// Add to TabBar — uses HTML5 drag API (no deps)
const [dragId, setDragId] = useState<string | null>(null);

// On each tab <div>:
draggable
onDragStart={() => setDragId(tab.id)}
onDragOver={(e) => e.preventDefault()}
onDrop={() => {
  if (!dragId || dragId === tab.id) return;
  const from = tabs.findIndex((t) => t.id === dragId);
  const to = tabs.findIndex((t) => t.id === tab.id);
  const reordered = [...tabs];
  const [moved] = reordered.splice(from, 1);
  reordered.splice(to, 0, moved);
  reorderTabs(reordered); // dispatch to your state
  setDragId(null);
}}
onDragEnd={() => setDragId(null)}
```

Add `reorderTabs` action to your store/reducer:
```ts
reorderTabs: (tabs) => set({ tabs })
```

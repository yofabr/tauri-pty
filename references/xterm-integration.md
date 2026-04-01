# xterm.js Integration Reference

## Table of Contents
1. [Addon Overview](#addons)
2. [Themes](#themes)
3. [Font Configuration](#fonts)
4. [Terminal Options Reference](#options)
5. [Resize Handling](#resize)
6. [Search Addon](#search)
7. [WebGL Renderer (Performance)](#webgl)
8. [Custom Key Bindings](#keybindings)
9. [Writing Escape Sequences Directly](#escape-sequences)
10. [Fit Addon: Common Pitfalls](#fit-pitfalls)

---

## 1. Addon Overview {#addons}

| Addon | Package | Purpose |
|---|---|---|
| FitAddon | `@xterm/addon-fit` | Resize terminal to fill container — ESSENTIAL |
| WebLinksAddon | `@xterm/addon-web-links` | Clickable URLs in output |
| SearchAddon | `@xterm/addon-search` | Ctrl+F in-terminal search |
| Unicode11Addon | `@xterm/addon-unicode11` | Better emoji/Unicode width handling |
| ImageAddon | `@xterm/addon-image` | Inline images (Sixel / iTerm2 protocol) |
| WebglAddon | `@xterm/addon-webgl` | GPU-accelerated renderer — use for perf |
| SerializeAddon | `@xterm/addon-serialize` | Serialize/restore terminal buffer |

```tsx
import { FitAddon } from "@xterm/addon-fit";
import { WebLinksAddon } from "@xterm/addon-web-links";
import { SearchAddon } from "@xterm/addon-search";
import { Unicode11Addon } from "@xterm/addon-unicode11";
import { WebglAddon } from "@xterm/addon-webgl";

const fit = new FitAddon();
const links = new WebLinksAddon();
const search = new SearchAddon();
const unicode = new Unicode11Addon();

term.loadAddon(fit);
term.loadAddon(links);
term.loadAddon(search);
term.loadAddon(unicode);
term.unicode.activeVersion = "11";

term.open(containerRef.current!);

// WebGL: must load AFTER open()
try {
  const webgl = new WebglAddon();
  webgl.onContextLoss(() => webgl.dispose()); // Handle context loss
  term.loadAddon(webgl);
} catch {
  // Fall back to canvas renderer silently
}

fit.fit();
```

---

## 2. Themes {#themes}

### VS Code Dark+ (default recommended)
```ts
const vscodeDark = {
  background: "#1e1e1e",
  foreground: "#d4d4d4",
  cursor: "#aeafad",
  cursorAccent: "#1e1e1e",
  selectionBackground: "#264f7870",
  black: "#000000",       brightBlack: "#808080",
  red: "#cd3131",         brightRed: "#f14c4c",
  green: "#0dbc79",       brightGreen: "#23d18b",
  yellow: "#e5e510",      brightYellow: "#f5f543",
  blue: "#2472c8",        brightBlue: "#3b8eea",
  magenta: "#bc3fbc",     brightMagenta: "#d670d6",
  cyan: "#11a8cd",        brightCyan: "#29b8db",
  white: "#e5e5e5",       brightWhite: "#e5e5e5",
};
```

### Dracula
```ts
const dracula = {
  background: "#282a36",
  foreground: "#f8f8f2",
  cursor: "#f8f8f2",
  selectionBackground: "#44475a",
  black: "#21222c",       brightBlack: "#6272a4",
  red: "#ff5555",         brightRed: "#ff6e6e",
  green: "#50fa7b",       brightGreen: "#69ff94",
  yellow: "#f1fa8c",      brightYellow: "#ffffa5",
  blue: "#bd93f9",        brightBlue: "#d6acff",
  magenta: "#ff79c6",     brightMagenta: "#ff92df",
  cyan: "#8be9fd",        brightCyan: "#a4ffff",
  white: "#f8f8f2",       brightWhite: "#ffffff",
};
```

### One Dark
```ts
const oneDark = {
  background: "#282c34",
  foreground: "#abb2bf",
  cursor: "#528bff",
  selectionBackground: "#3e4451",
  black: "#3f4451",       brightBlack: "#4f5666",
  red: "#e06c75",         brightRed: "#be5046",
  green: "#98c379",       brightGreen: "#98c379",
  yellow: "#e5c07b",      brightYellow: "#d19a66",
  blue: "#61afef",        brightBlue: "#61afef",
  magenta: "#c678dd",     brightMagenta: "#c678dd",
  cyan: "#56b6c2",        brightCyan: "#56b6c2",
  white: "#abb2bf",       brightWhite: "#ffffff",
};
```

### Applying a theme at runtime (theme switching)
```tsx
const [currentTheme, setCurrentTheme] = useState(vscodeDark);

// To change theme after initialization:
termRef.current?.options.theme = newTheme;
```

---

## 3. Font Configuration {#fonts}

```tsx
const term = new Terminal({
  fontFamily: '"Cascadia Code", "Fira Code", "JetBrains Mono", Menlo, monospace',
  fontSize: 14,
  lineHeight: 1.2,
  letterSpacing: 0,
  fontWeight: "normal",
  fontWeightBold: "bold",
});
```

### Loading custom fonts in Tauri

In your `index.html` or CSS:

```css
@font-face {
  font-family: "Cascadia Code";
  src: url("/fonts/CascadiaCode.woff2") format("woff2");
  font-weight: normal;
}
```

Place the font file in `src-tauri/` under `assets/` and configure in `tauri.conf.json`:
```json
{
  "bundle": {
    "resources": ["assets/*"]
  }
}
```

**CRITICAL**: xterm.js must be opened AFTER fonts are loaded, or dimensions will be wrong.
Use a font-loading check:
```tsx
await document.fonts.ready; // Wait for custom fonts to load
term.open(containerRef.current!);
fitAddon.fit();
```

---

## 4. Terminal Options Reference {#options}

```tsx
const term = new Terminal({
  // Display
  cols: 80,                          // Initial width (overridden by fit addon)
  rows: 24,                          // Initial height (overridden by fit addon)
  fontSize: 14,
  lineHeight: 1.2,
  theme: vscodeDark,
  
  // Cursor
  cursorBlink: true,
  cursorStyle: "block",              // "block" | "underline" | "bar"
  cursorInactiveStyle: "none",       // Cursor style when terminal loses focus
  
  // Scrolling
  scrollback: 10000,                 // Lines in scrollback buffer
  smoothScrollDuration: 100,         // ms, 0 = instant
  fastScrollModifier: "shift",       // Hold shift for fast scroll
  fastScrollSensitivity: 5,
  
  // Input
  convertEol: true,                  // Convert \n → \r\n (helps on some shells)
  windowsMode: false,                // Set true on Windows for CR/LF handling
  macOptionIsMeta: true,             // Option key as Meta on macOS
  macOptionClickForcesSelection: false,
  
  // Right-click
  rightClickSelectsWord: false,
  
  // Performance
  allowProposedApi: true,            // Required for some addons (Unicode11, Image)
  
  // Accessibility
  screenReaderMode: false,           // Enable for screen reader support
});
```

---

## 5. Resize Handling {#resize}

There are three resize triggers — handle all of them:

```tsx
// 1. Window resize (user resizes the app window)
window.addEventListener("resize", () => fitAddon.fit());

// 2. Container resize (e.g., sidebar expand/collapse changes terminal panel size)
const ro = new ResizeObserver(() => {
  requestAnimationFrame(() => fitAddon.fit()); // rAF prevents layout thrash
});
ro.observe(containerRef.current!);

// 3. Tab visibility change (hidden tab becomes visible — CRITICAL for multi-tab)
// When a tab is shown after being hidden with display:none,
// call fit() manually since ResizeObserver doesn't fire for display:none changes:
useEffect(() => {
  if (isActive) {
    requestAnimationFrame(() => fitAddon.fit());
  }
}, [isActive]);

// Sync PTY size whenever xterm dimensions change
term.onResize(({ cols, rows }) => {
  pty.resize(cols, rows);
});
```

**Why `requestAnimationFrame`?**
`fitAddon.fit()` reads the container's `clientWidth`/`clientHeight`. If called synchronously
during a layout change, the browser may not have committed the new dimensions yet. Wrapping in
`rAF` ensures you read post-layout dimensions.

---

## 6. Search Addon {#search}

```tsx
import { SearchAddon } from "@xterm/addon-search";

const searchAddon = new SearchAddon();
term.loadAddon(searchAddon);

// Find next match
searchAddon.findNext("search term", {
  caseSensitive: false,
  wholeWord: false,
  regex: false,
  incremental: false,       // true = highlight as you type
  decorations: {
    matchBackground: "#ffff0040",
    matchBorder: "#ffff00",
    matchOverviewRuler: "#ffff00",
    activeMatchColorOverviewRuler: "#ff0000",
  },
});

// Find previous
searchAddon.findPrevious("search term");

// Wire to a UI search bar:
const [searchQuery, setSearchQuery] = useState("");
const handleSearch = (e: React.ChangeEvent<HTMLInputElement>) => {
  setSearchQuery(e.target.value);
  searchAddon.findNext(e.target.value, { incremental: true });
};
```

---

## 7. WebGL Renderer (Performance) {#webgl}

The default renderer is Canvas-based. For large outputs or frequent redraws, WebGL is ~3x faster:

```tsx
import { WebglAddon } from "@xterm/addon-webgl";

// MUST be loaded AFTER term.open()
const webgl = new WebglAddon();

// Handle WebGL context loss (GPU driver crash, etc.)
webgl.onContextLoss(() => {
  console.warn("WebGL context lost — falling back to canvas renderer");
  webgl.dispose();
});

try {
  term.loadAddon(webgl);
} catch (e) {
  // WebGL not available (e.g., software rendering) — canvas fallback is automatic
}
```

---

## 8. Custom Key Bindings {#keybindings}

```tsx
term.attachCustomKeyEventHandler((event: KeyboardEvent): boolean => {
  // Return true  → xterm processes the key normally
  // Return false → key is suppressed

  // Ctrl+Shift+C → copy selection
  if (event.type === "keydown" && event.ctrlKey && event.shiftKey && event.key === "C") {
    const sel = term.getSelection();
    if (sel) navigator.clipboard.writeText(sel);
    return false;
  }

  // Ctrl+Shift+V → paste
  if (event.type === "keydown" && event.ctrlKey && event.shiftKey && event.key === "V") {
    navigator.clipboard.readText().then((text) => pty.write(text));
    return false;
  }

  // Ctrl+Shift+F → open search
  if (event.type === "keydown" && event.ctrlKey && event.shiftKey && event.key === "F") {
    setSearchVisible((v) => !v);
    return false;
  }

  // Ctrl++ / Ctrl+- → font size
  if (event.ctrlKey && event.key === "=") {
    term.options.fontSize = (term.options.fontSize ?? 14) + 1;
    fitAddon.fit();
    return false;
  }
  if (event.ctrlKey && event.key === "-") {
    term.options.fontSize = Math.max(8, (term.options.fontSize ?? 14) - 1);
    fitAddon.fit();
    return false;
  }

  return true;
});
```

---

## 9. Writing Escape Sequences Directly {#escape-sequences}

Useful for status messages, prompts, or programmatic terminal output:

```tsx
// Colors
term.write("\x1b[32mGreen text\x1b[0m");
term.write("\x1b[1;31mBold red text\x1b[0m");
term.write("\x1b[48;5;236m\x1b[38;5;250m Grey background \x1b[0m");

// Cursor movement
term.write("\x1b[2J\x1b[H");           // Clear screen + move to top-left
term.write("\x1b[A");                   // Move cursor up one line
term.write("\x1b[2K");                  // Clear entire current line

// Hyperlinks (OSC 8)
term.write("\x1b]8;;https://example.com\x07Click here\x1b]8;;\x07");

// Bell
term.write("\x07");

// Newline (always use \r\n in PTY context, not just \n)
term.write("Hello world\r\n");
```

---

## 10. Fit Addon: Common Pitfalls {#fit-pitfalls}

| Pitfall | Fix |
|---|---|
| `fitAddon.fit()` called before `term.open()` | Always call `open()` first, then `fit()` |
| Container has `height: 0` at mount time | Use `requestAnimationFrame(() => fit.fit())` |
| Terminal wrong size after tab switch | Call `fit.fit()` whenever tab becomes visible |
| Font not loaded when `fit()` runs | `await document.fonts.ready` before opening |
| Terminal overflows container | Ensure container has explicit `height` in CSS, not just `flex: 1` with no parent height |
| `fit()` called too many times in rapid succession | Debounce the ResizeObserver callback if needed |

**Container CSS rule — this must be set or fit will always produce wrong dimensions:**
```css
.terminal-container {
  width: 100%;
  height: 100%;        /* explicit height required */
  overflow: hidden;    /* prevents scrollbar affecting measurements */
}
```

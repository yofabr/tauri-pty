# Best Practices, Performance & Common Gotchas

## Table of Contents
1. [The Non-Negotiable Rules](#non-negotiables)
2. [Performance Optimization](#performance)
3. [Memory Management & Cleanup](#memory)
4. [Cross-Platform Quirks](#cross-platform)
5. [Security Considerations](#security)
6. [Common Bugs and Fixes](#common-bugs)
7. [Debugging PTY Issues](#debugging)
8. [Production Checklist](#checklist)
9. [Tauri Events vs Channels — When to Switch](#events-vs-channels)
10. [Session Persistence & Restore](#session-persistence)

---

## 1. The Non-Negotiable Rules {#non-negotiables}

These will save hours of debugging. Follow all of them.

| Rule | Why |
|---|---|
| Always set `TERM=xterm-256color` | Without it: backspace broken, colors wrong, `clear` fails, vim/htop broken |
| Always `drop(pair.slave)` after spawning (Option B) | Leaking the slave fd blocks PTY from signaling EOF properly |
| PTY reads are **blocking** — use `std::thread::spawn`, not `tokio::spawn` | `tokio::spawn` is for async futures; blocking PTY reads will deadlock or starve the Tokio runtime |
| Hide inactive tabs with `display: none`, never unmount | Unmounting destroys the PTY session permanently |
| Call `fitAddon.fit()` inside `requestAnimationFrame` | Direct calls read stale dimensions before browser commits layout |
| Load `WebglAddon` AFTER `term.open()` | WebGL needs a mounted DOM canvas to initialize |
| `await document.fonts.ready` before opening xterm | Incorrect font metrics cause wrong column/row calculations |
| Use `\r\n` not `\n` when writing to PTY | PTY is in raw mode; `\n` alone won't move cursor to column 0 |

---

## 2. Performance Optimization {#performance}

### Use WebGL renderer
```tsx
// 3x faster than canvas for high-throughput output (log tailing, builds, etc.)
const webgl = new WebglAddon();
webgl.onContextLoss(() => webgl.dispose());
term.loadAddon(webgl); // After term.open()
```

### Batch writes with a write queue
When you're receiving very rapid PTY output (e.g., `cat bigfile.log`), batching writes reduces
layout thrashing:

```tsx
let writeBuffer = "";
let writeTimer: ReturnType<typeof setTimeout> | null = null;

pty.onData((data) => {
  writeBuffer += data;
  if (!writeTimer) {
    writeTimer = setTimeout(() => {
      term.write(writeBuffer);
      writeBuffer = "";
      writeTimer = null;
    }, 16); // ~1 frame at 60fps
  }
});
```

### Limit scrollback buffer
```tsx
const term = new Terminal({
  scrollback: 5000, // Default is 1000; more = more RAM. Don't set to Infinity.
});
```

### Debounce ResizeObserver
```tsx
let resizeTimeout: ReturnType<typeof setTimeout>;
const ro = new ResizeObserver(() => {
  clearTimeout(resizeTimeout);
  resizeTimeout = setTimeout(() => fitAddon.fit(), 50);
});
```

### Tauri Events vs Channels for high throughput
See section 9 for detail. For output > ~1MB/sec, switch to Tauri Channels.

---

## 3. Memory Management & Cleanup {#memory}

Always clean up on unmount. This is your full cleanup checklist:

```tsx
return () => {
  // 1. Stop all listeners
  resizeObserver.disconnect();
  window.removeEventListener("resize", handleResize);

  // 2. Remove Tauri event listeners (Option B only)
  unlistenOutput();
  unlistenExit();

  // 3. Kill the PTY session (Option B only)
  invoke("kill_pty", { sessionId });

  // 4. Dispose xterm (frees canvas, event listeners, WebGL context)
  term.dispose();

  // 5. Clear refs
  termRef.current = null;
  fitAddonRef.current = null;
};
```

**Memory leak signs:**
- RAM grows as new tabs are opened and old ones "closed"
- `term.dispose()` not being called (often caused by double-mounting in React StrictMode)

**React StrictMode double-mount fix:**
In development, React StrictMode mounts → unmounts → remounts components. This can spawn two PTY
sessions. Use a ref guard:

```tsx
const mountedRef = useRef(false);

useEffect(() => {
  if (mountedRef.current) return; // Prevent double-mount in StrictMode
  mountedRef.current = true;

  // ... spawn PTY
}, []);
```

---

## 4. Cross-Platform Quirks {#cross-platform}

### macOS
- Default shell is **zsh** since Catalina (10.15). Never hardcode `/bin/bash`.
- Read `$SHELL` env var: `std::env::var("SHELL").unwrap_or("/bin/zsh")`
- **macOS Sequoia+**: Tauri apps may need `NSAppleEventsUsageDescription` in Info.plist for shell access
- Option key: set `macOptionIsMeta: true` in xterm options for proper Meta key behavior

### Windows
- Use `powershell.exe` as fallback; prefer `pwsh` (PowerShell 7+) if available
- Set `windowsMode: true` in xterm for proper CRLF line ending handling
- Windows ConPTY is used by `portable_pty` automatically — no special config needed
- `TERM` env var isn't used the same way; still set it for compatibility but don't rely on it
- Some Windows terminals need `cmd.env("PSModulePath", "")` to avoid PowerShell module path issues

### Linux
- No special handling needed; `portable_pty` uses `openpty(3)` directly
- `$SHELL` env var is the right way to detect user's shell

---

## 5. Security Considerations {#security}

### Never pass unsanitized user input directly as shell commands
```rust
// NEVER do this with user-supplied data:
cmd.args(&["--rcfile", &user_provided_path]); // Path traversal risk

// Instead: validate paths, use allowlists for shells
let allowed_shells = ["/bin/bash", "/bin/zsh", "/usr/bin/fish"];
if !allowed_shells.contains(&shell.as_str()) {
    return Err("Shell not in allowlist".to_string());
}
```

### Tauri CSP
In `tauri.conf.json`, ensure your CSP doesn't block xterm's WebGL canvas or font loading:
```json
{
  "security": {
    "csp": "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; font-src 'self' data:; img-src 'self' data: blob:"
  }
}
```

### Don't expose shell commands as Tauri commands callable from JS without auth
If your app has multiple windows or loads external content, wrap PTY commands with session
validation to prevent unauthorized PTY spawning.

---

## 6. Common Bugs and Fixes {#common-bugs}

### Bug: Backspace doesn't work
**Cause**: `TERM` env var not set, or wrong stty settings.
**Fix**: Set `TERM=xterm-256color` when spawning. Also ensure `convertEol: true` in xterm options.

### Bug: Terminal is the wrong size / text wraps incorrectly
**Cause**: `fitAddon.fit()` called before DOM dimensions are available, or custom font not loaded.
**Fix**:
```tsx
await document.fonts.ready;
term.open(containerRef.current!);
requestAnimationFrame(() => fitAddon.fit());
```

### Bug: Colors don't work in vim/htop/etc.
**Cause**: `TERM` or `COLORTERM` not set.
**Fix**: Set `TERM=xterm-256color` and `COLORTERM=truecolor`.

### Bug: Arrow keys produce `^[[A` etc. instead of navigating
**Cause**: Shell's readline isn't initialized properly, or `TERM` is wrong.
**Fix**: Set `TERM=xterm-256color`. Also confirm the shell is interactive: pass `-i` flag if needed.

### Bug: Terminal shows garbled output after resize
**Cause**: PTY dimensions not synced to xterm dimensions.
**Fix**: Ensure `term.onResize(({ cols, rows }) => pty.resize(cols, rows))` is wired.

### Bug: React StrictMode spawns two PTY sessions
**Cause**: useEffect runs twice in development StrictMode.
**Fix**: Use a `mountedRef` guard (see section 3).

### Bug: Tab switch leaves terminal wrong size
**Cause**: `fit.fit()` not called when tab becomes visible.
**Fix**: Call `fit.fit()` inside a double-`requestAnimationFrame` when `isActive` becomes `true`.

### Bug: Ctrl+C doesn't work in the terminal
**Cause**: Browser intercepts the event before xterm.
**Fix**: Use `attachCustomKeyEventHandler` to pass Ctrl+C through to xterm (return `true`).

### Bug: Tauri window closes but PTY process keeps running
**Cause**: PTY child process not killed on window close.
**Fix**: Listen to Tauri's `window-destroyed` event and kill all PTY sessions.

```rust
// In lib.rs, add window close handler:
tauri::Builder::default()
    .on_window_event(|window, event| {
        if let tauri::WindowEvent::Destroyed = event {
            // Drop PtyManager state here
        }
    })
```

---

## 7. Debugging PTY Issues {#debugging}

### Check if the PTY process is actually running
```rust
// After spawn, log the child PID:
let child = pair.slave.spawn_command(cmd)?;
println!("Spawned child PID: {:?}", child.process_id());
```

### Log all PTY output to a file (development only)
```rust
std::thread::spawn(move || {
    let mut reader = pair.master.try_clone_reader().unwrap();
    let mut log = std::fs::File::create("/tmp/pty-debug.log").unwrap();
    let mut buf = [0u8; 4096];
    loop {
        match reader.read(&mut buf) {
            Ok(n) if n > 0 => {
                log.write_all(&buf[..n]).ok();
                // Also emit to frontend...
            }
            _ => break,
        }
    }
});
```

### Check terminal dimensions
```tsx
console.log(`Terminal: ${term.cols}x${term.rows}`);
console.log(`Container: ${containerRef.current?.clientWidth}x${containerRef.current?.clientHeight}`);
```

### Use strace/dtrace to inspect PTY fd activity (Linux/macOS)
```bash
strace -e trace=read,write -p <pid>
```

---

## 8. Production Checklist {#checklist}

Before shipping:

- [ ] `TERM=xterm-256color` set on PTY spawn
- [ ] `COLORTERM=truecolor` set on PTY spawn
- [ ] `LANG=en_US.UTF-8` set on PTY spawn
- [ ] Shell auto-detected per platform (zsh macOS, $SHELL Linux, pwsh Windows)
- [ ] `drop(pair.slave)` called after spawn (Option B)
- [ ] `term.dispose()` called in useEffect cleanup
- [ ] ResizeObserver disconnected in cleanup
- [ ] Tauri event listeners unlistened in cleanup (Option B)
- [ ] PTY killed on window close
- [ ] Inactive tabs use `display: none` (not unmounted)
- [ ] `fit.fit()` called on tab switch
- [ ] `await document.fonts.ready` before xterm open (if using custom fonts)
- [ ] React StrictMode double-mount guard in place
- [ ] WebGL context loss handler attached
- [ ] Scrollback limit set (not infinite)
- [ ] Custom key handler passes Ctrl+C through to xterm
- [ ] Tauri CSP allows xterm WebGL/canvas

---

## 9. Tauri Events vs Channels {#events-vs-channels}

**Tauri Events** (`app.emit()` → `listen()`) are fine for typical terminal output.

Switch to **Tauri Channels** when:
- Output throughput is very high (building large projects, log tailing, ffmpeg, etc.)
- You notice dropped output or UI lag on fast output

```rust
// Tauri Channel approach (Tauri 2 unstable feature)
use tauri::ipc::Channel;

#[tauri::command]
async fn start_pty_channel(on_data: Channel<String>) -> Result<(), String> {
    // ...spawn pty...
    std::thread::spawn(move || {
        let mut buf = [0u8; 4096];
        loop {
            match reader.read(&mut buf) {
                Ok(n) if n > 0 => {
                    let data = String::from_utf8_lossy(&buf[..n]).to_string();
                    on_data.send(data).ok();
                }
                _ => break,
            }
        }
    });
    Ok(())
}
```

```tsx
// Frontend
import { Channel } from "@tauri-apps/api/core";

const channel = new Channel<string>();
channel.onmessage = (data) => term.write(data);
await invoke("start_pty_channel", { onData: channel });
```

---

## 10. Session Persistence & Restore {#session-persistence}

The PTY process itself cannot be paused/restored across app restarts (the OS process dies when
the app closes). What you CAN persist:

### Save terminal output to a scrollback buffer
```tsx
import { SerializeAddon } from "@xterm/addon-serialize";

const serialize = new SerializeAddon();
term.loadAddon(serialize);

// Before closing: save buffer
const snapshot = serialize.serialize();
localStorage.setItem(`terminal-snapshot-${sessionId}`, snapshot);

// On open: restore (display only, not live PTY output)
const snapshot = localStorage.getItem(`terminal-snapshot-${sessionId}`);
if (snapshot) term.write(snapshot);
```

### Save the working directory
On macOS/Linux, read `/proc/{pid}/cwd` or use `lsof` to get the shell's current directory,
then pass it as `cwd` when re-spawning:

```rust
// Get cwd of child process
#[cfg(unix)]
fn get_process_cwd(pid: u32) -> Option<String> {
    std::fs::read_link(format!("/proc/{}/cwd", pid))
        .ok()
        .map(|p| p.to_string_lossy().to_string())
}
```

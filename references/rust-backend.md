# Rust Backend Reference

## Table of Contents
1. [Cargo.toml Setup](#cargo-setup)
2. [Plugin Registration (tauri-plugin-pty)](#plugin-registration)
3. [Manual portable_pty (Option B)](#manual-portablerty)
4. [State Management for Multiple PTYs](#state-management)
5. [Custom Tauri Commands](#custom-commands)
6. [Cross-Platform Shell Detection in Rust](#shell-detection)
7. [Environment Variables](#environment-variables)
8. [Tauri Permissions (v2)](#tauri-permissions)

---

## 1. Cargo.toml Setup {#cargo-setup}

### Option A — tauri-plugin-pty (recommended)
```toml
[package]
name = "your-app"
version = "0.1.0"
edition = "2021"

[dependencies]
tauri = { version = "2", features = [] }
tauri-plugin-pty = "0.1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"

[build-dependencies]
tauri-build = { version = "2", features = [] }
```

### Option B — portable_pty manually
```toml
[dependencies]
tauri = { version = "2", features = ["unstable"] }
portable-pty = "0.9"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
anyhow = "1"
```

---

## 2. Plugin Registration (tauri-plugin-pty) {#plugin-registration}

This is all you need on the Rust side for Option A:

```rust
// src-tauri/src/lib.rs
pub fn run() {
    tauri::Builder::default()
        .plugin(tauri_plugin_pty::init())
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

```rust
// src-tauri/src/main.rs
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]

fn main() {
    your_app_lib::run();
}
```

The plugin handles PTY creation, output streaming, input writing, and resizing entirely through
the JS/TS `tauri-pty` npm package. No additional Rust code is needed.

---

## 3. Manual portable_pty (Option B) {#manual-portablerty}

Use this when you need:
- Programmatic PTY lifecycle control (kill, restart, pause)
- PTY output logging/recording
- Shared PTY state across multiple windows
- Custom shell environment injection at spawn time

### Full implementation:

```rust
// src-tauri/src/pty_manager.rs
use portable_pty::{CommandBuilder, PtySize, native_pty_system, PtySystem, MasterPty};
use std::collections::HashMap;
use std::io::{Read, Write};
use std::sync::{Arc, Mutex};
use tauri::{AppHandle, Emitter};

pub struct PtySession {
    pub writer: Arc<Mutex<Box<dyn Write + Send>>>,
    pub master: Arc<Mutex<Box<dyn MasterPty + Send>>>,
}

pub struct PtyManager {
    pub sessions: Mutex<HashMap<String, PtySession>>,
}

impl PtyManager {
    pub fn new() -> Self {
        PtyManager {
            sessions: Mutex::new(HashMap::new()),
        }
    }

    pub fn spawn(
        &self,
        session_id: String,
        shell: &str,
        cols: u16,
        rows: u16,
        app: AppHandle,
    ) -> Result<(), String> {
        let pty_system = native_pty_system();

        let pair = pty_system
            .openpty(PtySize {
                rows,
                cols,
                pixel_width: 0,
                pixel_height: 0,
            })
            .map_err(|e| e.to_string())?;

        let mut cmd = CommandBuilder::new(shell);
        cmd.env("TERM", "xterm-256color");
        cmd.env("COLORTERM", "truecolor");
        cmd.env("LANG", "en_US.UTF-8");

        pair.slave
            .spawn_command(cmd)
            .map_err(|e| e.to_string())?;

        // Release slave handle after spawn — IMPORTANT
        drop(pair.slave);

        let writer = Arc::new(Mutex::new(
            pair.master.take_writer().map_err(|e| e.to_string())?,
        ));
        let master = Arc::new(Mutex::new(pair.master));

        // Store session
        {
            let mut sessions = self.sessions.lock().unwrap();
            sessions.insert(
                session_id.clone(),
                PtySession {
                    writer: writer.clone(),
                    master: master.clone(),
                },
            );
        }

        // Spawn a DEDICATED OS THREAD for reading — PTY reads are blocking
        // Do NOT use tokio::spawn here — it will deadlock or starve the runtime
        let sid = session_id.clone();
        let app_handle = app.clone();
        std::thread::spawn(move || {
            let mut reader = {
                let m = master.lock().unwrap();
                m.try_clone_reader().expect("failed to clone reader")
            };

            let mut buf = [0u8; 4096];
            loop {
                match reader.read(&mut buf) {
                    Ok(0) => break, // EOF — shell exited
                    Ok(n) => {
                        let data = String::from_utf8_lossy(&buf[..n]).to_string();
                        // Emit output to frontend — event name is "pty-output:{session_id}"
                        app_handle
                            .emit(&format!("pty-output:{}", sid), data)
                            .ok();
                    }
                    Err(_) => break,
                }
            }
            // Notify frontend that session ended
            app_handle
                .emit(&format!("pty-exit:{}", sid), ())
                .ok();
        });

        Ok(())
    }

    pub fn write(&self, session_id: &str, data: &str) -> Result<(), String> {
        let sessions = self.sessions.lock().unwrap();
        if let Some(session) = sessions.get(session_id) {
            let mut writer = session.writer.lock().unwrap();
            writer
                .write_all(data.as_bytes())
                .map_err(|e| e.to_string())?;
        }
        Ok(())
    }

    pub fn resize(&self, session_id: &str, cols: u16, rows: u16) -> Result<(), String> {
        let sessions = self.sessions.lock().unwrap();
        if let Some(session) = sessions.get(session_id) {
            let master = session.master.lock().unwrap();
            master
                .resize(PtySize {
                    rows,
                    cols,
                    pixel_width: 0,
                    pixel_height: 0,
                })
                .map_err(|e| e.to_string())?;
        }
        Ok(())
    }

    pub fn kill(&self, session_id: &str) {
        let mut sessions = self.sessions.lock().unwrap();
        sessions.remove(session_id); // Drop session, which closes master, which kills child
    }
}
```

### Tauri Commands for Option B:

```rust
// src-tauri/src/lib.rs
mod pty_manager;
use pty_manager::PtyManager;
use tauri::State;

#[tauri::command]
async fn spawn_pty(
    session_id: String,
    shell: Option<String>,
    cols: u16,
    rows: u16,
    app: tauri::AppHandle,
    manager: State<'_, PtyManager>,
) -> Result<(), String> {
    let detected_shell = shell.unwrap_or_else(detect_shell);
    manager.spawn(session_id, &detected_shell, cols, rows, app)
}

#[tauri::command]
fn write_pty(session_id: String, data: String, manager: State<'_, PtyManager>) -> Result<(), String> {
    manager.write(&session_id, &data)
}

#[tauri::command]
fn resize_pty(session_id: String, cols: u16, rows: u16, manager: State<'_, PtyManager>) -> Result<(), String> {
    manager.resize(&session_id, cols, rows)
}

#[tauri::command]
fn kill_pty(session_id: String, manager: State<'_, PtyManager>) {
    manager.kill(&session_id)
}

fn detect_shell() -> String {
    if cfg!(target_os = "windows") {
        // Prefer PowerShell 7+ (pwsh), fall back to powershell.exe
        if std::process::Command::new("pwsh").arg("--version").output().is_ok() {
            "pwsh".to_string()
        } else {
            "powershell.exe".to_string()
        }
    } else if cfg!(target_os = "macos") {
        // Respect user's shell from env, fall back to zsh (macOS default)
        std::env::var("SHELL").unwrap_or_else(|_| "/bin/zsh".to_string())
    } else {
        // Linux — respect $SHELL, fall back to bash
        std::env::var("SHELL").unwrap_or_else(|_| "/bin/bash".to_string())
    }
}

pub fn run() {
    tauri::Builder::default()
        .manage(PtyManager::new())
        .invoke_handler(tauri::generate_handler![
            spawn_pty,
            write_pty,
            resize_pty,
            kill_pty,
        ])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

---

## 4. State Management for Multiple PTYs {#state-management}

When managing multiple PTY sessions (tabs), use a `HashMap<String, PtySession>` keyed by a
`session_id` (e.g., UUID). Each session owns its writer and master handle independently.

Key rules:
- Generate `session_id` on the frontend with `crypto.randomUUID()`
- Pass it into every `spawn_pty`, `write_pty`, `resize_pty`, `kill_pty` call
- Use per-session event names: `pty-output:{session_id}` to avoid cross-talk between tabs

---

## 5. Cross-Platform Shell Detection in Rust {#shell-detection}

See `detect_shell()` in section 3 above. Always:
- On macOS/Linux: read `$SHELL` environment variable first
- On Windows: probe for `pwsh` (PowerShell 7+) before falling back to `powershell.exe`
- Never hardcode `/bin/bash` on macOS — it ships bash 3.x; zsh is the default since Catalina

---

## 6. Environment Variables {#environment-variables}

Always inject these when spawning a PTY:

```rust
cmd.env("TERM", "xterm-256color");      // Enables colors, backspace, arrow keys
cmd.env("COLORTERM", "truecolor");       // Tells apps (neovim, etc.) true 24-bit color is available
cmd.env("LANG", "en_US.UTF-8");          // Correct character encoding
cmd.env("LC_ALL", "en_US.UTF-8");        // Ensures UTF-8 everywhere
```

Without `TERM=xterm-256color`:
- Backspace may not work
- `clear` may fail
- Colors will be wrong in vim, tmux, etc.

---

## 7. Tauri Permissions (v2) {#tauri-permissions}

For `tauri-plugin-pty`, add to `src-tauri/capabilities/default.json`:

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "default",
  "description": "Default capabilities",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "pty:default"
  ]
}
```

For manual PTY with shell spawning, no additional capabilities are needed beyond `core:default`,
since you're using Rust-native OS APIs, not Tauri's shell plugin.

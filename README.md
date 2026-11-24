# Antigravity in Sandboxie: Issues & Root Cause Analysis

Running Google Antigravity inside a Sandboxie-Plus container encounters three distinct subsystem failures: CLI subprocess execution failure, Electron GUI startup failure, and OAuth login conflict.

---

## 1. CLI Issue: `CreateDesktop: Access is denied`

### Problem
When `agy.exe` executes commands (`run_command`, terminal actions, bash/powershell), every command immediately fails with:
```text
Command failed to run: CreateDesktop: Access is denied..
```

### What is Happening Under the Hood
1. **Hidden Desktop Allocation:** On Windows, `agy.exe` creates a dedicated, hidden Win32 Desktop (`CreateDesktopW`) to run console sub-processes silently so that command prompt windows do not flash or steal focus from the user.
2. **Window Station Isolation:** Sandboxie isolates the interactive Window Station (`WinSta0`) and desktops by default (`OpenWndStation=n`), blocking processes from creating or opening desktops.
3. **Token Stripping:** By default, Sandboxie converts elevated administrator tokens into restricted tokens (`DropAdminRights=y`), which lack the `WINSTA_CREATEDESKTOP` permission on `WinSta0`.
4. **Console Proxy Conflict:** Sandboxie's helper service (`SbieSvc`) attempts to detach the process from the existing console and attach it to an internal proxy console, conflicting with ConPTY.

### The Fix
```ini
NoSandboxieConsole=y
OpenWndStation=y
UseSbieDeskHack=y
UseSbieWndStation=y
OpenWinClass=*
NoSecurityIsolation=y
DropAdminRights=n
```
* Bypasses the `SbieSvc` console interception and allows native Windows console handling.
* Allows creation of desktop objects on `WinSta0` via `OpenWndStation=y` and `UseSbieDeskHack=y`.
* Switches the sandbox to **Application Compartment mode** (`NoSecurityIsolation=y`), preserving token integrity while maintaining 100% filesystem and registry virtualization.

---

## 2. GUI Issue: Antigravity IDE Closes Immediately or Crashes

### Problem
Launching `Antigravity.exe` inside Sandboxie results in:
* Silent exit with code `0` within one second, OR
* Renderer processes terminate with no user interface visible.

### What is Happening Under the Hood
1. **Single-Instance Lock Collision:**
   * Antigravity GUI is an Electron application. On startup, it calls `app.requestSingleInstanceLock()`, creating an exclusive lock on `%APPDATA%\Antigravity\lockfile` and opening local named pipes (`\\.\pipe\LOCAL\mojo.*`).
   * When Antigravity is already running on the host OS, the sandboxed instance detects the host instance through shared pipes/paths, delegates focus to the host window, and cleanly exits (`code 0`).
2. **Chromium Sandbox Collision:**
   * Chromium's internal renderer sandbox (`--enable-sandbox`) attempts to create nested Windows Job Objects and token filters inside Sandboxie. Windows rejects nested Job restrictions, causing renderer processes to crash.

### The Fix
```ini
SpecialImage=chrome,Antigravity.exe
CustomChromiumFlags=--no-sandbox
```
* `SpecialImage=chrome,Antigravity.exe` tells Sandboxie's driver to treat `Antigravity.exe` as a Chromium/Electron executable, enabling Chromium-specific IPC and handle-sharing hooks.
* `CustomChromiumFlags=--no-sandbox` automatically injects the `--no-sandbox` flag into main processes, preventing the internal Chromium sandbox from clashing with Sandboxie.
* *Note on Parallel Execution:* To run the sandboxed GUI simultaneously with the host GUI, start the sandboxed GUI with an isolated profile: `--user-data-dir="%APPDATA%\Antigravity-sbx"`.

---

## 3. Auth Issue: Browser Conflict During OAuth Login

### Problem
Clicking "Sign In" inside Antigravity GUI triggers an OAuth web flow in the default browser (`chrome.exe`), which fails to open or opens with an unauthenticated profile.

### What is Happening Under the Hood
* When Antigravity opens the authentication URL, Sandboxie tries to keep `chrome.exe` confined inside the sandbox.
* Inside the sandbox, `chrome.exe` has an empty/isolated profile, lacking the user's active Google account session from the host.

### The Fix
```ini
BreakoutProcess=chrome.exe
```
* Instructs Sandboxie to release `chrome.exe` onto the host OS when invoked from the sandbox.
* The login link opens in the user's host browser with their active Google account, completes OAuth, and redirects the callback to the local loopback server.

---

## Complete Minimal Sandbox Section

Add this block directly under your target box in `C:\Windows\Sandboxie.ini`:

```ini
[test1]
NoSandboxieConsole=y
OpenWndStation=y
UseSbieDeskHack=y
UseSbieWndStation=y
OpenWinClass=*
NoSecurityIsolation=y
DropAdminRights=n
SpecialImage=chrome,Antigravity.exe
CustomChromiumFlags=--no-sandbox
BreakoutProcess=chrome.exe
```

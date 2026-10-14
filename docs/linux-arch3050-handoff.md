# Linux / arch3050 investigation handoff

Updated: 2026-10-14
Target: QuotaBar v0.6.0 AppImage on `arch3050` (Arch Linux, Hyprland, Caelestia Shell / Quickshell)

This is a source-based handoff for the next real-machine investigation. The
observations in the real-machine section were reported by the user; Cloud did
not launch the AppImage or reproduce them. No credentials were inspected.

## Checkout and investigation state

- Fork: `takashi1211/quotabar`; upstream: `majiayu000/quotabar`.
- At the time of this handoff, local branch `work` was clean at
  `18ad73d7a05458353149d4f9ed66d18a6c6d402c`, matching `origin/main`.
- The earlier repository survey changed no tracked files. This handoff is the
  only file added for the current save operation.
- The local branch listing and `git ls-remote` showed only `main` on `origin`
  before this handoff branch was created.
- GitHub pull-request / issue API access returned HTTP 403 in this environment.
  Therefore open PR state and a complete live issue search remain unverified.
  No PR was created by this work.

## Real-machine findings reported by the user

These are user observations from the official v0.6.0 Linux AppImage on
`arch3050`, not Cloud test results:

- The AppImage started.
- Codex quota retrieval succeeded and HTTP 200 was observed.
- The GUI flickered and went black intermittently.
- Setting `WEBKIT_DISABLE_COMPOSITING_MODE=1` did not fix the display problem.
- Caelestia's bar displayed duplicate QuotaBar icons.
- Clicking a bar icon did not open the GUI.
- Processes named `quotabar` and after the AppImage were observed. Those two
  observations alone do not establish that two app instances were running.

All six symptoms remain open for real-machine diagnosis. The successful Codex
request establishes that the AppImage could read a working Codex login and
reach the quota endpoint during that run; it does not explain the window or
tray symptoms.

## Architecture and provider behavior

QuotaBar is a Tauri 2 desktop application. Rust owns the native process,
provider access, quota and cost data, tray icons, window lifecycle, and Tauri
commands. React 19 and TypeScript build the interface with Vite. The frontend
uses Tauri IPC to invoke Rust commands. On Linux, Tauri's Wry webview uses
WebKitGTK; the lockfile includes `webkit2gtk` through `tauri-runtime-wry`.
Useful entry points are [`src-tauri/src/lib.rs`](../src-tauri/src/lib.rs#L33),
[`src-tauri/src/commands`](../src-tauri/src/commands),
[`src-tauri/src/services`](../src-tauri/src/services),
[`src/main.tsx`](../src/main.tsx#L1), and [`src/App.tsx`](../src/App.tsx#L379).

The five provider tray identities in the current source are Claude Code,
Codex, Cursor, Grok Build, and Antigravity. Settings and connection state
decide which icons are shown; tray cycling can hide some while retaining them
in the app. Antigravity is hidden by default until its quota can be tracked
(see `src-tauri/src/services/tray.rs` and `src/services/tray_visibility.ts`).
Several visible provider icons can therefore be intentional. A second app
instance is another possibility, but the user report does not distinguish
these cases.

### Codex authentication and quota request

`src-tauri/src/services/codex.rs` resolves `CODEX_HOME` when set, otherwise
`~/.codex`, and reads `auth.json`. The quota function reads
`tokens.access_token`, obtains the ChatGPT account ID from the ID token when
available, then sends an authenticated request to
`https://chatgpt.com/backend-api/wham/usage`. It logs the HTTP status and
latency without logging the access token. The user's HTTP 200 report is
consistent with this request path succeeding on that run. This investigation
did not read the real machine's `auth.json` or any token.

Other Linux-relevant authentication paths need provider-by-provider
verification. Claude's non-macOS/non-Windows system credential reader returns
an unsupported-OS error and asks for `CLAUDE_CODE_OAUTH_TOKEN`; Cursor looks
under the platform config directory at `Cursor/User/globalStorage/state.vscdb`;
Grok resolves `GROK_HOME` or `~/.grok`. Do not assume every provider's login
path is supported because Codex worked.

## Linux app, tray, windows, and background lifecycle

`src-tauri/tauri.conf.json` declares two initially hidden webview windows:

- `main`: transparent, undecorated, always-on-top, skipped from the taskbar;
  this is the quota popover/tray frontend (`index.html?window=tray`).
- `analysis`: normal decorated, resizable workspace window.

`src-tauri/src/lib.rs::run` initializes the Tauri plugins, registers commands,
then calls `setup_tray` and `setup_workspace`. Hiding or closing either window
does not quit the process: the tray popover hides on focus loss/close, and the
analysis window close is intercepted and converted to hide. The tray menu's
Quit item exits the process. This is an in-process background tray app; the
source does not define a separate Linux daemon or systemd service. The
autostart plugin is initialized, but its real Linux behavior has not been
tested here.

`TrayService` gives each provider its own tray ID. The frontend calls
`update_tray_icon`; Rust creates or updates the matching native item. The menu
has “Show / Hide Window”, “Open QuotaBar”, and “Quit”. The first action toggles
the `main` popover; “Open QuotaBar” calls `window::show_workspace` for the
`analysis` window. The Rust click callback also tries to show/focus `main`.
The implementation is in
[`src-tauri/src/services/tray.rs`](../src-tauri/src/services/tray.rs#L505)
and [`src-tauri/src/services/window.rs`](../src-tauri/src/services/window.rs#L32).

There is a significant Linux limitation in the pinned tray dependency:
`Cargo.lock` resolves `tray-icon` 0.21.3. Its Linux GTK/AppIndicator backend
uses `libappindicator`; its `rect()` returns no rectangle, and its public API
documents Linux tray-rectangle and `set_show_menu_on_left_click` as
unsupported. In the inspected GTK backend, the icon is registered with an
AppIndicator and a GTK menu, while no pointer-click callback is wired there.
This is strong source evidence that the app's `TrayIconEvent::Click` callback
may not be delivered on Linux. Confirm on the target machine. Without a tray
rectangle, `position_window_near_tray` returns without positioning the
popover. These constraints can affect click-to-open and placement independently
of whether Quickshell displays the item.

The Linux tray uses the AppIndicator/StatusNotifier integration path rather
than a macOS status item or Windows notification-area implementation. It
depends on the desktop session's D-Bus tray host. Hyprland and Quickshell
details, item registration, click/menu semantics, and duplicate registration
have not been inspected on `arch3050`. The bar icon appearing is evidence from
the user that some tray integration was active, not proof that every protocol
event works.

## Build and distribution facts

`tauri.conf.json` lists AppImage, DMG, MSI, and NSIS bundle targets. The release
workflow has a Linux x64 job on Ubuntu 22.04 producing an AppImage and checksum.
It installs `libwebkit2gtk-4.1-dev`, GTK 3, Ayatana AppIndicator, librsvg,
OpenSSL, build tools, `patchelf`, `libfuse2`, and related tools. The exact
Arch package mapping and runtime library requirements have not been verified
on the target PC. The AppImage's user-reported successful launch is a machine
observation; this Cloud review did not build or run it.

The regular CI workflow's OS matrix is macOS and Windows only. Linux is
therefore part of the release artifact matrix but is not currently a regular
CI test platform. Existing Linux release packaging fixes are in history:
`e48fcce` (portable AppImage icon links, PR #197) and `503e1c0` (AppImage
launcher permissions). These historical packaging fixes are not evidence that
the current tray or WebKit rendering works on Arch.

## Compatibility questions still open

Confirmed from source: AppIndicator tray rectangles are unavailable through
this tray API, and the app's window placement code relies on that rectangle.
Not confirmed: the exact click, focus, activation, or window positioning
behavior under Hyprland/Wayland, X11, or other panels. Wayland compositors
control global window placement, so an X11-style request to place a popup next
to a status item may not be portable. Test both the normal Wayland session and,
if practical, an XWayland comparison before designing a general solution.

The black/flickering GUI remains untriaged. WebKitGTK version, graphics driver,
GPU renderer, Hyprland version, whether the affected surface is `main` or
`analysis`, and whether the issue follows show/hide or resizing are unknown.
The user's failed `WEBKIT_DISABLE_COMPOSITING_MODE=1` attempt should remain in
the report; do not mark that workaround as successful. Compare display paths
and renderer diagnostics before choosing a WebKit workaround.

The duplicate icon report also has multiple possible explanations: several
enabled provider icons are intentional; two running app instances could each
register their own provider icons; a Quickshell model/rendering issue is also
possible. The process names alone do not settle this.

## arch3050 next-session checklist

Record the AppImage hash/version, launch command, time, and exact reproduction
steps. Avoid collecting shell environments wholesale because they may contain
provider secrets. Capture only relevant display and version facts, such as
`XDG_SESSION_TYPE`, whether `WAYLAND_DISPLAY`/`DISPLAY` are set, Hyprland and
Quickshell versions, and installed WebKitGTK/GTK/Mesa/Ayatana package versions.

1. **Map processes to tray items.** Capture a process tree using PID, PPID,
   state, and executable/command name, then resolve `/proc/<pid>/exe` for each
   relevant PID. Do not dump `/proc/<pid>/environ` or full command lines.
   Determine whether an AppImage launcher parent and an extracted `quotabar`
   child belong to the same launch. Compare the count and IDs of registered
   StatusNotifier items with the enabled provider count in QuotaBar settings.
2. **Trace both tray open paths.** Test left click, right click/context menu,
   “Show / Hide Window”, and “Open QuotaBar” separately. Capture QuotaBar's
   `[Tray] ... clicked: show=... focus=... pos=...` logs. Check whether the
   main popover or analysis window becomes visible, whether focus-out hides it,
   and whether the menu action succeeds even when direct click does not.
3. **Inspect D-Bus and Quickshell.** Check which user-session
   `StatusNotifierWatcher` owns the watcher and which QuotaBar item IDs it has
   registered. Correlate this with Quickshell's tray model/logs and with
   QuotaBar process lifetime. Test after a clean app exit and a single relaunch
   to separate stale items from duplicate live registrations.
4. **Isolate rendering.** Record whether flicker affects the transparent
   `main` popover, the `analysis` window, or both. Compare Wayland and XWayland
   launches if available; note whether show/hide, focus changes, resizing, or
   moving workspaces trigger it. Record WebKitGTK and Mesa renderer versions.
   Preserve the result of the already-tried compositing environment variable.
5. **Retest a candidate fix.** Repeat the same launch and interaction steps,
   verify no extra tray items or orphaned process remain after Quit, and test
   both popover and workspace. Keep the same session/renderer versions so the
   comparison is meaningful.

Do not share auth JSON, tokens, full process environments, or logs containing
authorization headers. The Codex service currently logs response status and
latency; inspect logs for unexpected sensitive data before sharing them.

## Suggested first Linux engineering increment

Start with a small, observable tray acceptance task: on one clean Hyprland /
Quickshell session, determine whether the context menu's “Open QuotaBar” action
opens `analysis` while icon left-click does not, and correlate the result with
the backend callback log and SNI item count. This distinguishes an absent
Linux click event from a window/focus/WebKit failure without changing the
provider or rendering code. If the source-level limitation is confirmed, add
or adjust a Linux-specific activation path behind a narrow platform boundary,
and retain shared Tauri window and provider logic. Add a focused test for any
new event-to-action mapping; validate behavior on at least one Wayland panel
and one other Linux tray host before claiming broad support.

Keep upstream integration easy: isolate Linux-specific behavior in small
`cfg(target_os = "linux")` helpers or a focused tray adapter; avoid rewriting
shared provider services and avoid Arch-only paths. Keep the Linux CI gap
visible, then add a Linux CI check for formatting/build/tests once the
environment has the required GTK/WebKitGTK/Ayatana packages.

## Related source/history pointers

- Tray lifecycle, click handling, menu actions, positioning:
  [`tray.rs`](../src-tauri/src/services/tray.rs#L505),
  [`window.rs`](../src-tauri/src/services/window.rs#L32),
  [`lib.rs`](../src-tauri/src/lib.rs#L76).
- Window definitions and AppImage target: [`tauri.conf.json`](../src-tauri/tauri.conf.json#L12).
- Codex auth and `/wham/usage` request: [`codex.rs`](../src-tauri/src/services/codex.rs#L55),
  [`codex.rs`](../src-tauri/src/services/codex.rs#L288).
- Linux Claude auth limitation: [`claude.rs`](../src-tauri/src/services/claude.rs#L258).
- Linux AppImage build dependencies and CI matrix:
  [`release-artifacts.yml`](../.github/workflows/release-artifacts.yml#L62),
  [`ci.yml`](../.github/workflows/ci.yml#L9).
- Existing issue #186 concerns a Windows 11/4K tray geometry scenario; it is
  not evidence for these Linux symptoms:
  <https://github.com/majiayu000/quotabar/issues/186>.
- GitHub issue/PR live status could not be checked from this Cloud environment
  (GitHub API returned HTTP 403).

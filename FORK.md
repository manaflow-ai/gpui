# manaflow-ai/gpui

A fork of [zed-industries/zed](https://github.com/zed-industries/zed) for GPUI
work that cmux needs before (or instead of) upstream. It is a full fork of the
zed repository, not a crate extraction: GPUI's crates use zed's workspace
(`workspace = true` dependencies, shared lints), so a full fork merges upstream
with a plain `git merge upstream/main` and every change stays a normal diff on
zed's own files. Consumers pin a commit of a branch by git rev.

Branches: `messageslab/external-host` (base: zed `main` at a84689073d29,
2026-10-03).

## Changes (each one commit, written so it can go upstream)

1. **gpui: `WindowParent` and `WindowOptions::parent`.** A window can render
   into a native view owned by another toolkit instead of creating its own
   top-level window. The handle is a `raw_window_handle::RawWindowHandle`, so the
   API has one shape on every platform (`NSView` now; `HWND` child windows and
   X11 child windows / Wayland subsurfaces can follow).
   `Platform::supports_window_parent()` (default `false`); `Window::new` returns
   an error when a parent is given on a platform without support.
   Files: `crates/gpui/src/platform.rs`, `crates/gpui/src/window.rs`.

2. **gpui_macos / gpui_platform: hosted platform.** `MacPlatform::new_hosted()`
   and `gpui_platform::hosted_application()`. The host owns `NSApplication`, its
   delegate, the run loop, the menu bar, the Dock menu and activation:
   `Platform::run` only runs the launch callback and returns (use the existing
   `Application::run_embedded`), and `quit`, `activate`, `set_menus` and
   `set_dock_menu` do nothing. GPUI work runs on the main dispatch queue, which
   the host's run loop drains. Files: `crates/gpui_macos/src/platform.rs`,
   `crates/gpui_platform/src/gpui_platform.rs`.

3. **gpui_macos: render into a host `NSView`.** With `WindowOptions::parent`,
   `MacWindow::open` builds the window as usual (never shown or focused), then
   moves the GPUI view (its `CAMetalLayer`) into the host view, sized to it and
   autoresizing. The GPUI-created `NSWindow` stays offscreen as the view's
   owner: it keeps the window state ivar and observes the host window's
   key, resign-key, occlusion and screen notifications, so GPUI's existing
   delegate handlers run unchanged. `native_window` becomes the host window, so
   scale, display link, occlusion and key state come from it.
   - Coordinates: pointer events, `mouse_position` and the IME rectangle
     (`firstRectForCharacterRange`) use the view's position inside the host
     window, not the window's content view.
   - Focus: a hosted view takes part in the host's responder chain
     (`acceptsFirstResponder`); a click makes it first responder; GPUI's window
     is active when the host window is key *and* the GPUI view is first
     responder, and focus moving between host controls and GPUI is reported
     as activation (`becomeFirstResponder` / `resignFirstResponder`).
     `PlatformWindow::activate` makes the view first responder; it never orders
     the window or activates the app.
   - Host menus: `copy:`, `cut:`, `paste:`, `selectAll:`, `undo:` and `redo:`
     sent down the host's responder chain reach GPUI as the matching
     keystrokes, so the GPUI keymap stays the single place that binds them.
   - Accessibility: the AccessKit adapter subclasses the GPUI view (not the host
     window's content view), so GPUI's tree sits under the host view.
   - Window-level operations that belong to the host (title, edited flag,
     background, resize, zoom, minimize, fullscreen, window drag) do nothing.
   - Drop removes the observers and the view, then closes the owner window.
   Not yet: file drag and drop into a hosted view (registered on the owner
   window), host view moving to another window after attach.
   File: `crates/gpui_macos/src/window.rs`.

Consumer: manaflow-ai/messageslab `gpui/` (C ABI `gpui/src/embed.rs`, AppKit
host `gpui-embed/`, design and stage 2 plan in `gpui/EMBED.md`).

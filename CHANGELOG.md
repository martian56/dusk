# Changelog

## 0.2.0 - 2026-11-22

- Protocol plugins: long-running programs that talk to dusk over JSON-RPC on stdin and stdout (`protocol = true` in a `[[plugin]]`). They hear about opened, changed, saved, and closed files and cursor moves; add commands and key bindings; read, edit, and open buffers; and show messages and status line items. See `docs/protocol.md`.
- Plugins start at startup or when a file of their language opens (`activation`). A crashed plugin restarts on next use (three times at most), a flooding one is stopped, and requests time out after 10 seconds, without freezing the editor.
- `:plugin log <name>` shows a plugin's log.
- The [dusk-plugin](https://github.com/martian56/dusk-plugin) package writes plugins in Raven.

## 0.1.0 - 2026-11-22

First release.

- Modal editing: normal, insert, visual, and visual-line modes; `d c y gc` operators with motions and text objects; counts; `.` repeat; `f t F T ; ,`.
- Everyday keys in every mode (Ctrl+S, Ctrl+Z, Ctrl+P, Ctrl+F, Ctrl+K), Shift selection that typing replaces, Ctrl+arrow word moves, Alt+Up/Down to move lines.
- Mouse: click, drag, double-click, shift-click, wheel, middle-click paste, file tree, and tabs.
- Highlighting for 32 languages; more through the config.
- File tree, fuzzy file finder, project search, go to definition, rename in file, command palette.
- Themes (dusk, dawn, gruvbox, nord) and per-color overrides.
- Config at `~/.config/dusk/config.toml` and `.dusk.toml`: options, key bindings per mode, languages, and plugins that run external commands on the buffer, selection, or line.
- `--keys` replays keys and mouse events without a terminal, for scripted checks.

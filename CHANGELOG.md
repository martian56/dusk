# Changelog

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

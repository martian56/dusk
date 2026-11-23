# Changelog

## 0.6.0 - 2026-11-24

- Changes made outside dusk are highlighted: after a reload, added and changed lines get a gutter bar and a tinted background, and removed lines a mark. `] c` and `[ c` jump between changes, and `:clear_changes` drops them. Plugins set what to compare against with `changes/baseline`, so rook shows exactly what it edited. Themes color them with `ui.diff.add`, `ui.diff.change`, and `ui.diff.delete`.
- Shift with the wheel, or a sideways wheel, scrolls long lines.
- A panel docked on the right no longer hides the end of a long line.
- Help lines wrap; long paths shrink from the front so the cursor position stays on the status bar; the tab bar keeps the active tab in view and marks hidden tabs with `‹` and `›`.
- Undo back to the saved text leaves the buffer clean, `dd` on an empty buffer changes nothing, and a click after wide characters lands where it should.
- Jumping to the end of a long file is much faster (a 60,000-line file: 550 ms to 35 ms).
- Built on plumage 0.5.3, perch 0.4.1, and magpie 0.5.1.

## 0.5.0 - 2026-11-24

- Theme files: every `~/.config/dusk/themes/*.toml` adds a theme with a name, a description, an optional built-in base, and optional `[colors]`, `[styles]` (bold, italic, underline, dim per syntax kind), `[glyphs]` (brand, tab and tree markers, borders, corners, indent guide), and `[labels]` (mode names) tables.
- Plugins add themes: a `dusk-plugin.toml` with `themes = "themes"` contributes a directory of theme files and needs no command, and running plugins can `theme/define`, `theme/set`, and `theme/list`.
- `:theme <name>` with an unknown name says so instead of falling back to dusk.
- Fixed: `[theme]` overrides of `ui.*` colors, such as `"ui.bg"`, were ignored.

## 0.4.0 - 2026-11-22

- Files changed by other programs (another editor, git, rook in a terminal) reload by themselves. When the buffer has unsaved edits, dusk asks: `r` reloads it, `k` keeps your edits.
- Plugins get the real project directory as `root`, start their builds in the plugin's directory, and quit together, so leaving dusk with several plugins is quick.
- Picks answer by position, so two rows with the same label stay distinct; long status items are cut; save-as tells plugins the old name closed and the new one opened; a plugin restarted after a long run gets its three restarts back; `"error": null` counts as success and a request with a non-numeric id is reported.
- `close_panel` closes the focused panel only.
- Built on Raven 3.4.0.

## 0.3.0 - 2026-11-22

- Plugin UI that dusk draws: panels docked right or at the bottom, made of markdown, lists, text inputs, buttons, and progress; pickers and prompts; and marks in the gutter such as diagnostics. F6 and Shift+F6 move focus between the editor and panels, and the mouse works in them.
- rook runs inside dusk: add it as a plugin with `args = ["--dusk"]` and press `space r`.
- `:plugin install <git url>[@tag]`, `update`, `remove`, and `list` manage plugins in `~/.config/dusk/plugins`, described by a `dusk-plugin.toml`. Try `:plugin install github.com/martian56/dusk-plugin@v0.2.0` for a TODO finder.
- A plugin can tell dusk it changed files; buffers without unsaved edits reload.
- `--keys` scripts can wait with `<sleep:ms>`.

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

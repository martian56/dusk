# dusk

[![CI](https://github.com/martian56/dusk/actions/workflows/ci.yml/badge.svg)](https://github.com/martian56/dusk/actions/workflows/ci.yml)

A terminal code editor written in [Raven](https://github.com/martian56/raven). It edits like Vim (modes, operators,
motions, text objects, counts, `.` repeat) and also like an ordinary editor:
Ctrl+S saves, Ctrl+Z undoes, Ctrl+P opens files, and the mouse works
everywhere.

## Install

Download a build for Linux (tar, deb, rpm) or Windows (zip, msi) from the
[releases page](https://github.com/martian56/dusk/releases), or build it with
Raven 3.3.0 or newer:

```
git clone https://github.com/martian56/dusk
cd dusk
rvpm build
./target/raven-out/dusk src/main.rv
```

Press F1 inside dusk for every key and command. It runs in any terminal with
VT input and SGR mouse reporting, which covers Windows Terminal and every
mainstream Linux terminal.

## What it does

- **Modal editing**: normal, insert, visual, and visual-line modes; `d c y gc`
  operators with motions (`w b e 0 ^ $ gg G f t F T % { }`) and text objects
  (`iw aw i( a[ i{ i" i'`); counts (`3dd`, `5j`); `.` repeats the last change;
  `;` and `,` repeat a find; registers and the system clipboard.
- **Everyday keys in every mode**: Ctrl+S save, Ctrl+Q quit, Ctrl+Z / Ctrl+Y
  undo and redo, Ctrl+P find a file, Ctrl+F search, Ctrl+K command palette,
  Ctrl+B file tree, Ctrl+A select all, Ctrl+C / Ctrl+X / Ctrl+V copy, cut,
  paste, Ctrl+G go to line, Ctrl+O jump back, F2 rename, F12 config.
- **Mouse**: click to place the cursor, drag to select, double-click a word,
  drag in the line numbers to select lines, wheel to scroll, middle-click to
  paste, click the tree to open files and folders, click tabs to switch and
  `×` to close.
- **Highlighting for 32 languages**: Raven, Python, JavaScript, TypeScript,
  Java, C, C++, C#, Go, Rust, Kotlin, Swift, Ruby, PHP, Lua, Bash, PowerShell,
  SQL, HTML/XML, CSS, JSON, YAML, TOML/INI, Markdown, Zig, Dart, Haskell,
  Elixir, Dockerfile, Makefile, R, and Scala. Add more in the config.
- **Project tools**: a file tree, a fuzzy file finder, project-wide search
  (`space g` or `:grep text`), go to definition (`gd`), rename in file (F2),
  open the path under the cursor (`gf`), buffers with tabs.
- **Editing help**: auto-indent, auto-closing brackets and quotes, comment
  toggling for every language, indent guides, matching-bracket highlight,
  search highlight, `:s` and `:%s` replace, `:sort`.
- **Themes**: dusk, dawn (light), gruvbox, and nord, plus per-color overrides.

## Configuration

`F12` or `:config` opens `~/.config/dusk/config.toml` (created with every
option documented on first use). A `.dusk.toml` in the project directory is
read after it and wins.

```toml
[editor]
theme = "nord"
tab_width = 2
line_numbers = "relative"

[theme]
keyword = "#ff79c6"

[keys.normal]
"space w" = "save"

[keys.insert]
"j k" = "normal_mode"
```

Any command from the palette (`Ctrl+K`) can be bound in any mode.

## Files changed elsewhere

dusk notices when another program (git, a formatter, rook in a terminal)
changes an open file. A buffer without unsaved edits reloads, and one undo
brings back what you had; for a buffer with unsaved edits dusk asks whether
to reload it (`r`) or keep yours (`k`).

## Plugins

A plugin is a program dusk runs as a command named `plugin.<name>`. It can
read the buffer, the selection, or the current line on stdin, and dusk uses
what it prints: replace the input, insert at the cursor, show a message or a
panel, or reload the file. Plugins can run on save or open, and take a key.

```toml
[[plugin]]
name = "format"
description = "format with raven fmt"
command = "raven"
args = ["fmt", "--write", "{file}"]
output = "reload"
on = "save"

[[plugin]]
name = "sort"
command = "sort"
input = "selection"
output = "replace"
key = "f9"
```

Placeholders in `command` and `args`: `{file}`, `{dir}`, `{line}`, `{col}`,
`{word}`, `{lang}`. Run one from `:plugin.sort`, from the palette, from its
key, or on a visual selection with `:'<,'>plugin.sort`.

### Protocol plugins

A protocol plugin is a program that keeps running beside dusk and talks to it
over JSON-RPC on stdin and stdout. It hears about opened, changed, saved, and
closed files, and it can add commands and key bindings, read and edit buffers,
open files, show messages, and put items in the status line. It can also show
UI that dusk draws: docked panels with markdown, lists, text inputs, buttons,
and progress; pickers and prompts; and marks in the gutter such as
diagnostics. F6 moves focus between the editor and panels. Write one in any
language; in Raven, the [dusk-plugin](https://github.com/martian56/dusk-plugin)
package does the protocol for you.

```toml
[[plugin]]
name = "wordcount"
command = "wordcount"
protocol = true
activation = ["startup"]      # or ["language:raven"]
```

Install a plugin from git, pinned to a tag:

```
:plugin install github.com/martian56/dusk-plugin@v0.2.0
:plugin update todo          # to its newest tag, or todo@v0.2.0
:plugin remove todo
:plugin list
```

Installed plugins live in `~/.config/dusk/plugins` and start with dusk. A
plugin repository has a `dusk-plugin.toml`:

```toml
name = "todo"
version = "0.1.0"
description = "List TODO lines"
command = "./todo"            # relative to the plugin's directory, or on PATH
args = []
activation = ["startup"]
build = ["rvpm", "build"]     # optional, run once after cloning

[windows]                     # command and build may differ per platform
command = "./todo.exe"
```

A `[[plugin]]` entry in `config.toml` with the same name wins over an
installed one, which helps while developing a plugin.

To run rook, the coding agent, in a panel beside your code:

```toml
[[plugin]]
name = "rook"
command = "rook"
args = ["--dusk"]
protocol = true
```

A protocol plugin's commands show up in the palette and in `:plugins`, and its
key bindings never override yours or dusk's. A plugin that crashes is
restarted the next time you use it (three times at most), one that floods
output is stopped, and one that does not answer within 10 seconds times out;
none of these freeze the editor. `:plugin log <name>` shows what it wrote to
stderr and what dusk noticed about it. The messages are described in
[docs/protocol.md](docs/protocol.md).

## Languages

```toml
[[language]]
name = "nim"
extensions = ["nim"]
line_comment = "#"
block_comment = ["#[", "]#"]
keywords = ["proc", "let", "var", "if", "else", "for", "while", "return"]
types = ["int", "string", "bool"]
constants = ["true", "false", "nil"]
```

Also available: `builtins`, `strings`, `multiline_strings`, `files`,
`capitalized_types`, `directive`, `key_separator`, `identifier_chars`, and
`case_insensitive`.

## Testing without a terminal

`--keys` replays keys and mouse events, then `--screen` prints the final
screen and `--dump` the buffer:

```
dusk --keys "ggdd:w<cr>" --dump file.rv
dusk --keys "<click:40,5><drag:52,5>y" --size 100x30 --screen file.rv
```

## Development

```
rvpm build
rvpm test
rvpm fmt --check
```

## License

MIT

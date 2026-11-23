# dusk plugin protocol

Protocol version 1.

A protocol plugin is a program dusk starts and keeps running. They talk
JSON-RPC 2.0, one JSON object per line, over the plugin's stdin (dusk to
plugin) and stdout (plugin to dusk). Lines end with `\n`. Anything the plugin
writes to stderr goes to its log, shown by `:plugin log <name>`.

Requests carry an `id` and get exactly one response. Notifications have no
`id`. Both sides may send requests.

## Configuration

```toml
[[plugin]]
name = "wordcount"
command = "wordcount"         # looked up on PATH, or a path
args = []
protocol = true
activation = ["startup"]      # "startup" or "language:<name>"
```

## Lifecycle

1. dusk starts the plugin when an activation fires: at startup, or when a file
   of the named language is first opened.
2. dusk sends the `initialize` request. The plugin answers with what it
   contributes.
3. On quit, dusk sends the `shutdown` request, waits half a second, and then
   ends the process.

A plugin that exits on its own is reported once. Running one of its commands
starts it again, at most three times; after that it stays stopped.

Limits:

- A request dusk sends that is not answered in 10 seconds fails with a
  timeout message.
- A plugin that writes more than 1000 messages in one 50 ms tick, or more
  than 200 per tick five ticks in a row, is stopped.
- A line that is not valid JSON-RPC is logged and reported once.

## Positions and paths

Rows and columns count from 0; columns count characters, not bytes. Paths are
as dusk shows them, with forward slashes, relative to the directory dusk was
started in unless they were opened with an absolute path. Every change to a
buffer increases its `version`.

## Requests from dusk

### initialize

Params:

```json
{"protocol": 1, "root": ".", "version": "0.2.0"}
```

Result:

```json
{
  "name": "wordcount",
  "commands": [{"name": "wordcount.show", "title": "Show the word count"}],
  "bindings": [{"mode": "normal", "keys": "space w", "command": "wordcount.show"}]
}
```

Command names are `<plugin>.<command>`. Binding modes are `normal`, `insert`,
`visual`, and `global`; keys are written as in `config.toml` (`"g h"`,
`"ctrl+k"`). A binding that dusk or the user already has is ignored.

### command/run

One of the plugin's commands was run from the palette, a key, or `:`.

Params:

```json
{"command": "wordcount.show", "path": "src/main.rv", "version": 12, "selection": ""}
```

`selection` is the selected text in visual mode, otherwise empty. Result: any
object. An error response is shown in the message line.

### shutdown

No params. Answer, then exit.

## Notifications from dusk

| Method | Params |
| --- | --- |
| `buffer/opened` | `{"path"}` |
| `buffer/changed` | `{"path", "version"}`, at most once per tick per buffer |
| `buffer/saved` | `{"path"}` |
| `buffer/closed` | `{"path"}` |
| `cursor/moved` | `{"path", "row", "col"}`, at most every 100 ms |

`buffer/changed` carries no text; ask for it with `buffer/text`.

## Requests from a plugin

### buffer/text

Params: `{"path"}`. Result: `{"text", "version"}` for an open buffer. For a
file that is not open, `version` is -1. An empty path means the current
buffer.

### buffer/edit

Params:

```json
{
  "path": "src/main.rv",
  "version": 12,
  "edits": [{"start": [0, 0], "end": [0, 0], "text": "// hi\n"}]
}
```

Each edit replaces the text from `start` to `end` (both `[row, col]`) with
`text`. The edits apply as one undo step. If `version` is not the buffer's
current version the edit is refused with an error, and dusk shows why.
Result: `{"version"}`, the buffer's new version.

### buffer/open

Params: `{"path", "line"}`, with `line` counted from 1. Opens the file and
moves the cursor there. Result: `{}`.

## Notifications from a plugin

| Method | Params | Effect |
| --- | --- | --- |
| `ui/message` | `{"text", "error"?}` | Shows the text in the message line |
| `status/set` | `{"id", "text"}` | Sets a status line item; empty text removes it |
| `commands/register` | `{"commands"}` | Adds commands, as in `initialize` |
| `keys/bind` | `{"bindings"}` | Adds key bindings, as in `initialize` |
| `files/changed` | `{"paths"}` | The plugin changed these files on disk: an open buffer without unsaved edits reloads (one undo brings the old text back); one with unsaved edits stays and dusk says so |

## UI

dusk draws everything; a plugin describes it.

### ui/panel (notification from a plugin)

Opens a panel, or replaces the plugin's panel with the same `id`. Replacing
keeps the scroll position, focus, list selections that still exist, and text
the user typed into an input block whose new `text` is empty.

```json
{
  "id": "results",
  "title": "Search results",
  "dock": "right",
  "blocks": [
    {"id": "intro", "kind": "text", "text": "# Results\nThree **matches**."},
    {"id": "hits", "kind": "list", "items": ["a.rv", {"label": "b.rv", "detail": "2 hits"}]},
    {"id": "ask", "kind": "input", "text": ""},
    {"id": "act", "kind": "buttons", "items": ["Run", "Stop"]},
    {"id": "busy", "kind": "progress", "text": "searching", "value": 40}
  ]
}
```

`dock` is `right` (beside the text, up to 60 columns) or `bottom`. Block
kinds:

| Kind | Fields | Shows |
| --- | --- | --- |
| `text` | `text` | Markdown |
| `list` | `items`: strings or `{"label", "detail"}` | Selectable rows |
| `input` | `text` | A text box; Enter submits, Ctrl+N adds a line |
| `buttons` | `items`: strings | A row of buttons |
| `progress` | `text`, `value` | A bar for `value` 0 to 100, otherwise a spinner |

Blocks of other kinds are skipped and logged.

F6 moves focus from the editor through each panel and back, Shift+F6 goes the
other way, Esc returns to the editor, and a click focuses what it lands on.
In a panel, Tab moves between its lists, inputs, and buttons.

### ui/close (notification from a plugin)

Params: `{"id"}`. Closes the plugin's panel.

### ui/event (notification from dusk)

Something happened in a plugin's panel.

```json
{"panel": "results", "block": "hits", "kind": "choose", "value": 1}
```

| Kind | Value |
| --- | --- |
| `choose` | Index of the list row (Enter, or a second click) |
| `submit` | The input's text; the input then clears |
| `press` | Index of the button |
| `closed` | `null`; the user closed the panel (block is empty) |

### ui/pick (request from a plugin)

Params: `{"title", "items"}`, items as in a list block. Shows a picker.
Result: `{"index", "label"}`, with index -1 when the user dismissed it.

### ui/prompt (request from a plugin)

Params: `{"title", "text"}`, where `text` is the initial input. Result:
`{"text"}`, or `{"cancelled": true}`.

A pick or prompt that arrives while the user is typing or another one is
open waits until dusk is back in normal mode.

## Themes

### theme/define (notification from a plugin)

Adds a theme, or replaces one the plugin defined before. Colors not listed
come from `base`, a built-in theme (`dusk` when absent). Every color must be
`#rrggbb`; a bad one rejects the whole theme and dusk says why. Redefining
the current theme repaints it.

```json
{"name": "synthwave", "base": "dusk", "description": "Neon over a purple night", "colors": {"ui.bg": "#241b2f", "keyword": "#fede5d"}}
```

### theme/set (notification from a plugin)

Params: `{"name"}`. Switches to that theme; an unknown name is reported and
the current theme stays.

### theme/list (request from a plugin)

No params. Result: `{"themes": [{"name", "description"}], "current"}`.

Plugins can also ship theme files with no code at all: see `themes` in
`dusk-plugin.toml`.

### marks/set (notification from a plugin)

Replaces the plugin's marks on a file, which need not be open yet.

```json
{"path": "src/main.rv", "marks": [{"row": 3, "severity": "error", "message": "undefined name", "sign": "E"}]}
```

`severity` is `error`, `warning`, `info`, or `hint`. The gutter shows `sign`
(default: the severity's first letter), the most severe mark wins on a row,
and the message shows when the cursor rests on the row. Send an empty list to
clear them.

When a plugin stops, its panels, marks, status items, and any waiting pick or
prompt go with it.

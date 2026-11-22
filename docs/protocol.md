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

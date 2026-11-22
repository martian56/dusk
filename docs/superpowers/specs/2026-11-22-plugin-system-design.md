# dusk plugin system

## Intent

dusk should grow an ecosystem: extensions with their own UI, written by anyone
in any language, and an AI agent (rook) working inside the editor. Today a
dusk plugin is one `[[plugin]]` config entry that runs a command over the
buffer and uses its output; it cannot react to events, keep state, add
commands at runtime, or show UI.

## Goals

- Plugins are separate programs that talk to dusk over a documented protocol,
  so they can be written in any language and installed without rebuilding dusk.
- A plugin can react to editor events, edit buffers, register commands and key
  bindings, and show UI that dusk draws: panels, lists, pickers, prompts,
  messages, status items, and per-line marks such as diagnostics.
- rook runs inside dusk as a plugin: a chat panel where it reads and edits the
  project, with approvals asked in dusk.
- Plugins install from git like rook's plugins, pinned by tag.
- A Raven SDK package makes writing a plugin in Raven a few lines.

## Non-goals

- Plugins drawing arbitrary cells on the screen. UI is a fixed, growing set of
  widgets that dusk renders.
- Running plugin code inside the dusk process (no dynamic loading, no embedded
  interpreter).
- Sandboxing. A plugin runs as the user, like VS Code extensions and Neovim
  plugins; installing one is trusting it.

## Anti-goals

- A plugin that crashes, hangs, or floods output must not freeze or crash dusk.
- rook must keep working exactly as today: its terminal UI, print mode,
  plugins, hooks, MCP, tests, and evals do not change behavior.
- Existing `[[plugin]]` config entries keep working unchanged.

## Constraints

- Raven compiles ahead of time and cannot load code at runtime.
- `std.process` spawns a child with piped stdin and stdout; `read_stdout`
  returns whatever arrived since the last read without blocking. dusk polls
  plugin output each frame.
- plumage drives dusk with one update and view per event or tick. A plugin
  message has to arrive through the tick, so dusk ticks every 50 ms while any
  plugin is running (it ticks every 500 ms otherwise).
- Linux and Windows, in the terminals dusk already supports.

## Approaches considered

1. **Separate programs over a JSON protocol (chosen).** Language-neutral,
   crash-isolated, installs without a rebuild, and rook already is a separate
   program. UI is limited to the widgets dusk draws, a set that can grow.
2. Raven packages compiled into dusk. Full power and speed, but every install
   needs the Raven toolchain and a rebuild of dusk, and a bad plugin crashes
   the editor.
3. An embedded scripting language through the C FFI (Lua, as Neovim does).
   Fast and in-process, but plugin authors write Lua, the bindings are a large
   surface, and a script can still hang the editor.

The user chose approach 1.

## Design

### Transport

JSON-RPC 2.0, one message per line (newline-delimited JSON) on the plugin's
stdin and stdout, the same framing MCP's stdio transport uses. Stderr is
collected into a per-plugin log that `:plugin log <name>` shows. Requests
carry an `id` and get one response; notifications do not.

### Lifecycle

1. dusk starts a plugin when an activation event fires: `startup`, a file
   language (`language:raven`), or one of its commands being run.
2. dusk sends `initialize` with its version, the protocol version, and the
   workspace root; the plugin answers with its name and what it contributes
   (commands, key bindings, events it wants).
3. On quit, dusk sends `shutdown`, waits briefly, then kills the process.
4. A plugin that exits unexpectedly is reported once in the message line; its
   commands stay listed and restart it on next use, up to three restarts.

### Messages from dusk

Notifications: `buffer/opened`, `buffer/changed` (path and version; text on
request), `buffer/saved`, `buffer/closed`, `cursor/moved` (debounced),
`ui/event` (a widget the plugin owns was used: item chosen, input submitted,
button pressed, panel closed).

Requests: `command/run` (a contributed command was invoked, with the
selection if any).

### Messages from a plugin

- Buffers: `buffer/text` (read an open buffer, or a file on disk),
  `buffer/edit` (replace ranges, rejected if the buffer version moved on),
  `buffer/open` (open a path at a line).
- Contributions at runtime: `commands/register`, `keys/bind` (per mode, with
  dusk's own bindings and user config taking precedence).
- UI: `ui/message` (message line), `ui/pick` (a picker; the answer comes back
  as the response), `ui/prompt` (one-line input), `ui/panel` (open or replace
  a panel), `ui/close`, `status/set` (a status-line item),
  `marks/set` (per-line marks for a buffer: severity, message, and gutter
  sign, used for diagnostics, test results, and the like).

### Panels

A panel is docked on the right or at the bottom, has a title, and holds a
list of blocks that dusk lays out top to bottom:

- `text`: markdown, rendered with magpie.
- `list`: selectable rows with an optional detail; choosing one sends
  `ui/event`.
- `input`: a multi-line text box with a submit key, for chat.
- `buttons`: a row of labelled actions.
- `progress`: a spinner or a bar with a label.

The plugin replaces a panel's content by sending `ui/panel` again with the
same id; dusk keeps scroll position and focus. F6 moves focus to the next
panel and back to the editor (Shift+F6 goes the other way), and a click
focuses whatever it lands on. Ctrl+W keeps closing the current file.

### Robustness

- Each plugin has a bounded inbox; a plugin that floods output is throttled,
  then stopped with a message.
- A request to a plugin that does not answer within 10 seconds fails with a
  timeout; dusk never waits on a plugin inside a frame.
- Protocol errors are logged and shown once, never thrown into the editor.

### Install and configuration

- A plugin is a git repository with `dusk-plugin.toml`: `name`, `version`,
  `description`, `command`, `args`, `activation`, and optional per-platform
  commands.
- `:plugin install <git url>[@tag]` clones into `~/.config/dusk/plugins/<name>`
  and records the source and tag in a lock file; `update`, `remove`, and
  `list` manage them, the same model as rook's plugins.
- `[[plugin]]` entries in `config.toml` keep working as today ("action
  plugins"). A config entry can also point at a protocol plugin by path, for
  local development.

### Raven SDK

A `dusk-plugin` Raven package handles the protocol: framing, request ids, a
typed event loop, and builders for panels and marks. A plugin in Raven is a
`main` that registers handlers. Example plugins ship with it (a word counter
with a status item, and a TODO finder with a panel list).

### rook as the AI plugin

rook gains one additive entry point, `rook --dusk`, in a new `src/dusk/`
module. It speaks the dusk protocol on stdin and stdout and reuses the agent
exactly as print mode does: an `AgentState`, `run_turn` on a goroutine, and a
loop that drains display events.

- It opens a chat panel: transcript as markdown text blocks, an input block,
  and a progress block while a turn runs.
- Tool approvals and questions are asked with `ui/pick` in dusk.
- Tools keep writing files on disk. dusk reloads an open buffer whose file
  changed if the buffer has no unsaved edits, and asks before replacing one
  that does. A later version can route edits of open buffers through
  `buffer/edit`.
- `@file` mentions and the current selection can be sent from dusk with a
  command (`rook.ask_about_selection`).

Not breaking rook means: no changes to existing modules beyond small, tested
extractions needed for reuse; the terminal UI, print mode, sessions, plugins,
hooks, and MCP behave as before; all existing tests and behavior evals pass;
the new mode has its own tests against a fake dusk and a mock model server.

## Delivery

Each step ships on its own and keeps everything working:

1. Protocol host in dusk (spawn, framing, lifecycle, robustness), messages,
   status items, commands, and key bindings, with the Raven SDK and a word
   counter example.
2. Panels, pickers, prompts, and marks, with the TODO finder example.
3. `rook --dusk` and the chat panel.
4. `:plugin install` and friends.

## Testing

- Protocol framing, parsing, and widget layout as unit tests.
- dusk's headless `--keys` mode drives a scripted fake plugin, so host
  behavior (startup, crash restart, flood throttling, timeouts, panels) is
  tested end to end without a terminal.
- rook's mode is tested against a fake dusk on stdin and stdout and rook's
  existing mock model server.

## Calls made without the user

- JSON-RPC over newline-delimited JSON, matching MCP stdio.
- A 50 ms tick while plugins run.
- rook writes to disk and dusk reloads, for the first version.
- Delivery order above, with install last.
- F6 and Shift+F6 to move focus between the editor and panels.
- The Raven SDK as a separate package.

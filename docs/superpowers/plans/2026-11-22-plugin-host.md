# Plugin Host (Delivery Step 1) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** dusk starts protocol plugins as child processes, talks JSON-RPC with them, and lets them show messages and status items, register commands and key bindings, read, edit, and open buffers, and react to buffer events; a Raven SDK makes writing one easy.

**Architecture:** `src/rpc.rv` frames and encodes JSON-RPC lines; `src/host.rv` owns each plugin process and turns its messages into `HostAction` values without knowing about the editor; `editor.rv` polls hosts on every tick, applies actions, and emits buffer events. A fake plugin built as a workspace member drives host and editor tests. The SDK is a separate package, `github.com/martian56/dusk-plugin`.

**Tech Stack:** Raven 3.3.3, rvpm workspaces, `std.process` (`spawn`, non-blocking `read_stdout`), `std.json`, plumage/perch.

**Spec:** `docs/superpowers/specs/2026-11-22-plugin-system-design.md`

## Global Constraints

- Raven 3.3.3 in CI (`raven-version: "3.3.3"`), Linux and Windows.
- `rvpm fmt --check`, `rvpm build --workspace`, `rvpm test`, and `rvpm doc` pass.
- Existing `[[plugin]]` entries without `protocol = true` behave exactly as today.
- No em or en dashes and no " - " in code, comments, docs, commits, or PRs; minimal comments.
- Every PR has `--assignee martian56`; plain `git commit`, no AI trailers.
- Wire format: JSON-RPC 2.0, one JSON object per line, `\n` terminated.
- Request timeout 10 seconds; at most 3 restarts per plugin; tick 50 ms when any protocol plugin is configured, 500 ms otherwise.
- Plugin commands are named `<plugin>.<command>`.

## Review Focus

- A plugin writes half a line, or several messages in one chunk: framing buffers until `\n` and yields each message once (Task 1).
- A plugin's command does not exist: one error message naming it, no crash, no restart loop (Task 3).
- A plugin exits while dusk waits on its `command/run`: the wait ends with an error message and the restart is counted (Task 3).
- `buffer/edit` with a stale version: rejected with an error response, buffer untouched (Task 5).
- A garbage line on stdout: logged once as a protocol error; later valid messages still work (Task 3).

---

### Task 1: JSON-RPC framing and codec

**Files:**
- Create: `src/rpc.rv`
- Test: `src/rpc_test.rv`

**Interfaces:**
- Produces:
  - `class LineReader { buf: str }` with `fun push(self, chunk: str) -> list[str]` returning complete lines (without `\n`, `\r` trimmed) and keeping the tail.
  - `class RpcMsg { kind: str, id: int, method: str, params: JsonValue, result: JsonValue, error: str }`, where `kind` is `"request"`, `"notification"`, `"response"`, or `"invalid"`; `id` is -1 when absent.
  - `fun parse_msg(line: str) -> RpcMsg`
  - `fun encode_request(id: int, method: str, params: JsonValue) -> str`, `fun encode_notification(method: str, params: JsonValue) -> str`, `fun encode_result(id: int, result: JsonValue) -> str`, `fun encode_error(id: int, code: int, message: str) -> str`; each returns one line ending in `\n`.
  - Builders: `fun obj(pairs: list[(str, JsonValue)]) -> JsonValue`, `fun jstr(s: str) -> JsonValue`, `fun jint(n: int) -> JsonValue`, `fun get_str(v: JsonValue, key: str) -> str`, `fun get_int(v: JsonValue, key: str, default: int) -> int`.

- [ ] **Step 1: Write the failing tests**

```rust
fun test_reader_joins_partial_lines() {
    let r = LineReader("")
    assert_eq_int(len(r.push("{\"a\":")), 0)
    let lines = r.push("1}\n{\"b\":2}\r\n{\"c\"")
    assert_eq_int(len(lines), 2)
    assert_eq_str(lines[1], "{\"b\":2}")
    assert_eq_str(r.buf, "{\"c\"")
}

fun test_parse_kinds() {
    assert_eq_str(parse_msg("{\"jsonrpc\":\"2.0\",\"id\":3,\"method\":\"x\",\"params\":{}}").kind, "request")
    assert_eq_str(parse_msg("{\"jsonrpc\":\"2.0\",\"method\":\"x\"}").kind, "notification")
    let r = parse_msg("{\"jsonrpc\":\"2.0\",\"id\":3,\"error\":{\"code\":-1,\"message\":\"no\"}}")
    assert_eq_str(r.kind, "response")
    assert_eq_str(r.error, "no")
    assert_eq_str(parse_msg("not json").kind, "invalid")
}

fun test_encode_round_trip() {
    let line = encode_request(7, "buffer/text", obj([("path", jstr("a.rv"))]))
    assert_true(line.ends_with("\n"))
    let m = parse_msg(line.trim())
    assert_eq_int(m.id, 7)
    assert_eq_str(get_str(m.params, "path"), "a.rv")
}
```

- [ ] **Step 2: Run `rvpm test`; expect failures for undefined `LineReader`, `parse_msg`, `encode_request`.**
- [ ] **Step 3: Implement `src/rpc.rv` with `std.json` (`parse`, `stringify`, `JsonValue`).**
- [ ] **Step 4: Run `rvpm test`; the three tests pass.**
- [ ] **Step 5: Commit `feat: JSON-RPC line framing and codec`.**

### Task 2: Fake plugin workspace member and CI

**Files:**
- Modify: `rv.toml` (add `[workspace]` with `members = ["tests/fake-plugin"]`, `default-member = "dusk"`)
- Create: `tests/fake-plugin/rv.toml` (package `fake-plugin`, no dependencies), `tests/fake-plugin/src/main.rv`
- Modify: `.github/workflows/ci.yml` (`rvpm build --workspace` before `rvpm test`)

**Interfaces:**
- Produces: binary `target/raven-out/fake-plugin` (`.exe` on Windows), run as `fake-plugin <mode>`. It reads stdin with `std.io.read_line` and writes lines with `print`. Modes:
  - `echo`: answers `initialize` with `{"name":"fake","commands":[{"name":"fake.hello","title":"Say hello"},{"name":"fake.edit","title":"Insert X"},{"name":"fake.count","title":"Count chars"}],"bindings":[{"mode":"normal","keys":"g h","command":"fake.hello"}]}`. On `command/run` `fake.hello` it sends `ui/message` `{"text":"hello from fake"}` and `status/set` `{"id":"fake","text":"said hello"}`, then answers `{}`. On `fake.edit` it requests `buffer/edit` with `{"path":<params.path>,"version":<params.version>,"edits":[{"start":[0,0],"end":[0,0],"text":"X"}]}` and answers `{}` after the edit response. On `fake.count` and on `buffer/opened` and `buffer/changed` it requests `buffer/text` for the path and sends `status/set` `{"id":"chars","text":"chars: <n>"}`. On `shutdown` it answers and exits 0.
  - `crash`: answers `initialize`, then exits 3.
  - `flood`: answers `initialize`, then writes 200000 `ui/message` notifications as fast as it can.
  - `silent`: reads stdin and never writes.
  - `garbage`: answers `initialize`, writes the line `this is not json`, then behaves as `echo`.

- [ ] **Step 1: Write `tests/fake-plugin` and the workspace entries.**
- [ ] **Step 2: Run `rvpm build --workspace`; expect `target/raven-out/fake-plugin(.exe)` to exist.**
- [ ] **Step 3: Run `printf '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}\n' | target/raven-out/fake-plugin echo`; expect one response line with `"id":1` and `"commands"`.**
- [ ] **Step 4: Update CI and commit `test: fake protocol plugin for host tests`.**

### Task 3: Plugin host process

**Files:**
- Create: `src/host.rv`
- Test: `src/host_test.rv`

**Interfaces:**
- Consumes: Task 1 (`LineReader`, `parse_msg`, encoders, builders); Task 2 binary.
- Produces:
  - `class PluginSpec { name: str, command: str, args: list[str], activation: list[str] }`
  - `class Contribution { name: str, title: str }`, `class Binding { mode: str, keys: str, command: str }`
  - `class Edit { start_row: int, start_col: int, end_row: int, end_col: int, text: str }`
  - `enum HostAction { Message(str, str, bool), Status(str, str, str), Commands(str, list[Contribution]), Bind(str, list[Binding]), ReadText(str, int, str), EditBuffer(str, int, str, int, list[Edit]), OpenBuffer(str, int, str, int), Failed(str, str) }` where the first field is always the plugin name; `Message(plugin, text, is_error)`, `Status(plugin, id, text)`, `ReadText(plugin, request_id, path)`, `EditBuffer(plugin, request_id, path, version, edits)`, `OpenBuffer(plugin, request_id, path, line)`, `Failed(plugin, reason)`.
  - `class PluginProc` with `state: str` (`"running"`, `"restarting"`, `"stopped"`, `"failed"`), `log: list[str]` (capped at 500 lines), `restarts: int`, and methods:
    - `fun start_plugin(spec: PluginSpec, root: str) -> PluginProc` (spawns and sends `initialize` with `{"protocol":1,"root":root,"version":<dusk version>}`)
    - `fun poll(self, now_ms: int) -> list[HostAction]`
    - `fun notify(self, method: str, params: JsonValue)`
    - `fun run_command(self, name: str, params: JsonValue, now_ms: int)` (sends `command/run`)
    - `fun respond(self, id: int, result: JsonValue)`, `fun respond_error(self, id: int, message: str)`
    - `fun shutdown(self)` (sends `shutdown`, waits up to 500 ms, then kills)
  - Rules inside `poll`: an `initialize` response yields `Commands` and `Bind`; an unparsable line is logged and reported once per plugin as `Message(..., true)`; more than 1000 messages in one poll, or 5 polls in a row over 200, stops the plugin with `Failed(name, "flooding")`; a dusk request unanswered after 10000 ms yields `Message(name, "<method> timed out", true)`; an exit yields `Message(name, "exited with code <code>", true)` once and sets state `"exited"`; the next `run_command` on an exited plugin restarts it (`restarts` + 1) before sending, and after 3 restarts it yields `Failed(name, "exited <code> after 3 restarts")` and stays `"failed"`; a pending dusk request on a plugin that exits ends with `Message(name, "<method> failed: plugin exited", true)`; stderr lines go to `log`.

- [ ] **Step 1: Write the failing tests** (each polls in a loop with `sleep_millis(20)` up to 5 s, using `fake()` to build a spec for `target/raven-out/fake-plugin[.exe]`):

```rust
fun test_initialize_yields_contributions()      // Commands has "fake.hello"; Bind has mode "normal", keys "g h"
fun test_command_emits_message_and_status()      // after run_command("fake.hello"): Message "hello from fake", Status ("fake", "said hello")
fun test_missing_binary_fails_once()             // command "no-such-plugin-binary": exactly one Failed, no restart
fun test_crash_restarts_on_next_use_then_fails() // mode "crash": one exit message; each run_command restarts; after 3 restarts, Failed and state == "failed"
fun test_exit_during_command_reports_failure()  // mode "crash" with a command sent before it exits: "command/run failed: plugin exited"
fun test_flood_is_stopped()                      // mode "flood": Failed(..., "flooding"); state == "failed"
fun test_garbage_line_reported_once_then_works() // mode "garbage": one Message is_error; then fake.hello still answers
fun test_silent_request_times_out()              // mode "silent": run_command then poll with now_ms + 10001: Message "command/run timed out"
fun test_shutdown_stops_the_process()            // after shutdown(), state == "stopped" and the child is not running
```

- [ ] **Step 2: Run `rvpm build --workspace && rvpm test`; expect failures for the undefined host API.**
- [ ] **Step 3: Implement `src/host.rv`.** Keep a `pending: dict[int, (str, int)]` of request id to (method, sent_ms) for dusk requests; answer plugin requests only through `respond`/`respond_error`.
- [ ] **Step 4: Run the tests; all nine pass.**
- [ ] **Step 5: Commit `feat: plugin host process with restarts, timeouts, and flood control`.**

### Task 4: Config, startup, commands, bindings, messages, and status

**Files:**
- Modify: `src/config.rv` (`Plugin` gains `protocol: bool = false`, `activation: list[str] = ["startup"]`; read `protocol` and `activation`)
- Modify: `src/editor.rv` (`Ed` gains `hosts: list[PluginProc]`, `plugin_commands: dict[str, Contribution]`, `status_items: dict[str, str]`; methods below)
- Modify: `src/main.rv` (tick 50 ms when `cfg.plugins` has a protocol plugin; `update` calls `ed.plugin_tick(now_millis())` on `Tick`; call `ed.shutdown_plugins()` after `run` returns and at the end of `headless`)
- Modify: `src/view.rv` (status items, sorted by id, drawn left of the language name)
- Test: `src/plugin_test.rv`

**Interfaces:**
- Consumes: Task 3.
- Produces on `Ed`: `fun start_plugins(self)` (starts plugins whose `activation` has `startup`; called at the end of `new_editor`; a plugin whose activation is `language:<name>` starts when a buffer of that language first opens), `fun plugin_tick(self, now_ms: int)`, `fun shutdown_plugins(self)`, `fun apply_action(self, a: HostAction)`. Registered commands appear in the command palette and `:commands`, run through `run()` by full name, and bindings merge into `keys` without overriding a user or default binding. `:plugins` lists protocol plugins with their state.

- [ ] **Step 1: Write the failing tests** (an editor built with a config whose plugin is the fake in `echo` mode; a helper `wait_for(ed, pred)` ticks up to 5 s):

```rust
fun test_plugin_command_runs_from_palette()  // ed.run("fake.hello", 1); wait; ed.message == "hello from fake"
fun test_plugin_binding_runs_command()       // keys "gh" in normal mode lead to the same message
fun test_status_item_is_shown()              // after fake.hello, the rendered status line contains "said hello"
fun test_user_binding_wins()                 // with config keys.normal "g h" = "file_start", "gh" moves to the top and shows no fake message
fun test_action_plugins_unchanged()          // a [[plugin]] without protocol still runs once via :plugin.<name>
fun test_language_activation_waits_for_a_file() // activation ["language:raven"]: not running after startup; running after opening a .rv file
```

- [ ] **Step 2: Run `rvpm build --workspace && rvpm test`; expect failures.**
- [ ] **Step 3: Implement the config fields, `Ed` methods, tick wiring, and status drawing.**
- [ ] **Step 4: Run all tests; existing 21 and the new ones pass.**
- [ ] **Step 5: Commit `feat: run protocol plugins and show their commands, bindings, and status`.**

### Task 5: Buffer events and buffer requests

**Files:**
- Modify: `src/doc.rv` (`Doc` gains `version: int = 0`, bumped in `touched`)
- Modify: `src/editor.rv` (emit events; answer `ReadText`, `EditBuffer`, `OpenBuffer`)
- Test: `src/plugin_test.rv`

**Interfaces:**
- Consumes: Tasks 3 and 4.
- Produces: notifications `buffer/opened` and `buffer/closed` `{"path"}`, `buffer/changed` `{"path","version"}` (sent from `plugin_tick` at most once per tick per buffer), `buffer/saved` `{"path"}`, `cursor/moved` `{"path","row","col"}` (at most every 100 ms). `ReadText` answers `{"text","version"}` for an open buffer, else the file's text, else an error. `EditBuffer` applies edits in reverse order as one undo step and answers `{"version"}`; a version that differs from the buffer's answers the error `stale version <sent>, buffer is at <current>`. `OpenBuffer` opens the path, moves to the line, and answers `{}`. Paths are as the buffer shows them (forward slashes).

- [ ] **Step 1: Write the failing tests:**

```rust
fun test_opening_a_file_updates_char_status()     // open a 5-char file: status shows "chars: 5"
fun test_edits_notify_and_recount()               // type "ab" in insert mode: status becomes "chars: 7"
fun test_plugin_edit_applies_and_undoes()         // run "fake.edit": line 0 starts with "X"; "u" restores it
fun test_stale_edit_is_rejected()                 // change the buffer between command and edit via a version bump: buffer has no "X", message contains "stale version"
fun test_save_and_close_send_events()             // the fake's log of received methods (written to its stderr, read from host.log) includes buffer/saved and buffer/closed
```

- [ ] **Step 2: Run the tests; expect failures.**
- [ ] **Step 3: Implement the events and request handling.**
- [ ] **Step 4: Run all tests; they pass.**
- [ ] **Step 5: Commit `feat: buffer events and buffer requests for plugins`.**

### Task 6: Plugin logs and documentation

**Files:**
- Modify: `src/editor.rv` (`:plugin log <name>` opens a new buffer named `[log <name>]` holding the log)
- Modify: `README.md` (a Plugins section: action plugins as today, protocol plugins with a config example, message reference link)
- Create: `docs/protocol.md` (every message in this step with params and results, copied from the interfaces above)
- Test: `src/plugin_test.rv`

- [ ] **Step 1: Write `test_plugin_log_opens_a_buffer` (after the garbage mode reports, `:plugin log fake` opens a buffer containing "this is not json").**
- [ ] **Step 2: Run it; it fails.**
- [ ] **Step 3: Implement the command and write the docs.**
- [ ] **Step 4: Run all tests, `rvpm fmt --check`, and `rvpm doc`; they pass.**
- [ ] **Step 5: Commit `feat: plugin logs and protocol documentation`.**

### Task 7: The dusk-plugin SDK and the word-count example

**Files (new repository `testing/dusk-plugin`, pushed as `github.com/martian56/dusk-plugin`):**
- Create: `rv.toml` (package `dusk-plugin`, version `0.1.0`, no dependencies), `lib.rv`, `lib_test.rv`, `examples/wordcount.rv`, `README.md`, `LICENSE` (copied from dusk), `.gitignore`, `.gitattributes`, `.github/workflows/ci.yml` (build, test, fmt check on Linux and Windows with Raven 3.3.3)

**Interfaces:**
- Produces:
  - `fun plugin(name: str) -> Plugin`
  - `class Plugin` with `fun command(self, name: str, title: str, run: fun(Ctx, JsonValue) -> Unit) -> Plugin`, `fun on(self, event: str, run: fun(Ctx, JsonValue) -> Unit) -> Plugin`, `fun bind(self, mode: str, keys: str, command: str) -> Plugin`, `fun serve(self)` (reads stdin until `shutdown` or end of input; answers `initialize` with the registered commands and bindings)
  - `class Ctx` with `fun message(self, text: str)`, `fun status(self, id: str, text: str)`, `fun buffer_text(self, path: str) -> Result[str, str]` (sends `buffer/text` and waits for its response, queueing other messages), `fun edit(self, path: str, version: int, edits: list[(int, int, int, int, str)]) -> Result[int, str]`, `fun open(self, path: str, line: int)`
  - `examples/wordcount.rv`: on `buffer/opened` and `buffer/changed`, sets status `words` to `words: <n>`; command `wordcount.show` shows `<n> words, <m> lines` as a message.

- [ ] **Step 1: Write the failing tests in `lib_test.rv`:** `test_initialize_lists_commands_and_bindings` and `test_buffer_text_queues_other_messages`, both driving the message handling through an in-memory input and output so no process is needed.
- [ ] **Step 2: Run `rvpm test`; expect failures.**
- [ ] **Step 3: Implement `lib.rv` and the example.**
- [ ] **Step 4: Run the tests; they pass. Build the example and run it under dusk with a `[[plugin]]` entry (`protocol = true`); `--keys` on a 3-word file shows `words: 3` in `--screen`.**
- [ ] **Step 5: Create the GitHub repository, push, open a PR, merge on green CI, tag `v0.1.0`, and create the release.**

### Task 8: Release dusk 0.2.0

**Files:**
- Modify: `rv.toml` and `src/main.rv` (version `0.2.0`), `CHANGELOG.md`

- [ ] **Step 1: Run `rvpm build --workspace`, `rvpm test`, `rvpm fmt --check`, `rvpm doc`, and the CI smoke script locally; all pass.**
- [ ] **Step 2: Open the PR (assignee martian56), merge on green CI, tag `v0.2.0`; the release workflow publishes the packages.**

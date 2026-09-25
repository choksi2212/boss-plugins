# BOSS Hackathon 2026 - Plugin Demo Walkthrough

Three high-impact plugins shipped by `choksi2212` for the BOSS Contributor
Hackathon. Each section is a four-to-six-step walkthrough: install, open
the tab/panel, run the documented action, observe the result. The result
is shown in plain-text form so a reviewer can match the on-disk state to
the description without needing a screenshot.

Plugins covered:

1. [`boss-plugin-markdown-preview`](https://github.com/choksi2212/boss-plugin-markdown-preview)
   - Renders any `.md` file as a side-by-side editor and live preview.
2. [`boss-plugin-test-explorer`](https://github.com/choksi2212/boss-plugin-test-explorer)
   - Parses JUnit XML test reports into a navigable tree.
3. [`boss-plugin-mcp-tool-playground`](https://github.com/choksi2212/boss-plugin-mcp-tool-playground)
   - Lists every MCP tool contributed by every loaded plugin and lets the
     operator invoke any of them directly, bypassing host MCP policy.

---

## 1. Markdown Preview - side-by-side `.md` rendering

**Plugin:** [boss-plugin-markdown-preview](https://github.com/choksi2212/boss-plugin-markdown-preview)
**Repo:** `choksi2212/boss-plugin-markdown-preview`
**Tab type:** `markdown-preview` (registered by the plugin at load time).
**Auto-reload:** the tab subscribes to `ApplicationEventBus.fileChanges()`
on the opened file and re-renders 500 ms after the last write - so any
external save (editor, `git checkout`, IDE auto-save) is reflected in the
preview without a manual Reload click.

### Step 1 - install

```bash
cd ~/boss-plugins/boss-plugin-markdown-preview
./gradlew buildPluginJar
# Drop the produced jar into ~/.boss/plugins/dev/<pluginId>/v<ms>/<pluginId>.jar
# and add an entry in ~/.boss/plugins/installed.json - the helper at
# ~/boss-plugins/scripts/install-all.py already does both for all 19 plugins.
```

### Step 2 - open any `.md` file from BOSS

From the editor, right-click `README.md` -> **Open as Markdown Preview**.
The host hands the file path to the plugin's tab factory and the
`MarkdownTabComponent` is created.

### Step 3 - what the panel renders

A two-column surface:

```
+----------------------------+----------------------------+
|  EDITOR (left, host-owned) |  PREVIEW (right, plugin)   |
|                            |                            |
|  # Title                   |  Title (h1, large)         |
|  Hello **world**.          |  Hello world. (bold)       |
|                            |                            |
|  - [x] task A              |  [x] task A                 |
|  - [ ] task B              |  [ ] task B                 |
|                            |                            |
|  | col | col |             |  +---+---+  (table)        |
|  | --- | --- |             |  |col|col|                |
|  |  a  |  b  |             |  | a | b |                |
|                            |                            |
|  ```kotlin                 |  val x = 1  (highlighted)  |
|  val x = 1                 |                            |
|  ```                       |                            |
+----------------------------+----------------------------+
```

The preview is a self-contained Compose tree (no third-party parser).
GFM features covered: h1-h6 with anchor links, bold/italic/strike, inline
and fenced code (Kotlin, Java, Python, JS/TS, Bash, JSON, YAML, SQL),
links, ordered/unordered lists with nesting, task lists with checkbox
glyphs, blockquotes, GFM tables with per-column alignment, horizontal
rules, escaped HTML entities.

### Step 4 - toolbar actions

- **Open source** - opens the same file in the host's code editor tab.
- **Copy as HTML** - writes the rendered HTML fragment to the system
  clipboard via the host's `ClipboardProvider`. The fragment is identical
  to what the HTML-export MCP tool returns.
- **Theme** - light, dark, GitHub, Dracula for code-block colouring.
- **Reload** - re-reads the file via `FileSystemDataProvider`.

### Step 5 - auto-reload on save

Edit `README.md` in the host editor and save. Within 500 ms the preview
re-renders without a manual click; the watcher is held in a
`DisposableEffect` so closing the tab stops it.

### Step 6 - MCP equivalent (for an in-terminal agent)

The plugin exposes:

| Tool | Purpose |
|------|---------|
| `markdown_preview_render(path)` | Render a `.md` file to the same HTML fragment the preview panel uses. |
| `markdown_preview_render_to_html(path)` | Same as above, returns the HTML string only. |

An agent can call `markdown_preview_render("/path/to/README.md")` and
hand the rendered fragment to another tool - useful for a code-review
agent that needs to surface a doc's rendered view.

### Verification

The build/test job on the plugin repo (`boss-plugin-markdown-preview`)
runs `MarkdownParserTest` and `MarkdownRendererTest` against a corpus of
`src/test/resources/fixtures/*.md` (covering every GFM feature). The
plugin's own tests live next to the source under
`src/test/kotlin/.../markdown/`.

---

## 2. Test Explorer - JUnit XML tree

**Plugin:** [boss-plugin-test-explorer](https://github.com/choksi2212/boss-plugin-test-explorer)
**Repo:** `choksi2212/boss-plugin-test-explorer`
**Panel position:** `left_bottom`, priority 64.
**Sandbox:** in-process (no separate JVM), max 4 threads.

### Step 1 - install

```bash
cd ~/boss-plugins/boss-plugin-test-explorer
./gradlew buildPluginJar
# Same dev-install dance as the other plugins - the jar lands in
# ~/.boss/plugins/dev/<pluginId>/v<ms>/<pluginId>.jar.
```

### Step 2 - point at a JUnit XML directory

Open the Test Explorer panel (left sidebar bottom slot). Pick a directory
that contains JUnit XML output - Gradle's default is
`<module>/build/test-results/test/`. The panel re-parses the directory
on every selection.

### Step 3 - what the panel renders

```
[Test Explorer]
  /home/me/proj/build/test-results/test/         [Watch: Off]
+----------------------------------------+
|  Tests: 87   Passed: 85   Failed: 2    |
|  Skipped: 0   Time: 12.345s             |
+----------------------------------------+
| [All] [Passed] [Failed] [Skipped]      |
| Search: [_________________]            |
+----------------------------------------+

  v com.example.FooTest  (5 cases, 0.872s)
    > bar           PASSED  0.123s
    > baz           PASSED  0.045s
    > qux           FAILED  0.034s
      AssertionError: expected 1 but was 2
        at com.example.FooTest.qux(FooTest.kt:42)
    > ...2 more
  v com.example.BarTest  (3 cases, 0.115s)
    > ...
```

Each row is one `<testcase>`. Status icons colour-code the row:
green for passed, red for failed/errored, grey for skipped. Clicking a
FAILED row expands a panel with the throwable type, message, and full
stack trace.

### Step 4 - the four MCP tools

| Tool | Args | Purpose |
|------|------|---------|
| `test_explorer_summarize(dirPath)` | required | Counts + the list of failures with class, method, message, type. |
| `test_explorer_failure_messages(dirPath)` | required | One line per failure: `report \| suite \| class#method \| message`. |
| `test_explorer_watch_start(dirPath)` | required | Begin watching (re-parse every 2 s). Idempotent. |
| `test_explorer_watch_stop(dirPath)` | required | Stop watching. |

Example - agent triage:

```text
> mcp__boss__test_explorer_summarize({"dirPath":"/home/me/proj/build/test-results/test"})

Tests: 87   Passed: 85   Failed: 2   Skipped: 0   Time: 12.345s

Failures:
- com.example.FooTest#qux
    type:    java.lang.AssertionError
    message: expected 1 but was 2
- com.example.BarTest#edgeCase
    type:    java.lang.NullPointerException
    message: receiver must not be null
```

### Step 5 - the parser's defensive layers

`JunitXmlParser` defends against XML-bomb variants:

- **Streaming pull parser** (`XMLInputFactory`) with DTD and external
  entity resolution disabled - so billion-laughs has nothing to expand.
- **Element-depth cap** at `MAX_ELEMENT_DEPTH = 256` - quadratic blowup
  via deep nesting is bounded.
- **Byte caps** - `MAX_XML_BYTES_PER_FILE = 16 MiB`,
  `MAX_XML_BYTES_TOTAL = 64 MiB` per parse pass.

The parser uses `XMLInputFactory` (pull), not `DocumentBuilder` (DOM),
so memory grows with the output tree's *recursion*, not with the input
file's byte count.

### Step 6 - watch toggle

Click the **Watch: Off** toggle to start a 2-second re-parse loop. A
fresh test run appears in the tree without manual intervention. Toggling
off stops the loop. The same watcher is shared with the MCP
`test_explorer_watch_start` tool, so a watch started from MCP is
reflected in the panel's toggle and vice versa.

### Verification

The plugin's tests live at
`src/test/kotlin/.../testexplorer/JunitXmlParserTest.kt` and cover the
golden XML cases (single suite, multi-suite, top-level
`<testsuites>` wrapper, ignored/skipped rows, error rows, deep nesting)
plus the byte-budget and depth-cap defenses.

---

## 3. MCP Tool Playground - browse and invoke any MCP tool

**Plugin:** [boss-plugin-mcp-tool-playground](https://github.com/choksi2212/boss-plugin-mcp-tool-playground)
**Repo:** `choksi2212/boss-plugin-mcp-tool-playground`
**Panel position:** `left_bottom`, priority 74.
**Sandbox:** isolated process, max 4 threads.

### Step 1 - install

```bash
cd ~/boss-plugins/boss-plugin-mcp-tool-playground
./gradlew buildPluginJar
# Same dev-install dance as the others.
```

### Step 2 - open the panel

Left sidebar bottom slot. A warning banner is always visible at the top:

> Calls made here BYPASS host MCP policy. Use only for plugin development.

This is intentional - the playground's whole point is to call a tool
*by hand*, including before an Always Allow rule has been written. The
banner states it explicitly; the production `mcp__boss__<tool>` path is
the one to use for traffic that should count against a persistent rule.

### Step 3 - browse the tool list

Left column: filter box + a tool list grouped by the plugin that
contributed the tool (`providerId`). Example layout once a handful of
plugins are loaded:

```
[ filter: ____________ ]

v ai.rever.boss.plugin.dynamic.testexplorer (4)
    > test_explorer_summarize           [read-only]
    > test_explorer_failure_messages    [read-only]
    > test_explorer_watch_start         [side-effect]
    > test_explorer_watch_stop          [side-effect]
v ai.rever.boss.plugin.dynamic.markdownpreview (2)
    > markdown_preview_render           [read-only]
    > markdown_preview_render_to_html   [read-only]
v ai.rever.boss.plugin.dynamic.playground (4)
    > mcp_playground_list_tools         [read-only]
    > mcp_playground_schema             [read-only]
    > mcp_playground_call               [side-effect]
    > mcp_playground_history            [read-only]
v ai.rever.boss.plugin.dynamic.flowlint (4)
    > flow_lint_validate_graph          [read-only]
    > flow_lint_apply_quickfix          [side-effect]
    > ...
```

Each row shows the tool name, read-only vs side-effect marker, and a
short description on hover.

### Step 4 - pick a tool, edit args, click Call

Right column: tool name + description, an args editor pre-filled with
required fields from the tool's `inputSchema`, **Call** and **Clear**
buttons. After Call, the result panel shows:

```
[ok]  test_explorer_summarize            14 ms    [copy]
----------------------------------------------------------------
Tests: 87   Passed: 85   Failed: 2   Skipped: 0   Time: 12.345s

Failures:
- com.example.FooTest#qux
    type:    java.lang.AssertionError
    message: expected 1 but was 2
```

Errors get an `[error]` prefix in red and the same copy button. Duration
is wall-clock from `PlaygroundDispatcher.invoke()` returning to the
result landing in the panel.

### Step 5 - history list

Below the result panel, the last 20 calls in this session, most-recent
first. Each row is re-clickable to re-copy the result. The history is
in-memory only; closing the panel clears it.

### Step 6 - the four MCP tools of its own

| Tool | Args | Purpose |
|------|------|---------|
| `mcp_playground_list_tools()` | none | Every tool in the registry (including denied ones), JSON list. |
| `mcp_playground_schema(toolName)` | required | JSON Schema string for one tool. |
| `mcp_playground_call(toolName, argsJson)` | required | Invoke a tool. Bypasses policy. |
| `mcp_playground_history()` | none | Last 20 calls. Read-only. |

These four tools share one dispatcher with the panel, so a call from
`mcp_playground_call` lands in the panel's history list, and a click in
the panel lands in the MCP tool's history. An in-terminal agent can list,
describe, and call any plugin's tool through this surface without going
through the host's `mcp__boss__<tool>` path.

### Verification

`src/test/kotlin/.../playground/PlaygroundDispatcherTest.kt` exercises
the dispatcher with a stub `McpToolRegistry` (success, error, missing
tool, malformed args); `ArgsSchemaDefaultTest.kt` covers the
schema-driven default-args population. The plugin's tests run as part
of the standard `./gradlew test` cycle.

---

## How a reviewer can verify all three

1. Clone the [boss-plugins](https://github.com/choksi2212/boss-plugins)
   meta-repo (`git clone --recurse-submodules`).
2. Build each plugin in turn with `./gradlew buildPluginJar` (or run
   `./scripts/verify-plugin.sh <plugin-dir>` for a one-shot
   build + jar presence check).
3. Drop the produced jars into `~/.boss/plugins/dev/<pluginId>/v<ms>/`
   and add entries to `~/.boss/plugins/installed.json` -
   `scripts/install-all.py` automates this for all 19 plugins.
4. Launch BOSS; the plugin tab/panel appears in the next session.
5. Walk through the steps in each section above.

If a reviewer would rather read the source first, every section links
back to the plugin's source tree - all three are MIT-licensed and
shipped as git submodules under `choksi2212/`.

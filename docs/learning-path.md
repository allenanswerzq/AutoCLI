# AutoCLI Learning Path

A structured guide to fully understanding how AutoCLI works, from foundation types to AI-powered generation.

## Project Overview

AutoCLI is a **Rust CLI tool** that fetches data from **55+ websites** (Twitter, Reddit, YouTube, Bilibili, HackerNews, etc.) with a single command. It compiles to a **4.7MB binary** with zero runtime dependencies, delivering **12x speed** and **10x less memory** vs the original Node.js version.

The core architecture: **declarative YAML adapters** define how to scrape each website, a **pipeline engine** executes them step-by-step, and a **browser bridge** reuses your real Chrome sessions for authenticated sites.

### Workspace Structure (8 Rust Crates)

| Crate | Role |
|-------|------|
| `autocli-cli` | Entry point — Clap-based CLI, dynamic command tree, routes to executor |
| `autocli-core` | Foundation types: `CliCommand`, `Registry`, `Strategy`, `ArgDef` |
| `autocli-pipeline` | Pipeline executor — runs `fetch`, `select`, `map`, `evaluate`, etc. |
| `autocli-browser` | Browser automation via Chrome Extension + Daemon relay |
| `autocli-discovery` | Scans `adapters/` folder, parses YAML → `CliCommand` |
| `autocli-output` | Renders results as table / JSON / YAML / CSV / Markdown |
| `autocli-ai` | LLM-powered adapter generation and website exploration |
| `autocli-external` | Wraps external CLIs (`gh`, `docker`, `kubectl`) as subcommands |

---

## Phase 1: Foundation — Core Types & Data Model

Read these first to understand what everything is built on.

| # | File | What You'll Learn |
|---|------|-------------------|
| 1 | `crates/autocli-core/src/command.rs` | `CliCommand` struct — the central type representing one command (name, description, domain, strategy, browser flag, args, pipeline steps, columns) |
| 2 | `crates/autocli-core/src/args.rs` | `ArgDef` and `ArgType` — how arguments are defined (type, default, required, positional, choices) |
| 3 | `crates/autocli-core/src/strategy.rs` | `Strategy` enum — public (no auth), cookie (browser session), login (form submission) |
| 4 | `crates/autocli-core/src/registry.rs` | `Registry` — HashMap of `site → HashMap<name, CliCommand>`, the central command lookup |

**Key takeaway:** Everything revolves around `CliCommand`. A YAML adapter file becomes a `CliCommand`. The CLI, pipeline, and output all operate on this type.

---

## Phase 2: Declarative Adapters — The Heart of the System

Read YAML adapters from simple → complex to understand the pipeline DSL.

| # | File | Pattern |
|---|------|---------|
| 5 | `adapters/hackernews/top.yaml` | Simplest: public API, `fetch` → `select` → `map`. Two-pass fetch (IDs then details) with `buffer_unordered(10)` concurrency |
| 6 | `adapters/hackernews/search.yaml` | Adds template expressions `${{ args.query }}`, conditional URLs `${{ args.sort == 'date' ? ... }}`, query params |
| 7 | `adapters/bilibili/hot.yaml` | Browser + cookies + `evaluate` (inline JS calling internal API with `credentials: 'include'`) |
| 8 | `adapters/twitter/search.yaml` | Complex: browser auth, `navigate` → `scroll` → `evaluate` (DOM scraping with CSS selectors) |
| 9 | `adapters/reddit/hot.yaml` | Hybrid: `navigate` to set cookie context, then `evaluate` calls JSON API inside browser |

**Key takeaway:** Every website is different, but the pipeline normalizes them all into `[{rank, title, author, ...}, ...]`. The website-specific knowledge lives inside individual pipeline steps (URLs, JS code, field mappings), while the pipeline engine is completely generic.

### The Three Adapter Patterns

```
Pattern 1 — Public API:     fetch → select → map → limit
Pattern 2 — Internal API:   navigate → evaluate(JS fetch) → map → limit
Pattern 3 — DOM Scraping:   navigate → scroll → evaluate(JS DOM) → map → limit
```

### The Normalization Flow

Every website returns data differently, but the pipeline flattens them all into the same shape:

```
HackerNews API     →  { by: "pg", descendants: 45, ... }
Bilibili API       →  { owner: { name: "UP主" }, stat: { view: 1000 }, ... }
Twitter DOM        →  <article data-testid="tweet">...</article>
Reddit JSON API    →  { data: { children: [{ data: { author: "user" } }] } }

          ↓  (each adapter's steps absorb the differences)

ALL become:  [{ rank: 1, title: "...", author: "...", score: 123 }, ...]
```

### Where the Website Knowledge Lives

| Pipeline Step | What it absorbs |
|---|---|
| `fetch` url/params | API endpoint structure, query parameters |
| `select` | Response JSON nesting (e.g., `data.children` vs `hits` vs `data.list`) |
| `evaluate` | DOM structure, internal API calls, auth headers, cookie handling |
| `map` | Field name differences (`item.by` vs `item.author` vs `item.owner.name`) |
| `filter` | Site-specific junk (deleted posts, dead stories) |

---

## Phase 3: Discovery — YAML → CliCommand

Understand how YAML files on disk become executable commands at runtime.

| # | File | What You'll Learn |
|---|------|-------------------|
| 10 | `crates/autocli-discovery/src/yaml_parser.rs` | YAML deserialization → `CliCommand` struct. Field mapping, arg parsing, pipeline step extraction |
| 11 | `crates/autocli-discovery/src/builtin.rs` | `discover_builtin_adapters()` — recursively scans `adapters/` folder, groups by site name |
| 12 | `crates/autocli-discovery/src/user.rs` | `discover_user_adapters()` — loads custom adapters from `~/.autocli/adapters/`, same format |

**Key takeaway:** Discovery is just "scan folders → parse YAML → insert into Registry". Adding a new site = dropping a YAML file in the right folder.

---

## Phase 4: Pipeline Engine — Step-by-Step Execution

The execution engine that makes adapters actually run.

| # | File | What You'll Learn |
|---|------|-------------------|
| 13 | `crates/autocli-pipeline/src/executor.rs` | `execute_pipeline()` — loops through steps, pipes output of step N as input to step N+1 |
| 14 | `crates/autocli-pipeline/src/step_registry.rs` | `StepRegistry` — maps step type names ("fetch", "map", "evaluate") to handler implementations |
| 15 | `crates/autocli-pipeline/src/steps/fetch.rs` | HTTP fetch step — 3 modes: simple URL, object params, per-item concurrent fetch (`buffer_unordered(10)`) |
| 16 | `crates/autocli-pipeline/src/steps/transform.rs` | `select`, `map`, `filter`, `sort`, `limit` — the data transformation steps |
| 17 | `crates/autocli-pipeline/src/steps/browser.rs` | `navigate`, `click`, `type`, `evaluate`, `scroll` — browser automation steps |
| 18 | `crates/autocli-pipeline/src/steps/intercept.rs` | `intercept` — network request sniffer for capturing hidden API calls |
| 19 | `crates/autocli-pipeline/src/template/` | Template engine: parsing & evaluating `${{ expr }}` expressions with access to `args`, `item`, `index`, `data` |

**Key takeaway:** The pipeline is a chain of `(data, args) → data` transformations. Each step handler implements the `StepHandler` trait with one method: `async fn execute(page, params, data, args) → Result<Value>`. The `StepRegistry` is the plugin system.

### StepHandler Trait

```rust
#[async_trait]
trait StepHandler {
    fn name(&self) -> &'static str;
    async fn execute(
        &self,
        page: Option<Arc<dyn IPage>>,  // browser page (None for public commands)
        params: &Value,                  // YAML config for this step
        data: &Value,                    // output from previous step
        args: &HashMap<String, Value>,   // user CLI arguments
    ) -> Result<Value, CliError>;
}
```

### Template Context

Templates `${{ expr }}` have access to:
- `args.*` — user-provided CLI arguments
- `item.*` — current item when iterating (map, filter, per-item fetch)
- `index` — current iteration index (0-based)
- `data` — the full accumulated data from previous step

### Deep Dive: The `fetch` Step (3 Modes)

The fetch step in `steps/fetch.rs` has three operating modes:

**Mode 1 — Simple URL:** `- fetch: https://api.com/data.json`
Single GET request, returns JSON.

**Mode 2 — Object params:**
```yaml
- fetch:
    url: https://api.com/search
    method: POST
    params: { query: "${{ args.query }}" }
    headers: { Authorization: "Bearer ${{ args.token }}" }
```

**Mode 3 — Per-item concurrent fetch:**
When `data` is an array AND the URL contains `${{ item.* }}`, it automatically fans out:
```yaml
- fetch:
    url: https://api.com/item/${{ item.id }}.json
```
Fires up to **10 concurrent requests** using `futures::stream::buffer_unordered(10)`.
The decision is made by `is_per_item_url()` which scans for `${{ item }}` patterns with word-boundary detection (so `items` or `my_item` don't false-match).

### Deep Dive: The 5 Transform Steps

**`select`** — Drill into nested JSON with dot paths and array indices:
```yaml
- select: data.results[0].items    # walks: response["data"]["results"][0]["items"]
```

**`map`** — Reshape every item in the array:
```yaml
- map:
    rank: ${{ index + 1 }}
    title: ${{ item.title }}
    author: ${{ item.by }}           # rename fields to a common schema
```

**`filter`** — Keep items matching a condition (JavaScript-like truthiness):
```yaml
- filter: item.title && !item.deleted && !item.dead
```

**`sort`** — Order by a field (ascending or descending):
```yaml
- sort: { by: score, order: desc }
```

**`limit`** — Truncate to N items (supports template expressions):
```yaml
- limit: ${{ args.limit }}
```

### Deep Dive: The `intercept` Step (Network Sniffer)

Some sites (like Twitter) don't expose usable APIs and embed data via hidden XHR/fetch calls. The `intercept` step captures these by monkey-patching `window.fetch` and `XMLHttpRequest` inside the browser:

```yaml
- intercept:                     # 1. Set up the trap (install-only, don't wait)
    pattern: "*/SearchTimeline*"
    collect: false
- navigate:                      # 2. Trigger the page load (API calls fire)
    url: https://x.com/search?q=${{ args.query }}
- evaluate: |                    # 3. Harvest the captured API responses
    window.__autocli_intercepted
```

**Why intercept first, navigate second?** The interceptor must be installed *before* the page loads, otherwise the API calls happen during load and are missed. It's like setting up a camera before the action.

| Param | Default | Purpose |
|-------|---------|---------|
| `pattern` | required | URL glob to match (e.g., `*/api/v1/search*`) |
| `wait` | 5s | How long to wait for network activity |
| `collect` | true | If `false`, only installs the interceptor — returns data unchanged |

### Why `intercept` Exists

Modern SPAs (Twitter, Facebook, etc.) work like this:
1. HTML is basically empty (`<div id="root">`)
2. JavaScript app boots and makes hidden API calls with secret auth headers
3. API returns clean JSON, JS renders it into DOM

You can't `fetch` these APIs directly (missing auth tokens). DOM scraping is fragile (CSS classes change). But `intercept` captures the hidden API responses — you get clean structured JSON using the browser's own authenticated session.

---

## Phase 5: Browser Layer — The 3-Part Relay System

AutoCLI does **not** launch or embed a browser. It uses a relay architecture where your real Chrome does the actual work:

```
┌──────────┐     HTTP POST      ┌──────────┐    WebSocket     ┌─────────────────┐
│ autocli  │ ──────────────────► │  Daemon  │ ◄──────────────► │ Chrome Extension │
│   CLI    │  localhost:19925    │ (axum)   │   /ext endpoint  │ (in your Chrome) │
│          │ ◄────────────────── │          │                  │                  │
└──────────┘   JSON response    └──────────┘                  └─────────────────┘
```

| # | File | What You'll Learn |
|---|------|-------------------|
| 20 | `crates/autocli-browser/src/bridge.rs` | `BrowserBridge` — entry point, checks Chrome running, spawns daemon, waits for extension |
| 21 | `crates/autocli-browser/src/daemon.rs` | Axum HTTP+WebSocket server, relays commands between CLI and extension |
| 22 | `crates/autocli-browser/src/daemon_client.rs` | HTTP client that sends commands to the daemon with retry/backoff |
| 23 | `crates/autocli-browser/src/page.rs` | `DaemonPage` — implements `IPage` trait, every method generates JS and sends via daemon |
| 24 | `crates/autocli-browser/src/dom_helpers.rs` | JS code generators — `click_js()`, `auto_scroll_js()`, `install_interceptor_js()`, etc. |
| 25 | `crates/autocli-browser/src/cdp.rs` | Direct Chrome DevTools Protocol (alternative to daemon path) |
| 26 | `extension/src/background.ts` | Extension service worker — receives WebSocket commands, dispatches to Chrome APIs |
| 27 | `extension/src/cdp.ts` | `chrome.debugger` API — attaches to tabs, runs `Runtime.evaluate` |
| 28 | `extension/src/protocol.ts` | Command/Result types shared between daemon and extension |

### Connection Lifecycle (`BrowserBridge.connect()`)

```
connect() →
  ① Is Chrome running? (checks via tasklist/pgrep)
  ② Is the daemon running? (GET /health on port 19925)
     No → spawn `autocli --daemon --port 19925` as detached process
  ③ Is the Chrome extension connected? (GET /status → checks WebSocket)
     No → try to wake Chrome, wait up to 30 seconds
  ④ Return a DaemonPage
```

### The Daemon Server (Axum Routes)

| Endpoint | Purpose |
|----------|---------|
| `GET /health` | Liveness check |
| `GET /status` | Is extension connected? |
| `POST /command` | CLI sends commands here → forwarded to extension via WebSocket |
| `GET /ext` | Chrome extension connects via WebSocket |
| `POST /ai-generate` | Proxy AI requests to autocli.ai |

The daemon auto-shuts down after **5 minutes of inactivity** and sends WebSocket **heartbeats every 15 seconds**.

### The 10 WebSocket Actions (Daemon → Chrome Extension)

| Action | Chrome API Used | What it Does |
|--------|----------------|-------------|
| `exec` | `chrome.debugger` → `Runtime.evaluate` | Run arbitrary JS in page |
| `navigate` | `chrome.tabs.update(tabId, { url })` | Go to a URL |
| `tabs` | `chrome.tabs.query/create/remove` | List, create, close, switch tabs |
| `cookies` | `chrome.cookies.getAll({ domain })` | Read cookies (including HttpOnly) |
| `screenshot` | `chrome.debugger` → `Page.captureScreenshot` | Capture page as PNG/JPEG |
| `close-window` | `chrome.windows.remove(windowId)` | Close automation window |
| `cdp` | `chrome.debugger.sendCommand(method)` | Raw DevTools Protocol call |
| `set-file-input` | `chrome.debugger` → `DOM.setFileInputFiles` | Set files on `<input type="file">` |
| `read-article` | navigate + `Runtime.evaluate` (Readability.js) | Extract article content |
| `sessions` | Internal | List active automation sessions |

**Key insight:** Most pipeline operations ultimately become `exec` (run JS). The Rust side generates JavaScript strings in `dom_helpers.rs` and sends them via `exec`. Only 5 things need dedicated actions because they can't be done from page JavaScript: navigation, cookies (HttpOnly), screenshots, tab management, and raw CDP.

### Command Round-Trip Flow

```
CLI (Rust)                  Daemon (localhost:19925)          Chrome Extension
    │                              │                              │
    │  POST /command               │                              │
    │  { id: "abc123",             │                              │
    │    action: "exec",           │                              │
    │    code: "document.title" }  │                              │
    │─────────────────────────────►│                              │
    │                              │  WebSocket send              │
    │                              │─────────────────────────────►│
    │                              │                              │
    │                              │  chrome.debugger.sendCommand │
    │                              │  (Runtime.evaluate)          │
    │                              │                              │
    │                              │  WebSocket receive           │
    │                              │  { id: "abc123",             │
    │                              │    ok: true,                 │
    │                              │    data: "Hacker News" }     │
    │                              │◄─────────────────────────────│
    │  HTTP response               │                              │
    │◄─────────────────────────────│                              │
```

### Security Model

Chrome does NOT allow random programs to run scripts. The security relies on:

| Layer | Protection |
|-------|-----------|
| **User consent** | You manually install the extension and approve permissions (`debugger`, `scripting`, `tabs`, `cookies`) |
| **Manifest V3** | Chrome's strictest extension security model |
| **Localhost only** | Daemon binds to `127.0.0.1:19925` — no remote access possible |
| **Header check** | Daemon requires `X-AutoCLI` header on every `POST /command` |
| **Isolated window** | All automation runs in a dedicated Chrome window, not your browsing tabs |
| **URL filtering** | Extension refuses to debug `chrome://` or `chrome-extension://` pages |
| **Idle cleanup** | Automation window auto-closes after 30 seconds of inactivity |

The extension uses `chrome.debugger` API — the same protocol Chrome DevTools (F12) uses internally. It's like having DevTools controlled programmatically instead of manually.

### Why This Architecture (Not Puppeteer/Playwright)

```
Puppeteer/Playwright → Launches NEW browser → No logged-in sessions
Selenium WebDriver   → Uses browser driver   → Detectable by anti-bot systems
Direct CDP           → Needs --remote-debugging-port flag → User must restart Chrome

AutoCLI Chrome Extension → Reuses REAL Chrome with REAL sessions
                         → Already logged into Twitter, Bilibili, etc.
                         → Just tells Chrome "go to this page" and "run this JS"
```

**Key takeaway:** The architecture reuses the human's browser session instead of trying to automate authentication. This is why `strategy: cookie` commands work — the browser already has the cookies.

---

## Phase 6: CLI & Command Dispatch — Tying It All Together

Now see how user input flows through the entire system.

| # | File | What You'll Learn |
|---|------|-------------------|
| 29 | `crates/autocli-cli/src/main.rs` | Entry point: builds Clap CLI dynamically from Registry, parses args, dispatches |
| 30 | `crates/autocli-cli/src/execution.rs` | `execute_command()` — connects browser if needed, runs pipeline, renders output, handles timeouts (60s default) |
| 31 | `crates/autocli-cli/src/args.rs` | Argument validation & type coercion (string → int, choice validation, defaults) |

**Key takeaway:** The CLI is dynamically generated from the Registry at startup. There's no hardcoded command for "hackernews" or "twitter" — they're all discovered from YAML files and registered as Clap subcommands.

### Full Request Flow

```
$ autocli bilibili hot --limit 5 --format json

  ① main.rs        → Clap parses: site="bilibili", cmd="hot", args={limit:5, format:"json"}
  ② Registry       → Looks up CliCommand from discovered YAML adapters
  ③ execution.rs   → Sees strategy=cookie → connects BrowserBridge to daemon
  ④ Pipeline        → Runs steps sequentially:
                      evaluate(JS) → map → limit
                      Each step: render ${{ }} templates → execute → pipe data forward
  ⑤ Output          → Formats as JSON (--format json), prints to stdout
  ⑥ Cleanup         → Releases browser page
```

---

## Phase 7: Output Rendering

| # | File | What You'll Learn |
|---|------|-------------------|
| 32 | `crates/autocli-output/src/render.rs` | `render()` — dispatches to format-specific renderer based on `--format` flag |
| 33 | `crates/autocli-output/src/table.rs` | Default table output — uses `columns` from adapter to select/order fields |

**Key takeaway:** The `columns` field in YAML controls what appears in table output. All fields are always available in JSON/YAML output regardless of `columns`.

---

## Phase 8: AI & Advanced Features

| # | File | What You'll Learn |
|---|------|-------------------|
| 34 | `crates/autocli-ai/src/llm.rs` | LLM-based adapter generation — sends page data to server API, gets back YAML |
| 35 | `crates/autocli-ai/src/explore.rs` | AI website exploration — analyzes a site's APIs and structure |
| 36 | `crates/autocli-ai/src/cascade.rs` | Auth strategy probing — determines if a site needs public/cookie/login |
| 37 | `crates/autocli-external/src/loader.rs` | External CLI integration — wraps `gh`, `docker`, `kubectl` as autocli subcommands |

---

## Practice Exercise

After reading phases 1–6, try creating your own adapter:

1. Pick a public JSON API (e.g., `https://api.github.com/repos/nashsu/autocli/releases`)
2. Create `adapters/github/releases.yaml`:
   ```yaml
   site: github
   name: releases
   description: List GitHub releases
   domain: api.github.com
   strategy: public
   browser: false

   args:
     repo:
       type: str
       required: true
       positional: true
       description: "owner/repo"
     limit:
       type: int
       default: 10

   pipeline:
     - fetch:
         url: "https://api.github.com/repos/${{ args.repo }}/releases"
     - map:
         name: ${{ item.name }}
         tag: ${{ item.tag_name }}
         date: ${{ item.published_at }}
         author: ${{ item.author.login }}
     - limit: ${{ args.limit }}

   columns: [name, tag, date, author]
   ```
3. Run: `autocli github releases nashsu/autocli`

This exercises the full flow: discovery → arg parsing → pipeline (fetch → map → limit) → table output.

---

## Key Design Patterns

1. **Registry Pattern** — Dynamic command generation from YAML adapters. No hardcoded commands for any site.
2. **Declarative Pipeline** — Each step is `(data, args) → data`. Sequenced, composable, testable.
3. **Template Engine** — `${{ expr }}` syntax enables parameterized pipelines with access to `args`, `item`, `index`, `data`.
4. **Browser Session Reuse** — Single daemon shares sessions across commands. Chrome extension syncs real login sessions.
5. **Strategy-Based Auth** — `public` (direct HTTP), `cookie` (browser cookies), `login` (form submit). Determines if browser is needed.
6. **Extensibility** — User adapters in `~/.autocli/adapters/`, external CLIs in `~/.autocli/external/`, step handlers via `StepRegistry`.
7. **Per-Item Fan-Out** — The fetch step auto-detects `${{ item }}` references and fires concurrent requests (10 at a time).
8. **JS Code Generation** — `dom_helpers.rs` generates JavaScript strings that get sent to the browser. Rust never touches the DOM directly.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                     autocli-cli                         │
│  main.rs → args.rs → execution.rs                       │
└────────────┬───────────────┬───────────────┬────────────┘
             │               │               │
    ┌────────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
    │  autocli-     │ │  autocli-   │ │  autocli-   │
    │  discovery    │ │  pipeline   │ │  output     │
    │  (YAML→Cmd)   │ │  (executor) │ │  (render)   │
    └───────────────┘ └──────┬──────┘ └─────────────┘
                             │
                    ┌────────▼────────┐
                    │ Step Handlers   │
                    │ fetch, map,     │
                    │ evaluate, etc.  │
                    └────────┬────────┘
                             │
              ┌──────────────▼──────────────┐
              │      autocli-browser        │
              │  BrowserBridge → Daemon     │
              │  → CDP → Chrome Extension   │
              └─────────────────────────────┘

    ┌───────────────┐  ┌───────────────┐
    │  autocli-ai   │  │  autocli-     │
    │  (LLM gen)    │  │  external     │
    └───────────────┘  │  (gh, docker) │
                       └───────────────┘
```

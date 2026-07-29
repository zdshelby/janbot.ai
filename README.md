# Janbot

**On-device native AI agent for Mac.** Janbot connects to LM Studio, Ollama, Claude, HuggingFace, OpenRouter, or another OpenAI-compatible API — whichever you prefer — and acts as a persistent AI agent that can complete tasks, answer questions from your own documents, and operate autonomously via scheduled workflows.

> macOS 14+ · Apple Silicon · Tauri v2 · Rust backend

---

## Overview

Janbot started out as a fun way to troll my co-founder Jan Jongboom by replacing him as a Dutch swearing bot - and quickly esclated to a full agent app written in Rust/Tauri. Try it out and share your vision for what Janbot could become. Interested to turn this into a real product? I am looking for a founder team to make this happen.

Janbot is not a chatbot or coding harness. It is an open **agentic app**: a persistent AI that can write and edit files, run shell commands, search the web, browse pages, recall past conversations, and execute scheduled workflows — from the desktop app or via connectors like telegram or slack.

---

## Features

### Inference & Agents

- **Multi-provider inference** — LM Studio, Ollama (with in-app model browser and one-click pull/switch/delete), **vLLM** (port 8000), **llama.cpp** (port 8080), Claude API, HuggingFace Inference, OpenRouter, or any OpenAI-compatible Custom API endpoint; auto-detect picks the first reachable local server; streaming over SSE with tool-calling support
- **ReAct agent loop** — up to 45 think → tool-call → observe rounds with **tier-aware budgets**: observation byte budget and per-tool result truncation scale by tier (Fast 16KB, Large 64KB, God 96KB) so weak local models don't blow their context and capable models get the room they need. State-mutating tools (writes, edits, bash) execute serially in emission order; reads and web tools run in parallel. Auto-falls back to **Plan & Execute** mode for long-horizon tasks (unified parallel tool batch + merged system context). **Incremental SSE streaming** on OpenAI-compatible providers for faster time-to-first-token. **Session tool cache** deduplicates idempotent reads/web fetches within a run. **Gap critic** (Settings → Agent opt-in) runs a Low-effort completeness check before `respond` and can request one retry; when the turn used web search/fetch tools it also checks draft citations against source excerpts (`citation_gap`). High/XHigh skips the critic. **Thinking adapters** on God tier for Claude, HuggingFace, OpenRouter, Ollama, and local Qwen/GLM/DeepSeek templates.
- **Persisted action history** — every tool call in a run is stored in SQLite and shown as collapsible **Agent activity** on assistant messages (including after reload).
- **Sub-agent delegation** — `spawn_subtask` tool runs an isolated read-only research worker for parallel web research or long document reads.
- **Harness observability** — loop checkpoints survive approval pauses; per-run metrics (rounds, tool calls, token in/out) stored in `harness_metrics` and shown in Settings → Advanced → Traces. Streaming completions record API usage when available, with a character-based estimate as fallback. Background agent tasks emit `workflow-run-started` / `workflow-run` events with a live **Running…** indicator in the task config bar.
- **UI display (Settings → Agent)** — light/dark/system theme and **text size** (small → extra large); shared form components for consistency
- **Settings layout** — opening Settings replaces the chat sidebar with a settings sidebar and a top **Back to app** control, while the selected page starts at the top of the right pane. Navigation covers Agent, Models, **MCP**, Safety, **Connectors** (minimal filterable connector cards; Connect/Manage opens a setup dialog), then **Advanced** (Stats, Traces, Memory when enabled). The Memory tab lists stored memories with inline edit and remove; removing a wiki entry deletes its markdown file so re-index does not bring it back.
- **Chat chrome** — overlay title bar with **Janbot**; model and reasoning effort live in the composer. Connection status and traces are in Settings (Models / Advanced → Traces). Chat messages sit in a centered reading column; agent activity uses a bordered timeline that stays open after the turn finishes.
- **Models settings** — Claude, HuggingFace, and OpenRouter first; **Local servers** (LM Studio with auto-detect, Ollama, vLLM, llama.cpp) grouped together; Custom API last (auto-lists models from the endpoint for pool dropdowns). The built-in test suite reports in the UI only (no files written under your workspace).
- **Surface- and tier-filtered tools** — Telegram/Slack get a reduced tool set; Fast tier drops heavy tools (`browser_fetch`, `office`, `computer_use`) to reduce MoE schema noise.
- **Conversation-scoped capabilities** — the composer’s **Capabilities** picker (“Allowed for this chat”) is a **policy** lens: it enables or disables builtin namespaces and configured MCP servers only for the open chat. Built-ins start on; MCP servers inherit Settings defaults. Switching or reopening a chat resets the picker, while the first send preserves a draft chat’s choices. Servers that still need OAuth authorization are disabled in the picker until Connect succeeds in Settings → MCP.
- **Progressive tool discovery** — the model discovers extended built-in and MCP tools via `tool_search` (shortlist: name + one-line description) then `activate_tools` (load up to 5 full schemas into a sticky working set). A small **hot set** of control-plane and core I/O tools is always visible; everything else is discovered on demand. Claude `defer_loading` mapping is implemented but **disabled by default** (opt-in + supported-model gate; Anthropic’s contract ties the field to hosted tool search which Janbot does not wire). Local / OpenAI-compat providers keep hot+sticky only. **Code mode** (`run_tool_script`) lets the model fan out over allowed sticky/hot tools from a sandboxed JS guest via host-brokered `callTool` (approval fail-closed; summary-only model output; ~20s guest wall-clock timeout). The system prompt lists enabled **namespaces** (including `connected_apps` connectors with example tool names) and MCP servers; a credential-free catalog mirror under `.janbot/tool-catalog/` is available for grep/read. Follow-up messages with ticket-style IDs or explicit server names may auto-activate ≤2 continuity tools; alias-gated **connector intent** (e.g. email/gmail/inbox while `connected_apps` is allowed) shares that ≤2 sticky cap. Oversized MCP / bash / web-fetch / script results (≥12 KiB) spill to `.janbot/tool-results/` with a short stub in context.
- **MCP client** — review saved servers in a compact overview, then add or select one to replace the overview with its focused editor in Settings → MCP. Janbot supports local stdio and remote Streamable HTTP servers, keeps stateful sessions alive, accepts environment variables and HTTP authorization/custom headers, and supports **native OAuth** (Connect opens a browser loopback flow for servers like Linear; tokens are saved in Application Support beside other Janbot config). It reports connection health and discovered tools. Chat-enabled MCP servers appear in the namespace index; the model discovers their tools via `tool_search` → `activate_tools` (readable `mcp__{server}__{tool}` names). Follow-up messages that reference ticket-style IDs from prior MCP results (e.g. “more details about PROJ-123”) may auto-activate continuity tools. A chat may opt into a globally disabled server without changing its saved default. Calls require in-chat approval by default; trusting a server is an explicit per-server opt-in. MCP tool calls use a 65s inference timeout aligned with the MCP session budget.
- **Live plan tracking** — the model owns its task plan via an `update_plan` tool. The loop re-injects the live plan state into context every round, emits `plan-state-updated` + `executing_step` events to the UI for accurate progress, and runs a **respond mandate**: the model can't exit with text-only output while plan steps remain incomplete. A **file-based verifier** then `stat`s every file the plan claims to have produced — empty or missing files trigger automatic **tier escalation** (Fast → Large → God).
- **Read-before-edit gate** — `edit_file` rejects edits on files not yet read this conversation, forcing the model to see actual contents before targeting substrings. Soft-caps more than 6 `edit_file` calls on the same path per turn so large docs get few section-sized patches instead of micro-edit spirals. Combined with steering language that prefers `edit_file` over whole-file `write_file`, this avoids the 50KB request-bloat failure mode on large files. Truncated/invalid `write_file` JSON is rejected before the approval gate (with a chunked-write hint) instead of pausing for a useless `write_file · raw` approve.
- **In-loop context management** — before every model call: optional **context reset + handoff** on long runs (Low-effort structured summary, wipe history, keep files-read set; max 1 reset / 2 on deep-research), else the deterministic ladder (snip → microcompact → elide). Cross-turn LLM capsules still run post-store. **Diminishing-returns nudge** only after 3 consecutive stalled rounds (short text and no real tools — research/read/edit calls count as progress).
- **Automatic retry** — transient network errors, timeouts, and HTTP 429 rate-limits are retried transparently (exponential backoff with jitter, up to 2 retries) on all inference paths: chat, Telegram, Slack DMs, Slack channel mentions.
- **Reasoning effort harness (SOTA)** — single configured model per provider with compact chat prompt controls: **Model** (default or any enabled pool slot) and **Reason** (`Auto` / Low / Medium / High / XHigh). Auto scales effort from message complexity; High+ enables deep thinking. If the default model/provider is unreachable, Janbot tries other enabled pool models. Effort escalates on incomplete work; Claude maps `xhigh` → API `max`, HF/Kimi/GLM stop at `high`, and failed escalations back off instead of hard-stopping. Transient drops in `chat_with_tools` are retried, then fall back to another pool model when effort cannot go higher. Default loop: up to 45 rounds / 120 tool calls with near-limit compaction; progressive discovery via `tool_search` + `activate_tools`; `spawn_subtask` runs research / batch / orchestrate / background subagents.
- **Model pool (Settings → Models)** — green enable toggle per provider, up to three model slots each, and a default-model dropdown at the top. Claude models are listed from the Anthropic API with search.

### Tool System

21 built-in tools (+ MCP tools when configured):

| Category | Tools |
|---|---|
| **File** | `read_file` (optional `offset`/`limit` for mid-file sections), `write_file` (≤~8KB/call; chunk via `edit_file`), `edit_file`, `list_files`, `search_files` (dir or single file; prefer over bash grep/rg), `create_directory`, `read_document` (PDF/Excel/Word/PowerPoint), `create_document` (native Markdown → `.docx`/`.xlsx`/`.pdf`/`.pptx`), `office` (officecli edits; always allowed, prefer `batch_json` for multi-word text) |
| **Shell** | `bash` — execute commands in the working directory; automatically routes through [RTK](https://github.com/rtk-ai/rtk) when installed for token-optimised output (60–90% fewer tokens on git, cargo, find, and other common tools) |
| **Web** | `web_search` (Brave → Wikipedia → short DuckDuckGo attempt), `web_fetch`, `browser_fetch` (headless Chrome), `fetch_urls` (batch, up to 20) |
| **Memory** | `store_memory`, `recall` — hybrid BM25+vector long-term memory |
| **Wiki** | `get_wiki_backlinks`, `list_wiki_entries` — navigate the LLM Wiki knowledge base |
| **Agent** | `update_scratchpad`, `edit_personality`, `get_agent_info` |
| **Connectors** | `manage_workflow`, `search_slack`, `gmail_search`, `calendar_list`, `drive_search`, `google_create_doc`, `google_create_presentation` |
| **Meta** | `respond` — signal final answer; `spawn_subtask` — delegate read-only research to a sub-agent; `run_tool_script` — sandboxed JS fanout over allowed tools via `callTool` |

Per-tool timeouts prevent stalled loops (65 s MCP, 30 s browser, 45 s web, 30 s bash, 15 s local).

### Skills System

Slash-command skills live as markdown folders under `{working_dir}/agent/skills/`. Bundled defaults: `/research`, `/summarize`, `/write`, `/remember`, `/wiki`, `/pdf`, `/docx`, `/xlsx`, `/pptx`, `/office`, `/create-skill`. The skills index is injected into every system prompt so the agent can suggest them. Skills are hot-reloadable; the agent (or you) can add new ones. A skills curator pins bundled skills and archives unused agent-created skills after 90 days.

### Memory

Hybrid BM25 (FTS5) + vector (cosine similarity) search with Reciprocal Rank Fusion; top-3 relevant memories injected into every conversation; citations tracked per response. All embeddings run on-device via fastembed ONNX (all-MiniLM-L6-v2, 384 dims).

All memories use the **LLM Wiki** backend: raw observations are queued, merged by the LLM into structured markdown files under `{working_dir}/wiki/`, and indexed into searchable chunks. The direct-to-SQLite ("episodic") write path exists only in the benchmark test suite for side-by-side comparison; it is not a user-facing mode.

### Persistent Context

- `SOUL.md` — minimal Janbot personality for operating well (Settings → Agent).
- `USER.md` — who the user is + preferences (Settings → Agent).

### Connectors

- **Telegram** — bidirectional bot; persists chat history to DB; **inbound photos** are downloaded and passed to the model; **outbound images** when the assistant includes `![alt](path/to/file.jpg)` in its reply (file under the working directory); workflow results delivered to Telegram; per-chat rate limiting (5 messages / 60 s per user)
- **Slack** — indexes channel history into searchable memory; responds to DMs and channel mentions; **inbound image files** downloaded for vision; **outbound images** via `![alt](path)` in replies; per-channel rate limiting (5 messages / 60 s per channel)
- **Local folder mounts** — grant read or read-write access to any directory; tools respect the permission boundary
- **Google Workspace OAuth** — choose Gmail / Calendar / Drive (read) and Docs & Slides (read/write), then connect with a browser-based desktop OAuth flow that requests only those scopes. Search and create tools follow the selected services; write actions stay approval-gated. Local builds load the Desktop OAuth client from gitignored `.env` / `.env.local` (`JANBOT_GOOGLE_WORKSPACE_CLIENT_ID` and `JANBOT_GOOGLE_WORKSPACE_CLIENT_SECRET` from the client JSON); restart Tauri after changing those files. PKCE is still used, but Google's token endpoint also requires the Desktop client's `client_secret` (Google treats that value as embeddable for installed apps — still keep it out of git).

Both Telegram and Slack auto-reconnect at startup if a token was saved, and retry inference calls on transient failures.

### Scheduled & Goal Tasks

Two kinds of **agent tasks** live in the sidebar (not Settings):

- **Scheduled tasks** — cron-style jobs with a persistent prompt. Each task has its own chat thread; each run stores the prompt bubble then the result so reopen shows context. Optionally mirror to Telegram or Slack.
- **Goal tasks** — a sticky long-term goal Janbot works on over time. After each check-in the agent states when it will look again (e.g. tomorrow, next week); no fixed cron schedule.
- **Sidebar** — Tasks, Scheduled, and Goals are matching accordions (label, chevron, +). Collapse state and accordion open/close preferences persist across relaunches (localStorage). Active **Tasks** lists only conversations touched in the last 7 days, newest first; older idle tasks appear under **Archived → Older chats** (soft-hide, no DB archive). Manually archived chats still restore from **Archived chats**. Agent tasks archive from the config bar. Regular tasks use a header bar (rename, archive, delete). New Scheduled/Goal threads show a kind-specific orientation empty state without the chat logo. Scheduled/Goal tasks use the **default model** with configurable **reasoning effort** (Auto / Low / Medium / High / XHigh) and notify channel (auto-saved), with Reasoning and Notify using the shared menu select in the config bar, plus ▲/▼ expand controls.
- Both task kinds support **archive** to hide from the active sidebar while keeping history. A background scheduler checks every 30 s. Results are cached for 5 minutes to avoid re-running identical prompts.

Schedule formats: `every Xm`, `every Xh`, `daily HH:MM`, `weekly DAY HH:MM`

### Platform

- **Tray app** — runs in the menu bar; window close hides rather than quits; autostart on login
- **4-step onboarding wizard** — hardware detection, model recommendations by RAM, provider setup
- **File attachments** — attach via button or drag-and-drop PDF, Excel, Word, PowerPoint, and images into the prompt area; image previews in chat. Cached attachments restore a real extension when Finder drops `.pdf`, and the prompt steers the agent to `read_document` (not bash) for PDFs/Office. Malformed tool JSON with an unquoted `path` is repaired automatically.
- **Rich replies** — inline images (PNG/JPEG/GIF/WebP/SVG, relative or absolute paths), Open Graph link previews, and document preview cards (PDF/Office excerpts) in chat; connector image delivery for Telegram/Slack
- **Chat turn timeline** — thinking, tools, approvals, and questions evolve inside the assistant bubble (not separate HUD cards). User prompts are left-aligned as turn separators with a local date/time stamp. Interim model commentary streams into Thinking rows; the final `respond` payload is the response body. Collapses to “Thought for Ns · N actions” when the turn completes; historical tool traces still load via ActionHistory when no live timeline is present.
- **Office & PDF documents** — native `create_document` converts Markdown to Word/Excel/PDF/PowerPoint in-process (no bash toolchain discovery). `read_document` sniffs magic bytes so extensionless PDFs still extract. Bash PDF extractors (`pdftotext`, `textutil` on confirmations, etc.) are redirected to `read_document`. Format skills `/docx`, `/xlsx`, `/pptx`, `/pdf` teach create (`create_document`), read (`read_document`), and edit (`office`). Umbrella `/office` remains. Legacy `/pdf` fpdf2 venv remains as fallback.
- **Persistent conversation history** — SQLite; resumes your latest task chat on restart (no automatic new conversation)

### In-Chat Approval & Safety

Janbot pauses the ReAct loop when the model wants to use dangerous tools (`write_file`, `edit_file`, `create_directory`, `bash`, `manage_workflow`, or `edit_personality`) and shows a **compact inline approval gate** in the assistant turn timeline (one-line summaries; details on demand; action buttons pinned outside the scroll rail). Batch multiple tool calls into a single approval ask.

- **Sandbox-first shell** — with bash Seatbelt **On** (default), shell commands skip the approval pause; dangerous patterns are still blocked at execute time. Allowlist matching is flexible (first-token / basename / pipelines), so `grep` covers `grep -n "…" file`. Prefer native tools (`read_file`, `list_files`, `search_files`, `write_file` / `edit_file`, `read_document`) over bash `cat`/`ls`/`grep`/PDF extractors. Protected-folder guards match real `rm`/`mv` tokens (not substrings like “confirm”). Bash in-place edits of workspace markdown (`perl -i`, `sed -i`, python rewrite) are redirected to `edit_file`. Seatbelt allows Apple Git / Xcode CLT reads so `git log` / `git status` work under sandbox (still no `git push` auto-approve).
- **Hooks** — optional `config/hooks.json` for PreToolUse, PostToolUse, UserPromptSubmit, PreCompact, Stop (deny / audit_log / redact)
- **Session trust** — "Allow & trust session" grants the current conversation auto-approval for all gated tools until it ends or the app restarts (ephemeral, in-memory only)
- **Per-tool auto-approve toggles** — Settings → Safety lets you disable approval for specific tools (e.g. allow all `write_file` calls inside the working directory, or allow `bash` globally). New installs default working-directory writes and create_directory to on.
- **Workflow approval gate** — toggle whether scheduled workflow-initiated tool calls also pause for approval (default: auto-approve for hands-off operation)
- **Tool argument preview** — expand "Show details" in the approval card to see raw tool names and arguments before deciding
- **Clearer questions** — `ask_user` supports multiple-choice `options` so the agent asks one decisive question instead of vague “how should I proceed?” prompts
- **Document cards** — markdown links to `.md`/Office/PDF assets render as compact horizontal cards (format badge, filename → Finder, Open). Successful `write_file` / `create_document` / `edit_file` rows in the turn timeline also surface a document card from the tool path.

### Search & Retrieval

- **TF-IDF fallback** — if ONNX embedding fails, falls back to hash-based TF-IDF for memory retrieval
- **Wiki lint** — background health checks for broken links, orphan pages, and frontmatter issues
- **Message sources** — every response tracks its source URLs and files; clickable links in the UI

### Prompt System

- **Agent policy / prompt profiles** — surface-aware formatting: DesktopChat, SlackDm, Workflow, Compact
- **Prompt builder** — instruction hierarchy with trusted/untrusted context blocks

### CLI

- **`janbot-cli`** — standalone binary for headless operation (`src/bin/cli.rs`)

---

## Todo

- **Skill autonomous followup & self-improvement** — agent periodically reviews skill usage and refines or proposes skill updates

---

## Getting Started

### Prerequisites

- macOS 14 (Sonoma) or later, Apple Silicon
- [Rust](https://rustup.rs/) (latest stable)
- Node.js 18+
- [Tauri v2 prerequisites](https://v2.tauri.app/start/prerequisites/)
- One of: [LM Studio](https://lmstudio.ai/) on `localhost:1234`, [Ollama](https://ollama.ai/), or a Claude / HuggingFace / Custom API key
- (Optional) [RTK](https://github.com/rtk-ai/rtk) — when installed, bash tool output is automatically routed through RTK for token-optimised results; detected at startup, no configuration needed

### Development

```bash
git clone https://github.com/zdshelby/janbot-app
cd janbot-app
npm install   # also enables Husky pre-commit (rustfmt on staged Rust files)
npm run tauri dev
# with logging:
RUST_LOG=info npm run tauri dev
# or more verbose:
RUST_LOG=debug npm run tauri dev
```

Starts the Vite dev server on port 1420 alongside the Rust backend. The frontend hot-reloads; the backend recompiles on Rust changes. See [CONTRIBUTING.md](CONTRIBUTING.md) for the rustfmt pre-commit hook.

### Spark email MCP

Janbot can use Readdle’s [Spark](https://sparkmailapp.com) MCP server (the same bridge as Claude Desktop’s Spark extension) for inbox, calendar, contacts, and meetings via the local `spark` CLI.

**Requirements:** Spark Desktop running, CLI enabled under **Settings → AI Agents → Spark CLI Setup**, and Node 18+.

1. Install the MCP package once (from the Claude Desktop extension, or unpack [`Spark.mcpb`](https://github.com/readdle/spark-claude-extension/releases/latest/download/Spark.mcpb) into Application Support):

```bash
mkdir -p "$HOME/Library/Application Support/Janbot/mcp"
# If you already installed Spark in Claude Desktop:
cp -R "$HOME/Library/Application Support/Claude/Claude Extensions/local.mcpb.spark-mail-limited.spark" \
  "$HOME/Library/Application Support/Janbot/mcp/spark"
```

2. Add a stdio server in **Settings → MCP** (or merge into `~/Library/Application Support/Janbot/mcp_servers.json`):

```json
{
  "id": "spark",
  "name": "Spark",
  "transport": "stdio",
  "command": "/usr/local/bin/node",
  "args": [
    "/Users/YOU/Library/Application Support/Janbot/mcp/spark/server/index.js"
  ],
  "env": [{ "key": "SPARK_PATH", "value": "/usr/local/bin/spark" }],
  "enabled": true,
  "auto_approve": false
}
```

3. Optional: install the companion skill into the working directory — `spark skill --install "{working_dir}/agent/skills"`.

Leave `auto_approve` off unless you trust write/triage tools. Enable **Spark** in the chat Capabilities picker; tools appear as `mcp__spark__*`. Account access levels are controlled in Spark Desktop (read-only / triage / send).

### Cursor MCP (live UI debugging)

In debug builds, Janbot starts a local MCP socket so Cursor can screenshot, read the DOM, click, and type into the running app. The Rust plugin is gated behind `#[cfg(debug_assertions)]` and is never registered in release builds.

1. Ensure the guest JS package is installed (`npm install` — includes `tauri-plugin-mcp`).
2. Add this to Cursor MCP settings (`.cursor/mcp.json` or Cursor Settings → MCP):

```json
{
  "mcpServers": {
    "tauri-mcp": {
      "command": "npx",
      "args": ["-y", "tauri-plugin-mcp-server"],
      "env": {
        "TAURI_MCP_IPC_PATH": "/tmp/tauri-mcp-janbot.sock"
      }
    }
  }
}
```

3. Run `npm run tauri dev`, then use the `tauri-mcp` tools from Cursor (e.g. `take_screenshot`, `query_page`, `click`, `type_text`).
4. For native mouse hover/drag (and some click modes), grant **Accessibility** to the terminal running `tauri dev` (System Settings → Privacy & Security → Accessibility). Selector-based clicks work without it.
5. DevTools actions (`open_devtools` / `is_devtools_open`) require the plugin `devtools` feature (already enabled in `Cargo.toml`).
6. UX smoke checklist for chat/settings regressions: [docs/ux-smoke.md](docs/ux-smoke.md). Chat streaming helpers live under `src/chat/` (event listeners, message paging, pending-gate rehydration).

### Production build

```bash
npm run tauri build
# → src-tauri/target/release/bundle/macos/Janbot.app
# → src-tauri/target/release/bundle/dmg/Janbot_0.1.0_aarch64.dmg
```

To build a release and copy the DMG to `~/janbot.ai/` with a stable symlink:

```bash
npm run build:release
# → ~/janbot.ai/janbot-<version>-<timestamp>.dmg
# → ~/janbot.ai/janbot-latest.dmg  (symlink to latest)
```

### Code signing & notarization (required for public distribution)

Unsigned DMGs are blocked by macOS Gatekeeper. To distribute safely you need:

1. **Apple Developer ID Application certificate** — request from Apple Developer portal.
2. Set the signing identity in `src-tauri/tauri.conf.json`:
   ```json
   "macOS": {
     "signingIdentity": "Developer ID Application: YOUR NAME (TEAM_ID)"
   }
   ```
3. **Notarization credentials** (one of):
   - App Store Connect API key: set `APPLE_API_KEY` and `APPLE_API_ISSUER`
   - Apple ID + app-specific password: set `APPLE_ID`, `APPLE_PASSWORD`, `APPLE_TEAM_ID`
4. Run `npm run tauri build` — Tauri will sign, package, and staple the notarization ticket automatically.

For ad-hoc / local-only signing during development, leave `signingIdentity: "-"` (the default).

### Model setup

| Provider | How to connect |
|---|---|
| LM Studio | Start the app, load a model, enable local server on port 1234 |
| Ollama | Install from [ollama.com/download](https://ollama.com/download), launch the app (or run `ollama serve`); Janbot discovers installed models and lets you pull new ones from Settings → Models |
| vLLM | `python -m vllm.entrypoints.openai.api_server --model <model> --port 8000`; click **Auto-detect** or Use in Settings → Models |
| llama.cpp | `./llama-server -m model.gguf --port 8080`; click **Auto-detect** or Use in Settings → Models |
| Claude API | Paste your Anthropic API key in Settings → Models |
| HuggingFace | Paste your HF token in Settings → Models |
| OpenRouter | Paste your OpenRouter API key in Settings → Models, enable the provider, then pick up to three models. Janbot sends `provider.data_collection="deny"` on every OpenRouter request so providers that may store or train on prompts are excluded. Requests are paced client-side and 429/503 retries honor `Retry-After` when OpenRouter or an upstream provider asks Janbot to slow down. |
| Custom API | Enter a base URL (`https://host` or `https://host/v1`) and optional Bearer API key in Settings → Models. When the base URL is set, Janbot lists models from `GET /v1/models` for Primary / Secondary / Fallback dropdowns; you can still add an unlisted model ID by hand. Agent tool rounds use at least **16k max_tokens** so large `write_file` tool JSON is less likely to truncate mid-string on vLLM/LiteLLM gateways. |

Recommended models by RAM:

| RAM | Model | LM Studio search | `ollama pull` |
|---|---|---|---|
| 8 GB | Qwen3 4B (4-bit) | `qwen3-4b` | `ollama pull qwen3:4b` |
| 16 GB | Qwen3 8B (4-bit) | `qwen3-8b` | `ollama pull qwen3:8b` |
| 32 GB | Qwen3 30B-A3B (4-bit) | `qwen3-30b-a3b` | `ollama pull qwen3:30b-a3b` |

The Settings → Models → Ollama panel exposes one-click pulls for these models plus a live list of everything installed. Pulls are delegated to Ollama itself.

### Comparing providers

```bash
node scripts/benchmark.mjs \
  --lm-model qwen3-8b \
  --ollama-model qwen3:8b \
  --preset medium \
  --runs 3
```

Measures TTFT, total wall time, throughput (tokens/s), and tokens streamed — side-by-side with a per-metric winner. Pass `--json` for machine-readable output.

### Tests

Requires [cargo-nextest](https://nexte.st/) (`cargo install cargo-nextest --locked`).

```bash
# All Rust tests
cargo nextest run --manifest-path src-tauri/Cargo.toml
# or:
npm run test:rust
```

---

## Architecture

For the developer-facing architecture map and Mermaid diagrams, see [docs/architecture.md](docs/architecture.md).

```
┌──────────────────────────────────────────────────────────────┐
│                 React Frontend (TypeScript)                    │
│         Chat · Settings · Onboarding · Workflows              │
│                   Zustand · Vite                              │
└─────────────────────────┬────────────────────────────────────┘
                           │  invoke() / listen()
                           │  Tauri IPC
┌─────────────────────────▼────────────────────────────────────┐
│                   Rust Backend (Tokio)                        │
│                                                               │
│  AppState                                                     │
│  ├── inference: Arc<Mutex<InferenceEngine>>                   │
│  │     LM Studio · Ollama · Claude · HuggingFace · CustomAPI  │
│  ├── db: Arc<Mutex<Database>>          SQLite                 │
│  ├── local_embeddings                  fastembed ONNX         │
│  ├── model_list_cache                  30 s TTL               │
│  ├── inference_cache                   5 min TTL (workflows)  │
│  ├── session_tool_cache                30 min TTL             │
│  ├── extra_folders / read_only_folders folder access control  │
│  └── telegram / slack / connector_config                      │
│                                                               │
│  Background tasks (cancellable via CancellationToken)         │
│  ├── Workflow scheduler     every 30 s                        │
│  ├── Goals worker           every 2–6 h (randomized)         │
│  ├── Wiki ingest worker     every 5 s                         │
│  └── Cache cleanup          every 5 min                       │
└───────────────────────────────────────────────────────────────┘
```

### IPC model

The frontend calls Rust via `invoke("command_name", args)`. All registered commands return `CommandResult<T> = Result<T, AppError>` where `AppError` is a Tauri-serializable struct `{ message, code, recoverable }`. The backend pushes streaming data via `app.emit("event-name", payload)`.

Key events: `message-token` (streaming inference), `conversation-updated`, `provider-changed`, `workflow-run`, `skill-matched`, `slack-reply`, `inference-step`.

**Developer reference:** [docs/ipc.md](docs/ipc.md) (full command and event catalog), [docs/architecture.md](docs/architecture.md) (diagrams), [docs/README.md](docs/README.md) (doc index), [CONTRIBUTING.md](CONTRIBUTING.md).

### Inference & error flow

```
send_message / telegram_poll / slack_dm_poll / workflow_scheduler
        │
        ▼
  retry_with_config (max 2, exponential backoff + jitter)
        │
        ▼
  InferenceEngine::chat_with_tools()   ← ReAct + Plan&Execute loop
        │
   on transient error (Network / Timeout / RateLimit)
        │
        └──► retry → retry → return AppError { recoverable: true }
```

### Agent loop

`chat_with_tools()` implements a two-tier reasoning loop:

1. **ReAct loop** — up to 10 rounds of think → tool call → observe. Each round the model can call any tool; results are fed back into context. Mid-loop cancellation is supported via an atomic flag.
2. **Plan & Execute fallback** — if unresolved after 10 rounds, a planner decomposes the task into 2–5 steps; each step runs in its own 6-round mini-loop, then a final synthesis pass produces the answer.

### System prompt assembly

Each conversation builds a layered system prompt at call time:

1. `SOUL.md` + `USER.md` — agent identity + user context (source of truth; editable in Settings → Agent)
2. Working directory context + tool usage instructions
3. Current UTC date/time
4. Recent activity — last 5 conversations + last 3 workflow runs
5. Relevant memories — top-3 results from hybrid BM25+vector search (when wiki memory is enabled)
6. Skills index — all available slash-command skills with triggers and descriptions
7. Platform hint — plain text for Telegram, Slack markdown for Slack

### Skills

Skills are markdown folders in `{working_dir}/agent/skills/{name}/` with `SKILL.md` (YAML frontmatter + prompt) and optional `README.md`:

```markdown
---
name: research
description: Deep web research with a structured report
trigger: /research
args: topic
---
Research **{{args}}** thoroughly...
```

When a user types `/research climate tech`, the trigger is matched and `{{args}}` substituted before the message reaches the model. The agent can create new skills via `/create-skill` or `write_file` when it solves a novel problem.

### Memory

| Tier | Storage | When used |
|---|---|---|
| In-context | Conversation history | Every message |
| Personality | `SOUL.md` + `USER.md` | Injected at session start (identity + user context) |
| LLM Wiki | SQLite + wiki markdown + ONNX embeddings | Top-3 retrieved per message (toggle in Settings → Agent → Advanced) |

Chunks: 512 chars, 64-char overlap. Embeddings: 384-dim all-MiniLM-L6-v2 (fastembed, on-device). Retrieval: BM25 (FTS5) + cosine vector search via Reciprocal Rank Fusion (k=60). Citations tracked per response.

### Caching

| Cache | Key | TTL | Purpose |
|---|---|---|---|
| `model_list_cache` | provider URL string | 30 s | Avoids hammering LM Studio / Ollama on each UI poll |
| `inference_cache` | md5(system\_prompt + user\_prompt) | 5 min | Deduplicates identical workflow prompts (daily summaries etc.) |
| `session_tool_cache` | tool name + arguments | 30 min | Reuses web search / file read results across messages in the same conversation |

---

## Project Structure

```
janbot-app/
├── src/                        React + TypeScript frontend
│   ├── App.tsx                 Entry, Tauri event listeners
│   ├── store/appStore.ts       Zustand state
│   ├── types/index.ts          Shared TypeScript interfaces
│   └── components/
│       ├── Chat/               Streaming chat UI, skill autocomplete
│       ├── Layout/             AppShell, Sidebar
│       ├── Onboarding/         4-step setup wizard
│       └── Settings/           Agent, Models, Connectors, Workflows
├── src-tauri/
│   ├── src/
│   │   ├── commands/           Tauri IPC handlers — all return CommandResult<T>
│   │   │   ├── chat.rs         send_message (retry + streaming)
│   │   │   ├── settings.rs     provider config, personality, wiki
│   │   │   ├── models.rs       LM Studio / Ollama (cached model lists)
│   │   │   ├── memory.rs       memory CRUD and benchmarks
│   │   │   ├── connectors.rs   folder mounts
│   │   │   ├── workflows.rs    CRUD + scheduler delivery
│   │   │   ├── telegram.rs     bot connect + poll loop (rate-limited + retry)
│   │   │   ├── slack.rs        connect + DM/mention loop (rate-limited + retry)
│   │   │   └── testing.rs      provider test harness
│   │   ├── inference/
│   │   │   ├── mod.rs          InferenceEngine — all providers, ReAct, streaming
│   │   │   └── plan.rs         Live plan state, rendering, and file verifier helpers
│   │   ├── utils/
│   │   │   ├── cache.rs        MemoryCache<K,V> — TTL + LRU eviction + stats
│   │   │   ├── retry.rs        RetryConfig, RetryStrategy (fixed/exponential/linear)
│   │   │   ├── rate_limit.rs   TokenBucket, SlidingWindow, FixedWindow
│   │   │   └── async_utils.rs  ConcurrencyLimiter, TaskQueue, TaskManager
│   │   ├── error.rs            JanbotError (typed), AppError (Tauri-serializable),
│   │   │                       CommandResult<T> alias
│   │   ├── state.rs            AppState — fields, init, inference_cache_key()
│   │   ├── tools.rs            Tool registry and executor
│   │   ├── skills.rs           Skill loading, expansion, default seeds
│   │   ├── db.rs               SQLite schema and queries
│   │   ├── memory.rs           Memory storage, BM25, vector search, citations
│   │   ├── embeddings.rs       fastembed ONNX wrapper
│   │   ├── local_embeddings.rs fastembed integration
│   │   ├── personality.rs      Personality file loader + bundled default
│   │   ├── routing.rs          Per-request tier routing
│   │   ├── rich_files.rs       PDF, Excel, Word, PowerPoint, image parsing
│   │   ├── documents.rs        Native Markdown → docx/xlsx/pdf/pptx creation
│   │   ├── hf_config.rs        HuggingFace per-model parameter overrides
│   │   ├── wiki.rs             Wiki frontmatter and file format
│   │   ├── wiki_index.rs       Wiki indexing into chunks + embeddings
│   │   ├── wiki_ingest.rs      Ingest queue processor
│   │   ├── wiki_lint.rs        Wiki health checks and lint rules
│   │   ├── agent_policy.rs     Surface-aware prompt profiles (DesktopChat, SlackDm, Workflow, Compact)
│   │   ├── memory_store_guard.rs  Deduplicates store_memory calls within a single turn
│   │   ├── prompt_builder.rs   Instruction hierarchy, trusted/untrusted context blocks
│   │   ├── text_utils.rs       UTF-8 truncation and text helpers
│   │   ├── bin/cli.rs          Standalone CLI entry point
│   │   └── lib.rs              Tauri setup, background tasks, invoke_handler
│   ├── tests/
│   │   ├── memory_backend_bench.rs
│   │   ├── memory_store_guard_bench.rs
│   │   └── wiki_ingest_incremental.rs
│   └── tauri.conf.json
└── scripts/
    ├── benchmark.mjs           Provider speed benchmark (TTFT, throughput)
    ├── download-models.sh
    └── test-models.sh
```

---

## Roadmap

**Phase 1 — Working Chat** ✅
Tauri app, streaming chat, multi-provider inference, SQLite persistence, personality system, onboarding, Telegram, Slack, folder mounts, scheduled workflows, tool system, on-device embeddings, memory with BM25+vector search.

**Phase 2 — Agent Runtime** ✅
ReAct agent loop, Plan & Execute fallback, skills system, hybrid BM25+vector retrieval, temporal awareness, platform-aware output, live agent-info tool, skills index in system prompt.

**Phase 3 — Persistent Self-Model** ✅
User profile file, memory-notes file (curated standing facts across sessions), auto-skill-synthesis nudge, persistent agent UUID.

**Phase 4 — Production Hardening** ✅
- In-chat tool approval system: interactive approval cards in the chat stream (Allow / Deny / Edit request) for write_file, edit_file, bash, and other mutating tools; replaces native OS dialogs; state persisted in `pending_approvals` table
- Typed error handling: `JanbotError` enum → `AppError` (Tauri-serializable `{message, code, recoverable}`) across Tauri commands
- Automatic retry with exponential backoff on all `chat_with_tools` call sites (chat, Telegram, Slack ×2, workflows)
- Per-user/channel rate limiting on Telegram and Slack connectors
- Model-list cache (30 s TTL) on LM Studio and Ollama endpoints
- Workflow inference result cache (5 min TTL, md5 key) to skip duplicate LLM calls
- Removed dead code: `ConfigManager` (file-backed JSON config never wired in), `ModularEngine` (no inference capability), `integration_example.rs`
- Removed dead dependencies: `llama-cpp-2`, `mlx-rs`, `mlx-lm`, `tokenizers`, `hf-hub` (4,293 lines deleted)
- Tauri capabilities hardened: removed `shell:allow-spawn`, `shell:allow-execute`, `fs:read-all`, `fs:write-all` from global scope
- Strict Content-Security-Policy: restricts connect-src to known API domains only
- Bash tool blocklist expanded: `eval`, backticks, `$()`, `python -c`, `node -e`, and other obfuscation vectors
- URL validation: `open_url` restricted to `http://` and `https://` only
- Git-aware file safety: `edit_file` and `write_file` create timestamped `.janbot-backup.<timestamp>` copies before overwriting files not tracked by git (keeps only the latest backup per file). Deleting files inside protected folders (`Reports`/`Projects`/`agent`) is allowed; deleting those top-level folders themselves is blocked.
- Background cache cleanup task (every 5 min, cancellable)
- RTK integration: bash tool auto-detects the [RTK](https://github.com/rtk-ai/rtk) binary at startup and rewrites commands to their token-optimised equivalents (`git status` → `rtk git status`, etc.); falls back silently when RTK is not installed; status shown in Settings → Models

**Phase 5 — SOTA Harness** ✅
Single-model agent loop with reasoning effort levels (`/reasoning low|medium|high|xhigh`) instead of Fast/Large/God tier routing. Default harness is SOTA mode: one ReAct loop (45 rounds / 120 tool calls), no Plan & Execute fallback, **no plan-first / `update_plan` mandates**. **Near-limit compaction** (threshold + target ratios) keeps in-loop history under budget without running every round. **Subagents (`spawn_subtask`)** — research / batch (≤3 parallel) / orchestrate (batch + synthesize) / background modes; auto-summarize oversized fetches (>48KB); depth-limited. **Progressive tool discovery** via `tool_search` (shortlist) + `activate_tools` (sticky LRU, cap 5). Prompt-cache discipline: sorted trusted blocks + tool schemas. Gap critic **opt-in**. **Delivery pressure**. Interleaved `reasoning_content`. Model pool with auto-fallback. **Skills curator** tracks usage, pins bundled skills, archives agent-created skills unused 90+ days. Inline turn timeline for thinking / tools / approvals.

**Phase 5b — Multi-Agent & connectors** ✅
**Google Workspace** connector (Gmail / Calendar / Drive search plus approval-gated Docs / Slides creation). Settings → Connectors stores connection state in `google_workspace.json`; refresh/access tokens and provider API keys are kept in the secret store (macOS Keychain in signed release builds, AES-GCM vault under Application Support during `tauri dev` so unsigned rebuilds do not re-prompt). OAuth app credentials are supplied by runtime env loaded from `.env.local` for local Tauri dev/build runs, or by compile-time `JANBOT_GOOGLE_WORKSPACE_CLIENT_ID` / `JANBOT_GOOGLE_WORKSPACE_CLIENT_SECRET` for packaging (both required; Google Desktop token exchange rejects missing `client_secret` even with PKCE). **Richer subagent orchestration** — parallel batch + `orchestrate` synthesize pass. **OS-level sandboxing** — macOS Seatbelt for `bash` (`off` / `on` in Settings → Safety; default **on**). On: read + write scoped to workspace/mounts (+ minimal system paths for bash); network denied by default. Allowlisted `curl`/`wget` get network inside Seatbelt (so scheduled FX fetches work without turning sandbox off). Prefer `web_search` / `web_fetch` / `fetch_urls` for HTTP. Legacy `standard`/`strict` values migrate to `on`. **Hooks system** — `config/hooks.json` for PreToolUse, PostToolUse, UserPromptSubmit, PreCompact, Stop (deny / audit_log / redact).

---

## Data & Privacy

Everything is local. Conversations, memories, settings, and indexed content are stored in SQLite at:

```
~/Library/Application Support/Janbot/janbot.db
```

Optional HuggingFace tier defaults live in `~/Library/Application Support/Janbot/huggingface-models.json` (routing and per-model params only — model lists are fetched from HuggingFace like OpenRouter).

The working directory (`~/Janbot` by default) holds `SOUL.md`, `USER.md`, and agent output. Top-level folders **`Reports/`** (with `REPORTS.md`), **`Projects/`** (with `PROJECTS.md`), and **`agent/`** (skills, wiki, context) are created on startup if missing and protected from rename/delete — see Settings → Agent. Code lives in project subfolders under Projects. Long chat threads open at the start of the latest prompt + response and only pre-load that turn; scroll up (or use Earlier messages) to load history. Scheduled/Goal runs persist a user prompt bubble before each result so the thread shows the same prompt→response context.

Nothing leaves your Mac unless you explicitly configure a cloud inference provider (Claude API, HuggingFace Inference, OpenRouter, or a Custom API endpoint you control).

---

## Tech Stack

| Layer | Technology |
|---|---|
| Desktop framework | Tauri v2 |
| Frontend | React 18 + TypeScript 5 + Vite |
| State management | Zustand 5 |
| Backend | Rust + Tokio async runtime |
| Database | SQLite via rusqlite (bundled) |
| Inference | OpenAI-compatible HTTP — LM Studio, Ollama, vLLM, llama.cpp, Claude, HuggingFace, Custom API |
| Embeddings | fastembed ONNX (all-MiniLM-L6-v2, 384-dim, fully on-device) |
| File parsing | pdf-extract, calamine (Excel), zip (Word/PowerPoint) |
| Document create | docx-rs, rust_xlsxwriter, printpdf, zip (pptx), pulldown-cmark |
| Error handling | thiserror, typed `JanbotError` → `AppError` (Tauri-serializable) |
| Caching | `MemoryCache<K,V>` — TTL + LRU eviction + background cleanup |
| Retry | Exponential backoff with jitter, `RetryConfig` / `RetryStrategy` |
| Rate limiting | `SlidingWindow` per-user/channel on all connector inbound paths |

---

*Built by bots, for people.*

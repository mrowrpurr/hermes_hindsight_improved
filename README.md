# Hindsight Memory Provider (improved)

Long-term memory with knowledge graph, entity resolution, and multi-strategy retrieval. Supports cloud, local embedded, and local external modes.

Distributed as the **`hindsight_improved`** plugin — an enhanced, drop-in variant of the built-in `hindsight` provider. It adds synchronous (same-turn) recall, relevance/novelty filtering (`recall_top_k`, `recall_suppress_window`, `recall_min_score`), and bank-config control (`observations_mission`, extraction mode). Set `memory.provider` to `hindsight_improved` to use it; config still lives under `~/.hermes/hindsight/`.

## Install

```bash
hermes plugins install mrowrpurr/hermes_hindsight_improved
```

That clones this repo's `main` branch into `~/.hermes/plugins/` and prompts to enable the `hindsight_improved` plugin. If you skip the prompt, enable it later:

```bash
hermes plugins enable hindsight_improved
```

Then select it as your memory provider (or use the **Setup** wizard below):

```bash
hermes config set memory.provider hindsight_improved
```

Update to the latest with `hermes plugins update hindsight_improved`. Config lives under `~/.hermes/hindsight/` (shared with the built-in provider), so an existing Hindsight setup carries over.

## Requirements

- **Cloud:** API key from [ui.hindsight.vectorize.io](https://ui.hindsight.vectorize.io)
- **Local Embedded:** API key for a supported LLM provider (OpenAI, Anthropic, Gemini, Groq, OpenRouter, MiniMax, Ollama, or any OpenAI-compatible endpoint). Embeddings and reranking run locally — no additional API keys needed.
- **Local External:** A running Hindsight instance (Docker or self-hosted) reachable over HTTP.

## Setup

```bash
hermes memory setup    # select "hindsight_improved"
```

The setup wizard will install dependencies automatically via `uv` and walk you through configuration.

Or manually (cloud mode with defaults):
```bash
hermes config set memory.provider hindsight_improved
echo "HINDSIGHT_API_KEY=your-key" >> ~/.hermes/.env
```

### Cloud

Connects to the Hindsight Cloud API. Requires an API key from [ui.hindsight.vectorize.io](https://ui.hindsight.vectorize.io).

### Local Embedded

Hermes spins up a local Hindsight daemon with built-in PostgreSQL. Requires an LLM API key for memory extraction and synthesis. The daemon starts automatically in the background on first use and stops after 5 minutes of inactivity.

Supports any OpenAI-compatible LLM endpoint (llama.cpp, vLLM, LM Studio, etc.) — pick `openai_compatible` as the provider and enter the base URL.

Daemon startup logs: `~/.hermes/logs/hindsight-embed.log`
Daemon runtime logs: `~/.hindsight/profiles/<profile>.log`

To open the Hindsight web UI (local embedded mode only):
```bash
hindsight-embed -p hermes ui start
```

### Local External

Points the plugin at an existing Hindsight instance you're already running (Docker, self-hosted, etc.). No daemon management — just a URL and an optional API key.

## Config

Config file: `~/.hermes/hindsight/config.json`

### Connection

| Key | Default | Description |
|-----|---------|-------------|
| `mode` | `cloud` | `cloud`, `local_embedded`, or `local_external` |
| `api_url` | `https://api.hindsight.vectorize.io` | API URL (cloud and local_external modes) |

### Memory Bank

| Key | Default | Description |
|-----|---------|-------------|
| `bank_id` | `hermes` | Memory bank name (static fallback used when `bank_id_template` is unset or resolves empty) |
| `bank_id_template` | — | Optional template to derive the bank name dynamically. Placeholders: `{profile}`, `{workspace}`, `{platform}`, `{user}`, `{session}`. Example: `hermes-{profile}` isolates memory per active Hermes profile. Empty placeholders collapse cleanly (e.g. `hermes-{user}` with no user becomes `hermes`). |
| `bank_mission` | — | Reflect mission (identity/framing for reflect reasoning). Pushed to the bank on session start. |
| `bank_retain_mission` | — | Retain mission (steers what gets extracted from each turn). Pushed to the bank on session start. |
| `observations_mission` | — | Observations mission — steers what gets synthesized into durable observations (the layer recall surfaces). The highest-leverage knob for observation quality. Pushed to the bank on session start. |
| `retain_extraction_mode` | — | Fact extraction mode: `concise` / `verbose` / `custom`. Pushed to the bank on session start. |
| `retain_custom_instructions` | — | Custom extraction prompt (only active when `retain_extraction_mode` is `custom`). Pushed to the bank on session start. |

> The five bank knobs above are pushed to the Hindsight bank on session start (best-effort, in the background). Only keys you set are sent; an empty value leaves the bank's current setting untouched.

### Recall

| Key | Default | Description |
|-----|---------|-------------|
| `recall_budget` | `mid` | Recall thoroughness: `low` / `mid` / `high` |
| `recall_prefetch_method` | `recall` | Auto-recall method: `recall` (raw facts) or `reflect` (LLM synthesis) |
| `recall_max_tokens` | `4096` | Maximum tokens for recall results |
| `recall_max_input_chars` | `800` | Maximum input query length for auto-recall |
| `recall_prompt_preamble` | — | Custom preamble for recalled memories in context |
| `recall_tags` | — | Tags to filter when searching memories |
| `recall_tags_match` | `any` | Tag matching mode: `any` / `all` / `any_strict` / `all_strict` |
| `recall_types` | `observation` | Fact types surfaced by recall (both auto-recall and the `hindsight_recall` tool). Comma-separated string or JSON list. **Default narrowed to `observation` only** (see "Behavior change" below). Set to `observation,world,experience` to also include raw facts. |
| `auto_recall` | `true` | Automatically recall memories before each turn |
| `recall_top_k` | `8` | Max observations injected per auto-recall turn (`0` = unlimited). |
| `recall_suppress_window` | `-1` | Novelty window. `-1` (default) = never re-inject an observation already shown this session; `0` = off (repeats allowed every turn); `N` > 0 = an observation may reappear after N recall turns. Resets each new session. |
| `recall_min_score` | `0.0` | Drop auto-recall results below this cross-encoder relevance score (0.0–1.0; `0` = off). When > 0, recall is fetched with trace to obtain scores. Try ~0.05–0.1 to drop the low-relevance tail. |
| `recall_timeout` | `30` | Timeout in seconds for auto-recall, which blocks the turn. Kept below the general request `timeout` so a slow recall can't stall a turn for the full budget. Raise it if you use the slower `reflect` prefetch method. |

> **Behavior change — auto-recall is synchronous.**
>
> Auto-recall runs synchronously against the **current** message and injects the result into the same turn. (The upstream provider pre-fetched on the previous turn's message and surfaced it one turn late, so the model answered each question with the *previous* message's memories.) The trade-off is that recall latency is on the turn's critical path, bounded by `recall_timeout`.
>
> `recall_suppress_window` defaults to never-repeat (`-1`): once an observation has been injected this session it isn't shown again, so the model isn't re-fed the same facts every turn. State is per-session and stored under `~/.hermes/hindsight/seen/`.

> **Behavior change — `recall_types` defaults to `observation` only.**
>
> Previously recall returned all three fact types. It now returns only observations.
>
> Per [Hindsight's docs](https://hindsight.vectorize.io/developer/observations), observations are the **consolidated** knowledge layer Hindsight builds on top of raw facts: deduplicated beliefs grounded in evidence, refined as new facts arrive, with proof counts and freshness signals. Raw `world` / `experience` facts are the individual supporting evidence that feeds them. For per-turn context injection, observations are denser per token and avoid feeding the model multiple raw facts that one observation already summarizes.
>
> Restore the broad recall with `"recall_types": "observation,world,experience"` (string or JSON list) in `~/.hermes/hindsight/config.json`. This applies to **both** auto-recall and the `hindsight_recall` tool — both read the same `recall_types` setting (the tool schema has no per-call `types` argument), so narrowing the default narrows both paths.

### Retain

| Key | Default | Description |
|-----|---------|-------------|
| `auto_retain` | `true` | Automatically retain conversation turns |
| `retain_async` | `true` | Process retain asynchronously on the Hindsight server |
| `retain_every_n_turns` | `1` | Retain every N turns (1 = every turn) |
| `retain_context` | `conversation between Hermes Agent and the User` | Context label for retained memories |
| `retain_tags` | — | Default tags applied to retained memories; merged with per-call tool tags |
| `retain_source` | — | Optional `metadata.source` attached to retained memories |
| `retain_user_prefix` | `User` | Label used before user turns in auto-retained transcripts |
| `retain_assistant_prefix` | `Assistant` | Label used before assistant turns in auto-retained transcripts |

### Integration

| Key | Default | Description |
|-----|---------|-------------|
| `memory_mode` | `hybrid` | How memories are integrated into the agent |

**memory_mode:**
- `hybrid` — automatic context injection + tools available to the LLM
- `context` — automatic injection only, no tools exposed
- `tools` — tools only, no automatic injection

### Local Embedded LLM

| Key | Default | Description |
|-----|---------|-------------|
| `llm_provider` | `openai` | `openai`, `anthropic`, `gemini`, `groq`, `openrouter`, `minimax`, `ollama`, `lmstudio`, `openai_compatible` |
| `llm_model` | per-provider | Model name (e.g. `gpt-4o-mini`, `qwen/qwen3.5-9b`) |
| `llm_base_url` | — | Endpoint URL for `openai_compatible` (e.g. `http://192.168.1.10:8080/v1`) |

The LLM API key is stored in `~/.hermes/.env` as `HINDSIGHT_LLM_API_KEY`.

## Tools

Available in `hybrid` and `tools` memory modes:

| Tool | Description |
|------|-------------|
| `hindsight_retain` | Store information with auto entity extraction; supports optional per-call `tags` |
| `hindsight_recall` | Multi-strategy search (semantic + entity graph) |
| `hindsight_reflect` | Cross-memory synthesis (LLM-powered) |

## Environment Variables

| Variable | Description |
|----------|-------------|
| `HINDSIGHT_API_KEY` | API key for Hindsight Cloud |
| `HINDSIGHT_LLM_API_KEY` | LLM API key for local mode |
| `HINDSIGHT_API_LLM_BASE_URL` | LLM Base URL for local mode (e.g. OpenRouter) |
| `HINDSIGHT_API_URL` | Override API endpoint |
| `HINDSIGHT_BANK_ID` | Override bank name |
| `HINDSIGHT_BUDGET` | Override recall budget |
| `HINDSIGHT_MODE` | Override mode (`cloud`, `local_embedded`, `local_external`) |

## Client Version

Requires `hindsight-client >= 0.4.22`. The plugin auto-upgrades on session start if an older version is detected.

## Repository layout

This repo is a fork of [hermes-agent](https://github.com/NousResearch/hermes-agent). It carries three branches, each with a distinct job:

| Branch | Purpose |
|--------|---------|
| `main` | **The installable plugin** (this branch — what `hermes plugins install` consumes). Flat layout: `__init__.py`, `plugin.yaml`, and this `README.md` at the repo root. |
| `hermes-agent-hindsight-plugin-changes` | **Source of truth.** The provider changes on top of upstream hermes-agent (under `plugins/memory/hindsight/`), with the full test suite. Rebased periodically onto `hermes-agent` to stay current, and the basis for a possible upstream PR (undecided). |
| `hermes-agent` | A plain local mirror of upstream hermes-agent's `main`, kept so the source-of-truth branch can be rebased against it without network round-trips. |

The plugin code on `main` is identical to the source-of-truth branch except the two name strings (`plugin.yaml` `name` and the provider `name` property), which are `hindsight_improved` here so it installs alongside the built-in `hindsight` provider.

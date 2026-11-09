# Local models

Run your agents on models served from your own hardware: Ollama, LM Studio,
llama.cpp's `llama-server`, vLLM, SGLang, or anything else that speaks the OpenAI
or Anthropic API. T3 Code does not run models itself. A provider such as Pi or
OpenCode talks to your inference server, and T3 Code drives that provider.

The inference server must be reachable from the environment's machine. `localhost`
means the machine running the T3 Code server, even when you are connected from a
phone or another computer.

For routing proxies that use `x-proxy-session-id` and `x-proxy-turn-id`, the managed
gateway supplies private identifiers from the authenticated thread and active run.
Tool calls in one run share a turn; a new run gets a new turn. Background requests
and requests without an identified active run remain untracked. These headers do
not authenticate requests or prevent the proxy from routing to a cloud model.

## Use a local model

The quickest start is **Use a local model**: on first run under **Agents**, in **Settings →
Inference**, from the model picker, or from the command palette. T3 Code looks for model
servers on the environment's machine (the usual ports for Ollama, LM Studio, llama.cpp, vLLM and
SGLang, and on Linux every port the machine listens on) and on hosts you already registered. Pick
a model and an agent, and it registers the server, links the agent to it, and makes that model
the default for new threads. Running it again reuses what it set up before.

The search never sends a saved API key. A server that wants a key shows as **Needs an API key**;
add it below with its key. A server on another machine is found only on its usual ports; enter
its address if it uses another one.

## Register your inference servers

Open **Settings → Inference** and add each model server with **Add server**: its **Base URL**
(such as `http://localhost:11434/v1`), an optional name and API key, and optionally **Max
concurrent requests**. T3 Code checks each server and shows whether it is ready, the models it
serves with their context windows, and its current load when the server reports one. **Refresh**
checks again.

Each machine keeps its own list, because `localhost` means a different computer on each. To use a
server you registered on one machine from another, such as a Spark added over SSH, select that
machine at the top of **Settings → Inference**: servers registered on your other connected machines
are listed with **Add here**, with `localhost` already replaced by the name of the machine that
runs them. A key cannot be copied between machines, so enter it again if the server uses one.
Machines added over SSH from the desktop app run the app's own server; if a machine shows that its
server has no inference servers, restart the desktop app to install it there.

When a server stops answering, its row offers **Check again**, **Find server**, and **Change
address**, and a thread whose agent is linked to it says so above the message box. **Find
server** searches that server's machine and marks the address serving the same model; picking it
moves the server there and keeps its name, key, and linked agents. Threads already running keep
the old address until their agent session restarts.

Each server also shows when it was last checked, marked overdue when checks have stopped, whether
it reports cached prompt tokens, and the last request through T3 Code that failed or was rejected
in the past hour. Cached tokens show as `unknown` until a request in the last day comes back with a
cached count, so a server that never reports one cannot show its cache reuse anywhere.

A TensorFold server also shows its prompt cache, such as `Prompt cache 8.5 / 8.5 GiB · 25 saved
prompts · 85% of prompt tokens cached recently`: how much of its cache budget holds saved
prompts, how many it holds, and how much of the prompts sent since the previous check were read
from cache instead of computed again. A full cache is normal. A larger prompt cache keeps more
threads warm, so returning to one starts quickly; T3 Code warns only when the cache is full and
the cached share is falling, which means threads are being pushed out. A server without a prompt
cache shows `No prompt cache: every request reads its whole prompt`. On TensorFold, the cache size
is set with `--prompt-cache-gib` when the server starts.

A server can name another as its **Fallback server**, for when the first is down or unreachable.
T3 Code's gateway sends a request to the fallback only when the first server's last check did not
find it ready, or connecting to it failed, and only if the fallback serves the same model. It is
not for overflow: while the first server is up, requests wait in its queue even when the fallback
is idle. A request that already started is never moved.

Then link a provider instance to a server with the **Inference server** setting on its card in
**Settings → Providers**. A linked instance counts as local while its server is enabled, and the
model picker shows the server's name and readiness next to the provider, such as `Pi · rtx ·
Ready`. T3 Code also caps the context window it assumes for the instance at what the server
actually serves. Changing the link, or a server's URL, key or enabled switch, applies to sessions
that start afterwards; a running session stays on the server and URL it started with until it ends.

### Pi on a registered server

A Pi instance linked to a server needs no `models.json` entry. T3 Code registers the server in Pi
as `t3-<server>` with every model the server reports, sized from what the server advertises, so
pick a model such as `t3-rtx/your-model-id` in the model picker. These requests go through T3
Code's gateway, which:

- sends at most **Max concurrent requests** to the server at once and queues the rest, so
  parallel threads take turns instead of overloading the GPU. Chat requests go ahead of
  background writing, and a background request that has waited 30 seconds is treated as chat so
  none waits forever. A waiting thread shows `Waiting for <server> · 2 ahead`. A request still
  queued after 10 minutes, or one arriving when 64 are already queued for that server, fails.
- records each request's prompt, cached prompt and output tokens, queue wait, time to first token
  and decode speed, without the prompt or response text. Records are kept for 30 days.
- stops a streaming reply that has run away, so it no longer holds the GPU: one that keeps
  repeating the same text, one that has reasoned for about 128 KB without answering or calling a
  tool, and one that sends nothing for 5 minutes before its first token or for 2 minutes after.
  The thread then shows why, with **Continue** and **Choose model**. To turn this off for one
  server, add `"watchdog": { "enabled": false }` to that server under `inferenceServers` in `settings.json`.

Pi and OpenCode pass your image attachments to a linked model unless it is known to read text
only, as its engine (Ollama, LM Studio, llama.cpp) reports or T3 Code's check found. To check a
model, press **Check images** next to it in **Settings → Inference**; attaching an image to a model
nobody has checked starts the check on its own. T3 Code asks the model to name the colour of four
test images, through the same queue as your threads, and marks it **Reads images** or **Text
only**. A text-only result applies to sessions that start afterwards. A day later the button
returns as **Check images again**, in case the server now runs different weights under the same
name.

Linked Pi instances also offer Pi a lean set of T3 tools by default; see
[T3 Tools](./providers-pi.md#t3-tools). They also compact long threads from the server's prompt
cache; see [Compaction on Linked Servers](./providers-pi.md#compaction-on-linked-servers).

### See what your hardware did

The **Local compute** section of **Usage → Cost** and **Usage → Tokens** totals those records by server, model and purpose (chat or background
writing) for the last day or week: requests, failures, tokens, the share of the prompt served
from cache, and median and 95th percentile queue wait, time to first token and decode speed. The
cached share counts only requests whose server reported cache use, and shows `–` when none did.
Usage opens on **Tokens** when you have an enabled inference server, until you pick a tab
yourself. Local models show **Local** instead of a price, even when a cloud API sells a model with
the same name, and never add to the cost total. Set a price for one in Usage to see an estimated
equivalent API spend on its row.

In a thread, the details panel's **Last request** row shows the most recent request's prompt
size, cached share and time to first token. Its tooltip names the server that answered,
including when a fallback did, and notes when the instructions or tools changed or a different
server answered than for the previous request. Compare those notes with the cached share when a
request starts slowly.

## Choose a provider

| Provider | Configure the endpoint in                                                           | Needs                               |
| -------- | ----------------------------------------------------------------------------------- | ----------------------------------- |
| Pi       | Pi's `~/.pi/agent/models.json`                                                      | OpenAI Chat Completions             |
| OpenCode | OpenCode's `opencode.json`, or a link to a registered server                        | OpenAI Chat Completions             |
| Claude   | The instance's **Environment variables** in T3 Code                                 | Anthropic Messages (`/v1/messages`) |
| Codex    | Codex's `config.toml`, with the instance's setup set to use your existing Codex CLI | OpenAI Responses (`/v1/responses`)  |

Pi and OpenCode work with the widest range of servers. Use the model's exact ID
as your server reports it at `/v1/models`, and give the provider the context
window you actually deployed so long threads compact in time.

### Pi

Add your server as a provider in `~/.pi/agent/models.json`:

```json
{
  "providers": {
    "local": {
      "name": "Local",
      "baseUrl": "http://localhost:8080/v1",
      "api": "openai-completions",
      "apiKey": "local",
      "models": [{ "id": "your-model-id", "contextWindow": 131072, "maxTokens": 32768 }]
    }
  }
}
```

Set it as Pi's default with `defaultProvider` and `defaultModel` in
`~/.pi/agent/settings.json`, enable Pi in **Settings → Providers**, and refresh
it. See [Pi](./providers-pi.md). To skip this file, register the server in **Settings →
Inference** and link the Pi instance to it instead; see
[Pi on a registered server](#pi-on-a-registered-server).

### OpenCode

Add an OpenAI-compatible provider to `opencode.json`:

```json
{
  "provider": {
    "local": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Local",
      "options": { "baseURL": "http://localhost:8080/v1" },
      "models": { "your-model-id": { "limit": { "context": 131072, "output": 32768 } } }
    }
  }
}
```

OpenCode's **Server URL** setting in T3 Code is for an OpenCode server, not your
inference server. Leave it empty. See [OpenCode](./providers-opencode.md).

To skip this file, register the server in **Settings → Inference** and link the OpenCode
instance to it. T3 Code then adds the server to OpenCode as `t3-<server>` with every model it
reports, sized from what the server advertises, and sends its requests through the gateway, like
[Pi on a registered server](#pi-on-a-registered-server). Pick a model such as
`t3-rtx/your-model-id`. Requests are counted per thread, except OpenCode's own title requests,
which count toward the instance. This needs the server to serve OpenAI Chat
Completions, and does not apply when **Server URL** points at an OpenCode server you run yourself;
the provider's status says when the link is not used and why.

### Claude

Add a Claude instance with its own config directory, then set these
**Environment variables** on it:

| Variable               | Value                                               |
| ---------------------- | --------------------------------------------------- |
| `ANTHROPIC_BASE_URL`   | Your server's root, such as `http://localhost:8080` |
| `ANTHROPIC_AUTH_TOKEN` | Any value your server accepts                       |
| `ANTHROPIC_API_KEY`    | An explicitly empty value                           |

Add your model's ID with **Add custom model** and select it in the chat model
picker. The built-in Claude models stay listed but your server will not serve
them. See [Claude](./providers-claude.md#other-routers).

### Codex

Codex needs a server with the Responses API. Define it in Codex's `config.toml`:

```toml
model = "your-model-id"
model_provider = "local"

[model_providers.local]
name = "Local"
base_url = "http://localhost:8080/v1"
wire_api = "responses"
```

Use a Codex instance set up with your existing Codex CLI, not a ChatGPT
connection: a ChatGPT connection always uses OpenAI. To keep your main Codex
config on the cloud, put the same settings in the instance's launch arguments
instead, such as `-c model_provider="ollama" -c model="your-model-id"` for
Codex's built-in Ollama or `lmstudio` providers. Use `-c` overrides rather than
`--oss`: chat runs `codex app-server`, which accepts only `-c`.

Linking a Codex instance to a registered server that serves the Responses API does this for you,
and is **experimental**. T3 Code starts Codex with a `t3-<server>` provider that goes through the
gateway, the model picker offers only that server's models, and Codex's multi-agent and web search
tools are turned off. Codex sends T3 Code's own tools as a tool group that some servers reject
("function tools only"); on such a server, threads fail until it accepts them. While the linked
server is down or cannot serve the Responses API, the instance lists no models and its threads
wait: Codex never falls back to your OpenAI login. Disable the server to use that login again.

## Mark the instance as local

An instance linked to an enabled inference server already counts as local. For any other
instance, in **Settings → Providers**, turn on **Runs local models** for each instance whose
models all run on hardware you control. Leave it off for an instance that mixes
local and cloud models, and split those into separate instances instead.

Marked instances:

- come first in the model picker and in provider lists;
- become the default for new threads when no default model is set;
- write thread titles, commit messages, pull request text, branch names, and voice
  transcript cleanup, until you choose a text generation model yourself. Resetting
  that setting goes back to the local model.

## Background writing fallback

**Settings → Threads → Text generation fallback** decides where background writing
goes when its model cannot run:

- **Local models only** (default): another ready local instance, or an error. Your
  diffs and messages never move to a cloud provider on their own.
- **Any provider**: local instances first, then any signed-in provider.
- **Don't fall back**: report the error.

The fallback never replaces a model you selected that is working. With **Local
models only** and no local model set up yet, background writing pauses: threads keep
their first message as the title, and you write commit and pull request text yourself.

Pull request review guides need an agent that can investigate the checkout, which
means Codex or Claude. To keep them local, point a Claude instance at your server
and mark it local.

## Voice input

Dictation already runs on your own machine. See [Voice input](./voice-input.md).

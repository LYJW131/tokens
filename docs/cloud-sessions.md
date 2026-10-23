# Cloud agent sessions

Claude Code on the web runs the agent in a cloud VM, not on your machine, so
`tokens serve` never sees that usage. `tokens cloud submit` reports it from
inside the VM, at the end of each turn.

Codex cloud is not supported: in its current environments the agent does not run
in the container, so there is nothing there to report (see
[Codex cloud](#codex-cloud)).

## What is sent, and what is not

Exactly what `tokens submit` sends — daily token totals, model names, client
names, cost — with three differences:

- **Device identity is the session, hashed.** The device id is
  `cloud_<provider>_<32 hex>`, a domain-separated SHA-256 of the session id. The
  session id itself never leaves the container. A session that resumes in a new
  container lands on the same device, so its usage is merged, not counted twice;
  separate sessions add up.
- **No name from the container.** The device is labelled "Claude Code cloud" or
  "Codex cloud" by the server. The container's hostname is never sent, and the
  name in the payload is ignored for cloud devices.
- **No MCP server list.** The container's MCP config is the repository's, so a
  cloud submit leaves your account's list unchanged.

Hook payloads (Claude Code's stdin JSON, Codex's `notify` argument) carry your
prompt and transcript path. Only the session id field is parsed; nothing else
in them is read.

`tokens cloud status` shows the provider, where the session id came from, and
the hashed device id — without submitting. `tokens cloud submit --dry-run`
shows the totals.

## Anti-cheat

On top of the usual checks (cross-device duplicate reclaim, per-day regression
guard, bans), the server enforces for every `cloud_` device:

- the key must be well-formed and name a supported provider;
- it may only report its own agent's client — a Claude Code cloud session
  reports `claude`, a Codex cloud session `codex`. A container cannot be used to
  replay a laptop's history for other clients under a fresh device;
- no imported (`backfill`) history and no `--replace` coverage;
- it never adopts the account's legacy pre-device history.

The CLI refuses to submit a session it cannot name rather than inventing a
random id, since a random id is exactly what would double-count a resumed
session.

On public profiles all of a provider's sessions are shown as one row
("Claude Code cloud · 12 sessions").

## Claude Code on the web

All of this is set in the cloud environment's settings (the gear beside the
environment in the environment menu), except the hooks file in option B below.

### 1. API token: network secret, not an environment variable

Create a token in [Settings](https://tokens.ci/settings). Prefer to give it to
the environment's egress proxy rather than to the container:

- **Recommended: Network secrets** (labelled API credentials in older apps).
  Add a secret for host `tokens.ci` that sends
  `Authorization: Bearer <your token>`. The proxy adds it to requests to
  tokens.ci on their way out, so the token is never inside the container, where
  the agent, the repository's scripts and anything they install could read it.
  Then add the environment variable `TOKENS_API_TOKEN=injected-by-proxy`: the
  CLI refuses to submit without a token, and this placeholder is the
  `Authorization` header the proxy replaces with the real one.
- **Otherwise: environment variable.** Set `TOKENS_API_TOKEN` to the token
  itself. It works the same, but every process in the session can read it.

Also add `TZ=<your IANA zone>` (e.g. `TZ=Asia/Shanghai`) so days are bucketed in
your timezone rather than the container's UTC.

### 2. Network access

Allow `tokens.ci` and the npm registry (to install the CLI). Also allow
`models.dev` and `openrouter.ai`: without them the pricing fetch fails and costs
are computed from the remaining pricing sources only.

### 3. Hooks

The hooks are `async`, so a submit runs in the background and never holds up the
session, and each has a `timeout` (seconds) so a stalled install or network call
is killed rather than left running. Pick one of two ways to install them.

**A. Setup script: every session in this environment.** Paste this into the
environment's **Setup script**. It installs the CLI and adds the Stop hook to
the container's user settings (`~/.claude/settings.json`), merged with whatever
is already there, so it covers every repository you open in the environment and
nothing is committed. The setup script cannot read the environment's variables
or secrets, and does not need to.

```bash
#!/bin/bash
npm i -g tokens-cli@latest
mkdir -p ~/.claude
node -e '
const fs = require("fs");
const file = process.env.HOME + "/.claude/settings.json";
let settings = {};
try { settings = JSON.parse(fs.readFileSync(file, "utf8")); } catch {}
const command = "tokens cloud submit >/dev/null 2>&1; true";
const hooks = (settings.hooks ??= {});
const stop = (hooks.Stop ??= []);
if (!JSON.stringify(stop).includes("tokens cloud submit")) {
  stop.push({ hooks: [{ type: "command", command, async: true, timeout: 120 }] });
}
fs.writeFileSync(file, JSON.stringify(settings, null, 2) + "\n");
'
```

**B. Hooks file in the repository: every session on this repository.** Commit
this to the repository's `.claude/settings.json` (merge it in if the file
exists). It applies to that repository in any environment, and the SessionStart
hook installs the CLI itself, so no setup script is needed.

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "[ \"$CLAUDE_CODE_REMOTE\" = true ] && ! command -v tokens >/dev/null && npm i -g tokens-cli@latest >/dev/null 2>&1; true",
            "async": true,
            "timeout": 180
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "command -v tokens >/dev/null && tokens cloud submit >/dev/null 2>&1; true",
            "async": true,
            "timeout": 120
          }
        ]
      }
    ]
  }
}
```

Use one of the two, not both, or every turn submits twice (harmless, since a
submit replaces the session's stored total, but wasted).

The provider and session are detected from `CLAUDE_CODE_REMOTE` and
`CLAUDE_CODE_REMOTE_SESSION_ID`. On a local machine `tokens cloud submit` finds
no cloud session and does nothing, so the committed hooks are safe for
teammates who work locally. Because the install hook runs in the background,
the first turn of a fresh session can end before the CLI is installed; that
turn's usage is still reported, by the next turn's submit, since every submit
carries the session's whole total.

`tokens cloud status` in the session shows whether it is detected and which
device it submits as.

### Submit frequency

Every Stop hook submits, one request per turn. There is no reliable hook for
the end of a cloud session, so a turn that is skipped may never be caught up.
`--min-interval MINUTES` throttles submits to one per session every MINUTES if
you prefer fewer requests, at that cost: a turn that ends inside the window is
reported only by the session's next submit, and the usage of a session's last
turns is lost if the session ends within the window of its previous submit.

### Resumed sessions

An idle session's VM is paused with its files saved, and the next message
restores the same VM, transcript included; the setup script does not run again.
The next submit carries the whole session's total, which replaces the stored
copy on the same device.

If the paused VM has been reclaimed, reopening the session provisions a fresh VM
with the conversation restored. `CLAUDE_CODE_REMOTE_SESSION_ID` is the session's
id, so the device is the same. If the restored transcript is incomplete, the
fresh VM submits a smaller total than the one stored, and the per-day regression
guard keeps the stored copy: usage is undercounted until the new VM's total
overtakes it, never counted twice.

## Codex cloud

Not supported. In Codex cloud environments (Settings › Codex cloud in ChatGPT)
the agent loop runs on OpenAI's side; the environment's container only runs
`codex exec-server --remote …` to execute the agent's commands. So in the
container:

- there are no rollout files (`rollout-*.jsonl`) anywhere, and `CODEX_HOME`
  (`/run/codex-environment/codex-home`) holds only `auth.json` — the CLI has no
  Codex usage to parse;
- Codex's `notify` program is never run, since it is invoked by the agent loop;
- commands run with network disabled by default (`CODEX_SANDBOX_NETWORK_DISABLED=1`).

`CODEX_THREAD_ID` and environment variables (including secrets) are available to
the agent's commands, so a session could be named, but its usage would have to
come from somewhere other than the container. The `codex` provider stays in the
CLI and the server for that case.

Legacy Codex cloud environments (Settings › Codex cloud (legacy)), still used
for code review and the Linear and GitHub integrations, were not tested.

## Other cloud agents

Any agent whose sessions the CLI already parses can report the same way once
its provider is added to `CloudProvider` (cli/tokens-cli/src/cloud.rs) and
`CLOUD_DEVICE_PROVIDERS` (web/src/lib/devices/cloud.ts).

## Tested

End to end on 2026-10-13, Claude Code 2.1.42 on the web, against a local server
reached through a tunnel, with the hooks above and `--min-interval 0` (the
default since):

- the Stop hook submits at the end of every turn; the device key matches
  `tokens cloud status`; the stored device name is empty (labelled by the
  server) and the account's MCP list is untouched;
- turns of one session update one device to the session's running total
  (576,770 → 841,408 → 1,437,468 → 1,848,481), never a sum of submits,
  including after the VM was paused for 30 minutes and restored;
- a second session is a second device and the account total is the sum; the
  profile shows one "Claude Code cloud · 2 sessions" row;
- `--min-interval 5` (then the default; the default is now 0) skips a submit 2
  minutes after the previous one;
- the server rejects a cloud device reporting another client, a malformed or
  unknown-provider key, and backfill provenance; a smaller same-day resubmit
  is preserved with a warning.

Not observed: a session resumed in a fresh VM after its VM was reclaimed, which
cannot be triggered on demand.

## Known limitation

If you pull a cloud session down to your machine (`claude --teleport`), its
transcript is copied into your local `~/.claude/projects`, and your local
`tokens serve` counts it again under your machine. The cross-device reclaim
only removes copies whose whole client-day matches exactly, so a day that mixes
local work with a teleported session is counted on both devices.

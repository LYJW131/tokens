# Cloud agent sessions

Claude Code on the web and Codex cloud run the agent in a throwaway container,
not on your machine, so `tokens serve` never sees that usage. `tokens cloud
submit` reports it from inside the container, at the end of each turn.

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

1. Create a token in [Settings](https://tokens.ci/settings) and add it to the
   cloud environment's **environment variables** as `TOKENS_API_TOKEN`. Add
   `TZ=<your IANA zone>` (e.g. `TZ=Asia/Shanghai`) so days are bucketed in your
   timezone rather than the container's UTC.
2. Allow `tokens.ci` (and the npm registry, to install the CLI) in the
   environment's network access settings.
3. Commit these hooks to the repository's `.claude/settings.json`:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "[ \"$CLAUDE_CODE_REMOTE\" = true ] && ! command -v tokens >/dev/null && npm i -g tokens-cli@latest >/dev/null 2>&1; true"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "command -v tokens >/dev/null && tokens cloud submit >/dev/null 2>&1; true"
          }
        ]
      }
    ]
  }
}
```

The provider and session are detected from `CLAUDE_CODE_REMOTE` and
`CLAUDE_CODE_REMOTE_SESSION_ID`. On a local machine `tokens cloud submit` finds
no cloud session and does nothing, so the hooks are safe for teammates who work
locally. Submits are throttled to one per session every 5 minutes
(`--min-interval`); a missed final turn is caught up by the session's next
submit, including after a resume.

## Codex cloud

Codex cloud exposes secrets only to the setup script, and has no end-of-task
hook, so the setup script saves the token and registers `tokens` as Codex's
`notify` program, which Codex runs after every turn with a JSON payload whose
`thread-id` keys the device:

```sh
npm i -g tokens-cli@latest
tokens login --token "$TOKENS_API_TOKEN"   # TOKENS_API_TOKEN stored as a secret
mkdir -p ~/.codex
echo 'notify = ["tokens", "cloud", "submit", "--provider", "codex"]' >> ~/.codex/config.toml
```

The agent phase needs internet access to `tokens.ci`. This path depends on Codex
cloud keeping its rollout files under `~/.codex/sessions` and honouring
`notify`; if your Codex cloud setup does neither, set `TOKENS_CLOUD_PROVIDER=codex`
and `TOKENS_CLOUD_SESSION_ID` yourself and run `tokens cloud submit` at the end
of the task.

## Other cloud agents

Any agent whose sessions the CLI already parses can report the same way once
its provider is added to `CloudProvider` (cli/tokens-cli/src/cloud.rs) and
`CLOUD_DEVICE_PROVIDERS` (web/src/lib/devices/cloud.ts).

## Known limitation

If you pull a cloud session down to your machine (`claude --teleport`), its
transcript is copied into your local `~/.claude/projects`, and your local
`tokens serve` counts it again under your machine. The cross-device reclaim
only removes copies whose whole client-day matches exactly, so a day that mixes
local work with a teleported session is counted on both devices.

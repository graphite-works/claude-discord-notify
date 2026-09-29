# claude-discord-notify

> Send **Discord notifications** from **[Claude Code](https://claude.com/claude-code)** — build results, summaries, screenshots, and "I'm done" pings, posted to a Discord webhook you own.

![Claude Code posting to Discord: test results to the default channel, a screenshot to #work, a silent deploy summary to #alerts, then piped output and a rich embed sent straight from the shell](https://raw.githubusercontent.com/graphite-works/claude-discord-notify/main/docs/demo.gif)

[![CI](https://github.com/graphite-works/claude-discord-notify/actions/workflows/ci.yml/badge.svg)](https://github.com/graphite-works/claude-discord-notify/actions/workflows/ci.yml)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![npm version](https://img.shields.io/npm/v/claude-discord-notify.svg)](https://www.npmjs.com/package/claude-discord-notify)
[![npm downloads](https://img.shields.io/npm/dm/claude-discord-notify.svg)](https://www.npmjs.com/package/claude-discord-notify)
[![Node](https://img.shields.io/badge/node-%3E%3D18-brightgreen.svg)](https://nodejs.org)
[![Dependencies](https://img.shields.io/badge/dependencies-0-brightgreen.svg)](package.json)

**claude-discord-notify** is a [Claude Code](https://claude.com/claude-code) skill and zero-dependency Node.js command-line tool that sends Discord notifications from your terminal via a Discord webhook — plain messages, file attachments, and rich embeds.

Long-running work in Claude Code means watching a terminal. This skill removes the watching: you tell Claude *"notify me on Discord when the tests pass"* and the message arrives in your channel. Install is one interactive command, and the only thing you provide is a Discord webhook URL.

It is deliberately small — no dependencies, no daemon, no account. A single ~680-line Node script posts to a webhook you own, with a `chmod 600` config file next to it.

## ✨ Features

- **Zero dependencies** — pure Node.js ≥18, using the built-in `fetch`, `FormData`, and `Blob`. Nothing to audit but the script itself.
- **One-command interactive install** — `npx claude-discord-notify` prompts for your webhook, installs the skill, and offers a test message.
- **Natural-language usage** — once installed you never touch the CLI; you just tell Claude what to send.
- **File attachments** — up to 10 per message, from a local path or an `http(s)` URL, with MIME detection so Discord renders images, video, audio, and PDFs inline.
- **Rich embeds** — title, description, accent color, and up to 25 inline/block fields, with every one of Discord's limits validated locally before a byte goes out.
- **Multiple channels** — configure named webhooks and target them with `--to work`, keeping a default for everything else.
- **Safe by default** — `@everyone`/`@here`/role/user mentions in message text never ping anyone, and `--quiet` suppresses the push notification for low-priority updates.
- **`--dry-run` previews** — runs every validation a real send performs and prints exactly what would be posted, without posting it.
- **Resilient sending** — HTTP 429 and 5xx are retried with backoff that honors Discord's own `retry_after`; messages over 2000 characters are split on line boundaries automatically.
- **Tested** — 174 tests across 36 suites, run on Node 18, 20, and 22 in CI.

## 📋 Table of Contents

- [Installation](#-installation)
- [Quick Start](#-quick-start)
- [Usage](#-usage)
  - [Talking to Claude](#talking-to-claude)
  - [CLI reference](#cli-reference)
  - [Named webhooks (`--to`)](#named-webhooks---to)
  - [Quiet sends (`--quiet`)](#quiet-sends---quiet)
  - [Attachments (`--file`)](#attachments---file)
  - [Rich embeds (`--embed-*`)](#rich-embeds---embed-)
  - [Previewing (`--dry-run`)](#previewing---dry-run)
- [Configuration](#️-configuration)
- [Examples](#-examples)
  - [Notify when a test run finishes](#notify-when-a-test-run-finishes)
  - [Wrap any long-running command](#wrap-any-long-running-command)
  - [Alert on a failed deploy](#alert-on-a-failed-deploy)
  - [Ship a release notification](#ship-a-release-notification)
  - [Run a nightly job and report quietly](#run-a-nightly-job-and-report-quietly)
  - [Auto-notify when Claude Code finishes a turn](#auto-notify-when-claude-code-finishes-a-turn)
- [Limits](#-limits)
- [How It Works](#-how-it-works)
- [Development](#️-development)
- [Testing](#-testing)
- [Troubleshooting](#-troubleshooting)
- [Security](#-security)
- [Contributing](#-contributing)
- [License](#-license)
- [Support](#-support)
- [Related Tools](#-related-tools)

## 🚀 Installation

### Prerequisites

- **Node.js 18 or higher** — required for the built-in `fetch`, `FormData`, and `Blob` globals. Check with `node --version`.
- **[Claude Code](https://claude.com/claude-code)** — the skill installs into `~/.claude/skills/`.
- **A Discord webhook URL** — see below.

### Install

```bash
npx claude-discord-notify
```

The installer will:

1. Ask for your Discord webhook URL (this becomes the default target).
2. Optionally let you add or update **named webhooks** — extra Discord channels you can target by name with `--to <name>` (e.g. a `work` channel alongside your default).
3. Install the skill into `~/.claude/skills/discord-notify/`.
4. Save your webhook(s) to a `chmod 600` `config.json`.
5. Optionally send a test message (to the default webhook only).

Re-run the same command any time to update the webhook, or to add/update named webhooks. Re-running shows any existing named webhooks (masked) and leaves the ones you don't touch untouched.

### Getting a webhook URL

In Discord: **Server Settings → Integrations → Webhooks → New Webhook → Copy Webhook URL**.

It looks like `https://discord.com/api/webhooks/<id>/<token>`. The installer validates the format and accepts `discord.com`, `discordapp.com`, and the `canary.`/`ptb.` subdomains, with or without an API version segment.

### From source

```bash
git clone https://github.com/graphite-works/claude-discord-notify.git
cd claude-discord-notify
node bin/cli.js
```

There is no build step and nothing to install first — the package has no dependencies at all, runtime or dev.

## ⚡ Quick Start

Install, then just talk to Claude Code:

> **You:** run the test suite and notify me on discord when it's done

Claude runs your tests and posts the result to your channel. Nothing else to configure.

To confirm the plumbing yourself:

```bash
node ~/.claude/skills/discord-notify/discord_send.js "hello from Claude Code"
```

**Output:**

```
✅ sent message to Discord
```

## 📖 Usage

### Talking to Claude

Once installed, this is the whole interface:

> "notify me on discord when the tests pass"
>
> "send that summary to my discord"
>
> "post the build results as a status card"
>
> "send me that screenshot on Discord"

Claude picks the flags — you don't supply file paths or webhook names yourself. Everything below documents what Claude runs under the hood, and is equally usable by hand or from a script.

### CLI reference

```bash
node ~/.claude/skills/discord-notify/discord_send.js [flags] [message text...]
```

Message text is every argument that isn't a flag or a flag value, joined with spaces. If no text, no attachment, and no embed is given, the message is read from **stdin**:

```bash
some-command | node ~/.claude/skills/discord-notify/discord_send.js
```

| Flag | Value | Description |
|---|---|---|
| `--to <name>` | webhook name | Send to a named webhook from `config.json` instead of the default |
| `--username <name>` | text | Override the sender name shown in Discord |
| `--file <path\|url>` | local path or `http(s)` URL | Attach a file. Repeatable, up to 10. Alias: `--attach` |
| `--quiet` | *(boolean)* | Suppress Discord's client-side push/desktop notification. Alias: `--silent` |
| `--dry-run` | *(boolean)* | Validate and print what would be sent, without posting |
| `--embed-title <text>` | ≤256 chars | Embed title |
| `--embed-description <text>` | ≤4096 chars | Embed description |
| `--embed-color <value>` | `#RRGGBB`, `RRGGBB`, or `0`–`16777215` | Embed accent color bar |
| `--embed-field <name=value>` | repeatable | Full-width ("block") embed field |
| `--embed-field-inline <name=value>` | repeatable | Inline embed field (sits side-by-side) |
| `--help` | *(boolean)* | Print usage and exit `0`. Alias: `-h` |
| `--` | *(separator)* | Stop parsing flags; every argument after it is message text |

Flags may appear in any order and compose freely with each other. Run `--help` for
the same reference at the terminal.

An unrecognized `--flag` is rejected rather than treated as message text, so a typo
fails locally instead of being delivered to your channel:

```console
$ node ~/.claude/skills/discord-notify/discord_send.js --fiel shot.png "oops"
error: unknown flag: --fiel
  run with --help to see the available flags
  if that is message text, put it after --: discord_send.js -- "--fiel"
```

To send message text that genuinely begins with dashes, put it after `--`:

```bash
node ~/.claude/skills/discord-notify/discord_send.js -- "--- release notes ---"
```

A single-dash argument (`-3 degrees overnight`, a markdown bullet) is still ordinary
message text — only `--`-prefixed arguments are checked against the flag list.

**Exit codes:** `0` success · `1` any validation or send failure (message on stderr, prefixed `error:`) · `2` nothing to send (no text, no attachment, no embed).

### Named webhooks (`--to`)

If you configured extra named webhooks during install, target one with `--to`:

```bash
node ~/.claude/skills/discord-notify/discord_send.js --to work "deploy finished"
```

Omitting `--to` always sends to the default `webhookUrl`. `--to <name>` with a name that isn't configured fails locally (no message is sent) with an error listing the names that *are* available.

### Quiet sends (`--quiet`)

Pass `--quiet` (or its alias `--silent`) to suppress Discord's client-side push/desktop notification for the message:

```bash
node ~/.claude/skills/discord-notify/discord_send.js --quiet "background sync finished"
```

The message still posts and is fully visible in the channel — only the notification ping to the client is suppressed. This is separate from (and composes with) the always-on mention guard: `allowed_mentions` blocks pings caused by `@everyone`/`@here`/role/user mentions written in the message *text*, while `--quiet`/`--silent` suppresses the generic new-message notification regardless of what the text contains. Use `--quiet` for routine or low-priority updates you don't want to interrupt someone with, even when the message contains no mentions at all.

### Attachments (`--file`)

Attach a file (image, log, screenshot, PDF, …) with `--file` — repeat it for up to 10 files, and pass either a local path or an `http(s)` URL:

```bash
node ~/.claude/skills/discord-notify/discord_send.js --file shot.png "here's the screenshot"
node ~/.claude/skills/discord-notify/discord_send.js --file a.png --file build.log "results"
node ~/.claude/skills/discord-notify/discord_send.js --file https://example.com/img.png "remote image"
```

In conversation you don't supply paths yourself — just say *"send me that image on Discord"* and Claude fills in the path to the file it's working with.

- URLs are fetched and verified to actually load before anything is uploaded; a URL or path that can't be read fails the send with a non-zero exit and nothing is posted.
- A message is optional when attaching files — `--file shot.png` with no text is a valid send. With text, the message is posted first and files follow on the final chunk.
- Combined attachment size across all `--file` values is capped at **8MB** (Discord's default non-boosted webhook limit), checked locally after loading and before any upload attempt.

Recognized file types (assigned the right MIME so Discord renders them inline):

| Kind    | Extensions                                  |
|---------|---------------------------------------------|
| Image   | `png` `jpg`/`jpeg` `gif` `webp` `bmp` `svg` |
| Video   | `mp4` `webm` `mov`                          |
| Audio   | `mp3` `wav` `ogg`                           |
| Docs    | `pdf` `txt` `log` `md` `csv` `html` `json`  |
| Archive | `zip`                                       |

Any other extension still uploads fine — it just arrives as `application/octet-stream`, a generic download. Discord previews most of the above inline; a few (e.g. `bmp`, `svg`) may show as a download depending on the client.

### Rich embeds (`--embed-*`)

Attach one rich, structured embed (title, description, accent color, and up to 25 name/value fields) alongside — or instead of — plain message text:

```bash
node ~/.claude/skills/discord-notify/discord_send.js \
  --embed-title "Build passed" \
  --embed-description "All 42 tests green" \
  --embed-color "#5865F2" \
  --embed-field "Duration=3m12s" \
  --embed-field-inline "Branch=main" \
  "deploy finished"
```

- **Exactly one embed per message** is supported. `--embed-field` and `--embed-field-inline` may each be repeated (up to 25 fields total, combined) and render in the order the flags are given on the command line.
- `--embed-color` checks the hex format first: a 6-character value made only of digits (e.g. `123456`) is read as **hex**, not decimal — pass `#`-prefixed or non-numeric-looking hex to be unambiguous if you mean a specific hex color.
- `--embed-field`/`--embed-field-inline` split on the **first** `=` only, so the name can't contain `=` but the value can (`"Formula=x=y+z"` → name `Formula`, value `x=y+z`).
- An embed with no message text is a valid send on its own — none of the embed flags require accompanying text, and text doesn't require an embed.
- With a multi-chunk message (over Discord's 2000-character limit), the embed is attached to the **last** chunk only, exactly like `--file`.
- Discord's documented per-embed limits are enforced locally, before any network request: title ≤256 chars, description ≤4096 chars, ≤25 fields, field name ≤256 chars, field value ≤1024 chars, and the combined total of title + description + all field names + all field values ≤6000 chars. A violation fails immediately with a specific error naming the limit that was exceeded — nothing is sent.
- `--dry-run` previews the embed exactly as it would be sent: title, full description, color shown as both hex and decimal (e.g. `#5865F2 (5793266)`), and every field's name, value, and inline state.
- **Not supported** (deliberately out of scope for this skill): embed URL, footer, image/thumbnail, author, timestamp, or multiple embeds per message — there are no `--embed-url` / `--embed-footer` / `--embed-image` / `--embed-thumbnail` / `--embed-author` / `--embed-timestamp` flags.

### Previewing (`--dry-run`)

Pass `--dry-run` to see exactly what *would* be sent — every message chunk (with its character count and full text), every attachment (name, MIME type, size, and whether it came from a local path or a URL), the combined attachment size vs. the 8MB limit, the embed if any, and whether `--quiet`/`--silent` would be applied — without actually posting anything to Discord:

```bash
node ~/.claude/skills/discord-notify/discord_send.js --dry-run "preview this message"
node ~/.claude/skills/discord-notify/discord_send.js --dry-run --file shot.png "with an attachment"
```

Two things worth knowing:

- **Every validation a real send performs still runs.** Flag parsing, embed limit checks, the 10-attachment count check, webhook resolution (`--to`), loading every `--file`, and the 8MB combined-size check all happen exactly as they would for a real send — only the final POST(s) to Discord are skipped. A `--dry-run` that exits 0 means a real send would succeed; the same errors and exit codes surface at the same points if something's wrong.
- **`--file` URLs are still fetched over the network.** An `http(s)://` attachment reference is downloaded for real even under `--dry-run` — that's needed to report its actual size and MIME type and to exercise the size-limit check meaningfully. Only local paths avoid any I/O beyond a local file read either way.

The webhook URL itself is never printed unmasked — the report shows only the `--to` name (or `default`) plus a masked form of the URL (e.g. `.../AbCd…z890`), the same masking the installer uses.

## ⚙️ Configuration

All configuration lives in a single file written by the installer, next to the sender script.

| Path | Purpose |
|---|---|
| `~/.claude/skills/discord-notify/SKILL.md` | Skill definition Claude Code reads to know when and how to use this |
| `~/.claude/skills/discord-notify/discord_send.js` | The sender |
| `~/.claude/skills/discord-notify/config.json` | Your webhook(s) — `chmod 600` |

### `config.json` schema

```json
{
  "webhookUrl": "https://discord.com/api/webhooks/111/aaa",
  "webhooks": {
    "work": "https://discord.com/api/webhooks/222/bbb"
  }
}
```

| Key | Type | Required | Description |
|---|---|---|---|
| `webhookUrl` | `string` | Yes | The default target, used whenever `--to` is omitted |
| `webhooks` | `object` | No | Map of name → webhook URL for extra channels, targetable with `--to <name>` |

`webhooks` is omitted entirely (not even `{}`) when you haven't configured any named webhooks, so a config with just `webhookUrl` keeps working exactly as before. Webhook names must match `[A-Za-z0-9_-]+`.

Edit the file directly, or re-run `npx claude-discord-notify` to update it interactively.

There are **no environment variables** — the webhook is read from `config.json` only, so it never appears in your shell history or process listings.

## 💡 Examples

These are complete workflows rather than flag demos. They all assume this helper,
which just saves repeating the path:

```bash
notify() { node ~/.claude/skills/discord-notify/discord_send.js "$@"; }
```

Put it in your `~/.bashrc` or `~/.zshrc`. Every example works without it — substitute
the full path.

### Notify when a test run finishes

The workflow from the demo at the top of this README: report the counts on success,
attach the whole log on failure.

```bash
if npm test > /tmp/test.log 2>&1; then
  notify "tests passed — $(awk '$2=="pass"{print $3}' /tmp/test.log)/$(awk '$2=="tests"{print $3}' /tmp/test.log) green"
else
  notify --file /tmp/test.log \
    "tests FAILED — $(awk '$2=="fail"{print $3}' /tmp/test.log) failing, log attached"
fi
```

The `awk` lookups read `node --test`'s summary lines (`ℹ tests 174`, `ℹ pass 174`,
`ℹ fail 0`); adjust the field names for another runner. Attaching the log on the
failure branch is the point — it makes a red build diagnosable from your phone
without opening a laptop.

### Wrap any long-running command

A general "tell me when this is done" wrapper. It preserves the wrapped command's
exit code, so it composes with `&&` and CI scripts.

```bash
notify-done() {
  local start=$SECONDS
  "$@"
  local code=$?
  local elapsed=$(( SECONDS - start ))
  local took="$(( elapsed / 60 ))m$(( elapsed % 60 ))s"
  if [ $code -eq 0 ]; then
    notify "\`$*\` finished in $took"
  else
    notify "\`$*\` exited $code after $took"
  fi
  return $code
}
```

Then prefix anything slow:

```bash
notify-done npm run build
notify-done make -j8 release
notify-done pytest tests/integration
```

**Output:**

```
`npm run build` finished in 4m12s
```

### Alert on a failed deploy

With no message, no attachment, and no embed, the sender reads the message body from
stdin — so any command's output pipes straight in.

```bash
make deploy 2>&1 | tail -20 | notify --username "Deploy"
```

Output longer than 2000 characters is split across multiple messages on line
boundaries, so a long stack trace arrives intact rather than truncated.

### Ship a release notification

An embed reads better than plain text for structured results, and the fields can be
filled from the repo itself:

```bash
notify \
  --embed-title "Deployed to production" \
  --embed-color "#57F287" \
  --embed-field-inline "Version=$(node -p "require('./package.json').version")" \
  --embed-field-inline "Commit=$(git rev-parse --short HEAD)" \
  --embed-field "Message=$(git log -1 --pretty=%s)"
```

Green accent bar, two side-by-side fields, one full-width field, and no message text —
an embed-only send.

### Run a nightly job and report quietly

`--quiet` delivers the message without triggering a push notification, which is what
you want for a job that succeeds every night. Cron has no shell functions, so this one
spells out the path:

```cron
0 3 * * * cd "$HOME/site" && ./backup.sh > /tmp/backup.log 2>&1; \
  node "$HOME/.claude/skills/discord-notify/discord_send.js" \
    --to work --quiet --file /tmp/backup.log "nightly backup finished"
```

`--to work` targets a named webhook from `config.json`, keeping routine noise out of
your default channel.

### Auto-notify when Claude Code finishes a turn

A `Stop` hook fires when Claude finishes responding, and receives the turn's final
message on stdin as `last_assistant_message`. Forwarding it to Discord means you get
Claude's own summary pushed to your phone without asking for it:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "[ -f ~/.notify-claude ] || exit 0; node -e 'let s=\"\";process.stdin.on(\"data\",d=>s+=d).on(\"end\",()=>process.stdout.write(JSON.parse(s).last_assistant_message||\"done\"))' | node ~/.claude/skills/discord-notify/discord_send.js --username \"Claude Code\""
          }
        ]
      }
    ]
  }
}
```

`Stop` fires on *every* turn, which is noise while you are sitting at the keyboard, so
the command begins with a gate: it exits silently unless `~/.notify-claude` exists.
Run `touch ~/.notify-claude` when you step away and `rm ~/.notify-claude` when you get
back. Put the block in `~/.claude/settings.json` for every project, or in a project's
`.claude/settings.local.json` for just that one.

### Checking a send before committing to it

```bash
node ~/.claude/skills/discord-notify/discord_send.js --dry-run --file ./report.pdf "monthly report"
```

**Output:**

```
DRY RUN — nothing was sent. All validation a real send performs has passed.

target: default (https://discord.com/api/webhooks/111/AbCd…z890)
quiet/silent: off — no flags key would be included
allowed_mentions: { parse: [] } is always included, regardless of --quiet/--silent

message 1/1 — 14 chars:
monthly report

attachments — carried on message 1/1, 1 file:
  - report.pdf (application/pdf, 0.42MB, origin: local)
  combined: 0.42MB / 8.00MB limit

DRY RUN SUMMARY: 1 Discord API call would have been made — none were. Re-run without --dry-run to actually deliver this.
```

## 📏 Limits

Every limit below is enforced locally, before any network request:

| Limit | Value | Behavior when exceeded |
|---|---|---|
| Message length | 2000 characters | Split automatically on newline boundaries |
| Attachments per message | 10 | Fails with exit 1 |
| Combined attachment size | 8MB | Fails with exit 1 (Discord's non-boosted webhook limit) |
| Embeds per message | 1 | Only one embed's worth of flags is parseable |
| Embed title | 256 characters | Fails with exit 1 |
| Embed description | 4096 characters | Fails with exit 1 |
| Embed fields | 25 | Fails with exit 1 |
| Embed field name / value | 256 / 1024 characters | Fails with exit 1 |
| Embed total (title + description + all field names/values) | 6000 characters | Fails with exit 1 |
| Retries on 429/5xx | 5 | Error reports how many retries were spent |

Retry backoff prefers Discord's own `retry_after` (from the JSON body or the `Retry-After` header); when neither is present it falls back to exponential backoff from 500ms, jittered, capped at 30s per wait. Non-retryable failures (a bad webhook, a rejected payload) fail immediately.

## 🔍 How It Works

```
  You ─────► Claude Code ─────► SKILL.md ─────► discord_send.js ─────► Discord webhook
                                (when to use)   (config.json,          (your channel)
                                                 chmod 600)
```

1. **`bin/cli.js`** is the `npx` installer. It validates your webhook URL against a Discord-specific regex, copies `SKILL.md` and `discord_send.js` into `~/.claude/skills/discord-notify/`, and writes `config.json` with mode `0600`.
2. **`skill/SKILL.md`** carries the frontmatter Claude Code matches against your request ("send that to discord", "notify me when done") plus guidance on *which* flags fit which request — when an embed beats plain text, when to ask rather than guess a `--to` target, when a dry run is what you actually wanted.
3. **`skill/discord_send.js`** does the work: parse flags → build and validate the embed (pure, zero I/O) → resolve the webhook → load attachments → check the size cap → chunk the message → POST. Text chunks go first; attachments and the embed ride on the final chunk so they appear after the content.

Every outgoing payload sets `allowed_mentions: { parse: [] }` unconditionally — that guard is not exposed as a flag, so a message can never ping a channel by accident. Requests carry a `claude-discord-notify/1.0` User-Agent, which avoids the Cloudflare bot block (error 1010) that default library user-agents can trigger.

## 🛠️ Development

### Setup

```bash
git clone https://github.com/graphite-works/claude-discord-notify.git
cd claude-discord-notify
npm test      # no install step — the package has no dependencies
```

### Project structure

```
claude-discord-notify/
├── bin/
│   └── cli.js                   # npx installer: prompts, validates, installs, chmod 600
├── skill/
│   ├── SKILL.md                 # skill definition + usage guidance Claude reads
│   └── discord_send.js          # the sender — flags, embeds, attachments, chunking, retry
├── test/
│   ├── cli.test.js              # installer: webhook/name validation, config building, masking
│   └── discord_send.test.js     # sender: chunking, MIME, embeds, dry-run, retry, end-to-end
├── .github/workflows/ci.yml     # npm test on Node 18.x / 20.x / 22.x
├── package.json
├── LICENSE
└── README.md
```

Only `bin/` and `skill/` are published to npm (see the `files` field in `package.json`).

### Conventions

- **ES modules** (`"type": "module"`) throughout — `import`, not `require`.
- **No runtime dependencies.** New functionality should use Node built-ins; adding a dependency is a design decision, not a convenience.
- **Both entry points are dual-mode**: `bin/cli.js` and `skill/discord_send.js` run `main()` only when invoked directly (an `import.meta.url` vs. `process.argv[1]` check) and otherwise export their internals for tests. Keep new logic in small exported functions so it can be unit-tested without spawning a process.
- **Validate locally before sending.** Every Discord limit is checked before a network call, with an error naming the specific limit exceeded. Follow that pattern for anything new.
- **Never print a webhook unmasked.** `mask()` in `bin/cli.js` and `maskWebhookUrl()` in `skill/discord_send.js` are intentionally duplicated so the sender stays standalone.

There is no linter or formatter configured; match the surrounding style (2-space indent, double quotes, semicolons).

## 🧪 Testing

Tests use Node's built-in test runner — no framework, no dependencies.

```bash
npm test                                # run everything
node --test test/cli.test.js            # one file
node --test --test-name-pattern "embed" # filter by name
```

Current suite: **174 tests across 36 suites**, all passing. CI runs them on Node **18.x, 20.x, and 22.x** for every push and pull request against `main`.

Coverage spans unit tests of the pure helpers (chunking, MIME mapping, color/field parsing, embed validation, byte formatting, URL masking, config building) and end-to-end tests that spawn `discord_send.js` as a child process against a local stub server — covering multipart uploads, retry/backoff on 429 and 5xx, `--to` resolution failures, `--quiet` flags, exit codes, and full `--dry-run` output.

### Writing tests

Import the exported internals directly for unit tests:

```javascript
import { test } from "node:test";
import assert from "node:assert/strict";
import { parseEmbedField } from "../skill/discord_send.js";

test("splits on the first = only", () => {
  assert.deepEqual(parseEmbedField("Formula=x=y+z"), { name: "Formula", value: "x=y+z" });
});
```

For anything that sends, spawn the script as a child process pointed at a stub webhook rather than the real Discord API — `test/discord_send.test.js` has the existing patterns.

## 🔧 Troubleshooting

| Message | Cause | Fix |
|---|---|---|
| `no config found at …` | The skill isn't installed, or `config.json` was deleted | Run `npx claude-discord-notify` |
| `config at … is not valid JSON` | Hand-edited config with a syntax error | Fix the JSON, or re-run the installer |
| `config at … is missing "webhookUrl"` | Config exists but has no default webhook | Re-run the installer |
| `no webhook named "x" in config.json` | `--to` name isn't configured | The error lists the available names; use one, or add it via the installer |
| `file not found: …` | Bad `--file` path | Check the path; relative paths resolve against your working directory |
| `attachments too large: …` | Combined attachments over 8MB | Send fewer/smaller files, or split across messages |
| `HTTP 401`/`404` | Webhook was deleted or the token is wrong | Create a new webhook in Discord and re-run the installer |
| `HTTP 429 … (gave up after 5 retries)` | Sustained rate limiting | Send less frequently; the retry budget is already exhausted |
| Exit code 2, `no message or attachment provided` | Nothing to send | Provide text, `--file`, or embed flags |

If the test message during install fails, the webhook URL is the first thing to re-check — the installer validates the *format*, not that the webhook still exists.

## 🔒 Security

- **Your webhook is a secret.** Anyone who has it can post to that channel. It is stored only in your local `config.json`, which is written `chmod 600` and is git-ignored.
- Webhook URLs are **never printed unmasked** — not by the installer, not by `--dry-run`.
- Mentions (`@everyone`, `@here`, role and user mentions) in message content are **suppressed unconditionally**. This is not configurable, by design.
- A webhook grants posting rights to one channel only — it cannot read messages, and it cannot be used to authenticate as you.
- To rotate: delete the webhook in Discord, create a new one, and re-run `npx claude-discord-notify`.

## 🤝 Contributing

Issues and pull requests are welcome at [github.com/graphite-works/claude-discord-notify](https://github.com/graphite-works/claude-discord-notify).

1. Fork the repository and create a branch: `git checkout -b feature/your-feature`
2. Make your changes, keeping to the [conventions](#conventions) above
3. Add tests — both entry points export their internals specifically so new logic can be unit-tested
4. Run `npm test` and make sure all tests pass
5. Commit, push, and open a Pull Request describing what changed and why

Two things worth raising in an issue before building:

- **New dependencies.** Zero dependencies is a feature of this project, not an accident.
- **Scope expansion of the embed flags.** Embed URL/footer/image/thumbnail/author/timestamp and multi-embed support are deliberately out of scope; the current surface is what fits a notification skill.

CI (Node 18/20/22) must pass before a PR can merge.

## 📄 License

[MIT](./LICENSE) © 2026 Blake Daniel

## 🆘 Support

- **Bugs and feature requests** — [GitHub Issues](https://github.com/graphite-works/claude-discord-notify/issues)
- **Skill behavior reference** — [`skill/SKILL.md`](skill/SKILL.md) documents exactly what Claude is told about when and how to use each flag
- **Discord webhook docs** — [Discord Developer Portal: Webhook Resource](https://discord.com/developers/docs/resources/webhook)
- **Claude Code docs** — [claude.com/claude-code](https://claude.com/claude-code)
- **Package page** — [claude-discord-notify on npm](https://www.npmjs.com/package/claude-discord-notify)

## 🔗 Related Tools

- [claude-code-statusline-progress](https://github.com/graphite-works/claude-code-statusline-progress) — a shell status line for Claude Code, with a Claude-powered setup prompt.

---

Made with ❤️ by [Blake Daniel](https://github.com/blakeadaniel)

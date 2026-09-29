# System Architecture

`claude-discord-notify` is a zero-dependency Node.js package that installs a
Claude Code skill capable of posting messages, file attachments, and rich
embeds to a Discord channel through an incoming webhook.

## Overview

There is no server, no database, and no frontend. The system is two small
Node CLIs with **two completely separate lifecycles** that meet only through
one directory on disk:

| Lifecycle | Entry point | Runs | Talks to |
|---|---|---|---|
| **Install time** | [bin/cli.js](../../bin/cli.js) | Once, by the human, via `npx claude-discord-notify` | Terminal (readline), filesystem, Discord (one test POST) |
| **Run time** | [skill/discord_send.js](../../skill/discord_send.js) | Every time Claude sends a notification | Filesystem (config + attachments), Discord webhook API |

The installer's only job is to materialize `~/.claude/skills/discord-notify/`
with three files. The sender's only job is to read one of them and POST. They
never import each other — see [Key Design Decisions](#key-design-decisions).

## Architecture Style

**Two-phase CLI tool with file-based configuration.** Closest named pattern is
a *pipe-and-filter* command-line utility: text/args in → validation pipeline →
HTTP out. There is no long-lived process, no IPC, and no shared runtime state;
all state is the single `config.json` on disk.

## System Components

```mermaid
graph TB
    subgraph dist["Development & Distribution"]
        REPO["GitHub repo<br/>graphite-works/claude-discord-notify"]
        CI["GitHub Actions CI<br/>.github/workflows/ci.yml<br/>node 18.x / 20.x / 22.x"]
        NPM["npm registry<br/>claude-discord-notify"]
    end

    subgraph inst["Install Time — npx claude-discord-notify"]
        CLI["bin/cli.js<br/>interactive installer"]
        PKGSKILL["packaged skill/<br/>SKILL.md + discord_send.js"]
    end

    subgraph home["Installed Skill — ~/.claude/skills/discord-notify/"]
        SKILLMD["SKILL.md<br/>frontmatter + usage docs<br/>read by Claude, not by code"]
        SENDER["discord_send.js<br/>sender CLI"]
        CONFIG[("config.json<br/>chmod 600<br/>webhookUrl + optional webhooks map")]
    end

    subgraph rt["Run Time"]
        CLAUDE["Claude Code<br/>matches skill, shells out to node"]
        STDIN["stdin<br/>piped command output"]
        LOCAL["Local files<br/>--file ./shot.png"]
    end

    subgraph ext["External Services"]
        DISCORD["Discord Webhook API<br/>POST /api/webhooks/:id/:token"]
        REMOTE["Remote https URLs<br/>--file https://..."]
    end

    REPO -->|push / PR| CI
    REPO -->|npm publish| NPM
    NPM -->|npx download| CLI
    CLI -->|reads from package| PKGSKILL
    PKGSKILL -->|copyFileSync| SKILLMD
    PKGSKILL -->|copyFileSync| SENDER
    CLI -->|writeFileSync mode 0600| CONFIG
    CLI -.->|optional test POST| DISCORD

    CLAUDE -->|node discord_send.js ...| SENDER
    STDIN -.->|only when no text/file/embed arg| SENDER
    SENDER -->|readFileSync, same dir| CONFIG
    SENDER -->|readFileSync| LOCAL
    SENDER -->|fetch, even under --dry-run| REMOTE
    SENDER -->|"fetch POST — JSON or multipart"| DISCORD
    SKILLMD -.->|instructs which flags to use| CLAUDE

    style DISCORD fill:#5865F2,stroke:#3c45a5,color:#ffffff
    style REMOTE fill:#fff3e0,stroke:#c2410c,color:#1f2937
    style CONFIG fill:#ffe0b2,stroke:#c2410c,color:#1f2937
    style CLAUDE fill:#e3f2fd,stroke:#1d4ed8,color:#1f2937
    style CI fill:#fff9c4,stroke:#a16207,color:#1f2937
```

**Note on the dotted `SKILL.md → Claude` edge:** `SKILL.md` is never parsed by
any code in this repo. It is a prompt — Claude reads its frontmatter to decide
when the skill applies, and its body to decide which flags to pass. It is
"executed" by a language model, which makes it the one component whose
correctness the test suite cannot assert.

## Data Flow

### Install flow — `npx claude-discord-notify`

```mermaid
sequenceDiagram
    actor User
    participant CLI as bin/cli.js
    participant FS as ~/.claude/skills/discord-notify/
    participant Discord as Discord Webhook API

    User->>CLI: npx claude-discord-notify
    CLI->>FS: read config.json (may not exist)
    alt config exists
        FS-->>CLI: { webhookUrl, webhooks? }
        CLI->>User: show masked existing webhook
    end

    loop until WEBHOOK_RE matches (or Enter keeps existing)
        CLI->>User: "Discord webhook URL: "
        User-->>CLI: pasted URL
    end

    CLI->>User: "Add or update a named webhook for --to? [y/N]"
    opt user answers y
        loop per named webhook
            CLI->>User: name (NAME_RE) then URL (WEBHOOK_RE)
            User-->>CLI: name, url
            CLI->>CLI: webhooks[name] = url
        end
    end

    CLI->>FS: mkdirSync recursive
    CLI->>FS: copyFileSync SKILL.md, discord_send.js
    CLI->>FS: writeFileSync config.json, mode 0600
    CLI->>FS: chmodSync 0600 (in case the file pre-existed)

    CLI->>User: "Send a test message now? [Y/n]"
    opt yes (the default)
        CLI->>Discord: POST { content: "👋 discord-notify is set up..." }
        alt 2xx
            Discord-->>CLI: 204
            CLI->>User: ✅ test message sent
        else non-2xx
            Discord-->>CLI: HTTP error
            CLI->>User: ⚠️ test send failed — re-run the installer
        end
    end
```

Re-running the installer is the documented way to change a webhook: existing
named webhooks are read back in and merged, and pressing Enter at the first
prompt keeps the current default. Skill files are always overwritten with the
packaged copies, so a re-run doubles as an upgrade.

### Send flow — `node discord_send.js ...`

```mermaid
sequenceDiagram
    participant Claude as Claude Code
    participant Send as discord_send.js
    participant Cfg as config.json
    participant Files as Local FS / remote URLs
    participant Discord as Discord Webhook API

    Claude->>Send: node discord_send.js --file a.png --embed-title "Build passed" "done"

    Send->>Send: parse flags — any order; unknown --flags fail, rest is message text
    Send->>Send: fileRefs.length > 10 → fail(1)
    Send->>Send: buildEmbed() — pure, zero I/O, validates Discord embed limits
    opt no text, no files, no embed
        Send->>Send: read stdin — still empty → exit 2
    end

    Send->>Cfg: loadWebhook(--to)
    Cfg-->>Send: webhookUrl (default or named)

    loop each --file ref
        Send->>Files: readFileSync path, or fetch URL
        Files-->>Send: { name, type, data, origin }
    end
    Send->>Send: totalFileBytes > 8MB → fail(1)

    alt --dry-run
        Send->>Claude: buildDryRunReport() — masked URL, chunks, files, embed
        Note over Send,Discord: no POST is made — exit 0
    else real send
        Send->>Send: chunks(msg, 2000)
        loop each chunk i of n
            Send->>Discord: POST content, allowed_mentions {parse: []},<br/>flags 4096 if --quiet,<br/>files + embeds only on chunk n
            alt 2xx
                Discord-->>Send: 204
            else 429 or 5xx, attempt < 5
                Discord-->>Send: retry_after / Retry-After
                Send->>Send: sleep(retryDelayMs) then recurse
            else other 4xx
                Discord-->>Send: error body
                Send->>Claude: error: HTTP ... → exit 1
            end
        end
        Send->>Claude: ✅ sent message + embed + 1 attachment to Discord
    end
```

## Validation Pipeline

The ordering here is deliberate and load-bearing: everything cheap and local
runs before anything that touches the network or the secret. `--dry-run` is
defined as *this entire pipeline minus the final POST*, which is what lets a
clean dry run guarantee a real send would also pass validation.

```mermaid
flowchart TD
    ARGS["process.argv.slice(2)"] --> PARSE{"parse flags"}
    PARSE -->|"--username / --to<br/>--file / --attach<br/>--quiet / --silent / --dry-run<br/>--embed-*"| COUNT
    PARSE -->|"missing flag value"| FAIL1["fail() → exit 1"]
    PARSE -->|"--help / -h"| HELP["print USAGE → exit 0"]
    PARSE -->|"unknown --flag"| FAIL1
    PARSE -->|"-- (separator)"| TEXT
    PARSE -->|"anything else"| TEXT["rest[] → message text"]
    TEXT --> COUNT

    COUNT{"fileRefs ≤ MAX_FILES (10)?"} -->|no| FAIL1
    COUNT -->|yes| EMBED{"buildEmbed()<br/>pure, zero I/O"}

    EMBED -->|"title >256, desc >4096,<br/>fields >25, name >256,<br/>value >1024, total >6000"| FAIL1
    EMBED -->|ok or null| MSG{"any text, file, or embed?"}

    MSG -->|"none yet"| STDIN["read stdin"]
    STDIN --> MSG2{"still nothing?"}
    MSG2 -->|yes| FAIL2["exit 2 — no message or attachment"]
    MSG2 -->|no| HOOK
    MSG -->|yes| HOOK

    HOOK["loadWebhook(--to)"] -->|"missing config / bad JSON /<br/>unknown --to name"| FAIL1
    HOOK --> LOAD["loadFile() per --file<br/>local read or network fetch"]
    LOAD -->|"ENOENT / EISDIR / HTTP error / empty"| FAIL1
    LOAD --> SIZE{"combined ≤ 8MB?"}
    SIZE -->|no| FAIL1
    SIZE -->|yes| DRY{"--dry-run?"}

    DRY -->|yes| REPORT["buildDryRunReport()<br/>masked URL only → exit 0"]
    DRY -->|no| CHUNK["chunks(msg, 2000)"]
    CHUNK --> POST["post() per chunk"]

    style FAIL1 fill:#ffcdd2,stroke:#b91c1c,color:#1f2937
    style FAIL2 fill:#ffcdd2,stroke:#b91c1c,color:#1f2937
    style REPORT fill:#c8e6c9,stroke:#15803d,color:#1f2937
    style HELP fill:#c8e6c9,stroke:#15803d,color:#1f2937
    style POST fill:#5865F2,stroke:#3c45a5,color:#ffffff
```

Two details worth calling out because they are easy to get wrong when editing:

- **`buildEmbed()` runs before `loadWebhook()`.** A malformed embed must not be
  able to trigger a config read or a network fetch.
- **URL attachments are genuinely fetched under `--dry-run`.** That is not an
  oversight — the reported size and MIME type must come from real bytes, and
  the 8MB check must be meaningfully exercised.

## Retry State Machine

`post()` is recursive: each retry re-enters `post()` with `attempt + 1`.

```mermaid
stateDiagram-v2
    [*] --> Posting
    Posting --> Success: res.ok
    Posting --> Classify: non-2xx

    Classify --> Waiting: 429 or 5xx, attempt < MAX_RETRIES (5)
    Classify --> Failed: 4xx other than 429
    Classify --> Exhausted: 429 or 5xx, attempt = 5

    Waiting --> Posting: attempt + 1

    Success --> [*]
    Failed --> [*]
    Exhausted --> [*]

    note right of Waiting
        retryDelayMs() prefers, in order:
        1. retry_after in the JSON body
        2. the Retry-After header
        3. 500ms * 2^attempt
        plus 0-49ms jitter, capped at 30s
    end note

    note right of Failed
        Bad webhook, bad payload,
        401/403/404 — permanent,
        throws immediately
    end note
```

## Persistent State

`config.json` is the only thing this system stores. It is not a database; the
diagram below documents its shape and the one relationship inside it.

```mermaid
erDiagram
    SKILL_DIR ||--|| CONFIG_JSON : contains
    SKILL_DIR ||--|| SENDER : contains
    SKILL_DIR ||--|| SKILL_MD : contains
    CONFIG_JSON ||--o{ NAMED_WEBHOOK : "optional webhooks map"

    SKILL_DIR {
        path location "~/.claude/skills/discord-notify/"
    }
    CONFIG_JSON {
        string webhookUrl "required — the default target"
        object webhooks "optional — key omitted entirely when empty"
        octal mode "0600, enforced twice on write"
    }
    NAMED_WEBHOOK {
        string name PK "NAME_RE: A-Za-z0-9_- only"
        string url "WEBHOOK_RE-validated at install time"
    }
    SENDER {
        string file "discord_send.js — resolves config via __dirname"
    }
    SKILL_MD {
        string file "SKILL.md — frontmatter consumed by Claude"
    }
```

`buildConfig()` deliberately omits the `webhooks` key rather than writing
`"webhooks": {}`, so a user who never configures named channels gets the same
minimal two-line config the tool has always written.

## Module & Function Relationships

Both files are flat ES modules — no classes, so there is no class diagram to
draw. The real structure is the export surface, which exists almost entirely
so the test suite can reach pure logic without spawning a process.

```mermaid
graph LR
    subgraph sender["skill/discord_send.js"]
        MAIN["main()"]
        BUILDEMBED["buildEmbed()"]
        PARSECOLOR["parseEmbedColor()"]
        PARSEFIELD["parseEmbedField()"]
        LOADHOOK["loadWebhook()"]
        LOADFILE["loadFile()"]
        MIMEFOR["mimeFor()"]
        EXTFOR["extFor()"]
        TOTALBYTES["totalFileBytes()"]
        CHUNKS["chunks()"]
        POSTFN["post()"]
        RETRY["retryDelayMs()"]
        DRYREPORT["buildDryRunReport()"]
        MASK2["maskWebhookUrl()"]
        FMTBYTES["formatBytes()"]
        FMTCOLOR["formatEmbedColor()"]
        FAIL["fail()"]
    end

    subgraph installer["bin/cli.js"]
        CLIMAIN["main()"]
        PROMPTHOOK["promptWebhook()"]
        PROMPTNAMED["promptNamedWebhooks()"]
        BUILDCFG["buildConfig()"]
        INSTALL["install()"]
        SENDTEST["sendTest()"]
        MASK1["mask()"]
    end

    MAIN --> PARSECOLOR
    MAIN --> PARSEFIELD
    MAIN --> BUILDEMBED
    MAIN --> LOADHOOK
    MAIN --> LOADFILE
    MAIN --> TOTALBYTES
    MAIN --> CHUNKS
    MAIN --> DRYREPORT
    MAIN --> POSTFN
    LOADFILE --> MIMEFOR
    LOADFILE --> EXTFOR
    POSTFN --> RETRY
    POSTFN --> POSTFN
    DRYREPORT --> MASK2
    DRYREPORT --> FMTBYTES
    DRYREPORT --> FMTCOLOR
    DRYREPORT --> TOTALBYTES
    BUILDEMBED --> FAIL
    LOADHOOK --> FAIL
    LOADFILE --> FAIL

    CLIMAIN --> PROMPTHOOK
    CLIMAIN --> PROMPTNAMED
    CLIMAIN --> INSTALL
    CLIMAIN --> SENDTEST
    INSTALL --> BUILDCFG
    PROMPTHOOK --> MASK1
    PROMPTNAMED --> MASK1

    MASK1 -.->|"deliberate duplicate,<br/>not an import"| MASK2

    style FAIL fill:#ffcdd2,stroke:#b91c1c,color:#1f2937
    style POSTFN fill:#5865F2,stroke:#3c45a5,color:#ffffff
    style MASK1 fill:#fff9c4,stroke:#a16207,color:#1f2937
    style MASK2 fill:#fff9c4,stroke:#a16207,color:#1f2937
```

## Test Topology

174 test cases across 36 suites (29 of them top-level `describe` blocks) —
roughly 2,550 lines of test against
930 lines of source. The suite splits into two strategies, and the split is
forced by `fail()`: it calls `process.exit(1)`, so every error path has to be
asserted from a child process rather than in-process.

```mermaid
graph TB
    subgraph unit["In-process unit tests — pure exports"]
        U1["chunks, mimeFor, extFor"]
        U2["parseEmbedColor, parseEmbedField,<br/>formatEmbedColor"]
        U3["buildEmbed — valid inputs only"]
        U4["totalFileBytes, formatBytes,<br/>maskWebhookUrl"]
        U5["buildDryRunReport"]
        U6["buildConfig, mask,<br/>WEBHOOK_RE, NAME_RE"]
        U7["promptNamedWebhooks<br/>via scripted fake readline"]
    end

    subgraph fetchmock["In-process, global.fetch stubbed"]
        F1["post retry/backoff"]
        F2["allowed_mentions always sent"]
        F3["quiet → flags 4096"]
        F4["post embeds param"]
    end

    subgraph child["Child-process tests — execFile / spawn, temp $HOME"]
        C1["attachment size limit e2e"]
        C2["webhook resolution --to"]
        C3["--quiet / --silent parsing"]
        C4["URL-sourced attachment limits"]
        C5["--dry-run e2e"]
        C6["--embed-* flag errors"]
        C7["bin/cli.js e2e, scripted stdin,<br/>corrupted-config recovery"]
    end

    SRC1["skill/discord_send.js"] --> unit
    SRC1 --> fetchmock
    SRC1 --> child
    SRC2["bin/cli.js"] --> unit
    SRC2 --> child

    child -->|"why: fail() calls process.exit"| REASON["exit codes 1 and 2<br/>can only be observed<br/>out-of-process"]

    style REASON fill:#fff9c4,stroke:#a16207,color:#1f2937
    style SRC1 fill:#e3f2fd,stroke:#1d4ed8,color:#1f2937
    style SRC2 fill:#e3f2fd,stroke:#1d4ed8,color:#1f2937
```

## Distribution & CI

```mermaid
graph TB
    DEV["Local dev<br/>npm test → node --test"]
    PUSH["git push / PR to main"]
    GH["GitHub Actions<br/>ubuntu-latest"]
    N18["node 18.x"]
    N20["node 20.x"]
    N22["node 22.x"]
    PUB["npm publish<br/>files: bin/ + skill/ only"]
    NPX["end user: npx claude-discord-notify"]
    MACHINE["~/.claude/skills/discord-notify/"]

    DEV --> PUSH --> GH
    GH --> N18
    GH --> N20
    GH --> N22
    DEV --> PUB --> NPX --> MACHINE

    style GH fill:#fff9c4,stroke:#a16207,color:#1f2937
    style MACHINE fill:#ffe0b2,stroke:#c2410c,color:#1f2937
```

`engines.node` is `>=18` and CI matches it. That floor is real, not decorative:
the code uses global `fetch`, `FormData`, and `Blob`, plus
`node:readline/promises` — none of which exist on Node 16. `files` in
`package.json` ships only `bin/` and `skill/`, so tests and docs never reach a
user's machine.

## Directory Structure

```mermaid
graph TD
    ROOT["claude-discord-notify/"]

    ROOT --> BIN["bin/"]
    ROOT --> SKILL["skill/"]
    ROOT --> TEST["test/"]
    ROOT --> GHDIR[".github/workflows/"]
    ROOT --> DOCS["docs/architecture/"]
    ROOT --> PKG["package.json"]
    ROOT --> RM["README.md"]
    ROOT --> LIC["LICENSE"]

    BIN --> CLIJS["cli.js — 243 lines<br/>installer"]
    SKILL --> SKMD["SKILL.md — 223 lines<br/>skill manifest"]
    SKILL --> SENDJS["discord_send.js — 683 lines<br/>sender"]
    TEST --> T1["discord_send.test.js — 2203 lines"]
    TEST --> T2["cli.test.js — 345 lines"]
    GHDIR --> CIYML["ci.yml"]
    DOCS --> ARCH["README.md — this file"]

    style ROOT fill:#e1f5fe,stroke:#1d4ed8,color:#1f2937
    style SKILL fill:#c5e1a5,stroke:#4d7c0f,color:#1f2937
    style BIN fill:#c5e1a5,stroke:#4d7c0f,color:#1f2937
    style TEST fill:#fff59d,stroke:#a16207,color:#1f2937
    style DOCS fill:#f8bbd0,stroke:#be185d,color:#1f2937
```

Note that `skill/` is both source and payload: the directory is copied verbatim
to the user's machine at install time. There is no build step, no bundler, and
no transpile — what is in the repo is what runs.

## Key Design Decisions

### 1. The two CLIs never import each other

**Context:** `mask()` in the installer and `maskWebhookUrl()` in the sender are
the same six lines of code.

**Decision:** Duplicate it, and document the duplication in-source.

**Consequences:** `discord_send.js` stays a standalone file that can be copied
anywhere and run with `node` alone. Sharing a helper would mean shipping a
`lib/` the installer must also copy, or a bundling step. The cost is one
function that has to be changed in two places — bounded, and tested on both
sides.

### 2. Config lives next to the sender, resolved via `__dirname`

**Context:** The sender needs a webhook URL and no environment variable or CLI
flag supplies one.

**Decision:** `CONFIG_PATH = path.join(__dirname, "config.json")`.

**Consequences:** Zero configuration at call time — Claude just runs the script.
The trade-off is that **running `skill/discord_send.js` from a repo checkout
will not find a config**, because the checkout has no `config.json`; only the
installed copy under `~/.claude/skills/discord-notify/` does. That is why the
child-process tests build a temp `$HOME` and a temp skill dir rather than
invoking the repo file directly.

### 3. `allowed_mentions: { parse: [] }` is unconditional

**Context:** Claude generates the message text, and that text can contain
`@everyone` by accident.

**Decision:** Every outgoing payload sets it. There is no flag to turn it off.

**Consequences:** This tool can never ping a channel, full stop. `--quiet` is a
*separate*, orthogonal control — it sets the `SUPPRESS_NOTIFICATIONS` flag
(4096) to suppress the generic new-message notification, which is a different
mechanism from mention suppression. Both can apply to the same message.

### 4. `--dry-run` skips only the final POST

**Context:** A preview that skips validation would give false confidence.

**Decision:** Run the entire pipeline — including real network fetches for URL
attachments — and branch out only immediately before `post()`.

**Consequences:** A clean dry run is a genuine guarantee that the real send
would pass validation. The cost is that `--dry-run` is not free: it makes
network requests, just not to Discord. The report masks the webhook token, so
dry-run output is safe to paste back into a conversation.

### 5. Attachments and embeds ride the *final* message chunk

**Context:** Messages over 2000 characters are split into several Discord
messages.

**Decision:** Text chunks post in order; `files` and `embeds` are attached only
when `i === parts.length - 1`. `quiet` applies to every chunk, since it is a
per-message flag.

**Consequences:** Attachments visually follow the text they belong to. It also
means a multi-chunk send is N sequential API calls with N chances to hit a rate
limit — which is why the retry logic sits inside `post()` rather than around
the loop.

### 6. Exit code 2 is distinct from exit code 1

**Context:** "You gave me nothing to send" is a different failure from "the
thing you gave me is invalid."

**Decision:** Empty input exits 2; every `fail()` path exits 1.

**Consequences:** A caller can distinguish "nothing to do" from "something went
wrong" without parsing stderr.

## Security Architecture

- **The webhook URL is a bearer credential.** Anyone holding it can post to that
  channel. It is written with `mode: 0o600` and then `chmodSync(0o600)` again,
  because `writeFileSync`'s mode is ignored when the file already exists.
- **The URL is never printed in full.** The installer masks it when echoing an
  existing config; `--dry-run` prints only `maskWebhookUrl()` output plus the
  `--to` name. Both keep the numeric id and show 4 leading + 4 trailing token
  characters.
- **`WEBHOOK_RE` is an input filter, not a security boundary.** It constrains
  install-time input to `discord.com` / `discordapp.com` (plus `canary.` and
  `ptb.`) so a typo or a pasted non-Discord URL cannot become the send target.
  Note that it validates only at install time — a hand-edited `config.json` is
  used as-is.
- **Mention suppression is on by default** and not overridable (decision 3).
- **`--file` accepts arbitrary local paths.** The sender uploads whatever it is
  pointed at with no allowlist, so the caller decides what leaves the machine.
  In practice the caller is Claude, and `SKILL.md` is where that judgment is
  specified.

## Scalability & Performance

Scalability is not a meaningful axis here — one process, one message, then exit.
The limits that do bind are Discord's, and all of them are enforced locally
before any upload:

| Limit | Value | Constant |
|---|---|---|
| Message content | 2,000 chars | `MAX` |
| Attachments per message | 10 | `MAX_FILES` |
| Combined attachment size | 8 MB | `MAX_ATTACHMENT_BYTES` |
| Embed title / description | 256 / 4,096 chars | `EMBED_TITLE_MAX` / `EMBED_DESCRIPTION_MAX` |
| Embed fields | 25 | `EMBED_FIELDS_MAX` |
| Embed field name / value | 256 / 1,024 chars | `EMBED_FIELD_NAME_MAX` / `EMBED_FIELD_VALUE_MAX` |
| Embed combined total | 6,000 chars | `EMBED_TOTAL_MAX` |
| Retries before giving up | 5 | `MAX_RETRIES` |
| Backoff base / cap | 500 ms / 30 s | `BASE_DELAY_MS` / `MAX_DELAY_MS` |

Files are read fully into memory as Buffers; at an 8 MB ceiling that is fine and
streaming would add complexity for no gain.

## Observability

There is none beyond process output, and that is proportionate. Success prints
`✅ sent <what> to Discord` to stdout; failures print `error: ...` to stderr and
exit non-zero. Retries are silent — a caller sees only the final outcome. The
installer surfaces a failed test send as a warning with a re-run hint rather
than aborting, since the skill files are already installed correctly at that
point.

## Known Extension Points

Deliberate scope boundaries, documented in `SKILL.md` so Claude declines rather
than improvises around them:

- **Exactly one embed per invocation.** No `--embed-url`, `--embed-footer`,
  `--embed-image`, `--embed-thumbnail`, `--embed-author`, `--embed-timestamp`,
  and no multi-embed support.
- **8 MB attachment ceiling** is Discord's non-boosted default. Boosted servers
  allow more; the constant is not currently configurable.
- **Webhooks only.** No bot token, so no reading messages, no reactions, no
  threads, no editing or deleting after send.

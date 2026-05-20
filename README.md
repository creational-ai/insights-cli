<!--
  SOURCE OF TRUTH for the public creational-ai/insights-cli repo's README.md.
  Edit HERE (monorepo, version-controlled, reviewed). `/deploy` Phase 6
  copies this file to ~/Dev/insights-cli/README.md and commits+pushes it to
  the insights-cli `main` (idempotent — only when content changed). Never
  hand-edit ~/Dev/insights-cli/README.md directly; it is a deploy artifact.
-->
# insights-cli

The client for **insights** — a multi-project analytics control plane. Install it on your Mac, point it at your Genesis server, and pull data dumps or AI-authored narrative reports for any project from its repo.

* Self-updating: `insights upgrade` verifies a server-published sha256 against the public wheel and reinstalls in place — no manual scp
* Workspace-aware: run verbs from any product repo with an `.insights/` dir; the CLI walks up to find the project — no slug to remember
* `insights pull` — version-sliced data dumps to markdown; `insights generate` — AI-authored narrative reports
* `insights bundle init` / `insights project init` — scaffold new report bundles and projects from the product repo
* Bearer auth + server URL from a single mode-0600 `~/.insights/config`

This repo is the **public distribution surface only** — it carries no source, just built-wheel Release assets that `insights upgrade` pulls from. The platform/server internals live in the private `insights` monorepo; this README is the complete client story (install, config, everyday use).

## Table of Contents

- [Getting started](#getting-started)
- [Configuration](#configuration)
- [Working with projects](#working-with-projects)
- [Pulling data](#pulling-data)
- [AI-authored narratives](#ai-authored-narratives)
- [Authoring report bundles](#authoring-report-bundles)
- [Keeping the CLI current](#keeping-the-cli-current)
- [Exit codes](#exit-codes)
- [Source & contributing](#source--contributing)
- [License](#license)

## Getting started

Requires Python 3.12+ and [`uv`](https://docs.astral.sh/uv/).

**1. Install.** One public URL, pasted once. Replace `vX.Y.Z` / `X.Y.Z` with a published release (resolve the newest tag with the one-liner below), then run `insights upgrade` immediately:

```bash
uv tool install https://github.com/creational-ai/insights-cli/releases/download/vX.Y.Z/insights-X.Y.Z-py3-none-any.whl
insights upgrade   # mandatory, immediately after first install
```

Resolve the newest published tag at any time:

```bash
gh release view --repo creational-ai/insights-cli --json tagName -q .tagName
```

> The literal `vX.Y.Z` in the install URL is **intentionally not version-stamped**. `/deploy` keeps this README in sync from the monorepo source but never substitutes a concrete version into the install incantation per ship, so the literal deliberately lags the newest release. The mandatory trailing `insights upgrade` converges whatever you installed to the server's recommended version — the lagging literal and the always-fresh one-liner land on the same place after that one upgrade.

**2. Configure.** Point the CLI at your Genesis server with a mode-0600 `~/.insights/config`:

```bash
mkdir -p ~/.insights
cat > ~/.insights/config <<'EOF'
[server]
url = "http://192.168.0.54:8004"
token = "<your write-scope bearer>"
EOF
chmod 600 ~/.insights/config
```

**3. First use.** From any product repo that has an `.insights/` workspace:

```bash
$ cd ~/Development/hexario-revive
$ insights pull daily-standard --window 5d

INFO     wrote .insights/reports/daily-standard-2026-04-20_08-50-12-0700-data.md (4842 bytes)
```

That's it — the CLI resolved the project from the repo's `.insights/`, authenticated from `~/.insights/config`, ran the pull server-side on Genesis, and wrote the report back into your repo. No slug, no flags, no server login.

## Configuration

The CLI reads its server URL and bearer from `~/.insights/config` (TOML, **mode-0600 enforced**):

```toml
[server]
url   = "http://192.168.0.54:8004"   # Genesis on the home LAN; 127.0.0.1:8004 only works ON Genesis
token = "<32-char hex write-scope bearer>"   # must match a Server-side registration
```

**Resolution order** (each tier overrides the next):

| What | Resolution order |
|------|------------------|
| Server URL | `INSIGHTS_SERVER_URL` env → `[server].url` in `~/.insights/config` → `http://127.0.0.1:8004` default |
| Bearer token | `INSIGHTS_TOKEN_GENESIS_CLI` env → `[server].token` in `~/.insights/config` |

> A **missing** config file silently falls back to `127.0.0.1:8004` — which is unreachable from a Mac and will look like a connection failure. Always set `[server].url` to the Genesis LAN IP on a client machine. Malformed TOML exits 3.

The mode-0600 requirement is enforced — a group/world-readable config is refused with the exact fix in the error (no silent token-exposure widening):

```bash
$ insights pull daily-standard
config error: refusing to read ~/.insights/config: must be 0600 or 0400 (currently 0o644); run: chmod 600 ~/.insights/config
```

**Rotation.** Edit the `token` value (your operator rotates the Server-side registration in lockstep). The next invocation picks it up — no shell re-source.

> The token is a write-scope bearer your operator provisions on the Genesis Server; the CLI does not mint it. If you get `4` (auth error), confirm the value matches the current Server-side registration.

**Verify your setup.** `insights version --full` is the one-line diagnostic that confirms which Server URL and which auth tier actually resolved:

```bash
$ insights version --full
insights 0.2.5
python 3.12.3
server http://192.168.0.54:8004
auth   ~/.insights/config [server.token]
```

The `auth` line names the winning tier (env var name, config field, or file path) — the fastest way to debug "is my token actually being read?". Bare `insights version` prints just the first line; `insights --version` / `insights --version --full` are byte-identical aliases. Always exits 0, even when no token is configured (shows `auth   <none>`).

## Working with projects

A project lives in its product repo under `.insights/` (connection descriptor, bundles, generated reports). The daily verbs (`pull`, `generate`, `lint`, `ask`, `schedule status`) **walk up from your CWD** to the nearest `.insights/` — so you just `cd` into the repo and run the verb.

Scaffold a new project's `.insights/` workspace from inside the product repo:

```bash
$ cd ~/Development/my-product
$ insights project init
# interactive prompts: slug, name, BQ project/dataset/views, GA4 property, min_version
# → scaffolds .insights/ (config.yaml + gitignored subdirs) and registers it on the Server

$ cp ~/Downloads/sa-my-product.json .insights/creds/
$ insights project update              # lints, PATCHes metadata, ships bundle files, uploads the SA key to Genesis (mode 0600, atomic) — single 3-leg verb
```

List the projects the Server knows about:

```bash
$ insights project list
hexario
my-product
```

Remove a project (destructive — sweeps Server-side state + local `.insights/` + orphan systemd schedules in one verb):

```bash
$ cd ~/Development/my-product
$ insights project remove my-product
About to remove project 'my-product':
  Server-side: rm -rf /home/pi/Dev/insights/projects/my-product (via DELETE /api/v1/projects/my-product on 192.168.0.54:8004)
  Local: rm -rf /Users/you/Development/my-product/.insights (includes SA key at creds/sa-my-product.json)
  Schedules: will sweep any insights-my-product-*.{service,timer} systemd units on Genesis

This is destructive and cannot be undone.
Recovery: re-issue SA key from GCP console + insights project update
Continue? [y/N]: y
✓ Removed project 'my-product' (2 orphan systemd units swept by Server)
```

| Flag | Effect |
|------|--------|
| `--yes` / `-y` | Skip the confirmation prompt |
| `--keep-local` | DELETE Server-side only; preserve the local `.insights/` |

The verb is idempotent: a second invocation prints `nothing to remove` (exit 3). If your CWD's `.insights/config.yaml` belongs to a *different* slug than the positional you typed, the local tree is left untouched and a stderr advisory names the mismatch — you can't accidentally `rm -rf` an unrelated project's workspace.

Overriding the walked-up project:

| Override | Effect |
|----------|--------|
| `<slug>` positional | `insights pull hexario daily-standard` — explicit project (works anywhere) |
| `--slug <slug>` | Same, as a flag; strict-mismatch with the walked-up workspace exits 3 |
| `INSIGHTS_PROJECT=<repo path>` | Sticky context across `cd` into dirs **without** their own `.insights/` (a CWD walk-up still wins) |

## Pulling data

`insights pull [<slug>] <bundle>` — a version-sliced data dump rendered to markdown (one table per bundle section + the computed indicators block).

```bash
$ insights pull daily-standard --window 5d

INFO     wrote .insights/reports/daily-standard-2026-04-20_08-50-12-0700-data.md (4842 bytes)
```

One section only (filename gains the section before `-data`):

```bash
$ insights pull daily-standard --section adoption --window 7d

INFO     wrote .insights/reports/daily-standard-adoption-2026-04-20_09-02-44-0700-data.md (1318 bytes)
```

| Flag | Purpose |
|------|---------|
| `--window <Nd>` | Lookback window, e.g. `7d`, `30d` (bundle default if omitted) |
| `--section <id>` | Dump a single bundle section instead of all |

> `pull` is read-only and runs a server-side lint pre-flight — a bundle with a silent-wrongness pattern (lexicographic `ver` compare, raw `events_*`, unknown indicator) exits 1 with `<file>:<line>: <rule>: <message>` and writes nothing.

## AI-authored narratives

`insights generate [<slug>] <bundle>` — an AI-authored narrative report (writes both the narrative and its `-data.md` companion).

```bash
$ insights generate daily-standard --window 5d

INFO     wrote .insights/reports/daily-standard-2026-04-20_08-18-32-0700.md (10301 bytes)
```

~2–5 min wall-time on Pi5 with the default substrate (~2–3 min author + ~10s review for `daily-standard`).

| Flag | Purpose |
|------|---------|
| `--window <Nd>` | Lookback window |
| `--no-review` | Skip the review pass — faster when iterating interactively |

> `generate` is one synchronous call; nothing lands on disk if the author leg fails (exit 6) — just re-run. If the review leg fails (exit 8) the narrative is already written; re-run only the review.

## Authoring report bundles

A bundle is the 4-file unit (`design.yaml`, `template.md`, `instructions.md`, `CHANGELOG.md`) that defines what a report contains. Scaffold one in the product repo, author the files, then ship it to the Server:

```bash
$ cd ~/Development/hexario-revive
$ insights bundle init weekly-deep        # scaffolds .insights/reports/designs/weekly-deep/
$ $EDITOR .insights/reports/designs/weekly-deep/{design.yaml,template.md,instructions.md,CHANGELOG.md}
$ insights project update                 # syncs all local bundles to the Genesis Server
```

List a project's bundles, or remove one:

```bash
$ insights bundle list --output table
$ insights bundle remove weekly-deep
```

> `bundle init` is idempotent in the **Server-outage-recovery** sense: re-running after a failed POST recovers cleanly via a 4-state (local × Server) probe. When both local and Server already have the bundle it refuses with exit 3 ("already aligned" — use `insights project update` to ship edits). A partial local scaffold is re-completed; content you already authored to the 3 required files is preserved.

Every verb has `--help` for its full flag surface, and the list verbs (`bundle list`, `project list`, `report list`, `query list`, `compute list`) accept `--output json|yaml|table`.

## Keeping the CLI current

`insights upgrade` self-updates the installed CLI from the published wheel.

```bash
$ insights upgrade
Already at v0.2.4
```

When a required upgrade is available it downloads, verifies, and reinstalls; when an upgrade is *available but you're still compatible* it prints a nag and does nothing (intentional — re-run once it becomes required):

```bash
$ insights upgrade
An upgrade is available: v0.2.3 → v0.2.5 (you are still compatible — min v0.2.0).
Re-run `insights upgrade` once it becomes a required upgrade, or pin manually.
```

The four version-compare outcomes (`packaging.Version`, never lexicographic):

| Local vs recommended | Behavior |
|----------------------|----------|
| equal | "Already at vX.Y.Z" — exit 0 |
| newer | informational — exit 0 |
| older but ≥ min-compatible | upgrade-available nag, **no download/install** — exit 0 |
| older than min-compatible | download → sha256-verify → `uv tool install --reinstall` |

> **Two independent trust roots.** The server publishes the wheel's sha256 through a bearer-auth metadata endpoint; the wheel bytes come unauthenticated from this repo's public GitHub Release. `upgrade` sha256's the downloaded bytes and compares to the server-published digest **before** any install runs — a mismatch aborts with the prior install untouched. The server never serves wheel bytes, only metadata.

**scp fallback.** If `insights upgrade` is broken on a host, the legacy path (`uv build` → `scp dist/*.whl` → `uv tool install --reinstall`) from the private monorepo remains a supported recovery route. It is the fallback, not the routine path.

## Exit codes

| Code | Meaning |
|------|---------|
| `0` | Success (incl. already-current / compatible-nag for `upgrade`) |
| `1` | Lint or partial-report issues (report still written where applicable) |
| `2` | Usage / CLI argument error |
| `3` | Config error — bad `~/.insights/config`, mode-not-0600, project or bundle not registered on Server (404 — run `insights project init` / `insights bundle init`), incomplete bundle, product mismatch / unparseable manifest version (`upgrade`) |
| `4` | Authentication error — bad/expired bearer (401/403) |
| `5` | Transport / filesystem error — Server unreachable, can't write report; wheel download or sha256 mismatch (`upgrade`) |
| `6` | Author-leg backend failure in `generate`; or `uv tool install` non-zero in `upgrade` — nothing on disk, re-run |
| `7` | Host not ready (server-side prerequisite, e.g. user-linger) |
| `8` | Review-leg backend failure in `generate` — narrative already on disk; re-run the review |

## Source & contributing

This repo carries **no source code** — only built-wheel Release assets. The `insights` source, design docs, tests, and the `/deploy` pipeline that publishes here live in the private `creational-ai` monorepo. File client issues against the monorepo; this public repo is purely the distribution channel `insights upgrade` pulls from.

## License

[MIT](LICENSE).

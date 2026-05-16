# insights-cli

Public distribution channel for the **insights** CLI — a multi-project analytics
control plane. This repository serves built-wheel **Release assets**; it carries
no source code. The source of truth is the private `insights` monorepo; this repo
is the throwaway public surface that `/deploy` publishes wheels to and that
`insights upgrade` pulls from.

## Install

First install on a new machine — one public URL, pasted once. Replace
`vX.Y.Z` / `X.Y.Z` with a published release version (see "Always-fresh
version" below), then run `insights upgrade` immediately:

```bash
uv tool install https://github.com/creational-ai/insights-cli/releases/download/vX.Y.Z/insights-X.Y.Z-py3-none-any.whl
insights upgrade   # mandatory, immediately after first install
```

The literal `vX.Y.Z` above is **intentionally allowed to lag** the newest
release: `/deploy` publishes Release assets here but **never commits to this
repo**, so this README is not re-stamped per ship. That is by design — the
trailing `insights upgrade` is the §8 self-heal: it reads the server's
`GET /v1/cli/version` manifest and converges whatever you installed (an old
literal tag, or the fresh tag) to the server's recommended version. Both the
lagging literal and the always-fresh one-liner below land on the same place
after that one mandatory `insights upgrade`.

### Always-fresh version

To resolve the newest published tag at any time (no lag):

```bash
gh release view --repo creational-ai/insights-cli --json tagName -q .tagName
```

Substitute the printed `vX.Y.Z` into the `uv tool install` URL above (the
asset filename is `insights-X.Y.Z-py3-none-any.whl`, version without the
leading `v`). Then run `insights upgrade` once.

### scp fallback

If `insights upgrade` is broken on a host, the legacy monorepo path
(`uv build` → `scp dist/*.whl mbp:/tmp/` → `uv tool install --reinstall`)
remains a supported recovery route — see `ux-design.md` Flow 6 / F3 in the
private monorepo. It is the fallback, not the routine path.

## How it works

- `/deploy` (monorepo) builds the wheel, publishes it as a Release asset here,
  and stamps the wheel's sha256 + release tag into the server's version-metadata
  constants.
- The server exposes a bearer-auth `GET /v1/cli/version` endpoint publishing the
  recommended version, minimum-compatible floor, release tag, and sha256.
- `insights upgrade` reads that manifest, downloads the Release asset from this
  public repo, verifies the sha256 against the server-published value (two
  independent trust roots), and runs `uv tool install --reinstall`.

The server never serves wheel bytes — only metadata. Bytes come from GitHub
Releases on this repo.

## License

[MIT](LICENSE).

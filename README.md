# insights-cli

Public distribution channel for the **insights** CLI — a multi-project analytics
control plane. This repository serves built-wheel **Release assets**; it carries
no source code. The source of truth is the private `insights` monorepo; this repo
is the throwaway public surface that `/deploy` publishes wheels to and that
`insights upgrade` pulls from.

## Install

> **Placeholder.** The final, verified first-install incantation is authored in
> Task 8 / Step 5 (`ux-cli-distribution` plan) once the first real Release is
> published by `/deploy`. Until then, use the manual scp fallback documented in
> the monorepo (`ux-design.md` Flow 6 / F3).

```bash
# (final command added in Step 5 — installs the latest published wheel via
#  uv tool install from this repo's Releases, then `insights upgrade` self-updates)
```

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

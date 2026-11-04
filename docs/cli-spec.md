# Spec: `otg` — Go CLI client for the hub (Pi-like toolset)

Status: design only, no implementation.
Scope: a second client in this repo, alongside the JupyterLab extension, that
pairs with the Go hub and exposes a Pi-like read/write/glob/exec toolset so an
agent can be developed against the same hub without a browser.

## 1. Goals

- Pair with the hub using the existing `/pair/challenge` + `/pair` flow
  (PoW + per-CIDR rate limit), store credentials locally, reuse them.
- Speak MCP JSON-RPC 2.0 to `/mcp/{pair}/{token}` for the hub's existing
  tools (`file_*`, `notebook_*`, `site_info`, `job_*`).
- Additionally act as a **runner** on `/run?pair=` so the CLI host can
  execute jobs locally — this is what makes `exec` possible without Jupyter.
- Expose a Pi-like toolset (see §4) to a local agent loop.

## 2. Non-goals

- No browser/Jupyter dependency; no notebook tools required in CLI mode.
- No new hub endpoints; the CLI uses only the existing wire contract.
- No shell access by default (see §5).

## 3. Layout

```
cmd/otg/            # main package, thin
internal/otgcli/    # pair, MCP client, runner, tool dispatch
internal/wire/      # shared JSON-RPC / frame types (also imported by hub code later)
```

`internal/wire` is the single owner of the wire contract; hub code should be
refactored to import it when convenient (not required for v1).

## 4. Toolset (Pi-like)

Mirrors pi's built-in tools (`read`, `write`, `edit`, `bash`, `grep`, `find`,
`ls`) where a hub equivalent exists; adds `exec` as the raw-exec primitive.

| tool | impl | notes |
| --- | --- | --- |
| `ls` / `glob` | hub `file_list` with pattern | fallback: local `filepath.Glob` when `--local` |
| `read` | hub `file_read` | binary/npy handled per hub shaping |
| `write` / `edit` | hub `file_write` | `edit` = read+patch locally, then write |
| `grep` | local over files, or hub-side later | v1: local |
| `find` | local `filepath.WalkDir` | v1: local |
| `exec` | **local runner**: `exec.Command(argv...)`, no shell | requires `--allow-exec` |
| `bash` | **local runner**: `bash -c <cmd>` (fallback `sh -c`) | requires `--allow-exec` |

`exec` vs `bash` mirrors pi exactly: `exec` is raw argv (`shell: false`
equivalent), `bash` goes through a shell for pipes/redirection.

## 5. Security model

- Pair token = full toolset access. Once `--allow-exec` is on, the token is
  equivalent to shell access on the CLI host. Document this in README.
- `--allow-exec` is an explicit opt-in flag on `otg run`; without it the
  runner answers `exec`/`bash` jobs with an error frame.
- Credentials stored at `~/.config/otg/credentials.json` (0600).
- Hub-side protections (PoW, rate limit, TTL sweep) are unchanged; the CLI
  just consumes them.

## 6. Commands

```
otg pair --hub https://hub.example.com --label mycli   # PoW, save credentials
otg call <tool> [json-args]                            # one-shot MCP tools/call
otg run  [--allow-exec] [--dir .]                      # persistent runner on /run + local agent loop
otg status                                             # /health + site_info
```

`otg run` connects the WebSocket, sends `auth` then `hello`, and dispatches
incoming `job` frames to the local toolset (or hub MCP for remote tools),
replying with `result`/`progress` frames per the wire contract.

## 7. Execution semantics (from pi, to mirror)

- Timeout per exec (default e.g. 120 s), SIGTERM then SIGKILL after grace.
- Kill the whole process tree on cancel (`job_cancel` → `cancel` frame).
- Stream `progress` frames for long-running commands (stdout chunks).

## 8. Open questions

- Should `grep`/`find` be hub-side tools in the hub itself (useful for the
  lab extension too)? Recommend yes as a follow-up; v1 keeps them local.
- MCP transport: hub is stateless HTTP POST per call; CLI uses plain
  `net/http`, no SSE needed.

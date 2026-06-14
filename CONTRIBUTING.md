# Contributing

Thanks for improving `codex-mimo-sidecar`.

## Development

This repository is intentionally small. Keep changes focused on the sidecar wrapper, Codex skill metadata, and documentation.

Before opening a pull request:

```bash
bash -n scripts/codex-mimo-sidecar
bash -n scripts/codex-mimo-subagent
bash -n scripts/terminal-chat
python3 -m py_compile scripts/mimo-responses-proxy
scripts/mimo-responses-proxy --self-test
```

If you have `shellcheck` installed, also run:

```bash
shellcheck scripts/codex-mimo-sidecar scripts/codex-mimo-subagent scripts/terminal-chat
```

## Design Principles

- Keep the main agent responsible for planning and synthesis.
- Keep this project responsible for sidecar session execution and lifecycle controls.
- Prefer small flags over broad framework behavior.
- Preserve backward-compatible behavior unless there is a clear reason to break it.
- Avoid passing host environment variables by default; use `--pass-env` for explicit needs.

## Pull Requests

Please include:

- What changed.
- Why it changed.
- How you tested it.
- Any compatibility notes for existing users.

# Codex MiMo Sidecar

Run MiMo-backed Codex sidecars for bounded worker tasks while your main GPT session stays responsible for planning, review, and final judgment.

This project installs as a Codex skill. It gives Codex a small launcher, a local Responses-compatible proxy, and session bookkeeping so the main agent can delegate repetitive or token-heavy work to MiMo.

## Why This Exists

Use this when you want GPT to remain the lead agent, but want cheaper worker agents for:

- slow test runs and benchmark sweeps
- CI log inspection
- broad codebase search
- first-pass bug triage
- independent implementation attempts
- repeated verification tasks

The intended operating model is simple: GPT decides what matters, MiMo does bounded work, GPT reviews the result.

This project is inspired by `codex-deepseek-sidecar`, which established the pattern of using a Codex skill to launch lower-cost sidecar agents while keeping the main GPT session in charge of planning and synthesis.

## Repository Layout

```text
.
|-- SKILL.md
|-- agents/
|   `-- openai.yaml
|-- scripts/
|   |-- codex-mimo-sidecar
|   |-- codex-mimo-subagent
|   |-- mimo-responses-proxy
|   `-- terminal-chat
|-- .github/workflows/shellcheck.yml
|-- CONTRIBUTING.md
|-- LICENSE
`-- README.md
```

## Requirements

- Codex CLI
- Python 3.11 or newer
- Bash-compatible shell
- A MiMo API key for the upstream MiMo endpoint

On Windows, Git Bash works. The launchers also understand a Windows-style `CODEX_HOME` such as `D:/CodexHome`.

## Install

Clone this repository into your Codex skills directory:

```bash
git clone https://github.com/Iiiiibaraki/codex-mimo-sidecar ~/.codex/skills/mimo-codex-subagent
cd ~/.codex/skills/mimo-codex-subagent
chmod +x scripts/*
```

Then ask Codex:

```text
Use the mimo-codex-subagent skill. Configure MiMo if needed, then start a MiMo sidecar for bounded worker tasks in this repository.
```

## Configure Codex

The current Codex CLI profile system loads `$CODEX_HOME/<profile>.config.toml`. The default MiMo profile is:

```text
$CODEX_HOME/mimo.config.toml
```

Expected contents:

```toml
model_provider = "mimo"
model = "mimo-v2.5-pro"
model_reasoning_effort = "high"
approval_policy = "never"
model_auto_compact_token_limit = 600000

[model_providers.mimo]
name = "MiMo via codex-mimo-sidecar"
base_url = "http://127.0.0.1:12360/v1"
wire_api = "responses"
supports_websockets = false
```

You can create or verify this profile with:

```bash
scripts/codex-mimo-sidecar --profile mimo --configure
```

The launcher deliberately avoids writing legacy `[profiles.mimo]` tables into `config.toml`, because recent Codex versions expect profile files instead.

## Start The Local Proxy

Run the built-in proxy with your MiMo key:

```bash
MIMO_API_KEY=... scripts/mimo-responses-proxy
```

By default it listens on:

```text
http://127.0.0.1:12360/v1
```

Quick checks:

```bash
curl http://127.0.0.1:12360/health
curl http://127.0.0.1:12360/v1/models
```

## Call MiMo Directly Through Codex

Smoke test the Codex profile:

```bash
codex exec --profile mimo --skip-git-repo-check --sandbox danger-full-access -C . "Say exactly: MIMO_OK"
```

Expected run header:

```text
model: mimo-v2.5-pro
provider: mimo
```

## Use As A Sidecar

Start a bounded worker task:

```bash
scripts/codex-mimo-sidecar --cd "$PWD" --task-id log-review \
  "Analyze the latest CI log. Report the root cause and exact evidence."
```

Check status:

```bash
scripts/codex-mimo-sidecar --cd "$PWD" --task-id log-review --status
```

Resume:

```bash
scripts/codex-mimo-sidecar --cd "$PWD" --task-id log-review --resume \
  "Continue from the previous context and verify the proposed fix."
```

For non-interactive automation, add:

```bash
--no-monitor
```

## Use As A Subagent

`codex-mimo-subagent` has the same profile and session behavior, but uses subagent naming in prompts and session files:

```bash
scripts/codex-mimo-subagent --cd "$PWD" --task-id tests \
  "Run the focused tests and summarize failures with command output."
```

## Passing Environment Variables

The launcher starts Codex in a clean environment. Pass only what the worker needs:

```bash
API_TOKEN=... scripts/codex-mimo-sidecar --cd "$PWD" --pass-env API_TOKEN \
  "Run the integration check that requires API_TOKEN."
```

## Verification

Before publishing changes:

```bash
bash -n scripts/codex-mimo-sidecar
bash -n scripts/codex-mimo-subagent
bash -n scripts/terminal-chat
python -m py_compile scripts/mimo-responses-proxy
scripts/mimo-responses-proxy --self-test
```

Optional:

```bash
shellcheck scripts/codex-mimo-sidecar scripts/codex-mimo-subagent scripts/terminal-chat
```

## Troubleshooting

### `--profile mimo` fails with legacy profile errors

Remove old `[profiles.mimo]` or `[model_providers.mimo]` tables from `$CODEX_HOME/config.toml`. Keep MiMo settings in `$CODEX_HOME/mimo.config.toml`.

### Proxy is reachable but Codex does not use MiMo

Run:

```bash
codex exec --profile mimo --help
```

If the help command fails, Codex cannot load the profile file.

### No token usage appears

Confirm the request reaches the local proxy. The proxy access log should show:

```text
POST /v1/responses
```

The proxy response also includes a `usage` object when the upstream returns token accounting.

### Windows path issues

Use forward slashes for environment paths passed through Git Bash:

```bash
CODEX_HOME="D:/CodexHome"
```

## License

Apache-2.0. See [LICENSE](LICENSE).

## Acknowledgements

Inspired by the design and workflow of `codex-deepseek-sidecar`.

# Codex MiMo Sidecar

让 Codex 启动由 MiMo 驱动的 sidecar worker，把重复、耗 token、边界清晰的任务交给更便宜的模型处理，同时让主 GPT 会话继续负责规划、审查和最终判断。

这个项目以 Codex skill 的形式安装。它提供 sidecar 启动脚本、本地 Responses 兼容代理，以及会话记录机制，让主 agent 可以把测试、日志分析、代码搜索等工作分派给 MiMo。

## 为什么需要它

适合把这些任务交给 MiMo worker：

- 慢测试和 benchmark sweep
- CI 日志分析
- 大范围代码搜索
- bug 初筛
- 独立实现尝试
- 重复性验证工作

推荐的协作方式很简单：GPT 判断什么重要，MiMo 执行边界清晰的工作，最后 GPT 复核和综合。

本项目灵感来自 `codex-deepseek-sidecar`。它证明了一个很好用的模式：通过 Codex skill 启动低成本 sidecar agent，同时保持主 GPT 会话负责规划和总结。

## 仓库结构

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

## 环境要求

- Codex CLI
- Python 3.11 或更新版本
- Bash 兼容 shell
- MiMo API key

Windows 上可以使用 Git Bash。启动脚本支持类似 `D:/CodexHome` 这样的 Windows 路径。

## 安装

把仓库克隆到 Codex skills 目录：

```bash
git clone https://github.com/Iiiiibaraki/codex-mimo-sidecar ~/.codex/skills/mimo-codex-subagent
cd ~/.codex/skills/mimo-codex-subagent
chmod +x scripts/*
```

然后可以直接对 Codex 说：

```text
使用 mimo-codex-subagent skill。如果还没配置 MiMo，请先配置好，然后为当前仓库启动一个 MiMo sidecar，处理适合分派的 worker 任务。
```

## 配置 Codex

当前 Codex CLI 的 profile 机制会加载 `$CODEX_HOME/<profile>.config.toml`。默认 MiMo profile 是：

```text
$CODEX_HOME/mimo.config.toml
```

推荐内容：

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

可以用下面的命令创建或检查 profile：

```bash
scripts/codex-mimo-sidecar --profile mimo --configure
```

启动脚本不会再把旧式 `[profiles.mimo]` 表写进 `config.toml`。新版 Codex 更推荐使用独立 profile file。

## 启动本地代理

使用你的 MiMo key 启动内置代理：

```bash
MIMO_API_KEY=... scripts/mimo-responses-proxy
```

默认监听：

```text
http://127.0.0.1:12360/v1
```

快速检查：

```bash
curl http://127.0.0.1:12360/health
curl http://127.0.0.1:12360/v1/models
```

## 直接通过 Codex 调用 MiMo

先做一个 smoke test：

```bash
codex exec --profile mimo --skip-git-repo-check --sandbox danger-full-access -C . "Say exactly: MIMO_OK"
```

预期运行头里会出现：

```text
model: mimo-v2.5-pro
provider: mimo
```

## 作为 Sidecar 使用

启动一个边界清晰的 worker 任务：

```bash
scripts/codex-mimo-sidecar --cd "$PWD" --task-id log-review \
  "Analyze the latest CI log. Report the root cause and exact evidence."
```

查看状态：

```bash
scripts/codex-mimo-sidecar --cd "$PWD" --task-id log-review --status
```

继续同一个会话：

```bash
scripts/codex-mimo-sidecar --cd "$PWD" --task-id log-review --resume \
  "Continue from the previous context and verify the proposed fix."
```

用于自动化时可以加：

```bash
--no-monitor
```

## 作为 Subagent 使用

`codex-mimo-subagent` 的 profile 和会话行为与 sidecar 版本一致，只是命名和会话文件使用 subagent 表述：

```bash
scripts/codex-mimo-subagent --cd "$PWD" --task-id tests \
  "Run the focused tests and summarize failures with command output."
```

## 传递环境变量

启动脚本默认使用干净环境。只显式传递 worker 真正需要的变量：

```bash
API_TOKEN=... scripts/codex-mimo-sidecar --cd "$PWD" --pass-env API_TOKEN \
  "Run the integration check that requires API_TOKEN."
```

## 验证

提交改动前建议运行：

```bash
bash -n scripts/codex-mimo-sidecar
bash -n scripts/codex-mimo-subagent
bash -n scripts/terminal-chat
python -m py_compile scripts/mimo-responses-proxy
scripts/mimo-responses-proxy --self-test
```

可选：

```bash
shellcheck scripts/codex-mimo-sidecar scripts/codex-mimo-subagent scripts/terminal-chat
```

## 常见问题

### `--profile mimo` 报 legacy profile 错误

从 `$CODEX_HOME/config.toml` 里移除旧的 `[profiles.mimo]` 或 `[model_providers.mimo]` 表。MiMo 配置应保存在 `$CODEX_HOME/mimo.config.toml`。

### 代理可访问，但 Codex 没有使用 MiMo

运行：

```bash
codex exec --profile mimo --help
```

如果这个命令失败，说明 Codex 没有正确加载 profile file。

### 没看到 token usage

确认请求是否打到本地代理。代理 access log 应该出现：

```text
POST /v1/responses
```

如果上游返回 token 统计，代理响应里也会包含 `usage` 字段。

### Windows 路径问题

通过 Git Bash 传环境变量时建议使用正斜杠：

```bash
CODEX_HOME="D:/CodexHome"
```

## 致谢

本项目设计和工作流受到 `codex-deepseek-sidecar` 启发。

## License

Apache-2.0。详见 [LICENSE](LICENSE)。

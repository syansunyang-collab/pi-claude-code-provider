# pi-claude-code-provider（Windows 维护分支）

本仓库是 [chem/pi-claude-code-provider](https://github.com/chem/pi-claude-code-provider) 的分支。上游在 2026-09-27 发布 v0.6.0 后归档，不再维护。本分支在 v0.6.0 基础上适配 Claude Code 2.1.292 与 Windows，保留原 MIT 许可证与作者署名。分割线以下为上游英文文档，安装地址与模型名已按本分支更新。

## 使用边界

- 插件只在本机启动已安装的官方 `claude` 可执行文件，使用其公开文档中的非交互模式（`claude -p` / `--print`）；登录与认证全部由 Claude Code 自身完成。
- 插件不读取、复制或转发 Claude 凭据与 OAuth token，不模拟官方客户端流量，不使用 Anthropic API key，也不使用 Agent SDK。
- 每位使用者使用自己的 Claude 订阅账号，用量计入该账号的套餐额度。不支持共享账号。
- Anthropic 帮助中心 [Use the Claude Agent SDK with your Claude plan](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan)（2026-10-09 查阅）原文：

  > **Update October 7, 2026:** … You can still use the Claude Agent SDK, `claude -p`, and third-party apps with your subscription limits.
  >
  > **Update June 15, 2026:** We've paused the previously-announced changes to Claude Agent SDK usage. For now, nothing has changed: Claude Agent SDK, `claude -p`, and third-party app usage still draw from your subscription limits.

  [Monthly API credits for Max and Team plans](https://support.claude.com/en/articles/17154008) 说明：以订阅登录时，`claude -p` 的用量计入套餐额度，不使用 API 额度。
- 上述说明带有 “For now” 表述，政策可能调整。使用前以 Anthropic 当前的条款与帮助中心说明为准。

## 与上游 v0.6.0 的差异

| 改动 | 上游 v0.6.0 | 本分支 |
| --- | --- | --- |
| 图片传输 | 图片写入私有目录，在 prompt 末尾以 `@"路径"` 引用。Claude Code 2.1.292 对超过 256 KiB 的 `@` 文件既不附图也不报错；附件叙述排在整段记录之前，每加一张图都重写缓存前缀 | 图片以 base64 块经 stdin 发送，紧跟引用它的记录；缓存断点放在最后一条文本记录上。每次出现都计入 20 张、单张 20 MB、总计 100 MB 的限制 |
| 内置插件隔离 | 2.1.292 在 print 模式默认加载 `cc-plugin-plugin-authoring`，初始化隔离检查报 `unexpected customizations`，所有请求失败 | `--settings` 中加入 `"plugin-authoring@builtin": false` |
| 摘要请求缓存 | Pi 对压缩、分支摘要传 `cacheRetention: "none"` 时，Claude Code 仍在尾部放 1h 断点，整段摘要写入缓存 | 这类请求给 Claude 子进程设 `DISABLE_PROMPT_CACHING=1`，缓存写入为 0；普通请求不受影响 |
| 模型选择器 | 列出 `sonnet`、`fable`、`opus`、`haiku` 四个别名 | 列出别名在 2.1.292 中解析到的全名 `claude-sonnet-5-5`、`claude-fable-5-1`、`claude-opus-5-5`、`claude-haiku-4-5`；别名换到新型号时，doctor 提示 `picker still offers <旧 id>` |
| 会话图片目录 | 每个会话维护私有图片目录与请求租约 | 图片不再落盘，删除图片目录与租约代码。旧版本遗留的图片目录在 Linux 与 macOS 上由过期目录回收处理；Windows 不执行过期目录回收，遗留目录不会自动删除，需在确认旧版本 Pi 进程全部退出后，手动删除临时目录中的 `pi-claude-code-provider-images-*` |
| 维护脚本 | `capture-claude-breakpoints.js`、`model-matrix.js` 按别名查型号表 | 随型号全名一起更新，避免启动即失败 |
| doctor 类型检查 | 较新 Pi 的 `ProviderModelConfig` 为联合类型，读取 `contextWindow` 无法通过类型检查 | 增加类型判断 |
| 单元测试 | 按图片落盘与别名编写 | 按内联图片与型号全名更新 |

兼容性变化：旧会话中保存的 `pi-claude-code-provider/opus` 等别名 id 不再可选，恢复这类会话需重新选择模型。完整记录见 [CHANGELOG.md](CHANGELOG.md)。

## 已验证环境

| 项 | 版本 |
| --- | --- |
| 系统 | Windows 11 x64 |
| Pi | 0.99.1 |
| Claude Code | 2.1.292 |
| Node.js | 24.12.0 |

`npm test`（Windows）：382 条，348 通过，34 条为平台相关跳过，0 失败；`npm run check` 通过。上游的付费验证矩阵（`npm run test:paid:*`）未在本分支运行，doctor 因此把上述 Pi 与 Claude Code 版本标为 unverified。Linux 与 macOS 未在本分支验证。

## 安装

```bash
pi install git:github.com/syansunyang-collab/pi-claude-code-provider@v0.6.0-fork.1
```

已安装 npm 版上游时，先执行 `pi remove npm:pi-claude-code-provider`。

Claude Code 固定在 2.1.292，并在 Claude Code 设置的 `env` 中将 `DISABLE_AUTOUPDATER` 设为 `"1"`，关闭自动更新：

```bash
claude install 2.1.292
```

Windows 下 `claude` 若是 npm 生成的 `.cmd` 包装脚本，需安装原生 `claude.exe`，或将 `PI_CLAUDE_CODE_PROVIDER_PATH` 设为 `claude.exe` 的完整路径。

## 使用

- 在 `/model` 中选择 `pi-claude-code-provider` 下的型号，或执行 `pi --model pi-claude-code-provider/claude-sonnet-5-5`。
- 安装插件、升级 Claude Code 后运行 `/pi-claude-code-provider-doctor`，检查版本、型号与工具桥接，不消耗订阅额度。
- 已有其他搜索工具时设 `PI_CLAUDE_CODE_PROVIDER_WEB_SEARCH=off`，插件不再注册自带的 Claude 搜索工具。

## 已知限制与排障

- **升级 Claude Code 后所有请求报 `Claude Code loaded unexpected customizations (plugins: cc-plugin-…)`**：新版本加入了内置插件。按报出的名称，把 `<去掉 cc-plugin- 前缀的名称>@builtin: false` 加入 `src/claude-args.ts` 的 `enabledPlugins`。
- **所有请求报 `Claude Code initialized with an unexpected tool set`**：Pi 声明的工具与 Claude Code 实际加载的工具不一致。Claude Code 2.1.292 会跳过参数 schema 顶层含 `oneOf`、`anyOf` 或 `allOf` 的 MCP 工具，插件的工具集校验随即拒绝请求。比对报错中的 expected 清单与实际工具，找出缺失的工具，把它的参数 schema 改为顶层 `type: "object"` 加 `properties`，字段互斥或条件必填改在执行入口校验。
- **别名换到新型号**：doctor 提示 `picker still offers <旧 id>` 时，按新型号更新 `src/catalog.ts`。
- **运行单元测试**：先执行 `npm run setup:dev`。测试假定 `PI_CLAUDE_CODE_PROVIDER_WEB_SEARCH` 为默认值，环境中设为 `off` 时，需在测试进程中去掉该变量。

---

# Upstream README (0.6.0, install and model names updated for this fork)

> **This project is mothballed after the release of v0.6.0.** Pi and Claude Code are both extremely fast-moving projects that publish breaking changes regularly, and this was a hobby project rather than a professional venture, so I have other plans for my time and my tokens. I encourage people to look for other providers, such as [pi-claude-bridge](https://github.com/elidickinson/pi-claude-bridge), which is built on the Agent SDK. Please do not report further issues or submit pull requests. If Pi and Claude Code stabilize in future months, I may revisit this project. I thank my users for their kind words and wish everyone good luck with their own efforts.

A [Pi](https://pi.dev) package that creates a provider for Claude family models from a subscription-authenticated Claude Code installation by launching Anthropic's installed `claude` executable in documented non-interactive print mode. Pi remains fully in charge of the session: branching, compaction, and history behave like any other Pi provider, and every tool runs visibly in Pi — the Claude process can propose tool calls but never execute anything on its own. The goal is simple: the convenience of your Claude subscription in Pi, with the fewest possible surprises.

This package never imitates private OAuth traffic, does not use the Agent SDK, and does not modify Claude's internal session files. It never reads Claude credentials or uses an Anthropic API key.

This project was developed using frontier AI models under human guidance. Almost all of the docs and code were written by machines except for this introductory material. The project may be over-engineered in some respects; that's fine. If you enjoy this package, please star it on github.

## Requirements

- [Pi](https://pi.dev) 0.86.1 or newer, installed from npm or a standalone build
- Claude Code 2.1.281 or newer
- Claude Code logged in to an eligible Pro, Max, Team, or Enterprise claude.ai subscription
- Node.js 22.19 or newer only when Pi itself is installed from npm; the standalone build needs no separate Node installation

These are minimum versions; see the [compatibility baseline](DEVELOPING.md#compatibility-baseline) for tested versions and platforms. The doctor warns about older versions and unverified platforms.

The provider requires first-party subscription authentication. API keys and routing through Bedrock, Vertex, or Foundry are unsupported. If `claude` is not on `PATH`, set `PI_CLAUDE_CODE_PROVIDER_PATH` to its executable path.

## Install

```bash
pi install git:github.com/syansunyang-collab/pi-claude-code-provider@v0.6.0-fork.1
```

To install directly from GitHub's default branch:

```bash
pi install git:github.com/syansunyang-collab/pi-claude-code-provider
```

Add `-l` for a project-local installation. Pi loads project packages only after the project is trusted; use `pi config` to enable or disable the extension.

For a local checkout, use `pi install /absolute/path/to/pi-claude-code-provider`. The startup `[Extensions]` list shows `syansunyang-collab/pi-claude-code-provider` for Git and `pi-claude-code-provider` for npm or a local checkout with that directory name. A renamed checkout shows its directory name. These labels apply to npm and standalone Pi alike.

## Use

Open `/model` and choose `claude-sonnet-5-5`, `claude-fable-5-1`, `claude-opus-5-5`, or `claude-haiku-4-5` under `pi-claude-code-provider`.

To select one directly:

```text
/model pi-claude-code-provider/claude-sonnet-5-5
```

From the command line, use `pi --model pi-claude-code-provider/claude-sonnet-5-5`.

Sonnet, Fable, and Opus support Pi thinking levels from `low` through `max`. Haiku uses Claude Code's default thinking, even when Pi shows thinking as off. Sonnet, Fable, and Opus have a 1M context window on every plan, including Pro; Haiku has 200K.

Fable availability and billing vary by subscription tier; see Anthropic's [Fable plan policy](https://support.claude.com/en/articles/15424964-claude-fable-5-on-your-plan).

To see which model served a response, inspect `responseModel` in Pi's JSON output.

Pi's active tools are advertised through MCP. If the model omits the MCP prefix on exactly `bash`, `read`, `edit`, or `write`, the provider accepts that name only when the same lowercase Pi tool is active. Bare capitalized names such as `Bash` and bare names of other tools remain errors. Arguments must still follow Pi's advertised schema; the provider does not translate Claude Code's built-in argument fields or timeout units.

After installation or an upstream update, run:

```text
/pi-claude-code-provider-doctor
```

The doctor checks versions, model aliases, and the tool bridge without consuming subscription quota. It names the provider version Pi actually loaded and its install directory, which exposes an older project-local or duplicate installation. It also reports recent prompt-cache reuse and context-window mismatches. Its last-request metrics describe the request whose process cleanup and lifecycle finished most recently; overlapping requests can finish out of start order, and a terminal response can appear before its metrics finalize.

Run `/pi-claude-code-provider-doctor report` for a content-free diagnostic report. Inspect it before sharing it.

The `pi_claude_code_provider_web_search` tool uses Claude's WebSearch and WebFetch. It always uses Sonnet at medium effort, regardless of the selected model, and has a three-minute limit. Pi offers it to every model, including other providers' models, so each call consumes Claude subscription capacity. Its prompt guidance defers to any other web-search tool you have. To remove it entirely, set `PI_CLAUDE_CODE_PROVIDER_WEB_SEARCH=off`; for a single launch, `pi --exclude-tools pi_claude_code_provider_web_search` also works. If it is unexpectedly unavailable, run the doctor, which reports its state, then check that variable, Pi's tool filters, and whether another extension owns the name.

## Subscription usage

Provider and web-search requests consume Claude subscription capacity; canceling a running request may still consume it. Optional usage credits may incur additional spend after plan limits. The package reports token counts when available, but shows zero monetary cost because it cannot determine subscription billing.

This project uses Anthropic's documented [`claude --print` interface](https://code.claude.com/docs/en/cli-reference). Anthropic explains [subscription limits for third-party usage](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan) and [usage credits](https://support.claude.com/en/articles/11145838-use-claude-code-with-your-pro-or-max-plan).

Pi shows Claude's rate-limit warnings and reset times when available.

## Compatibility limitation

Claude Code's public headless protocol cannot accept arbitrary past assistant messages or tool results. The provider therefore sends Pi's current history on every request. Pi still owns branching, compaction, and tool execution, but this transport uses more context than Anthropic's Messages API. See [DESIGN.md](DESIGN.md#compatibility-and-performance) for caching and performance details.

Images remain available throughout the current Pi context. Each request allows up to 20 images, subject to size limits; [DESIGN.md](DESIGN.md#request-and-transcript-transport) describes their transport.

## Configuration

| Variable | Purpose |
| --- | --- |
| `PI_CLAUDE_CODE_PROVIDER_ACKNOWLEDGED_PLATFORM` | Hide the startup advisory for one exact platform/architecture (for example `linux/arm64`). The doctor still reports its verification status. |
| `PI_CLAUDE_CODE_PROVIDER_BORROW_SOLE_DIRECTORY` | `on` lets a tool-bearing side request without a cwd declaration borrow the sole registered session's directory. Off by default because that directory may be wrong. |
| `PI_CLAUDE_CODE_PROVIDER_PATH` | Override the `claude` executable path. |
| `PI_CLAUDE_CODE_PROVIDER_METRICS_LOG` | Append content-free request and search metrics as JSONL. |
| `PI_CLAUDE_CODE_PROVIDER_IDLE_TIMEOUT_MS` | Override the five-minute protocol-idle timeout for provider requests, in positive milliseconds. |
| `PI_CLAUDE_CODE_PROVIDER_TOTAL_TIMEOUT_MS` | Override the 30-minute timeout from Claude launch through response processing, including response observers, in positive milliseconds. |
| `PI_CLAUDE_CODE_PROVIDER_MCP_READY_TIMEOUT_MS` | Override the five-second tool bridge readiness timeout, in positive milliseconds. |
| `PI_CLAUDE_CODE_PROVIDER_THINKING_DISPLAY` | `summarized` (default), `omitted` (hide thinking text), or `off` (disable the display request if Claude Code rejects it). |
| `PI_CLAUDE_CODE_PROVIDER_WEB_SEARCH` | `on` (default) or `off`. `off` leaves `pi_claude_code_provider_web_search` unregistered, so no model sees the tool or its prompt guidance. Any other value also leaves it unregistered and shows a warning. Takes effect at the next Pi start or `/reload`. |
| `PI_CLAUDE_CODE_PROVIDER_TRANSCRIPT_BREAKPOINT` | `on` (default) or `off`. Turn it off only if Claude Code rejects excess cache breakpoints; this disables the provider's prompt caching. |

Metrics exclude prompts, messages, queries, output, credentials, stderr, and temporary paths. On POSIX, the log is mode 0600; Windows uses the selected location's ACL.

Claude receives an allowlisted environment, including `CLAUDE_CONFIG_DIR` for a relocated configuration and `NODE_EXTRA_CA_CERTS` for a proxy's CA bundle.

## Security and troubleshooting

Pi packages run with your permissions; review the source before installation. Claude runs in Pi's session working directory and can read some project files at startup. Its proposed file and shell actions run as visible Pi tools. The provider suppresses user and project Claude customizations, but administrator-managed settings, hooks, and MCP policy can still run. See [DESIGN.md](DESIGN.md#what-claude-code-adds-on-its-own) for startup behavior and [SECURITY.md](SECURITY.md) for vulnerability reporting.

Tool-bearing side requests need a registered Pi session or a working-directory declaration in the system prompt. That declaration is caller-controlled; see [DESIGN.md](DESIGN.md#process-and-storage-lifecycle) for routing details.

The provider rejects detected tool arguments aimed at its private request and image directories, including equivalent path spellings resolved against the request's working directory. It also recognizes the temporary directory's alias spellings, such as macOS's `/var/folders` for `/private/var/folders`. This guard is a heuristic: it does not follow other symlinks or interpret arbitrary shell expressions. Pi tools run with your permissions.

### Troubleshooting

- **Provider missing or unavailable:** run `/pi-claude-code-provider-doctor`, correct the problem it reports, then run `/reload`.
- **Windows reports Claude Code missing although `claude` works in your shell:** that `claude` is probably a `.cmd` or `.bat` shim, such as an npm install creates, which cannot run without a shell. Install the native Claude Code (`claude.exe`), or set `PI_CLAUDE_CODE_PROVIDER_PATH` to Claude Code's JavaScript entry point.
- **Requests fail right after Claude Code updated:** run the doctor. If it reports your Claude Code version as unverified, install the tested version it names with `claude install <version>`. To avoid a repeat, set `"autoUpdatesChannel": "stable"` in Claude Code's settings, which waits about a week and skips releases with major regressions, or set `DISABLE_AUTOUPDATER` to `"1"` in their `env`. See [Claude Code's setup guide](https://code.claude.com/docs/en/setup).
- **Authentication or subscription failure:** run `claude auth status` and sign in with an eligible subscription. For rate-limit or billing errors, check your subscription limits and usage-credit settings. Logins through `CLAUDE_CODE_OAUTH_TOKEN` are unsupported.
- **Tools fail or requests report `mcp_startup`:** run the doctor to check the tool bridge handshake.
- **"The model refused to complete the request":** Fable, Opus 5.5, and Opus 5 run safety classifiers, most often triggered by cybersecurity and biology content, including context such as project files. Claude Code can re-run a flagged request on another model, but the provider turns that switch off because it cannot publish a response rewritten mid-stream, so the request ends with this error and Pi does not retry it. See Anthropic's [automatic model fallback](https://code.claude.com/docs/en/model-config#automatic-model-fallback).
- **A request keeps failing:** run `/pi-claude-code-provider-doctor report` and inspect the report before sharing it.

## Development and license

See [DEVELOPING.md](DEVELOPING.md), [CONTRIBUTING.md](CONTRIBUTING.md), and [DESIGN.md](DESIGN.md). Licensed under MIT.

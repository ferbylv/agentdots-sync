# meta-sync-manager

<p align="right">
  <a href="./README.md">English</a> | <b>简体中文</b>
</p>

`meta-sync-manager` 用于安全地规划和执行本地 AI Agent 指令、规则与 Skill 资产和 Git 远程仓库之间的精确同步。它提供平台 Profile，同时保留显式路径覆盖能力，适用于非标准目录和项目级布局。

本工具只同步文件，不负责在不同平台之间转换指令格式。

## 支持的平台 Profile

| Profile | 默认根目录 | 默认资产 | 说明 |
| --- | --- | --- | --- |
| `codex` | `~/.codex` | `AGENTS.md`、`rules`、`skills` | 保留原有 `.codex_backups` 备份目录。 |
| `claude-code` | `~/.claude` | `CLAUDE.md`、`rules`、`skills` | 别名：`claude`。 |
| `opencode` | `~/.config/opencode` | `skills` | 配置文件可能包含环境相关内容，应显式选择。 |
| `grok-build` | `~/.grok` | `skills` | 别名：`grok`；配置文件应显式选择。 |
| `codebuddy-code` | `~/.codebuddy` | `rules`、`skills` | 别名：`codebuddy`。 |
| `workbuddy` | 必须显式指定 | `skills` | 使用 `--root` 指向解包后的本地 Skill 或技能包工作区；WorkBuddy 界面导入仍需单独执行。 |
| `custom` | 必须显式指定 | 必须显式指定 | 用于其他兼容 Agent Skills 的目录布局。 |

这些默认值面向用户级资产。同步项目级资产时，请把项目目录作为 `--root`，并通过重复的 `--asset` 参数精确选择项目相对路径。

平台目录依据当前官方文档：[Claude Agent Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)、[OpenCode Skills](https://opencode.ai/v2/docs/skills/)、[Grok Build Skills](https://docs.x.ai/build/features/skills-plugins-marketplaces)、[CodeBuddy Code Skills](https://www.workbuddy.cn/docs/cli/skills) 和 [WorkBuddy 技能导入](https://www.workbuddy.cn/docs/workbuddy/From-Beginner-to-Expert-Guide/Function-Description/Skills-Market)。平台目录和导入方式可能变化，发布前应重新核对这些来源。

## 安全模型

- 计划、平台清单和环境诊断均为只读操作。
- 所有变更模式都必须显式传入 `--yes`。
- apply、restore 或 rollback 替换文件前，会先备份现有的选定本地资产。
- rollback 使用经过校验的 manifest，只恢复 manifest 中存在且本次明确请求的资产。
- 每个根目录的变更命令通过 `.meta-sync-lock` 串行执行。
- 选定资产不能逃逸根目录，也不能使用绝对符号链接或跨资产边界的符号链接。
- push 只暂存和校验选定路径，永远不会强制推送。
- 错误输出会脱敏常见 URL 凭据、Bearer Token 和凭据查询参数。

## 环境要求

- Python 3.9 或更高版本。
- 远程操作要求 `PATH` 中存在 Git。
- 必须在能够访问所选本地根目录的执行环境中运行。只有网页聊天能力、无法访问用户本机文件的产品不能使用本工具同步本地配置。

## 查看平台和运行条件

```bash
python3 scripts/sync.py --list-platforms
python3 scripts/sync.py --platform claude-code --doctor
python3 scripts/sync.py --platform opencode --plan
python3 scripts/sync.py --platform claude-code --apply --plan \
  --remote https://example.com/account/agent-assets.git --branch main
```

默认平台仍为 `codex`，因此不传 `--platform` 的旧命令会继续使用原来的根目录、资产和备份位置。

## 应用、推送与回滚

先运行计划，核对规范平台名、根目录、资产列表、备份目录、远程仓库和分支。确认无误后，只执行已经明确授权的操作：

```bash
python3 scripts/sync.py --platform claude-code --restore --yes \
  --remote https://example.com/account/agent-assets.git --branch main

python3 scripts/sync.py --platform grok-build --push --yes \
  --remote https://example.com/account/agent-assets.git --branch main \
  --message "Update Grok skills"

python3 scripts/sync.py --platform codebuddy-code --rollback --yes
```

需要时可以覆盖默认值：

```bash
python3 scripts/sync.py --platform custom --root /path/to/agent-config \
  --asset instructions.md --asset skills --plan

python3 scripts/sync.py --platform opencode --root /path/to/project \
  --asset AGENTS.md --asset .opencode/skills --plan
```

`--backup-dir` 只能指定所选根目录内部的安全子路径。非 Codex Profile 默认使用 `.meta-sync-backups`。

## 测试与发布检查

```bash
python3 -m unittest discover -s tests -v
python3 scripts/sync.py --help
python3 scripts/sync.py --list-platforms
```

发布前需要校验 `SKILL.md`、检查发行文件清单，并在隔离的临时根目录中测试每个平台 Profile。发行包不得包含 `.DS_Store`、凭据、真实远程地址、生成的备份或锁文件。

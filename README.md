# template

GitHub Actions 可复用 workflow 模板库。业务仓库通过 `workflow_call` 引用本仓库中的 workflow 文件。

## Workflows

| 文件 | 说明 |
|------|------|
| `java.yml` / `java-17.yml` | Java 应用构建与 Docker 镜像 |
| `java-lib.yml` / `java-lib-17.yml` | Java 库构建 |
| `docker.yml` | Docker 镜像构建 |
| `js.yml` | 前端构建 |
| `cpp.yml` | C++ 构建 |
| `artifact-zip.yml` | 产物打包 |
| `auto-sync-features.yml` | 多 feature 分支合并到 dev |
| **`claude-pr-review.yml`** | **PR 自动 Code Review（Claude）** |

## Claude PR Review

基于 [anthropics/claude-code-action@v1](https://github.com/anthropics/claude-code-action)，在 PR 打开/更新时自动审查；默认在存在 blocking 问题时失败 job，可用于分支保护。

### 业务仓库接入

1. 安装 [Claude GitHub App](https://github.com/apps/claude)
2. 在业务仓库添加 Secret：`ANTHROPIC_API_KEY`（使用 API 代理时再加可选的 `ANTHROPIC_BASE_URL`）
3. 新建 `.github/workflows/pr-claude-review.yml`（可参考 [examples/claude-pr-review-caller.yml](examples/claude-pr-review-caller.yml)）：

```yaml
name: PR Claude Review

on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]

jobs:
  claude-review:
    uses: YOUR_ORG/template/.github/workflows/claude-pr-review.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

将 `YOUR_ORG/template` 替换为本模板库的实际 `owner/repo`。

4. **合并前必须通过审查**：在目标分支（如 `main`）的 Branch protection 中勾选 **Require status checks**，选择 **`claude-review`**（与 caller 中 `jobs` 的 key 一致）。

### 可选参数

| input | 默认 | 说明 |
|-------|------|------|
| `max_turns` | `15` | Claude 最大对话轮数 |
| `track_progress` | `true` | PR 上显示审查进度 |
| `fail_on_blocking` | `true` | blocking 问题时 job 失败 |
| `use_code_review_plugin` | `false` | 使用官方 code-review 插件（无 JSON gate） |
| `model` | 空 | 指定模型，如 `claude-sonnet-4-6` |
| `extra_review_instructions` | 空 | 追加审查说明 |
| `anthropic_custom_headers` | 空 | 自定义请求头 JSON（设置 `ANTHROPIC_CUSTOM_HEADERS`） |
| `load_template_guidelines` | `true` | 从模板库拉取 `REVIEW.md` / `CLAUDE.md` |
| `template_repository` | 空 | 模板库 `owner/repo`（空则自动从 `uses` 解析） |
| `template_ref` | 空 | 模板库 ref（空则与 `uses` 的 `@ref` 一致） |

| secret | 必填 | 说明 |
|--------|------|------|
| `ANTHROPIC_API_KEY` | 是 | Claude API Key |
| `ANTHROPIC_BASE_URL` | 否 | API 代理基地址 |

示例：

```yaml
jobs:
  claude-review:
    uses: YOUR_ORG/template/.github/workflows/claude-pr-review.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    with:
      extra_review_instructions: |
        重点关注 API 兼容性与数据库迁移脚本。
```

使用自建 API 代理（或兼容 Anthropic API 的网关）时，在业务仓库配置 Secret `ANTHROPIC_BASE_URL` 并传入：

```yaml
jobs:
  claude-review:
    uses: YOUR_ORG/template/.github/workflows/claude-pr-review.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      ANTHROPIC_BASE_URL: ${{ secrets.ANTHROPIC_BASE_URL }}
```

### 审查规范（金融）

| 文件 | 用途 |
|------|------|
| [REVIEW.md](REVIEW.md) | **PR 审查专用**，定义 Important/Nit、必查项；优先级最高 |
| [CLAUDE.md](CLAUDE.md) | 通用项目规范 |

**无需在业务仓库复制**：`load_template_guidelines: true`（默认）时，workflow 会根据 `uses: YOUR_ORG/template/...@ref` 自动从模板库拉取上述文件到 runner，再交给 Claude 阅读。

- 业务仓库**已有**根目录 `REVIEW.md` / `CLAUDE.md` → **以业务仓库为准**（覆盖模板）
- 业务仓库**没有** → 使用模板库当前 ref 上的默认版本
- 仅某项目要定制 → 在该业务仓提交自己的 `REVIEW.md` 即可

同组织私有库需保证 `GITHUB_TOKEN` 能读模板库（通常可复用 workflow 所在模板仓权限）；跨组织时可设 `load_template_guidelines: false` 并在业务仓自备规范文件。

### 说明

- 默认模式会要求 Claude 写入 `.github/claude-review-result.json`，并由 gate 步骤解析；**推荐用于合并门槛**。
- `use_code_review_plugin: true` 时使用官方插件，评论更丰富，但**不会**根据 JSON 自动 fail job；仅适合“只评论、不挡合并”的场景。

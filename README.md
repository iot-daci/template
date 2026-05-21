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
    permissions:
      contents: read
      pull-requests: write
      issues: write
      id-token: write
    uses: YOUR_ORG/template/.github/workflows/claude-pr-review.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    with:
      model: claude-sonnet-4-6   # 改成你的 API/代理支持的模型 ID
```

将 `YOUR_ORG/template` 替换为本模板库的实际 `owner/repo`（例如 `iot-daci/template`）。

**若报错** `is only allowed 'pull-requests: none'`：说明 caller 的 job 未声明 `permissions`，按上面补上即可。若仓库 Settings → Actions → General 里 workflow 权限为只读，也需改为 **Read and write**。

4. **合并门槛（按 GitHub 计划选择）**：
   - **公开仓库 + 免费版**：可在 Branch protection 中勾选 **Require status checks** → `claude-review`（或 `PR Claude Review / claude-review`）。
   - **私有仓库 + 免费版**：通常**无法**使用「必须通过指定 status check」（需 Pro/Team）。见下方 [免费版替代方案](#免费版替代方案)。

### 免费版替代方案

| 方式 | 说明 |
|------|------|
| **自动审查 + 人工合并** | 保留 `on.pull_request`，合并前人工看 PR 的 Checks 是否通过、评论是否处理（流程约束，非技术强制） |
| **仅手动审查** | 去掉 `pull_request`，只保留 `workflow_dispatch`，在 Actions 页输入 PR 编号运行（见 [examples/claude-pr-review-caller.yml](examples/claude-pr-review-caller.yml)） |
| **基础分支保护（免费私有库可用）** | Settings → Branches：禁止直接 push 到 `main`、**Require a pull request before merging**（不勾选 required checks 也能强制走 PR） |
| **CODEOWNERS** | 要求指定人 approve（与 Claude 审查互补，不依赖 status check） |
| **PR 标签** | 审查通过后由维护者打 `review-passed` 标签再合并（可配合自动评论里的 summary） |

手动触发有三种方式：

| 方式 | 做法 |
|------|------|
| **PR 评论 `/review`** | 复制 [examples/claude-pr-review-on-comment.yml](examples/claude-pr-review-on-comment.yml) 到业务仓，在 PR 对话里评论 `/review` |
| **Actions 手动 Run** | 见 [examples/claude-pr-review-caller.yml](examples/claude-pr-review-caller.yml) 的 `workflow_dispatch` |
| **PR 自动** | 同上 caller 的 `pull_request` 触发（可关掉，只留 `/review`） |

`/review` 示例 workflow：

```yaml
on:
  issue_comment:
    types: [created]

jobs:
  claude-review:
    if: |
      github.event.issue.pull_request &&
      github.event.comment.user.type != 'Bot' &&
      (github.event.comment.body == '/review' || startsWith(github.event.comment.body, '/review '))
    permissions:
      contents: read
      pull-requests: write
      issues: write
      id-token: write
    uses: YOUR_ORG/template/.github/workflows/claude-pr-review.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    with:
      pr_number: ${{ github.event.issue.number }}
      model: claude-sonnet-4-6
```

评论 `/review` 或 `/review 请重点看支付模块` 均可触发；仅匹配以 `/review` 开头的评论，避免误触。

### 可选参数

| input | 默认 | 说明 |
|-------|------|------|
| `max_turns` | `50` | 最大对话轮数；仍超限可设 `60`～`80` 或缩小 PR |
| `post_inline_comments` | `false` | `true` 时在 diff 行发评论（**每条多占轮数**，大 PR 易超限） |
| `track_progress` | `true` | PR 上显示审查进度（`workflow_dispatch` 下自动关闭） |
| `fail_on_blocking` | `true` | blocking 问题时 job 失败 |
| `use_code_review_plugin` | `false` | 使用官方 code-review 插件（无 JSON gate） |
| `model` | `claude-sonnet-4-6` | **务必**与令牌/代理可用模型一致；403 时在 caller 中覆盖 |
| `extra_review_instructions` | 空 | 追加审查说明 |
| `anthropic_custom_headers` | 空 | 自定义请求头 JSON（设置 `ANTHROPIC_CUSTOM_HEADERS`） |
| `require_review_md` | `true` | 是否要求当前业务仓库根目录存在 `REVIEW.md` |

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

| 文件 | 位置 | 说明 |
|------|------|------|
| [REVIEW.md](REVIEW.md) | **业务仓库根目录** | PR 审查专用，Important/Nit；**必需**（可复制模板库示例再改） |
| [CLAUDE.md](CLAUDE.md) | **业务仓库根目录** | 通用规范，**可选** |

Workflow 只读取**当前被审查项目**（PR 分支 checkout 后的仓库），**不会**从 `uses: org/vcard@main` 的 workflow 所在仓拉取规范。

在 `vbiz-forge/vcard` 等业务项目中：

1. 将 `REVIEW.md`（及可选 `CLAUDE.md`）放在**项目根目录**并提交到 PR 分支  
2. 可从本模板库 [REVIEW.md](REVIEW.md) / [CLAUDE.md](CLAUDE.md) 复制后按项目修改  

若暂时不需要 `REVIEW.md` 也可设 `require_review_md: false`（不推荐金融项目）。

### 说明

- 默认模式会要求 Claude 写入 `.github/claude-review-result.json`，并由 gate 步骤解析；**推荐用于合并门槛**。
- `use_code_review_plugin: true` 时使用官方插件，评论更丰富，但**不会**根据 JSON 自动 fail job；仅适合“只评论、不挡合并”的场景。
- 若报错 `Reached maximum number of turns`：① 保持 `post_inline_comments: false`（默认）② 增大 `max_turns` ③ 拆小 PR ④ 规则复杂时精简 `REVIEW.md` 或删「每次必查」中与项目无关的项。根因通常是 **inline comment 过多 / diff 过大**，不是规则字数本身。
- 若报错 `403` / `无权访问模型 claude-opus-4-7`：在 caller 显式指定 `model`（勿依赖 action 默认 opus），例如 `model: claude-sonnet-4-6` 或代理文档中的模型名。

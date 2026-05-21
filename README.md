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
| **`claude-pr-review.yml`** | **PR Code Review（Claude，/review 触发，预先生成 diff 文件）** |
| **`claude-pr-review-auto.yml`** | **PR Code Review（Claude，/review 触发，由 Claude 自行通过 gh 拉取 diff）** |

## Claude PR Review

基于 [anthropics/claude-code-action@v1](https://github.com/anthropics/claude-code-action)，在 PR 中评论 **`/review`** 触发审查，结果以 **PR 评论** 呈现（不写 `claude-review-result.json`）。

提供两种模板，按需选择：

| 模板 | 说明 | 适用 |
|------|------|------|
| `claude-pr-review.yml` | workflow step 中预先 `gh pr diff` 写入 `.github/claude-pr-diff.patch`，Claude 直接读文件 | diff 较大或希望减少 Claude 调用 `gh` 的轮数 |
| `claude-pr-review-auto.yml` | 不预先生成 diff，Claude 在容器内自行通过 `gh pr view` / `gh pr diff` 拉取 | 流程更简洁，无中间文件；小到中等 PR 推荐 |

### 业务仓库接入

1. 安装 [Claude GitHub App](https://github.com/apps/claude)
2. Secret：`ANTHROPIC_API_KEY`（代理场景可加 `ANTHROPIC_BASE_URL`）
3. 项目根目录添加 `REVIEW.md`（可从本库 [REVIEW.md](REVIEW.md) 复制）
4. 复制其中一个 caller 到业务仓 `.github/workflows/pr-claude-review.yml`：
   - [examples/claude-pr-review-caller.yml](examples/claude-pr-review-caller.yml)（预生成 diff 版）
   - [examples/claude-pr-review-auto-caller.yml](examples/claude-pr-review-auto-caller.yml)（自动拉取 diff 版）

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

将 `YOUR_ORG/template` 换成实际模板库路径。在 PR **Conversation** 发 `/review` 即可。

**权限报错** `pull-requests: none`：caller job 须声明上面 `permissions`，且仓库 Actions 设为 **Read and write**。

### 可选参数

| input | 默认 | 说明 |
|-------|------|------|
| `pr_number` | （必填） | `/review` 时传 `github.event.issue.number` |
| `model` | `claude-sonnet-4-6` | 与 API/代理可用模型一致 |
| `max_turns` | `50` | 对话轮数上限 |
| `post_inline_comments` | `false` | `true` 发 diff 行内评论（耗轮数） |
| `track_progress` | `true` | PR 上显示进度评论 |
| `require_review_md` | `true` | 是否必须有 `REVIEW.md` |
| `extra_review_instructions` | 空 | 追加审查说明 |

| secret | 必填 | 说明 |
|--------|------|------|
| `ANTHROPIC_API_KEY` | 是 | API Key |
| `ANTHROPIC_BASE_URL` | 否 | 代理地址 |

### 审查规范

| 文件 | 说明 |
|------|------|
| [REVIEW.md](REVIEW.md) | 业务仓根目录，审查规则（Important / Nit） |
| [CLAUDE.md](CLAUDE.md) | 可选，通用编码规范 |

### 常见问题

- **`Reached maximum number of turns`**：`post_inline_comments: false`、增大 `max_turns`、或拆小 PR
- **`403` 模型无权限**：在 `with` 中设置你有权限的 `model`
- **评论无响应**：确认打在 PR 主对话且内容为 `/review` 或 `/review …` 开头

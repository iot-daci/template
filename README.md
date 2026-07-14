# template

GitHub Actions 可复用 workflow 模板库。业务仓库通过 `workflow_call` 引用本仓库中的 workflow 文件。

## Workflows

| 文件 | 说明 |
|------|------|
| `java.yml` / `java-17.yml` | Java 应用构建与 Docker 镜像 |
| `java-lib.yml` / `java-lib-17.yml` | Java 库构建 |
| `docker.yml` | Docker 镜像构建 |
| `js.yml` | 前端构建 |
| `npm-publish.yml` | npm 包发布（pnpm monorepo，阿里云私服） |
| `cpp.yml` | C++ 构建 |
| `artifact-zip.yml` | 产物打包 |
| `auto-sync-features.yml` | 多 feature 分支合并到 dev |
| **`anxiaolong-scan.yml`** | **安小龙 AI 代码审计（打包源码 → 上传 → 轮询 → 下载结果 artifact）** |
| **`claude-pr-review.yml`** | **PR Code Review（Claude，/review 触发，预先生成 diff 文件）** |
| **`claude-pr-review-auto.yml`** | **PR Code Review（Claude，/review 触发，由 Claude 自行通过 gh 拉取 diff）** |

## 安小龙 AI 代码审计

可复用 workflow：将业务仓源码打包上传至安小龙，创建扫描任务并轮询至完成，结果文件以 Actions artifact 保留。

### 业务仓库接入

1. 配置 Secrets（均必填）：
   - `AXL_SERVER_URL`：安小龙基础地址（仅 `https://` 主机，不含接口路径）
   - `AXL_API_KEY`：API Key
   - `AXL_SKILL_ID`：技能 ID（数字）
   - `AXL_ALLOWED_HOSTS`：允许调用的域名白名单（逗号分隔，须包含 `AXL_SERVER_URL` 主机）
2. 复制 [examples/anxiaolong-scan-caller.yml](examples/anxiaolong-scan-caller.yml) 到业务仓 `.github/workflows/`

```yaml
on:
  push:
    branches: ["main"]
  workflow_dispatch:

jobs:
  anxiaolong-scan:
    permissions:
      contents: read
    uses: YOUR_ORG/template/.github/workflows/anxiaolong-scan.yml@main
    secrets:
      AXL_SERVER_URL: ${{ secrets.AXL_SERVER_URL }}
      AXL_API_KEY: ${{ secrets.AXL_API_KEY }}
      AXL_SKILL_ID: ${{ secrets.AXL_SKILL_ID }}
      AXL_ALLOWED_HOSTS: ${{ secrets.AXL_ALLOWED_HOSTS }}
```

将 `YOUR_ORG/template` 换成实际模板库路径。审计结果在 Actions **Artifacts** 中下载（默认名 `anxiaolong-scan-results`）。

| input | 默认 | 说明 |
|-------|------|------|
| `runner_type` | `simple` | 安小龙 runner 类型 |
| `enable_thinking` | `false` | 是否开启 thinking |
| `source_name` | `github-actions` | 任务来源标识 |
| `artifact_name` | `anxiaolong-scan-results` | 结果 artifact 名称 |
| `artifact_retention_days` | `7` | artifact 保留天数 |
| `timeout_minutes` | `120` | Job 超时（分钟） |
| `max_package_mb` | `500` | 源码包最大体积（MB） |
| `max_single_file_mb` | `50` | 单个源文件最大体积（MB） |
| `max_result_files` | `20` | 结果文件数量上限 |
| `max_result_file_mb` | `30` | 单个结果文件最大体积（MB） |
| `max_total_result_mb` | `100` | 结果文件总大小上限（MB） |
| `allowed_result_extensions` | `.docx,.xlsx,.pdf,.txt,.json` | 允许下载的扩展名 |
| `user_input_prefix` | （默认审计提示） | 创建任务时的用户提示前缀 |

| secret | 必填 | 说明 |
|--------|------|------|
| `AXL_SERVER_URL` | 是 | 安小龙 `https://` 基础地址 |
| `AXL_API_KEY` | 是 | API Key |
| `AXL_SKILL_ID` | 是 | 技能 ID |
| `AXL_ALLOWED_HOSTS` | 是 | 域名白名单 |

| output | 说明 |
|--------|------|
| `task_id` | 安小龙任务 ID |
| `task_status` | 最终状态（成功时为 `FINISHED`） |

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

## npm 包发布

基于 `lzk90s/js-develop-env:alpine`（内置 `/root/.npmrc`），发布 pnpm monorepo 中的单个 package。

### 业务仓库接入

1. 复制 [examples/npm-publish-caller.yml](examples/npm-publish-caller.yml) 到业务仓 `.github/workflows/`
2. 按需修改 `paths`、`workdir`、`package_path`、`publish_command`
3. Secret **`NPM_AUTH_TOKEN`**（必填）：阿里云 npm auth token；workflow 会在 publish 前写入 `/root/.npmrc`（字面量 token，避免 pnpm OIDC 回退失败）

```yaml
jobs:
  publish:
    uses: iot-daci/template/.github/workflows/npm-publish.yml@main
    with:
      workdir: admin-portal
      package_path: packages/admin-core
      publish_command: pnpm publish:admin-core
    secrets:
      NPM_AUTH_TOKEN: ${{ secrets.NPM_AUTH_TOKEN }}
```

| input | 默认 | 说明 |
|-------|------|------|
| `workdir` | `.` | monorepo 根目录 |
| `package_path` | （必填） | package 相对路径，用于读 `package.json` 做版本检查 |
| `publish_command` | 空 | 自定义发布命令；空则 `pnpm -C <package_path> publish --no-git-checks` |
| `skip_if_exists` | `true` | 同版本已存在则跳过 |
| `registry_url` | 阿里云 npm 私服 | publish / npm view 使用的 registry |

| secret | 必填 | 说明 |
|--------|------|------|
| `NPM_AUTH_TOKEN` | 是 | 阿里云 npm auth token |

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

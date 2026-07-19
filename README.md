# template

GitHub Actions 可复用 workflow 模板库。业务仓库通过 `workflow_call` 引用本仓库中的 workflow 文件。

## Workflows

| 文件 | 说明 |
|------|------|
| `java.yml` / `java-17.yml` | Java 应用构建与 Docker 镜像；**main 构建成功后**追加 `changelog.md` |
| `java-lib.yml` / `java-lib-17.yml` | Java 库构建与部署 |
| `js.yml` | 前端构建与 Docker 镜像；**main 构建成功后**追加 `changelog.md` |
| `append-changelog.yml` | 构建后写 changelog（由 java/js/docker 等调用）：查关联已合并 PR → prepend → push |
| `actions/require-pr-changelog` | PR 必须含非空 `## Changelog`；业务仓本地 job 调用，Ruleset 勾 **`PR Changelog`** |
| **`code-quality.yml`** | **代码质量检查**（可复用）：`kind=server` 时并行调 `actions/java-unit-test` + `actions/java-spotbugs` |
| `actions/java-unit-test` | Java 单测 + JaCoCo（供 code-quality 等调用） |
| `actions/java-spotbugs` | SpotBugs + FindSecBugs（供 code-quality 等调用） |
| `docker.yml` | Docker 镜像构建；**main 构建成功后**追加 `changelog.md` |
| `npm-publish.yml` | npm 包发布（pnpm monorepo，阿里云私服） |
| `cpp.yml` | C++ 构建与 Docker 镜像；**main 构建成功后**追加 `changelog.md` |
| `artifact-zip.yml` | 产物打包 |
| `auto-sync-features.yml` | 多 feature 分支合并到 dev（可配置 pattern、重建 dev） |
| **`anxiaolong-scan.yml`** | **安小龙 AI 代码审计（打包源码 → 上传 → 轮询 → 下载结果 artifact）** |
| **`claude-pr-review.yml`** | **PR Code Review（Claude，/review 触发，预先生成 diff 文件）** |
| **`claude-pr-review-auto.yml`** | **PR Code Review（Claude，/review 触发，由 Claude 自行通过 gh 拉取 diff）** |
| **`claude-feature-doc-review.yml`** | **Feature 设计文档审查（Claude，feature-* push 且 `dev-doc/docs` 变更时自动审核需求/技术方案）** |

## Build 后 Changelog（main）

`java.yml` / `java-17.yml` / `js.yml` / `docker.yml` / `cpp.yml` 在 **build 成功且当前分支为 `main`** 时，会调用 `append-changelog.yml`：

1. 用 merge commit / SHA 查找关联已合并 PR
2. 优先提取 PR 正文中的 `## Changelog` 段落；否则用标题 + 正文摘要 + commits
3. 将条目 **prepend** 到仓库根目录 `changelog.md` 并 push

打 Docker 镜像的模板（`java` / `java-17` / `js` / `docker` / `cpp`）**必须**传入 `GH_TOKEN`；main 构建成功后会写 `changelog.md`，缺 token 则失败：

```yaml
jobs:
  build:
    uses: iot-daci/template/.github/workflows/java-17.yml@main
    with:
      workdir: server
      docker_context: server/vpay-starter
      docker_image: com.lz.vpay.server
    secrets:
      DOCKER_USER: ${{ secrets.DOCKER_USER }}
      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
      GH_TOKEN: ${{ secrets.GH_TOKEN }}
```

PR 使用业务仓 `.github/pull_request_template.md`（示例：[examples/pull_request_template.md](examples/pull_request_template.md)），必须保留非空 `## Changelog`。

硬约束：业务仓接入 [examples/require-pr-changelog-caller.yml](examples/require-pr-changelog-caller.yml)（本地 job + composite action，避免嵌套长名），Ruleset 勾 **`PR Changelog`**（与 `Code Quality Gate` 并列）。缺段落、空内容或仍是模板占位 `-` 时检查失败。

## Code Quality Gate（Java）

目前仅覆盖 **Java（server）**。业务仓用独立 caller，**只在 PR → main** 触发（不要挂在 feature push 构建流水线上）。

```
PR → main
 └─ Code Quality Checks
      ├─ Detect changes（是否改了 server/**）
      ├─ 有变更则跑 code-quality.yml（单测 + SpotBugs）
      └─ Code Quality Gate  ✅ 分支保护只勾这一项
```

| PR 改动 | Gate |
|---------|------|
| 改了 server | 真跑检测 |
| 未改 server | skip 检测，Gate 直接 pass |
| draft | 整条 workflow 跳过 |

**约定**

1. Workflow **不要**用 `on.pull_request.paths` 过滤（否则未改时 Gate 不会出现）。始终触发，用 `changes` job 判断。
2. 分支保护只勾 `Code Quality Gate`（或业务仓里同名 Gate），**不要**勾 Unit tests / SpotBugs 子 job。
3. 不要用账号 / Org Ruleset 绑这些检查。

```yaml
jobs:
  quality:
    uses: iot-daci/template/.github/workflows/code-quality.yml@main
    with:
      kind: server      # 当前仅实现 server；后续可加 frontend / docs 等
      workdir: server   # 或 yudao-cloud
      # modules: -pl xxx -am
```

示例 caller：[examples/code-quality-caller.yml](examples/code-quality-caller.yml)。

## Auto Sync Features → Dev

将多个 `feature-*` 分支（及可选的 `main`）自动 merge 到 `dev`。冲突时停止后续合并，并 push 冲突前已成功 merge 的部分。

### 业务仓库接入

1. 仓库 Secret 配置 **`GH_TOKEN`**（PAT 或 GitHub App token，需 repo 写权限）
2. 复制 [examples/auto-sync-features-caller.yml](examples/auto-sync-features-caller.yml) 到业务仓 `.github/workflows/`

checkout / push 走 `GH_TOKEN`，**caller 不需要** `permissions: contents: write`（默认 `contents: read` 即可）：

```yaml
jobs:
  sync-dev:
    uses: YOUR_ORG/template/.github/workflows/auto-sync-features.yml@main
    secrets:
      GH_TOKEN: ${{ secrets.GH_TOKEN }}
    with:
      branch_pattern: feature-*
      include_main: true
      force_recreate_dev: false   # 删远程 dev 后重建时设为 true
```

| input | 默认 | 说明 |
|-------|------|------|
| `branch_pattern` | `feature-*` | 要合并的远程分支 glob（不含 `origin/`） |
| `include_main` | `true` | 每次 sync 是否 merge `base_branch` |
| `base_branch` | `main` | 创建/重建 dev 时的基线分支 |
| `target_branch` | `dev` | 集成分支 |
| `force_recreate_dev` | `false` | 从 `base_branch` 重建 dev 后再 merge（配合删远程 dev） |

| secret | 必填 | 说明 |
|--------|------|------|
| `GH_TOKEN` | 是 | PAT 或 GitHub App token，需 `contents: write` |

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

## Claude Feature Doc Review

当往 **`feature-*`** 分支 push，且 **`dev-doc/docs`**（需求 PRD + 技术设计）有变更时，自动触发 Claude 审核。有关联 PR 时结论发到 **一条 PR 评论**（每条问题附可点击的 `path:line` 文档链接）；否则发到对应 commit comment。

> 说明：`anthropics/claude-code-action@v1` 不支持 `push` 事件，本模板改为直接调用 Claude Code CLI（headless）。

### 业务仓库接入

1. 安装 [Claude GitHub App](https://github.com/apps/claude)
2. Secret：`ANTHROPIC_API_KEY`（代理场景可加 `ANTHROPIC_BASE_URL`）
3. 项目根目录添加 `DOC_REVIEW.md`（可从本库 [DOC_REVIEW.md](DOC_REVIEW.md) 复制）
4. 复制 [examples/claude-feature-doc-review-caller.yml](examples/claude-feature-doc-review-caller.yml) 到业务仓 `.github/workflows/feature-doc-claude-review.yml`

```yaml
on:
  push:
    branches:
      - 'feature-*'
    paths:
      - 'dev-doc/docs/**'

jobs:
  claude-doc-review:
    permissions:
      contents: write
      pull-requests: write
      issues: write
      id-token: write
    uses: YOUR_ORG/template/.github/workflows/claude-feature-doc-review.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    with:
      before_sha: ${{ github.event.before }}
      after_sha: ${{ github.sha }}
      branch: ${{ github.ref_name }}
      docs_path: dev-doc/docs
      model: claude-sonnet-4-6
```

将 `YOUR_ORG/template` 换成实际模板库路径。文档目录若不同，改 caller 的 `paths` 与 `docs_path`。

**权限**：caller 须声明上面 `permissions`（无关联 PR 时用 `contents: write` 发 commit comment），且仓库 Actions 设为 **Read and write**。

| input | 默认 | 说明 |
|-------|------|------|
| `before_sha` | （必填） | `github.event.before` |
| `after_sha` | （必填） | `github.sha` |
| `branch` | （必填） | `github.ref_name` |
| `docs_path` | `dev-doc/docs` | 设计文档目录 |
| `base_branch` | `main` | 新建分支时对比基线 |
| `model` | `claude-sonnet-4-6` | 与 API/代理可用模型一致 |
| `max_turns` | `40` | 对话轮数上限 |
| `require_doc_review_md` | `true` | 是否必须有 `DOC_REVIEW.md` |
| `extra_review_instructions` | 空 | 追加审查说明 |

| secret | 必填 | 说明 |
|--------|------|------|
| `ANTHROPIC_API_KEY` | 是 | API Key |
| `ANTHROPIC_BASE_URL` | 否 | 代理地址 |

### 审查规范

| 文件 | 说明 |
|------|------|
| [DOC_REVIEW.md](DOC_REVIEW.md) | 业务仓根目录，需求/技术方案审查规则（Important / Nit） |

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

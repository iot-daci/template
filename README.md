# template

GitHub Actions 可复用 workflow 模板库。业务仓库通过 `workflow_call` 引用本仓库中的 workflow 文件。

## Workflows

| 文件 | 说明 |
|------|------|
| `java.yml` / `java-17.yml` | Java 应用构建与 Docker 镜像；**main 构建成功后**追加 `changelog.md` |
| `java-lib.yml` / `java-lib-17.yml` | Java 库构建与部署 |
| `js.yml` | 前端构建与 Docker 镜像；**main 构建成功后**追加 `changelog.md` |
| `notify-image.yml` | 构建后在关联 PR 评论 Docker 镜像 tag（由 java/js/docker 等调用；不再写 changelog.md） |
| `actions/validate-pr` | 一次校验 PR 标题（`type(scope): subject`）+ 非空 `## Changelog`；Ruleset 勾 **`PR Check`** |
| **`unit-tests.yml`** | **PR 单测**（可复用）：`language=java\|js\|go`；只应由 PR→main caller 调用 |
| `actions/java-unit-test` | Java 单测 + JaCoCo |
| `actions/js-unit-test` | JS 单测（pnpm/npm/yarn） |
| `actions/go-unit-test` | Go 单测（`go test ./...`） |
| `actions/java-spotbugs` | SpotBugs（可选；默认不挂在 unit-tests 上） |
| `actions/resolve-docker-image` | 解析 registry / 镜像名 / tag |
| `actions/docker-build-prep` | 登录 registry、timestamp、metadata、QEMU/Buildx |
| `actions/docker-build-push` | 标准 build-push（inline registry cache） |
| `actions/build-job-summary` | 构建 Job Summary |
| `actions/fix-workspace-permissions` | self-hosted container chown workspace |
| `docker.yml` | Docker 镜像构建；**main 构建成功后**追加 `changelog.md` |
| `npm-publish.yml` | npm 包发布（pnpm monorepo，阿里云私服） |
| `cpp.yml` | C++ 构建与 Docker 镜像；**main 构建成功后**追加 `changelog.md` |
| `artifact-zip.yml` | 产物打包 |
| `auto-sync-features.yml` | 多 feature 分支合并到 dev（可配置 pattern、重建 dev） |
| `auto-sync-gray.yml` | main + gray-* 分别合并到 gray 和 dev，无灰度分支则跳过 |
| **`anxiaolong-scan.yml`** | **安小龙 AI 代码审计（打包源码 → 上传 → 轮询 → 下载结果 artifact）** |
| **`claude-pr-review.yml`** | **PR Code Review（Claude，/review 触发，预先生成 diff 文件）** |
| **`claude-pr-review-auto.yml`** | **PR Code Review（Claude，/review 触发，由 Claude 自行通过 gh 拉取 diff）** |
| **`claude-feature-doc-review.yml`** | **Feature 设计文档审查（Claude，feature-* push 且 `dev-doc/docs` 变更时自动审核需求/技术方案）** |
| **`api-auto-test.yml`** | **Python/pytest API 自动化测试**（可复用）：装依赖 → 注入 `.env`/mTLS 证书 → 跑测 → **Job Summary** + JUnit artifact |

## Self-hosted runner（container job 权限）

### Docker 镜像仓库

打镜像 workflow（`java` / `java-17` / `js` / `docker` / `cpp`）通过 composite **`actions/resolve-docker-image`** 解析镜像地址，**优先读 runner 环境变量**（`actions-runner/.env`），未设置则用 GHCR 默认：

| 变量 | 默认 | 说明 |
|------|------|------|
| `REGISTRY` | `ghcr.io` | 镜像仓库 host |
| `IMAGE_NAMESPACE` | `<repository_owner>` | registry 与镜像名之间的路径，如 `iot-daci` 或 `team-gs` |
| `IMAGE_TAG` | 当前 git 分支名 | 主 tag（push + cache-from） |
| `CACHE_TAG_FALLBACK` | `main` | cache-from 备用 tag |
| `DOCKER_USERNAME` | — | 非 GHCR 时登录用户名（写 `.env`） |
| `DOCKER_PASSWORD` | — | 非 GHCR 时登录密码（写 `.env`） |

默认完整镜像名：

```text
ghcr.io/<repository_owner>/<docker_image>:<IMAGE_TAG>
```

示例（切回阿里云 ACR + 固定 main tag）：

```bash
REGISTRY=registry.cn-hangzhou.aliyuncs.com
IMAGE_NAMESPACE=team-gs
IMAGE_TAG=main
DOCKER_USERNAME=your-acr-user
DOCKER_PASSWORD=your-acr-pass
```

- `REGISTRY=ghcr.io` 时：登录用 `GH_TOKEN`（需 `write:packages` + `contents`）
- 其他 registry：登录用 runner `.env` 里的 `DOCKER_USERNAME` / `DOCKER_PASSWORD`
- **caller job** 须声明 `permissions: packages: write`（否则 reusable workflow 校验失败）（GHCR 推送需要）

### Docker Buildx 缓存

使用 **inline cache**（`cache-to: type=inline`），从同镜像的 **分支 tag / main tag** 读取（`cache-from: type=registry,ref=…`）。缓存元数据随镜像 push 一并写入，无单独 cache export 步骤。

带 `container:` 的 job 在容器内以 root 写 workspace；self-hosted 的 `_work` 目录会跨 job 保留，下次 checkout 可能出现 `Permission denied`。

各 workflow 在 job 末尾有 **Fix workspace permissions**（仅 `runner.environment == 'self-hosted'` 时执行），将 `$GITHUB_WORKSPACE` chown 回 runner 用户。GitHub-hosted runner 跳过此步。

**每台 self-hosted runner** 在 `actions-runner/.env` 追加（UID/GID 用该机器上 runner 服务用户的实际值）：

```bash
RUNNER_UID=1001   # id -u <runner-user>
RUNNER_GID=1001   # id -g <runner-user>

# 可选：覆盖 Docker 仓库与 tag（见上表）
# REGISTRY=ghcr.io
# IMAGE_NAMESPACE=iot-daci
# IMAGE_TAG=main
```

改 `.env` 后重启 runner 服务。若未配置，job 会打 warning 并跳过 chown。

已卡住时一次性清理（在 runner 主机上）：

```bash
sudo chown -R <runner-user>:<runner-user> /home/<runner-user>/actions-runner/_work
```

## Build 后镜像通知（main）

`java.yml` / `java-17.yml` / `js.yml` / `docker.yml` / `cpp.yml` 在 **build 成功、当前分支为 `main`，且 `generate_changelog` 为 true（默认；兼容旧参数名）** 时，会调用 `notify-image.yml`：

1. 用 merge commit / SHA 查找关联已合并 PR  
2. 在该 PR 下发表（或更新）一条 **sticky 评论**，写明完整镜像名 / tag / `docker pull`  
3. 同时写入 Actions **Job Summary**（无关联 PR 时至少还有 Summary）

**不再**往仓库 `changelog.md` 自动 commit。

同一 PR 多服务构建：每个服务一条评论（marker `djn-image-bot:<service>`），重跑会更新对应评论。

不需要通知时传入 `generate_changelog: false`。

**其它 CI 提交说明**（与 Conventional Commits 对齐）：

| 来源 | 提交首行 |
|------|----------|
| `auto-sync-features.yml` 合并 | `Auto-merge: <branch> → <target>`（hook 已放行） |
| 人工解决 auto-sync 冲突（文档示例） | `chore(merge): resolve conflicts with feature-xxx` |

业务仓若 **强制 squash 合 main**：PR 标题写成 `type(scope): subject`，启用 `examples/validate-pr-caller.yml`；Settings 里 Squash 默认 message 选 **Pull request title**。

评论示例：

```markdown
### Docker image · `com.lz.vcard.admin-portal`

| | |
|---|---|
| **Image** | `ghcr.io/iot-daci/com.lz.vcard.admin-portal:20260720102407` |
| **Tag** | `20260720102407` |

docker pull ghcr.io/iot-daci/com.lz.vcard.admin-portal:20260720102407
```

打 Docker 镜像的模板在默认开启通知时**必须**传入 `GH_TOKEN`（推包 + 评 PR）：

```yaml
jobs:
  build:
    permissions:
      contents: read
      packages: write
      pull-requests: write
    uses: iot-daci/template/.github/workflows/java-17.yml@ghcr
    with:
      workdir: server
      docker_context: server/vpay-starter
      docker_image: com.lz.vpay.server
      # generate_changelog: false  # 可选，关闭 PR 镜像评论
    secrets:
      GH_TOKEN: ${{ secrets.GH_TOKEN }}
```

PR 使用业务仓 `.github/pull_request_template.md`（示例：[examples/pull_request_template.md](examples/pull_request_template.md)），保留非空 `## Changelog`（给人读的发布说明；镜像看 PR 评论）。

硬约束：业务仓接入 [examples/validate-pr-caller.yml](examples/validate-pr-caller.yml)（本地 job + `actions/validate-pr`），Ruleset 勾 **`PR Check`**（与 `Code Quality Gate` 并列）。校验 PR 标题（Conventional Commits）与非空 `## Changelog`；缺段落、空内容或仍是模板占位 `-` 时检查失败。

## Unit Tests Gate（PR → main）

与 **编译打包模板分开**：单测只在「要合进 main 的 PR」上跑，不挂在 feature push / 每次 commit 的打包流水线。

```
PR → main
 └─ Unit Tests
      ├─ Detect changes（按路径 → java / js / go）
      ├─ 有变更则调 unit-tests.yml（language=…）
      └─ Code Quality Gate  ✅ Ruleset 只勾这一项（名称保持兼容）
```

| PR 改动 | Gate |
|---------|------|
| 改了对应语言路径 | 跑该语言单测 |
| 未改相关路径 | 对应 job skip，Gate pass |
| draft | 整条跳过 |

**约定**

1. **不要**用 `on.pull_request.paths` 过滤（否则未改时 Gate 不出现）。始终触发，用 `changes` 判断。
2. Ruleset 只勾 **`Code Quality Gate`**，不要勾各语言子 job。
3. 不要把 `unit-tests.yml` 接到 java.yml/js.yml 等打包 workflow。

```yaml
jobs:
  test-java:
    uses: iot-daci/template/.github/workflows/unit-tests.yml@ghcr
    with:
      language: java
      workdir: server   # 或 yudao-cloud
      # modules: -pl xxx -am
  # test-js / test-go 同理
```

示例 caller：[examples/unit-tests-caller.yml](examples/unit-tests-caller.yml)。

## Auto Sync Features → Dev

将多个 `feature-*` 分支（及可选的 `main`）自动 merge 到 `dev`。冲突时停止后续合并，并 push 冲突前已成功 merge 的部分。

### 业务仓库接入

1. 仓库 Secret 配置 **`GH_TOKEN`**（PAT 或 GitHub App token，需 repo 写权限）
2. 复制 [examples/auto-sync-features-caller.yml](examples/auto-sync-features-caller.yml) 到业务仓 `.github/workflows/`

checkout / push 走 `GH_TOKEN`，**caller 不需要** `permissions: contents: write`（默认 `contents: read` 即可）：

```yaml
jobs:
  sync-dev:
    uses: YOUR_ORG/template/.github/workflows/auto-sync-features.yml@ghcr
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

## 灰度分支自动同步

复制 [examples/auto-sync-gray-caller.yml](examples/auto-sync-gray-caller.yml) 到业务仓
`.github/workflows/auto-sync-gray.yml`，配置 `GH_TOKEN`，引用
`iot-daci/template/.github/workflows/auto-sync-gray.yml@ghcr`。

`main` 或 `gray-*` push 时，共享模板将 `main` 和所有 `gray-*` 分别合并到 `gray`、`dev`。
无 `gray-*` 时两个目标都跳过；一个目标冲突不影响另一个目标。
模板复用 `auto-sync-features.yml` 的合并与冲突处理，按目标分支加锁，避免与 feature → dev 同时推送。
原 feature 工作流继续负责 `main` + `feature-*` → `dev`。

接入前先将两个共享工作流发布到模板仓的 `ghcr` 分支。

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
    uses: YOUR_ORG/template/.github/workflows/claude-pr-review.yml@ghcr
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
    uses: YOUR_ORG/template/.github/workflows/claude-feature-doc-review.yml@ghcr
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
    uses: iot-daci/template/.github/workflows/npm-publish.yml@ghcr
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

## API Auto Test（pytest）

可复用模板：对外部 sandbox/staging 跑 **HTTP API 自动化**（如 vcard-auto-test）。  
**定时（每天一次）写在业务仓 caller**（`schedule` 不能挂在 `workflow_call` 上）。跑完写入 **GitHub Job Summary**（用例总数 / 通过 / 失败 / 跳过 + 失败列表），并上传 JUnit artifact。

### 业务仓库接入

1. 配置 Secrets：
   - **`VCARD_AT_TEST_ENV`**（或自命名）：多行 `KEY=VALUE`，写入工作目录 `.env` 后 `source`（账号、密码、`client_id`/`client_secret`、sign secret、base URL 等）
   - **`VCARD_OPEN_CLIENT_CERT`** / **`VCARD_OPEN_CLIENT_KEY`**（可选）：OpenAPI mTLS PEM 全文
2. 复制 [examples/api-auto-test-caller.yml](examples/api-auto-test-caller.yml) 到业务仓 `.github/workflows/api-auto-test.yml`
3. 按仓库结构调整 `workdir` / `pip_requirements` / `pytest_args`

```yaml
on:
  schedule:
    - cron: "0 1 * * *"   # 每天 UTC 01:00 ≈ 北京 09:00
  workflow_dispatch:

jobs:
  api-test:
    permissions:
      contents: read
    uses: iot-daci/template/.github/workflows/api-auto-test.yml@ghcr
    with:
      workdir: "."
      python_version: "3.11"
      pip_requirements: "requirements.txt"
      pytest_args: '-m "smoke or critical" -v --tb=short'
      summary_title: "VCard API Auto Test"
    secrets:
      TEST_ENV: ${{ secrets.VCARD_AT_TEST_ENV }}
      CLIENT_CERT: ${{ secrets.VCARD_OPEN_CLIENT_CERT }}
      CLIENT_KEY: ${{ secrets.VCARD_OPEN_CLIENT_KEY }}
```

| input | 默认 | 说明 |
|-------|------|------|
| `workdir` | `.` | 测试工程目录 |
| `python_version` | `3.11` | Python 版本 |
| `pip_requirements` | `requirements.txt` | 相对 `workdir` |
| `pytest_args` | `-m "smoke or critical" …` | 不含 `--junitxml`（模板自动加） |
| `junit_path` | `report/junit.xml` | JUnit 输出 |
| `artifact_name` | `api-auto-test-report` | artifact 前缀 |
| `summary_title` | `API Auto Test Summary` | Job Summary 标题 |
| `fail_on_test_failure` | `true` | 失败是否让 job 红 |
| `cert_dir` / `cert_filename` / `key_filename` | `certs` / `client.crt` / `client.key` | 证书落盘路径 |

| secret | 必填 | 说明 |
|--------|------|------|
| `TEST_ENV` | 否* | 多行 env；*无则需 runner 上已有所需变量 |
| `CLIENT_CERT` | 否 | mTLS 证书 PEM |
| `CLIENT_KEY` | 否 | mTLS 私钥 PEM |

跑完后在 Actions 运行页打开 **Summary** 查看结果；详细 XML 在 Artifacts。

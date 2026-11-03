# 复制到业务仓 .github/pull_request_template.md
# 与 append-changelog.yml 配套：CI 提取 ## Changelog 段落写入 changelog.md
#
# 强制 squash 合 main 时：
#   - PR 标题必须是 type(scope): subject（validate-pr / PR Check；标题即进 main 的 commit）
#   - 正文须有非空 ## Changelog；Summary 写人话即可
# 细则：ai-spec/coding/conventional-commits.md

## Summary

<!-- 一句话说明这个 PR 做什么、为什么做 -->

## Changelog

<!--
合入 main 且构建镜像成功后，CI 会把本段写入仓库根目录 changelog.md。
请用面向发布的说明（用户/运维可读），不要贴大段 diff。
无发布影响时可写：无对外变更 / internal only
-->

-

## Type of change

- [ ] Feature
- [ ] Bug fix
- [ ] Refactor / chore
- [ ] Docs only
- [ ] Hotfix / hotfix

## AI / 编码规范自检

<!-- 真源：standard-spec-rules/ai-spec/；业务仓 sync 后也会合并本段 -->

- [ ] **PR 标题**符合 `type(scope): subject`（squash 后进 main；见 `coding/conventional-commits.md`）
- [ ] 符合 `ai-spec/AGENTS.md` 及适用规范
- [ ] 逻辑阶段有空行、无压缩语句；必要语义注释完整，符合 `coding/code-readability-and-style.md`
- [ ] 无假想扩展点、无意义包装类或转发链，符合 `coding/simple-design.md`
- [ ] 无无关重构、无 secrets / 凭证
- [ ] 若改 `foundation-platform` 共享 npm 包：已升版并同步引用方
- [ ] 核心业务（流水线/编排 Service）：类头、分块、入口语义注释已补全（若适用）
- [ ] 业务逻辑变更有配套单元测试，且本地可跑（若适用）
- [ ] 若改 Gateway 读 body Filter：未新增 join+release 共享 Netty buffer；优先 `cacheRequestBody`（见 `ai-spec/coding/gateway-request-body-caching.md`）

## Test plan

- [ ]

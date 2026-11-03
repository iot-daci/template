# 复制到业务仓 .github/pull_request_template.md（真源也可 sync 自 standard-spec-rules/ai-spec/snippets/）
# 与 append-changelog.yml / validate-pr（PR Check）配套

<!--
【PR 标题】强制 squash 合 main 时，标题 = 进 main 的 commit，必须为：

  type(scope): subject

允许的 type（选一个）：
  feat      新功能
  fix       缺陷修复（含 hotfix）
  refactor  重构（无行为变化）
  chore     杂务（依赖、changelog、工具、非功能改动等）
  docs      文档
  test      测试
  perf      性能
  ci        CI 配置
  build     构建 / 打包
  style     纯格式（不影响逻辑）
  revert    回滚

示例：
  feat(card): ppShareCardApi 支持共享预算
  fix(payment): 修复取消支付幂等校验
  chore(mybatis): 逻辑删除补 delete_ts
  docs(readme): 补充部署说明

规则：scope 必填、小写；type(scope): 后有空格再写 subject。
CI job「PR Check」会校验标题 + 下方 ## Changelog。
细则：ai-spec/coding/conventional-commits.md
-->

## Summary

<!-- 一句话说明这个 PR 做什么、为什么做（人话即可，不必套 type(scope)） -->

## Changelog

<!--
合入 main 且构建镜像成功后，CI 会把本段写入仓库根目录 changelog.md。
请用面向发布的说明（用户/运维可读），不要贴大段 diff。
无发布影响时可写：无对外变更 / internal only
-->

-

## AI / 编码规范自检

<!-- 真源：standard-spec-rules/ai-spec/；业务仓 sync 后也会合并本段 -->

- [ ] **PR 标题**已按文件顶部注释写成 `type(scope): subject`（squash 后进 main）
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

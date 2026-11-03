# 复制到业务仓；真源：standard-spec-rules/ai-spec/snippets/pull_request_template.md
# 配套：validate-pr（PR Check）+ append-changelog.yml

<!--
【PR 标题】= 进 main 的 commit（强制 squash），必须：

  type(scope): subject

type：feat | fix | refactor | chore | docs | test | perf | ci | build | style | revert
示例：feat(card): 支持共享预算 · fix(payment): 修复取消支付幂等

scope 必填、小写。CI「PR Check」校验标题 + 下方 Changelog。
细则：ai-spec/coding/conventional-commits.md
-->

## Changelog

<!--
发布说明（写入 changelog.md），与标题分工不同：
- 标题：给 git / 开发看的 type(scope): subject
- 这里：给用户/运维看的变更说明；无对外影响写「无对外变更」
-->

-

## AI / 编码规范自检

<!-- 真源：standard-spec-rules/ai-spec/ -->

- [ ] **PR 标题**已按顶部注释写成 `type(scope): subject`
- [ ] 符合 `ai-spec/AGENTS.md`；可读性 / 简单设计；无 secrets
- [ ] 若改共享 npm 包：已升版并同步引用方
- [ ] 核心业务：类头/分块/入口语义 + 配套单测（若适用）
- [ ] Gateway 读 body：未 join+release 共享 buffer（若适用）

## Test plan

- [ ]

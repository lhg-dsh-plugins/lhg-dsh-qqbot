# BUGFIX — 问答按钮自 0.5.0 起从未成功发送：keyboard 的 group_id 被平台判为无效（上游 issue #56）

## 标题

`ask_user_question` 选项按钮在真实环境从不出现：`question-renderer` 给每个按钮设置的
`group_id: q-<question.id>` 被 QQ 平台判为无效，整条 keyboard 被拒后静默降级为纯文本。
对应上游 issue：https://github.com/tencent-connect/dsh-qqbot/issues/56（open，基于 0.5.2 / c8b9158）

## 证据链

1. 上游 issue #56 报告平台原样报错：
   `ApiError [/v2/users/<openid>/messages]: 请求参数 rows.buttons.group_id 无效`
2. 源码实锤（上游 c8b9158）：
   - `src/features/question-renderer.ts`：`group_id: \`q-${question.id}\``；
     真实 question.id 如 `continue_or_cancel` → group_id 长 20 字符；
   - `src/features/approval-renderer.ts`：`group_id: 'approval'`（8 字符）→ 平台通过，
     与 issue 中「审批按钮正常、问题按钮从不出现」的现象吻合；
   - 现有单测只用一字符 id（`'q'` → `'q-q'`），永远测不出该问题。
3. 离线最小复现（本机，修复前失败 → 修复后通过）：
   脚本以真实 id `continue_or_cancel` 调用 `buildKeyboard()` 并断言按钮不带 `group_id`：
   - 修复前输出：`group_id=q-continue_or_cancel len=20`，2 例断言全失败；
   - 修复后：2 例全通过。
4. 发送失败后 `question-channel.ts` 静默降级为纯文本（仅 `logger.warn`），
   故用户侧只看到正常文字问题，掩盖了该 bug 三个发行版（0.5.0 / 0.5.1 / 0.5.2）。

## 根因

`group_id` 本只负责「点一个后同组其他按钮变灰」的外观效果，其值由不受控的
`question.id` 派生，长度随 id 增长，越过平台有效范围后被整条拒收
（平台是整条拒收而非丢弃单个字段）。状态机本就按 `button_data` 中的 `q`
识别并忽略过期点击，功能不依赖 `group_id`。

## 改动（4 文件，最小 diff）

- `src/features/question-renderer.ts`：删除 `group_id: \`q-${question.id}\``，注释说明原因；
- `src/features/approval-renderer.ts`：删除两个按钮的 `group_id: 'approval'`，同步更新doc 注释
  （与 issue 报告者实机验证过的修复 `fix/keyboard-group-id` 一致：统一移除该字段，防未来重蹈覆辙）；
- `src/features/question-channel.test.ts`：断言由 `expect(btn?.group_id).toBe('q-q')`
  改为 `expect(btn).not.toHaveProperty('group_id')`；
- `src/features/approval-channel.test.ts`：两个 `group_id` 断言同理改 `not.toHaveProperty('group_id')`。

另：`package.json` version 0.5.2 → 0.5.3、repository.url 改指本 fork（见 UPSTREAM.md），非功能改动。

## 复验

环境：node v24.20.0 / pnpm 11.7.0（`pnpm install --frozen-lockfile`），基线与修复后同环境对照：

- 基线（未改上游 c8b9158）：`vitest run` 134/134 通过；`tsc --noEmit` 与
  `tsc -p tsconfig.client.json --noEmit` 均通过；
- 最小复现：修复前 2/2 失败（输出 `group_id=q-continue_or_cancel len=20`）→ 修复后 2/2 通过；
- 修复后：`vitest run` 134/134 通过（无回归）；两次 typecheck 均通过。

## 边界

- 未做真实 QQ 网关/账号验证：平台拒收证据来自上游 issue #56 报告者的实机记录
  （含其修复后私聊/群聊按钮可点击、走按钮路径的实机验证）；本机只做离线逻辑验证。
- 移除 `group_id` 后果仅为外观：点选后同组其他按钮不再自动变灰；
  `click_limit: 1` 与状态机的过期点击识别保持不变，功能不受影响。
- issue #56 的两个提议（降级可观测性、TIM 客户端限制文档）不在本次最小修复范围内。

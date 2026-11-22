# NOTICE

本仓库是 **[Moeblack/dsh-message-edit](https://github.com/Moeblack/dsh-message-edit) 的非官方移植版**，不是原作者发布的分发。

## 来源

- 上游：`dsh-message-edit` 0.2.3（MIT），commit `b78a167064ca`（「性能修复: timeline 按需分层读取 + 客户端刷新风暴治理」）。
- 上游实际发布渠道是 npm 包 `dsh-message-edit`（0.2.1 / 0.2.2 / 0.2.3）。
- 上游 `package.json` 声明 `license: MIT`，但没有 `LICENSE` 文件；本仓库补上 MIT 全文，并把原作者署名放在第一位。

## 我改了什么

只改**构建产物**，没有改上游的 TypeScript 源码（本仓库不含 `src/`，因此也无法用 `npm run build` 重建）：

| 文件 | 改动 |
| --- | --- |
| `index.mjs` | 读取会话日志改用 `ctx.sessionQuery.readSession()`；seed 补 `session/end-seed` 标记；`message-edit/version` 加 `ignorable: true`；合成 `assistant/message` 补 `stream: []` 并去掉 `sourceEventSeqs`；新增 `queuedInputBoundary()` 修正「编辑提问却重跑原任务」；新增版本分支自动归档 |
| `client.js` | 去掉对 `@deepseek-ai/dsh-client-runtime/client` 的依赖（自带 mini snapshot store）；`inject` 改为真实存在的服务名；注册改用 `ctx.slots.inject`；`label` 改为函数；导航改用 `ctx.get('uiWorkspace').openSession()` |
| `package.json` | 版本号 0.3.0；`dsh.client.inject` 去掉不存在的 runtime 包；`private: true`（不发布到 npm，避免占用上游的包名） |

## 边界

- 本版本**没有**发布到 npm，也不打算发布；包名 `dsh-message-edit` 属于原作者。
- 如果原作者发布官方 0.2.x 适配版，请以官方版为准，本仓库可以直接弃用。
- 上游若要求删除本仓库，我会立即删除。

## 兼容性

- 已在 DSH `0.2.2-rc.5`（desktop profile，Windows）实测：GET timeline、编辑用户文本、编辑助手回复、reroll、retry、版本切换、撤销/重施加。
- 未在 0.1.x 上验证；`client.js` 的改动依赖 0.2.x 的插槽与服务名。

# Agent Note: 全局对话背景插件

Status: implemented

[English](2026-08-14-conversation-background-plugin.md) | 中文

## 问题

对话列是一个平板表面：它的背景来自一个主题基础 token（`--dsw-alias-bg-base`），用户无法把自定义图片放到聊天之后。设置壁纸曾一度被硬编码进 `ui-conversation` 的 `ConversationRoot.module.css`，把单个资源路径永久钉进共享组件，且用户没有任何控制。产品需要一个用户可拥有、持久化、进程级的壁纸并带可读性控制。它必须以独立插件的形式发布（上游接受插件，而不是对核心仓库的 PR 改动），并且不得要求修改 stock 的 `ui-conversation` shell——这样任何 dsh 安装都能原样挂载它。

## Decision

**由一个独立客户端插件端到端地拥有该功能，且不修改 shell。** `@deepseek-ai/dsh-client-ui-background` 通过覆盖渲染出的 DOM 来工作：它把 CSS 变量写到文档根，并安装一个插件旗下的 `<style>` 标签。这是经典的「userstyle」扩展模型——stock shell 的源码完全不动，因此插件自足，可以按 `dsh-plugin` 话题发布。

**持久化走设置缝。** 它的 node 半在 `ctx.settings` 上注册 `chat-background` 命名空间（字段 `imageDataUrl`、`opacity`），并把该命名空间加入 `api-proxy` 的 `WEB_SETTINGS_NAMESPACES`，以便浏览器能够读写。背景图以内联 data URL 存储（最长边下采样到不超过 2560px，支持时重编码为 WebP、否则 JPEG），这既让 `settings.yaml` 保持小巧，又不必触碰带 session 授权的附件读取路径。

**浏览器界面是纯 CSS，没有 shell 槽位。** 运行时绑定命名空间并把 `--dsh-bg-image` / `--dsh-bg-wash-opacity` 写到 `document.documentElement`。增强样式表通过 `[data-phase]::before` 在对话列内容之下铺满壁纸，通过 `[data-phase]::after` 叠加可读性遮罩（遮罩强度保证任意图片上方文字可读，并保留每套配色的底色），并通过 `[data-phase][data-phase='active'] [data-composer-seat]` 把 docked 输入框变成悬浮孤岛——把 seat 收窄到卡片宽度、居中、移除底部渐隐，从而输入框旁的壁纸保持可见。规则匹配的是稳定的 stock data 属性（`data-phase`、`data-conversation-scroll`、`data-composer-seat`），因此未来 shell 改名会让覆盖优雅降级，而不是破坏 shell。

**设置界面是一个 General 设置区的行。** 该功能注册进 `settings.general.item`（order 30），提供上传、可读性遮罩滑杆（0–100）和清除；产品文案注册在 `settings.background` locale 命名空间下。

## Alternatives considered

**复用会话附件路径存字节。** 被否决：附件读取是带 session 授权的（`session.attachment`），而背景是全局的；暴露一个无条件按 id 读取会扩大该授权边界。设置在文档里的内联 data URL 不需要新的 host 读取通道。

**给 `ui-conversation` 加一个 `conversation.background` 槽并注册 React 层。** 这是更早的做法；它需要向核心 shell 提交改动，上游不接受，也违背「独立插件」的目标。CSS 覆盖模型彻底移除了这一依赖。

**直接编辑 `ConversationRoot.module.css` 来绘制壁纸。** 这是那个被回滚的硬编码所做的；它把功能耦合进共享 chrome。样式覆盖让 shell 保持功能无关。

## Consequences

用户可以为对话设置一个全局壁纸、调整叠加在它上面的可读性遮罩、一键清除，全部持久化在 `settings.yaml` 中，并在运行中的列上即时生效。默认状态（无图片）渲染与之前完全一致。壁纸与悬浮输入框覆盖都由一张由 CSS 变量驱动的注入样式表产生，因此插件不需要 shell 槽位、能挂载到 stock 的 `ui-conversation` 上。降级是优雅但真实的：覆盖依赖它们所匹配的 stock data 属性，因此未来 shell 重构需要重新对准这个插件（已在 README 中说明）。该功能不引入任何模型可见工具或会话事件；它唯一的持久化产物就是这两个字段的设置段。

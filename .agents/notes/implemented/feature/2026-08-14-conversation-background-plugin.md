# Agent Note: Global conversation-background plugin

Status: implemented

English | [中文](2026-08-14-conversation-background-plugin.zh.md)

## Problem

The conversation column was a flat surface: its background came from one theme base token (`--dsw-alias-bg-base`), and there was no way for a user to put their own image behind the chat. Setting a wallpaper was once hardcoded into `ui-conversation`'s `ConversationRoot.module.css`, which permanently stamped one resource path into a shared component and gave the user no control. The product needs a user-owned, durable, per-process wallpaper with legibility control. It must ship as a standalone plugin (the upstream accepts plugins, not core-PR edits) and must not require changes to the stock `ui-conversation` shell so any dsh install can mount it as-is.

## Decision

**A standalone client plugin owns the feature end to end, without editing the shell.** `@deepseek-ai/dsh-client-ui-background` works by overriding the rendered DOM: it writes CSS variables onto the document root and installs one plugin-owned `<style>` tag. This is the classic "userstyle" extension model — the stock shell's source is untouched, so the plugin is self-sufficient and can be published under the `dsh-plugin` topic.

**Persistence rides the settings seam.** Its host half registers the `chat-background` namespace (fields `imageDataUrl`, `opacity`) on `ctx.settings`, and the namespace is added to `api-proxy`'s `WEB_SETTINGS_NAMESPACES` so the browser can read and write it. The background image is stored as an inline data URL (downsampled to at most 2560px on the longest side, re-encoded to WebP when supported else JPEG), which keeps `settings.yaml` small and avoids touching the session-authorized attachment read path.

**The browser surface is pure CSS, no shell slot.** The runtime binds the namespace and writes `--dsh-bg-image` / `--dsh-bg-wash-opacity` on `document.documentElement`. The enhancement stylesheet paints the wallpaper behind the column via `[data-phase]::before`, tints it with a readable wash via `[data-phase]::after` (the wash opacity keeps text readable over any picture and preserves each palette's tint), and turns the docked input into a floating island via `[data-phase][data-phase='active'] [data-composer-seat]` — narrowing the seat to the card width, centering it, and removing the stock bottom fade so the wallpaper shows beside the input. The rules match stable stock data attributes (`data-phase`, `data-conversation-scroll`, `data-composer-seat`), so a future shell rename degrades the override gracefully instead of breaking the shell.

**The settings surface is a General-section row.** The feature registers into `settings.general.item` (order 30) with upload, a wash-opacity slider (0–100), and clear — product copy registered under the `settings.background` locale namespace.

## Alternatives considered

**Reuse the session attachment path for bytes.** Rejected: attachment reads are session-authorized (`session.attachment`), but the background is global; exposing an unconditional read-by-id would widen that authorization boundary. An inline data URL in the settings document needs no new host read channel.

**Add a `conversation.background` slot to `ui-conversation` and register a React layer.** This is the earlier approach; it requires shipping an edit to the core shell, which upstream does not accept and which breaks the "standalone plugin" goal. The CSS-override model removes that dependency entirely.

**Paint the wallpaper by editing `ConversationRoot.module.css` directly.** This is what the reverted hardcode did; it couples the feature to shared chrome. Style overrides keep the shell feature-agnostic.

## Consequences

A user can set a global conversation wallpaper, tune a readable wash over it, and clear it, all persisted in `settings.yaml` and applied immediately across the running column. The default state (no image) renders exactly as before. The wallpaper and the floating-composer override are both produced by one injected stylesheet driven by CSS variables, so the plugin needs no shell slot and mounts on a stock `ui-conversation`. Degradation is graceful but real: the overrides depend on the stock data attributes they match, so a future shell restructure requires re-targeting this plugin (documented in the README). The feature adds no model-visible tool or session event; its only durable artifact is the two-field settings section.

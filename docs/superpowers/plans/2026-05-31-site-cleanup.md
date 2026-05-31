# Site Cleanup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a Chrome / Edge popup tool that manually clears the current page origin's Service Worker + cache or the current page origin's full site data.

**Architecture:** Keep cleanup rules in a small background helper so popup UI stays thin and the deletion scope stays reviewable. Expose one background message for cleanup, gate it behind optional `browsingData` permission, and surface a compact control panel in the existing More page.

**Tech Stack:** TypeScript, React, Ant Design, Chrome Extensions MV3, Node test runner

---

## File Structure

- Create: `src/background/site-cleanup.ts`
  - Own cleanup scope validation, origin parsing, and `browsingData.remove` payload generation.
- Create: `src/popup/more/site-cleanup.tsx`
  - Render the manual cleanup panel and request optional permission on demand.
- Create: `tests/site-cleanup.test.mjs`
  - Lock the origin parsing and cleanup payload rules.
- Modify: `src/types/message.d.ts`
  - Add the background cleanup message contract.
- Modify: `src/background/index.ts`
  - Handle the cleanup message and call the helper.
- Modify: `src/popup/more/index.tsx`
  - Mount the cleanup panel in the More tab.
- Modify: `src/locales/zh_CN.json`
  - Add Chinese strings for cleanup labels, descriptions, warnings, and results.
- Modify: `src/locales/en_US.json`
  - Add English strings for cleanup labels, descriptions, warnings, and results.
- Modify: `manifest.ts`
  - Add Chromium-only optional `browsingData` permission.

## Task 1: Lock Cleanup Scope Rules

**Files:**
- Create: `D:/codex_project/test/fingerprint/my-fingerprint/src/background/site-cleanup.ts`
- Create: `D:/codex_project/test/fingerprint/my-fingerprint/tests/site-cleanup.test.mjs`

- [ ] Define origin parsing so only `http:` and `https:` URLs are accepted.
- [ ] Define a `cache-lite` scope with `cacheStorage` and `serviceWorkers`.
- [ ] Define a `site-data` scope with `cacheStorage`, `serviceWorkers`, `cookies`, `indexedDB`, `localStorage`, `fileSystems`, and `webSQL`.
- [ ] Verify the helper with a Node test file before wiring UI.

## Task 2: Wire Background Cleanup

**Files:**
- Modify: `D:/codex_project/test/fingerprint/my-fingerprint/src/types/message.d.ts`
- Modify: `D:/codex_project/test/fingerprint/my-fingerprint/src/background/index.ts`
- Reuse: `D:/codex_project/test/fingerprint/my-fingerprint/src/background/site-cleanup.ts`

- [ ] Add a `site.cleanup` background message.
- [ ] Return structured success / failure results so popup can show precise feedback.
- [ ] Keep unsupported URLs and unsupported browsers on the failure path rather than guessing.

## Task 3: Add Popup Control Panel

**Files:**
- Create: `D:/codex_project/test/fingerprint/my-fingerprint/src/popup/more/site-cleanup.tsx`
- Modify: `D:/codex_project/test/fingerprint/my-fingerprint/src/popup/more/index.tsx`
- Modify: `D:/codex_project/test/fingerprint/my-fingerprint/manifest.ts`

- [ ] Add Chromium-only optional `browsingData` permission.
- [ ] Show the current page hostname / origin state in the More tab.
- [ ] Request `browsingData` permission on demand when the user clicks a cleanup button.
- [ ] Expose two manual actions only:
  - [ ] Clear current origin Service Worker + cache
  - [ ] Clear current origin full site data
- [ ] Show clear warnings for the full-site-data action and tell the user to refresh after cleanup.

## Task 4: Localize and Verify

**Files:**
- Modify: `D:/codex_project/test/fingerprint/my-fingerprint/src/locales/zh_CN.json`
- Modify: `D:/codex_project/test/fingerprint/my-fingerprint/src/locales/en_US.json`
- Verify: `D:/codex_project/test/fingerprint/my-fingerprint/tests/site-cleanup.test.mjs`

- [ ] Add new labels, descriptions, and result strings in Chinese and English.
- [ ] Run `node --test tests/site-cleanup.test.mjs`.
- [ ] Run `npm run build`.

## Plan Self-Review

- Spec coverage: the plan covers the two approved cleanup actions, Chromium-only compatibility, optional permission flow, and popup integration.
- Placeholder scan: no `TODO` / `TBD` markers remain.
- Type consistency: `cache-lite`, `site-data`, and `site.cleanup` are the only new public names and are reused consistently.

## Post-Implementation Notes

- The popup help text now explains the intended use cases for both cleanup modes:
  - `cache-lite`: for cache / Service Worker state issues when preserving login state is preferred.
  - `site-data`: for a more complete current-origin reset when signing out is acceptable.
- The help text also clarifies that `browsingData` is requested on demand and may not show an obvious Edge / Chromium confirmation prompt, which matches real browser behavior for this permission.
- The popup now shows a persistent warning above the full-cleanup button so users do not need to open the tooltip to notice the sign-out risk.
- The Chinese tooltip copy was rewritten into short scenario-driven phrases and avoids exposing `origin` terminology to non-technical users.

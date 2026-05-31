# Timezone Dynamic Offset Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make preset and custom timezones resolve the correct UTC offset from the selected IANA zone for the current date, while keeping manual offset as a fallback when the zone is missing or invalid.

**Architecture:** Extract timezone validation and offset-resolution logic into a dedicated utility module, then reuse it from both the popup UI and the runtime hook. The runtime will derive date-specific offsets from the selected zone instead of trusting the preset JSON's static `offset` field.

**Tech Stack:** TypeScript, React, built-in `Intl` APIs, Node `node:test`

---

### Task 1: Add a failing timezone utility test

**Files:**
- Create: `tests/timezone.test.mjs`
- Create: `src/utils/timezone.ts`

- [ ] **Step 1: Write the failing test**

```js
import test from 'node:test'
import assert from 'node:assert/strict'
import {
  isValidTimeZone,
  resolveTimeZoneOffset,
} from '../src/utils/timezone.ts'

test('resolveTimeZoneOffset uses DST-aware offsets for preset IANA zones', () => {
  const winter = new Date('2026-01-15T12:00:00.000Z')
  const summer = new Date('2026-06-15T12:00:00.000Z')

  assert.equal(resolveTimeZoneOffset({
    offset: -8,
    zone: 'America/Los_Angeles',
    locale: 'en-US',
  }, winter), -8)
  assert.equal(resolveTimeZoneOffset({
    offset: -8,
    zone: 'America/Los_Angeles',
    locale: 'en-US',
  }, summer), -7)
})

test('resolveTimeZoneOffset falls back to manual offset when zone is invalid', () => {
  assert.equal(resolveTimeZoneOffset({
    offset: 8,
    zone: 'Invalid/Zone',
    locale: 'zh-CN',
  }, new Date('2026-06-15T12:00:00.000Z')), 8)
})

test('isValidTimeZone accepts real IANA zones and rejects invalid ones', () => {
  assert.equal(isValidTimeZone('America/New_York'), true)
  assert.equal(isValidTimeZone('Invalid/Zone'), false)
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `node --test tests/timezone.test.mjs`
Expected: FAIL with module-not-found or missing-export errors for `src/utils/timezone.ts`

- [ ] **Step 3: Write minimal implementation**

```ts
export const isValidTimeZone = (zone?: string | null): zone is string => {
  // validate with Intl.DateTimeFormat
}

export const resolveTimeZoneOffset = (
  info: Pick<TimeZoneInfo, 'offset' | 'zone'>,
  date = new Date(),
) => {
  // derive offset from zone when valid, otherwise use info.offset
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `node --test tests/timezone.test.mjs`
Expected: PASS

### Task 2: Use the timezone utility in the popup and runtime hook

**Files:**
- Modify: `src/popup/config/special/timezone.tsx`
- Modify: `src/core/tasks.ts`
- Modify: `public/settings/timezone.json`
- Test: `tests/timezone.test.mjs`

- [ ] **Step 1: Extend tests for preset coverage**

```js
test('DST-sensitive presets change offset across the year', () => {
  assert.equal(resolveTimeZoneOffset({
    offset: -5,
    zone: 'America/New_York',
    locale: 'en-US',
  }, new Date('2026-01-15T12:00:00.000Z')), -5)
  assert.equal(resolveTimeZoneOffset({
    offset: -5,
    zone: 'America/New_York',
    locale: 'en-US',
  }, new Date('2026-06-15T12:00:00.000Z')), -4)
})
```

- [ ] **Step 2: Run test to verify it fails if utility is incomplete**

Run: `node --test tests/timezone.test.mjs`
Expected: FAIL until runtime-oriented helper paths are complete

- [ ] **Step 3: Write minimal integration code**

```ts
const offset = resolveTimeZoneOffset(tzValue, new Date())
const zoneIsValid = isValidTimeZone(modeValue.zone)

// popup:
// - label preset options with current resolved offset
// - disable offset input when zoneIsValid

// runtime:
// - resolve offset per date instance instead of using one fixed offset
// - use the resolved zone for Intl / Date hooks
```

- [ ] **Step 4: Run test and build verification**

Run: `node --test tests/timezone.test.mjs`
Expected: PASS

Run: `npm run build`
Expected: build succeeds

### Task 3: Write the repair note

**Files:**
- Create: `docs/timezone-dst-fix.md`

- [ ] **Step 1: Write the repair document**

```md
# 时区夏令时修复说明

## 修复目标
- 说明预设时区固定 offset 的问题
- 说明本次改为按日期动态解析 offset
- 说明自定义 zone 合法时 offset 自动计算、非法时回退手填

## 修复过程
- 记录问题定位
- 记录测试设计
- 记录代码修改点
- 记录验证结果
```

- [ ] **Step 2: Verify the document exists and references the final behavior**

Run: `Get-Content -Raw docs/timezone-dst-fix.md`
Expected: includes goal, process, and verification summary

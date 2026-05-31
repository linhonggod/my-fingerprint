# WebRTC Leak Protection Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace hard WebRTC disable with Chromium `disable_non_proxied_udp` leak-protection mode that falls back to hard disable when the browser policy API is unavailable or fails.

**Architecture:** Add a background-only WebRTC policy helper that computes a runtime `webrtcPolicyState` and an "effective storage" snapshot for injection. UI and storage expose a third `enabled` mode; injected page code only hard-disables WebRTC when the effective mode resolves to `disabled`.

**Tech Stack:** TypeScript, Chrome Extensions MV3, React, Zustand, Node test runner, Vite

---

## File Structure

- Create: `src/background/webrtc-policy.ts`
  - Own the Chromium WebRTC IP policy calls, runtime state resolution, and injection-time storage rewriting.
- Create: `tests/webrtc-policy.test.mjs`
  - Lock behavior for policy success, unsupported API fallback, and effective injected WebRTC mode.
- Modify: `src/background/storage.ts`
  - Track non-persisted runtime `webrtcPolicyState`, synchronize policy on init/config changes, expose effective storage for injection.
- Modify: `src/background/script.ts`
  - Use effective storage instead of raw storage when registering fast-inject user scripts.
- Modify: `src/background/index.ts`
  - Use effective storage instead of raw storage for compatibility-mode injection.
- Modify: `src/core/tasks.ts`
  - Only hard-disable WebRTC when the effective mode is `disabled`.
- Modify: `src/types/storage.d.ts`
  - Extend `webrtc` mode to include `EnableHookMode`.
- Modify: `src/popup/config/group/strong.tsx`
  - Expose `default / enabled / disabled` for WebRTC.
- Modify: `src/locales/zh_CN.json`
  - Rewrite WebRTC description and `type.enabled` label to “防泄露（推荐）”.
- Modify: `src/locales/en_US.json`
  - Rewrite WebRTC description and `type.enabled` label to “Leak Protection (Recommended)”.
- Modify: `manifest.ts`
  - Add `privacy` permission to the Chromium manifest path only.

## Task 1: Lock WebRTC Policy Behavior with Tests

**Files:**
- Create: `D:/codex_project/test/fingerprint/my-fingerprint/tests/webrtc-policy.test.mjs`
- Create: `D:/codex_project/test/fingerprint/my-fingerprint/src/background/webrtc-policy.ts`

- [ ] **Step 1: Write the failing test file**

Create `tests/webrtc-policy.test.mjs` with these cases before any implementation exists:

```js
import test from 'node:test'
import assert from 'node:assert/strict'

import { HookType } from '../src/types/enum.ts'
import {
  WEBRTC_IP_POLICY,
  applyWebRtcPolicyMode,
  getInjectedStorageForWebRtc,
} from '../src/background/webrtc-policy.ts'

const createChromeApi = ({
  getValue = WEBRTC_IP_POLICY,
  setThrows = null,
  hasPolicy = true,
} = {}) => {
  if (!hasPolicy) {
    return {}
  }

  return {
    runtime: { lastError: undefined },
    privacy: {
      network: {
        webRTCIPHandlingPolicy: {
          set(details, callback) {
            if (setThrows) {
              throw setThrows
            }
            this.lastSet = details
            callback?.()
          },
          get(details, callback) {
            callback({ value: getValue, levelOfControl: 'controlled_by_this_extension' })
          },
          clear(details, callback) {
            callback?.()
          },
        },
      },
    },
  }
}

const createStorage = (type) => ({
  version: '2.7.3',
  config: {
    enable: true,
    seed: { browser: 1, global: 2 },
    fp: {
      navigator: {
        clientHints: { type: HookType.default },
        languages: { type: HookType.default },
        hardwareConcurrency: { type: HookType.default },
      },
      screen: {
        size: { type: HookType.default },
        depth: { type: HookType.default },
      },
      normal: {
        gpuInfo: { type: HookType.default },
      },
      other: {
        timezone: { type: HookType.default },
        canvas: { type: HookType.default },
        audio: { type: HookType.default },
        webgl: { type: HookType.default },
        webrtc: { type },
        font: { type: HookType.default },
        webgpu: { type: HookType.default },
        domRect: { type: HookType.default },
        serviceWorker: { type: HookType.default },
      },
    },
    action: { fastInject: false },
    input: { globalSeed: '2' },
    subscribe: { url: 'config.json' },
    prefs: { language: 'en-US', theme: 'system', logLevel: 'ERROR' },
  },
  policies: {
    whitelist: [],
    blacklist: [],
    isBlacklistMode: false,
  },
})

test('applyWebRtcPolicyMode enables Chromium leak protection for enabled mode', async () => {
  const chromeApi = createChromeApi()

  const state = await applyWebRtcPolicyMode({
    chromeApi,
    browser: 'chrome',
    mode: HookType.enabled,
  })

  assert.equal(state, 'policy-enabled')
  assert.deepEqual(
    chromeApi.privacy.network.webRTCIPHandlingPolicy.lastSet,
    { value: WEBRTC_IP_POLICY }
  )
})

test('applyWebRtcPolicyMode falls back to hard disable when policy API is missing', async () => {
  const state = await applyWebRtcPolicyMode({
    chromeApi: createChromeApi({ hasPolicy: false }),
    browser: 'chrome',
    mode: HookType.enabled,
  })

  assert.equal(state, 'fallback-disabled')
})

test('applyWebRtcPolicyMode falls back to hard disable for Firefox enabled mode', async () => {
  const state = await applyWebRtcPolicyMode({
    chromeApi: createChromeApi(),
    browser: 'firefox',
    mode: HookType.enabled,
  })

  assert.equal(state, 'fallback-disabled')
})

test('getInjectedStorageForWebRtc rewrites enabled mode to disabled during fallback', () => {
  const original = createStorage(HookType.enabled)
  const effective = getInjectedStorageForWebRtc(original, 'fallback-disabled')

  assert.equal(effective.config.fp.other.webrtc.type, HookType.disabled)
  assert.equal(original.config.fp.other.webrtc.type, HookType.enabled)
})

test('getInjectedStorageForWebRtc preserves enabled mode when policy is active', () => {
  const original = createStorage(HookType.enabled)
  const effective = getInjectedStorageForWebRtc(original, 'policy-enabled')

  assert.equal(effective.config.fp.other.webrtc.type, HookType.enabled)
})
```

- [ ] **Step 2: Run the test to verify it fails**

Run:

```powershell
node --test tests/webrtc-policy.test.mjs
```

Expected:

- FAIL with `ERR_MODULE_NOT_FOUND` or missing export errors for `src/background/webrtc-policy.ts`

- [ ] **Step 3: Write the minimal helper implementation**

Create `src/background/webrtc-policy.ts`:

```ts
import { HookType } from '@/types/enum'

export const WEBRTC_IP_POLICY = 'disable_non_proxied_udp' as const

export type WebRtcPolicyState =
  | 'default'
  | 'policy-enabled'
  | 'fallback-disabled'

type ChromePolicyApi = {
  runtime?: { lastError?: Error | undefined }
  privacy?: {
    network?: {
      webRTCIPHandlingPolicy?: {
        set: (details: { value: typeof WEBRTC_IP_POLICY }, callback?: () => void) => void
        get: (
          details: {},
          callback: (details: { value?: string, levelOfControl?: string }) => void
        ) => void
        clear: (details: {}, callback?: () => void) => void
      }
    }
  }
}

const cloneStorage = <T>(value: T): T => JSON.parse(JSON.stringify(value))

const getPolicyController = (chromeApi: ChromePolicyApi) =>
  chromeApi.privacy?.network?.webRTCIPHandlingPolicy

const setPolicyValue = async (chromeApi: ChromePolicyApi) => {
  const controller = getPolicyController(chromeApi)
  if (!controller) {
    return false
  }

  await new Promise<void>((resolve, reject) => {
    try {
      controller.set({ value: WEBRTC_IP_POLICY }, () => {
        const err = chromeApi.runtime?.lastError
        if (err) {
          reject(err)
          return
        }
        resolve()
      })
    } catch (error) {
      reject(error)
    }
  })

  const result = await new Promise<{ value?: string }>((resolve) => {
    controller.get({}, (details) => resolve(details))
  })

  return result.value === WEBRTC_IP_POLICY
}

export const applyWebRtcPolicyMode = async ({
  chromeApi,
  browser,
  mode,
}: {
  chromeApi: ChromePolicyApi
  browser?: BrowserType
  mode: HookType
}): Promise<WebRtcPolicyState> => {
  const controller = getPolicyController(chromeApi)

  if (mode === HookType.default) {
    controller?.clear({}, () => {})
    return 'default'
  }

  if (mode === HookType.disabled) {
    controller?.clear({}, () => {})
    return 'fallback-disabled'
  }

  if (browser !== 'chrome' || !controller) {
    controller?.clear({}, () => {})
    return 'fallback-disabled'
  }

  try {
    return await setPolicyValue(chromeApi) ? 'policy-enabled' : 'fallback-disabled'
  } catch {
    return 'fallback-disabled'
  }
}

export const getInjectedStorageForWebRtc = (
  storage: LocalStorage,
  state: WebRtcPolicyState,
) => {
  if (state !== 'fallback-disabled') {
    return storage
  }

  const next = cloneStorage(storage)
  next.config.fp.other.webrtc = { type: HookType.disabled }
  return next
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run:

```powershell
node --test tests/webrtc-policy.test.mjs
```

Expected:

- PASS
- 5 tests
- 0 failures

- [ ] **Step 5: Commit the helper and tests if git actions are approved**

```powershell
git add tests/webrtc-policy.test.mjs src/background/webrtc-policy.ts
git commit -m "test(webrtc): 添加策略与回退测试"
```

## Task 2: Wire Runtime Policy State into Background Injection

**Files:**
- Modify: `D:/codex_project/test/fingerprint/my-fingerprint/src/background/storage.ts`
- Modify: `D:/codex_project/test/fingerprint/my-fingerprint/src/background/script.ts`
- Modify: `D:/codex_project/test/fingerprint/my-fingerprint/src/background/index.ts`
- Reuse Test: `D:/codex_project/test/fingerprint/my-fingerprint/tests/webrtc-policy.test.mjs`

- [ ] **Step 1: Extend the test with one more integration-focused case**

Append this case to `tests/webrtc-policy.test.mjs` so the storage rewrite contract is explicit before wiring background code:

```js
test('getInjectedStorageForWebRtc leaves disabled mode untouched', () => {
  const original = createStorage(HookType.disabled)
  const effective = getInjectedStorageForWebRtc(original, 'fallback-disabled')

  assert.equal(effective.config.fp.other.webrtc.type, HookType.disabled)
})
```

- [ ] **Step 2: Run the test to verify the new case fails if helper behavior changes**

Run:

```powershell
node --test tests/webrtc-policy.test.mjs
```

Expected:

- PASS right now
- This is the guardrail before touching background wiring

- [ ] **Step 3: Add runtime `webrtcPolicyState` and effective storage accessors**

Modify `src/background/storage.ts` in three places.

1. Add imports near the top:

```ts
import { getBrowser } from "@/utils/equipment";
import {
  applyWebRtcPolicyMode,
  getInjectedStorageForWebRtc,
  type WebRtcPolicyState,
} from "./webrtc-policy";
```

2. Extend `LocalStorageContext` and initialize it:

```ts
type LocalStorageContext = {
  storage: LocalStorage
  whitelistHelper: SiteListHelper
  blacklistHelper: SiteListHelper
  configNonce: number
  policiesNonce: number
  webrtcPolicyState: WebRtcPolicyState
}

const genStorageContent = (storage: LocalStorage): LocalStorageContext => ({
  storage,
  whitelistHelper: new SiteListHelper(storage.policies.whitelist),
  blacklistHelper: new SiteListHelper(storage.policies.blacklist),
  configNonce: randNonce(),
  policiesNonce: randNonce(),
  webrtcPolicyState: 'default',
})
```

3. Add a sync helper and exported effective-storage helper:

```ts
const syncWebRtcPolicyState = async (ctx: LocalStorageContext) => {
  ctx.webrtcPolicyState = await applyWebRtcPolicyMode({
    chromeApi: chrome,
    browser: getBrowser(navigator.userAgent),
    mode: ctx.storage.config.fp.other.webrtc.type,
  })
}

export const getInjectedStorage = async () => {
  const ctx = await getLocalStorage()
  return getInjectedStorageForWebRtc(ctx.storage, ctx.webrtcPolicyState)
}
```

4. Call `await syncWebRtcPolicyState(...)` before `reRegisterScript()` in both `initLocalStorage()` and `updateContext()`:

```ts
mContent = genStorageContent(_storage)
await syncWebRtcPolicyState(mContent)
await chrome.storage.local.set(_storage)
await reRegisterScript()
reRequestHeader()
void applySubscribeStorage()
```

and:

```ts
await syncWebRtcPolicyState(ctx)
saveContextToLocalStorage()
await reRegisterScript()
reRequestHeader()
```

- [ ] **Step 4: Update both injection paths to use effective storage**

Modify `src/background/script.ts`:

```ts
import { getInjectedStorage, updateContext } from "./storage";

export const reRegisterScript = async () => {
  const storage = await getInjectedStorage()
  if (!ensureFastInject(storage)) return;

  logger.debug('update injectScript in fast mode:', storage);
  if (storage.config.enable) {
    const scripts: chrome.userScripts.RegisteredUserScript[] = [{
      id: REG_ID,
      allFrames: true,
      runAt: 'document_start',
      world: 'MAIN',
      matches: ["*://*/*"],
      js: [{ code: getRegScriptCode(storage) }],
    }]
    // keep existing update/register logic
  } else {
    chrome.userScripts.unregister({ ids: [REG_ID] }).catch(() => { });
  }
}
```

Modify `src/background/index.ts` inside `tabs.onUpdated`:

```ts
import { getInjectedStorage, getLocalStorage, importContext, initLocalStorage, updateContext } from "./storage";

chrome.tabs.onUpdated.addListener(async (tabId, changeInfo, tab) => {
  if (changeInfo.status === 'loading') {
    const ctx = await getLocalStorage()
    const { whitelistHelper, blacklistHelper } = ctx
    const storage = await getInjectedStorage()

    logger.debug('chrome.tabs.onUpdated:', tab.title || tab.url || tab.id);
    logger.debug('injectScript with storage:', storage)

    injectScript(tabId, storage)
    // keep existing whitelist/blacklist logic
  }
});
```

- [ ] **Step 5: Run tests and the build**

Run:

```powershell
node --test tests/webrtc-policy.test.mjs
npm run build
```

Expected:

- Tests: PASS
- Build: PASS

- [ ] **Step 6: Commit the background runtime plumbing if git actions are approved**

```powershell
git add src/background/storage.ts src/background/script.ts src/background/index.ts
git commit -m "feat(webrtc): 接入运行时策略状态"
```

## Task 3: Expose Leak Protection Mode in Config, Locale, and Manifest

**Files:**
- Modify: `D:/codex_project/test/fingerprint/my-fingerprint/src/types/storage.d.ts`
- Modify: `D:/codex_project/test/fingerprint/my-fingerprint/src/popup/config/group/strong.tsx`
- Modify: `D:/codex_project/test/fingerprint/my-fingerprint/src/locales/zh_CN.json`
- Modify: `D:/codex_project/test/fingerprint/my-fingerprint/src/locales/en_US.json`
- Modify: `D:/codex_project/test/fingerprint/my-fingerprint/manifest.ts`

- [ ] **Step 1: Update WebRTC mode typing**

Modify `src/types/storage.d.ts`:

```ts
  other: {
    timezone: DefaultHookMode | ValueHookMode<TimeZoneInfo>
    canvas: DefaultHookMode | RandomHookMode
    audio: DefaultHookMode | RandomHookMode
    webgl: DefaultHookMode | RandomHookMode
    webrtc: DefaultHookMode | EnableHookMode | DisableHookMode
    font: DefaultHookMode | RandomHookMode
    webgpu: DefaultHookMode | RandomHookMode
    domRect: DefaultHookMode | RandomHookMode
    serviceWorker: DefaultHookMode | DisableHookMode
  }
```

- [ ] **Step 2: Add Chromium `privacy` permission**

Modify `manifest.ts` so Chromium gets `privacy` while Firefox keeps the current permission set:

```ts
const VALUES = {
  permissions: [
    'storage',
    'tabs',
    'activeTab',
    'webNavigation',
    'scripting',
    'declarativeNetRequest',
    'clipboardRead',
    'clipboardWrite',
  ] as chrome.runtime.ManifestPermissions[],
  chrome_only_permissions: [
    'privacy',
  ] as chrome.runtime.ManifestPermissions[],
  optional_permissions: [
    "userScripts",
  ] as chrome.runtime.ManifestPermissions[],
  // ...
}

export const chromeManifest: ManifestV3Export = {
  ...baseManifest,
  // ...
  permissions: [
    ...VALUES.permissions,
    ...VALUES.chrome_only_permissions,
    ...VALUES.optional_permissions,
  ],
  // ...
}

export const firefoxManifest: ManifestV3Export & { [key: string]: any } = {
  ...baseManifest,
  permissions: VALUES.permissions,
  optional_permissions: VALUES.optional_permissions,
  // ...
}
```

- [ ] **Step 3: Expose `enabled` in the WebRTC selector**

Modify `src/popup/config/group/strong.tsx`:

```ts
const baseTypes = [HookType.default, HookType.page, HookType.browser, HookType.domain, HookType.global]
const webrtcTypes = [HookType.default, HookType.enabled, HookType.disabled]
const disabledTypes = [HookType.default, HookType.disabled]

// ...

<HookModeProvider obj={fp.other} name='webrtc'>
  <HookModeCard color='warning' isDescArray tags={unstableTag}>
    <HookModeSelector types={webrtcTypes} />
  </HookModeCard>
</HookModeProvider>
```

- [ ] **Step 4: Rewrite locale labels and description**

Modify `src/locales/zh_CN.json`:

```json
"type": {
  "default": "系统值",
  "value": "自定义值",
  "page": "每个标签页随机值",
  "browser": "每次启动浏览器随机值",
  "domain": "根据访问域名随机值",
  "global": "根据全局种子随机值",
  "enabled": "防泄露（推荐）",
  "disabled": "禁用"
},
"webrtc": [
  "防泄露模式下，Chrome / Edge 会优先启用 Chromium 的 WebRTC 代理防泄露策略，而不是直接把 WebRTC 整体关闭。",
  "若浏览器不支持该策略，扩展会自动回退为禁用 WebRTC，避免在失败时裸露真实地址信息。",
  "该模式依赖你本机真实代理环境；若代理规则把目标命中到 DIRECT，扩展无法覆盖该直连决策。"
]
```

Modify `src/locales/en_US.json`:

```json
"type": {
  "default": "System",
  "value": "Custom",
  "page": "Random per tab",
  "browser": "Random per browser launch",
  "domain": "Random by domain",
  "global": "Random by global seed",
  "enabled": "Leak Protection (Recommended)",
  "disabled": "Disabled"
},
"webrtc": [
  "In leak-protection mode, Chrome / Edge first enable Chromium's WebRTC proxy leak protection instead of removing WebRTC entirely.",
  "If the policy API is unavailable or fails, the extension automatically falls back to disabling WebRTC so the browser does not expose addresses in a half-configured state.",
  "This mode still depends on a real local proxy environment; if your proxy rules send a target to DIRECT, the extension cannot override that direct route."
]
```

- [ ] **Step 5: Run the build to verify type and manifest changes**

Run:

```powershell
npm run build
```

Expected:

- PASS
- No TypeScript errors around `EnableHookMode`
- No manifest typing errors around `privacy`

- [ ] **Step 6: Commit the config/locale/manifest changes if git actions are approved**

```powershell
git add src/types/storage.d.ts src/popup/config/group/strong.tsx src/locales/zh_CN.json src/locales/en_US.json manifest.ts
git commit -m "feat(webrtc): 暴露防泄露模式配置"
```

## Task 4: Restrict Hard Disable to Effective Disabled Mode and Verify End-to-End

**Files:**
- Modify: `D:/codex_project/test/fingerprint/my-fingerprint/src/core/tasks.ts`
- Reuse Test: `D:/codex_project/test/fingerprint/my-fingerprint/tests/webrtc-policy.test.mjs`
- Verify: `D:/codex_project/test/fingerprint/my-fingerprint/docs/superpowers/specs/2026-05-30-webrtc-proxy-leak-protection-design.md`

- [ ] **Step 1: Change the WebRTC task gate**

Modify `src/core/tasks.ts`:

```ts
{
  condition: ({ conf }) => conf.fp.other.webrtc.type === HookType.disabled,
  onEnable: ({ win, useDisownKeys, useDefine }) => {
    if (!win) return;

    {
      const keys: any = [
        'mediaDevices', 'getUserMedia', 'mozGetUserMedia', 'webkitGetUserMedia',
      ]
      useDisownKeys(win.navigator, keys);
      useDisownKeys(win.Navigator.prototype, keys);
      useDefine([win.Navigator.prototype, win.navigator], keys, {
        value: undefined,
        enumerable: false,
      });
      useDefine(win.Navigator.prototype, keys, {
        value: undefined,
        enumerable: false,
      });
    }

    // keep the existing RTCPeerConnection / RTCDataChannel undefined logic
  },
},
```

- [ ] **Step 2: Re-run targeted tests and full build**

Run:

```powershell
node --test tests/webrtc-policy.test.mjs
npm run build
```

Expected:

- Tests: PASS
- Build: PASS

- [ ] **Step 3: Manually verify the acceptance paths**

Run through these checks in a Chromium build:

```text
1. Set WebRTC to "防泄露（推荐）"
2. Reload the extension
3. Open a WebRTC detection page
4. Confirm RTCPeerConnection exists instead of being undefined
5. Confirm the browser no longer looks like "WebRTC fully disabled"
6. Toggle to "禁用" and confirm RTCPeerConnection becomes unavailable again
7. Toggle back to "系统值" and confirm the extension clears its policy override
```

For Firefox:

```text
1. Set WebRTC to "防泄露（推荐）"
2. Reload the extension
3. Confirm the behavior matches the disabled fallback path
```

- [ ] **Step 4: Re-read the spec and verify plan coverage before claiming completion**

Checklist:

```text
- Chrome / Edge use disable_non_proxied_udp in enabled mode
- Failure path falls back to hard disable
- Effective injection storage rewrites enabled -> disabled only during fallback
- UI exposes default / enabled / disabled
- Manifest adds Chromium privacy permission only
- WebRTC hard-disable gate now depends on effective disabled mode only
```

- [ ] **Step 5: Commit the runtime gate change if git actions are approved**

```powershell
git add src/core/tasks.ts
git commit -m "feat(webrtc): 仅在回退时禁用接口"
```

## Plan Self-Review

- Spec coverage: all confirmed requirements are mapped to Tasks 1-4. The only implementation-derived addition is keeping `privacy` Chromium-only in `manifest.ts`, which preserves Firefox scope.
- Placeholder scan: no `TODO` / `TBD` markers remain.
- Type consistency: `HookType.enabled`, `WebRtcPolicyState`, `getInjectedStorage`, and `applyWebRtcPolicyMode` use one naming set throughout the plan.

## 2026-05-31 Follow-up

- Add a background `webrtc.status` message so the popup can show the actual runtime state instead of forcing users to infer it from third-party test pages.
- Surface the effective WebRTC state under the selector: browser default, leak protection active, fallback disabled, or manual disabled.
- Restore the English locale split between the extension-wide `g.enabled` label and the WebRTC-specific `type.enabled` label.
- Make Edge routing into the Chromium WebRTC policy path explicit in `getBrowser()` so the behavior is intentional and reviewable.
- Update the WebRTC tooltip copy so it matches observed runtime behavior: WebRTC usually remains present while host / srflx candidates are restricted, the implementation is based on `disable_non_proxied_udp`, and Firefox is explicitly marked unsupported for this path.

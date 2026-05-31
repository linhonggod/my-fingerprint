# WebRTC 代理防泄露设计

日期：2026-05-30

状态：已确认，待用户审阅

## 背景

当前项目对 WebRTC 的处理方式是“硬禁用”：

- 存储类型只允许 `default` / `disabled`
- 配置 UI 只暴露这两档
- 运行时会把 `navigator.mediaDevices`、`getUserMedia`、`RTCPeerConnection` 等接口直接设为 `undefined`

这能阻止页面直接通过 WebRTC 读取地址信息，但副作用也很明显：

- WebRTC 被完全关闭的痕迹过于明显
- 检测页容易直接判断“WebRTC 不可用”
- 与用户当前使用方式不匹配

用户当前使用方式与目标：

- 主要使用 Chrome / Edge
- 平常开启系统代理，不依赖“仅 TUN 模式”
- 不在乎 Google Meet 等 WebRTC 功能失效
- 核心目标不是“伪装成默认浏览器”，而是“尽量避免 WebRTC 绕过代理泄露真实出口 / 本地地址”

## 目标

1. 在 Chrome / Edge 上，优先把 WebRTC 从“硬禁用”改成“浏览器级防泄露模式”
2. 尽量让 WebRTC 保持“接口存在但被严格约束”，而不是直接整个消失
3. 依赖 Chromium 的 `chrome.privacy.network.webRTCIPHandlingPolicy`，将模式设置为 `disable_non_proxied_udp`
4. 在策略不可用或设置失败时，自动回退到当前硬禁用逻辑
5. 保持实现范围最小，只修改与 WebRTC 配置、后台策略和现有禁用逻辑直接相关的部分

## 非目标

1. 不尝试做 Chromium patch 级别的底层网络伪装
2. 不承诺所有检测站都表现为“完全默认浏览器”
3. 不保证 WebRTC 会议、语音、P2P 站点正常工作
4. 不在本次改动中引入复杂的 candidate 文本伪造、`getStats()` 全量伪造或 SDP 深度改写
5. 不处理用户本地代理规则中的 `DIRECT` 风险口子；这属于环境侧约束，不属于扩展内逻辑

## 当前实现

当前 WebRTC 相关位置：

- `src/types/storage.d.ts`
  - `webrtc: DefaultHookMode | DisableHookMode`
- `src/popup/config/group/strong.tsx`
  - UI 只给 `default` / `disabled`
- `src/core/tasks.ts`
  - 在 `webrtc.type !== HookType.default` 时，把 WebRTC 相关接口直接抹掉
- `manifest.ts`
  - 当前没有 `privacy` 权限

这意味着当前实现本质上没有“开启但防泄露”的中间状态。

## 方案对比

### 方案 A：维持现状，继续硬禁用

优点：

- 改动最小
- 当前逻辑简单直接

缺点：

- WebRTC 不可用过于显眼
- 不符合当前目标

### 方案 B：纯页面层伪装 WebRTC

优点：

- 表面上更像 WebRTC 还开着

缺点：

- 对真实泄露控制弱
- 容易出现页面层结果与底层行为不一致
- 实现复杂度高，但收益不稳定

### 方案 C：浏览器级防泄露策略 + 禁用回退

优点：

- 直接利用 Chromium 已有的 WebRTC IP handling 策略
- 比硬禁用更自然
- 与“系统代理 + 严格防泄露”的目标匹配
- 实现范围中等，可控

缺点：

- 只能在 Chromium 路线生效
- 不是底层全伪装
- 仍依赖用户真实代理环境与本地规则

推荐：方案 C

## 设计概览

### 配置模型

将 WebRTC 存储类型从：

- `DefaultHookMode | DisableHookMode`

调整为：

- `DefaultHookMode | EnableHookMode | DisableHookMode`

其中三档行为定义为：

1. `default`
   - 不主动干预浏览器 WebRTC 策略
   - 不执行当前硬禁用逻辑

2. `enabled`
   - 语义重定义为“防泄露模式”
   - 在 Chrome / Edge 上尝试设置 `chrome.privacy.network.webRTCIPHandlingPolicy = disable_non_proxied_udp`
   - 设置成功时，不执行当前硬禁用逻辑
   - 设置失败或 API 不可用时，回退为当前硬禁用逻辑

3. `disabled`
   - 保持当前硬禁用逻辑

说明：

- 底层仍然沿用通用 `HookType.enabled`
- 仅在 WebRTC 文案和行为上把 `enabled` 呈现为“防泄露（推荐）”
- 这样改动最小，不需要为 WebRTC 单独新增一个全新枚举值

### 浏览器行为矩阵

#### Chrome / Edge

- 在当前项目中，Edge 复用 Chromium 路线，归入 `chrome` 分支处理，不单独新增 `BrowserType`
- `default`：不设置 policy，不禁用 WebRTC
- `enabled`：
  - 调用 `chrome.privacy.network.webRTCIPHandlingPolicy.set({ value: 'disable_non_proxied_udp' })`
  - 成功后保留 WebRTC API
  - 失败后回退到硬禁用
- `disabled`：直接硬禁用

#### Firefox

- `default`：不处理
- `enabled`：回退到硬禁用
- `disabled`：直接硬禁用

原因：

- 本次目标明确以 Chrome / Edge 为主
- 不在本次设计中为 Firefox 单独发明另一套 WebRTC 防泄露策略

### 回退策略

`enabled` 模式不是“尽力而为后什么都不做”，而是明确回退：

1. 先尝试设置 Chromium policy
2. 若浏览器没有 `chrome.privacy.network.webRTCIPHandlingPolicy`
3. 或策略设置报错
4. 或读取回显发现设置未生效
5. 则进入当前 `disabled` 的硬禁用路径

这样可以保证：

- 用户选择“防泄露”时，不会因为 API 不支持而变成“完全裸奔”
- 最坏情况仍然退回今天已经存在、已知可靠的防线

## 组件与职责

### 1. manifest

文件：

- `manifest.ts`

职责：

- 新增 `privacy` 权限

约束：

- 不额外引入 `proxy` 权限
- 不修改与本次需求无关的其他权限

### 2. 后台策略管理

候选位置：

- `src/background/index.ts`
- 或新增一个与浏览器策略相关的轻量工具模块

职责：

- 在配置变更时设置 / 清除 WebRTC IP handling policy
- 为 popup / storage 层提供“是否成功设置”的结果

建议：

- 将 policy 设置与清除封装成独立函数
- 不把 WebRTC policy 逻辑散落到多个消息分支里

### 3. 配置存储与类型

文件：

- `src/types/storage.d.ts`

职责：

- 扩展 `webrtc` 的模式类型

### 4. 配置 UI

文件：

- `src/popup/config/group/strong.tsx`
- `src/locales/zh_CN.json`
- `src/locales/en_US.json`

职责：

- 将 WebRTC 选项改为三档
- 将 `enabled` 在 UI 中显示为“防泄露（推荐）”而不是泛化的“启用”
- 在说明文案中明确：
  - Chrome / Edge 会优先启用 Chromium 防泄露策略
  - 若浏览器不支持，则回退为禁用
  - 依赖真实代理环境，不保证命中 `DIRECT` 规则的目标不直连

### 5. 页面注入任务

文件：

- `src/core/tasks.ts`

职责：

- 仅在以下情况执行当前硬禁用逻辑：
  - 模式明确为 `disabled`
  - 或模式为 `enabled` 但浏览器策略设置失败

不再在“防泄露模式已成功启用”时直接抹掉 WebRTC API。

## 数据流

1. 用户在 popup 中选择 WebRTC 模式
2. 配置写入 storage
3. 后台根据当前浏览器与模式决定是否设置 policy
4. 页面注入逻辑读取最终状态：
   - 成功启用 policy：保留 WebRTC API
   - policy 不可用 / 设置失败：走硬禁用
5. 检测页最终看到的行为：
   - Chromium 成功路径：WebRTC 仍存在，但受策略约束
   - 回退路径：WebRTC 被禁用

## 错误处理

需要显式处理以下场景：

1. `chrome.privacy` 不存在
2. `chrome.privacy.network` 不存在
3. `webRTCIPHandlingPolicy` 不存在
4. `set()` 调用抛错
5. `clear()` 调用抛错
6. 配置切换过快导致旧状态残留

处理原则：

- 对用户目标而言，“策略失败 -> 回退禁用”优先于“策略失败 -> 继续放行”
- 日志应清楚记录当前走的是：
  - policy 成功
  - policy 不支持
  - policy 失败并回退禁用

## 测试与验证

### 自动化验证

建议新增测试覆盖：

1. WebRTC 模式类型扩展后的存储兼容性
2. Chromium policy 设置函数在：
   - 成功
   - API 缺失
   - 设置报错
   三种场景下的返回值
3. 页面注入任务在不同最终状态下是否进入硬禁用路径

### 手动验证

重点手测场景：

1. Chrome + 系统代理开启 + `enabled`
   - WebRTC API 仍存在
   - 不再表现为直接 `undefined`
   - 检测页观察 candidate / 暴露结果是否比当前禁用更自然

2. Edge + 系统代理开启 + `enabled`
   - 与 Chrome 行为一致

3. Chromium 路线手动模拟 policy 不支持 / 设置失败
   - 观察是否自动回退到硬禁用

4. Firefox + `enabled`
   - 应明确回退为硬禁用

5. 模式切换链路
   - `default -> enabled -> disabled -> default`
   - 确认 policy 设置与清除、页面行为和 UI 状态一致

### 验收标准

满足以下条件即可认为本次设计实现达标：

1. Chrome / Edge 在 `enabled` 模式下优先走 `disable_non_proxied_udp`
2. 成功路径下 WebRTC API 不再被直接抹掉
3. 失败路径下会自动回退为硬禁用
4. UI 文案准确表达“防泄露模式”的真实含义和边界
5. 不引入与 WebRTC 无关的配置或行为改动

## 风险与权衡

1. 依赖用户真实代理环境
   - 若本机没有实际代理可走，策略收益有限

2. 受本地规则影响
   - 若代理软件把目标命中到 `DIRECT`，扩展无法覆盖该决策

3. 跨浏览器行为不一致
   - 本次明确接受 Chrome / Edge 优先，Firefox 回退

4. 兼容性不作为本次优先目标
   - Meet / 语音 / P2P 失效可接受

## 实现边界

本次实现只应修改以下区域：

- `manifest.ts`
- `src/types/storage.d.ts`
- `src/background/*` 中与 policy 管理直接相关的最小范围
- `src/popup/config/group/strong.tsx`
- `src/locales/zh_CN.json`
- `src/locales/en_US.json`
- `src/core/tasks.ts`
- 对应测试与说明文档

不应顺手扩展到：

- 其他强指纹项模式重构
- 通用权限系统大改
- Firefox 专项 WebRTC 方案
- JS 层 candidate 深度伪造

## 下一步

用户审阅本 spec。

若确认无误，再进入 `writing-plans`，生成实现计划。

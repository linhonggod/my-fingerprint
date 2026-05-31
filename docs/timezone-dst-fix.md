# 时区夏令时修复说明

## 修复目标

本次修复聚焦弱指纹中的“时区”配置，目标分为两部分：

1. 修复预设地区使用固定 `offset` 的问题
   - 让存在夏令时/冬令时切换的预设地区，按当前日期自动解析正确时差
   - 避免出现“洛杉矶全年都显示 `-8`”“纽约全年都显示 `-5`”这类会暴露异常的情况

2. 优化自定义时区的行为
   - 当用户填写合法的 IANA 时区名时，运行时自动根据该时区和当前日期计算 `offset`
   - 此时 `offset` 输入框禁用，避免用户误以为手填值仍然生效
   - 当 `zone` 为空或非法时，仍允许使用手动 `offset` 作为兜底

## 相对原项目的完整改动范围

这次修复相对原项目，实际改动集中在“时区运行时逻辑”“时区配置 UI”“测试”“修复说明文档”四个部分，没有扩展到其他无关模块。

### 1. 运行时时区逻辑修复

- 预设地区不再依赖固定 `offset` 伪装时区
- 合法 IANA 时区会按当前日期动态计算偏移，自动区分夏令时与标准时
- 自定义 `zone` 合法时，运行时优先信任 `zone`，不再信任手填 `offset`
- 自定义 `zone` 非法或为空时，才回退到手动 `offset`

### 2. 底层泄露路径修复

- 修复了不仅 `getTimezoneOffset()`，还包括基于本地化时间字符串差值推导偏移时的泄露问题
- 根因是部分时区换算辅助逻辑会读取已被 Hook 的 `Date` getter，导致二次计算时混入宿主真实行为
- 现在统一捕获并使用原生 `Date` / `Intl.DateTimeFormat` / 原生 getter 做时区换算，避免被当前 Hook 反向污染

### 3. 时区配置 UI 修复

- 预设时区列表改为显示当前日期下的真实偏移，而不是写死的静态偏移
- 预设列表的展开项增加季节标记：
  - 中文：`夏令时` / `标准时`
  - 英文：`DST` / `Standard`
- 预设列表展开项的最终显示格式调整为“地区在前，时区信息在后”
  - 中文示例：`洛杉矶（-7，夏令时）`
  - 英文示例：`Los Angeles (-7, DST)`
- 收起后的已选显示与展开列表分离：
  - 中文显示城市名，例如 `洛杉矶`
  - 英文显示原项目风格的缩写或 key，例如 `LAX`
- 修复了预设项初次选择后收起态错误显示 `LAX` 的问题，原因是原先预设列表按展开时才懒加载，已改为组件挂载后预加载
- 合法 IANA 时区名生效时，`offset` 输入框禁用，并展示自动解析出的当前偏移
- `zone` 输入框改为受控输入，便于用户清空或改错后正确回退到手动 `offset`

### 4. 测试与文档补充

- 新增独立时区工具模块测试
- 新增修复说明文档，便于后续整理 issue / PR
- 本地另外生成了 Edge 测试 zip 产物，但这属于本地验证产物，不属于建议提交到 PR 的源码改动

## 问题定位

项目中的时区预设清单位于：

- `public/settings/timezone.json`

原始实现中的核心问题有两个：

1. 预设项把 `offset` 写成了固定值
   - 例如 `America/Los_Angeles` 固定写成 `-8`
   - 例如 `America/New_York` 固定写成 `-5`

2. 运行时 Hook 直接信任这个固定值
   - `src/core/tasks.ts` 中的时区逻辑会直接用 `tzValue.offset` 参与 `Date`、`getTimezoneOffset`、`setter` 等行为计算
   - 这会导致目标地区处于夏令时时，浏览器暴露出的时差信息与真实地区不一致

另外，后续联调中还发现一个补充问题：

3. 存在第二条偏移泄露路径
   - 某些检测页面除了读取 `getTimezoneOffset()`，还会通过本地化时间字符串与 GMT 时间做差来推导时区
   - 如果换算辅助逻辑内部再次读取已被 Hook 的 `Date` getter，就会把宿主真实行为带回结果中
   - 这会造成第一条偏移路径修好了，但第二条路径仍然可能暴露异常

## 需要动态判断的预设地区

根据当前预设清单，并结合时区数据库在 2026 年的表现，以下预设地区会在一年内出现不同 `offset`，因此不能继续使用固定值：

- `Europe/London`
- `Europe/Paris`
- `Europe/Berlin`
- `Africa/Cairo`
- `Australia/Sydney`
- `Pacific/Auckland`
- `America/New_York`
- `America/Chicago`
- `America/Denver`
- `America/Los_Angeles`
- `America/Anchorage`

这些地区现在统一改为：运行时按 `Intl` 时区数据库和当前日期动态解析偏移。

## 修复过程

### 1. 先把时区解析逻辑独立出来

新增文件：

- `src/utils/timezone.ts`

该工具模块负责：

- 判断 `zone` 是否为合法时区标识
- 优先从合法 IANA 时区中解析当前日期对应的真实 `offset`
- 当 `zone` 非法时，退回到手动 `offset`
- 提供“目标时区本地时间”和“真实 UTC 时间戳”之间的换算函数，供 `Date` 构造和 setter 使用
- 捕获原生 `Date`、`Intl.DateTimeFormat` 和原生日期 getter，避免辅助计算被已安装的 Hook 干扰

### 2. 先写失败测试，再补实现

新增测试：

- `tests/timezone.test.mjs`

测试覆盖了以下行为：

- `America/Los_Angeles` 在冬季解析为 `-8`，在夏季解析为 `-7`
- `America/New_York` 在冬季解析为 `-5`，在夏季解析为 `-4`
- 非法 `zone` 时会回退到手动 `offset`
- 合法与非法时区名的识别
- 目标时区 wall clock 与 UTC 时间戳之间的换算
- 即使 `Date` getter 已被 Hook，底层换算仍继续使用原生日期部件
- UI 展示辅助函数可以识别当前日期对应的是夏令时还是标准时

### 3. 修改运行时时区 Hook

修改文件：

- `src/core/tasks.ts`

本次替换的核心点：

1. `Intl.DateTimeFormat`
   - 不再直接使用原配置里的固定 `zone/offset` 组合
   - 改为使用解析后的时区标识

2. `Date#getTimezoneOffset`
   - 不再返回固定值
   - 改为按 `thisArg` 对应日期动态解析目标时区偏移

3. `Date` 构造器与 `Date` setter
   - 不再使用单个固定 `diffMs`
   - 改为通过时区工具函数，把“目标时区本地时间”与真实时间戳互相换算
   - 这样同一地区在不同日期会自动落到不同的正确偏移

4. 原生 API 防污染
   - 运行时辅助逻辑不再依赖当前环境下已被改写过的 `Date` getter
   - 改为统一调用预先捕获的原生 getter 与原生 `Intl.DateTimeFormat`
   - 这样可以同时修正多条偏移暴露路径，而不是只修补单一 API 表面结果

### 4. 修改配置 UI

修改文件：

- `src/popup/config/special/timezone.tsx`

本次 UI 调整包括：

1. 预设列表中的标签改为动态显示当前日期对应的 `offset`
2. 当 `zone` 为合法 IANA 时区名时：
   - `offset` 输入框禁用
   - 输入框显示当前自动解析出来的偏移
3. 当 `zone` 非法或为空时：
   - `offset` 输入框恢复可编辑
4. `zone` 输入框改为受控输入，允许用户清空后回退到手动 `offset`
5. 预设列表展开项增加季节标记，并改成“地区在前，时区信息在后”
6. 收起后的已选显示单独处理：
   - 中文显示城市名
   - 英文显示缩写 / key
7. 预设项改为预加载，修复初次选中后收起态错误显示 `LAX` 的问题

## 涉及文件

本次相对原项目的主要源码与文档改动如下：

- `src/utils/timezone.ts`
- `src/core/tasks.ts`
- `src/popup/config/special/timezone.tsx`
- `tests/timezone.test.mjs`
- `docs/timezone-dst-fix.md`

如果准备发 PR，建议以以上源码、测试和文档为主；本地测试产物 zip 不建议纳入 PR。

## 验证结果

### 自动化测试

执行：

```bash
node --test tests/timezone.test.mjs
```

结果：

- 8 条测试全部通过

### 构建验证

执行：

```bash
npm run build
```

结果：

- 构建成功
- Vite 仅给出原有的大 chunk 警告，本次修复未引入新的构建错误

## 最终效果

修复完成后：

- 预设地区不再把夏令时地区伪装成全年固定偏移
- 自定义合法 IANA 时区会自动跟随当前日期切换正确偏移
- `locale` 仍然可以独立填写，不与 `zone` 强绑定
- 手动 `offset` 仍保留为兜底能力，但不会和合法 `zone` 同时生效造成误导

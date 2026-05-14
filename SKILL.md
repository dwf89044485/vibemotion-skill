---
name: motion-connect
description: >
  将组件动效接入 vibemotion 调参面板。当用户说「加动效面板」「接入调参」
  「给 XX 组件加参数调节」「接动效」「vibemotion」「调参面板」「动效编辑」时触发。
  也适用于用户要求让某个组件的动画参数可以实时调节预览的场景。
---

# Motion Connect

将组件动效接入 vibemotion 调参面板——让设计师拖滑杆调参数，调完复制发给 AI，产品不留痕迹。

---

## 使用指引

当用户问"怎么用"时，展示以下内容：

```
Motion Connect 帮你把项目里的动效接入可视化调参面板。

用法：告诉我你想调哪个组件或动效的参数，比如：
- "帮我把首页的入场动画接入调参面板"
- "我想调 sidebar 的展开收起动效"
- "给这个卡片的 hover 效果加参数面板"

我会：
1. 分析那个组件，提炼出你能感知的「感觉旋钮」
2. 列给你确认（你可以增减）
3. 写好代码，面板就出来了
4. 调到满意后，复制参数发给我更新代码，然后移除面板

不需要懂代码，拖滑杆就行。
```

---

## 执行流程

### 阶段 0：环境检测

#### 0A. vibemotion 包

检查 `package.json` 是否包含 `vibemotion`：
- 有 → 跳过
- 没有 → 安装：`pnpm add vibemotion`（或匹配项目的包管理器）

#### 0B. 编辑器面板

搜索项目代码是否已有 `vibeset-editor` 或 `VibesetProvider`：
- 有 → 跳过
- 没有 → 根据技术栈挂载：

**React**：App 入口包裹 Provider

```tsx
import { VibesetProvider } from "vibemotion/react";

<VibesetProvider enabled theme="dark">
  {/* 原有内容 */}
</VibesetProvider>
```

**非 React**：入口文件创建 store + 挂载 editor

```js
import { createVibeset } from "vibemotion";

const store = createVibeset();
const editor = document.createElement("vibeset-editor");
editor.theme = "dark";
editor.store = store;
document.body.appendChild(editor);
```

挂载后右下角应出现「动效编辑」按钮。没出现则检查 import。

---

### 阶段 1：扫描与确认

**入口**：用户指定了目标组件。未指定则问：「你想把哪个组件或动效接入调参面板？」

#### 1A. 提炼感觉旋钮（静默）

读组件源码，找所有动画可调数值。**分析过程不输出给用户。**

扫描以下类型的数值：
- 时间参数：`duration`、`delay`、`staggerChildren`、`transition` 配置
- 弹簧参数：`damping`、`stiffness`、`mass`
- 变换目标值：`scale`、`rotate`、`x`、`y`、`opacity`
- 视觉效果：`blur`、`borderRadius`、`shadow`
- 布局数值：间距、偏移、角度、宽高等自定义值
- CSS animation：`@keyframes` 里的起止值、`animation-duration`、`animation-delay`

提炼规则：
- 过滤用户感知不到差别的底层量（纯内部索引、中间计算变量、fill-mode 等）
- 可聚合的合成一根旋钮（translateY + scale + blur → "悬浮强度"）
- 中文 label（"入场速度"不是"durationBase"）
- min/max 只给有效感知区间，极值不崩
- 按用户心智模型分组（"入场节奏""视觉强度"），不按代码结构分

区分关键参数与噪音：
- **推荐暴露**：调了视觉变化明显、用户直觉能理解
- **建议跳过**：颗粒度太细、单独调容易破坏效果（如 spring 三参数中只暴露"弹性感"一根聚合旋钮）
- 判断不了的，标为「可选」放在列表末尾让用户决定

#### 1B. 发现可切换状态（静默）

**全面扫描以下所有类型**，不要遗漏：

- **业务模式（props 传入）**：看组件 props/interface 定义，找控制视觉差异的 prop（如 `mode`、`variant`、`collapsed`）。注意：props 不在组件内部声明，容易漏。
- **操作状态（内部 state）**：看 `useState` 声明，找控制视觉切换的 state（如 `isOpen`、`isExpanded`、`activeTab`）
- **交互态**：`hover`、`active`、`focus` 等由鼠标/键盘触发的视觉反馈

判断哪些值得暴露：
- **推荐暴露**：状态间视觉差异大，且切换过程有动画需要调
- **建议跳过**：状态间没有动画过渡（纯静态内容替换），或差异太小

不同组件的重点不同——sidebar 的核心可能是展开/收起，卡片的核心可能是 hover 动效。不要预设哪类更重要，全部找出来再筛。

#### 1C. 呈现给用户

**不展示代码、文件路径、变量名。** 直接给：

```
建议暴露以下 N 个感觉旋钮：

1. 入场速度 — 整体动画快慢
2. 错峰节奏 — 元素一个接一个出现的间隔
3. 上浮距离 — 元素从下方升起的幅度
...
N+1. 全部（含底层参数）

可切换状态：
- 默认 / 悬停 / 自由（不干预）

去掉哪个？或直接确认。
```

**阻断**：纯静态组件无可调参数 → 告知用户先做动效再接面板，终止。

**等用户确认后进入阶段 2。**

---

### 阶段 2：写代码

用户确认后一口气完成。

#### 命名规范

| 字段 | 规则 | 示例 |
|------|------|------|
| 导出常量名 | `UPPER_SNAKE_CASE` + `_MOTION` 后缀 | `SIDEBAR_MOTION` |
| id | 全小写 kebab-case | `"chat-input"` |
| label | 中文 | `"聊天输入框"` |
| schema.group | 中文，简短 | `"入场节奏"`、`"视觉强度"` |
| schema.key | camelCase，语义化 | `"entrySpeed"`、`"hoverLift"` |
| states.value | 全小写英文 | `"hover"`、`"collapsed"` |
| states.label | 中文 | `"悬停"`、`"收起"` |

#### React

```tsx
import { useVibeset } from "vibemotion/react";
import type { MotionTargetDef } from "vibemotion";

const COMPONENT_MOTION: MotionTargetDef = {
  id: "组件id",
  label: "中文名称",
  schema: [/* 用户确认的旋钮 */],
  defaultConfig: { /* 当前硬编码值 */ },
  states: [
    { value: "free", label: "自由" },
    // ... 用户确认的状态
  ],
  defaultState: "free",
};

function Component() {
  const { ref, config, previewState, lastCommit } = useVibeset("组件id", COMPONENT_MOTION);
  // ① config 驱动渲染（替换硬编码）
  // ② previewState 切换视觉状态（"free" 时不干预）
  // ③ lastCommit 变化 → 跑一遍 preview（必须接！）
  return <div ref={ref} data-motion-target-id="组件id">...</div>;
}
```

#### 非 React

```js
store.register({ id, label, schema, defaultConfig, states, defaultState }, element);

store.bus.on("change", (d) => { /* 用 getConfig 驱动渲染 */ });
store.bus.on("param-commit", (d) => { /* 走一遍 preview（必须接！） */ });
```

#### param-commit 必须接

松手后必须走一遍 preview。写完立即验证：调参 → 松手 → preview 动了没。没动就是漏了。

#### 验证

```bash
npx tsc --noEmit && npm run build
```

---

### 阶段 3：报告

```
✅ [组件名] 已接入调参面板

修改：[文件] — N 个旋钮，K 个状态
验证：tsc ✅ build ✅

用法：点右下角「动效编辑」→ 选中组件 → 拖滑杆。
收尾：点「复制代码」→ 发给 AI 更新参数 → 删掉 vibemotion 代码。
```

---

## 注意事项

- `defaultConfig` 值必须和当前硬编码一致（接入前后行为不变）
- 无状态需求时省略 states/defaultState
- 始终保留 "free"（自由）状态
- 纯静态组件 → 先做动效再接面板
- 面板热插拔：上线前删掉 vibemotion 代码即可

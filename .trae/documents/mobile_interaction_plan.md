# 移动端与桌面端交互逻辑统一优化计划 (Mobile Interaction Optimization Plan)

## 1. 摘要 (Summary)
当前项目在桌面端通过鼠标左中右键实现了完善的建造与视角控制，但在移动端存在操作缺陷（无法单指旋转视角，且点击即建造导致容易误触）。
本计划将优化 Pointer 事件逻辑，通过区分“点击(Tap)”与“滑动(Swipe/Drag)”，在移动端实现与桌面端体验一致的简易操作：**单指点击建造/拆除，单指滑动旋转视角，双指滑动平移与缩放**。

## 2. 现状分析 (Current State Analysis)
- `OrbitControls` 配置中，`touches.ONE` 被设置为 `null`，导致移动端用户无法通过单指滑动来旋转视角。
- `onPointerDown` 直接触发了建造/拆除逻辑。这在移动端意味着只要手指接触屏幕就会立刻放置建筑，这与常见的“滑动屏幕来浏览”的直觉相冲突。
- 移动端缺少类似鼠标 Hover 的状态，导致在 `pointerdown` 触发时 `cursor.visible` 可能为 `false`，从而产生首次点击无效的 bug。

## 3. 提出的修改 (Proposed Changes)

### 3.1 修改 `OrbitControls` 触摸配置
**文件**: `index.html` (约 275 行)
- 将 `controls.touches.ONE` 从 `null` 更改为 `THREE.TOUCH.ROTATE`。
- **效果**: 允许移动端用户使用单指滑动来顺滑地旋转 3D 视角。

### 3.2 优化 Pointer 事件流 (区分点击与滑动)
**文件**: `index.html`
- **新增变量**: 在交互系统区域新增 `let pointerDownCoords = { x: 0, y: 0 };` 记录手指/鼠标按下的初始坐标。
- **改造 `onPointerDown`**:
  - 移除原有的建造/拆除逻辑。
  - 仅用于记录起始坐标：`pointerDownCoords = { x: event.clientX, y: event.clientY };`
- **新增 `onPointerUp` 监听器**:
  - 计算 `pointerup` 和 `pointerdown` 时的坐标距离（`Math.hypot`）。
  - **阈值判定**: 如果距离大于 `10px`，则判定为用户的滑动操作（视角控制），直接 `return` 不执行建造。
  - 如果距离小于 `10px`，则判定为“点击 (Tap)”。
  - 在“点击”判定成立后，主动执行射线检测 (`raycaster.setFromCamera`) 获取当前精确的三维网格坐标 `gridX, gridZ`。
  - 将原 `onPointerDown` 中的边界判定、拆除、检查占用、旋转、替换及建造逻辑完整迁移到 `onPointerUp` 中。
- **更新事件绑定**:
  - 在 `renderer.domElement` 上添加 `addEventListener('pointerup', onPointerUp)`。

### 3.3 更新 UI 操作指南说明
**文件**: `index.html` (约 64 行)
- 使用 Tailwind CSS 的响应式类名（`hidden md:block` 和 `md:block` vs `md:hidden`），为桌面端和移动端分别展示最合适的操作提示：
  - **桌面端**：左键点击放置/切换、右键拖动旋转、滚轮/中键缩放与平移。
  - **移动端**：单指点击放置/切换、单指滑动旋转、双指滑动缩放与平移。

## 4. 假设与决策 (Assumptions & Decisions)
- **采用 Pointer Events**：浏览器的 Pointer API 原生支持 `event.clientX/Y`，无论鼠标还是触摸都会触发，统一逻辑可以极大简化代码。
- **10像素防抖阈值**：手机屏幕上触摸可能会有微小的抖动，设定 10px 的误差容限可以有效防止将正常的点击误判为滑动。
- **保留双指操作**：`TWO: THREE.TOUCH.DOLLY_PAN` 保持不变，满足移动端的缩放和平移需求。

## 5. 验证步骤 (Verification)
1. **桌面端测试**：鼠标左键点击空地能正常放置建筑，左键按住拖动不会放置且不影响原有控制，右键拖动正常旋转视角。
2. **移动端模拟测试**（使用浏览器开发者工具）：单指轻触屏幕能准确放置或替换建筑，单指滑动屏幕能顺滑旋转视角且不触发放置，双指滑动能正常缩放和平移。
3. UI 面板在不同屏幕尺寸下能正确显示对应的操作指南。
# 车辆逻辑车道重构计划 (Logical Lanes Refactoring Plan)

## 1. 摘要 (Summary)
当前项目中小车的“靠右行驶”仅仅是通过模型子节点的视觉偏移（Visual Offset）来实现的，小车的实际逻辑坐标（`mesh.position`）仍然在道路正中间。
本计划将彻底重构车辆的底层运动逻辑，将 1x1 的道路网格在逻辑上划分为两条独立的车道。车辆将真实地在其对应的物理车道上行驶，并在路口通过计算车道交点来实现精确转向，从而达到类似现实世界中多车道独立运行的效果。

## 2. 现状分析 (Current State Analysis)
- **生成逻辑**：`spawnCar` 将车辆生成在网格正中心 `(gridX, gridZ)`。
- **运动逻辑**：车辆直接向目标方向移动。在接近网格中心（`distToCenter < 0.05`）时触发路口判断逻辑并直接改变速度。
- **偏移逻辑**：在渲染循环中，通过修改 `car.mesh.children` 的 `position.x/z` 实现虚假的视觉偏移，底层碰撞和位置判断仍在道路中心。
- **掉头逻辑**：在断头路掉头时，只反转了速度和旋转，没有切换车道坐标。

## 3. 提出的修改 (Proposed Changes)

### 3.1 移除视觉偏移代码
**文件**: `index.html` (约 2253-2272 行)
- **修改**: 删除 `car.currentOffset` 及 `car.mesh.children.forEach` 中设置 `child.position.x/z` 的代码。车辆模型的子节点将始终保持在局部坐标原点。

### 3.2 车辆生成坐标对应物理车道
**文件**: `index.html` (`spawnCar` 函数)
- **修改**: 在车辆生成时，根据初始随机的 `velocity` 向量和 `rightOffset = 0.2`，直接计算出车辆应处的物理车道坐标，并将 `carMesh.position` 设置在该坐标上。
  - 若向 +X 行驶，Z 坐标 = `gridZ + 0.2`
  - 若向 -X 行驶，Z 坐标 = `gridZ - 0.2`
  - 若向 +Z 行驶，X 坐标 = `gridX - 0.2`
  - 若向 -Z 行驶，X 坐标 = `gridX + 0.2`

### 3.3 重构路口决策与车道交点转向逻辑
**文件**: `index.html` (`updateTraffic` 函数)
- **状态追踪**: 为车辆新增属性 `car.currentGridKey`。当车辆的 `mesh.position` 进入一个新的网格时，立即触发导航决策，而不是等到靠近中心才触发。
- **决策提前**: 将原先基于网格中心的转向决策逻辑（如直行、转弯、十字路口随机等）提取并在进入新网格时立即执行，计算出车辆在该网格内的目标速度 `targetVelocity`。
- **计算转向交点 (Turn Point)**:
  - 根据几何特性，两条靠右行驶的车道相交点可以直接通过公式求出：
    `turnX = gridX - (currentVelocity.z + targetVelocity.z) * 0.2`
    `turnZ = gridZ + (currentVelocity.x + targetVelocity.x) * 0.2`
  - 将此交点存入 `car.turnPoint`。
- **精确转向执行**: 
  - 车辆沿着当前车道直线行驶，当其越过 `turnPoint` 对应坐标时（如向 +X 行驶时 `position.x >= turnPoint.x`），瞬间将 `position` 贴合到 `turnPoint`。
  - 更新车辆的 `velocity` 为 `targetVelocity`，并更新 `rotation.y` 以朝向新方向。

### 3.4 修正掉头(U-Turn)逻辑
**文件**: `index.html` (`updateTraffic` 函数，越界/断头路处理)
- **修改**: 触发掉头时，除了反转速度（`-vX, -vZ`）和旋转模型（`+Math.PI`）外，必须将车辆的物理坐标平移到对向车道上，以保持逻辑一致性。

## 4. 假设与决策 (Assumptions & Decisions)
- **硬转弯与平滑旋转**: 为了保证逻辑网格的严谨性与碰撞准确性，车辆在 `turnPoint` 会立即改变速度向量（硬转弯）。为了视觉效果，我们可以保留在 `turnPoint` 处瞬间转向，或通过插值让车身在几帧内平滑旋转。为了稳定性，本计划优先保证轨迹的精准。
- **车道偏移量**: 统一使用 `0.2` 作为靠右偏移量（相对于1x1的网格中心）。

## 5. 验证步骤 (Verification)
1. **直线行驶**：车辆应严格偏离网格中心线行驶，不产生任何子对象相对位移。
2. **十字路口交汇**：对向行驶的两辆车在十字路口相遇时，应在各自的物理车道上错开，不发生重叠。
3. **路口转弯**：车辆在 T 型路口或十字路口转弯时，应精确行驶到车道交点再转向，并在转向后完美契合目标道路的右侧车道。
4. **断头路掉头**：车辆在断头路掉头后，应正确切换到另一侧的对向车道继续行驶。
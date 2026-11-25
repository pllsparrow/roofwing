# Roofwing 技术 SPEC

版本 0.1 · 2026-11-25 · 选定架构，代码实现待开始。

## 1. 运行架构

浏览器加载 Vite 构建的 HTML、CSS、JavaScript 和 Rapier WASM。
TypeScript 管理应用状态、输入、模拟、相机和场景；Three.js 负责渲染；Rapier 负责
碰撞查询和运动约束；localStorage 保存小型版本化进度。单机版本由静态站点托管。

选定依赖在代码初始化时固定版本并提交 lockfile。Node.js 开发环境优先使用 24 LTS，
实际 engines 约束以届时固定的 Vite 版本要求为准。浏览器目标为支持所选 Three.js
版本渲染要求的桌面 Chrome/Edge 与 Firefox；Safari 需要额外实测后声明支持。

桌面客户端为后续阶段。模拟与互动规则通过输入动作和存档接口工作，保持与 DOM、
浏览器存储和托管地址分离。未来再根据平台、包体、分发与更新需求评估 Tauri、
Electron 等封装方案；客户端的数据迁移、文件权限和打包需要另行验证。

## 2. 模块与调用顺序

```text
src/
  main.ts                 初始化与销毁
  app/                    菜单、应用状态和资源启动
  input/                  键盘、鼠标、焦点与动作映射
  simulation/             固定步长、鸟状态、目标与互动规则
  physics/                Rapier World、碰撞体和查询
  world/                  场景配置、建筑、栖息点与巢材
  rendering/              Three.js 场景、麻雀模型与动画
  camera/                 第三人称、平滑与遮挡处理
  persistence/            存档解析、验证和恢复
  ui/                     DOM HUD、菜单和可访问状态提示
```

每帧收集输入 → 累计时间 → 执行固定模拟步 → 更新可展示状态 → 插值姿态与相机 →
更新 DOM 变化 → 渲染。模块通过明确的状态与动作数据通信。
渲染层不能发放巢材或修改存档进度。

## 3. 时间与运动

模拟固定步长 1/60 秒，每帧最多补偿 5 步。过大的帧间隔截断并放弃积压时间，避免
恢复标签页后长距离跳跃。暂停清空累计时间和按键状态；恢复使用新的帧时间基准。
位置和速度单位为米和米/秒。角色具有一个 Rapier kinematic body 与胶囊碰撞体。

鸟的期望运动由自定义控制器计算，再由 Rapier 的碰撞查询/character controller
约束位移。飞行模式关闭平台游戏式自动台阶与自动吸地。
通过实际形状扫掠处理高速接触，通过法线去除进入表面的速度分量。
斜顶、薄墙、建筑转角和世界边界作为专门用例。

固定步长提升模拟一致性，游戏仍需验证不同帧率下的控制结果。涉及三角函数、
输入采样或不同依赖版本时，不声明跨浏览器逐位确定性。

## 4. 鸟状态与操作合同

| 状态 | 进入条件 | 主要行为 | 退出条件 |
| --- | --- | --- | --- |
| Grounded | 有有效地面支撑 | WASD 行走与朝向 | Space 起飞，或失去支撑进入 Glide |
| Takeoff | Grounded/Perched 时按 Space | 有限起飞脉冲，检测头顶障碍 | 短起飞阶段结束后进入 Flap/Glide |
| Flap | 空中按住 Space | 有上限的爬升与推进，振翅动画 | 松开 Space 进入 Glide |
| Glide | 空中松开 Space | 惯性推进与受控下降 | 按 Space 振翅，或进入合法降落 |
| Landing | 接近可支撑表面，满足速度与姿态条件 | 平滑减速并接触表面 | 接触稳定后进入 Grounded/Perched |
| Perched | 停在指定屋檐/栏杆/巢附近的安全栖息点 | 静止观察，可开始起飞 | Space 起飞 |

W/S 控制前进/减速，A/D 控制转向。空中速度有上限，向后输入用于减速。
鼠标拖动旋转观察相机，释放拖动不改变鸟的运动意图。
E 根据当前提示执行拾取或交付；Esc 暂停。所有动作允许之后重映射。
鼠标转动相机与键盘转向的具体灵敏度在运动测试场验收后固定。

合法降落的初始调参目标为接近表面、速度不高于 3 m/s、接触法线距上方向不超过
35 度。空中最高速度初始为 10 m/s。数值保留在独立配置，基于手感实测调整并记录。
高速撞墙进入减速/滑移反馈。掉出场景、无效数值或卡死恢复动作返回安全栖息点，
携带物与交付进度保持一致。

## 5. 互动、场景与相机

场景对象使用稳定 ID，渲染 Mesh 和物理 Collider 引用相同配置。每片巢材只有一个
权威状态 `available | carried | delivered`。一次只能携带一片，交付是可重复调用但
只改变一次进度的状态迁移。拾取需要距离足够、当前可交互和双脚已稳定落地。
三片 delivered 后进入 completed，世界继续可探索。

相机以目标点和期望位置为输入，按时间进行指数平滑，使用目标到相机的遮挡查询
缩短距离。恢复位置、重开或回巢时重置插值历史。禁止保存相机穿墙后的临时偏移。

重复窗户、树冠等共享几何/材质的物体优先用 InstancedMesh。地面和建筑使用简单
静态碰撞体，装饰 Mesh 不逐一附加物理体。阴影只保留关键光源和有限覆盖范围。
窗口 resize 更新渲染尺寸与相机投影，设备像素比默认封顶 1.5。

## 6. 存档合同

存储键 `roofwing.save.v1`，JSON 包含 `schemaVersion`、`worldVersion`、`updatedAt`、
每片巢材的状态、`carriedItemId`、`safePerchId`、音量与设置。
世界位置通过稳定栖息点恢复；重新进入游戏时从安全点起步。

每次拾取、交付、合法栖息和设置改变后保存；保存前验证状态一致性。解析时验证
版本、ID 白名单、数量、枚举值和有限数值。新世界删除的栖息点回退初始巢边，
未知版本保留原存档并提示恢复选择。损坏存档先保存可下载的原始副本，再经玩家
确认重置。存储被拒绝或额度满时提示无法持久保存，当前会话可继续。
重新开始只清理本项目的存档键，并需确认。多标签页检测 `storage` 事件，暂停当前
会话并提供载入较新进度或明确覆盖选择，避免自动覆盖另一页的进度。

## 7. 应用生命周期与故障

应用状态为 `loading | title | playing | paused | completed | error`。
加载失败、WASM 初始化失败、WebGL 不支持或 context lost 都有明确提示和恢复路径。
context lost 时暂停模拟；context restored 后重建必要资源并从当前安全状态恢复。
失焦/隐藏时暂停并清空输入；音频由首次用户手势解锁，音量设置持续生效。
销毁时取消 RAF、移除事件监听、释放 Rapier World 和 Three.js geometry/material/
texture，关闭或释放音频资源。重复进入场景需要验证资源数量不会持续增长。

## 8. 检查与性能证据

Vitest 针对状态迁移、互动幂等性、帧间隔处理、存档校验和恢复。固定模拟输入
分别经过 30/60/120 FPS 调度，检查位移与目标结果的误差边界。
真实浏览器验收覆盖第三人称手感、碰撞、相机遮挡、暂停、声音与完整筑巢流程。
自动化仅覆盖可重复规则与构建；实际浏览器行为由作者验收，除非主动委托浏览器操作。

初始性能目标是在明确记录的集成显卡桌面环境、1080p、DPR 1 下，预热 10 秒后
运行 60 秒固定路径，p95 帧耗时不超过 33.3 ms；60 FPS 为优化方向。记录设备、
OS、浏览器、提交、场景、分辨率、DPR、帧耗时分布、draw calls 和初始下载大小。
当前没有性能测量结果。性能目标必须由实际构建验证后才能进入 README 成果描述。

## 9. 技术依据

- [Three.js game architecture](https://threejs.org/manual/pages/game.html)
- [Three.js instancing](https://threejs.org/docs/pages/InstancedMesh.html)
- [Rapier controllers](https://rapier.rs/docs/user_guides/javascript/character_controller/)
- [Rapier determinism conditions](https://rapier.rs/docs/user_guides/javascript/determinism/)
- [localStorage behavior](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage)

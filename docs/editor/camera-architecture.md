# 摄影机与关键帧架构

本文把 Flashback `90dfa34` 的源码结论与 Playback 本分支的实现边界放在一起。Flashback 是行为参考，不是 Java/Minecraft API 的直接移植目标。

## Flashback 源码结论

### 关键帧求值

Flashback 的权威数据是 `EditorScene.keyframeTracks` 中的 `KeyframeTrack`。每条轨道用 `TreeMap<Integer, Keyframe>` 按整数 tick 保存关键帧；`createKeyframeChange(float tick, RealTimeMapping)` 在当前小数 tick 上即时求值，不预先生成一组插值帧。

求值流程如下：

```text
Replay tick (fractional)
  -> EditorState.applyKeyframes
  -> enabled KeyframeTrack.createKeyframeChange
  -> KeyframeChange.apply(KeyframeHandler)
```

每个关键帧同时有左、右侧插值语义。相邻两端的 Hold、Linear/Ease、Smooth 和 Hermite 会合并决定这一段的结果；Smooth 使用前后邻帧的 Catmull-Rom 风格求值，Hermite 会在连续的非 Hold 区间收集多点。启用 `RealTimeMapping` 时，插值比例先经过速度关键帧定义的 real-time 映射。

### 摄影机与最终渲染

- `CameraKeyframe` 保存 position、yaw、pitch、roll。
- `FOVKeyframe` 是独立轨道；其平滑插值先转换到焦距空间，再转换回 FOV，因此不是简单地把 FOV 塞进 CameraKeyframe。
- `MinecraftKeyframeHandler` 在切换 camera entity 时只发送一次 `spectate`，随后以 `player.snapTo(...)` 写入姿态、取消玩家原生插值并清零速度；这不是每帧循环执行 `/tp`。
- `MixinCamera` 在渲染边界应用精确 partial-tick 姿态、roll、镜头抖动和 FOV/投影。

### 关键帧预览与摄影机轨道

Flashback 没有发现“每个关键帧预生成一张图片”的 `KeyframePreview` 数据结构。预览是实时 viewport、时间轴拖动/跳转、选中关键帧后的 Apply，以及 `CameraPath` 绘制的空间路径。`CameraPath` 通过捕获型 Handler 重复调用 `EditorState.applyKeyframes` 生成线段，仅负责编辑器可视化，不是第二套运行时相机轨道。

## Playback 本分支重构

### 单一权威模型

`EditorStateExt::cameras` 中的 `CameraEntity` 现在同时承载：

- `keys/path/rig/preset` 四类相机数据源；
- `enabled`（是否参与求值）与 `pathVisible`（是否绘制编辑器路径）；
- binding、limiter、shake、locked 等轨道状态。

旧的 `CameraTrackExt`、`cameraTracks`、`activeCameraIndex` 和 `active` 镜像已删除。`hasCameraSource(camera, kind)` 与 `isCameraRenderable(camera)` 是类型分派和回退的共同不变量：禁用相机或当前类型没有 backing data 时，不能成为预览、Sequence、导出或路径 overlay 的来源。

### 求值与时间

`CameraTimelineEvaluator` 接受 `ReplaySampleTime`（有理数 numerator/denominator），每次采样按 fractional tick 即时计算。当前支持：

- Keyframe 的 Hold、Ease、Cubic Bezier、AutoSmooth/Catmull-Rom；
- 局部双切线 Hermite 段；
- Path、Rig、Preset/Orbit、limiter 与确定性 shake；
- yaw/pitch/roll 的最短角插值，FOV 峰值偏移作用于 FOV。

这与 Flashback 的“按当前小数 tick 求值”保持一致，但当前 Hermite 是单段局部切线，不是 Flashback 的全局多点 Hermite；`RealTimeMapping`、独立焦距空间 FOV 轨道和逐关键帧左右侧插值仍未实现。

`ReplaySession::getRenderSampleTime(Timer::mAlpha)` 是预览时间入口；导出使用有理数样本。一个 render pass 只发布一个不可变 `CameraTimelineRenderContext`，相机 Hook 与导出实体 render scope 共享该样本，避免各模块各自累积 float 时间。

### 摄影机应用边界

`CameraRenderHooks` 在 Bedrock 原生 `LevelRendererPlayer::setupCamera` 完成后才覆盖最终 `mce::Camera` 的 position/orientation/FOV。回放实体仍由原生 movement packet 与 Bedrock 插值驱动；本分支不把编辑器姿态写回 `LocalPlayer`，也不增加第二套输入/网络控制源。

`ClientCameraCapture` 是控制器和 viewport 共用的捕获入口，统一读取 position、yaw、pitch、roll、FOV。viewport 的自由相机控制只发布 render-only override，添加关键帧时优先捕获该 override。

### 时间轴与预览交互

- 点击相机行或关键帧会选择并请求该相机预览；请求无效时控制器清除旧的 preview id，避免残留上一台相机。
- 点击关键帧一次会 seek 到其 tick；拖动时先显示本地临时 marker，松开后只提交一次可 undo 的 `MoveCameraKeyframe`。
- 拖动遵守前后关键帧边界，并可按 20 tick 网格 snap。
- `enabled` 控制运行时来源；`pathVisible` 只控制 overlay。锁定相机阻止关键帧/类型/删除等内容编辑，但仍允许切换这两个轨道级可见性状态。
- overlay 只绘制选中、启用、当前类型有 backing data 且 `pathVisible` 的相机；路径和当前 marker 都通过指定相机 evaluator 采样，不会回退到 Sequence 或首个相机。Keyframe kind 才绘制关键帧圆点。

### 绑定相机的当前边界

`createBindingCamera` 会创建 `Preset` backing data，并捕获 SubActor 创建时的 position/rotation，确保新相机立即可求值。当前没有把回放实体每 tick 的实际姿态注入 `SubActor`/evaluator 的数据链，因此 `FollowEntity` 仍是数据模型与占位分支，不应宣称为实时跟随。

## 仍需补齐的 Flashback 对齐项

1. 逐关键帧左右侧插值语义和跨 Hold 区间的全局 Hermite。
2. `RealTimeMapping` 以及独立 FOV/focal-length 轨道。
3. 绑定实体的实时 pose provider（包括 seek、切段和实体消失时的 fail-closed 行为）。
4. 更完整的相机投影策略与导出帧确认协议。

## 验证边界

`xmake run camera-timeline-tests` 验证纯 evaluator、Registry 精确采样、命令不变量、绑定相机初始化和 fractional tick 数学；`xmake -y playback` 与 `git diff --check HEAD` 验证编译/静态完整性。Minecraft 内的实际预览、拖动、seek、Sequence 切换、导出首帧和停止后的 Hook/Registry 清理仍需要手工运行时验收。

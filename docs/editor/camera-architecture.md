# 摄影机与关键帧架构

本文记录 Playback 当前实现、`magicobs0z/Playback:feat-Camera` 和 Flashback 的边界，并定义本分支采用的运行时契约。

## 对照结论

| 维度 | 当前 `develop` | `feat-Camera` | Flashback | 本分支 |
| --- | --- | --- | --- | --- |
| 时间线 | `CameraTimelineEvaluator` + 原子 Registry | `CameraSampler` 另起一套，控制器直接推送姿态 | `EditorState -> KeyframeTrack -> KeyframeChange` | 保留 Registry 单一发布入口，增强唯一 evaluator |
| fractional tick | 导出时钟模块内计算，预览公式分散在渲染 Hook | `ReplaySession::getCameraTime` 自建累积量 | 每次渲染读取 `getPartialReplayTick()` | `ReplaySession::getRenderSampleTime(Timer::mAlpha)` 统一提供精确样本 |
| 相机应用 | `setupCamera` 最终覆盖 `mce::Camera` | 主要写回 `LocalPlayer`，并加多个运动/输入/网络拦截 | `MinecraftKeyframeHandler` 写玩家姿态，`MixinCamera` 修正最终相机 | 独立 `CameraRenderHooks` 在原生 `setupCamera` 后一次覆盖，不修改 Player |
| 实体运动 | 原生回放运动为主，导出时对渲染位置做临时 scope | 为宿主玩家增加 `moveTo`、能力修改和多处阻断 | 原生实体状态为主，精确位置在相机渲染阶段修正 | 保持原生 movement packet 链路；仅导出使用实体 render scope |
| 插值 | Hold/easing/线性 | Bezier/AutoSmooth，但与主 evaluator 重复 | Hold、Linear/Ease、Smooth、Hermite、RealTimeMapping | Hold/easing/Bezier/AutoSmooth、路径 spline、rig/preset 基础运动、limiter/shake |
| 生命周期 | Preview/Export 两个原子 timeline binding | 相机状态和 Player 能力分布在 ReplaySession | EditorState 生命周期集中 | Registry 发布带 shared ownership；hook 与 context 可独立安装/清理 |

`feat-Camera` 中的调试 HTTP 上报、废弃的空 `CameraRenderOverride` 和将镜头写回宿主玩家的方案没有合入：它们会引入第二个相机控制源，并把编辑器时间线与回放实体模拟耦合在一起。

## 运行时数据流

```text
ReplaySession::mCurrentTick + Timer::mAlpha
        |
        v
ReplaySession::getRenderSampleTime()
        |
        +--> CameraTimelineRenderContext (thread-local, one render pass)
        |          |
        |          +--> Preview: CameraTimelineRegistry::sampleCameraTimeline()
        |          |
        |          +--> Export: immutable sample prepared by publishOfflineRenderClockSample()
        |
        +--> ReplayEntityInterpolator (export scope only)

LevelRendererPlayer::setupCamera(origin)
        |
        +--> CameraRenderHooks: apply sampled mce::Camera state
```

### 时间契约

- `mCurrentTick` 是唯一已应用的整数回放 tick；待处理 seek 不会污染渲染样本。
- 播放时样本位于 `[max(mCurrentTick - 1, 0), max(mCurrentTick - 1, 0) + Timer::mAlpha]`，与实体 pose history 的 `[tick - 1, tick]` 对齐。
- 暂停时样本为 `mCurrentTick/1`，不会读取旧的 alpha。
- 导出使用 `ReplaySampleTime` 的有理数值，不经过 float 累积。

### Hook 边界

- `OfflineRenderClockHooks` 只负责 `MinecraftGame::updateGraphics` 的离线 Timer 覆盖和导出实体 render scope。
- `CameraRenderHooks` 只负责 `LevelRendererPlayer::setupCamera` 的最终相机写入，预览和导出共享同一入口。
- `CameraTimelineRenderContext` 只在一次 `origin(updateGraphics)` 调用期间有效，渲染 Hook 不持有编辑器对象或裸指针。
- 导出帧在进入 Bedrock 前先按同一个 `ReplaySampleTime` 求出 camera sample；`CameraRenderHooks` 应用后回执，导出只有在实体 pose 和 camera 都确认后才认为时钟已应用。
- 回放玩家继续接受原生 movement packet；任何需要精确到 fractional tick 的实体结果通过 `ReplayEntityInterpolator` 的临时渲染覆盖完成。
- 连续播放/连续导出保留 Bedrock 的 movement interpolation；seek 或 snapshot 的跳变只清零该 actor 的 pending interpolator steps，等价于 Flashback 的 `interpolation.cancel()` 边界。

## 后续扩展

1. 将 RealTimeMapping、Hermite 和实体绑定相机作为 evaluator 的纯数据输入扩展，不新增渲染 Hook。
2. 需要改变投影时，在 `CameraRenderHooks` 增加独立 projection policy，避免把 FOV/投影状态塞进回放 tick。
3. 运行时验收应覆盖播放、暂停捕获、seek、Sequence 硬切、导出首帧和停止后的 Hook/Registry 清理；构建通过不等价于这些行为已被证明。

# AGENTS.md

给 AI 代理与后来的维护者的工程导航。读完这一份，应该能在不改坏东西的前提下定位任何一处玩法逻辑。

- **工程类型**：Unity 6（URP）3D 柏青哥（pachinko）模拟，纯 C#，无第三方插件业务代码
- **物理后端**：Unity 3D 物理（PhysX），`Rigidbody` + `PhysicMaterial` + 碰撞回调，**没有**自写求解器
- **原始版本**：Unity `6000.0.41f1`（见 git 里的 `ProjectSettings/ProjectVersion.txt`）
- **入门文档**：先看 `README.md`（作者原版说明，英文），玩法与美术流程在那边；代码逐行解析看 `docs/SourceCodeAnalysis.md`

## 一分钟跑起来

1. 用 Unity Hub 安装 `6000.0.x`（≥ 6000.0.30 均可，用低补丁号打开会弹版本确认框，点 Continue 即可）
2. 打开工程目录 → 首次导入会拉 UPM 包并重建 `Library/`（几分钟）
3. 打开场景 `Assets/__Scenes/MainMenu.unity`（Build Settings 里 MainMenu → MainGame 两场景），点 ▶ Play

键位：`空格`/鼠标左键 按住蓄力、放松发射；`V` 切换四机位；`N` 换歌；`P` 暂停。

> **首次导入卡住的已知问题**：GUI 模式下有时会卡在 "Compiling Scripts" 空转（UPM 包其实已下完）。可靠做法是先跑一次无界面导入：
> `Unity.exe -batchmode -quit -projectPath <工程目录> -logFile <日志>`，跑完再开 GUI，秒进。

## 目录结构

```
Assets/
  Scripts/
    Core/       系统层：GameManager(中枢) Player(输入) BallPoolManager(对象池)
                SFXManager(音效) Jukebox+Song(音乐) StuckBallTracker(卡球)
    Prefabs/    台面件：Ball BallLauncher Bumper Pin Spinner ScorePocket
                CollectionTray Cabinet + 三个自定义 Inspector(BumperEditor/
                ScorePocketEditor/SpinnerEditor)
    UI/         9 个面板/控件脚本（主菜单、选项、结算、蓄力条、冷却条…）
  Prefabs/      实际预制体（Game Elements/Playfield|Cabinet|Game State, Audio, UI）
  Materials/    包含 Physics/ 下 7 个 PhysicMaterial —— 手感核心
  __Scenes/     MainMenu.unity（入口）、MainGame.unity（台面）
  Settings/     URP 管线资产（PC/Mobile RPAsset）
  Audio/        Music(10 首分层曲) + SFX
```

## 运行时装配（谁是谁）

四个单例，全部在场景/预制体里摆放，没有运行时动态创建：

| 单例 | 入口 | 职责 |
|---|---|---|
| `GameManager.Instance` | `Game Elements/Game State/GameManager.prefab` | 计分、加球、4 机位切换、结束判定、最高分存档 |
| `BallPoolManager.Instance` | 同目录 | 球的池化、预热、卡球回收、队列洗牌 |
| `SFXManager.Instance` | `Audio/SFXManager.prefab` | 所有音效的统一出口（按名称/语义方法播放） |
| `AudioManager.Jukebox.Instance` | `Audio/AudioManager (Jukebox).prefab` | 分层音乐（同一首歌的 intensity 轨道交叉淡入） |

数据流：`Player`（输入）→ `BallLauncher`（蓄力/发射）→ `BallPoolManager`（取球）→ 台面碰撞件（`Bumper`/`Pin`/`Spinner`）→ `ScorePocket`（判定得分）→ `GameManager.AddPoints` → UI。

**改玩法优先改这条链上的组件，不要去改场景里散落的数值**——大部分手感参数都 `[SerializeField]` 在组件上，Inspector 可见。

## 物理约定（改之前必读）

- 重力 `-9.81`，`Fixed Timestep = 0.02`（50Hz），`Maximum Allowed Timestep = 0.3333`
- **`Bounce Threshold = 2`**：相对速度低于 2 m/s 的碰撞 Unity 直接不反弹。想让小球轻微抖动也弹起来，必须改这个值（在 Project Settings → Physics）
- 球体重心低、半径小，靠 `PhysicMaterial` 而不是脚本调手感；7 个材质的完整数值见 `docs/SourceCodeAnalysis.md`
- 球的 `PhysicMaterial` 的 Combine 模式设成了 **Maximum**，所以实际弹性由「球 vs 台面件」两者取大值决定，不是球自己的 0.2
- 球标签必须是 `Ball`：`OnCollisionEnter` / `OnTriggerEnter` 全部用 `CompareTag("Ball")` 过滤，改标签会让所有交互静默失效

## 修改守则

1. **不要提交 `Library/` `Temp/` `Logs/` `UserSettings/`**（已在 `.gitignore`），也不要提交 `ProjectSettings/ProjectVersion.txt` 的本地降级改动
2. **不要用 `git checkout --` 还原 LFS 大文件**。仓库用 Git LFS 管理 wav/png/exr，工作区是真二进制、索引里是指针文本，`git status` 会常驻 40+ 条假差异，这是正常的
3. 新增台面件沿用现有三段式：`OnCollisionEnter`/`OnTriggerEnter` 只做判定 → 施力/计分 → 反馈（音效 + 灯光 + 缩放脉冲）
4. 需要新的可调参数，加 `[SerializeField] private` + `[Header]`，不要 `public` 字段；外部只读用 getter（参考 `BallLauncher.GetCurrentLaunchForce()`）
5. 需要新面板，参考 `UI/` 下现成脚本的 `Show/Hide + Fade` 套路，并从 `GameManager` 拿引用，不要在面板里 `FindObjectOfType`

## 已知坑（都验证过，别再踩）

| 现象 | 原因 | 处理 |
|---|---|---|
| GUI 首次打开卡 "Compiling Scripts" 40 分钟不动 | Unity 6 首次导入的 UPM/编译阶段 GUI 死锁 | 先 `-batchmode -quit` 预热 `Library` |
| `SFXManager.Instance is null` 警告 | 场景里 SFXManager 预制体初始化顺序晚于消耗方 | 无害，功能自动降级；要清干净就在 `GameManager` 之前摆 SFXManager |
| 键盘输入完全没反应 | `Player` 用老 `Input` API，且 `useKeyboardControls` 可被关掉 | 检查 Inspector 勾选；PIE 里先点一下 Game 视图 |
| 计分洞偶尔不记分 | `ScorePocket.OnTriggerEnter` 有 `!isProcessingBall` 门，处理上一颗球时进来的球不会被跟踪 | 设计取舍，要修就改成用队列而非单标志 |
| 球卡在钉阵里不落 | 有 `StuckBallTracker`（速度 <0.05 + 位移 <0.01 持续 3s）+ 池子低水位回收（10s 一次，连续 2 次触发） | 调 `StuckBallTracker.IsStuck` 阈值，或池子 `poolCheckInterval` |
| 暂停后蓄力条/结算不刷新 | 部分协程用 `WaitForSeconds`（受 `Time.timeScale` 影响），暂停时挂起 | 需要实时语义就换 `WaitForSecondsRealtime` |

## 文档索引

- `README.md` —— 作者原版概述（英文）
- `docs/SourceCodeAnalysis.md` —— 27 个脚本逐个解析：参数、状态机、物理公式、作者的设计取舍

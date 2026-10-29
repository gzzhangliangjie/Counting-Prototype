# Counting-Prototype 源码解析

这份文档逐脚本、按真实代码路径解析这个 3D 柏青哥工程。所有数值都是从 `Assets/Scripts/*.cs`、`Assets/Materials/Physics/*.physicMaterial` 和 `ProjectSettings/` 直接读出来的，不是推测（除个别标注「推导」的地方）。

- 工程：Unity 6（`6000.0.41f1`），URP 17
- 物理：Unity 内置 PhysX 3D，`Rigidbody` + `PhysicMaterial` + 碰撞回调
- 脚本：27 个 C# 文件，约 4600 行；无 Assembly Definition，全部在默认程序集

---

## 0. 全局一览

```
Player(输入) ──► BallLauncher(蓄力/发射) ──► BallPoolManager(取池中球)
                                                   │
                                                   ▼
                              台面件：Pin / Bumper / Spinner / Cabinet(碰撞回调)
                                                   │
                                                   ▼
                              ScorePocket(触发+沉降判定) ──► GameManager.AddPoints
                                                   │                    │
                                                   ▼                    ▼
                              CollectionTray(托特动画)          UI(分数/蓄力条/结算)
```

四个单例，全部由场景预制体提供（`Assets/Prefabs/Game Elements/Game State/`、`Assets/Prefabs/Audio/`）：
`GameManager`、`BallPoolManager`、`SFXManager`、`AudioManager.Jukebox`。

物理时间步：`Fixed Timestep 0.02`（50Hz），重力 `-9.81`，`Bounce Threshold 2`，`Default Contact Offset 0.01`。

---

## 1. Core/GameManager.cs（21.7KB，中枢）

单例，持有几乎所有 UI 引用和相机引用。用户能感知的一切「全局状态」都在这里。

**状态字段**
- `currentScore`：当前分（`AddPoints` 累加，`isGameOver` 后拒绝加分）
- `totalBallsEarned`：本局累计获得的奖励球（结算面板用）
- `isGameOver`：结束锁
- `activeCamera`：四机位中的当前机位

**四机位**
`mainCamera`（含玻璃正视）→ `noGlassCamera`（去玻璃正视）→ `highCamera`（含玻璃俯视）→ `highNoGlassCamera`（去玻璃俯视）→ 回到 main。
切换走 `TransitionToCamera()` 协程：时长 `1 / cameraSwitchSpeed`（默认 2 → 0.5 秒），做法是把旧相机 `backgroundColor.a` 从 1 淡到 0 做伪交叉淡入，同时把新相机 `depth` 设为 `旧depth - 1` 藏在后面，切完再复位。全程手动管理 `AudioListener`（保证任何时刻只有一个开启），这点做得很干净。

**结束判定（两条并行的路径）**
1. `BallLauncher.LaunchBall()` 每次发射后调 `CheckGameOver()` → 等一个 `FixedUpdate` + `EndOfFrame` 让物理结算到位，再数场上 `Ball` 标签对象
2. `GameOverCheckRoutine()` 常驻协程：初始等 1 秒，之后每 2 秒查一次

判定条件：
- `launcher 剩余球 == 0 && 场上球 == 0` → 结束
- `launcher 剩余球 == 0 && 场上有球但全部卡住超过 3 秒` → 结束（调用 `CheckForStuckBalls`）

> 两条路径都可能触发 `EndGameSequence()`，靠 `isGameOver = true` 在第一行就把门关掉，实际不会重入。

**EndGameSequence()**（`endGameDelay = 2` 秒）
1. 置 `isGameOver`、隐藏暂停按钮、`player.DisablePauseInput()`
2. 等 2 秒（让最后一颗球的计分动画走完）
3. 把场上残余球 `SetActive(false)`
4. 存档：`PlayerPrefs` 两个键 —— `PachinkoHighScore`（最高分）、`BallsEarnedHighScore`（最多奖励球）
5. 破纪录播 `PlayNewHighScoreSound()`，否则播 `GameOver`
6. 若是含玻璃机位，自动切到对应去玻璃机位（看结局更清楚）
7. 把数据塞进 `GameOverPanel`（`SetHighScore / SetFinalScore / SetBallsEarned / SetBallsEarnedHighScore`）后 `StartFadeIn()`

**其他**
- `AddBalls(n)`：转发给 `BallLauncher.AddBalls`，累加 `totalBallsEarned`，并弹 `BonusBallNotification`
- `UpdatePowerMeter(p)`：转发 `PowerMeterUI` 后强制 `Canvas.ForceUpdateCanvases()`（作者为了避免 UI 掉帧手动刷的）
- `RestartGame()`：重载当前场景

---

## 2. Core/Player.cs（11.2KB，输入）

不继承任何输入框架，直接读老 `Input` API（`Input.GetKeyDown` 等）。

| 参数 | 默认 | 说明 |
|---|---|---|
| `launchKey` | `Space` | 蓄力/发射 |
| `cameraToggleKey` | `V` | 切机位 |
| `nextSongKey` | `N` | 下一首 |
| `pauseKey` | `P` | 暂停 |
| `useKeyboardControls` / `useMouseControls` | 都 true | 键鼠双通道；鼠标是左键 |
| `inputDebounceTime` | `0.1` | 每次按键后锁输入 0.1s 防连发 |

**发射状态机**：`GetKeyDown` → `StartCharging()`（先查 `ballLauncher.IsCoolingDown()`，冷却中只播点击音并刷蓄力条）→ 置 `isChargingLaunch = true` → `GetKeyUp` → `LaunchBall()`（算 0~1 功率百分比、调发射器、触发 `OnBallLaunched` 事件）。

**暂停**：`Time.timeScale = 0` + 隐藏 `gameplayElements` + 显示 `OptionsPanel`（并在 `Start` 里把 `optionsPanel.disableEscKeyHandling = true`，把 ESC 的处置权收到自己手里）。`OnDestroy` 里会把 `timeScale` 复位，避免退出时游戏卡在暂停态。

**相机/音乐**：`ToggleCameraView()` 转发 `GameManager`；`PlayNextSong()` 直接找 `Jukebox`，用 `(current + 1) % songCount` 循环。

---

## 3. Core/BallPoolManager.cs（10.9KB，对象池）

| 参数 | 默认 | 说明 |
|---|---|---|
| `initialPoolSize` | `20` | 初始池大小 |
| `expandIfNeeded` | `true` | 池空时新建 |
| `poolCheckInterval` | `10` | 巡检周期（秒） |

**预热 `PrewarmPool()`**：把 20 个球逐个 `SetActive(true)` → `WakeUp()` → 清速度 → `SetActive(false)`。目的是让 PhysX 提前分配刚体/碰撞体，避免第一次发射时卡帧。这一步在做物理游戏时很关键。

**`GetBall()`** 的防御性写法值得学：
- 出队的球如果 `activeInHierarchy` 为真（说明池里混进了在场景中的球），塞回队尾并继续找，`maxAttempts` 用初始池量兜底，不会死循环
- 拿到的球统一清 `linearVelocity` / `angularVelocity`、`isKinematic = false`
- **不在这里激活**，把 `SetActive(true)` 留给发射器「摆好位置之后再激活」

**`ReturnBall()`**：优先调 `Ball.ResetForPool()`（顺带通知所有计分洞移除跟踪），没有 `Ball` 组件时退化为「遍历所有 `ScorePocket` 调用 `OnBallReturnedToPool`」，最后 `SetActive(false)` + 入队。

**`PeriodicPoolCheck()`**：每 10 秒
- 池量 < `initialPoolSize * 0.25`（即 <5）时计数，连续 2 次 → `RecoverStuckBalls()`（把 `StuckBallTracker.IsStuck()` 为真的球直接回收）
- 池量健康时做 `ShufflePool()`：Fisher-Yates 洗队列。目的写在注释里——避免总是复出队首那几颗球，导致某些球的物理状态被反复沿用

---

## 4. Core/StuckBallTracker.cs（2.8KB，卡球检测）

极简但够用：速度 ≤ `0.05` 时才累计 `positionCheckTimer`，每满 `0.5` 秒比较一次位移，位移 `< 0.01` 就加 `stuckTime`；`IsStuck(阈值=3.0)` 供 `GameManager` 与 `BallPoolManager` 查询。

因为加在球预制体上（`Pachinko Ball.prefab`），所以是每球独立计时。`GameManager.CheckForStuckBalls` 里如果发现球上没这组件会 `AddComponent` 补上（容错）。

---

## 5. Core/SFXManager.cs（21.8KB，音效）

统一音效出口，内部维护一组 `AudioSource`（`AudioSourcePrefab SFX.prefab`）。

- 通用入口：`PlaySFX(string sfxName, float volume = -1, float pitch = 1)`
- 直接播 clip：`PlaySound(AudioClip, Vector3? position, volume, pitch)` —— 传 position 就是 3D 空间音
- 语义封装：`PlayScoreSound(int points)`、`PlayBallLaunchSound(power)`、`PlayPinHitSound(force)`、`PlayBumperHitSound(force)`、`PlaySpinnerSound(speed)`、`PlayButtonClickSound()`、`PlayDrainSound()`、`PlayGameOverSound()`、`PlayNewHighScoreSound()`、`PlayWoodenImpactSound(position, force)`
- 音量：`SetMasterVolume` / `SetSFXVolume`

设计上「算法在调用方，音效只负责播」：碰撞速度的归一化（`impactForce / 5f` 等）都是在 `Pin`/`Bumper` 里算好再传进来的。

---

## 6. Core/Jukebox.cs + Song.cs（音乐）

`Song` 是 `ScriptableObject`：`songName`、`reverbTail`、`intensityClips`（同一首歌的多个强度轨道，`List<AudioClip>`）。

`Jukebox` 对 `AudioMixer` 操作：
- `PlaySong(int songIndex)` / `PlaySong(PlaySongOptions)`，`PlaySongOptions` 支持 `song / intensity / startTime / blendOutTime / blendInTime`
- `SetIntensity(int)` 及带淡入淡出时长的重载 —— 玩法紧张度可切轨道（本工程里没做自动强度变化，留了接口）
- `StopSong(fadeOutDuration)`、`GetCurrentSongIndex()`、`GetSongCount()`、`GetClipTimeRemaining()`

工程里有 10 首分层曲（`Assets/Audio/Music/Clips/` 下 4 个曲目文件夹，每首含 Main / Intensity 1 / Intensity 2 / Cut 30 / Cut 60 等变体），总量约 216MB，用 Git LFS 管理。

---

## 7. Prefabs/Ball.cs（2.6KB）

薄壳，两个职责：
1. `containingPockets` 列表：记录这颗球当前在哪些 `ScorePocket` 里（避免球被回收后洞口残留陈旧引用）
2. `ResetForPool()`：清线速度/角速度、`isKinematic = false`、把材质 alpha 复位到 1（消除洞口淡出动画的残留）、逐个 `pocket.OnBallReturnedToPool(gameObject)`、清列表

`OnDisable` 里再做一次清列表兜底。

---

## 8. Prefabs/Pin.cs（1.6KB，钉阵）

**纯反馈，不施力**——钉子的物理完全交给 `PhysicMaterial Pin`（动/静摩擦 0.2、弹性 0.4），脚本只在 `OnCollisionEnter` 里播音效：

- 撞的是 `Ball` 才处理
- `impactForce = Clamp(relativeVelocity.magnitude, 0, 5)`，归一化 `/5` 后传给 `PlayPinHitSound`
- 0.2 秒冷却防风铃式连响（`lastSoundTime` 初值 0.1，所以开局 0.3 秒内的第一下会被吞掉，属小瑕疵）

---

## 9. Prefabs/Bumper.cs（3.3KB，保险杠）

| 参数 | 默认 |
|---|---|
| `bounceForce` | `5` |
| `minImpactVelocity` | `2` |
| `cooldownTime` | `0.1` |

命中流程（`OnCollisionEnter`）：

```csharp
// 1. 门：球 且 相对速度 >= 2
// 2. 方向 = (球心 - 保险杠心).normalized      ← 用位置差，不用接触法线
// 3. ballRb.linearVelocity = Vector3.zero;   ← 先清零
// 4. ballRb.AddForce(bounceDirection * 5, ForceMode.Impulse);
// 5. 音效（按 relativeVelocity 调力度）+ 灯光闪烁 + 0.1s 冷却
```

**先清零再给固定冲量**是这个工程最核心的设计取舍：牺牲真实能量守恒，换「每次弹开速度可预测、不会越撞越快」。作者在注释里明确写了 "Cancel existing velocity for more predictable bounce behavior"。

灯光反馈 `FlashLightRoutine`：从 `原亮度 × 2` 在 0.2 秒内线性降到 **0**（注意不是降到原亮度，所以原本常亮的灯撞一次后会熄灭，想要常亮得自己改回 `originalLightIntensity`）。

---

## 10. Prefabs/Spinner.cs（9.3KB，转盘，工程里最讲究的一个）

绕**本地 Z 轴**（`Vector3.forward`）自转，纯脚本驱动（刚体约束死位置和 X/Y 旋转）。

| 参数 | 默认 | 作用 |
|---|---|---|
| `maxRotationSpeed` | `360` | 角速度上限（度/秒） |
| `spinRateDecay` | `0.5` | 每秒按 50% 衰减 |
| `spinnerFriction` | `0.5` | 摩擦减速系数 |
| `ballSpinTorque` | `2.0` | 给球的扭矩 |
| `minImpactVelocity` | `0.5` | 触发门槛 |
| `hitImpulseMultiplier` | `10` | 切向冲量 → 角速度的放大倍数 |
| `bounceForce` | `3.0` | 弹开力基数 |
| `hitScaleDuration` / `hitScaleFactor` | `0.1` / `1.1` | 缩放脉冲 |

**旋转积分（避开欧拉角 ±180° 翻转 bug）**

```csharp
// 不用 transform.Rotate，改累计角度 + 四元数基线
totalRotationAngle += currentRotationSpeed * Time.deltaTime;
transform.rotation = initialRotation * Quaternion.AngleAxis(totalRotationAngle, Vector3.forward);
currentRotationSpeed -= currentRotationSpeed * spinRateDecay * Time.deltaTime;
```

作者的注释直接吐槽了：「When a rotation 180 normally it flips to -180, causing the breaking of the spinner we saw」。这是 3D 转盘类物件最常见的坑，值得抄。

**命中计算（把球速分解到切向）**

```csharp
Vector3 radial  = 归一化(接触点 - 转盘中心).xy;   // 半径方向
Vector3 tangent = (-radial.y, radial.x, 0);        // 切向（逆时针 90°）
float tangentialForce = Vector3.Dot(relativeVelocity, tangent) * relativeVelocity.magnitude;
currentRotationSpeed += tangentialForce * 10;      // 二次量 × 10，然后夹到 ±360
```

注意 `Dot × magnitude` 是**二次量**，所以轻碰几乎不转、重碰一下直接到上限——手感偏「弹一下飞转」，不是线性响应。

**对球的三个作用力**（`ApplyPhysicsEffects`）

```csharp
ballRb.linearVelocity *= (1f - 0.5f * Time.fixedDeltaTime);          // 摩擦减速
ballRb.AddTorque(spinDirection * 2f * Sign(currentRotationSpeed), Impulse);
ballRb.AddForce(spinDirection * (|speed| / 100f) * 3f, Impulse);     // 顺着旋转方向推
ballRb.AddForce(contactNormal * 3f * 0.5f, Impulse);                 // 法向弹开
```

**刚体配置**：`useGravity = false`、`Interpolate`、`Discrete`、`FreezePosition | FreezeRotationX | FreezeRotationY`。

**缩放脉冲**：0.1 秒内先 30% 时间放大到 1.1 倍，剩下 70% 时间回弹（快进慢出）。

---

## 11. Prefabs/ScorePocket.cs（24.5KB，计分洞，工程里最复杂的一个）

两层判定：**触发进入** + **沉降确认**。

| 参数 | 默认 | 说明 |
|---|---|---|
| `pocketType` | 枚举 | `LowScore / MediumScore / HighScore / Jackpot` |
| `pointValue` | `100`（≤0 时按类型补默认） | 类型默认值：100 / 500 / 1000 / 5000 |
| `requiredSettleTime` | `0.2` | 速度达标需持续多久才算进洞 |
| `velocityThreshold` | `1.0` | 线速度与角速度都必须低于它 |
| `maxSettleWaitTime` | `2.0` | 等太久强制判定（防死等） |
| `scoreDelay` | `0.2` | 播完音效到真正加分之间的延迟 |
| `returnDirectlyToPool` | `false` | false 走「飞向收集托盘」动画，true 走淡出回收 |
| `ballAwardChance` | `0.2` | 奖励球基础概率 |
| 档位倍率 | `1.0 / 1.2 / 1.5 / 3.0` | Low / Medium / High / Jackpot |

**进入/离开的误判处理**：`OnTriggerEnter` 把球丢进 `Dictionary<GameObject, BallSettleData>`；`OnTriggerExit` 不直接删，而是用 `triggerCollider.bounds.Intersects(ballRenderer.bounds)` 复核，只有「真的在外」才移除——解决了球贴边抖动导致的假退出。

**沉降判定（`UpdateBallSettlement`，每帧）**
```csharp
if (linearVel < 1 && angularVel < 1) settleTimer += dt;  // 达标才累加
else settleTimer = 0;                                     // 一动就清零
if (settleTimer >= 0.2) → 判定进洞
if (totalTimeInPocket >= 2.0) → 强制判定（保底）
```
先 `WakeUp()` 保证读到的速度是真实的（睡眠刚体读速度不可靠，这细节做得对）。

**得分与球的去向**
```
ProcessScoredBall → 播音效 → 启动 DelayedScoring
  0.2s 后：GameManager.AddPoints(pointValue)
  然后：returnDirectlyToPool ? 淡出 0.5s 后回收 : 1 秒 Lerp 飞向 collectionPoint/收集托盘，再回收
```

**奖励球（`AttemptBallAward`）——注意代码与注释不一致**

```csharp
bool shouldAwardBalls =
    pocketType == LowScore || pocketType == HighScore || pocketType == Jackpot
    || Random.value <= actualChance;
```

因为前三个档位被 `||` 短路成恒真，所以 **Low / High / Jackpot 是 100% 出球，只有 Medium 走概率（0.2 × 1.2 = 24%）**。也就是说四个档位的 `chanceMultiplierFor*` 里，三个实际是死代码。

各档实际给球数：

| 档位 | 是否必给 | 球数 |
|---|---|---|
| LowScore | 必给 | 1 |
| MediumScore | 24% 概率 | `Random.Range(2,6)` = 2~5 |
| HighScore | 必给 | `Random.Range(5,11)` = 5~10 |
| Jackpot | 必给 | `Random.Range(10,21)` = 10~20 **＋** `min(20, 分数/5000)` **＋**（分数 > 50000 时再加 10） |

Jackpot 的球数是**随当前分数成长**的：分数越高，一次头奖给的球越多（最高 20+10+20=50 球）。

**其他**：`OnDrawGizmos` 按档位画不同颜色线框（蓝/紫/黄/红），`returnDirectlyToPool` 时额外画一个「向下消失」箭头，Scene 视图里非常直观。

---

## 12. Prefabs/CollectionTray.cs（12.2KB，收集托盘）

| 参数 | 默认 |
|---|---|
| `ballRetentionTime` | `0.5` |
| `immediateReturn` | `true` |
| `maxCapacity` | `5` |
| `ballFadeOutTime` | `0.3` |
| `velocityThreshold` | `0.1` / `maxSettleTime` `3` |
| 反馈 | 粒子 `trayParticles` + 灯光闪烁 `0.3s`（默认绿色）+ Animator 触发器 `"Collect"` |

负责「球进洞后短暂停留、再消失/回收」的观感层。容量满了会提前清。`ClearTray()` 供重开一局时清空。

---

## 13. Prefabs/BallLauncher.cs（9.2KB，发射器）

| 参数 | 默认 | 说明 |
|---|---|---|
| `minLaunchForce` | `5` | 蓄力下限 |
| `maxLaunchForce` | `15` | 蓄力上限 |
| `chargeRate` | `10`/秒 | 从 5 涨到 15 需要 1 秒 |
| `powerCurve` | `EaseInOut(0,0,1,1)` | 蓄力条显示曲线 |
| `cooldownTime` | `1` | 发射后 1 秒才能再蓄力 |
| `startingBalls` | `200` | 起始球数 |

**蓄力**：`Update` 里 `currentLaunchForce = Min(current + dt * 10, 15)`，同时刷 UI（曲线作用在 `(force-5)/10` 上，让进度条前段慢后段快）。

**发射（`LaunchBall`）**
1. 冷却中/无球/游戏结束直接返回
2. 启动 1 秒冷却协程，通知 `GameManager.ResetCooldownBar()`
3. `BallPoolManager.GetBall()` 取球 → `position = launchPoint.position` → 清速度 → `SetActive(true)`
4. `ballsRemaining--`，刷文本，`GameManager.CheckGameOver()`
5. **关键**：力不是立刻加的，而是

```csharp
IEnumerator DelayedLaunchForce(GameObject ball, float force) {
    yield return new WaitForFixedUpdate();               // 等物理注册新位置
    ballRb.AddForce(launchPoint.forward * force, ForceMode.Impulse);
}
```

因为刚 `SetActive(true)` 的刚体在这一帧还没进 PhysX 的 broadphase，直接施力会被吞或方向错乱。等一个 `FixedUpdate` 是最稳的做法，很多人踩过这个坑。

`ResetLauncher(n)` 把球数/冷却/蓄力全部复位（重开一局用）；`AddBalls(n)` 加球。

---

## 14. Prefabs/Cabinet.cs（2.1KB，机箱）

木箱边框的撞击音：`minImpactVelocity 0.5`、`impactForceScale 1.0`、冷却 0.1 秒，`Clamp(速度 × 1, 0, 10)` 后调 `PlayWoodenImpactSound(contactPoint, force)`（3D 定位音）。

---

## 15. UI 层（9 个脚本）

| 脚本 | 职责 | 关键点 |
|---|---|---|
| `PowerMeterUI` | 蓄力条 | `UpdatePower(p)` + `ForceRefreshText()`（供 `GameManager.ForceRefreshPowerMeter` 强制刷） |
| `CooldownBarUI` | 冷却条 | `ResetCooldown()` / `SetReady()` / `UpdateCooldown(p)` 三态 |
| `GameOverPanel` | 结算面板 | `SetFinalScore` / `SetHighScore(bool)` / `SetBallsEarned` / `SetBallsEarnedHighScore` + `StartFadeIn()` |
| `BonusBallNotification` | 奖励球飘字 | 静态 `ShowNotification(int)`，自带排队/淡出 |
| `MainMenuManager` | 主菜单 | 切场景、开场动画 |
| `OptionsPanel` | 选项面板 | `ShowPanel/HidePanel`、音量滑杆（存 `PlayerPrefs`）、返回主菜单；`disableEscKeyHandling` 让 `Player` 接管 ESC。`HideSaveConfirmation()` 用的是 **`WaitForSecondsRealtime`**——因为「保存成功」提示出现时游戏常常处于暂停（`timeScale = 0`），用普通 `WaitForSeconds` 会让提示永远不消失（上游最后一次提交修的就是这个） |
| `HowToPlayPanel` / `CreditsPanel` | 说明 / 制作名单 | 纯展示 |
| `OptionsButton` | 小按钮封装 | 只做点击音 |

---

## 16. 自定义 Editor（3 个，工程规范性的体现）

`BumperEditor` / `ScorePocketEditor` / `SpinnerEditor`：给组件加 `[CustomEditor]`，提供
- 一键填参数（如按档位自动写入 `pointValue` 与颜色）
- 场景内 Gizmos 可视化（保险杠作用半径、洞口范围与颜色、转盘的切向方向）
- 参数合法性检查（负值/零值提示）

这部分说明作者不只是「能跑就行」，而是按工具化标准搭的，改动时建议沿用同类做法而不是在 Inspector 里手调。

---

## 17. 物理参数全表

### 全局（`ProjectSettings/DynamicsManager.asset`、`TimeManager.asset`）

| 项 | 值 |
|---|---|
| Gravity | `(0, -9.81, 0)` |
| Fixed Timestep | `0.02`（50Hz） |
| Maximum Allowed Timestep | `0.3333` |
| Bounce Threshold | `2` ← **低于 2 m/s 的相对速度不产生反弹** |
| Default Contact Offset | `0.01` |
| Default Material | 无（全靠各碰撞体自带的材质） |

### 7 个 PhysicMaterial（`Assets/Materials/Physics/`）

`FrictionCombine` / `BounceCombine` 存的是枚举序号：`0 = Average`、`1 = Multiply`、`2 = Minimum`、`3 = Maximum`。

| 材质 | 动摩擦 | 静摩擦 | 弹性 | 摩擦组合 | 弹性组合 |
|---|---|---|---|---|---|
| **Ball** | 0.1 | 0.1 | 0.2 | Minimum (2) | **Maximum (3)** |
| **Bumper** | 0.2 | 0.2 | 0.5 | Average (0) | Maximum (3) |
| **Pin** | 0.2 | 0.2 | 0.4 | Average (0) | Maximum (3) |
| **Glass** | 0.3 | 0.05 | **0.7** | Minimum (2) | Maximum (3) |
| **Spinner** | 0.2 | 0.1 | 0.5 | Minimum (2) | Maximum (3) |
| **Wood** | 0.2 | 0.3 | 0.4 | Average (0) | Average (0) |
| **Soft Wood** | **1.0** | 0.1 | 0.1 | Minimum (2) | Minimum (2) |

**怎么读这张表**（Unity 的组合规则：两材质组合模式不同时，优先级 `Average < Minimum < Multiply < Maximum`，高优先级者胜）：

- 球的弹性组合是 **Maximum**，而大多数台面件也是 Maximum → **实际弹性取两者较大值**，所以同样的球在玻璃上弹 0.7、在钉子上弹 0.4、在木箱上弹 0.4，球自己的 0.2 几乎从不生效。这就是「一套球材质适配全台面」的实现方式
- 球的摩擦组合是 **Minimum** → **实际摩擦取两者较小值**，球本身 0.1 就够低，所以球在几乎所有表面上都很滑（柏青哥要的就是球能一直滚，不能"粘"在钉子上）
- 想改整体「弹不弹」，优先动 `Bounce Threshold`（全局）和 `Glass` / `Pin` 的 Bounciness；想改「滚不滚得动」，动 `Ball` 的 0.1（注意它是 Minimum，只能把结果压更低，压不上去）
- `Soft Wood` 的弹性 0.1 + Minimum 组合被球的 Maximum 压制，实际仍取 0.2 —— 它的「软」是通过 1.0 的动摩擦（再被 Minimum 压回 0.1）体现的，这组材质实际效果接近于「无特殊作用」，属于可优化项

---

## 18. 「想改 X 就改 Y」速查

| 想调的东西 | 改哪里 |
|---|---|
| 球整体弹不弹 | `ProjectSettings → Physics → Bounce Threshold`（现 2） |
| 玻璃墙特别弹 | `Glass.physicMaterial → Bounciness`（0.7） |
| 球滚得太快/太滑 | `Ball.physicMaterial → Dynamic Friction`（0.1，注意 Minimum 组合） |
| 保险杠弹多用力 | `Bumper.bounceForce`（5）＋ `minImpactVelocity`（2） |
| 保险杠连击节奏 | `Bumper.cooldownTime`（0.1） |
| 转盘转多快 / 隔多久停 | `Spinner.hitImpulseMultiplier`（10）/ `maxRotationSpeed`（360）/ `spinRateDecay`（0.5） |
| 蓄力手感（多久充满） | `BallLauncher.chargeRate`（10/s）＋ `powerCurve` |
| 发射力度区间 | `minLaunchForce` 5 / `maxLaunchForce` 15 |
| 发射节奏（间隔） | `BallLauncher.cooldownTime`（1s） |
| 进洞判定严不严 | `ScorePocket.requiredSettleTime`（0.2）/ `velocityThreshold`（1.0） |
| 出球（奖励球）多少 | `ScorePocket` 的 `ballAwardChance` 与四个 `chanceMultiplierFor*`（注意上文：只有 Medium 真的走概率） |
| 结算快慢 | `GameManager.endGameDelay`（2s） |
| 相机切换速度 | `GameManager.cameraSwitchSpeed`（2 → 0.5s 过渡） |
| 卡球判定 | `StuckBallTracker` 的三个阈值（0.05 / 0.5 / 0.01、3s） |
| 池子报警/回收 | `BallPoolManager.poolCheckInterval`（10s）、`initialPoolSize`（20，低水位 = 25%） |
| 起始球数 | `BallLauncher.startingBalls`（200） |

---

## 19. 代码里值得注意的瑕疵（改的时候顺手修）

1. **`ScorePocket` 的 `||` 短路**：Low/High/Jackpot 恒给球，三个倍率成死代码。如果要按文档意图「概率随档位提升」，应该改成对四档都做 `Random.value <= actualChance`，再把必给逻辑单独表达
2. **`ScorePocket.isProcessingBall` 是单标志**：处理一颗球期间，其他球进场不会被跟踪，可能「漏分」。改成队列即可
3. **`BallLauncher` 蓄力用 `Time.deltaTime` 累加**：高帧率/掉帧时蓄力速度不一致（掉帧会慢）。要严格一致改用 `Time.fixedDeltaTime` 或按 `fixedTime` 累加
4. **`Bumper.FlashLightRoutine` 结束把亮度设成 0**：原本常亮的灯撞一次就灭，若要复位应设回 `originalLightIntensity`
5. **`Pin` 的 `lastSoundTime` 初值 0.1**：开局 0.3 秒内的第一下撞击不出声
6. **协程大量用 `WaitForSeconds`**：受 `Time.timeScale` 影响，暂停时结算/淡出会挂起。需要实时语义的地方（如 UI 淡出）应换 `WaitForSecondsRealtime`
7. **`GameManager.CheckGameOver` 与 `GameOverCheckRoutine` 双路径**：能跑，但每帧 `FindGameObjectsWithTag("Ball")` 是 GC 热点，量大时建议换注册表（发射时登记、回收时注销）
8. **`Jukebox` 的强度切换接口没被玩法调用**：想加「越接近头奖音乐越紧张」，在 `ScorePocket`/`GameManager` 里调 `SetIntensity` 即可，架构已经铺好了

---

## 20. 与同目录另一个工程（UE5 `PinballBD`）的对照

| 维度 | Counting-Prototype（Unity / PhysX） | PinballBD（UE5 / Chaos） |
|---|---|---|
| 玩法 | 柏青哥：落珠 + 钉阵 + 计分洞 | 弹珠台：挡板（flipper）+ 保险杠 |
| 逻辑载体 | C# 明文 + ScriptableObject + 预制体 | 蓝图二进制（.uasset） |
| 物理手感来源 | 7 个 PhysicMaterial + 少量脚本施力 | 蓝图内 AddImpulse / Timeline 旋转 |
| 核心交互件 | Bumper / Pin / Spinner / ScorePocket | BP_Bumper / BP_Paddle / BP_SpeedStrip |
| 计分 | `GameManager.AddPoints`（集中式） | GameMode `PinballMode.AddScore`（集中式） |
| 对象管理 | 自建对象池 + 预热 + 卡球回收 | 单球 + 重置 |
| 输入 | 老 Input Manager（KeyCode） | Enhanced Input（IMC 资产） |
| 可读性 | 高（逐行可读、有注释） | 低（需解析二进制） |

两者共同点：物理手感都不靠「真实还原」，而是**先清零/规整速度，再给固定冲量**，用可预测性换手感。这个思路在柏青哥/弹珠台这类高频碰撞场景里是对的——真实物理在这种密集碰撞下反而会发散。

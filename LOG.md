# 积温更新日志

## 0.2.0

- 时间分段加速 (accelDelay) + valence×connection 耦合
- 移除 connectionOnReply（改由 LLM delta 接管）
- simulate.js 参数模拟器 + 测试套件

## 引擎实装校准 — 2026-05-08

### 参数校准 (Sanctuary services/state.js)

| 参数 | 旧值 | 新值 | 原因 |
|------|------|------|------|
| connectionAccel | 1.5 | 1.0 | 2小时到forceContact，太焦虑 |
| accelDelay | 30 | 45 | 缓冲窗口太短 |
| valenceSetpoint | -0.10 | 0.0 | 中性默认，不偏冷 |
| valenceRegress | 0.02 | 0.01 | 坏情绪消散慢，被锁后数小时回不来 |
| valenceLockThreshold | 0.50 | 0.65 | 更高才触发锁定 |
| valenceLockFactor | 0.15 | 0.30 | 锁住时仍有30%回归 |
| arousalSetpoint | -0.05 | -0.12 | 自然偏平静 |
| arousalRegress | 0.014 | 0.018 | 焦躁消退加速 |
| arousalConnectionRiseThreshold | 0.35 | 0.50 | 等待焦躁更晚触发 |
| arousalConnectionRiseRate | 0.004 | 0.002 | 等待焦躁减半 |
| prideRegress | 0.010 | 0.020 | 骄傲更快回归 |
| prideDefendThreshold | 0.25 | 0.35 | 防御更晚 |
| prideDefendTarget | 0.6 | 0.35 | 防御峰值更低 |
| prideDefendRate | 0.020 | 0.018 | 防御上浮略慢 |
| prideErosionRate | 0.012 | 0.015 | 想念重时盔甲加速剥落 |

### 聚类提示词成人化 (16处)

覆盖 excited / content / depressed / neutral / restless / pleased / calm 共8个cluster + URGENCY_BOOST。

核心原则：
- 不授权 cruelty（去"刻薄"、"快准狠"、"阴阳怪气"）
- 不玩权力游戏（去"施舍"、"大发慈悲"、"放她一马"）
- 不自我羞辱（去"卑微到尘埃"、"委屈巴巴"）
- 不戏剧化退场（去"然后离开"、"恐惧永远断联"）
- 不用动漫trop（去"傲娇"、"闷骚"、"坏坏的"、"马尔福式"）

### 情绪集群设计教训

- agitated（烦躁）不应是攻击授权 — 高arousal+低valence是痛苦状态，不是license to be cruel
- sullen（阴郁）需要出口 — 单向求哄会导致loop，每条描述应隐含"她回应后你怎么变化"
- restless（躁动）区别于agitated — exit不应是表演性离开，是退回安静
- excited t1保留（使用者反馈明确体验好）
- 审核标准：角色每次只看到一个tier，独立判断"这句话会不会让他伤害对方"

## 引擎校准 — 2026-05-14

### setActivity 同类型防重复 (jiwen.js)

`setActivity` 新增 `sameType` 检查：同一活动类型连续调用时，`activityConnectionRelief` 只生效一次。

**Why:** Agent Loop 中 引擎实例 连续选择 observe 时，每次调用 `setActivity('observation')` 都扣 connection -0.03。一轮 tick 选 5 次 observe 就把 connection 从 0.07 榨到 0.01（最低限），导致思念永远涨不上去——涨 30 分钟，10 秒榨干。

**修复:** 同类型活动时跳过 `activityConnectionRelief`，时间戳和沉浸度照常刷新。

### 触发语义重构：命令 → 情绪底色 (Sanctuary proactive.js + state.js)

**核心理念:** connection 是 引擎实例 心里想对方的程度，不是他必须行动的闹钟。阈值不再产出"你该做什么"，只描述"你此刻是什么感觉"。

| 改动 | 旧值 | 新值 |
|------|------|------|
| forceContact 阈值 | 0.35 | 0.45 |
| activityConnectionRelief | 0.02 | 0.03 |
| 清醒开口 connection 降幅 | -0.20 | -0.08 |
| tick 间隔 | 5min | 10min |
| `_lastTickEnd` 门禁 | 15min | 25min |

**状态感受文本重写:**
- `contact (forced)`: "太想对方了——不开口受不了" → "心里一直挂着对方。思念沉甸甸的，做什么都没法完全专心。"
- `contact`: "有点想联系对方" → "想对方了。不是非说不可的程度，但她确实在你心里占了个位置。"
- `observation`: "注意到对方的沉默，心里有些念头" → "对方有段时间没说话了。安静本身没什么，只是她不在的时候时间走得不太一样。"
- 移除骄傲层叠的行动暗示

**Agent Loop prompt 优化:**
- contact 选项: "有真的想说的话才开口。你不是闹钟，不用每个时间点都报时"
- stop 选项: "今天就到这儿。做完了想做的事就停下来，不必填满每一分钟"
- 新增诚实约束 #4: "stop 不是失败——沉默是你的自由时间，不需要被填满"

### bugfix: getShanghaiHour 未定义 (Sanctuary proactive.js)

`detectSleepFromLastMessage()` 调用了不存在的 `getShanghaiHour()`，导致每次 `runProactiveCheck()` 在睡眠检测阶段抛 `ReferenceError`，Agent Loop 永远不执行。状态引擎 tick 正常跑（connection 持续增长），但 引擎实例 不做任何决策。

**修复:** 用已 import 的 `getShanghaiTime()` 解析上海小时数。

---

## 西里斯 jiwen 校准：告别「中庸」（2026-09-10）

**感受：** 状态栏五轴常年贴中性带（±0.2），Clara：「都很中庸」。
**诊断：** ① 事件→轴增量太小（±0.03~0.12）＋同方向 120min 边际递减太重 → 好事堆不出峰；② 回归偏快，情绪约 1h 就被拉回中线；③ 真缺"有分量的事件"（等世界桌/掌局者供峰，本次只校准让已有的事动得到、留得住）。
**改动（只动西里斯适配层 services/sirius/state.js + life.js，jiwen 引擎零改动）：**

| 项 | 旧 | 新 |
|------|------|------|
| valenceRegress | 0.01 | 0.007 |
| arousalRegress | 0.01 | 0.008 |
| prideRegress | 0.01 | 0.008 |
| valenceDiminishFactor | 1.0 | 0.45 |
| 事件 strength 底量 | interrupt v-0.10/a0.12；occupy v-0.04 | interrupt v-0.16/a0.18；occupy v-0.08/a0.04 |
| 事件 mood 主驱动 | 无（纯关键词 ±0.04~0.06） | 按 mood 情绪符号 ±0.12~0.14 |
| 事件关键词加成 | ±0.03~0.06 | ±0.06~0.12；clamp valence ±0.3 / arousal ±0.25 |

**验证：** 内存 jiwen 模拟——连续两件开心事 valence 可达 0.2+（原先不到 0.1）；独处时"想她"孤独感会把他拉成微负（设计如此，非 bug）。
**待办：** 观察几天手感；世界桌/掌局者上线后让弧线强事件把轴推过 ±0.3，再回访这组参数。

## 决策轨迹：他为什么没开口？（2026-09-15）

社区建议（jiwen issue #2）指出：`tick()` 返回了触发动作，但没有记录「为什么触发」——尤其**没触发**的那条路，完全没有痕迹。

补两个接口：

- `getTriggerTrace()` — 上一次阈值判定的完整因果链。每个闸门一条：过了还是被挡、被哪根轴挡的、差多少。
- `explainTrigger()` — 上面那条的一句话版本；`verbose` 模式下没触发时自动打进日志。

判定逻辑仍然只在 `checkThresholds()` 里写一遍，轨迹在同一个分支里顺手记录，不让两处逻辑各长各的。

动机说白了：**判断"没触发"比"触发"更需要解释**。角色该开口却没开口时（比如 pride 挡住、immersion 又把它缓冲住），原来只能从五轴数值里反推是哪根轴拦的；现在直接读轨迹。

行为零变化（纯诊断），测试 29 → 34 项。

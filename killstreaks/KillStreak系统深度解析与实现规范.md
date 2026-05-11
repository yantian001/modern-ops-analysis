# KillStreak 连杀奖励系统深度解析与迁移规范

> Modern Ops: Gun Shooting Games v9.79 | Unity 6000.0.69f1 | 逆向分析文档
>
> 数据来源：AssetRipper 反编译 + JSON 配置文件

---

## 目录

1. [系统全景](#1-系统全景)
2. [数据层：配置文件与 JSON 结构](#2-数据层配置文件与-json-结构)
3. [核心数据类（非战斗）](#3-核心数据类非战斗)
4. [战斗状态机](#4-战斗状态机)
5. [战斗运行时：三大实现分支](#5-战斗运行时三大实现分支)
6. [管理器体系](#6-管理器体系)
7. [网络协议](#7-网络协议)
8. [视图系统](#8-视图系统)
9. [积分追踪与击杀积累](#9-积分追踪与击杀积累)
10. [装备/背包系统](#10-装备背包系统)
11. [AI 集成](#11-ai-集成)
12. [单机迁移实现方案](#12-单机迁移实现方案)

---

## 1. 系统全景

KillStreak（连杀奖励）系统由以下五个相互协作的子系统组成：

```
┌──────────────────────────────────────────────────────────────┐
│                     KillStreak 系统架构                        │
├────────────┬──────────────┬─────────────┬──────────┬─────────┤
│  配置数据层 │  装备/背包层  │  战斗运行时  │  网络层  │  视图层  │
│ CRCD13.json│ KillStreakSlot│ BattleKS*  │  Photon  │ First/  │
│ CRCD34.json│ LocalKSSlot  │ Manager    │  Events  │ Third/  │
│ JSON_KS.cs │ Inventory    │ Controller │  Ops     │ TopDown │
└────────────┴──────────────┴─────────────┴──────────┴─────────┘
```

### 关键数量

| 指标 | 值 |
|------|---|
| 唯一 KillStreak 类型 | 12种（sn维度）/ 11种（KillStreakType维度，Copter含2种） |
| 每种 KillStreak 升级等级 | 5～7级 |
| 配置记录总数 | 76条（CRCD13）+ 64条升级链（CRCD34） |
| 每个玩家装备槽位 | 3个（Slot.First/Second/Last） |
| 每套装备（Kit）支持 KS 数 | 3个 |
| 并发数量限制 | 按 GameModeType + KS类型由服务器配置 |

---

## 2. 数据层：配置文件与 JSON 结构

### 2.1 配置文件来源

| 文件 | 描述 | 记录数 |
|------|------|-------|
| `Assets/Resources/config/crc/CRCD13.json` | 所有 KillStreak 基础数据 | 76 |
| `Assets/Resources/config/crc/CRCD34.json` | KillStreak 升级链 | 64 |
| `Assets/Resources/config/loccrc/LOC_CRCD24.json` | 本地化文本（名称+描述） | 24 |

### 2.2 CRCD13.json 字段结构（JSON_KillStreak）

```csharp
// GameLogic/JSON/JSON_KillStreak.cs
[Serializable]
public class JSON_KillStreak : JSON_BaseItem
{
    public int   i;    // 唯一 ID（主键）
    public short t;    // KillStreakType 枚举值
    public int   c;    // CostExecution：激活所需击杀积分
    public int   cd;   // Cooldown：冷却时间（秒）
    public int   ord;  // 排序权重（UI展示顺序）
    public float stDi; // Range：范围/射程
    public float stDa; // Damage：伤害值
    public float stSp; // Mobility：移动速度（可负数）
    public float stAm; // Ammo：弹药量
    public float stHe; // Health：实体生命值
    public float stAr; // Armor：实体护甲值
    public float stSt; // Strength：爆炸力
    public float stDu; // DurationSec：持续秒数
    public short stLv; // FakeLevel：UI显示等级（展示用）
    public float stDps;// FakeDps：UI显示DPS（展示用）
    public int   aXp;  // ArmoryExp：军械库经验值
}
```

**sc（购买费用）字段：** 根类型 `JSON_BaseItem` 提供，格式为嵌套对象：
```json
{ "tPv": 1000 }   // 软货币（金币）
{ "tPr": 500 }    // 高级货币（钻石）
```

**sn（系统名）字段：** 继承自 `JSON_BaseItem`，字符串类型，对应 LOC key 前缀 `ks_{sn}_name`。

### 2.3 CRCD34.json 字段结构（JSON_KillStreakUpgrade）

```csharp
// 继承自 JSON_BaseUpgrade，字段命名规律与装备系统一致
{
    "u_id": 3,      // 升级链 ID（同一 sn 的所有升级共享）
    "lvl":  2,      // 升级阶段（1=第一次升级）
    "sid":  301,    // 升级前的 KillStreak ID（source）
    "rid":  302,    // 升级后的 KillStreak ID（result）
    "nlvl": 10,     // 玩家等级需求
    "ena":  1,      // 是否激活（1=是）
    "sc": { "tPv": 75000 }  // 升级费用
}
```

### 2.4 JSON_InventoryKillStreak（玩家背包数据）

```csharp
// GameLogic/JSON/JSON_InventoryKillStreak.cs
[Serializable]
public class JSON_InventoryKillStreak
{
    public int id;   // 拥有的 KillStreak ID
    public int u_id; // 对应的升级链 ID
    public int u_eD; // 升级解锁数据（bitmask）
}
```

### 2.5 运行时数据合约（KitKillstreaksDC）

战斗开始时，服务器将玩家装备的连杀奖励发送给客户端，封装在 `KitKillstreaksDC` 中：

```csharp
// Penta/Network/Photon/Battle/DataContract/UserKit/KitKillstreaksDC.cs
public sealed class KitKillstreaksDC : IReadOnlyDictionary<byte, KillstreakDC>
{
    // key = slot index (0/1/2)
    private readonly Dictionary<byte, KillstreakDC> _killstreaks;

    public sealed class KillstreakDC
    {
        public readonly int            ID;                  // KillStreak ID
        public readonly KillStreakType Type;               // 类型枚举
        public readonly int            CoolDown;           // 冷却毫秒数
        public readonly short          Cost;               // 激活所需积分
        public readonly bool           InterruptOnOwnersDeath; // 死亡中断
        public readonly bool           IsTempKillstreak;   // 是否临时KS
        public readonly int?           DurationMSec;       // 持续时间（毫秒）
        public readonly BattleUnitType? BType;             // 战斗单元类型
        public readonly Dictionary<byte, object> WeaponData; // 武器KS的武器配置
    }
}
```

---

## 3. 核心数据类（非战斗）

### 3.1 KillStreak（商店/背包实体）

```csharp
// KillStreak.cs
public sealed class KillStreak : CCItem, IComparable<KillStreak>, IUpgradableCCItem
{
    // 只读属性（从 JSON_KillStreak 构造）
    public readonly KillStreakType KType;
    public readonly int    Cooldown;        // 秒
    public readonly int    CostExecution;   // 激活所需击杀积分
    public readonly float  Range;
    public readonly float  Damage;
    public readonly float  Mobility;        // 正=加速 负=减速
    public readonly float  Ammo;
    public readonly float  Health;
    public readonly float  Armor;
    public readonly float  Strength;
    public readonly float  DurationSec;
    public readonly float  FakeDps;
    public          short  FakeLevel { get; }
    public          int    ArmoryExp { get; }

    // 升级系统接口（IUpgradableCCItem）
    public int         UpgradeID { get; set; }  // 对应 CRCD34.u_id
    public BaseUpgrade Upgrade   { get; set; }  // KillStreakUpgrade 实例
    public bool CanBeUpgradedToNextLevel => false; // AssetRipper截断

    // 工具方法
    public static bool IsWeaponKillstreakType(KillStreakType t); // 判断是否为武器替换型
    public static string GetKillstreakModelnameByType(KillStreakType type); // 获取3D模型名
}
```

### 3.2 KillStreakUpgrade（升级链节点）

```csharp
// KillStreakUpgrade.cs
public class KillStreakUpgrade : BaseUpgrade, IComparable<KillStreakUpgrade>
{
    // BaseUpgrade 提供：
    // - CurrentShopItem (KillStreak)  当前等级
    // - NextShopItem    (KillStreak)  下一等级
    // - Cost            (ShopCost)    升级费用
    // - RequiredLevel   (int)         玩家等级需求

    public override BaseUpgrade GetLastLevel();      // 最高等级
    public override BaseUpgrade GetLastBuyedLevel(); // 最高已购等级
}
```

### 3.3 KillStreakSlot（装备槽）

```csharp
// KillStreakSlot.cs
public class KillStreakSlot : BaseSlot<KillStreak>
{
    public enum Slot
    {
        None   = 0,
        First  = 1,
        Second = 2,
        Last   = 3   // 第3个槽位
    }

    public KillStreak this[Slot slot] => ...; // 获取槽位上装备的KS

    public bool IsSet(KillStreak ks, Slot slot); // 某KS是否在某槽
    public bool IsSet(Slot slot);                // 某槽是否已装备
    public Slot IsSet(KillStreak ks);            // 某KS在哪个槽
}
```

### 3.4 LocalKillStreakSlot（本地玩家装备槽，带服务器同步）

```csharp
// LocalKillStreakSlot.cs
public class LocalKillStreakSlot : KillStreakSlot
{
    public readonly int UserKitSlot; // 所属套装槽位编号

    // 事件
    public event SetKillStreak   OnSet;
    public event SetKillStreakGroup OnSetGroup;
    public event UnsetKillStreak OnUnset;
    public event OnKillStreakChangesResponseEventHandler OnKillStreakChangedResponse;

    // 操作
    public void Set(KillStreak ks, Slot slot, bool delayedRequest = false);
    public void Set(KillStreak ks_1, Slot slot_1, KillStreak ks_2, Slot slot_2,
                    bool delayedRequest = false); // 批量装备（减少请求次数）
    public void Unset(KillStreak ks, Slot slot);

    // 内部：装备变更后向服务器发送 Ajax 请求
    private void SendChangesToAjax(bool delayedRequest = false);
    private void OnKillStreakChangesCompleted(object result, AjaxRequest request);
}
```

---

## 4. 战斗状态机

每个 `BattleKillstreakBase` 实例维护一个 **7态状态机**：

```
        玩家按技能键
            │
            ▼
    ┌──────────────┐
    │   Launching  │  ← 发起请求（向服务器Launch请求）
    │   (发起中)   │
    └──────┬───────┘
           │ 服务器确认/动画完成
           ▼
    ┌──────────────┐
    │   Launched   │  ← 已发起，等待激活
    │   (已发起)   │
    └──────┬───────┘
           │ 服务器激活确认
           ▼
    ┌──────────────┐
    │  Activating  │  ← 激活过渡（部分类型需要）
    │   (激活中)   │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │  Activated   │  ← 技能生效中（计时/战斗中）
    │   (激活态)   │
    └──────┬───────┘
           │ 弹药耗尽/持续时间到/手动结束
           ▼
    ┌──────────────┐
    │  Finishing   │  ← 结束过渡
    │   (结束中)   │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │   Finished   │  ← 进入冷却，重置计数器
    │   (已结束)   │
    └──────────────┘
           │ 冷却完毕
           ▼
         None（可再次发起）
```

```csharp
public enum KillstreakState
{
    None       = 0, // 初始/冷却完毕，可使用
    Launching  = 1, // 正在向服务器请求发起
    Launched   = 2, // 服务器确认，本地已发起
    Activating = 3, // 激活过渡
    Activated  = 4, // 效果进行中
    Finishing  = 5, // 正在结束
    Finished   = 6, // 结束完成，冷却开始
}
```

**状态转换触发源：**

| 转换 | 触发源 | 网络事件 |
|------|-------|---------|
| None → Launching | 玩家按键 | `BattleKillstreaksController.Launch()` |
| Launching → Launched | 服务器 Response | `LaunchKillstreakResponse` |
| Launched → Activated | 服务器 Event | `ActivateKillstreakEvent` |
| Activated → Finished | 时间到/弹药耗尽 | `FinishKillstreakEvent` |
| Any → None(cooldown) | 冷却计时器 | `Update()` 循环检测 |

---

## 5. 战斗运行时：三大实现分支

`BattleKillstreakBase` 是所有战斗态 KillStreak 的抽象基类，下分三个主要实现分支：

```
BattleKillstreakBase（抽象）
├── BattleKillstreakRadar           ── 定时器型（Radar / AntiRadar）
│   └── BattleKillstreakAntiRadar
├── BattleKillstreakBattleUnitBase  ── 自主 AI 单元型
│   ├── BattleKillstreakDog           （军犬）
│   ├── BattleKillstreakTurret        （炮塔）
│   ├── BattleKillstreakCopter        （直升机）
│   ├── BattleKillstreakHelicopter    （直升机变种）
│   ├── BattleKillstreakRadioCar      （无线电通讯车）
│   └── BattleKillstreakRocketArtillery（炮火袭击）
├── BattleKillstreakWeaponBase      ── 武器替换型（玩家手持）
│   ├── BattleKillstreakFlamer        （火焰喷射器）
│   ├── BattleKillstreakRocketLauncher（火箭筒）
│   ├── BattleKillstreakHandGunShield （盾牌+手枪）
│   └── BattleKillstreakWeaponPursuer （武器追踪者）
└── BattleKillstreakSpy             ── 特殊型（隐身间谍）
```

### 5.1 BattleKillstreakBase（抽象基类）

```csharp
// Battle/Killstreaks/BattleKillstreakBase.cs
public abstract class BattleKillstreakBase
{
    // 不可变配置（构造时从 KitKillstreaksDC 读取）
    public readonly int            ID;
    public readonly KillStreakType Type;
    public readonly int            CooldownMSec;        // 冷却毫秒
    public readonly bool           InterruptOnOwnerDeath;// 死亡中断
    public readonly string         SystemName;           // 如 "FL_Flamethrower"
    public readonly int            SystemNameHash;       // GetHashCode()，用于并发限制
    public readonly string         ModelName;            // 3D 资产名
    public readonly bool           IsTempKillstreak;     // 临时赠送型

    protected readonly IBattlePlayer _owner;             // 技能拥有者

    // 运行时状态（只读）
    public short              Cost { get; }             // 激活积分门槛
    public int                CurrentTimeMSec => ...;   // 当前战斗时间
    public BaseItemViewController KillstreakView { get; }// 关联的 View

    // 必须实现的状态查询
    public abstract bool           HasLaunchAnimation { get; }
    public abstract KillstreakState CurrentState       { get; }
    public abstract float           CooldownProgress   { get; } // 0~1
    public abstract bool            IsInCooldown        { get; }
    public abstract int?            TimeToFinish        { get; } // 剩余时间ms

    // 生命周期方法（抽象，子类必须实现）
    public abstract void ResetForReusing();
    public abstract void Launch(KillstreakLaunching onLaunchedHandler);
    public abstract void Activate(int? timeOfFinishingMSec);
    public abstract void Finish();
    public abstract bool Update();
    public abstract void CleanOnBattleFinish();
    public abstract void CleanOnPlayerLeave();

    // 可选检查（基类返回 false）
    public virtual bool CanLaunch()   => false;
    public virtual bool CanActivate() => false;
    public virtual bool CanFinish()   => false;
}
```

**构造参数：**
```csharp
BattleKillstreakBase(KitKillstreaksDC.KillstreakDC ksData, IBattlePlayer owner)
```

### 5.2 分支一：定时器型（BattleKillstreakRadar）

**代表：** Radar（雷达）、AntiRadar（雷达干扰）

```csharp
public class BattleKillstreakRadar : BattleKillstreakBase
{
    public readonly int DurationMSec; // 持续时间（毫秒）

    private KillstreakState _currentState;
    private int _timeToFinish;        // 到期时间戳（毫秒）
    private int _timeToFinishCooldown;// 冷却到期时间戳

    public override bool HasLaunchAnimation => false; // 无发起动画
}
```

**工作流程：**
1. `Launch(handler)` → 直接完成，调用 `handler(this, true)`
2. `Activate(timeOfFinishingMSec)` → 记录 `_timeToFinish`，切换到 `Activated`
3. `Update()` → 检查当前时间 vs `_timeToFinish`，到期自动调用 `Finish()`
4. `Finish()` → 切换到 `Finished`，开始记录 `_timeToFinishCooldown`
5. `Update()` → 冷却结束，切换回 `None`

**AntiRadar 区别：** 继承 `BattleKillstreakRadar`，激活时通知 Mini-map 系统隐藏敌方。

### 5.3 分支二：自主 AI 单元型（BattleKillstreakBattleUnitBase）

**代表：** Dog（军犬）、Turret（炮塔）、Copter（直升机）、RadioCar（炸弹车）、RocketArtillery（炮击）

```csharp
public abstract class BattleKillstreakBattleUnitBase : BattleKillstreakBase
{
    public readonly int DurationMSec;
    public BattleUnitType BattleUnitType { get; protected set; }

    private KillstreakState _currentState;
    private int _timeToFinish;
    private int _timeToFinishCooldown;
    public float TimeToFinishProgress => ...; // 0~1 剩余时间进度
}
```

**工作流程：**
1. `Launch(handler)` → 在场景中 **生成 BattleUnit**（NavMesh 寻路 AI），调用 `handler`
2. `Activate(timeOfFinishingMSec)` → BattleUnit 开始攻击行为
3. `Update()` → 驱动 BattleUnit，检查持续时间
4. `Finish()` → 销毁 BattleUnit，进入冷却

**子类特化差异：**

| 子类 | 特化点 |
|-----|-------|
| `BattleKillstreakDog` | 需要 `MapWaypointsManager` + `GameModeType`；使用 `DogBattleUnitSpawnPoint` 找最近出生点；`MAX_NAV_MESH_SAMPLE_DIST = 10f` |
| `BattleKillstreakTurret` | 玩家手动放置（有 `_onLaunchedHandler` 延迟）；`HasLaunchAnimation = false` |
| `BattleKillstreakCopter` | 空中单元；`MAX_NAV_MESH_SAMPLE_DIST = 10f`（NavMesh 采样） |
| `BattleKillstreakHelicopter` | Copter 变种（可能是 HelicopterTurret 类型） |
| `BattleKillstreakRadioCar` | 地面自走炸弹；`MAX_NAV_MESH_SAMPLE_DIST = 30f`（地图范围大） |
| `BattleKillstreakRocketArtillery` | 需要 `RocketArtilleryWaypoints`（沿航点轰炸）；`MapWaypointsManager` 依赖 |

### 5.4 分支三：武器替换型（BattleKillstreakWeaponBase）

**代表：** Flamer（火焰喷射器）、RocketLauncher（火箭筒）、HandGunShield（盾牌+手枪）

```csharp
public abstract class BattleKillstreakWeaponBase : BattleKillstreakBase
{
    private KillstreakState _currentState;
    private int _activationTime;
    private int _timeToFinishCooldown;

    public BattleWeapon KillstreakWeapon { get; } // 激活期间使用的武器
    public BattleWeapon StoredWpn        { get; } // 激活前存储的原武器
}
```

**工作流程：**
1. `Launch(handler)` → 调用 `handler(this, true)`（通常无发起动画）
2. `Activate(timeOfFinishingMSec)` →
   - 记录 `StoredWpn`（保存当前武器）
   - 替换为 `KillstreakWeapon`（武器替换）
   - 弹药数量来自 `KitKillstreaksDC.WeaponData`
3. `Update()` → 检查弹药，耗尽时 `Finish()`
4. `Finish()` →
   - 恢复 `StoredWpn`
   - 开始冷却

**武器替换的完整流程：**

```
Activate()
    │
    ├── 记录 StoredWpn = owner.CurrentWeapon
    ├── 创建/取出 KillstreakWeapon（从对象池或工厂）
    ├── 设置弹药：KillstreakWeapon.Ammo = ksData.WeaponData["ammo"]
    ├── owner.SetWeapon(KillstreakWeapon)
    └── 切换视图（首视角/第三视角 适配）

Finish()
    │
    ├── owner.SetWeapon(StoredWpn)
    ├── 回收 KillstreakWeapon 到对象池
    └── 开始冷却计时
```

**视角工厂分离原则：** 每种武器替换型 KS 都有三个视角实现类：
```
BattleKillstreakFlamer（基础逻辑）
├── FirstLookBattleKillstreakFlamer   （第一人称专属视图）
├── ThirdLookBattleKillstreakFlamer   （第三人称专属视图）
└── TopDownLocalPlayerBattleKillstreakFlamer（俯视角专属视图）
```

### 5.5 特殊型：BattleKillstreakSpy（间谍/隐身）

```csharp
public class BattleKillstreakSpy : BattleKillstreakBase, IDisposable
{
    private const string SPY_MATERIAL_NAME             = "Spy";
    private const string SPY_TOTAL_INVISIBLE_MATERIAL_NAME = "SpyTotalInvisible";

    public readonly int DurationMSec;
    private readonly IPlayersRelationshipsController _playersRelationshipsCtrl;

    // 异步材质加载（取消令牌防止竞态）
    private CancellationTokenSource _onLoadSkinSpyMaterialCancelToken;
    private CancellationTokenSource _onLoadWeaponSkinCancelToken;
    private ConcurrentDictionary<int, CancellationTokenSource> _onLoadWeaponPartMaterialCancelToken;
}
```

**特性：**
- 替换玩家全身材质为 "Spy"（半透明）
- 对敌方阵营：显示 "SpyTotalInvisible"（完全不可见）
- 异步加载材质，使用 CancellationToken 处理取消
- 武器材质也同步处理（Skin + WeaponPart 分层处理）
- 实现 `IDisposable` 确保取消令牌被释放

---

## 6. 管理器体系

### 6.1 BattleKillstreaksController（玩家级别控制器）

每个玩家拥有一个 Controller 实例，负责管理该玩家的所有 KillStreak：

```csharp
// Battle/Killstreaks/BattleKillstreaksController.cs
public abstract class BattleKillstreaksController
{
    // 并发控制 mask（静态）
    public static readonly byte[,] KILLSTREAK_USAGE_MASK;
    public const byte TEMP_KILLSTREAK_SLOT_INDEX = 255; // 临时KS专用槽

    // 数据
    protected readonly Dictionary<byte, BattleKillstreakBase> _curKitKillstreaks; // 槽index→KS
    private readonly Dictionary<int, BattleKillstreakBase>    _activeKillstreaks;  // ID→激活的KS
    private readonly Dictionary<int, BattleKillstreakBase>    _cache;              // 复用缓存

    private short _killstreakScores; // 当前积分
    public  short KillstreakScores { get; }

    // 关键属性
    public BattleKillstreakBase LastLaunchedKillstreak { get; }
    public List<BattleKillstreakBase> KillstreaksInLaunching { get; }
    public bool IsBusyByKillstreak => ...; // 是否正在使用武器替换型KS
    public bool IsSpyKillstreakActive { get; }

    // 事件
    public event KillstreakScoreChanging OnKillstreakScoreHasBeenChangedEvent;
}
```

**核心方法签名：**

```csharp
// 发起（玩家按键触发）
public bool Launch(byte ksSlotIndex, KillstreakLaunching onDoneHandler,
                   out BattleKillstreakBase launchedKS);

// 激活（服务器确认后触发）
public bool Activate(byte ksSlotIndex, int? timeOfFinishingMSec,
                     out BattleKillstreakBase activatedKS);

// 结束（自动或服务器通知）
public bool Finish(int ksID, out BattleKillstreakBase finishedKS);

// 积分管理
public void IncreaseKillstreakScore(short scoresToAdd);
public void ResetKillstreakScores();
public void AddRewardKillstreakScore(RewardData r);
public float GetKillstreakScoreProgress(BattleKillstreakBase killStreak); // 0~1

// 查询
public BattleKillstreakBase GetKillstreak(KillStreakType ksType);
public BattleKillstreakBase GetKillstreak(byte index);
public BattleKillstreakBase GetKillstreak(int ksID, bool tryGetFromActivated);
public BattleKillstreakWeaponBase GetWeaponKillstreak(KillStreakType ksType, bool tryGetFromActivated);

// 初始化
public void InitUserKillstreaksData(KitKillstreaksDC kssData, IBattlePlayer owner,
                                    MapWaypointsManager mapSplinesMngr, short? killstreakScores);

// 临时 KS（从战场拾取）
public void SetTempKillstreak(Dictionary<byte, object> tempKillstreakData,
                              MapWaypointsManager mapWaypointsMngr);

// 死亡处理
public virtual void ProcessOwnerDeath();
```

**并发约束（KILLSTREAK_USAGE_MASK）：**

二维 `byte` 数组，第一维为"当前激活的KS类型"，第二维为"想要发起的KS类型"，值为0表示可同时使用，非0表示互斥。具体值由游戏逻辑硬编码或配置表决定（AssetRipper 代码截断，无法直接读取）。

```csharp
private bool CanUseTogetherWith(KillStreakType activeKS, KillStreakType newKS)
{
    return KILLSTREAK_USAGE_MASK[(int)activeKS, (int)newKS] == 0;
}
```

### 6.2 BattleKillstreaksManager（全局游戏管理器）

```csharp
// Battle/Killstreaks/BattleKillstreaksManager.cs
public class BattleKillstreaksManager
{
    // 维护所有活跃 KS 的历史记录（用于断线重连/观战）
    private readonly Dictionary<byte, List<BattleKillstreakBase>> _killstreaksWereActiveOnGameState;

    public KillstreakLimitsController Limiter { get; } // 并发数量限制器

    // 本地玩家接口
    public bool TryLaunch(byte ksIndex);
    public void Launch(byte ksIndex);
    public void Activate(byte ksIndex, int? timeOfFinishingMSec);
    public bool TryFinish(byte ksIndex);
    public void Finish(int ksID);

    // 第三方玩家接口（观察其他玩家的 KS 效果）
    public void LaunchForThirdPerson(ThirdLookBattlePlayer p, byte ksIndex);
    public void ActivateForThirdPerson(ThirdLookBattlePlayer p, byte ksIndex, int? timeOfFinishingMSec);
    public bool FinishForThirdPerson(ThirdLookBattlePlayer p, int ksID);

    // 每帧驱动
    public void CustomUpdate();

    // 断线重连/观战恢复
    public void SetupPlayersActivatedKillstreaksData(Dictionary<byte, object> data,
                                                     IBattlePlayersList playerList);
    // 并发检查
    public bool IsKillstreakCountLimitReached(IBattlePlayer player, BattleKillstreakBase ks);
}
```

**AI 变体：** `BattleKillstreaksManagerAI : BattleKillstreaksManager`
- 支持 Bot（AI玩家）使用 KillStreak
- 额外方法：`SendLaunchKillstreakByBotRequest`, `LaunchByBot`, `ActivateByBot`, `FinishByBot`

### 6.3 BattleKillstreaksFactory（对象工厂）

```csharp
// Battle/Killstreaks/BattleKillstreaksFactory.cs
public class BattleKillstreaksFactory
{
    private BaseWeaponKillstreaksFactory _weaponKillstreaksFactory;

    public BattleKillstreakBase CreateKillStreak(
        KitKillstreaksDC.KillstreakDC ksData,
        IBattlePlayer player,
        IBattlePlayersList playersCtrl,
        IBattleUnitsList battleUnitsList,
        IPlayersRelationshipsController playersRelationshipsCtrl,
        IBattlePlayersLayersManager battlePlayerLayersMngr,
        MapWaypointsManager mapWaypointsMngr,
        GameModeType gameModeType,
        ReadOnlyCollection<float> wpnSpreadRandomArray
    );
}
```

**工厂内部逻辑（推断）：**
```csharp
// switch on ksData.Type
switch (ksData.Type)
{
    case KillStreakType.Radar:         return new BattleKillstreakRadar(ksData, player);
    case KillStreakType.AntiRadar:     return new BattleKillstreakAntiRadar(ksData, player);
    case KillStreakType.Dog:           return new BattleKillstreakDog(ksData, player, mapWaypointsMngr, gameModeType);
    case KillStreakType.Turret:        return new BattleKillstreakTurret(ksData, player);
    case KillStreakType.Copter:        return new BattleKillstreakCopter(ksData, player);
    case KillStreakType.RCBombCar:     return new BattleKillstreakRadioCar(ksData, player);
    case KillStreakType.RocketArtillery: return new BattleKillstreakRocketArtillery(ksData, player, waypoints);
    case KillStreakType.Flamer:
    case KillStreakType.Juggernaut:
    case KillStreakType.Shield:
    case KillStreakType.RocketLauncher:
        return _weaponKillstreaksFactory.CreateWeaponKillstreak(ksData, player, ...);
    case KillStreakType.Spy:           return new BattleKillstreakSpy(ksData, player, playersRelCtrl);
}
```

**武器替换型视角工厂（BaseWeaponKillstreaksFactory）：**
```csharp
// 抽象基类，三个视角分别实现
public abstract class BaseWeaponKillstreaksFactory
{
    public abstract BattleKillstreakWeaponBase CreateWeaponKillstreak(
        KitKillstreaksDC.KillstreakDC ksData,
        IBattlePlayer player, ...
    );
}

// 三个具体实现
public class FirstLookWeaponKillstreaksFactory : BaseWeaponKillstreaksFactory { ... }
public class ThirdLookWeaponKillstreaksFactory : BaseWeaponKillstreaksFactory { ... }
public class TopDownLocalPlayerWeaponKillstreaksFactory : BaseWeaponKillstreaksFactory { ... }
```

### 6.4 KillstreakLimitsController（并发数量限制器）

```csharp
// Battle/Killstreaks/KillstreakLimitsController.cs
public sealed class KillstreakLimitsController
{
    // 结构：Team → (KS系统名Hash → 当前激活数量)
    private readonly Dictionary<BattleTeam, Limit> _limits;
    private readonly GameModeType _gmt;

    public bool CanLaunchKillstreakDueToLimit(BattleTeam team, int ksSystemNameHashCode);
    public void Inc(BattleTeam team, int ksSystemNameHashCode); // 激活时+1
    public void Dec(BattleTeam team, int ksSystemNameHashCode); // 结束时-1
}
```

**配置来源：** `KillstreaksLimitForTeamConfig`
```csharp
// GameLogic/Config/KillstreaksLimitForTeamConfig.cs
public sealed class KillstreaksLimitForTeamConfig
{
    // GameModeType → (KS系统名Hash → 每队最大并发数)
    private readonly Dictionary<GameModeType, Dictionary<int, byte>> _data;

    public byte Get(GameModeType gmt, int ksSystemNameHash);
}
```

---

## 7. 网络协议

### 7.1 协议流程总览

```
客户端                          服务器（Photon）                    其他客户端
  │                                  │                                  │
  ├── LaunchKillstreak Request ──────►│                                  │
  │                                  ├── 验证积分/状态/限制              │
  │◄── LaunchKillstreakResponse ─────┤                                  │
  │     IsSuccessful, SlotIndex      │                                  │
  │                                  ├── LaunchKillstreakEvent ─────────►│
  │                                  │   KillstreakSlotIndex, ActorID   │
  │                                  │                                  │
  ├── ActivateKillstreak Request ────►│                                  │
  │   SlotIndex, Position?, Rotation?│                                  │
  │◄── ActivateKillstreakResponse ───┤                                  │
  │   TimeOfFinishing, Rewards[]     │                                  │
  │                                  ├── ActivateKillstreakEvent ───────►│
  │                                  │   SlotIndex, TimeOfFinishingMSec │
  │                                  │                                  │
  │    （效果进行中...）              │                                  │
  │                                  │                                  │
  ├── FinishKillstreak Request ──────►│（或服务器主动推送）              │
  │                                  ├── FinishKillstreakEvent ─────────►│
  │◄── FinishKillstreakResponse ─────┤   KillstreakID, ActorID          │
```

### 7.2 事件结构

```csharp
// 发起事件（广播给所有玩家，用于播放他人动画）
class LaunchKillstreakEvent
{
    public readonly byte KillstreakSlotIndex; // 0/1/2
    public readonly byte ActorID;             // Photon Actor ID
}

// 激活事件（广播激活时间，用于同步）
class ActivateKillstreakEvent
{
    public readonly byte  KillstreakSlotIndex;
    public readonly int?  TimeOfFinishingMSec; // null = 无时限型
    public readonly byte  ActorID;
}

// 结束事件（广播，用于清理他人视图）
class FinishKillstreakEvent
{
    public readonly int  KillstreakID; // 唯一实例ID（不是配置ID）
    public readonly byte ActorID;
}

// 积分增加事件
class IncreaseKillstreakScoresEvent
{
    public readonly short ScoresToAdd;
    public readonly byte? ActorID; // null = 广播给所有人
}

// 临时 KS 赋予事件（从地图拾取）
class SetTempKillstreakEvent
{
    public readonly byte ActorID;
    public readonly Dictionary<byte, object> TempKillstreakData; // 序列化的 KillstreakDC
}
```

### 7.3 Response 结构

```csharp
// 发起响应
class LaunchKillstreakResponse
{
    public readonly bool   IsSuccessful;
    public readonly short  ResponseReturnCode;
    public readonly string DebugMessage;
    public byte  KillstreakSlotIndex { get; }
    public byte? BotActorID          { get; } // Bot发起时有效
}

// 激活响应
class ActivateKillstreakResponse
{
    public readonly bool IsSuccessful;
    public byte   KillstreakSlotIndex { get; }
    public int?   TimeOfFinishing     { get; } // 到期时间戳（毫秒）
    public byte?  BotActorID          { get; }
    public byte[] Rewards             { get; } // 激活奖励
}
```

---

## 8. 视图系统

### 8.1 视图层次

```
BattleKillstreaksController（抽象）
├── FirstLookBattleKillstreaksController   （第一人称：本地玩家默认）
├── ThirdLookBattleKillstreaksController   （第三人称：观战/他人）
└── TopDownLocalPlayerBattleKillstreaksController（俯视角模式）
```

每个 Controller 实现 `InitKillstreakView(BattleKillstreakBase ks)` 和 `GetWeaponKillstreaksFactory()` 的视角专属版本。

### 8.2 View 加载机制

```csharp
// FirstLookBattleKillstreaksController
private BaseItemViewController LoadOrGetView(string systemName)
{
    // 1. 尝试从 _cache 取出已加载的 View
    // 2. 未命中 → 从 Resources 异步加载预制体
    // 3. 加载完成后调用 KillstreakView.SetView(view)
}

// 死亡时的回收
private void ResetActivatedKillstreaksForReusing() { ... }
private void AddToPool() { ... }
```

### 8.3 LoadedKillstreakInfo

```csharp
public struct LoadedKillstreakInfo
{
    public GameObject           KillstreakObject;   // 场景中的 GameObject
    public readonly BaseItemViewController KillstreakView;
    public readonly bool        IsKillstreakFromPool; // 是否从对象池取出
}
```

### 8.4 UI 组件

| 组件 | 功能 |
|------|------|
| `HUDKillStreakActivationPanelV2` | 战斗 HUD 中的 KS 激活面板（显示3个槽位进度条） |
| `HUDKillStreakAction` | 单个 KS 的 HUD 动作按钮 |
| `HUDPlaceKillStreakBtnV2` | 炮塔等需要手动放置的 KS 放置按钮 |
| `KillStreaksWindowV2` | 装备界面 KS 选择窗口 |
| `KillStreakItemViewV2` | 装备界面单个 KS 卡片 |
| `KillstreakUpgradePanelV2` | 升级面板 |
| `KillStreakView3DContainer` | 3D 模型预览容器 |

---

## 9. 积分追踪与击杀积累

### 9.1 积分来源

积分（KillstreakScores）通过以下途径累加：

```csharp
// 来源 1：击杀奖励（RewardData）
public void AddRewardKillstreakScore(RewardData r);

// 来源 2：服务器直接推送（IncreaseKillstreakScoresEvent）
public void IncreaseKillstreakScore(short scoresToAdd);
```

**积分值对应：**
| 事件 | 典型积分 |
|------|---------|
| 普通击杀 (Kill) | +1 |
| 助攻 (Assist) | +0 或 +0.5 |
| 爆头 (Headshot) | +1 |
| 多杀加成 | 额外 +X |

### 9.2 积分消耗

```csharp
// 使用 KillStreak 时扣除
// Cost 来自服务器（KitKillstreaksDC.KillstreakDC.Cost = CRCD13.c）
// 例：Radar.c = 15，激活时扣除 15 分
```

### 9.3 进度查询

```csharp
// 返回 0~1 进度（用于 HUD 进度条显示）
public float GetKillstreakScoreProgress(BattleKillstreakBase killStreak)
{
    return Mathf.Clamp01((float)_killstreakScores / killStreak.Cost);
}
```

### 9.4 死亡重置

当玩家死亡时，`KillstreakScores` 被重置为 0（`ResetKillstreakScores()`）。但如果有 KillStreak 已处于 Activated 状态，根据 `InterruptOnOwnerDeath` 决定是否中断。

---

## 10. 装备/背包系统

### 10.1 系统关系图

```
玩家背包（服务器同步）
    │
    ▼
JSON_InventoryKillStreak[]
    │ id, u_id, u_eD
    ▼
KillStreak（内存中的全量配置对象）
    │ CRCD13 数据 + CRCD34 升级链
    ▼
LocalKillStreakSlot（本地玩家装备槽）
    │ Slot.First / Second / Last
    ├── Set(ks, slot) ──→ Ajax Request
    ├── Unset(ks, slot) ─→ Ajax Request
    └── 事件：OnSet / OnUnset / OnSetGroup

                    ▼ 战斗开始
KitKillstreaksDC（Photon 数据合约）
    │ 由 Matchmaker 从服务器读取装备配置打包
    ▼
BattleKillstreaksController.InitUserKillstreaksData()
    │ 为每个槽位创建 BattleKillstreakBase 实例
    ▼
BattleKillstreakBase（战斗运行时）
```

### 10.2 WearRefKillstreak（装扮关联 KS）

某些装扮（Wear）自带 KillStreak 能力：

```csharp
public class WearRefKillstreakManager
{
    // wearId → (KillstreakFirst, KillstreakSecond)
    private Dictionary<int, WearKillstreak> _wearRefKillstreak;

    public class WearKillstreak
    {
        public readonly int WearId;
        public readonly int KillstreakFirstId;
        public readonly int KillstreakSecondId;
        public readonly KillStreak KillstreakFirst;
        public readonly KillStreak KillstreakSecond;
    }

    // 根据装扮 ID 和槽位获取关联 KS
    public KillStreak GetKillstreak(int wearId, KillStreakSlot.Slot slot);
}
```

### 10.3 BankKillstreak（商店购买项）

```csharp
public class BankKillstreak
{
    public readonly int    ID;
    public readonly int    Order;
    public bool            Enabled     { get; }
    public ShopCost        Cost        { get; }  // 购买价格
    public ShopCost        PrevCost    { get; }  // 折前价格
    public int             Discount    { get; }  // 折扣百分比
    public PopularityType  Popularity  { get; }  // 热度标签
    public string          NameSprite  => ...;   // UI 图标名
    public KillstreakIAP   IAP         { get; }  // 内购关联
}
```

---

## 11. AI 集成

### 11.1 AI 使用条件

```csharp
// Penta/Battle/AI/Behavior/Conditions/Common/CanUseKillstreakCondition.cs
// 继承自 NodeCanvas Condition 节点
// 检查：
// - KillstreakScores >= Cost (积分足够)
// - !IsInCooldown (不在冷却)
// - !IsBusyByKillstreak (未被武器KS占用)
// - 返回 true 时 AI 行为树允许执行 UseKillstreakAction
```

### 11.2 AI 使用动作

```csharp
// Penta/Battle/AI/Behavior/Actions/Common/UseKillstreakAction.cs
[Category("Custom/Actions")]
[Name("UseKillstreak")]
public class UseKillstreakAction : BaseAction
{
    // OnEnter: 选择最高积分满足的 KS 槽位
    // OnProcess: 调用 BattleKillstreaksManagerAI.TryLaunchByBot()
    // OnExit: 清理临时状态
}
```

### 11.3 AI 管理器（BattleKillstreaksManagerAI）

```csharp
public class BattleKillstreaksManagerAI : BattleKillstreaksManager
{
    public bool SendLaunchKillstreakByBotRequest(ThirdLookBattlePlayer bot, byte ksIndex);
    public bool TryLaunchByBot(ThirdLookBattlePlayer bot, byte ksIndex);
    public void LaunchByBot(ThirdLookBattlePlayer bot, byte ksIndex);
    public void ActivateByBot(ThirdLookBattlePlayer bot, byte ksIndex, int? timeOfFinishingMSec);
    public bool TryFinishByBot(ThirdLookBattlePlayer bot, byte ksIndex);
    public void FinishByBot(ThirdLookBattlePlayer bot, int ksID);
}
```

---

## 12. 单机迁移实现方案

本章提供将 KillStreak 系统迁移到**纯单机项目**的完整实现指南。去除网络层，保留完整的战斗逻辑。

### 12.1 迁移原则

| 原系统组件 | 迁移策略 |
|-----------|---------|
| Photon 网络事件 | 本地直接调用替代 |
| 服务器 Launch/Activate 验证 | 本地规则校验器替代 |
| Ajax 装备同步 | 本地 JSON 文件持久化 |
| `KitKillstreaksDC`（网络数据合约） | 本地 `LocalKitKillstreaksDC` 替代 |
| `IBattlePlayersList` | 本地玩家列表实现 |

### 12.2 数据加载（离线版）

```csharp
// 1. 从本地 JSON 加载配置
public class KillStreakDatabase
{
    private static Dictionary<int, KillStreakConfig> _configs;
    private static Dictionary<int, KillStreakUpgradeConfig> _upgrades;

    // 对应 CRCD13.json
    public struct KillStreakConfig
    {
        public int   ID;
        public KillStreakType Type;
        public string SystemName;
        public int   CostExecution;  // 激活所需积分
        public int   CooldownSec;    // 冷却秒数
        public float Range, Damage, Mobility, Ammo;
        public float Health, Armor, Strength, DurationSec;
    }

    // 对应 CRCD34.json
    public struct KillStreakUpgradeConfig
    {
        public int UpgradeChainID;
        public int Level;
        public int SourceID;     // sid
        public int ResultID;     // rid
        public int RequiredPlayerLevel;
        public int CostGold;     // tPv
        public int CostGems;     // tPr
    }

    public static void Load(TextAsset crcd13, TextAsset crcd34)
    {
        // 解析 JSON，建立 ID→Config 字典
    }

    public static KillStreakConfig Get(int id) => _configs[id];
}
```

### 12.3 本地 KillStreakSlot（无网络）

```csharp
public class LocalOfflineKillStreakSlot
{
    // 三个槽位存储 KS 配置 ID
    private int[] _slots = new int[3] { 0, 0, 0 };

    public void Equip(int ksID, int slotIndex)
    {
        _slots[slotIndex] = ksID;
        SaveToLocal();
    }

    public void Unequip(int slotIndex)
    {
        _slots[slotIndex] = 0;
        SaveToLocal();
    }

    // 持久化到本地
    private void SaveToLocal()
    {
        string json = JsonUtility.ToJson(new SlotData { slots = _slots });
        File.WriteAllText(Application.persistentDataPath + "/ks_slots.json", json);
    }
}
```

### 12.4 战斗管理器（本地版）

```csharp
public class LocalKillStreakManager
{
    // 玩家持有的 KS 战斗实例列表
    private Dictionary<int, KillStreakRuntimeInstance> _instances = new();
    private short _score = 0;

    // 积分累加（原 IncreaseKillstreakScoresEvent）
    public void AddScore(short amount)
    {
        _score += amount;
        OnScoreChanged?.Invoke(_score);
    }

    // 玩家按键触发
    public bool TryUse(int slotIndex)
    {
        var inst = _instances[slotIndex];
        if (_score < inst.Config.CostExecution) return false;
        if (inst.IsInCooldown) return false;
        if (!CanUseTogether(inst)) return false;

        _score -= (short)inst.Config.CostExecution;
        inst.Activate();
        return true;
    }

    // 每帧驱动（替代 CustomUpdate）
    public void Update()
    {
        foreach (var inst in _instances.Values)
            inst.Update(Time.time);
    }

    // 玩家死亡
    public void OnPlayerDeath()
    {
        _score = 0;
        foreach (var inst in _instances.Values)
        {
            if (inst.Config.InterruptOnOwnerDeath && inst.IsActive)
                inst.Finish();
        }
    }
}
```

### 12.5 运行时实例（简化版状态机）

```csharp
public class KillStreakRuntimeInstance
{
    public readonly KillStreakDatabase.KillStreakConfig Config;
    public KillstreakState State { get; private set; }

    private float _activatedAt;
    private float _finishedAt;

    public bool IsInCooldown => State == KillstreakState.Finished
                                && Time.time < _finishedAt + Config.CooldownSec;
    public bool IsActive     => State == KillstreakState.Activated;
    public float CooldownProgress => IsInCooldown
        ? 1f - ((_finishedAt + Config.CooldownSec - Time.time) / Config.CooldownSec)
        : 1f;

    public void Activate()
    {
        State = KillstreakState.Activated;
        _activatedAt = Time.time;
        OnActivated?.Invoke(this);
        // 根据类型执行具体效果
        ExecuteEffect();
    }

    public void Update(float now)
    {
        if (State != KillstreakState.Activated) return;
        bool expired = Config.DurationSec > 0 && now > _activatedAt + Config.DurationSec;
        if (expired) Finish();
    }

    public void Finish()
    {
        State = KillstreakState.Finished;
        _finishedAt = Time.time;
        OnFinished?.Invoke(this);
        RollbackEffect();
    }

    // 根据 Type 派发效果
    private void ExecuteEffect()
    {
        switch (Config.Type)
        {
            case KillStreakType.Radar:
                RadarSystem.Instance.Activate(Config.DurationSec);
                break;
            case KillStreakType.AntiRadar:
                RadarSystem.Instance.JamEnemyRadar(Config.DurationSec);
                break;
            case KillStreakType.Flamer:
            case KillStreakType.Juggernaut:
            case KillStreakType.RocketLauncher:
            case KillStreakType.Shield:
                WeaponSwapper.Instance.SwapTo(Config.Type, Config.Ammo);
                break;
            case KillStreakType.Dog:
            case KillStreakType.Turret:
            case KillStreakType.Copter:
            case KillStreakType.RCBombCar:
            case KillStreakType.RocketArtillery:
                BattleUnitSpawner.Instance.Spawn(Config.Type, Config);
                break;
        }
    }

    public event Action<KillStreakRuntimeInstance> OnActivated;
    public event Action<KillStreakRuntimeInstance> OnFinished;
}
```

### 12.6 并发限制（本地版）

```csharp
// 硬编码互斥规则（原 KILLSTREAK_USAGE_MASK）
public static class KillStreakCompatibility
{
    // 武器替换型不能同时使用
    private static readonly HashSet<KillStreakType> WeaponTypes = new()
    {
        KillStreakType.Flamer,
        KillStreakType.Juggernaut,
        KillStreakType.RocketLauncher,
        KillStreakType.Shield,
    };

    public static bool CanUseTogether(KillStreakType active, KillStreakType newKS)
    {
        // 武器类互斥
        if (WeaponTypes.Contains(active) && WeaponTypes.Contains(newKS))
            return false;
        // Spy 与武器类互斥
        if (active == KillStreakType.Spy && WeaponTypes.Contains(newKS))
            return false;
        return true;
    }
}
```

### 12.7 AI Bot 使用 KS（本地版）

```csharp
public class BotKillStreakBehavior : MonoBehaviour
{
    private LocalKillStreakManager _manager;
    private float _useInterval = 3f; // 每隔3秒尝试使用

    private void Update()
    {
        _useInterval -= Time.deltaTime;
        if (_useInterval > 0) return;
        _useInterval = Random.Range(2f, 5f);

        // 选择第一个可用的 KS 槽位
        for (int i = 0; i < 3; i++)
        {
            if (_manager.TryUse(i))
                break;
        }
    }
}
```

### 12.8 迁移工作量估算

| 模块 | 工作量 | 难点 |
|------|-------|------|
| 数据加载（CRCD13/34解析） | 0.5天 | 无 |
| KillStreakRuntimeInstance（状态机） | 1天 | 冷却/持续时间计时 |
| 积分追踪 | 0.5天 | 无 |
| 雷达/AntiRadar | 1天 | 需 MinimapSystem |
| 武器替换型（Flamer/Jugg/Shield/RL） | 2天 | 武器切换 + 视图适配 |
| AI 自主单元（Dog/Turret/Copter/Car） | 3天 | NavMesh + BattleUnit AI |
| 炮火袭击（RocketArtillery） | 1天 | 航点系统 |
| 间谍（Spy） | 2天 | 材质异步加载 + 分队可见性 |
| HUD UI | 1天 | 进度条 + 按钮状态 |
| 装备界面 UI | 2天 | 列表 + 升级面板 |
| 本地持久化 | 0.5天 | JSON 读写 |
| **合计** | **~14天** | 最难：AI单元 + 间谍材质 |

---

## 附录 A：完整类继承图

```
CCItem（基类）
└── KillStreak

BaseUpgrade
└── KillStreakUpgrade

BaseSlot<KillStreak>
└── KillStreakSlot
    └── LocalKillStreakSlot

BattleKillstreakBase（抽象）
├── BattleKillstreakRadar
│   └── BattleKillstreakAntiRadar
├── BattleKillstreakBattleUnitBase（抽象）
│   ├── BattleKillstreakDog
│   ├── BattleKillstreakTurret
│   ├── BattleKillstreakCopter
│   ├── BattleKillstreakHelicopter
│   ├── BattleKillstreakRadioCar
│   └── BattleKillstreakRocketArtillery
├── BattleKillstreakWeaponBase（抽象）
│   ├── BattleKillstreakFlamer
│   ├── BattleKillstreakRocketLauncher
│   ├── BattleKillstreakHandGunShield
│   └── BattleKillstreakWeaponPursuer
└── BattleKillstreakSpy（独立）

BattleKillstreaksController（抽象）
├── FirstLookBattleKillstreaksController
├── ThirdLookBattleKillstreaksController
└── TopDownLocalPlayerBattleKillstreaksController

BattleKillstreaksManager
└── BattleKillstreaksManagerAI
```

## 附录 B：关键接口

```csharp
// 战斗玩家接口（KS 需要这些接口）
interface IBattlePlayer
{
    BattleTeam Team { get; }
    bool       IsDead { get; }
    BattleWeapon CurrentWeapon { get; }
    void SetWeapon(BattleWeapon weapon);
    // ... 位置、旋转、LayerManager 等
}

// 关系判断接口（Spy/AntiRadar 需要）
interface IPlayersRelationshipsController
{
    bool IsEnemy(IBattlePlayer a, IBattlePlayer b);
    bool IsAlly(IBattlePlayer a, IBattlePlayer b);
}
```

## 附录 C：sn ↔ KillStreakType 映射表

| sn（SystemName） | KillStreakType | 中文名 | 实现类 |
|----------------|--------------|-------|-------|
| Radar | Radar=2 | 雷达 | BattleKillstreakRadar |
| AntiRadar | AntiRadar=3 | 雷达干扰 | BattleKillstreakAntiRadar |
| RL_RPG7 | RocketLauncher=1 | 火箭筒 | BattleKillstreakRocketLauncher |
| HGWS_Berreta9MWS | Shield=7 | 护盾+手枪 | BattleKillstreakHandGunShield |
| TR_GatlingTurret | Turret=4 | 炮塔 | BattleKillstreakTurret |
| RadioCar | RCBombCar=5 | 炸弹车 | BattleKillstreakRadioCar |
| GG_M134 | Juggernaut=8 | 主宰者 | FirstLookBattleKillstreakJuggernaut |
| FL_Flamethrower | Flamer=9 | 火焰喷射器 | BattleKillstreakFlamer |
| Dog | Dog=10 | 军犬 | BattleKillstreakDog |
| RocketArtillery | RocketArtillery=13 | 炮火袭击 | BattleKillstreakRocketArtillery |
| Copter | Copter=14 | 直升机 | BattleKillstreakCopter |
| CopterFlamer | Copter=14 | 火焰直升机 | BattleKillstreakCopter（变种） |

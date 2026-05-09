# Ability 技能系统 — 深度解析与实现规范

> 逆向分析对象：Modern Ops: Gun Shooting Games v9.79（Unity 6000.0.69f1）  
> 分析日期：2026-05-09 | 适用于：系统迁移、独立实现参考

---

## 目录

1. [系统架构总览](#1-系统架构总览)
2. [数据合约层（Data Contract）](#2-数据合约层)
3. [内存数据层（AbilityData）](#3-内存数据层)
4. [管理器层（AbilityManagerV2）](#4-管理器层)
5. [战斗应用层（效果类型完整枚举）](#5-战斗应用层)
6. [UI 展示层](#6-ui-展示层)
7. [解锁 / 购买 / 升级系统](#7-解锁--购买--升级系统)
8. [完整数据流](#8-完整数据流)
9. [迁移实现规范](#9-迁移实现规范)
10. [关键文件路径索引](#10-关键文件路径索引)

---

## 1. 系统架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                         Ability 系统架构                         │
├────────────────┬────────────────┬──────────────┬────────────────┤
│   数据合约层    │   内存数据层    │   管理器层    │   UI 展示层    │
│  JSON_ABILITIES │  AbilityData  │AbilityManager│AbilityWindowV2 │
│  JSON_Ability_  │  AbilityItem  │     V2       │AbilityPanelV2  │
│  Item           │  AbilityProp  │              │AbilityInfoPanel│
│  JSON_Property  │  AbilityType  │              │AbilityItemView │
│  JSON_ShopCost  │  AvailType    │              │AbilitySubType  │
└────────────────┴────────────────┴──────────────┴────────────────┘
         │                │               │               │
         ▼                ▼               ▼               ▼
   PlayerPrefs      运行时内存       Ajax/Photon        I2 Loc
   "CRCD4"         BaseData         HTTP API          LocalizePar
                   .Abilities       OP=121            amsManager
```

**核心设计原则：**
- 技能定义数据（150个技能 × 5级）由服务器通过 CRC 机制下发，本地 PlayerPrefs 缓存
- 玩家已购买技能（`IsBuyed`）由登录时服务器下发的 `BaseData.Abilities` 决定
- 战斗中技能效果**由服务器端预计算**后通过 SpawnDC/KitDC 下发，客户端直接使用结果值
- 所有货币扣除、技能解锁均通过 HTTP/Photon 请求向服务器确认

---

## 2. 数据合约层

### 2.1 顶层数据容器

**文件：** `GameLogic/Data/JSON_ABILITIES.cs`

```csharp
public class JSON_ABILITIES : JSON_DATA_ITEM   // JSON_DATA_ITEM 有 int crc 字段
{
    public JSON_Ability_Item[] d;   // 全部技能条目（723 条 = 150 技能 × 最多5级）
}
```

**存储/读取：**
- PlayerPrefs 键：`"CRCD4"`（前缀 `"CRCD"` + `CRCType.ABILITY = 4`）
- 反序列化：`JsonUtility.FromJson<JSON_ABILITIES>(PlayerPrefs.GetString("CRCD4"))`

---

### 2.2 单条技能 Entry

**文件：** `GameLogic/JSON/JSON_Ability_Item.cs`

```csharp
[Serializable]
public class JSON_Ability_Item
{
    public int    i;    // 技能基础 ID（同一技能所有等级共享相同 i）
    public int    l;    // 等级/Rank（1~5，每级独立一条 entry）
    public int    t;    // 技能大类型（AbilityType 枚举值 1~5）
    public int    o;    // 子类型/排序序号（order）
    public int    nl;   // 解锁所需玩家等级（0 = 无限制）
    public JSON_ShopCost sc;  // 购买/升级费用
    public string v;    // 效果数组 JSON 字符串（见 2.3）
    public int    en;   // 启用状态（0=禁用, 1=正常购买）
    public int    bid;  // 蓝图 ID（en=0 且 bid>0 时为蓝图合成类型）
    public int    bcn;  // 所需蓝图数量（blueprint count needed）
}
```

**技能大类型（t 字段）对应关系：**

| t 值 | AbilityType 枚举 | 含义 |
|------|-----------------|------|
| 1 | ABILITY_TYPE_ATTACK | 攻击（增强对敌伤害） |
| 2 | ABILITY_TYPE_GAIN / Boost | 强化（装弹/射程/弹量等） |
| 3 | ABILITY_TYPE_DEFENSE | 防御（减少受到伤害） |
| 4 | ABILITY_TYPE_HEALTH | 生命值（血量/护甲/回复） |
| 5 | ABILITY_TYPE_ACCURACY | 精准度（散布/射程/移速） |

---

### 2.3 技能效果字段（v 字段）

`v` 是一个 JSON 字符串，需要二次反序列化。

**文件：** `GameLogic/JSON/JSON_Property.cs` + `JSON_Ability_Property.cs`

```csharp
// v 字段内容示例（攻击类技能第1级，+2%手枪伤害）
// v = "[{\"t\":\"2\",\"v\":\"2\",\"vt\":\"2\"}]"
// 注意：实际格式有两种，见下方说明

[Serializable]
public class JSON_Property
{
    public JSON_Ability_Property[] prop;   // 效果数组（一个技能可有多个效果）
}

[Serializable]
public class JSON_Ability_Property
{
    public int    t;   // 效果类型 ID（AbilityPropertyType 枚举，见第5章）
    public string v;   // 效果数值（字符串形式，需自行转换）
    public int    vt;  // 数值类型：1=绝对值(NUMBER)，2=百分比(PERCENT)
}
```

**⚠️ v 字段格式注意：**
- 部分技能 v 字段为 `"[{...}]"` 格式（JSON 数组直接），无外层 `prop` 包裹
- 部分技能 v 字段为空字符串（如 ID=12、13，禁用技能）
- 实现时需容错处理两种格式

**推荐解析代码：**
```csharp
AbilityProperty[] ParseEffects(string v)
{
    if (string.IsNullOrEmpty(v)) return Array.Empty<AbilityProperty>();
    try {
        // 优先尝试 prop 包裹格式
        var wrapper = JsonUtility.FromJson<JSON_Property>(v);
        if (wrapper?.prop != null) return ConvertProps(wrapper.prop);
        // 回退尝试数组格式
        var arr = JsonUtility.FromJson<JSON_Ability_PropertyArray>(v);
        return ConvertProps(arr?.items);
    } catch { return Array.Empty<AbilityProperty>(); }
}
```

---

### 2.4 费用结构

**文件：** `GameLogic/JSON/JSON_ShopCost.cs` / `ShopCost.cs`

```csharp
public class JSON_ShopCost
{
    public uint v;   // 金币（VCur）
    public uint r;   // 钻石（RCur）
    public uint p;   // PVP积分（PCur）
    public uint s;   // 赛季积分（SCur）
    public uint b;   // 蓝图碎片（BCur）
    public uint g;   // 荣耀积分（GCur）
    public uint t;   // 特殊货币（TCur）
    public uint a;   // 技能货币（ACur）★ 技能系统主货币
    public uint d;   // DCur
    public uint f;   // FCur
}

// 货币枚举
public enum Currency { VCur=0, RCur=1, PVPCur=2, SCur=3, BCur=4,
                       GCur=5, TCur=6, ACur=7, DCur=8, FCur=9 }
```

**技能系统货币：`ACur`（Ability Currency = 技能点），对应 JSON 的 `sc.a` 字段。**  
JSON 数据示例中的 `tPg` 字段为旧格式（已标记 Obsolete），新版用 `sc.a`。

---

### 2.5 玩家已拥有技能数据

**文件：** `GameLogic/JSON/JSON_BaseDataAbility.cs`

```csharp
[Serializable]
public class JSON_BaseDataAbility
{
    public int i;   // 技能 ID
    public int l;   // 技能等级（已解锁到的 rank）
}
```

**来源：** 登录时 HTTP 响应 `JSON_BaseData.abi`（`JSON_BaseDataAbility[]`）  
**存储：** 运行时 `BaseData.Abilities`（`JSON_BaseDataAbility[]` 静态字段）

---

### 2.6 额外描述数据（CRC 52）

**文件：** `GameLogic/Data/JSON_AbilityAvDesc.cs`

```csharp
public class JSON_AbilityAvDescData
{
    public string i;   // 描述键（key）
    public string v;   // 描述值（value）
}
public class JSON_AbilityAvDesc : JSON_DATA_ITEM
{
    public JSON_AbilityAvDescData[] d;
}
// PlayerPrefs 键："CRCD52"（CRCType.AV_DESC_ABILITY = 52）
```

---

## 3. 内存数据层

### 3.1 AbilityData 类结构

**文件：** `Abilities/AbilityData.cs`

```csharp
public class AbilityData
{
    public readonly AbilityItem[] Items;   // 全部 150×5 条技能（含所有等级）

    public AbilityData(JSON_Ability_Item[] abil)
    {
        Items = abil.Select(j => new AbilityItem(j)).ToArray();
        // 与 BaseData.Abilities 合并，设置 IsBuyed
        foreach (var item in Items)
        {
            item.IsBuyed = BaseData.Abilities
                ?.Any(a => a.i == item.Id && a.l == item.Level) ?? false;
        }
    }
}
```

---

### 3.2 AbilityItem 类

```csharp
public class AbilityData
{
    public class AbilityItem
    {
        // ── 枚举 ──────────────────────────────────────────────────────────
        public enum AbilityType
        {
            NONE = 0,
            ABILITY_TYPE_ATTACK    = 1,   // 攻击
            ABILITY_TYPE_GAIN      = 2,   // 强化（Boost）
            ABILITY_TYPE_DEFENSE   = 3,   // 防御
            ABILITY_TYPE_HEALTH    = 4,   // 生命值
            ABILITY_TYPE_ACCURACY  = 5    // 精准度
        }

        public enum AvailabilityType
        {
            Disabled  = 0,   // en=0 且无蓝图 → 完全禁用
            Available = 1,   // en=1 → 正常购买（用 ACur）
            InGacha   = 2,   // 通过抽卡获得
            Blueprint = 3    // 通过蓝图合成解锁（bid > 0）
        }

        // ── 核心字段 ───────────────────────────────────────────────────────
        public readonly int             Id;           // JSON: i
        public readonly int             Level;        // JSON: l（等级 1-5）
        public readonly AbilityType     Type;         // JSON: t
        public readonly int             Order;        // JSON: o
        public readonly int             NeedLevel;    // JSON: nl（解锁需求等级）
        public readonly ShopCost        ShopCost;     // JSON: sc
        public          AbilityProperty[] Property;   // JSON: v（解析后）
        public readonly int             BlueprintID;  // JSON: bid
        public readonly int             BlueprintCountToUpgrade; // JSON: bcn
        public          string          v;            // 原始 JSON v 字符串（备用）
        public          AbilityAdditionalValues AdditionalValues; // CRC52 额外描述
        public          bool            IsBuyed;      // 与 BaseData 合并后确定
        public readonly AvailabilityType Availability; // 根据 en/bid 计算

        // ── 可用性判断逻辑 ─────────────────────────────────────────────────
        // Availability = (en==1) ? Available : (bid>0) ? Blueprint : (InGacha or Disabled)
        // 注：InGacha 与 Disabled 通过 CRCD51 配置区分

        public AbilityItem(JSON_Ability_Item j)
        {
            Id          = j.i;
            Level       = j.l;
            Type        = (AbilityType)j.t;
            Order       = j.o;
            NeedLevel   = j.nl;
            ShopCost    = new ShopCost(j.sc);
            BlueprintID = j.bid;
            BlueprintCountToUpgrade = j.bcn;
            v           = j.v;
            Availability = j.en == 1 ? AvailabilityType.Available
                         : j.bid > 0  ? AvailabilityType.Blueprint
                         : AvailabilityType.Disabled; // or InGacha
            Property    = ParseEffects(j.v);
        }
    }

    // ── AbilityProperty（单个效果） ────────────────────────────────────────
    public class AbilityProperty
    {
        public readonly AbilityPropertyType  PropertyType;   // 效果类型枚举
        public readonly string               Value;          // 数值字符串
        public readonly AbilityPropertyValueType ValueType;  // NUMBER=1 / PERCENT=2

        public AbilityProperty(JSON_Ability_Property j)
        {
            PropertyType = (AbilityPropertyType)j.t;
            Value        = j.v;
            ValueType    = (AbilityPropertyValueType)j.vt;
        }

        // 格式化显示值
        public string DisplayValue => ValueType == AbilityPropertyValueType.PERCENT
            ? Value + "%" : Value;
    }
}
```

---

### 3.3 AbilityAdditionalValues

```csharp
public class AbilityAdditionalValues
{
    public class AbilityAvDesc
    {
        public string Key;     // 描述键
        public string Value;   // 描述内容
    }
    public AbilityAvDesc[] Descriptions;
}
```

---

## 4. 管理器层

### 4.1 AbilityManagerV2 完整接口

**文件：** `Abilities/AbilityManagerV2.cs`  
**模式：** 线程安全单例（`_sync` 对象锁，懒加载）

```csharp
public class AbilityManagerV2
{
    // ── 单例 ───────────────────────────────────────────────────────────────
    private static readonly object _sync = new object();
    private static AbilityManagerV2 _instance;
    public static AbilityManagerV2 Instance
    {
        get { lock (_sync) { return _instance ??= new AbilityManagerV2(); } }
    }

    // ── 数据 ───────────────────────────────────────────────────────────────
    public AbilityData Data { get; private set; }

    // ── 初始化 ─────────────────────────────────────────────────────────────
    public void Init(JSON_ABILITIES json)
    {
        Data = new AbilityData(json.d);   // 构建内存数据，合并 BaseData.Abilities
    }

    public void InitAdditionalValues(JSON_AbilityAvDesc json)
    {
        // 将 CRC52 数据填充到对应 AbilityItem.AdditionalValues
    }

    // ── 查询 ───────────────────────────────────────────────────────────────
    // 同一 ID 的最高 rank（1-5）
    public int GetLastRank(AbilityData.AbilityItem ability);

    // 返回同一 ID 最高 rank 的 AbilityItem
    public AbilityData.AbilityItem GetLastRankAbility(AbilityData.AbilityItem ability);

    // 返回当前 Item 的下一级 Item（level+1），不存在返回 null
    public AbilityData.AbilityItem GetNextAbilityRank(AbilityData.AbilityItem ability);

    // 是否存在下一级
    public bool IsNextAbilityRankExist(AbilityData.AbilityItem ability);

    // 是否有任意可购买技能
    public bool CanBuyAnyAbility();
    public bool CanBuyAnyAbilityByType(AbilityData.AbilityItem.AbilityType type);

    // ── 购买（直接用 ACur）─────────────────────────────────────────────────
    public void BuyAbility(AbilityData.AbilityItem ability)
    {
        // 发送 Ajax HTTP POST → 服务器确认
        // Photon 操作码：OperationCode.AddAbility = 121
    }

    private void OnBuyAbility(object result, AjaxRequest request)
    {
        // 成功：ApplyBuyedAbility(ability)
        // 失败：触发 OnBuyAbilityRequestDone(ability, false)
    }

    public void ApplyBuyedAbility(AbilityData.AbilityItem ability)
    {
        ability.IsBuyed = true;
        OnBuyAbilityRequestDone?.Invoke(ability, true);
    }

    // ── 蓝图合成 ───────────────────────────────────────────────────────────
    public void CraftAbility(AbilityData.AbilityItem ability);
    private void OnCraftAbility(object result, AjaxRequest request);

    // ── 准备使用（供 ShopManager 调用统一入口）──────────────────────────────
    public bool PrepareItemForUsing(AbilityData.AbilityItem ability);

    // ── 事件 ───────────────────────────────────────────────────────────────
    public event Action<AbilityData.AbilityItem, bool> OnBuyAbilityRequestDone;
    public event Action<AbilityData.AbilityItem, bool> OnCraftAbilityRequestDone;
}
```

---

### 4.2 AbilityGachaManager 接口

**文件：** `AbilityGacha/AbilityGachaManager.cs`

```csharp
public class AbilityGachaManager
{
    public static AbilityGachaManager Instance => ...;   // 单例

    public List<AbilityGachaStepData> StepData { get; private set; }
    public int CurrentStep { get; private set; }    // 当前抽卡阶段（递增）

    public void Init(JSON_ABILITY_GACHA json);      // 从 CRCD51 加载
    public void PrepareSteps();                     // 准备阶段数据
    public bool CanSpin();                          // 余额够且未 MAX
    public bool IsAvailableSpin();                  // 功能已解锁
    public void Spin();                             // 执行抽卡（Ajax）

    public event Action<AbilityData.AbilityItem, bool> OnSpinGachaAbilityRequestDone;
    public event Action OnStepsPrepared;
}

public class AbilityGachaStepData
{
    public readonly int       Id;     // 阶段 ID
    public readonly ShopCost  Cost;   // 该阶段花费（通常用 ACur）
}
```

**抽卡响应：**
```csharp
public class JSON_AbilityGachaSpinResponse : JSON_SendToGMRequest
{
    public class JSON_AbilityGacha { public int i; public int l; }
    public JSON_AbilityGacha a;   // 抽到的技能 {ID, level}
}
```

---

## 5. 战斗应用层

### 5.1 AbilityPropertyType 完整枚举

**文件：** `Abilities/AbilityData.cs`（内嵌枚举）

```csharp
public enum AbilityPropertyType
{
    NONE = 0,

    // ── 伤害增强（ATTACK 类，t=1）──────────────────────────────────────────
    ABILITY_DAMAGE_COLDARMS         = 1,    // 近战武器伤害 +X%
    ABILITY_DAMAGE_HANDGUN          = 2,    // 手枪伤害 +X%
    ABILITY_DAMAGE_MACHINEGUN       = 3,    // 突击步枪伤害 +X%
    ABILITY_DAMAGE_SUBMACHINEGUN    = 4,    // 冲锋枪伤害 +X%
    ABILITY_DAMAGE_GATLINGUN        = 5,    // 轻机枪伤害 +X%
    ABILITY_DAMAGE_SHOTGUN          = 6,    // 霰弹枪伤害 +X%
    ABILITY_DAMAGE_SNIPERRIFLE      = 7,    // 狙击枪伤害 +X%
    ABILITY_DAMAGE_GRENADE          = 30,   // 手雷伤害 +X%
    ABILITY_DAMAGE_HEADSHOT         = 50,   // 爆头伤害 +X%
    ABILITY_RANGE_EXPLOSIVE_GRENADE = 70,   // 高爆手雷爆炸半径 +X%
    ABILITY_RANGE_TOXIC_GRENADE     = 71,   // 毒气手雷范围 +X%
    ABILITY_DAMAGE_IMPACT_FIRE      = 90,   // 燃烧效果伤害 +X%
    ABILITY_DAMAGE_IMPACT_BLOOD     = 91,   // 流血效果伤害 +X%
    ABILITY_DAMAGE_IMPACT_POISON    = 92,   // 中毒效果伤害 +X%

    // ── 穿甲增强（ATTACK 子类）────────────────────────────────────────────
    ABILITY_PENETRATION_HANDGUN         = 110,
    ABILITY_PENETRATION_MACHINEGUN      = 111,
    ABILITY_PENETRATION_SUBMACHINEGUN   = 112,
    ABILITY_PENETRATION_GATLINGUN       = 113,
    ABILITY_PENETRATION_SHOTGUN         = 114,
    ABILITY_PENETRATION_SNIPERRIFLE     = 115,

    // ── 击杀奖励伤害（GAIN/Boost 子类）───────────────────────────────────
    ABILITY_DAMAGE_KS_ROCKET_LAUNCHER   = 160,  // 火箭弹道伤害
    ABILITY_DAMAGE_KS_ROCKET_FLAMER     = 161,  // 火焰弹道伤害

    // ── 移速（持枪时，ACCURACY/GAIN 类）──────────────────────────────────
    ABILITY_RAPIDITY_COLDARMS       = 190,
    ABILITY_RAPIDITY_HANDGUN        = 191,
    ABILITY_RAPIDITY_MACHINEGUN     = 192,
    ABILITY_RAPIDITY_SUBMACHINEGUN  = 193,
    ABILITY_RAPIDITY_GATLINGUN      = 194,
    ABILITY_RAPIDITY_SHOTGUN        = 195,
    ABILITY_RAPIDITY_SNIPERRIFLE    = 196,

    // ── 装弹速度（GAIN/Boost 类）──────────────────────────────────────────
    ABILITY_RELOAD_HANDGUN          = 210,
    ABILITY_RELOAD_MACHINEGUN       = 211,
    ABILITY_RELOAD_SUBMACHINEGUN    = 212,
    ABILITY_RELOAD_GATLINGUN        = 213,
    ABILITY_RELOAD_SHOTGUN          = 214,
    ABILITY_RELOAD_SNIPERRIFLE      = 215,

    // ── 弹药容量（GAIN/Boost 类）──────────────────────────────────────────
    ABILITY_AMMO_HANDGUN            = 240,
    ABILITY_AMMO_MACHINEGUN         = 241,
    ABILITY_AMMO_SUBMACHINEGUN      = 242,
    ABILITY_AMMO_GATLINGUN          = 243,
    ABILITY_AMMO_SHOTGUN            = 244,
    ABILITY_AMMO_SNIPERRIFLE        = 245,

    // ── 击杀奖励费用降低（GAIN 类）────────────────────────────────────────
    ABILITY_COST_KS_ROCKET_LAUNCHER     = 270,
    ABILITY_COST_KS_RADAR               = 271,
    ABILITY_COST_KS_ANTIRADAR           = 272,
    ABILITY_COST_KS_TURRET              = 273,
    ABILITY_COST_KS_RC_BOMB_CAR         = 274,
    ABILITY_COST_KS_HELICOPTER_TURRET   = 275,
    ABILITY_COST_KS_SHIELD              = 276,
    ABILITY_COST_KS_JUGGERNAUT          = 277,
    ABILITY_COST_KS_FLAMER              = 278,
    ABILITY_COST_KS_HUNCHED_DOG         = 279,
    ABILITY_COST_KS_MG_PLANE            = 280,

    // ── 击杀积分增益（GAIN 类）────────────────────────────────────────────
    ABILITY_POINT_KS_BYKILL             = 300,

    // ── 持枪移动速度（ACCURACY/Boost 类）─────────────────────────────────
    ABILITY_MSPEED_COLDARMS         = 320,
    ABILITY_MSPEED_HANDGUN          = 321,
    ABILITY_MSPEED_MACHINEGUN       = 322,
    ABILITY_MSPEED_SUBMACHINEGUN    = 323,
    ABILITY_MSPEED_GATLINGUN        = 324,
    ABILITY_MSPEED_SHOTGUN          = 325,
    ABILITY_MSPEED_SNIPERRIFLE      = 326,

    // ── 受到伤害减免（DEFENSE 类，t=3）───────────────────────────────────
    ABILITY_DECDAMAGE_COLDARMS          = 360,
    ABILITY_DECDAMAGE_HANDGUN           = 361,
    ABILITY_DECDAMAGE_MACHINEGUN        = 362,
    ABILITY_DECDAMAGE_SUBMACHINEGUN     = 363,
    ABILITY_DECDAMAGE_GATLINGUN         = 364,
    ABILITY_DECDAMAGE_SHOTGUN           = 365,
    ABILITY_DECDAMAGE_SNIPERRIFLE       = 367,
    ABILITY_DECDAMAGE_EXPLOSIVE_GRENADE = 390,
    ABILITY_DECDAMAGE_HEADSHOT          = 410,
    ABILITY_DECDAMAGE_IMPACT_FIRE       = 430,
    ABILITY_DECDAMAGE_IMPACT_BLOOD      = 431,
    ABILITY_DECDAMAGE_IMPACT_POISON     = 432,

    // ── 穿甲减免（DEFENSE 子类）──────────────────────────────────────────
    ABILITY_DECPENETRATION_HANDGUN          = 450,
    ABILITY_DECPENETRATION_MACHINEGUN       = 451,
    ABILITY_DECPENETRATION_SUBMACHINEGUN    = 452,
    ABILITY_DECPENETRATION_GATLINGUN        = 453,
    ABILITY_DECPENETRATION_SHOTGUN          = 454,
    ABILITY_DECPENETRATION_SNIPERRIFLE      = 455,

    // ── 生命值 / 护甲（HEALTH 类，t=4）──────────────────────────────────
    ABILITY_HEALTH              = 470,   // 最大生命值 +X
    ABILITY_ARMOR               = 471,   // 最大护甲值 +X
    ABILITY_RESTORE_HEALTH_FSHOT= 480,   // 首发后回复生命值
    ABILITY_RESTORE_ARMOR_FSHOT = 481,   // 首发后回复护甲
    ABILITY_RESTORE_HEALTH_KILL = 482,   // 击杀后回复生命值
    ABILITY_RESTORE_ARMOR_KILL  = 483,   // 击杀后回复护甲
    ABILITY_BOOSTER_TIME        = 490,   // 增益器持续时间延长

    // ── 武器有效射程（ACCURACY 类，t=5）─────────────────────────────────
    ABILITY_WEAPON_MRANGE_COLDARMS      = 520,
    ABILITY_WEAPON_MRANGE_HANDGUN       = 521,
    ABILITY_WEAPON_MRANGE_MACHINEGUN    = 522,
    ABILITY_WEAPON_MRANGE_SUBMACHINEGUN = 523,
    ABILITY_WEAPON_MRANGE_GATLINGUN     = 524,
    ABILITY_WEAPON_MRANGE_SHOTGUN       = 525,
    ABILITY_WEAPON_MRANGE_SNIPERRIFLE   = 526,

    // ── 水平散射上限（ACCURACY 类）───────────────────────────────────────
    ABILITY_WEAPON_HSMAX_HANDGUN        = 550,
    ABILITY_WEAPON_HSMAX_MACHINEGUN     = 551,
    ABILITY_WEAPON_HSMAX_SUBMACHINEGUN  = 552,
    ABILITY_WEAPON_HSMAX_GATLINGUN      = 553,
    ABILITY_WEAPON_HSMAX_SHOTGUN        = 554,
    ABILITY_WEAPON_HSMAX_SNIPERRIFLE    = 555,

    // ── 移动散射最小值（ACCURACY 类）─────────────────────────────────────
    ABILITY_WEAPON_HSMINM_HANDGUN       = 580,
    ABILITY_WEAPON_HSMINM_MACHINEGUN    = 581,
    ABILITY_WEAPON_HSMINM_SUBMACHINEGUN = 582,
    ABILITY_WEAPON_HSMINM_GATLINGUN     = 583,
    ABILITY_WEAPON_HSMINM_SHOTGUN       = 584,
    ABILITY_WEAPON_HSMINM_SNIPERRIFLE   = 585,

    // ── 每种武器各属性（新增通用枚举区段）────────────────────────────────
    ABILITY_WEAPON_HANDGUN      = 600,
    ABILITY_WEAPON_MACHINEGUN   = 601,
    ABILITY_WEAPON_SUBMACHINEGUN= 602,
    ABILITY_WEAPON_SNIPERRIFLE  = 603,
}

public enum AbilityPropertyValueType
{
    NONE    = 0,
    NUMBER  = 1,   // 绝对值（直接加减）
    PERCENT = 2    // 百分比（乘以(1 + value/100)）
}
```

---

### 5.2 战斗中的技能效果应用机制

**核心结论：技能效果由服务器端预计算，客户端被动接收已计算后的属性值。**

```
玩家登录
  → 服务器读取 BaseData.Abilities（已解锁技能列表）
  → 服务器遍历每个技能的 Property 数组
  → 按 AbilityPropertyType 将效果累加到角色属性

对战开始（Spawn）
  → SpawnDC.Health  ← 已含技能加成后的最大生命值
  → KitDC.Speed     ← 已含技能移速加成的移动速度

受到伤害
  → ImpactEvent.HealthDamage / EnergyDamage ← 服务器计算，穿甲/减甲已处理

客户端作用
  → 直接使用服务器下发数值，不需要在客户端重新计算技能加成
  → 技能数据仅用于 UI 展示（当前等级、升级费用、效果描述）
```

**客户端侧只需关注的字段：**
- `IsBuyed` → 当前 UI 状态（已购/未购）
- `Availability` → 购买入口显示
- `Property[].Value + ValueType` → 效果描述文本中的 `{[value]}` 填充
- `ShopCost.ACur` → 购买按钮显示价格

---

## 6. UI 展示层

### 6.1 窗口层级结构

```
AbilityWindowV2  (WindowType = 57)
├── AbilityToggleView × 5          // 顶部 Tab（Attack/Boost/Defense/Health/Accuracy）
│   ├── _type: AbilityType
│   └── OnTogglePressed: Action<AbilityType>
│
└── AbilityPanelV2 × 5             // 每类型一个面板
    ├── _abilityType: AbilityType
    ├── _gridContainer: Transform  // 网格容器（Scroll View 内）
    ├── _abilityInfoPanel: AbilityInfoPanelV2   // 右侧详情
    ├── _abilityItems: List<AbilityItem>
    └── _abilityItemsView: List<AbilityItemViewV2>
```

---

### 6.2 技能格子（AbilityItemViewV2）

```csharp
public class AbilityItemViewV2 : MonoBehaviour
{
    private Image    _icon;            // 技能图标（Sprite）
    private Image    _bg;              // 背景图（已购/未购不同）
    private Image    _bgGlow;          // 发光效果
    private GameObject _lockedState;  // 显示等级锁
    private GameObject _buyedState;   // 显示已购买状态
    private GameObject _notBuyedState;// 显示未购买状态
    private RectTransform _grid;      // 等级星级网格
    private GameObject _maxLevelGO;  // MAX 标识
    private SmallPriceViewV2 _priceView; // 小号价格显示

    public AbilityData.AbilityItem Item { get; }

    // 填充：技能图标、数据、点击回调
    public void Fill(Sprite tex, AbilityData.AbilityItem item,
                     Action<AbilityItemViewV2> onClick);

    // 外部触发升级后刷新格子状态
    public void SetUpgradeState(AbilityData.AbilityItem item);
}
```

---

### 6.3 技能详情面板（AbilityInfoPanelV2）

```csharp
public class AbilityInfoPanelV2 : MonoBehaviour
{
    // 名称与描述（I2 Localization 组件）
    private Localize _abilityName;                     // SetTerm("Ability/ability_name_" + id)
    private Localize _abilityDescription;              // SetTerm("Ability/ability_desc_" + id)
    private LocalizationParamsManager _descParams;     // 注入 {[value]} 参数
    private Image    _imgAbility;                      // 技能图标

    // 三种互斥状态
    private GameObject _lockedState;     // 等级不足：显示需求等级
    private GameObject _buyedState;      // 已购买：显示升级面板
    private GameObject _notBuyedState;  // 未购买：显示购买按钮

    // 组件
    private BigPriceViewV2         _buyBtnPriceView;     // 购买价格按钮
    private LocalizationParamsManager _needLvlParams;    // "需要Lv.X"文本参数
    private AbilityLevelInfoViewV2 _levelView;           // 等级/星级进度
    private UpgradeAbilityPanelV2  _upgradePanel;        // 升级按钮面板

    // 填充选中技能的所有详情
    public void Fill(AbilityItemViewV2 selectedItem)
    {
        var item = selectedItem.Item;
        // 1. 名称本地化
        _abilityName.SetTerm("Ability/ability_name_" + item.Id);
        // 2. 描述本地化 + 参数替换
        _abilityDescription.SetTerm("Ability/ability_desc_" + item.Id);
        _descParams.SetParameterValue("value", item.Property[0].DisplayValue);
        // 3. 按可用性切换状态面板
        SetAbilityItemState(item);
        // 4. 等级进度显示
        int maxRank = AbilityManagerV2.Instance.GetLastRank(item);
        _levelView.FillLevel(item.Level, maxRank, item.IsBuyed);
        // 5. 属性行展示（AbilitySubTypeV2 × N）
        foreach (var prop in item.Property)
            subTypeView.Fill(prop, item);
    }

    public void OnBuyBtnClicked()
        => AbilityManagerV2.Instance.BuyAbility(_currentItem);
}
```

---

### 6.4 本地化键名规则

| 键名格式 | 示例 | 含义 |
|---------|------|------|
| `Ability/ability_name_{id}` | `Ability/ability_name_1` | 技能名称 |
| `Ability/ability_desc_{id}` | `Ability/ability_desc_1` | 技能描述（含 `{[value]}`） |
| `a_type_{n}` | `a_type_1` → "Attack" | 技能类型名称 |

**占位符替换机制（I2 Localization）：**
```
本地化字符串：  "Enemies take {[value]} more damage from knives"
I2 Params:    key="value", value="25%"（来自 Property[0].DisplayValue）
渲染结果：     "Enemies take 25% more damage from knives"
```

---

### 6.5 升级面板（UpgradeAbilityPanelV2）

```csharp
// 状态枚举
protected enum UpgradeState
{
    None       = 0,   // 未初始化
    Unlocked   = 1,   // 可升级（等级足够，有下一rank）
    Locked     = 2,   // 升级锁定（等级不足）
    Max        = 3,   // 已最高级
    InProgress = 4    // 升级中（延迟升级，服务器处理中）
}

// 判断逻辑
void DetermineUpgradeState(int currentRank, int maxRank, int neededLvl,
                            uint upgradePrice, bool isDelayed, bool isInProgress)
{
    if (currentRank >= maxRank) → Max
    else if (isInProgress)      → InProgress
    else if (playerLevel < neededLvl) → Locked（显示"需要Lv.X"）
    else                        → Unlocked（显示价格按钮）
}
```

---

## 7. 解锁 / 购买 / 升级系统

### 7.1 货币系统

| 货币类型 | 枚举值 | 技能系统用途 |
|---------|-------|------------|
| ACur | 7 | **技能升级专用货币（主要）** |
| VCur | 0 | 金币（部分低级技能） |
| RCur | 1 | 钻石（高级技能） |

```csharp
// 玩家 ACur 余额（LocalUser.cs）
private int _aCur;
public int ACur
{
    get => _aCur;
    set { _aCur = value; OnACurAmountChangedEvent?.Invoke(_aCur); }
}
// PlayerPrefs 存储键: FUFPSUserSaveKeys.MainStat_ACur = 10
```

---

### 7.2 可用性类型决策树

```
JSON 字段 en / bid
│
├── en == 1
│   └── → AvailabilityType.Available（正常购买，用 ACur）
│
├── en == 0 && bid > 0
│   └── → AvailabilityType.Blueprint（蓝图合成）
│
├── en == 0 && bid == 0 && （在 CRCD51 的 Gacha 列表中）
│   └── → AvailabilityType.InGacha（抽卡获得）
│
└── en == 0 && bid == 0 && （不在 Gacha 列表中）
    └── → AvailabilityType.Disabled（完全禁用）
```

---

### 7.3 三种解锁流程

#### 流程一：直接购买（Available）
```
点击购买
 → AbilityInfoPanelV2.OnBuyBtnClicked()
 → AbilityManagerV2.Instance.BuyAbility(ability)
 → Ajax POST（含 ability.Id、ability.Level、ShopCost）
 → Photon OperationCode.AddAbility = 121
 → 服务器验证（余额 / 等级 / 已购状态）
 → 成功：ApplyBuyedAbility(ability) → IsBuyed=true → 触发 OnBuyAbilityRequestDone
 → UI 刷新：格子变为已购状态，详情面板切换到升级视图
 → Analytics.BuyAbility(ability.Id, cost, ability.Level)
```

#### 流程二：蓝图合成（Blueprint）
```
点击合成
 → AbilityManagerV2.Instance.CraftAbility(ability)
 → 检查 BlueprintID 碎片数量 ≥ BlueprintCountToUpgrade
 → Ajax POST
 → 成功：OnCraftAbilityRequestDone?.Invoke(ability, true)
 → Analytics.BuyAbilityBlueprint(id, cost, level, blueprintId, blueprintCount)
```

#### 流程三：Gacha 抽卡（InGacha）
```
点击抽卡
 → AbilityGachaManager.Instance.Spin()
 → 检查 CanSpin()（余额 + 已解锁）
 → Ajax POST（含 CurrentStep）
 → 成功：解析 JSON_AbilityGachaSpinResponse.a.{i, l}
 → 找到对应 AbilityItem → IsBuyed = true
 → OnSpinGachaAbilityRequestDone?.Invoke(abilityItem, true)
 → CurrentStep++
 → Analytics.BuyAbilityGacha(id, cost, level, step)
```

---

### 7.4 升级流程（已购技能升到下一级）

```
玩家已拥有 ability（Level=N），点击升级
 → AbilityInfoPanelV2.OnUpgradeBtnClicked()
 → AbilityManagerV2.Instance.GetNextAbilityRank(currentAbility) → nextAbility
 → 检查：playerLevel ≥ nextAbility.NeedLevel
 → 检查：ACur ≥ nextAbility.ShopCost.ACur
 → 走与购买相同的流程（BuyAbility(nextAbility)）
 → 成功后：currentAbility.IsBuyed 保持，nextAbility.IsBuyed = true
```

---

## 8. 完整数据流

```
                        ┌───────────────────────────────┐
                        │            服务器               │
                        └───────────────────────────────┘
                               │         │         │
                    ┌──────────┘    ┌────┘    ┌────┘
                    ▼               ▼         ▼
            GET /crc          GET /data/4  GET /basedata
         JSON_DATA_CRC       CRCD4.json   JSON_BaseData
         （版本校验）         （技能定义）  （.abi = 已购技能）
                │                  │         │
                ▼                  ▼         ▼
         比对 PlayerPrefs    PlayerPrefs   BaseData.Abilities
         ["CRC"]["4"]       ["CRCD4"]     （运行时缓存）
                                   │
                                   ▼
                        ┌─────────────────────────┐
                        │ DataManager.GetData      │
                        │  (CRCType.ABILITY)       │
                        │ → PlayerPrefs["CRCD4"]   │
                        │ → JsonUtility.FromJson   │
                        │   <JSON_ABILITIES>(str)  │
                        └─────────────┬───────────┘
                                      │
                                      ▼
                        ┌─────────────────────────┐
                        │ AbilityManagerV2.Init    │
                        │  (JSON_ABILITIES json)   │
                        │ → new AbilityData(json.d)│
                        │   ┌──────────────────┐   │
                        │   │ foreach entry:   │   │
                        │   │  new AbilityItem │   │
                        │   │  Parse v field   │   │
                        │   │  Set IsBuyed via │   │
                        │   │  BaseData.Abil.  │   │
                        │   └──────────────────┘   │
                        └─────────────┬───────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                   ▼
            ┌─────────────┐  ┌───────────────┐  ┌──────────────┐
            │ AbilityWindow│  │战斗 SpawnDC   │  │ ShopManager  │
            │ V2（UI）    │  │已含技能加成的  │  │ PrepareItem  │
            │ 展示列表/详情│  │Health/Speed等 │  │ ForUsing()   │
            └─────────────┘  └───────────────┘  └──────────────┘
                    │
         点击购买/升级
                    │
                    ▼
         AbilityManagerV2
         .BuyAbility(item)
                    │
                    ▼
         Ajax POST → 服务器
                    │
         ┌──────────┴──────────┐
         ▼ 成功                ▼ 失败
  ApplyBuyedAbility      OnBuyAbilityRequest
  IsBuyed = true         Done(item, false)
  触发事件 → UI 刷新
```

---

## 9. 迁移实现规范

### 9.1 最小可行实现（MVP）

迁移时按以下顺序实现，每步可独立验证：

**Step 1：数据层**
```csharp
// 1a. 定义数据合约类（完全按照第2章的类结构）
[Serializable] class JSON_Ability_Item { ... }
[Serializable] class JSON_Property { ... }
[Serializable] class JSON_Ability_Property { ... }
[Serializable] class JSON_ABILITIES { public JSON_Ability_Item[] d; }

// 1b. 定义内存数据类
class AbilityData { ... }
class AbilityData.AbilityItem { ... }
class AbilityData.AbilityProperty { ... }

// 1c. 定义枚举
enum AbilityType { ATTACK=1, GAIN=2, DEFENSE=3, HEALTH=4, ACCURACY=5 }
enum AbilityPropertyType { ... }  // 见第5章完整枚举
enum AbilityPropertyValueType { NUMBER=1, PERCENT=2 }
enum AvailabilityType { Disabled, Available, InGacha, Blueprint }
```

**Step 2：加载层**
```csharp
// 数据存储（根据目标平台选择）
// 选项A：PlayerPrefs（与原项目一致）
string json = PlayerPrefs.GetString("CRCD4");
var abilities = JsonUtility.FromJson<JSON_ABILITIES>(json);

// 选项B：本地文件
string json = File.ReadAllText(Application.streamingAssetsPath + "/CRCD4.json");
var wrapper = JsonConvert.DeserializeObject<AbilitiesWrapper>(json);
var abilities = wrapper.d;   // 注意原始JSON顶层有 "crc" 和 "d" 字段

// 选项C：HTTP 请求
// GET /api/data?type=4 → 返回同格式 JSON

AbilityManagerV2.Instance.Init(abilities);
```

**Step 3：玩家数据合并**
```csharp
// 登录后将服务器返回的已购技能列表合并
void MergePlayerAbilities(PlayerAbilityRecord[] ownedAbilities)
{
    foreach (var item in AbilityManagerV2.Instance.Data.Items)
    {
        item.IsBuyed = ownedAbilities
            .Any(a => a.abilityId == item.Id && a.level == item.Level);
    }
}
```

**Step 4：购买系统**
```csharp
// 简化版（无 Photon，用 HTTP）
async Task BuyAbility(AbilityData.AbilityItem ability)
{
    var response = await Http.Post("/api/ability/buy", new {
        abilityId = ability.Id,
        level = ability.Level
    });
    if (response.success)
    {
        ability.IsBuyed = true;
        OnAbilityPurchased?.Invoke(ability);
        // 扣减本地货币显示
        LocalUser.ACur -= (int)ability.ShopCost.ACur;
    }
}
```

**Step 5：战斗中应用（服务器端或客户端计算）**
```csharp
// 方案A：服务器端计算（推荐，与原项目一致）
// 服务器在角色属性计算时遍历已购技能，无需客户端处理

// 方案B：客户端计算（无服务器或单机模式）
float ApplyAbilityBonus(float baseValue, AbilityPropertyType propType, WeaponType weaponType)
{
    // 找到玩家已购的、包含此 propType 效果的最高级技能
    var relevant = Data.Items.Where(a =>
        a.IsBuyed &&
        a.Property.Any(p => p.PropertyType == propType)
    ).OrderByDescending(a => a.Level).FirstOrDefault();

    if (relevant == null) return baseValue;

    var prop = relevant.Property.First(p => p.PropertyType == propType);
    float value = float.Parse(prop.Value);
    return prop.ValueType == AbilityPropertyValueType.PERCENT
        ? baseValue * (1 + value / 100f)   // 百分比加成
        : baseValue + value;               // 绝对值加成
}
```

---

### 9.2 架构选择建议

| 场景 | 推荐方案 |
|------|---------|
| 多人在线游戏 | 服务器端计算（与原项目一致），客户端只做 UI |
| 单机/本地验证 | 客户端计算，数据存 JSON 文件 |
| 弱联网 | 本地计算 + 定期同步已购技能列表 |
| 数据存储 | PlayerPrefs（小数据）/ SQLite（大量数据）/ Addressables（CDN 分发） |

---

### 9.3 v 字段解析注意事项

原始 JSON 数据中 `v` 字段存在两种格式，需容错处理：

```csharp
AbilityProperty[] ParseEffects(string v)
{
    if (string.IsNullOrEmpty(v) || v.Trim().Length <= 2)
        return Array.Empty<AbilityProperty>();
    
    try {
        // 格式1：数组格式 [{t,v,vt}, ...]
        if (v.TrimStart().StartsWith("[")) {
            var items = JsonConvert.DeserializeObject<JSON_Ability_Property[]>(v);
            return items?.Select(i => new AbilityProperty(i)).ToArray()
                   ?? Array.Empty<AbilityProperty>();
        }
        // 格式2：包裹格式 {prop:[{t,v,vt},...]}
        var wrapper = JsonConvert.DeserializeObject<JSON_Property>(v);
        return wrapper?.prop?.Select(i => new AbilityProperty(i)).ToArray()
               ?? Array.Empty<AbilityProperty>();
    }
    catch { return Array.Empty<AbilityProperty>(); }
}
```

---

### 9.4 本地化迁移

```csharp
// 方案A：使用 I2 Localization（与原项目一致）
_abilityName.SetTerm("Ability/ability_name_" + ability.Id);
_descParams.SetParameterValue("value", ability.Property[0].DisplayValue);

// 方案B：自定义本地化（不使用 I2）
string GetAbilityName(int abilityId, string lang = "en")
{
    string key = $"a_{abilityId}_name";
    return LocalizationDict.TryGetValue(key, out var entry)
        ? (lang == "zh" ? entry.zhCN : entry.en)
        : $"[Ability {abilityId}]";
}

string GetAbilityDesc(int abilityId, string valueText, string lang = "en")
{
    string key = $"a_{abilityId}_desc";
    string template = LocalizationDict.TryGetValue(key, out var entry)
        ? (lang == "zh" ? entry.zhCN : entry.en) : "";
    // 替换占位符（去掉颜色标签）
    return template
        .Replace("[feca18ff]", "")
        .Replace("[-]", "")
        .Replace("{[value]}", valueText);
}
```

---

### 9.5 关键依赖项清单

| 依赖 | 用途 | 迁移替代方案 |
|------|------|------------|
| I2 Localization | 多语言文本 + 参数替换 | 自实现 Dictionary + string.Replace |
| Unity PlayerPrefs | JSON 数据本地缓存 | File.ReadAllText / SQLite |
| Photon Network | 购买操作码传输 | HTTP REST API |
| Ajax 请求类 | 异步 HTTP | UnityWebRequest / UniTask |
| JsonUtility / Newtonsoft | JSON 反序列化 | 任意 JSON 库 |
| BaseData 静态类 | 玩家已购技能存储 | 任意 UserDataManager 单例 |

---

## 10. 关键文件路径索引

| 文件 | 职责 |
|------|------|
| `GameLogic/Data/CRCType.cs` | ABILITY=4, ABILITY_GACHA=51, AV_DESC_ABILITY=52 |
| `GameLogic/Data/JSON_ABILITIES.cs` | 技能数据顶层容器 |
| `GameLogic/Data/DataManager.cs` | PlayerPrefs 读写，键 "CRCD4" |
| `GameLogic/JSON/JSON_Ability_Item.cs` | 单条技能 entry（全字段定义） |
| `GameLogic/JSON/JSON_Property.cs` | v 字段容器（prop 数组） |
| `GameLogic/JSON/JSON_Ability_Property.cs` | v 字段单个属性（t/v/vt） |
| `GameLogic/JSON/JSON_ShopCost.cs` | 费用结构（a=ACur 为主） |
| `GameLogic/JSON/JSON_BaseDataAbility.cs` | 玩家已购技能（i+l） |
| `GameLogic/JSON/JSON_BaseData.cs` | 登录响应（含 .abi 字段） |
| `GameLogic/Data/JSON_AbilityAvDesc.cs` | 额外描述（CRC 52） |
| `GameLogic/Data/JSON_ABILITY_GACHA.cs` | 抽卡步骤（CRC 51） |
| `Abilities/AbilityData.cs` | 核心枚举 + AbilityItem + AbilityProperty |
| `Abilities/AbilityManagerV2.cs` | 单例管理器（加载/查询/购买/升级） |
| `Abilities/AbilityWindowV2.cs` | 主 UI 窗口（WindowType=57） |
| `Abilities/AbilityPanelV2.cs` | 按类型分组面板 |
| `Abilities/AbilityInfoPanelV2.cs` | 技能详情（名称/描述/价格/升级） |
| `Abilities/AbilityItemViewV2.cs` | 格子视图（图标/星级/价格） |
| `Abilities/AbilitySubTypeV2.cs` | 属性行（类型名+值+升级差值） |
| `Abilities/UpgradeAbilityPanelV2.cs` | 升级按钮状态面板 |
| `Abilities/AbilityLevelInfoViewV2.cs` | 等级/星级进度显示 |
| `AbilityGacha/AbilityGachaManager.cs` | Gacha 抽卡管理器 |
| `BaseData.cs` | 玩家已购技能运行时缓存 |
| `LocalUser.cs` | ACur 余额（_aCur 字段） |
| `LocalizationHelper.cs` | Loc 路径常量（AbilityTerm 等） |
| `ShopCost.cs` | 货币类型枚举（ACur=7） |
| `Penta/Network/Photon/GM/OperationCode.cs` | AddAbility=121 |
| `FUFPSUserSaveKeys.cs` | MainStat_ACur=10（服务器存储键） |

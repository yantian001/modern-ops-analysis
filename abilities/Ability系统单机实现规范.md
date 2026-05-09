# Ability 技能系统 — 单机实现规范

> 基于 Modern Ops v9.79 逆向分析结果  
> 本文档目标：**去除所有服务器依赖**，所有数据存储、货币计算、战斗效果计算均在本地完成  
> 参考原始文档：`Ability系统深度解析与实现规范.md`  
> 编写日期：2026-05-09

---

## 目录

1. [架构对比：联机版 vs 单机版](#1-架构对比联机版-vs-单机版)
2. [本地数据存储设计](#2-本地数据存储设计)
3. [技能定义数据加载](#3-技能定义数据加载)
4. [玩家存档系统](#4-玩家存档系统)
5. [本地货币系统](#5-本地货币系统)
6. [购买 / 升级系统（本地实现）](#6-购买--升级系统本地实现)
7. [战斗效果计算（本地实现）](#7-战斗效果计算本地实现)
8. [AbilityManager 单机版完整实现](#8-abilitymanager-单机版完整实现)
9. [UI 层适配](#9-ui-层适配)
10. [完整数据流（单机版）](#10-完整数据流单机版)
11. [实现清单与迁移步骤](#11-实现清单与迁移步骤)

---

## 1. 架构对比：联机版 vs 单机版

### 1.1 核心差异总览

| 模块 | 联机版（原始） | 单机版（本文） |
|------|--------------|--------------|
| **技能定义数据来源** | 服务器 CRC 机制下发，PlayerPrefs 缓存 | 本地 StreamingAssets/Resources 文件 |
| **玩家已购技能存储** | 服务器数据库，登录时下发 `BaseData.abi` | 本地 JSON 存档文件 |
| **购买/升级流程** | Ajax POST → 服务器验证 → Photon 通知 | 本地校验 → 直接写入存档 |
| **货币系统（ACur）** | 服务器记账，客户端镜像 | 本地存档，PlayerPrefs 持久化 |
| **战斗效果计算** | 服务器预计算，SpawnDC/KitDC 下发 | 客户端实时计算，`ApplyAbilityBonus()` |
| **蓝图/Gacha 解锁** | 服务器抽卡，结果下发 | 本地伪随机 / 直接配置解锁 |
| **网络依赖** | Photon + HTTP | 零网络依赖 |

### 1.2 单机版架构图

```
┌──────────────────────────────────────────────────────────────────────┐
│                        单机 Ability 系统架构                           │
├──────────────┬───────────────┬──────────────┬────────────────────────┤
│  定义数据层   │   存档数据层   │   管理器层   │      战斗计算层          │
│  ability_    │  PlayerSave   │  Ability     │  AbilityCalculator     │
│  data.json   │  .abilities[] │  Manager     │  ApplyBonus()          │
│  (只读配置)  │  (可读写存档) │  (单例)      │  GetWeaponStats()      │
└──────────────┴───────────────┴──────────────┴────────────────────────┘
       │               │               │                │
       ▼               ▼               ▼                ▼
  Resources        Application     运行时内存         战斗开始时
  StreamingAssets  .persistentData  AbilityData       一次性计算角色
  (打包进安装包)   Path/save.json  (BaseData同结构)  全部属性快照
```

### 1.3 单机版移除的组件

以下组件在单机版中**完全不需要**：

- `Ajax` / `AjaxRequest` / `UnityWebRequest`（HTTP 请求）
- `Photon.Realtime` / `OperationCode.AddAbility = 121`
- `CRC 版本管理机制`（CRC.json / LOC_CRC.json）
- `BaseData` 中依赖服务器下发的 `.abi` 字段
- `AbilityGachaManager`（或改为本地伪随机，见第6章）
- `FUFPSUserSaveKeys.MainStat_ACur`（改为统一存档管理）

---

## 2. 本地数据存储设计

### 2.1 存储分类

单机版中数据分为两类，严格区分：

```
只读配置（游戏发布时打包，不可修改）
├── ability_data.json          ← 技能定义（原 CRCD4.json 内容）
├── ability_localization.json  ← 多语言文本（原 LOC_CRCD1.json 内容）
└── ability_icons/             ← 技能图标 Sprite（Addressables 或 Resources）

可读写存档（运行时生成，随时覆盖）
└── save.json                  ← 玩家全部进度（含已购技能 + 货币）
```

### 2.2 只读配置文件格式

将原始 `CRCD4.json` 直接复制为 `ability_data.json`，格式完全兼容：

```json
{
  "d": [
    {
      "i": 1,
      "l": 1,
      "t": 1,
      "o": 1,
      "nl": 0,
      "sc": { "a": 10 },
      "v": "[{\"t\":\"2\",\"v\":\"2\",\"vt\":\"2\"}]",
      "en": 1,
      "bid": 0,
      "bcn": 0
    }
  ]
}
```

**存放路径：**
- `Assets/StreamingAssets/config/ability_data.json`（Android/iOS 通用）
- 或 `Assets/Resources/config/ability_data` （使用 `Resources.Load<TextAsset>`）

### 2.3 本地存档格式设计

```json
{
  "version": 1,
  "playerLevel": 15,
  "currencies": {
    "gold": 5000,
    "diamonds": 200,
    "abilityPoints": 480,
    "blueprints": {}
  },
  "abilities": [
    { "id": 1, "level": 1 },
    { "id": 1, "level": 2 },
    { "id": 3, "level": 1 }
  ],
  "lastSaveTime": "2026-05-09T12:00:00Z"
}
```

字段说明：
- `abilities[]` — 玩家已解锁的技能条目，每条对应一个 `{id, level}` 组合
- `currencies.abilityPoints` — 对应原始 `ACur`（技能点）
- `currencies.blueprints` — 字典，key 为蓝图 ID，value 为持有数量

---

## 3. 技能定义数据加载

### 3.1 数据合约类（与原版完全兼容）

```csharp
// 文件: Data/AbilityContracts.cs
// 注意：字段名与原始 CRCD4.json 完全一致，可直接解析原始数据

[Serializable]
public class AbilityDataJson
{
    public AbilityItemJson[] d;
}

[Serializable]
public class AbilityItemJson
{
    public int    i;    // 技能基础 ID
    public int    l;    // 等级（1~5）
    public int    t;    // 大类型（1~5）
    public int    o;    // 子类型序号
    public int    nl;   // 解锁所需玩家等级
    public ShopCostJson sc;   // 购买费用
    public string v;    // 效果 JSON 字符串
    public int    en;   // 启用状态（0/1）
    public int    bid;  // 蓝图 ID
    public int    bcn;  // 所需蓝图数量
}

[Serializable]
public class ShopCostJson
{
    public uint v;  // 金币
    public uint r;  // 钻石
    public uint a;  // 技能点（主要）
    public uint b;  // 蓝图碎片
}

[Serializable]
public class AbilityPropertyJson
{
    public int    t;   // 效果类型 ID
    public string v;   // 数值
    public int    vt;  // 1=绝对值 2=百分比
}
```

### 3.2 数据加载器

```csharp
// 文件: Data/AbilityDataLoader.cs

public static class AbilityDataLoader
{
    // ── StreamingAssets 方式（Android/iOS 推荐）──────────────────────────
    public static IEnumerator LoadFromStreamingAssets(
        string fileName,
        System.Action<AbilityDataJson> onComplete)
    {
        string path = System.IO.Path.Combine(
            Application.streamingAssetsPath,
            "config", fileName);

#if UNITY_ANDROID
        // Android 上 StreamingAssets 在 APK 内，需用 UnityWebRequest 读取
        using var req = UnityWebRequest.Get(path);
        yield return req.SendWebRequest();
        if (req.result == UnityWebRequest.Result.Success)
            onComplete?.Invoke(Parse(req.downloadHandler.text));
#else
        string text = System.IO.File.ReadAllText(path);
        onComplete?.Invoke(Parse(text));
        yield break;
#endif
    }

    // ── Resources 方式（简单项目）─────────────────────────────────────────
    public static AbilityDataJson LoadFromResources(string resourcePath = "config/ability_data")
    {
        var asset = Resources.Load<TextAsset>(resourcePath);
        if (asset == null)
        {
            Debug.LogError($"[AbilityDataLoader] 找不到资源: {resourcePath}");
            return null;
        }
        return Parse(asset.text);
    }

    // ── 解析（容错两种 v 字段格式）─────────────────────────────────────────
    private static AbilityDataJson Parse(string json)
    {
        // 原始数据顶层可能是 { "crc": 12345, "d": [...] }
        // JsonUtility 会忽略未定义字段，只取 d 数组
        return JsonUtility.FromJson<AbilityDataJson>(json);
    }
}
```

### 3.3 v 字段效果解析

```csharp
// 文件: Data/AbilityEffectParser.cs

public static class AbilityEffectParser
{
    /// <summary>
    /// 解析 v 字段，容错两种格式：
    ///   格式1: "[{\"t\":2,\"v\":\"5\",\"vt\":2}]"  (JSON 数组)
    ///   格式2: "{\"prop\":[{\"t\":2,\"v\":\"5\",\"vt\":2}]}"  (prop 包裹)
    /// </summary>
    public static AbilityPropertyJson[] Parse(string v)
    {
        if (string.IsNullOrEmpty(v) || v.Trim().Length <= 2)
            return System.Array.Empty<AbilityPropertyJson>();

        string trimmed = v.Trim();
        try
        {
            if (trimmed.StartsWith("["))
            {
                // 格式1：直接数组，JsonUtility 不支持顶层数组，改用 Newtonsoft
                return Newtonsoft.Json.JsonConvert
                    .DeserializeObject<AbilityPropertyJson[]>(trimmed)
                    ?? System.Array.Empty<AbilityPropertyJson>();
            }
            else
            {
                // 格式2：prop 包裹对象
                var wrapper = JsonUtility.FromJson<PropWrapper>(trimmed);
                return wrapper?.prop ?? System.Array.Empty<AbilityPropertyJson>();
            }
        }
        catch (System.Exception e)
        {
            Debug.LogWarning($"[AbilityEffectParser] 解析失败: {e.Message}\n原始v: {v}");
            return System.Array.Empty<AbilityPropertyJson>();
        }
    }

    [Serializable]
    private class PropWrapper
    {
        public AbilityPropertyJson[] prop;
    }
}
```

---

## 4. 玩家存档系统

### 4.1 存档数据结构

```csharp
// 文件: Save/PlayerSave.cs

[Serializable]
public class PlayerSave
{
    public int    version       = 1;
    public int    playerLevel   = 1;
    public CurrencyData currencies = new CurrencyData();
    public List<OwnedAbility> abilities = new List<OwnedAbility>();
    public long   lastSaveTime;

    [Serializable]
    public class CurrencyData
    {
        public long gold          = 1000;  // VCur
        public long diamonds      = 0;     // RCur
        public long abilityPoints = 100;   // ACur（技能点）
        public SerializableDict<int, int> blueprints
            = new SerializableDict<int, int>(); // blueprintId → count
    }

    [Serializable]
    public class OwnedAbility
    {
        public int id;     // 技能基础 ID
        public int level;  // 已解锁等级
    }

    // ── 便捷查询 ────────────────────────────────────────────────────────
    public bool HasAbility(int abilityId, int level)
        => abilities.Any(a => a.id == abilityId && a.level == level);

    public bool HasAbilityAnyLevel(int abilityId)
        => abilities.Any(a => a.id == abilityId);

    public int GetMaxLevel(int abilityId)
        => abilities.Where(a => a.id == abilityId)
                    .Select(a => a.level)
                    .DefaultIfEmpty(0).Max();
}
```

### 4.2 存档管理器

```csharp
// 文件: Save/SaveManager.cs

public class SaveManager : MonoBehaviour
{
    public static SaveManager Instance { get; private set; }
    public PlayerSave Save { get; private set; }

    // 存档文件路径（Android/iOS 可写路径）
    private static string SavePath =>
        System.IO.Path.Combine(Application.persistentDataPath, "save.json");

    private void Awake()
    {
        if (Instance != null) { Destroy(gameObject); return; }
        Instance = this;
        DontDestroyOnLoad(gameObject);
        Load();
    }

    // ── 读取存档 ──────────────────────────────────────────────────────────
    public void Load()
    {
        if (System.IO.File.Exists(SavePath))
        {
            string json = System.IO.File.ReadAllText(SavePath);
            Save = JsonUtility.FromJson<PlayerSave>(json) ?? new PlayerSave();
            // 版本迁移
            MigrateIfNeeded();
        }
        else
        {
            Save = new PlayerSave();  // 首次启动，默认存档
        }
    }

    // ── 写入存档 ──────────────────────────────────────────────────────────
    public void Write()
    {
        Save.lastSaveTime = System.DateTimeOffset.UtcNow.ToUnixTimeSeconds();
        string json = JsonUtility.ToJson(Save, prettyPrint: false);
        System.IO.File.WriteAllText(SavePath, json);
    }

    // ── 存档迁移（版本升级时使用）────────────────────────────────────────
    private void MigrateIfNeeded()
    {
        // 预留：if (Save.version < 2) { ... Save.version = 2; }
    }

    // ── 货币操作（带事件广播）─────────────────────────────────────────────
    public event System.Action<long> OnAbilityPointsChanged;

    public bool TrySpendAbilityPoints(long amount)
    {
        if (Save.currencies.abilityPoints < amount) return false;
        Save.currencies.abilityPoints -= amount;
        Write();
        OnAbilityPointsChanged?.Invoke(Save.currencies.abilityPoints);
        return true;
    }

    public void AddAbilityPoints(long amount)
    {
        Save.currencies.abilityPoints += amount;
        Write();
        OnAbilityPointsChanged?.Invoke(Save.currencies.abilityPoints);
    }

    // ── 技能操作 ──────────────────────────────────────────────────────────
    public event System.Action<int, int> OnAbilityUnlocked; // (abilityId, level)

    public void UnlockAbility(int abilityId, int level)
    {
        if (!Save.HasAbility(abilityId, level))
        {
            Save.abilities.Add(new PlayerSave.OwnedAbility { id = abilityId, level = level });
            Write();
            OnAbilityUnlocked?.Invoke(abilityId, level);
        }
    }

    // ── 蓝图操作 ──────────────────────────────────────────────────────────
    public int GetBlueprintCount(int blueprintId)
    {
        Save.currencies.blueprints.TryGetValue(blueprintId, out int count);
        return count;
    }

    public bool TrySpendBlueprints(int blueprintId, int amount)
    {
        int current = GetBlueprintCount(blueprintId);
        if (current < amount) return false;
        Save.currencies.blueprints[blueprintId] = current - amount;
        Write();
        return true;
    }
}
```

---

## 5. 本地货币系统

联机版货币记录在服务器，本文将其全部迁移为本地存档驱动。

### 5.1 货币类型映射

| 原始枚举 | Currency 值 | 存档字段 | 说明 |
|---------|-----------|---------|------|
| ACur | 7 | `currencies.abilityPoints` | **技能点，最核心** |
| VCur | 0 | `currencies.gold` | 金币 |
| RCur | 1 | `currencies.diamonds` | 钻石 |
| BCur | 4 | `currencies.blueprints[id]` | 蓝图碎片（按 ID 分类） |

### 5.2 货币管理器

```csharp
// 文件: Currency/CurrencyManager.cs

public static class CurrencyManager
{
    // ── ACur（技能点）────────────────────────────────────────────────────
    public static long AbilityPoints
        => SaveManager.Instance.Save.currencies.abilityPoints;

    public static bool CanAffordAbilityPoints(long cost)
        => AbilityPoints >= cost;

    public static bool SpendAbilityPoints(long cost)
        => SaveManager.Instance.TrySpendAbilityPoints(cost);

    public static void EarnAbilityPoints(long amount)
        => SaveManager.Instance.AddAbilityPoints(amount);

    // ── 金币 ─────────────────────────────────────────────────────────────
    public static long Gold
        => SaveManager.Instance.Save.currencies.gold;

    public static bool SpendGold(long cost)
    {
        if (Gold < cost) return false;
        SaveManager.Instance.Save.currencies.gold -= cost;
        SaveManager.Instance.Write();
        return true;
    }

    // ── 通用：根据 ShopCost 扣减（支持多货币）────────────────────────────
    public static bool CanAfford(ShopCost cost)
    {
        if (cost.AbilityPoints > 0 && !CanAffordAbilityPoints(cost.AbilityPoints)) return false;
        if (cost.Gold > 0        && Gold < cost.Gold)        return false;
        if (cost.Diamonds > 0   && Diamonds < cost.Diamonds) return false;
        return true;
    }

    public static bool Spend(ShopCost cost)
    {
        if (!CanAfford(cost)) return false;
        if (cost.AbilityPoints > 0) SpendAbilityPoints(cost.AbilityPoints);
        if (cost.Gold > 0)          SpendGold(cost.Gold);
        if (cost.Diamonds > 0)      SpendDiamonds(cost.Diamonds);
        return true;
    }
}

// 内存中使用的货币数据结构
public struct ShopCost
{
    public long AbilityPoints;  // sc.a
    public long Gold;           // sc.v
    public long Diamonds;       // sc.r
    public int  BlueprintId;    // bid（蓝图合成时使用）
    public int  BlueprintCount; // bcn

    public static ShopCost FromJson(ShopCostJson j) => new ShopCost
    {
        AbilityPoints = j.a,
        Gold          = j.v,
        Diamonds      = j.r,
    };
}
```

---

## 6. 购买 / 升级系统（本地实现）

### 6.1 购买流程决策树

```
点击购买按钮
     │
     ▼
检查 AvailabilityType
     │
     ├── Available ──→ CheckLevel & CheckACur ──→ SpendACur & UnlockAbility & Save
     │                                                       │
     │                                              触发 OnAbilityPurchased 事件
     │                                                       │
     │                                               UI 刷新格子 + 详情面板
     │
     ├── Blueprint ──→ CheckBlueprintCount ──→ SpendBlueprints & UnlockAbility & Save
     │
     ├── InGacha ───→ 本地 Gacha 抽卡（见 6.3）
     │
     └── Disabled ──→ 按钮置灰，不响应点击
```

### 6.2 核心购买逻辑

```csharp
// 文件: Abilities/LocalAbilityManager.cs

public class LocalAbilityManager
{
    private static readonly object _sync = new object();
    private static LocalAbilityManager _instance;
    public static LocalAbilityManager Instance
    {
        get { lock (_sync) { return _instance ??= new LocalAbilityManager(); } }
    }

    public AbilityData Data { get; private set; }

    // ── 初始化（游戏启动时调用）──────────────────────────────────────────
    public void Init(AbilityDataJson json)
    {
        // 1. 构建内存数据（定义层）
        var items = json.d.Select(j => new AbilityItem(j)).ToArray();

        // 2. 合并本地存档（玩家进度层）
        var save = SaveManager.Instance.Save;
        foreach (var item in items)
        {
            item.IsBuyed = save.HasAbility(item.Id, item.Level);
        }

        Data = new AbilityData(items);
    }

    // ── 查询接口 ──────────────────────────────────────────────────────────
    public int GetMaxLevel(int abilityId)
        => Data.Items.Where(a => a.Id == abilityId)
                     .Select(a => a.Level).DefaultIfEmpty(0).Max();

    public AbilityItem GetNextRank(AbilityItem current)
        => Data.Items.FirstOrDefault(a => a.Id == current.Id && a.Level == current.Level + 1);

    public bool IsNextRankExist(AbilityItem current)
        => GetNextRank(current) != null;

    // ── 购买（Available 类型）────────────────────────────────────────────
    public PurchaseResult BuyAbility(AbilityItem ability)
    {
        // 前置检查
        if (ability.IsBuyed)
            return PurchaseResult.AlreadyOwned;

        int playerLevel = SaveManager.Instance.Save.playerLevel;
        if (playerLevel < ability.NeedLevel)
            return PurchaseResult.LevelNotMet;

        if (!CurrencyManager.CanAfford(ability.Cost))
            return PurchaseResult.InsufficientFunds;

        // 执行购买
        CurrencyManager.Spend(ability.Cost);
        ability.IsBuyed = true;
        SaveManager.Instance.UnlockAbility(ability.Id, ability.Level);

        // 通知系统（战斗属性快照需要重建）
        OnAbilityUnlocked?.Invoke(ability);
        OnAbilityPurchased?.Invoke(ability, true);

        return PurchaseResult.Success;
    }

    // ── 蓝图合成（Blueprint 类型）────────────────────────────────────────
    public PurchaseResult CraftAbility(AbilityItem ability)
    {
        if (ability.BlueprintId <= 0)
            return PurchaseResult.InvalidOperation;

        int have = SaveManager.Instance.GetBlueprintCount(ability.BlueprintId);
        if (have < ability.BlueprintCount)
            return PurchaseResult.InsufficientBlueprints;

        if (!SaveManager.Instance.TrySpendBlueprints(ability.BlueprintId, ability.BlueprintCount))
            return PurchaseResult.InsufficientBlueprints;

        ability.IsBuyed = true;
        SaveManager.Instance.UnlockAbility(ability.Id, ability.Level);
        OnAbilityUnlocked?.Invoke(ability);
        OnCraftAbilityDone?.Invoke(ability, true);

        return PurchaseResult.Success;
    }

    // ── 升级（已购技能 → 下一级）─────────────────────────────────────────
    public PurchaseResult UpgradeAbility(AbilityItem currentLevel)
    {
        var next = GetNextRank(currentLevel);
        if (next == null)        return PurchaseResult.AlreadyMaxLevel;
        if (!currentLevel.IsBuyed) return PurchaseResult.PrerequisiteNotMet;

        return BuyAbility(next);  // 复用购买逻辑
    }

    // ── 事件 ──────────────────────────────────────────────────────────────
    public event System.Action<AbilityItem>       OnAbilityUnlocked;    // 供战斗系统监听
    public event System.Action<AbilityItem, bool> OnAbilityPurchased;
    public event System.Action<AbilityItem, bool> OnCraftAbilityDone;
}

public enum PurchaseResult
{
    Success,
    AlreadyOwned,
    LevelNotMet,
    InsufficientFunds,
    InsufficientBlueprints,
    AlreadyMaxLevel,
    PrerequisiteNotMet,
    InvalidOperation
}
```

### 6.3 本地 Gacha 抽卡（InGacha 类型，可选）

```csharp
// 文件: Abilities/LocalAbilityGacha.cs
// 说明：无服务器，用本地权重随机代替服务器抽卡

public class LocalAbilityGacha
{
    // 抽卡池（从 ability_data.json 中筛选 en=0 且未在 Blueprint 列表中的技能）
    private List<AbilityItem> _pool;

    public void InitPool(AbilityData data)
    {
        _pool = data.Items
            .Where(a => a.Availability == AvailabilityType.InGacha && !a.IsBuyed)
            .ToList();
    }

    // 单次抽卡（消耗 ACur）
    public PurchaseResult Spin(long cost, out AbilityItem result)
    {
        result = null;
        if (_pool.Count == 0) return PurchaseResult.InvalidOperation;
        if (!CurrencyManager.SpendAbilityPoints(cost)) return PurchaseResult.InsufficientFunds;

        // 简单均匀随机（可改为权重随机）
        result = _pool[UnityEngine.Random.Range(0, _pool.Count)];
        result.IsBuyed = true;
        _pool.Remove(result);
        SaveManager.Instance.UnlockAbility(result.Id, result.Level);

        return PurchaseResult.Success;
    }
}
```

---

## 7. 战斗效果计算（本地实现）

**这是单机版与联机版最大的差异所在。** 联机版中效果由服务器预计算后通过 SpawnDC/KitDC 下发，单机版需要客户端自行根据已购技能计算角色属性加成。

### 7.1 角色属性快照

战斗开始时（或技能变更后），计算一次角色完整属性快照，存入 `CombatStats`：

```csharp
// 文件: Combat/CombatStats.cs

/// <summary>
/// 战斗中使用的角色属性快照，由 AbilityCalculator 在战斗开始时生成
/// </summary>
public class CombatStats
{
    // ── 生命与护甲 ────────────────────────────────────────────────────────
    public float MaxHealth        = 100f;
    public float MaxArmor         = 0f;
    public float HealthRestoreOnKill = 0f;
    public float ArmorRestoreOnKill  = 0f;

    // ── 各武器类型的伤害倍率（乘算到武器基础伤害）────────────────────────
    public float DmgMultKnife      = 1f;
    public float DmgMultHandgun    = 1f;
    public float DmgMultMachinegun = 1f;
    public float DmgMultSMG        = 1f;
    public float DmgMultLMG        = 1f;
    public float DmgMultShotgun    = 1f;
    public float DmgMultSniper     = 1f;
    public float DmgMultGrenade    = 1f;
    public float DmgMultHeadshot   = 1f;  // 爆头额外倍率

    // ── 穿甲值加成（加算到武器基础穿甲）─────────────────────────────────
    public float PenetrationHandgun    = 0f;
    public float PenetrationMachinegun = 0f;
    public float PenetrationSMG        = 0f;
    public float PenetrationLMG        = 0f;
    public float PenetrationShotgun    = 0f;
    public float PenetrationSniper     = 0f;

    // ── 受到伤害减免倍率（0.1 = 减少10%）────────────────────────────────
    public float DmgReductKnife      = 0f;
    public float DmgReductHandgun    = 0f;
    public float DmgReductMachinegun = 0f;
    public float DmgReductSMG        = 0f;
    public float DmgReductLMG        = 0f;
    public float DmgReductShotgun    = 0f;
    public float DmgReductSniper     = 0f;
    public float DmgReductGrenade    = 0f;
    public float DmgReductHeadshot   = 0f;

    // ── 装弹速度乘算（0.1 = 快10%）───────────────────────────────────────
    public float ReloadMultHandgun    = 1f;
    public float ReloadMultMachinegun = 1f;
    public float ReloadMultSMG        = 1f;
    public float ReloadMultLMG        = 1f;
    public float ReloadMultShotgun    = 1f;
    public float ReloadMultSniper     = 1f;

    // ── 弹药容量加算 ─────────────────────────────────────────────────────
    public float AmmoAddHandgun    = 0f;
    public float AmmoAddMachinegun = 0f;
    public float AmmoAddSMG        = 0f;
    public float AmmoAddLMG        = 0f;
    public float AmmoAddShotgun    = 0f;
    public float AmmoAddSniper     = 0f;

    // ── 移动速度乘算（持特定武器时）──────────────────────────────────────
    public float MoveSpeedMultHandgun    = 1f;
    public float MoveSpeedMultMachinegun = 1f;
    public float MoveSpeedMultSMG        = 1f;
    public float MoveSpeedMultLMG        = 1f;
    public float MoveSpeedMultShotgun    = 1f;
    public float MoveSpeedMultSniper     = 1f;

    // ── 武器有效射程乘算 ─────────────────────────────────────────────────
    public float RangeMultHandgun    = 1f;
    public float RangeMultMachinegun = 1f;
    public float RangeMultSMG        = 1f;
    public float RangeMultLMG        = 1f;
    public float RangeMultShotgun    = 1f;
    public float RangeMultSniper     = 1f;

    // ── 散布精度（水平散射上限，减少=更准，用乘数表达）─────────────────
    public float AccuracyMultHandgun    = 1f;
    public float AccuracyMultMachinegun = 1f;
    public float AccuracyMultSMG        = 1f;
    public float AccuracyMultLMG        = 1f;
    public float AccuracyMultShotgun    = 1f;
    public float AccuracyMultSniper     = 1f;
}
```

### 7.2 属性计算器

```csharp
// 文件: Combat/AbilityCalculator.cs

public static class AbilityCalculator
{
    /// <summary>
    /// 根据玩家当前已购技能，计算战斗属性快照
    /// 战斗开始前调用一次，结果缓存在 CombatContext.Stats
    /// </summary>
    public static CombatStats Calculate(AbilityData data)
    {
        var stats = new CombatStats();

        // 只处理"已购买"且等级最高的技能（同一 ID 只取最高级）
        var ownedItems = GetHighestOwnedPerAbility(data);

        foreach (var item in ownedItems)
        {
            foreach (var prop in item.Properties)
            {
                ApplyProperty(stats, prop);
            }
        }

        return stats;
    }

    // ── 获取每个技能 ID 的最高已购等级 ────────────────────────────────────
    private static IEnumerable<AbilityItem> GetHighestOwnedPerAbility(AbilityData data)
        => data.Items
               .Where(a => a.IsBuyed)
               .GroupBy(a => a.Id)
               .Select(g => g.OrderByDescending(a => a.Level).First());

    // ── 核心：将单个 AbilityProperty 应用到 CombatStats ──────────────────
    private static void ApplyProperty(CombatStats stats, AbilityProperty prop)
    {
        float val = prop.FloatValue;
        bool  isPercent = prop.ValueType == AbilityPropertyValueType.PERCENT;

        switch (prop.PropertyType)
        {
            // ── 伤害增强 ──────────────────────────────────────────────────
            case AbilityPropertyType.ABILITY_DAMAGE_COLDARMS:
                stats.DmgMultKnife      *= Mult(val, isPercent); break;
            case AbilityPropertyType.ABILITY_DAMAGE_HANDGUN:
                stats.DmgMultHandgun    *= Mult(val, isPercent); break;
            case AbilityPropertyType.ABILITY_DAMAGE_MACHINEGUN:
                stats.DmgMultMachinegun *= Mult(val, isPercent); break;
            case AbilityPropertyType.ABILITY_DAMAGE_SUBMACHINEGUN:
                stats.DmgMultSMG        *= Mult(val, isPercent); break;
            case AbilityPropertyType.ABILITY_DAMAGE_GATLINGUN:
                stats.DmgMultLMG        *= Mult(val, isPercent); break;
            case AbilityPropertyType.ABILITY_DAMAGE_SHOTGUN:
                stats.DmgMultShotgun    *= Mult(val, isPercent); break;
            case AbilityPropertyType.ABILITY_DAMAGE_SNIPERRIFLE:
                stats.DmgMultSniper     *= Mult(val, isPercent); break;
            case AbilityPropertyType.ABILITY_DAMAGE_GRENADE:
                stats.DmgMultGrenade    *= Mult(val, isPercent); break;
            case AbilityPropertyType.ABILITY_DAMAGE_HEADSHOT:
                stats.DmgMultHeadshot   *= Mult(val, isPercent); break;

            // ── 穿甲增强（加算） ──────────────────────────────────────────
            case AbilityPropertyType.ABILITY_PENETRATION_HANDGUN:
                stats.PenetrationHandgun    += val; break;
            case AbilityPropertyType.ABILITY_PENETRATION_MACHINEGUN:
                stats.PenetrationMachinegun += val; break;
            case AbilityPropertyType.ABILITY_PENETRATION_SUBMACHINEGUN:
                stats.PenetrationSMG        += val; break;
            case AbilityPropertyType.ABILITY_PENETRATION_GATLINGUN:
                stats.PenetrationLMG        += val; break;
            case AbilityPropertyType.ABILITY_PENETRATION_SHOTGUN:
                stats.PenetrationShotgun    += val; break;
            case AbilityPropertyType.ABILITY_PENETRATION_SNIPERRIFLE:
                stats.PenetrationSniper     += val; break;

            // ── 受到伤害减免（加算减免率）────────────────────────────────
            case AbilityPropertyType.ABILITY_DECDAMAGE_COLDARMS:
                stats.DmgReductKnife      += val / 100f; break;
            case AbilityPropertyType.ABILITY_DECDAMAGE_HANDGUN:
                stats.DmgReductHandgun    += val / 100f; break;
            case AbilityPropertyType.ABILITY_DECDAMAGE_MACHINEGUN:
                stats.DmgReductMachinegun += val / 100f; break;
            case AbilityPropertyType.ABILITY_DECDAMAGE_SUBMACHINEGUN:
                stats.DmgReductSMG        += val / 100f; break;
            case AbilityPropertyType.ABILITY_DECDAMAGE_GATLINGUN:
                stats.DmgReductLMG        += val / 100f; break;
            case AbilityPropertyType.ABILITY_DECDAMAGE_SHOTGUN:
                stats.DmgReductShotgun    += val / 100f; break;
            case AbilityPropertyType.ABILITY_DECDAMAGE_SNIPERRIFLE:
                stats.DmgReductSniper     += val / 100f; break;
            case AbilityPropertyType.ABILITY_DECDAMAGE_EXPLOSIVE_GRENADE:
                stats.DmgReductGrenade    += val / 100f; break;
            case AbilityPropertyType.ABILITY_DECDAMAGE_HEADSHOT:
                stats.DmgReductHeadshot   += val / 100f; break;

            // ── 装弹速度（百分比减少装弹时间，转为乘数）──────────────────
            case AbilityPropertyType.ABILITY_RELOAD_HANDGUN:
                stats.ReloadMultHandgun    *= (1f - val / 100f); break;
            case AbilityPropertyType.ABILITY_RELOAD_MACHINEGUN:
                stats.ReloadMultMachinegun *= (1f - val / 100f); break;
            case AbilityPropertyType.ABILITY_RELOAD_SUBMACHINEGUN:
                stats.ReloadMultSMG        *= (1f - val / 100f); break;
            case AbilityPropertyType.ABILITY_RELOAD_GATLINGUN:
                stats.ReloadMultLMG        *= (1f - val / 100f); break;
            case AbilityPropertyType.ABILITY_RELOAD_SHOTGUN:
                stats.ReloadMultShotgun    *= (1f - val / 100f); break;
            case AbilityPropertyType.ABILITY_RELOAD_SNIPERRIFLE:
                stats.ReloadMultSniper     *= (1f - val / 100f); break;

            // ── 弹药容量（加算额外弹量） ──────────────────────────────────
            case AbilityPropertyType.ABILITY_AMMO_HANDGUN:
                stats.AmmoAddHandgun    += val; break;
            case AbilityPropertyType.ABILITY_AMMO_MACHINEGUN:
                stats.AmmoAddMachinegun += val; break;
            case AbilityPropertyType.ABILITY_AMMO_SUBMACHINEGUN:
                stats.AmmoAddSMG        += val; break;
            case AbilityPropertyType.ABILITY_AMMO_GATLINGUN:
                stats.AmmoAddLMG        += val; break;
            case AbilityPropertyType.ABILITY_AMMO_SHOTGUN:
                stats.AmmoAddShotgun    += val; break;
            case AbilityPropertyType.ABILITY_AMMO_SNIPERRIFLE:
                stats.AmmoAddSniper     += val; break;

            // ── 持枪移动速度 ──────────────────────────────────────────────
            case AbilityPropertyType.ABILITY_MSPEED_HANDGUN:
                stats.MoveSpeedMultHandgun    *= Mult(val, isPercent); break;
            case AbilityPropertyType.ABILITY_MSPEED_MACHINEGUN:
                stats.MoveSpeedMultMachinegun *= Mult(val, isPercent); break;
            case AbilityPropertyType.ABILITY_MSPEED_SUBMACHINEGUN:
                stats.MoveSpeedMultSMG        *= Mult(val, isPercent); break;
            case AbilityPropertyType.ABILITY_MSPEED_GATLINGUN:
                stats.MoveSpeedMultLMG        *= Mult(val, isPercent); break;
            case AbilityPropertyType.ABILITY_MSPEED_SHOTGUN:
                stats.MoveSpeedMultShotgun    *= Mult(val, isPercent); break;
            case AbilityPropertyType.ABILITY_MSPEED_SNIPERRIFLE:
                stats.MoveSpeedMultSniper     *= Mult(val, isPercent); break;

            // ── 生命值/护甲（加算）────────────────────────────────────────
            case AbilityPropertyType.ABILITY_HEALTH:
                stats.MaxHealth += val; break;
            case AbilityPropertyType.ABILITY_ARMOR:
                stats.MaxArmor  += val; break;
            case AbilityPropertyType.ABILITY_RESTORE_HEALTH_KILL:
                stats.HealthRestoreOnKill += val; break;
            case AbilityPropertyType.ABILITY_RESTORE_ARMOR_KILL:
                stats.ArmorRestoreOnKill  += val; break;

            // ── 有效射程 ──────────────────────────────────────────────────
            case AbilityPropertyType.ABILITY_WEAPON_MRANGE_HANDGUN:
                stats.RangeMultHandgun    *= Mult(val, isPercent); break;
            case AbilityPropertyType.ABILITY_WEAPON_MRANGE_MACHINEGUN:
                stats.RangeMultMachinegun *= Mult(val, isPercent); break;
            case AbilityPropertyType.ABILITY_WEAPON_MRANGE_SUBMACHINEGUN:
                stats.RangeMultSMG        *= Mult(val, isPercent); break;
            case AbilityPropertyType.ABILITY_WEAPON_MRANGE_SNIPERRIFLE:
                stats.RangeMultSniper     *= Mult(val, isPercent); break;

            // ── 精准度（散布减少=乘数<1）─────────────────────────────────
            case AbilityPropertyType.ABILITY_WEAPON_HSMAX_HANDGUN:
                stats.AccuracyMultHandgun    *= (1f - val / 100f); break;
            case AbilityPropertyType.ABILITY_WEAPON_HSMAX_MACHINEGUN:
                stats.AccuracyMultMachinegun *= (1f - val / 100f); break;
            case AbilityPropertyType.ABILITY_WEAPON_HSMAX_SUBMACHINEGUN:
                stats.AccuracyMultSMG        *= (1f - val / 100f); break;
            case AbilityPropertyType.ABILITY_WEAPON_HSMAX_SNIPERRIFLE:
                stats.AccuracyMultSniper     *= (1f - val / 100f); break;

            default:
                // 未映射的 PropertyType，记录但不崩溃
                // Debug.Log($"[AbilityCalculator] 未处理 PropertyType: {prop.PropertyType}");
                break;
        }
    }

    // ── 百分比转乘数（5% → 1.05f）──────────────────────────────────────────
    private static float Mult(float val, bool isPercent)
        => isPercent ? (1f + val / 100f) : val;
}
```

### 7.3 战斗中使用快照

```csharp
// 文件: Combat/CombatContext.cs

public class CombatContext
{
    public static CombatStats PlayerStats { get; private set; }

    // 战斗开始前调用
    public static void BeginCombat()
    {
        PlayerStats = AbilityCalculator.Calculate(
            LocalAbilityManager.Instance.Data);
    }

    // 武器开火时：计算最终伤害
    public static float GetFinalDamage(float baseDamage, WeaponType weaponType,
                                        bool isHeadshot = false)
    {
        float dmgMult = weaponType switch
        {
            WeaponType.Handgun    => PlayerStats.DmgMultHandgun,
            WeaponType.Machinegun => PlayerStats.DmgMultMachinegun,
            WeaponType.SMG        => PlayerStats.DmgMultSMG,
            WeaponType.Shotgun    => PlayerStats.DmgMultShotgun,
            WeaponType.Sniper     => PlayerStats.DmgMultSniper,
            _                     => 1f
        };
        float damage = baseDamage * dmgMult;
        if (isHeadshot) damage *= PlayerStats.DmgMultHeadshot;
        return damage;
    }

    // 受到伤害时：计算减免后实际伤害
    public static float GetReceivedDamage(float incomingDamage, WeaponType attackerWeapon)
    {
        float reduct = attackerWeapon switch
        {
            WeaponType.Handgun    => PlayerStats.DmgReductHandgun,
            WeaponType.Machinegun => PlayerStats.DmgReductMachinegun,
            WeaponType.Shotgun    => PlayerStats.DmgReductShotgun,
            WeaponType.Sniper     => PlayerStats.DmgReductSniper,
            _                     => 0f
        };
        // 减免上限钳位（防止无敌）
        reduct = Mathf.Clamp(reduct, 0f, 0.75f);
        return incomingDamage * (1f - reduct);
    }

    // 装弹时：应用速度倍数
    public static float GetReloadTime(float baseReloadTime, WeaponType weaponType)
    {
        float mult = weaponType switch
        {
            WeaponType.Handgun    => PlayerStats.ReloadMultHandgun,
            WeaponType.Machinegun => PlayerStats.ReloadMultMachinegun,
            WeaponType.Shotgun    => PlayerStats.ReloadMultShotgun,
            WeaponType.Sniper     => PlayerStats.ReloadMultSniper,
            _                     => 1f
        };
        return Mathf.Max(0.1f, baseReloadTime * mult);  // 防止装弹时间为0
    }

    // 击杀后回血
    public static void OnKill(PlayerController player)
    {
        if (PlayerStats.HealthRestoreOnKill > 0)
            player.Heal(PlayerStats.HealthRestoreOnKill);
        if (PlayerStats.ArmorRestoreOnKill > 0)
            player.RestoreArmor(PlayerStats.ArmorRestoreOnKill);
    }
}
```

### 7.4 技能变更后的快照重建

```csharp
// 文件: GameManager.cs（或任意初始化入口）

// 购买技能后，战斗属性快照需要重建
LocalAbilityManager.Instance.OnAbilityUnlocked += (ability) =>
{
    // 如果在战斗中购买（少见），立刻重建快照
    if (GameManager.IsInCombat)
        CombatContext.BeginCombat();
    // 否则，下次进入战斗时自动重建（BeginCombat 在战斗开始时调用）
};
```

---

## 8. AbilityManager 单机版完整实现

### 8.1 内存数据结构（兼容原版）

```csharp
// 文件: Abilities/AbilityData.cs

public class AbilityData
{
    public readonly AbilityItem[] Items;

    public AbilityData(AbilityItem[] items)
    {
        Items = items;
    }

    // ── 查询便捷方法 ────────────────────────────────────────────────────────
    public AbilityItem[] GetByType(AbilityType type)
        => Items.Where(a => a.Type == type).ToArray();

    public AbilityItem GetByIdAndLevel(int id, int level)
        => Items.FirstOrDefault(a => a.Id == id && a.Level == level);
}

public class AbilityItem
{
    // ── 枚举（与原版完全一致）────────────────────────────────────────────
    public enum AbilityType
    {
        NONE = 0, ATTACK = 1, BOOST = 2, DEFENSE = 3, HEALTH = 4, ACCURACY = 5
    }

    public enum AvailabilityType
    {
        Disabled = 0, Available = 1, InGacha = 2, Blueprint = 3
    }

    // ── 字段 ─────────────────────────────────────────────────────────────
    public readonly int             Id;
    public readonly int             Level;
    public readonly AbilityType     Type;
    public readonly int             Order;
    public readonly int             NeedLevel;
    public readonly ShopCost        Cost;
    public readonly AbilityProperty[] Properties;
    public readonly int             BlueprintId;
    public readonly int             BlueprintCount;
    public readonly AvailabilityType Availability;
    public          bool            IsBuyed;

    public AbilityItem(AbilityItemJson j)
    {
        Id             = j.i;
        Level          = j.l;
        Type           = (AbilityType)j.t;
        Order          = j.o;
        NeedLevel      = j.nl;
        Cost           = ShopCost.FromJson(j.sc);
        BlueprintId    = j.bid;
        BlueprintCount = j.bcn;
        Properties     = AbilityEffectParser.Parse(j.v)
                            .Select(p => new AbilityProperty(p))
                            .ToArray();
        Availability   = j.en == 1 ? AvailabilityType.Available
                       : j.bid > 0  ? AvailabilityType.Blueprint
                       : AvailabilityType.Disabled; // or InGacha（由外部标注）
    }
}

public class AbilityProperty
{
    public readonly AbilityPropertyType      PropertyType;
    public readonly string                   RawValue;
    public readonly AbilityPropertyValueType ValueType;

    public float  FloatValue  => float.TryParse(RawValue, out float f) ? f : 0f;
    public string DisplayValue => ValueType == AbilityPropertyValueType.PERCENT
        ? RawValue + "%" : RawValue;

    public AbilityProperty(AbilityPropertyJson j)
    {
        PropertyType = (AbilityPropertyType)j.t;
        RawValue     = j.v;
        ValueType    = (AbilityPropertyValueType)j.vt;
    }
}
```

### 8.2 完整枚举（与原版一致）

```csharp
public enum AbilityPropertyType
{
    NONE = 0,
    // 伤害增强
    ABILITY_DAMAGE_COLDARMS         = 1,
    ABILITY_DAMAGE_HANDGUN          = 2,
    ABILITY_DAMAGE_MACHINEGUN       = 3,
    ABILITY_DAMAGE_SUBMACHINEGUN    = 4,
    ABILITY_DAMAGE_GATLINGUN        = 5,
    ABILITY_DAMAGE_SHOTGUN          = 6,
    ABILITY_DAMAGE_SNIPERRIFLE      = 7,
    ABILITY_DAMAGE_GRENADE          = 30,
    ABILITY_DAMAGE_HEADSHOT         = 50,
    ABILITY_RANGE_EXPLOSIVE_GRENADE = 70,
    ABILITY_RANGE_TOXIC_GRENADE     = 71,
    ABILITY_DAMAGE_IMPACT_FIRE      = 90,
    ABILITY_DAMAGE_IMPACT_BLOOD     = 91,
    ABILITY_DAMAGE_IMPACT_POISON    = 92,
    // 穿甲增强
    ABILITY_PENETRATION_HANDGUN         = 110,
    ABILITY_PENETRATION_MACHINEGUN      = 111,
    ABILITY_PENETRATION_SUBMACHINEGUN   = 112,
    ABILITY_PENETRATION_GATLINGUN       = 113,
    ABILITY_PENETRATION_SHOTGUN         = 114,
    ABILITY_PENETRATION_SNIPERRIFLE     = 115,
    // 移速（持枪时）
    ABILITY_RAPIDITY_COLDARMS       = 190,
    ABILITY_RAPIDITY_HANDGUN        = 191,
    ABILITY_RAPIDITY_MACHINEGUN     = 192,
    ABILITY_RAPIDITY_SUBMACHINEGUN  = 193,
    ABILITY_RAPIDITY_GATLINGUN      = 194,
    ABILITY_RAPIDITY_SHOTGUN        = 195,
    ABILITY_RAPIDITY_SNIPERRIFLE    = 196,
    // 装弹速度
    ABILITY_RELOAD_HANDGUN          = 210,
    ABILITY_RELOAD_MACHINEGUN       = 211,
    ABILITY_RELOAD_SUBMACHINEGUN    = 212,
    ABILITY_RELOAD_GATLINGUN        = 213,
    ABILITY_RELOAD_SHOTGUN          = 214,
    ABILITY_RELOAD_SNIPERRIFLE      = 215,
    // 弹药容量
    ABILITY_AMMO_HANDGUN            = 240,
    ABILITY_AMMO_MACHINEGUN         = 241,
    ABILITY_AMMO_SUBMACHINEGUN      = 242,
    ABILITY_AMMO_GATLINGUN          = 243,
    ABILITY_AMMO_SHOTGUN            = 244,
    ABILITY_AMMO_SNIPERRIFLE        = 245,
    // 持枪移动速度
    ABILITY_MSPEED_COLDARMS         = 320,
    ABILITY_MSPEED_HANDGUN          = 321,
    ABILITY_MSPEED_MACHINEGUN       = 322,
    ABILITY_MSPEED_SUBMACHINEGUN    = 323,
    ABILITY_MSPEED_GATLINGUN        = 324,
    ABILITY_MSPEED_SHOTGUN          = 325,
    ABILITY_MSPEED_SNIPERRIFLE      = 326,
    // 受到伤害减免
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
    // 穿甲减免
    ABILITY_DECPENETRATION_HANDGUN          = 450,
    ABILITY_DECPENETRATION_MACHINEGUN       = 451,
    ABILITY_DECPENETRATION_SUBMACHINEGUN    = 452,
    ABILITY_DECPENETRATION_GATLINGUN        = 453,
    ABILITY_DECPENETRATION_SHOTGUN          = 454,
    ABILITY_DECPENETRATION_SNIPERRIFLE      = 455,
    // 生命值 / 护甲
    ABILITY_HEALTH              = 470,
    ABILITY_ARMOR               = 471,
    ABILITY_RESTORE_HEALTH_FSHOT= 480,
    ABILITY_RESTORE_ARMOR_FSHOT = 481,
    ABILITY_RESTORE_HEALTH_KILL = 482,
    ABILITY_RESTORE_ARMOR_KILL  = 483,
    ABILITY_BOOSTER_TIME        = 490,
    // 有效射程
    ABILITY_WEAPON_MRANGE_COLDARMS      = 520,
    ABILITY_WEAPON_MRANGE_HANDGUN       = 521,
    ABILITY_WEAPON_MRANGE_MACHINEGUN    = 522,
    ABILITY_WEAPON_MRANGE_SUBMACHINEGUN = 523,
    ABILITY_WEAPON_MRANGE_GATLINGUN     = 524,
    ABILITY_WEAPON_MRANGE_SHOTGUN       = 525,
    ABILITY_WEAPON_MRANGE_SNIPERRIFLE   = 526,
    // 水平散射上限
    ABILITY_WEAPON_HSMAX_HANDGUN        = 550,
    ABILITY_WEAPON_HSMAX_MACHINEGUN     = 551,
    ABILITY_WEAPON_HSMAX_SUBMACHINEGUN  = 552,
    ABILITY_WEAPON_HSMAX_GATLINGUN      = 553,
    ABILITY_WEAPON_HSMAX_SHOTGUN        = 554,
    ABILITY_WEAPON_HSMAX_SNIPERRIFLE    = 555,
    // 移动散射最小值
    ABILITY_WEAPON_HSMINM_HANDGUN       = 580,
    ABILITY_WEAPON_HSMINM_MACHINEGUN    = 581,
    ABILITY_WEAPON_HSMINM_SUBMACHINEGUN = 582,
    ABILITY_WEAPON_HSMINM_GATLINGUN     = 583,
    ABILITY_WEAPON_HSMINM_SHOTGUN       = 584,
    ABILITY_WEAPON_HSMINM_SNIPERRIFLE   = 585,
}

public enum AbilityPropertyValueType
{
    NONE = 0, NUMBER = 1, PERCENT = 2
}
```

---

## 9. UI 层适配

UI 层改动最小，主要是将事件来源从 Ajax 回调改为本地管理器回调。

### 9.1 购买按钮点击处理

```csharp
// 原版（联机）
public void OnBuyBtnClicked()
    => AbilityManagerV2.Instance.BuyAbility(_currentItem);  // 发 Ajax

// 单机版
public void OnBuyBtnClicked()
{
    var result = LocalAbilityManager.Instance.BuyAbility(_currentItem);

    switch (result)
    {
        case PurchaseResult.Success:
            // 购买成功，UI 已通过事件自动刷新
            break;
        case PurchaseResult.InsufficientFunds:
            UINotify.Show("技能点不足！");
            break;
        case PurchaseResult.LevelNotMet:
            UINotify.Show($"需要 Lv.{_currentItem.NeedLevel}");
            break;
        case PurchaseResult.AlreadyOwned:
            // 不应出现（按钮应被禁用）
            break;
    }
}
```

### 9.2 升级状态判断

```csharp
// 单机版 UpgradeState 计算（去除 isInProgress/isDelayed 概念）
UpgradeState GetUpgradeState(AbilityItem current)
{
    int playerLevel = SaveManager.Instance.Save.playerLevel;
    int maxLevel    = LocalAbilityManager.Instance.GetMaxLevel(current.Id);

    if (current.Level >= maxLevel)               return UpgradeState.Max;
    if (!current.IsBuyed)                        return UpgradeState.NotOwned;
    if (playerLevel < GetNextRank(current).NeedLevel)
                                                 return UpgradeState.Locked;
    if (!CurrencyManager.CanAffordAbilityPoints((long)GetNextRank(current).Cost.AbilityPoints))
                                                 return UpgradeState.CannotAfford;
    return UpgradeState.Unlocked;
}

protected enum UpgradeState
{
    NotOwned, Locked, CannotAfford, Unlocked, Max
}
```

### 9.3 本地化文本（无 I2 Localization 时的替代）

```csharp
// 文件: Localization/AbilityLocalization.cs
// 从 LOC_CRCD1.json 解析后构建 Dictionary 查询

public static class AbilityLocalization
{
    private static Dictionary<string, LocalizedEntry> _dict;

    [Serializable]
    public class LocalizedEntry
    {
        public string en;
        public string zhCN;
        // 按需添加其他语言
    }

    public static void Load(string locJson)
    {
        _dict = ParseI2Format(locJson);
    }

    public static string GetName(int abilityId, string lang = "en")
        => Get($"a_{abilityId}_name", lang);

    public static string GetDesc(int abilityId, string valueText, string lang = "en")
    {
        string template = Get($"a_{abilityId}_desc", lang);
        return template
            .Replace("[feca18ff]", "")      // 去除颜色标签
            .Replace("[-]", "")
            .Replace("{[value]}", valueText);
    }

    public static string GetTypeName(int abilityType, string lang = "en")
        => Get($"a_type_{abilityType}", lang);

    private static string Get(string key, string lang)
    {
        if (_dict == null || !_dict.TryGetValue(key, out var entry))
            return $"[{key}]";
        return lang == "zh" ? (entry.zhCN ?? entry.en) : entry.en;
    }

    // 解析 I2Localization 格式
    // 格式: [i2category]...[/i2category]\nKeys[*]Type[*]...[*]English[*]Chinese...[ln]key[*][*]...[*]EN_text[*]ZH_text[ln]...
    private static Dictionary<string, LocalizedEntry> ParseI2Format(string raw)
    {
        var result = new Dictionary<string, LocalizedEntry>(StringComparer.OrdinalIgnoreCase);
        string[] rows = raw.Split(new[] { "[ln]" }, StringSplitOptions.RemoveEmptyEntries);

        // 第一行是 header，找 English 和 Chinese(Simplified) 列索引
        if (rows.Length < 2) return result;

        string[] header = rows[0].Split(new[] { "[*]" }, StringSplitOptions.None);
        int enIdx = System.Array.IndexOf(header, "English");
        int zhIdx = System.Array.IndexOf(header, "Chinese(Simplified)");
        // header 里第0列是 Keys，跳过列 = 实际数据列需减去前3列（Keys/Type/Desc）
        // 精确计算：列0=key, 1=Type, 2=Desc, 3=EN, 4=RU, ...
        // 建议直接用上方 IndexOf 定位

        for (int i = 1; i < rows.Length; i++)
        {
            string[] cols = rows[i].Split(new[] { "[*]" }, StringSplitOptions.None);
            if (cols.Length <= enIdx) continue;
            string key = cols[0].Trim();
            if (string.IsNullOrEmpty(key)) continue;

            result[key] = new LocalizedEntry
            {
                en   = enIdx >= 0 && enIdx < cols.Length ? cols[enIdx] : "",
                zhCN = zhIdx >= 0 && zhIdx < cols.Length ? cols[zhIdx] : ""
            };
        }
        return result;
    }
}
```

---

## 10. 完整数据流（单机版）

```
游戏启动
    │
    ▼
SaveManager.Load()
    │  读取 Application.persistentDataPath/save.json
    │  （首次启动：生成默认存档）
    │
    ▼
AbilityDataLoader.LoadFromResources("config/ability_data")
    │  加载 CRCD4.json 内容（只读定义数据）
    │
    ▼
AbilityLocalization.Load(locJson)
    │  加载 LOC_CRCD1.json（多语言文本）
    │
    ▼
LocalAbilityManager.Instance.Init(abilityDataJson)
    │  ┌──────────────────────────────────────┐
    │  │ foreach entry in json.d:             │
    │  │   new AbilityItem(entry)             │
    │  │   ParseEffects(entry.v)              │
    │  │   item.IsBuyed = save.HasAbility()   │  ← 本地存档
    │  └──────────────────────────────────────┘
    │
    ├──────────────────────────────────────────────────────────────┐
    ▼                                                              ▼
AbilityWindowV2                                          CombatContext.BeginCombat()
（UI 展示）                                              （战斗开始前调用）
    │                                                              │
    │  点击购买/升级                                               ▼
    ▼                                                    AbilityCalculator.Calculate(data)
LocalAbilityManager.BuyAbility(item)                     遍历所有 IsBuyed=true 技能
    │                                                     累加到 CombatStats 快照
    │  本地校验 ─── 失败 ───→ 返回 PurchaseResult 错误
    │
    │  成功:
    ├── CurrencyManager.SpendAbilityPoints(cost)          战斗中使用
    ├── item.IsBuyed = true                               ────────────────────────
    ├── SaveManager.UnlockAbility(id, level)              CombatContext.GetFinalDamage()
    ├── SaveManager.Write()                               CombatContext.GetReceivedDamage()
    └── OnAbilityUnlocked?.Invoke(item)                  CombatContext.GetReloadTime()
              │                                          CombatContext.OnKill()
              ▼
        UI 事件响应
        格子刷新 + 详情面板更新
```

---

## 11. 实现清单与迁移步骤

### 11.1 需要新建的文件

| 文件 | 职责 | 优先级 |
|------|------|--------|
| `Data/AbilityContracts.cs` | JSON 数据合约类（兼容原版字段） | P0 必须 |
| `Data/AbilityEffectParser.cs` | v 字段双格式解析 | P0 必须 |
| `Data/AbilityDataLoader.cs` | 从本地文件加载定义数据 | P0 必须 |
| `Abilities/AbilityData.cs` | 内存数据结构（AbilityItem/AbilityProperty） | P0 必须 |
| `Abilities/AbilityPropertyType.cs` | 完整枚举定义（约80个值） | P0 必须 |
| `Abilities/LocalAbilityManager.cs` | 单例管理器（无网络版本） | P0 必须 |
| `Save/PlayerSave.cs` | 存档数据结构 | P0 必须 |
| `Save/SaveManager.cs` | 存档读写管理器 | P0 必须 |
| `Currency/CurrencyManager.cs` | 本地货币管理 | P0 必须 |
| `Combat/CombatStats.cs` | 战斗属性快照 | P1 联机→单机关键 |
| `Combat/AbilityCalculator.cs` | 技能效果计算器 | P1 联机→单机关键 |
| `Combat/CombatContext.cs` | 战斗期间属性查询入口 | P1 联机→单机关键 |
| `Localization/AbilityLocalization.cs` | I2 Localization 替代方案 | P2 可用 I2 原版 |
| `Abilities/LocalAbilityGacha.cs` | 本地 Gacha 随机抽卡 | P3 可选功能 |

### 11.2 需要移除/替换的组件

| 原始组件 | 替换方案 |
|---------|---------|
| `AjaxRequest` / HTTP 调用 | 直接本地调用 |
| `Photon.Realtime` / `OperationCode` | 删除，不再需要 |
| `BaseData.Abilities`（服务器下发） | `SaveManager.Save.abilities`（本地存档） |
| `PlayerPrefs["CRCD4"]` | `Resources.Load` 或 `StreamingAssets` 文件 |
| `CRC 版本校验机制` | 删除，数据随包体发布 |
| `LocalUser._aCur`（服务器镜像） | `SaveManager.Save.currencies.abilityPoints` |

### 11.3 实现顺序（推荐）

```
Step 1  定义所有数据合约类（AbilityContracts.cs）
          └─ 验证：能正确解析 ability_data.json

Step 2  实现存档系统（PlayerSave.cs + SaveManager.cs）
          └─ 验证：首次启动生成存档，修改后能正确读回

Step 3  实现 LocalAbilityManager.Init()
          └─ 验证：IsBuyed 与存档中的已购技能对应

Step 4  实现购买/升级逻辑（BuyAbility/UpgradeAbility）
          └─ 验证：扣款、解锁、存档更新正确

Step 5  实现 AbilityCalculator（战斗属性计算）
          └─ 验证：购买技能后，战斗属性快照数值正确

Step 6  接入 CombatContext，替换原 SpawnDC.Health 等硬编码
          └─ 验证：血量/移速/伤害受技能影响

Step 7  UI 适配（改购买按钮事件，改文本查询）
          └─ 验证：界面显示正确，购买成功 UI 正确刷新

Step 8（可选）实现本地 Gacha 逻辑
```

### 11.4 注意事项与边界条件

| 场景 | 处理方式 |
|------|---------|
| 同一技能存在多个 `Properties`（多效果技能） | `AbilityCalculator` 需遍历所有 `Properties`，不只取第一个 |
| 同一 `AbilityPropertyType` 被多个技能影响 | 按设计累加（乘算类用 `*=`，加算类用 `+=`） |
| 减免类属性需设置上限 | 建议 `DmgReductXxx` 最大值钳位至 `0.75f`（75%减免） |
| 装弹时间最小值保护 | `GetReloadTime` 结果 `Mathf.Max(0.1f, ...)` |
| 存档被篡改 / 格式错误 | `try/catch` + 生成新默认存档 |
| 数据文件不存在（漏打包） | 日志报错 + 返回空数据（不崩溃） |
| 技能等级不连续（玩家直接拥有Lv3但无Lv1/2） | `AbilityCalculator` 只看 `IsBuyed=true`，无需检查连续性 |
| 货币溢出（long 类型上限约 9.2×10^18） | long 足够，无需特殊处理 |

---

## 附录：与联机版对应关系速查

| 联机版代码 | 单机版对应 | 变化说明 |
|-----------|-----------|---------|
| `AbilityManagerV2` | `LocalAbilityManager` | 去掉 Ajax/Photon，改为本地校验+存档 |
| `BaseData.Abilities` | `SaveManager.Save.abilities` | 从服务器下发改为本地存档 |
| `LocalUser.ACur` | `SaveManager.Save.currencies.abilityPoints` | 从服务器镜像改为本地存档 |
| `AbilityManagerV2.BuyAbility()` → Ajax | `LocalAbilityManager.BuyAbility()` → 本地 | 同步调用，无回调延迟 |
| `OperationCode.AddAbility = 121` | 删除 | 无 Photon |
| `SpawnDC.Health` / `KitDC.Speed`（服务器计算） | `CombatStats.MaxHealth` / `MoveSpeedMult`（本地计算） | **最核心变化** |
| `PlayerPrefs["CRCD4"]`（服务器缓存） | `Resources.Load("config/ability_data")` | 本地只读文件 |
| `DataManager.GetData(CRCType.ABILITY)` | `AbilityDataLoader.LoadFromResources()` | 去掉 CRC 机制 |
| `AbilityGachaManager.Spin()` → Ajax | `LocalAbilityGacha.Spin()` → 本地随机 | 改为伪随机 |
| I2 Localization | `AbilityLocalization`（自实现）| 可选，可保留 I2 |

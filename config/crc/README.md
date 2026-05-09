# config/crc — 游戏数据库（CRC缓存文件）

> Modern Ops v9.79 | 数据来源：`Assets/Resources/config/crc/` | 总 CRC=1770287727

## 目录说明

本目录包含游戏所有核心数据的 JSON 文件。文件名格式为 `CRCD{N}.json`，其中 N 对应 `CRCType` 枚举值。

**数据加载机制：**
1. 客户端启动时从 `CRC.json` 读取所有数据类型的校验和
2. 与本地 PlayerPrefs 中缓存的 CRC 值对比
3. 若不一致，向服务器请求最新 JSON 并更新 PlayerPrefs 缓存
4. 离线时回退使用本目录内的捆绑默认数据

---

## 文件索引

| 文件名 | CRCType | 数据类型 | 记录数 | 大小 |
|--------|---------|---------|-------|------|
| [CRC.json](./CRC.json) | — | 各类型 CRC 校验和索引 | — | 0.7 KB |
| [CRCD3.json](./CRCD3.json) | MAP=3 | 地图/关卡配置 | 15 | 0.8 KB |
| [CRCD4.json](./CRCD4.json) | ABILITY=4 | 技能/能力定义 | 723 | 83.7 KB |
| [CRCD5.json](./CRCD5.json) | ACHIEVEMENT=5 | 成就定义 | 139 | 10.7 KB |
| [CRCD6.json](./CRCD6.json) | CONTRACT=6 | 合同/任务系统 | 61 | 3.0 KB |
| [CRCD8.json](./CRCD8.json) | BANK_PACKAGE=8 | 充值礼包定义 | 242 | 50.8 KB |
| [CRCD9.json](./CRCD9.json) | BANK_MONEY=9 | 货币兑换包 | — | 空 |
| [CRCD11.json](./CRCD11.json) | BANK_PREMIUM=11 | 高级货币包 | — | 空 |
| [CRCD12.json](./CRCD12.json) | BANK_IAP=12 | 内购商品定义 | 374 | 28 KB |
| [CRCD13.json](./CRCD13.json) | KILLSTREAK=13 | 连杀奖励定义 | 76 | 9.1 KB |
| [CRCD14.json](./CRCD14.json) | CARD=14 | 卡片道具 | 73 | 2.4 KB |
| [CRCD15.json](./CRCD15.json) | FLAG=15 | 旗帜/标志 | 24 | 0.8 KB |
| [CRCD16.json](./CRCD16.json) | NLVLREWARD=16 | 等级提升奖励 | 149 | 18.4 KB |
| [CRCD17.json](./CRCD17.json) | SEASON=17 | 赛季数据 | 7 | 26.8 KB |
| [CRCD18.json](./CRCD18.json) | GLORY=18 | 荣耀/排行榜物品 | 55 | 2.0 KB |
| [CRCD20.json](./CRCD20.json) | BLUEPRINT=20 | 武器蓝图配方 | 89 | 5.5 KB |
| [CRCD21.json](./CRCD21.json) | MEDAL=21 | 奖牌/徽章 | 144 | 5.5 KB |
| [CRCD22.json](./CRCD22.json) | LOGO=22 | 徽标/图案 | — | 空 |
| [CRCD24.json](./CRCD24.json) | SETTING=24 | 游戏配置参数 | 43 | 0.9 KB |
| [CRCD25.json](./CRCD25.json) | LEAGUEVALUE=25 | 联赛积分段位范围 | — | 0.2 KB |
| [CRCD26.json](./CRCD26.json) | NEWS=26 | 游戏内新闻公告 | 6 | 24 KB |
| [CRCD27.json](./CRCD27.json) | WEAPONS=27 | **武器完整数据库** | 748 | 117.5 KB |
| [CRCD28.json](./CRCD28.json) | WEARS=28 | 装扮/皮肤数据库 | 572 | 61 KB |
| [CRCD29.json](./CRCD29.json) | GRENADES=29 | 手雷/投掷物 | 12 | 1.5 KB |
| [CRCD30.json](./CRCD30.json) | BOOSTERS=30 | 加速器/增益道具 | 4 | 0.5 KB |
| [CRCD31.json](./CRCD31.json) | WEAPON_PARTS=31 | **武器配件数据库** | 3454 | 437.9 KB |
| [CRCD32.json](./CRCD32.json) | WEAPON_UPGRADES=32 | 武器升级路径 | 621 | 49 KB |
| [CRCD33.json](./CRCD33.json) | WEAR_UPGRADES=33 | 装扮升级路径 | 468 | 34.3 KB |
| [CRCD34.json](./CRCD34.json) | KILLSTREAK_UPGRADES=34 | 连杀奖励升级 | 64 | 4.8 KB |
| [CRCD35.json](./CRCD35.json) | CHESTS=35 | 箱子/宝箱定义 | 24 | 0.6 KB |
| [CRCD36.json](./CRCD36.json) | GAME_EVENT=36 | 游戏活动/事件 | 195 | 8.8 KB |
| [CRCD37.json](./CRCD37.json) | GAME_EVENT_GROUP=37 | 活动分组 | 42 | 39.6 KB |
| [CRCD38.json](./CRCD38.json) | ASSET_BUNDLE=38 | 资源包配置 | — | 空 |
| [CRCD39.json](./CRCD39.json) | SUBSCRIBE=39 | 订阅/战斗通行证 | 3 | 0.4 KB |
| [CRCD40.json](./CRCD40.json) | WEAR_REF_KILLSTREAK=40 | 装扮-连杀奖励关联 | — | 空 |
| [CRCD41.json](./CRCD41.json) | CLAN_LOGO=41 | 战队徽标 | 1 | 0.1 KB |
| [CRCD42.json](./CRCD42.json) | CHEST_LOC=42 | 箱子本地化描述 | 22 | 21 KB |
| [CRCD43.json](./CRCD43.json) | CLAN_UPGRADE=43 | 战队升级定义 | 4 | 0.3 KB |
| [CRCD44.json](./CRCD44.json) | GAME_EVENT_GROUP_CIRCULAR=44 | 循环活动组 | 8 | 3.7 KB |
| [CRCD45.json](./CRCD45.json) | AV_DESC_WEAR=45 | 装扮附加值描述 | 41 | 2.1 KB |
| [CRCD46.json](./CRCD46.json) | AV_DESC_WEAPON=46 | 武器附加值描述 | 245 | 11.6 KB |
| [CRCD47.json](./CRCD47.json) | AV_DESC_GRENADE=47 | 手雷附加值描述 | 6 | 0.3 KB |
| [CRCD48.json](./CRCD48.json) | AV_DESC_BOOSTER=48 | 加速器附加值描述 | 4 | 0.2 KB |
| [CRCD49.json](./CRCD49.json) | AV_DESC_KILLSTREAK=49 | 连杀附加值描述 | 26 | 1.2 KB |
| [CRCD100.json](./CRCD100.json) | LANG_BANK_PACKAGE=100 | 本地化充值包名称 | 24 | 4.8 KB |

---

## 通用 JSON 结构

所有 CRCD 文件的顶层结构：
```json
{
  "crc": 1234567890,   // 本文件的 CRC 校验和（与 CRC.json 中对应值一致）
  "d": [ ... ]         // 数据数组
}
```

---

## 重点文件字段说明

### CRCD3.json — 地图 (MAP)
| 字段 | 含义 |
|------|------|
| `i` | 地图ID |
| `n` | 地图系统名称 |
| `m` | 地图模式标志（bitmask） |
| `ena` | 是否启用 |
| `nlvl` | 解锁等级 |

### CRCD13.json — 连杀奖励 (KILLSTREAK)
| 字段 | 含义 |
|------|------|
| `i` | 连杀奖励ID |
| `sn` | 系统名称 |
| `t` | 类型（2=无人机/雷达, 3=反无人机 等） |
| `c` | 连杀数要求 |
| `cd` | 冷却时间（秒） |
| `stDu` | 持续时间（秒） |
| `ena` | 是否启用 |
| `rar` | 稀有度 |

### CRCD17.json — 赛季 (SEASON)
| 字段 | 含义 |
|------|------|
| `i` | 赛季ID |
| `gid` | 组ID |
| `ena` | 是否启用 |
| `il` | 赛季奖励列表（i=道具ID, v=数量, vT=类型） |

### CRCD24.json — 游戏设置 (SETTING)
| 字段 | 含义 |
|------|------|
| `i` | 设置项ID |
| `v` | 设置值（可为数字或 JSON 字符串） |

### CRCD28.json — 装扮 (WEARS)
| 字段 | 含义 |
|------|------|
| `i` | 装扮ID |
| `wt` | 装扮类型（8=护甲 等） |
| `sn` | 系统名称 |
| `stArm` | 护甲值 |
| `ena` | 1=基础版, 2=升级版 |
| `ord` | 排序权重 |

### CRCD29.json — 手雷 (GRENADES)
| 字段 | 含义 |
|------|------|
| `i` | 手雷ID |
| `gt` | 手雷类型（1=M67高爆, 3=燃烧弹 等） |
| `scnt` | 起始数量 |
| `am` | 最大携带数 |
| `stDa` | 伤害值 |
| `lt` | 存活时间（毫秒） |

### CRCD31.json — 武器配件 (WEAPON_PARTS)
| 字段 | 含义 |
|------|------|
| `i` | 配件ID |
| `pt` | 配件类型（1=光学/2=枪口/3=下挂/4=弹匣/5=枪托/6=皮肤/7=挂件/8=弹药） |
| `sn` | 配件系统名称 |
| `v` | 配件附加值描述（JSON字符串） |
| `ena` | 是否启用 |

### CRCD32.json — 武器升级 (WEAPON_UPGRADES)
| 字段 | 含义 |
|------|------|
| `u_id` | 武器基础ID |
| `lvl` | 升级等级 |
| `w_id` | 当前等级武器ID |
| `rw_id` | 升级后武器ID |
| `nlvl` | 所需玩家等级 |
| `sc` | 升级费用 |

### CRCD36.json — 游戏活动 (GAME_EVENT)
| 字段 | 含义 |
|------|------|
| `i` | 活动ID |
| `v` | 活动值/积分 |
| `sn` | 系统名称 |
| `r` | 奖励类型 |
| `o` | 活动类型 |

### CRCD37.json — 活动组 (GAME_EVENT_GROUP)
| 字段 | 含义 |
|------|------|
| `i` | 活动组ID |
| `sD` / `eD` | 开始/结束时间（UTC） |
| `ena` | 是否启用 |
| `l` / `lM` | 最小/最大等级要求 |
| `eids` | 包含的活动ID列表（逗号分隔） |

---

## CRCType 枚举完整对照

```csharp
// GameLogic/Data/CRCType.cs
public enum CRCType {
    NONE = 0, MAP = 3, ABILITY = 4, ACHIEVEMENT = 5, CONTRACT = 6,
    BANK_PACKAGE = 8, BANK_MONEY = 9, BANK_PREMIUM = 11, BANK_IAP = 12,
    KILLSTREAK = 13, CARD = 14, FLAG = 15, NLVLREWARD = 16, SEASON = 17,
    GLORY = 18, BLUEPRINT = 20, MEDAL = 21, LOGO = 22, SETTING = 24,
    LEAGUEVALUE = 25, NEWS = 26, WEAPONS = 27, WEARS = 28, GRENADES = 29,
    BOOSTERS = 30, WEAPON_PARTS = 31, WEAPON_UPGRADES = 32, WEAR_UPGRADES = 33,
    KILLSTREAK_UPGRADES = 34, CHESTS = 35, GAME_EVENT = 36, GAME_EVENT_GROUP = 37,
    ASSET_BUNDLE = 38, SUBSCRIBE = 39, WEAR_REF_KILLSTREAK = 40, CLAN_LOGO = 41,
    CHEST_LOC = 42, CLAN_UPGRADE = 43, GAME_EVENT_GROUP_CIRCULAR = 44,
    AV_DESC_WEAR = 45, AV_DESC_WEAPON = 46, AV_DESC_GRENADE = 47,
    AV_DESC_BOOSTER = 48, AV_DESC_KILLSTREAK = 49, GACHA = 50,
    ABILITY_GACHA = 51, AV_DESC_ABILITY = 52, ARMORY = 53,
    LANG_BANK_PACKAGE = 100
}
```

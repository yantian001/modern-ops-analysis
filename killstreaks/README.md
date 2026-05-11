# killstreaks — 连杀奖励技能数据总览

> Modern Ops: Gun Shooting Games v9.79 | 数据来源：`Assets/Resources/config/crc/CRCD13.json` + `CRCD34.json`

## 概述

游戏共有 **12 种连杀奖励技能**（11 种 KillStreakType + Copter 的火焰直升机变种），每种技能有 **5～7 个升级等级**，总计 76 条记录。

| sn | 中文名称 | 英文名称 | 类型(t) | KillStreakType | 击杀激活 | 冷却(s) | 稀有度 |
|----|---------|---------|---------|--------------|---------|---------|------|
| Radar | 雷达 | RADAR | 2 | Radar | 15 | 40 | 普通 |
| AntiRadar | 雷达干扰 | RADAR INTERFERENCE | 3 | AntiRadar | 20 | 45 | 普通 |
| RL_RPG7 | 火箭筒 | ROCKET LAUNCHER | 1 | RocketLauncher | 30 | 70 | 普通 |
| HGWS_Berreta9MWS | 护盾和手枪 | SHIELD WITH HAND GUN | 7 | Shield | 20 | 70 | 普通 |
| TR_GatlingTurret | 炮塔 | TURRET | 4 | Turret | 40 | 80 | 稀有 |
| RadioCar | 无线电通讯车 | RADIO CAR | 5 | RCBombCar | 25 | 60 | 稀有 |
| GG_M134 | 主宰者 | JUGGERNAUT | 8 | Juggernaut | 45 | 90 | 史诗 |
| FL_Flamethrower | 火焰喷射器 | FLAMETHROWER | 9 | Flamer | 35 | 80 | 史诗 |
| Dog | 军犬 | BATTLE DOG | 10 | Dog | 30 | 75 | 史诗 |
| RocketArtillery | 炮火袭击 | ARTILLERY BOMBARDMENT | 13 | RocketArtillery | 75 | 100 | 普通 |
| Copter | 直升机 | HELICOPTER | 14 | Copter | 50 | 85 | 稀有 |
| CopterFlamer | 火焰直升机 | HELICOPTER (FLAMER) | 14 | Copter | 60 | 95 | 稀有 |

**稀有度说明：** rar=1→普通，rar=2→稀有，rar=3→史诗

---

## 数据文件说明

### CRCD13.json — KILLSTREAK（76 条记录）

| 字段 | C# 属性 | 说明 |
|------|---------|------|
| `i` | ID | 唯一标识符 |
| `sn` | — | 技能名称键（与 LOC 本地化对应） |
| `t` | KType (KillStreakType) | 技能类型枚举值 |
| `c` | CostExecution | 激活所需击杀数 |
| `cd` | Cooldown | 冷却时间（秒） |
| `stDi` | Range | 范围/射程 |
| `stDa` | Damage | 伤害值 |
| `stSp` | Mobility | 移动速度加成（可为负数） |
| `stAm` | Ammo | 弹药量 |
| `stHe` | Health | 生命值 |
| `stAr` | Armor | 护甲值 |
| `stSt` | Strength | 爆炸力/强度 |
| `stDu` | DurationSec | 持续时间（秒） |
| `stDps` | FakeDps | 显示用DPS（UI展示） |
| `stLv` | FakeLevel | 显示用等级（UI展示） |
| `nl` | — | 解锁所需玩家等级 |
| `rar` | — | 稀有度（1=普通, 2=稀有, 3=史诗） |
| `sc` | — | 初始购买费用 |

**`sc` 货币类型：**
- `tPv`：软货币（金币）
- `tPr`：高级货币（钻石）

### CRCD34.json — KILLSTREAK_UPGRADES（64 条记录）

| 字段 | 说明 |
|------|------|
| `u_id` | 升级链 ID（对应同一类技能） |
| `lvl` | 升级级别（从 1 开始） |
| `sid` | 升级前的技能 ID |
| `rid` | 升级后的技能 ID |
| `nlvl` | 执行升级所需的玩家等级 |
| `ena` | 是否启用（1=是） |
| `sc` | 升级费用（tPv/tPr） |

---

## KillStreakType 枚举

```csharp
public enum KillStreakType
{
    None = 0,
    RocketLauncher = 1,   // RL_RPG7 火箭筒
    Radar = 2,            // Radar 雷达
    AntiRadar = 3,        // AntiRadar 雷达干扰
    Turret = 4,           // TR_GatlingTurret 炮塔
    RCBombCar = 5,        // RadioCar 无线电通讯车
    HelicopterTurret = 6,
    Shield = 7,           // HGWS_Berreta9MWS 护盾+手枪
    Juggernaut = 8,       // GG_M134 主宰者
    Flamer = 9,           // FL_Flamethrower 火焰喷射器
    Dog = 10,             // Dog 军犬
    MGPlane = 11,
    PlaneBombingLine = 12,
    RocketArtillery = 13, // RocketArtillery 炮火袭击
    Copter = 14,          // Copter/CopterFlamer 直升机
    RocketHomingLauncher = 15,
    Wall = 16,
    EnergyShield = 17,
    Spy = 18,
    SnowStorm = 19,
    SnowGrenadeLauncher = 20,
    BioStorm = 21,
    FlameGrenadeLauncher = 22,
    Cannon = 23,
    WeaponPursuer = 24,
    Samurai = 25,
}
```

---

## 详细数据

见 [killstreaks.md](./killstreaks.md)

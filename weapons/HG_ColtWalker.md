# HG_ColtWalker 武器属性文档

> 数据来源：CRCD27.json（内置武器数据库，CRC=1770287727）| 游戏版本：v9.79 | 生成日期：2026-05-09

## 基本信息

| 属性 | 值 |
|------|-----|
| **系统名称 (sn)** | HG_ColtWalker |
| **武器ID (i)** | 2130 |
| **武器类型 (wt)** | HAND_GUN (手枪) |
| **稀有度 (rar)** | 稀有 (Rare) |
| **解锁等级 (nl)** | 无限制 |
| **综合威力 (stPo)** | 3400 |

## 实际战斗数值（基础版）

| 属性 | 值 | 说明 |
|------|-----|------|
| **伤害 (stDa)** | 190 | 单次命中基础伤害 |
| **射速 (stRa)** | 125 发/分钟 | 每分钟射击次数 |
| **射程 (stDi)** | 50 | 有效射程评分 |
| **精度 (stDe)** | 86 | 精准度评分 |
| **换弹速度 (stRt)** | 0.385 | 换弹速率系数 |
| **弹匣容量 (am)** | 6 发 | 单弹匣装弹量 |
| **备弹总量 (amt)** | 36 发 | 含弹匣的总弹药 |



## 升级等级数据

| 版本ID | 所需等级 | 伤害(stDa) | 射速(stRa) | 射程(stDi) | 精度(stDe) | 威力(stPo) |
|--------|---------|-----------|-----------|-----------|-----------|-----------|
| 2130 (基础版) | — | 190 | 125 | 50 | 86 | 3400 || 2131 (升级版) | 17 | 211 | 125 | 50 | 86 | 3540 |
| 2132 (升级版) | 17 | 234 | 125 | 50 | 86 | 3700 |
| 2133 (升级版) | 25 | 260 | 125 | 50 | 86 | 3870 |
| 2134 (升级版) | 30 | 288 | 125 | 50 | 86 | 4070 |
| 2135 (升级版) | 35 | 320 | 125 | 50 | 86 | 4280 |

## 战斗属性（服务器通过 Photon 网络下发字段）

以下属性通过 KitWeaponsDC 在 Photon 房间 Hash 中以 WeaponKeys 枚举值为 Key 传递：

| WeaponKey | Byte值 | 说明 |
|-----------|-------|------|
| Velocity | 97 | 子弹飞行速度（短整型） |
| Rapidity | 94 | 射击间隔（毫秒） |
| ReloadTime | 93 | 换弹时间（毫秒） |
| Ammo | 92 | 弹匣容量 |
| AmmoTotal | 91 | 备弹总量 |
| ShortRange | 74 | 近距离衰减临界点（米） |
| MediumRange | 73 | 中距离衰减临界点（米） |
| LongRange | 72 | 远距离衰减临界点（米） |
| MinStand | 59 | 站立最小散布 |
| MaxStand | 57 | 站立最大散布 |
| IncreaseStand | 56 | 站立散布增长量/发 |
| MinCrouch | 63 | 蹲伏最小散布 |
| MaxCrouch | 61 | 蹲伏最大散布 |
| IncreaseCrouch | 60 | 蹲伏散布增长量/发 |
| MinStandOptics | 67 | ADS站立最小散布 |
| MaxStandOptics | 65 | ADS站立最大散布 |
| IncreaseStandOptics | 64 | ADS站立散布增长量/发 |
| ShowingDurationMSec | 45 | 出枪动画时长（毫秒） |
| HidingDurationMSec | 44 | 收枪动画时长（毫秒） |
| ZoomInDurationMSec | 43 | ADS开镜时长（毫秒） |
| ZoomOutDurationMSec | 46 | 关镜时长（毫秒） |
| ReloadingType | 42 | 换弹类型（0=整匣/1=逐发） |
| BoltAction | 103 | 是否需要拉栓（狙击枪） |

## 武器配件可影响属性（WeaponPart）

| 配件属性字段 | 说明 |
|------------|------|
| FireratePercent | 射速提升百分比 |
| RangePoint | 射程点数加成 |
| DamagePercent | 伤害提升百分比 |
| AccuracyPoint | 精度点数加成 |
| ZoomPoint | 瞄准镜倍率 |
| ReloadTimePercent | 换弹速度百分比 |
| AmmoTotalPercent | 备弹量百分比 |
| IsInvisibleForRadar | 雷达隐身 |
| DamageAtDistancePercent | 远程伤害衰减补偿 |
| PrecisionInMotionPercent | 移动精度提升 |
| AdditionalAmmo | 额外弹药 |
| ArmorPenetration | 护甲穿透 |
| AimingTime | 开镜速度调整 |
| HeadDamage | 爆头伤害加成 |
| MovementSpeed | 移速调整 |

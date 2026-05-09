# GG_SIGSauerMG338 武器属性文档

> 数据来源：CRCD27.json（内置武器数据库，CRC=1770287727）| 游戏版本：v9.79 | 生成日期：2026-05-09

## 基本信息

| 属性 | 值 |
|------|-----|
| **系统名称 (sn)** | GG_SIGSauerMG338 |
| **武器ID (i)** | 180 |
| **武器类型 (wt)** | GATLING_GUN (轻机枪) |
| **稀有度 (rar)** | 史诗 (Epic) |
| **解锁等级 (nl)** | Lv.20 |
| **综合威力 (stPo)** | 4550 |

## 实际战斗数值（基础版）

| 属性 | 值 | 说明 |
|------|-----|------|
| **伤害 (stDa)** | 14 | 单次命中基础伤害 |
| **射速 (stRa)** | 576.9 发/分钟 | 每分钟射击次数 |
| **射程 (stDi)** | 49 | 有效射程评分 |
| **精度 (stDe)** | 21 | 精准度评分 |
| **换弹速度 (stRt)** | 0.161 | 换弹速率系数 |
| **弹匣容量 (am)** | 50 发 | 单弹匣装弹量 |
| **备弹总量 (amt)** | 150 发 | 含弹匣的总弹药 |



## 升级等级数据

| 版本ID | 所需等级 | 伤害(stDa) | 射速(stRa) | 射程(stDi) | 精度(stDe) | 威力(stPo) |
|--------|---------|-----------|-----------|-----------|-----------|-----------|
| 180 (基础版) | 20 | 14 | 576.9 | 49 | 21 | 4550 || 1801 (升级版) | 20 | 14 | 588.2 | 50 | 21 | 4790 |
| 1802 (升级版) | 20 | 14 | 594.1 | 51 | 21 | 5060 |
| 1803 (升级版) | 20 | 14 | 606.1 | 52 | 21 | 5370 |
| 1804 (升级版) | 30 | 14 | 612.2 | 53 | 21 | 5730 |
| 1805 (升级版) | 32 | 14 | 625 | 54 | 21 | 6130 |

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

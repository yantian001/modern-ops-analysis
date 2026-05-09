# SR_Crossbow 武器属性文档

> 数据来源：AssetRipper 逆向导出 v9.79 | 生成日期：2026-05-09

## 基本信息

| 属性 | 值 |
|------|-----|
| **系统名称** | SR_Crossbow |
| **武器类型** | SNIPER_RIFLE（狙击步枪） |
| **模型缩放倍率** | 1.2 |
| **FOV 覆盖** | 0 |
| **角色握持位置偏移** | 0.026143178} |

## 配件槽位支持

| 配件类型 | 状态 |
|---------|------|
| OPTIC | ✅ 支持 |
| MUZZLE | ✅ 支持 |
| UNDERBARREL | ✅ 支持 |
| MAGAZINE | ✅ 支持 |
| BUTT | ✅ 支持 |
| SKIN | ✅ 支持 |
| TRINKET | ✅ 支持 |
| AMMO | ✅ 支持 |

## 射击参数

| 参数 | 值 |
|------|-----|
| **射击配置** | autoShotDelay=0.275 | aimClampedSpeed=1.8 |

## 后坐力参数

| 参数 | 值 |
|------|-----|
| **相机后坐力** | 无专属相机后坐力配置 |

## 战斗属性说明（服务器下发字段）

以下属性通过 Photon 网络由服务器实时下发，本地无静态配置文件：

| 字段Key | 字段名 | 说明 |
|---------|--------|------|
| WeaponID (80) | 武器ID | 服务器内唯一武器标识符 |
| WeaponType (98) | 武器类型 | 枚举值，对应 WeaponType enum |
| Velocity (97) | 子弹飞行速度 | 单位：游戏内速度单位 |
| Rapidity (94) | 射速（开火间隔） | 单位：毫秒（越小越快） |
| ReloadTime (93) | 换弹时间 | 单位：毫秒 |
| Ammo (92) | 弹匣容量 | 单发装填数量 |
| AmmoTotal (91) | 备弹总量 | 包含弹匣内弹药 |
| ShortRange (74) | 近距离定义 | 近程衰减临界点（米） |
| MediumRange (73) | 中距离定义 | 中程衰减临界点（米） |
| LongRange (72) | 远距离定义 | 远程衰减临界点（米） |
| MinStand (59) | 站立最小散布 | 精度下限（站立静止） |
| MinMovingStand (58) | 站立移动散布 | 精度下限（站立移动） |
| MaxStand (57) | 站立最大散布 | 精度上限（连续射击） |
| IncreaseStand (56) | 站立散布增长 | 每发增加量 |
| MinCrouch (63) | 蹲下最小散布 | 精度下限（蹲伏静止） |
| MinMovingCrouch (62) | 蹲下移动散布 | 精度下限（蹲伏移动） |
| MaxCrouch (61) | 蹲下最大散布 | 精度上限（蹲伏连射） |
| IncreaseCrouch (60) | 蹲下散布增长 | 每发增加量 |
| MinStandOptics (67) | 开镜站立最小散布 | ADS精度下限 |
| MaxStandOptics (65) | 开镜站立最大散布 | ADS精度上限 |
| IncreaseStandOptics (64) | 开镜站立散布增长 | ADS每发增加量 |
| MinCrouchOptics (71) | 开镜蹲下最小散布 | ADS蹲伏精度下限 |
| ShowingDurationMSec (45) | 出枪时间 | 武器切换动画时长（毫秒） |
| HidingDurationMSec (44) | 收枪时间 | 切换其他武器动画时长（毫秒） |
| ZoomInDurationMSec (43) | 开镜时间 | ADS动画时长（毫秒） |
| ZoomOutDurationMSec (46) | 关镜时间 | 退出ADS动画时长（毫秒） |
| ReloadingType (42) | 换弹类型 | 0=整弹匣换弹，1=逐颗上弹（霰弹枪） |
| BoltAction (103) | 拉栓动作 | 是否需要单发拉栓（如狙击枪） |

## 武器配件影响属性（WeaponPart）

配件安装后可影响以下属性：

| 配件属性 | 说明 |
|---------|------|
| FireratePercent | 射速提升百分比 |
| RangePoint | 射程点数 |
| DamagePercent | 伤害提升百分比 |
| AccuracyPoint | 精度点数 |
| ZoomPoint | 变焦倍率 |
| ReloadTimePercent | 换弹速度百分比 |
| AmmoTotalPercent | 备弹总量百分比 |
| IsInvisibleForRadar | 是否雷达隐身 |
| DamageAtDistancePercent | 远程伤害衰减补偿 |
| PrecisionInMotionPercent | 移动精度提升 |
| AdditionalAmmo | 额外弹药 |
| ArmorPenetration | 护甲穿透 |
| AimingTime | 开镜速度调整 |
| HeadDamage | 爆头伤害加成 |
| MovementSpeed | 移速调整 |

## UI显示属性（客户端本地统计值）

| 属性字段 | 对应JSON字段 | 说明 |
|---------|------------|------|
| Damage | stDa | 伤害评分（0-100） |
| Range | stRa | 射程评分（0-100） |
| Firerate | stRt | 射速评分（0-100） |
| Accuracy | stDe | 精度评分（0-100） |
| ReloadSpeed | — | 换弹速度评分 |
| MoveSpeed | stSp | 移速评分（0-100） |
| Power | stPo | 综合威力评分 |
| Crit | krit | 暴击加成 |

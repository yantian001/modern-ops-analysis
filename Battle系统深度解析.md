# Modern Ops: Gun Shooting Games — Battle 系统深度解析

> 版本：v9.79 | Unity：6000.0.69f1 | 逆向来源：AssetRipper  
> 文档日期：2026-05-09

---

## 目录

1. [系统总览](#1-系统总览)
2. [战斗单元体系（BattleUnit）](#2-战斗单元体系-battleunit)
3. [战斗玩家接口（IBattlePlayer）](#3-战斗玩家接口-ibattleplayer)
4. [武器系统与弹道计算](#4-武器系统与弹道计算)
5. [命中判定流程](#5-命中判定流程)
6. [伤害计算与效果系统](#6-伤害计算与效果系统)
7. [抛物体轨迹系统](#7-抛物体轨迹系统)
8. [网络同步机制](#8-网络同步机制)
9. [AI战斗控制](#9-ai战斗控制)
10. [关键枚举速查](#10-关键枚举速查)
11. [架构关系图](#11-架构关系图)

---

## 1. 系统总览

Battle 系统是整个游戏的核心战斗逻辑层，位于：

```
Assets/Scripts/Assembly-CSharp/Battle/
```

### 主要子模块

| 子目录 | 职责 |
|--------|------|
| `BattleUnits/` | 战斗单元定义（炮塔、火箭、手雷、无人机等） |
| `CombatControl/` | 武器控制、弹道计算、射击判定 |
| `CombatControl/WeaponCalculation/` | 各类武器的具体算法（狙击、霰弹、火焰等） |
| `Network/` | 网络位置/旋转插值同步 |
| `Killstreaks/` | 连杀奖励战斗单元 |
| `Players/` | 本地/第三人称玩家的战斗逻辑 |
| `Effects/` | 武器弹孔 Decal、发光动画效果 |
| `NavMeshManagement/` | AI 导航网格管理 |
| `AI/` | 机器人视野感知 |
| `MapWaypoints/` | 地图路点管理 |
| `PreloadingControllers/` | 战斗场景预加载控制器 |
| `CombatUnitsInfoView/` | 血条、护甲条、指示器等 UI |

---

## 2. 战斗单元体系（BattleUnit）

### 2.1 层次结构

```
BattleUnitBase（抽象基类）
├── 核心属性
│   ├── UnitID : short          — 单元唯一 ID
│   ├── Type   : BattleUnitType — 单元类型枚举
│   └── OwnerActorID : byte?    — 所属玩家的 ActorID
│
├── 抽象接口
│   ├── CurrentPosition  : Vector3
│   ├── BattleUnitView   : BattleUnitViewBase
│   ├── InstantiateAndPrepareView(pos, rot)
│   ├── ProcessNewPositionAndRotation(nTrData)
│   ├── CustomUpdateView()
│   ├── RemoveUnit()
│   ├── DestroyUnit(DestroyReason)
│   ├── Clean()
│   └── TurnIntoControllable(data)  — 切换为玩家可控状态
│
└── 生命周期
    BattleUnitsManager.Add() → Add(BattleUnitBase) → InstantiateAndPrepareView()
    BattleUnitsManager.Destroy(buID, reason) → DestroyUnit(reason)
```

### 2.2 BattleUnitType 枚举（完整列表）

```csharp
public enum BattleUnitType : byte
{
    None                        = 0,
    Turret                      = 1,   // 固定炮塔
    Helicopter                  = 2,   // 直升机
    Rocket                      = 3,   // 直线火箭
    Grenade                     = 4,   // 手雷
    RadioCar                    = 5,   // 遥控车
    BombInBombGameMode          = 6,   // 炸弹模式中的炸弹
    MGPlane                     = 7,   // 机枪战机
    PlaneBombingLine            = 8,   // 轰炸机航线
    PlaneBombingLineProjectile  = 9,   // 轰炸机投弹
    RocketArtilleryLauncher     = 10,  // 火箭炮发射器
    RocketArtilleryProjectile   = 11,  // 火箭炮弹头
    Dog                         = 12,  // 军犬（连杀奖励）
    SupplyCase                  = 13,  // 补给箱
    Shield                      = 14,  // 可放置盾牌
    Provision                   = 15,  // 补给站
    CS_DefuseKit                = 16,  // CS拆弹套件
    Copter                      = 17,  // 小型无人机
    CopterFlamer                = 18,  // 喷火无人机
    RocketHoming                = 19,  // 追踪火箭
    Wall                        = 20,  // 可放置掩体墙
    EnergyShield                = 21,  // 能量护盾
    WeaponPursuer               = 22,  // 武器追踪器
    ClusterExplosiveGrenadeChild= 23,  // 集束手雷子弹药
    TurretMovable               = 24,  // 可移动炮台
    DeathArea                   = 100  // 死亡区域
}
```

### 2.3 单元销毁原因（DestroyReason）

```csharp
public enum DestroyReason
{
    None               = 0,
    Killed             = 1,   // 被击毁
    SuppressedByOtherUnit = 3, // 被其他单元压制取代
    Timeout            = 5,   // 存活时间到期
    Utilized           = 6,   // 主动回收
    HasNoAmmo          = 8,   // 弹药耗尽
    KillstreakFinish   = 9    // 连杀奖励结束
}
```

### 2.4 BattleUnitsManager（单元管理器）

负责所有战斗单元的创建、激活、破坏、同步：

```csharp
// 核心操作
BattleUnitBase Add(BattleUnitDataContract buData)         // 从网络数据创建
BattleUnitBase Add(IBattlePlayer owner, buData)           // 由玩家创建
BattleUnitBase Activate(short buID, bool success)         // 激活/拒绝激活
BattleUnitBase Break(short buID)                          // 破坏（保留残骸）
BattleUnitBase Destroy(short buID, DestroyReason reason)  // 完全销毁
BattleUnitBase Remove(short buID)                         // 从列表移除

// 网络同步
void ProcessNewPositionAndRotation(short buID, NetworkTransformData)
void ProcessShootingData(short buID, BattleUnitShotData)
void ProcessBlowingData(short buID, Vector3 pos, int? nextTimeToBlow)

// 查询
IEnumerable<BattleUnitBase> GetPlayersUnits(byte actorID)
bool TryGetBattleUnit(short unitID, out BattleUnitBase bu)
```

---

## 3. 战斗玩家接口（IBattlePlayer）

`IBattlePlayer` 是连接战斗逻辑与玩家对象的核心接口，所有玩家行为通过此接口操作。

### 3.1 核心属性

```csharp
public interface IBattlePlayer : IPoolObject
{
    int    UserID    { get; }          // 用户ID
    byte   ActorID   { get; }          // 房间内Actor编号
    short  Team      { get; set; }     // 队伍（0/1）
    string UserName  { get; }          // 用户名

    bool IsLocalPlayer  { get; }       // 是否本地玩家
    bool IsDead         { get; }       // 是否死亡
    bool IsInvulnerable { get; }       // 是否无敌

    float? Health    { get; }          // 当前血量（null=不显示）
    float? MaxHealth { get; }
    float? Armor     { get; }          // 当前护甲
    float? MaxArmor  { get; }

    int Level      { get; }            // 玩家等级
    int League     { get; }            // 段位
    string ClanTag { get; }            // 战队标签
    int ClanLogoId { get; }            // 战队图标
}
```

### 3.2 战斗控制器引用

```csharp
IWeaponController       WeaponController       { get; }  // 武器切换/使用
IBaseGrenadesController BaseGrenadesCtrl        { get; }  // 手雷投掷
BoosterController       BoosterController       { get; }  // 增益使用
IColdWeaponController   ColdWeaponController    { get; }  // 冷兵器控制
BattleKillstreaksController BattleKillstreakCtrl { get; } // 连杀奖励
```

### 3.3 核心战斗方法

```csharp
// 伤害与死亡
void UpdateHealth(float? health, float? maxHealth, float? armor, float? maxArmor)
void ImpactUpdate(ImpactType type, short hpDmg, short apDmg, byte value)  // 状态效果伤害
void Kill(bool endGame, Vector3 shotImpulse, PlayerHitZone hz, KillerType kt)
void InstantKill(KillerType kt)                                            // 秒杀

// CS模式专用
void LaunchPlant(int whenPlantingWillFinishMSec)   // 开始埋弹
void AbortPlant()                                   // 中止埋弹
void FinishPlant()                                  // 完成埋弹
void LaunchDefuse(int whenDefusingWillFinishMSec)  // 开始拆弹
void AbortDefuse()
void FinishDefuse()

// 生命周期
void Spawn(SpawnDC spawnData, byte defaultWpnIndex, int invulnerabilityDurMSec)
void FinishPlayerInvulnerability()                 // 结束出生无敌

// 武器
bool OnShot(BattleWeapon shootWeapon)              // 成功开枪回调
bool ShowWeapon(Action<BattleWeapon> onDoneHandler)
bool HideWeapon(Action<BattleWeapon> onDoneHandler)
void HardSwitchWeapon(byte wpnSlotIndexToSelect)
bool SwitchWeaponTo(byte wpnSlotIndex, int? timeWhenSwitchingWillBeFinished)
BattleWeapon PickDroppedWeapon(BattleWeapon wpn)
```

### 3.4 事件系统

```csharp
event HealthEnergyChanging OnHealthChangedEvent   // 血量变化
event HealthEnergyChanging OnEnergyChangedEvent   // 护甲变化
event TeamChanged          OnTeamChangedEvent     // 队伍变化
event Action<BaseCombatUnit, ImpactType, byte> OnEffectStart   // 状态效果开始
event Action<BaseCombatUnit, ImpactType>       OnEffectRefresh // 状态效果刷新
event Action<BaseCombatUnit, ImpactType>       OnEffectEnd     // 状态效果结束
```

---

## 4. 武器系统与弹道计算

### 4.1 BattleWeapon — 武器主类

`BattleWeapon`（抽象类）是所有战斗武器的基类，通过静态工厂方法创建实例：

```csharp
// 四种创建途径
static BattleWeapon CreateBattleWeapon(...)                 // 玩家普通武器
static BattleWeapon CreateBattleWeaponForBattleUnit(...)    // 战斗单元的武器（炮塔/无人机）
static BattleWeapon CreateKillstreakBattleWeapon(...)       // 连杀奖励武器
static BattleWeapon CreateBattleWeapon(Dictionary<byte,object> data, ...) // 从网络数据恢复
```

#### 核心状态属性

```csharp
WeaponState CurrentState      // 当前武器状态（空闲/射击/换弹/切枪等）
WeaponState PrevState         // 上一个状态
int  ShotCount                // 累计射击次数
bool HasInvalidTarget         // 当前目标是否无效（用于自动炮塔）
bool IsWeaponImpliesSniperOverlay  // 是否使用狙击镜覆盖层
int  TimeWhenReloadingWillBeFinished  // 换弹完成时间戳（毫秒）
```

#### 关键操作

```csharp
Shot Fire(PlayerMotionState, Transform shotOrigin, out Ray originalRay)
Shot BotFire(PlayerMotionState, Transform shotOrigin, WeaponSpreadModification)
void StartReloading()
void ForceInterruptReloading()
bool InterruptReloading(short loadedAmmo, short reservedAmmo)
void Bolt(Action onComplete)              // 拉栓动作（如狙击枪）
bool NeedAutoReload()
bool CanFire() / CanFireBot()
bool CanReload()
bool HasAmmo()
void SetZoom(bool value) / SetZoomState(ZoomState)  // 开镜/关镜
float GetAimClampedSpeed()               // 瞄准速度限制
void AttachWeaponPart(WeaponPart.Type)   // 安装配件
void DetachWeaponPart(WeaponPart.Type)   // 拆卸配件
```

### 4.2 WeaponCalculator — 弹道计算器基类

所有具体武器的计算逻辑都从 `WeaponCalculator` 派生：

```
WeaponCalculator（基类）
├── AutoShotCalculator        — 自动射击（全自动步枪）
├── SniperCalculator          — 狙击枪（单发，拉栓延迟）
├── ShotgunCalculator         — 霰弹枪（8颗弹丸散射）
├── HandGunWithBoltCalculator — 带拉栓手枪
├── ColdWeaponCalculator      — 冷兵器（刀/近战）
├── RocketLauncherCalculator  — 火箭筒（直线火箭）
├── RocketHomingLauncherCalculator — 追踪火箭筒
├── GrenadeLauncherCalculator — 榴弹发射器（抛物线）
├── FlamerCalculator          — 喷火枪（近距离锥形检测）
├── AcidThrowerCalculator     — 酸液喷射器
└── SnowStormCalculator       — 冰晶风暴武器
```

#### 核心计算参数

```csharp
// 射线检测常量
const float CAST_CYLINDER        = 0.05f;  // 射线检测圆柱体半径
const float SCAN_CYLINDER        = 0.05f;  // 目标扫描圆柱体半径
const float HAND_WEAPON_SCAN_CYLINDER = 1.5f; // 冷兵器近战检测半径
const int   CAN_HIT_TARGET_RESET_DELAY = 150; // 命中目标判断重置延迟(ms)
const float MAX_DIST_MULTIPLIER  = 10f;   // 最大射程倍数

// Tag 标签
const string BATTLE_UNIT_TAG    = "BattleUnit";
const string PLAYER_COLLIDER_TAG= "PlayerCollider";
const string BROKEN_ITEM_TAG    = "BrokenItem";
```

#### ScanResult — 扫描结果结构体

```csharp
public struct ScanResult
{
    bool           HasTarget;         // 是否有目标
    float          DistanceToTarget;  // 到目标距离
    ShotTargetType TargetType;        // 目标类型
    int            TargetID;          // 目标 ID

    static ScanResult HasNoTarget;    // 无目标常量
}
```

#### 弹道计算内部流程

```
Fire(motionState, shotOrigin) 调用链：
  ↓
CalculateShotRay(motionState, shotOrigin, out originalRay)
  — 从摄像机视口中心投射射线
  — 应用 WeaponSpreadCalculator 计算散布偏移
  — 根据 PlayerMotionState（静止/行走/跑步/跳跃）调整散布
  ↓
CalculateShot(ray)
  — Cast(ray, out hit)  [Physics.SphereCast 圆柱体检测]
  — ShotTargetFromRaycastHit(hit)
      ├── StraightShotPlayer(hit) → 玩家命中
      └── StraightShotBattleUnit(hit) → 战斗单元命中
  — 若未命中：CreateMissingShot(ray) → 弹着地面
  ↓
返回 Shot 对象（含所有命中目标列表）
```

### 4.3 各类武器计算特点

#### 霰弹枪（ShotgunCalculator）

```csharp
const int SHOTGUN_BULLET_COUNT = 8;  // 每次射击发射 8 颗弹丸
const int DELAY_BETWEEN_SINGLE_AMMO_LOADING_TO_SHOOT_MSEC = 100; // 逐颗上弹延迟

// 支持逐颗上弹的特殊换弹逻辑
// 换弹过程可被射击打断（InterruptReloading）
// 每颗弹丸独立进行射线检测
```

#### 狙击枪（SniperCalculator）

```csharp
// 特点：
// 1. 有开镜检测（SetZoom → 影响散布）
// 2. 有目标延迟（targetTime），需稳定持续瞄准才可开枪
// 3. 单发换弹（拉栓：_shot / _isBoltNeededAfterReloading）
// 4. 机器人使用 CanFireBot() 判断开镜后是否可射
```

#### 冷兵器（ColdWeaponCalculator）

```csharp
const int MIN_DIFF_MSEC_BETWEEN_LAUNCHING_AND_SHOOTING = 50;  // 攻击出手最短间隔
const int MAKE_SHOT_ERROR_TO_RESET_KNIFE_STATE_MSEC = 200;    // 状态重置延迟
const int ADDITIONAL_VALUE_FOR_QUICK_SHOT_SHOT_DUR_MSEC = 300; // 快速攻击额外时长
const float MAX_ANGLE = 90f;  // 近战有效角度（前方90度扇形）

// 检测方式：SegmentShotV2(weaponDistance) — 线段检测而非射线
// 近战范围检测：ManualIsClose() 直接计算距离，不依赖物理射线
```

#### 火焰/酸液/雪暴（FlamerCalculator / AcidThrowerCalculator / SnowStormCalculator）

```csharp
// 共同特点：SegmentShot() — 近距离区段检测
// CanHitTarget(Ray, out RaycastHit, float distance, bool isBot)
// 不依赖散布偏移，而是锥形区域检测
```

#### 火箭筒（RocketLauncherCalculator）

```csharp
// RocketShot() 输出：
void RocketShot(Transform shotOriginTr,
    out Vector3 shotOrigin,    // 发射起点
    out Vector3 hitPoint,      // 预测落点
    out Vector3 hitNormal,     // 落点法线
    out Vector3 shotDir,       // 飞行方向
    out int landingTime)       // 预测到达时间戳
// 直线飞行，BattleUnit实体在服务器驱动下运动
```

#### 追踪火箭（RocketHomingLauncherCalculator）

```csharp
// RocketShot() 仅输出方向，追踪逻辑在 RocketHomingBattleUnit 中处理
void RocketShot(Transform shotOriginTr, out Vector3 shotOrigin, out Vector3 shotDir)
```

---

## 5. 命中判定流程

### 5.1 命中区域定义（PlayerHitZone）

```csharp
public enum PlayerHitZone : byte
{
    BASE       = 0,   // 躯干（基础伤害）
    NUTS       = 16,  // 腹部（特殊判定区）
    HEAD       = 32,  // 头部（爆头倍率）
    LEFT_HAND  = 48,  // 左手
    RIGHT_HAND = 64,  // 右手
    LEFT_LEG   = 80,  // 左腿
    RIGHT_LEG  = 96,  // 右腿
    EMPTY      = 97   // 无效/穿过
}
```

各部位对应不同的伤害倍率，HEAD（头部）为最高，四肢为最低。

### 5.2 碰撞体绑定

每个玩家角色的各身体部位都挂有 `BasePlayerBodyPart` 组件：

```csharp
// 第一人称（用于被射击检测）
class FirstLookPlayerBodyPart : BasePlayerBodyPart
{
    FirstLookBattlePlayer _player;   // 所属玩家引用
    PlayerHitZone _hitZone;          // 该碰撞体代表的命中区域
}

// 第三人称（第三人称视角被打时检测）
class ThirdLookPlayerBodyPart : BasePlayerBodyPart
{
    ThirdLookBattlePlayer _player;
    PlayerHitZone _hitZone;
}

// 俯视角（如俯视地图模式）
class TopDownPlayerBodyPart : BasePlayerBodyPart { ... }
```

战斗单元也有类似的碰撞体绑定：

```csharp
class BattleUnitHitCollider : MonoBehaviour
{
    Transform _root;               // 根节点
    BattleUnitViewBase _view;      // 视图对象
    PlayerHitZone _hitZone;        // 命中区域
}
```

### 5.3 射击目标类型（ShotTargetType）

```csharp
public enum ShotTargetType : byte
{
    NONE        = 0,
    PLAYER      = 1,   // 玩家
    PLAYER_ITEM = 2,   // 玩家物品（已废弃）
    GATES       = 4,   // 大门
    WALLS       = 5,   // 可破坏墙体
    BATTLE_UNIT = 6    // 战斗单元（炮塔等）
}
```

### 5.4 射击目标数据（ShotTarget）

```csharp
public sealed class ShotTarget : IShotTarget
{
    ShotTargetType TargetType;    // 目标类型
    short          TargetID;      // 目标唯一ID
    PlayerHitZone  HitZone;       // 命中区域
    int            HealthDamage;  // 对血量造成的伤害
    int            EnergyDamage;  // 对护甲造成的伤害
    Vector3        HitPoint;      // 世界空间命中点（用于特效）
    WallHitZone    WallHitZone;   // 墙面材质类型（用于弹孔特效）
}
```

### 5.5 墙面命中区域（WallHitZone）

不同材质表面会触发不同的弹孔/粒子特效：

```csharp
public enum WallHitZone : byte
{
    NONE   = 0,
    METAL  = 1,   // 金属 → 金属弹孔特效
    WOOD   = 2,   // 木材 → 木屑特效
    STONE  = 3,   // 石材 → 石块飞溅
    GRASS  = 4,   // 草地 → 泥土飞溅
    GLASS  = 5,   // 玻璃 → 碎裂特效
    FABRIC = 6,   // 布料 → 布料破损
    PLAST  = 7    // 塑料 → 塑料碎片
}
```

对应 `TrajectoryUtils` 中的 Tag 常量：
```csharp
"HitZoneMetal" / "HitZoneStone" / "HitZoneWood"
"HitZoneGrass" / "HitZoneGlass" / "HitZoneFabric" / "HitZonePlast"
```

### 5.6 命中判定完整流程图

```
玩家按下射击键
       ↓
BattleWeapon.Fire()
       ↓
WeaponCalculator.CalculateShotRay()
  ├── 从摄像机中心发出原始射线 (originalRay)
  ├── WeaponSpreadCalculator.CalculateSpread()
  │     — 根据 minAngle/maxAngle/increaseAngle 计算散布角度
  │     — 累加射击次数增加散布 (shotCount)
  │     — PlayerMotionState 修正（跑步 > 行走 > 静止/蹲下）
  └── 返回带散布偏移的最终射线
       ↓
WeaponCalculator.CalculateShot(ray)
       ↓
WeaponCalculator.Cast(ray, out hit)
  — Physics.SphereCast(radius=CAST_CYLINDER=0.05f)
  — LayerMask 过滤（只检测玩家碰撞层 + 地图碰撞层）
       ↓
  命中玩家碰撞体 ("PlayerCollider" tag)
       → StraightShotPlayer(hit)
           → hit.collider.GetComponent<BasePlayerBodyPart>()
           → 获取 HitZone / 所属 Player
           → 构建 ShotTarget{TargetType=PLAYER, HitZone, TargetID}

  命中战斗单元 ("BattleUnit" tag)
       → StraightShotBattleUnit(hit)
           → 构建 ShotTarget{TargetType=BATTLE_UNIT, HitZone, TargetID}

  命中地图墙面
       → CreateWallShotTarget(hit)
           → GetWallHitZoneByTag(hit.transform) → WallHitZone枚举
           → 构建 ShotTarget{TargetType=WALLS, WallHitZone}

  未命中任何目标
       → CreateMissingShot(ray)
           → 以最大射程终点创建 ShotTarget{TargetType=NONE}
       ↓
返回 Shot 对象（含所有 ShotTarget 的列表）
       ↓
发送给服务器验证 → 服务器广播伤害结果
```

### 5.7 ShootingUtils — 射击辅助判断

```csharp
public static class ShootingUtils
{
    // 根据Tag确定墙面材质类型
    static WallHitZone GetWallHitZoneByTag(Transform tr)

    // 根据战斗单元类型确定被打的材质类型
    static WallHitZone GetWallHitZoneForBattleUnit(BattleUnitType t)

    // 判断战斗单元是否可被射击（检查阵营关系）
    static bool CanBattleUnitBeShot(
        BattleUnitViewBase battleUnitView,
        IBattlePlayer calculatePlayer,
        IPlayersRelationshipsController playersRelationshipsCtrl)

    // 判断玩家是否可被射击（阵营关系 + 无敌状态）
    static bool CanPlayerBeShot(
        IBattlePlayer enemy,
        IBattlePlayer calculatedPlayer,
        IPlayersRelationshipsController playersRelationshipsCtrl)
}
```

---

## 6. 伤害计算与效果系统

### 6.1 KillerType — 击杀来源类型

```csharp
public enum KillerType
{
    None        = 0,
    Weapon      = 1,   // 武器（子弹/爆炸）
    BattleUnit  = 3,   // 战斗单元（炮塔/无人机）
    Impact      = 4,   // 持续状态效果（毒/火）
    GameMode    = 5,   // 游戏模式规则（如死亡区域）
    FallingDown = 6,   // 坠落伤害
    Cheating    = 7    // 反作弊击杀
}
```

### 6.2 ImpactType — 状态效果类型

```csharp
public enum ImpactType
{
    None      = 0,
    Fire      = 1,   // 燃烧（持续掉血）
    Blood     = 2,   // 流血
    Poison    = 3,   // 中毒
    Electro   = 4,   // 电击（减速/眩晕）
    Frost     = 5,   // 冰冻（移速降低）
    Biohazard = 6,   // 生化危机（类毒）
    Stunning  = 7,   // 眩晕
    Blindness = 8    // 致盲（黑屏/模糊）
}
```

### 6.3 BaseImpactController — 状态效果控制器

```csharp
public class BaseImpactController
{
    BaseCombatUnit Owner;                                    // 所属角色
    Dictionary<ImpactType, BaseImpactEffect> _effects;       // 当前激活的效果

    void UpdateEffect(ImpactType type, byte value)           // 激活/刷新某个效果
    void RemoveEffect(ImpactType type)                       // 移除某个效果
    void CustomUpdate()                                      // 每帧更新（持续伤害结算）

    // 子类重写
    protected virtual void OnActivateEffect(ImpactType type, byte value)    // 激活已有效果
    protected virtual void OnActivateNewEffect(ImpactType type, byte value) // 激活全新效果
    protected virtual void OnUpdateffect(ImpactType type)                   // 效果每帧更新
}
```

两个具体实现：
- **`FirstlLookImpactController`**：本地玩家（第一人称），仅处理视觉特效（血腥边框、毒雾等屏幕效果）
- **`ThirdlLookImpactController`**：第三人称玩家/网络玩家，还需处理骨骼抖动（`BoneShaker`）

```csharp
// ThirdlLookImpactController 额外能力
Dictionary<PlayerHitZone, BoneShaker> _impactParts;  // 各部位骨骼抖动器

void Hit(PlayerHitZone hitZone, Vector3 hitDirection) // 骨骼物理反馈

// 根据骨骼名字判断命中区域（物理碰撞回调）
static PlayerHitZone GetHitZone(string boneName)

// 效果类型 → 特效预制体名称映射
static string ImpactTypeToEffectName(ImpactType type)
```

### 6.4 BattlePlayerEffectImpactModification — 特效修饰

```csharp
public class BattlePlayerEffectImpactModification
{
    readonly string PooledEffectName;   // 替换用的特效池对象名
}
// 例如：某角色能力使普通击中特效变为火焰特效
```

### 6.5 BaseCombatUnit — 属性与事件

```csharp
public abstract class BaseCombatUnit : MonoBehaviour, ITargetPointsContainer
{
    bool  IsDead     { get; protected set; }
    short Team       { get; set; }           // 触发 OnTeamChangedEvent
    float? Health    { get; }                // null = 不显示血条
    float? MaxHealth { get; }
    float? Armor     { get; }
    float? MaxArmor  { get; }

    AssistTarget[] _targetPoints;    // 多个辅助目标点
    AssistTarget   _baseTargetPoint; // 基准目标点（身体中心）

    protected void DispatchOnHealthHasBeenChanged(bool changedByDamage)
    protected void DispatchOnEnergyHasBeenChanged(bool changedByDamage)
}
```

---

## 7. 抛物体轨迹系统

### 7.1 TrajectoryUtils — 轨迹计算工具

所有可投掷物（手雷、集束弹等）的飞行轨迹通过 `TrajectoryUtils` 预计算：

```csharp
// 旋转类型（投掷物飞行时的旋转方式）
public enum RotationType : byte
{
    Empty              = 0, // 不旋转
    RollOnGround       = 1, // 在地面滚动
    OneTurn            = 2, // 翻转一次
    ForwardToNextPos   = 3, // 朝向下一个路径点
    OneTurnToSpecificRot = 4, // 翻转到特定方向
    InfiniteRotation   = 5  // 无限旋转
}

// 物理模拟计算
static BattleUnitThrowTrajectory CalculateLikeOldGrenades(
    int startTimestamp, Vector3 launchPos, Vector3 dir,
    int simulationDurationMSec,
    float velocity,                      // 初速度
    float moveSpeedScale,                // 速度衰减
    float gravity,                       // 重力值
    float reflectVectorGravityPercent,   // 反弹后保留的重力分量比例
    RotationType rotationType,
    float rotationValue,
    bool stopOnHitWithHorizontalSurface, // 落地停止
    int collisionLayerMask
)

// 路径点优化（Douglas-Peucker算法，节省网络带宽）
static List<Vector3> SimplifyTrajectoryDouglasPeucker(List<Vector3> points, float threshold)
```

### 7.2 散布扩散系统（BattleUnitSpreadCalculator）

```csharp
public class BattleUnitSpreadCalculator
{
    // 按预设计算散布角度（随射击次数增大）
    float CalculateSpreadAngleByPreset(int shotCount, WeaponSpread weaponSpread)

    // 内部公式
    private float CalculateSpreadAngle(
        float minAngle,      // 最小散布角度
        float maxAngle,      // 最大散布角度
        float increaseAngle, // 每次射击增加的角度
        float shotCount      // 已射击次数
    )

    // 将散布角度转化为实际射线
    Ray CalculateRayAngular(float angle)
    private Ray CustomViewportPointToRay(Vector2 viewportPoint)
}
```

---

## 8. 网络同步机制

### 8.1 位置插值器

```csharp
// 双重线性插值（减少网络抖动）
class NetworkInterpolatorDoubleLerp  : INetworkInterpolator
// 四元数旋转插值（平滑旋转同步）
class NetworkInterpolatorQuaternion  : INetworkInterpolator
// 简单线性插值
class NetworkInterpolatorSimple      : INetworkInterpolator
```

### 8.2 同步管线

```
服务器广播 NetworkTransformData（位置+旋转+时间戳）
    ↓
MoveSnapshotsInterpolator 缓冲队列
    ↓
逐帧插值（Lerp/Slerp）
    ↓
驱动 BattleUnit / ThirdLookBattlePlayer 的 Transform
```

### 8.3 投掷物轨迹同步

```csharp
// 抛物线轨迹通过 Douglas-Peucker 压缩后发送给所有客户端
// 客户端在 NetworkThrowTrajectoryInterpolator 中重放轨迹
class NetworkThrowTrajectoryInterpolator : IInterpolatedUnitThrowTrajectory
```

### 8.4 ServerNetworkSyncMonitor

监控服务器网络响应延迟，用于调整状态效果超时时间（`_serverResponseCheckoutTime`），防止网络包丢失导致状态效果永久残留。

---

## 9. AI战斗控制

### 9.1 机器人射击接口

```csharp
// 机器人判断是否可以开枪
bool CheckIfCanHitTargetForBot(BotVisionManager botVisionManager)

// 机器人开枪（带散布修正）
Shot BotFire(PlayerMotionState motionState, Transform shotOrigin,
             WeaponSpreadModification spreadModification)

bool CanFireBot()  // 每种武器类型独立实现
```

### 9.2 行为树驱动（各模式 AI）

AI 行为由 `Resources/aibehaviourtrees/` 下的行为树文件驱动：

| 行为树目录 | 对应模式 |
|-----------|---------|
| `deathmatch/` | 自由乱斗 |
| `team_deathmatch/` | 团队死亡竞赛 |
| `bomb/` | 炸弹模式（进攻/防守） |
| `control_points/` | 控制点模式 |
| `gun_game/` | 枪械升级模式 |
| `money_bag/` | 金钱袋模式 |
| `common/` | 公共行为（寻路/补给/换弹） |

### 9.3 BattleUnitsManagerAI

继承自 `BattleUnitsManager`，重写 `MustUnitBeControlled()` 决定哪些单元由本地 AI 控制（而非服务器网络数据驱动）。

---

## 10. 关键枚举速查

### PlayerHitZone（命中区域）

| 值 | 名称 | 说明 |
|----|------|------|
| 0 | BASE | 躯干（基础伤害 ×1.0） |
| 16 | NUTS | 腹部 |
| 32 | HEAD | 头部（最高倍率） |
| 48 | LEFT_HAND | 左手（低倍率） |
| 64 | RIGHT_HAND | 右手（低倍率） |
| 80 | LEFT_LEG | 左腿（低倍率） |
| 96 | RIGHT_LEG | 右腿（低倍率） |
| 97 | EMPTY | 无效命中 |

### ImpactType（状态效果）

| 值 | 名称 | 效果描述 |
|----|------|---------|
| 0 | None | 无 |
| 1 | Fire | 燃烧（每秒掉血） |
| 2 | Blood | 流血 |
| 3 | Poison | 中毒 |
| 4 | Electro | 电击 |
| 5 | Frost | 冰冻（移速降低） |
| 6 | Biohazard | 生化毒素 |
| 7 | Stunning | 眩晕 |
| 8 | Blindness | 致盲 |

### WallHitZone（墙面材质）

| 值 | 材质 | 弹孔特效 |
|----|------|---------|
| 0 | NONE | 无 |
| 1 | METAL | 火星/金属弹孔 |
| 2 | WOOD | 木屑 |
| 3 | STONE | 石尘 |
| 4 | GRASS | 泥土 |
| 5 | GLASS | 碎玻璃 |
| 6 | FABRIC | 布料破损 |
| 7 | PLAST | 塑料碎裂 |

### KillerType（击杀来源）

| 值 | 类型 | 用途 |
|----|------|------|
| 0 | None | 未知 |
| 1 | Weapon | 玩家武器击杀 |
| 3 | BattleUnit | 战斗单元击杀（炮塔/无人机） |
| 4 | Impact | 状态效果致死 |
| 5 | GameMode | 规则击杀（死亡区域） |
| 6 | FallingDown | 坠落致死 |
| 7 | Cheating | 反作弊 |

### BattleUnitType（战斗单元类型，25种）

| 值 | 名称 | 分类 |
|----|------|------|
| 1 | Turret | 固定炮台（连杀奖励） |
| 2 | Helicopter | 直升机（连杀奖励） |
| 3 | Rocket | 直线火箭（连杀/技能） |
| 4 | Grenade | 手雷（投掷物） |
| 5 | RadioCar | 遥控车（探路/炸弹） |
| 6 | BombInBombGameMode | CS模式炸弹 |
| 7 | MGPlane | 机枪战机 |
| 8/9 | PlaneBombingLine/Projectile | 轰炸机+投弹 |
| 10/11 | RocketArtillery Launcher/Projectile | 火箭炮 |
| 12 | Dog | 军犬 |
| 13 | SupplyCase | 补给箱 |
| 14 | Shield | 可放置盾牌 |
| 17/18 | Copter/CopterFlamer | 无人机/喷火无人机 |
| 19 | RocketHoming | 追踪火箭 |
| 20 | Wall | 可放置掩体 |
| 21 | EnergyShield | 能量护盾 |
| 24 | TurretMovable | 可移动炮台 |
| 100 | DeathArea | 死亡区域 |

---

## 11. 架构关系图

```
┌─────────────────────────────────────────────────────────────┐
│                        BattleManager                        │
│  (游戏入口，持有所有子系统引用，驱动 CustomUpdate)            │
└────────────┬──────────────────────┬─────────────────────────┘
             │                      │
    ┌────────▼────────┐   ┌─────────▼──────────────────┐
    │ BattleUnitsManager │   │   IBattlePlayer (1..N)    │
    │  战斗单元管理器    │   │  FirstLookBattlePlayer    │
    │  Dictionary<short, │   │  ThirdLookBattlePlayer    │
    │  BattleUnitBase>   │   └───────────┬───────────────┘
    └────────┬───────────┘               │
             │                           │
    ┌────────▼─────────────────────────────────────────────┐
    │              BattleUnitBase（各类战斗单元）            │
    │  Turret / Rocket / Grenade / Helicopter / Dog ...     │
    │  └── BattleUnitViewBase（视图/渲染）                  │
    └───────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                       武器系统调用链                          │
│                                                             │
│  IBattlePlayer                                              │
│    └── IWeaponController                                    │
│          └── BattleWeapon（当前武器）                       │
│                └── WeaponCalculator（弹道算法）              │
│                      ├── CalculateShotRay()                 │
│                      │     ├── 摄像机中心射线               │
│                      │     └── WeaponSpreadCalculator 散布  │
│                      ├── CalculateShot(ray)                 │
│                      │     └── Physics.SphereCast()         │
│                      │           ├── PlayerCollider → HitZone│
│                      │           ├── BattleUnit → TargetID  │
│                      │           └── Wall → WallHitZone     │
│                      └── 返回 Shot{ ShotTarget[] }          │
│                                                             │
│    └── BaseImpactController                                 │
│          └── ImpactType 效果的激活/持续/停止               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    命中区域碰撞体层级                         │
│                                                             │
│  Character Prefab                                           │
│  └── Skeleton Root                                          │
│      ├── Head Bone     → PlayerBodyPart (HEAD=32)           │
│      ├── Chest Bone    → PlayerBodyPart (BASE=0)            │
│      ├── Abdomen Bone  → PlayerBodyPart (NUTS=16)           │
│      ├── LeftArm       → PlayerBodyPart (LEFT_HAND=48)      │
│      ├── RightArm      → PlayerBodyPart (RIGHT_HAND=64)     │
│      ├── LeftLeg       → PlayerBodyPart (LEFT_LEG=80)       │
│      └── RightLeg      → PlayerBodyPart (RIGHT_LEG=96)      │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    网络同步管线                               │
│                                                             │
│  Server                                                     │
│    └── BattleUnitShotData / NetworkTransformData            │
│          ↓ Photon Network                                   │
│  Client BattleUnitsManager.ProcessNewPositionAndRotation()  │
│    └── MoveSnapshotsInterpolator 缓冲队列                   │
│          └── NetworkInterpolatorDoubleLerp / Quaternion     │
│                └── Transform.position / rotation 更新       │
└─────────────────────────────────────────────────────────────┘
```

---

## 附录：关键文件路径索引

| 文件 | 相对路径 | 描述 |
|------|---------|------|
| `BattleUnitBase.cs` | `Battle/BattleUnits/` | 所有战斗单元基类 |
| `BattleUnitType.cs` | `Battle/BattleUnits/` | 单元类型枚举（25种） |
| `BattleUnitsManager.cs` | `Battle/BattleUnits/` | 单元生命周期管理 |
| `IBattlePlayer.cs` | `Battle/` | 玩家战斗接口 |
| `BaseCombatUnit.cs` | `Battle/` | 战斗实体基类（MonoBehaviour） |
| `BattleWeapon.cs` | `Battle/CombatControl/` | 武器主类（工厂/状态/开火） |
| `WeaponCalculator.cs` | `Battle/CombatControl/WeaponCalculation/` | 弹道计算基类 |
| `AutoShotCalculator.cs` | `Battle/CombatControl/WeaponCalculation/` | 自动步枪算法 |
| `SniperCalculator.cs` | `Battle/CombatControl/WeaponCalculation/` | 狙击枪算法 |
| `ShotgunCalculator.cs` | `Battle/CombatControl/WeaponCalculation/` | 霰弹枪（8弹丸）算法 |
| `ColdWeaponCalculator.cs` | `Battle/CombatControl/WeaponCalculation/` | 冷兵器（近战）算法 |
| `RocketLauncherCalculator.cs` | `Battle/CombatControl/WeaponCalculation/` | 火箭筒算法 |
| `FlamerCalculator.cs` | `Battle/CombatControl/WeaponCalculation/` | 喷火枪算法 |
| `BaseImpactController.cs` | `Battle/` | 状态效果控制器 |
| `ThirdlLookImpactController.cs` | `Battle/` | 第三人称骨骼碰撞反馈 |
| `TrajectoryUtils.cs` | `Battle/` | 弹道/轨迹工具（抛物线计算） |
| `PlayerHitZone.cs` | `Scripts/Assembly-CSharp/` | 命中区域枚举（7部位） |
| `ImpactType.cs` | `Scripts/Assembly-CSharp/` | 状态效果枚举（8种） |
| `WallHitZone.cs` | `Scripts/Assembly-CSharp/` | 墙面材质枚举（7种） |
| `ShotTarget.cs` | `Scripts/Assembly-CSharp/` | 射击目标数据类 |
| `ShootingUtils.cs` | `WeaponCalculation/` | 射击辅助判断（可射击性） |
| `BasePlayerBodyPart.cs` | `WeaponCalculation/` | 身体部位碰撞体基类 |
| `BattleUnitSpreadCalculator.cs` | `Battle/BattleUnits/` | 战斗单元散布计算 |

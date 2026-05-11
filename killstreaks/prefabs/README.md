# KillStreak Prefabs 提取说明

> **来源：** Modern Ops v9.79 AssetRipper 导出  
> **提取时间：** 2026-05-11  
> **总计：** 32 个 prefab 文件，分 4 个目录

---

## 目录结构

```
prefabs/
├── firstlook/      # 第一人称视角 — 使用者本地视图（含手持控制器）
├── thirdlook/      # 第三人称视角 — 其他玩家看到的 KS 图标/平板
├── menu/           # 商店/菜单展示模型（仅 MeshFilter + MeshRenderer）
└── resources/      # 运行时动态加载（Resources.Load 路径）
```

---

## firstlook/ — 第一人称视角 Prefab（10个）

使用者激活 KillStreak 时，本地玩家屏幕上看到的手持设备/触发动画。

| 文件 | 对应 KillStreak | 根节点组件 | 特殊说明 |
|------|----------------|-----------|---------|
| `Radar.prefab` | 雷达（Radar） | `RadarKillStreakViewController` + `FirstLookWearController` + `RenderController` | 标准第一人称视图 |
| `AntiRadar.prefab` | 反雷达（AntiRadar） | `RadarKillStreakViewController` + `FirstLookWearController` + `RenderController` | 同雷达类结构 |
| `Copter.prefab` | 无人机（Copter） | `RadarKillStreakViewController` + `FirstLookWearController` + `RenderController` | 手持平板召唤 |
| `CopterFlamer.prefab` | 火焰无人机（CopterFlamer） | `RadarKillStreakViewController` + `FirstLookWearController` + `RenderController` | 手持平板召唤 |
| `MGPlane.prefab` | 炮艇（MGPlane） | `RadarKillStreakViewController` + `FirstLookWearController` + `RenderController` | 手持平板召唤 |
| `RadioCar.prefab` | 遥控车（RadioCar） | `RadarKillStreakViewController` + `FirstLookWearController` + `RenderController` | 手持平板召唤 |
| `RocketArtillery.prefab` | 炮击（RocketArtillery） | `RadarKillStreakViewController` + `FirstLookWearController` + `RenderController` | 手持平板召唤 |
| `TR_GatlingTurret.prefab` | 炮塔（GatlingTurret） | `RadarKillStreakViewController` + `FirstLookWearController` + `RenderController` | 手持平板召唤 |
| `TR_GatlingTurretPlacer.prefab` | 炮塔放置器 | `PlacerObject` + `PlacerRaycastPoint`×3 + `PlacerBlockTrigger` | **特殊**：放置预览用，含3个射线检测点 |
| `Dog.prefab` | 军犬（Dog） | `FirstLookWearController` + `WhistleKillstreakController` | **特殊**：无平板，用口哨召唤；含完整CAT骨骼Rig |

### firstlook 通用 GameObject 层级

```
[KillstreakName]               ← 根节点，挂 RadarKillStreakViewController / WhistleKillstreakController
├── Shirts                     ← 上衣 WearSlot（皮肤插槽）
├── Pants                      ← 裤子 WearSlot（皮肤插槽）
├── CATRigPelvis               ← 骨骼：骨盆
│   └── CATRigSpine            ← 骨骼：脊椎
│       └── CATRigRibcage      ← 骨骼：胸腔
│           ├── CATRigLArm...  ← 左臂骨骼链
│           └── CATRigRArm...  ← 右臂骨骼链
└── [MeshRenderer 节点]        ← 手持设备网格（平板、无线电等）
```

### TR_GatlingTurretPlacer 特殊结构

```
TR_GatlingTurretPlacer          ← PlacerObject（放置逻辑）
├── PlaceRaycastPoint           ← PlacerRaycastPoint（中央检测点）
├── PlaceRaycastPoint           ← PlacerRaycastPoint（左侧检测点）
├── PlaceRaycastPoint           ← PlacerRaycastPoint（右侧检测点）
└── Turret_LOW                  ← PlacerBlockTrigger（碰撞检测/预览模型）
```

---

## thirdlook/ — 第三人称视角 Prefab（9个）

其他玩家看到的 KillStreak 使用图标/平板（通常是悬浮在玩家头顶/手中的设备）。  
所有 thirdlook prefab 共享**完全相同的 3 个脚本**：

| 组件 | 脚本 | 作用 |
|------|------|------|
| `KillstreakSettings` | `KillstreakSettings.cs` | KillStreak 数值配置（同步自服务器） |
| `RenderController` | `RenderController.cs` | LOD/渲染控制 |
| `ThirdLookTabletViewController` | `ThirdLookTabletViewController.cs` | 平板视图动画控制 |

| 文件 | 对应 KillStreak | 子节点 |
|------|----------------|--------|
| `Radar.prefab` | 雷达 | Dummy_delete（占位）+ Dummy_tablet（平板模型 LOD0/LOD1） |
| `AntiRadar.prefab` | 反雷达 | 同上 |
| `Copter.prefab` | 无人机 | 同上 |
| `CopterFlamer.prefab` | 火焰无人机 | 同上 |
| `MGPlane.prefab` | 炮艇 | 同上 |
| `RadioCar.prefab` | 遥控车 | 同上 |
| `RocketArtillery.prefab` | 炮击 | 同上 |
| `TR_GatlingTurret.prefab` | 炮塔 | 同上 |
| `Dog.prefab` | 军犬 | 同上 |

### thirdlook 通用 GameObject 层级

```
[KillstreakName]               ← 根节点，挂 KillstreakSettings + RenderController
├── Dummy_delete               ← 已废弃的占位节点（编辑器残留）
└── Dummy_tablet               ← ThirdLookTabletViewController
    ├── Tablet_lod_0           ← MeshFilter + MeshRenderer（近距离高精度）
    └── Tablet_lod_1           ← MeshFilter + MeshRenderer（远距离低精度）
```

---

## menu/ — 商店展示 Prefab（12个）

仅用于商店/背包界面的 3D 旋转展示，**无任何游戏逻辑脚本**。  
结构统一：根节点 `Transform` → 子节点 `Model`（MeshFilter + MeshRenderer）。

| 文件 | 对应 KillStreak 类型 |
|------|---------------------|
| `Radar.prefab` | 雷达 |
| `AntiRadar.prefab` | 反雷达 |
| `Dog.prefab` | 军犬 |
| `Copter.prefab` | 无人机 |
| `CopterFlamer.prefab` | 火焰无人机 |
| `RadioCar.prefab` | 遥控车 |
| `RocketArtillery.prefab` | 炮击 |
| `TR_GatlingTurret.prefab` | 加特林炮塔 |
| `FL_Flamethrower.prefab` | 火焰喷射器（武器型KS） |
| `GG_M134.prefab` | M134 加特林（武器型KS） |
| `HGWS_Berreta9MWS.prefab` | Berreta 9MWS（武器型KS） |
| `RL_RPG7.prefab` | RPG-7 火箭筒（武器型KS） |

> **注意：** menu prefab 中武器型 KS（FL_Flamethrower/GG_M134/HGWS_Berreta9MWS/RL_RPG7）**没有对应的 firstlook/thirdlook prefab**，因为武器型 KS 直接复用武器系统的 prefab（`Assets/prefabs/weapons/`）。

---

## resources/ — 运行时 Prefab（1个）

| 文件 | 加载路径 | 用途 |
|------|---------|------|
| `Dog.prefab` | `Resources/killstreaks/Dog` | **军犬 AI 单位**，通过 `Resources.Load("killstreaks/Dog")` 在战场上动态实例化；这是实际在场景中行走的军犬 3D 模型，与 firstlook/thirdlook 的视图 prefab 不同 |

---

## 关键脚本映射

| GUID | 脚本文件 | 职责 |
|------|---------|------|
| `2154d3cae0f8582766f9b56a7398bd75` | `RadarKillStreakViewController.cs` | 第一人称 KS 视图总控制器（平板系KS通用） |
| `cd46d95b11fcb93cc5539005dd0af2fc` | `FirstLookWearController.cs` | 第一人称手臂皮肤/装扮控制 |
| `32a605a4785634decd246de6230b34b5` | `WearSlot.cs` | 皮肤插槽组件 |
| `02254666fe44325ac971628c7bc36dcc` | `RenderController.cs` | LOD/渲染状态控制 |
| `84f94c9ae0d47a32f255d0b889626e4a` | `WhistleKillstreakController.cs` | 军犬口哨召唤控制器（Dog专用） |
| `62a30d1306782244677d2186d3e0393b` | `KillstreakSettings.cs` | thirdlook KS 数值配置 |
| `df0586122d8276e12adf57803e0c3032` | `ThirdLookTabletViewController.cs` | thirdlook 平板视图控制 |
| `f6d7f5ad9b0c0614b2cf7ad646994b21` | `PlacerObject.cs` | 炮塔放置系统：放置对象主控 |
| `b15b5a2273d5ec64bb201e1e7db47a5f` | `PlacerRaycastPoint.cs` | 炮塔放置：地面射线检测点 |
| `8be45fc54192f444d842517d6ec4f8aa` | `PlacerBlockTrigger.cs` | 炮塔放置：碰撞阻挡检测 |

---

## Prefab 与 KillStreak 类型对应

| KillStreak | firstlook | thirdlook | menu | resources |
|-----------|-----------|-----------|------|-----------|
| Radar（雷达） | ✅ | ✅ | ✅ | ❌ |
| AntiRadar（反雷达） | ✅ | ✅ | ✅ | ❌ |
| Dog（军犬） | ✅ | ✅ | ✅ | ✅ |
| Copter（无人机） | ✅ | ✅ | ✅ | ❌ |
| CopterFlamer（火焰无人机） | ✅ | ✅ | ✅ | ❌ |
| MGPlane（炮艇机） | ✅ | ✅ | ✅ | ❌ |
| RadioCar（遥控车） | ✅ | ✅ | ✅ | ❌ |
| RocketArtillery（炮击） | ✅ | ✅ | ✅ | ❌ |
| TR_GatlingTurret（炮塔） | ✅+Placer | ✅ | ✅ | ❌ |
| FL_Flamethrower（火焰喷射器） | ❌ | ❌ | ✅ | ❌ |
| GG_M134（加特林） | ❌ | ❌ | ✅ | ❌ |
| HGWS_Berreta9MWS（Berreta WS） | ❌ | ❌ | ✅ | ❌ |
| RL_RPG7（RPG7） | ❌ | ❌ | ✅ | ❌ |

> 武器型 KS（Flamethrower/M134/Berreta/RPG7）无专属 firstlook/thirdlook prefab，因为它们直接替换玩家武器槽，复用武器系统 prefab。

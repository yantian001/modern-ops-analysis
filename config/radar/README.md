# config/radar — 地图雷达/小地图配置

> Modern Ops v9.79 | 数据来源：`Assets/Resources/config/radar/` | 共 22 张地图

## 目录说明

本目录包含所有游戏地图的雷达（小地图）坐标配置。每个 JSON 文件定义了小地图图片在 3D 世界空间中的位置、旋转和缩放。

---

## 文件结构

所有文件格式相同：
```json
{
  "rmlbP": { "x": -70.0, "y": 0.0, "z": -70.0 },  // 左下角坐标 (Position)
  "rmlbR": { "x": 0.0,  "y": 0.0,  "z": 0.0  },   // 左下角旋转 (Rotation)
  "rmlbS": { "x": 1.0,  "y": 1.0,  "z": 1.0  },   // 左下角缩放 (Scale)
  "rmrtP": { "x": 70.0, "y": 0.0,  "z": 70.0 },   // 右上角坐标 (Position)
  "rmrtR": { "x": 0.0,  "y": 0.0,  "z": 0.0  },   // 右上角旋转 (Rotation)
  "rmrtS": { "x": 1.0,  "y": 1.0,  "z": 1.0  },   // 右上角缩放 (Scale)
  "sc":    1024                                     // 贴图尺寸（像素）
}
```

**字段说明：**
| 字段 | 含义 |
|------|------|
| `rmlbP/R/S` | Radar Map Left Bottom（小地图左下角）的 Position/Rotation/Scale |
| `rmrtP/R/S` | Radar Map Right Top（小地图右上角）的 Position/Rotation/Scale |
| `sc` | 小地图纹理分辨率（通常为 1024×1024） |

通过左下角和右上角坐标，可将玩家的世界坐标映射到小地图 UV 坐标。

---

## 地图文件列表

| 文件名 | 地图名称 | 地图范围（X轴） |
|--------|---------|---------------|
| [Bolywood.json](./Bolywood.json) | Bolywood（宝莱坞） | — |
| [Bomb_02_Ruzin.json](./Bomb_02_Ruzin.json) | Bomb 02 Ruzin（爆炸模式2） | — |
| [Building.json](./Building.json) | Building（建筑物） | — |
| [Building_.json](./Building_.json) | Building（变体） | — |
| [Building_02.json](./Building_02.json) | Building 02 | — |
| [Building_02_.json](./Building_02_.json) | Building 02（变体） | — |
| [CSGO_Aztec.json](./CSGO_Aztec.json) | CS:GO 阿兹特克 | — |
| [CSGO_DistillerDM.json](./CSGO_DistillerDM.json) | CS:GO Distiller（死亡竞赛） | — |
| [CSGO_PoolWay.json](./CSGO_PoolWay.json) | CS:GO Pool Way | — |
| [Election_day.json](./Election_day.json) | Election Day（选举日） | ±102.4 |
| [Fishing_spot.json](./Fishing_spot.json) | Fishing Spot（钓鱼点） | — |
| [Foundary_angar.json](./Foundary_angar.json) | Foundary Hangar（铸造厂机库） | — |
| [Foundry_day.json](./Foundry_day.json) | Foundry Day（铸造厂白天） | ±70.0 |
| [London.json](./London.json) | London（伦敦） | — |
| [Radar_Bomb.json](./Radar_Bomb.json) | Bomb Mode（炸弹模式） | — |
| [Redbarn.json](./Redbarn.json) | Red Barn（红仓库） | — |
| [Sawmill.json](./Sawmill.json) | Sawmill（锯木厂） | — |
| [Sawmill_Bomb.json](./Sawmill_Bomb.json) | Sawmill Bomb Mode | — |
| [Sawmill_Bomb_NEW.json](./Sawmill_Bomb_NEW.json) | Sawmill Bomb（新版） | — |
| [Sawmill_NEW.json](./Sawmill_NEW.json) | Sawmill（新版） | — |
| [Train_station.json](./Train_station.json) | Train Station（火车站） | — |
| [Train_station_.json](./Train_station_.json) | Train Station（变体） | — |

> 注：带 `_` 后缀的变体通常对应不同时间（白天/夜晚）或不同游戏模式（TDM/炸弹）的同一场景。

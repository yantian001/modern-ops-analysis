# config — 游戏配置数据总览

> Modern Ops: Gun Shooting Games v9.79 | 数据来源：`Assets/Resources/config/` | 总大小：~6.8 MB

## 目录结构

```
config/
├── crc/           # 游戏核心数据库（48个JSON，CRC缓存机制）
├── loccrc/        # 多语言本地化文本（45个JSON，I2Localization格式）
├── radar/         # 地图小地图坐标配置（22个JSON）
├── GameConfig.asset      # Unity全局游戏配置（FOV等）
├── VolumetricLightBeam.asset  # 体积光束渲染配置
└── rewards.json   # 战斗奖励类型与图标映射
```

---

## 子目录说明

### 📁 [crc/](./crc/) — 核心数据库
**48 个 JSON 文件** | 总计 ~1.4 MB（格式化后更大）

游戏所有核心数据，包括：武器（748条）、装扮（572条）、武器配件（3454条）、
成就（139条）、连杀奖励（76条）、赛季、活动等所有在线内容。

采用 CRC 校验机制进行版本管理，客户端启动时与服务器对比 `CRC.json` 中的校验和，
按需更新。详见 [crc/README.md](./crc/README.md)。

**最重要的文件：**
| 文件 | 内容 | 记录数 |
|------|------|-------|
| [crc/CRCD27.json](./crc/CRCD27.json) | 所有武器数据（含6档升级） | 748 |
| [crc/CRCD31.json](./crc/CRCD31.json) | 所有武器配件数据 | 3454 |
| [crc/CRCD28.json](./crc/CRCD28.json) | 所有装扮/皮肤数据 | 572 |
| [crc/CRCD32.json](./crc/CRCD32.json) | 武器升级路径 | 621 |
| [crc/CRCD4.json](./crc/CRCD4.json) | 技能/能力定义 | 723 |
| [crc/CRCD12.json](./crc/CRCD12.json) | 内购商品定义 | 374 |
| [crc/CRCD8.json](./crc/CRCD8.json) | 充值礼包定义 | 242 |
| [crc/CRCD13.json](./crc/CRCD13.json) | 连杀奖励定义 | 76 |

---

### 📁 [loccrc/](./loccrc/) — 多语言本地化
**45 个 JSON 文件** | 总计 ~5.3 MB

游戏全部 UI 文本的15种语言翻译，基于 I2 Localization 插件。
包含武器名称、装扮描述、成就说明、活动文案等所有玩家可见文字。

**语言支持：** 英文、俄文、德文、法文、西班牙文、葡萄牙文（巴西）、
简体中文、日文、韩文、泰文、意大利文、土耳其文、繁体中文、波兰文、印尼文

**最大文件：**
| 文件 | 内容 | 大小 |
|------|------|------|
| [loccrc/LOC_CRCD19.json](./loccrc/LOC_CRCD19.json) | 武器名称与描述（多语言） | 483.3 KB |
| [loccrc/LOC_CRCD1.json](./loccrc/LOC_CRCD1.json) | 技能/能力文本 | 423.1 KB |
| [loccrc/LOC_CRCD32.json](./loccrc/LOC_CRCD32.json) | 游戏活动文本 | 198.0 KB |
| [loccrc/LOC_CRCD10.json](./loccrc/LOC_CRCD10.json) | 装扮名称与描述 | 243.9 KB |

详见 [loccrc/README.md](./loccrc/README.md)。

---

### 📁 [radar/](./radar/) — 小地图配置
**22 个 JSON 文件** | 总计 ~17 KB

每张游戏地图的雷达小地图坐标配置，定义地图纹理在3D世界坐标中的映射范围。
包含 22 张地图（Foundry、Sawmill、London、Train Station 等）的配置。

详见 [radar/README.md](./radar/README.md)。

---

## 根目录文件

### GameConfig.asset
Unity MonoBehaviour 配置资产，包含：
```yaml
WeaponCameraFOV: 23.141   # 武器相机视野角（第一人称持枪FOV）
```

### VolumetricLightBeam.asset
Unity 体积光束（VolumetricLightBeam插件）全局渲染配置，控制场景中光线散射效果的参数。

### [rewards.json](./rewards.json)
战斗奖励类型与 UI 图标的映射表。定义了每种得分事件对应的奖励图标：

| rType | 奖励类型 |
|-------|---------|
| 0 | 无奖励 |
| 2 | Kill（击杀） |
| 3 | KillAssist（助攻） |
| 4 | Headshot（爆头） |
| 5 | Nutshot（下体命中） |
| 6 | FirstBlood（第一滴血） |
| 7 | Revenge（复仇） |
| 8 | Domination（统治） |
| 9 | GrenadeKill（手雷击杀） |

---

## 数据更新机制（CRC缓存系统）

```
启动
  │
  ▼
读取 Resources/config/crc/CRC.json
（本地内置校验和清单）
  │
  ▼
对比 PlayerPrefs["CRC"]
（上次从服务器缓存的校验和）
  │
  ├── 一致 ──→ 使用 PlayerPrefs["CRCD{N}"] 中的缓存数据
  │
  └── 不一致 ──→ HTTP请求服务器 GET /data/{N}
                    │
                    ▼
                接收新 JSON
                    │
                    ▼
                写入 PlayerPrefs["CRCD{N}"]
                更新 PlayerPrefs["CRC"]
                    │
                    ▼
                加载新数据到内存
```

**PlayerPrefs 存储路径：**
- Android: `/data/data/com.gamedevltd.modernops/shared_prefs/unity.com.gamedevltd.modernops.xml`
- iOS: `NSUserDefaults` plist

**本地化数据路径：**
- 同上，键名为 `"LOC_CRCD{N}"` 和 `"LOC_CRC"`

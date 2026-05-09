# armor — 防弹衣与头部护具数据库

> Modern Ops v9.79 | 数据来源：`CRCD28.json` (wt=1/2/8) + `CRCD33.json` (升级路径)
> 提取日期：2026-05-09

---

## 目录

| 文件 | 内容 | 条目数 |
|------|------|------|
| [body_armor.md](./body_armor.md) | 防弹衣（wt=8, Body Armor） | 8种 × 多档升级 = 83条 |
| [head_gear.md](./head_gear.md) | 头部护具（wt=2, Helmets & Masks） | 49种 × 多档升级 = 352条 |
| [spec_suits.md](./spec_suits.md) | 特种兵套装（wt=1, Spec Suits） | 6种 × 8档升级 = 48条 |

---

## 分类说明

### 防弹衣（Body Armor，wt=8）

游戏主要护甲类型，提供高护甲值和部分特殊效果。

| 名称 | 稀有度 | 基础购买 | 最高护甲 | 特色 |
|------|-------|---------|---------|------|
| Armor1Black | — | 金币 500 | 360 | 入门款，仅护甲+生命 |
| Armor6Green | 普通 | 金币 5000 | 440 | 中档，护甲+生命，需Lv.15 |
| Armor2Brown | 稀有 | 钻石 2000 | 480 | 护甲+生命+移速+爆炸防护 |
| Armor4Brown | 稀有 | 钻石 50000 | 250 | 特殊来源，手雷+爆炸防护 |
| Armor3Black | 传奇 | 无需购买 | 500 | 护甲+移速+手雷伤+狙击穿甲防护 |
| ArmorSupreme | 传奇 | 钻石 7000 | 700 | 高端，步枪/轻机枪/狙击穿甲防护，需Lv.24 |
| ArmorSupremeLight | 传奇 | 无需购买 | 670 | 移速型，步枪+冲锋枪穿甲防护，需Lv.24 |
| ArmorSupRed | 传奇 | 无需购买 | 850 | 最高护甲，步枪/霰弹/冲锋枪穿甲防护，需Lv.70 |

**穿甲防护说明：** `stPD{XX}` 字段减少受到对应武器穿甲子弹的额外伤害（穿甲弹比普通弹多的那部分伤害）。

---

### 头部护具（Helmets & Head Gear，wt=2）

49种头部装备，分为：
- **战术头盔**（helm_*, helmet_*, PoliceHelmet, TacticalHelmet 等）：提供护甲+生命
- **机动型头饰**（hood, cap 等）：提供生命+移速，后期可获护甲
- **面部防护**（gas_mask, SkullGas 等）：提供专项伤害防护
- **纯外观**（ski_googl_band, headph_glass, strik_mask 等）：无战斗属性

关键头部护具（满级属性）：

| 名称 | 最高护甲 | 最高生命 | 特色 |
|------|---------|---------|------|
| helmet_k63b_venom | 50 | 50 | 基础款战术头盔 |
| helm_bal_swat | 30 | 30 | SWAT头盔 |
| hood | 100 | 85 | 满级获得护甲，移速+手雷伤 |
| cap | 0 | 85 | 纯移速+生命型，最高移速+14%，含多种穿甲防护 |
| TacticalHelmet | 高 | — | 战术头盔系列 |
| Swat_Elite_Helm | 高 | — | 精英SWAT头盔 |
| PlagueDoc | — | — | 瘟疫博士面具 |

---

### 特种兵套装（Spec Suits，wt=1）

6种特种兵套装，每种8档升级。护甲值较低（最高65），但提供丰富的穿甲防护加成。
购买后可升级，升级会显著提升穿甲防护百分比。

| 套装 | 稀有度 | 解锁等级 | 特色 |
|------|-------|---------|------|
| Spec_Rookie | — | Lv.8 | 步枪+轻机枪穿甲防护，移速（入门款） |
| Spec_Sharpshooter | 稀有 | Lv.10 | 步枪+冲锋枪穿甲防护（专注步枪系）|
| Spec_Grenadier | 稀有 | Lv.10 | 额外手雷+爆炸防护+手雷伤加成（手雷专用）|
| Spec_Fortress | 稀有 | Lv.10 | 狙击+冲锋枪穿甲防护，移速加成（防狙击）|
| Spec_Equalizer | 稀有 | Lv.10 | 刀伤+手枪伤直接防护，霰弹+冲锋枪穿甲防护 |
| Spec_Alpha | 传奇 | Lv.10 | 护甲=生命值，爆头防护+爆头伤害，步枪+轻机枪穿甲防护（全面型）|

---

## 数据字段完整说明

### 来源文件
- `CRCD28.json` — 装扮数据库（含护甲/头盔/套装）
- `CRCD33.json` — 装扮升级路径（u_id=升级链ID, w_id=当前ID, rw_id=升级后ID, sc=升级费用）

### 字段映射表（JSON_Wear.cs → Wear.cs）

| JSON字段 | C# 属性名 | 含义 |
|---------|---------|------|
| `stArm` | Additional_Armor | 护甲值（叠加到最大护甲） |
| `stAHe` | Additional_Health | 额外生命值（叠加到最大血量） |
| `stMSp` | Additional_MovementSpeed | 移动速度 +X% |
| `stPHe` | Additional_HeadshotProtection | 爆头伤害减免 % |
| `stAGr` | Additional_Grenades | 额外携带手雷数 |
| `stPEx` | Additional_ExplosionProtection | 爆炸伤害减免 % |
| `stDHe` | Additional_HeadDamage | 爆头伤害加成 % |
| `stDGr` | Additional_GrenadeDamage | 手雷伤害加成 % |
| `stPen` | Additional_ArmorPenetration | 穿甲值加成 |
| `stPDa` | Additional_DamageProtection | 全面伤害保护 |
| `stD01` | Additional_KnifeProtection | 刀具伤害减免 % |
| `stD03` | Additional_HandGunProtection | 手枪伤害减免 % |
| `stD04` | Additional_MachineGunProtection | 突击步枪伤害减免 % |
| `stD06` | Additional_GatlingGunProtection | 轻机枪伤害减免 % |
| `stD07` | Additional_ShotgunProtection | 霰弹枪伤害减免 % |
| `stD10` | Additional_SniperRifleProtection | 狙击枪伤害减免 % |
| `stD17` | Additional_SubMachineGunProtection | 冲锋枪伤害减免 % |
| `stPD01` | (对应穿甲版) | 刀具穿甲伤害减免 % |
| `stPD03` | (对应穿甲版) | 手枪穿甲伤害减免 % |
| `stPD04` | (对应穿甲版) | 突击步枪穿甲伤害减免 % |
| `stPD06` | (对应穿甲版) | 轻机枪穿甲伤害减免 % |
| `stPD07` | (对应穿甲版) | 霰弹枪穿甲伤害减免 % |
| `stPD10` | (对应穿甲版) | 狙击枪穿甲伤害减免 % |
| `stPD11` | (对应穿甲版) | 特殊武器(SNOW_GUN)穿甲伤害减免 % |
| `stPD17` | (对应穿甲版) | 冲锋枪穿甲伤害减免 % |

### WeaponType 枚举（C#）

```csharp
public enum WeaponType : byte {
    ONE_HANDED_COLD_ARMS = 1,   // 单手刀具 (stD01/stPD01)
    HAND_GUN = 3,               // 手枪 (stD03/stPD03)
    MACHINE_GUN = 4,            // 突击步枪 (stD04/stPD04)
    GATLING_GUN = 6,            // 轻机枪 (stD06/stPD06)
    SHOT_GUN = 7,               // 霰弹枪 (stD07/stPD07)
    SNIPER_RIFLE = 10,          // 狙击枪 (stD10/stPD10)
    SNOW_GUN = 11,              // 特殊武器 (stPD11)
    SUB_MACHINE_GUN = 17,       // 冲锋枪 (stD17/stPD17)
}
```

### CCWearType 枚举（wt 字段对应）

```csharp
public enum CCWearType {
    Hats = 1,     // 特种兵套装（Spec_*）
    Masks = 2,    // 头部护具（头盔/面罩）
    Gloves = 3,   // 手套
    Shirts = 4,   // 上衣
    Pants = 5,    // 裤子
    Boots = 6,    // 靴子
    Backpacks = 7,// 背包
    Others = 8,   // 防弹衣（Armor*）
    Heads = 9     // 头部（预留）
}
```

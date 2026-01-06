# Echo Alchemist 属性系统代码分析

## 一、子母剑系统 (Flying Sword)

### 1. 核心机制
- **获取方式**：穿透(pierce)属性弹珠撞击穿透类型钉子时，有概率变异为子母剑
- **升级方式**：子母剑弹珠撞击子母剑钉子时，提升等级

### 2. 战斗行为
- **母剑**：带有 flying_sword 属性的主弹丸
- **子剑生成**：母剑击中敌人时，如果仍有穿透次数(piercesLeft)，在击中位置生成子剑
- **子剑行为**：
  1. 启动延迟（随机，增加视觉层次）
  2. 自动索敌（寻找最近活跃敌人）
  3. 冲刺打击（高速穿过敌人造成伤害）
  4. 连续攻击（次数由 multicast 决定）
  5. 回收/消失

### 3. 等级系统
| 等级 | 颜色 | 效果 |
|------|------|------|
| Lv.1 | 蓝色 #0ea5e9 | 基础追踪打击 |
| Lv.2 | 紫色 #6366f1 | 伤害与速度提升 |
| Lv.3 | 红色 #f43f5e | 最强攻击频率和破坏力 |

### 4. 属性交互
- **穿透 (Pierce)**：决定母剑能生成多少颗子剑
- **多重射击 (Multicast)**：决定每颗子剑能连续打击目标的次数
- **元素属性**：子剑可以继承母剑的元素属性（火/冰/雷），触发对应效果

### 5. 关键代码逻辑
```javascript
// 子剑生成条件
if (this.piercesLeft > 0) {
    if (this.config.flying_sword) {
        // 生成子剑
        const spawnX = e.pos.x + (Math.random()-0.5)*20;
        const spawnY = e.pos.y + (Math.random()-0.5)*20;
        // ...
    }
}

// 子剑伤害配置
flying_sword: {
    resonanceDamageMult: 0.5, // 共鸣额外伤害倍率
    recallDamageMult: 0.5,    // 回收伤害倍率
    dashDamageMult: 0.6,      // 穿透伤害倍率
    sonSwordDelayBase: 20,    // 子剑生成基础延迟
    sonSwordDelayMin: 2       // 子剑生成最小延迟
}
```

---

## 二、风属性系统 (Wind)

### 1. 核心机制：锚点 (Anchors)
- 风属性子弹击中目标或墙壁时生成**风之锚点**
- **最大数量**：场景中最多同时存在 4 个锚点
- **自动替换**：第 5 个锚点生成时，最早的锚点消失并触发小旋风爆炸（2倍子弹伤害）
- **法阵触发**：锚点数量达到 4 个时，自动判定几何形状并触发法阵

### 2. 法阵类型与触发条件

| 优先级 | 法阵类型 | 解锁等级 | 几何判定条件 | 效果 |
|--------|----------|----------|--------------|------|
| 1 | 蝴蝶法阵 (Butterfly) | Lv2 | 四边形存在自相交（交叉形） | 从屏幕外飞入大量风刃 |
| 2 | 风道 (Wind Tunnel) | Lv3 | 不相交，包围盒宽高比 ≥ 3.0 | 沿长轴方向的强风流光 |
| 3 | 风暴核心 (Storm Core) | Lv1 | 不相交，包围盒面积 < 6000 | 动态核心，子弹进入时充能 |
| 4 | 暴风绞杀 (Strangle) | Lv1 | 默认（形状较大且方正） | 风刃向中心旋转坍塌 |

### 3. 伤害公式（v2.5 重构）

#### 多段伤害公式
- **总打击次数** = `1 + scatter`（散射属性层数）
- **单次打击伤害** = `基础伤害 * (1 + multicast) / (1 + scatter)`
- **打击间隔**：每段 100ms

#### 属性联动加成
- **火旋风联动**：敌人燃烧状态时，单次打击伤害 x2
- **冰旋风联动**：敌人冰冻状态时，单次打击伤害 x2
- **黑洞进化**：风 + 激光 → 暴风绞杀进化为黑洞，造成 9999 即死伤害

### 4. 风暴核心特殊机制
- 平时保持 30% 透明度
- 子弹进入半径内时过渡至 100% 透明度
- 子弹在核心内时持续充能
- 能量满后释放大旋风（4倍伤害，分4段打击）
- 每回合能量衰减

### 5. 关键代码逻辑
```javascript
// 锚点生成
combat_wind_addAnchor(x, y, bulletDamage, bulletConfig) {
    if (this.windAnchors.length >= 4) {
        const old = this.windAnchors.shift();
        this.combat_wind_triggerSmallWhirlwindDamage(old.x, old.y, bulletDamage, bulletConfig);
    }
    this.windAnchors.push({ x, y, life: 1.0, bulletDamage, bulletConfig });
    if (this.windAnchors.length === 4) {
        this.combat_wind_triggerMagicCircle();
    }
}

// 法阵类型判定
if (this.isBowtieShape(this.windAnchors)) {
    if (windLevel >= 2) {
        this.combat_wind_triggerButterflyCircle(); // 蝴蝶法阵
        return;
    }
}
if (ratio >= 3.0 && windLevel >= 3) {
    type = 'tunnel'; // 风道
} else if (area < 6000) {
    type = 'storm_core'; // 风暴核心
} else {
    type = 'burst'; // 暴风绞杀
}
```

---

## 三、关键设计启示

### 1. 子母剑的"编程"本质
- 子剑不是简单的分裂，而是**独立的战斗单位**
- 子剑有自己的行为模式（索敌、冲刺、回收）
- 子剑的属性由母剑决定，但行为独立

### 2. 风属性的"几何触发"机制
- 通过玩家操控子弹落点，形成特定几何图形
- 不同几何形状触发不同效果
- 等级解锁更高级的几何判定

### 3. 属性联动设计
- 物理属性（pierce, scatter, multicast）影响数量/次数
- 元素属性（火/冰/雷）影响伤害类型和特效
- 高级属性（风/剑/激光）可以与元素属性叠加

### 4. 等级系统的作用
- 不是简单的数值提升
- 解锁新的机制（如蝴蝶法阵需要 Lv2，风道需要 Lv3）
- 视觉表现随等级变化

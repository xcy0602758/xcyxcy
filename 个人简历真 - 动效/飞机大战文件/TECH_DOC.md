# XCY 飞机大战 · 技术实现文档 v2.2

> 面向对象：Solo Builder（AI 或独立开发者）
> 目标：交付一个「双击即可玩」的竖屏弹幕射击网页小游戏
> 视觉风格：赛博科技风（深空底 + 霓虹发光 + Orbitron 字体）

---

## 0. 产品一句话

一款竖屏（9:16）的飞机大战弹幕射击游戏：玩家驾驶战机遇编队敌机、精英敌机与 Boss，通过升级火力、拾取道具、选择 Roguelike 技能芯片不断变强，冲击最高分。屏幕中央常驻大写「XCY」霓虹 Logo。

---

## 1. 技术选型（固定方案）

| 项目 | 选择 | 理由 |
|------|------|------|
| 游戏引擎 | **Phaser 3.90** | WebGL 渲染、内置 Arcade 物理、场景/粒子/补间齐全，最适合单文件小游戏 |
| 运行方式 | **单 HTML 文件 + 本地 phaser.min.js** | 无需 Node.js / 构建工具，双击 `index.html` 即可玩 |
| 语言 | **JavaScript（ES6 class）** | 无编译步骤，直接在浏览器运行 |
| 物理 | **Arcade Physics**（重力 0） | 轻量，子弹/碰撞检测足够用 |
| 数据持久化 | **localStorage** | 存最高分、已解锁战机 |
| 适配 | **Phaser.Scale.FIT + 9:16** | 自动缩放居中，手机/桌面通吃 |

> **约定游戏分辨率：720 × 1280（9:16）**

> 备选：若后续装了 Node.js，可迁移到 Vite + TypeScript，游戏逻辑无需重写（见 §11）。

---

## 2. 文件结构

```
飞机大战/
├── index.html              # 游戏唯一入口（含全部 CSS + JS）
├── phaser.min.js           # Phaser 3.90 本地副本（离线可玩）
├── TECH_DOC.md             # 本文档
└── assets/                 # 正式美术资源（替换占位精灵用）
    ├── player/
    │   ├── default.png     # 默认战机  → assets/player/default.png
    │   ├── lightning.png   # 闪电战机  → assets/player/lightning.png
    │   └── guardian.png    # 守护战机  → assets/player/guardian.png
    ├── enemy/              # 敌机（侦察/战斗/重型/精英）
    ├── boss/               # Boss 1 / Boss 2
    ├── bullet/             # 子弹贴图
    ├── powerup/            # 道具贴图
    └── fx/                 # 粒子、爆炸帧
```

> **占位阶段**：用 `generatePlaceholderSprites(scene)` 通过 Phaser Graphics 程序化生成几何贴图，先把逻辑跑通。美术替换见 §10。

---

## 3. 游戏设计总览

### 3.1 核心循环

```
开始菜单 → 战斗场景 → (波次推进) → 精英/Boss 战 → 通关 → Roguelike 三选一 → 下一关
                     ↘ 死亡 → 结算（最高分） → 重开
```

### 3.2 三架战机（可解锁 / 玩家已有素材）

| 战机 | 定位 | HP | 火力 | 移速 | 特殊能力 |
|------|------|----|----|----|----|
| 默认战机 `default` | 平衡 | 3 | 中 | 中 | 无明显短板 |
| 闪电战机 `lightning` | 高机动/速射 | 2 | 高(速射) | 快 | 快速射击 |
| 守护战机 `guardian` | 坦克 | 5 | 中(广域) | 慢 | 自带护盾/更高容错 |

**解锁条件**：默认战机开局可用；闪电战机累计击毁 100 敌机解锁；守护战机通关第 1 关（击败 Boss 1）解锁。解锁状态写入 localStorage。

> **双人模式战机分配**：双人模式下 P1 / P2 各选一架战机（同屏合作，不可重复）。默认 P1=默认战机、P2=闪电战机，可在选机界面手动切换。三张图正好覆盖：1P 一架 + 2P 两架。

### 3.3 敌机类型（难度递进）

| 类型 | 体积 | HP | 行为 | 分数 |
|------|------|----|----|----|
| 侦察机 Scout | 小 | 1 | 直线俯冲 | 100 |
| 战斗机 Fighter | 中 | 3 | 俯冲 + 散弹 | 300 |
| 重装机 Heavy | 大 | 8 | 慢速 + 密集弹幕 | 800 |
| 精英 Elite | 中 | 15 | 曲线机动 + 追踪弹 | 2000 |

### 3.4 Boss（2 个，P1 就实现第一个）

**Boss 1 「守护者 Guardian」**
- HP：800，体型巨大
- 阶段 1（HP>50%）：扇形散弹 + 两侧僚机激光
- 阶段 2（HP≤50%）：进入狂暴，弹幕变旋转螺旋阵 + 周期性召唤小怪
- 击败后：掉落多个道具，通关进入 Roguelike 选择

**Boss 2 「利维坦 Leviathan」**
- HP：2000，体型巨大
- 阶段 1：扇形 + 追踪弹幕
- 阶段 2：全屏弹幕矩阵 + 激光扫射
- 阶段 3：极度狂暴，弹幕加密 + 移动加速

> Boss 战是「创新玩法亮点」，第一版必须完整实现 Boss 1（含二阶段狂暴）。

### 3.5 Roguelike 技能芯片（击杀 Boss / 每关通关后三选一）

| 芯片 | 效果 |
|------|------|
| 火力强化 | 永久 +1 火力等级 |
| 攻速强化 | 射击间隔 -15% |
| 僚机 | 获得一架僚机，多一路输出 |
| 护盾 | 获得可抵挡 1 次伤害的护盾 |
| 弹幕分裂 | 子弹命中后分裂 2 颗 |
| 生命恢复 | 恢复 1 HP（满血则转化为护盾） |

### 3.6 双人模式（2P Co-op）

同屏竖屏合作，两名玩家共同抵御敌机，共享波次，各自独立 HP 与火力。

| 玩家 | 控制战机 | 移动按键 | 加速射击 |
|------|---------|---------|---------|
| P1 | 默认/闪电/守护（自选） | `W/S/A/D` | 按住 `SPACE` |
| P2 | 默认/闪电/守护（自选，不与 P1 重复） | `↑/←/↓/→` 方向键 | 按住 `ENTER`（或右 `Shift`） |

- **同屏规则**：两架战机在同一 720×1280 画布内自由活动，各自独立碰撞判定
- **HP 独立**：任一玩家 HP 归零即阵亡；两名玩家都阵亡才游戏结束
- **计分**：各自累计击毁分数，结算时分别显示并可比较
- **道具**：道具被哪名玩家碰到就作用于该玩家（火力/回血/护盾独立）
- **开始方式**：菜单选择「双人模式」卡片进入，进入前各自选战机

> 建议：双人模式在画布中央用一条半透明竖线，或给 P1/P2 机身加不同颜色的描边/光环（P1 青色、P2 粉色），便于区分。

### 3.7 操作说明（游戏内弹窗）

操作说明**不再常驻页面**，改为可随时呼出的游戏内弹窗，避免遮挡主界面与中央「XCY」Logo。

- **呼出方式**：菜单页点击「操作说明 / HOW TO PLAY」按钮；战斗中按 `H`（或 `F1`）
- **弹窗样式**：半透明深色遮罩 + 中央圆角面板，显示当前模式（单人/双人）对应的按键表
- **关闭方式**：点击「CLOSE」按钮 / 再按 `H` / 按 `ESC`
- **模式联动**：弹窗内容根据 `GameState.mode` 实时切换单人 / 双人按键表

**单人模式按键表**

| 动作 | 按键 / 操作 |
|------|----------|
| 移动 | `W/S/A/D` 或 方向键 |
| 跟随移动 | 鼠标 / 触屏指针（战机跟随指针） |
| 加速射击 | 按住 `SPACE` |
| 呼出操作说明 | `H` |
| 暂停 / 继续 | `ESC` |

**双人模式按键表**

| 动作 | P1（默认战机） | P2（另一架战机） |
|------|--------------|----------------|
| 移动 | `W/S/A/D` | `↑/←/↓/→` 方向键 |
| 加速射击 | 按住 `SPACE` | 按住 `ENTER` |
| 呼出操作说明 | `H`（双方共用） | — |
| 暂停 / 继续 | `ESC`（双方生效） | — |

> 实现：用 `HelpScene`（`scene.launch('HelpScene')` 叠加，不中断主场景）或场景内 `helpPanel` Container（遮罩 + 面板 + 文字）配合 `toggleHelp()` 显示/隐藏。

---

## 4. 场景（Scene）设计

| 场景 | 职责 |
|------|------|
| `BootScene` | 生成占位精灵 → 跳 MenuScene |
| `MenuScene` | 标题 / **模式选择（单人 / 双人）** / 战机选择 / 操作说明按钮（弹窗） / 开始按钮 / 最高分 |
| `HelpScene` | 操作说明弹窗（叠加场景，可随时呼出/关闭） |
| `GameScene` | 主战斗循环（敌人/子弹/碰撞/波次/Boss） |
| `BossScene` | Boss 战（可合并进 GameScene，标记 `bossActive`） |
| `UpgradeScene` | Roguelike 三选一 |
| `GameOverScene` | 结算界面（得分 / 最高分 / 双人分屏计分 + MVP）+ 重开 / 返回菜单 |

> 建议 MVP 先合并 BossScene 到 GameScene 内，减少场景切换复杂度。

### 4.1 结算界面（Game Over / Settlement）

游戏结束后弹出结算界面（`GameOverScene`），按模式展示不同内容。

**通用内容**
- 「GAME OVER」大标题（霓虹粉）
- 本局得分 `SCORE`
- 历史最高分 `HIGH SCORE`（破纪录时显示「★ NEW HIGH SCORE ★」金色闪烁）

**双人模式专属**
- 分屏两列展示：`P1 得分` vs `P2 得分`（各自战机图标 + 颜色区分：P1 青、P2 粉）
- 统计双方各自击毁数，标注「MVP = P1 / P2」（得分高者）
- 汇总合击总分 `TOTAL`

**操作按钮**
- `RESTART` 重开本模式
- `MAIN MENU` 返回菜单切换模式 / 战机

**进入方式**：单人 HP 归零、或双人两名玩家都阵亡时触发；Boss 通关进入 Roguelike 选择而非结算（除非中途阵亡）。

---

## 5. 数据结构与全局状态

```js
const GameState = {
  score: 0,
  highScore: parseInt(localStorage.getItem('xcy_high_score') || '0', 10),
  lives: 3,
  level: 1,          // 关卡
  wave: 0,           // 当前波次
  elapsed: 0,
  firepowerLevel: 0, // 火力等级 0-5
  energy: 0,         // 能量（大招用）
  bombs: 3,          // 清屏炸弹数
  gameOver: false,
  // 解锁状态
  unlocked: JSON.parse(localStorage.getItem('xcy_unlocked') || '{"default":true,"lightning":false,"guardian":false}'),
  selectedPlane: 'default',
  mode: 'single',     // 游戏模式：'single' 单人 / 'coop' 双人
  player1: 'default', // 双人模式 P1 战机
  player2: 'lightning', // 双人模式 P2 战机
};
```

### 战机属性表（玩家实例属性）

```js
player.fireRate;       // 射击间隔 ms（闪电战机更小）
player.fastFireRate;   // 按住空格/加速时更小
player.moveSpeed;      // 移动速度
player.lives;          // 当前血量（守护战机更多）
player.invincibleTimer;// 无敌时间（受击后闪烁）
player.lastFireTime;   // 上次射击时间戳
```

---

## 6. 核心系统实现要点

### 6.1 PlaceholderFactory（占位精灵生成）

用 `scene.add.graphics()` + `generateTexture()` 程序化生成几何贴图：
- `player`（青色三角战机）、`enemy_scout`、`enemy_fighter`、`enemy_heavy`、`enemy_elite`
- `bullet_normal`（金色圆）、`bullet_enemy`（粉红圆）
- `powerup_firepower`（金）、`powerup_heal`（绿）、`powerup_shield`（蓝）、`powerup_bomb`（橙）
- `bg_tile`（深空网格）、`particle`

### 6.2 玩家控制

- **键盘**：`W/S/A/D`（或方向键）移动，`SPACE` 加速射击，`ESC` 暂停
- **鼠标/触屏**：指针在画布上即跟随（飞机平滑插值向指针移动，无需按住）
- **统一输入**：`this.input.on('pointermove'/'pointerdown')` 切到鼠标模式，按 WASD 切回键盘模式

> ⚠️ 关键坑：按键用 `addKey(keyCode)` 逐个注册，不要用 `addKeys('W,A,S,D')`（返回大写键名易错）；子弹不要在 create 里调用不存在的 `setAllowGravity()`（全局 gravity 已为 0）。

### 6.3 子弹与对象池

- 玩家子弹：`this.physics.add.group()` 复用，双发平行向上，速度 -700
- 敌弹：向下 +300，越界销毁
- 避免每帧 `new Image`，子弹/敌机销毁后复用（`maxSize` 限制）

### 6.4 波次管理 WaveManager

```js
spawnInterval = 1200;   // 基础刷敌间隔
spawnWave() {
  this.waveIndex++;
  count = 3 + Math.floor(waveIndex / 3);  // 敌机数量随波次递增
  // 横向编队，从 y=-40 进入
}
```

后期波次引入重型/精英，Boss 波（`waveIndex % 10 === 0`）触发 Boss。

### 6.5 碰撞检测（4 组 overlap）

1. 玩家子弹 ↔ 敌机 → 敌机扣血，子弹销毁
2. 敌弹 ↔ 玩家 → 玩家受击（受击无敌帧）
3. 敌机 ↔ 玩家 → 双方受创
4. 玩家 ↔ 道具 → 拾取效果

### 6.6 Boss 系统

- `this.bossActive` 标记进入 Boss 战，清空普通敌机
- Boss 血条 UI 顶部常驻
- 阶段切换：HP 阈值判断，切换弹幕模式（扇形→螺旋→追踪）
- 狂暴状态：射击频率/移动速度提升

### 6.7 双人模式实现

- 维护 `this.players = [p1, p2]`，各自独立的 `HP`、`fireRate`、`moveSpeed`、无敌帧
- 输入分流：P1 读 WASD（`keyLeft/right/up/down`），P2 读方向键（`cursors`）
- 双倍子弹管理：`playerBullets1`、`playerBullets2` 两个 group
- 碰撞独立：敌弹/敌机分别与 p1、p2 做 `physics.overlap`
- 死亡判定：任一玩家阵亡后停止其输入与射击，两名都阵亡 → `GameOverScene`
- Boss 战支持双人：Boss 弹幕根据两名玩家位置分别生成/瞄准
- 菜单模式切换：读取 `GameState.mode`，选机界面据此显示 1 个或 2 个战机槽位，并切换操作说明

### 6.8 菜单页模式展示（直接展示不同模式）

菜单页顶部放两张模式卡片，进入时直接可见，点击切换：

- **单人模式卡片**：图标 1 架战机 + 文案「SOLO」
- **双人模式卡片**：图标 2 架战机 + 文案「CO-OP」
- 选中态高亮（霓虹描边），并联动下方战机槽位；操作说明独立为「HOW TO PLAY」弹窗按钮（§3.7）
- 单人模式只显示 1 个战机槽位；双人模式显示 2 个并排的槽位（默认战机 / 闪电战机，可点选切换）

---

## 7. UI / UX 设计（科技风）

- **配色**：深空底 `#0a0a1a`、主霓虹青 `#00e5ff`、强调霓虹粉 `#ff3366`、金色 `#ffd700`
- **字体**：标题用 Orbitron（备选 Courier New），正文 Rajdhani
- **屏幕中央「XCY」Logo**：128px，加粗，白色描边青色霓虹发光 + 呼吸动画，CSS 实现在 `#xcy-logo`，进入战斗后隐藏
- **HUD**：顶部固定条显示 `SCORE / HP(❤) / WAVE`
- **特效**：粒子尾焰（`startFollow` 跟随玩家）、命中闪烁、爆炸粒子、相机震动 `cameras.main.shake`

### 关键 CSS

```css
#xcy-logo {
  position: fixed; top: 50%; left: 50%;
  transform: translate(-50%, -50%);
  font-family: 'Orbitron', 'Courier New', monospace;
  font-size: 128px; font-weight: 900; color: #fff;
  -webkit-text-stroke: 2px #00e5ff;
  text-shadow: 0 0 10px #00e5ff, 0 0 30px #00e5ff, 0 0 60px #ff00ff;
  pointer-events: none; z-index: 9999;
  mix-blend-mode: screen;
  animation: breathe 2s ease-in-out infinite;
}
```

---

## 8. 开发优先级（MVP 计划）

### P0（第一版必须能玩）
- [ ] 单文件 index.html + 本地 phaser.min.js，双击可玩
- [ ] 菜单 + 战斗 + 结算三场景
- [ ] 玩家移动（键盘+鼠标）+ 自动射击
- [ ] 编队敌机 + 敌弹 + 4 组碰撞
- [ ] 火力升级道具 + 回血道具
- [ ] 分数 / HP / 波次 HUD
- [ ] 中央 XCY 霓虹 Logo
- [ ] 最高分 localStorage

### P1（创新玩法亮点，优先做）
- [ ] **Boss 1 守护者（二阶段狂暴）**
- [ ] 三架战机可解锁/切换
- [ ] 多敌机类型（侦察/战斗/重型/精英）
- [ ] Roguelike 技能芯片（通关三选一）
- [ ] 僚机系统
- [ ] 能量/炸弹必杀

### P2（打磨）
- [ ] Boss 2 利维坦（三阶段）
- [ ] 音效（BGM + 射击/爆炸/拾取）
- [ ] 弹幕类型扩展（螺旋/追踪/扇形）
- [ ] 粒子细节 / 屏幕震动
- [ ] 移动端触控优化

---

## 9. 运行方式

```bash
# 最简单：双击 index.html 即可（phaser.min.js 已本地化）

# 或本地起 HTTP 服务器（推荐，避免某些浏览器 file:// 限制）：
cd 飞机大战
python -m http.server 5173
# 浏览器打开 http://127.0.0.1:5173/index.html
```

---

## 10. 美术替换指南（占位 → 正式）

占位精灵通过 `generatePlaceholderSprites(scene)` 生成。替换正式美术时：

1. 将图片放入 `assets/` 对应目录
2. 在 `+BootScene.preload()` 加载：

```js
preload() {
  this.load.image('player_default', 'assets/player/default.png');
  this.load.image('player_lightning', 'assets/player/lightning.png');
  this.load.image('player_guardian', 'assets/player/guardian.png');
  this.load.image('enemy_scout', 'assets/enemy/scout.png');
  // ...
}
```

3. 把 `create()` 里 `add.image(x, y, 'player')` 改为对应 key（如 `player_default`）
4. **删除占位生成逻辑**（或保留 `getTexture` 判断：若已加载真图则跳过占位）

> 建议规格：精灵使用 **透明 PNG**；战机约 48×56px，Boss 256×256px，子弹 20×20px。若用序列帧动画，改为 `this.load.spritesheet()` + `anims.create()`。

---

## 11. 未来迁移 TypeScript（可选）

当前单文件 JS 已满足「双击即玩」。若后续装上 Node.js 需工程化：

- `npm create vite@latest` + 引入 Phaser 依赖
- 将 `GameScene`、`Player`、`Enemy`、`WaveManager`、`BulletPool` 拆成独立 `.ts` 模块
- 游戏逻辑不变，仅做模块化与类型标注

---

## 附：已确认需求清单（v2.0 决策）

1. Boss 精简为 2 个（守护者 + 利维坦）
2. Boss 战提前到 P1，第一版就要有该创新玩法
3. 先用占位精灵跑通逻辑，再替换正式美术
4. 屏幕中央大写「XCY」，霓虹发光 + Orbitron 科技字体
5. 竖屏 9:16（720×1280）
6. 玩家已有 3 张战机图：默认 / 闪电 / 守护，作为玩家可选战机
7. 新增双人模式（同屏合作），菜单页直接展示「单人 / 双人」模式卡片（§3.6 / §6.8）
8. 操作说明改为游戏内弹窗（HOW TO PLAY，`H` 键呼出，§3.7）；新增游戏结束结算界面（含双人分屏计分与 MVP，§4.1）
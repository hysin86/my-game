# 生存射击游戏 — Agent 分步构建指南

> 本文档写给 AI Agent（盘古 / DeepSeek），请按步骤逐一完成，每步都附验收标准。完成一步并自检通过后，再进入下一步。
>
> 游戏的完整设计规格见：[GAME_DESIGN_v2.md](GAME_DESIGN_v2.md)。本文档是"你要做什么"的行动清单，设计文档是"做成什么样"的规格书。两者共同作为开发依据。

---

## 通用规范（贯穿所有步骤）

### 代码组织

| 规则 | 说明 |
|---|---|
| 文件名 | 英文命名，文件名翻译成中文后能表达文件用途。例如：`player.c`（玩家）、`collision.c`（碰撞检测） |
| 注释 | 使用英文注释 |
| 单元测试 | 每个 `.c` 模块尽量配一个 `_test.c`（如 `player_test.c`）。推荐 C 测试框架：Unity 或 CUnit，由你自行选择。编译时 `make test` 应能运行全部测试 |
| 可调数值 | 所有可调参数（速度、冷却时间、生成速率公式、计分公式等）集中在 `src/config.h`，禁止硬编码在业务逻辑中 |
| C 标准 | C99 或 C11，由你决定，但需在 README 中注明 |
| 编译产物 | 最终产出为一个 `.wasm` 文件 + JS 胶水文件 + 承载的 HTML 页面，部署到 GitHub Pages 后可通过手机浏览器访问 |

### 提交要求

每完成一个步骤，提交一次 Git，commit message 格式：
```
step<N>: <英文简短描述>
```
例如：`step0: setup emscripten + sdl2 build environment`

---

## 步骤 0：环境搭建

### 你要做什么

在用户电脑上安装以下工具链，并完成"Hello World"级验证。

1. **Emscripten SDK（emsdk）**
   - 安装最新稳定版
   - 验证：`emcc --version` 正常输出
2. **SDL2 开发库**
   - 通过 Emscripten ports 安装：`embuilder build sdl2`
   - 验证：用 `emcc` 编译一个调用 `SDL_Init()` 的 C 文件不报错
3. **SDL2_mixer（音频）**
   - 通过 Emscripten ports 安装：`embuilder build sdl2_mixer`
   - 验证：用 `emcc` 编译一个调用 `Mix_OpenAudio()` 的 C 文件不报错
4. **编译验证脚本**
   - 在项目根目录创建 `build.sh`（或 Windows 下的 `build.bat`），后续用这个脚本一键编译

### 目录结构初始化

在完成环境搭建后，按以下结构创建空项目骨架：

```
my-game/
├── src/                 # C 源代码（暂时为空）
├── assets/              # 音效文件（暂时为空）
├── web/                 # 前端承载层
│   ├── index.html       # 页面骨架
│   ├── style.css        # 样式
│   ├── input.js         # 触控输入桥接
│   ├── storage.js       # localStorage 读写
│   ├── manifest.json    # PWA 配置
│   └── sw.js            # Service Worker
├── build.sh / build.bat # 编译脚本
├── Makefile             # 构建规则 + test 目标
├── GAME_DESIGN.md       # 旧版设计文档（已存在，不要动）
├── GAME_DESIGN_v2.md    # 新版设计文档（已存在，不要动）
└── BUILD_GUIDE.md       # 本文档（已存在，不要动）
```

### 验收标准

- [ ] `emcc --version` 正常
- [ ] 能编译链接 SDL2 和 SDL2_mixer
- [ ] 编译脚本可执行
- [ ] 目录结构完整

---

## 步骤 1：场地渲染 + 玩家 8 方向移动 + 边界限制

### 你要做什么

创建 `src/main.c`、`src/player.c`、`src/player.h`、`src/config.h`。

- **`config.h`**：定义游戏窗口尺寸（960×640 逻辑坐标）、玩家移动速度（220 px/s）、玩家碰撞半径（14px）等
- **`player.c` / `player.h`**：
  - 玩家结构体：位置（x, y）、碰撞半径、速度、无敌计时器、生命值
  - `player_init()`、`player_update(dt, input_x, input_y)`、`player_render(renderer)`
  - 移动逻辑：根据输入方向做 8 方向移动，斜向做速度归一化（用 `sqrt` 避免斜向更快）
  - 边界限制：玩家坐标不会超出 0~960 和 0~640
- **`main.c`**：
  - 初始化 SDL2、创建窗口和渲染器
  - 游戏主循环：接收 `input.js` 传来的摇杆方向，调用 `player_update()`，清屏 + 渲染玩家（先用纯色圆表示）

### JS 端（`web/input.js`）

创建左下角固定虚拟摇杆的 DOM 元素和触摸事件处理。将摇杆方向归一化为 (-1~1, -1~1) 传给 C 端。

### 验收标准

- [ ] Canvas 显示深色背景，中央有一个代表玩家的圆
- [ ] 触屏摇杆（或键盘 WASD）控制圆移动，8 方向均可
- [ ] 圆不会移出屏幕边界
- [ ] 斜向移动速度与正交移动一致

---

## 步骤 2：子弹系统

### 你要做什么

创建 `src/bullet.c`、`src/bullet.h`。

- 子弹结构体：位置（x, y）、方向向量（dx, dy，已归一化）、速度（500 px/s）、碰撞半径（5px）、是否存活标记
- `bullet_spawn(x, y, direction_x, direction_y)`：从玩家位置生成一颗子弹
- `bullet_update(dt)`：按方向和速度移动，飞出边界自动标记销毁
- `bullet_render(renderer)`：渲染子弹（小圆点）
- 发射冷却 0.25 秒（最快每秒 4 发），在 `config.h` 中定义
- 子弹方向：**朝玩家最后一次移动方向**。玩家静止且从未移动过时默认朝上方（0, -1）

### 验收标准

- [ ] 点击发射按钮（或空格键），子弹从玩家位置出现
- [ ] 子弹沿玩家最后一次移动方向飞行
- [ ] 连续发射受 0.25 秒冷却限制
- [ ] 子弹飞出场地边界后自动消失
- [ ] 停止移动期间发射，子弹朝最后一动方向（或默认上方）

---

## 步骤 3：怪物生成与追踪

### 你要做什么

创建 `src/monster.c`、`src/monster.h`、`src/spawn.c`、`src/spawn.h`。

- **怪物结构体**：位置（x, y）、速度（80~120 随机，px/s）、碰撞半径（14px）、存活标记
- **怪物行为**：`monster_update(dt, player_x, player_y)`，每帧计算指向玩家的方向向量并移动
- **怪物速度增长**：每过 60 秒，已有和新生成怪物的基础速度 ×1.1（设置上限 500 px/s）
- **生成器（Spawn Director）**：用累加器方式实现
  - 公式：`速率 = 1 + floor(存活秒数 / 10)` 只/秒
  - 生成位置：从场地四边（上 y=0 以上、下 y=640 以下、左 x=0 以左、右 x=960 以右）均匀随机

### 验收标准

- [ ] 怪物按时从四边出现并向玩家移动
- [ ] 前 10 秒约每秒 1 只，10~20 秒约每秒 2 只，以此类推
- [ ] 每过 1 分钟，怪物明显变快
- [ ] 怪物速度不超过 500 px/s

---

## 步骤 4：碰撞规则全部实现

### 你要做什么

创建 `src/collision.c`、`src/collision.h`。

所有碰撞体为圆，判定公式：**两圆心距离 < 半径之和**。每帧按以下顺序检测：

1. **子弹 × 怪物**
   - 命中 → 子弹销毁、怪物销毁、击杀数 +1、得分 +1
2. **怪物 × 怪物**
   - 相撞 → 两只都销毁，**不加分**
3. **怪物 × 玩家**
   - 玩家非无敌 → 怪物销毁、生命 -1、触发 3 秒无敌
   - 玩家无敌中 → 怪物销毁、不掉命、不加分

- 倒序遍历实体数组，避免删除时跳过元素（或用标记删除 + 统一清理）
- 玩家无敌期间，渲染时以 0.15 秒周期闪烁（半透明/全透明交替）

### 验收标准

- [ ] 子弹碰到怪物，两者同时消失，得分 +1
- [ ] 两只怪物相撞，同时消失，得分不变
- [ ] 怪物碰到玩家，生命 -1，怪物消失
- [ ] 受击后 3 秒内角色闪烁，再次被碰到不掉命且怪物消失
- [ ] 生命归零后游戏暂停（先不做结算，下两步做）

---

## 步骤 5：计分系统 + HUD + 最高分存档

### 你要做什么

创建 `src/score.c`、`src/score.h`。

- 计分公式：
  - 击杀得分 = 击杀数 × 1
  - 存活得分 = min(t, 180) × 1 + max(t - 180, 0) × 2
  - 总分 = 击杀得分 + 存活得分
- 每整秒结算一次存活分（避免浮点累加）
- HUD：在 Canvas 顶部渲染"得分：xxx  生命：♥×N  时间：MM:SS"
- JS 端（`web/storage.js`）：
  - `save_high_score(score)` → localStorage
  - `get_high_score()` → 返回存档值或 0
  - C 端通过回调接口读写

### JS ↔ C 接口补充

在前几步的基础上，完善以下回调：
```
C 端 → JS 端：
  - update_score(current_score)    // HUD 得分更新
  - update_lives(lives)            // HUD 生命更新
  - update_time(seconds)           // HUD 时间更新
  - game_over(final_score, time, kills)  // 游戏结束
```

### 验收标准

- [ ] HUD 实时显示得分、生命、存活时间
- [ ] 前 3 分钟每秒 +1，之后每秒 +2
- [ ] 击杀一只怪物分数 +1
- [ ] 游戏结束后最高分持久化，刷新页面不丢失

---

## 步骤 6：音效系统

### 你要做什么

创建 `src/audio.c`、`src/audio.h`。准备 5 个音效文件放到 `assets/` 目录。

| 文件 | 事件 | 说明 |
|---|---|---|
| `assets/shoot.wav` | 发射子弹 | 短促射击音 |
| `assets/hit.wav` | 子弹命中怪物 | 命中反馈 |
| `assets/crash.wav` | 怪物互撞 | 碰撞音 |
| `assets/hurt.wav` | 玩家受击 | 低音提示 |
| `assets/gameover.wav` | 生命耗尽 | 结束音 |

- 使用 SDL2_mixer 加载和播放
- 初始化：`Mix_OpenAudio(44100, MIX_DEFAULT_FORMAT, 2, 2048)`
- 播放：`Mix_PlayChannel(-1, chunk, 0)`
- 所有音效在游戏开始时预加载，避免播放时延迟
- 如果没有音效文件，请生成简单的合成音效（方波或正弦波即可）

### 验收标准

- [ ] 5 种事件分别触发对应音效
- [ ] 音效在手机浏览器中正常播放
- [ ] 无延迟或卡顿

---

## 步骤 7：游戏状态机 + 结束/重开流程

### 你要做什么

实现三态状态机：**主菜单 → 游戏中 → 游戏结束 → 主菜单**。

- **主菜单**：
  - 显示游戏标题"生存射击"
  - 显示历史最高分
  - 显示操作说明图（虚拟摇杆 + 发射按钮图示）
  - 「开始游戏」按钮
- **游戏中**：已在前面步骤实现，不变
- **游戏结束**：
  - 停止所有更新（怪物不再生成、实体不再移动）
  - 显示结算面板：本次得分、历史最高分、存活时间、击杀数
  - 「再来一局」按钮，点击后重置所有状态回到游戏中，不经过主菜单
  - 「返回菜单」按钮，点击后回到主菜单
- 状态切换时清理所有实体数组，避免状态残留

### 验收标准

- [ ] 打开页面 → 显示主菜单
- [ ] 点击开始 → 游戏正常运行
- [ ] 生命耗尽 → 显示结算面板，数据正确
- [ ] 结算面板"再来一局"→ 直接重开游戏
- [ ] 结算面板"返回菜单"→ 回到主菜单
- [ ] 反复"开始→死→重开"不崩溃、不残留旧实体

---

## 步骤 8：手机横屏适配 + PWA

### 你要做什么

修改 `web/index.html`、`web/style.css`、`web/manifest.json`、`web/sw.js`。

- **横屏锁定**：竖屏时覆盖一层提示"请旋转手机至横屏"，横屏后隐藏提示并显示游戏
- **Canvas 缩放**：游戏逻辑坐标 960×640，根据实际屏幕宽高等比例缩放 Canvas，居中显示，留黑边
- **禁用默认手势**：禁止页面滚动、双击缩放、长按弹出菜单（`touch-action: none`）
- **虚拟摇杆和发射按钮适配**：位置和尺寸按屏幕宽度的百分比计算
  - 摇杆：左下角，占屏幕宽度的 20%
  - 发射按钮：右下角，直径 ≥ 60 CSS px
- **manifest.json**：填写应用名、图标路径、`display: standalone`、`orientation: landscape`
- **Service Worker**：缓存关键文件，支持离线访问
- 生成一个 192×192 和 512×512 的 PNG 图标放到 `web/` 目录

### 验收标准

- [ ] 手机浏览器打开，竖屏时显示旋转提示
- [ ] 横屏后 Canvas 等比例缩放，不变形，居中显示
- [ ] 虚拟摇杆和发射按钮在手机上位置合适、触控灵敏
- [ ] 页面不滚动、不缩放
- [ ] 浏览器菜单"添加到主屏幕"后，桌面图标全屏启动
- [ ] 离线时仍可打开已缓存的页面

---

## 步骤 9：编译与 GitHub Pages 部署

### 你要做什么

- 完善 `Makefile` / `build.sh`：
  - 默认目标：编译 C → .wasm + .js 胶水文件，复制到 `web/` 目录
  - `make test`：编译并运行所有单元测试
  - `make clean`：清理构建产物
- Emscripten 编译选项要点：
  - `-s USE_SDL=2`
  - `-s USE_SDL_MIXER=2`
  - `-s ALLOW_MEMORY_GROWTH=1`
  - `-s EXPORTED_FUNCTIONS='["_main", ...]'`（按需导出 JS 接口函数）
  - `--preload-file assets/`（将音效文件打包进 .data 文件）
- 编译后验证 `web/` 目录下包含以下文件：
  - `index.html`
  - `style.css`
  - `input.js`
  - `storage.js`
  - `manifest.json`
  - `sw.js`
  - `index.js`（Emscripten 胶水文件）
  - `index.wasm`
  - `index.data`（音效等资源）

### 验收标准

- [ ] `build.sh` 一键编译成功
- [ ] `make test` 通过所有单元测试
- [ ] 本地 `python -m http.server`（或其他方式）启动后在手机浏览器可玩
- [ ] 所有文件已推送到 GitHub 仓库 hysin86/my-game
- [ ] 仓库 Settings → Pages 已启用（Source: main 分支，根目录或 `web/` 目录）
- [ ] GitHub Pages 生成的链接在手机浏览器可访问并正常游戏

---

## 步骤 10：平衡性试玩与数值微调

### 你要做什么

这一步不需要写新代码，而是**调整 `config.h` 中的数值**以达到以下体验目标：

- 3 分钟存活是"有挑战但可实现"的水平（中等玩家约 50% 概率打到 3 分钟）
- 游戏全程（0~5 分钟）在手机端稳定 60fps，无明显卡顿
- 前 30 秒压力不大，给玩家学习空间
- 3 分钟后生存分翻倍带来的成就感明显

### 可调旋钮（均在 config.h）

| 参数 | 默认值 | 调整方向 |
|---|---|---|
| `PLAYER_SPEED` | 220 | 太容易死就调高，太简单就调低 |
| `BULLET_SPEED` | 500 | 影响射击手感 |
| `FIRE_COOLDOWN` | 0.25 | 影响火力强度 |
| `MONSTER_SPEED_MIN/MAX` | 80/120 | 基础速度 |
| `MONSTER_SPEED_SCALE_INTERVAL` | 60 秒 | 速度增长间隔 |
| `MONSTER_SPEED_SCALE_FACTOR` | 1.1 | 每次速度倍率 |
| `MONSTER_SPEED_CAP` | 500 | 速度上限 |
| `SPAWN_RATE_FORMULA` | `1 + floor(t/10)` | 生成速率公式 |
| `INVINCIBLE_DURATION` | 3 秒 | 无敌时长 |
| `SURVIVAL_SCORE_THRESHOLD` | 180 秒 | 存活分翻倍节点 |

### 验收标准

- [ ] 试玩 5 局以上，3 分钟存活率约 50%
- [ ] 手机端全程 60fps 不卡顿
- [ ] 各个阶段（早期/中期/后期）都有可玩性
- [ ] 调整后的数值已提交并推送

---

## 附录 A：JS ↔ C 接口完整清单

以下是 C 端需要暴露给 JS 调用的函数，以及 C 端回调 JS 的函数签名。

### JS 调用 C（输入）

```c
// 虚拟摇杆方向，x/y 归一化为 -1.0 ~ 1.0，静止时为 (0, 0)
void set_joystick(float x, float y);

// 发射按钮按下（每次按下调用一次）
void trigger_fire(void);

// 通知 C 端 Canvas 的实际渲染尺寸，用于缩放计算
void set_canvas_size(int width, int height);
```

### C 回调 JS（输出）

```c
// 得分变化时调用（每击杀 +1、每整秒结算存活分时调用）
void update_score(int score);

// 生命值变化时调用
void update_lives(int lives);

// 存活时间变化（每秒触发，参数为秒数）
void update_time(int seconds);

// 游戏结束
void game_over(int final_score, int survival_seconds, int total_kills);
```

---

## 附录 B：文件清单与中文对照

| 文件 | 中文含义 |
|---|---|
| `src/main.c` | 游戏主入口 |
| `src/player.c` / `.h` | 玩家实体 |
| `src/bullet.c` / `.h` | 子弹实体 |
| `src/monster.c` / `.h` | 怪物实体 |
| `src/spawn.c` / `.h` | 怪物生成器 |
| `src/collision.c` / `.h` | 碰撞检测 |
| `src/score.c` / `.h` | 计分逻辑 |
| `src/audio.c` / `.h` | 音效管理 |
| `src/config.h` | 可调参数集中管理 |
| `assets/shoot.wav` | 发射音效 |
| `assets/hit.wav` | 命中音效 |
| `assets/crash.wav` | 怪物互撞音效 |
| `assets/hurt.wav` | 受击音效 |
| `assets/gameover.wav` | 游戏结束音效 |
| `web/index.html` | 页面骨架 |
| `web/style.css` | 手机适配样式 |
| `web/input.js` | 触控输入桥接 |
| `web/storage.js` | 最高分存储 |
| `web/manifest.json` | PWA 配置 |
| `web/sw.js` | 离线缓存 |
| `build.sh` / `build.bat` | 编译脚本 |
| `Makefile` | 构建规则 + 测试 |
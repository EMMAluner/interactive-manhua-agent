# P1 完整产品：本地 Web 产品化 README（P0 补充文档）

> 本文档是 P0 [`README.md`](./README.md) 的补充，**不重复** P0 已写的内容（环境要求、基础安装、7 Agent 流水线原理、P0 CLI 用法、schema 设计等请回看主 README）。
>
> 本文聚焦：P1 为什么这么做、怎么设计、怎么用。

---

## 目录

1. [产品定位与设计理念](#1-产品定位与设计理念)
2. [产品架构设计](#2-产品架构设计)
3. [P1 新增的技术方案](#3-p1-新增的技术方案)
4. [快速安装启动](#4-快速安装启动)
5. [素材 API 配置说明](#5-素材-api-配置说明)
6. [动静比例配置接口](#6-动静比例配置接口)
7. [导出格式与抖音互动空间对齐](#7-导出格式与抖音互动空间对齐)
8. [P1 新增目录结构](#8-p1-新增目录结构)

---

## 1. 产品定位与设计理念

### 1.1 P0 与 P1 是什么关系

P0 交付的是**引擎**：一条能把「小说 / 关键词」变成结构化 `game.json` 的 7-Agent LangGraph 流水线，CLI 触发、一次跑完、结果落盘。它证明了「生成能力可行」。

P1 交付的是**产品**：把 P0 引擎包裹成创作者能直接用的本地 Web 工具。P1 不改 P0 的生成语义，只解决 P0 直接落到用户面前时缺的四件事：

| P0 引擎的缺口 | P1 的补法 |
| --- | --- |
| 命令行门槛高，参数写错要重跑 | 步骤式向导 UI，参数图形化 |
| 一次跑完，中间出问题无法干预 | Planner 后暂停，人工审阅编辑分支再续跑 |
| 只有 JSON，创作者看不到可玩形态 | 内嵌 H5 播放器，即刻试玩 |
| 无法直接对接分发渠道 | 一键导出符合抖音互动空间要求的 zip 包 |

**一句话**：P0 让机器能写出来，P1 让人能创作、验证、交付。P0 CLI 完全保留，P1 不构成替换关系。

### 1.2 三条设计理念

**① 只包不改**：P1 复用 P0 的所有 Agent 节点函数（`node_parse` / `node_plan` / ... / `node_structure`），不重写生成逻辑；把「一次跑完」改造成「可暂停 / 可续跑」的分阶段驱动。这保证 P0 和 P1 出来的 `game.json` 语义完全一致，P0 测试用例继续绿。

**② 人机协作 > 全自动**：AI 生成剧情骨架很快但不总正确，让用户在**信息量最集中、修改成本最低**的节点（Planner 后）介入一次，胜过让用户读 21 个节点回头改。

**③ 无 Key 也能跑通**：素材 API 不填时**跳过而非报错**，前端回落到程序化占位底图，整个游戏依然完整可玩、可导出。这让创作者可以先用 dry-run 白嫖跑通全链路，再决定是否付费接图片/视频生成。

---

## 2. 产品架构设计

### 2.1 前后端分层

```
                         浏览器（http://127.0.0.1:8000）
                                       │
                       ┌───────────────┴───────────────┐
                       │      本地向导前端 · web/       │
                       │  纯 HTML+JS，无构建，后端托管  │
                       │  轮询进度 / HITL 编辑 / 预览    │
                       └───────────────┬───────────────┘
                                       │ REST + 静态
                       ┌───────────────┴───────────────┐
                       │ FastAPI 后端 · server/        │
                       │ 任务生命周期 / 后台线程 / 素材静态服务 │
                       └───────┬────────────┬──────────┘
                               │            │
             ┌─────────────────┴─┐      ┌───┴─────────────────┐
             │ 分阶段流水线执行器  │      │ 动静混合素材生成器    │
             │ pipeline_steps.py │      │ assets/generator.py │
             └─────────────────┬─┘      └───┬─────────────────┘
                               │            │
                               │  复用 P0    │
                               ▼            ▼
                       ┌────────────────────────┐
                       │  P0 LangGraph 节点函数   │
                       │  graph.py 的 node_*     │
                       └────────────────────────┘
```

分层原则：
- **前端**做交互与展示，不含业务逻辑，重启后端后前端刷新即可跟上。
- **后端**是薄薄一层，把 P0 引擎与素材生成器包成异步任务，向前端暴露 REST + 静态。
- **P0 引擎**不知道自己被网页调用，可维护性零成本。

### 2.2 HITL 节点为什么选在 Planner 后

7 个 Agent 里，可能出现的干预时机有很多，为什么只在 Planner 后设一个断点？

| 阶段 | 产物 | 用户能干预吗 | 干预 ROI |
| --- | --- | --- | --- |
| Parser | 世界观 / 人设 | 能，但产物短、错误少 | 低 |
| **Planner** | **分支骨架 + 节拍** | **能，产物结构化、决定后续所有内容** | **⭐️高** |
| Writer | 节点正文 | 能，但节点数量爆炸(21+)，改起来累 | 低 |
| Consistency | 报告 | 只读产物 | - |
| Gamification | 选项 / 道具 | 能，但依赖 Writer 结果 | 中 |
| Direction | 演出档位 | 能，但已接近终稿 | 低 |
| Structurer | 合并 | 机械合并，不需要审 | - |

Planner 后的骨架是**最少的信息量决定最多下游产物**的锁定点：`branches × beats` 大概 10-20 条文本，改一行就能让整个后半段剧情走向不同。这是产品设计上最经济的介入点。

编辑范围（前端 UI 直接暴露）：
- 分支名称（`label`）
- 冲突来源（`conflict_source`）
- 结局走向（`ending_direction`）
- 每条分支的节拍列表（`beats`）

变量、章节结构、选择点等**不开放编辑**（一旦改动会破坏下游 Writer / Gamification 的语义假设），保留为骨架的只读元数据。

### 2.3 动静混合决策机制

**问题**：漫剧节点数以十计，全走视频包体炸掉且成本高、全走静态图节奏平；工程上不能让用户逐节点手动选。

**方案**：把「视频还是静态图」做成 **打分-排序-截断** 的三步机制。

```
   每个 Node                              全体节点
      │                                     │
      ▼                                     ▼
  ┌─────────┐ ①打分  ┌──────────────┐ ②降序排 ┌──────────┐ ③取前 N ┌─────────┐
  │ Node 特征 │──────▶│ video_score  │────────▶│ ranked[] │─────────▶│ video 集 │
  └─────────┘         └──────────────┘         └──────────┘  N=ratio*│         │
                                                              总数    │ 剩余    │
                                                                      │ image  │
                                                                      └─────────┘
```

**video_score 的组成**（`assets/generator.py::_video_score`）：

| 特征 | 增/减分 | 依据 |
| --- | --- | --- |
| `emotion_intensity`（基础） | +0.0 ~ +1.0 | 情绪越强越适合动 |
| `type == "ending"` | +0.35 | 结局值得画面 |
| `type == "minigame"` | +0.30 | 交互瞬间 |
| 有选项 | +0.20 | 转折点 |
| 章节开场 / 结尾 | +0.20 / +0.20 | 幕间张力 |
| 内容含 打斗/告别/反转/决战/生死/离别/爆发/背叛/真相 | +0.25 | 高情绪关键词 |
| `staging == "full-video"` / `"semi-dynamic"` | +0.30 / +0.10 | 演出档位倾向 |
| `type == "system"` / `"status_change"` | -0.30 | 次要节点扣分 |
| `type == "dialogue"` / `"narration"` | -0.05 | 对话推进为主 |

**为什么用「打分 + 排序 + 截断」而不是「阈值判定」**：
- 阈值方案（例如 `score > 0.7 就视频`）无法保证比例——不同作品得分分布不同，实际视频率可能 5% 也可能 60%。
- 排序截断方案对 `video_ratio` 严格保证：`round(ratio × total_nodes)` 个节点做视频，可精确控成本、控包体。
- 变量比例只需改一个数：**只改 `video_ratio` 就能调整动静比例，不需要动代码**。

---

## 3. P1 新增的技术方案

### 3.1 FastAPI 本地服务

选型理由：本地单机工具，选 FastAPI 而非 Flask 因为：
- 天然 async（虽然 P1 没大量用，但未来素材生成可以并发）
- Pydantic v2 已是项目内主 schema 库，请求体自动校验零成本
- `StaticFiles` + `FileResponse` 直接托管前端与素材，不用另起 nginx

服务边界：**只监听 127.0.0.1**，无鉴权。这是本地创作工具的合理定位，暴露到公网需自行加反代 + 鉴权。

任务模型：**内存字典 + 后台线程**。设计选择：
- 内存字典而非 SQLite / Redis：单机工具，进程退出即结束，无跨会话诉求
- 线程而非 asyncio：P0 pipeline 是同步阻塞的，塞进事件循环会卡；扔到 `threading.Thread` 里跑最简单直接，前端只需轮询就能拿到状态
- 一把 `threading.Lock` 保护状态字典读写，避免读到中间态

### 3.2 流水线暂停 / 恢复机制

**关键决策：不使用 LangGraph 的 checkpointer**。

LangGraph 官方支持 checkpoint interrupt/resume，但需要引入外部存储、状态序列化，且和 P0 的 `_run_sequential` fallback 路径互不兼容。我们要的只是「跑到某个节点后停一下」，用不着完整快照。

采用方案：**手动驱动 P0 已有的节点函数**（`pipeline_steps.py::PipelineSession`）：

```python
# 阶段 A：跑到 Planner 后停
node_parse(state)       # 状态字典就是 P0 的 _State
node_plan(state)        # 停在这里，把 state["skeleton"] 交给前端

# 用户编辑，state["skeleton"] = 用户提交的 StorySkeleton

# 阶段 B：继续跑
for attempt in range(max_retry+1):
    node_write(state)
    node_check(state)
    if state["consistency"].passed: break
    node_rewrite_bump(state)
node_gamify(state)
node_direct(state)
node_structure(state)
```

好处：
- **和 P0 语义完全一致**：调的是同一批函数，走完出来的 `game.json` 和 P0 CLI 一样
- **零依赖**：不需要引入任何持久化组件
- **进度回调自然**：每调一次节点函数就上报一次状态，前端轮询立刻可见

### 3.3 素材生成策略

**核心原则：能不生成就不生成，能降级就降级**。素材生成是整条链路里唯一涉及外部付费 API 的环节，鲁棒性优先级最高。

策略分层：
1. **决策先行**：`decide_media_types()` 无论 API 是否配置都会跑，写回 `Node.media_type`。这保证前端预览与导出永远知道每个节点是"静态还是动态位"。
2. **API 就绪判断**：`AssetEndpoint.is_ready()` 检查 `api_key` 和 `model` 都非空。两个 API 都没就绪 → 整个生成阶段短路，`media_url` 全部留 `None`。
3. **单点失败隔离**：单个节点生成失败不影响其他节点，异常统一收进 `AssetResult.errors`。
4. **视频→图片自动回落**：video API 未配置或调用失败时，视频节点自动改用图片 API 生成；两个 API 都没时最终留空。
5. **占位不空场**：`media_url` 空时播放器渲染程序化 CSS 渐变 + emoji 占位图，视觉不塌，游戏依旧完整。

OpenAI 客户端健壮性小坑（**已在代码中处理**）：某些环境的 `no_proxy` 环境变量含 IPv6 CIDR（如 `fd00::/8`），会导致 httpx 解析代理时抛 `InvalidURL`。素材生成 API 通常直连，故我们给 OpenAI 客户端注入了 `trust_env=False` 的 httpx.Client，绕开这个环境陷阱。

### 3.4 H5 播放器与 zip 导出

**统一引擎**：`player/player.js` 是唯一的播放引擎，同时给两个消费者用：
- 预览：向导 UI 在预览步骤 mount 播放器，`assetBase='/api/jobs/{id}/'`，素材从后端拿
- 导出：zip 包里的 `index.html` 加载同一份 `player.js`，`assetBase=''`，素材同目录

这个复用是 P1 架构的关键红利——播放器改一次，两处生效，避免预览和导出行为不一致。

**数据注入方式**：导出包用 `game-data.js` 内联游戏数据（`window.__GAME__ = {...}`），**不用 fetch()**。原因：
- 双击本地 `index.html` 走 `file://` 协议，`fetch()` 会因 CORS 报错
- 抖音互动空间容器环境不保证 XHR 可用
- 用 `<script>` 加载 JS 变量最原始也最兼容

**包结构极简**：只有 5 类文件（`index.html` + `player.js/.css` + `game-data.js` + `assets/*`），无外链、无 CDN、无字体，全离线可玩。

**大小控制**：
- 每张图默认 512×512，PNG ~10-30KB，21 节点 ~500KB
- 视频节点默认按同尺寸出，如接 MP4 单个 200-500KB，视频节点通常 <5 个
- 播放器 + 引擎共 <25KB
- 典型全 dry-run 包 <15KB，含 21 图 <1MB，含视频 <5MB，均远低于 8MB 阈值

**超限告警**：`ExportResult.oversize` 为 `True` 时前端亮警告，用户可在配置里下调 `video_ratio` 或改小 `IMAGE_SIZE` 重新生成。

### 3.5 角色形象一致性约束（代码层面）

漫剧的观感底线是「同一个角色，在不同分镜/场景里长得一样」。P1 把推理侧最常用的一致性三件套沉淀成**角色资产表**，由新模块 `assets/character_art.py` 统一驱动，落在 `Character` 模型的三个新字段上：

| 字段 | 作用 | 一致性手段 |
| --- | --- | --- |
| `appearance_prompt` | 锁定的外貌/服装/发型固定描述词 | **锁定 prompt 模板**：首次生成时抽取一次并锁定，之后同角色所有立绘 + 节点场景图都复用，只替换视角/场景/动作词 |
| `ref_image` | 正面全身主图路径 | **参考图注入**：先出正面主图，再把它作为 reference image 传给同角色的侧面/表情立绘与节点场景图（Seedream @图片参考 / IP-Adapter 语义） |
| `art_seed` | 由角色名派生的稳定随机种子 | **固定 seed**：若图片 API 支持 `seed` 则每次生成都传入，进一步稳定形象 |

关键实现点：

- **生成顺序**：角色立绘（`generate_character_art`）先于节点场景图（`generate_assets`）执行——先建立参考图 / seed，节点场景图再复用。
- **注入通道**：seed 与参考图通过图片请求的 `extra_body`（`seed` + `image` data-uri）传入；`assets/generator.py` 的单图入口 `generate_single_image` 为两条链路共用。
- **鲁棒降级**：带 seed / 参考图的请求若被目标 API 拒绝，自动降级为普通生成重试一次；未配置图片 API 时仍会锁定 `appearance_prompt` 与 `art_seed`（只是不出图），配置后重跑即可产出一致立绘。

### 3.6 角色档案页（前端第 4.5 步）

在向导的「人工介入（HITL）」与「预览 & 导出」之间新增一步「角色档案」，把上面的角色资产表可视化：

- **展示**：每个角色的名字 + 身份（role）+ 外貌描述（persona / 锁定 prompt）+ seed + 已生成的多视角立绘（正面 / 侧面 / 表情×3）。
- **放大**：点击任意立绘弹出灯箱放大查看。
- **手动替换**：每个角色可选择视角并上传本地图片替换立绘（`POST /characters/{id}/replace`），替换后写回资产表并同步到预览 / 导出。
- **未配置占位**：没有配置图片 API 时，卡片显示「未生成立绘（未配置图片 API）」占位，并在页首提示仍已锁定 prompt/seed 资产表。

数据来源：`GET /api/jobs/{id}/characters` 返回结构化角色明细；生成阶段结束后，前端先跳到本页（`gotoStep(5)`），用户确认后再进入预览（`gotoStep(6)`）。

---

## 4. 快速安装启动

**基础安装参考** P0 [`README.md` 第 10 章](./README.md#10-安装与使用quick-start)。

**P1 只需两步增量**：

```bash
# 1. 装 P1 Web 依赖（extras 方式，只加 fastapi / uvicorn / python-multipart）
pip install -e ".[web]"
# 或： pip install fastapi>=0.110 uvicorn>=0.27 python-multipart>=0.0.9

# 2. 启动向导
manhua-agent-web                          # 默认 127.0.0.1:8000
manhua-agent-web --port 9000              # 换端口
manhua-agent-web --host 0.0.0.0 --port 8000 --reload   # 开发模式（谨慎）
# 等价：python -m manhua_agent.server
```

**打开浏览器**：http://127.0.0.1:8000

**验证**：
```bash
# 运行 P1 冒烟测试（无需任何 API Key）
python tests/p1_smoke_test.py
```

预期输出：
```
✔ 分阶段执行 + HITL 编辑 通过
✔ 动静混合决策（video_ratio 可控）通过
✔ 导出 zip（根目录含 index.html，≤8M）通过
✔ Web API 全链路（create→HITL→finish→export→download）通过
```

---

## 5. 素材 API 配置说明

P1 需要两个**独立**的 OpenAI-Compatible 端点：图片一个、视频一个。**都是可选项**：不填就跳过素材生成，游戏依旧可玩。

### 5.1 图片生成 API

在 `.env` 里配置 `IMAGE_*` 前缀：

```dotenv
IMAGE_API_KEY=sk-your-key            # 必填，才算"启用"
IMAGE_BASE_URL=https://api.openai.com/v1   # 可选，不填复用 LLM_BASE_URL
IMAGE_MODEL=dall-e-3                 # 必填，才算"启用"
IMAGE_SIZE=512x512                   # 默认 512x512
```

**协议约定**：POST `${IMAGE_BASE_URL}/images/generations`，请求体符合 OpenAI Images API：
```json
{ "model": "...", "prompt": "...", "size": "512x512", "n": 1, "response_format": "b64_json" }
```
响应中取 `data[0].b64_json` 落盘为 PNG；若只返回 `url` 也支持自动下载。

**已验证兼容**（协议一致就行，未做厂商特化）：
- OpenAI 官方 DALL·E / gpt-image
- 火山方舟 doubao-seedream 系列
- 智谱 cogview 系列（需按其 base_url 填）
- 任何暴露 OpenAI-Compatible images endpoint 的服务

### 5.2 视频生成 API

在 `.env` 里配置 `VIDEO_*` 前缀：

```dotenv
VIDEO_API_KEY=sk-your-key
VIDEO_BASE_URL=https://api.openai.com/v1
VIDEO_MODEL=doubao-seedance
VIDEO_SIZE=512x512
```

**协议约定**：POST `${VIDEO_BASE_URL}/videos/generations`，请求体：
```json
{ "model": "...", "prompt": "...", "size": "512x512", "n": 1 }
```
响应中取 `data[0].b64_json`（写 mp4）或 `data[0].url`（自动下载）。

**说明**：视频生成的行业标准接口尚未定型，P1 用了"最像 OpenAI"的形态。若你的视频服务参数结构不同，改 `assets/generator.py::_gen_video` 一个函数即可，其他环节不受影响。

### 5.3 生效判定与降级

```
┌───────────────────┬────────────────┬────────────────┬─────────────────┐
│ IMAGE 就绪？      │ VIDEO 就绪？   │ 视频节点做什么   │ 图片节点做什么    │
├───────────────────┼────────────────┼────────────────┼─────────────────┤
│ ✗                 │ ✗              │ 占位            │ 占位             │
│ ✓                 │ ✗              │ 回落图片         │ 图片             │
│ ✗                 │ ✓              │ 视频            │ 占位             │
│ ✓                 │ ✓              │ 视频            │ 图片             │
└───────────────────┴────────────────┴────────────────┴─────────────────┘
```

`GET /api/config` 会实时返回两个 API 的就绪状态，前端会用 🟢/⚪ 徽章标出。

---

## 6. 动静比例配置接口

`video_ratio` 是 P1 唯一的动静混合调节旋钮，值域 `[0.0, 1.0]`，含义是"视频节点数 / 总节点数"。默认 `0.2`（约 20% 节点动，80% 节点静）。

### 6.1 三种修改方式

**方式 A：全局默认（推荐用于长期稳定的生产配置）**
```dotenv
# .env
VIDEO_RATIO=0.15
```
在 `manhua-agent-web` 启动前生效，作用于所有新任务。

**方式 B：向导 UI 临时覆盖（推荐用于单次实验）**

步骤 2「配置参数」页有 `动态素材比例 video_ratio` 滑杆，改动只作用于当前任务，不写回 `.env`。

**方式 C：API 直接传参（推荐用于批量脚本）**

```bash
curl -X POST http://127.0.0.1:8000/api/jobs \
  -H "Content-Type: application/json" \
  -d '{"mode":"keywords","keywords":["宫斗"],"video_ratio":0.3,"dry_run":true}'
```

优先级：**API 请求参数 > 向导 UI 值 > `.env` VIDEO_RATIO > 内置默认 0.2**。

### 6.2 效果参考（21 节点作品实测）

| video_ratio | 视频节点数 | 静态节点数 | 典型包体（含真实素材） |
| --- | --- | --- | --- |
| 0.0 | 0 | 21 | ~500KB |
| 0.1 | 2 | 19 | ~1.5MB |
| **0.2（默认）** | 4 | 17 | ~3MB |
| 0.3 | 6 | 15 | ~4MB |
| 0.5 | 10 | 11 | ~6MB |
| 1.0 | 21 | 0 | 有超 8MB 风险 |

包体估算按图 20KB / 视频 500KB，视频服务实际大小差异较大，建议先小规模试跑再放量。

### 6.3 我不想改 ratio，但想让某些节点必出视频怎么办？

给节点设 `staging: "full-video"` 会 +0.30 分，或让节点 `emotion_intensity` 高一些（P0 演出 Agent 已按情绪打分）。硬要指定某节点必出视频的能力**目前不开放**——保持"排序截断"机制的比例可控性优先级最高。如需强制，可以在 `direction.py` 后加一个用户自定义的白名单钩子，属于二次开发范畴。

---

## 7. 导出格式与抖音互动空间对齐

### 7.1 导出包结构

```
your-game.zip                          （≤ 8MB）
├── index.html          ★根目录必须有，作为入口
├── player.css          播放器样式
├── player.js           播放器引擎
├── game-data.js        window.__GAME__ = { ...GameOutput... };
└── assets/
    ├── n_b_truth_01.png
    ├── n_b_truth_02.mp4
    └── ...             只包含 media_url 引用到的文件
```

### 7.2 抖音互动空间要求对照

抖音互动空间对上传包的关键约束（截止 2026 年 7 月）：

| 约束 | P1 对齐情况 |
| --- | --- |
| zip 包根目录含 `index.html` | ✅ `exporter.py` 强制打到根 |
| 包体 ≤ 8MB | ✅ 打包后 `ExportResult.oversize` 校验，超限告警 |
| 纯静态资源，无服务端依赖 | ✅ 全静态，`file://` 也能跑 |
| 数据 fetch 兼容性 | ✅ 用 `game-data.js` 内联而非 `fetch()` |
| 无外链依赖（字体 / CDN） | ✅ 无外链，播放器 + 素材全打包 |
| 移动端 9:16 布局 | ✅ 播放器根容器 `aspect-ratio: 9/16`，`max-width: 480px` |

### 7.3 导出流程

```
POST /api/jobs/{id}/export
  └─▶ 后端 zip 化落盘到 output/webruns/{id}/game.zip
       └─▶ 前端拿到 ExportResult 元数据（size / oversize / file_count）
            └─▶ 用户点「下载」按钮 → GET /api/jobs/{id}/download

    浏览器下载后：解压 → 双击 index.html → 本地即玩
              上传：直接拖入抖音互动空间控制台
```

### 7.4 本地验证游玩

导出 zip 后，最简单的验证方式：

```bash
unzip your-game.zip -d test_play/
cd test_play
# macOS
open index.html
# Linux
xdg-open index.html
# Windows
start index.html
```

浏览器直接打开，无需起本地服务器（因为我们用 `game-data.js` 内联规避了 fetch 问题）。

---

## 8. P1 新增目录结构

以下**只列 P1 新增或修改**的部分，P0 结构参见主 README 第 11 章。

```
interactive-manhua-agent/
│
├── README.md                        （已修改）新增 P1 章节
├── README_P1.md                     （新增）★ 本文件
├── pyproject.toml                   （已修改）加 [web] extras + manhua-agent-web 入口 + player 资源
├── requirements.txt                 （已修改）加 fastapi / uvicorn / python-multipart
├── .env.example                     （已修改）加 VIDEO_RATIO / IMAGE_* / VIDEO_*
│
├── web/                             ★ P1 新增 · 本地向导前端（纯 HTML+JS，无构建）
│   ├── index.html                   步骤式向导页面（含第 4.5 步「角色档案」面板 + 立绘灯箱）
│   ├── style.css                    向导样式（含角色卡片 / 灯箱样式）
│   └── app.js                       向导逻辑（步骤切换 / 轮询 / HITL / 角色档案 / 预览 / 导出）
│
├── output/webruns/                  P1 运行时目录（每个任务一个子目录）
│   └── {job_id}/
│       ├── assets/                  该任务生成的素材（预览与导出共用）
│       └── game.zip                 该任务导出的可玩 zip
│
├── tests/
│   ├── smoke_test.py                P0 原有测试（未改）
│   └── p1_smoke_test.py             ★ P1 新增冒烟测试（HITL / video_ratio / 导出 / Web API）
│
└── src/manhua_agent/
    │
    ├── config.py                    （已修改）加 AssetEndpoint / video_ratio / IMAGE_* / VIDEO_*
    ├── models.py                    （已修改）Node 加 media_type / media_url；Character 加 appearance_prompt / art_seed / art_views（角色资产表）
    │
    ├── pipeline_steps.py            ★ P1 新增 · 可暂停流水线执行器（HITL 核心）
    ├── exporter.py                  ★ P1 新增 · zip 导出器
    │
    ├── assets/                      ★ P1 新增 · 动静混合素材
    │   ├── __init__.py
    │   ├── generator.py             动静决策 + 图片/视频生成（节点场景图复用角色参考图/seed）
    │   └── character_art.py         ★ P1 新增 · 角色立绘 + 一致性资产表（锁定 prompt / 参考图 / seed）
    │
    ├── player/                      ★ P1 新增 · H5 播放器引擎（预览与导出共用）
    │   ├── __init__.py              `player_dir()` 定位资源目录
    │   ├── player.css               播放器样式
    │   ├── player.js                播放器引擎（条件 AST / HUD / 选项 / 小游戏 / 结局）
    │   └── index_template.html      导出包的 index.html 模板
    │
    └── server/                      ★ P1 新增 · 本地 FastAPI 服务
        ├── __init__.py
        ├── __main__.py              支持 python -m manhua_agent.server
        ├── app.py                   FastAPI 应用（路由 + 静态托管 + main 入口；新增 /characters 与 /characters/{id}/replace）
        └── jobs.py                  任务生命周期管理 + 后台线程（新增 generating_characters 阶段 + 角色档案 / 立绘替换）
```

**未列出的 P0 目录（`agents/` / `prompts/` / `graph.py` / `cli.py` / `llm.py` / `vectorstore.py`）** 均**未改动**，P0 行为完全保留。

---

## 相关文档

- 主 README（P0 产品设计 + 引擎原理 + CLI 用法）：[`README.md`](./README.md)
- 架构 FAQ：[`docs/architecture-faq.md`](./docs/architecture-faq.md)
- 输入 / 输出示例：[`examples/`](./examples/)
- P1 冒烟测试：[`tests/p1_smoke_test.py`](./tests/p1_smoke_test.py)

—— 陆子桐

# 架构说明 · Architecture

本文说明 `index.html` 的内部结构、分层职责与调用约定，以及 `docs/diagrams/` 中 4 张图的重生成方式。

---

## 1. 总体形态

整套应用是**一个 HTML 文件**，没有后端、没有构建、没有依赖：

```
index.html
├── <style>     设计系统（CSS 变量 + 组件样式 + 响应式）
├── <body>      静态骨架：侧边栏 / 顶栏 / 主区容器 / 移动端 Tab
└── <script>    应用代码（数据层 → 计算层 → 渲染层 → 事件层 → 启动）
```

运行时唯一的外部资源是浏览器自身的 `localStorage`。

### 为什么不做成框架项目

目标形态是「任何人双击就能打开、且十年后还能打开」。任何 CDN、构建产物或依赖锁都会破坏这一点。
代价是：需要手动维护渲染函数与 DOM 的同步 —— 这部分由下面第 4 节的调用约定来约束。

---

## 2. 分层结构

```
                    ┌─────────────────────────────┐
   用户操作  ──────▶ │  事件层（全局委托 onAction）  │
                    └──────────────┬──────────────┘
                                   │ 改 state
                    ┌──────────────▼──────────────┐
                    │  数据层（state / load / save） │  ← 唯一读写点
                    └──────────────┬──────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
      ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
      │  计算层        │   │  画布引擎      │   │  localStorage │
      │  逾期/焦点/日期 │   │  布局/缩放/导出 │   │  持久化        │
      └───────┬───────┘   └───────┬───────┘   └───────────────┘
              └───────────┬───────┘
                          ▼
                  ┌───────────────┐
                  │  渲染层 render* │  → 只写 DOM，不回写 state
                  └───────────────┘
```

### 各层职责

| 层 | 关键函数 | 职责 | 禁止事项 |
|---|---|---|---|
| **种子层** | `seedData` `RES_SEED` `SPRINT_SEED` `JDMAP_SEED` `STAGES` `COGNITION` | 提供首屏示例与静态内容配置 | 不读写 DOM |
| **数据层** | `loadData` `saveData` `flashSync` | 唯一的 state 读写与持久化点 | 不碰 DOM（除同步状态指示器） |
| **计算层** | `collectAlerts` `pickFocus` `dateAdd` `daysBetween` `fmtMD` `todayStr` | 纯函数：聚合、排序、日期运算 | 不写 state、不碰 DOM |
| **画布引擎** | `cv*` 系列（见下表） | 布局求解、尺寸度量、SVG 绘制与导出 | 不写 state（除节点位置） |
| **渲染层** | `renderNav` `renderTop` + 8 个视图渲染函数 | 读取 state，生成 HTML 写入容器 | **不得互调** |
| **调度层** | `refreshAll` | 唯一刷新入口，按固定顺序调用渲染层 | 不含业务逻辑 |
| **事件层** | `onAction` + `document` 上的全局委托 | 解析 `data-act`，修改 state，调用 `refreshAll` | 不直接操作 DOM |
| **启动层** | `boot` | 顺序：绑定 → 加载数据 → 初始化主题 → `refreshAll()` | —— |

---

## 3. 完整函数清单（67 个顶层函数）

<details>
<summary>展开</summary>

**种子与常量**
`seedData` `RES_SEED` `SPRINT_SEED` `JDMAP_SEED`

**数据层**
`loadData` `saveData` `flashSync`

**工具（纯函数，无依赖）**
`todayStr` `dateAdd` `daysBetween` `fmtMD` `esc` `uid` `nl2br`

**调度与导航**
`refreshAll` `go` `renderNav` `renderTop` `toast`

**计算层**
`collectAlerts` `pickFocus`

**渲染层（10）**
`renderNav` `renderTop` `renderToday` `renderKanban` `renderSprint` `renderPlan` `renderRes` `renderJdmap` `renderCanvas` `renderProfile`

**交互绑定**
`bindKanbanDnD` `cvBind` `cvEditText`

**画布引擎（12）**
`cvColor` `cvTextW` `cvWrap` `cvMetrics` `cvNodeById` `cvEdgesOf` `cvNodeSize` `cvPath` `cvPaint` `cvBuildSVG` `layoutAuto` `layoutFit`

**一键生成**
`genStageMap` `genFlow` `genFromPlan` `addNode`

**导出**
`download` `exportSVG` `exportPNG` `exportJSON` `importJSON`

**业务动作（事件层调用）**
`moveTarget` `toggleTarget` `doneTask` `delTask` `clearDoneTasks` `toggleSprint` `resetSprint` `cycleJd` `setTheme` `addTask` `addTarget` `toggleRes` `onAction`

**启动**
`boot`

</details>

---

## 4. 调用链约定（重要）

### 4.1 强制单向 —— DAG

调用关系必须构成**有向无环图**。这条规则不是洁癖，而是为了避免一类静态检查抓不到的真实 bug：

> `renderCalendar()` 内部调用 `renderTodayList()`，而 `renderTodayList()` 又调用 `renderCalendar()`
> → 无限递归 → 栈溢出。语法完全正确，运行时才崩。

**正确做法**：所有联动通过唯一入口调度。

```js
// ✅ 唯一刷新入口，按固定顺序调用，各渲染函数互不感知
function refreshAll(){
  renderNav();
  renderTop();
  if(VIEW==='today')        renderToday();
  else if(VIEW==='kanban')  renderKanban();
  else if(VIEW==='sprint')  renderSprint();
  else if(VIEW==='plan')    renderPlan();
  else if(VIEW==='res')     renderRes();
  else if(VIEW==='jdmap')   renderJdmap();
  else if(VIEW==='canvas')  renderCanvas();
  else if(VIEW==='profile') renderProfile();
}

// ✅ 事件 → 改数据 → 统一刷新
function toggleSprint(id){
  const it = state.sprint.find(x=>x.id===id);
  it.done = !it.done;
  saveData();
  refreshAll();
}

// ❌ 禁止：渲染函数互相调用
function renderCanvas(){ /* ... */ renderToday(); }  // ← 会形成环
```

### 4.2 事件委托

全部交互走 `document` 上的一次委托，避免为动态生成的元素重复绑定：

```js
document.addEventListener('click', e=>{
  const btn = e.target.closest('[data-act]');
  if(btn) onAction(btn.dataset.act, btn, e);
  const nv = e.target.closest('.nav button[data-view], .mobtab button[data-view]');
  if(nv) go(nv.dataset.view);
});
```

`onAction` 是一个 `switch`，把 `data-act` 映射到具体动作函数。新增功能时：
在 HTML 里写 `data-act="yourAction"` → 在 `onAction` 里加一个 `case` → 实现动作函数（内部改 state 后调 `refreshAll()`）。

### 4.3 启动顺序

```js
function boot(){
  // 1) 绑定一次性事件（导出 / 导入 / 主题 / 全局委托）
  // 2) loadData()：读 localStorage，缺失或损坏时注入 seedData()
  // 3) 恢复主题偏好
  // 4) refreshAll()：一次性渲染全部模块
}
document.addEventListener('DOMContentLoaded', boot);
```

**所有 `addEventListener` 都在 DOM 就绪后执行**；首屏渲染由 `refreshAll()` 一次性完成，渲染函数内部不再触发其他渲染。

---

## 5. 画布引擎

「无限画布」是本项目最复杂的部分，独立成一层。

| 函数 | 职责 |
|---|---|
| `cvTextW(s, size)` | 文本宽度估算：CJK 按 `1em`、拉丁按 `0.55em` —— 用于节点自适应与换行 |
| `cvWrap(text, maxW, size)` | 按实测宽度折行（不是按字符数） |
| `cvMetrics(node)` | 计算节点的最终行数与尺寸 |
| `cvNodeSize(node)` | 输出 `{w, h}`，供布局与命中检测复用 |
| `layoutAuto()` | **拓扑分层布局**：Kahn 拓扑排序分层 → 同层按父节点分列 → 叶子成块排布 |
| `layoutFit()` | 计算包围盒并按容器缩放居中，设最小缩放下限保证可读 |
| `cvPath(a,b)` | 生成两点之间的折线路径 |
| `cvPaint()` | 重绘画布：网点背景、连线、节点、缩放提示、节点/连线计数 |
| `cvBuildSVG()` | 把当前画布导出为**独立可用的 SVG 字符串**（脱离页面样式） |
| `genStageMap / genFlow / genFromPlan` | 三个一键生成器，产出 nodes + edges 后交给 `layoutAuto()` |
| `cvBind / cvEditText` | 平移、滚轮缩放、节点拖拽吸附、双击改字、连线模式 |

### 已修复的两个关键坑

1. **分层算法回推父节点**
   早期实现沿边反向遍历，会把根节点从第 0 层推到第 3 层，导致 **17 组节点互相压叠**。
   修复：改为**单向** Kahn 拓扑分层，并让同层节点按父节点分列。现在重叠数恒为 0。

2. **长文本溢出卡片**
   早期按字符数估算宽度，对中文严重低估（中文 ≈ 2× 拉丁宽度）。
   修复：改用 `cvTextW` 实测宽度 + 自动折行。现在截断数恒为 0。

另外 `layoutFit()` 设有**最小缩放下限**——宁可溢出滚动，也不要缩到看不清。

---

## 6. 设计系统

### CSS 变量分层

```css
:root{ /* 与主题无关：字体栈、圆角、缓动 */ }

body[data-theme="dark"]{  /* 暗色（默认）：背景 / 文字 / 边框 / 语义色 / 画布专用色 */ }
body[data-theme="light"]{ /* 亮色：同名字段覆盖 */ }
```

所有颜色都走变量，**不在组件内硬编码**。画布专用变量（`--cv-dot` `--cv-edge` `--cv-leaf` `--cv-hint` `--cv-shadow`）单独分组，
因为 SVG 需要按主题切换描边与填充，且不能用 CSS 类继承（导出 SVG 时需内联）。

### 语义色

| 变量 | 用途 |
|---|---|
| `--acc` | 唯一强调色，只用于「现在该做什么」 |
| `--bad` | 逾期 / 危险 |
| `--warn` | 需注意 / 待补 |
| `--ok` | 已完成 / 已具备 |
| `--info` | 中性提示 |
| `--purple` | 分类标记 |

### 响应式断点

| 断点 | 变化 |
|---|---|
| `>1080px` | 侧边栏常驻 + 主区多列 |
| `768–1080px` | 侧边栏收窄 |
| `<768px` | 侧边栏隐藏 → 底部 Tab；卡片单列；`.tbl` 表格转卡片（`td[data-l]::before` 显示字段名） |

移动端细节：按钮 ≥44px、输入框字号 ≥16px（避免 iOS 聚焦时自动放大）、
`padding-bottom: env(safe-area-inset-bottom)` 适配 iPhone 底部安全区。

---

## 7. 架构图重生成

`docs/diagrams/*.html` 由 [archify](https://github.com/tt-a1i/archify)（MIT）从 `docs/diagrams/src/*.json` 规格生成。
规格是纯 JSON（组件 / 阶段 / 泳道 / 连线 / 卡片），可读可改。

```bash
# 1) 取到 archify（若你已安装，直接用其 bin/archify.mjs）
git clone https://github.com/tt-a1i/archify.git /path/to/archify

# 2) 校验（showcase 质量档要求 0 error / 0 warning）
cd /path/to/archify
node bin/archify.mjs validate architecture docs/diagrams/src/architecture.json --quality showcase --json
node bin/archify.mjs validate dataflow     docs/diagrams/src/dataflow.json     --quality showcase --json
node bin/archify.mjs validate sequence     docs/diagrams/src/sequence.json     --quality showcase --json
node bin/archify.mjs validate workflow     docs/diagrams/src/workflow.json     --quality showcase --json

# 3) 生成 HTML
node bin/archify.mjs deliver architecture docs/diagrams/src/architecture.json docs/diagrams/architecture.html --quality showcase --json
node bin/archify.mjs deliver dataflow     docs/diagrams/src/dataflow.json     docs/diagrams/dataflow.html     --quality showcase --json
node bin/archify.mjs deliver sequence     docs/diagrams/src/sequence.json     docs/diagrams/sequence.html     --quality showcase --json
node bin/archify.mjs deliver workflow     docs/diagrams/src/workflow.json     docs/diagrams/workflow.html     --quality showcase --json
```

### 改规格时的经验

archify 的校验器会给出**带几何建议的诊断**（例如「标签压住节点，建议 `labelAt [x,y]`」）。踩过的点：

- **同一节点多条出边的标签会打架**：共用的水平残段 + 各自竖直走廊，标签几乎必然相撞。
  解法：给分支加 `channelX` 把竖直走廊错开，再用 `labelAt` 把标签放到自己的走廊上。
- **`fromSide` 必须与 `via` 配合**：单独指定 `fromSide` 会被路由器忽略并报 `endpoint-side-direction`。
- **可读性有下限**：`viewBox` 太宽会导致缩放后字号投影低于 6px（`composition/desktop-readability`）。
  解法：收窄 `viewBox` 或缩短节点副标题 —— 别指望靠放大解决。
- **时序图纵向很挤**：消息间距有 28px 下限，可用高度决定了消息条数上限。宁可减少消息，不要压间距。

---

## 8. 已知边界与扩展点

| 方向 | 现状 | 扩展方式 |
|---|---|---|
| 云端同步 | 仅本地 `localStorage` | 在 `loadData` / `saveData` 内接入远端读写；成功后再更新 UI 同步状态 |
| 多用户 / 账号 | 无（单用户本地优先） | 需引入鉴权与远端存储，属于破坏「零依赖」约束的设计变更 |
| 通知 / 提醒 | 无系统推送 | 靠「今日要处理」置顶区等价于提醒；如需推送需引入 Service Worker |
| 数据导入格式 | 仅本应用的 JSON | 可在 `importJSON` 中加 CSV / Markdown 解析分支 |
| 导图格式 | SVG / PNG | 需 XMind / OPML 支持时，在 `exportSVG` 同层新增导出器 |
| 国际化 | 中文单语 | 文案集中在 `VIEWS` / `STAGES` / 各 `*_SEED`，可抽成字典后按 `meta.locale` 切换 |

---

## 9. 自检清单

改完代码后按这个清单走一遍，有一条不过就别提交：

1. **调用链无环** —— 画出函数调用关系，确认无 `A→B→A`
2. **初始化链通畅** —— 从 `DOMContentLoaded` 出发，确认每个被调函数都已定义且在调用点之前
3. **DOM 存在性** —— 所有 `$('#id')` 的目标元素在静态 HTML 或已渲染的视图中存在
4. **空数据不崩** —— `localStorage` 为空时种子数据正确填充；清空示例后页面不报错
5. **日期边界** —— 跨月（1/31→2/1）、跨年（12/31→1/1）计算正确
6. **事件绑定时机** —— 所有 `addEventListener` 在目标元素渲染完成后执行
7. **变量作用域** —— 无全局命名冲突，循环内闭包变量正确捕获
8. **主题双向切换** —— 暗 ↔ 亮切换后，画布 SVG、内联样式、表格都跟随变化
9. **移动端** —— 窄屏下无横向溢出，表格转卡片正常
10. **规模闸门** —— 新增模块后仍能一屏看清；模块过多时先分期，不要一次堆满

### 无头回归测试

```bash
# 语法检查
node --check <(python -c "
import re,sys
t=open('index.html',encoding='utf-8').read()
sys.stdout.write(re.findall(r'<script>(.*?)</script>',t,re.S)[0])
")
```

在浏览器里打开页面后，可在控制台快速自测：

```js
// 数据完整性
state.targets.length + state.tasks.length + state.resources.length + state.sprint.length + state.jdmap.length

// 调用链无环（粗检：反复切换视图不应报错）
['today','kanban','sprint','plan','res','jdmap','canvas','profile']
  .forEach(v => { go(v); });
```

---

## 10. 图表索引

| 图 | 规格 | 产物 | 说明 |
|---|---|---|---|
| 运行时架构 | `src/architecture.json` | `architecture.html` | 分层、边界、唯一读写点 |
| 数据流 | `src/dataflow.json` | `dataflow.html` | 从一次点击到落盘 |
| 调用时序 | `src/sequence.json` | `sequence.html` | 启动 + 一次交互 |
| 使用流程 | `src/workflow.json` | `workflow.html` | 泳道图：打开 → 每日一轮 → 沉淀资产 |

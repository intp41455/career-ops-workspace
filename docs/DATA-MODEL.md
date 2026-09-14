# 数据模型 · Data Model

说明 `state` 的结构、持久化契约与扩展方式。所有字段名均与 `index.html` 中的实现一致。

---

## 1. 存储契约

| 项 | 值 |
|---|---|
| 载体 | `localStorage` |
| key | `career_ops_v1` |
| 格式 | 单个 JSON 字符串 |
| 每次写入 | `JSON.stringify(state)` 全量覆盖 |
| 读取失败 | `try/catch` 捕获 → 回落到 `seedData()` |
| 首次打开 | 无 key → 注入 `seedData()` 并立即 `saveData()` |

```js
const LS_KEY = 'career_ops_v1';
```

> Fork 后请更换 `LS_KEY`，否则可能与同域下的其他副本互相覆盖。

---

## 2. 顶层结构

```js
const DEFAULT_DATA = {
  version: 2,                                                // 数据结构版本
  targets:   [],   // 目标看板卡片
  tasks:     [],   // 今日待办
  plans:     [],   // 21 天计划（当前由 STAGES 常量驱动，留作扩展）
  resources: [],   // 学习资料库
  sprint:    [],   // 投递冲刺 14 天清单
  jdmap:     [],   // JD 关键词映射
  canvas:    { nodes: [], edges: [], view: { x: 0, y: 0, z: 1 } },
  skipExample: false                                         // 是否已跳过示例数据
};
```

### 版本迁移

`loadData()` 在解析成功后执行一次**补齐式迁移**：只补缺，不覆盖。

```js
if(!state.sprint || !state.sprint.length) state.sprint = SPRINT_SEED();
if(!state.jdmap  || !state.jdmap.length)  state.jdmap  = JDMAP_SEED();
```

这样老版本导出的备份在新版本页面里依然能打开，且新增模块会自动带上默认内容。

---

## 3. 各集合字段

### 3.1 `targets` — 目标看板

```js
{
  id:     'g1',                    // 唯一 id
  title:  'FastAPI 接口服务：含 Pydantic 校验 / 异常处理 / 日志',
  stage:  'S1 筑基',               // 归属阶段（对应 STAGES[].key）
  pri:    'P0',                    // P0 | P1 | P2
  status: 'doing',                 // todo | doing | done | blocked
  due:    '2026-09-19',            // YYYY-MM-DD
  res:    '可运行的文件上传 + RAG 接口服务'   // 可验收产出物
}
```

**看板四栏与 `status` 的映射**

| 列 | status | 说明 |
|---|---|---|
| 待启动 | `todo` | —— |
| 进行中 | `doing` | —— |
| 已完成 | `done` | —— |
| 卡住了 | `blocked` | 显式承认卡点，而不是静默拖延 |

拖拽卡片即修改 `status`（`moveTarget(id, newStatus)`）。

### 3.2 `tasks` — 今日待办

```js
{
  id:     'k1',
  title:  '把「零成本全栈交付」话术写成 30 秒口述版本',
  pri:    'P0',
  status: 'today',        // today | todo | doing | done
  due:    '2026-09-14',
  src:    '话术准备'       // 来源标签（自由文本）
}
```

### 3.3 `resources` — 学习资料

```js
{
  id:    'r1',
  stage: 'S1 筑基',                    // 分组依据
  pri:   'P0',
  kind:  'B站',                        // B站 | GitHub | 文档
  type:  '视频',                       // 视频 | 仓库 | 规范
  title: 'FastAPI 从入门到实战（完整项目）',
  by:    '搜索关键词：FastAPI 教程 2025',   // 作者 / 检索入口
  url:   'https://search.bilibili.com/...',
  why:   '你有 Python 脚本基础但没写过 Web 服务……',   // 为什么推荐
  hours: '10–12h',
  done:  false
}
```

`why` 是这份资料库的核心字段 —— 每条资料都必须说明**为什么给这个人推荐**，避免变成链接堆放。

### 3.4 `sprint` — 投递冲刺清单

```js
{
  id:    'sp1',
  day:   'D1',
  due:   '2026-09-15',
  out:   '简历事实核对表',               // 当天产出的交付物
  title: '把所有量化数字核实一遍，一个都不许含混',
  items: ['员工规模口径：……', '考勤数据量：……'],   // 子项清单
  hours: '2h',
  risk:  '别在这一步反复纠结措辞，先核数字，措辞 D14 统一打磨',
  done:  false
}
```

设计原则：**产出驱动而非学习驱动** —— 每天一个能拿给人看的成品。

### 3.5 `jdmap` — JD 关键词映射

```js
{
  id:     'j1',
  kw:     'Python 开发',                    // JD 要求
  freq:   '必备',                           // 必备 | 核心 | 高频 | 加分
  resume: '考勤数据治理流水线 / docx 标准化引擎',   // 你手里的弹药（— 表示暂无）
  ready:  'green',                          // green | blue | red
  note:   '措辞用「独立编写数据处理脚本与自动化工具」，别自称精通。'
}
```

**掌握度三态**（点击行内状态可轮转）

| 值 | 含义 | 视觉 |
|---|---|---|
| `green` | 已有真弹药，能讲清细节 | 绿 |
| `blue` | 需打磨到能脱口而出 | 蓝 |
| `red` | 必须补出来 | 红 |

`resume: '—'` 表示该关键词目前没有对应经验 —— 这类条目就是真缺口，应与 `sprint` / `STAGES` 对齐。

### 3.6 `canvas` — 无限画布

```js
canvas: {
  nodes: [
    {
      id:    'n1',
      x:     60,            // 画布坐标（非屏幕坐标，受 view 变换）
      y:     230,
      w:     214,           // 可选：显式宽度；缺省则按文本内容自动计算
      text:  'AI 应用落地工程师',
      type:  'root',        // root | branch | leaf
      color: 0              // 调色板索引，见下
    }
  ],
  edges: [ ['n1','n2'], ['n1','n3'] ],       // [fromId, toId] 二元组
  view:  { x: 0, y: 0, z: 1 }                // 平移与缩放（由 layoutFit 计算）
}
```

**节点类型的渲染差异**

| type | 填充 | 文字 | 描边 |
|---|---|---|---|
| `root` | 主题强调色实底 | 白色，首行加粗 | 2px |
| `branch` | 调色板色 + 低透明度 | 调色板色，加粗 | 1.5px |
| `leaf` | `--cv-leaf` | `--cv-leaf-ink` | 1.5px |

**调色板**：索引 `0..5`，暗/亮主题各一套（`PALETTE` / `PALETTE_DK`）。
浅底改深底时需要整体提亮，否则在暗色下会「糊」在背景里 —— 由 `cvColor()` 统一处理。

节点尺寸不存进 state，由 `cvMetrics()` / `cvNodeSize()` 按内容实时计算，保证改字后布局自动跟随。

---

## 4. 导出 / 导入格式

「导出备份」产出的 JSON：

```js
{
  "app": "career-ops-workspace",
  "exportedAt": "2026-09-14T07:12:33.412Z",
  "data": { /* 完整的 state */ }
}
```

文件名：`AI转型作战台-备份-YYYY-MM-DD.json`

**导入行为**：读取 `payload.data`（或直接把整个对象当 state 兜底），然后与 `DEFAULT_DATA` 做**浅合并 + 嵌套修复**，保证缺失字段有默认值。没有条数上限。

> 迁移建议：换设备时先在旧设备导出 → 新设备打开页面 → 导入。数据不经过任何服务器。

---

## 5. 静态配置（不入 state）

以下内容写在代码里，是「模板」而非「数据」；首次渲染时用于填充 state：

| 常量 | 作用 |
|---|---|
| `STAGES` | 21 天计划的 5 个阶段（key / name / days / goal / focus / tasks） |
| `COGNITION` | 认知取向 → 执行协议条目（`{t: 现象, k: 含义与做法}`） |
| `START_DATE` | 计划起算日，计划轴与冲刺 D1–D14 都基于它推算 |
| `VIEWS` | 视图注册表（key / label / short / icon / group / sub） |
| `seedData()` | 首次打开时的示例数据（含 1 条逾期，用于让用户立刻看到效果） |

改这些常量 = 改「出厂设置」。已存在于 `localStorage` 的用户数据不会被覆盖。

---

## 6. 扩展点

### 新增一个模块

1. 在 `DEFAULT_DATA` 加字段
2. 写 `XXX_SEED()` 并在 `loadData()` 的迁移区补一行补齐逻辑
3. 写 `renderXxx()`（只读 state、只写 DOM）
4. 在 `VIEWS` 注册视图
5. 在 `refreshAll()` 的 `if/else` 链里加分支
6. 在 `onAction` 的 `switch` 里加动作（如需交互）
7. 在 `renderNav()` 的分组里出现（`VIEWS` 驱动，通常无需改）

### 换成远端存储

改造点集中在两个函数，不要散落到各模块：

```js
async function loadData(){ /* localStorage.getItem → fetch 远端 */ }
async function saveData(){ /* localStorage.setItem → 防抖后 push 远端 */ }
function flashSync(mode){ /* 已预留 saving / ok / off 三态，可直接复用 */ }
```

要点：
- 保留 `localStorage` 作为**离线缓存层**，断网时写入本地，恢复后批量同步
- 冲突处理以时间戳较新者为准，合并时保留双方新增条目
- 失败时降级为纯本地模式，并在顶栏提示「离线模式」

---

## 7. 数据安全须知

- `localStorage` 属于**浏览器站点数据**，清除浏览数据 / 换浏览器 / 无痕模式都会丢失
- 部署到公网后，**链接是公开可访问的**，但数据仍只存在于访问者自己的浏览器中
- 仓库内所有示例数据均为虚构；Fork 后请勿把真实姓名、薪资、公司内部数据 commit 进仓库
- 页面的「清空数据」带二次确认；养成定期「导出备份」的习惯

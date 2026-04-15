---
name: xhs-xueqiu-sentiment
description: 小红书+雪球双源股票舆情监测 Skill。当用户想监测小红书/雪球股票舆情、了解散户情绪风向、获取反向风险信号、进行每日舆情日报、复盘前日判断、追踪板块个股趋势时，必须使用此 Skill。触发场景：提到"小红书舆情"、"雪球舆情"、"雪球热帖"、"雪球讨论"、"散户情绪"、"反向指标"、"今日舆情"、"跑一次舆情"、"舆情日报"、"帮我加入watchlist"、"监测这只股"、"跟踪这个板块"、"复盘昨天"、"下午四点跑"、"刷新日报"。输出每日 HTML 日报，含小红书反向情绪 + 雪球中性参考双源评分、热点发现、Watchlist 追踪、前日复盘、多日趋势推断。
compatibility: "需要 Claude in Chrome（已连接，已登录小红书和雪球）"
---

# 股票舆情监测 Skill v3

## 核心定位

**双源情绪模型：**
- 🔴 **小红书** = 散户反向风向标（热度越高风险越大，以最新排序抓今日新帖）
- 🟡 **雪球** = 专业投资者中性参考（热股榜 + 讨论量反映市场关注度）

**四大能力：**
1. 双源数据采集（token 高效，合并抓取）
2. Watchlist 个股/板块深度追踪
3. 前日复盘（结合收盘价验证判断准确性）
4. 多日趋势智能推断（连续信号识别）

**输出：HTML 日报文件**，每日刷新，可在浏览器直接查看。

---

## Watchlist 管理

**首次使用流程：**

1. 读取 `references/watchlist.md`，检查是否有标的
2. 若 Watchlist 为空（暂无标的），则：
   - 询问用户："你想监测哪些个股或板块？可以告诉我名称，我来添加。"
   - 若用户暂时没有，默认只跑**宏观大盘**模式（不含 Watchlist 专项追踪）
3. 若 Watchlist 有标的，直接进入正常日报流程

**用户说"监测XX / 加入watchlist / 跟踪XX"时，记录：**
```
名称 | 类型(个股/板块/ETF) | XHS关键词×3 | 雪球搜索词 | 备注
```
建议3个小红书关键词供用户确认后写入 watchlist.md。

**宏观大盘模式（无 Watchlist 时的默认输出）：**
- 小红书大盘情绪 TOP5
- 雪球热股榜 Top9
- 今日热点标的自动发现
- 综合风险指数
- 趋势推断（基于大盘关键词的多日数据）

---

## 执行流程（精简为4步）

### Step 0：前置检查 + 读取昨日数据

```javascript
// 检查连接
tabs_context_mcp()
```

同时读取 `references/memory.md`，获取：
- 昨日综合判断
- 昨日各 Watchlist 标的的判断方向（看涨/看跌/中性）
- 昨日收盘价（若用户已填入）

---

### Step 1：雪球数据采集（快速，2个操作）

**1A：热股榜（DOM 抓取，10秒完成）**

导航到 `https://xueqiu.com/hq#hot`，执行：
```javascript
const names = [...document.querySelectorAll('.style_hot-stock-name_2PL')]
  .map(el => el.innerText?.trim());
const pcts = [...document.querySelectorAll('[class*="hot-stock-percent"]')]
  .map(el => el.innerText?.trim());
JSON.stringify(names.slice(0,9).map((n,i) => ({ rank:i+1, name:n, change:pcts[i] })))
```

**1B：Watchlist 标的讨论量（每个标的一次导航）**

对每个 Watchlist 标的，导航到：
`https://xueqiu.com/statuses/search.json?q=<标的名>&count=10&page=1`

用 `document.body.innerText` 读取 JSON，提取：
- `total_count`（总讨论量）
- 前5条 statuses 的 `text`（去HTML标签）、`like_count`、`reply_count`

> **Token节省点**：雪球接口直接返回JSON，无需解析DOM，每个标的一次导航即可。

---

### Step 2：小红书数据采集（最新排序，批量执行）

**每个关键词固定三步（1A导航 + 1B切换最新 + 1C抓取）：**

**1B 切换最新排序（MutationObserver，每次必执行）：**
```javascript
new Promise((resolve) => {
  const already = window.__INITIAL_STATE__?.search?.searchContext
    ?.filters?.find(f=>f.type==='sort_type')?.tags?.[0] === 'time_descending';
  if (already) { resolve('ok'); return; }
  const obs = new MutationObserver(() => {
    document.querySelectorAll('div.tags').forEach(el => {
      if (el.innerText?.trim()==='最新' && !el.classList.contains('active')) {
        el.click(); obs.disconnect(); resolve('switched');
      }
    });
  });
  obs.observe(document.body, { childList:true, subtree:true });
  document.querySelector('div.filter')?.click();
  setTimeout(() => { obs.disconnect(); resolve('timeout'); }, 3000);
})
```

**1C 抓取数据：**
```javascript
// 等待1秒后执行
const results = [];
document.querySelectorAll('section.note-item').forEach((el,i) => {
  if (i >= 15) return;
  const title = el.querySelector('a.title span')?.innerText?.trim();
  const author = el.querySelector('.name span')?.innerText?.trim();
  const likes = el.querySelector('.count')?.innerText?.trim();
  const link = el.querySelector('a[href*="/explore/"]')?.href;
  const noteId = link?.match(/\/explore\/([a-f0-9]+)/)?.[1];
  if (title) results.push({ title, author, likes, noteId });
});
JSON.stringify(results)
```

**每日必搜关键词（10个，从 references/keywords.md 读取）**

同时，对每个 Watchlist 标的也执行相同的三步抓取流程。

> **Token节省点**：每个关键词只取前15条（够用），不做逐条深度分析，统一在 Step 3 批量处理。

---

### Step 3：分析处理（纯文本推理，不额外抓取）

在已有数据基础上完成所有分析，**不再发起新的网络请求**：

**3A：计算今日风险指数**
读取 `references/scoring.md`，对小红书数据评分，归一化到0-100。

**3B：提取热点标的**
从小红书帖子标题 + 雪球热股榜，合并识别今日冒头标的 Top 3。

**3C：Watchlist 情绪分类**
对每个 Watchlist 标的的帖子，分类为：追涨/解套/割肉/做T/新入场。
计算各类占比，评定套牢压力（高/中/低）。

**3D：前日复盘**
对照 memory.md 中的昨日判断：
- 若判断"看涨"而标的今日上涨 → ✅ 准确
- 若判断"看涨"而标的今日下跌 → ❌ 失误，分析原因
- 归纳失误模式，更新判断校准权重

**3E：多日趋势推断（核心智能）**
读取 watchlist.md 历史数据，识别连续信号：
- 连续3日追涨情绪上升 + 雪球讨论量放大 → 🔴 过热预警
- 连续3日割肉情绪占主导 + 热度下降 → 🟢 潜在底部信号
- 雪球讨论量突增但小红书无反应 → 🟡 机构关注但散户未跟进
- 小红书热度暴增而雪球讨论平稳 → ⚠️ 纯散户追涨，风险高
若有值得主动推送的信号，在日报顶部加 **🔔 智能预警** 模块。

---

### Step 4：生成 HTML 日报

**输出到 `/mnt/user-data/outputs/daily_report_YYYYMMDD.html`**

读取 `references/report_template.html` 作为基础样式，填入数据。

**HTML 日报结构：**
```
[🔔 智能预警]（有信号时显示，否则隐藏）
[📅 日期 | 数据源 | 抓取量]
[🌡️ 今日综合风险指数]（仪表盘可视化）
[📰 前日复盘]（昨日判断 vs 实际表现）
[🔥 大盘情绪 TOP5]（小红书）
[📈 雪球热股榜]（涨跌榜）
[🆕 今日热点标的]（双源融合发现）
[📌 Watchlist 专项追踪]（每个标的独立卡片）
[🧠 多日趋势推断]（连续信号归纳）
[⚠️ 风险声明]
```

---

### Step 5：更新记忆文件

日报生成后，自动更新两个文件：

**更新 `references/memory.md`：**
```markdown
## 最新一日记录（YYYY-MM-DD）
综合风险指数：[N]
大盘判断：[看涨/中性/看跌] + 理由一句话
Watchlist判断：
- 恒生科技：[方向] | 信心：[高/中/低]
- （其他标的）
待用户填入收盘价：[留空]
```

**更新 `references/watchlist.md` 趋势表：**
追加今日数据行（帖子数/主导情绪/热度指数/套牢压力）

---

## 异常处理

| 情况 | 处理 |
|------|------|
| 小红书登录失效 | 提示重新登录，跳过XHS，仅用雪球数据 |
| 雪球热股榜加载失败 | 重试一次，仍失败则跳过该模块 |
| MutationObserver超时 | 记录"排序切换失败"，继续抓取（数据仍有参考价值）|
| Watchlist标的无结果 | 标注"今日无明显讨论"，保留雪球侧数据 |
| memory.md不存在 | 跳过复盘模块，仅输出当日数据 |

---

## 参考文件

- `references/keywords.md` — 每日必搜关键词（分级）
- `references/scoring.md` — 风险评分规则
- `references/watchlist.md` — 用户Watchlist + 历史趋势数据
- `references/memory.md` — 每日判断记录（复盘用）
- `references/report_template.html` — HTML日报样式模板

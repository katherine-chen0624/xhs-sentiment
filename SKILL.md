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

**五大能力：**
1. 双源数据采集（token 高效，合并抓取）
2. 过热板块主动定位与简要分析
3. Watchlist 个股/板块深度追踪
4. 预测复盘追踪（每日给出方向判断，次日校准准确性）
5. 多日趋势智能推断（连续信号识别）

**输出：HTML 日报文件**，每日刷新，浏览器直接查看。

---

## Watchlist 管理

**首次使用流程：**

1. 读取 `references/watchlist.md`，检查是否有标的
2. 若 Watchlist 为空，询问：「你想监测哪些个股或板块？告诉我名称，我来添加。若暂无，直接跑宏观大盘模式。」
3. 若有标的，直接进入正常日报流程

**用户说"监测XX / 加入watchlist / 跟踪XX"时，记录：**
名称 | 类型(个股/板块/ETF) | XHS关键词×3 | 雪球搜索词 | 备注

建议3个小红书关键词供用户确认后写入 watchlist.md。

**宏观大盘模式（无 Watchlist 时的默认输出）：**
- 小红书大盘情绪 TOP5 + 过热板块识别
- 雪球热股榜 Top9
- 今日热点标的自动发现
- 综合风险指数
- 趋势推断

---

## 执行流程

### Step 0：前置检查 + 初始化 + 读取昨日数据

检查 Chrome 连接（tabs_context_mcp），同时读取：
- `references/tracking.md` — 追踪股票 + 历史预测准确率
- `references/memory.md` — 昨日综合判断和收盘价

**首次使用检测（tracking.md 追踪股票为空时）：**

在开始抓取前先问用户：

> 「在开始今日舆情分析前，想设置一下**预测复盘追踪**：
> 希望我每日对哪几支股票给出涨跌方向判断，第二天对照收盘价复盘校准？
> 建议3支以内。可告诉我股票名称+代码，或说"跳过"直接出日报。」

- 用户提供股票 → 写入 tracking.md，继续执行
- 用户跳过 → 跳过复盘模块，直接执行日报

**非首次使用（已有追踪股票）：**

检查 memory.md 昨日判断是否缺少收盘价：
- 缺失 → 日报开头提醒：「⚠️ 昨日复盘待完成，请告知 [股票名] 昨日收盘价」
- 已有 → 直接进入复盘计算

---

### Step 1：雪球数据采集（快速，2个操作）

**1A：热股榜（DOM 抓取）**

导航到 `https://xueqiu.com/hq#hot`，执行：
```javascript
const names = [...document.querySelectorAll('.style_hot-stock-name_2PL')]
  .map(el => el.innerText?.trim());
const pcts = [...document.querySelectorAll('[class*="hot-stock-percent"]')]
  .map(el => el.innerText?.trim());
JSON.stringify(names.slice(0,9).map((n,i) => ({ rank:i+1, name:n, change:pcts[i] })))
```

**1B：Watchlist + 追踪股票雪球讨论量**

对每个标的，导航到：
`https://xueqiu.com/statuses/search.json?q=<标的名>&count=10&page=1`

用 `document.body.innerText` 读取 JSON，提取 `total_count`、前5条的 `text`（去HTML标签）、`like_count`、`reply_count`。

---

### Step 2：小红书数据采集（最新排序）

**每个关键词固定三步：导航 → 切换最新 → 抓取**

**切换最新排序（MutationObserver）：**
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

**抓取数据（等待1秒后执行）：**
```javascript
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

**每日必搜关键词（从 references/keywords.md 读取，当前15个）：**

情绪类（大盘风向）：抄底了、买股票、第一次买股票、割肉了、套牢了、理财搭子、瞒着老公买了、黄金买吗、ETF

板块热度类（新增）：板块今天、涨停板块、A股主线、今日复盘

过热信号类（新增）：XX能追吗、XX还能涨吗（动态替换为当日雪球热股榜 Top3 名称）

同时对 Watchlist 和追踪股票各执行一次专项搜索。

---

### Step 3：分析处理（纯推理，不额外抓取）

**3A：计算今日综合风险指数**
读取 `references/scoring.md`，对小红书数据评分，归一化到0-100。

**3B：过热板块主动定位（新增核心能力）**

从「板块今天」「涨停板块」「A股主线」抓取结果中：
1. 提取高频出现的板块/个股名称（出现≥2次 或 点赞≥50）
2. 结合雪球热股榜涨幅异常（单日涨幅>8%）交叉验证
3. 对识别出的过热标的进行简要分析：
   - 是什么在驱动（题材/业绩/政策/情绪）
   - 散户追涨情绪程度（轻度/中度/亢奋）
   - 风险提示（是否已经高位/连板/换手异常）

**3C：Watchlist 情绪分类**
帖子分为：追涨/解套/割肉/做T/新入场，计算占比，评定套牢压力。

**3D：预测复盘计算**
对照 tracking.md 中昨日预测方向 vs 实际收盘价：
- 看涨 + 实际上涨 → ✅ 准确
- 看涨 + 实际下跌 → ❌ 失误，分析原因
- 更新 tracking.md 准确率统计和校准笔记

**今日预测输出（写入 memory.md）：**
对每支追踪股票给出：方向（看涨/中性/看跌）+ 信心（高/中/低）+ 理由（基于双源舆情）

**3E：多日趋势推断**
读取 watchlist.md 历史数据，识别：
- 连续3日追涨情绪上升 + 雪球讨论量放大 → 🔴 过热预警
- 连续3日割肉情绪占主导 + 热度下降 → 🟢 潜在底部信号
- 雪球讨论量突增但小红书无反应 → 🟡 机构关注但散户未跟进
- 小红书热度暴增而雪球讨论平稳 → ⚠️ 纯散户追涨，风险高

---

### Step 4：生成 HTML 日报

输出到 `/mnt/user-data/outputs/daily_report_YYYYMMDD.html`

读取 `references/report_template.html` 填入数据。

**日报结构：**
```
[🔔 智能预警]（有信号时显示）
[📅 日期 | 数据源 | 抓取量]
[⚠️ 复盘提醒]（缺收盘价时显示）
[🌡️ 今日综合风险指数]
[📰 前日复盘]（预测 vs 实际，含准确率）
[🔥 大盘情绪 TOP5]（小红书·最新）
[🚨 今日过热板块识别]（新增）
[📈 雪球热股榜 Top9]
[🆕 今日热点标的]（双源融合）
[📌 Watchlist 专项追踪]
[🎯 今日预测]（追踪股票涨跌方向判断）
[🧠 多日趋势推断]
[⚠️ 风险声明]
```

---

### Step 5：更新记忆文件 + 收盘价提醒

**更新 `references/memory.md`：**
```
## 最新一日记录（YYYY-MM-DD）
综合风险指数：[N]
大盘判断：[方向] + 理由
追踪股票判断：
- [股票名]：[看涨/中性/看跌] | 信心：[高/中/低] | 理由：[一句话]
待填入收盘价：[股票名] [留空]
```

**更新 `references/watchlist.md` 趋势表：**
追加今日数据行。

**更新 `references/tracking.md`：**
追加今日预测记录行。

**日报末尾主动提醒用户：**
> 「📌 今日预测已记录。明日运行日报前，请告知今日收盘价以完成复盘：
> [股票1名称]：___  [股票2名称]：___  [股票3名称]：___」

---

## 异常处理

| 情况 | 处理 |
|------|------|
| 小红书登录失效 | 提示重新登录，跳过XHS，仅用雪球数据 |
| 雪球热股榜加载失败 | 重试一次，仍失败则跳过 |
| MutationObserver超时 | 记录"排序切换失败"，继续抓取 |
| Watchlist标的无结果 | 标注"今日无明显讨论"，保留雪球侧数据 |
| memory.md不存在 | 跳过复盘，仅输出当日数据 |
| tracking.md为空 | 执行首次设置询问流程 |

---

## 参考文件

- `references/keywords.md` — 每日必搜关键词（分级）
- `references/scoring.md` — 风险评分规则
- `references/watchlist.md` — 用户Watchlist + 历史趋势数据
- `references/tracking.md` — 预测复盘追踪股票 + 准确率记录
- `references/memory.md` — 每日判断记录（复盘用）
- `references/report_template.html` — HTML日报样式模板

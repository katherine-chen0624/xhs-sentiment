---
name: xhs-sentiment
description: 小红书股票舆情监测 Skill。当用户想监测小红书上的股票/投资舆情、了解散户情绪风向、获取反向风险信号、进行每日舆情日报分析时，必须使用此 Skill。触发场景包括：提到"小红书舆情"、"散户情绪"、"反向指标"、"股票舆情监测"、"小红书股票"、"今日舆情"、"跑一次舆情"、"舆情日报"、"帮我加入watchlist"、"监测这只股"、"跟踪这个板块"。此 Skill 会自动用 Claude in Chrome 抓取小红书数据，计算风险评分，并输出专业金融分析师视角的舆情日报，涵盖当日热门个股/板块自动发现 + 用户自定义 Watchlist 深度追踪。
compatibility: "需要 Claude in Chrome 扩展（已连接 + 用户已登录小红书）"
---

# 小红书股票舆情监测 Skill v2

## 核心定位

小红书是**散户情绪的反向风向标**：讨论量越高、情绪越亢奋，市场风险越大。本 Skill 以专业金融分析师视角，每日从小红书抓取舆情，输出三层分析：
1. **大盘情绪层** — 综合风险指数 + 散户情绪温度
2. **热点发现层** — 当日小红书自动冒头的强势个股/板块
3. **Watchlist 层** — 用户指定标的的深度舆情追踪

---

## Watchlist 管理（优先处理）

**当用户说"加入watchlist"、"监测XX"、"跟踪XX板块/股票"时：**

立即询问并记录：
```
标的名称：[股票名/板块名]
类型：[个股 / 板块]
小红书搜索关键词：[建议3个，用户确认]
加入原因备注：[用户填写，如"恒生科技2025年被推很多，散户套牢严重"]
```

将 Watchlist 存储在对话记忆中，格式：
```json
{
  "watchlist": [
    {
      "name": "恒生科技",
      "type": "板块",
      "keywords": ["恒生科技", "港股科技", "恒科ETF"],
      "note": "2025年被推极多，大量散户套牢，需持续监测解套/割肉情绪",
      "added": "2026-04-14"
    }
  ]
}
```

**每次日报自动包含全部 Watchlist 标的的专项分析。**

---

## 执行前置检查

```javascript
tabs_context_mcp()
```
若未连接，提示：**"请确保 Claude in Chrome 扩展已连接，并已在浏览器中登录小红书"**

---

## Step 1：大盘情绪关键词抓取

读取 `references/keywords.md`，按每日必搜10个词依次执行。

**每个关键词的完整抓取流程（3步，缺一不可）：**

### Step 1-A：导航到搜索页

```
https://www.xiaohongshu.com/search_result?keyword=<关键词>&source=web_explore_feed
```

### Step 1-B：切换到「最新」排序

导航完成后，执行以下 JS 打开筛选面板并自动点击"最新"：

```javascript
// 用 MutationObserver 等待筛选面板渲染完成后点击"最新"
new Promise((resolve) => {
  const observer = new MutationObserver(() => {
    const tags = document.querySelectorAll('div.tags');
    tags.forEach(el => {
      if (el.innerText?.trim() === '最新' && !el.classList.contains('active')) {
        el.click();
        observer.disconnect();
        resolve("✅ 已切换到最新排序");
      }
    });
  });
  observer.observe(document.body, { childList: true, subtree: true });
  // 点击筛选按钮打开面板
  document.querySelector('div.filter')?.click();
  // 3秒超时保护
  setTimeout(() => {
    observer.disconnect();
    resolve("⚠️ 超时，当前排序: " + (window.__INITIAL_STATE__?.search?.searchContext?.filters?.find(f=>f.type==='sort_type')?.tags?.[0] || 'unknown'));
  }, 3000);
})
```

> 💡 **说明**：小红书筛选面板是异步渲染的，必须用 MutationObserver 等待"最新"按钮出现后再点击。点击后面板自动关闭并刷新结果，无需再点确认。可通过检查 `window.__INITIAL_STATE__.search.searchContext.filters` 中 `sort_type` 是否为 `time_descending` 来验证生效。

### Step 1-C：等待结果刷新后抓取数据

排序切换后等待约 1 秒让页面刷新，再执行抓取：

```javascript
const results = [];
document.querySelectorAll('section.note-item').forEach((el, i) => {
  if (i >= 20) return;
  const title = el.querySelector('a.title span')?.innerText?.trim();
  const author = el.querySelector('.name span')?.innerText?.trim();
  const likes = el.querySelector('.count')?.innerText?.trim();
  // 提取发布时间（最新排序下可见）
  const timeEl = el.querySelector('.author-wrapper span:last-child, [class*="time"]');
  const time = timeEl?.innerText?.trim();
  const link = el.querySelector('a[href*="/explore/"]')?.href;
  const noteId = link?.match(/\/explore\/([a-f0-9]+)/)?.[1];
  if (title || noteId) results.push({ title, author, likes, time, noteId });
});
JSON.stringify(results)
```

> 📌 **最新排序的价值**：能抓到今日刚发的帖子（如"15分钟前"），是当日实时情绪的最真实反映，而非历史高赞帖的堆积。

---

## Step 2：当日热门个股/板块自动发现

**这是 v2 新增的核心能力。** 通过两个维度自动识别当日小红书上冒头的强势标的：

### 2A — 从已抓取数据中提取标的名

扫描 Step 1 的所有帖子标题，提取出现的：
- A股股票名（如"比亚迪"、"宁德时代"、"中芯国际"）
- 港股名（如"腾讯"、"美团"、"阿里"、"恒生科技"）
- 美股名（如"英伟达"、"特斯拉"、"纳斯达克"）
- 板块名（如"半导体"、"光模块"、"AI算力"、"新能源"、"黄金"）
- ETF名（如"科创ETF"、"纳指ETF"、"恒科ETF"）

统计每个标的出现次数 + 累计点赞数 → 按热度排序取 Top 5

### 2B — 专项热点搜索

额外搜索以下动态关键词，捕捉当日爆发标的：

```
今天涨停
今天暴涨
今日强势
板块今天
```

**抓取后提取标题中的具体标的名**，与 2A 合并去重，得到**今日热点标的候选列表**。

### 2C — 对 Top 3 热点标的做深度小红书搜索

对候选列表中热度最高的 3 个标的，各执行一次专项搜索：
```
https://www.xiaohongshu.com/search_result?keyword=<标的名>&source=web_explore_feed
```
抓取前15条，提取：标题、点赞数、情绪倾向（正面/负面/中性）

---

## Step 3：Watchlist 专项追踪

对用户 Watchlist 中的**每一个标的**，依次执行：

1. **搜索小红书**：用该标的的3个关键词各搜一次，合并结果
2. **情绪分类**：将帖子分为 追涨/解套/割肉/长期持有/新入场 五类
3. **套牢深度评估**（特别针对历史高热标的如恒生科技）：
   - 检测"回本"、"解套"、"亏了多少"、"还要等多久"等词
   - 统计负面情绪帖比例，判断散户套牢压力
4. **趋势对比**：与上次日报的热度数据对比，标注"↑升温/↓降温/→持平"
5. **web_search 归因**：搜索该标的近48小时内的新闻/公告

**Watchlist 标的输出格式：**
```
【标的名】[个股/板块] 📌 Watchlist
━━━━━━━━━
今日热度：[帖子数] 条 | 趋势：↑↓→
情绪分布：追涨[N]% | 解套[N]% | 割肉[N]% | 新入场[N]%
套牢压力：[高/中/低]（基于负面情绪帖比例）
代表帖子：
  · "[最高赞追涨帖标题]" 👍[数]  — 🔴追涨风险
  · "[最高赞解套帖标题]" 👍[数]  — 🟡套牢情绪
今日归因：[新闻/公告摘要，含来源]
多方论点：[看涨的主要论据]
空方论点：[看跌/风险因素]
客观判断：[分析师中性视角]
```

---

## Step 4：利好利空快讯整合

用 `web_search` 搜索今日市场重要消息，补充进日报：

搜索词：
- `A股 今日 重要公告 利好`
- `A股 今日 政策 消息`
- `[当日热点板块] 今日 消息`

筛选原则：只收录 **A级信源**（财联社/巨潮/证监会官网），时效48小时内。

---

## Step 5：综合输出

### 完整日报模板 v2

```
📊 小红书股票舆情日报 v2
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📅 [DATE]  |  🔍 抓取关键词 [N]个  |  📝 采集帖子 [N]条

━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🌡️ 今日综合风险指数：[分]/100  [等级emoji]
[风险等级描述]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔥 今日大盘情绪 TOP 5 话题

[同 v1 格式，热门情绪词 + 代表帖 + 情绪特征]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🆕 今日小红书热点标的发现

🏆 #1 [标的名] — 热度指数 [分]  ⚠️[风险等级]
板块：[所属板块]  |  类型：[个股/板块/ETF]
热帖摘要：
  · "[帖子标题]" 👍[数]
  · "[帖子标题]" 👍[数]
散户情绪：[一句话描述]
今日驱动：[基于web_search的归因，含信源]
分析师判断：[中性客观视角]

🏆 #2 ...
🏆 #3 ...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📌 Watchlist 专项追踪

[每个 Watchlist 标的按上方格式输出]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📰 今日重要快讯（A级信源）

· [时间] [标题] — [来源]
· [时间] [标题] — [来源]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📈 趋势追踪（连续多日数据）

[标的名]：[日期1: 热度X] → [日期2: 热度Y] → 今日: 热度Z  趋势↑
[如无历史数据则跳过此节]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠️ 风险声明
本报告基于小红书公开内容的舆情分析，仅供参考，不构成投资建议。
小红书散户情绪作为反向指标使用，历史规律不代表未来表现。
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 异常处理

| 情况 | 处理方式 |
|------|---------|
| 页面弹出登录框 | 暂停，提示用户重新登录后继续 |
| 出现滑块验证码 | 提示用户手动完成验证，等待后继续 |
| 某关键词搜索结果为空 | 跳过，记录在日报备注 |
| Watchlist 标的无相关帖子 | 标注"今日无明显讨论"，仍输出 web_search 归因 |
| 网络超时 | 重试一次，仍失败则跳过 |

---

## 参考文件

- `references/keywords.md` — 完整情绪关键词库（分级 + 每日执行列表）
- `references/scoring.md` — 风险评分规则（情绪词库 + 计算公式）
- `references/watchlist.md` — 用户 Watchlist 持久化存储（由 Skill 自动维护）

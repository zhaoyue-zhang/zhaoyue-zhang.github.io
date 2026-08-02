---
title: "复活了一个停更的 GitHub 项目：weread2notion-pro"
date: 2026-08-02
draft: false
tags: ["Personal Project"]
description: "把 2024 年停更的 weread2notion-pro 重新 fork、切换到微信读书 2025 年开放的官方 Agent API Gateway，跑通 256 本书、1851 条划线、2878 小时阅读记录的自动同步。复盘一路上踩到的 7 个坑。"
---

今天闲得没事，在 Notion 里翻旧页面。翻到一个叫"我的微信读书"的东西——啊，这是我两年前通过一个博主搭过的开源项目，当时用了一个叫 [`weread2notion-pro`](https://github.com/malinkang/weread2notion-pro) 的开源项目，试用 GitHub Action，每天可以自动把微信读书里的划线、笔记、阅读时长全同步到 Notion。

我点开看了看，发现页面还在，但已经是 2024 年最后一次同步的状态了。**2 年没动过**。

我又去 GitHub 搜了一下，发现这个项目——**作者 2024 年就停更了**，README 第一行赫然写着"目前项目已不能使用"。原因：原版是直接打微信读书私有 API 的，腾讯后来加了风控，Cookie 经常失效，作者没精力继续维护了。

我看着 Notion 里那堆 2 年前的划线，心里有点痒：**要不……把它搞活？**

## 怎么搞

我先想了两个方向：

1. **找替代项目**——但翻了几个都不满意。pro 版的"按 bookmarkId 增量更新"特性很重要（不会覆盖我在 Notion 里的手写批注），其他项目要么太简单，要么也很久没维护。原作者维护了一个新的商业项目，和一个功能不全的初版项目，都不太符合我的要求。
2. **自己 fork 出来改**——反正我也想学学微信读书的 API。

我选了第二条。

但一上来就有个好消息：**微信读书在 2025 年 4 月开放了官方 Agent API Gateway**。也就是说有了一个正经的鉴权方式——拿一个 `WEREAD_API_KEY`，不用再跟 Cookie 较劲了。Key 申请地址：`https://weread.qq.com/r/weread-skills`，微信扫码登录就能拿到。

## 开搞

先 fork 仓库到自己的 GitHub（[`zhaoyue-zhang/weread2notion-pro-skill_version`](https://github.com/zhaoyue-zhang/weread2notion-pro-skill_version)），clone 下来一看代码结构还挺清晰的——`weread_api.py` 是和微信读书打交道的核心，其他 `notion_helper.py` / `book.py` / `read_time.py` 都是处理 Notion 那一侧的。

我的策略很明确：

- **`weread_api.py` 整层重写**：从私有 API 切到官方 gateway
- **其他 Notion 侧代码一字不动**：这样原作者花了很多心思的"增量更新"、"按 bookmarkId 去重"逻辑都保留

这一步倒是顺利——gateway 文档比较全，照着 endpoint 一个个映射过去就行。`/user/notebooks`、`/book/bookmarklist`、`/book/chapterinfo`、`/book/info`、`/review/list/mine`、`/readdata/detail`——都按原来的语义接上。

## 跑通最小闭环

接好之后，我先跑了个最小流程试试：把一本书的划线同步到 Notion。

**成功了！** 250 多本书、1800 多条划线都进了我建好的 Notion database。

我正得意，问题来了。

## 第一个坑：12 个 database 真的全要手搭吗？

pro 版的设计是：你**自己手搭 11 个 database**（外加 2 个代码自动建的），然后在环境变量里告诉代码每个 database 的名字。

我 2 年前搭过，但现在早就忘了结构了。让我再搭一遍？光想想要建 11 个库、设好字段、接好 relation 关系就头大。

我决定先写个**一键建库脚本**。这其实不难——直接调 Notion REST API 建库就行。`scripts/setup_databases.py` 大概 400 行，把 11 个库的 schema 都定义在里面，跑完就建好了。

最关键的是：**idempotent**——跑第二次会跳过已存在的库，只建缺失的。

## 第二个坑：书的「分类」一直是空的

我跑完同步，256 本书全进了 Notion，但我发现「分类」字段全空。

翻代码一看，原版读的是 `book["categories"]`（数组），但官方 API 返回的是 `book["category"]`（单数字符串）——值像 "精品小说-社会小说" 这种。我改成读 `category`，然后按 `-` 拆开多级分类。

写了个 `scripts/backfill_categories.py` 一次性回填，256 本书全补上了。

## 第三个坑：阅读时长全是 0

接下来是阅读时长——这是 pro 版最有意思的功能，会按"日" database 记录每天读多久，然后聚合到"年/月/周"。

我跑了 `read_time` 命令，结果发现：**所有 day page 的「时长」都是 0**。

翻代码一看，原版调用 `get_api_data()`（annually 模式），这个端点只返回 12 个月分桶的汇总，**不是真实 daily**——每天的 duration 全是 0s。

我得换 monthly 模式自己循环 12 个月。但这里还有个时区坑：官方 API 用 UTC 0 时区的月 1 号作为 `baseTime`，**返回的 key 也是 UTC 0 时区**。而我之前为了避免时区混乱，day page 用的是**上海 0 时区**的 timestamp。

两边一拼：**同一个日历日会产生 2 个 day page**，差 8 小时。

**修法**：调用 `/readdata/detail?mode=monthly&baseTime=<上海 0 时区月 1 号>`，这样 key 也用上海 0 时区，跟 day page 统一。**实测这个 `baseTime` 接受历史年**，所以可以一次性拉 5 年的数据。

新写了 `get_daily_data(years=range(cur-4, cur+1))` 拉近 5 年，然后 `scripts/backfill_historical_days.py` 一次性回填。

5 年的数据全回来了——总共 **2878 小时**。看着挺壮观。

## 第四个坑：同一天还是有 2 个 day page

回填完之后我跑 `aggregate_durations.py` 算年/月/周总和，结果数字对不上。查了一下发现：因为历史原因（之前几版代码交替），数据库里还残留着 142 个**同一天的重复 day page**（一个用 UTC 0 时区 timestamp，一个用上海 0 时区）。

我写了个 `scripts/dedupe_day_pages.py`，按日期 group，保留时长最大的那个，其余 archive 掉。

然后 aggregate 数字终于对了。

## 第五个坑：Notion embed 缓存 SVG 不更新

pro 版有个"阅读热力图"功能——根据每天的阅读时长渲染一个 GitHub contribution 风格的 SVG 网格。SVG 生成之后上传到 GitHub，Notion 用 embed block 引用。

但有个问题：**Notion embed block 第一次抓 SVG 之后会缓存，不会重新拉 raw.githubusercontent.com 的新版本**。我改了 SVG 内容后，Notion 那边还是显示老的。

**解决方案**：在 URL 后面加 `?v=<utc_timestamp>` 做 cache-busting。`raw.githubusercontent.com` 会忽略 query string 正常返回最新内容，但 Notion 看到新 URL 会重新抓。

更狠的一招是 `notion_helper.py` 之前用了一个 `HEATMAP_BLOCK_ID` 环境变量硬编码 embed block id——但我那个 block 之前被我不小心 archive 过一次，新建的 block id 跟 secret 里写的不一样，silent fail 永远不更新。

我把 `HEATMAP_BLOCK_ID` 干掉了，改成**扫页面找 `raw.githubusercontent.com/.../weread-heatmap.svg` 的 embed block**——找到就用，找不到就在 workflow 日志里提示用户手动加 embed。这样 secret 少一个，配置更稳。

## 第六个坑：Notion API 不支持 duration 字段

我跑了一年数据出来，2024 年 = **1118 小时**。然后我去 Notion 改年 page 的「总阅读时长」字段。

我心想：1118 直接放进去也太丑了，Notion 有 duration 字段类型吧？

**没有。**

Notion UI 里有「Duration」字段，但**API 不支持**（截止 2025-09-03 版本）——PATCH 字段定义会直接 400 报 "duration should be defined"。

只能换路子。我用 number + formula 组合：

- number 字段「总阅读时长」：存**分钟整数**（数据层）
- formula 字段「总阅读时长（格式化）」：自动渲染成 "1h 23m" / "46d 14h"

公式（`if / empty / floor / mod / format` 拼）：

```javascript
if(empty(prop("总阅读时长")), "",
if(prop("总阅读时长") == 0, "",
if(prop("总阅读时长") >= 1440,
   format(floor(prop("总阅读时长")/1440)) + "d " + format(floor(mod(prop("总阅读时长"),1440)/60)) + "h",
if(prop("总阅读时长") >= 60,
   format(floor(prop("总阅读时长")/60)) + "h " + format(mod(prop("总阅读时长"),60)) + "m",
format(prop("总阅读时长")) + "m"))))
```

完美。**2024 年 = "46d 14h"**，2025 年第 52 周 = "15h 10m"，2026 年 8 月 = "3m"——看起来都顺眼了。

## 第七个坑：GitHub Actions 失败看不到 log

我部署好之后，让 GitHub Actions 每天自动跑（cron `0 */3 * * *`）。第一次跑失败——`dedupe day pages` 步骤挂了。

我想看 log 查原因，**结果发现 GitHub Actions 的 log 要登录 GitHub 才能看**。我没在本地配 gh CLI。

代码我看了一遍，dedupe 脚本有个 `r.raise_for_status()`——只要 query 出一次 5xx，整个脚本就 crash。Notion API 偶尔会有 rate limit 或者 5xx，这种"严格一个错就死"的设计在 CI 上是行不通的。

我加了 try/except 包裹、retry、continue-on-error，让脚本失败不会让整个 workflow 挂——下次跑会重算。

修完又跑，dedupe 过了，**但 aggregate 挂了**——同一个问题。换上同样的修法，又过了。

## 效果

最终效果（截至今天）：

- **同步书籍**：256 本
- **划线**：1851 条
- **笔记**：一堆
- **5 年阅读总时长**：2878 小时
- **GitHub 仓库**：[zhaoyue-zhang/weread2notion-pro-skill_version](https://github.com/zhaoyue-zhang/weread2notion-pro-skill_version)（private）
- **自动同步频率**：weread note sync 每 2 小时 / read time sync 每 3 小时

## 一些感受

1. **官方 skill API 救了这个项目**。原作者在 2024 年放弃是因为 Cookie 失效，2025 年微信读书开放了 Agent API Gateway，相当于给所有类似项目续了命。
2. **Notion API 的能力比 UI 弱一截**。Duration、Advanced filter 这些 UI 里的功能，API 还没有，临时方案就是 formula 拼字符串。
3. **CI 上的 Notion API 容易踩 rate limit**。我后来每个 batch 加 50ms 间隔就稳了。
4. **GitHub Actions log 要登录才能看**——这个对 agent 调试很不友好，建议尽量在本地复现。
5. **fork 一个 2 年没动的项目，比从头写还累**——因为要兼容老数据格式。`RENAMED` fallback（用户把"年" database 改名叫"阅读数据"）、`utc/上海 0 时区` 兼容、读 `category` 单数字符串……每个坑都是历史遗留。

但搞完之后的爽感是无与伦比的：5 年的阅读记录、256 本书的分类、1800 多条划线——所有数据在 Notion 里完好无损，**还能继续同步下去**。

## 后记

我把这个 fork 开在了 GitHub 上：[zhaoyue-zhang/weread2notion-pro-skill_version](https://github.com/zhaoyue-zhang/weread2notion-pro-skill_version)。

如果你也有同样的需求，欢迎 fork 走（改 3 个 secret 就能跑）：

- `NOTION_TOKEN`
- `NOTION_PAGE`
- `WEREAD_API_KEY`

跑 `python3 scripts/setup_databases.py` 一键建 12 个库，不用手搭。

如果你不熟 Notion API 的 `duration` 字段，记得跑 `python3 scripts/migrate_to_duration.py` 给已有库加 friendly 显示列。

有什么想要改动的功能，也可以自己酌情添加！
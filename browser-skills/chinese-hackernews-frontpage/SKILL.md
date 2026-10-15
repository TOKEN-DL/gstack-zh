---
name: hackernews-frontpage
description: 抓取 Hacker News 首页(标题、得分、评论数)。
host: news.ycombinator.com
trusted: true
source: human
version: 1.0.0
args: []
triggers:
  - scrape hacker news frontpage
  - scrape hn frontpage
  - get hn top stories
  - latest hacker news stories
---


# Hacker News 首页抓取

抓取 Hacker News (`news.ycombinator.com`) 首页,并以 JSON 形式返回前 30 条新闻。每条新闻包含其排名、标题、链接 URL、得分和评论数。

## 用法

```
$ $B skill run hackernews-frontpage
{
  "stories": [
    { "rank": 1, "title": "...", "url": "...", "points": 412, "comments": 87 },
    ...
  ],
  "count": 30
}
```

## 工作原理

1. 通过守护进程导航到 `https://news.ycombinator.com`。
2. 读取页面 HTML。
3. 解析每条新闻行(HN 稳定的 `tr.athing` 结构)为类型化的 `Story` 记录。
4. 在 stdout 上输出单个 JSON 文档。

## 为什么这是参考技能

`hackernews-frontpage` 是最小且有趣的浏览器技能(browser-skill):无需鉴权、HTML 稳定、输出确定、对文件友好。每个 Phase 1 组件(SDK、scoped tokens、三级查询、spawn 生命周期)都会被 `$B skill run hackernews-frontpage` 以及随附的 `script.test.ts` 跑一遍。

当 HN 的 HTML 调整导致我们的选择器失效时,测试会在用户察觉之前,基于已抓取的 fixture 失败。这正是重点所在。
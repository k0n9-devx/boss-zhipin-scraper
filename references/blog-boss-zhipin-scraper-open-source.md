---
title: 我做了一个 BOSS直聘抓取工具，开源了
author: eatmoreduck
date: 2026-06-11
source: https://blog.xiaohuangyu.space/p/boss-zhipin-scraper-open-source/
tags: [Open-Source, Python, Web Scraping, BOSS直聘, Career]
---

# 我做了一个 BOSS直聘抓取工具，开源了

> 本文转录自作者博客 [blog.xiaohuangyu.space](https://blog.xiaohuangyu.space/p/boss-zhipin-scraper-open-source/)（2026-06-11，作者 eatmoreduck），存档在仓库内便于离线阅读与 agent 理解项目背景。版权归原作者所有。
>
> **注意**：文中「在页面里注入 JS 同步调用搜索 API」描述的是 v2.x 早期实现。当前代码已改为通过 CDP `Network` 域**被动捕获**页面自身发出的 `joblist.json` 响应，翻页靠滚动触发无限加载（每页 15 条），见 `AGENTS.md`「架构关键点」与 `CHANGELOG.md`（#53/#30）。以当前 `README.md` 和代码为准。

**摘要**：开源了一个基于 Chrome CDP 的 BOSS直聘职位抓取工具，绕过字体反爬拿到明文薪资，输出 JSON/CSV。支持持久隔离 Chrome profile、筛选、详情页、增量写入，也可以作为 Hermes Agent Skill 使用。

## 起因

最近在看机会，天天刷 BOSS直聘。刷多了就想：这些职位数据能不能批量拉下来分析一下？比如哪些技能出现频率最高、不同城市的薪资分布、哪些公司在招 AI Agent 方向的人。

但 BOSS直聘有个很烦的东西：**字体反爬**。

薪资数字在前端显示的不是正常文字，而是用一套自定义字体渲染的。你抓下来看到的是方块或者乱码。很多爬虫教程告诉你去解析 woff 字体文件做映射，但 BOSS直聘的字体是动态生成的，每次访问都不一样，这条路基本走不通。

后来我发现了一个更简单的办法。

## 核心思路：绕过前端，直接调 API

BOSS直聘的前端页面用了字体反爬，但它自己的前端代码也要拿到薪资数据来渲染。怎么拿的？通过一个内部 API：

```
/wapi/zpgeek/search/joblist.json
```

这个接口返回的 JSON 里有 `salaryDesc` 字段——**明文薪资**，比如 `30-60K·15薪`。字体反爬只是前端渲染时加的一层，API 层面根本没有。

所以思路就简单了：不要去解析 HTML、不要去破解字体，直接调这个 API 就行。

但问题是这个 API 有 cookie 校验，你得在已登录的 BOSS直聘页面上下文里才能调通。这就引出了技术方案：Chrome CDP。

## 技术方案：Chrome DevTools Protocol

CDP（Chrome DevTools Protocol）就是 Chrome 浏览器的远程调试协议。你平时按 F12 打开的 DevTools 就是通过这个协议和 Chrome 通信的。

用 CDP 可以做到：

1. 连接到一个正在运行的 Chrome 实例
2. 在任意页面里执行 JavaScript
3. 读取页面数据、模拟用户操作

我们的做法是：

1. 启动一个开了 CDP 端口的 Chrome（使用 BOSS 专用 profile）
2. 通过 WebSocket 连接 CDP
3. 在 BOSS直聘页面里注入 JS，用 `XMLHttpRequest` 同步调用那个搜索 API
4. 拿到 JSON 结果，提取明文薪资和其他字段

整个过程就像你在浏览器控制台里手动敲代码一样，只不过自动化了。

## Chrome profile：后来改成了隔离目录

这里踩过一个坑。

最早我想复用自己的 Chrome 登录态，甚至尝试过软链接 profile。这个思路看起来省事，但风险太高：CDP Chrome 如果指向主 Chrome profile，理论上能看到那一整个浏览器环境里的 cookie 和登录态。更糟的是，Chrome 自己也不喜欢多个进程同时碰同一个 profile，容易出现锁冲突或者登录态异常。

所以现在的方案改成了持久隔离 profile：

```text
~/.boss-zhipin-scraper/chrome-profile
```

`--setup-chrome` 会创建或复用这个目录，然后启动一个 BOSS 专用 Chrome。首次使用时，在这个专用 Chrome 里登录 zhipin.com。登录态会留在这个目录里，重启机器后也还在；重复执行 `--setup-chrome` 不会清空它。

默认情况下，它不会软链接你的主 Chrome，也不会复制你的 Gmail、GitHub 或其他网站 cookie。只有你显式加 `--copy-login-state`，才会把主 Chrome 里的 `Local State` 和 Cookie 相关文件复制到这个隔离 profile。日常使用不建议这么做，手动在专用 Chrome 里登录更干净。

## 一行命令抓取

安装好后，抓取就一行命令：

```bash
python3 scripts/boss_cdp_raw.py --keyword "AI Agent" --city 上海 --pages 3 --format csv
```

输出：

```
✓ AI Agent运行环境工程师 | 30-60K·15薪 | 上海 | 字节跳动 | 10000人以上
✓ AI Agent 开发实习生 | 300-600元/天 | 上海 | 阿里巴巴集团 | 10000人以上
✓ AI Agent工程师-Java | 35-55K | 上海 | 滴滴 | 1000-9999人
...
```

## 支持的功能

**搜索和筛选**

- 按关键词、城市搜索
- 按公司规模、薪资范围、工作经验、学历筛选
- 支持分页（每页 30 条，最多 10 页）

**输出格式**

- JSON：结构化数据，方便程序处理
- CSV：UTF-8 BOM 编码，Excel 直接打开不乱码

**详情页抓取**

- 默认会抓每个岗位的完整 JD
- 只想抓列表可以加 `--no-detail`
- 自动提取技能标签
- 进入详情页时会带上列表 API 返回的 `securityId` / `lid` 上下文

**增量写入**

- 每页抓完立即保存，Ctrl+C 中断不丢数据
- 按 job_id 去重，重复运行不会产生重复数据
- 默认输出到 `~/.boss-zhipin-scraper/job-result`

**合并多次抓取**

```bash
# 先抓上海的
python3 scripts/boss_cdp_raw.py --keyword "AI Agent" --city 上海 --pages 3 --output jobs.json
# 再抓北京的，合并到同一个文件
python3 scripts/boss_cdp_raw.py --keyword "AI Agent" --city 北京 --pages 3 --merge jobs.json --output jobs.json
```

**环境检查**

```bash
python3 scripts/boss_cdp_raw.py --check
```

一键检测 Chrome CDP 是否可用、Python 依赖是否齐全、BOSS直聘是否已登录。

**一键启动 Chrome**

```bash
python3 scripts/boss_cdp_raw.py --setup-chrome
```

自动创建或复用隔离 profile，启动 Chrome CDP，然后等待你登录 zhipin.com。脚本会确认搜索接口能返回明文 `salaryDesc` 后再继续。

**Smoke test**

```bash
python3 scripts/boss_cdp_raw.py --smoke-test
```

这个命令只做一次真实 Chrome/CDP + BOSS 搜索 API 检查，不写结果文件。改代码或者换机器之后，可以先用它确认环境没问题。

## 反爬策略

BOSS直聘的反爬不弱，做了几件事来降低被封的风险：

- 翻页间隔 12-22 秒随机延迟（模拟真人浏览节奏）
- 单次最多 10 页（300 条），上限保护
- 用真实 Chrome，但只使用 BOSS 专用 profile
- API 请求从页面内发起，和正常浏览行为一致
- 默认禁用 DOM fallback，避免字体反爬导致的薪资乱码混进结果。只有显式加 `--allow-dom-fallback` 才会降级到 DOM 提取

## 作为 Hermes Agent Skill 使用

这个工具也打包成了 Hermes Agent Skill。装上之后，你直接跟 AI 说"帮我搜一下上海 AI Agent 岗位"，它会自动检查环境、启动 Chrome、运行抓取、返回结果。

安装方式：

```bash
# curl 一键安装
mkdir -p ~/.hermes/skills/data-science/boss-zhipin-scraper/scripts && \
curl -sL https://raw.githubusercontent.com/eatmoreduck/boss-zhipin-scraper/master/SKILL.md \
  -o ~/.hermes/skills/data-science/boss-zhipin-scraper/SKILL.md && \
curl -sL https://raw.githubusercontent.com/eatmoreduck/boss-zhipin-scraper/master/scripts/boss_cdp_raw.py \
  -o ~/.hermes/skills/data-science/boss-zhipin-scraper/scripts/boss_cdp_raw.py
```

## 安装和使用

```bash
# 克隆仓库
git clone https://github.com/eatmoreduck/boss-zhipin-scraper.git
cd boss-zhipin-scraper

# 安装依赖
pip install -r requirements.txt

# 启动 Chrome（首次使用）
python3 scripts/boss_cdp_raw.py --setup-chrome
# 首次使用时，在弹出的 BOSS 专用 Chrome 里登录 zhipin.com

# 检查环境
python3 scripts/boss_cdp_raw.py --check

# 开始抓取
python3 scripts/boss_cdp_raw.py --keyword "AI Agent" --city 上海 --pages 3 --format csv
```

## 数据安全

说几个需要注意的点：

- `--setup-chrome` 默认使用 `~/.boss-zhipin-scraper/chrome-profile`，不软链接、不复制主 Chrome profile，也不会关闭你的主 Chrome
- `--copy-login-state` 是显式选项，不是默认行为。它会复制主 Chrome 的 Local State 和 Cookie 相关文件到隔离 profile；日常使用更建议在专用 Chrome 里手动登录
- 需要清空 BOSS 专用浏览器登录态时，用 `--reset-chrome-profile`
- 这个工具仅用于个人求职研究，不要拿去大规模爬取
- 尊重 robots.txt 和网站使用条款

## 仓库地址

[https://github.com/eatmoreduck/boss-zhipin-scraper](https://github.com/eatmoreduck/boss-zhipin-scraper)

觉得有用的话给个 star。

---
name: social-inquiry-monitor
description: Use when 用户要搜社媒采购询盘/求购信息、监控全球买家采购信号、关键词找采购商、社媒获客巡检时。组合式技能：多平台搜采购帖→意图过滤→轻度背调→可信度分级→入库/日报。不重写抓取和背调，路由到现有工具。
version: 1.0.0
---

# 社媒询盘监控（Social Inquiry Monitor）

**一句话**：输入产品关键词，去全球主流社媒/公开网络上把"正在找供应商的采购帖"捞出来，自动查证来源、轻度背调、按可信度分级，输出可直接跟进的询盘清单。

## 设计原则（别违背）

1. **组合，不重写**：抓取走社媒抓取工具 / web 搜索，背调走你的背调技能（快筛即可），入库走你的 CRM 链路。本技能只做"编排 + 采购意图过滤 + 分级"这一层。
2. **只捞公开信息**：公开群组帖子、公开社媒帖、B2B 求购板块。不碰私密群、不冒充买家、不绕登录。
3. **不自动发布任何内容**：发布/私信一律人来发。本技能只负责"找出来、查清楚、排好序"。
4. **证据纪律**：每条询盘必须带来源 URL + 发帖时间；查证不到的标"未确认"，不许脑补公司信息。

## Step 1：构建搜索面（关键词 × 意图词）

采购询盘 = 产品词 × 采购意图词。两组词做笛卡尔组合搜索。

**产品词**（用户提供，用你自己的品类关键词，中/英/目标市场语种各一组，示例占位 `<我方产品>`）
**采购意图词**（内置，按语种）：

| 语种 | 意图词 |
|------|--------|
| 英语 | "looking for supplier" / "looking for manufacturer" / "sourcing" / "RFQ" / "request for quotation" / "need quote for" / "who can supply" / "recommend a supplier" / "MOQ" + 询价句式 |
| 西语 | "buscando proveedor" / "cotización" / "necesito un proveedor" |
| 法语 | "recherche fournisseur" / "demande de devis" |
| 阿语 | "نبحث عن مورد" / "مطلوب مورد" |

**渠道搜索式**（用任一搜索引擎工具逐个跑）：
```
<我方产品> ("looking for supplier" OR "need quote" OR sourcing) site:facebook.com
<我方产品> ("looking for" OR "sourcing" OR RFQ) site:linkedin.com/posts
<我方产品> (supplier OR sourcing) site:reddit.com
<我方产品> ("buscando proveedor" OR "cotización")
<我方产品> (buy OR purchase OR procurement) ("wholesale" OR "distributor wanted") — 新兴市场优先
```

**Facebook 群组深挖**（公开群组，免费抓取工具即可）：
- 群组定位词：`<行业词> wholesale/distributors + 目标国家名`（如 `<行业词> wholesale + 国家名`、`<行业词> distributors + 国家名`）
- 拉公开帖子后在文本里搜意图词

**X/Twitter**：有 X 搜索工具（grok-search / gooseworks 等）走原生通道；未配则用 web 搜索 `site:x.com` 降级。

## Step 2：采购意图过滤（防卖家帖混入）

对每条命中的帖子打分，**≥3 分才算询盘**：

| 信号 | 分 |
|------|----|
| 明确采购动词（looking for / sourcing / need / buscando） | +2 |
| 询价句式（quote / price / MOQ / lead time / cotización） | +2 |
| 采购主体身份词（we are a distributor / we need for our market / our project） | +1 |
| 时间信号（this month / urgent / new project） | +1 |
| **卖家信号（出现即扣分）**：for sale / we manufacture / factory direct / wholesale offer / 现货 | **-3** |
| 纯转发/招聘/求职 | 直接丢弃 |

**核心判别法**：看主语——**"我找供应商"=询盘，"我找经销商"=卖家**。同一句 "looking for"，主语不同方向相反。

## Step 3：轻度背调 + 可信度分级（5 分钟/家）

**只做快筛，不做全量报告**（全量交给你的深度背调技能）：

1. **来源查证**：点开原帖 URL，确认帖子真实存在、发帖人是公司账号还是个人号、发帖时间在近 90 天内
2. **主体查证**：发帖人挂的公司名/域名 → 官网存在吗？主营匹配吗？国家对吗？（web 搜索 + 网页提取足够）
3. **分级**：

| 级 | 标准 | 动作 |
|----|------|------|
| **A** | 来源真实 + 公司官网存在且主营匹配 + 明确采购意图 | 入 CRM（高优先级），进当日跟进候选 |
| **B** | 来源真实 + 公司存在但信息不全 | 入 CRM（普通优先级），备注缺什么 |
| **C** | 来源真实但主体查证不到 / 个人号 | 只进日报清单，不入库 |
| **丢** | 查证发现是卖家/同行/假帖 | 直接丢弃，不入库 |

**排除规则**（使用方按自己市场配置）：同行/竞品/已回复客户/团队抑制名单——命中即丢。默认建议排除自家市场同行为第一步。

## Step 4：输出与联动

- **询盘清单**：JSON（公司/国家/意图摘要/来源URL/时间/可信度/查证备注）落本地 `data/inquiry_<date>.json`
- **日报**：≤8 行数据驱动：`[社媒询盘]: 命中N条 → A级X/B级Y/丢弃Z | Top: 公司(国家/意图) | 全文: <路径>`
- **A 级入库**：写入你的 CRM（飞书 Bitable / Airtable / Excel / 任一表格），来源平台标"社媒询盘"，后续走你的开发信/跟进流程
- **闭环定位**：本技能是获客流水线的"社媒分支"，产出直接喂给背调和邮件触达环节

## 二期（按需扩展，别自作主张）

- cron 定时巡检（建议每日 1 次，避开你其他重任务时段）
- Instagram 通道（同类免费抓取套件）
- 视频剪辑（ffmpeg：切片/竖屏转换/字幕烧录可做）
- 自动发布（**默认不做**，封号风险；如要做，只能"生成草稿→人确认→人发"）

## Pitfalls

- 社媒搜索结果常重复（同一帖多引擎命中）→ 按 canonical URL 去重再打分
- 帖子会删 → 打分前先抓原文快照存 JSON，别只存 URL
- 意图词会误伤（"looking for distributors"是卖家招商帖不是采购帖）→ 主语判别是过滤的核心
- FB 大规模抓取要代理；日常巡检小批量不用
- X 搜索工具未配 key → 直接走 web 搜索降级，别停下来等配置
- 新闻稿/目录站/商城页不是询盘 → 卖家信号词见一个丢一个

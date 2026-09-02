# ChoosyHome 选房工具 软件需求规格说明书（SRS）

| 项目 | 内容 |
| :--- | :--- |
| **文档名称** | ChoosyHome 选房工具 软件需求规格说明书 |
| **文档版本** | V1.2 |
| **项目代号** | ChoosyHome（你专属的选房专家） |
| **编制日期** | 2026-09-01 |
| **文档状态** | 已评审 · 基线版 |
| **适用范围** | 个人使用 · 非商业产品 |
| **编制依据** | IEEE Std 830-1998《Recommended Practice for Software Requirements Specifications》 |
| **技术路线基线** | Next.js 15 + React 19 + Supabase + PWA |

---

## 目录

1. [引言](#1-引言)
2. [总体描述](#2-总体描述)
3. [具体功能需求](#3-具体功能需求)
4. [非功能需求](#4-非功能需求)
5. [数据模型](#5-数据模型)
6. [接口需求](#6-接口需求)
7. [附录](#7-附录)

> **V1.1 变更摘要**（2026-09-01）：
> 1. 共享访客权限管理从未来版本提前至 V1.0 范围（新增 3.8 共享与权限管理模块、`household_members` 与 `permissions` 表）。
> 2. 房源手动录入补充"楼龄"与"楼型"字段（`houses.building_type`、`viewings.build_year`、`viewings.building_type`）。

> **V1.2 变更摘要**（2026-09-01）：
> 1. 修复 C-3 合规约束与共享访客机制的表述冲突（明确共享访客可被授权使用 URL 解析）。
> 2. 修复 RLS 越权漏洞：访客无法将自身 user_id 写入 houses.user_id 等字段（新增 `get_writable_owner_uid()` 函数 + WITH CHECK 约束）。
> 3. 修复 NFR-SEC-01 表述（"仅所属用户可读写"已不准确）。
> 4. FR-AUTH-09 注销范围补全 `household_members/permissions/operation_logs` 及 Storage 图片清理，区分主用户与访客注销路径。
> 5. FR-VIEW-02 与 `viewings.house_id` 改为 NOT NULL，强制自动创建草稿房源并关联。
> 6. FR-IO 模块"用户"统一为主用户，共享访客不可导出。
> 7. FR-HOUSE-08 重复判定逻辑明确：从看房记录路径触发时自动关联已有房源。
> 8. FR-HOUSE-05 意向等级补全 unevaluated 枚举。
> 9. FR-HOUSE-22 软删除联动恢复机制补充说明。
> 10. 数据模型新增 `operation_logs` 表与图片总配额约束。
> 11. 补充 16 处边界情况说明（邀请状态机、多家庭组归属、PWA 缓存登出清理、跨时区、限流器存储等）。

---

## 1. 引言

### 1.1 编写目的

本规格说明书旨在完整、准确地描述 **ChoosyHome 选房工具** 的功能需求、非功能需求、数据模型、接口约束及设计边界，作为以下活动的统一依据：

- 项目技术架构选型与脚手架搭建
- 模块开发任务分解与迭代排期
- 设计评审与代码审查基线
- 验收测试（UAT）与回归测试依据
- 后续功能扩展与重构对照参照

**预期读者**：项目负责人（即用户本人）、AI 编程协作助手、未来可能参与的协作者。

### 1.2 文档范围

本说明书覆盖 ChoosyHome 选房工具 V1.2 全部功能与质量需求，包括：

- 用户账户与身份认证
- 共享访客（家人）邀请与权限管理
- 房源信息管理（候选房源库，含楼龄、楼型字段）
- 看房记录管理（实地踏勘记录，含楼龄、楼型字段）
- 候选房源综合对比
- 跨端访问与 PWA 体验
- 数据获取合规边界

**不包含**以下范围（明确排除）：

- 商业化运营功能（广告、付费会员、房源分发）
- 房产中介经纪人协作端
- 多租户/SaaS 化能力
- 大规模批量爬虫与公开数据 API 服务
- VR/AR 看房集成
- 电子合同与资金监管
- 法律文书与税费计算器（可在未来版本规划）

### 1.3 术语与定义

| 术语 | 缩写 | 定义 |
| :--- | :--- | :--- |
| PWA | — | Progressive Web App，渐进式 Web 应用，可"安装"到桌面、具备离线缓存、独立窗口、秒开体验 |
| BaaS | — | Backend as a Service，后端即服务，提供数据库/认证/存储等托管能力 |
| SRS | — | Software Requirements Specification，软件需求规格说明书 |
| 房源 | — | 一处可交易/可考察的房产标的，对应链家/贝壳上的房源详情页或用户实地踏勘的房产 |
| 候选房源 | — | 用户在选房过程中关注、纳入对比池的房源 |
| 看房记录 | — | 用户对单处房源实地踏勘后录入的观察笔记 |
| 对比清单 | — | 多处候选房源横向对比的命名集合 |
| RLS | — | Row Level Security，Postgres 行级权限，用于数据隔离 |
| SW | — | Service Worker，PWA 后台脚本，负责缓存与离线响应 |
| 主用户 | — | ChoosyHome 注册账户中数据所有者，对数据有全部权限，可邀请共享访客 |
| 共享访客 | — | 经主用户邀请的次要用户，按主用户授予的权限范围操作数据 |
| 家庭组 | — | 主用户与其邀请的共享访客组成的数据共享单元 |
| 权限矩阵 | — | 共享访客对各模块（房源/看房/对比）的操作权限配置（view/edit/delete） |
| 楼型 | — | 楼房建筑结构类型，常见分类：塔楼、板楼、塔板结合 |

### 1.4 参考资料与法规依据

| 类别 | 名称 |
| :--- | :--- |
| 需求规格标准 | IEEE Std 830-1998 |
| 数据合规 | 《中华人民共和国个人信息保护法》（PIPL, 2021） |
| 数据合规 | 《中华人民共和国数据安全法》（2021） |
| 数据合规 | 《中华人民共和国网络安全法》（2017） |
| 反不正当竞争判例 | 2022 年北京知识产权法院（2022）京 73 民终 3643 号判决（贝壳房源数据抓取案，判赔 500 万元） |
| 房源公开数据 | 链家网（`{city}.lianjia.com`）、贝壳找房（`{city}.ke.com`）公开页面 |
| 技术规范 | W3C PWA 规范、Web App Manifest、Service Worker API |
| 后端规范 | Supabase 官方文档（Postgres + Auth + Realtime + Storage） |
| 前端框架 | Next.js 15 官方文档、React 19 文档 |

### 1.5 文档概述

- 第 1 章：引言，定义文档目的、范围与术语。
- 第 2 章：总体描述，说明产品定位、用户特征、操作环境与约束。
- 第 3 章：具体功能需求，按模块列出可测试的功能点。
- 第 4 章：非功能需求，涵盖性能、安全、可用性、合规等质量属性。
- 第 5 章：数据模型，定义实体关系与字段。
- 第 6 章：接口需求，定义外部与内部接口。
- 第 7 章：附录，含字段对照表、修订记录。

---

## 2. 总体描述

### 2.1 产品愿景

**让买房者拥有一个属于自己的、轻量、跨端、合规的选房决策助手。**

不在商业平台的流量池里反复滑动，而是把分散在链家/贝壳/App 里的房源信息、实地踏勘笔记、配套观察、价格走势整合到一处；电脑前做对比，地铁上看记录，看房途中录笔记，所有数据多端同步。

### 2.2 产品定位

| 维度 | 定位 |
| :--- | :--- |
| 用户规模 | 个人单人使用（最多家人 2-3 人共享） |
| 商业模式 | 无商业化、无广告、无付费 |
| 数据所有权 | 数据归用户所有，用户可随时导出/删除 |
| 部署形态 | 单实例部署在 Vercel，关联 Supabase 免费tier |
| 产品形态 | PWA（可"安装"到桌面/任务栏，亦可浏览器直接访问） |
| 终端覆盖 | 手机浏览器、电脑浏览器、桌面安装版、移动端主屏图标 |
| 迭代节奏 | 按用户实际选房进度推进，无固定发版周期 |

### 2.3 用户类别与特征

| 用户类别 | 描述 | 主要任务 |
| :--- | :--- | :--- |
| **主用户（买房者本人）** | 唯一注册账户，全权用户 | 录入/编辑/删除房源与看房记录、生成对比、邀请共享访客并分配权限、跨端查看 |
| 共享访客（家人） | 经主用户邀请的次要用户 | 按主用户授予的权限范围（查看/编辑/删除）操作房源与看房记录 |
| 系统管理员 | 即主用户本人 | 数据备份、账户注销、权限回收 |

> V1.2 实现主用户单账户 + 共享访客（家人）权限管理。多租户/SaaS 化仍列入未来版本规划。

### 2.4 操作环境

#### 2.4.1 客户端环境

| 项目 | 最低配置 | 推荐配置 |
| :--- | :--- | :--- |
| 移动浏览器 | iOS Safari 15+/Android Chrome 100+ | iOS Safari 17+/Android Chrome 120+ |
| 桌面浏览器 | Edge/Chrome 100+ | Edge/Chrome 120+ |
| 屏幕宽度 | 360px | 390px / 1440px |
| 网络 | 4G 弱网可用 | 5G / 宽带 |
| PWA 安装 | Chrome/Edge 桌面端、Android Chrome、iOS Safari 主屏 | 同左 |

#### 2.4.2 服务端环境

| 组件 | 平台 | 计费层级 |
| :--- | :--- | :--- |
| Web 托管 | Vercel Hobby | 免费 |
| 数据库/认证/存储 | Supabase Free Tier | 免费（500MB 数据库、1GB 文件存储） |
| 域名 | 任意注册商 | ~50 元/年 |
| CDN | Vercel Edge Network 自带 | 免费 |

### 2.5 设计约束

#### 2.5.1 合规约束（强制）

- **C-1**：不得实现自动批量爬取链家/贝壳房源数据的爬虫模块。
- **C-2**：不得向第三方分发、出售或公开源自链家/贝壳的房源数据。
- **C-3**：URL 自动解析功能仅限主用户或被授予 `houses.edit` 权限的共享访客主动粘贴自有链接的场景，解析结果仅存入主用户家庭组私有库，不向任何第三方分发。
- **C-4**：URL 解析实现需对目标站点请求频率做严格限流（同一站点 ≤ 10 请求/分钟，单日 ≤ 200 请求）。
- **C-5**：用户数据需提供一键导出与永久删除能力（合规于 PIPL 第 45 条"自动化决策"相关要求）。

#### 2.5.2 技术约束

- **T-1**：前端须基于 Next.js 15 App Router 实现。
- **T-2**：UI 组件优先采用 shadcn/ui，避免引入重型组件库（如 Ant Design / Material UI）。
- **T-3**：后端数据存储须采用 Supabase（Postgres）。
- **T-4**：必须实现 PWA 能力（manifest.json + Service Worker + 离线 fallback 页）。
- **T-5**：核心数据须启用 Supabase RLS，保证数据仅所属用户可读写。
- **T-6**：免费部署，年度运行成本 ≤ 100 元（仅域名）。

#### 2.5.3 设计原则

| 原则 | 含义 |
| :--- | :--- |
| 轻量优先 | 单页 JS 包 ≤ 200KB（gzip） |
| 离线友好 | 离线状态下可查看已缓存房源与看房记录 |
| 美观简洁 | 一屏一焦点，shadcn/ui 风格，移动优先布局 |
| 个人可读 | 数据可读可导出，无锁定 |
| 合规优先 | 数据获取方式严守合规边界，宁可功能略弱 |

### 2.6 假设与依赖

#### 2.6.1 假设

- A-1：用户已注册 Supabase 账户（邮箱登录）。
- A-2：用户主要数据规模 ≤ 1000 条候选房源、≤ 500 条看房记录、≤ 50 个对比清单（在 Supabase Free Tier 内）。
- A-3：用户粘贴的房源 URL 来自公开可访问的链家/贝壳页面。
- A-4：用户使用现代浏览器（见 2.4.1），不需要兼容 IE。

#### 2.6.2 依赖

- D-1：Vercel 免费tier 服务可用性 ≥ 99.9%。
- D-2：Supabase Free Tier 服务可用性 ≥ 99.5%。
- D-3：链家/贝壳公开页面结构在 V1.0 周期内相对稳定（如页面结构变更，URL 解析器需相应维护）。
- D-4：网络可达目标房源站点（中国大陆境内）。

### 2.7 用户故事（核心场景）

#### 场景 1：电脑前选房
> 主用户在电脑浏览器打开 ChoosyHome，浏览候选房源列表，调整筛选条件（区域：朝阳；总价 500-700 万；户型：3 室），定位到 3 处候选，新建对比清单"朝阳三选"，将 3 处房源加入清单，横向对比价格/楼层/朝向/配套/楼龄，决定周末去看其中 2 处。

#### 场景 2：看房途中录入
> 主用户在地铁上用手机打开 ChoosyHome PWA，进入"新增看房记录"页面，选择已建候选房源，录入看房日期、配套（学区 ✓ / 地铁 ✓ / 商场 ✗ / 医院 ✗）、朝向（南）、楼层（中楼层）、楼龄（10 年）、便利度（4/5）、备注文字、照片 3 张。提交后数据实时同步到云端。

#### 场景 3：URL 粘贴解析
> 主用户在链家 App 看到一处心仪房源，分享链接到 ChoosyHome，进入"新增房源"页面，粘贴 URL，系统自动解析出户型/面积/总价/单价/朝向/楼层/小区名/建成年代/挂牌日期，用户补充看房意向（强/中/弱），保存到候选库。

#### 场景 4：周末复盘
> 主用户周末打开 ChoosyHome 首页，看到本周新增 5 条房源、3 条看房记录，打开对比清单"朝阳三选"，按价格/楼层/楼龄/配套评分重新排序，导出 CSV 备份到本地。

#### 场景 5：跨端无缝衔接
> 主用户早上在电脑上新增了 2 处候选房源，地铁上用手机 PWA 打开，发现数据已实时同步，可直接添加新的看房记录。

---

## 3. 具体功能需求

> 编号规则：`FR-{模块}-{序号}`，优先级分 P0（必做）/ P1（应做）/ P2（可做）。

### 3.1 用户账户与认证模块（FR-AUTH）

| 编号 | 需求描述 | 优先级 | 验收标准 |
| :--- | :--- | :--- | :--- |
| FR-AUTH-01 | 用户可通过邮箱+密码注册账户 | P0 | 注册成功后自动登录并跳转首页；密码须符合 ≥ 8 位 + 字母+数字规则 |
| FR-AUTH-02 | 用户可通过邮箱+密码登录 | P0 | 登录失败给出明确错误（账户不存在/密码错误），连续 5 次失败锁定 5 分钟 |
| FR-AUTH-03 | 用户可通过微信扫码登录（OAuth） | P1 | 走 Supabase Provider 配置，登录后绑定到邮箱账户或匿名账户升级 |
| FR-AUTH-04 | 用户登录态 7 天内自动续期 | P0 | JWT 有效期 1 小时，Refresh Token 有效期 7 天，自动刷新 |
| FR-AUTH-05 | 用户可登出 | P0 | 登出后本地缓存清理，跳转登录页 |
| FR-AUTH-06 | 用户可修改密码 | P0 | 修改密码需输入当前密码验证；修改后所有会话失效 |
| FR-AUTH-07 | 用户可重置密码（邮箱链接） | P0 | 点击"忘记密码"输入邮箱，收到一次性链接，链接 1 小时有效 |
| FR-AUTH-08 | 未登录用户访问受保护路由自动跳转登录页 | P0 | 中间件拦截 `/houses`、`/viewings`、`/compare` 等路由 |
| FR-AUTH-09 | 用户可一键注销账户并删除所有数据 | P1 | 注销前二次确认，删除该用户数据：作为主用户删除 `profiles/houses/viewings/comparisons/comparison_items/household_members/permissions/operation_logs` 及 Storage 图片；作为共享访客则解除其所有 `household_members` 关系并清空 `created_by`/`deleted_by` 标记为已注销用户 ID |
| FR-AUTH-10 | 用户可导出本人全部数据 | P1 | 一键导出 ZIP（含 JSON 与 CSV） |

### 3.2 房源信息管理模块（FR-HOUSE）

#### 3.2.1 房源新增

| 编号 | 需求描述 | 优先级 | 验收标准 |
| :--- | :--- | :--- | :--- |
| FR-HOUSE-01 | 用户可通过"粘贴 URL"自动解析房源字段 | P0 | 粘贴链家/贝壳二手房 URL，5 秒内返回解析结果（户型/面积/总价/单价/朝向/楼层/小区名/建成年代/挂牌日期/经纪人）；失败时给出明确错误 |
| FR-HOUSE-02 | 用户可手动录入房源字段（不依赖 URL） | P0 | 至少包含：城市、区域、小区、户型、面积、总价、楼龄（建成年代）、楼型（塔楼/板楼/塔板结合/其他）、URL（可选） |
| FR-HOUSE-03 | 解析后的字段可被用户修改与补充 | P0 | URL 解析后所有字段处于可编辑状态，用户可调整 |
| FR-HOUSE-04 | 用户可上传房源图片（最多 9 张） | P1 | 单张 ≤ 5MB，自动压缩为 webp，存到 Supabase Storage |
| FR-HOUSE-05 | 用户可为房源设置"意向等级"（强/中/弱/淘汰/未评估） | P0 | 枚举：strong/medium/weak/eliminated/unevaluated；默认 unevaluated，用于对比排序权重 |
| FR-HOUSE-06 | 用户可为房源添加自由文本备注 | P0 | 单条备注 ≤ 2000 字符 |
| FR-HOUSE-07 | 用户可为房源打标签（如"近地铁""学区房""电梯房"） | P1 | 标签可复用，列表筛选时可用 |
| FR-HOUSE-08 | 保存时若该城市+区域+小区已有未软删除房源则提示"已有该小区房源" | P2 | 城市同区域同小区视为重复；用户可选"查看已有"或"仍创建"。从看房记录录入路径触发 FR-VIEW-02 时自动关联已有房源，不创建草稿 |

#### 3.2.2 房源列表与筛选

| 编号 | 需求描述 | 优先级 | 验收标准 |
| :--- | :--- | :--- | :--- |
| FR-HOUSE-10 | 房源列表分页展示，默认 20 条/页 | P0 | 列表项展示：小区名、户型、面积、总价、单价、意向等级、看房次数、最近看房日期 |
| FR-HOUSE-11 | 支持多条件组合筛选 | P0 | 筛选维度：城市、区域、户型、面积区间、总价区间、单价区间、意向等级、标签 |
| FR-HOUSE-12 | 支持按字段排序 | P0 | 排序字段：录入时间、总价、单价、面积、最近看房日期、意向等级；升降序可切换 |
| FR-HOUSE-13 | 支持关键词搜索 | P0 | 搜索范围：小区名、备注、标签；模糊匹配 |
| FR-HOUSE-14 | 列表项可一键加入/移出对比清单 | P0 | 实时反映到清单数量徽标 |
| FR-HOUSE-15 | 列表视图与卡片视图可切换 | P2 | 默认卡片视图，移动端单列 |
| FR-HOUSE-16 | 移动端支持下拉刷新与无限滚动 | P1 | 替代分页器 |

#### 3.2.3 房源详情与编辑

| 编号 | 需求描述 | 优先级 | 验收标准 |
| :--- | :--- | :--- | :--- |
| FR-HOUSE-20 | 房源详情页展示完整字段 | P0 | 含基本信息、配套、价格走势（如有）、图片、标签、备注、关联看房记录列表 |
| FR-HOUSE-21 | 用户可编辑房源任意字段 | P0 | 编辑后保留修改时间戳与修改人 |
| FR-HOUSE-22 | 用户可删除房源（软删除） | P0 | 软删除 30 天后物理清除；删除前二次确认；如有关联未软删除的看房记录则一并软删除；如该房源在对比清单中，对比清单中标记"已软删除"并允许保留以备复盘 |
| FR-HOUSE-23 | 用户可恢复 30 天内删除的房源 | P2 | 在"回收站"页面恢复；恢复时一并恢复 30 天内被联动软删除的看房记录；超 30 天已物理清除的看房记录不可恢复 |
| FR-HOUSE-24 | 详情页展示该房源的看房历史时间线 | P0 | 按时间倒序，最近 1 条摘要置顶 |
| FR-HOUSE-25 | 详情页可一键发起新增看房记录（自动带房源信息） | P0 | 跳转到看房记录表单，房源字段已预填 |

### 3.3 看房记录管理模块（FR-VIEW）

#### 3.3.1 看房记录录入

| 编号 | 需求描述 | 优先级 | 验收标准 |
| :--- | :--- | :--- | :--- |
| FR-VIEW-01 | 用户可关联已有候选房源录入看房记录 | P0 | 关联后该记录出现在房源详情页时间线 |
| FR-VIEW-02 | 用户可为未在候选库的房源录入看房记录（自动创建意向等级为 unevaluated 的草稿房源并关联） | P1 | 草稿房源 `intention_level='unevaluated'`，`viewings.house_id` 不可为空，必须关联到草稿或已有房源 |
| FR-VIEW-03 | 录入字段：城市、区域、小区、户型、面积、楼层、楼龄、楼型（塔楼/板楼/塔板结合/其他）、朝向 | P0 | 详见数据模型 5.2 |
| FR-VIEW-04 | 录入字段：配套基础设施勾选（学区/医院/商场/地铁/公园） | P0 | 多选勾选，可补充距离说明（如"地铁 300m"） |
| FR-VIEW-05 | 录入字段：出行便利度评分（1-5 星） | P0 | 单选必填 |
| FR-VIEW-06 | 录入字段：综合评分（1-10） | P0 | 单选必填，用于后续对比排序 |
| FR-VIEW-07 | 录入字段：看房日期、看房时段（上午/下午/晚上） | P0 | 默认今日 |
| FR-VIEW-08 | 录入字段：自由文本备注（≤ 5000 字符） | P0 | 支持基础 Markdown（粗体/列表/标题） |
| FR-VIEW-09 | 录入字段：现场照片（最多 9 张） | P1 | 自动压缩 webp，存到 Storage |
| FR-VIEW-10 | 录入字段：陪同人/经纪人姓名与电话（可选） | P2 | 自动格式化电话号；不可被导出 CSV |
| FR-VIEW-11 | 录入字段：价格观察（挂牌价、用户心理价、还价空间） | P1 | 三字段组合，用于后续策略 |
| FR-VIEW-12 | 表单支持断点续录（草稿自动保存） | P1 | 每 30 秒本地草稿，刷新/退出可恢复 |
| FR-VIEW-13 | 移动端表单适配：单字段分屏滚动、键盘弹起不遮挡输入 | P0 | 移动端友好 |

#### 3.3.2 看房记录列表与编辑

| 编号 | 需求描述 | 优先级 | 验收标准 |
| :--- | :--- | :--- | :--- |
| FR-VIEW-20 | 看房记录时间线视图（按日期倒序） | P0 | 卡片展示小区、日期、综合评分、首图 |
| FR-VIEW-21 | 支持按城市、区域、日期区间、综合评分区间筛选 | P0 | 与房源列表筛选一致交互 |
| FR-VIEW-22 | 用户可编辑看房记录任意字段 | P0 | 修改后保留修改时间戳 |
| FR-VIEW-23 | 用户可删除看房记录（软删除） | P0 | 同 FR-HOUSE-22 |
| FR-VIEW-24 | 看房记录可在地图上标注（调用高德地图） | P2 | 仅展示，不做导航 |

### 3.4 候选房源综合对比模块（FR-CMP）

| 编号 | 需求描述 | 优先级 | 验收标准 |
| :--- | :--- | :--- | :--- |
| FR-CMP-01 | 用户可创建对比清单（命名 + 备注） | P0 | 一个清单最多含 4 处房源 |
| FR-CMP-02 | 用户可从房源列表/详情页"加入对比" | P0 | 加入后清单徽标实时更新 |
| FR-CMP-03 | 对比页以表格形式横向展示候选房源 | P0 | 行=字段，列=房源；移动端转纵向卡片对比 |
| FR-CMP-04 | 对比字段可自定义勾选展示 | P0 | 默认含：总价、单价、面积、户型、朝向、楼层、楼龄、综合评分、配套勾选、最近看房日期 |
| FR-CMP-05 | 对比页支持为每处房源在清单内打分（1-10）与权重 | P1 | 自定义评分项：如"价格 30%、户型 20%、配套 30%、通勤 20%" |
| FR-CMP-06 | 对比页可按加权得分排序 | P1 | 加权得分实时计算展示 |
| FR-CMP-07 | 用户可为对比清单添加结论备注 | P1 | 单条 ≤ 2000 字符，支持 Markdown |
| FR-CMP-08 | 对比清单可导出为 PDF / PNG 截图 | P2 | 用于分享给家人讨论 |
| FR-CMP-09 | 对比清单可导出 CSV | P1 | 字段对照附录 7.3 |
| FR-CMP-10 | 用户可删除对比清单（不删除房源本身） | P0 | 删除前二次确认 |
| FR-CMP-11 | 清单中可一键移除某房源 | P0 | 不影响房源本身 |

### 3.5 数据导入导出模块（FR-IO）

> 本模块"用户"特指主用户。共享访客不可触发任何导出操作（见 FR-SHARE-23）。

| 编号 | 需求描述 | 优先级 | 验收标准 |
| :--- | :--- | :--- | :--- |
| FR-IO-01 | 主用户可一键导出全部数据为 ZIP | P1 | 含 JSON 全量 + CSV 表格 + 图片目录 |
| FR-IO-02 | 主用户可导出单个对比清单为 CSV | P1 | 字段对齐附录 7.3 |
| FR-IO-03 | 主用户可从 CSV 批量导入房源 | P2 | 模板下载 + 字段映射校验 + 错误报告 |
| FR-IO-04 | 主用户可导出看房记录为 Markdown 文档 | P2 | 一房源一章节，按日期排序 |

### 3.6 PWA 与跨端访问模块（FR-PWA）

| 编号 | 需求描述 | 优先级 | 验收标准 |
| :--- | :--- | :--- | :--- |
| FR-PWA-01 | 提供 `manifest.json` 与图标资源 | P0 | 含 192/512/maskable 多尺寸，name/short_name/theme_color/background_color |
| FR-PWA-02 | 注册 Service Worker 缓存核心资源 | P0 | App Shell + 静态资源预缓存，二次访问秒开 |
| FR-PWA-03 | 离线状态下显示离线 fallback 页 | P0 | 离线访问任意路由时显示"已离线"页面 + 已缓存房源可查看 |
| FR-PWA-04 | 桌面 Chrome/Edge 触发"安装应用"提示 | P0 | beforeinstallprompt 事件正确触发 |
| FR-PWA-05 | iOS Safari 支持添加到主屏 | P0 | apple-touch-icon 配置正确，打开后独立窗口无地址栏 |
| FR-PWA-06 | 安装后启动展示 Splash 启动画面 | P1 | manifest 配置 background_color + icons |
| FR-PWA-07 | 数据多端实时同步 | P0 | Supabase Realtime 订阅，新增/编辑后其他端 2 秒内可见 |
| FR-PWA-08 | 长表单支持断网自动重试提交 | P1 | 离线录入暂存 IndexedDB，恢复网络后自动同步 |

### 3.7 设置与个人化模块（FR-SET）

| 编号 | 需求描述 | 优先级 | 验收标准 |
| :--- | :--- | :--- | :--- |
| FR-SET-01 | 用户可在设置页查看账户信息 | P0 | 邮箱、注册时间、最近登录、数据条数统计 |
| FR-SET-02 | 用户可切换主题（浅色/深色/跟随系统） | P1 | localStorage 持久化 + Tailwind dark mode |
| FR-SET-03 | 用户可设置默认城市（影响首页默认筛选） | P1 | 单选必填 |
| FR-SET-04 | 用户可配置看房记录评分项自定义 | P2 | 加分项名称与权重 |
| FR-SET-05 | 用户可查看操作日志 | P2 | 最近 100 条增删改记录 |
| FR-SET-06 | 用户可清空所有缓存 | P2 | 不影响云端数据，仅清理本地 |

### 3.8 共享与权限管理模块（FR-SHARE）

#### 3.8.1 家庭成员邀请

| 编号 | 需求描述 | 优先级 | 验收标准 |
| :--- | :--- | :--- | :--- |
| FR-SHARE-01 | 主用户可通过邮箱邀请共享访客（家人） | P0 | 邀请链接 7 天有效，被邀请人注册后自动绑定到主用户家庭组 |
| FR-SHARE-02 | 主用户可查看已邀请的成员列表 | P0 | 列表项：邮箱、昵称、加入时间、当前权限、最近活跃时间 |
| FR-SHARE-03 | 主用户可取消未接受的邀请 | P0 | 取消后邀请链接失效 |
| FR-SHARE-04 | 共享访客接受邀请后自动获得"只读"默认权限 | P0 | 默认权限为 view，主用户可后续调整 |
| FR-SHARE-05 | 单个主用户最多邀请 5 位共享访客 | P0 | 防止滥用，符合家庭场景 |

#### 3.8.2 权限授予与回收

| 编号 | 需求描述 | 优先级 | 验收标准 |
| :--- | :--- | :--- | :--- |
| FR-SHARE-10 | 主用户可为每位共享访客单独设置权限 | P0 | 权限粒度：房源(view/edit/delete)、看房记录(view/edit/delete)、对比清单(view/edit/delete) |
| FR-SHARE-11 | 权限以矩阵形式展示与编辑 | P0 | 行=模块，列=操作，单元格勾选 |
| FR-SHARE-12 | 权限变更即时生效 | P0 | 通过 Supabase Realtime 推送，被授权方 2 秒内生效 |
| FR-SHARE-13 | 主用户可一键回收某成员所有权限 | P0 | 回收后该成员立即失去所有访问能力 |
| FR-SHARE-14 | 主用户可移除共享访客（解除家庭组关系） | P0 | 移除后该账户变为独立账户，不再可见主用户数据 |
| FR-SHARE-15 | 共享访客不可邀请其他成员 | P0 | 仅主用户有邀请权 |
| FR-SHARE-16 | 共享访客不可修改他人权限 | P0 | 仅主用户有权限管理权 |

#### 3.8.3 共享访客操作约束

| 编号 | 需求描述 | 优先级 | 验收标准 |
| :--- | :--- | :--- | :--- |
| FR-SHARE-20 | 共享访客的操作受权限矩阵约束 | P0 | 无 view 权限的模块不可见；无 edit 权限的模块只读；无 delete 权限的删除按钮置灰 |
| FR-SHARE-21 | 共享访客编辑的数据自动标记作者 | P0 | houses/viewings 表 created_by 字段记录操作者 user_id |
| FR-SHARE-22 | 共享访客删除操作走软删除并记录操作者 | P0 | deleted_by 字段记录操作者 user_id |
| FR-SHARE-23 | 共享访客不可导出数据 | P1 | 仅主用户可触发全量导出 |
| FR-SHARE-24 | 共享访客不可注销主账户 | P0 | 仅主用户可注销 |

---

## 4. 非功能需求

### 4.1 性能需求

| 编号 | 指标 | 目标值 |
| :--- | :--- | :--- |
| NFR-PERF-01 | 首屏加载时间（LCP） | 移动 4G ≤ 2.5s；桌面宽带 ≤ 1.5s |
| NFR-PERF-02 | 页面切换交互（INP） | ≤ 200ms |
| NFR-PERF-03 | 累计布局偏移（CLS） | ≤ 0.1 |
| NFR-PERF-04 | 单页 JS Bundle（gzip） | ≤ 200KB（首屏） |
| NFR-PERF-05 | 图片懒加载 | 视口外图片延迟加载，webp 格式 |
| NFR-PERF-06 | URL 解析响应时间 | P95 ≤ 5s（含目标站点请求） |
| NFR-PERF-07 | 列表查询响应时间 | 1000 条内 P95 ≤ 500ms |
| NFR-PERF-08 | 多端数据同步延迟 | ≤ 2s |

### 4.2 安全需求

| 编号 | 需求 |
| :--- | :--- |
| NFR-SEC-01 | 所有数据库表启用 RLS，数据仅所属用户（主用户）及其授权的活跃共享访客可读写，访客写入 user_id 必须为主用户 ID（防越权） |
| NFR-SEC-02 | 所有用户输入需经 zod schema 校验，防止 SQL 注入与 XSS |
| NFR-SEC-03 | URL 解析 API 需登录鉴权（中间件 + Supabase getUser） |
| NFR-SEC-04 | URL 解析目标站点域名白名单：仅 `*.lianjia.com`、`*.ke.com` |
| NFR-SEC-05 | URL 解析频率限流：同一用户 10 请求/分钟、200 请求/日 |
| NFR-SEC-06 | 用户密码不在客户端存储，走 Supabase Auth 托管 |
| NFR-SEC-07 | HTTPS 强制启用（Vercel 自动配置 + HSTS） |
| NFR-SEC-08 | 敏感字段（如经纪人电话）不进入导出 CSV |
| NFR-SEC-09 | 用户上传图片经服务端 MIME 校验，禁止可执行文件 |
| NFR-SEC-10 | Supabase Service Role Key 仅在 Server 端使用，不进入客户端 Bundle |

### 4.3 可用性需求

| 编号 | 需求 |
| :--- | :--- |
| NFR-USE-01 | 响应式布局：360px / 768px / 1024px / 1440px 四档断点 |
| NFR-USE-02 | 移动端单手可操作：主要按钮位于屏幕下半部 |
| NFR-USE-03 | 表单输入支持移动端键盘适配（inputmode、autofocus） |
| NFR-USE-04 | 关键操作（删除/注销）二次确认 |
| NFR-USE-05 | 错误提示友好：不展示技术堆栈，给"建议操作" |
| NFR-USE-06 | 空状态有引导插画与 CTA |
| NFR-USE-07 | 加载状态骨架屏 + 进度提示 |
| NFR-USE-08 | 符合 WCAG 2.1 AA：色彩对比 ≥ 4.5:1，键盘可达 |
| NFR-USE-09 | 中文界面，无中英文混排不一致 |

### 4.4 可靠性需求

| 编号 | 需求 |
| :--- | :--- |
| NFR-REL-01 | 服务可用性 ≥ 99%（依赖 Vercel + Supabase SLA） |
| NFR-REL-02 | 网络异常时本地可继续录入，恢复网络后自动同步 |
| NFR-REL-03 | 表单提交失败自动重试 3 次，指数退避 |
| NFR-REL-04 | 数据库每日自动备份（Supabase 免费 7 天 PITR） |
| NFR-REL-05 | 用户可手动触发云端数据全量导出作为本地备份 |

### 4.5 可维护性需求

| 编号 | 需求 |
| :--- | :--- |
| NFR-MAINT-01 | 代码组织：`app/` 路由 / `components/` UI / `lib/` 工具 / `types/` 类型 |
| NFR-MAINT-02 | TypeScript strict 模式启用，禁用 any |
| NFR-MAINT-03 | ESLint + Prettier 强制规范 |
| NFR-MAINT-04 | 提交前 husky + lint-staged 钩子 |
| NFR-MAINT-05 | 关键业务逻辑（评分计算、URL 解析）单元测试覆盖率 ≥ 70% |
| NFR-MAINT-06 | 数据库 schema 用 Supabase migration 管理，版本化 |
| NFR-MAINT-07 | 环境变量区分 `.env.local` / `.env.production` |

### 4.6 合规性需求（强制）

| 编号 | 需求 |
| :--- | :--- |
| NFR-COMP-01 | 不实现任何主动批量爬虫模块 |
| NFR-COMP-02 | URL 解析仅限用户本人主动粘贴，单次单条 |
| NFR-COMP-03 | URL 解析请求频率严守 4.2 NFR-SEC-05 限流 |
| NFR-COMP-04 | 解析结果不向任何第三方分发、出售、公开 |
| NFR-COMP-05 | 用户可一键注销账户并删除所有数据（PIPL 第 47 条） |
| NFR-COMP-06 | 用户数据可一键导出（PIPL 第 45 条"自动化决策"相关要求） |
| NFR-COMP-07 | 隐私政策页面公示数据采集与使用范围 |
| NFR-COMP-08 | 不收集非必要个人信息（位置、设备 ID 等） |

### 4.7 可移植性需求

| 编号 | 需求 |
| :--- | :--- |
| NFR-PORT-01 | 部署不依赖特定云厂商专有服务（除 Supabase 外可平替为 Postgres + Auth） |
| NFR-PORT-02 | 数据库 schema 与应用代码解耦，可独立迁移 |
| NFR-PORT-03 | 用户数据导出格式为标准 JSON / CSV，无锁定 |

---

## 5. 数据模型

### 5.1 实体关系图（ERD）

```
┌─────────────┐        ┌────────────────────┐
│   profiles  │ 1─────N│ household_members  │
│  (用户)     │        │  (家庭成员)        │
└─────────────┘        └────────────────────┘
        │ 1                     │ 1
        │                       │
        ▼                       ▼ N
┌──────────────┐        ┌────────────────────┐
│   houses     │ 1      │   permissions      │
│  (候选房源)  │ ────┐  │  (权限授予)        │
└──────────────┘     │  └────────────────────┘
      │ 1             │
      │               │
      ▼ N             ▼ N
┌──────────────┐   ┌──────────────────┐
│  viewings     │   │ comparison_items  │
│ (看房记录)    │ N │  (对比项)         │
└──────────────┘   └──────────────────┘
                            │ N
                            │
                            ▼ 1
                    ┌──────────────────┐
                    │  comparisons     │
                    │  (对比清单)      │
                    └──────────────────┘
```

**关系说明**：
- `profiles` 1:N `household_members`（主用户邀请的共享访客）
- `household_members` 1:N `permissions`（每位成员对各模块的权限）
- `profiles` 1:N `houses`、`viewings`、`comparisons`（数据所有权）
- `houses` 1:N `viewings`、`comparison_items`（业务关联）

### 5.2 实体定义

#### 5.2.1 `profiles`（用户扩展表）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| id | uuid | PK, FK→auth.users | 用户唯一 ID |
| email | text | NOT NULL | 邮箱 |
| default_city | text | — | 默认城市 |
| theme | text | default='system' | 主题：light/dark/system |
| created_at | timestamptz | default now() | 注册时间 |
| last_login_at | timestamptz | — | 最近登录 |

#### 5.2.2 `houses`（候选房源）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| id | uuid | PK | 房源 ID |
| user_id | uuid | FK→profiles.id, NOT NULL | 所属用户 |
| source | text | NOT NULL | 来源：lianjia/beike/manual |
| source_url | text | — | 原始 URL（如来自链家） |
| city | text | NOT NULL | 城市 |
| district | text | — | 区域 |
| community | text | NOT NULL | 小区名 |
| layout | text | — | 户型（如"3 室 2 厅"） |
| area | numeric | — | 建筑面积（㎡） |
| total_price | numeric | — | 总价（万元） |
| unit_price | numeric | — | 单价（元/㎡） |
| orientation | text | — | 朝向 |
| floor | text | — | 楼层描述 |
| total_floor | int | — | 总楼层 |
| build_year | int | — | 建成年代（楼龄） |
| building_type | text | — | 楼型：tower（塔楼）/slab（板楼）/mixed（塔板结合）/other |
| listing_date | date | — | 挂牌日期 |
| transaction_price | numeric | — | 成交价（如已成交） |
| transaction_date | date | — | 成交日期 |
| lowest_price | numeric | — | 历史最低价 |
| intention_level | text | default='unevaluated' | 意向：strong/medium/weak/eliminated/unevaluated |
| tags | text[] | default='{}' | 标签数组 |
| notes | text | — | 备注 |
| images | jsonb | default='[]' | 图片路径数组 |
| is_deleted | bool | default=false | 软删除标记 |
| deleted_at | timestamptz | — | 删除时间 |
| created_by | uuid | FK→auth.users | 创建者（主用户或共享访客） |
| deleted_by | uuid | FK→auth.users, nullable | 删除操作者 |
| created_at | timestamptz | default now() | 创建时间 |
| updated_at | timestamptz | default now() | 更新时间 |

#### 5.2.3 `viewings`（看房记录）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| id | uuid | PK | 记录 ID |
| user_id | uuid | FK→profiles.id, NOT NULL | 所属用户 |
| house_id | uuid | FK→houses.id, NOT NULL | 关联房源（未在候选库时由系统自动创建草稿房源并关联） |
| city | text | NOT NULL | 看房城市 |
| district | text | — | 区域 |
| community | text | — | 小区 |
| layout | text | — | 户型 |
| area | numeric | — | 面积 |
| floor | text | — | 楼层 |
| build_year | int | — | 楼龄（建成年代） |
| building_type | text | — | 楼型：tower（塔楼）/slab（板楼）/mixed（塔板结合）/other |
| orientation | text | — | 朝向 |
| has_school | bool | default=false | 学区 |
| has_hospital | bool | default=false | 医院 |
| has_mall | bool | default=false | 商场 |
| has_subway | bool | default=false | 地铁 |
| has_park | bool | default=false | 公园 |
| facility_notes | jsonb | default='{}' | 配套距离备注 |
| commute_score | int | CHECK 1-5 | 出行便利度 |
| overall_score | int | CHECK 1-10, NOT NULL | 综合评分 |
| viewing_date | date | NOT NULL | 看房日期 |
| viewing_period | text | — | 时段：morning/afternoon/evening |
| listing_price | numeric | — | 挂牌价 |
| target_price | numeric | — | 心理价 |
| bargain_space | numeric | — | 还价空间 |
| notes | text | — | 备注（Markdown） |
| images | jsonb | default='[]' | 图片路径数组 |
| agent_name | text | — | 经纪人姓名 |
| agent_phone | text | — | 经纪人电话（不导出 CSV） |
| is_deleted | bool | default=false | 软删除 |
| deleted_at | timestamptz | — | 删除时间 |
| created_by | uuid | FK→auth.users | 创建者（主用户或共享访客） |
| deleted_by | uuid | FK→auth.users, nullable | 删除操作者 |
| created_at | timestamptz | default now() | 创建时间 |
| updated_at | timestamptz | default now() | 更新时间 |

#### 5.2.4 `comparisons`（对比清单）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| id | uuid | PK | 清单 ID |
| user_id | uuid | FK→profiles.id, NOT NULL | 所属用户 |
| name | text | NOT NULL | 清单名称 |
| notes | text | — | 结论备注 |
| weight_config | jsonb | default='{}' | 评分项权重配置 |
| created_at | timestamptz | default now() | 创建时间 |
| updated_at | timestamptz | default now() | 更新时间 |

#### 5.2.5 `comparison_items`（对比项）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| id | uuid | PK | 项 ID |
| comparison_id | uuid | FK→comparisons.id, ON DELETE CASCADE | 所属清单 |
| house_id | uuid | FK→houses.id | 关联房源 |
| score | int | CHECK 1-10 | 该房源在本清单的评分 |
| sort_order | int | default=0 | 显示顺序 |
| created_at | timestamptz | default now() | 创建时间 |

#### 5.2.6 `household_members`（家庭成员）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| id | uuid | PK | 成员关系 ID |
| owner_id | uuid | FK→auth.users, NOT NULL | 主用户 ID（数据所有者） |
| member_id | uuid | FK→auth.users, NOT NULL | 共享访客 ID |
| nickname | text | — | 主用户给访客起的昵称 |
| status | text | default='pending', CHECK in ('pending','active','expired','revoked') | 状态：待接受/活跃/过期/撤销 |
| invited_at | timestamptz | default now() | 邀请时间 |
| accepted_at | timestamptz | — | 接受邀请时间 |
| revoked_at | timestamptz | — | 撤销时间 |
| last_active_at | timestamptz | — | 最近活跃时间 |
| 唯一约束 | — | UNIQUE(owner_id, member_id) | 同一主用户对同一访客仅一条记录 |
| CHECK 约束 | — | CHECK (owner_id <> member_id) | 防止主用户邀请自己 |
| CHECK 约束 | — | CHECK (status='active' OR (member_id NOT IN (SELECT id FROM household_members WHERE status='active' AND owner_id <> this.owner_id)) | 防止同一访客在多个家庭组同时 active（可选，由应用层保证） |

> **状态机**：pending →（用户接受）→ active →（主用户撤销 OR 用户主动退出）→ revoked；pending →（7 天后由定时任务）→ expired。
> **多家庭组归属**：一个 member_id 可被多个 owner_id 邀请，但仅能有一个 status='active' 的关系；访客登录后默认看到首个 active 家庭组数据，可在设置页"切换家庭组"。

#### 5.2.7 `permissions`（权限授予）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| id | uuid | PK | 权限 ID |
| household_member_id | uuid | FK→household_members.id, ON DELETE CASCADE | 所属成员关系 |
| module | text | NOT NULL | 模块：houses/viewings/comparisons |
| can_view | bool | default=true | 可见 |
| can_edit | bool | default=false | 可编辑 |
| can_delete | bool | default=false | 可删除 |
| updated_at | timestamptz | default now() | 权限变更时间 |
| 唯一约束 | — | UNIQUE(household_member_id, module) | 每成员每模块一条权限记录 |

> **初始化规则**：household_members 由 pending→active 时（用户接受邀请），系统自动为每个模块（houses/viewings/comparisons）插入一条 permissions 记录，默认 `can_view=true, can_edit=false, can_delete=false`（即默认只读）。
> **5 位上限约束**：每个 owner_id 的 status IN ('pending','active') 记录数 ≤ 5，由应用层校验 + 触发器 RAISE EXCEPTION 保证。

#### 5.2.8 `operation_logs`（操作日志）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| id | uuid | PK | 日志 ID |
| user_id | uuid | FK→auth.users, NOT NULL | 数据所属主用户 |
| operator_id | uuid | FK→auth.users, NOT NULL | 操作者（主用户或访客） |
| module | text | NOT NULL | 模块：houses/viewings/comparisons/household/permissions |
| record_id | uuid | NOT NULL | 被操作记录 ID |
| action | text | NOT NULL | 操作：create/update/delete/restore |
| diff | jsonb | — | 修改前后字段差异（仅 update） |
| created_at | timestamptz | default now() | 操作时间 |

> 用途：支持 FR-SET-05 操作日志查看（最近 100 条），保留 90 天后自动清理。
> 4 处房源上限约束：comparisons 表通过触发器校验 `comparison_items` 计数 ≤ 4。
> 图片总配额约束：每 user_id 的 houses.images + viewings.images 总数 ≤ 2000 张，约 400MB，符合 Supabase Free 1GB 限制（留 600MB 给其他文件）。超出时拒绝上传并提示用户清理或导出后删除。
> 备选方案：未来若用户图片需求增长，接入 Cloudflare R2（免费 10GB 存储）作为外部存储，不占 Supabase 额度。

### 5.3 RLS 策略

所有表启用行级安全，统一策略：

```sql
-- 示例：houses 表 RLS（主用户 + 共享访客双重授权，防越权写入）
ALTER TABLE houses ENABLE ROW LEVEL SECURITY;

-- 辅助函数：判断当前用户是否可访问某主用户的数据
CREATE OR REPLACE FUNCTION can_access_owner_data(owner_uid uuid)
RETURNS boolean AS $$
  -- 是数据所有者本人
  SELECT auth.uid() = owner_uid
  -- 或者是被授权的活跃共享访客
  OR EXISTS (
    SELECT 1 FROM household_members hm
    WHERE hm.owner_id = owner_uid
      AND hm.member_id = auth.uid()
      AND hm.status = 'active'
  )
$$ LANGUAGE sql STABLE;

-- 辅助函数：判断当前用户对某模块是否有某权限
CREATE OR REPLACE FUNCTION has_permission(owner_uid uuid, mod text, perm text)
RETURNS boolean AS $$
  -- 主用户本人拥有全部权限
  SELECT auth.uid() = owner_uid
  OR EXISTS (
    SELECT 1 FROM household_members hm
    JOIN permissions p ON p.household_member_id = hm.id
    WHERE hm.owner_id = owner_uid
      AND hm.member_id = auth.uid()
      AND hm.status = 'active'
      AND p.module = mod
      AND (
        (perm = 'view' AND p.can_view)
        OR (perm = 'edit' AND p.can_edit)
        OR (perm = 'delete' AND p.can_delete)
      )
  )
$$ LANGUAGE sql STABLE;

-- 辅助函数：判断当前用户可作为哪个 owner 写入数据
-- 防越权关键：访客写入的 houses.user_id 必须等于其归属的主用户 owner_id
CREATE OR REPLACE FUNCTION get_writable_owner_uid()
RETURNS uuid AS $$
  -- 主用户本人只能写入自己的 user_id
  SELECT auth.uid()
  UNION ALL
  -- 共享访客只能写入其归属主用户的 owner_id（防止访客写入自己 user_id 越权）
  SELECT hm.owner_id FROM household_members hm
  WHERE hm.member_id = auth.uid() AND hm.status = 'active'
$$ LANGUAGE sql STABLE;

-- houses 表策略
CREATE POLICY "可查看：主用户或被授权 view 的共享访客"
ON houses FOR SELECT
USING (can_access_owner_data(user_id));

CREATE POLICY "可新增：写入 user_id 必须为当前用户可作为 owner 的 ID"
ON houses FOR INSERT
WITH CHECK (
  has_permission(user_id, 'houses', 'edit')
  AND user_id = ANY(SELECT get_writable_owner_uid())
);

CREATE POLICY "可修改：主用户或被授权 edit 的共享访客，不可改 user_id"
ON houses FOR UPDATE
USING (has_permission(user_id, 'houses', 'edit'))
WITH CHECK (
  has_permission(user_id, 'houses', 'edit')
  AND user_id = ANY(SELECT get_writable_owner_uid())
);

CREATE POLICY "可软删除：主用户或被授权 delete 的共享访客"
ON houses FOR UPDATE
USING (has_permission(user_id, 'houses', 'delete'));
```

其他表（`viewings`、`comparisons`、`comparison_items`）采用相同策略，仅替换模块名参数。`comparison_items` 通过 `comparison_id` 关联的 `comparisons.user_id` 间接校验。

**关键安全约束**：
- INSERT/UPDATE 的 WITH CHECK 强制校验 `user_id = ANY(get_writable_owner_uid())`，访客无法将自己 user_id 写入 `houses.user_id` 字段，从根上堵住越权漏洞。
- UPDATE 时不可修改 user_id 字段（应用层加约束 + 触发器校验）。
- 前端调用 Supabase Client 时由 RLS 自动应用策略，无需手写 SQL。

**特殊表 RLS**：
- `household_members`：仅 owner_id 或 member_id 本人可读；仅 owner_id 可写。
- `permissions`：仅 household_member_id 对应的 owner_id 可读写；member_id 可读自己被授予的权限。

### 5.4 索引设计

| 表 | 索引字段 | 用途 |
| :--- | :--- | :--- |
| houses | (user_id, city, district) | 列表筛选 |
| houses | (user_id, intention_level) | 意向筛选 |
| houses | (user_id, is_deleted, deleted_at) | 回收站 |
| viewings | (user_id, viewing_date DESC) | 时间线 |
| viewings | (user_id, house_id) | 房源关联查询 |
| comparison_items | (comparison_id, sort_order) | 对比页排序 |
| household_members | (owner_id, status) | 主用户查询家庭成员 |
| household_members | (member_id, status) | 访客查询自己归属的家庭 |
| permissions | (household_member_id, module) | 权限查询 |
| operation_logs | (user_id, created_at DESC) | 操作日志时间线 |
| operation_logs | (operator_id, created_at DESC) | 按操作者筛选 |

---

## 6. 接口需求

### 6.1 外部接口

#### 6.1.1 链家/贝壳房源页面（数据源）

- **协议**：HTTPS GET
- **域名白名单**：`*.lianjia.com`、`*.ke.com`
- **请求频率限制**：
  - 单用户：≤ 10 次/分钟
  - 单用户日总额：≤ 200 次
- **User-Agent**：使用真实浏览器 UA
- **超时**：单请求 10 秒
- **失败重试**：3 次，指数退避（1s / 2s / 4s）
- **解析失败处理**：返回明确错误码（404 页面不存在 / 反爬触发 / 字段缺失）
- **合规声明**：仅解析用户主动粘贴的 URL，不缓存目标站点数据，解析结果仅入用户私有库
- **限流器存储介质**：Vercel Serverless 无状态，限流计数器存于 Supabase `rate_limit_counters` 表（字段：user_id、window_start、count、updated_at），或接入 Upstash Redis 免费tier（10K 命令/天）
- **邀请链接形态**：基于一次性 token 的 URL（`/invite?token={uuid}`），token 与 owner_id、目标邮箱绑定存储于 Supabase `invitations` 表（可在 V1.3 抽离为独立表，V1.2 复用 household_members.pending 状态实现）。链接发送渠道：复制链接 + Supabase Auth 邮件 magic link（同时校验 token 与目标邮箱匹配，防止链接泄漏后被他人接受）
- **微信 OAuth 注销处理**：用户注销账户时同时调用 Supabase Auth Admin API 删除 auth.users 记录与 auth.identities 中的微信 unionid 绑定；再次微信扫码将创建新账户，与历史数据无关
- **跨时区处理**：所有 timestamptz 字段存储 UTC，前端展示统一转换为 Asia/Shanghai 时区（用户主为中国大陆用户）；date 字段无时区，按用户所在日历日解析
- **Storage 图片清理**：houses/viewings 物理删除时（含软删除超 30 天转物理删除）触发器同步删除 Storage 中对应路径 `users/{user_id}/{module}/{file}.webp`；用户注销时清理整个 `users/{user_id}/` 前缀
- **PWA 缓存与登出清理**：登出时调用 `caches.delete('choosyhome-runtime')` 清理运行时缓存中可能包含的用户数据；仅保留 App Shell 静态资源缓存（不含数据）
- **邀请链接邮箱匹配**：用户点击邀请链接注册/登录时，必须使用与邀请目标邮箱一致的账户，否则提示"邀请邮箱与登录邮箱不匹配"
- **经纪人电话合规说明**：`viewings.agent_phone` 字段属于第三方个人信息，按 PIPL 应取得经纪人明示同意。建议录入前口头告知经纪人"将记录联系方式用于家庭内部决策，不向第三方分发"。字段在数据库中以 pgcrypto 对称加密存储，仅在用户主账户登录时解密展示；不导出 CSV（NFR-SEC-08 已强制）
- **URL 解析失败处理**：部分字段解析失败时允许用户手动补全后保存；目标页面 404 或反爬触发时返回明确错误码，用户可改走 FR-HOUSE-02 手动录入路径

#### 6.1.2 Supabase（BaaS）

- **Auth API**：邮箱密码登录、OAuth 微信、JWT 续期
- **Database API**：通过 `@supabase/supabase-js` 客户端
- **Realtime API**：订阅 `houses`、`viewings`、`comparisons` 表变更
- **Storage API**：图片上传，路径规则 `users/{user_id}/{module}/{file}.webp`
- **RLS**：所有 API 调用通过 JWT 自动应用 RLS

#### 6.1.3 高德地图（可选）

- **API**：JS API Web 服务
- **用途**：看房记录地点展示（仅展示坐标，不做导航）
- **密钥**：用户自有 Key，存于前端环境变量

### 6.2 内部接口（Next.js API Routes）

| 路由 | 方法 | 鉴权 | 用途 |
| :--- | :--- | :--- | :--- |
| `/api/houses/parse-url` | POST | 必须 | 解析粘贴的房源 URL |
| `/api/houses` | GET/POST | 必须 | 房源 CRUD（也可直接走 Supabase Client） |
| `/api/houses/[id]` | GET/PUT/DELETE | 必须 | 单房源操作 |
| `/api/viewings` | GET/POST | 必须 | 看房记录 CRUD |
| `/api/comparisons` | GET/POST | 必须 | 对比清单 CRUD |
| `/api/export/all` | POST | 必须 | 全量导出 ZIP |
| `/api/export/comparison/[id]` | GET | 必须 | 单清单 CSV 导出 |
| `/api/auth/callback` | GET | — | OAuth 回调 |

> 说明：常规 CRUD 可直接在前端走 Supabase Client + RLS，无需经 API Route；URL 解析、导出等需要服务端逻辑的走 API Route。

### 6.3 接口错误码约定

| HTTP 状态码 | 含义 | 示例响应 |
| :--- | :--- | :--- |
| 200 | 成功 | `{ "data": ... }` |
| 400 | 参数错误 | `{ "error": "INVALID_PARAMS", "message": "..." }` |
| 401 | 未登录 | `{ "error": "UNAUTHORIZED" }` |
| 403 | 越权访问 | `{ "error": "FORBIDDEN" }` |
| 404 | 资源不存在 | `{ "error": "NOT_FOUND" }` |
| 429 | 频率限制 | `{ "error": "RATE_LIMITED", "retryAfter": 60 }` |
| 500 | 服务异常 | `{ "error": "INTERNAL_ERROR" }` |

---

## 7. 附录

### 7.1 字段对照表：用户原需求 ↔ 数据模型

| 用户原始需求 | 对应数据表 | 对应字段 |
| :--- | :--- | :--- |
| 自动获取特定区域的房源信息 | houses | city/district/community |
| 户型 | houses / viewings | layout |
| 大小 | houses / viewings | area |
| 成交价格 | houses | transaction_price |
| 成交时间 | houses | transaction_date |
| 均价 | houses | unit_price |
| 最低价 | houses | lowest_price |
| 城市记录 | viewings | city |
| 区域 | viewings | district |
| 小区 | viewings | community |
| 配套基础设施（学区） | viewings | has_school + facility_notes |
| 配套基础设施（医院） | viewings | has_hospital + facility_notes |
| 配套基础设施（商场） | viewings | has_mall + facility_notes |
| 配套基础设施（地铁） | viewings | has_subway + facility_notes |
| 出行便利 | viewings | commute_score |
| 南北朝向 | houses / viewings | orientation |
| 楼层 | houses / viewings | floor + total_floor |
| 楼龄 | houses / viewings | build_year |
| 楼型（塔楼/板楼等） | houses / viewings | building_type |
| 候选房综合对比 | comparisons + comparison_items | name/score/weight_config |
| 共享访客权限管理 | household_members + permissions | owner_id/member_id/module/can_view/can_edit/can_delete |

### 7.2 优先级矩阵

| 优先级 | 含义 | V1.2 必交付 |
| :--- | :--- | :--- |
| P0 | 必做（Must Have） | ✓ |
| P1 | 应做（Should Have） | 尽力交付 |
| P2 | 可做（Nice to Have） | V1.3+ 规划 |

### 7.3 CSV 导出字段对照

| 列名 | 来源 | 类型 |
| :--- | :--- | :--- |
| 城市 | houses | text |
| 区域 | houses | text |
| 小区 | houses | text |
| 户型 | houses | text |
| 面积 | houses | numeric |
| 总价 | houses | numeric |
| 单价 | houses | numeric |
| 朝向 | houses | text |
| 楼层 | houses | text |
| 楼龄 | houses | int |
| 楼型 | houses | text（塔楼/板楼/塔板结合/其他） |
| 意向等级 | houses | text |
| 标签 | houses | text（逗号分隔） |
| 看房次数 | viewings | int |
| 最近看房日期 | viewings | date |
| 综合评分 | viewings | int |
| 出行便利度 | viewings | int |
| 配套勾选 | viewings | text（如"学区,地铁"） |

### 7.4 名词索引

- App Shell：PWA 启动时先渲染的最小 UI 框架
- LCP / INP / CLS：Core Web Vitals 三项核心指标
- RLS：Postgres 行级安全策略
- PITR：Point-in-Time Recovery，时间点恢复备份
- PIPL：个人信息保护法

### 7.5 修订记录

| 版本 | 日期 | 修订人 | 内容 |
| :--- | :--- | :--- | :--- |
| V1.0 | 2026-09-01 | 项目负责人 | 初版基线发布 |
| V1.1 | 2026-09-01 | 项目负责人 | 1. 共享访客权限管理从未来版本提前至 V1.0 范围（新增 3.8 模块、`household_members` 与 `permissions` 表、RLS 辅助函数 `can_access_owner_data` 与 `has_permission`）；2. 房源手动录入与看房记录均补充"楼龄"与"楼型"字段 |
| V1.2 | 2026-09-01 | 项目负责人 | 开发前文档评审修复：修复 10 处逻辑矛盾（M1-M10）+ 16 处边界情况遗漏（B1-B16），新增 `operation_logs` 表、`get_writable_owner_uid()` 防 RLS 越权、邀请状态机、图片总配额 2000 张、限流器存储介质、邀请邮箱匹配、Storage 清理、跨时区、经纪人电话加密、微信 OAuth 注销处理等说明。详见目录后 V1.2 变更摘要。 |

---

**文档结束**

本说明书作为 V1.2 基线，后续如有重大调整需发起变更评审，记录于附录 7.5 修订记录。次要字段细化、UI 交互细节、API 字段精确定义在设计与开发阶段补充。

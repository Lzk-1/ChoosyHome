# ChoosyHome 开发方案

| 项目 | 内容 |
| :--- | :--- |
| **文档名称** | ChoosyHome 开发方案 |
| **文档版本** | V1.0 |
| **编制日期** | 2026-09-01 |
| **关联 SRS** | [SRS V1.2](./SRS.md) |
| **文档性质** | 施工蓝图 · 持续迭代 |
| **适用范围** | 个人项目 · AI 协作开发 |

---

## 1. 技术栈锁定

### 1.1 运行时与包管理

| 项目 | 版本 | 说明 |
| :--- | :--- | :--- |
| Node.js | ≥ 20.11 LTS | 使用 fnm/nvm 管理 |
| 包管理器 | pnpm 9.x | 比 npm 快、磁盘省 |
| TypeScript | 5.5+ | strict 模式 |

### 1.2 前端核心依赖

```jsonc
// package.json 关键依赖（锁定 major 版本）
{
  "dependencies": {
    "next": "^15.0.0",              // App Router
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "@supabase/supabase-js": "^2.45.0",
    "@supabase/ssr": "^0.5.0",       // 服务端 Auth helper
    "zustand": "^4.5.0",             // 客户端轻量状态
    "@tanstack/react-query": "^5.50.0", // 服务端数据缓存
    "react-hook-form": "^7.52.0",
    "zod": "^3.23.0",
    "@hookform/resolvers": "^3.9.0",
    "date-fns": "^3.6.0",
    "clsx": "^2.1.0",
    "tailwind-merge": "^2.5.0",
    "lucide-react": "^0.400.0",      // shadcn/ui 图标
    "next-pwa": "^5.6.0"            // PWA
  },
  "devDependencies": {
    "typescript": "^5.5.0",
    "tailwindcss": "^3.4.0",
    "autoprefixer": "^10.4.0",
    "postcss": "^8.4.0",
    "eslint": "^9.7.0",
    "eslint-config-next": "^15.0.0",
    "prettier": "^3.3.0",
    "prettier-plugin-tailwindcss": "^0.6.0",
    "husky": "^9.0.0",
    "lint-staged": "^15.2.0",
    "vitest": "^2.0.0",
    "@testing-library/react": "^16.0.0",
    "@playwright/test": "^1.45.0",
    "supabase": "^1.190.0"           // Supabase CLI
  }
}
```

### 1.3 后端/BaaS

| 组件 | 版本/层级 | 用途 |
| :--- | :--- | :--- |
| Supabase Cloud | Free Tier | Postgres 15 + Auth + Storage + Realtime |
| Postgres | 15 | 通过 Supabase 托管 |
| pgcrypto | 系统自带 | agent_phone 字段加密 |
| Supabase CLI | 1.190+ | 本地开发 + migration 管理 |

### 1.4 部署与监控

| 组件 | 用途 |
| :--- | :--- |
| Vercel | Next.js 托管（Hobby 免费） |
| Vercel Analytics | Web Vitals 监控 |
| Supabase Logs | 数据库与 Auth 日志 |
| Sentry（可选） | 错误监控免费 tier 5K events/月 |

---

## 2. 项目目录结构

```
d:\ChoosyHome\
├── docs/                           # 文档
│   ├── SRS.md                      # 软件需求规格说明书 V1.2
│   └── DEV.md                      # 本文档（开发方案）
├── supabase/                       # Supabase 本地项目
│   ├── config.toml
│   ├── migrations/                 # 数据库 migration（版本化）
│   │   ├── 0001_init_profiles.sql
│   │   ├── 0002_init_houses.sql
│   │   ├── 0003_init_viewings.sql
│   │   ├── 0004_init_comparisons.sql
│   │   ├── 0005_init_household.sql
│   │   ├── 0006_init_operation_logs.sql
│   │   ├── 0007_rls_functions.sql
│   │   ├── 0008_rls_policies.sql
│   │   ├── 0009_triggers.sql
│   │   └── 0010_storage_buckets.sql
│   └── seed.sql                    # 测试种子数据
├── public/
│   ├── manifest.json               # PWA 配置
│   ├── icons/                      # PWA 图标 192/512/maskable
│   └── offline.html                # 离线 fallback 页
├── src/
│   ├── app/                        # Next.js App Router
│   │   ├── (auth)/                 # 未登录路由组
│   │   │   ├── login/page.tsx
│   │   │   ├── register/page.tsx
│   │   │   ├── reset-password/page.tsx
│   │   │   └── invite/[token]/page.tsx
│   │   ├── (app)/                  # 已登录路由组
│   │   │   ├── layout.tsx          # 含导航栏/侧边栏
│   │   │   ├── page.tsx            # 首页
│   │   │   ├── houses/
│   │   │   │   ├── page.tsx        # 列表
│   │   │   │   ├── new/page.tsx    # 新增（URL解析+手动）
│   │   │   │   └── [id]/
│   │   │   │       ├── page.tsx    # 详情
│   │   │   │       └── edit/page.tsx
│   │   │   ├── viewings/
│   │   │   │   ├── page.tsx        # 时间线
│   │   │   │   ├── new/page.tsx
│   │   │   │   └── [id]/edit/page.tsx
│   │   │   ├── compare/
│   │   │   │   ├── page.tsx        # 对比清单列表
│   │   │   │   ├── new/page.tsx
│   │   │   │   └── [id]/page.tsx   # 对比详情
│   │   │   ├── settings/
│   │   │   │   ├── page.tsx        # 账户/主题
│   │   │   │   ├── members/page.tsx # 家庭成员管理
│   │   │   │   └── recycle/page.tsx # 回收站
│   │   ├── api/                    # API Routes
│   │   │   ├── auth/callback/route.ts
│   │   │   ├── houses/parse-url/route.ts
│   │   │   └── export/
│   │   │       ├── all/route.ts
│   │   │       └── comparison/[id]/route.ts
│   │   ├── layout.tsx              # 根 layout
│   │   ├── error.tsx               # 错误边界
│   │   ├── not-found.tsx
│   │   └── globals.css
│   ├── components/
│   │   ├── ui/                     # shadcn/ui 组件
│   │   ├── houses/                 # 房源相关组件
│   │   ├── viewings/               # 看房记录组件
│   │   ├── compare/                # 对比组件
│   │   ├── share/                  # 共享权限组件
│   │   └── common/                 # 通用组件
│   ├── lib/
│   │   ├── supabase/
│   │   │   ├── client.ts           # 浏览器端 client
│   │   │   ├── server.ts           # 服务端 client
│   │   │   ├── middleware.ts       # Auth middleware
│   │   │   └── types.ts            # Database 类型生成
│   │   ├── parsers/
│   │   │   ├── lianjia.ts          # 链家页面解析
│   │   │   └── beike.ts            # 贝壳页面解析
│   │   ├── utils/
│   │   │   ├── cn.ts                # className 合并
│   │   │   ├── date.ts              # 时区转换
│   │   │   ├── image.ts            # 图片压缩 webp
│   │   │   └── rate-limit.ts       # 限流器
│   │   └── validations/            # zod schema
│   │       ├── house.ts
│   │       ├── viewing.ts
│   │       └── comparison.ts
│   ├── hooks/                      # 自定义 React Hooks
│   ├── stores/                     # zustand stores
│   └── types/                      # 全局类型
├── tests/
│   ├── unit/                       # vitest 单元测试
│   └── e2e/                        # playwright e2e
├── .env.local                      # 本地环境变量（git ignored）
├── .env.production                  # 生产环境变量
├── .env.example                    # 环境变量模板
├── next.config.mjs                 # 含 PWA 配置
├── tailwind.config.ts
├── tsconfig.json
├── package.json
└── README.md
```

---

## 3. 环境变量清单

### 3.1 `.env.local`（本地开发，git ignored）

```bash
# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://xxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJxxx...
SUPABASE_SERVICE_ROLE_KEY=eyJxxx...    # 仅服务端使用，绝不进客户端 bundle

# 应用
NEXT_PUBLIC_APP_URL=http://localhost:3000
NODE_ENV=development

# URL 解析限流
RATE_LIMIT_WINDOW=60                   # 秒
RATE_LIMIT_MAX=10                      # 每窗口最大请求数
RATE_LIMIT_DAILY_MAX=200

# 可选
SENTRY_DSN=                            # 错误监控
AMAP_KEY=                              # 高德地图（V1.3+）
```

### 3.2 `.env.production`

```bash
NEXT_PUBLIC_SUPABASE_URL=https://xxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJxxx...
SUPABASE_SERVICE_ROLE_KEY=eyJxxx...
NEXT_PUBLIC_APP_URL=https://choosyhome.vercel.app
```

> **安全约束**（NFR-SEC-10）：`SUPABASE_SERVICE_ROLE_KEY` 只能在 Next.js Server Component / API Route / Middleware 中通过 `process.env` 访问，绝不写入 `NEXT_PUBLIC_*` 前缀变量。

---

## 4. MVP 范围定义

MVP 必须交付一个完整可用的核心闭环：**注册登录 → 录入房源 → 录入看房记录 → 查看对比**。

### 4.1 MVP 必交付（P0 + 关键 P1）

| 模块 | 需求编号 | 说明 |
| :--- | :--- | :--- |
| 认证 | FR-AUTH-01/02/04/05/06/07/08 | 邮箱密码全流程 + 中间件保护 |
| 房源管理 | FR-HOUSE-01/02/03/05/06/10/11/12/13/14/20/21/22/24/25 | URL 解析 + 手动 + 列表 + 详情 |
| 看房记录 | FR-VIEW-01/03/04/05/06/07/08/13/20/21/22/23 | 录入 + 时间线 + 编辑 |
| 对比清单 | FR-CMP-01/02/03/04/10/11 | 创建 + 加房源 + 横向对比 |
| PWA | FR-PWA-01/02/03/04/05/07 | manifest + SW + 离线页 + 实时同步 |
| 设置 | FR-SET-01 | 账户信息 |

### 4.2 MVP 明确不交付（后续 Sprint）

- 共享访客功能（FR-SHARE 全部）→ Sprint 5
- 微信 OAuth（FR-AUTH-03）→ Sprint 5
- 导出导入（FR-IO 全部）→ Sprint 5
- 数据库 operation_logs 表（FR-SET-05）→ Sprint 6
- 草稿房源/回收站（FR-VIEW-02/FR-HOUSE-23）→ Sprint 5
- 加权评分（FR-CMP-05/06）→ Sprint 5
- 图片上传（FR-HOUSE-04/FR-VIEW-09）→ Sprint 4
- 地图（FR-VIEW-24）→ V1.3+ 规划

### 4.3 MVP 验收标准

1. 用户能邮箱注册并登录
2. 用户能通过 URL 解析或手动录入创建房源
3. 用户能为房源录入看房记录并查看时间线
4. 用户能创建对比清单并横向对比 ≤4 处房源
5. 用户能安装为 PWA，离线可查看已缓存数据
6. 多端数据 2 秒内同步

---

## 5. 迭代计划（Sprint）

每个 Sprint 约 1-2 周，按依赖关系排序。

### Sprint 1：基础设施（必做）

**目标**：跑通本地开发环境 + 部署 Hello World

| 任务 | 产出 |
| :--- | :--- |
| 初始化 Next.js 15 项目 | `npx create-next-app@latest` + Tailwind |
| 配置 shadcn/ui | `pnpm dlx shadcn-ui@latest init` |
| 配置 ESLint/Prettier/husky | 提交前钩子 |
| 注册 Supabase 项目 | 拿到 URL/ANON/SERVICE_KEY |
| 配置 Supabase CLI 本地 | `supabase init` + `supabase start` |
| 部署到 Vercel | Hello World 上线 |
| 编写 README | 本地启动命令 |

**验收**：访问 `localhost:3000` 和 Vercel URL 均能看到首页。

### Sprint 2：认证 + 数据库 schema

**目标**：用户能注册登录，数据库表已建好

| 任务 | 产出 |
| :--- | :--- |
| 编写 migration 0001-0010 | 7 张表 + RLS + 触发器 |
| 实现 `lib/supabase/{client,server,middleware}.ts` | Auth helper |
| 实现登录/注册/重置密码页面 | FR-AUTH-01/02/06/07 |
| 实现中间件保护 | FR-AUTH-08 |
| 实现登出 | FR-AUTH-05 |
| 配置 JWT 续期 | FR-AUTH-04 |

**验收**：注册→登录→访问受保护路由→登出 全流程通。

### Sprint 3：房源管理核心闭环

**目标**：完成房源 CRUD + URL 解析

| 任务 | 产出 |
| :--- | :--- |
| 实现 `/houses/new` 手动录入表单 | FR-HOUSE-02 |
| 实现 URL 解析 API Route | FR-HOUSE-01 + 6.1.1 接口 |
| 编写 `lib/parsers/{lianjia,beike}.ts` | Cheerio 解析 |
| 实现 `/houses` 列表（筛选+排序+搜索） | FR-HOUSE-10/11/12/13 |
| 实现 `/houses/[id]` 详情页 | FR-HOUSE-20/24/25 |
| 实现编辑/软删除 | FR-HOUSE-21/22 |
| 实现"加入对比"徽标 | FR-HOUSE-14 |

**验收**：粘贴链家 URL→解析→保存→列表查看→详情→编辑→删除 全闭环。

### Sprint 4：看房记录 + 对比清单

**目标**：完成 SRS MVP 全部业务闭环

| 任务 | 产出 |
| :--- | :--- |
| 实现 `/viewings/new` 录入表单 | FR-VIEW-01/03-08/13 |
| 实现 `/viewings` 时间线视图 | FR-VIEW-20/21 |
| 实现编辑/软删除 | FR-VIEW-22/23 |
| 实现图片上传（含压缩） | FR-HOUSE-04 + FR-VIEW-09 |
| 实现 `/compare` 清单 CRUD | FR-CMP-01/10/11 |
| 实现对比表格页（横向对比） | FR-CMP-02/03/04 |

**验收**：录入看房→房源详情时间线→创建对比清单→横向对比 全闭环。MVP 完成。

### Sprint 5：共享权限 + 进阶功能

**目标**：扩展家庭成员协作 + 提升体验

| 任务 | 产出 |
| :--- | :--- |
| 实现 `/settings/members` 成员邀请 | FR-SHARE-01-05 |
| 实现权限矩阵 UI | FR-SHARE-10/11/12 |
| 实现权限回收/移除 | FR-SHARE-13/14 |
| 实现 Realtime 权限推送 | FR-SHARE-12 |
| 实现 OAuth 微信登录 | FR-AUTH-03 |
| 实现数据导出 ZIP/CSV | FR-IO-01/02 |
| 实现草稿房源 + 回收站 | FR-VIEW-02 + FR-HOUSE-23 |
| 实现加权评分对比 | FR-CMP-05/06/07 |
| 实现 PWA 安装提示/Splash | FR-PWA-04/06 |
| 实现断网续录 | FR-PWA-08 + FR-VIEW-12 |

### Sprint 6：打磨与上线

| 任务 | 产出 |
| :--- | :--- |
| 实现操作日志表 + UI | FR-SET-05 |
| 实现主题切换 | FR-SET-02 |
| 实现默认城市设置 | FR-SET-03 |
| 编写单元测试（解析器/评分/限流） | NFR-MAINT-05 |
| 编写关键路径 e2e 测试 | 登录→录入→对比 |
| 性能优化（Bundle 分析） | NFR-PERF-04 |
| 隐私政策页面 | NFR-COMP-07 |
| 正式上线 + 域名 | V1.0 发布 |

---

## 6. 关键技术实现要点

### 6.1 Supabase Migration 顺序

按外键依赖排序，分 10 个 migration 文件：

```
0001 profiles              → 依赖 auth.users
0002 houses                → 依赖 profiles
0003 viewings              → 依赖 profiles + houses
0004 comparisons/items     → 依赖 profiles + houses
0005 household_members/permissions → 依赖 auth.users
0006 operation_logs        → 依赖 auth.users
0007 RLS 辅助函数          → can_access_owner_data/has_permission/get_writable_owner_uid
0008 RLS 策略              → 所有表的 SELECT/INSERT/UPDATE/DELETE
0009 触发器                → 4房源上限/5访客上限/权限初始化/操作日志
0010 Storage buckets       → houses-images/viewings-images（私有桶）
```

执行命令：
```bash
supabase db push            # 推送所有 migration 到云端
supabase db reset           # 本地重置（开发时）
```

### 6.2 RLS 防越权实现要点

参考 SRS 5.3，关键点：

```typescript
// lib/supabase/client.ts
import { createBrowserClient } from '@supabase/ssr'

export const createClient = () =>
  createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
```

访客通过 Supabase Client 写入 houses 时，前端**不可传 user_id**，由数据库触发器自动填充：

```sql
-- 触发器：BEFORE INSERT/UPDATE on houses
-- 若 user_id 与当前用户 auth.uid() 不匹配且非授权访客，RAISE EXCEPTION
-- 共享访客写入时，user_id 必须等于其归属主用户的 owner_id
```

### 6.3 URL 解析器实现

技术：**Cheerio**（轻量，无浏览器引擎），避免 Playwright 体积过大：

```typescript
// lib/parsers/lianjia.ts
import { load } from 'cheerio'
import { withRetry } from '@/lib/utils/retry'

export async function parseLianjia(url: string, userId: string) {
  await checkRateLimit(userId)  // 限流
  const html = await fetchHtml(url)  // 含 UA、超时、重试
  const $ = load(html)
  return {
    community: $('.community .info').text().trim(),
    layout: $('.house-info .mainInfo').text(),
    area: parseArea($('.house-info .subInfo').text()),
    totalPrice: parsePrice($('.price .total').text()),
    unitPrice: parseUnitPrice($('.price .unitPriceValue').text()),
    orientation: $('.base .content ul li:nth-child(7)').text(),
    floor: $('.base .content ul li:nth-child(2)').text(),
    buildYear: parseBuildYear($('.base .content ul li:nth-child(8)').text()),
    // ...
  }
}
```

**合规要点**：
- 仅解析用户主动传入的 URL
- 不缓存目标站点 HTML
- 限流：10 次/分钟 + 200 次/日（`lib/utils/rate-limit.ts`）
- 失败时降级到 FR-HOUSE-02 手动录入

### 6.4 PWA 配置

```javascript
// next.config.mjs
import withPWAInit from 'next-pwa'

const nextConfig = {
  // ...
}

export default withPWAInit({
  dest: 'public',
  register: false,            // 手动注册 SW 以控制登出清理
  disable: process.env.NODE_ENV === 'development',
  runtimeCaching: [
    {
      urlPattern: /^https:\/\/.*\.supabase\.co\/storage\/.*/,
      handler: 'CacheFirst',
      options: {
        cacheName: 'choosyhome-images',
        expiration: { maxEntries: 200, maxAgeSeconds: 7 * 24 * 3600 },
      },
    },
  ],
})(nextConfig)
```

```typescript
// app/layout.tsx 中手动注册 SW
'use client'
import { useEffect } from 'react'

export function ServiceWorkerRegister() {
  useEffect(() => {
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.register('/sw.js')
    }
  }, [])
  return null
}
```

```typescript
// 登出时清理缓存
async function signOut() {
  await supabase.auth.signOut()
  if ('caches' in window) {
    caches.delete('choosyhome-runtime')
    caches.delete('choosyhome-images')  // 清理可能含用户数据的缓存
  }
  router.push('/login')
}
```

### 6.5 图片压缩与上传

```typescript
// lib/utils/image.ts
import { compress, convertToWebp } from 'browser-image-compression'

export async function compressImage(file: File): Promise<File> {
  const compressed = await compress(file, {
    maxSizeMB: 0.5,
    maxWidth: 1920,
    quality: 0.8,
  })
  return convertToWebp(compressed)
}

// 上传到 Supabase Storage
export async function uploadImage(
  file: File,
  userId: string,
  module: 'houses' | 'viewings'
): Promise<string> {
  const compressed = await compressImage(file)
  const path = `${userId}/${module}/${crypto.randomUUID()}.webp`
  const { error } = await supabase.storage
    .from(`${module}-images`)
    .upload(path, compressed, { contentType: 'image/webp' })
  if (error) throw error
  return path
}
```

### 6.6 离线续录（IndexedDB）

```typescript
// lib/offline/queue.ts
import { openDB } from 'idb'

const db = await openDB('choosyhome-offline', 1, {
  upgrade(d) {
    d.createObjectStore('pending-records', { keyPath: 'id' })
  },
})

export async function savePending(record: any) {
  await db.put('pending-records', { ...record, id: crypto.randomUUID() })
}

export async function syncPending(supabase: SupabaseClient) {
  const all = await db.getAll('pending-records')
  for (const record of all) {
    try {
      await supabase.from(record.table).insert(record.data)
      await db.delete('pending-records', record.id)
    } catch (e) {
      // 重试机制
    }
  }
}
```

监听 `online` 事件触发同步。

### 6.7 数据库类型生成

```bash
# 生成 TypeScript 类型
pnpm supabase gen types typescript --project-id xxxx > src/lib/supabase/types.ts
```

```typescript
// 使用
import { Database } from '@/lib/supabase/types'
type House = Database['public']['Tables']['houses']['Row']
```

### 6.8 agent_phone 加密

```sql
-- migration 0003 中
ALTER TABLE viewings ADD COLUMN agent_phone_encrypted bytea;

-- 触发器：写入时用 pgcrypto 加密
CREATE OR REPLACE FUNCTION encrypt_agent_phone()
RETURNS trigger AS $$
BEGIN
  IF NEW.agent_phone IS NOT NULL THEN
    NEW.agent_phone_encrypted := pgp_sym_encrypt(NEW.agent_phone, current_setting('app.encryption_key'));
    NEW.agent_phone := NULL;  -- 清空明文
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

读取时由服务端 API Route 解密，前端不直接读 agent_phone。

---

## 7. 开发环境搭建

```bash
# 1. 克隆仓库
cd d:\ChoosyHome

# 2. 安装依赖
pnpm install

# 3. 复制环境变量
cp .env.example .env.local
# 编辑 .env.local 填入 Supabase 凭据

# 4. 启动 Supabase 本地（首次会拉镜像）
supabase start

# 5. 推送 migration 到本地 Postgres
supabase db reset

# 6. 启动 Next.js
pnpm dev
# 访问 http://localhost:3000

# 7. 可选：seed 测试数据
supabase db reset --use-seed
```

---

## 8. 测试策略

### 8.1 单元测试（vitest）

覆盖范围（NFR-MAINT-05 ≥ 70%）：
- `lib/parsers/lianjia.ts` / `beike.ts`：解析逻辑
- `lib/utils/rate-limit.ts`：限流计数
- `lib/utils/image.ts`：图片压缩
- `lib/validations/*.ts`：zod schema
- 评分计算函数（FR-CMP-05/06 加权逻辑）

```bash
pnpm test              # 单次运行
pnpm test:watch        # 监听
pnpm test:coverage     # 覆盖率
```

### 8.2 E2E 测试（Playwright）

关键路径覆盖：
- 登录→录入房源→查看列表
- 录入看房记录→查看时间线
- 创建对比清单→横向对比
- 共享访客权限变更生效（Sprint 5）

```bash
pnpm e2e
```

### 8.3 手动验收

按 SRS V1.2 第 3 章功能需求逐条验收，记录在 `docs/UAT-checklist.md`（开发阶段创建）。

---

## 9. 部署流程

### 9.1 首次部署

```bash
# 1. 推送 migration 到生产 Supabase
supabase db push --linked

# 2. 在 Vercel 导入 Git 仓库
# 3. 配置环境变量（从 .env.production）
# 4. 触发首次部署
# 5. 在 Vercel 配置自定义域名
```

### 9.2 后续迭代

```bash
# 1. 本地开发
git checkout -b feature/xxx
# ... 编码 + 测试
pnpm test && pnpm e2e

# 2. 合并到 main
git push origin main

# 3. Vercel 自动部署

# 4. 数据库 migration
supabase db push --linked
```

### 9.3 回滚

- **代码**：Vercel 上一键回滚到前一个部署
- **数据库**：Supabase PITR（7 天内时间点恢复，免费 tier 支持）

---

## 10. 风险与缓解

| 风险 | 概率 | 影响 | 缓解措施 |
| :--- | :--- | :--- | :--- |
| 链家/贝壳页面结构变更 | 中 | URL 解析失败 | 解析失败降级到手动录入；监听关键选择器并报警 |
| Supabase Free Tier 容量超限 | 低 | 数据无法写入 | 图片配额 2000 张；监控数据库大小；超限可升级 Pro $25/月 |
| PWA 在 iOS Safari 行为差异 | 中 | 体验降级 | iOS 不支持 beforeinstallprompt，引导用户"分享→添加到主屏" |
| 共享访客 RLS 越权 | 高 | 数据泄露 | 已用 `get_writable_owner_uid()` + WITH CHECK + 触发器三重防护 |
| 微信 OAuth 配置复杂 | 中 | 登录受阻 | Sprint 5 才接入，先用邮箱密码打通 MVP |
| 限流器在 Serverless 失效 | 中 | 反爬触发 | 使用 Supabase 表存储计数器或 Upstash Redis |

---

## 11. 文档维护

- 本开发方案随 Sprint 推进持续迭代
- 每完成一个 Sprint 在本文档末尾追加 Sprint 回顾
- SRS（V1.2）作为基线契约，重大需求变更才修订
- API 字段精确定义、组件 props 详细设计在开发时按需补充到本文档

---

**文档结束**

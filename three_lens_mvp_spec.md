# 三镜头投资报告 · MVP 最终规格

**版本：** v2.0 Final  
**日期：** 2026-06-24  
**执行对象：** Claude Code  
**域名：** multiple-lens.com  
**仓库：** No5fisher/investment-research  
**定价货币：** CAD

---

## 一、产品定位

面向海外华人投资者的专业投资研究报告服务。用户付费后提交公司名称，系统自动生成巴菲特 / 蒂尔 / 施梅尔三镜头分析报告，以 PDF 文件发送至注册邮箱。

---

## 二、功能范围（极简 MVP）

### 包含功能
- 用户注册 / 登录（Email）
- 查看框架使用说明页
- 选择套餐并通过 Stripe Payment Link 完成支付
- 提交请求：公司名称（必填）+ 分析侧重（可选）
- 系统自动生成报告并发送 PDF 至注册邮箱
- 后台日志：查询记录、交付状态、扣款记录（不对用户展示）

### 不包含功能（P1 后续迭代）
- 报告在线展示
- 用户报告历史页面
- 状态实时轮询进度条
- 微信支付
- 管理后台界面
- 月度订阅自动续费

---

## 三、技术架构

```
用户浏览器
    │
    ▼
Next.js 落地页（Vercel 托管 · multiple-lens.com）
    │
    ├── 支付：Stripe Payment Link（跳转 Stripe 托管结账页）
    │         付款成功 → Stripe Webhook → 触发报告生成
    │
    ▼
Vercel Serverless Functions
    │
    ├── Redis（Upstash）— 缓存层，按财报季度缓存 PDF URL
    │
    ├── 生成层（Anthropic API claude-sonnet-4-6）
    │         注入 Finnhub 实时数据 + 三镜头 Prompt
    │         temperature=0.3 保证质量一致性
    │
    ├── PDF 转换（html-pdf-node）
    │
    ├── 存储层（Cloudflare R2）
    │         上传 PDF，返回 72 小时有效签名链接
    │
    └── 邮件（Resend）
              发送 PDF 链接至用户注册邮箱
```

### 技术栈
| 层级 | 选择 |
|---|---|
| 前端 / 托管 | Next.js + Vercel |
| 数据库（用户 / 日志） | Vercel Postgres 或 Supabase（免费层） |
| 缓存 | Upstash Redis |
| AI 生成 | Anthropic API (claude-sonnet-4-6) |
| PDF 转换 | html-pdf-node |
| 文件存储 | Cloudflare R2 |
| 支付 | Stripe Payment Link |
| 邮件 | Resend |
| 市场数据 | Finnhub |

---

## 四、文件结构

```
three-lens-reports/
├── .env.local                    # 所有密钥（不入 git）
├── .env.example                  # 密钥模板（入 git）
├── .gitignore                    # 包含 .env.local
├── package.json
├── pages/
│   ├── index.js                  # 落地页：说明 + 套餐 + 登录入口
│   ├── login.js                  # 注册 / 登录页
│   ├── dashboard.js              # 登录后：提交请求 + 消费记录
│   ├── guide.js                  # 框架使用说明页（静态）
│   └── api/
│       ├── auth/
│       │   ├── register.js       # 注册
│       │   └── login.js          # 登录
│       ├── webhook-stripe.js     # Stripe 支付回调 → 触发生成
│       ├── generate-report.js    # 核心：生成 / 缓存 / 存储 / 发邮件
│       └── status.js             # 查询生成状态（轮询用）
├── lib/
│   ├── anthropic.js              # Claude API + Prompt 模板
│   ├── cache.js                  # Redis 缓存逻辑
│   ├── pdf.js                    # HTML → PDF
│   ├── storage.js                # R2 上传 + 签名链接
│   ├── email.js                  # Resend 发送
│   ├── market-data.js            # Finnhub 数据获取
│   └── db.js                     # 数据库操作（用户 / 日志）
└── prompts/
    └── three-lens-framework.js   # 三镜头 Prompt 模板
```

---

## 五、支付流程（Payment Link 模式）

```
1. 用户在 dashboard.js 选择套餐
   - 单份报告 → 跳转 STRIPE_PLINK_SINGLE
   - 10份套餐 → 跳转 STRIPE_PLINK_10PACK

2. Stripe 托管结账页完成支付

3. Stripe 向 /api/webhook-stripe 发送 checkout.session.completed 事件

4. webhook-stripe.js 处理：
   a. 验证 Stripe 签名（STRIPE_WEBHOOK_SECRET）
   b. 从 metadata 取出 user_id + company + ticker
   c. 扣减用户配额（写数据库）
   d. 调用 generate-report.js 异步生成报告
   e. 返回 200 给 Stripe

5. 生成完成后发送邮件至用户注册邮箱
```

**Payment Link 配置要求（在 Stripe Dashboard 设置）：**
- 在每个 Payment Link 的 "After payment" 设置中填写：
  `https://multiple-lens.com/dashboard?payment=success`
- 在 Metadata 中传入 `user_id` 和 `company`（通过 URL 参数 `?prefilled_email=` 和自定义 metadata）

---

## 六、缓存策略

```javascript
// 缓存键：公司代码 + 财报季度
const cache_key = `report:${ticker}:${getFiscalQuarter()}`
// 例：report:GOOGL:2026Q2

// TTL：90天（覆盖一个完整财报周期）
// 同一公司同一季度内，所有用户共享同一份 PDF
// 季度切换后第一个请求触发重新生成
```

---

## 七、Prompt 规格

文件：`prompts/three-lens-framework.js`

**关键参数：**
- Model: `claude-sonnet-4-6`
- Temperature: `0.3`（降低随机性，保证质量一致性）
- Max tokens: `8000`

**数据注入（来自 Finnhub）：**
- 公司基础信息（名称、交易所、行业）
- 当前股价、市值
- 最新财年收入及增速、净利润、自由现金流
- P/E 比率
- 最近5条重要新闻标题

**输出格式：**
- 完整 HTML 报告
- 内嵌 Adobe 设计系统 CSS（Libre Baskerville + DM Mono + Inter）
- 三栏布局：Buffett（绿 #1d4e2f）/ Thiel（紫 #2b1f4e）/ Schmehl（红 #7a2020）
- 每位投资者6个分析条目 + 评分（0-100）
- 报告结构：masthead → 数据条 → 情境三栏 → 分隔线 → 三栏分析 → 评分带 → 核心问题 → 综合叙事 → 页脚

---

## 八、页面规格

### index.js（落地页，未登录可见）
- 产品标题 + 一句话描述
- 三位投资者简介
- 套餐卡片：单份 CA$18 / 10份套餐 CA$148（原价CA$180，打折）
- 体验报告示例链接 → no5fisher.github.io/investment-research
- 登录 / 注册按钮
- 语言：中文为主

### login.js（注册 / 登录）
- Email + 密码
- 发送验证邮件（Resend）
- 登录成功跳转 dashboard.js

### dashboard.js（登录后核心页面）
- 显示账户余额（剩余报告份数）
- 提交请求表单：
  - 公司名称或股票代码（必填）
  - 分析侧重（可选：默认三镜头 / 偏巴菲特 / 偏蒂尔 / 偏施梅尔）
  - 提交邮箱确认（默认用注册邮箱）
- 提交后显示："已收到请求，报告将在 5-10 分钟内发送至 [email]"
- 历史记录列表：日期 / 公司名 / 状态（生成中 / 已发送）

### guide.js（使用说明，静态页面）
- 直接嵌入 framework_guide.html 的内容
- 或链接跳转到 GitHub Pages 版本

---

## 九、数据库表设计（最小化）

```sql
-- 用户表
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  credits INTEGER DEFAULT 0,      -- 剩余报告份数
  created_at TIMESTAMP DEFAULT NOW()
);

-- 查询日志
CREATE TABLE report_requests (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  company TEXT NOT NULL,
  ticker TEXT,
  status TEXT DEFAULT 'pending',  -- pending / generating / sent / failed
  cache_hit BOOLEAN DEFAULT FALSE,
  pdf_url TEXT,
  email_sent_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

-- 支付记录
CREATE TABLE payments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  stripe_session_id TEXT UNIQUE,
  amount_cad DECIMAL(10,2),
  credits_added INTEGER,
  created_at TIMESTAMP DEFAULT NOW()
);
```

---

## 十、环境变量（完整）

参见项目根目录 `.env.local`（不入 git）。

主要变量：
```
ANTHROPIC_API_KEY         # Claude API
FINNHUB_API_KEY           # 市场数据
UPSTASH_REDIS_REST_URL    # Redis
UPSTASH_REDIS_REST_TOKEN  # Redis
RESEND_API_KEY            # 邮件
FROM_EMAIL                # reports@multiple-lens.com
R2_ENDPOINT               # Cloudflare R2
R2_BUCKET_NAME            # multiple-lens-report
R2_ACCESS_KEY_ID          # R2 访问密钥
R2_SECRET_ACCESS_KEY      # R2 密钥
STRIPE_PUBLISHABLE_KEY    # Stripe 前端
STRIPE_SECRET_KEY         # Stripe 后端
STRIPE_PLINK_SINGLE       # 单份支付链接
STRIPE_PLINK_10PACK       # 10份套餐支付链接
STRIPE_WEBHOOK_SECRET     # 部署后补充
NEXT_PUBLIC_APP_URL       # https://multiple-lens.com
REPORT_LINK_EXPIRY_HOURS  # 72
CACHE_TTL_DAYS            # 90
```

---

## 十一、Claude Code 执行顺序

```bash
# Step 1：初始化项目
npx create-next-app@latest three-lens-reports --js --tailwind --no-app
cd three-lens-reports
npm install @anthropic-ai/sdk @upstash/redis @aws-sdk/client-s3 \
  html-pdf-node resend stripe finnhub-js bcryptjs jsonwebtoken \
  @vercel/postgres

# Step 2：复制 .env.local 到项目根目录

# Step 3：实现顺序（由内向外）
# 3a. lib/market-data.js     — Finnhub 数据获取
# 3b. prompts/three-lens-framework.js — Prompt 模板
# 3c. lib/anthropic.js       — Claude 调用
# 3d. lib/cache.js           — Redis 缓存
# 3e. lib/pdf.js             — HTML → PDF
# 3f. lib/storage.js         — R2 上传
# 3g. lib/email.js           — Resend 发送
# 3h. lib/db.js              — 数据库操作
# 3i. pages/api/generate-report.js — 串联所有 lib
# 3j. pages/api/webhook-stripe.js  — 支付回调
# 3k. pages/api/auth/         — 注册登录
# 3l. pages/index.js          — 落地页
# 3m. pages/login.js          — 登录页
# 3n. pages/dashboard.js      — 用户主页

# Step 4：本地测试顺序
# 4a. 用 mock 数据测试 generate-report.js（跳过支付）
# 4b. 确认 PDF 输出质量
# 4c. 测试 Redis 缓存命中逻辑
# 4d. 测试 Stripe Webhook（用 stripe listen 本地转发）
# 4e. 端到端测试完整流程

# Step 5：部署
vercel --prod
# 在 Vercel Dashboard 设置所有环境变量
# 在 Stripe Dashboard 配置 Webhook → 获取 STRIPE_WEBHOOK_SECRET
# 在 Resend 验证 multiple-lens.com 域名
# 在 Cloudflare 将 DNS 指向 Vercel
```

---

## 十二、上线检查清单

- [ ] 本地端到端测试通过
- [ ] Vercel 部署成功，域名绑定
- [ ] Stripe Webhook 配置，STRIPE_WEBHOOK_SECRET 已填入 Vercel
- [ ] Resend 域名验证完成，测试邮件正常收到
- [ ] R2 Bucket 权限确认（可写入，签名链接有效）
- [ ] Redis 缓存逻辑测试（同一公司第二次请求命中缓存）
- [ ] GitHub Pages 体验版链接放入落地页
- [ ] 生产环境 Stripe 收款测试（小额真实支付）

---

*本文档为最终执行版，供 Claude Code 直接使用。凭证详情见 .env.local。*

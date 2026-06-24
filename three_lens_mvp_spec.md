# 三镜头投资报告 · MVP 产品需求与技术规格

**文档版本：** v1.0  
**日期：** 2026-06-23  
**执行对象：** Claude Code  
**产品名称：** 三镜头判决 · 投资研究报告服务  
**目标用户：** 海外华人投资者（主要），国内投资者（兼顾）

---

## 一、产品概述

### 核心体验
用户通过一个独立页面，完成付费 → 输入请求 → 接收报告的完整闭环。  
系统接到请求后，优先返回缓存报告（按财报周期），否则实时生成，最终以 **PDF 下载链接**形式交付给用户。

### 体验路径（一句话描述）
> 用户扫码/点链接进入页面 → 选择付费套餐 → 输入公司名称 → 系统处理（实时或缓存）→ 收到 PDF 报告链接。

---

## 二、系统架构（路径 A 极简 MVP）

```
用户浏览器
    │
    ▼
静态落地页（Vercel 托管）
    │  输入：公司名、联系方式（邮件/微信）
    │  付费：Stripe / 微信支付（Payjs）验证
    ▼
Vercel Serverless Function（API Route）
    │
    ├── 查询缓存层（Upstash Redis）
    │       命中：直接返回已有 PDF 的 CDN 链接
    │       未命中：触发生成流程
    │
    ├── 生成层（Anthropic API）
    │       调用三镜头框架 Prompt
    │       生成 HTML 报告内容
    │
    ├── PDF 转换（Puppeteer / html-pdf-node）
    │       HTML → PDF
    │
    ├── 存储层（Cloudflare R2）
    │       存储 PDF 文件
    │       返回带时效的签名下载链接（72小时有效）
    │
    └── 通知层（Resend 邮件）
            发送包含 PDF 链接的邮件给用户
```

### 技术栈
| 层级 | 选择 | 说明 |
|---|---|---|
| 前端 / 托管 | Vercel | 零配置部署，免费额度够 MVP |
| Serverless 函数 | Vercel Functions (Node.js) | 与前端同仓库 |
| 缓存 | Upstash Redis | Serverless 友好，按请求计费 |
| AI 生成 | Anthropic API (claude-sonnet-4-6) | 与当前报告生成能力一致 |
| PDF 转换 | html-pdf-node（轻量）或 Puppeteer | 将 HTML 报告转为 PDF |
| 文件存储 | Cloudflare R2 | 兼容 S3 API，无出流量费 |
| 国际支付 | Stripe | 支持 CAD 定价，信用卡 |
| 微信支付 | Payjs | 无需企业资质，扫码支付 |
| 邮件 | Resend | 简洁 API，免费额度 3000封/月 |
| 网页数据 | 搜索 API 或 Finnhub | 注入实时财务数据到 Prompt |

---

## 三、文件结构

```
project-root/
├── package.json
├── .env.local                    # 所有密钥（不入 git）
├── .env.example                  # 密钥示例（入 git）
├── public/
│   └── (静态资源)
├── pages/
│   ├── index.js                  # 落地页（付费 + 输入表单）
│   ├── success.js                # 支付成功 + 状态轮询页
│   └── api/
│       ├── create-payment.js     # 创建 Stripe / Payjs 支付
│       ├── webhook-stripe.js     # Stripe Webhook 回调
│       ├── webhook-payjs.js      # Payjs 支付回调
│       └── generate-report.js   # 核心：生成 / 缓存 / 返回报告
├── lib/
│   ├── anthropic.js              # Claude API 调用 + Prompt 模板
│   ├── cache.js                  # Upstash Redis 缓存逻辑
│   ├── pdf.js                    # HTML → PDF 转换
│   ├── storage.js                # Cloudflare R2 上传 / 签名链接
│   ├── email.js                  # Resend 邮件发送
│   └── validate-payment.js       # 支付验证工具函数
└── prompts/
    └── three-lens-framework.js   # 完整的三镜头 Prompt 模板（见第五节）
```

---

## 四、核心业务逻辑

### 4.1 缓存策略（关键设计）

**缓存键规则：**
```
cache_key = `report:${ticker}:${fiscal_quarter}`
例如：report:AAPL:2026Q2
```

**财报季度判断逻辑：**
```javascript
function getFiscalQuarter(date = new Date()) {
  const year = date.getFullYear();
  const month = date.getMonth() + 1;
  const quarter = Math.ceil(month / 3);
  return `${year}Q${quarter}`;
}
```

**缓存流程：**
```
接收请求（ticker）
    │
    ▼
生成 cache_key = `report:${ticker}:${getFiscalQuarter()}`
    │
    ├── Redis GET(cache_key)
    │       命中 → 返回 { pdf_url, generated_at, expires_at }
    │       未命中 → 进入生成流程
    │
    ▼（未命中）
调用 Claude API 生成报告 HTML
    │
    ▼
HTML → PDF 转换
    │
    ▼
上传 PDF 到 Cloudflare R2
    │
    ▼
Redis SET(cache_key, { pdf_url, ... }, EX: 90天)
    │
    ▼
返回 pdf_url 给用户
```

**缓存过期机制：**
- TTL 设为 **90天**（覆盖一个完整财报周期）
- 用户请求时若距上次生成 > 30天且有重大新闻，可触发强制刷新（后期功能）

### 4.2 支付验证流程

**Stripe（国际用户）：**
```
前端 → create-payment API（创建 PaymentIntent）
→ 前端用 Stripe.js 完成支付
→ Stripe Webhook 回调 webhook-stripe.js
→ 验证签名 → 标记支付成功 → 触发报告生成
→ 邮件发送 PDF 链接
```

**Payjs（微信支付）：**
```
前端 → create-payment API（获取 Payjs 二维码）
→ 用户微信扫码支付
→ Payjs 回调 webhook-payjs.js
→ 验证签名 → 标记支付成功 → 触发报告生成
→ 邮件发送 PDF 链接
```

### 4.3 报告生成 API（核心接口）

**接口：** `POST /api/generate-report`

**请求体：**
```json
{
  "company": "Alphabet",
  "ticker": "GOOGL",
  "payment_id": "pi_xxx",
  "email": "user@example.com",
  "lang": "zh"
}
```

**处理流程：**
1. 验证 payment_id 已成功支付（查 Redis 支付状态表）
2. 检查缓存：`report:GOOGL:2026Q2`
3. **命中缓存**：直接取 PDF URL，发送邮件，返回链接
4. **未命中缓存**：
   a. 调用 Finnhub / 搜索 API 获取公司最新数据
   b. 将数据注入 Prompt（见第五节）
   c. 调用 Claude API 生成报告 HTML
   d. HTML → PDF（html-pdf-node）
   e. 上传 PDF 到 R2，获取签名链接
   f. 写入 Redis 缓存
   g. 发送邮件
   h. 返回 PDF 链接
5. **响应超时处理**：生成可能需要 60-120秒，采用异步模式：
   - 立即返回 `{ status: "processing", job_id: "xxx" }`
   - 前端轮询 `GET /api/status?job_id=xxx`
   - 完成后前端展示下载链接

---

## 五、三镜头 Prompt 模板规格

### 5.1 Prompt 结构

文件路径：`prompts/three-lens-framework.js`

```javascript
// prompts/three-lens-framework.js

export function buildThreeLensPrompt(companyData) {
  const { 
    companyName,      // 公司名称（中英文）
    ticker,           // 股票代码
    exchange,         // 交易所
    currentPrice,     // 当前股价
    marketCap,        // 市值
    revenue,          // 最新年度收入
    revenueGrowth,    // 收入增长率
    netIncome,        // 净利润
    freeCashFlow,     // 自由现金流
    pe,               // 市盈率
    recentNews,       // 最近重要新闻摘要（3-5条）
    sector,           // 行业
    reportDate,       // 报告日期
    fiscalQuarter,    // 财报季度
  } = companyData;

  return `你是一个投资研究分析师，需要以三位不同的投资大师视角对同一家公司进行深度分析。

## 公司基础数据
- 公司：${companyName}（${ticker} · ${exchange}）
- 当前股价：${currentPrice}
- 市值：${marketCap}
- 最新财年收入：${revenue}（YoY ${revenueGrowth}）
- 净利润：${netIncome}
- 自由现金流：${freeCashFlow}
- P/E：${pe}
- 行业：${sector}
- 最新重要事件：
${recentNews.map((n, i) => `  ${i+1}. ${n}`).join('\n')}

## 分析框架说明

请严格按照以下三个投资者的独立哲学框架，对上述公司进行分析。**每个视角必须完全独立，不得相互影响。**

### 视角一：Warren Buffett（巴菲特）
核心问题：十年后这家公司的护城河是否还在？
分析维度：
- 护城河的性质与持久性（品牌、转换成本、网络效应、规模经济、成本优势）
- 定价权验证
- 自由现金流的可预期性
- 管理层资本配置纪律
- 当前估值的安全边际
- 判决：[明确买入/观望/回避] + 一句话理由

### 视角二：Peter Thiel（蒂尔）
核心问题：这家公司建立在什么别人还不相信的秘密之上？
分析维度：
- 原始秘密是否仍然有效（是否已成为常识）
- 垄断特征（专有技术10×、网络效应、规模经济、品牌）
- 从小市场到大市场的扩张路径
- 创始人特质
- 是否有新的零到一潜力
- 判决：[积极/中性/消极] + 一句话理由

### 视角三：Mark Schmehl（施梅尔）
核心问题：市场对这家公司正在发生的变化，定价正确吗？
分析维度：
- 正在发生的核心正面变化及市场定价程度
- 是否存在负面事件过度反应的机会
- 明确催化剂的识别
- 股价动量与基本面动量的对齐程度
- 卖出触发条件
- 判决：[强烈关注/观望/已充分定价] + 一句话理由

## 输出格式要求

请输出完整的 HTML 报告，严格遵循以下格式规范：

1. 使用指定的 CSS 设计系统（见下方 CSS 变量）
2. 报告结构：公司概述 → 数据条 → 当前处境（三个情境块）→ 分节分隔线 → 三栏人物分析 → 评分汇总 → 核心判断题 → 综合叙事 → 页脚
3. 每位投资者的分析包含：人物标题 → 核心问题 → 判决框 → 6个标准分析条目（每条含评级徽章）
4. 评分：每位投资者给出 0-100 分，三人共识取平均分
5. 语言：中文为主，专业术语保留英文
6. 务必提供深度、独立、有实质内容的分析，每个分析条目不少于100字

## CSS 设计系统（必须完整嵌入报告）

\`\`\`css
@import url('https://fonts.googleapis.com/css2?family=Libre+Baskerville:ital,wght@0,400;0,700;1,400&family=DM+Mono:wght@400;500&family=Inter:wght@300;400;500;600&display=swap');

:root {
  --ink: #1a1a1a; --ink-2: #3d3d3d; --ink-3: #7a7a7a; --ink-4: #b8b8b8;
  --paper: #f7f4ef; --paper-2: #eeebe4; --rule: #d8d4cc;
  --buffett: #1d4e2f; --buffett-l: #e8f2ec;
  --thiel: #2b1f4e; --thiel-l: #ece9f5;
  --schmehl: #7a2020; --schmehl-l: #f5eaea;
  --gold: #b8972a; --gold-l: #fdf6e3;
  --serif: 'Libre Baskerville', Georgia, serif;
  --mono: 'DM Mono', monospace;
  --sans: 'Inter', system-ui, sans-serif;
}
\`\`\`

报告页脚需包含：公司名称 · 股票代码 · 三镜头投资判决框架 · 生成日期 ${reportDate} · ${fiscalQuarter}版本

请立即开始生成完整的 HTML 报告。`;
}
```

### 5.2 数据注入层

文件路径：`lib/market-data.js`

```javascript
// lib/market-data.js
// 从 Finnhub 或 Alpha Vantage 获取公司基础财务数据
// 并结合搜索 API 获取最近重要新闻

export async function fetchCompanyData(ticker) {
  // 1. 基础财务数据（Finnhub 免费 API）
  const profile = await fetch(
    `https://finnhub.io/api/v1/stock/profile2?symbol=${ticker}&token=${process.env.FINNHUB_API_KEY}`
  ).then(r => r.json());

  const quote = await fetch(
    `https://finnhub.io/api/v1/quote?symbol=${ticker}&token=${process.env.FINNHUB_API_KEY}`
  ).then(r => r.json());

  const financials = await fetch(
    `https://finnhub.io/api/v1/stock/metric?symbol=${ticker}&metric=all&token=${process.env.FINNHUB_API_KEY}`
  ).then(r => r.json());

  // 2. 最近新闻（Finnhub News API）
  const news = await fetch(
    `https://finnhub.io/api/v1/company-news?symbol=${ticker}&from=${thirtyDaysAgo()}&to=${today()}&token=${process.env.FINNHUB_API_KEY}`
  ).then(r => r.json());

  const topNews = news.slice(0, 5).map(n => n.headline);

  return {
    companyName: profile.name,
    ticker,
    exchange: profile.exchange,
    currentPrice: `$${quote.c}`,
    marketCap: formatMarketCap(profile.marketCapitalization),
    revenue: formatRevenue(financials.metric?.revenuePerShareTTM),
    revenueGrowth: `${financials.metric?.revenueGrowthTTMYoy?.toFixed(1)}%`,
    netIncome: formatRevenue(financials.metric?.netProfitMarginTTM),
    freeCashFlow: formatRevenue(financials.metric?.freeCashFlowTTM),
    pe: financials.metric?.peBasicExclExtraTTM?.toFixed(1) || 'N/A',
    sector: profile.finnhubIndustry,
    recentNews: topNews,
    reportDate: new Date().toLocaleDateString('zh-CN'),
    fiscalQuarter: getFiscalQuarter(),
  };
}
```

---

## 六、落地页规格（pages/index.js）

### 页面内容结构（中文）

```
┌─────────────────────────────────────────────┐
│  三镜头投资判决                              │
│  专业级公司深度研究报告                      │
│  以巴菲特 · 蒂尔 · 施梅尔三种视角独立评判    │
├─────────────────────────────────────────────┤
│  [体验版报告示例]  → 链接到 GitHub Pages     │
├─────────────────────────────────────────────┤
│  选择套餐                                   │
│  ┌──────────┐  ┌──────────┐               │
│  │单份报告  │  │包月不限  │               │
│  │ CA$18    │  │ CA$198   │               │
│  │Stripe/微信│  │/月       │               │
│  └──────────┘  └──────────┘               │
├─────────────────────────────────────────────┤
│  支付方式选择                               │
│  [信用卡（Stripe）] [微信支付（扫码）]       │
├─────────────────────────────────────────────┤
│  输入公司请求                               │
│  公司名称或股票代码：[_______________]       │
│  您的邮箱（接收报告）：[_______________]     │
│  备注（可选）：[_______________]             │
│  [提交请求]                                 │
├─────────────────────────────────────────────┤
│  📌 报告将在 2-5 分钟内发送到您的邮箱        │
│  📌 同一公司季度内重复请求直接返回缓存版本    │
│  📌 报告有效期：PDF 链接 72 小时有效         │
└─────────────────────────────────────────────┘
```

### 关键 UI 说明
- 移动端优先，支持微信内置浏览器
- 支付按钮根据选择切换 Stripe 或 Payjs 二维码弹窗
- 表单提交后立即跳转 success.js 页面，显示进度轮询
- 语言：中文为主，CAD 定价

---

## 七、状态追踪页（pages/success.js）

```
付款确认 ✓
─────────────────────────────
正在为您生成《${companyName}》三镜头投资报告...

[░░░░░░░░░░░░░░░░░░░░] 处理中...

预计等待时间：2-5 分钟
报告完成后将自动显示下载链接，并发送至您的邮箱。

如等待超过 10 分钟请联系：support@yourdomain.com
```

- 每 5 秒轮询 `GET /api/status?job_id=xxx`
- 完成后自动展示：**[立即下载 PDF]** 按钮
- 同时提示：邮件已发送，链接 72 小时有效

---

## 八、环境变量清单

文件：`.env.example`

```bash
# Anthropic
ANTHROPIC_API_KEY=sk-ant-xxx

# Upstash Redis
UPSTASH_REDIS_REST_URL=https://xxx.upstash.io
UPSTASH_REDIS_REST_TOKEN=xxx

# Cloudflare R2
CLOUDFLARE_ACCOUNT_ID=xxx
R2_ACCESS_KEY_ID=xxx
R2_SECRET_ACCESS_KEY=xxx
R2_BUCKET_NAME=three-lens-reports

# Stripe
STRIPE_SECRET_KEY=sk_live_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx
STRIPE_PRICE_SINGLE=price_xxx    # CA$18 单份
STRIPE_PRICE_MONTHLY=price_xxx   # CA$198 包月

# Payjs（微信支付）
PAYJS_MCHID=xxx
PAYJS_KEY=xxx

# Resend（邮件）
RESEND_API_KEY=re_xxx
FROM_EMAIL=reports@yourdomain.com

# Finnhub（市场数据）
FINNHUB_API_KEY=xxx

# 应用
NEXT_PUBLIC_APP_URL=https://yourdomain.com
REPORT_LINK_EXPIRY_HOURS=72
CACHE_TTL_DAYS=90
```

---

## 九、定价策略

| 套餐 | 价格 | 说明 |
|---|---|---|
| 单份报告 | CA$18 | 一次付费，一份报告，PDF 链接 72小时有效 |
| 月度订阅 | CA$198/月 | 不限份数，同季度缓存版本即时返回 |

**体验版（免费）：** GitHub Pages 上保留 3-5 份示例报告 + 框架说明，作为产品入口。

---

## 十、MVP 范围界定

### 必须实现（P0）
- [ ] 落地页（移动端适配）
- [ ] Stripe 支付（信用卡）
- [ ] Payjs 支付（微信扫码）
- [ ] 核心生成 API（Claude + Prompt）
- [ ] Upstash Redis 缓存（按季度）
- [ ] HTML → PDF 转换
- [ ] Cloudflare R2 存储
- [ ] 邮件发送（PDF 链接）
- [ ] 状态轮询页

### 暂不实现（P1，后续迭代）
- 用户账号系统
- 报告历史记录
- 微信通知推送
- 报告强制刷新（重大新闻触发）
- 管理员后台
- 月度订阅自动续费（Stripe Billing）

---

## 十一、Claude Code 执行指令

按以下顺序执行：

1. **初始化项目**
   ```bash
   npx create-next-app@latest three-lens-reports --js --tailwind --no-app
   cd three-lens-reports
   npm install @anthropic-ai/sdk @upstash/redis @aws-sdk/client-s3 \
     html-pdf-node resend stripe finnhub-js
   ```

2. **创建目录结构**（按第三节文件结构）

3. **实现优先级**
   - 先实现 `lib/anthropic.js`（核心生成逻辑，含完整 Prompt）
   - 再实现 `lib/cache.js`（Redis 缓存）
   - 再实现 `lib/pdf.js`（HTML → PDF）
   - 再实现 `lib/storage.js`（R2 上传）
   - 再实现 `lib/email.js`（Resend）
   - 再实现 `pages/api/generate-report.js`（主接口，串联所有 lib）
   - 再实现 `pages/api/create-payment.js` 和两个 Webhook
   - 最后实现 `pages/index.js` 和 `pages/success.js`

4. **测试顺序**
   - 先用 mock 数据测试 `generate-report.js`（跳过支付验证）
   - 确认 PDF 生成质量与 HTML 版本一致
   - 再集成 Stripe 测试支付（test mode）
   - 再集成 Payjs（沙盒模式）
   - 最后端到端测试完整流程

5. **部署**
   ```bash
   vercel --prod
   ```
   - 在 Vercel Dashboard 设置所有环境变量
   - 在 Stripe Dashboard 配置 Webhook（指向 `/api/webhook-stripe`）
   - 在 Payjs 配置回调 URL（指向 `/api/webhook-payjs`）

---

## 十二、质量保证机制

### Prompt 一致性
- Prompt 模板固化在 `prompts/three-lens-framework.js`，版本控制管理
- 三位投资者的框架描述文字从本项目现有报告中提炼，已经过10份报告的实际验证
- 温度参数（temperature）设为 **0.3**（降低随机性，提高一致性）

### 缓存一致性
- 同一公司同一季度，所有用户获取同一份 PDF
- 季度更新自动触发（新财报发布后第一个请求触发重新生成）

### PDF 质量
- PDF 使用与 GitHub Pages 版本完全一致的 CSS 设计系统
- 字体通过 Google Fonts URL 嵌入（html-pdf-node 支持在线字体加载）
- A4 纸张，打印友好

---

*本文档由 Claude 生成，供 Claude Code 执行使用。如有任何歧义，以本文档为准，不依赖对话上下文。*

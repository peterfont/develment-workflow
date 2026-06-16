# Phase 0 — 需求设计教程

> 目标：把"我想做一个XXX"变成可以执行的描述
>
> **真实案例**：本教程全程以 [Lukittu](https://www.lukittu.com/)（开源软件授权保护服务）作为示例系统。源码：[github.com/KasperiP/lukittu](https://github.com/KasperiP/lukittu)

---

## 为什么需求设计容易被跳过

程序员（尤其前端出身）的本能反应是：
> "我知道要做什么了，直接开干！"

然后写了两周代码，发现：
- 这个功能根本没人用
- 用户要的根本不是这个
- 某个边界情况没想到，要改数据库

**需求设计不是在浪费时间，是在省时间。**

---

## 第一步：找出所有"角色"

问自己：**这个系统有哪些人在用？**

以本项目为例：

```
角色1：普通用户
  → 购买插件授权的主播/运营
  
角色2：渠道商（代理商）
  → 批量购买授权转卖的人
  
角色3：管理员
  → 系统维护者（你自己）
```

**写法：每个角色一句话描述"他是谁，他的目标是什么"**

---

### 💡 Lukittu 是怎么做的

Lukittu 的用户角色分两个维度：

**平台维度（谁在用 Lukittu）：**
```
软件开发者（User）
  → 注册 Lukittu，在 Dashboard 管理自己软件的授权
  → 邮箱密码 / Google / GitHub / TOTP 双因素认证登录

终端用户（Customer）
  → 开发者软件的购买者，持有 License Key
  → 不登录 Lukittu，通过开发者的软件间接使用
```

**团队维度（同一个团队里谁有什么权限）：**
```
Owner（所有者）
  → 创建/删除 Team，管理集成（Stripe/Discord 等）
  → 拥有所有 Member 权限

Member（成员）
  → 管理证书、产品、客户，查看日志统计
  → 不能删除 Team / 修改付费订阅
```

> **设计要点**：Lukittu 没有独立的角色表，Owner 身份通过 `Team.ownerId === User.id` 判断——这是二元模型的简洁实现，适合早期产品。当权限复杂度提升时可再引入 RBAC。

---

## 第二步：写用户故事

格式固定：
> 作为 **[角色]**，我想要 **[做什么]**，以便 **[达到什么目的]**

### 示例：本项目的用户故事

**普通用户：**
```
作为用户，我想要注册账号，以便管理我的授权
作为用户，我想要输入激活码，以便激活插件使用权限
作为用户，我想要续费，以便在到期后继续使用
作为用户，我想要查看到期时间，以便提前安排续费
```

**渠道商：**
```
作为渠道商，我想要生成激活码，以便分发给我的客户
作为渠道商，我想要查看剩余配额，以便决定是否补货
作为渠道商，我想要查看激活记录，以便了解销售情况
```

**管理员：**
```
作为管理员，我想要批量生成卡密，以便直销给用户
作为管理员，我想要给渠道商分配配额，以便发展代理
作为管理员，我想要禁用某个账号，以便处理异常情况
```

### 为什么用这个格式？

因为它强迫你回答三个问题：
1. **谁在用**（角色）
2. **用来干什么**（功能）
3. **为什么要这个功能**（价值）

没有"以便XXX"的功能，往往是不必要的功能。

---

### 💡 Lukittu 是怎么做的

**软件开发者（Dashboard 用户）：**
```
作为开发者，我想要在付款后自动给客户发放 License Key，
  以便不用手动操作，节省时间
作为开发者，我想要绑定 IP + HWID 双重限制，
  以便防止一个授权被多人共用
作为开发者，我想要查看每次验证请求的详细日志（IP、国家、设备），
  以便追踪软件使用情况和异常
作为开发者，我想要邀请团队成员共同管理，
  以便协作运营我的软件业务
作为开发者，我想要集成 Stripe / BuiltByBit / Polymart，
  以便与我已有的销售渠道无缝打通
```

**终端用户（持有 License Key）：**
```
作为终端用户，我想要启动软件时自动验证 License Key，
  以便无感知地确认授权有效
作为终端用户，我想要绑定 Discord 账号，
  以便在购买后自动获得相应的服务器角色
```

---

## 第三步：画核心流程图

不用任何工具，用文字画就行。

**最重要的流程：新用户从零到能用插件**

```
访问官网/门户
    ↓
点击注册
    ↓
填写邮箱 + 密码
    ↓
收验证码邮件
    ↓
验证成功，账号激活
    ↓
登录，进入 Dashboard
    ↓
看到"未激活"状态
    ↓
点击"激活"
    ↓
输入激活码
    ↓
激活成功，获得套餐权限
    ↓
安装 Chrome 插件
    ↓
插件内登录同一账号
    ↓
插件功能解锁，正常使用
```

**关键：画出来之后，问自己每一步**
- 这步可能失败吗？失败了怎么办？
- 用户在这步可能不知道怎么做吗？

---

### 💡 Lukittu 的核心验证流程

Lukittu 的最重要流程是「License Key 验证」，软件每次启动都会经过：

```
软件启动
    ↓
POST /v1/client/teams/{teamId}/verification/verify
  （携带 licenseKey、ipAddress、hwid）
    ↓
服务器执行一系列检查：
  ① 证书是否存在（licenseKeyLookup 哈希查找）
  ② 是否被暂停（suspended）
  ③ 是否已过期（expirationType + expirationDate/Days）
  ④ IP 绑定限制（ipLimit，超出则拒绝）
  ⑤ HWID 绑定限制（hwidLimit，超出则拒绝）
  ⑥ 黑名单检查（IP / HWID / 国家）
  ⑦ 产品/客户/版本绑定匹配
    ↓
返回 { valid: true/false, details: "..." }
    ↓
软件决定继续运行 or 拒绝启动
```

**第三方集成自动发证流程（以 Stripe 为例）：**
```
用户在 Stripe 完成付款
    ↓
Stripe 发送 Webhook → /v1/integrations/stripe?teamId=xxx
    ↓
服务器验签 webhook_secret（防伪造）
    ↓
识别事件类型：
  checkout.session.completed → 创建 License + 发邮件通知客户
  invoice.paid               → 更新/续期 License
  customer.subscription.deleted → 暂停 License
```

> **设计要点**：验证端点 `/v1/client/...` 是公开端点，不需要 API Key。这样客户端软件无需分发敏感凭据，安全性更好。

---

## 第四步：列出边界情况

边界情况 = 不按正常流程走的情况

```
激活流程的边界情况：
  → 激活码输错了？       提示"激活码不存在"
  → 激活码已被用过？     提示"该激活码已使用"
  → 账号未登录就激活？   提示"请先登录"
  → 重复激活（续费）？   累加到期时间，不报错
  → 激活码已过期？       提示"激活码已失效"
```

**这些情况在 Phase 3 写代码时都要处理，现在列出来不会遗漏。**

---

### 💡 Lukittu 的边界情况处理

Lukittu 在 License 验证环节穷举了所有边界：

```
License Key 验证的边界情况：
  → licenseKey 不存在？         返回 valid: false
  → 证书已暂停（suspended）？    返回 valid: false
  → 证书已过期？
      expirationType = DATE      → 检查 expirationDate
      expirationType = DURATION  → 检查创建/首次激活时间 + expirationDays
      expirationType = NEVER     → 永不过期，跳过检查
  → IP 超出 ipLimit？
      hwidTimeout 内同 IP 再次请求 → 不新增，允许（防误判断网）
      真正超出限制 → 拒绝
  → HWID 超出 hwidLimit？         同上逻辑
  → IP / HWID / 国家 在黑名单？   返回 valid: false，hits 计数 +1
  → 证书没有关联该产品？           返回 valid: false（strictProducts 模式）
  → 证书没有关联该客户？           返回 valid: false（strictCustomers 模式）
  → 证书关联的版本不包含请求版本？  返回 valid: false（strictReleases 模式）
```

> **设计要点**："遗忘"机制（forgotten 字段）允许管理员软删除某个 IP/HWID 绑定，让该设备可以重新绑定，而不丢失历史审计记录——这是从用户需求（"我换了路由器/电脑"）出发的设计。

---

## 第五步：确定 MVP 范围

MVP = Minimum Viable Product = 最小可用版本

问自己：**哪些功能是第一版必须有的？哪些可以等有用户了再加？**

### 优先级划分方法

```
P0（必须有，没有就不能用）：
  ✅ 用户注册/登录
  ✅ 激活码激活
  ✅ 插件鉴权检查

P1（重要，但可以晚一点）：
  ⬜ 邮件验证（第一版可以不验证）
  ⬜ 用户门户界面（Postman 测接口也能跑）
  ⬜ 管理后台界面（命令行操作数据库也行）

P2（锦上添花，等有用户再做）：
  ⬜ 渠道商体系
  ⬜ 小红书号绑定限制
  ⬜ 云数据同步
  ⬜ 数据统计图表
```

**P0 做完，系统就能卖了。**

---

### 💡 Lukittu 的功能分层

Lukittu 按订阅套餐来体现功能优先级，这是 SaaS 产品的常见做法：

```
免费层（Free）：
  ✅ 基础 License 管理（证书 CRUD）
  ✅ 产品管理
  ✅ 客户管理
  ✅ 基础验证日志

付费层解锁（对应 P1/P2）：
  ⬜ 更多 License 数量上限（maxLicenses）
  ⬜ 更长日志保留时间（logRetention）
  ⬜ Java 字节码水印（allowWatermarking）
  ⬜ 类加载器保护（allowClassloader）
  ⬜ Stripe / Discord / BuiltByBit / Polymart 集成
  ⬜ Webhook 对外推送
```

> **设计要点**：Lukittu 用 `Limits` 表存每个 Team 的资源上限，而不是硬编码在代码里。这样修改套餐限制只需更新数据库一条记录，不用重新部署。

---

## 第六步：技术选型

> ⚠️ 这步在真实公司里由**架构师**决策，个人项目自己判断。

### 为什么技术选型属于架构师的工作

架构师需要综合评估：
- **当前团队能力**：选大家都会的，不是选最先进的
- **业务规模预期**：1000用户和100万用户选型完全不同
- **长期维护成本**：冷门技术 = 将来招不到人维护
- **生态和社区**：出了问题能不能搜到解决方案
- **技术债风险**：某个技术会不会1年后停止维护

**个人项目选型原则（简化版）：选你最熟悉的、社区最活跃的。**

---

### 选型决策框架

每个技术选型回答三个问题：

```
1. 能不能做到？        → 功能满足需求
2. 出问题好不好查？    → 社区活跃、文档完整
3. 后面好不好扩展？    → 不要把自己逼死
```

---

### 各层技术选型参考

**数据库**

| 选项 | 适合场景 | 备注 |
|------|---------|------|
| SQLite | 本地开发、单机工具 | ❌ 不适合生产并发写入 |
| **MySQL** | 中小型生产环境 | ✅ 大多数项目首选 |
| PostgreSQL | 需要复杂查询/JSON支持强 | 也很好，和MySQL差不多 |
| MongoDB | 数据结构经常变、文档型 | 关系数据不要用 |
| Redis | 缓存、验证码、限流 | 配合MySQL使用，不替代 |

> 架构师视角：数据有明确关联关系（用户→订单→商品）→ 必须用关系型数据库。用 MongoDB 存关系型数据是常见的错误决策。

**后端框架**

| 选项 | 特点 | 适合 |
|------|------|------|
| Node.js + Express | 轻量灵活 | 中小项目，JS全栈 |
| Node.js + NestJS | 重量级，强约束 | 大团队，需要统一规范 |
| Node.js + Fastify | 比Express快 | 性能敏感场景 |
| Python + FastAPI | AI友好 | 有AI功能的项目 |
| Go | 高并发，低资源占用 | 性能要求极高，学习成本高 |

> 架构师视角：NestJS 对个人开发者来说过重，学习曲线陡，Express 够用。等团队超过5人再考虑 NestJS。

**前端框架**

| 选项 | 特点 | 适合 |
|------|------|------|
| Next.js | SSR+CSR，全栈，SEO友好 | 用户门户、官网 |
| React + Vite | 纯前端SPA | 管理后台 |
| Ant Design | 企业组件库，功能全 | 管理后台 |
| shadcn/ui | 现代、可定制 | 用户门户、C端 |
| TailwindCSS | 原子化CSS，开发快 | 配合任何框架使用 |

**ORM vs 原生SQL**

| 选项 | 优点 | 缺点 |
|------|------|------|
| 原生SQL（mysql2） | 直观，性能好 | 字符串拼接，容易出错 |
| Prisma | 类型安全，自动迁移 | 有学习成本，复杂查询较繁琐 |
| Drizzle | 轻量，接近原生SQL | 新兴，生态还在成熟 |
| Sequelize | 老牌ORM | 偏重，新项目不推荐 |

> 架构师视角：TypeScript 项目首选 Prisma（类型推导极好）；JS 项目用原生 SQL + 参数化查询就够；Drizzle 是趋势，值得关注。

---

### 💡 Lukittu 的技术选型

```
前端框架：  Next.js 15 (App Router) + React
数据库：    PostgreSQL + Prisma ORM
缓存/限流：  Redis (ioredis)
文件存储：  Cloudflare R2（S3 兼容接口）
邮件：      SMTP（支持自定义模板）
付款：      Stripe
错误监控：  Sentry
验证码：    Cloudflare Turnstile
Discord Bot：discord.js
Monorepo：  pnpm workspace + Turborepo
部署：      Docker Compose
```

**为什么选 Next.js 而不是前后端分离？**
- Dashboard 和 API 在同一个 Next.js 项目里，减少部署复杂度
- App Router 的 Server Actions / Route Handlers 直接调用数据库，省掉一层独立后端服务
- 对于中小规模的 SaaS，这是最省力的架构

**为什么选 PostgreSQL 而不是 MySQL？**
- Prisma + PostgreSQL 组合生态成熟，迁移文件自动管理
- PostgreSQL 的 JSON 查询能力更强（Metadata 表的扩展字段场景）
- Cloudflare、Vercel、Railway 等平台对 PostgreSQL 支持更好

---

### 技术选型的输出物

一张表，10分钟定完，后面不随便换：

```
数据库：      MySQL 8.0
ORM：         mysql2（原生SQL）
后端框架：    Node.js + Express
用户门户：    Next.js + Tailwind + shadcn/ui
管理后台：    React + Ant Design
部署：        PM2 + Nginx + Ubuntu
数据库GUI：   TablePlus（本地调试用）
接口测试：    Postman
```

**重要：** 技术选型一旦确定，中途不要随便换。换技术栈的代价极高，除非有根本性的理由（性能瓶颈、安全漏洞等）。

---

## 需求设计的输出物

完成 Phase 0，你应该有：

```
1. 角色列表（3-5个角色，每个一句话描述）
2. 用户故事列表（每个角色 3-10 条）
3. 核心流程图（至少画出最重要的那条）
4. 边界情况列表
5. MVP 功能优先级
6. 技术选型表（一张表）
```

不需要写几十页文档，一张 A4 纸就够。

---

## 本项目 Phase 0 结论

```
角色：用户 / 渠道商 / 管理员

MVP 核心链路：
  注册 → 登录 → 激活 → 插件鉴权

第一版不做：
  邮件验证、渠道商、小红书号绑定、数据统计

边界情况已确认：
  重复激活=续费累加、激活码无效/已用/已过期有提示
```

**Phase 0 完成，进入 Phase 1 数据库设计。**

---

### 💡 Lukittu Phase 0 结论（供参考）

```
角色：
  软件开发者（Lukittu 用户）/ 终端用户（持有 License Key 的客户）
  Team Owner / Team Member

MVP 核心链路：
  开发者注册 → 创建 Team → 创建产品 → 创建证书
  → 客户软件启动调用验证 API → 返回 valid: true/false

第一版不做（后续迭代）：
  Stripe/Discord 集成、Java 水印、渠道商体系

边界情况已确认：
  IP/HWID 绑定超限、黑名单、过期三种模式、"遗忘"绑定后可重绑

技术选型：
  Next.js 15 + PostgreSQL + Prisma + Redis + Docker Compose
```

> 参考源码：[github.com/KasperiP/lukittu](https://github.com/KasperiP/lukittu) | 线上产品：[lukittu.com](https://www.lukittu.com/)

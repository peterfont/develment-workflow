# Phase 3 — 后端实现教程

> 目标：把接口设计变成真实可运行的代码
>
> **真实案例**：本教程全程以 [Lukittu](https://www.lukittu.com/) 作为示例系统。源码：[github.com/KasperiP/lukittu](https://github.com/KasperiP/lukittu)

---

## 项目初始化

```bash
# 新建项目目录
mkdir xhs-license-api
cd xhs-license-api

# 初始化 npm 项目
npm init -y

# 安装核心依赖
npm install express          # Web 框架
npm install mysql2           # MySQL 驱动
npm install jsonwebtoken     # JWT 生成和验证
npm install bcryptjs         # 密码加密
npm install dotenv           # 环境变量
npm install cors             # 跨域处理

# 安装开发依赖
npm install -D nodemon       # 热重载（修改代码自动重启）
```

---

### 💡 Lukittu 的项目初始化（Monorepo 方式）

Lukittu 用 **pnpm workspace + Turborepo** 管理 Monorepo：

```bash
# Lukittu 的项目结构
lukittu/
├── apps/
│   └── web/              # Next.js 主应用（前端 + API 一体）
│       ├── src/
│       │   ├── app/
│       │   │   ├── (dash)/   # Dashboard 路由组
│       │   │   ├── (ext)/    # 对外接口路由组
│       │   │   ├── (integ)/  # 集成 Webhook 路由组
│       │   │   └── (int)/    # 内部任务路由组
│       │   └── lib/
│       │       ├── prisma.ts     # Prisma 客户端单例
│       │       ├── redis.ts      # Redis 客户端
│       │       └── utils/
├── packages/
│   ├── database/         # 共享的 Prisma schema
│   └── utils/            # 共享工具函数
├── pnpm-workspace.yaml
└── turbo.json
```

**为什么用 Next.js 而不是独立后端？**
```
独立后端（Express）：
  前端 → HTTP → Express API → 数据库

Next.js Route Handlers：
  浏览器 → Next.js（同一个进程直接查数据库）

优势：
  ✅ 少维护一个服务
  ✅ 类型共享（前后端同一个 TypeScript 项目）
  ✅ 部署简单（只有一个 Docker 镜像）
  
适用场景：中小型 SaaS，团队 < 10 人
```

---

## 项目目录结构

```
xhs-license-api/
├── src/
│   ├── server.js            # 入口文件
│   ├── database.js          # 数据库连接
│   ├── middleware/
│   │   ├── auth.js          # JWT 鉴权中间件
│   │   └── validate.js      # 参数校验中间件
│   ├── routes/
│   │   ├── auth.js          # 注册/登录路由
│   │   ├── activate.js      # 激活/续费路由
│   │   ├── user.js          # 用户信息路由
│   │   └── admin/
│   │       ├── auth.js      # 管理员登录
│   │       ├── users.js     # 用户管理
│   │       └── cardKeys.js  # 卡密管理
│   └── utils/
│       └── response.js      # 统一响应格式工具
├── .env                     # 环境变量（不提交 git）
├── .env.example             # 环境变量示例（提交 git）
├── .gitignore
└── package.json
```

---

## 第一步：配置环境变量

`.env` 文件（**不要提交到 git**）：

```bash
# 数据库
DB_HOST=localhost
DB_PORT=3306
DB_NAME=xhs_license
DB_USER=root
DB_PASSWORD=your_password

# JWT
JWT_SECRET=your-very-long-random-secret-key-change-this
JWT_EXPIRES_IN=30d

# 服务器
PORT=3000
NODE_ENV=development
```

`.env.example`（提交 git，给其他人参考）：

```bash
DB_HOST=localhost
DB_PORT=3306
DB_NAME=xhs_license
DB_USER=root
DB_PASSWORD=

JWT_SECRET=
JWT_EXPIRES_IN=30d

PORT=3000
NODE_ENV=development
```

`.gitignore`：

```
node_modules/
.env
*.db
logs/
```

---

### 💡 Lukittu 的环境变量设计

Lukittu 的 `.env` 涵盖了生产级 SaaS 的完整配置，值得参考：

```bash
# 数据库
DATABASE_URL="postgresql://user:pass@localhost:5432/lukittu"

# Redis（缓存 + 限流）
REDIS_URL="redis://localhost:6379"

# Session
NEXTAUTH_SECRET="随机强密钥"
NEXTAUTH_URL="https://app.lukittu.com"

# 文件存储（Cloudflare R2）
R2_ACCOUNT_ID=""
R2_ACCESS_KEY_ID=""
R2_SECRET_ACCESS_KEY=""
R2_BUCKET_NAME=""

# 邮件（SMTP）
SMTP_HOST=""
SMTP_PORT=465
SMTP_USER=""
SMTP_PASSWORD=""

# Stripe
STRIPE_SECRET_KEY=""
STRIPE_WEBHOOK_SECRET=""

# Discord OAuth
DISCORD_CLIENT_ID=""
DISCORD_CLIENT_SECRET=""

# 内部任务认证
INTERNAL_API_KEY="随机强密钥"

# 错误监控
SENTRY_DSN=""

# 验证码
TURNSTILE_SECRET_KEY=""
```

> **安全要点**：`INTERNAL_API_KEY` 保护 `/api/(int)/...` 内部接口（数据清理、Webhook 重试），防止外部任意触发。所有 Secret 都要用足够长的随机值，不要用简单字符串。

---

## 第二步：数据库连接

`src/database.js`：

```javascript
const mysql = require('mysql2/promise')
require('dotenv').config()

// 创建连接池（比单个连接更好，支持并发）
const pool = mysql.createPool({
  host: process.env.DB_HOST,
  port: process.env.DB_PORT,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  waitForConnections: true,
  connectionLimit: 10,       // 最多10个并发连接
  queueLimit: 0
})

// 测试连接
pool.getConnection()
  .then(conn => {
    console.log('✅ 数据库连接成功')
    conn.release()
  })
  .catch(err => {
    console.error('❌ 数据库连接失败:', err.message)
    process.exit(1)
  })

module.exports = pool
```

---

## 第三步：统一响应格式工具

`src/utils/response.js`：

```javascript
// 成功响应
const success = (res, data, message = '操作成功', statusCode = 200) => {
  return res.status(statusCode).json({
    success: true,
    message,
    data
  })
}

// 失败响应
const fail = (res, message, error = 'ERROR', statusCode = 400) => {
  return res.status(statusCode).json({
    success: false,
    error,
    message
  })
}

module.exports = { success, fail }
```

---

## 第四步：JWT 鉴权中间件

`src/middleware/auth.js`：

```javascript
const jwt = require('jsonwebtoken')
const { fail } = require('../utils/response')

// 用户鉴权中间件
const authMiddleware = (req, res, next) => {
  // 从 Header 取 token
  const authHeader = req.headers.authorization
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return fail(res, '请先登录', 'UNAUTHORIZED', 401)
  }

  const token = authHeader.split(' ')[1]

  try {
    // 验证 token
    const payload = jwt.verify(token, process.env.JWT_SECRET)
    req.user = payload  // 把用户信息挂到 req 上，后续路由可以用
    next()
  } catch (err) {
    if (err.name === 'TokenExpiredError') {
      return fail(res, 'Token 已过期，请重新登录', 'TOKEN_EXPIRED', 401)
    }
    return fail(res, 'Token 无效', 'INVALID_TOKEN', 401)
  }
}

// 管理员鉴权中间件
const adminMiddleware = (req, res, next) => {
  authMiddleware(req, res, () => {
    if (req.user.role !== 'admin' && req.user.role !== 'super_admin') {
      return fail(res, '无权限', 'FORBIDDEN', 403)
    }
    next()
  })
}

module.exports = { authMiddleware, adminMiddleware }
```

---

## 第五步：实现注册/登录路由

`src/routes/auth.js`：

```javascript
const express = require('express')
const bcrypt = require('bcryptjs')
const jwt = require('jsonwebtoken')
const db = require('../database')
const { success, fail } = require('../utils/response')
const { authMiddleware } = require('../middleware/auth')

const router = express.Router()

// POST /api/auth/register - 注册
router.post('/register', async (req, res) => {
  const { email, password } = req.body

  // 1. 参数校验
  if (!email || !password) {
    return fail(res, '邮箱和密码不能为空', 'MISSING_PARAMS')
  }
  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
    return fail(res, '邮箱格式不正确', 'INVALID_EMAIL')
  }
  if (password.length < 6) {
    return fail(res, '密码至少6位', 'PASSWORD_TOO_SHORT')
  }

  try {
    // 2. 检查邮箱是否已注册
    const [existing] = await db.execute(
      'SELECT id FROM users WHERE email = ?',
      [email]
    )
    if (existing.length > 0) {
      return fail(res, '该邮箱已被注册', 'EMAIL_EXISTS')
    }

    // 3. 加密密码（saltRounds=10，标准值）
    const passwordHash = await bcrypt.hash(password, 10)

    // 4. 插入用户
    const [result] = await db.execute(
      'INSERT INTO users (email, password_hash) VALUES (?, ?)',
      [email, passwordHash]
    )

    return success(res, {
      id: result.insertId,
      email
    }, '注册成功', 201)

  } catch (err) {
    console.error('注册失败:', err)
    return fail(res, '服务器错误', 'SERVER_ERROR', 500)
  }
})

// POST /api/auth/login - 登录
router.post('/login', async (req, res) => {
  const { email, password } = req.body

  if (!email || !password) {
    return fail(res, '邮箱和密码不能为空', 'MISSING_PARAMS')
  }

  try {
    // 1. 查用户
    const [users] = await db.execute(
      'SELECT id, email, password_hash, product_key, expires_at, status FROM users WHERE email = ?',
      [email]
    )

    if (users.length === 0) {
      return fail(res, '邮箱或密码错误', 'INVALID_CREDENTIALS')
    }

    const user = users[0]

    // 2. 检查账号状态
    if (user.status === 'disabled') {
      return fail(res, '账号已被禁用', 'ACCOUNT_DISABLED')
    }

    // 3. 验证密码
    const passwordMatch = await bcrypt.compare(password, user.password_hash)
    if (!passwordMatch) {
      return fail(res, '邮箱或密码错误', 'INVALID_CREDENTIALS')
    }

    // 4. 生成 JWT token
    const token = jwt.sign(
      { id: user.id, email: user.email, role: 'user' },
      process.env.JWT_SECRET,
      { expiresIn: process.env.JWT_EXPIRES_IN || '30d' }
    )

    // 5. 更新最后登录时间
    await db.execute(
      'UPDATE users SET last_login_at = NOW() WHERE id = ?',
      [user.id]
    )

    return success(res, {
      token,
      user: {
        id: user.id,
        email: user.email,
        product_key: user.product_key,
        expires_at: user.expires_at
      }
    }, '登录成功')

  } catch (err) {
    console.error('登录失败:', err)
    return fail(res, '服务器错误', 'SERVER_ERROR', 500)
  }
})

// GET /api/auth/me - 获取当前用户信息（需要登录）
router.get('/me', authMiddleware, async (req, res) => {
  try {
    const [users] = await db.execute(
      'SELECT id, email, product_key, expires_at, created_at FROM users WHERE id = ?',
      [req.user.id]
    )
    if (users.length === 0) {
      return fail(res, '用户不存在', 'USER_NOT_FOUND', 404)
    }
    return success(res, users[0])
  } catch (err) {
    return fail(res, '服务器错误', 'SERVER_ERROR', 500)
  }
})

module.exports = router
```

---

## 第六步：实现激活路由

`src/routes/activate.js`：

```javascript
const express = require('express')
const db = require('../database')
const { success, fail } = require('../utils/response')
const { authMiddleware } = require('../middleware/auth')

const router = express.Router()

// POST /api/activate - 激活/续费
router.post('/', authMiddleware, async (req, res) => {
  const { code } = req.body
  const userId = req.user.id

  if (!code) {
    return fail(res, '激活码不能为空', 'MISSING_CODE')
  }

  // 使用事务保证数据一致性
  const conn = await db.getConnection()
  try {
    await conn.beginTransaction()

    // 1. 查激活码
    const [keys] = await conn.execute(
      'SELECT * FROM card_keys WHERE code = ?',
      [code.trim().toUpperCase()]
    )

    if (keys.length === 0) {
      await conn.rollback()
      return fail(res, '激活码不存在', 'CODE_NOT_FOUND')
    }

    const cardKey = keys[0]

    // 2. 检查激活码状态
    if (cardKey.status === 'used') {
      await conn.rollback()
      return fail(res, '该激活码已被使用', 'CODE_USED')
    }
    if (cardKey.status === 'disabled') {
      await conn.rollback()
      return fail(res, '该激活码已失效', 'CODE_DISABLED')
    }

    // 3. 查当前用户的到期时间
    const [userRows] = await conn.execute(
      'SELECT expires_at, product_key FROM users WHERE id = ?',
      [userId]
    )
    const user = userRows[0]
    const prevExpires = user.expires_at

    // 4. 计算新的到期时间（续费：在现有到期时间基础上累加）
    let baseTime
    if (prevExpires && new Date(prevExpires) > new Date()) {
      // 未到期：在现有到期时间上加
      baseTime = new Date(prevExpires)
    } else {
      // 已到期或首次激活：从现在开始算
      baseTime = new Date()
    }
    const newExpires = new Date(baseTime)
    newExpires.setDate(newExpires.getDate() + cardKey.duration_days)

    const isRenewal = !!(prevExpires && new Date(prevExpires) > new Date())

    // 5. 写激活记录
    await conn.execute(
      `INSERT INTO activations
       (user_id, code, product_key, duration_days, prev_expires, new_expires, is_renewal)
       VALUES (?, ?, ?, ?, ?, ?, ?)`,
      [userId, code, cardKey.product_key, cardKey.duration_days,
       prevExpires, newExpires, isRenewal ? 1 : 0]
    )

    // 6. 更新卡密状态
    await conn.execute(
      'UPDATE card_keys SET status="used", used_by=?, used_at=NOW() WHERE id=?',
      [userId, cardKey.id]
    )

    // 7. 更新用户授权状态
    await conn.execute(
      'UPDATE users SET product_key=?, expires_at=? WHERE id=?',
      [cardKey.product_key, newExpires, userId]
    )

    await conn.commit()

    return success(res, {
      product_key: cardKey.product_key,
      expires_at: newExpires,
      duration_days: cardKey.duration_days,
      is_renewal: isRenewal,
      added_days: cardKey.duration_days
    }, isRenewal ? '续费成功' : '激活成功')

  } catch (err) {
    await conn.rollback()
    console.error('激活失败:', err)
    return fail(res, '服务器错误', 'SERVER_ERROR', 500)
  } finally {
    conn.release()
  }
})

// GET /api/activate/check - 鉴权检查（插件专用）
router.get('/check', authMiddleware, async (req, res) => {
  try {
    const [users] = await db.execute(
      'SELECT product_key, expires_at FROM users WHERE id = ? AND status = "active"',
      [req.user.id]
    )

    if (users.length === 0) {
      return fail(res, '用户不存在', 'USER_NOT_FOUND', 404)
    }

    const user = users[0]

    // 未激活
    if (!user.product_key || !user.expires_at) {
      return success(res, {
        valid: false,
        reason: 'NOT_ACTIVATED',
        message: '请先激活'
      })
    }

    // 已过期
    if (new Date(user.expires_at) < new Date()) {
      return success(res, {
        valid: false,
        reason: 'EXPIRED',
        expires_at: user.expires_at,
        message: '授权已过期，请续费'
      })
    }

    // 有效：查套餐权限
    const [products] = await db.execute(
      'SELECT permissions, max_xhs_accounts FROM products WHERE `key` = ?',
      [user.product_key]
    )

    const permissions = products.length > 0
      ? JSON.parse(products[0].permissions || '[]')
      : []

    const daysRemaining = Math.ceil(
      (new Date(user.expires_at) - new Date()) / (1000 * 60 * 60 * 24)
    )

    return success(res, {
      valid: true,
      product_key: user.product_key,
      expires_at: user.expires_at,
      days_remaining: daysRemaining,
      permissions
    })

  } catch (err) {
    console.error('鉴权检查失败:', err)
    return fail(res, '服务器错误', 'SERVER_ERROR', 500)
  }
})

module.exports = router
```

---

## 第七步：入口文件

`src/server.js`：

```javascript
const express = require('express')
const cors = require('cors')
require('dotenv').config()

const app = express()

// 中间件
app.use(cors())                          // 允许跨域
app.use(express.json())                  // 解析 JSON 请求体
app.use(express.urlencoded({ extended: true }))

// 路由
app.use('/api/auth', require('./routes/auth'))
app.use('/api/activate', require('./routes/activate'))
app.use('/api/user', require('./routes/user'))
app.use('/api/admin', require('./routes/admin/auth'))

// 健康检查
app.get('/api/health', (req, res) => {
  res.json({ status: 'ok', time: new Date().toISOString() })
})

// 404 处理
app.use((req, res) => {
  res.status(404).json({ success: false, message: '接口不存在' })
})

// 错误处理
app.use((err, req, res, next) => {
  console.error(err)
  res.status(500).json({ success: false, message: '服务器内部错误' })
})

const PORT = process.env.PORT || 3000
app.listen(PORT, () => {
  console.log(`🚀 服务启动：http://localhost:${PORT}`)
})
```

---

## 关键概念：事务（Transaction）

激活操作涉及多张表的写入，必须用事务：

```javascript
// 事务的含义：
// 以下3步操作要么全部成功，要么全部回滚
// 不能出现"卡密标记已用，但用户授权没更新"这种半成功状态

const conn = await db.getConnection()
try {
  await conn.beginTransaction()    // 开始事务

  // 步骤1：写激活记录
  await conn.execute('INSERT INTO activations ...')

  // 步骤2：更新卡密状态
  await conn.execute('UPDATE card_keys ...')

  // 步骤3：更新用户授权
  await conn.execute('UPDATE users ...')

  await conn.commit()              // 全部成功，提交

} catch (err) {
  await conn.rollback()            // 任何一步失败，全部回滚
  throw err
} finally {
  conn.release()                   // 归还连接到连接池
}
```

---

### 💡 Lukittu 的验证接口核心逻辑

Lukittu 的证书验证是整个系统最核心的代码，完整展示了多步校验的写法：

```typescript
// apps/web/src/app/api/(ext)/v1/client/teams/[teamId]/verification/verify/route.ts

export async function POST(req: Request, { params }) {
  const { teamId } = params
  const body = await req.json()
  const { licenseKey, productId, releaseVersion, hardwareIdentifier } = body

  // 1. 计算 HMAC 查找索引（防止明文存储泄漏）
  const lookup = generateHMAC(licenseKey + teamId)

  // 2. 查证书（用 lookup 走唯一索引，速度快）
  const license = await prisma.license.findUnique({
    where: { licenseKeyLookup: lookup },
    include: { products: true, customers: true, releases: true }
  })

  if (!license) {
    await logRequest({ teamId, status: 'NOT_FOUND', ... })
    return Response.json({ valid: false, details: 'License not found' })
  }

  // 3. 检查是否暂停
  if (license.suspended) {
    return Response.json({ valid: false, details: 'License is suspended' })
  }

  // 4. 检查过期
  const expireCheck = checkExpiration(license)  // 根据 expirationType 分支处理
  if (!expireCheck.valid) {
    return Response.json({ valid: false, details: expireCheck.reason })
  }

  // 5. 检查黑名单（IP / HWID / 国家）
  const clientIp = getClientIp(req)
  const blacklisted = await checkBlacklist(teamId, clientIp, hardwareIdentifier)
  if (blacklisted) {
    await prisma.blacklist.update({ where: ..., data: { hits: { increment: 1 } } })
    return Response.json({ valid: false, details: 'Blacklisted' })
  }

  // 6. 检查/更新 IP 绑定（含 ipTimeout 窗口期逻辑）
  const ipCheck = await handleIpBinding(license, clientIp, teamId)
  if (!ipCheck.valid) {
    return Response.json({ valid: false, details: 'IP limit exceeded' })
  }

  // 7. 检查/更新 HWID 绑定（同上）
  if (hardwareIdentifier) {
    const hwidCheck = await handleHwidBinding(license, hardwareIdentifier, teamId)
    if (!hwidCheck.valid) {
      return Response.json({ valid: false, details: 'HWID limit exceeded' })
    }
  }

  // 8. 记录请求日志（无论成功失败都记）
  await prisma.requestLog.create({
    data: {
      teamId, licenseId: license.id,
      ipAddress: clientIp, country: getCountry(clientIp),
      hardwareIdentifier,
      status: 'VALID', responseTime: Date.now() - startTime,
      type: 'VERIFICATION'
    }
  })

  // 9. 根据 ReturnedFields 配置决定返回哪些字段
  const settings = await getTeamSettings(teamId)
  const details = buildReturnedFields(license, settings.returnedFields)

  return Response.json({ valid: true, details })
}
```

> **设计要点**：每一步校验失败都立即返回，不继续执行（快速失败原则）。请求日志在所有校验结束后统一写入，而不是分散在各个 if 里，保持代码清晰。

---

## package.json 脚本配置

```json
{
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js",
    "db:init": "node src/db-init.js"
  }
}
```

---

## Phase 3 完成标志

```
□ 项目能启动（npm run dev 不报错）
□ 数据库连接成功
□ Postman 测试注册接口成功
□ Postman 测试登录接口，拿到 token
□ Postman 带 token 测试激活接口成功
□ Postman 带 token 测试鉴权检查接口
□ 错误情况测试（重复注册、激活码无效等）
```

**Phase 3 完成，进入 Phase 4 核心联调。**

---

### 💡 Lukittu Phase 3 结论（供参考）

```
技术栈：Next.js 15 Route Handlers + Prisma + Redis
项目结构：Monorepo（pnpm workspace + Turborepo）

核心实现要点：
  1. licenseKeyLookup = crypto.createHmac('sha256', secret)
                              .update(licenseKey + teamId).digest('hex')
     → 数据库只存哈希，防泄漏

  2. Redis 限流（验证接口防刷）：
     const limit = await rateLimit(redis, ip, '100/min')
     if (limit.exceeded) return 429

  3. Webhook 重试状态机：
     PENDING → IN_PROGRESS → DELIVERED / FAILED / RETRY_SCHEDULED
     nextRetryAt 指数退避：1min → 5min → 30min → 2h → 1day

  4. 所有写操作用 Prisma 事务：
     await prisma.$transaction([
       prisma.license.update(...),
       prisma.requestLog.create(...),
     ])
```

> 参考源码：[github.com/KasperiP/lukittu](https://github.com/KasperiP/lukittu) | 线上产品：[lukittu.com](https://www.lukittu.com/)

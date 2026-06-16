# Phase 4-6 — 联调、前端、上线教程

> **真实案例**：本教程全程以 [Lukittu](https://www.lukittu.com/) 作为示例系统。源码：[github.com/KasperiP/lukittu](https://github.com/KasperiP/lukittu)

---

# Phase 4 — 核心联调

> 目标：不依赖任何界面，用工具验证完整业务流程

## 工具：Postman

Postman 是测试 API 的标准工具，下载地址：https://www.postman.com

### 基础使用

```
1. 新建一个 Collection（集合），命名为"XHS License API"
2. 在里面按模块新建 Request（请求）
3. 设置好 URL、Method、Headers、Body 后点 Send
```

### 设置 Token 的技巧

不用每次手动复制粘贴 token，可以用 Postman 的环境变量：

**登录接口的 Tests 标签里写：**
```javascript
// 登录成功后自动保存 token
const response = pm.response.json()
if (response.success && response.data.token) {
  pm.environment.set('token', response.data.token)
}
```

**其他需要登录的接口，Authorization 设置：**
```
Type: Bearer Token
Token: {{token}}    ← 引用环境变量，自动填入
```

---

## 完整测试流程

### 步骤1：准备测试数据

先在数据库里插入一个套餐和一张测试卡密：

```sql
-- 插入套餐
INSERT INTO products (key, name, permissions, max_xhs_accounts)
VALUES ('xhs_pro', '专业版', '["kick","auto_reply","rotation_shelf"]', 3);

-- 插入一张测试卡密（30天）
INSERT INTO card_keys (code, product_key, duration_days, created_by)
VALUES ('TEST-1234-ABCD-5678', 'xhs_pro', 30, 1);

-- 插入管理员账号（密码需要先用 bcrypt 加密）
-- 临时方法：在 Node.js 里运行一下这个
-- const bcrypt = require('bcryptjs')
-- console.log(bcrypt.hashSync('admin123', 10))
INSERT INTO admin_users (username, password, role)
VALUES ('admin', '这里填bcrypt加密后的密码', 'super_admin');
```

### 步骤2：按顺序测试

```
Test 1: 注册
  POST http://localhost:3000/api/auth/register
  Body: { "email": "test@test.com", "password": "123456" }
  期望: success=true, 返回 user.id

Test 2: 登录
  POST http://localhost:3000/api/auth/login
  Body: { "email": "test@test.com", "password": "123456" }
  期望: success=true, 返回 token
  → token 自动保存到环境变量

Test 3: 获取用户信息（验证 token 有效）
  GET http://localhost:3000/api/auth/me
  Authorization: Bearer {{token}}
  期望: 返回用户信息，product_key=null（未激活）

Test 4: 鉴权检查（验证未激活状态）
  GET http://localhost:3000/api/activate/check
  Authorization: Bearer {{token}}
  期望: valid=false, reason="NOT_ACTIVATED"

Test 5: 激活
  POST http://localhost:3000/api/activate
  Authorization: Bearer {{token}}
  Body: { "code": "TEST-1234-ABCD-5678" }
  期望: success=true, 返回 product_key, expires_at

Test 6: 鉴权检查（验证激活后状态）
  GET http://localhost:3000/api/activate/check
  Authorization: Bearer {{token}}
  期望: valid=true, permissions 数组有内容

Test 7: 重复激活同一个码（测试边界情况）
  POST http://localhost:3000/api/activate
  Body: { "code": "TEST-1234-ABCD-5678" }
  期望: success=false, error="CODE_USED"

Test 8: 激活码不存在
  POST http://localhost:3000/api/activate
  Body: { "code": "FAKE-CODE-XXXX" }
  期望: success=false, error="CODE_NOT_FOUND"
```

### 步骤3：验证数据库

测试完成后，检查数据库是否正确写入：

```sql
-- 查激活记录
SELECT * FROM activations;

-- 查用户的当前套餐
SELECT id, email, product_key, expires_at FROM users;

-- 查卡密状态（应该变成 used）
SELECT code, status, used_by, used_at FROM card_keys;
```

### Phase 4 完成标志

```
□ 8个测试用例全部通过
□ 数据库记录正确
□ 边界情况有正确的错误提示
```

---

### 💡 Lukittu 的联调测试策略

Lukittu 的核心联调场景比简单系统更复杂，需要额外测试：

**验证链路完整测试：**
```
Test 1: 创建团队 → 拿到 teamId
Test 2: 创建产品 → 拿到 productId
Test 3: 创建证书（ipLimit=2, hwidLimit=1, expirationType=DURATION, expirationDays=30）
Test 4: 验证证书（valid=true）
Test 5: 再次验证（同一 IP）→ valid=true，IP 不重复计数
Test 6: 用不同 IP 验证 3 次 → 第 3 次 valid=false（超过 ipLimit=2）
Test 7: 暂停证书 → 验证返回 valid=false, 'License is suspended'
Test 8: 把 IP 加入黑名单 → 验证返回 valid=false，blacklist.hits + 1
Test 9: 管理员"遗忘" IP 绑定 → 该 IP 释放，重新绑定成功
```

**Stripe Webhook 测试（用 stripe-cli 本地转发）：**
```bash
# 安装 Stripe CLI
brew install stripe/stripe-cli/stripe

# 本地转发 Webhook 到开发服务器
stripe listen --forward-to localhost:3000/api/(integ)/stripe?teamId=xxx

# 触发测试事件
stripe trigger checkout.session.completed

# 验证：数据库里是否自动创建了 License，客户邮箱是否收到通知邮件
```

> **设计要点**：集成测试时用真实的第三方 Webhook，不要只 mock。因为 Webhook 签名验证、事件类型解析这些逻辑只有真实请求才能完整测出来。

---

# Phase 5 — 前端实现

> 目标：给后端接口套上可以操作的界面

## 原则：接口稳了再写前端

Phase 4 没完成不要开始写前端，否则：
- 接口返回格式变了，前端要跟着改
- 字段名不一致，调试浪费大量时间

## 前端对接后端的关键代码

### 封装 API 请求（axios）

`src/lib/api.ts`（Next.js 项目）：

```typescript
import axios from 'axios'

const API_BASE = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3000'

const api = axios.create({
  baseURL: API_BASE,
  timeout: 10000
})

// 请求拦截：自动带 token
api.interceptors.request.use(config => {
  const token = localStorage.getItem('token')
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

// 响应拦截：统一处理 401
api.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token')
      window.location.href = '/login'
    }
    return Promise.reject(error)
  }
)

export default api
```

### 注册页面核心逻辑

```typescript
const handleRegister = async (e: React.FormEvent) => {
  e.preventDefault()
  setLoading(true)
  setError('')

  try {
    const res = await api.post('/api/auth/register', { email, password })
    if (res.data.success) {
      router.push('/login?registered=true')
    } else {
      setError(res.data.message)
    }
  } catch (err: any) {
    setError(err.response?.data?.message || '注册失败，请稍后重试')
  } finally {
    setLoading(false)
  }
}
```

### 登录后存 token

```typescript
const handleLogin = async () => {
  const res = await api.post('/api/auth/login', { email, password })
  if (res.data.success) {
    localStorage.setItem('token', res.data.data.token)
    localStorage.setItem('user', JSON.stringify(res.data.data.user))
    router.push('/dashboard')
  }
}
```

### 路由守卫（未登录跳转到登录页）

Next.js App Router 中，在需要登录的页面加：

```typescript
// src/app/(portal)/layout.tsx
'use client'
import { useEffect } from 'react'
import { useRouter } from 'next/navigation'

export default function PortalLayout({ children }) {
  const router = useRouter()

  useEffect(() => {
    const token = localStorage.getItem('token')
    if (!token) {
      router.push('/login')
    }
  }, [])

  return <>{children}</>
}
```

---

# Phase 6 — 测试上线

> 目标：让线上服务稳定可访问

## 上线前安全检查

### 必须做的

```
□ 密码用 bcrypt 加密（不能明文存储！）
□ JWT_SECRET 改成随机强密钥（不能用默认值）
□ 数据库密码不能在代码里硬编码，用 .env
□ .env 文件不能提交到 git（检查 .gitignore）
□ SQL 查询用参数化（防 SQL 注入）
  ✅ db.execute('SELECT * FROM users WHERE email = ?', [email])
  ❌ db.execute(`SELECT * FROM users WHERE email = '${email}'`)
□ HTTPS（上线必须用 HTTPS，不能 HTTP）
```

### 建议做的

```
□ 登录接口加限流（防暴力破解）
  npm install express-rate-limit
□ 请求日志（出问题方便排查）
  npm install morgan
□ 生产环境不返回详细错误信息给前端
```

---

### 💡 Lukittu 的安全清单

Lukittu 作为授权保护服务，安全设计比普通项目更严格：

```
✅ 已实现的安全措施：
  □ licenseKey 加密存储 + HMAC 查找（即使 DB 泄漏也无法批量利用）
  □ 密码 bcrypt 加密（saltRounds=12，比默认的10更安全）
  □ Session Cookie（HttpOnly + Secure + SameSite=Strict）
    → 比 localStorage 存 JWT 更安全，防 XSS 窃取
  □ Cloudflare Turnstile 验证码（防机器人注册/暴力破解）
  □ Redis 限流（验证接口 + 登录接口）
  □ Webhook 签名验证（HMAC-SHA256，防伪造 Webhook）
  □ INTERNAL_API_KEY 保护内部接口（防外部触发数据清理）
  □ Prisma 参数化查询（天然防 SQL 注入）
  □ HTTPS + 生产环境 Sentry 错误监控
  □ AuditLog 记录所有管理员操作（谁在何时改了什么）

⚠ 特别注意：
  ReturnedFields 配置控制验证接口返回的字段
  → 不要默认返回客户邮箱等隐私信息
  → 开发者可以按需开关，最小化信息暴露
```

---

## 服务器部署步骤

```bash
# 1. 服务器安装 Node.js
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# 2. 安装 MySQL
sudo apt install mysql-server
sudo mysql_secure_installation

# 3. 创建数据库和用户
mysql -u root -p
CREATE DATABASE xhs_license CHARACTER SET utf8mb4;
CREATE USER 'xhs_api'@'localhost' IDENTIFIED BY '强密码';
GRANT ALL PRIVILEGES ON xhs_license.* TO 'xhs_api'@'localhost';
FLUSH PRIVILEGES;

# 4. 上传代码
git clone 你的仓库
cd xhs-license-api
npm install --production

# 5. 配置 .env（生产环境）
cp .env.example .env
nano .env   # 填入生产环境配置

# 6. 初始化数据库（执行建表 SQL）
mysql -u xhs_api -p xhs_license < schema.sql

# 7. 用 PM2 启动（进程守护，崩溃自动重启）
npm install -g pm2
pm2 start src/server.js --name xhs-api
pm2 save
pm2 startup   # 设置开机自启

# 8. 配置 Nginx 反向代理
sudo nano /etc/nginx/sites-available/api.xhs.com
```

Nginx 配置：

```nginx
server {
  listen 443 ssl;
  server_name api.xhs.com;

  ssl_certificate     /path/to/fullchain.pem;
  ssl_certificate_key /path/to/privkey.pem;

  location / {
    proxy_pass http://localhost:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
}

# HTTP 强制跳转 HTTPS
server {
  listen 80;
  server_name api.xhs.com;
  return 301 https://$host$request_uri;
}
```

## 上线后验证

```bash
# 测试接口是否正常
curl https://api.xhs.com/api/health

# 查看进程状态
pm2 status

# 查看日志
pm2 logs xhs-api

# 查看数据库连接
pm2 logs xhs-api | grep "数据库"
```

## Phase 6 完成标志

```
□ 线上接口能正常访问
□ HTTPS 已开启
□ PM2 运行中，自动重启已配置
□ 用真实环境走一遍完整流程（注册→激活→鉴权）
□ 有数据库备份策略（至少每天自动备份一次）
```

---

### 💡 Lukittu 的部署方案（Docker Compose）

Lukittu 提供了完整的 `docker-compose.yml`，可以一键启动所有依赖：

```yaml
# docker-compose.yml（简化版）
version: '3.8'
services:
  web:
    build: .
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://lukittu:password@postgres:5432/lukittu
      REDIS_URL: redis://redis:6379
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: lukittu
      POSTGRES_USER: lukittu
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

**本地运行 Lukittu：**
```bash
git clone https://github.com/KasperiP/lukittu
cd lukittu
cp apps/web/.env.example apps/web/.env
# 填写 .env 里的必要配置

docker compose up -d
# 访问 http://localhost:3000
```

**与 PM2 + Nginx 方案的对比：**

| | PM2 + Nginx | Docker Compose |
|---|---|---|
| 学习成本 | 低 | 中 |
| 依赖隔离 | ❌ 共享系统环境 | ✅ 容器隔离 |
| 迁移便携 | ❌ 需重新配置 | ✅ 复制文件即可 |
| 适合团队 | 个人/小团队 | 中小团队 ✅ |
| 生产推荐 | 简单单机服务 | 有多个服务的项目 |

---

# 总结：从 0 到上线的最短路径

```
Day 1   Phase 0+1+2：需求 + 数据库设计 + 接口清单
Day 2-4 Phase 3：后端 4 个核心接口（注册/登录/激活/鉴权）
Day 5   Phase 4：Postman 完整测试，跑通所有用例
Day 6-7 Phase 5：前端对接（登录/注册/激活页面）
Day 8   Phase 6：部署上线

= 8天，系统可以给真实用户用了
```

之后的渠道商、小红书号绑定等功能，都是在这个稳定基础上往上加，不影响已上线的功能。

---

### 💡 Lukittu 给独立开发者的启示

Lukittu 是一个值得学习的真实开源项目，它展示了一个**从 0 到生产可用**的完整 SaaS 系统是什么样子：

```
完整性：
  ✅ 用户注册/登录（含 TOTP 双因素认证）
  ✅ 多租户（Team 隔离）
  ✅ 核心业务（License 生命周期管理）
  ✅ 第三方集成（Stripe / Discord / BuiltByBit / Polymart）
  ✅ 操作审计（AuditLog）
  ✅ 错误监控（Sentry）
  ✅ Docker 部署

值得借鉴的模式：
  → licenseKeyLookup HMAC：低成本高价值的安全设计
  → Limits 表存资源上限：套餐变更不需要改代码
  → ReturnedFields 配置：API 响应字段可按需开关
  → Webhook 重试状态机：可靠的异步任务设计
  → 软删除（deletedAt / forgotten）：保留审计历史

适合作为参考的场景：
  需要做软件授权、API Key 管理、用户订阅体系的项目
```

> 源码：[github.com/KasperiP/lukittu](https://github.com/KasperiP/lukittu) | 线上产品：[lukittu.com](https://www.lukittu.com/)

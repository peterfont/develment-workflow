# Phase 2 — 接口设计教程

> 目标：定义前后端之间的"合同"，在写代码前确定数据格式
>
> **真实案例**：本教程全程以 [Lukittu](https://www.lukittu.com/) 作为示例系统。源码：[github.com/KasperiP/lukittu](https://github.com/KasperiP/lukittu)

---

## 什么是 API 接口

接口（API）就是前端和后端之间的约定：

```
前端说：我要注册，给你邮箱和密码
后端说：好，给你返回一个 token

这个约定就是接口
```

用 HTTP 来表达：

```
前端发送：
  POST /api/auth/register
  Content-Type: application/json
  { "email": "test@test.com", "password": "123456" }

后端返回：
  200 OK
  { "success": true, "message": "注册成功" }
```

---

## 接口设计五要素

每个接口都需要确定：

```
1. 方法（Method）   GET / POST / PUT / DELETE
2. 路径（Path）     /api/auth/register
3. 请求参数         发什么数据过去
4. 响应格式         返回什么数据
5. 鉴权要求         需要登录吗
```

---

## HTTP 方法的含义

```
GET     查询（获取数据，不改变数据）
POST    创建（新增数据）
PUT     更新（修改整条数据）
PATCH   部分更新（只改某几个字段）
DELETE  删除
```

**实际项目中的简化用法：**
```
很多项目全用 POST，也能跑
规范做法是区分 GET/POST/PUT/DELETE
本项目建议用规范做法，养成好习惯
```

---

## 统一响应格式

定好一个格式，所有接口都用同一种格式返回：

```json
// 成功
{
  "success": true,
  "data": { ... },
  "message": "操作成功"
}

// 失败
{
  "success": false,
  "error": "INVALID_CODE",
  "message": "激活码不存在或已使用"
}
```

**好处：**
- 前端代码统一处理（只判断 `success` 字段）
- 报错信息统一（用户看到的提示语一致）

---

## HTTP 状态码

```
200  OK              成功
201  Created         创建成功（POST 返回）
400  Bad Request     请求参数有问题
401  Unauthorized    未登录或 token 无效
403  Forbidden       没有权限
404  Not Found       资源不存在
500  Server Error    服务器内部错误
```

**实际使用建议：**
- 业务逻辑错误（激活码无效）→ 返回 `200` + `success: false`（因为请求本身是成功的）
- 系统级错误（未登录、没权限）→ 返回对应 4xx 状态码

---

## 本项目接口清单

### 用户认证模块

```
POST   /api/auth/register      注册         无需登录
POST   /api/auth/login         登录         无需登录
POST   /api/auth/logout        登出         需要登录
GET    /api/auth/me            获取当前用户信息  需要登录
POST   /api/auth/send-code     发送邮箱验证码   无需登录
POST   /api/auth/verify-email  验证邮箱     无需登录（MVP可先跳过）
```

### 激活模块

```
POST   /api/activate           激活/续费    需要登录
GET    /api/activate/check     鉴权检查（插件用）  需要登录
GET    /api/activate/history   激活历史     需要登录
```

### 用户门户模块

```
GET    /api/user/profile       获取个人信息    需要登录
PUT    /api/user/profile       更新个人信息    需要登录
PUT    /api/user/password      修改密码    需要登录
```

### 管理员模块

```
POST   /api/admin/auth/login   管理员登录      无需登录
GET    /api/admin/users        用户列表    需要管理员登录
GET    /api/admin/users/:id    用户详情    需要管理员登录
PUT    /api/admin/users/:id    修改用户    需要管理员登录
GET    /api/admin/card-keys    卡密列表    需要管理员登录
POST   /api/admin/card-keys/generate  批量生成卡密  需要管理员登录
GET    /api/admin/activations  激活记录    需要管理员登录
GET    /api/admin/stats        数据统计    需要管理员登录
```

---

## 关键接口详细设计

### 注册接口

```
POST /api/auth/register

请求体：
{
  "email": "user@example.com",    // 必填
  "password": "mypassword123"     // 必填，至少6位
}

成功响应（200）：
{
  "success": true,
  "message": "注册成功",
  "data": {
    "id": 1,
    "email": "user@example.com"
  }
}

失败响应：
{
  "success": false,
  "error": "EMAIL_EXISTS",
  "message": "该邮箱已被注册"
}
```

**边界情况处理：**
```
邮箱格式不对      → 400, "邮箱格式不正确"
密码太短          → 400, "密码至少6位"
邮箱已注册        → 400, "该邮箱已被注册"
```

---

### 登录接口

```
POST /api/auth/login

请求体：
{
  "email": "user@example.com",
  "password": "mypassword123"
}

成功响应（200）：
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIs...",   // JWT token
    "user": {
      "id": 1,
      "email": "user@example.com",
      "product_key": "xhs_pro",            // 当前套餐，没有则 null
      "expires_at": "2026-07-12T00:00:00Z" // 到期时间，没有则 null
    }
  }
}

失败响应：
{
  "success": false,
  "error": "INVALID_CREDENTIALS",
  "message": "邮箱或密码错误"
}
```

---

### 激活/续费接口

```
POST /api/activate
Authorization: Bearer <token>     // 必须登录

请求体：
{
  "code": "XXXX-XXXX-XXXX-XXXX"  // 激活码
}

成功响应（200）：
{
  "success": true,
  "message": "激活成功",
  "data": {
    "product_key": "xhs_pro",
    "product_name": "专业版",
    "expires_at": "2026-07-12T00:00:00Z",
    "duration_days": 30,
    "is_renewal": false,          // true=续费，false=首次激活
    "added_days": 30
  }
}

失败情况：
{
  "success": false,
  "error": "CODE_NOT_FOUND",     // 激活码不存在
  "message": "激活码不存在"
}
{
  "success": false,
  "error": "CODE_USED",          // 已使用
  "message": "该激活码已被使用"
}
{
  "success": false,
  "error": "CODE_DISABLED",      // 已禁用
  "message": "该激活码已失效"
}
```

---

### 鉴权检查接口（插件专用）

```
GET /api/activate/check
Authorization: Bearer <token>

无请求参数

成功响应（有效授权）：
{
  "success": true,
  "data": {
    "valid": true,
    "product_key": "xhs_pro",
    "expires_at": "2026-07-12T00:00:00Z",
    "days_remaining": 25,
    "permissions": ["kick", "auto_reply", "rotation_shelf"]
  }
}

成功响应（未激活）：
{
  "success": true,
  "data": {
    "valid": false,
    "reason": "NOT_ACTIVATED",
    "message": "请先激活"
  }
}

成功响应（已过期）：
{
  "success": true,
  "data": {
    "valid": false,
    "reason": "EXPIRED",
    "expires_at": "2026-06-01T00:00:00Z",
    "message": "授权已过期，请续费"
  }
}
```

---

## JWT Token 机制

### 什么是 JWT

JWT（JSON Web Token）是一种登录凭证，像一张临时通行证：

```
用户登录成功
  → 服务器生成 token，返回给前端
  → 前端存在 localStorage
  → 后续每个请求带上这个 token
  → 服务器验证 token 有效就放行
```

### Token 长什么样

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.   ← Header（算法信息）
eyJpZCI6MSwiZW1haWwiOiJ0ZXN0QHRlc3QuY29tIn0.  ← Payload（用户信息）
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c   ← Signature（签名，防篡改）
```

**Payload 里存什么：**
```json
{
  "id": 1,
  "email": "test@test.com",
  "role": "user",
  "iat": 1718150400,   // 签发时间
  "exp": 1720742400    // 过期时间（30天后）
}
```

### 前端如何使用 Token

```javascript
// 登录后存储
localStorage.setItem('token', response.data.token)

// 每次请求带上（用 axios 拦截器统一处理）
axios.interceptors.request.use(config => {
  const token = localStorage.getItem('token')
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

// Token 过期（401）自动跳转登录页
axios.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token')
      window.location.href = '/login'
    }
    return Promise.reject(error)
  }
)
```

---

## 接口版本管理

URL 里加版本号，方便以后升级：

```
/api/v1/auth/register   ← 当前版本
/api/v2/auth/register   ← 未来升级，老版本继续兼容
```

---

### 💡 Lukittu 的 API 分层设计

Lukittu 把接口按调用方分成 4 个独立路由组，每组有独立的认证方式：

```
/api/(dash)/...        Dashboard 接口（开发者管理用）
  → 认证：Cookie Session（需要登录）
  → 示例：管理证书、产品、客户、团队设置

/api/(ext)/v1/...      对外公开接口（你的软件客户端调用）
  → /client/...        公开端点，无需 API Key
  → /dev/...           开发者后端调用，需要 API Key（Header: x-api-key）
  → 示例：验证证书、心跳检测、下载版本文件

/api/(integ)/...       第三方集成接口（Webhook 接收）
  → 认证：Webhook Secret 签名验证
  → 示例：接收 Stripe 付款、BuiltByBit 购买通知

/api/(int)/...         内部定时任务接口（不对外暴露）
  → 认证：INTERNAL_API_KEY 环境变量
  → 示例：清理过期数据、重试失败的 Webhook
```

**为什么要这样分层？**

```
❌ 错误做法：所有接口放在 /api/...，靠代码里的 if 判断权限
  → 混乱，容易漏掉权限检查
  → 新加接口时不知道该用哪种认证

✅ 正确做法（Lukittu）：路由分组 = 认证层隔离
  → 每个分组在 layout/middleware 层统一做认证
  → 新加 Dashboard 接口自动继承登录校验，不会漏
```

---

### 💡 Lukittu 的核心接口示例

**证书验证接口（客户端无需认证）：**
```
POST /api/(ext)/v1/client/teams/{teamId}/verification/verify

请求体：
{
  "licenseKey": "XXXX-XXXX-XXXX-XXXX",
  "productId": "uuid-of-product",     // 可选，strictProducts 模式需要
  "releaseVersion": "1.2.0",          // 可选
  "hardwareIdentifier": "hwid-string" // 可选
}

成功响应（有效）：
{
  "valid": true,
  "details": {
    "licenseKey": "XXXX-...",          // 根据 ReturnedFields 配置决定返回哪些字段
    "expirationType": "DATE",
    "expirationDate": "2027-01-01T00:00:00Z",
    "ipLimit": 3,
    "customer": { "email": "...", "fullName": "..." },
    "product": { "name": "MyPlugin" }
  }
}

失败响应：
{
  "valid": false,
  "details": "License is suspended"
}
```

**通过 Dev API 创建证书（需要 API Key）：**
```
POST /api/(ext)/v1/dev/teams/{teamId}/licenses
x-api-key: your-api-key

请求体：
{
  "ipLimit": 2,
  "hwidLimit": 1,
  "expirationType": "DURATION",
  "expirationStart": "ACTIVATION",
  "expirationDays": 30,
  "customerId": "uuid-optional",
  "productIds": ["uuid-of-product"]
}

响应：
{
  "license": {
    "id": "uuid",
    "licenseKey": "XXXX-XXXX-XXXX-XXXX",  // 明文，只在创建时返回一次
    "expirationType": "DURATION",
    ...
  }
}
```

> **设计要点**：`licenseKey` 明文只在创建时返回一次，之后数据库里只存加密值和 HMAC 查找索引。这和密码哈希的设计思路相同。

---

## Phase 2 完成标志

```
□ 所有接口都列出来了（方法、路径、说明）
□ 核心接口有详细的请求/响应格式
□ 每个接口的错误情况都考虑到了
□ 统一响应格式已定义
□ 前端同学看到文档能知道调什么、传什么、收什么
```

---

### 💡 Lukittu Phase 2 结论（供参考）

```
接口分 4 个路由组，认证方式各自独立：
  (dash)  → Cookie Session     管理面板
  (ext)   → 无 / API Key       客户端验证 / 开发者 API
  (integ) → Webhook Secret     第三方平台回调
  (int)   → INTERNAL_API_KEY   内部定时任务

统一响应格式：
  成功：{ valid: true, details: { ... } }     （验证类接口）
  成功：{ [resource]: { ... } }               （CRUD 类接口）
  失败：HTTP 4xx + { message: "错误说明" }

关键设计：
  licenseKey 创建时明文返回一次，后续只存 HMAC 查找索引
  ReturnedFields 配置控制验证接口返回哪些字段（隐私保护）
```

> 参考源码：[github.com/KasperiP/lukittu](https://github.com/KasperiP/lukittu) | 线上产品：[lukittu.com](https://www.lukittu.com/)

**Phase 2 完成，进入 Phase 3 后端实现。**

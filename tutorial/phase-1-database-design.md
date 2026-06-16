# Phase 1 — 数据库设计教程

> 目标：把需求里的"名词"变成数据库里的"表"
>
> **真实案例**：本教程全程以 [Lukittu](https://www.lukittu.com/) 作为示例系统。源码：[github.com/KasperiP/lukittu](https://github.com/KasperiP/lukittu)

---

## 核心思想：找名词

数据库表对应的是现实世界的"事物"（名词），不是"动作"（动词）。

```
❌ 错误思维：我需要一个"注册功能"
✅ 正确思维：我需要存储"用户"这个东西
```

**从用户故事里找名词：**

```
"用户输入激活码激活账号"
        ↑        ↑    ↑
       用户    激活码  账号（同一个东西）

"渠道商生成激活码分配配额"
   ↑        ↑       ↑
 渠道商    激活码    配额

"管理员查看激活记录"
              ↑
           激活记录
```

提取出的实体：
- 用户（users）
- 激活码（card_keys）
- 渠道商（channels）
- 激活记录（activations）
- 产品/套餐（products）

---

## 第一步：画 ER 图（实体关系图）

ER 图只需要搞清楚：**实体之间是什么关系**

### 三种关系

```
一对一（1:1）
  用户 — 云数据
  一个用户只有一份云数据

一对多（1:N）
  用户 — 激活记录
  一个用户可以有多条激活记录

多对多（M:N）
  用户 — 产品（通过激活记录关联）
  一个用户可以激活过多种产品
  一种产品可以被多个用户激活
```

### 本项目 ER 图

```
products（产品）
    ↑ 1
    │ N
card_keys（卡密）──── N ──── 1 ──── channels（渠道商）
    │ 1
    │ N
activations（激活记录）
    │ N
    │ 1
  users（用户）
    │ 1
    │ 1
user_cloud_data（云数据）
```

---

### 💡 Lukittu 的 ER 图（核心部分）

Lukittu 的实体关系比简单项目多了几个值得注意的设计：

**核心授权链路：**
```
Team（团队，核心隔离单元）
  │ 1:N
  ├── Product（产品）
  │     │ 1:N
  │     ├── ReleaseBranch（版本分支，如 stable/beta）
  │     │     │ 1:N
  │     │     └── Release（具体版本，含 DRAFT/PUBLISHED/ARCHIVED 状态）
  │     │           │ 0:1
  │     │           └── ReleaseFile（版本文件，存 R2/S3）
  │     └── ...
  │
  ├── Customer（终端用户/客户）
  │
  └── License（证书）
        │ M:N  ← 一张证书可绑定多个产品、多个客户、多个版本
        ├── Product
        ├── Customer
        ├── Release
        │
        │ 1:N
        ├── IpAddress（IP 绑定记录）
        └── HardwareIdentifier（HWID 绑定记录）
```

**多态扩展设计（Metadata 表）：**
```
Metadata（键值对扩展）
  ├── customerId  FK → Customer
  ├── licenseId   FK → License
  ├── productId   FK → Product
  └── releaseId   FK → Release
  （5个外键只会有1个有值，其余为 null）
```

> **设计要点**：License 和 Product/Customer/Release 是多对多关系，这意味着：
> - 一张证书可以授权给多个产品（套餐包）
> - 一个客户可以有多张证书
> - 表中需要中间表（`_LicenseToProduct` 等 Prisma 隐式关联表）存储这些映射

---

## 第二步：设计每张表的字段

### 如何确定字段

问自己：**要展示/查询这个实体，需要知道它的哪些属性？**

以 `users` 表为例：
```
要知道：
  - 他是谁？        → email（唯一标识）
  - 他能登录吗？    → password_hash
  - 他买了什么套餐？→ product_key
  - 套餐什么时候到期？→ expires_at
  - 账号状态正常吗？→ status
  - 什么时候注册的？→ created_at（每张表都要有）
```

### 字段类型选择指南

```
字符串类型：
  VARCHAR(n)   有最大长度限制，如邮箱 VARCHAR(255)
  TEXT         不限长度，如备注、JSON字符串
  ENUM(...)    固定几个值，如 status ENUM('active','disabled')

数字类型：
  INT          整数，如 id、duration_days
  DECIMAL(m,n) 小数，如金额 DECIMAL(10,2)
  TINYINT(1)   布尔值（0/1），如 is_verified

时间类型：
  DATETIME     日期+时间，如 created_at、expires_at
  DATE         只有日期，如 birth_date

特殊：
  JSON         存结构化数据，如 permissions JSON
               适合经常变化、不需要查询的字段
```

---

## 第三步：字段命名规范

**统一用 snake_case（下划线分隔），不要驼峰**

```sql
✅ 正确
  created_at
  product_key
  expires_at
  is_verified

❌ 错误
  createdAt
  productKey
  expiresAt
  isVerified
```

**每张表必须有的字段：**
```sql
id          INT PRIMARY KEY AUTO_INCREMENT  -- 主键，唯一标识
created_at  DATETIME DEFAULT NOW()           -- 创建时间，排查问题必用
```

**经常用到的可选字段：**
```sql
updated_at  DATETIME DEFAULT NOW() ON UPDATE NOW()  -- 更新时间
status      ENUM('active','disabled') DEFAULT 'active'  -- 软删除用
deleted_at  DATETIME  -- 软删除标记（不真删数据）
```

---

### 💡 Lukittu 的字段命名风格

Lukittu 用 Prisma ORM + PostgreSQL，字段命名用 **camelCase**（因为 Prisma schema 是 TypeScript 风格）：

```prisma
model License {
  id              String    @id @default(uuid())
  licenseKey      String                          // 加密存储的证书 Key
  licenseKeyLookup String   @unique               // HMAC(licenseKey + teamId)，查找索引
  ipLimit         Int       @default(0)           // 0 = 不限制
  hwidLimit       Int       @default(0)
  expirationType  LicenseExpirationType           // NEVER / DATE / DURATION
  expirationStart LicenseExpirationStart?         // CREATION / ACTIVATION
  expirationDate  DateTime?
  expirationDays  Int?
  suspended       Boolean   @default(false)
  teamId          String
  createdAt       DateTime  @default(now())
}
```

> **设计要点**：`licenseKey` 加密存储，`licenseKeyLookup` 存 `HMAC(licenseKey + teamId)` 哈希值作为查找索引。数据库泄漏时攻击者无法直接拿到明文证书 Key，这是一个低成本高价值的安全设计。

---

## 第四步：确定主键和外键

### 主键（PRIMARY KEY）
每张表用 `id INT AUTO_INCREMENT` 作为主键，简单统一。

```sql
id INT PRIMARY KEY AUTO_INCREMENT
```

### 外键（FOREIGN KEY）
用来关联两张表，比如激活记录要知道是哪个用户的：

```sql
-- activations 表里
user_id INT NOT NULL,
FOREIGN KEY (user_id) REFERENCES users(id)
```

**通俗理解：外键就是"这条记录属于谁"**

---

### 💡 Lukittu 的主键选择：UUID vs 自增 INT

Lukittu 所有表都用 **UUID** 作为主键，而不是自增 INT：

```prisma
model License {
  id String @id @default(uuid())
  ...
}
```

**为什么用 UUID？**

| | 自增 INT | UUID |
|---|---|---|
| 可预测性 | ❌ `/licenses/1` → 攻击者能猜到有 `/licenses/2` | ✅ 随机，无法枚举 |
| 分布式 | ❌ 多节点写入会冲突 | ✅ 全局唯一，天然支持分布式 |
| 存储/性能 | ✅ 4字节，索引快 | ⚠ 16字节，略慢但可接受 |
| 暴露信息 | ❌ 总量/增长速度可推断 | ✅ 不泄露业务量 |

> **建议**：对外暴露的 ID（在 URL 或 API 里出现的）用 UUID；内部关联表可以用自增 INT。Lukittu 的所有资源 ID 都在 API 里暴露，所以全部用 UUID。

---

## 第五步：写建表 SQL

### 本项目建表 SQL（MVP 版本）

```sql
-- 1. 产品/套餐表
CREATE TABLE products (
  id           INT PRIMARY KEY AUTO_INCREMENT,
  key          VARCHAR(50) UNIQUE NOT NULL,  -- 如 'xhs_pro'
  name         VARCHAR(100) NOT NULL,        -- 如 '专业版'
  description  TEXT,
  permissions  JSON,           -- ["kick","auto_reply","rotation_shelf"]
  max_xhs_accounts INT DEFAULT 1,   -- 可绑定的小红书账号数
  is_active    TINYINT(1) DEFAULT 1,
  created_at   DATETIME DEFAULT NOW()
);

-- 2. 用户表（前台用户，不是管理员）
CREATE TABLE users (
  id             INT PRIMARY KEY AUTO_INCREMENT,
  email          VARCHAR(255) UNIQUE NOT NULL,
  password_hash  VARCHAR(255) NOT NULL,      -- bcrypt 加密
  username       VARCHAR(100),
  -- 当前授权状态（冗余字段，加速鉴权，不用每次 JOIN）
  product_key    VARCHAR(50),                -- 当前套餐
  expires_at     DATETIME,                   -- 到期时间
  -- 账号状态
  status         ENUM('active','disabled') DEFAULT 'active',
  email_verified TINYINT(1) DEFAULT 0,
  -- 来源（用于统计渠道效果）
  channel_id     INT,
  -- 时间戳
  created_at     DATETIME DEFAULT NOW(),
  updated_at     DATETIME DEFAULT NOW() ON UPDATE NOW(),
  last_login_at  DATETIME
);

-- 3. 卡密表
CREATE TABLE card_keys (
  id            INT PRIMARY KEY AUTO_INCREMENT,
  code          VARCHAR(64) UNIQUE NOT NULL,   -- 用户输入的激活码
  product_key   VARCHAR(50) NOT NULL,          -- 对应套餐
  duration_days INT NOT NULL,                  -- 授权天数
  price         DECIMAL(10,2),                 -- 成本价（管理用）
  status        ENUM('unused','used','disabled') DEFAULT 'unused',
  -- 使用信息
  used_by       INT,                           -- users.id
  used_at       DATETIME,
  -- 管理信息
  created_by    INT NOT NULL,                  -- admin_users.id
  batch_no      VARCHAR(50),                   -- 批次号，方便批量管理
  note          TEXT,
  created_at    DATETIME DEFAULT NOW(),
  FOREIGN KEY (used_by) REFERENCES users(id)
);

-- 4. 激活记录表（每次激活/续费都记一条）
CREATE TABLE activations (
  id            INT PRIMARY KEY AUTO_INCREMENT,
  user_id       INT NOT NULL,
  code          VARCHAR(64) NOT NULL,          -- 使用的激活码
  product_key   VARCHAR(50) NOT NULL,
  duration_days INT NOT NULL,
  -- 续费计算
  prev_expires  DATETIME,                      -- 续费前的到期时间
  new_expires   DATETIME NOT NULL,             -- 续费后的到期时间
  is_renewal    TINYINT(1) DEFAULT 0,          -- 是否是续费（非首次激活）
  activated_at  DATETIME DEFAULT NOW(),
  FOREIGN KEY (user_id) REFERENCES users(id)
);

-- 5. 管理员表（后台管理员，与前台用户完全分开）
CREATE TABLE admin_users (
  id          INT PRIMARY KEY AUTO_INCREMENT,
  username    VARCHAR(50) UNIQUE NOT NULL,
  password    VARCHAR(255) NOT NULL,           -- bcrypt 加密
  role        ENUM('super_admin','admin','operator') DEFAULT 'admin',
  created_at  DATETIME DEFAULT NOW(),
  last_login  DATETIME
);
```

---

## 第六步：验证表设计

用业务场景来验证表是否够用：

### 验证场景1：用户注册并激活

```
1. 注册：INSERT INTO users (email, password_hash) VALUES (...)
2. 激活：
   - 查卡密：SELECT * FROM card_keys WHERE code = 'XXXX' AND status = 'unused'
   - 算到期：new_expires = NOW() + INTERVAL 30 DAY
   - 记录激活：INSERT INTO activations (user_id, code, ..., new_expires) VALUES (...)
   - 更新卡密：UPDATE card_keys SET status='used', used_by=?, used_at=NOW() WHERE id=?
   - 更新用户：UPDATE users SET product_key='xhs_pro', expires_at=? WHERE id=?
3. 鉴权：SELECT product_key, expires_at FROM users WHERE id=? AND status='active'
```

✅ 表结构够用

### 验证场景2：续费

```
1. 查当前到期：SELECT expires_at FROM users WHERE id=?
2. 计算新到期：new_expires = MAX(NOW(), expires_at) + INTERVAL 30 DAY
3. 记录激活：INSERT INTO activations (is_renewal=1, prev_expires=旧值, new_expires=新值)
4. 更新用户：UPDATE users SET expires_at=new_expires WHERE id=?
```

✅ 表结构够用

---

### 💡 Lukittu 如何验证表设计

Lukittu 的验证流程要查多张表，用这些场景验证是否够用：

**场景：软件启动验证证书**
```
1. 接收 licenseKey，计算 HMAC(licenseKey + teamId) → licenseKeyLookup
2. 查证书：SELECT * FROM licenses WHERE licenseKeyLookup = ?
3. 检查 suspended = false
4. 检查过期（根据 expirationType 选择不同逻辑）
5. 查 IP 绑定记录数：SELECT count FROM ip_addresses WHERE licenseId = ?
   → 超过 ipLimit 则拒绝
6. 查黑名单：SELECT * FROM blacklists WHERE teamId = ? AND value = ? AND type = 'IP'
7. 检查产品绑定（License M:N Product 中间表）
8. 写入 RequestLog（ip, country, hwid, status, responseTime）
9. 返回 { valid: true/false }
```

✅ 所有步骤都有对应的表

**场景：管理员"遗忘"一个 HWID 绑定（用户换了设备）**
```
1. 找到 hardware_identifiers 记录
2. 更新：UPDATE hardware_identifiers SET forgotten=true, forgottenAt=NOW() WHERE id=?
3. 下次该 HWID 验证时：查询时过滤 forgotten=false，空位释放
4. 历史记录保留，forgotten=true 的记录不参与 hwidLimit 计数
```

✅ forgotten 字段的设计经过了业务场景验证

---

## 常见设计错误

### 错误1：把所有东西放一张表

```sql
❌ 错误
CREATE TABLE all_data (
  user_email TEXT,
  card_key TEXT,
  activation_time TEXT,
  product_name TEXT,
  ...  -- 一张表几十个字段
);

✅ 正确：拆成 users、card_keys、activations 三张表
```

### 错误2：用数字代替状态

```sql
❌ 错误
status INT   -- 0=未使用, 1=已使用, 2=已禁用（谁记得住？）

✅ 正确
status ENUM('unused','used','disabled')
```

### 错误3：不存历史记录

```sql
❌ 错误：用户续费时直接更新 users.expires_at，没有记录历史

✅ 正确：每次激活/续费都在 activations 表插一条记录
         这样可以追溯：用户什么时候激活的、用了哪个码、续了多少天
```

### 错误4：冗余和一致性问题

```sql
-- users 表存了 product_key 和 expires_at（冗余字段）
-- activations 表也有这些信息

这是故意的！原因：
  ✅ 鉴权时只需要查 users 一张表，速度快
  ✅ activations 是历史记录，users 是当前状态
  ⚠ 代价：更新时要同时更新两个地方（用事务保证一致性）
```

---

### 💡 Lukittu 中体现的反模式对比

**关于冗余字段（RequestLog）：**

Lukittu 的 `RequestLog` 表存了 `ipAddress`、`country`、`hardwareIdentifier` 等字段，这些数据在请求时记录一次就固定了：

```
✅ 正确做法：
  RequestLog.ipAddress = "1.2.3.4"（记录当时的 IP，不关联 IpAddress 表）
  
❌ 错误做法：
  RequestLog.ipAddressId → IpAddress 表（IpAddress 可能被遗忘/删除，日志就断了）
```

> 日志表要求「不可变」，所以冗余存储比外键关联更合适。

**关于软删除（Team.deletedAt）：**

```sql
-- Lukittu 删除 Team 不是 DELETE，而是：
UPDATE teams SET deleted_at = NOW() WHERE id = ?

-- 查询时过滤：
SELECT * FROM teams WHERE deleted_at IS NULL

-- 好处：
-- ① 所有关联数据（证书、客户等）还在，可以恢复
-- ② 定时任务（/v1/internal/data/cleanup）在 N 天后才真正清理
```

---

## 数据库选型

| | SQLite | MySQL | PostgreSQL |
|--|--------|-------|------------|
| 适合场景 | 本地开发、单机小项目 | **中小型生产环境** | 大型复杂查询 |
| 并发能力 | 极差（写操作会锁整个文件） | 好 | 很好 |
| 部署难度 | 零配置 | 简单 | 中等 |
| 本项目选择 | ❌（现有系统的问题所在） | ✅ | 也可以 |

**为什么要从 SQLite 换 MySQL：**
```
SQLite 的写操作会锁整个数据库文件
10个用户同时激活 → 9个会报错或排队很久
MySQL 支持行级锁，并发没问题
```

---

## Phase 1 完成标志

```
□ 画出 ER 图，关系清晰
□ 每张表的字段都能解释为什么需要
□ 用实际业务场景验证过，查询/写入都能实现
□ 建表 SQL 写好了，能直接执行
```

**Phase 1 完成，进入 Phase 2 接口设计。**

---

### 💡 Lukittu Phase 1 结论（供参考）

Lukittu 数据库分 6 个模块，共约 30 张表：

```
👤 用户认证模块：User / UserTOTP / UserRecoveryCode / UserDiscordAccount / Session
🏢 团队管理模块：Team / Invitation / ApiKey / KeyPair / Limits / Settings / Subscription
📦 产品发布模块：Product / ReleaseBranch / Release / ReleaseFile
🔑 核心授权模块：Customer / License / IpAddress / HardwareIdentifier / Blacklist / Metadata
📊 日志与监控：RequestLog / AuditLog / Webhook / WebhookEvent
🔌 第三方集成：StripeIntegration / DiscordIntegration / BuiltByBitIntegration / PolymartIntegration
```

**三个关键设计决策：**
1. `licenseKeyLookup` = HMAC 哈希查找，防数据库泄漏后证书被批量利用
2. `Metadata` 多态设计，一张表扩展多个实体的键值对（免得加一个字段就改表结构）
3. `IpAddress.forgotten` / `HardwareIdentifier.forgotten` 软删除绑定记录，支持"换设备不丢授权"

> 参考源码：[github.com/KasperiP/lukittu](https://github.com/KasperiP/lukittu) | 线上产品：[lukittu.com](https://www.lukittu.com/)

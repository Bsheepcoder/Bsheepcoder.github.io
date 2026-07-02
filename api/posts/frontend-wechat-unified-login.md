## 一、元认知：微信登录到底在解决什么问题

> 登录的本质不是"让用户进来"，而是"确认这个人是谁"。

这句话看似废话，但它揭示了微信登录体系最令人困惑的设计：**同一个用户，在公众号、小程序、网站应用、移动应用中有四个不同的 openid**。如果你不理解为什么，后面的实现做得再好也是空中楼阁。

### 1.1 openid 的分裂：设计哲学而非技术缺陷

微信的 openid 是**应用维度**的，不是**用户维度**的。一个用户在你的小程序里是 `oxABC123`，在你的网站应用里是 `oxDEF456`，在你的公众号里是 `oxGHI789`——三个完全不同的字符串，指向同一个人。

为什么这么设计？**隔离性**。每个应用只能看到自己的 openid，无法跨应用追踪用户。这是隐私保护的第一道防线：你的小程序无法知道用户在另一个小程序里是谁。

但业务需要打通。所以微信提供了 **UnionID 机制**：只要多个应用绑定到同一个微信开放平台账号，就能获取一个全局唯一的 `unionid`，作为用户身份的统一标识。

```
用户（张三）
  ├── 小程序 openid: oxAAA    ──┐
  ├── 网站应用 openid: oxBBB  ──┼── 都绑定到同一个开放平台 → unionid: ou_唯一
  └── 公众号 openid: oxCCC    ──┘
```

> **小而美原则**：如果你只有一个应用（纯小程序或纯网站），不需要 UnionID，直接用 openid 就够了。只有当你的产品矩阵需要跨端打通时，才值得引入 UnionID。

### 1.2 三种登录方式的本质

微信提供的登录能力可以归为三类：

| 方式 | 本质 | 适用场景 | 前提条件 |
|------|------|---------|---------|
| 网页扫码登录 | OAuth2（开放平台） | PC 网站、H5（非微信内） | 开放平台注册「网站应用」 |
| 小程序登录 | 微信私有协议 | 微信小程序 | 小程序 AppID |
| 网页授权登录 | OAuth2（服务号） | 公众号 H5 页面 | 已认证服务号 |

本文聚焦前两种——它们是"小而美"产品最常见的组合：**PC 端扫码 + 小程序端一键登录**。

---

## 二、搭积木：统一认证服务架构

### 2.1 数据库设计：一张表搞定

不需要两张表、三张表。一张 `user` 表，核心字段只有三个：

| 字段 | 类型 | 说明 |
|------|------|------|
| `unionid` | VARCHAR(64) UNIQUE | 微信统一标识，可为空 |
| `web_openid` | VARCHAR(64) | 网站应用的 openid |
| `mini_openid` | VARCHAR(64) | 小程序的 openid |

**设计决策**：`unionid` 允许为空。如果用户只用过小程序登录，且小程序没有绑定开放平台，就没有 unionid。用 openid 也能正常工作，只是无法跨端打通。

其余字段按需添加：`nickname`、`avatar_url`、`phone`、`created_at`、`last_login_at`、`last_login_src`（记录最近登录来源，方便分析）。

索引策略：
- `unionid` 唯一索引（主查询路径）
- `web_openid` 普通索引（兜底查询）
- `mini_openid` 普通索引（兜底查询）

### 2.2 后端架构（Spring Boot）

```
com.example.auth
├── controller/AuthController          ← API 端点（5个接口）
├── service/WechatWebAuthService       ← 网页扫码登录核心逻辑
├── service/WechatMiniAuthService      ← 小程序登录核心逻辑
├── security/JwtTokenProvider          ← JWT 签发/验证
└── config/WechatProperties            ← 微信配置（appid/secret）
```

配置结构（`application.yml`）：

```yaml
wechat:
  web:
    app-id: ${WECHAT_WEB_APP_ID}       # 开放平台网站应用 AppID
    app-secret: ${WECHAT_WEB_APP_SECRET}
    redirect-uri: https://yourdomain.com/api/auth/wechat/web/callback
  mini:
    app-id: ${WECHAT_MINI_APP_ID}      # 小程序 AppID
    app-secret: ${WECHAT_MINI_APP_SECRET}
  jwt:
    secret: ${JWT_SECRET}               # 至少 256 bit
    access-token-expire: 7200           # 2 小时
    refresh-token-expire: 2592000       # 30 天
```

> **安全原则**：所有密钥通过环境变量注入，绝不硬编码。Spring Boot 的 `@ConfigurationProperties` 自动绑定，配合 Docker/K8s 的 secret 管理，密钥不出现在代码和配置文件中。

---

## 三、案例即原理：核心实现步骤

### 3.1 网页扫码登录：六步完成

> **端点选择**：开放平台网站应用用 `connect/qrconnect`（scope=`snsapi_login`），服务号网页授权用 `connect/oauth2/authorize`（scope=`snsapi_userinfo`）。两者都能扫码，但前者是 PC 端专用，后者仅限微信内置浏览器。本文用前者。

网页扫码有两种呈现方式，选哪种取决于你的产品需求：

| 方式 | 体验 | 适用场景 |
|------|------|---------|
| 跳转式 | 跳到微信页面扫码，再跳回来 | 简单快速，适合 MVP |
| 内嵌式 | 二维码直接嵌在你的页面里 | 体验更好，适合正式产品 |

**跳转式流程**（六步）：

```
浏览器                    你的后端                    微信服务器
  │                         │                          │
  │  1. 点击"微信登录"        │                          │
  │ ──────────────────────> │                          │
  │                         │                          │
  │  2. 302 重定向到微信授权页  │                          │
  │ <────────────────────── │                          │
  │                         │                          │
  │  3. 用户扫码确认           │                          │
  │ ─────────────────────────────────────────────────> │
  │                         │                          │
  │  4. 微信回调 redirect_uri?code=xxx&state=yyy        │
  │ <───────────────────────────────────────────────── │
  │                         │                          │
  │  5. 前端将 code 发到后端    │                          │
  │ ──────────────────────> │                          │
  │                         │  6. 用 code 换 access_token │
  │                         │ ────────────────────────> │
  │                         │ <──────────────────────── │
  │                         │                          │
  │                         │  7. 查找/创建用户，签发 JWT   │
  │  8. 返回 JWT              │                          │
  │ <────────────────────── │                          │
```

**每一步做什么**：

**第一步：前端生成 state 并跳转**

前端生成一个随机字符串作为 `state`（防 CSRF），存入 `sessionStorage`，然后将用户重定向到后端的 `/api/auth/wechat/web/login?state=xxx`。

后端收到后，将 `state` 存入 Redis（设 5 分钟过期），然后 302 重定向到微信授权页：

```
https://open.weixin.qq.com/connect/qrconnect
  ?appid=你的网站应用AppID
  &redirect_uri=你的回调地址（需 URL 编码）
  &response_type=code
  &scope=snsapi_login
  &state=刚才的随机字符串
  #wechat_redirect
```

**第二步：用户扫码授权**

用户在微信中确认授权后，微信会将浏览器重定向到你配置的 `redirect_uri`，并附带 `code` 和 `state` 参数。

**第三步：后端校验 state 并换取 token**

后端收到回调后，先检查 Redis 中是否存在这个 `state`——存在则删除（一次性使用），不存在则拒绝请求（可能是 CSRF 攻击）。

校验通过后，用 `code` 调用微信接口换取 `access_token`：

```
GET https://api.weixin.qq.com/sns/oauth2/access_token
  ?appid=你的网站应用AppID
  &secret=你的网站应用Secret
  &code=回调带回的code
  &grant_type=authorization_code
```

返回结果包含：`access_token`（用户授权凭证）、`openid`（该用户在你网站应用中的标识）、`unionid`（如果已绑定开放平台）。

**第四步：查找或创建用户**

用 `unionid`（优先）或 `web_openid` 查询数据库：
- 找到 → 更新 `last_login_at`，补充缺失的 `unionid`
- 没找到 → 插入新用户记录

**第五步：签发 JWT**

生成两个 Token：
- `access_token`：有效期 2 小时，用于接口认证
- `refresh_token`：有效期 30 天，用于刷新 access_token

**第六步：前端存储 Token**

前端收到 JWT 后存入 `localStorage`（或 HttpOnly Cookie），后续请求通过 `Authorization: Bearer <token>` 头携带。

**内嵌式方案**（推荐用于正式产品）：

跳转式的体验不好——用户被带离你的页面，扫码后又跳回来，中间有白屏闪烁。微信提供了内嵌二维码方案，用户始终留在你的页面上。

步骤很简单：
1. 在页面中引入微信提供的 JS 文件：`http://res.wx.qq.com/connect/zh_CN/htmledition/js/wxLogin.js`
2. 在登录区域创建一个容器 `<div id="login_container"></div>`
3. 实例化 `WxLogin` 对象，传入你的 appid、scope、redirect_uri 等参数

二维码会自动渲染到容器中。用户扫码授权后，微信通过 **JS 跨域回调**将 `code` 返回给当前页面（而非跳转 redirect_uri），前端直接拿到 `code` 发给后端。

> **本地开发注意**：内嵌式方案的 `redirect_uri` 必须与当前页面同域。如果你在 `localhost:5173` 开发，redirect_uri 也必须是 `localhost:5173` 下的路径。这意味着后端回调接口需要在前端同域下，或者用 Vite proxy 转发。

### 3.2 小程序登录：三步完成

小程序登录比网页扫码简单得多——用户无感知，后台静默完成。

**第一步：小程序端调用 wx.login()**

```javascript
wx.login({
  success(res) {
    // res.code 是一次性凭证，有效期 5 分钟
    // 将 code 发送到你的后端
  }
})
```

**第二步：后端用 code 换取用户标识**

调用微信接口：

```
GET https://api.weixin.qq.com/sns/jscode2session
  ?appid=你的小程序AppID
  &secret=你的小程序Secret
  &js_code=前端传来的code
  &grant_type=authorization_code
```

返回结果包含：`session_key`（会话密钥，用于解密敏感数据）、`openid`（小程序维度的用户标识）、`unionid`（如果已绑定开放平台）。

> **安全红线**：`session_key` 绝不能返回给前端。它是解密 `encryptedData` 的密钥，泄露等于用户数据裸奔。只存在后端，建议存 Redis。

**第三步：查找或创建用户 + 签发 JWT**

逻辑与网页登录完全一致：用 `unionid` 或 `mini_openid` 查找用户，签发双 Token。

### 3.3 手机号获取：新版 vs 旧版

小程序获取手机号有两种方式，2023 年后微信改了接口：

**旧版（已废弃，但存量代码仍大量存在）**：
- 前端获取 `encryptedData` + `iv`
- 后端用 `session_key` 做 AES-128-CBC 解密
- 问题：`session_key` 需要在后端存储和传输，增加了泄露风险

**新版（推荐）**：
- 前端点击 `<button open-type="getPhoneNumber">`，获取 `code`
- 后端用 `code` 直接调 `getuserphonenumber` 接口换取手机号
- 无需 `session_key` 参与，更安全

```
POST https://api.weixin.qq.com/wxa/business/getuserphonenumber
  ?access_token=小程序全局access_token
Body: { "code": "前端传来的code" }
```

> **注意**：这里的 `access_token` 是小程序全局接口调用凭据（通过 appid + secret 获取），不是用户授权的 access_token。两者完全不同，别搞混。

### 3.4 前端 Token 管理：双 Token 刷新机制

单 Token 的困境：有效期短则频繁登录，有效期长则泄露风险大。

**双 Token 方案**：

```
access_token:   有效期 2 小时，每次请求携带
refresh_token:  有效期 30 天，仅在 access_token 过期时使用
```

**刷新流程**：

1. 请求返回 401 → 前端拦截器检测到
2. 用 `refresh_token` 调 `/api/auth/refresh` 获取新的双 Token
3. 用新的 `access_token` 重试原请求
4. 如果 `refresh_token` 也过期 → 跳转登录页

**关键细节**：
- 刷新过程中如果有多个并发 401 请求，只发一次 refresh 请求，其他请求排队等待
- `refresh_token` 一次性使用——每次刷新后旧的立即失效，返回新的
- 这是微信自己的做法：refresh_token 每次使用后都会更新

### 3.5 小程序端 Token 存储

小程序没有 `localStorage`，用 `wx.setStorageSync` 存 Token。后续请求在 header 中携带：

```javascript
// 封装请求方法
function request(url, data) {
  return new Promise((resolve, reject) => {
    wx.request({
      url: API_BASE + url,
      data,
      header: {
        'Authorization': 'Bearer ' + wx.getStorageSync('accessToken')
      },
      success: (res) => {
        if (res.statusCode === 401) {
          // 尝试刷新 token
          refreshToken().then(() => request(url, data).then(resolve).catch(reject))
        } else {
          resolve(res.data)
        }
      },
      fail: reject
    })
  })
}
```

---

## 四、缺陷与安全：文档不会告诉你的事

### 4.1 state 参数：你以为的 CSRF 防护够了吗

微信 OAuth2 要求 `state` 参数必填，用于防止 CSRF 攻击。但很多开发者只是随便传个固定字符串。

**正确做法**：
1. 前端生成随机 `state`，存入 `sessionStorage`
2. 后端同时在 Redis 中存一份（设 5 分钟过期）
3. 回调时双重校验：前端校验 `sessionStorage` 中的值，后端校验 Redis 中的值
4. 校验通过后立即删除，不可重复使用

### 4.2 session_key 绝不能下发前端

这是微信安全规范中最容易踩的坑。小程序的 `session_key` 是解密 `encryptedData` 的密钥，如果泄露，攻击者可以伪造任意用户数据。

**正确做法**：`session_key` 只存在后端（推荐 Redis），通过 openid 关联，有效期 30 分钟。前端需要解密数据时，把 `encryptedData` 和 `iv` 发给后端，后端用存储的 `session_key` 解密后返回明文。

### 4.3 两种 access_token 的区别

微信体系中有两种名字相同但完全不同的 `access_token`：

| 类型 | 用途 | 获取方式 | 有效期 |
|------|------|---------|--------|
| 用户 access_token | 换取用户信息、刷新授权 | 用 code 换取 | 2 小时 |
| 小程序全局 access_token | 调用服务端 API（如获取手机号） | 用 appid + secret 获取 | 2 小时 |

**别搞混**。用户 access_token 是用户维度的，全局 access_token 是应用维度的。后者建议缓存到 Redis，定时刷新。

### 4.4 防重放攻击

微信的 `code` 参数有一个重要特性：**只能使用一次，5 分钟过期**。但如果你的后端没有做好幂等性，攻击者可以在 code 被使用前抢先把请求发到你的服务器。

**防御方案**：用 Redis 做幂等锁。收到 code 后，先 `SETNX` 一个 key（key = code，过期 5 分钟），设置成功才继续处理，设置失败说明已被使用。

### 4.5 微信"快照页模式"的影响

2022 年 7 月起，微信对不规范使用 `snsapi_userinfo` 授权的网页，会默认打开"快照页模式"——用户看到的是静态页面快照，无法完成授权。

**触发条件**：页面一打开就直接跳转授权，没有任何用户交互。

**规避方法**：扫码登录必须由用户主动点击按钮触发，不要做页面加载时自动跳转。

### 4.6 JWT 安全要点

| 风险点 | 防御措施 |
|--------|---------|
| JWT 被盗 | 存 HttpOnly Cookie（防 XSS 读取），或短期有效 + refresh 机制 |
| JWT 泄露后无法撤销 | 维护一个 Redis 黑名单，logout 时将 token 加入 |
| secret 被破解 | 至少 256 bit，通过环境变量注入，定期轮换 |
| 并发刷新互踢 | refresh 请求加锁（Redis 分布式锁），只允许一个请求执行刷新 |

### 4.7 接口限流

登录接口是攻击者的重点目标。建议：
- 同一 IP 每分钟最多 10 次登录请求
- 同一 openid 每小时最多 20 次
- 触发限流后返回 429，前端提示"操作过于频繁"

---

## 五、总结：小而美的登录系统应该具备什么

### 最小可用功能清单

| 功能 | 必要性 | 说明 |
|------|--------|------|
| 网页扫码登录 | 必须 | PC 端主入口 |
| 小程序一键登录 | 必须 | 小程序端主入口 |
| JWT 双 Token | 必须 | 安全 + 体验的平衡 |
| UnionID 打通 | 推荐 | 跨端用户统一 |
| 手机号绑定 | 推荐 | 兜底登录方式 |
| Token 自动刷新 | 必须 | 前端无感续期 |
| 接口限流 | 必须 | 防暴力破解 |

### 扩展路径

```
v1: 小程序登录 + JWT
  ↓
v2: 加网页扫码登录 + UnionID 打通
  ↓
v3: 加手机号验证码登录（兜底）
  ↓
v4: 加 Apple 登录（iOS 必备）
  ↓
v5: 加多设备管理 + 踢人下线
```

### 本地开发：完整端到端流程

网页扫码登录的核心难题：微信回调需要一个**公网可访问的域名**，但你的后端跑在 `localhost:8080`。解决方案是**内网穿透**——把本地端口映射到公网域名。

**工具选择**：

| 工具 | 免费版 | 固定域名 | 速度 | 推荐场景 |
|------|--------|---------|------|---------|
| ngrok | 有 | 付费版才有（$8/月） | 快 | 海外项目、临时测试 |
| cpolar | 有 | 免费版支持 | 中等 | 国内项目首选 |
| frp | 开源 | 自建服务器 | 最快 | 有公网服务器时最自由 |

**完整步骤**（以 cpolar 为例）：

```
步骤 1: 启动后端
  Spring Boot → localhost:8080

步骤 2: 启动内网穿透
  cpolar http 8080
  → 获得公网地址，如 https://xxxx.cpolar.top

步骤 3: 配置微信开放平台
  登录 open.weixin.qq.com
  → 网站应用 → 授权回调域 → 填 xxxx.cpolar.top

步骤 4: 启动前端
  Vite dev server → localhost:5173
  vite.config.ts 中配置 proxy：
    /api → http://localhost:8080

步骤 5: 测试扫码登录
  浏览器打开 localhost:5173
  → 点击"微信登录"按钮
  → 跳转到微信扫码页（或内嵌二维码）
  → 手机扫码确认
  → 微信回调到 xxxx.cpolar.top/api/auth/wechat/web/callback
  → cpolar 转发到 localhost:8080
  → 后端处理，返回 JWT
  → 前端登录成功
```

**常见坑**：

| 问题 | 原因 | 解决 |
|------|------|------|
| "该链接无法访问" | redirect_uri 域名与开放平台配置不一致 | 确保回调域名完全匹配（不含 http:// 和路径） |
| 回调后页面白屏 | 前端没有处理回调路由 | 在 Vue Router 中添加 `/callback` 路由 |
| 内网穿透重启后失效 | 免费版域名变了 | 更新开放平台配置，或升级付费版固定域名 |
| 扫码后无反应 | state 校验失败 | 检查 sessionStorage 和 Redis 中的 state 是否一致 |

**小程序本地开发**：

小程序登录没有域名限制——微信开发者工具运行在本地，可以直接调用你的 `localhost:8080` 后端。只需在开发者工具中勾选「不校验合法域名」即可。

### 安全检查清单

上线前逐项确认：

- [ ] `state` 参数随机生成 + 一次性校验（Redis）
- [ ] `session_key` 只存后端，不下发前端
- [ ] JWT secret 至少 256 bit，通过环境变量注入
- [ ] `refresh_token` 一次性使用（刷新后旧的失效）
- [ ] `code` 做幂等锁，防重放
- [ ] 登录接口限流（10次/分钟/IP）
- [ ] 日志记录所有登录事件（不记录 token 本身）
- [ ] 敏感操作二次验证（如解绑手机号）
- [ ] 定期轮换 JWT secret（需考虑存量 token 兼容）

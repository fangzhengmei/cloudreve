# Cloudreve 用户会话鉴权机制深度解析

## 一、整体架构概览

Cloudreve 采用 **Master/Slave** 双模式架构，每种模式拥有独立的路由和中间件链。鉴权体系由四层机制构成：

| 鉴权方式 | 适用场景 | 核心文件 |
|---------|---------|---------|
| **JWT Token** | 用户浏览器登录、OAuth 客户端访问 | [jwt.go](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/auth/jwt.go) |
| **Session (Cookie)** | CSRF 防护、传统会话状态 | [session.go](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/middleware/session.go) |
| **HMAC 签名** | 主从节点通信、URL 签名、上传回调、文件下载 | [hmac.go](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/auth/hmac.go)、[auth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/auth/auth.go) |
| **HTTP Basic Auth** | WebDAV 协议访问 | [auth.go#L103-L175](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/middleware/auth.go#L103-L175) |

---

## 二、Master 模式请求中间件链与上下文注入顺序

请求进入 Master 模式时，中间件按以下顺序执行（参见 [router.go#L201-L239](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/routers/router.go#L201-L239)）：

```
HTTP Request
    │
    ▼
┌─────────────────────────────┐
│ 1. gin.Recovery()           │  异常恢复
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│ 2. InitializeHandling(dep)  │  注入依赖、Correlation ID、
│    [common.go#L96-L128]     │  RequestInfo、Logger、SlaveNodeID
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│ 3. Logging()                │  请求日志记录
│    [common.go#L142-L160]    │
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│ 4. CORS (可选)              │  跨域处理
│    [router.go#L181-L198]    │
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│ 5. gzip (排除 /api/)        │  响应压缩
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│ 6. SharePreview             │  分享预览
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│ 7. FrontendFileHandler      │  前端静态文件
└─────────────────────────────┘
    │
    ▼
┌───────────────────────────────────────┐
│ 8. Session(dep)  ◄── Cookie Session   │
│    [session.go#L22-L46]               │
│    • 创建基于 KV 的会话存储            │
│    • Cookie: cloudreve-session        │
│    • HttpOnly, MaxAge=60天, SameSite  │
└───────────────────────────────────────┘
    │
    ▼
┌───────────────────────────────────────┐
│ 9. CurrentUser()  ◄── JWT 鉴权        │
│    [auth.go#L48-L71]                  │
│    • 解析 Authorization: Bearer xxx   │
│    • 注入 UserIDCtx (int)             │
│    • 注入 ScopeContextKey ([]string)  │
│    • 调用 SetUserCtx → UserCtx        │
└───────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│ 10. CacheControl()          │  禁用缓存
└─────────────────────────────┘
    │
    ▼
      业务路由 (v4 Group)
```

### 上下文注入详细说明

#### 阶段一：InitializeHandling — 请求基础元数据注入（[common.go#L96-L128](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/middleware/common.go#L96-L128)）

这是最先执行的中间件，负责将所有基础设施注入请求 context：

```go
// 注入内容：
ctx = dep.ForkWithLogger(c.Request.Context(), l)                    // 依赖容器（带日志）
ctx = context.WithValue(ctx, logging.CorrelationIDCtx{}, cid)       // 关联 ID
ctx = context.WithValue(ctx, requestinfo.RequestInfoCtx{}, reqInfo) // 请求信息(IP/Host/UA/ClientID)
ctx = context.WithValue(ctx, logging.LoggerCtx{}, l)                // Logger
ctx = context.WithValue(ctx, cluster.SlaveNodeIDCtx{}, nodeId)      // 从机节点ID
c.Request = c.Request.WithContext(ctx)
```

**关键设计**：所有值注入到 `c.Request.Context()`（标准 `context.Context`），而非 gin 的 `c.Set()`。后续通过 `dependency.FromContext(c)` 从 gin.Context 获取依赖，底层通过 `c.Request.Context()` 传递。

#### 阶段二：Session 中间件 — Cookie 会话（[session.go#L22-L46](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/middleware/session.go#L22-L46)）

使用 `gin-contrib/sessions` 库，底层为自定义 KV 存储（[sessionstore.go](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/sessionstore/sessionstore.go)）：

```go
Store = sessionstore.NewStore(dep.KV(), []byte(sessionSecret))
// Cookie 配置：
// - Name: cloudreve-session
// - HttpOnly: true
// - MaxAge: 60 * 86400 (60天)
// - SameSite: configurable (default/none/strict/lax)
// - Secure: 跟随 CORS 配置
```

**用途**：主要用于 CSRF 防护（`CSRFInit` / `CSRFCheck`），**不用于用户身份识别**。详见第十章分析。

#### 阶段三：CurrentUser — JWT Token 鉴权（[auth.go#L48-L71](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/middleware/auth.go#L48-L71)）

这是核心的用户身份注入中间件：

```go
func CurrentUser() gin.HandlerFunc {
    return func(c *gin.Context) {
        dep := dependency.FromContext(c)
        // 步骤1：JWT 验证，提取 UID
        shouldContinue, err := dep.TokenAuth().VerifyAndRetrieveUser(c)
        // 步骤2：从 context 中取出 UID
        uid := inventory.UserIDFromContext(c)
        // 步骤3：根据 UID 查询用户完整信息，注入 UserCtx
        if err := SetUserCtx(c, uid); err != nil { ... }
        c.Next()
    }
}
```

**用户上下文键定义**（[user.go#L35-L37](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/inventory/user.go#L35-L37)）：

```go
type (
    UserCtx   struct{}  // 存储 *ent.User（完整用户对象）
    UserIDCtx struct{}  // 存储 int（用户ID，JWT 解析后先注入这个）
)
```

**注入机制** — 使用 `util.WithValue`（[common.go#L251-L253](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/util/common.go#L251-L253)）：

```go
func WithValue(c *gin.Context, key any, value any) {
    c.Request = c.Request.WithContext(context.WithValue(c.Request.Context(), key, value))
}
```

> **注意**：值被注入到 `c.Request.Context()`，而非 gin 的内部 map。这意味着跨中间件传递必须通过 `c.Request.Context()` 读取。

**两个上下文读取函数**（[user.go#L413-L430](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/inventory/user.go#L413-L430)）：

```go
// 从 context 中读取 *ent.User
func UserFromContext(ctx context.Context) *ent.User {
    u, _ := ctx.Value(UserCtx{}).(*ent.User)
    return u
}

// 从 context 中读取 UID：优先 UserIDCtx，其次从 UserCtx 中读取
func UserIDFromContext(ctx context.Context) int {
    uid, ok := ctx.Value(UserIDCtx{}).(int)
    if !ok {
        // 退化：如果 UserIDCtx 不存在，则从 UserCtx 中的用户对象获取
        u := UserFromContext(ctx)
        if u != nil {
            uid = u.ID
        }
    }
    return uid
}
```

---

## 三、JWT Token 鉴权机制详解

### 3.1 Token 结构

Token 采用双 Token 模式（Access + Refresh），定义在 [jwt.go#L45-L53](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/auth/jwt.go#L45-L53)：

```go
type Token struct {
    AccessToken    string    `json:"access_token"`   // 短期访问令牌
    RefreshToken   string    `json:"refresh_token"`  // 长期刷新令牌
    AccessExpires  time.Time `json:"access_expires"`
    RefreshExpires time.Time `json:"refresh_expires"`
    UID            int       `json:"-"`
}
```

### 3.2 Claims 结构

JWT Payload 采用自定义 Claims（[jwt.go#L75-L82](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/auth/jwt.go#L75-L82)）：

```go
type Claims struct {
    TokenType TokenType `json:"token_type"`  // "access" 或 "refresh"
    jwt.RegisteredClaims                      // 标准字段 (sub, nbf, exp)
    StateHash   []byte     `json:"state_hash,omitempty"`   // 用户状态哈希（仅 refresh token）
    RootTokenID *uuid.UUID `json:"root_token_id,omitempty"` // 根令牌ID（用于撤销）
    Scopes      []string   `json:"scopes,omitempty"`       // OAuth 权限范围
    ClientID    string     `json:"client_id,omitempty"`     // OAuth 客户端ID
}
```

### 3.3 签发流程 — `Issue()`（[jwt.go#L236-L291](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/auth/jwt.go#L236-L291)）

```
用户登录成功
    │
    ▼
生成 RootTokenID (UUID v4，若未提供)
    │
    ├─► Access Token:
    │     • sub = hashid.EncodeUserID(uid)   // 用户ID 经过混淆编码
    │     • token_type = "access"
    │     • nbf = 当前时间
    │     • exp = 当前时间 + AccessTokenTTL
    │     • scopes, client_id (OAuth 场景)
    │     • HS256(secret) 签名
    │
    └─► Refresh Token:
          • sub = hashid.EncodeUserID(uid)
          • token_type = "refresh"
          • root_token_id = RootTokenID
          • state_hash = SHA256(email/password/siteID)   // 密码变更检测
          • nbf, exp
          • scopes, client_id
          • HS256(secret) 签名
```

**安全设计**：
- **用户状态哈希**（[jwt.go#L295-L297](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/auth/jwt.go#L295-L297)）：`sha256.Sum256(email + "/" + password + "/" + siteID)`，用户修改密码后，旧 refresh token 自动失效。
- **RootTokenID 撤销机制**：在 KV 存储中设置 `jwt_revoke_{RootTokenID}` 标记，Refresh 时检查是否存在。
- **Subject 编码**：UID 不直接暴露，通过 `hashid` 混淆编码。

### 3.4 验证流程 — `VerifyAndRetrieveUser()`（[jwt.go#L197-L234](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/auth/jwt.go#L197-L234)）

```go
func (t *tokenAuth) VerifyAndRetrieveUser(c *gin.Context) (bool, error) {
    headerVal := c.GetHeader(AuthorizationHeader)

    // 关键分支1：如果是 HMAC 鉴权头（Bearer Cr ...），跳过 JWT 验证
    if strings.HasPrefix(headerVal, TokenHeaderPrefixCr) {
        return false, nil  // 返回 false 表示交给其他鉴权方式，不注入 UserIDCtx
    }

    // 关键分支2：去掉 "Bearer " 后为空（即空 Authorization 或无 Header）
    tokenString := strings.TrimPrefix(headerVal, TokenHeaderPrefix)
    if tokenString == "" {
        return true, nil  // 返回 true，Continue。不注入 UserIDCtx → uid=0
    }

    // 解析并验证 JWT
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, ...)
    claims, ok := token.Claims.(*Claims)

    // 必须是 Access Token
    if !ok || claims.TokenType != TokenTypeAccess {
        return false, serializer.NewError(...)
    }

    // 解码 UID
    uid, err := t.idEncoder.Decode(claims.Subject, hashid.UserID)

    // 注入 UserIDCtx
    util.WithValue(c, inventory.UserIDCtx{}, uid)

    // 如果是 OAuth 客户端令牌，注入 Scope
    if claims.ClientID != "" {
        util.WithValue(c, ScopeContextKey{}, claims.Scopes)
    }

    return false, nil
}
```

**鉴权优先级设计**（决定了 UID 是否为 0）：

| Authorization Header 情况 | 处理方式 | 返回值 | UserIDCtx 注入 |
|--------------------------|---------|-------|----------------|
| **`Bearer Cr <sign>`**   | 跳过 JWT，交给 HMAC 后续处理 | `(false, nil)` | ❌ 不注入（uid=0） |
| **`Bearer <jwt>`**       | 解析 JWT，提取 UID | `(false, nil)` | ✅ 注入有效 UID |
| **空字符串或无 Header**  | 回退到后续流程 | `(true, nil)` | ❌ 不注入（uid=0） |
| **`Bearer 无效token`**   | 错误 | `(false, error)` | ❌ 请求被中止 |

**关键结论**：空 Authorization 请求经过此函数后，**UserIDCtx 不会被注入任何值**，意味着后续 `UserIDFromContext(c)` 将返回默认值 **0**。

### 3.5 刷新流程 — `Refresh()`（[jwt.go#L125-L195](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/auth/jwt.go#L125-L195)）

```
客户端提交 Refresh Token
    │
    ▼
1. 解析 JWT，验证 token_type == "refresh"
    │
    ▼
2. 解码 sub 得到 UID，查询用户
    │
    ▼
3. 验证 StateHash：
   hashUserState(user) 是否与 claims.StateHash 一致
   → 不一致说明用户已修改密码，拒绝刷新
    │
    ▼
4. 检查 RootTokenID 是否已撤销：
   kv.Get("jwt_revoke_" + RootTokenID) 存在则拒绝
    │
    ▼
5. OAuth 场景额外检查：
   - 客户端是否仍有效
   - scopes 是否为授权范围的子集
   - 更新 grant last_used_at
    │
    ▼
6. 签发新 Token 对（保留原有 RootTokenID）
```

### 3.6 Scope 权限检查机制

定义在 [types.go#L395-L416](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/inventory/types/types.go#L395-L416)：

```go
const (
    ScopeUserInfoRead          = "UserInfo.Read"
    ScopeUserInfoWrite         = "UserInfo.Write"
    ScopeFilesRead             = "Files.Read"
    ScopeFilesWrite            = "Files.Write"
    ScopeAdminRead             = "Admin.Read"
    ScopeAdminWrite            = "Admin.Write"
    ScopeWorkflowRead          = "Workflow.Read"
    ScopeWorkflowWrite         = "Workflow.Write"
    ScopeSharesRead            = "Shares.Read"
    ScopeSharesWrite           = "Shares.Write"
    ScopeDavAccountRead        = "DavAccount.Read"
    ScopeDavAccountWrite       = "DavAccount.Write"
    ScopeOfflineAccess         = "offline_access"
    // ...
)
```

检查逻辑在 [jwt.go#L322-L347](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/auth/jwt.go#L322-L347)：

```go
func CheckScope(c *gin.Context, requiredScopes ...string) error {
    hasScopes, tokenScopes := GetScopesFromContext(c)
    if !hasScopes {
        // 没有 scope 信息 → Session 登录或非 OAuth 场景 → 放行
        return nil
    }

    // 写权限隐式包含读权限："File.Write" 自动包含 "File.Read"
    scopeSet := make(map[string]struct{})
    for _, scope := range tokenScopes {
        scopeSet[scope] = struct{}{}
        if resource, ok := extractWriteResource(scope); ok {
            scopeSet[resource+".Read"] = struct{}{}
        }
    }

    // 检查所有必需 scope
    for _, required := range requiredScopes {
        if _, ok := scopeSet[required]; !ok {
            return ErrInsufficientScope
        }
    }
    return nil
}
```

**关键特性**：
- Session 登录（浏览器用户）无 Scope 限制，`hasScopes == false` 直接通过
- OAuth 客户端必须携带 Scope，且写权限隐式授予读权限

---

## 四、HMAC 签名鉴权机制详解

### 4.1 核心数据结构

定义于 [hmac.go#L16-L19](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/auth/hmac.go#L16-L19) 和 [auth.go#L34-L42](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/auth/auth.go#L34-L42)：

```go
type HMACAuth struct {
    SecretKey []byte
}

type Auth interface {
    Sign(body string, expires int64) string
    Check(body string, sign string) error
}
```

### 4.2 签名算法 — `Sign()`（[hmac.go#L23-L32](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/auth/hmac.go#L23-L32)）

```go
func (auth HMACAuth) Sign(body string, expires int64) string {
    h := hmac.New(sha256.New, auth.SecretKey)
    expireTimeStamp := strconv.FormatInt(expires, 10)
    io.WriteString(h, body+":"+expireTimeStamp)
    // 输出格式：base64url(HMAC-SHA256(body:expires)):expires
    return base64.URLEncoding.EncodeToString(h.Sum(nil)) + ":" + expireTimeStamp
}
```

**签名格式**：`<Base64URL(HMAC-SHA256(body:timestamp))>:<unix_timestamp>`

### 4.3 验证算法 — `Check()`（[hmac.go#L35-L57](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/auth/hmac.go#L35-L57)）

```go
func (auth HMACAuth) Check(body string, sign string) error {
    signSlice := strings.Split(sign, ":")
    expires, err := strconv.ParseInt(signSlice[len(signSlice)-1], 10, 64)

    // 过期检查（expires == 0 表示永不过期）
    if expires < time.Now().Unix() && expires != 0 {
        return ErrExpired
    }

    // 恒时比较，防止时序攻击
    if subtle.ConstantTimeCompare([]byte(auth.Sign(body, expires)), []byte(sign)) != 1 {
        return ErrInvalidSign
    }
    return nil
}
```

**安全设计**：使用 `crypto/subtle.ConstantTimeCompare` 进行恒时比较，防止基于响应时间的时序攻击。

### 4.4 两种签名场景

#### 场景 A：请求 Header 签名（主从 RPC、回调等）

**签名** — `SignRequest()`（[auth.go#L46-L59](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/auth/auth.go#L46-L59)）：

```
Header: Authorization: Bearer Cr <signature>

签名内容 body 组成（[auth.go#L98-L122](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/auth/auth.go#L98-L122)）：
    1. URL Path (getUrlSignContent)
    2. 所有以 "X-Cr-" 开头的 Header（按字母排序，格式 "Key=Value&..."）
       排除 "X-Cr-Filename"
    3. 请求 Body（除从机上传数据接口外）

    通过 serializer.NewRequestSignString() 拼接为最终签名串
```

**验证** — `SignRequired` 中间件（[auth.go#L27-L45](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/middleware/auth.go#L27-L45)）：

```go
func SignRequired(authInstance auth.Auth) gin.HandlerFunc {
    return func(c *gin.Context) {
        var err error
        switch c.Request.Method {
        case http.MethodPut, http.MethodPost, http.MethodPatch:
            err = auth.CheckRequest(c, authInstance, c.Request) // Header 签名
        default:
            err = auth.CheckURI(c, authInstance, c.Request.URL)  // URL Query 签名
        }
        if err != nil {
            c.JSON(200, serializer.ErrWithDetails(c, serializer.CodeCredentialInvalid, ...))
            c.Abort()
            return
        }
        c.Next()
    }
}
```

#### 场景 B：URL Query 签名（文件下载链接）

**签名** — `SignURI()`（[auth.go#L125-L146](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/auth/auth.go#L125-L146)）：

```
URL: /api/v3/file/content/xxx?sign=<signature>

签名内容：仅 URL Path
```

**验证** — `CheckURI()`（[auth.go#L173-L181](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/auth/auth.go#L173-L181)）：

```go
func CheckURI(ctx context.Context, instance Auth, url *url.URL) error {
    queries := url.Query()
    sign := queries.Get("sign")
    queries.Del("sign")               // 移除 sign 参数后再计算
    url.RawQuery = queries.Encode()
    return instance.Check(getUrlSignContent(ctx, url), sign)
}
```

### 4.5 HMAC 的应用场景

| 场景 | 签名位置 | 使用的 Secret | 代码位置 |
|-----|---------|-------------|---------|
| Slave 节点 API 鉴权 | Header `Authorization: Bearer Cr` | Node.SlaveKey | [router.go#L115](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/routers/router.go#L115) |
| Master 接收 Slave RPC | Header `Authorization: Bearer Cr` | Node.SlaveKey | [cluster.go#L50-L75](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/middleware/cluster.go#L50-L75) |
| 文件下载 URL | URL Query `?sign=` | 系统全局 GeneralAuth | [router.go#L648-L659](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/routers/router.go#L648-L659) |
| 用户激活链接 | URL Query `?sign=` | 系统全局 GeneralAuth | [router.go#L400-L403](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/routers/router.go#L400-L403) |
| 上传回调验证 | Header `Authorization: Bearer Cr` | Node.SlaveKey | [auth.go#L220-L240](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/middleware/auth.go#L220-L240) |

---

## 五、WebDAV Basic Auth

实现于 [auth.go#L103-L175](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/middleware/auth.go#L103-L175)：

```
请求携带 Basic Auth Header
    │
    ▼
1. 解析 username/password (RFC 7617)
    │
    ▼
2. 查询用户：userClient.GetActiveByDavAccount(username, password)
   - DavAccount 为独立的 WebDAV 专用密码表
   - 不是用户登录密码
    │
    ▼
3. 检查用户组是否启用 WebDAV 权限
    │
    ▼
4. 检查 DavAccount 是否为只读：
   - 只读账号禁止 DELETE/PUT/MKCOL/COPY/MOVE/LOCK/UNLOCK
    │
    ▼
5. SetUserCtxByUser(c, expectedUser) 注入用户上下文
```

---

## 六、Slave 模式中间件链（对比）

Slave 模式下没有 JWT/Session，完全依赖 HMAC（参见 [router.go#L108-L178](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/routers/router.go#L108-L178)）：

```
HTTP Request
    │
    ▼
gin.Recovery
    │
    ▼
InitializeHandling(dep)
    │
    ▼
InitializeHandlingSlave()       ◄── 注入 MasterSiteID/MasterSiteUrl/MasterSiteVersion
    │
    ▼
Logging
    │
    ▼
CORS (可选)
    │
    ▼
SignRequired(dep.GeneralAuth()) ◄── 全局 HMAC 鉴权，所有接口必须签名
    │
    ▼
CacheControl
    │
    ▼
  业务路由
```

---

## 七、上传回调的完整鉴权链 — HMAC 验签与用户上下文注入

这是最复杂的一条鉴权链。回调请求来自第三方存储服务商（阿里云OSS、七牛、又拍云等）或从机节点，既不是浏览器也不是 API 客户端，因此**不走 JWT 也不走 Session**，而是通过 **UploadSession 的 UID 字段** 间接还原用户身份。

### 7.1 上传回调路由全景（[router.go#L472-L546](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/routers/router.go#L472-L546)）

| 存储类型 | 路由 | 中间件链 |
|---------|------|---------|
| Remote 远程/从机 | `POST callback/remote/:sessionID/:key` | UseUploadSession → RemoteCallbackAuth → ProcessCallback |
| 阿里云 OSS | `POST callback/oss/:sessionID/:key` | UseUploadSession → OSSCallbackAuth → OSSCallbackValidate → ProcessCallback |
| 七牛 Qiniu | `POST callback/qiniu/:sessionID/:key` | UseUploadSession → QiniuCallbackValidate → ProcessCallback |
| 又拍云 Upyun | `POST callback/upyun/:sessionID/:key` | UseUploadSession → UpyunCallbackAuth → ProcessCallback |
| OneDrive | `POST callback/onedrive/:sessionID/:key` | UseUploadSession → ProcessCallback |
| COS/S3/KS3/OBS | `GET callback/{cos,s3,ks3,obs}/:sessionID/:key` | UseUploadSession → ProcessCallback |

所有回调路由共享一个核心中间件：`UseUploadSession`。

### 7.2 UploadSession 数据结构（[fs.go#L263-L281](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/filemanager/fs/fs.go#L263-L281)）

```go
type UploadSession struct {
    UID             int                 // ★ 关键：发起上传的用户 ID
    Policy          *ent.StoragePolicy  // 存储策略（包含 SlaveKey 等）
    FileID          int                 // 占位文件 ID
    EntityID        int                 // 实体 ID
    Callback        string              // 回调 URL
    CallbackSecret  string              // 回调密钥（32位随机字符串）
    UploadID        string              // 分片上传 ID
    UploadURL       string
    Credential      string
    ChunkSize       int64
    SentinelTaskID  int
    NewFileCreated  bool
    Importing       bool
    EncryptMetadata *types.EncryptMetadata
    LockToken       string              // 文件锁令牌
    Props           *UploadProps        // 上传属性（路径、大小、过期时间等）
}
```

**创建位置**：用户在前端发起上传创建会话时，在 `DBFS.PrepareUpload()` 中生成（[dbfs/upload.go#L232-L252](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/filemanager/fs/dbfs/upload.go#L232-L252)）：

```go
session := &fs.UploadSession{
    Props:            &fs.UploadProps{...},
    FileID:           fileId,
    NewFileCreated:   !fileExisted,
    Importing:        req.ImportFrom != nil,
    EntityID:         entityId,
    UID:              f.user.ID,    // ★ 从当前已登录用户 f.user 获取 UID
    Policy:           policy,
    CallbackSecret:   util.RandStringRunesCrypto(32),
    LockToken:        lockToken,
}
```

**存储位置**：`manager.CreateUploadSession()` 将 UploadSession 序列化存入 KV（[manager.go#L28-L31](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/filemanager/manager/manager.go#L28-L31) 和 [upload.go#L136-L144](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/filemanager/manager/upload.go#L136-L144)）：

```go
const (
    UploadSessionCachePrefix = "callback_"   // KV Key 前缀
    UploadSessionCtx         = "uploadSession" // gin Context Key
)

// 存入 KV：key = "callback_" + sessionID，value = UploadSession 对象
err = m.kv.Set(
    UploadSessionCachePrefix + req.Props.UploadSessionID,
    *uploadSession,                                           // 注意值拷贝
    max(1, int(req.Props.ExpireAt.Sub(time.Now()).Seconds())), // TTL = 会话剩余时间
)
```

**关键点**：UploadSession 中 `UID` 字段保留了上传发起者的用户 ID，使得回调请求即使没有 JWT 也能通过 sessionID 还原用户。

### 7.3 UseUploadSession — 还原 UploadSession 并注入用户上下文（[auth.go#L178-L217](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/middleware/auth.go#L178-L217)）

这是回调链的第一个中间件，承担两个核心职责：
1. 从 KV 还原 UploadSession，放入 gin Context
2. 用 UploadSession.UID 注入用户上下文（UserCtx）

```go
func UseUploadSession(policyType types.PolicyType) gin.HandlerFunc {
    return func(c *gin.Context) {
        err := uploadCallbackCheck(c, policyType)
        if err != nil {
            c.JSON(CallbackFailedStatusCode, serializer.Err(c, err))
            c.Abort()
            return
        }
        c.Next()
    }
}

func uploadCallbackCheck(c *gin.Context, policyType types.PolicyType) error {
    // 步骤1：从 URL 路径获取 sessionID 参数
    sessionID := c.Param("sessionID")
    if sessionID == "" {
        return serializer.NewError(serializer.CodeParamErr, "Session ID cannot be empty", nil)
    }

    // 步骤2：从 KV 存储读取 UploadSession
    // key = "callback_" + sessionID
    dep := dependency.FromContext(c)
    callbackSessionRaw, exist := dep.KV().Get(manager.UploadSessionCachePrefix + sessionID)
    if !exist {
        return serializer.NewError(serializer.CodeUploadSessionExpired,
            "Upload session does not exist or expired", nil)
    }

    // 步骤3：类型断言，放入 gin Context（通过 c.Set，不是 request context）
    callbackSession := callbackSessionRaw.(fs.UploadSession)
    c.Set(manager.UploadSessionCtx, &callbackSession)  // gin.Context map

    // 步骤4：校验存储策略类型是否匹配（防止跨策略攻击）
    if callbackSession.Policy.Type != string(policyType) {
        return serializer.NewError(serializer.CodePolicyNotAllowed, "", nil)
    }

    // 步骤5：★★★ 注入用户上下文 ★★★
    // 使用 UploadSession 中保存的 UID 来还原当前用户
    // SetUserCtx 会：
    //   - 如果 UID > 0：从 DB 查用户，调用 util.WithValue 注入 UserCtx
    //   - 如果 UID = 0：构造 AnonymousUser
    if err := SetUserCtx(c, callbackSession.UID); err != nil {
        return err
    }

    return nil
}
```

**SetUserCtx 流程**（[auth.go#L74-L84](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/middleware/auth.go#L74-L84)）：

```go
func SetUserCtx(c *gin.Context, uid int) error {
    dep := dependency.FromContext(c)
    userClient := dep.UserClient()
    // GetLoginUserByID：uid>0 查DB，uid=0 构造匿名用户
    loginUser, err := userClient.GetLoginUserByID(c, uid)
    if err != nil {
        return serializer.NewError(serializer.CodeDBError, "failed to get login user", err)
    }
    // util.WithValue 写入 c.Request.Context() 的 UserCtx
    SetUserCtxByUser(c, loginUser)
    return nil
}
```

### 7.4 RemoteCallbackAuth — HMAC 验签（[auth.go#L219-L240](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/middleware/auth.go#L219-L240)）

这个中间件在 UseUploadSession **之后** 执行，因为它需要从 gin Context 中读取 UploadSession.Policy.Edges.Node.SlaveKey 作为验签密钥：

```go
func RemoteCallbackAuth() gin.HandlerFunc {
    return func(c *gin.Context) {
        // 必须先通过 UseUploadSession 才能拿到 session
        session := c.MustGet(manager.UploadSessionCtx).(*fs.UploadSession)

        // 检查存储策略是否绑定了节点（Node.SlaveKey 就是 HMAC 密钥）
        if session.Policy.Edges.Node == nil {
            c.JSON(CallbackFailedStatusCode,
                serializer.ErrWithDetails(c, serializer.CodeCredentialInvalid, "Node not found", nil))
            c.Abort()
            return
        }

        // 用节点的 SlaveKey 创建 HMACAuth 实例
        authInstance := auth.HMACAuth{SecretKey: []byte(session.Policy.Edges.Node.SlaveKey)}

        // 调用 CheckRequest 验证 Authorization: Bearer Cr <sign>
        if err := auth.CheckRequest(c, authInstance, c.Request); err != nil {
            c.JSON(CallbackFailedStatusCode,
                serializer.ErrWithDetails(c, serializer.CodeCredentialInvalid, err.Error(), err))
            c.Abort()
            return
        }

        c.Next()
    }
}
```

### 7.5 ProcessCallback — 最终业务处理（[callback/upload.go#L44-L60](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/service/callback/upload.go#L44-L60)）

```go
func ProcessCallback(c *gin.Context) error {
    dep := dependency.FromContext(c)
    // ★ 从 request context 获取当前用户（已由 UseUploadSession 注入）
    user := inventory.UserFromContext(c)

    // 创建文件管理器（关联到当前用户）
    m := manager.NewFileManager(dep, user)
    defer m.Recycle()

    // 从 gin Context 获取 UploadSession（由 UseUploadSession 放入）
    uploadSession := c.MustGet(manager.UploadSessionCtx).(*fs.UploadSession)

    // 使用用户身份完成上传（写入DB、扣减容量、触发工作流等）
    _, err := m.CompleteUpload(c, uploadSession)
    if err != nil {
        return fmt.Errorf("failed to complete upload: %w", err)
    }
    return nil
}
```

### 7.6 上传回调完整时序图

以 `POST /api/v4/callback/remote/:sessionID/:key` 为例：

```
第三方存储 / Slave 节点
    │
    │  POST callback/remote/session-abc-123/callback-key
    │  Authorization: Bearer Cr <HMAC-SHA256签名>
    ▼
┌──────────────────────────────────────────────────────────────┐
│ Master 全局中间件（已按顺序执行完毕）                          │
│   InitializeHandling → Session → CurrentUser → CacheControl  │
│                                                               │
│   注意：CurrentUser 时因 Authorization 是 "Bearer Cr..."      │
│         VerifyAndRetrieveUser 返回 (false, nil)               │
│         → 未注入 UserIDCtx → uid=0 → 注入 AnonymousUser      │
│         → UserCtx 此时指向匿名用户（ID=0）                     │
└──────────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────┐
│ 1. UseUploadSession(PolicyTypeRemote)                         │
│    [auth.go#L178-L217]                                        │
│                                                               │
│  ① 读取 c.Param("sessionID") = "session-abc-123"             │
│  ② KV.Get("callback_session-abc-123") → fs.UploadSession     │
│  ③ c.Set("uploadSession", &uploadSession)  ← gin Context     │
│  ④ 检查 Policy.Type == "remote" ?                            │
│  ⑤ ★ SetUserCtx(c, uploadSession.UID)                        │
│     → uid = 12345 (上传发起者)                                │
│     → GetLoginUserByID(12345) → 查询DB得到 *ent.User         │
│     → util.WithValue(c, UserCtx{}, user)                     │
│     → ★ 覆盖了之前的匿名用户！UserCtx 现在是真实用户         │
└──────────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────┐
│ 2. RemoteCallbackAuth()                                       │
│    [auth.go#L219-L240]                                        │
│                                                               │
│  ① c.MustGet("uploadSession") → UploadSession                │
│  ② session.Policy.Edges.Node.SlaveKey → "my-secret-key"      │
│  ③ auth.HMACAuth{SecretKey: []byte("my-secret-key")}         │
│  ④ auth.CheckRequest(c, authInstance, c.Request)             │
│     - 读取 Header "Authorization: Bearer Cr <sign>"          │
│     - 组装签名内容: Path + X-Cr-* Headers + Body             │
│     - HMAC-SHA256 + 恒时比较 + 过期检查                       │
│  ⑤ 签名通过 → c.Next()                                        │
└──────────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────┐
│ 3. ProcessCallback()                                          │
│    [callback/upload.go#L44-L60]                               │
│                                                               │
│  ① inventory.UserFromContext(c)                               │
│     → ctx.Value(UserCtx{}) → 真实用户 (ID=12345)              │
│  ② manager.NewFileManager(dep, user)  ← 以真实用户操作       │
│  ③ c.MustGet("uploadSession") → UploadSession                │
│  ④ m.CompleteUpload(c, uploadSession)                        │
│     - 更新文件实体状态                                        │
│     - 扣减用户存储容量                                        │
│     - 触发工作流（病毒扫描、缩略图生成等）                    │
│     - 删除 KV 中的 UploadSession                              │
└──────────────────────────────────────────────────────────────┘
    │
    ▼
  200 OK
```

### 7.7 两次用户上下文注入的关键区别

用户上下文在整个回调请求中被注入**两次**：

| 注入时机 | 注入位置 | UID | 用户对象 |
|---------|---------|-----|---------|
| 第一次 | 全局中间件 `CurrentUser` | **0** | AnonymousUser（匿名用户） |
| 第二次 | 路由级中间件 `UseUploadSession` | **UploadSession.UID** | 真实上传发起用户 |

**为什么第二次覆盖是正确的？**
- 第一次：在 `CurrentUser` 中执行，因 `Authorization: Bearer Cr ...` 不是 JWT，按"未携带身份凭证"处理，得到匿名用户
- 第二次：在 `UseUploadSession` 中执行，读取 KV 中的 UploadSession 得到原始 UID，**这是可信的**（因为 UploadSession 只在 Master 侧创建，UID 从已登录用户 `f.user.ID` 写入）
- 覆盖实现：`util.WithValue` 本质是 `context.WithValue`，基于不可变 context，每次调用生成新的 context 包装旧值，读取时从外向内查找，因此外层的新值会遮蔽旧值

### 7.8 上传回调中的两种 Context 存储对比

| 数据 | 存储方式 | Key | 读取方式 |
|-----|---------|-----|---------|
| **UploadSession** | `c.Set()` (gin map) | `manager.UploadSessionCtx = "uploadSession"` | `c.MustGet("uploadSession")` |
| **当前用户 UserCtx** | `c.Request.Context()` (标准 context) | `inventory.UserCtx{}` (struct{}) | `inventory.UserFromContext(c)` |
| **当前用户 UserIDCtx** | `c.Request.Context()` (标准 context) | `inventory.UserIDCtx{}` (struct{}) | `inventory.UserIDFromContext(c)` |

**设计意图**：
- `UploadSession` 仅在回调链的几个中间件内使用，通过 gin Context 快速存取即可
- `UserCtx` 可能被业务深层代码通过 `context.Context` 传递（如 DB 查询、任务队列等），必须注入标准 context

---

## 八、回调地址中的 key/secret 鉴权机制深度解析

这是之前文档未覆盖的关键环节。回调 URL 格式为 `callback/{policyType}/{sessionID}/{key}`，其中 `{key}` 就是 `CallbackSecret`，但它在不同存储类型的回调链路中的作用和校验方式截然不同。

### 8.1 CallbackSecret 的生成

CallbackSecret 是在 Master 侧创建 UploadSession 时生成的随机字符串，位于 [dbfs/upload.go#L250](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/filemanager/fs/dbfs/upload.go#L250)：

```go
session := &fs.UploadSession{
    // ... 其他字段 ...
    UID:              f.user.ID,
    Policy:           policy,
    CallbackSecret:   util.RandStringRunesCrypto(32),  // ★ 32位加密安全随机字符串
    LockToken:        lockToken,
}
```

**生成函数** `util.RandStringRunesCrypto` 使用 `crypto/rand` 标准库，确保不可预测性。

### 8.2 CallbackSecret 纳入回调 URL

回调 URL 通过 `MasterSlaveCallbackUrl()` 生成，位于 [routes.go#L46-L49](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/cluster/routes/routes.go#L46-L49)：

```go
func MasterSlaveCallbackUrl(base *url.URL, driver, id, secret string) *url.URL {
    // 路径拼接：/api/v4/callback/{driver}/{sessionID}/{CallbackSecret}
    apiBaseURI, _ := url.Parse(path.Join(constants.APIPrefix+"/callback", driver, id, secret))
    return base.ResolveReference(apiBaseURI)
}
```

**生成时机**：在各存储驱动的 `Token()` 方法中调用：

| 存储类型 | 生成位置 | 生成代码 |
|---------|---------|---------|
| Remote/从机 | [remote.go#L142](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/filemanager/driver/remote/remote.go#L142) | `routes.MasterSlaveCallbackUrl(siteURL, PolicyTypeRemote, uploadSessionID, CallbackSecret)` |
| OSS | [oss.go#L265](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/filemanager/driver/oss/oss.go#L265) | `routes.MasterSlaveCallbackUrl(siteURL, PolicyTypeOss, uploadSessionID, CallbackSecret)` |
| Qiniu | [qiniu.go#L366](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/filemanager/driver/qiniu/qiniu.go#L366) | `routes.MasterSlaveCallbackUrl(siteURL, PolicyTypeQiniu, uploadSessionID, CallbackSecret)` |
| Upyun | [upyun.go#L287](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/filemanager/driver/upyun/upyun.go#L287) | `routes.MasterSlaveCallbackUrl(siteURL, PolicyTypeUpyun, uploadSessionID, CallbackSecret)` |
| S3 | [s3.go#L345](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/filemanager/driver/s3/s3.go#L345) | `routes.MasterSlaveCallbackUrl(siteURL, PolicyTypeS3, uploadSessionID, CallbackSecret)` |
| OneDrive/COS/KS3/OBS | 各驱动 Token() 方法 | 类似模式 |

生成的 URL 被存入 `uploadSession.Callback` 字段，然后：
- **Remote 场景**：通过 RPC 传给 Slave 节点，Slave 在上传完成后发起回调
- **第三方存储场景（OSS/Qiniu/Upyun 等）**：作为回调 URL 配置给第三方存储服务商，由服务商在上传完成后发起回调

### 8.3 关键发现：`uploadCallbackCheck` 不校验 URL 中的 `:key`

位于 [middleware/auth.go#L192-L217](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/middleware/auth.go#L192-L217) 的核心校验函数：

```go
func uploadCallbackCheck(c *gin.Context, policyType types.PolicyType) error {
    // 步骤1：只读取 sessionID，完全不读取 :key
    sessionID := c.Param("sessionID")   // ✅ 读取
    // ❗ c.Param("key") 从未被读取 ❗

    // 步骤2：只通过 sessionID 从 KV 读取 UploadSession
    callbackSessionRaw, exist := dep.KV().Get("callback_" + sessionID)
    if !exist {
        return serializer.NewError(serializer.CodeUploadSessionExpired, ...)
    }

    // 步骤3：类型断言 + 策略类型校验
    callbackSession := callbackSessionRaw.(fs.UploadSession)
    if callbackSession.Policy.Type != string(policyType) {
        return serializer.NewError(serializer.CodePolicyNotAllowed, "", nil)
    }

    // 步骤4：注入用户上下文
    if err := SetUserCtx(c, callbackSession.UID); err != nil {
        return err
    }

    return nil
}
```

**惊人的事实**：URL 路径中的 `:key`（即 `CallbackSecret`）在 `UseUploadSession` 中**从未被读取，也从未与 `UploadSession.CallbackSecret` 进行比较**。这意味着：

1. 攻击者如果能猜到 `sessionID`（UUID v4，实际上不可猜），则 URL 中的 `:key` 可以是任意值
2. CallbackSecret 的安全作用**不在显式校验**，而在**其他机制**中

### 8.4 不同存储类型回调链路中 key/secret 的校验方式对比

回调 URL 格式统一为 `callback/{policyType}/{sessionID}/{key}`，但 7 种存储类型的校验强度差异巨大：

| 存储类型 | 中间件链 | key/secret 的作用 | 校验方式 | 安全强度 |
|---------|---------|-----------------|---------|---------|
| **Remote/从机** | `UseUploadSession` → `RemoteCallbackAuth` → `ProcessCallback` | **间接纳入 HMAC 签名** | ✅ URL Path（包含 CallbackSecret）作为 HMAC 签名的一部分；HMAC 验签通过即证明 URL 未被篡改 | 🔒🔒🔒🔒🔒 最高 |
| **OSS 阿里云** | `UseUploadSession` → `OSSCallbackAuth` → `OSSCallbackValidate` → `ProcessCallback` | **间接纳入阿里云回调签名** | ✅ 阿里云 SDK 验签 `oss.VerifyCallbackSignature`，签名包含 URL Path（含 CallbackSecret）；额外校验上传文件大小 | 🔒🔒🔒🔒 高 |
| **Qiniu 七牛** | `UseUploadSession` → `QiniuCallbackValidate` → `ProcessCallback` | **间接纳入七牛回调签名** | ✅ 七牛 SDK 验签 `mac.VerifyCallback`，签名包含 URL Path（含 CallbackSecret） | 🔒🔒🔒🔒 高 |
| **Upyun 又拍云** | `UseUploadSession` → `UpyunCallbackAuth` → `ProcessCallback` | **间接纳入又拍云回调签名** | ✅ 又拍云专有算法 `upyun.ValidateCallback`，签名包含 URL Path（含 CallbackSecret）、MD5、Date | 🔒🔒🔒🔒 高 |
| **OneDrive** | `UseUploadSession` → `ProcessCallback` | **仅作为 URL 标识，不校验** | ❌ 无额外签名校验；key 仅作为路径的一部分，不可枚举性提供最低限度防护 | 🔒 最低 |
| **COS/S3/KS3/OBS** | `UseUploadSession` → `ProcessCallback` | **仅作为 URL 标识，不校验** | ❌ 无额外签名校验；key 仅作为路径的一部分 | 🔒 最低 |

### 8.5 Remote 场景：CallbackSecret 间接纳入 HMAC 验签的完整链路

这是最复杂也最安全的场景，CallbackSecret 虽然不被显式比较，但通过 HMAC 签名机制被间接校验：

**阶段一：Master 侧生成回调 URL 和签名密钥**
```
Master 侧 PrepareUpload
    │
    ├─► 生成 CallbackSecret = "a1b2c3..." (32位随机)
    ├─► 生成回调 URL = "https://master.example.com/api/v4/callback/remote/sess-123/a1b2c3..."
    ├─► UploadSession { CallbackSecret: "a1b2c3...", Callback: URL, Policy: {Node: {SlaveKey: "slave-secret-456"}} }
    ├─► 存入 KV: "callback_sess-123" → UploadSession
    └─► 通过 RPC 将 UploadSession 传给 Slave 节点
```

**阶段二：Slave 侧上传完成，发起回调请求**（[local.go#L264-L277](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/filemanager/driver/local/local.go#L264-L277)）

```go
// Slave 侧 local.Driver.CompleteUpload()
func (handler *Driver) CompleteUpload(ctx context.Context, session *fs.UploadSession) error {
    // session.Callback = "https://master.example.com/api/v4/callback/remote/sess-123/a1b2c3..."
    resp := handler.httpClient.Request(
        "POST",
        session.Callback,       // ★ URL 包含 CallbackSecret
        nil,
        request.WithTimeout(...),
        request.WithCredential(
            auth.HMACAuth{[]byte(session.Policy.Edges.Node.SlaveKey)},  // ★ HMAC 密钥 = SlaveKey
            int64(handler.config.Slave().SignatureTTL),
        ),
        // ...
    )
}
```

`request.WithCredential` 最终调用 `auth.SignRequest()` 对请求进行签名（[auth.go#L46-L122](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/auth/auth.go#L46-L122)）：

```go
func SignRequest(ctx context.Context, instance Auth, r *http.Request, expire *time.Time) *http.Request {
    r.Header.Set(AuthorizationHeader, TokenHeaderPrefixCr+
        instance.Sign(
            getUrlSignContent(ctx, r.URL)+   // ★ 1. URL Path: "/api/v4/callback/remote/sess-123/a1b2c3..."
            serializer.NewRequestSignString(r)+ // ★ 2. X-Cr-* Header
            string(body),                        // ★ 3. Body
            expire.Unix(),
        ),
    )
    return r
}
```

**关键**：URL Path 包含 CallbackSecret，因此签名内容 = `"/api/v4/callback/remote/sess-123/a1b2c3..." + Headers + Body`。

**阶段三：Master 侧验签**（[auth.go#L219-L240](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/middleware/auth.go#L219-L240) → [auth.go#L60-L93](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/auth/auth.go#L60-L93)）

```go
// Master 侧 RemoteCallbackAuth()
func RemoteCallbackAuth() gin.HandlerFunc {
    return func(c *gin.Context) {
        session := c.MustGet(manager.UploadSessionCtx).(*fs.UploadSession)
        authInstance := auth.HMACAuth{SecretKey: []byte(session.Policy.Edges.Node.SlaveKey)}

        // auth.CheckRequest 内部：
        // 1. 从请求 Header 取出签名
        // 2. 按相同规则计算本地签名：
        //    body = getUrlSignContent(c, c.Request.URL) +  // "/api/v4/callback/remote/sess-123/a1b2c3..."
        //           X-Cr-* Headers + Body
        // 3. HMACAuth.Check(body, sign) → 恒时比较
        err := auth.CheckRequest(c, authInstance, c.Request)
        // ...
    }
}
```

**校验逻辑链**：
1. Slave 发起的请求 URL 为 `POST /api/v4/callback/remote/sess-123/a1b2c3...`，包含 CallbackSecret
2. Master 侧 HMAC 验签时，会重新计算签名，**签名内容包含完整 URL Path**
3. 如果攻击者篡改了 URL 中的 `:key`（如改成 `x9y8z7...`），则 Master 计算签名时使用的 Path 是 `/api/v4/callback/remote/sess-123/x9y8z7...`
4. 而 Slave 签名时 Path 是 `/api/v4/callback/remote/sess-123/a1b2c3...`
5. 签名内容不同 → HMAC 结果不同 → 验签失败

**结论**：在 Remote 场景中，CallbackSecret **通过 HMAC 签名机制被间接校验**，无需显式比较 `c.Param("key") == session.CallbackSecret`。

### 8.6 第三方存储场景：CallbackSecret 纳入服务商签名机制

OSS、Qiniu、Upyun 等第三方存储的回调签名机制不由 Cloudreve 控制，但原理类似：

```
第三方存储服务商回调时：
    1. 使用 Cloudreve 配置给它的 AccessKey/SecretKey 对回调请求进行签名
    2. 签名内容通常包含：HTTP Method + URL Path + Body + 其他元数据
    3. URL Path 中包含 CallbackSecret
    4. Master 侧使用对应 SDK 的 VerifyCallback 方法验签
    5. 验签通过即证明 URL（含 CallbackSecret）未被篡改
```

**OSS 验签**（[auth.go#L243-L259](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/middleware/auth.go#L243-L259) + [callback.go#L56-L77](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/routers/controllers/callback.go#L56-L77)）：
```go
// OSSCallbackAuth: 阿里云 SDK 验签（含 URL Path）
func OSSCallbackAuth() gin.HandlerFunc {
    return func(c *gin.Context) {
        err := oss.VerifyCallbackSignature(c.Request, dep.KV(), ...)
        // ...
    }
}

// OSSCallbackValidate: 额外校验文件大小（防篡改）
func OSSCallbackValidate(c *gin.Context) {
    uploadSession := c.MustGet(manager.UploadSessionCtx).(*fs.UploadSession)
    if uploadSession.Props.Size != callbackBody.Size {
        // 拒绝
    }
}
```

**Qiniu 验签**（[callback.go#L34-L54](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/routers/controllers/callback.go#L34-L54)）：
```go
func QiniuCallbackValidate(c *gin.Context) {
    session := c.MustGet(manager.UploadSessionCtx).(*fs.UploadSession)
    mac := qbox.NewMac(session.Policy.AccessKey, session.Policy.SecretKey)
    ok, err := mac.VerifyCallback(c.Request)  // 七牛 SDK 内部校验签名（含 URL Path）
    // ...
}
```

**Upyun 验签**（[callback.go#L79-L90](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/routers/controllers/callback.go#L79-L90) + [upyun.go#L356-L384](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/pkg/filemanager/driver/upyun/upyun.go#L356-L384)）：
```go
func UpyunCallbackAuth(c *gin.Context) {
    uploadSession := c.MustGet(manager.UploadSessionCtx).(*fs.UploadSession)
    err := upyun.ValidateCallback(c, uploadSession)
    // 内部校验:
    // 1. MD5(body) == Header["Content-Md5"]
    // 2. 签名 = sign(AccessKey, SecretKey, ["POST", URL.Path, Date, ContentMD5])
    // 3. 签名 == Header["Authorization"]
}
```

### 8.7 OneDrive/COS/S3/KS3/OBS 场景：CallbackSecret 仅作标识

这 5 种存储的回调链路**没有额外签名校验**，中间件链只有：
```
UseUploadSession → ProcessCallback
```

**安全保障**：
- CallbackSecret 是 32 位加密安全随机字符串，作为 URL 路径的一部分，具有不可枚举性
- sessionID 也是 UUID v4，同样不可枚举
- 两者结合形成 64+ 位的随机路径，暴力枚举在计算上不可行
- UploadSession 在 KV 中有 TTL（默认等于上传会话有效期），过期后自动删除

**潜在风险**：
- 如果攻击者通过其他方式获取了完整的回调 URL（如日志泄露），则可以在有效期内伪造回调
- 但即使成功，攻击者也无法控制回调中的文件大小、哈希等元数据（由 Master 在会话创建时确定）

### 8.8 CallbackSecret 还会返回给前端

在 [response.go#L162](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/service/explorer/response.go#L162) 和 [response.go#L179](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/service/explorer/response.go#L179) 中：

```go
type UploadSessionResponse struct {
    // ... 其他字段 ...
    CallbackSecret  string  `json:"callback_secret"`  // ★ 返回给前端
}

func BuildUploadSessionResponse(...) *UploadSessionResponse {
    return &UploadSessionResponse{
        // ...
        CallbackSecret:  session.CallbackSecret,  // 直接透传
    }
}
```

**设计意图**：某些存储的客户端 SDK（如分片上传、断点续传）需要知道完整的回调地址来完成上传流程。CallbackSecret 对前端是可见的，但这不是安全问题，因为：
1. 前端本来就是上传的发起者，知道自己的回调密钥是合理的
2. 回调的真正安全保障来自签名机制（Remote/HMAC、第三方存储签名），而非 CallbackSecret 的保密性

---

## 九、CallbackSecret、HMAC 签名、UploadSession 的安全职责划分

三者形成了**分层防御**的安全架构，各自承担不可替代的职责：

### 9.1 三者的安全职责对比表

| 安全组件 | 核心职责 | 攻击面防护 | 不可伪造性来源 | 被破解后的影响 |
|---------|---------|-----------|---------------|---------------|
| **CallbackSecret** | 1. URL 路径不可枚举<br>2. 间接纳入签名内容（作为 URL Path 的一部分）<br>3. 前端 SDK 构造完整回调地址 | 暴力枚举、路径遍历 | `crypto/rand` 生成 32 位随机字符串 | 攻击者可以构造回调 URL，但需要同时破解签名才能通过验签 |
| **HMAC 签名** | 1. 请求完整性校验（URL、Header、Body 未被篡改）<br>2. 身份认证（请求来自合法的 Slave 节点或第三方存储）<br>3. 时效性校验（签名过期时间） | 篡改请求、重放攻击、伪造请求 | `HMAC-SHA256` + `SlaveKey`（对称密钥）或第三方存储的 SecretKey + 恒时比较 | 攻击者可以篡改任何请求内容（包括 UID、文件大小等），以任意用户身份完成上传 |
| **UploadSession** | 1. 会话状态存储（UID、Policy、文件大小、过期时间）<br>2. 跨请求上下文传递（将上传发起者的身份传递给回调 Handler）<br>3. 绑定签名密钥（Policy.Edges.Node.SlaveKey） | 会话劫持、状态篡改 | Master 侧 KV 存储，服务端独占，不暴露给客户端 | 攻击者可以冒充任何上传会话，以任意用户身份完成任意大小文件的上传 |

### 9.2 分层防御架构图示

```
                                 ┌─────────────────────────────────────────────────┐
                                 │             HTTP Request to Callback            │
                                 │  POST /api/v4/callback/remote/sess-123/a1b2c3   │
                                 │  Authorization: Bearer Cr <hmac_signature>       │
                                 │  Body: { ... }                                   │
                                 └──────────────────────┬──────────────────────────┘
                                                        │
                     ┌──────────────────────────────────┼──────────────────────────────────┐
                     │                                  │                                  │
         ┌───────────▼──────────┐            ┌──────────▼──────────┐            ┌────────▼──────────┐
         │  第一层：URL 可寻址  │            │  第二层：完整性校验  │            │  第三层：状态绑定  │
         │  CallbackSecret      │            │  HMAC 签名          │            │  UploadSession     │
         │                      │            │                      │            │                    │
         │ • 32位加密随机字符串 │            │ • HMAC-SHA256        │            │ • Master 侧 KV 存储│
         │ • 作为 URL Path 的  │            │ • 恒时比较防时序攻击 │            │ • 存储 UID、Policy、│
         │   一部分，防暴力枚举 │            │ • 签名内容包含：     │            │   文件大小、过期时 │
         │ • 间接纳入签名内容   │            │   Path + Header + Body│           │   间等关键状态      │
         │                      │            │ • 过期时间戳防重放   │            │ • 提供签名密钥来源 │
         └───────────┬──────────┘            └──────────┬──────────┘            └──────────┬──────────┘
                     │                                  │                                  │
                     ▼                                  ▼                                  ▼
         攻击者无法通过       ⇒          攻击者无法篡改请求       ⇒          即使通过前两层，上
         枚举找到有效 URL                内容（包括 UID、文件          传的 UID、大小等状态
                                       大小），也无法伪造有效          已在 Master 侧绑定，
                                       签名                              不可篡改
```

### 9.3 三者的协作流程（Remote 场景）

```
1. 上传准备阶段（Master 侧）：
   ┌─ 生成 UploadSession { UID: 12345, CallbackSecret: rand(32), Policy: {Node: {SlaveKey: "abc"}} }
   ├─ 生成回调 URL（包含 CallbackSecret）
   ├─ 存入 KV
   └─ RPC 传给 Slave

2. 签名阶段（Slave 侧）：
   ┌─ 上传完成，读取 UploadSession.Callback（含 CallbackSecret）
   ├─ 用 SlaveKey 对 "URL Path + Header + Body" 进行 HMAC 签名
   └─ 发送请求给 Master

3. 验签阶段（Master 侧）：
   ┌─ UseUploadSession: 通过 sessionID 从 KV 读取 UploadSession
   │                       （包含正确的 UID、SlaveKey、文件大小等）
   ├─ RemoteCallbackAuth: 用 UploadSession.Policy.Node.SlaveKey 验签
   │                       重新计算签名时 URL Path 包含 CallbackSecret
   │                       → 验证通过证明 URL 未被篡改
   └─ ProcessCallback: 使用 UploadSession.UID 注入用户上下文
                         使用 UploadSession.Props.Size 校验文件大小
```

### 9.4 安全设计亮点

1. **密钥分离原则**：
   - CallbackSecret 是每个会话独有的，只用于 URL 路径
   - HMAC 密钥（SlaveKey）是节点级别的长期密钥，不暴露在 URL 中
   - UploadSession 中的状态数据完全服务端存储，客户端不可篡改

2. **纵深防御**：
   - 即使 CallbackSecret 泄露，HMAC 签名仍然保护请求完整性
   - 即使 HMAC 密钥泄露，UploadSession 中的文件大小、过期时间等状态仍然限制攻击者的操作范围

3. **最小权限原则**：
   - CallbackSecret 返回给前端，但不授予任何权限（只是 URL 的一部分）
   - HMAC 密钥只在 Master 和 Slave 节点间共享，前端无法获取
   - UploadSession 永远不离开 Master 侧的 KV 存储

4. **失效安全**：
   - UploadSession 有 TTL，过期自动删除
   - HMAC 签名有过期时间，防止重放攻击
   - 上传完成后立即删除 UploadSession，防止二次使用

---

## 十、匿名用户机制深度解析 — 为什么空 Authorization 不尝试从 Session 恢复身份

### 8.1 AnonymousUser 的构造（[user.go#L432-L445](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/inventory/user.go#L432-L445)）

```go
func (c *userClient) AnonymousUser(ctx context.Context) (*ent.User, error) {
    groupClient := NewGroupClient(c.client, "", nil)
    // 查询 ID=3 的默认匿名组
    anonymousGroup, err := groupClient.AnonymousGroup(ctx)
    if err != nil {
        return nil, fmt.Errorf("anyonymous group not found: %w", err)
    }

    // 构造内存中的临时 User 对象，ID=0，不写入数据库
    anonymous := &ent.User{
        Settings: &types.UserSetting{},
    }
    anonymous.SetGroup(anonymousGroup) // 关联匿名组权限
    return anonymous, nil
}
```

**匿名组定义**（[group.go#L19](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/inventory/group.go#L19)）：
```go
const AnonymousGroupID = 3  // 数据库初始化时创建
```

**判断是否匿名用户**（[user.go#L544-L547](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/inventory/user.go#L544-L547)）：
```go
func IsAnonymousUser(u *ent.User) bool {
    return u.ID == 0  // 匿名用户没有数据库记录，ID 恒为 0
}
```

### 8.2 GetLoginUserByID 的分支逻辑（[user.go#L380-L397](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/inventory/user.go#L380-L397)）

```go
func (c *userClient) GetLoginUserByID(ctx context.Context, uid int) (*ent.User, error) {
    ctx = context.WithValue(ctx, LoadUserGroup{}, true)
    if uid > 0 {
        // 分支1：有效用户ID → 查询数据库
        expectedUser, err := c.GetActiveByID(ctx, uid)
        if err == nil {
            return expectedUser, nil
        }
        return nil, fmt.Errorf("failed to get user by id: %w", err)
    }

    // 分支2：uid <= 0（通常是 0）→ 构造匿名用户
    anonymous, err := c.AnonymousUser(ctx)
    if err != nil {
        return nil, fmt.Errorf("failed to construct anonymous user: %w", err)
    }
    return anonymous, nil
}
```

### 8.3 完整链路：从空 Authorization 到 AnonymousUser

```
GET /api/v4/file/list    (无 Authorization Header 或空字符串)
    │
    ▼
[全局中间件 CurrentUser]  [auth.go#L48-L71]
    │
    ├─► dep.TokenAuth().VerifyAndRetrieveUser(c)
    │      │
    │      └─► headerVal = c.GetHeader("Authorization") = ""
    │          tokenString = strings.TrimPrefix("", "Bearer ") = ""
    │          tokenString == "" → return true, nil
    │          ★ 没有注入 UserIDCtx ★
    │
    ├─► uid = inventory.UserIDFromContext(c)
    │      │
    │      └─► ctx.Value(UserIDCtx{}) → 不存在 → ok=false
    │          u = UserFromContext(c) → nil
    │          uid = 0 (默认零值)
    │          ★ 返回 0
    │
    └─► SetUserCtx(c, 0)
           │
           └─► GetLoginUserByID(c, 0)
                  │
                  └─► uid == 0 → c.AnonymousUser(c)
                         │
                         └─► 查询 Group(ID=3) → AnonymousGroup
                             构造 &ent.User{ID=0, Group=AnonymousGroup}
                             util.WithValue(c, UserCtx{}, anonymous)
           ★ UserCtx 被设置为匿名用户（ID=0）
    │
    ▼
[业务 Handler]
    │
    └─► user = inventory.UserFromContext(c) → ID=0, AnonymousUser=true

[如果路由组使用了 LoginRequired()]  [auth.go#L91-L101]
    │
    └─► u := inventory.UserFromContext(c)
        !inventory.IsAnonymousUser(u) → false (因为 u.ID == 0)
        → 返回 401 "Login required"
```

### 8.4 关键问题解答：为什么不从 Session 中恢复身份？

从架构设计上看，**Cloudreve v4 是纯 JWT 驱动的无状态应用**，Session Cookie 有明确的职责边界：

| 维度 | 分析 |
|-----|------|
| **Session 的唯一用途** | 在代码中搜索 session 的使用，只有两处：`CSRFInit` 和 `CSRFCheck`，用于存储和验证 CSRF Token。Session Cookie 从未存储 UID。 |
| **架构选型** | Cloudreve v3 时代可能混用 Session + JWT，v4 重构后全面转向 JWT。Session 中间件保留仅为兼容 CSRF 机制（CSRF 需要服务端存储 token） |
| **API 场景** | Cloudreve 是前后端分离架构，前端（React/Vue）在登录后将 JWT Access Token 存入 localStorage/内存，每次请求通过 `Authorization: Bearer xxx` 主动携带，而不是依赖 Cookie 自动发送 |
| **Cookie 的 SameSite 限制** | 如果嵌入第三方页面（如外链预览、文件预览 iframe），Cookie 会因 SameSite 策略被浏览器拦截，JWT 在 Header 中不受此限制 |
| **Master/Slave 分布式** | Slave 节点不需要处理用户登录，JWT 的无状态特性天然适合分布式系统；若依赖 Session，则需要共享 KV 并处理跨域 Cookie，复杂度大增 |
| **OAuth 2.0 兼容** | OAuth 客户端（Scope 机制）不可能持有用户的 Cookie，必须通过 Header Token |
| **CSRF 与身份解耦** | 从安全角度，CSRF Token 不应该与身份认证绑定，避免 Cookie 被窃取时同时获得身份 |

**代码证据 — Session 中从未写入 UID**：
搜索整个代码库，`sessions.Default(c)` 或 `sessions.GetMany` 的返回值调用 `.Set()` 只在 CSRF 相关代码里操作，没有写入 UID 的逻辑。Session 中间件创建后，**没有任何一行代码** 将用户 ID 写入 session store。

### 10.5 Session 使用场景的全面分类与职责边界

之前的结论（"Session 只用于 CSRF 防护"）需要更精确的表述。通过全局搜索 `util.SetSession`、`util.GetSession`、`util.DeleteSession` 的所有调用，session 的使用场景可以分为 **四类**，其中只有一类属于正式主链路，且均不涉及用户身份恢复：

| 场景类型 | 具体用途 | 代码位置 | 是否参与正式鉴权链路 | 是否操作 UID |
|---------|---------|---------|-------------------|-------------|
| **主链路预留（未实际启用）** | CSRF 防护 | [session.go#L48-L67](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/middleware/session.go#L48-L67) | ⚠️ 代码定义但路由未引用 | ❌ |
| **测试辅助** | 单元测试模拟上下文 | [mock.go#L14-L24](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/middleware/mock.go#L14-L24) | ❌ 仅测试环境 | ❌ |
| **已注释废弃代码** | OAuth 回调状态传递 | [oauth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/service/callback/oauth.go) | ❌ 已注释不运行 | ❌（存的是 policyID） |
| **完全无关** | aria2 下载器会话 | `aria2.GetSessionInfo()` | ❌ 概念不同 | ❌ |

#### 10.5.1 场景一：CSRF 防护（主链路预留但未实际启用）

`CSRFInit` 和 `CSRFCheck` 虽然在 [middleware/session.go](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/middleware/session.go) 中明确定义：

```go
// CSRFInit 初始化CSRF标记 [session.go#L48-L54]
func CSRFInit() gin.HandlerFunc {
    return func(c *gin.Context) {
        util.SetSession(c, map[string]interface{}{"CSRF": true})  // 只写入 CSRF=true
        c.Next()
    }
}

// CSRFCheck 检查CSRF标记 [session.go#L56-L67]
func CSRFCheck() gin.HandlerFunc {
    return func(c *gin.Context) {
        if check, ok := util.GetSession(c, "CSRF").(bool); ok && check {  // 只读取 CSRF 字段
            c.Next()
            return
        }
        // ... 拒绝
    }
}
```

**关键事实**：在路由配置文件中全局搜索 `CSRFInit` 和 `CSRFCheck`，**没有找到任何引用**。这意味着这两个中间件虽然代码存在，但在 v4 版本的路由链中并未实际挂载。

**即使启用也不影响结论**：
- 只读写 `"CSRF"` 布尔字段，**从不操作 UID**
- 不调用 `SetUserCtx`，**不注入 UserCtx**
- 只是 CSRF 防护机制，与身份认证完全解耦

#### 10.5.2 场景二：测试辅助（仅单元测试使用）

`MockHelper` 是单元测试专用的辅助中间件，定义在 [middleware/mock.go#L14-L24](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/middleware/mock.go#L14-L24)：

```go
// SessionMock 测试时模拟Session
var SessionMock = make(map[string]interface{})

// ContextMock 测试时模拟Context
var ContextMock = make(map[string]interface{})

// MockHelper 单元测试助手中间件
func MockHelper() gin.HandlerFunc {
    return func(c *gin.Context) {
        // 将全局变量 SessionMock 写入会话
        util.SetSession(c, SessionMock)
        // 将全局变量 ContextMock 写入 gin Context
        for key, value := range ContextMock {
            c.Set(key, value)
        }
        c.Next()
    }
}
```

**为什么不影响结论**：
- **生产环境不启用**：测试代码不会编译到生产二进制中
- **不是正常鉴权流程**：由测试代码主动设置全局变量 `SessionMock`，不是从 Cookie 中恢复
- **操作内容可控**：测试者写入什么就是什么，不是真实的用户身份恢复机制

#### 10.5.3 场景三：已注释的废弃代码（OAuth 回调）

[service/callback/oauth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/service/callback/oauth.go) 中的 Google Drive 和 OneDrive OAuth 回调代码已全部被 `//` 注释：

```go
//// GDriveAuth Google Drive 更新认证信息
//func (service *OauthService) GDriveAuth(c *gin.Context) serializer.Response {
//    // ...
//    policyID, ok := util.GetSession(c, "googledrive_oauth_policy").(uint)  // 存的是 policyID，不是 UID
//    // ...
//    util.DeleteSession(c, "googledrive_oauth_policy")
//    // ...
//}
```

**为什么不影响结论**：
- **代码不运行**：已被注释，不参与编译和执行
- **存储的不是 UID**：即使曾经运行过，存储的是 `policyID`（存储策略ID），不是用户 ID
- **用途不同**：用于 OAuth 授权回调时传递存储策略状态，不是用户身份认证

#### 10.5.4 场景四：完全无关的 "session" 概念

`aria2.GetSessionInfo()` 是 aria2 下载器的 RPC 调用，这里的 "session" 指下载器的会话状态，与用户鉴权 session 是完全不同的概念，不相关。

#### 10.5.5 为什么所有这些场景都不影响核心结论

**核心结论**：**空 Authorization 请求不会从 session 恢复身份**。

无论 session 在其他场景如何被使用，这个结论都成立，原因有五层保障：

| 保障层级 | 具体内容 | 代码证据 |
|---------|---------|---------|
| **第一层：CurrentUser 硬编码逻辑** | `CurrentUser` 是唯一负责全局用户身份注入的中间件，它**只读取 Authorization Header**，**从不读取 session** | [auth.go#L48-L71](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/middleware/auth.go#L48-L71) |
| **第二层：没有写入 UID 的代码** | 全局搜索 `SetSession` 调用，**没有任何一行代码** 将用户 ID 写入 session store。即使想从 session 恢复，也没有 UID 可恢复 | 全局搜索结果 |
| **第三层：CSRF 与身份解耦** | `CSRFInit`/`CSRFCheck` 只操作 `"CSRF"` 布尔值，从不涉及用户身份 | [session.go#L48-L67](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/middleware/session.go#L48-L67) |
| **第四层：测试代码不进生产** | `MockHelper` 仅用于单元测试，生产环境不会启用 | [mock.go#L14-L24](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/middleware/mock.go#L14-L24) |
| **第五层：废弃代码不运行** | OAuth 相关代码已被注释，不参与编译执行 | [oauth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/service/callback/oauth.go) |

**最关键的逻辑链**：
```
空 Authorization 请求
    │
    ▼
CurrentUser 中间件执行 [auth.go#L48-L71]
    │
    ├─ VerifyAndRetrieveUser: 只看 Authorization Header
    │   空 Header → 不注入 UserIDCtx → uid = 0
    │
    └─ SetUserCtx(c, 0) → 构造 AnonymousUser
    │
    ▼
后续业务代码通过 inventory.UserFromContext(c) 读取
    │
    └─ 得到 AnonymousUser（ID=0）
```

在这个流程中，**没有任何一步读取 session**。session 是否在其他地方被使用，与这个鉴权链路完全无关。

### 10.6 Session 中间件 vs CurrentUser 中间件的职责划分

```
Session Middleware  [session.go]
    │  职责：CSRF 防护
    │
    │  • 解析 "cloudreve-session" Cookie
    │  • 通过 sessionstore（自定义 KV 后端）读取会话数据
    │  • 通过 gin-contrib/sessions 将 session 对象放入 gin Context
    │  • 供后续 CSRFInit/CSRFCheck 读取/写入 CSRF Token
    │
    │  ╳ 不涉及 UserCtx、UserIDCtx
    ▼

CurrentUser Middleware  [auth.go]
    │  职责：用户身份注入
    │
    │  • 解析 "Authorization: Bearer xxx" Header
    │  • JWT 验证 → 提取 UID → 注入 UserIDCtx
    │  • 如果是 HMAC 头（Bearer Cr ...）→ 跳过，交给后续中间件
    │  • 如果是空 Header → uid=0 → 构造 AnonymousUser
    │  • 统一调用 SetUserCtx → 注入 UserCtx
    │
    │  ╳ 不读取 Session Cookie
    ▼

两者完全独立，互不通信！
```

---

## 九、完整上下文注入时序图（扩展版）

以 Master 模式下三种典型请求为例，对比上下文注入差异：

### 9.1 场景 A：正常 JWT 登录请求 `GET /api/v4/file/list`
```
[InitializeHandling] 注入基础元数据 → [Session] 解析 Cookie → [CurrentUser]
    │
    ├─ VerifyAndRetrieveUser:
    │   Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
    │   → 解析 JWT → uid=12345
    │   → util.WithValue(UserIDCtx, 12345)
    │
    ├─ UserIDFromContext → 12345
    │
    └─ SetUserCtx(c, 12345):
       → GetActiveByID(12345) → *ent.User{ID:12345, Email:"test@test.com"}
       → util.WithValue(UserCtx, user)
    ▼
[LoginRequired] → IsAnonymousUser? → false → PASS
[RequiredScopes] → hasScopes=false（非OAuth） → PASS
[Handler] → inventory.UserFromContext(c) → ID:12345
```

### 9.2 场景 B：空 Authorization 请求 `GET /api/v4/share/public`
```
[InitializeHandling] → [Session] → [CurrentUser]
    │
    ├─ VerifyAndRetrieveUser:
    │   Authorization: ""
    │   → tokenString 为空 → return true, nil
    │   → 不注入 UserIDCtx
    │
    ├─ UserIDFromContext → 0
    │
    └─ SetUserCtx(c, 0):
       → AnonymousUser()
       → *ent.User{ID:0, Group:AnonymousGroup(ID=3)}
       → util.WithValue(UserCtx, anonymous)
    ▼
[Handler 无 LoginRequired]
→ inventory.UserFromContext(c) → ID:0 (Anonymous)
→ 通过匿名组权限判断是否可以访问公开分享
```

### 9.3 场景 C：上传回调请求 `POST /api/v4/callback/remote/:sessionID/:key`
```
[InitializeHandling] → [Session] → [CurrentUser]
    │
    ├─ VerifyAndRetrieveUser:
    │   Authorization: Bearer Cr hD1k...P8Q:1718234567
    │   → strings.HasPrefix("Bearer Cr ") → true
    │   → return false, nil
    │   → 不注入 UserIDCtx
    │
    ├─ UserIDFromContext → 0
    │
    └─ SetUserCtx(c, 0):
       → *ent.User{ID:0} (AnonymousUser)  ← 第一次注入
    ▼
[UseUploadSession(PolicyTypeRemote)]
    │
    ├─ KV.Get("callback_" + sessionID) → UploadSession{UID:12345, ...}
    ├─ c.Set("uploadSession", &session)  ← gin Context
    │
    └─ SetUserCtx(c, 12345):  ← 第二次注入（覆盖）
       → GetActiveByID(12345) → 真实用户
       → util.WithValue(UserCtx, realUser)
    ▼
[RemoteCallbackAuth]
    │
    ├─ c.MustGet("uploadSession") → UploadSession
    ├─ session.Policy.Edges.Node.SlaveKey → HMAC Key
    └─ CheckRequest → 验证通过
    ▼
[ProcessCallback]
    │
    ├─ inventory.UserFromContext(c) → ID:12345 (真实用户)
    ├─ NewFileManager(dep, user)
    └─ CompleteUpload → 以真实用户身份完成文件写入
```

---

## 十、总结：四种鉴权方式的定位

| 鉴权方式 | 认证主体 | 凭证位置 | 适用对象 | 注入 UserCtx 的位置 | 注入时机 |
|---------|---------|---------|---------|-------------------|---------|
| **JWT Bearer Token** | 用户 / OAuth 客户端 | HTTP Header `Authorization: Bearer` | Web 前端、第三方应用 | `CurrentUser` 中间件 | 全局（第9层） |
| **Session Cookie** | （仅 CSRF） | Cookie `cloudreve-session` | 浏览器 | ❌ 不注入用户身份 | 全局（第8层），仅用于 CSRF |
| **HMAC Header + UploadSession** | 回调发起方间接证明 | Header `Authorization: Bearer Cr` + URL `:sessionID` | 主从通信、存储服务回调 | `UseUploadSession` 路由中间件 | 路由级（覆盖之前的匿名用户） |
| **HTTP Basic Auth** | 用户 | HTTP Header `Authorization: Basic` | WebDAV 客户端 | `WebDAVAuth` 路由中间件 | 路由级（独立于 CurrentUser） |

整个鉴权体系采用 **"中间件分层 + 上下文传播"** 模式：
1. **全局基础层**：InitializeHandling 注入基础设施，Session 注入 CSRF 状态
2. **全局身份层**：CurrentUser 尝试 JWT 鉴权，失败则回退 AnonymousUser
3. **路由覆盖层**：针对特殊场景（回调、WebDAV），路由级中间件会**重新注入** UserCtx，覆盖全局层的结果
4. **业务读取层**：Handler 统一通过 `inventory.UserFromContext(c)` 读取，无感知身份来源

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

**用途**：主要用于 CSRF 防护（`CSRFInit` / `CSRFCheck`），**不用于用户身份识别**。详见第九章分析。

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

## 八、匿名用户机制深度解析 — 为什么空 Authorization 不尝试从 Session 恢复身份

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

### 8.5 Session 中间件 vs CurrentUser 中间件的职责划分

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

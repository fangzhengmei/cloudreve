# Cloudreve 用户会话鉴权机制深度解析

## 一、整体架构概览

Cloudreve 采用 **Master/Slave** 双模式架构，每种模式拥有独立的路由和中间件链。鉴权体系由三层机制构成：

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

**用途**：主要用于 CSRF 防护（`CSRFInit` / `CSRFCheck`），不用于用户身份识别。

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

    // 关键：如果是 HMAC 鉴权头（Bearer Cr ...），跳过 JWT 验证
    if strings.HasPrefix(headerVal, TokenHeaderPrefixCr) {
        return false, nil  // 返回 false 表示交给其他鉴权方式
    }

    tokenString := strings.TrimPrefix(headerVal, TokenHeaderPrefix) // 去掉 "Bearer "
    if tokenString == "" {
        return true, nil  // 无 Token，继续使用 Session/匿名用户
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

**鉴权优先级设计**：
1. **`Bearer Cr ...`** → HMAC 鉴权，跳过 JWT
2. **`Bearer xxx`** → JWT Access Token
3. **空 Header** → 尝试 Session / 回退到匿名用户

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
| 文件下载 URL | URL Query `?sign=` | 系统全局 GeneralAuth | [auth.go#L648-L659](file:///d:/fz/0601-1/solo-dogfeeding/code/44-Cloudreve/routers/router.go#L648-L659) |
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

## 七、完整上下文注入时序图

以 Master 模式下用户请求 `GET /api/v3/file/list` 为例：

```
时间轴 ──────────────────────────────────────────────────────────►

[InitializeHandling]
  │  ├─ context.WithValue(dep)              │ 依赖容器
  │  ├─ context.WithValue(CorrelationIDCtx) │ 关联 ID
  │  ├─ context.WithValue(RequestInfoCtx)   │ IP/UA/Host
  │  ├─ context.WithValue(LoggerCtx)        │ Logger
  │  └─ context.WithValue(SlaveNodeIDCtx)   │ 节点ID
  ▼
[CORS]
  │
  ▼
[Session Middleware]
  │  └─ gin-contrib/sessions 解析 Cookie
  │     写入 gin.Context map (c.Set)
  ▼
[CurrentUser]
  │
  ├─ JWT Header 存在？
  │   ├─ Yes: Bearer Cr ...?
  │   │   ├─ Yes ─► 跳过 JWT（HMAC 场景），uid=0
  │   │   └─ No  ─► 解析 JWT → 解码 UID
  │   │              ├─ util.WithValue(UserIDCtx, uid)
  │   │              └─ util.WithValue(ScopeContextKey, scopes) [OAuth]
  │   └─ No: uid=0 (匿名)
  │
  ├─ UserIDFromContext(c) 获取 uid
  │
  └─ SetUserCtx(c, uid)
        ├─ uid > 0 → GetActiveByID(uid) 查用户
        └─ uid = 0 → AnonymousUser() 构造匿名用户
        └─ util.WithValue(UserCtx, *ent.User)
  ▼
[LoginRequired] ◄── 可选路由组中间件
  │  └─ UserFromContext(c).ID != 0 ?
  ▼
[RequiredScopes] ◄── 可选路由组中间件
  │  └─ 有 Scope? 检查 OAuth Scope : 放行
  ▼
[业务 Handler]
  │  └─ inventory.UserFromContext(c) 获取当前用户
```

---

## 八、总结：四种鉴权方式的定位

| 鉴权方式 | 认证主体 | 凭证位置 | 适用对象 | 是否注入 UserCtx |
|---------|---------|---------|---------|----------------|
| **JWT Bearer Token** | 用户 / OAuth 客户端 | HTTP Header | Web 前端、第三方应用 | ✅ 是 |
| **Session Cookie** | （仅 CSRF） | Cookie | 浏览器 | ❌ 仅作 CSRF 标记 |
| **HMAC Header/URL** | 节点 / 系统 | Header 或 URL Query | 主从通信、文件下载链接 | ❌ 不直接注入（通过 UploadSession 等间接获取） |
| **HTTP Basic Auth** | 用户 | HTTP Header | WebDAV 客户端 | ✅ 是 |

整个鉴权体系采用 **"中间件分层 + 上下文传播"** 模式：基础元数据最先注入，身份识别在中间层完成，业务 Handler 通过 `inventory.UserFromContext(c)` 等无侵入方式读取当前用户。

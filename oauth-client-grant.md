# OAuth 授权码链路代码分析

## 目录

1. [OAuth Client 配置](#1-oauth-client-配置)
2. [Grant 生命周期](#2-grant-生命周期)
   - [2.2.5 Grant 在授权确认到换 Token 之间的状态变化](#225-grant-在授权确认到换-token-之间的状态变化补充)
3. [Scope 边界](#3-scope-边界)
   - [3.5 Scope 收缩后的实际生效边界](#35-scope-收缩后的实际生效边界补充)
4. [撤销风险分析](#4-撤销风险分析)
   - [4.2.6 撤销后的访问窗口：精确时间线](#426-撤销后的访问窗口精确时间线补充)

---

## 1. OAuth Client 配置

### 1.1 数据模型

OAuth Client 的数据模型定义在 [oauthclient.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/ent/schema/oauthclient.go) 中。

| 字段 | 类型 | 说明 |
|------|------|------|
| `guid` | string (255) | 客户端唯一标识 (client_id)，全局唯一 |
| `secret` | string (255) | 客户端密钥，敏感字段（JSON 序列化时忽略） |
| `name` | string (255) | 客户端名称 |
| `homepage_url` | string (2048) | 客户端主页 URL（可选） |
| `redirect_uris` | []string | 允许的重定向 URI 列表（JSON 存储） |
| `scopes` | []string | 客户端允许请求的 scope 列表（JSON 存储） |
| `props` | OAuthClientProps | 扩展属性（JSON 存储） |
| `is_enabled` | bool | 是否启用，默认 true |

**OAuthClientProps 扩展属性**（[types.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/inventory/types/types.go#L230-L234)）：

```go
type OAuthClientProps struct {
    Description     string `json:"description,omitempty"`
    Icon            string `json:"icon,omitempty"`
    RefreshTokenTTL int64  `json:"refresh_token_ttl,omitempty"` // 秒，0表示使用默认
}
```

### 1.2 系统内置客户端

系统预置了两个不可删除的 OAuth 客户端，定义在 [migration.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/inventory/migration.go#L278-L287)：

| 客户端 | GUID | 默认 scopes | RefreshTokenTTL |
|--------|------|-------------|-----------------|
| Desktop | `393a1839-f52e-498e-9972-e77cc2241eee` | profile, email, openid, offline_access, UserInfo.Write, Workflow.Write, Files.Write, Shares.Write | 7776000 秒（90天） |
| iOS | `220db97a-44a3-44f7-99b6-d767262b4daa` | profile, email, openid, offline_access, UserInfo.Write, UserSecurityInfo.Write, Workflow.Write, Files.Write, Shares.Write, Finance.Write, DavAccount.Write | 7776000 秒（90天） |

系统客户端保护逻辑在 [oauth_client.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/service/admin/oauth_client.go#L186-L188)：
- 不可删除
- 不可修改 GUID

### 1.3 客户端管理接口

管理员管理接口定义在 [admin.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/routers/router.go#L1075-L1109)：

| 方法 | 路径 | 功能 |
|------|------|------|
| POST | `/api/v3/admin/oauthClient` | 分页查询客户端列表 |
| GET | `/api/v3/admin/oauthClient/:id` | 获取单个客户端详情（含 grant 数量） |
| PUT | `/api/v3/admin/oauthClient` | 创建新客户端 |
| PUT | `/api/v3/admin/oauthClient/:id` | 更新客户端 |
| DELETE | `/api/v3/admin/oauthClient/:id` | 删除客户端 |
| POST | `/api/v3/admin/oauthClient/batch/delete` | 批量删除 |

**创建逻辑**（[oauth_client.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/inventory/oauth_client.go#L194-L217)）：
- 未提供 GUID 时自动生成 UUID v4
- 未提供 Secret 时自动生成 32 位随机字符串

**删除逻辑**（[oauth_client.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/inventory/oauth_client.go#L240-L251)）：
- 先级联删除所有关联的 grants
- 再删除客户端本身

---

## 2. Grant 生命周期

### 2.1 Grant 数据模型

OAuthGrant 实体定义在 [oauthgrant.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/ent/schema/oauthgrant.go)：

| 字段 | 类型 | 说明 |
|------|------|------|
| `user_id` | int | 用户 ID（外键） |
| `client_id` | int | 客户端 ID（外键） |
| `scopes` | []string | 用户授权的 scopes（JSON 存储） |
| `last_used_at` | time.Time | 最近使用时间（可选，可为空） |

**唯一约束**：`(user_id, client_id)` 联合唯一索引，即每个用户对每个客户端最多只有一条 grant 记录。

### 2.2 授权码流完整流程

授权码流程涉及的核心文件是 [oauth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/service/oauth/oauth.go)。

#### 阶段一：用户授权（获取 authorization code）

**接口**：`POST /api/v3/session/oauth/consent`

**流程**（[oauth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/service/oauth/oauth.go#L59-L123)）：

1. **获取客户端**：通过 `client_id` 查询启用的客户端及其对当前用户的 grant
2. **验证 redirect_uri**：必须与客户端注册的 `redirect_uris` 之一完全匹配
3. **验证 scopes**：请求的 scopes 必须是客户端允许 scopes 的子集
4. **强制 openid scope**：必须包含 `openid` scope
5. **Upsert Grant**：创建或更新用户对该客户端的授权记录
   - 首次授权：创建新 grant，记录 scopes
   - 再次授权：更新 scopes 和 `last_used_at`
6. **生成授权码**：128 位加密随机字符串
7. **存储授权码**：存入 KV 缓存，TTL = 600 秒（10 分钟）

**AuthorizationCode 结构**（[response.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/service/oauth/response.go#L73-L81)）：

```go
type AuthorizationCode struct {
    ClientID      string   // 客户端 ID
    UserID        int      // 用户 ID
    Scopes        []string // 授权的 scopes
    RedirectURI   string   // 重定向 URI
    CodeChallenge string   // PKCE code challenge
}
```

**PKCE 支持**：
- `code_challenge_method` 仅支持 `S256`（SHA256）
- 若提供 `code_challenge` 但未指定方法，默认使用 `S256`

#### 阶段二：令牌交换（authorization code → access token）

**接口**：`POST /api/v3/session/oauth/token`

**流程**（[oauth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/service/oauth/oauth.go#L136-L234)）：

1. **从 KV 获取授权码**：若不存在或已过期则失败
2. **立即删除授权码**：防止重放攻击（一次性使用）
3. **验证 client_id 一致性**：授权码中的 client_id 必须与请求的一致
4. **PKCE 验证**：若授权码包含 `code_challenge`，则验证 `code_verifier` 的 SHA256 哈希是否匹配
5. **验证 client_secret**：与数据库中存储的密钥明文比对
6. **二次验证 scopes**：确保授权时的 scopes 仍在客户端允许范围内
7. **获取用户信息**：验证用户存在且处于活跃状态
8. **签发令牌**：调用 `tokenAuth.Issue()` 生成 access_token 和 refresh_token
9. **更新 grant last_used_at**：记录本次使用时间
10. **返回响应**：包含 `offline_access` scope 时才返回 refresh_token

### 2.2.5 Grant 在授权确认到换 Token 之间的状态变化（补充）

此前容易误解为：grant 在 token 交换成功后才正式建立。实际代码逻辑并非如此——**grant 在用户同意授权（consent）那一刻就已写入数据库**，与授权码是否被用来换取 token 完全独立。

**关键时序拆解**：

1. **T0：调用 `/oauth/consent` 之前**
   - grant 可能不存在（首次授权），或已存在（再次授权）

2. **T1：`GrantService.Get` 执行第 5 步 UpsertGrant**（[oauth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/service/oauth/oauth.go#L99-L102) → [oauth_client.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/inventory/oauth_client.go#L100-L112)）
   ```go
   func (c *oauthClientClient) UpsertGrant(...) error {
       return c.client.OAuthGrant.Create().
           SetUserID(userID).SetClientID(clientID).
           SetScopes(scopes).SetLastUsedAt(time.Now()).
           OnConflict(sql.ConflictColumns(oauthgrant.FieldUserID, oauthgrant.FieldClientID)).
           UpdateScopes().UpdateLastUsedAt().
           Exec(ctx)
   }
   ```
   - **首次授权**：INSERT 一条新 grant 记录，`scopes` = 请求的 scopes，`last_used_at` = 当前时间
   - **再次授权**：ON CONFLICT UPDATE，覆盖 `scopes` 和 `last_used_at`
   - 到这一步，grant 已经在数据库中**持久存在**

3. **T2：授权码生成并写入 KV**（T1 之后，T1+10 分钟内有效）
   - 授权码是独立的短期凭证（存放在 KV，TTL 600 秒）
   - 它指向 grant，但 grant 本身不依赖授权码的存在

4. **T3：授权码过期或未使用**
   - KV 中的授权码自动失效（TTL 到期）
   - **但 grant 不会被清理**，仍然保留在数据库中
   - 下次用户再 consent，直接走 UPSERT 更新逻辑

5. **T4：调用 `/oauth/token` 交换令牌（在 T1+10 分钟内）**
   - 从 KV 取出授权码并**立即删除**（一次性）
   - 校验成功后，在第 9 步调用 `UpdateGrantLastUsedAt`（[oauth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/service/oauth/oauth.go#L210-L213)）：
     ```go
     if err := oAuthClient.UpdateGrantLastUsedAt(c, user.ID, app.ID); err != nil {
         dep.Logger().Warning("Failed to update grant last used at: %s", err)
     }
     ```
   - 这一步**只更新 `last_used_at` 字段**，不修改 `scopes`
   - 注意：如果更新失败，仅打 Warning 日志，**不会导致 token 签发失败**（非阻塞）

**状态变化表**：

| 时间点 | 操作 | grant.scopes | grant.last_used_at | 授权码状态 |
|--------|------|--------------|-------------------|-----------|
| T0 | consent 前 | 空（或旧值） | 空（或旧值） | 不存在 |
| T1 | UpsertGrant | **设为请求的 scopes** | **设为当前时间** | 不存在 |
| T1.5 | 授权码写入 KV | 不变 | 不变 | 存在（10分钟 TTL） |
| T2 | 授权码过期 | 不变 | 不变 | 已失效 |
| T4 | 交换 token 成功 | 不变 | **更新为当前时间** | 已删除 |
| T4+ | refresh token 刷新 | 不变 | **更新为当前时间** | 不存在 |

**核心结论**：
- Grant 的创建和 scope 写入发生在 consent 阶段，不是 token 交换阶段
- 即使授权码过期/被丢弃，grant 仍然存在且有效
- Token 交换和后续 refresh 只更新 `last_used_at`，不改变 scope
- Grant 一旦创建，只能通过显式删除（用户撤销 / 管理员删客户端）或 UPSERT 覆盖（再次 consent）来改变

#### 2.2.6 哪些操作会刷新 `grant.last_used_at`（代码对照）

通过全局搜索 `UpdateGrantLastUsedAt` 和 `UpsertGrant` 的所有调用点，整个代码库中只有 **3 处**会更新 `last_used_at`：

| # | 操作 | 代码位置 | 更新方式 | 是否同时更新 scopes |
|---|------|---------|---------|-------------------|
| 1 | 用户调用 `/oauth/consent` 同意授权 | [oauth.go L100](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/service/oauth/oauth.go#L100) → [oauth_client.go L100-L112](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/inventory/oauth_client.go#L100-L112) `UpsertGrant` | `ON CONFLICT ... UpdateLastUsedAt()` | ✅ 是（同时 `UpdateScopes()`） |
| 2 | 调用 `/oauth/token` 用授权码换 token | [oauth.go L211](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/service/oauth/oauth.go#L211) → [oauth_client.go L114-L119](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/inventory/oauth_client.go#L114-L119) `UpdateGrantLastUsedAt` | `UPDATE ... SET last_used_at = NOW()` | ❌ 否 |
| 3 | 用 Refresh Token 刷新新 token | [jwt.go L179](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/pkg/auth/jwt.go#L179) → [oauth_client.go L114-L119](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/inventory/oauth_client.go#L114-L119) `UpdateGrantLastUsedAt` | `UPDATE ... SET last_used_at = NOW()` | ❌ 否 |

**重要细节差异**：
- **操作 1 (UpsertGrant)** 同时更新 `scopes` 和 `last_used_at`。这是唯一能改变 grant.scopes 的代码路径（除此之外只有删除再重建）。
- **操作 2 (换 token)** 更新失败时**不报错**（只打 Warning 日志，见 [oauth.go L211-L213](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/service/oauth/oauth.go#L211-L213)），即使 `last_used_at` 更新失败，token 照常签发。
- **操作 3 (Refresh)** 更新失败时**报错终止**（`return nil, ErrInvalidRefreshToken`，见 [jwt.go L179-L181](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/pkg/auth/jwt.go#L179-L181)）。
- 日常使用 Access Token 调用 API **不会**更新 `last_used_at`——`VerifyAndRetrieveUser` 中没有任何数据库写操作。

---

### 2.3 Token 签发与刷新

#### Token 结构

Token 签发逻辑在 [jwt.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/pkg/auth/jwt.go#L236-L291)：

**Claims 结构**：

```go
type Claims struct {
    TokenType TokenType          // "access" 或 "refresh"
    jwt.RegisteredClaims        // 标准 JWT claims (sub, exp, nbf 等)
    StateHash   []byte           // 用户状态哈希（仅 refresh token）
    RootTokenID *uuid.UUID       // 根令牌 ID，用于撤销
    Scopes      []string         // OAuth scopes
    ClientID    string           // OAuth 客户端 ID
}
```

**默认 TTL**（由 `TokenAuth` 配置决定）：
- Access Token：由全局配置决定
- Refresh Token：由全局配置决定，或客户端 `props.RefreshTokenTTL` 覆盖

#### Refresh Token 刷新流程

刷新逻辑在 [jwt.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/pkg/auth/jwt.go#L125-L195)：

1. 解析 refresh token，验证类型为 `refresh`
2. 解码用户 ID，查询用户是否存在且活跃
3. **用户状态哈希校验**：比对 `state_hash`（由 `email/password/site_id` 哈希生成），检测密码变更
4. **根令牌撤销检查**：检查 `jwt_revoke_{root_token_id}` 是否存在于 KV 中
5. **OAuth 客户端校验**（仅 OAuth 颁发的 token）：
   - 客户端必须存在且启用
   - 用户的 grant 必须仍然存在
   - token 的 scopes 必须是 grant scopes 的子集
   - 更新 grant 的 `last_used_at`
6. 签发新的 token pair（access + refresh），保持同一 `root_token_id`

### 2.4 Grant 生命周期总结

```
用户授权
   ↓
Grant 创建（UpsertGrant）
   ↓
授权码生成（10分钟 TTL）
   ↓
令牌交换（授权码一次性销毁）
   ↓
Access Token 使用（更新 last_used_at）
   ↓
Refresh Token 刷新（验证 grant 有效性）
   ↓
...（循环刷新）
   ↓
用户撤销 / 客户端删除 / 用户改密
   ↓
Grant 失效 → Refresh Token 无法继续刷新
```

---

## 3. Scope 边界

### 3.1 Scope 定义

所有预定义 scope 常量在 [types.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/inventory/types/types.go#L395-L416)：

**OpenID Connect 标准 scopes**：

| Scope | 说明 |
|-------|------|
| `openid` | 强制要求，用于标识 OIDC 请求 |
| `profile` | 用户基本信息（name, preferred_username, picture, updated_at） |
| `email` | 用户邮箱（email, email_verified） |
| `offline_access` | 允许获取 refresh_token |

**资源 scopes（CRUD 风格）**：

| 资源域 | Read scope | Write scope |
|--------|-----------|-------------|
| 用户信息 | `UserInfo.Read` | `UserInfo.Write` |
| 用户安全信息 | `UserSecurityInfo.Read` | `UserSecurityInfo.Write` |
| 工作流 | `Workflow.Read` | `Workflow.Write` |
| 管理员 | `Admin.Read` | `Admin.Write` |
| 文件 | `Files.Read` | `Files.Write` |
| 分享 | `Shares.Read` | `Shares.Write` |
| 财务 | `Finance.Read` | `Finance.Write` |
| WebDAV 账户 | `DavAccount.Read` | `DavAccount.Write` |

### 3.2 Scope 校验机制

#### 客户端级 Scope 过滤

**授权时校验**（[oauth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/service/oauth/oauth.go#L89-L92)）：
- 请求的 scopes 必须是客户端 `scopes` 的子集
- 由 `auth.ValidateScopes()` 函数实现

**令牌交换时二次校验**（[oauth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/service/oauth/oauth.go#L182-L185)）：
- 再次验证 scopes 有效性，防止授权后客户端 scope 被收缩

**刷新 token 时校验**（[jwt.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/pkg/auth/jwt.go#L173-L176)）：
- token 的 scopes 必须是当前 grant scopes 的子集
- 防止 grant scope 收缩后旧 token 仍可使用

#### ValidateScopes 函数

[ValidateScopes](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/pkg/auth/jwt.go#L299-L312) 实现简单的子集校验：

```go
func ValidateScopes(requestedScopes, allowedScopes []string) bool {
    allowed := make(map[string]struct{}, len(allowedScopes))
    for _, scope := range allowedScopes {
        allowed[scope] = struct{}{}
    }
    for _, scope := range requestedScopes {
        if _, ok := allowed[scope]; !ok {
            return false
        }
    }
    return true
}
```

### 3.3 API 级 Scope 保护

`RequiredScopes` 中间件（[auth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/middleware/auth.go#L275-L289)）用于保护 API 端点。

**CheckScope 函数**（[jwt.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/pkg/auth/jwt.go#L322-L347)）核心逻辑：

1. **会话认证豁免**：若 token 不含 scopes（如 session cookie 认证），则跳过 scope 检查
2. **Write 隐式包含 Read**：拥有 `{Resource}.Write` scope 自动获得 `{Resource}.Read` 权限
3. **全部满足原则**：所有必需 scopes 都必须存在才放行

### 3.4 Scope 在 UserInfo 端点的应用

[GetUserInfo](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/service/oauth/oauth.go#L266-L315) 端点根据 scope 条件性返回字段：

| Scope | 返回字段 |
|-------|----------|
| `openid` | `sub`（始终返回） |
| `profile` | `name`, `preferred_username`, `picture`, `updated_at` |
| `email` | `email`, `email_verified` |

### 3.5 Scope 边界总结

```
客户端配置 scopes
    ↓ （子集校验）
用户授权 grant scopes
    ↓ （子集校验）
Access Token scopes
    ↓ （API 中间件校验）
API 端点访问
```

**重要特性**：
- Scope 只减不增：每次刷新都重新校验，客户端/grant 收缩 scope 后旧 token 立即失效
- Write 包含 Read：写入权限隐含读取权限
- 会话认证绕过 scope：非 OAuth 方式登录的请求不受 scope 限制

---

## 4. 撤销风险分析

### 4.1 撤销路径概览

| 撤销方式 | 影响范围 | 生效时机 | 实现位置 |
|----------|----------|----------|----------|
| 用户撤销 Grant | 单个用户对单个客户端的所有 token | 下次刷新时 | [DeleteOAuthGrant](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/service/oauth/oauth.go#L243-L259) |
| 用户主动登出（撤销 refresh token） | 单个 root_token_id 下的所有 refresh token | 下次刷新时 | [login.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/service/user/login.go#L194-L208) |
| 用户修改密码 | 该用户所有 refresh token | 下次刷新时 | [jwt.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/pkg/auth/jwt.go#L149-L153) |
| 管理员禁用客户端 | 该客户端所有用户的所有 token | 下次刷新时 | GetByGUID 过滤 is_enabled |
| 管理员删除客户端 | 该客户端所有用户的所有 token + grants | 下次刷新时 | [Delete](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/inventory/oauth_client.go#L240-L251) |
| 管理员收缩客户端 scopes | 该客户端超出新 scope 的 token | 下次刷新时 | Refresh 时 ValidateScopes |

### 4.2 关键风险点

#### 风险 1：Access Token 窗口期

**问题**：Access Token 在有效期内无法主动撤销。

**原因**：
- Access Token 是自包含 JWT，无状态验证
- 撤销操作（grant 删除、密码修改、token 撤销）仅影响 refresh 流程
- Access Token 只能等待自然过期

**代码依据**：
- [VerifyAndRetrieveUser](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/pkg/auth/jwt.go#L197-L234) 只验证 JWT 签名和过期时间，不检查 grant 状态或撤销列表

**影响**：
- 已签发的 Access Token 在过期前仍然有效
- 若 Access Token TTL 较长，撤销后存在较长的"危险窗口"

#### 风险 2：Grant 删除不主动失效当前 Access Token

**问题**：用户删除 grant 后，已签发的 Access Token 仍然可用直到过期。

**代码依据**：
- [DeleteOAuthGrantService.Delete](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/service/oauth/oauth.go#L243-L259) 仅删除数据库中的 grant 记录
- 不涉及 KV 中的 token 撤销
- 仅在下次 refresh 时才会因 grant 不存在而失败

#### 风险 3：系统内置客户端密钥硬编码

**问题**：Desktop 和 iOS 客户端的 secret 在代码中硬编码。

**代码依据**（[migration.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/inventory/migration.go#L279-L286)）：

```go
OAuthClientDesktopSecret = "8GaQIu3lOSdqYoDHi9cR8IZ4pvuMH8ya"
OAuthClientiOSSecret     = "1kxOW4IyVOkPlsKCnTwzfHyP8XrbpfaF"
```

**影响**：
- 由于客户端类型是桌面/移动应用，属于 public client，secret 无法保密
- 依赖 PKCE 机制提供安全保障
- 若 PKCE 未启用，存在客户端冒充风险

#### 风险 4：授权码存储在 KV 中可能的泄露

**问题**：授权码存储在 KV 缓存中，若 KV （如 Redis）被攻破，攻击者可窃取授权码交换 token。

**缓解因素**：
- 授权码 TTL 仅 10 分钟
- 授权码一次性使用，交换后立即删除
- 配合 PKCE 时，即使授权码泄露也无法使用（缺少 code_verifier）

#### 风险 5：重定向 URI 精确匹配

**验证**（[oauth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/service/oauth/oauth.go#L74-L84)）：

```go
redirectValid := false
for _, uri := range app.RedirectUris {
    if uri == s.RedirectURI {
        redirectValid = true
        break
    }
}
```

**安全性**：使用精确字符串匹配，而非前缀匹配或通配符，符合 OAuth 2.0 安全最佳实践。

### 4.3 安全机制总结

| 安全机制 | 作用 | 实现方式 |
|----------|------|----------|
| PKCE (S256) | 防止授权码劫持 | SHA256(code_verifier) 对比 |
| 授权码一次性 | 防止重放 | 令牌交换时立即从 KV 删除 |
| 授权码短 TTL | 缩小攻击窗口 | 10 分钟 |
| state 参数 | 防止 CSRF | 由客户端生成，原样返回 |
| 精确 redirect_uri 匹配 | 防止授权码泄露到恶意站点 | 字符串精确匹配 |
| 用户状态哈希 | 密码变更后 token 失效 | SHA256(email+password+site_id) |
| Root Token ID 撤销 | 主动吊销 refresh token | KV 中记录 `jwt_revoke_{id}` |
| Scope 逐级收缩校验 | 权限只降不升 | 授权/交换/刷新三次校验 |
| Secret 敏感字段保护 | 防止密钥泄露 | JSON 序列化时 `json:"-"` |

### 4.4 撤销时序图

```
用户撤销 Grant
    │
    ├─→ 数据库删除 grant 记录
    │
    ├─→ 当前 Access Token ──→ 继续有效直到过期 ❗
    │
    └─→ 下次 Refresh Token 刷新
            │
            ├─→ 查询 grant → 不存在 → 刷新失败 ✓
            └─→ 重新登录 → 创建新 grant ✓
```

### 4.5 改进建议（基于代码分析）

1. **缩短 Access Token TTL**：降低撤销窗口期
2. **Access Token 黑名单**：对高安全级别操作，可考虑引入短期访问令牌撤销列表
3. **Grant 删除时级联撤销根令牌**：删除 grant 时同时将该用户-客户端对应的所有 root_token_id 加入撤销列表
4. **强化 Public Client 检测**：对移动端/桌面端客户端强制要求 PKCE
5. **审计日志**：记录 grant 创建、撤销、token 刷新等安全事件

---

## 参考文件

- 核心服务：[service/oauth/oauth.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/service/oauth/oauth.go)
- 数据访问层：[inventory/oauth_client.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/inventory/oauth_client.go)
- JWT 认证：[pkg/auth/jwt.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/pkg/auth/jwt.go)
- OAuth 客户端 Schema：[ent/schema/oauthclient.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/ent/schema/oauthclient.go)
- OAuth Grant Schema：[ent/schema/oauthgrant.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/ent/schema/oauthgrant.go)
- Scope 常量：[inventory/types/types.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/inventory/types/types.go#L395-L416)
- 系统客户端初始化：[inventory/migration.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/inventory/migration.go#L278-L340)
- 路由配置：[routers/router.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/routers/router.go)
- 管理员服务：[service/admin/oauth_client.go](file:///d:/fz/0601-1/solo-dogfeeding/code/45-Cloudreve/service/admin/oauth_client.go)

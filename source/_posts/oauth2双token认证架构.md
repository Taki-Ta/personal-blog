---
title: OAuth2双Token认证架构
date: 2025-11-25 13:53:00
categories:
  - 技术文档
tags:
  - OAuth2
  - Token
  - 认证架构
---
## 目录

1. [核心概念](#核心概念)

1. [为什么需要双Token设计](#为什么需要双token设计)

1. [数据结构设计](#数据结构设计)

1. [完整流程与数据流向](#完整流程与数据流向)

1. [安全特性](#安全特性)

1. [代码实现参考](#代码实现参考)

1. [存储设计](#存储设计)

1. [最佳实践](#最佳实践)

---

## 核心概念

### AccessToken（访问令牌）

- **类型**: JWT (JSON Web Token)

- **用途**: 访问受保护的API资源

- **生命周期**: 短期（15分钟 - 1小时）

- **存储位置**: 前端内存/Cookie（HttpOnly）

- **状态**: 无状态（服务端不存储）

- **验证方式**: 签名验证

### RefreshToken（刷新令牌）

- **类型**: 随机字符串（Base64编码）

- **用途**: 获取新的AccessToken

- **生命周期**: 长期（7天 - 30天）

- **存储位置**: 前端Cookie（HttpOnly）+ 后端数据库

- **状态**: 有状态（服务端存储哈希值）

- **验证方式**: 数据库查询验证

---

## 为什么需要双Token设计

### 单一Token的问题

> **只使用AccessToken（长期有效）:**

### 双Token的优势

> **AccessToken（短期）+ RefreshToken（长期）:**

---

## 数据结构设计

### AccessToken结构（JWT）

```json
{
  "header": {
    "alg": "HS256",
    "typ": "JWT"
  },
  "payload": {
    "sub": "user_id_12345",           // 用户ID
    "username": "john_doe",            // 用户名
    "role": "admin",                   // 角色
    "iat": 1700000000,                 // 签发时间
    "exp": 1700003600,                 // 过期时间（1小时后）
    "iss": "your-app-name",            // 签发者
    "aud": "your-app-client"           // 受众
  },
  "signature": "HMACSHA256(...)"
}
```

**编码后的JWT**:

```javascript
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyX2lkXzEyMzQ1IiwidXNlcm5hbWUiOiJqb2huX2RvZSIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTcwMDAwMDAwMCwiZXhwIjoxNzAwMDAzNjAwfQ.signature_here
```

### RefreshToken结构（随机字符串）

**前端存储（明文）:**

```javascript
Suvzc22XnoHb0KgBO4+sfK5d0M7oiNGKZ9emKxBeDFs=
```

**后端存储（数据库记录）:**

```json
{
  "id": "unique_record_id",
  "userId": "user_id_12345",
  "tokenHash": "de6017bc62ae8542478ef14770f5afa...",  // SHA256(RefreshToken)
  "familyId": "family_uuid",                         // Token家族ID
  "createdAt": "2024-11-25T03:05:48Z",
  "expiresAt": "2024-12-25T03:05:48Z",               // 30天后过期
  "revokedAt": null,                                 // null=活跃，有值=已撤销
  "replacedByTokenHash": null,                       // 被哪个新token替换
  "reason": null,                                    // 撤销原因："rotated"/"reused"/"logout"
  "clientIp": "192.168.1.100",                       // 客户端IP
  "userAgent": "Mozilla/5.0..."                      // 用户代理
}
```

---

## 完整流程与数据流向

### 1. 用户登录流程

```javascript
┌─────────┐                 ┌─────────┐                 ┌──────────┐
│ 前端    │                 │ 后端    │                 │ 数据库   │
└────┬────┘                 └────┬────┘                 └────┬─────┘
     │                           │                           │
     │ 1. POST /api/auth/login   │                           │
     │   {username, password}    │                           │
     ├──────────────────────────>│                           │
     │                           │                           │
     │                           │ 2. 验证用户名密码          │
     │                           ├──────────────────────────>│
     │                           │<──────────────────────────┤
     │                           │   返回用户信息             │
     │                           │                           │
     │                           │ 3. 生成AccessToken (JWT)  │
     │                           │    payload = {sub, role...}│
     │                           │    sign with secret_key   │
     │                           │                           │
     │                           │ 4. 生成RefreshToken       │
     │                           │    random_bytes(32)       │
     │                           │    → Base64编码           │
     │                           │    = "Suvzc22X..."        │
     │                           │                           │
     │                           │ 5. 计算TokenHash          │
     │                           │    SHA256("Suvzc22X...")  │
     │                           │    = "de6017bc..."        │
     │                           │                           │
     │                           │ 6. 存储到数据库            │
     │                           ├──────────────────────────>│
     │                           │   INSERT RefreshTokenRecord│
     │                           │   {userId, tokenHash,     │
     │                           │    familyId, expiresAt}   │
     │                           │<──────────────────────────┤
     │                           │                           │
     │ 7. 返回响应               │                           │
     │ {                         │                           │
     │   accessToken: "eyJhb...",│                           │
     │   refreshToken:"Suvzc...",│  (明文)                   │
     │   expiresAt: "..."        │                           │
     │ }                         │                           │
     │<──────────────────────────┤                           │
     │ + Set-Cookie:             │                           │
     │   access_token=eyJhb...   │                           │
     │   refresh_token=Suvzc...  │                           │
     │                           │                           │
     │ 8. 存储到Cookie/内存      │                           │
     │                           │                           │
```

**关键数据流**:

```javascript
RefreshToken明文: "Suvzc22X..." → 前端Cookie
TokenHash哈希值:  "de6017bc..." → 数据库
```

---

### 2. 访问受保护资源

```javascript
┌─────────┐                 ┌─────────┐
│ 前端    │                 │ 后端    │
└────┬────┘                 └────┬────┘
     │                           │
     │ 1. GET /api/users/profile │
     │    Header:                │
     │    Authorization: Bearer  │
     │    eyJhbGciOiJIUzI1Ni...  │
     ├──────────────────────────>│
     │                           │
     │                           │ 2. 验证JWT签名
     │                           │    verify(token, secret_key)
     │                           │
     │                           │ 3. 检查过期时间
     │                           │    if (now > exp) → 401
     │                           │
     │                           │ 4. 提取用户信息
     │                           │    userId = payload.sub
     │                           │
     │                           │ 5. 处理业务逻辑
     │                           │    getUserProfile(userId)
     │                           │
     │ 6. 返回数据               │
     │ {profile: {...}}          │
     │<──────────────────────────┤
     │                           │
```

> **性能优势**: 不需要查询数据库，JWT自包含所有信息。

---

### 3. AccessToken过期后刷新

```javascript
┌─────────┐                 ┌─────────┐                 ┌──────────┐
│ 前端    │                 │ 后端    │                 │ 数据库   │
└────┬────┘                 └────┬────┘                 └────┬─────┘
     │                           │                           │
     │ 1. GET /api/users/profile │                           │
     │    Authorization: Bearer  │                           │
     │    eyJhbGci... (已过期)   │                           │
     ├──────────────────────────>│                           │
     │                           │                           │
     │ 2. 返回401 Unauthorized   │                           │
     │    {error: "token_expired"}│                           │
     │<──────────────────────────┤                           │
     │                           │                           │
     │ 3. 自动触发刷新           │                           │
     │    POST /api/auth/refresh │                           │
     │    Cookie: refresh_token= │                           │
     │            Suvzc22X...    │  (明文)                   │
     ├──────────────────────────>│                           │
     │                           │                           │
     │                           │ 4. 计算哈希值              │
     │                           │    hash = SHA256(         │
     │                           │      "Suvzc22X...")       │
     │                           │    = "de6017bc..."        │
     │                           │                           │
     │                           │ 5. 查询数据库              │
     │                           ├──────────────────────────>│
     │                           │   SELECT * FROM tokens    │
     │                           │   WHERE tokenHash=        │
     │                           │         "de6017bc..."     │
     │                           │<──────────────────────────┤
     │                           │   返回记录 (revokedAt=null)│
     │                           │                           │
     │                           │ 6. 验证token              │
     │                           │    - 检查是否过期          │
     │                           │    - 检查是否已撤销        │
     │                           │    - 检查用户状态          │
     │                           │                           │
     │                           │ 7. 生成新tokens           │
     │                           │    newAccessToken = JWT() │
     │                           │    newRefreshToken =      │
     │                           │      random() → "YzN2M..." │
     │                           │    newTokenHash =         │
     │                           │      SHA256("YzN2M...")   │
     │                           │      = "a8f3c19d..."      │
     │                           │                           │
     │                           │ 8. 撤销旧token             │
     │                           ├──────────────────────────>│
     │                           │   UPDATE tokens SET       │
     │                           │     revokedAt = now,      │
     │                           │     replacedByTokenHash = │
     │                           │       "a8f3c19d...",      │
     │                           │     reason = "rotated"    │
     │                           │   WHERE tokenHash =       │
     │                           │     "de6017bc..."         │
     │                           │<──────────────────────────┤
     │                           │                           │
     │                           │ 9. 插入新token记录         │
     │                           ├──────────────────────────>│
     │                           │   INSERT {                │
     │                           │     tokenHash:"a8f3c19d...",
     │                           │     familyId: (继承旧的),  │
     │                           │     revokedAt: null       │
     │                           │   }                       │
     │                           │<──────────────────────────┤
     │                           │                           │
     │ 10. 返回新tokens          │                           │
     │ {                         │                           │
     │   accessToken: "eyJuZX...",                           │
     │   refreshToken:"YzN2M...",│  (新明文)                 │
     │   expiresAt: "..."        │                           │
     │ }                         │                           │
     │<──────────────────────────┤                           │
     │ + Set-Cookie: (更新Cookie)│                           │
     │                           │                           │
     │ 11. 重试原请求            │                           │
     │    GET /api/users/profile │                           │
     │    Authorization: Bearer  │                           │
     │    eyJuZXci... (新token)  │                           │
     ├──────────────────────────>│                           │
     │                           │                           │
     │ 12. 成功返回数据          │                           │
     │<──────────────────────────┤                           │
     │                           │                           │
```

**Token轮换**:

```javascript
旧RefreshToken: "Suvzc22X..." → 已撤销
旧TokenHash:    "de6017bc..." → revokedAt = now, reason = "rotated"

新RefreshToken: "YzN2M..."    → 前端Cookie更新
新TokenHash:    "a8f3c19d..." → 数据库新记录
```

---

### 4. 用户登出流程

```javascript
┌─────────┐                 ┌─────────┐                 ┌──────────┐
│ 前端    │                 │ 后端    │                 │ 数据库   │
└────┬────┘                 └────┬────┘                 └────┬─────┘
     │                           │                           │
     │ 1. POST /api/auth/logout  │                           │
     │    Authorization: Bearer  │                           │
     │    eyJhbGci...            │                           │
     │    Cookie: refresh_token  │                           │
     ├──────────────────────────>│                           │
     │                           │                           │
     │                           │ 2. 验证AccessToken        │
     │                           │    获取userId             │
     │                           │                           │
     │                           │ 3. 撤销RefreshToken       │
     │                           ├──────────────────────────>│
     │                           │   UPDATE tokens SET       │
     │                           │     revokedAt = now,      │
     │                           │     reason = "logout"     │
     │                           │   WHERE userId = ?        │
     │                           │     AND revokedAt IS NULL │
     │                           │<──────────────────────────┤
     │                           │                           │
     │ 4. 返回成功               │                           │
     │ {message: "logged_out"}   │                           │
     │<──────────────────────────┤                           │
     │ + Clear-Cookie            │                           │
     │                           │                           │
     │ 5. 清除本地tokens         │                           │
     │    删除Cookie/内存        │                           │
     │                           │                           │
```

---

## 安全特性

### 1. Token Rotation（Token轮换）

**原理**: 每次刷新AccessToken时，同时生成新的RefreshToken，旧的立即撤销。

**好处**:

- 减少RefreshToken被盗用的窗口期

- 即使被截获，下次刷新后立即失效

```javascript
时间线：
T0: 登录    → RefreshToken_A (有效)
T1: 刷新    → RefreshToken_A (撤销) + RefreshToken_B (有效)
T2: 刷新    → RefreshToken_B (撤销) + RefreshToken_C (有效)
```

### 2. Token Family（Token家族）

**原理**: 同一用户的所有RefreshToken共享一个FamilyId，形成家族链。

**数据结构**:

```javascript
登录时生成 familyId = "f47ac10b-58cc-4372-a567-0e02b2c3d479"

Token_A: {tokenHash: "aaa...", familyId: "f47ac10b...", revokedAt: 2024-11-25T10:00:00, replacedBy: "bbb..."}
         ↓ 刷新
Token_B: {tokenHash: "bbb...", familyId: "f47ac10b...", revokedAt: 2024-11-25T11:00:00, replacedBy: "ccc..."}
         ↓ 刷新
Token_C: {tokenHash: "ccc...", familyId: "f47ac10b...", revokedAt: null}  ← 当前活跃
```

**追踪链**:

```javascript
Token_A → replacedBy → Token_B → replacedBy → Token_C (活跃)
```

### 3. Reuse Detection（重用检测）

**场景**: 攻击者窃取了已被轮换的旧RefreshToken，尝试使用。

**检测逻辑**:

```python
def refresh_token(incoming_refresh_token):
    incoming_hash = sha256(incoming_refresh_token)
    record = db.find_token(incoming_hash)

    if record is None:
        return error("Invalid token")

    # 关键检测：已被撤销的token被重用
    if record.revokedAt is not None:
        # 🚨 安全警报：检测到token重用！
        # 撤销整个家族的所有token
        db.revoke_family(record.familyId, reason="reused")
        log_security_alert(record.userId, "Token reuse detected")
        return error("Token revoked")

    if record.expiresAt < now():
        return error("Token expired")

    # 正常刷新流程...
```

**攻击场景示例**:

```javascript
1. 用户正常登录 → Token_A (有效)
2. 攻击者窃取了 Token_A
3. 用户正常刷新 → Token_A (撤销) + Token_B (有效)
4. 攻击者尝试使用被窃取的 Token_A → 🚨 检测到重用！
5. 系统撤销整个Family（Token_A, Token_B全部失效）
6. 用户被强制重新登录
```

> **防御效果**:

---

## 代码实现参考

### 1. 生成RefreshToken（通用伪代码）

```python
import os
import base64
import hashlib

def generate_refresh_token():
    """生成32字节随机RefreshToken"""
    random_bytes = os.urandom(32)  # 密码学安全的随机数
    refresh_token = base64.b64encode(random_bytes).decode('utf-8')
    return refresh_token
    # 返回: "Suvzc22XnoHb0KgBO4+sfK5d0M7oiNGKZ9emKxBeDFs="

def hash_token(token):
    """计算Token的SHA256哈希值"""
    return hashlib.sha256(token.encode('utf-8')).hexdigest()
    # 返回: "de6017bc62ae8542478ef14770f5afa696ca9ccbb..."
```

**各语言实现**:

```javascript
// Node.js
const crypto = require('crypto');

function generateRefreshToken() {
    return crypto.randomBytes(32).toString('base64');
}

function hashToken(token) {
    return crypto.createHash('sha256').update(token).digest('hex');
}
```

```java
// Java
import [java.security](http://java.security/).SecureRandom;
import [java.security](http://java.security/).MessageDigest;
import java.util.Base64;

public String generateRefreshToken() {
    SecureRandom random = new SecureRandom();
    byte[] bytes = new byte[32];
    random.nextBytes(bytes);
    return Base64.getEncoder().encodeToString(bytes);
}

public String hashToken(String token) throws Exception {
    MessageDigest digest = MessageDigest.getInstance("SHA-256");
    byte[] hash = digest.digest(token.getBytes("UTF-8"));
    return bytesToHex(hash);
}
```

```go
// Go
import (
    "crypto/rand"
    "crypto/sha256"
    "encoding/base64"
    "encoding/hex"
)

func generateRefreshToken() (string, error) {
    bytes := make([]byte, 32)
    if _, err := [rand.Read](http://rand.read/)(bytes); err != nil {
        return "", err
    }
    return base64.StdEncoding.EncodeToString(bytes), nil
}

func hashToken(token string) string {
    hash := sha256.Sum256([]byte(token))
    return hex.EncodeToString(hash[:])
}
```

```c#
// C#
using System;
using [System.Security](http://system.security/).Cryptography;
using System.Text;

public string GenerateRefreshToken()
{
    var randomBytes = new byte[32];
    using var rng = RandomNumberGenerator.Create();
    rng.GetBytes(randomBytes);
    return Convert.ToBase64String(randomBytes);
}

public string HashToken(string token)
{
    using var sha256 = SHA256.Create();
    var bytes = sha256.ComputeHash(Encoding.UTF8.GetBytes(token));
    var sb = new StringBuilder(bytes.Length * 2);
    foreach (var b in bytes)
    {
        sb.Append(b.ToString("x2"));
    }
    return sb.ToString();
}
```

---

### 2. 生成AccessToken（JWT）

```python
import jwt
import datetime

def generate_access_token(user, secret_key):
    """生成JWT AccessToken"""
    payload = {
        'sub': [user.id](http://user.id/),              # Subject（用户ID）
        'username': user.username,
        'role': user.role,
        'iat': datetime.datetime.utcnow(),  # Issued At
        'exp': datetime.datetime.utcnow() + datetime.timedelta(hours=1),  # Expiration
        'iss': 'your-app-name',      # Issuer
        'aud': 'your-app-client'     # Audience
    }

    access_token = jwt.encode(payload, secret_key, algorithm='HS256')
    return access_token
```

---

### 3. 登录逻辑

```python
def login(username, password):
    # 1. 验证用户
    user = db.get_user(username)
    if not user or not verify_password(password, user.password_hash):
        return error("Invalid credentials")

    # 2. 生成AccessToken (JWT)
    access_token = generate_access_token(user, SECRET_KEY)

    # 3. 生成RefreshToken (随机字符串)
    refresh_token = generate_refresh_token()
    token_hash = hash_token(refresh_token)

    # 4. 存储RefreshToken记录到数据库
    family_id = generate_uuid()
    db.insert_refresh_token({
        'user_id': [user.id](http://user.id/),
        'token_hash': token_hash,        # 存哈希值，不存明文
        'family_id': family_id,
        'created_at': now(),
        'expires_at': now() + timedelta(days=30),
        'revoked_at': None,
        'client_ip': request.client_ip,
        'user_agent': request.user_agent
    })

    # 5. 返回给前端（明文）
    return {
        'access_token': access_token,
        'refresh_token': refresh_token,  # 明文
        'expires_at': now() + timedelta(hours=1)
    }
```

---

### 4. Token刷新逻辑（含安全检测）

```python
def refresh_token(incoming_refresh_token):
    # 1. 计算传入token的哈希值
    incoming_hash = hash_token(incoming_refresh_token)

    # 2. 查询数据库
    existing = db.find_token_by_hash(incoming_hash)
    if not existing:
        return error("Invalid refresh token")

    # 3. 🚨 重用检测
    if existing.revoked_at is not None:
        # Token已被撤销但仍被使用 → 可能是攻击
        db.revoke_entire_family([existing.family](http://existing.family/)_id, reason="reused")
        log_security_alert(existing.user_id, "Token reuse detected")
        return error("Token has been revoked due to suspicious activity")

    # 4. 检查过期
    if existing.expires_at < now():
        return error("Refresh token expired")

    # 5. 验证用户状态
    user = db.get_user(existing.user_id)
    if not user or not [user.is](http://user.is/)_active:
        return error("User not found or inactive")

    # 6. 生成新的AccessToken
    new_access_token = generate_access_token(user, SECRET_KEY)

    # 7. 生成新的RefreshToken（Token轮换）
    new_refresh_token = generate_refresh_token()
    new_token_hash = hash_token(new_refresh_token)

    # 8. 撤销旧token
    db.update_token([existing.id](http://existing.id/), {
        'revoked_at': now(),
        'replaced_by_token_hash': new_token_hash,
        'reason': 'rotated'
    })

    # 9. 插入新token记录（继承family_id）
    db.insert_refresh_token({
        'user_id': [user.id](http://user.id/),
        'token_hash': new_token_hash,
        'family_id': [existing.family](http://existing.family/)_id,  # 继承家族ID
        'created_at': now(),
        'expires_at': now() + timedelta(days=30),
        'revoked_at': None,
        'client_ip': request.client_ip,
        'user_agent': request.user_agent
    })

    # 10. 返回新tokens（明文）
    return {
        'access_token': new_access_token,
        'refresh_token': new_refresh_token,  # 新的明文
        'expires_at': now() + timedelta(hours=1)
    }
```

---

### 5. API请求验证

```python
def protected_endpoint():
    # 1. 从请求头获取AccessToken
    auth_header = request.headers.get('Authorization')
    if not auth_header or not auth_header.startswith('Bearer '):
        return error(401, "Missing or invalid Authorization header")

    access_token = auth_header[7:]  # 移除 "Bearer " 前缀

    # 2. 验证JWT
    try:
        payload = jwt.decode(
            access_token,
            SECRET_KEY,
            algorithms=['HS256'],
            audience='your-app-client',
            issuer='your-app-name'
        )
    except jwt.ExpiredSignatureError:
        return error(401, "Access token expired")
    except jwt.InvalidTokenError:
        return error(401, "Invalid access token")

    # 3. 提取用户信息
    user_id = payload['sub']
    username = payload['username']
    role = payload['role']

    # 4. 处理业务逻辑
    data = get_user_data(user_id)
    return success(data)
```

---

## 存储设计

### 数据库表结构（SQL）

```sql
CREATE TABLE refresh_tokens (
    id                      VARCHAR(36) PRIMARY KEY,        -- UUID
    user_id                 VARCHAR(36) NOT NULL,           -- 用户ID（外键）
    token_hash              VARCHAR(64) NOT NULL,           -- SHA256哈希值（64字符）
    family_id               VARCHAR(36) NOT NULL,           -- Token家族ID
    replaced_by_token_hash  VARCHAR(64),                    -- 被哪个新token替换
    created_at              TIMESTAMP NOT NULL,             -- 创建时间
    expires_at              TIMESTAMP NOT NULL,             -- 过期时间
    revoked_at              TIMESTAMP,                      -- 撤销时间（NULL=活跃）
    reason                  VARCHAR(20),                    -- 撤销原因：rotated/reused/logout
    client_ip               VARCHAR(45),                    -- IPv4/IPv6
    user_agent              TEXT,                           -- 用户代理字符串

    INDEX idx_token_hash (token_hash),                      -- 查询优化
    INDEX idx_user_id (user_id),                            -- 查询用户所有token
    INDEX idx_family_id (family_id),                        -- 撤销家族
    INDEX idx_expires_at (expires_at),                      -- 清理过期token

    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

### NoSQL存储（MongoDB）

```json
{
  "_id": "674380cc7dc6091f2e57f10a",
  "userId": "40288ABA5BC7AE98015BC7B0A2E20001",
  "tokenHash": "de6017bc62ae8542478ef14770f5afa696ca9ccbb94d61533749fbe...",
  "familyId": "e8f7d6c5-4321-4f7a-9ab8-1234567890ab",
  "replacedByTokenHash": null,
  "createdAt": ISODate("2024-11-25T03:05:48.123Z"),
  "expiresAt": ISODate("2024-12-25T03:05:48.123Z"),
  "revokedAt": null,
  "reason": null,
  "clientIp": "192.168.1.100",
  "userAgent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36..."
}
```

**索引**:

```javascript
db.refresh_tokens.createIndex({ "tokenHash": 1 }, { unique: true })
db.refresh_tokens.createIndex({ "userId": 1 })
db.refresh_tokens.createIndex({ "familyId": 1 })
db.refresh_tokens.createIndex({ "expiresAt": 1 })
```

---

## 最佳实践

### 1. 时间配置建议

### 2. 存储位置

**Cookie配置示例**:

```javascript
Set-Cookie: access_token=eyJhbGci...; HttpOnly; Secure; SameSite=Strict; Max-Age=3600; Path=/
Set-Cookie: refresh_token=Suvzc22X...; HttpOnly; Secure; SameSite=Strict; Max-Age=2592000; Path=/api/auth/refresh
```

### 3. 安全检查清单

- [ ] RefreshToken使用密码学安全的随机数生成器（`crypto.randomBytes`, `os.urandom`, `SecureRandom`）

- [ ] RefreshToken在数据库中存储哈希值，不存明文

- [ ] 使用强密钥（至少256位）签名JWT

- [ ] JWT密钥不能硬编码，应从环境变量读取

- [ ] 实现Token轮换机制

- [ ] 实现Token Family追踪

- [ ] 实现Reuse Detection检测

- [ ] 登出时撤销RefreshToken

- [ ] HTTPS强制启用（生产环境）

- [ ] Cookie设置`HttpOnly; Secure; SameSite=Strict`

- [ ] 记录安全事件日志（登录、刷新、重用检测）

- [ ] 定期清理过期token记录（数据库维护任务）

### 4. 错误处理

```python
# 前端错误处理伪代码
async function apiRequest(url, options):
    response = await fetch(url, {
        ...options,
        headers: {
            'Authorization': f'Bearer {accessToken}'
        }
    })

    if response.status == 401:
        error = await response.json()

        if error.code == 'token_expired':
            # 尝试刷新token
            new_tokens = await refreshAccessToken()
            if new_tokens:
                # 更新本地token
                accessToken = new_tokens.access_token
                # 重试原请求
                return await apiRequest(url, options)
            else:
                # 刷新失败，跳转登录页
                redirectToLogin()

        elif error.code == 'token_revoked':
            # Token被撤销（可能检测到安全问题）
            clearTokens()
            redirectToLogin()
            showAlert('您的会话已过期，请重新登录')

    return response
```

### 5. 数据库维护

```sql
-- 定期清理过期且已撤销的token（建议每天运行）
DELETE FROM refresh_tokens
WHERE expires_at < NOW() - INTERVAL 30 DAY
  AND revoked_at IS NOT NULL;

-- 查找可疑的重用行为
SELECT user_id, COUNT(*) as reuse_count
FROM refresh_tokens
WHERE reason = 'reused'
  AND revoked_at > NOW() - INTERVAL 7 DAY
GROUP BY user_id
HAVING reuse_count > 3;
```

### 6. 监控指标

- **Token刷新成功率**: `(成功刷新次数 / 总刷新请求) * 100%`

- **Token重用检测次数**: 每天检测到的重用次数（正常应接近0）

- **RefreshToken平均寿命**: 从创建到撤销的平均时间

- **活跃RefreshToken数量**: `revokedAt IS NULL`的记录数

- **异常IP登录**: 同一用户从多个地理位置登录

---

## 架构图总结

```javascript
┌──────────────────────────────────────────────────────────────┐
│                         双Token架构                           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐                           ┌─────────────┐  │
│  │ AccessToken │                           │RefreshToken │  │
│  │   (JWT)     │                           │  (Random)   │  │
│  └──────┬──────┘                           └──────┬──────┘  │
│         │                                          │         │
│         │ 用途：访问API                            │ 用途：刷新AccessToken
│         │ 生命周期：短（1小时）                    │ 生命周期：长（30天）
│         │ 存储：前端Cookie                         │ 存储：前端Cookie + 后端DB
│         │ 状态：无状态                             │ 状态：有状态
│         │ 验证：签名验证                           │ 验证：数据库查询
│         │                                          │         │
│         ▼                                          ▼         │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                     前端存储                          │  │
│  │  Cookie: access_token=eyJhbGci...  (明文JWT)         │  │
│  │  Cookie: refresh_token=Suvzc22X... (明文随机串)      │  │
│  │          HttpOnly, Secure, SameSite=Strict           │  │
│  └──────────────────────────────────────────────────────┘  │
│                            │                                │
│                            │ API请求                        │
│                            ▼                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                     后端验证                          │  │
│  │  AccessToken  → JWT签名验证（无需查库，快速）        │  │
│  │  RefreshToken → SHA256哈希后查MongoDB（有状态）      │  │
│  └──────────────────────────────────────────────────────┘  │
│                            │                                │
│                            │ Token过期/刷新                 │
│                            ▼                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                   Token轮换                           │  │
│  │  旧RefreshToken → 撤销（revokedAt=now）              │  │
│  │  新RefreshToken → 生成并继承family_id                │  │
│  │  新AccessToken  → 生成新JWT                          │  │
│  └──────────────────────────────────────────────────────┘  │
│                            │                                │
│                            │ 检测到重用                     │
│                            ▼                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                 Reuse Detection                       │  │
│  │  撤销整个Token Family → 用户强制重新登录             │  │
│  │  记录安全警报 → 通知安全团队                         │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└──────────────────────────────────────────────────────────────┘

数据流向：
登录:   生成明文 → 计算哈希 → 哈希存DB → 明文返回前端
访问:   前端发JWT → 后端验证签名 → 返回数据
刷新:   前端发明文 → 后端算哈希查DB → 生成新tokens → 明文返回
登出:   前端请求 → 后端撤销DB记录 → 前端清除Cookie
```

---

## 附录：常见问题

### Q1: 为什么RefreshToken不用JWT？

**A**:

- JWT是无状态的，无法主动撤销（除非引入黑名单，失去无状态优势）

- RefreshToken需要撤销能力（登出、异常检测）

- 随机字符串 + 数据库存储提供更强的控制能力

### Q2: 可以只用RefreshToken访问API吗？

**A**: 不推荐。

- 每个API请求都查数据库，性能差

- RefreshToken生命周期长，频繁传输增加泄露风险

- 双Token设计是性能和安全的最佳平衡

### Q3: AccessToken放LocalStorage安全吗？

**A**: 不安全。

- LocalStorage易受XSS攻击（恶意脚本可读取）

- 推荐使用HttpOnly Cookie（JS无法访问）

### Q4: Token轮换会增加多少数据库负载？

**A**: 负载很小。

- 只有刷新时才查库（每1小时一次）

- 正常API请求用JWT，无需查库

- 索引优化后单次查询<10ms

### Q5: 移动应用如何存储Token？

**A**:

- iOS: Keychain（系统级加密存储）

- Android: EncryptedSharedPreferences（Android Keystore加密）

- React Native: react-native-keychain

- 不要用AsyncStorage/SharedPreferences明文存储

### Q6: 如何处理多设备登录？

**A**:

- 每个设备生成独立的RefreshToken（不同的FamilyId）

- 用户可以查看所有活跃会话

- 提供"登出所有设备"功能（撤销该用户所有RefreshToken）

### Q7: Token被盗用后最坏情况是什么？

**A**:

- **AccessToken被盗**: 最多1小时内有效，过期后失效

- **RefreshToken被盗且未被检测**: 攻击者可持续刷新，直到用户登出或Token过期

- **RefreshToken被盗且被检测**: Reuse Detection触发，所有token撤销，攻击者和用户都需重新登录

### Q8: 如何实现"记住我"功能？

**A**:

- 不要延长AccessToken时间（保持短期）

- 延长RefreshToken有效期（如90天）

- 前端保持RefreshToken，定期静默刷新AccessToken


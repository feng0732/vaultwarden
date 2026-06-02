# 双因素认证 (2FA) 和 WebAuthn 代码分析

## 目录
- [整体架构](#整体架构)
- [TwoFactorType 枚举](#twofactortype-枚举)
- [WebAuthn 流程](#webauthn-流程)
- [Authenticator (TOTP) 流程](#authenticator-totp-流程)
- [恢复码 (Recovery Code) 机制](#恢复码-recovery-code-机制)
- [禁用流程](#禁用流程)
- [Protected Actions 机制](#protected-actions-机制)
- [登录验证流程](#登录验证流程)
- [数据库模型](#数据库模型)

---

## 整体架构

### 模块位置
- 核心模块: [src/api/core/two_factor/mod.rs](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/mod.rs)
- WebAuthn: [src/api/core/two_factor/webauthn.rs](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/webauthn.rs)
- Authenticator: [src/api/core/two_factor/authenticator.rs](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/authenticator.rs)
- Protected Actions: [src/api/core/two_factor/protected_actions.rs](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/protected_actions.rs)
- 数据库模型: [src/db/models/two_factor.rs](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/db/models/two_factor.rs)

### 支持的 2FA 类型
| 类型 | 说明 |
|------|------|
| Authenticator (0) | TOTP 验证器 (Google Authenticator 等) |
| Email (1) | 邮件验证 |
| Duo (2) | Duo 安全 |
| YubiKey (3) | YubiKey 硬件密钥 |
| U2f (4) | 旧版 U2F (已迁移至 WebAuthn) |
| Remember (5) | 记住设备 |
| OrganizationDuo (6) | 组织级 Duo |
| Webauthn (7) | WebAuthn/FIDO2 |
| RecoveryCode (8) | 恢复码 |

---

## TwoFactorType 枚举

定义在 [src/db/models/two_factor.rs#L27-L47](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/db/models/two_factor.rs#L27-L47):

```rust
pub enum TwoFactorType {
    // 用户可见的类型 (0-999)
    Authenticator = 0,
    Email = 1,
    Duo = 2,
    YubiKey = 3,
    U2f = 4,
    Remember = 5,
    OrganizationDuo = 6,
    Webauthn = 7,
    RecoveryCode = 8,

    // 内部实现类型 (1000+)
    U2fRegisterChallenge = 1000,
    U2fLoginChallenge = 1001,
    EmailVerificationChallenge = 1002,
    WebauthnRegisterChallenge = 1003,
    WebauthnLoginChallenge = 1004,

    // Protected Actions 验证 (2000)
    ProtectedActions = 2000,
}
```

---

## WebAuthn 流程

### 1. WebAuthn 初始化

WebAuthn 实例使用懒加载初始化 [webauthn.rs#L33-L45](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/webauthn.rs#L33-L45):

```rust
static WEBAUTHN: LazyLock<Webauthn> = LazyLock::new(|| {
    let domain = CONFIG.domain();
    let domain_origin = CONFIG.domain_origin();
    let rp_id = Url::parse(&domain).map(|u| u.domain().map(str::to_owned)).ok().flatten().unwrap_or_default();
    let rp_origin = Url::parse(&domain_origin).unwrap();

    let webauthn = WebauthnBuilder::new(&rp_id, &rp_origin)
        .expect("Creating WebauthnBuilder failed")
        .rp_name(&domain)
        .timeout(Duration::from_mins(1));

    webauthn.build().expect("Building Webauthn failed")
});
```

**关键配置**:
- 从 DOMAIN 配置中提取 rp_id (relying party ID)
- 超时时间: 1 分钟
- 用户验证策略: `Discouraged` (作为 2FA 使用，不需要用户验证)

### 2. 启用 WebAuthn - 获取注册信息

**端点**: `POST /two-factor/get-webauthn` [webauthn.rs#L110-L129](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/webauthn.rs#L110-L129)

**流程**:
1. 验证用户密码或 OTP
2. 调用 `get_webauthn_registrations()` 获取已注册的密钥
3. 返回启用状态和密钥列表

### 3. 启用 WebAuthn - 生成注册挑战

**端点**: `POST /two-factor/get-webauthn-challenge` [webauthn.rs#L131-L170](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/webauthn.rs#L131-L170)

**流程**:
1. 验证用户密码或 OTP
2. 获取已注册的凭证 ID (避免重复注册)
3. 调用 `WEBAUTHN.start_passkey_registration()` 生成挑战
4. 保存挑战状态到数据库 (`TwoFactorType::WebauthnRegisterChallenge`)
5. 修改默认配置:
   - 清空扩展
   - 用户验证策略改为 `Discouraged`
6. 返回挑战给客户端

### 4. 启用 WebAuthn - 激活

**端点**: `POST /two-factor/webauthn` (或 PUT) [webauthn.rs#L255-L304](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/webauthn.rs#L255-L304)

**流程**:
1. 验证用户密码或 OTP
2. 从数据库读取并删除注册挑战状态
3. 调用 `WEBAUTHN.finish_passkey_registration()` 验证设备响应
4. 将新凭证添加到注册列表
5. 保存到数据库 (`TwoFactorType::Webauthn`)
6. 生成恢复码 (`generate_recover_code`)
7. 记录用户事件

**数据存储结构** [webauthn.rs#L73-L80](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/webauthn.rs#L73-L80):
```rust
pub struct WebauthnRegistration {
    pub id: i32,           // 密钥 ID (1-5)
    pub name: String,      // 用户自定义名称
    pub migrated: bool,    // 是否从 U2F 迁移
    pub credential: Passkey,  // WebAuthn 凭证
}
```

### 5. WebAuthn 登录验证

#### 5.1 生成登录挑战

**函数**: `generate_webauthn_login()` [webauthn.rs#L378-L416](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/webauthn.rs#L378-L416)

**流程**:
1. 加载用户所有 WebAuthn 凭证
2. 调用 `WEBAUTHN.start_passkey_authentication()` 生成挑战
3. 修改配置:
   - 用户验证策略: `Discouraged`
   - 添加 appid 扩展 (U2F 兼容)
4. 保存挑战状态 (`TwoFactorType::WebauthnLoginChallenge`)
5. 返回挑战

#### 5.2 验证登录响应

**函数**: `validate_webauthn_login()` [webauthn.rs#L418-L465](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/webauthn.rs#L418-L465)

**流程**:
1. 从数据库读取并删除登录挑战
2. 解析客户端响应
3. 检查并更新 `backup_eligible` 标志 (向后兼容)
4. 调用 `WEBAUTHN.finish_passkey_authentication()` 验证
5. 更新凭证计数器 (如果需要)
6. 返回成功

**备份标志更新逻辑** [webauthn.rs#L467-L518](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/webauthn.rs#L467-L518):
- 从 authenticator data 第 33 字节提取标志位
- `FLAG_BACKUP_ELIGIBLE` (0x08): 设备是否可备份
- `FLAG_BACKUP_STATE` (0x10): 当前备份状态

### 6. 删除 WebAuthn 密钥

**端点**: `DELETE /two-factor/webauthn` [webauthn.rs#L318-L365](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/webauthn.rs#L318-L365)

**流程**:
1. 验证主密码
2. 读取用户 WebAuthn 数据
3. 按 ID 查找并删除指定密钥
4. 如果是从 U2F 迁移的，同步删除 U2F 记录
5. 保存更新后的数据

---

## Authenticator (TOTP) 流程

### 1. 生成 TOTP 密钥

**端点**: `POST /two-factor/get-authenticator` [authenticator.rs#L21-L45](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/authenticator.rs#L21-L45)

**流程**:
1. 验证用户密码或 OTP
2. 检查是否已有 TOTP 配置
3. 如未配置，生成 20 字节随机密钥并 Base32 编码
4. 返回启用状态和密钥

### 2. 激活 TOTP

**端点**: `POST /two-factor/authenticator` (或 PUT) [authenticator.rs#L56-L94](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/authenticator.rs#L56-L94)

**流程**:
1. 验证用户密码或 OTP
2. 验证密钥格式 (Base32, 20 字节)
3. 验证用户输入的 TOTP 令牌
4. 生成恢复码
5. 记录用户事件

### 3. TOTP 验证

**函数**: `validate_totp_code()` [authenticator.rs#L115-L181](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/authenticator.rs#L115-L181)

**验证逻辑**:
1. 使用 SHA1 算法，30 秒时间步长，6 位数字
2. 时间漂移容忍: 默认 ±1 步 (可配置)
3. 防重放: 记录 `last_used` 时间步，拒绝重复使用
4. 更新 `last_used` 并保存

```rust
// 检查时间范围内的所有可能代码
for step in -steps..=steps {
    let time_step = current_timestamp / 30i64 + step;
    let generated = totp_custom::<Sha1>(30, 6, &decoded_secret, time);
    
    if generated == totp_code && time_step > twofactor.last_used {
        // 有效代码，更新 last_used
        twofactor.last_used = time_step;
        twofactor.save(conn).await?;
        return Ok(());
    }
}
```

### 4. 禁用 TOTP

**端点**: `DELETE /two-factor/authenticator` [authenticator.rs#L191-L219](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/authenticator.rs#L191-L219)

**流程**:
1. 验证主密码
2. 验证提供的密钥与数据库存储一致
3. 删除 TOTP 记录
4. 检查是否还有其他 2FA 方式，如无则强制执行组织策略

---

## 恢复码 (Recovery Code) 机制

### 生成恢复码

**函数**: `generate_recover_code()` [mod.rs#L121-L127](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/mod.rs#L121-L127)

```rust
async fn generate_recover_code(user: &mut User, conn: &DbConn) {
    if user.totp_recover.is_none() {
        let totp_recover = crypto::encode_random_bytes::<20>(&BASE32);
        user.totp_recover = Some(totp_recover);
        user.save(conn).await.ok();
    }
}
```

**特性**:
- 存储在 `User` 表的 `totp_recover` 字段中
- 仅在首次启用 2FA 时生成
- 20 字节随机数，Base32 编码

### 获取恢复码

**端点**: `POST /two-factor/get-recover` [mod.rs#L108-L119](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/mod.rs#L108-L119)

**流程**:
1. 验证用户密码或 OTP
2. 返回恢复码

### 使用恢复码登录

在 `twofactor_auth()` 中处理 [identity.rs#L860-L875](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/identity.rs#L860-L875):

**流程**:
1. 验证恢复码是否正确 (`user.check_valid_recovery_code()`)
2. **删除用户所有 2FA 配置** (`TwoFactor::delete_all_by_user()`)
3. 强制执行组织 2FA 策略
4. 清除用户的恢复码
5. 记录恢复事件

> **重要**: 使用恢复码后，所有 2FA 方式都会被清除，用户需要重新设置

---

## 禁用流程

### 通用禁用端点

**端点**: `POST /two-factor/disable` (或 PUT) [mod.rs#L137-L167](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/mod.rs#L137-L167)

**流程**:
1. 验证用户密码或 OTP
2. 根据 `type` 参数查找对应的 2FA 记录
3. 删除记录
4. 记录禁用事件
5. 如果用户没有其他 2FA 方式，强制执行组织策略

### 组织 2FA 策略强制执行

**函数**: `enforce_2fa_policy()` [mod.rs#L174-L206](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/mod.rs#L174-L206)

**逻辑**:
1. 查找用户所在的启用了 2FA 策略的组织
2. 对于非管理员/所有者成员:
   - 发送邮件通知
   - 撤销成员资格
   - 记录事件

这确保用户不能通过禁用所有 2FA 来绕过组织的强制 2FA 要求。

---

## Protected Actions 机制

用于验证敏感操作（如导出金库）的额外安全验证。

### 请求 OTP

**端点**: `POST /accounts/request-otp` [protected_actions.rs#L64-L97](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/protected_actions.rs#L64-L97)

**流程**:
1. 检查邮件功能是否启用
2. 冷却期检查 (30 秒)
3. 生成随机令牌
4. 保存到数据库 (`TwoFactorType::ProtectedActions`)
5. 发送邮件

**存储结构** [protected_actions.rs#L22-L62](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/protected_actions.rs#L22-L62):
```rust
pub struct ProtectedActionData {
    pub token: String,           // 生成的令牌
    pub token_sent: NaiveDateTime,  // 发送时间
    pub attempts: u64,           // 尝试次数
}
```

### 验证 OTP

**端点**: `POST /accounts/verify-otp` [protected_actions.rs#L106-L120](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/protected_actions.rs#L106-L120)

**函数**: `validate_protected_action_otp()` [protected_actions.rs#L122-L158](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/protected_actions.rs#L122-L158)

**验证逻辑**:
1. 增加尝试次数
2. 超过尝试次数限制 → 失败
3. 超过过期时间 → 删除并失败
4. 常量时间比较令牌
5. 验证成功后删除令牌 (可配置)

---

## 登录验证流程

主入口: `twofactor_auth()` [identity.rs#L761-L893](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/identity.rs#L761-L893)

### 整体流程

```
用户登录
  ↓
检查是否有 2FA 配置
  ├─ 无 → 检查组织策略 → 完成登录
  └─ 有 → 标记未完成登录
       ↓
获取可用的 2FA 提供者
  ↓
选择验证方式 (默认第一个)
  ↓
验证 2FA 令牌
  ├─ 失败 → 返回 2FA 要求错误
  └─ 成功 → 标记完成登录
       ↓
可选: 生成 Remember 令牌
  ↓
完成登录
```

### 2FA 提供者选择

在 `json_err_twofactor()` 中返回可用提供者信息 [identity.rs#L899-L1010](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/identity.rs#L899-L1010):

- **WebAuthn**: 预先生成登录挑战并返回
- **Duo**: 返回 Duo iframe 或 OIDC 信息
- **YubiKey**: 返回服务器信息
- **Email**: 自动发送邮件并返回提示

### Remember 设备机制

**类型**: `TwoFactorType::Remember` (5)

**流程** [identity.rs#L837-L858](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/identity.rs#L837-L858):
1. 检查设备是否有 `twofactor_remember` 令牌
2. 验证 JWT 令牌的有效性:
   - 设备匹配
   - 用户匹配
3. 验证失败则删除旧令牌并要求重新验证

**令牌生成**:
```rust
// 生成有效期 30 天的 JWT
pub fn generate_2fa_remember_claims(device_uuid: DeviceId, user_uuid: UserId) -> TwoFactorRememberClaims {
    let time_now = Utc::now();
    TwoFactorRememberClaims {
        nbf: time_now.timestamp(),
        exp: (time_now + TimeDelta::try_days(30).unwrap()).timestamp(),
        iss: JWT_2FA_REMEMBER_ISSUER.to_string(),
        sub: device_uuid,
        user_uuid,
    }
}
```

---

## 数据库模型

### TwoFactor 表结构

定义在 [src/db/models/two_factor.rs#L15-L25](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/db/models/two_factor.rs#L15-L25):

| 字段 | 类型 | 说明 |
|------|------|------|
| uuid | TwoFactorId | 主键 |
| user_uuid | UserId | 用户 ID |
| atype | i32 | 类型 (TwoFactorType) |
| enabled | bool | 是否启用 |
| data | String | JSON 格式的配置数据 |
| last_used | i64 | 上次使用时间 (TOTP 为时间步) |

**唯一约束**: `(user_uuid, atype)` - 每个用户每种类型只能有一条记录

### 关键数据库方法

| 方法 | 说明 |
|------|------|
| `find_by_user()` | 获取用户所有 2FA 配置 (过滤内部类型) |
| `find_by_user_and_type()` | 获取指定类型的 2FA 配置 |
| `delete_all_by_user()` | 删除用户所有 2FA 配置 |
| `migrate_u2f_to_webauthn()` | U2F 迁移到 WebAuthn |
| `migrate_credential_to_passkey()` | 凭证格式迁移 |

---

## 关键安全要点

1. **防重放**: TOTP 记录 `last_used` 防止代码重用
2. **常量时间比较**: 使用 `crypto::ct_eq()` 避免时序攻击
3. **时间漂移**: TOTP 支持 ±1 时间步容忍
4. **备份状态**: WebAuthn 支持检测可备份设备
5. **组织策略**: 禁用所有 2FA 会触发组织成员撤销
6. **冷却期**: Protected Actions 有 30 秒请求冷却
7. **尝试限制**: Protected Actions 有尝试次数限制
8. **未完成登录监控**: 超时未完成的 2FA 登录会发送邮件告警

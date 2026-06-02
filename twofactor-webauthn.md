# 双因素认证 (2FA) 和 WebAuthn 代码分析

## 目录
- [一、用户验证策略 (UserVerificationPolicy) 何时生效](#一用户验证策略-userverificationpolicy-何时生效)
- [二、恢复码在登录流程中的具体路径](#二恢复码在登录流程中的具体路径)
- [三、通用禁用与专用禁用接口的职责分工](#三通用禁用与专用禁用接口的职责分工)
- [附录 A：模块总览](#附录-a模块总览)
- [附录 B：TwoFactorType 枚举](#附录-btwofactortype-枚举)
- [附录 C：WebAuthn 完整流程](#附录-cwebauthn-完整流程)
- [附录 D：Authenticator (TOTP) 完整流程](#附录-dauthenticator-totp-完整流程)
- [附录 E：数据库模型](#附录-e数据库模型)

---

## 一、用户验证策略 (UserVerificationPolicy) 何时生效

### 1.1 为什么需要关注这个策略

WebAuthn 协议的 `userVerification` 参数决定了认证器是否需要验证用户身份（例如指纹、面部识别、PIN）。Vaultwarden 将 WebAuthn 作为 **2FA（第二因素）** 而非 **唯一认证（passkey as first factor）** 使用，因此刻意将策略设为 `Discouraged`，意为"不需要用户验证，只需证明你持有该设备"。

### 1.2 策略在注册阶段生效的两个位置

注册流程分为"生成挑战"和"激活"两步。`Discouraged` 策略在这两步中分别作用于不同对象：

#### 位置 1：返回给客户端的挑战对象（客户端可见）

在 [generate_webauthn_challenge()](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/webauthn.rs#L131-L170) 中：

```
WEBAUTHN.start_passkey_registration()   ← 库默认策略可能是 Preferred
         ↓
第 162-164 行：修改返回给客户端的 challenge
    asc.user_verification = UserVerificationPolicy::Discouraged_DO_NOT_USE;
         ↓
客户端浏览器收到 challenge → 创建 PublicKeyCredential 时
    authenticatorSelection.userVerification = "discouraged"
         ↓
认证器行为：只做"证明持有"，不做指纹/PIN
```

**生效时机**：客户端调用 `navigator.credentials.create()` 时，浏览器根据 `authenticatorSelection.userVerification` 字段决定是否要求用户进行生物识别。

#### 位置 2：保存在数据库中的服务端状态（验证时使用）

在同函数的第 152-154 行：

```
第 152-154 行：修改保存在 DB 中的 state
    state["rs"]["policy"] = "discouraged"
    state["rs"]["extensions"].clear()
         ↓
activate_webauthn() 读取此 state
         ↓
WEBAUTHN.finish_passkey_registration(&response, &state)
    ← 库根据 state 中的 policy 判断注册是否合法
```

**生效时机**：当 [activate_webauthn()](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/webauthn.rs#L255-L304) 调用 `finish_passkey_registration()` 时，webauthn-rs 库从反序列化的 `PasskeyRegistration` state 中读取 policy，校验客户端返回的响应是否与期望的策略一致。

> **关键点**：两处修改缺一不可。位置 1 控制客户端行为，位置 2 控制服务端校验。如果只改了客户端返回的 challenge 而不改 state，注册时服务端会因为策略不匹配而拒绝。

### 1.3 策略在登录阶段生效的两个位置

登录时同样有两处修改，逻辑与注册完全对称：

#### 位置 1：返回给客户端的挑战对象

在 [generate_webauthn_login()](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/webauthn.rs#L378-L416) 的第 398 行：

```rust
response.public_key.user_verification = UserVerificationPolicy::Discouraged_DO_NOT_USE;
```

**生效时机**：客户端调用 `navigator.credentials.get()` 时，浏览器根据此字段决定是否要求用户进行生物识别。

#### 位置 2：保存在数据库中的服务端状态

在同函数的第 391-396 行：

```rust
state["ast"]["policy"] = Value::String("discouraged".to_owned());
let app_id = format!("{}/app-id.json", CONFIG.domain());
state["ast"]["appid"] = Value::String(app_id.clone());
```

**生效时机**：当 [validate_webauthn_login()](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/webauthn.rs#L418-L465) 调用 `finish_passkey_authentication()` 时，库从 `PasskeyAuthentication` state 中读取 policy 进行校验。

### 1.4 策略生效的完整时序图

```
┌─────────── 注册阶段 ───────────┐

客户端                      服务端
  │                           │
  │  POST /get-webauthn-      │
  │  challenge                │
  │ ─────────────────────────>│
  │                           │ start_passkey_registration()
  │                           │   → 生成 challenge + state
  │                           │
  │                           │ 修改 challenge.public_key:
  │                           │   user_verification = Discouraged  ← 位置1
  │                           │ 修改 state["rs"]["policy"]:
  │                           │   = "discouraged"                  ← 位置2
  │                           │
  │                           │ 保存 state 到 DB
  │                           │   (atype=WebauthnRegisterChallenge)
  │                           │
  │  ← challenge (Discouraged)│
  │                           │
  │ navigator.credentials     │
  │   .create({               │
  │     userVerification:     │
  │       "discouraged"  ←── 客户端按此策略执行
  │   })                      │
  │                           │
  │  POST /webauthn           │
  │  (deviceResponse)         │
  │ ─────────────────────────>│
  │                           │ 从 DB 读取 state
  │                           │ finish_passkey_registration(
  │                           │   response, &state)
  │                           │   → 库按 state 中 "discouraged"
  │                           │     策略校验响应  ←── 位置2生效
  │                           │

┌─────────── 登录阶段 ───────────┐

客户端                      服务端
  │                           │
  │  POST /connect/token      │
  │  (首次登录, 无2FA token)   │
  │ ─────────────────────────>│
  │                           │ twofactor_auth() 发现需要2FA
  │                           │ json_err_twofactor()
  │                           │   → generate_webauthn_login()
  │                           │
  │                           │ start_passkey_authentication()
  │                           │   → 生成 challenge + state
  │                           │
  │                           │ 修改 response.public_key:
  │                           │   user_verification = Discouraged  ← 位置1
  │                           │ 修改 state["ast"]["policy"]:
  │                           │   = "discouraged"                  ← 位置2
  │                           │   state["ast"]["appid"] = ...      ← U2F兼容
  │                           │
  │                           │ 保存 state 到 DB
  │                           │   (atype=WebauthnLoginChallenge)
  │                           │
  │  ← 400 + challenge        │
  │     (Discouraged)         │
  │                           │
  │ navigator.credentials     │
  │   .get({                  │
  │     userVerification:     │
  │       "discouraged"  ←── 客户端按此策略执行
  │   })                      │
  │                           │
  │  POST /connect/token      │
  │  (two_factor_token=签名)  │
  │ ─────────────────────────>│
  │                           │ validate_webauthn_login()
  │                           │   从 DB 读取 state
  │                           │   finish_passkey_authentication(
  │                           │     response, &state)
  │                           │   → 库按 state 中 "discouraged"
  │                           │     策略校验响应  ←── 位置2生效
  │                           │
```

### 1.5 为什么 webauthn-rs 的默认策略不合适

`start_passkey_registration()` 和 `start_passkey_authentication()` 作为通用的 passkey API，默认 `userVerification` 为 `Preferred`。但 Vaultwarden 使用 WebAuthn 作为 **security key**（第二因素），代码注释（第 159-160 行）明确说明了这一点：

> *Because for this flow we abuse the passkeys as 2FA, and use it more like a securitykey we need to modify some of the default settings defined by `start_passkey_registration()`.*

因此必须在两个位置（客户端挑战 + 服务端状态）都手动降级为 `Discouraged`。

---

## 二、恢复码在登录流程中的具体路径

### 2.1 恢复码的本质：不在 TwoFactor 表中

恢复码与 TOTP 密钥、WebAuthn 凭证不同，它**不是** `TwoFactorType::RecoveryCode` 对应的一条数据库记录。恢复码存储在 [User 表的 `totp_recover` 字段](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/db/models/user.rs#L54) 中。

`TwoFactorType::RecoveryCode`（值=8）只在登录验证的 `match` 分支中作为逻辑标签存在，数据库里**永远不会**出现 `atype=8` 的 `TwoFactor` 行。

这意味着：
- [TwoFactor::find_by_user()](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/db/models/two_factor.rs#L129-L138) 查询 `atype < 1000`，恢复码不在其结果中
- 恢复码不会出现在 `twofactor_ids`（可用提供者列表）中
- 恢复码是**隐式可用**的——只要用户有 `totp_recover` 值且该值非空

### 2.2 恢复码的生成路径

恢复码在**首次启用任何 2FA 方式**时生成。代码路径：

```
activate_authenticator()         ← POST /two-factor/authenticator
    └→ generate_recover_code()   ← mod.rs#L121

activate_webauthn()              ← POST /two-factor/webauthn
    └→ generate_recover_code()   ← mod.rs#L121
```

[generate_recover_code()](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/mod.rs#L121-L127) 的逻辑：

```rust
async fn generate_recover_code(user: &mut User, conn: &DbConn) {
    if user.totp_recover.is_none() {          // 仅在尚未生成时生成
        let totp_recover = crypto::encode_random_bytes::<20>(&BASE32);
        user.totp_recover = Some(totp_recover);
        user.save(conn).await.ok();
    }
}
```

**关键**：`if user.totp_recover.is_none()` — 恢复码只在首次启用 2FA 时生成一次。后续启用其他 2FA 方式时不会覆盖已有的恢复码。

### 2.3 恢复码在登录验证中的完整路径

入口：[twofactor_auth()](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/identity.rs#L761-L893)

```
用户提交登录 (POST /connect/token)
  │
  ├─ twofactor_auth() 开始
  │
  ├─ TwoFactor::find_by_user() → 获取所有 TwoFactor 行
  │   注意：恢复码不在其中，因为它存在 User 表
  │
  ├─ 构建 twofactor_ids 列表
  │   = [Authenticator=0, Webauthn=7, ...]
  │   恢复码(type=8)不在此列表中
  │
  ├─ selected_id = data.two_factor_provider
  │   ← 客户端传来的选择
  │   ← 如果是 None，默认取 twofactor_ids[0]
  │
  ├─ 特殊检查（第 791-798 行）：
  │   if selected_id 不是 Remember 也不是 RecoveryCode
  │      && 不在 twofactor_ids 中
  │      → 报错 "Invalid two factor provider"
  │
  │   *** 关键：RecoveryCode 被跳过这个检查 ***
  │   *** 即使 RecoveryCode 不在 twofactor_ids 中 ***
  │   *** 客户端也可以选择它                  ***
  │
  ├─ match TwoFactorType::from_i32(selected_id)
  │   │
  │   ├─ TwoFactorType::RecoveryCode →    ← 第 860-875 行
  │   │   │
  │   │   ├─ user.check_valid_recovery_code(twofactor_code)
  │   │   │   → User 模型方法（user.rs#L169）
  │   │   │   → crypto::ct_eq(recovery_code, totp_recover.to_lowercase())
  │   │   │   → 常量时间比较，**用户必须输入小写**
  │   │   │
  │   │   ├─ TwoFactor::delete_all_by_user()    ← 清除所有 2FA！
  │   │   ├─ enforce_2fa_policy()                ← 可能撤销组织成员资格
  │   │   ├─ log_user_event(UserRecovered2fa)    ← 记录恢复事件
  │   │   ├─ user.totp_recover = None            ← 清除恢复码本身
  │   │   └─ user.save()                          ← 保存用户
  │   │
  │   └─ 其他类型 → 正常验证
  │
  └─ TwoFactorIncomplete::mark_complete() → 标记2FA完成
```

### 2.4 恢复码的大小写真实要求：用户必须输入小写

这是代码中一个容易被误解的细节。让我们追踪三个关键位置的大小写处理：

#### 位置 1：生成时 [mod.rs#L123](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/mod.rs#L123)

```rust
let totp_recover = crypto::encode_random_bytes::<20>(&BASE32);
```

`data-encoding` 库的 `BASE32` 遵循 RFC4648 标准，**默认输出大写字母**（A-Z, 2-7）。

#### 位置 2：存储时 [mod.rs#L124](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/mod.rs#L124)

```rust
user.totp_recover = Some(totp_recover);  // 大写原样存储
```

数据库 `users.totp_recover` 字段中存储的是**大写**。

#### 位置 3：校验时 [user.rs#L171](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/db/models/user.rs#L171)

```rust
crypto::ct_eq(recovery_code, totp_recover.to_lowercase())
```

**关键不对称**：
- 左边 `recovery_code`：用户原始输入，**未做大小写转换**
- 右边 `totp_recover.to_lowercase()`：数据库值被转成**小写**
- `ct_eq` 是 [subtle](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/crypto.rs#L112-L115) 库的**字节级精确比较**

**结论**：
| 用户输入 | 数据库存储（大写） | 比较的右侧（转小写） | 匹配结果 |
|---------|------------------|-------------------|---------|
| `ABCDE...`（大写） | `ABCDE...` | `abcde...` | ❌ 不匹配 |
| `abcde...`（小写） | `ABCDE...` | `abcde...` | ✅ 匹配 |
| `aBcDe...`（混合） | `ABCDE...` | `abcde...` | ❌ 不匹配 |

**用户必须输入全小写的恢复码才能通过校验。** 这与 TOTP 密钥的处理方式形成对比——TOTP 在 [authenticator.rs#L83](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/authenticator.rs#L83) 存储时会 `key.to_uppercase()` 统一转大写。

### 2.5 get-recover 接口：显示大写但输入要小写的矛盾

[get_recover()](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/mod.rs#L108-L119) 是用户查看恢复码的接口：

```rust
#[post("/two-factor/get-recover", data = "<data>")]
async fn get_recover(...) -> JsonResult {
    data.validate(&user, true, &conn).await?;  // 需要密码或 OTP 验证

    Ok(Json(json!({
        "code": user.totp_recover,          // ← 直接返回数据库中的大写
        "object": "twoFactorRecover"
    })))
}
```

**完整链条**：

```
用户启用 2FA
  ↓
generate_recover_code()
  ↓ BASE32 编码（大写）
user.totp_recover = Some("ABCDEFG...")  ← 大写存储
  ↓
用户 POST /two-factor/get-recover 查看恢复码
  ↓
返回 {"code": "ABCDEFG..."}  ← 显示给用户的是大写
  ↓
用户丢失设备，需要用恢复码登录
  ↓
用户输入 "ABCDEFG..."（大写）→ ❌ 校验失败
     （因为 check_valid_recovery_code 只把右边转小写）
  ↓
用户输入 "abcdefg..."（小写）→ ✅ 校验成功
```

**建议修复**：在 `get_recover()` 返回时转成小写，或者在 `check_valid_recovery_code()` 中将两边都转小写：

```rust
// 方案 A：返回小写（对用户友好）
"code": user.totp_recover.as_ref().map(|s| s.to_lowercase()),

// 方案 B：两边都转小写（更健壮）
crypto::ct_eq(
    recovery_code.to_lowercase().as_bytes(),
    totp_recover.to_lowercase().as_bytes()
)
```

---

### 2.6 为什么 2FA 错误响应中没有 RecoveryCode

当登录需要 2FA 时，服务端返回的 `TwoFactorProviders` 列表中**永远不会包含 RecoveryCode(type=8)**。这是有意为之的设计，有三层原因：

#### 第一层：RecoveryCode 不在 TwoFactor 表中

`twofactor_ids` 的构建在 [identity.rs#L779-L785](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/identity.rs#L779-L785)：

```rust
let twofactor_ids: Vec<_> = twofactors  // twofactors = TwoFactor::find_by_user()
    .iter()
    .filter_map(|tf| {
        let provider_type = TwoFactorType::from_i32(tf.atype)?;
        (tf.enabled && is_twofactor_provider_usable(&provider_type, Some(&tf.data)))
            .then_some(tf.atype)
    })
    .collect();
```

RecoveryCode 存在 `User.totp_recover` 字段，不在 `twofactor` 表中，所以自然不会出现在 `twofactor_ids` 里。

#### 第二层：json_err_twofactor 的 match 分支跳过

即使强行把 8 塞进 `twofactor_ids`，[json_err_twofactor()](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/identity.rs#L991-L1005) 的 match 也会跳过它：

```rust
match TwoFactorType::from_i32(*provider) {
    Some(TwoFactorType::Webauthn) => { ... 生成挑战 ... }
    Some(TwoFactorType::Duo) => { ... 生成签名 ... }
    Some(TwoFactorType::YubiKey) => { ... 返回元数据 ... }
    Some(TwoFactorType::Email) => { ... 发送邮件 ... }
    Some(
        TwoFactorType::Authenticator
        | TwoFactorType::RecoveryCode   // ← 在这里，什么也不做
        | TwoFactorType::Remember
        ...
    ) => { /* Nothing special to do */ }
}
```

#### 第三层：设计意图——恢复码是"紧急出口"

恢复码不应该作为常规的 2FA 选项出现在登录界面上，因为：
1. **不鼓励使用**：恢复码使用后会清除所有 2FA 配置，是破坏性操作
2. **隐藏更安全**：攻击者不知道还有恢复码这个选项
3. **需要用户主动发现**：用户只有在真正紧急（丢失所有设备）时才会去查找恢复码的用法

---

### 2.7 登录时 provider=8 + token 的完整配合流程

RecoveryCode 虽然不在提供者列表中，但客户端可以**主动发送** `two_factor_provider=8` 来使用恢复码。完整流程如下：

#### 场景：用户丢失了所有 2FA 设备，只有恢复码

```
第 1 次 POST /connect/token
  Body: {
    username: "user@example.com",
    password: "xxx",
    // 没有 two_factor_provider 和 two_factor_token
  }
  ↓
服务端发现需要 2FA
  ↓
twofactor_ids = [0, 7]  ← TOTP + WebAuthn，没有 8
  ↓
返回 400 + {
  "error": "invalid_grant",
  "TwoFactorProviders": ["0", "7"],   // ← 不包含 8
  "TwoFactorProviders2": { "0": null, "7": {...挑战...} }
}
  ↓
用户知道可以用恢复码，客户端选择"使用恢复码"选项
  ↓
第 2 次 POST /connect/token
  Body: {
    username: "user@example.com",
    password: "xxx",
    two_factor_provider: "8",    // ← 主动指定 type=8
    two_factor_token: "abcdefg..."  // ← 恢复码（必须小写！）
  }
  ↓
进入 twofactor_auth()
  ↓
selected_id = 8  ← 从请求中读取
  ↓
第 792-798 行的白名单检查：
  if ![Remember, RecoveryCode].contains(&selected_id)
     && !twofactor_ids.contains(&selected_id)
  → RecoveryCode 在白名单中，跳过检查
  ↓
match TwoFactorType::from_i32(8)
  ↓
TwoFactorType::RecoveryCode 分支 [identity.rs#L860-L875]
  ↓
user.check_valid_recovery_code("abcdefg...")
  → ct_eq("abcdefg...", "ABCDEFG...".to_lowercase())
  → ct_eq("abcdefg...", "abcdefg...") → ✅ 通过
  ↓
TwoFactor::delete_all_by_user()  ← 清除所有 2FA！
enforce_2fa_policy()             ← 可能撤销组织成员
user.totp_recover = None         ← 清除恢复码本身
user.save()
  ↓
登录成功！（但用户下次需要重新设置 2FA）
```

**关键点**：
- 第 792 行的白名单检查是 RecoveryCode 的"后门"
- `selected_twofactor`（第 808 行）对 RecoveryCode 是 `None`，但这个分支不需要它
- 恢复码验证成功后，**所有 2FA 配置被清空**，用户变成"无 2FA 状态"

---

### 2.8 恢复码的设计意图

| 特性 | 说明 |
|------|------|
| 不在 TwoFactor 表中 | 与具体 2FA 方式解耦，是用户级别的"紧急出口" |
| 只生成一次 | 首次启用任何 2FA 时生成，后续不覆盖 |
| 大小写要求 | 用户必须输入**全小写** |
| 使用后果严重 | 删除**所有** 2FA 配置 + 清除恢复码本身 + 可能触发组织策略 |
| 特殊通道进入 | 绕过 `twofactor_ids.contains()` 检查，客户端可主动选择 |

---

## 三、通用禁用与专用禁用接口的职责分工

### 3.1 两个禁用接口总览

| 维度 | 通用禁用 | 专用禁用 |
|------|---------|---------|
| 端点 | `POST /two-factor/disable` | `DELETE /two-factor/webauthn` |
| 定义位置 | [mod.rs#L137-L167](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/mod.rs#L137-L167) | [webauthn.rs#L318-L365](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/webauthn.rs#L318-L365) |
| 认证方式 | `PasswordOrOtpData`（密码或OTP） | `master_password_hash`（仅密码） |
| 粒度 | 整个 2FA 类型（一条 TwoFactor 记录） | 单个密钥（WebauthnRegistration 列表中的一项） |
| 适用范围 | 所有 2FA 类型 | 仅 WebAuthn |
| 组织策略 | ✅ 会检查 `enforce_2fa_policy` | ❌ 不检查 |
| U2F 迁移清理 | ❌ 不处理 | ✅ 同步删除 U2F 记录 |

### 3.2 通用禁用：`POST /two-factor/disable`

代码路径：

```rust
// mod.rs#L137-L167
async fn disable_twofactor(data: Json<DisableTwoFactorData>, headers: Headers, conn: DbConn) -> JsonResult {
    // 1. 验证身份：密码 或 OTP 都可以
    PasswordOrOtpData {
        master_password_hash: data.master_password_hash,
        otp: data.otp,
    }
    .validate(&user, true, &conn).await?;

    // 2. 按 type 参数定位整条 TwoFactor 记录
    let type_ = data.r#type.into_i32()?;
    if let Some(twofactor) = TwoFactor::find_by_user_and_type(&user.uuid, type_, &conn).await {
        // 3. 直接删除整条记录
        twofactor.delete(&conn).await?;
        log_user_event(EventType::UserDisabled2fa as i32, ...);
    }

    // 4. 检查是否还有其他 2FA，决定是否执行组织策略
    if TwoFactor::find_by_user(&user.uuid, &conn).await.is_empty() {
        enforce_2fa_policy(&user, ...).await?;
    }

    Ok(Json(json!({ "enabled": false, "type": type_, ... })))
}
```

**行为特点**：
- 删除 `atype=7` 的整条 `TwoFactor` 记录 → 用户所有 WebAuthn 密钥**一次性全部移除**
- 如果用户因此没有任何 2FA，触发 `enforce_2fa_policy()` 检查组织策略
- 通用接口对 WebAuthn 无特殊处理，不会清理 U2F 残留

### 3.3 专用禁用：`DELETE /two-factor/webauthn`

代码路径：

```rust
// webauthn.rs#L318-L365
async fn delete_webauthn(data: Json<DeleteU2FData>, headers: Headers, conn: DbConn) -> JsonResult {
    // 1. 验证身份：仅密码
    if !headers.user.check_valid_password(&data.master_password_hash) {
        err!("Invalid password");
    }

    // 2. 读取 WebAuthn 的 TwoFactor 记录
    let Some(mut tf) = TwoFactor::find_by_user_and_type(
        &headers.user.uuid, TwoFactorType::Webauthn as i32, &conn
    ).await else {
        err!("Webauthn data not found!")
    };

    // 3. 从 JSON 数组中移除指定 ID 的密钥
    let mut data: Vec<WebauthnRegistration> = serde_json::from_str(&tf.data)?;
    let Some(item_pos) = data.iter().position(|r| r.id == id) else {
        err!("Webauthn entry not found")
    };
    let removed_item = data.remove(item_pos);

    // 4. 保存更新后的数组（可能还有剩余密钥）
    tf.data = serde_json::to_string(&data)?;
    tf.save(&conn).await?;
    drop(tf);

    // 5. U2F 迁移清理：如果被删的密钥是从 U2F 迁移来的，
    //    同步删除 U2F 表中的对应记录
    if let Some(mut u2f) = TwoFactor::find_by_user_and_type(
        &headers.user.uuid, TwoFactorType::U2f as i32, &conn
    ).await {
        let mut data: Vec<U2FRegistration> = serde_json::from_str(&u2f.data)?;
        data.retain(|r| r.reg.key_handle != removed_item.credential.cred_id().as_slice());
        u2f.data = serde_json::to_string(&data)?;
        u2f.save(&conn).await?;
    }

    // 6. 返回结果中 "enabled": true → 因为可能还有剩余密钥
    Ok(Json(json!({ "enabled": true, "keys": keys_json, ... })))
}
```

**行为特点**：
- 精确移除**单个**密钥（按 `id` 定位），保留其他密钥
- 不会删除 `TwoFactor` 记录本身，只修改 `data` JSON 字段
- 处理 U2F 迁移残留：通过 `key_handle == cred_id()` 匹配清理
- **不检查** `enforce_2fa_policy()` —— 即使删到零个密钥也不触发组织策略
- 返回 `"enabled": true`（因为记录还在，即使数组为空）

### 3.4 两个接口的职责划分

```
客户端场景                         应调用
─────────────────────────────────────────────
用户想删除某个 WebAuthn 密钥       DELETE /two-factor/webauthn
（保留其他密钥）                    ← 专用禁用

用户想彻底关闭 WebAuthn 2FA        POST /two-factor/disable {type:7}
（移除所有密钥）                    ← 通用禁用

用户想关闭 TOTP                    POST /two-factor/disable {type:0}
                                   ← 只有通用禁用可用

用户想关闭 Email 2FA              POST /two-factor/disable {type:1}
                                   ← 只有通用禁用可用
```

### 3.5 删除最后一个 WebAuthn 密钥后的完整路径

当 `DELETE /two-factor/webauthn` 删到最后一个密钥时，数据库状态：
- `TwoFactor(atype=7)` 记录仍然存在
- `enabled` 字段仍为 `true`
- `data` 字段变为 `[]`（空 JSON 数组）
- **不会**调用 `enforce_2fa_policy()` 检查组织策略
- 返回 `"enabled": true`

接下来的三处代码路径会各自表现出不同的行为：

#### 路径 1：设置页面读取（GET /two-factor）

入口：[get_twofactor()](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/mod.rs#L90-L106)

```
GET /two-factor
  ↓
TwoFactor::find_by_user() → 包含 atype=7 记录
  ↓
is_twofactor_provider_usable(Webauthn, Some("[]"))
  └→ 检查 CONFIG.is_webauthn_2fa_supported()  ← 仅检查 DOMAIN 配置
      不检查 tf.data 是否为空
  └→ 返回 true
  ↓
TwoFactor::to_json_provider(tf)
  └→ 返回 {"enabled": true, "type": 7, ...}
  ↓
前端认为 WebAuthn 已启用，点击进入详情
  ↓
POST /two-factor/get-webauthn
  └→ get_webauthn_registrations()
      └→ 返回 (enabled=true, registrations=[])  ← 空数组
  └→ 返回 {"enabled": true, "keys": [], ...}
  ↓
前端显示 "已启用，0 个密钥"
```

关键点：[is_twofactor_provider_usable()](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/mod.rs#L39-L69) 的第 59 行 **不检查** `data` 是否为空，只检查全局配置。

#### 路径 2：登录时的提供者列表构建

入口：[twofactor_auth()](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/identity.rs#L779-L785)

```
POST /connect/token（首次登录）
  ↓
TwoFactor::find_by_user() → 包含 atype=7 记录
  ↓
构建 twofactor_ids:
  for tf in twofactors:
    if tf.enabled && is_twofactor_provider_usable(atype, &tf.data)
      → Webauthn: 条件成立（enabled=true && 配置有效）
      → 加入 twofactor_ids
  ↓
twofactor_ids = [7]  ← WebAuthn 在可用列表中
  ↓
json_err_twofactor([7], ...)
  └→ 遍历 providers，为每个类型生成 TwoFactorProviders2
  └→ 遇到 type=7 时调用 generate_webauthn_login()
      ↓
      generate_webauthn_login():
        creds = get_webauthn_registrations().1 → []（空数组）
        if creds.is_empty() {
          err!("No Webauthn devices registered")  ← 这里报错！
        }
  ↓
整个登录流程返回 500 错误
```

**后果**：用户无法登录。因为：
1. WebAuthn 被认定为"可用"并加入 `twofactor_ids`
2. 但生成挑战时发现无密钥，抛出错误
3. `json_err_twofactor()` 提前失败，不返回 2FA 提供者列表给客户端

#### 路径 3：如果有其他 2FA 方式（例如同时有 TOTP）

```
twofactor_ids = [0, 7]  ← TOTP + WebAuthn
  ↓
json_err_twofactor([0, 7], ...)
  ↓
遍历 [0, 7]:
  type=0 (Authenticator): 无特殊处理，返回 null 给客户端
  type=7 (Webauthn): 调用 generate_webauthn_login() → 报错
  ↓
整个流程失败
```

**即使有其他可用的 2FA 方式，WebAuthn 的空密钥问题也会导致整个登录失败。**

#### 修复点建议

在 [is_twofactor_provider_usable()](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/mod.rs#L39-L69) 中，WebAuthn 的判断应额外检查：

```rust
TwoFactorType::Webauthn =>
    CONFIG.is_webauthn_2fa_supported()
    && provider_data.is_some_and(|d| d != "[]"),  // ← 新增：排除空数组
```

或者在 [get_webauthn_registrations()](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/webauthn.rs#L367-L376) 返回 `(false, [])` 当数组为空时。

### 3.6 Authenticator 的专用禁用对比

作为参照，[disable_authenticator()](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/authenticator.rs#L191-L219) 的行为：

```rust
async fn disable_authenticator(data: Json<DisableAuthenticatorData>, ...) -> JsonResult {
    // 1. 仅密码验证
    if !user.check_valid_password(&data.master_password_hash) { ... }

    // 2. 额外校验：密钥必须匹配
    if twofactor.data == data.key {          ← TOTP 密钥要正确
        twofactor.delete(&conn).await?;
    } else {
        err!("TOTP key ... does not match");  ← 密钥不对就不让删
    }

    // 3. 检查组织策略
    if TwoFactor::find_by_user(...).is_empty() {
        enforce_2fa_policy(...).await?;
    }
}
```

与通用禁用的差异：额外要求提供正确的 TOTP 密钥才能禁用。但**会**检查组织策略。

### 3.7 三种禁用接口对比总结

| 维度 | 通用禁用 | WebAuthn 专用 | Authenticator 专用 |
|------|---------|-------------|-------------------|
| 粒度 | 整个类型 | 单个密钥 | 整个类型 |
| 认证 | 密码或OTP | 仅密码 | 仅密码+密钥匹配 |
| 组织策略检查 | ✅ | ❌ | ✅ |
| U2F 清理 | ❌ | ✅ | N/A |
| 删到空时的返回 | `enabled: false` | `enabled: true` | `enabled: false` |

---

## 附录 A：模块总览

### 文件位置

| 模块 | 路径 |
|------|------|
| 核心调度 | [src/api/core/two_factor/mod.rs](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/mod.rs) |
| WebAuthn | [src/api/core/two_factor/webauthn.rs](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/webauthn.rs) |
| Authenticator | [src/api/core/two_factor/authenticator.rs](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/authenticator.rs) |
| Protected Actions | [src/api/core/two_factor/protected_actions.rs](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/protected_actions.rs) |
| 数据库模型 | [src/db/models/two_factor.rs](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/db/models/two_factor.rs) |
| 登录验证 | [src/api/identity.rs](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/identity.rs) |
| 认证/JWT | [src/auth.rs](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/auth.rs) |

### 支持的 2FA 类型

| 类型值 | 名称 | 说明 |
|-------|------|------|
| 0 | Authenticator | TOTP 验证器 |
| 1 | Email | 邮件验证 |
| 2 | Duo | Duo 安全 |
| 3 | YubiKey | YubiKey 硬件密钥 |
| 4 | U2f | 旧版 U2F（已迁移至 WebAuthn） |
| 5 | Remember | 记住设备 |
| 6 | OrganizationDuo | 组织级 Duo |
| 7 | Webauthn | WebAuthn/FIDO2 |
| 8 | RecoveryCode | 恢复码（仅逻辑标签，无 DB 记录） |

---

## 附录 B：TwoFactorType 枚举

定义在 [src/db/models/two_factor.rs#L27-L47](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/db/models/two_factor.rs#L27-L47)：

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

**分区规则**：`find_by_user()` 过滤 `atype < 1000`，因此内部挑战类型不会暴露给客户端。

---

## 附录 C：WebAuthn 完整流程

### C.1 启用流程

```
1. POST /two-factor/get-webauthn
   验证密码/OTP → 返回已注册密钥列表

2. POST /two-factor/get-webauthn-challenge
   验证密码/OTP → 生成注册挑战
   修改策略为 Discouraged（客户端+服务端）
   保存 state 到 DB (WebauthnRegisterChallenge)
   返回挑战给客户端

3. POST /two-factor/webauthn (或 PUT)
   验证密码/OTP → 读取并删除 state
   finish_passkey_registration() → 验证设备响应
   新凭证加入列表 → 保存到 DB (Webauthn)
   generate_recover_code() → 生成恢复码（如未生成）
   记录事件
```

### C.2 登录流程

```
1. 首次 POST /connect/token（无 2FA token）
   → twofactor_auth() 发现需要 2FA
   → json_err_twofactor() 返回 400 + 提供者列表
   → 如果有 WebAuthn，调用 generate_webauthn_login()
     生成登录挑战，修改策略为 Discouraged
     保存 state 到 DB (WebauthnLoginChallenge)

2. 再次 POST /connect/token（带 two_factor_token）
   → twofactor_auth() 匹配 TwoFactorType::Webauthn
   → validate_webauthn_login()
     读取并删除 state
     检查并更新 backup_eligible 标志
     finish_passkey_authentication() → 验证签名
     更新凭证计数器
```

### C.3 删除单个密钥

```
DELETE /two-factor/webauthn
验证密码 → 从数组中移除指定 ID 的密钥
同步清理 U2F 记录（如适用）
```

---

## 附录 D：Authenticator (TOTP) 完整流程

### D.1 启用流程

```
1. POST /two-factor/get-authenticator
   验证密码/OTP → 返回 TOTP 密钥（或生成新的）

2. POST /two-factor/authenticator (或 PUT)
   验证密码/OTP → 校验密钥格式（Base32, 20字节）
   validate_totp_code() → 验证用户输入的 TOTP 码
   generate_recover_code() → 生成恢复码
   记录事件
```

### D.2 TOTP 验证算法

[validate_totp_code()](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/api/core/two_factor/authenticator.rs#L115-L181)：

- 算法：SHA1，30 秒步长，6 位数字
- 时间漂移容忍：默认 ±1 步（可通过 `authenticator_disable_time_drift` 配置为 0）
- 防重放：`last_used` 时间步，拒绝已用过的代码
- 安全：记录服务器时间和客户端 IP

---

## 附录 E：数据库模型

### TwoFactor 表

定义在 [src/db/models/two_factor.rs#L15-L25](file:///d:/fz/0601/solo-dogfeeding/code/4-vaultwarden/src/db/models/two_factor.rs#L15-L25)：

| 字段 | 类型 | 说明 |
|------|------|------|
| uuid | TwoFactorId | 主键 |
| user_uuid | UserId | 用户 ID |
| atype | i32 | 类型 (TwoFactorType) |
| enabled | bool | 是否启用 |
| data | String | JSON 格式的配置数据 |
| last_used | i64 | 上次使用时间 (TOTP 为时间步) |

**唯一约束**：`(user_uuid, atype)` — 每个用户每种类型只能有一条记录。

### 关键方法

| 方法 | 说明 |
|------|------|
| `find_by_user()` | 获取用户所有 2FA 配置（过滤 `atype < 1000`） |
| `find_by_user_and_type()` | 获取指定类型的 2FA 配置 |
| `delete_all_by_user()` | 删除用户所有 2FA 配置 |
| `save()` | upsert 语义，PostgreSQL 需先删除旧记录 |

### 恢复码存储

恢复码存储在 `users` 表的 `totp_recover` 字段（`Option<String>`），不在 `twofactor` 表中。

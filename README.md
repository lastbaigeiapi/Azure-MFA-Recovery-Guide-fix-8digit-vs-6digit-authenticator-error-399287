# Azure 个人账户 MFA 验证码位数不匹配导致无法登录的解决方案

## 问题描述

使用个人微软账户（如 Gmail）注册 Azure 后，无法登录 Azure Portal。具体表现为：

- Azure Portal 要求输入 **6 位**验证码
- Microsoft Authenticator 生成的却是 **8 位**验证码
- 选择推送验证时，Authenticator 不弹出验证通知
- 形成死循环：登不上 Azure → 改不了 MFA → 登不上 Azure

## 根本原因

个人微软账户（MSA）注册 Azure 时，系统会自动创建一个 Azure AD 租户。登录 Azure Portal 走的是**工作/学校账户**通道，而 Microsoft Authenticator 中绑定的是**个人账户**。

| | 个人账户 (MSA) | 工作/学校账户 (Azure AD) |
|--|--|--|
| TOTP 验证码 | **8 位** | **6 位** |
| 推送通知 | 个人账户的推送 | Azure AD 的推送（不会弹出） |
| 登录场景 | Outlook、Xbox、OneDrive | **Azure Portal**、Microsoft 365 |

这就是为什么验证码位数对不上、推送也收不到的原因——Authenticator 注册的身份和 Azure 要求验证的身份不是同一个。

## 前提条件

- Azure CLI（`az`）仍处于登录状态
- 账户拥有 Global Administrator 角色

## 解决步骤

### 第一步：创建临时应用获取 Graph API 权限

Azure CLI 默认令牌没有管理认证方法的权限，需要创建临时应用并授权。

```bash
# 创建应用和服务主体
APP_ID=$(az ad app create --display-name "MFA-Reset-Temp" --query appId -o tsv)
SP_ID=$(az ad sp create --id $APP_ID --query id -o tsv)
APP_SECRET=$(az ad app credential reset --id $APP_ID --query password -o tsv)

# 获取 Microsoft Graph 服务主体 ID
GRAPH_SP_ID=$(az ad sp list \
  --filter "appId eq '00000003-0000-0000-c000-000000000000'" \
  --query '[0].id' -o tsv)

# 授予 UserAuthenticationMethod.ReadWrite.All 权限
az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/servicePrincipals/$SP_ID/appRoleAssignments" \
  --body "{
    \"principalId\":\"$SP_ID\",
    \"resourceId\":\"$GRAPH_SP_ID\",
    \"appRoleId\":\"50483e42-d915-4231-9639-7fdb7fd190e5\"
  }"
```

### 第二步：获取令牌并删除旧的 MFA

```bash
TENANT_ID="<你的租户ID>"
USER_ID="<你的用户Object ID>"

# 获取应用令牌
TOKEN=$(curl -s -X POST \
  "https://login.microsoftonline.com/$TENANT_ID/oauth2/v2.0/token" \
  -d "client_id=$APP_ID&client_secret=$APP_SECRET&scope=https://graph.microsoft.com/.default&grant_type=client_credentials" \
  | python3 -c "import sys,json; print(json.load(sys.stdin).get('access_token',''))")

# 查看当前认证方法
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://graph.microsoft.com/v1.0/users/$USER_ID/authentication/methods" \
  | python3 -m json.tool

# 删除 Microsoft Authenticator（替换 <method-id>）
curl -X DELETE -H "Authorization: Bearer $TOKEN" \
  "https://graph.microsoft.com/v1.0/users/$USER_ID/authentication/microsoftAuthenticatorMethods/<method-id>"
```

### 第三步：登录并用 Google Authenticator 重新注册

1. 打开 Azure Portal，输入邮箱和密码
2. 系统强制进入 MFA 注册页面
3. **关键：点击"设置其他身份验证应用"**
4. 系统会显示一个标准 TOTP 二维码（6 位验证码）
5. 用 **Google Authenticator** 扫码完成注册

> Google Authenticator 使用标准 TOTP 协议，生成的是 6 位验证码，与 Azure AD 工作/学校账户要求一致。

### 第四步：清理

```bash
az ad app delete --id $APP_ID
```

## 如果第三步遇到 MFA 死循环

如果系统要求"先完成 MFA 验证才能注册新的 MFA"，需要额外操作：

```bash
# 查找 Policy.ReadWrite.AuthenticationMethod 权限 ID
PERM_ID=$(az ad sp list \
  --filter "appId eq '00000003-0000-0000-c000-000000000000'" \
  --query "[0].appRoles[?value=='Policy.ReadWrite.AuthenticationMethod'].id" -o tsv)

# 授予权限
az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/servicePrincipals/$SP_ID/appRoleAssignments" \
  --body "{
    \"principalId\":\"$SP_ID\",
    \"resourceId\":\"$GRAPH_SP_ID\",
    \"appRoleId\":\"$PERM_ID\"
  }"

# 重新获取令牌（同第二步）

# 启用 Temporary Access Pass 策略
curl -X PATCH -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"@odata.type":"#microsoft.graph.temporaryAccessPassAuthenticationMethodConfiguration","state":"enabled","includeTargets":[{"targetType":"group","id":"all_users"}]}' \
  "https://graph.microsoft.com/v1.0/policies/authenticationMethodsPolicy/authenticationMethodConfigurations/TemporaryAccessPass"

# 创建临时通行证
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"lifetimeInMinutes": 480, "isUsableOnce": false}' \
  "https://graph.microsoft.com/v1.0/users/$USER_ID/authentication/temporaryAccessPassMethods"
```

用返回的通行证完成 MFA 验证，然后按第三步注册 Google Authenticator。

## 总结

| 步骤 | 操作 |
|------|------|
| 1 | 通过 Azure CLI 创建临时应用，获取 Graph API 权限 |
| 2 | 调用 Graph API 删除旧的 Microsoft Authenticator |
| 3 | 登录时选"设置其他身份验证应用"，用 Google Authenticator 扫码 |
| 4 | 清理临时应用 |

**最简路径：删除旧 MFA → 登录 → 选"设置其他身份验证应用" → Google Authenticator 扫码。**

## 参考链接

- [Authenticator generates 8 digits and Azure requires 6 digits](https://learn.microsoft.com/en-us/answers/questions/5554083/authenticator-generates-8-digits-and-azure-require)
- [Azure wants 6 digits code but Authenticator generates 8 digits](https://learn.microsoft.com/en-us/answers/questions/2237947/azure-wants-6-digits-code-but-authenticator-genera)
- [Azure Portal login - 8-digit code but Azure requires 6-digit code](https://learn.microsoft.com/en-my/answers/questions/5811499/azure-portal-login-authenticator-generates-8-digit)

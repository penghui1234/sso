# AWS 账号 B 加入 Organization A + Kiro 权限管理指导（修正版）

> **场景**：账号 A（管理账号 `xxxxxxxxxxxx`，master_xxx），账号 B（`yyyyyyyyyyyy`，minion_xxx，已开通 Kiro Pro × 2）。
> **目标**：B 加入 A 的 Organization，回收 B 的 root，分配仅具备 Kiro 管理所需权限的 IAM 用户。

> ⚠️ **执行顺序铁律**：**先**在 B 内创建并验证 `kiro-admin`，**再**回收/移除 root。顺序反了（尤其先做集中式 root 移除）会导致你无法登录 B 创建用户，把自己锁在外面。

---

## 前置准备

| 检查项 | 说明 | 值 |
| --- | --- | --- |
| 账号 A（管理账号）Account ID | master_xxx | `xxxxxxxxxxxx` |
| 账号 B（成员账号）Account ID | minion_xxx | `yyyyyyyyyyyy` |
| 账号 A 是否已开通 Organizations | 必须是 Organization 的管理账号 | ✅ 是 |
| 账号 B 当前是否有自己的 Organization | ⚠️ 如果是，需先删除 | 需确认 |
| 账号 B 的 root email | 用于接收邀请 | xxx@example.com |
| 账号 B 上 Kiro 当前已开通的账号数 | 记录基线 | 2 个（Kiro Pro） |
| Kiro 用户 | user_b / user_a | Kiro Pro |

---

## 阶段一：将账号 B 加入 Organization A

### 步骤 1.0 — 前提：删除 B 自己的 Organization（如有）

> ⚠️ 如果账号 B 本身是某个 Organization 的管理账号，则无法接受 A 的邀请，必须先删除 B 的 Organization。

1. 用 **账号 B** 的 root 登录 AWS Console
2. 进入 **AWS Organizations** → **设置**
3. 点击 **删除组织（Delete organization）**（前提：B 的 Org 下没有其他成员账号）
4. 确认删除

**✅ 验证点**：B 不再是管理账号，变成独立账号；B 的 IAM 资源和 Kiro seats 不受影响。

---

### 步骤 1.1 — 在 A 中确认 Organization 状态

1. 用 **账号 A** 管理员登录
2. 进入 **AWS Organizations**
3. 确认 Organization 已创建，记录 Organization ID

**✅ 验证点**：能看到 Organization 信息和现有成员账号列表。

---

### 步骤 1.2 — 从 A 发送邀请给 B

1. Organizations 控制台 → **AWS 账户** → **添加 AWS 账户**
2. 选择 **邀请现有 AWS 账户**
3. 输入账号 B 的 Account ID：`yyyyyyyyyyyy`
4. （可选）添加备注 → **发送邀请**

**✅ 验证点**：邀请状态为 "OPEN"，在"邀请"页面可见。

> ⚠️ 不要填成 A 自己的 ID（`xxxxxxxxxxxx`），否则报 `DuplicateHandshakeException`。

---

### 步骤 1.3 — 在 B 中接受邀请

1. 用 **账号 B** 的 root 登录 → **AWS Organizations**
2. 页面显示收到的邀请 → 点击 **接受（Accept）**

**✅ 验证点**：B 显示已加入 Organization；A 的 Accounts 列表中 B 状态为 "Active"。

> 💡 **关于管理访问**：你是用"邀请"方式加入的，**不会**自动生成 `OrganizationAccountAccessRole`。因此 A **默认无法**以管理员身份直接进入 B——这与"B 受限自治"目标一致，符合预期。

---

### 步骤 1.4 — 验证 Kiro 服务不受影响

1. 用 **账号 B** 的 root 登录，进入 Kiro 管理页面
2. 确认 Kiro 用户仍在（user_b、user_a），功能正常

**✅ 验证点**：加入 Org 后 Kiro 未中断，已有 seats 正常。

> 💡 B 的 Kiro 绑定在 **B 本地的 IAM Identity Center 实例（account instance）** 上，加入 Org 后仍由 B 自己使用，与 A 的组织实例互不相通。

---

## 阶段二：在账号 B 中创建 `kiro-admin` IAM 用户

> 💡 **为什么不用 A 的 IAM Identity Center（SSO）方案？** 经测试，通过 A 的组织实例创建的 SSO 用户登录 B 后，Kiro 管理页面看不到 B 原有的 Kiro 用户（显示 0 个）——因为 Kiro 分配绑定在 B 本地的 account instance 上。必须在 B 账号内部用 IAM 用户才能正常管理。

> ⚠️ **本阶段必须在回收 root 之前完成。**

### 步骤 2.1 — 创建 IAM 用户

1. 用 **账号 B 的 root** 登录 → **IAM** → **Users** → **Create user**
2. 用户名：`kiro-admin`
3. 勾选 **Provide user access to the AWS Management Console**
4. 设置自定义密码，记录在密码管理器
5. 取消勾选"用户必须在下次登录时创建新密码"（可选）

### 步骤 2.2 — 创建并附加策略

**IAM** → **Policies** → **Create policy** → **JSON**：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "KiroAndSSOFullAccess",
      "Effect": "Allow",
      "Action": [
        "codewhisperer:*",
        "q:*",
        "sso:*",
        "sso-directory:*",
        "identitystore:*",
        "user-subscriptions:*",
        "organizations:DescribeOrganization",
        "organizations:ListAccounts"
      ],
      "Resource": "*"
    },
    {
      "Sid": "SelfService",
      "Effect": "Allow",
      "Action": [
        "iam:ChangePassword",
        "iam:GetUser",
        "iam:GetAccountPasswordPolicy",
        "sts:GetCallerIdentity"
      ],
      "Resource": "*"
    }
  ]
}
```

**策略说明：**

| Action | 用途 |
| --- | --- |
| `codewhisperer:*` | Kiro 核心功能（代码补全、对话） |
| `q:*` | Amazon Q 相关 API（可选，现订阅管理多走 `user-subscriptions`，留着无害） |
| `sso:*` | 读取 Identity Center 实例和应用分配（Kiro 管理页面需要） |
| `sso-directory:*` | Identity Center 目录服务 |
| `identitystore:*` | Identity Store API（列出/创建/删除用户） |
| `user-subscriptions:*` | Kiro 订阅管理（Pro plan 分配） |
| `organizations:Describe/List` | 读取 Organization 信息 |
| `iam:ChangePassword` | 改自己的密码 |
| `iam:GetUser` | 查看自己的用户信息 |
| `iam:GetAccountPasswordPolicy` | 改密时控制台需要读取密码策略 |
| `sts:GetCallerIdentity` | 确认当前身份 |

> ⚠️ `kiro:*` 不是有效的 IAM 服务前缀，Kiro 实际走 `codewhisperer` 和 `q`/`user-subscriptions` 命名空间。
>
> ⚠️ **权限边界提示**：此策略含 `sso:*` + `identitystore:*`，作用范围覆盖**整个 IAM Identity Center**，不只 Kiro。`kiro-admin` 因此**能够访问 Identity Center 控制台并管理其中的用户/应用分配**，这并非严格意义上的"仅 Kiro 最小权限"，而是为保证 Kiro 管理页面正常工作的实用授权。请勿再给该用户附加其他策略。

1. 策略名称：`KiroAdminOnlyPolicy` → **创建策略**
2. **Users** → `kiro-admin` → **Permissions** → **Add permissions** → **Attach policies directly** → 选择 `KiroAdminOnlyPolicy` → 附加

### 步骤 2.3 — 验证 IAM 用户登录

**登录地址**：`https://yyyyyyyyyyyy.signin.aws.amazon.com/console`

| 字段 | 值 |
| --- | --- |
| Account ID | `yyyyyyyyyyyy` |
| IAM username | `kiro-admin` |
| Password | 你设置的密码 |

### 步骤 2.4 — 验证 Kiro 管理权限

| 测试操作 | 期望结果 |
| --- | --- |
| 进入 Kiro → Dashboard | ✅ 正常显示 |
| 进入 Kiro → Users & Groups | ✅ 能看到 user_b、user_a |
| 看到 "Add user" 按钮 | ✅ 有按钮 |
| 管理 Kiro subscription（Change/Deactivate plan） | ✅ 正常 |
| 打开 EC2 / S3 控制台 | ❌ 无权限 |
| 打开 IAM Identity Center 控制台 | ✅ 可访问（管理 Kiro 所需，符合预期） |
| 修改 IAM 策略 / 创建用户 | ❌ 无权限 |
| `aws sts get-caller-identity` | ✅ 返回 kiro-admin 身份 |

> ⚠️ **排障**：
> - Dashboard 报 `user-subscriptions:ListApplicationClaims` → 缺 `user-subscriptions:*`
> - Users & Groups 报 `sso:ListInstances` → 缺 `sso:*`
> - 显示 0 个用户且无 "Add user" → 检查 `sso:*` 和 `identitystore:*` 是否都加了

> ✅ **必须在此步全部通过后，再进入阶段三回收 root。**

---

## 阶段三：回收 B 的 Root 账号

> 二选一：**方案 A**（自己保管 root）或 **方案 B**（集中移除 root，更彻底，推荐）。两者不要叠加——做了方案 B，方案 A 设的 MFA/密码会被一并清除而失去意义。

### 方案 A — 自己保管 root（手动加固）

**A.1 启用 MFA**
root 登录 → **IAM** → **Security credentials** → **Assign MFA device** → 绑定到**你**控制的设备。

**A.2 修改 root 密码**
**Security credentials** → **Change password** → 设强密码并记录在你的密码管理器 → 退出。

**A.3 删除 root Access Key（如有）**
**Security credentials** → "Access keys" → 有则立即 **Delete**。

**✅ 验证点**：root 的 MFA 在你手里、密码只有你知道、无任何 Access Key。

### 方案 B —（推荐）通过 Organization 集中移除 root 凭证

1. 用账号 A 登录 → **IAM** → **Root access management**
2. 启用集中管控，对成员账号 B 执行移除 root 凭证

> 💡 此功能 2024 年 11 月上线。启用后可从管理账号统一移除成员账号 root 的密码、Access Key、MFA、签名证书，成员账号将无法通过 root 登录或找回密码。需要 root-only 操作时，再通过管理账号临时恢复。

**✅ 验证点**：在 IAM → Root access management 确认 B 的 root 凭证已被移除（无人可用 root 登录）。

---

## 阶段四：最终验证清单

| # | 检查项 | 验证方法 | 状态 |
| --- | --- | --- | --- |
| 1 | 账号 B 已在 Organization A 中 | A 的 Organizations 控制台 | ☐ |
| 2 | `kiro-admin` 能登录并管理 Kiro | 登录后看到 user_b、user_a + Add user | ☐ |
| 3 | `kiro-admin` 无法访问 EC2/S3 等 | 控制台报无权限 | ☐ |
| 4 | `kiro-admin` 无法改 IAM 策略 | 无 Create/Put policy 权限 | ☐ |
| 5 | Kiro 已有 seats 不受影响 | 所有 Kiro 用户正常使用 | ☐ |
| 6 | root 已回收（方案 A 或 B） | A：MFA+强密码+无 AK；B：凭证已移除 | ☐ |

---

## 操作顺序总结

```
1. [Account B] 删除自己的 Organization（如有）→ 变成独立账号
2. [Account A] 确认 Organization 状态
3. [Account A] 发送邀请给 B（输入 Account B ID）
4. [Account B] 接受邀请
5. [Account B] 用 root 验证 Kiro 用户仍在（user_b、user_a）
6. [Account B] 用 root 创建 IAM 用户 kiro-admin + 附加策略   ← 必须先做
7. [Account B] 用 kiro-admin 登录，验证 Kiro 管理全部正常     ← 验证通过再继续
8. [回收 root] 方案 A：B 自行加固 MFA+密码+删 AK
   或          方案 B：A 在 Root access management 移除 B 的 root 凭证（推荐）
9. 最终综合验证
```

---

## 常见问题 & 排障

**Q1：B 无法接受 A 的邀请？** B 自己是某 Org 的管理账号 → 先删除 B 的 Organization。

**Q2：发送邀请报 `DuplicateHandshakeException`？** 旧邀请仍 OPEN，或填成了自己的 Account ID → 取消旧邀请重发，确认填的是对方 ID。

**Q3：`kiro-admin` 看不到 Kiro 用户（显示 0）？** 缺 `sso:*`、`identitystore:*`、`user-subscriptions:*` → 补齐。

**Q4：Kiro 页面顶部红色错误？** `sso:ListInstances` → 加 `sso:*`；`user-subscriptions:ListApplicationClaims` → 加 `user-subscriptions:*`。

**Q5：`kiro:*` 报错？** `kiro` 不是有效服务前缀，删除即可，功能由 `codewhisperer`/`q`/`user-subscriptions` 提供。

**Q6：为什么不用 A 的 IAM Identity Center（SSO）方案？** Kiro 分配绑定在 B 本地 account instance 上，A 的组织实例看不到 → 必须在 B 内建 IAM 用户。

**Q7：加入 Org 后 A 能直接管理 B 吗？** 邀请方式加入**不会**自动建 `OrganizationAccountAccessRole`，A 默认无法以管理员进入 B（符合受限自治目标）。

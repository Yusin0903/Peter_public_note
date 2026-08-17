---
sidebar_position: 18
---
<!-- generated from ~/peter-llm-wiki; edit source there, not here -->

# IAM（Identity and Access Management）

回答一個問題：**誰（Identity）可以對什麼資源（Resource）做什麼操作（Action）**。

---

## 核心關係

```
Identity                    Policy                      Resource
──────────────────────────────────────────────────────────────────
IAM User   ──attach──→  [ Effect: Allow/Deny      ]
IAM Group  ──attach──→  [ Action: s3:GetObject    ]  ──→  AWS Resource
IAM Role   ──attach──→  [ Resource: arn:aws:s3::: ]       (S3, EC2, RDS...)
AWS Service              [ Condition: {...}        ]
```

Policy 是獨立物件，可附加到任何 Identity。

---

## User vs Role

| | IAM User | IAM Role |
|---|---|---|
| 代表誰 | 真實的人或機器帳號 | 可被 assume 的身份 |
| 憑證類型 | 長期 Access Key / 密碼 | 臨時 STS token（自動過期）|
| 適合場景 | 本機開發、CLI 操作 | EC2、Lambda、EKS Pod、跨帳號 |
| 推薦程度 | 盡量少用 | 服務優先使用 Role |

> 服務不應該用 User。User 的 Access Key 洩漏後沒有自動過期機制。

---

## Policy 結構

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    }
  ]
}
```

Deny 優先於 Allow，加一個 Deny Statement 就覆蓋其他 Allow。

Policy 類型：
- **AWS Managed** — AWS 維護，如 `AmazonS3ReadOnlyAccess`，方便但粒度粗
- **Customer Managed** — 自己寫，精確控制，推薦
- **Inline** — 直接嵌在 User/Role 上，不建議，難重用

### Inline vs Managed 的實務差異

| | Inline | Managed（Customer）|
|---|---|---|
| 大小上限 | **2048 bytes / user** | 6144 chars |
| 重用 | 不能（綁死在單一身份上）| 可 attach 到多個身份 |
| 版本 | 無 | 最多 5 個 version，可回溯 |
| 隨身份刪除 | 是 | 否，要先 detach 再 delete |

> 常見坑：inline policy 一超過 2048 bytes 就 `LimitExceeded`，得改成 managed。

### Policy 改版會自動生效

身份 attach 的是 **policy ARN**，不是內容快照。更新 default version 後，
所有 attach 這個 ARN 的 user/role **立刻套新版**，不用 detach/re-attach：

```bash
aws iam create-policy-version --policy-arn <arn> \
  --policy-document file://policy.json --set-as-default
# 沒有 --set-as-default 只是存起來，不會生效
aws iam get-policy --policy-arn <arn> --query 'Policy.DefaultVersionId'  # 確認 active 版本
```

> managed policy 最多存 5 個 version，反覆磨 policy 會塞滿，滿了要先 `delete-policy-version` 舊的。

---

## CLI Profile 與憑證解析

`aws` CLI 的 **profile ≠ 身份**。profile 只是 `~/.aws/config` / `~/.aws/credentials`
裡一段有名字的認證設定，它**指向**什麼身份才是重點：

| profile 寫法 | 背後身份 |
|---|---|
| 直接填 `aws_access_key_id`（AKIA...）| **IAM User**（長期憑證）|
| `role_arn` + `source_profile` | **assume 一個 IAM Role**（臨時憑證）|
| `sso_*` 設定 | **SSO 臨時身份** |

判斷「我現在到底是誰」：

```bash
aws sts get-caller-identity --profile <名>
# Arn = .../user/xxx           → IAM User
# Arn = .../assumed-role/xxx/yyy → assume 了 Role
```

### 憑證優先序（重要踩坑）

環境變數 **優先於** `AWS_PROFILE`。若 `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` /
`AWS_SESSION_TOKEN` 存在，`export AWS_PROFILE=...` 會被無視，CLI 一直用舊身份：

```bash
env | grep -i aws | cut -d= -f1   # 只看變數名，不印值
# 若看到 AWS_ACCESS_KEY_ID 等，先 unset 才能讓 profile 生效：
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN AWS_SECURITY_TOKEN
```

### STS 臨時憑證 vs 長期 User 憑證：怎麼選

- **STS（assume role）**：憑證會自動過期，安全，服務/日常操作首選。
- **例外——「中間不能斷」的長操作**（如 Terraform state rm/import 到一半）：
  若 STS 過期卡在半途，連 rollback 都需要憑證，會卡死救不回。
  這種一次性操作可改用**長期 IAM User 憑證**（不會中途過期），
  代價是 key 不過期、外洩風險高 → 用完立刻刪 key + user。

> 最小權限實例：只做 state rm/import 的暫時 user，權限只需
> 「state bucket 讀寫 + DynamoDB lock + import 目標唯讀」；
> plan/apply 若走 CI，建/改/刪類權限完全不必給。

---

## 常見 Scenario

### EC2 存取 S3

```
EC2 Instance
  └── Instance Profile
        └── IAM Role: ec2-app-role
              └── Policy: Allow s3:GetObject on arn:aws:s3:::my-bucket/*
```

Trust Policy（允許 EC2 服務 assume 這個 role）：

```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
```

EC2 上的程式透過 IMDS 自動取得臨時 credentials，不需要 hardcode Access Key。

### EKS Pod 存取 DynamoDB

Node 的 Role 不應該直接給 Pod 用，需要 IRSA：

```
EKS Pod
  └── K8s ServiceAccount（annotation: eks.amazonaws.com/role-arn）
        └── IAM Role（Trust Policy 允許 OIDC provider assume）
              └── Policy: Allow dynamodb:GetItem / PutItem / Query
```

詳細設定見 [IRSA 指南](./irsa-guide)。

---

## Terraform 中 role 與 policy 是獨立 resource

Role（身份）和 Policy（權限）在 Terraform 裡是兩個獨立宣告的 resource，中間要用第三個 resource 把它們綁在一起：

```
aws_iam_role.app_sidecar          ← 身份（誰可以 assume：trust policy）
        +
aws_iam_policy.app_sidecar_s3     ← 權限（s3:ListBucket / PutObject / GetObject）
        ↕
aws_iam_role_policy_attachment    ← 把 policy 綁到 role 上
```

```hcl
resource "aws_iam_role" "app_sidecar" {
  name               = "app-sidecar"
  assume_role_policy = data.aws_iam_policy_document.trust.json
}

resource "aws_iam_policy" "app_sidecar_s3" {
  name   = "app-sidecar-s3"
  policy = data.aws_iam_policy_document.s3_access.json
}

resource "aws_iam_role_policy_attachment" "app_sidecar_s3" {
  role       = aws_iam_role.app_sidecar.name
  policy_arn = aws_iam_policy.app_sidecar_s3.arn
}
```

分開宣告的好處是 policy 可以重用——同一個 `aws_iam_policy` 可以用多個 `aws_iam_role_policy_attachment` 綁到不同的 role 上，不用重複寫一樣的權限內容。

---

## 一句話總結

| 概念 | 一句話 |
|---|---|
| IAM User | 代表人的長期身份，有密碼或 Access Key |
| IAM Role | 代表服務的臨時身份，憑證自動過期 |
| Policy | 定義 Allow / Deny 的 Action + Resource 規則 |
| Trust Policy | 定義誰可以 assume 這個 Role |
| Instance Profile | 把 Role 綁到 EC2 |
| IRSA | 把 Role 綁到 EKS Pod（透過 OIDC）|
| aws_iam_role_policy_attachment | Terraform 裡把 Policy 綁到 Role，讓 Policy 可跨 Role 重用 |

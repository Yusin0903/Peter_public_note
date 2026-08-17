---
sidebar_position: 10
---
<!-- generated from ~/peter-llm-wiki; edit source there, not here -->

# kubectl 連 EKS 的完整驗證鏈路，以及一個容易誤判的坑

`kubectl exec` 被 Forbidden 時，常見反射動作是「換一個權限更高的 AWS profile」，但如果卡住的地方不在 AWS IAM，而在更下游，換 profile 不會有任何效果。這篇整理完整鏈路，以及一個實際踩過、很容易忽略的環境變數優先權問題。

## 完整鏈路

```
1. 你執行 kubectl exec ...
        ↓
2. kubectl 查目前 current-context，取得對應的 cluster + user 設定
        ↓
3. kubectl 執行 user.exec 裡指定的指令
   （通常是 aws eks get-token --cluster-name ... --profile <profile>）
        ↓
4. aws CLI 用該 profile 的憑證去問 AWS STS：這組憑證代表誰？
        ↓
5. AWS 回一個短期的 EKS token 給 kubectl
        ↓
6. kubectl 拿這個 token 去打 EKS API server
        ↓
7. EKS API server 反查 aws-auth ConfigMap，
   把這個 AWS 身份映射成一個 k8s 使用者
        ↓
8. k8s RBAC 判斷這個映射後的使用者，有沒有權限做這個操作
        ↓
9. 有權限 → 執行；沒權限 → Forbidden
```

`aws eks update-kubeconfig` 做的事只是**把第 2-3 步需要的設定寫進 `~/.kube/config`**，不是建立連線，也不是建立 AWS profile——它只是引用一個早就存在的 profile。之後每次 `kubectl` 操作，才會即時重跑第 3-6 步換一次短期 token（token 通常幾分鐘到幾小時過期，所以每次動態換發，不是寫死永久憑證）。

## Profile 與 STS 不是同一層東西

- **Profile**：一組登入 AWS 的憑證設定，存在 `~/.aws/credentials` 或 `~/.aws/config`，是「你要用哪組身份」的靜態設定
- **STS（Security Token Service）**：AWS 的一個服務，用來驗證身份、換發臨時憑證。`aws sts get-caller-identity` 只是「問 AWS 我現在的憑證代表誰」，本身不是一組 profile，是拿來驗證用的 API

`--profile ti-eks-prod` 不會建立新 profile，是使用一個既有的 profile；如果這個 profile 不存在，指令會直接報錯找不到。

## Context 的 name 只是代稱，不是關鍵資料

kubeconfig 的 `context` 結構：

```yaml
contexts:
- name: <任意字串，習慣用 cluster ARN 命名，但不是規定>
  context:
    cluster: <cluster 名字，指向真正的 API server 位址與 CA 憑證>
    user: <user 名字，指向怎麼取得驗證用的 token>
```

`aws eks update-kubeconfig` 習慣用 cluster ARN 當 context 名稱只是命名慣例（ARN 全域唯一不會撞名），可以用 `kubectl config rename-context <old> <new>` 改成好記的名字，不影響功能。真正決定連線行為的是 `cluster` 和 `user` 兩個欄位，不是 `name`。

## 踩坑：shell 殘留的環境變數蓋過 kubeconfig 設定

kubeconfig 裡可以在 `user.exec.env` 明確指定要用哪個 profile：

```json
{
  "command": "aws",
  "args": ["eks", "get-token", "--cluster-name", "my-cluster"],
  "env": [{"name": "AWS_PROFILE", "value": "my-target-profile"}]
}
```

看起來應該保證每次都用 `my-target-profile`。但如果目前 shell session 裡已經有殘留的環境變數：

```bash
export AWS_PROFILE=some-other-profile
# 或更隱蔽的：
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...
```

**shell 環境變數的優先權高於 kubeconfig 裡 `exec.env` 設定的值**，`aws eks get-token` 實際執行時會用 shell 裡已存在的憑證，完全忽略 kubeconfig 裡寫的 profile。症狀是：不管怎麼改 kubeconfig、切換 profile，`kubectl` 換到的身份永遠不變。

排查方式：

```bash
# 1. 檢查 shell 是否有殘留的 AWS 相關環境變數
env | grep -i AWS

# 2. 確認目前 kubectl 實際換到的身份
kubectl config view --minify -o jsonpath='{.users[0].user.exec}'

# 3. 直接驗證當前 AWS CLI 憑證代表誰
aws sts get-caller-identity
```

如果 `env | grep -i AWS` 有輸出，尤其是 `AWS_SESSION_TOKEN`（代表是臨時憑證，可能是之前某個腳本或 assume-role 操作留下的），先清掉再重跑：

```bash
unset AWS_PROFILE AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN AWS_SECURITY_TOKEN
```

或者在指令當下直接覆蓋，不動全域環境：

```bash
env -u AWS_PROFILE -u AWS_ACCESS_KEY_ID -u AWS_SECRET_ACCESS_KEY -u AWS_SESSION_TOKEN \
  AWS_PROFILE=my-target-profile kubectl exec -it <pod> -- sh
```

## `pods/exec` Forbidden 時的排查順序

不要一開始就假設是 AWS IAM 權限不夠而急著換 profile，順著鏈路由下往上查：

```bash
# 1. 確認 kubectl 實際換到的身份是誰（排除環境變數污染）
env | grep -i AWS
aws sts get-caller-identity

# 2. 確認這個身份在 k8s 端實際被授權什麼
kubectl auth can-i create pods/exec -n <namespace>
kubectl auth can-i --list -n <namespace>

# 3. 查 EKS 的 AWS 身份 → k8s 使用者映射
kubectl get configmap aws-auth -n kube-system -o yaml
```

如果 `kubectl auth can-i` 回 `no`，代表問題出在 **k8s RBAC 沒有把 exec 權限授權給這個映射後的角色**，不是你的 AWS profile 或 IAM 設定問題——換多少個「權限更高」的 profile 都不會有幫助，除非那個 profile 映射到的 k8s 角色不同。

## 一句話總結

`kubectl exec` 的授權判斷發生在 k8s RBAC 這一層，不是 AWS IAM。換 profile 只在「不同 profile 映射到不同 k8s 角色」時才有意義；如果卡住的地方是 shell 殘留環境變數蓋過了 kubeconfig 設定，或是 k8s RBAC 本身沒開放權限，換多少 profile 都無效——先用 `aws sts get-caller-identity` 和 `kubectl auth can-i` 確認卡在哪一層，再對症下藥。

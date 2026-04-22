---
sidebar_position: 4
---

# K8s 部署工具比較 & Terraform 定位

---

## Terraform 不是 K8s 部署工具

Terraform 是 **infrastructure 管理工具**，K8s 只是它能管的其中一個東西。

### Terraform 能管的東西

```
AWS       → VPC, EKS, EC2, RDS, S3, IAM, Route53, ALB...
Azure     → AKS, VM, Storage, AD...
GCP       → GKE, Compute, Cloud SQL...
K8s       → Helm release, kubectl manifest, namespace...
其他      → GitHub repo, Datadog monitor, PagerDuty,
            Cloudflare DNS, Vault secrets...
```

基本上任何有 API 的服務都能用 Terraform 管。

---

## K8s 部署主流工具比較

| 工具 | 定位 | 適合 |
|---|---|---|
| **Terraform + Helm** | 建 infra + 部署 app 一起管 | Ops/Platform team |
| **Helm** 單獨用 | K8s app 部署 | 只管 K8s，不管雲端資源 |
| **ArgoCD / FluxCD** | GitOps — git push 自動部署到 K8s | 持續部署、app team |
| **kubectl** | 手動 apply YAML | debug、學習 |
| **Kustomize** | YAML overlay 管多環境 | 不想用 Helm template |
| **Pulumi** | 用程式語言（Python/TypeScript）寫 infra | 不想學 HCL |

---

## Terraform + Helm 搭配

```
Terraform 管：
  → K8s cluster 本身（EKS / AKS / GKE）
  → 雲端資源（Load Balancer, IAM, Secrets, Storage...）
  → 呼叫 Helm 部署 app

Helm 管：
  → K8s 裡面的 app（Deployment, Service, ConfigMap, RBAC...）
```

如果只用 Helm — 能部署 app 到 K8s，但誰來建 K8s cluster？Load Balancer？Secrets？

**Terraform 管整個 stack（雲端 + K8s），Helm 只管 K8s 裡面的 app。**

---

## Helm chart vs 手寫 YAML

### Helm chart — 幫你寫好 YAML

```
手動：
  手寫 Deployment YAML
  手寫 Service YAML
  手寫 ServiceAccount + RBAC YAML
  kubectl apply -f 一個一個套

Helm：
  chart 裡已包好所有 YAML template
  只需要填 values（image tag, replica count...）
  helm install 一次全部搞定
```

### kubectl_manifest — 還是自己寫 YAML

```hcl
# 放在 Terraform 裡，但 YAML 是自己寫的
yaml_body = <<-YAML
  apiVersion: apps/v1
  kind: Deployment
  ...
YAML
```

### 比較

| | 手動 kubectl | Terraform + Helm |
|---|---|---|
| 複雜元件 | 自己寫 20+ YAML | Helm chart 包好，填 values |
| 簡單資源 | `kubectl apply -f` | `kubectl_manifest`（一樣自己寫 YAML） |
| 版本控制 | 手動管 YAML | git + Terraform state 自動追蹤 |
| 多環境 | 每個環境複製一份改值 | 同一份 code 不同 inputs |
| 回滾 | apply 舊版本 YAML | Terraform state 追蹤，自動 diff |

**Terraform 的價值不是「幫你寫 YAML」，而是管理狀態 + 多環境 + 版本控制 + 依賴順序。**

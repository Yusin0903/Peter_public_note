---
sidebar_position: 3
---

# K8s & AWS 基礎名詞對照表

---

## Kubernetes 核心

| 名詞 | 一句話 | 類比 |
|---|---|---|
| **Pod** | 最小部署單位，跑一個或多個 container | 一台機器上跑的一個 process |
| **Deployment** | 管理 Pod 的副本數量、更新策略 | PM 說「這個服務要跑 3 份」 |
| **StatefulSet** | 有狀態的 Deployment，每個 Pod 有固定名稱和磁碟 | 資料庫 instance |
| **Service** | 給一組 Pod 一個穩定的內部 IP/DNS | 櫃台接待 |
| **Ingress** | L7 路由規則（host/path → Service） | 大樓門口指示牌 |
| **Namespace** | 資源隔離的虛擬分組 | 辦公室的不同樓層 |
| **ConfigMap** | 存設定檔（非機密） | 共用資料夾的 config |
| **Secret** | 存機密資料（密碼、token） | 保險箱裡的 config |
| **PVC** | PersistentVolumeClaim — Pod 的磁碟申請單 | 跟 IT 申請硬碟 |
| **PV** | PersistentVolume — 實際的磁碟 | IT 給你的硬碟 |
| **StorageClass** | 磁碟的規格表 | 硬碟型號目錄 |
| **CRD** | CustomResourceDefinition — 自訂資源類型 | 教 K8s 認識新東西 |
| **Operator** | 自動管理 CRD 的 controller | 機器人看 CRD，有變化就處理 |
| **DaemonSet** | 每個 Node 跑一個 Pod | 每台機器都裝的 agent |
| **Node** | K8s cluster 裡的一台機器 | 辦公室的一台電腦 |
| **Helm** | K8s 的套件管理工具 | 像 pip/npm |
| **kubectl** | K8s 的 CLI 工具 | 像 aws cli |

---

## AWS 基礎

| 名詞 | 一句話 |
|---|---|
| **EKS** | Elastic Kubernetes Service — AWS 代管的 K8s |
| **EC2** | Elastic Compute Cloud — 虛擬機 |
| **EBS** | Elastic Block Store — 網路磁碟 |
| **ALB** | Application Load Balancer — L7 負載均衡 |
| **ECR** | Elastic Container Registry — Docker image 倉庫 |
| **IAM** | Identity and Access Management — 權限管理 |
| **ACM** | AWS Certificate Manager — SSL/TLS 憑證管理 |
| **Secrets Manager** | 集中管理密碼和 token |
| **S3** | Simple Storage Service — 物件儲存 |
| **Route53** | DNS 服務 |
| **VPC** | Virtual Private Cloud — 虛擬網路 |
| **Transit Gateway** | 跨 VPC / 跨 Region 的網路路由 |

---

## Terraform

| 名詞 | 一句話 |
|---|---|
| **Terraform** | IaC 工具，用 HCL 定義雲端資源 |
| **Provider** | Terraform 跟雲端 API 的橋接 |
| **Module** | 一組 resource 打包成可重用模組 |
| **State** | Terraform 記錄的資源狀態 |
| **Plan** | 預覽變更 |
| **Apply** | 實際執行變更 |
| **Terragrunt** | Terraform 的 wrapper，解決多環境共用 |

---

## Monitoring

| 名詞 | 一句話 |
|---|---|
| **Prometheus** | 開源監控系統，pull-based metrics 收集 |
| **remote_write** | Prometheus 把 metrics 推到遠端儲存的標準 API |
| **external_labels** | 加在所有 metrics 上的固定 label |
| **PromQL** | Prometheus 的查詢語言 |
| **Grafana** | 視覺化 dashboard 工具 |
| **Datasource** | Grafana 連接資料來源的設定 |
| **Cardinality** | Metrics 的唯一 time series 數量 |

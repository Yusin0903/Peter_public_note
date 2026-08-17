---
sidebar_position: 10
---
<!-- generated from ~/peter-llm-wiki; edit source there, not here -->

# Terraform 頂層區塊全覽 (Top-Level Blocks)

## locals vs data 一句話區分

**locals 是你自己算出來的值；data 是問 provider 拿到的值。**

| | `locals` | `data` |
|---|---|---|
| 值的來源 | 純運算（字串組合、對 var/resource 屬性做處理） | 呼叫 provider API 查詢外部既有資源 |
| 是否呼叫外部 API | 否 | 是 |
| 何時知道值 | apply 前就能算出來 | 通常要到 plan/apply 時才知道 |
| Refresh 行為 | 沒有 | 每次 plan 都可能重新抓取（除非值已知或有 depends_on） |

```hcl
locals {
  name_prefix = "${var.env}-${var.project}"
  common_tags = { Team = "TI", Env = var.env }
}

data "aws_vpc" "default" {
  default = true
}
# data.aws_vpc.default.id 只有在 apply 時才知道實際值，取決於帳號/環境
```

## 跟 locals / data 同層級的所有頂層區塊

### 核心區塊

| Block | 用途 |
|---|---|
| `terraform` | 設定 Terraform 本身：required_version、required_providers、backend（state 存哪） |
| `provider` | 設定 provider 參數（region、profile、alias…） |
| `resource` | 宣告要建立/管理的實體資源，唯一會真的新增/修改/刪除東西的區塊 |
| `data` | 唯讀查詢外部已存在的資訊 |
| `variable` | 輸入參數，外部（tfvars、CLI、環境變數）可傳值進來 |
| `locals` | 內部計算出來的值，模組外看不到、不能被覆蓋 |
| `output` | 對外輸出值，給呼叫這個 module 的人或 `terraform output` CLI 用 |
| `module` | 呼叫另一包 Terraform 程式碼（本地路徑或 registry） |

### 進階/次要區塊

| Block | 用途 |
|---|---|
| `provisioner`（寫在 resource 裡） | apply 後在資源上跑指令（remote-exec、local-exec），官方建議少用 |
| `import` | 宣告要把既有資源匯入 state（新式寫法，取代 `terraform import` CLI） |
| `moved` | 宣告資源改名/搬動，避免 destroy+create |
| `check` | 自訂的持續驗證斷言 |
| `removed` | 宣告某資源要從 state 移除但不刪除實體 |

## 讀一份 .tf 檔案的判讀順序

1. `terraform{}` + `provider{}` — 操作對象是哪個雲、哪個 state
2. `variable{}` — 這個模組吃什麼輸入
3. `data{}` — 讀了哪些外部既有狀態
4. `locals{}` — 內部怎麼加工這些值
5. `resource{}` / `module{}` — 實際做了什麼（重頭戲）
6. `output{}` — 吐出什麼給外部用

`resource` 通常佔大宗、是最重要要讀懂的部分，其餘都是圍繞著它在準備輸入或處理輸出。

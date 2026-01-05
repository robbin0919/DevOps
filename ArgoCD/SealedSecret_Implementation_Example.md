# SealedSecret 完整指南：原理、操作與實例

在 GitOps 實踐中，`SealedSecret` 是實現「Secret 即程式碼」的關鍵組件。它由 Bitnami 開發，允許我們將敏感資料加密後安全地存放在版本控制系統中。

## 第一部分：SealedSecret 核心原理詳解

### 1. 什麼是 SealedSecret？

`SealedSecret` 是一種 Kubernetes 自定義資源 (Custom Resource Definition, CRD)。你可以將它想像成一個**「加密過的 Secret 保險箱」**：
*   **外殼**：標準的 YAML 格式，包含名稱、命名空間等後設資料。
*   **內容**：敏感資料（如資料庫帳密、API Key）被轉換為一串強效加密的亂碼。

### 2. 與標準 Kubernetes Secret 的比較

| 特性 | 標準 K8s Secret | SealedSecret |
| :--- | :--- | :--- |
| **資料格式** | **Base64 編碼** | **強效非對稱加密** |
| **安全性** | 僅是編碼，任何人皆可輕易解碼 (Base64 不是加密)。 | 只有目標叢集內的私鑰可以解密。 |
| **Git 版控** | **嚴格禁止**入版控。 | **安全**，可放心存放在 Git 儲存庫。 |
| **處理方式** | 手動建立或透過環境變數注入。 | 透過 `kubeseal` 指令產生。 |

### 3. 運作原理：非對稱加密

SealedSecret 的核心防禦力來自於**非對稱加密 (Asymmetric Encryption)** 機制：

1.  **公鑰 (Public Key)**：
    *   用於「加密」。
    *   開發者可以在本地環境取得公鑰，將明碼 Secret 加密成 SealedSecret。
2.  **私鑰 (Private Key)**：
    *   用於「解密」。
    *   **關鍵安全點**：這把鑰匙**僅存在於 Kubernetes 叢集內的 Controller 中**。它從不離開叢集，也不會被備份到 Git。

### 4. 工作流程 (Workflow)

1.  **加密 (開發端)**：開發者準備好 Secret YAML，執行 `kubeseal < secret.yaml > sealedsecret.yaml`。
2.  **交付 (Git 傳輸)**：將加密後的 `sealedsecret.yaml` 推送到 Git。
3.  **同步 (ArgoCD)**：ArgoCD 監控到 Git 異動，將 `SealedSecret` 部署到叢集。
4.  **還原 (叢集端)**：叢集內的 `SealedSecret Controller` 偵測到新資源，利用手中的私鑰進行解密，並在叢集中自動產生一個標準的 `Kind: Secret` 供應用程式使用。

### 5. 安全優勢總結

*   **即使 Git 被駭也安全**：駭客即使偷走了 Git 上的 `SealedSecret` 檔案，因為沒有叢集內部的私鑰，拿到的也只是一堆無法解讀的亂碼。
*   **符合自動化精神**：讓機密資訊的管理流程與一般程式碼完全一致，皆可透過 Pull Request、代碼審查與版本追蹤。

---

## 第二部分：SealedSecret 實戰操作範例

本節展示如何將含有敏感資訊的 Kubernetes Secret 轉換為安全的加密格式。

### 1. 階段一：加密前（原始 Secret）
*   **檔案名稱**：`db-secret.yaml`
*   **安全性**：**危險** ❌（絕對禁止提交至 Git）。
*   **說明**：這是開發者在本地編輯的檔案，包含真實的明碼資料。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-database-secret
  namespace: production
type: Opaque
stringData:
  # 直接寫入明碼資料
  username: "admin"
  password: "SuperSecretPassword123!"
```

### 2. 階段二：加密指令
使用 `kubeseal` CLI 工具進行轉換。該工具會與 Kubernetes 叢集通訊以取得公鑰（或使用本地存放的公鑰檔案）。

```bash
# 從標準輸入讀取明碼 YAML，並將加密後的結果輸出至 sealed-db-secret.yaml
kubeseal --format=yaml < db-secret.yaml > sealed-db-secret.yaml
```

### 3. 階段三：加密後（SealedSecret）
*   **檔案名稱**：`sealed-db-secret.yaml`
*   **安全性**：**安全** ✅（可放心提交至 Git 儲存庫）。
*   **說明**：原本的內容已變為加密亂碼，僅有目標叢集內的 Controller 能解密。

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: my-database-secret
  namespace: production
spec:
  # 加密後的亂碼資料
  encryptedData:
    username: AgBy3i4q... (加密後的長亂碼) ...
    password: AgB65j1z... (加密後的長亂碼) ...
  
  # 此模板定義了還原後的 Secret 樣貌
  template:
    metadata:
      name: my-database-secret
      namespace: production
      type: Opaque
```

### 4. 關鍵差異對照表

| 項目 | 加密前 (Secret) | 加密後 (SealedSecret) |
| :--- | :--- | :--- |
| **資源種類 (Kind)** | `Secret` | `SealedSecret` |
| **資料欄位** | `stringData` (或 `data`) | `encryptedData` |
| **可讀性** | 明碼 (或 Base64 編碼，易破解) | 強效加密亂碼 (無法解析) |
| **處理規則** | **嚴禁入版控** | **建議入版控** |

### 5. 注意事項
*   **本地檔案處理**：產生 `SealedSecret` 後，請務必手動刪除包含明碼的原始 `db-secret.yaml`。
*   **Namespace 綁定**：預設情況下，SealedSecret 是與 Namespace 綁定的。如果您將 YAML 移動到不同的 Namespace 部署，解密將會失敗。
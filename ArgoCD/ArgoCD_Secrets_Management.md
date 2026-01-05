# ArgoCD 與 GitOps 環境下的機密資訊管理 (Secrets Management)

在 GitOps 架構下，最大的挑戰之一是：**「如何將敏感資料（如資料庫帳密）放入公開透明的 Git，卻不外洩機密？」**

與 HTTPS 憑證（通常由 cert-manager 自動在叢集內產生）不同，資料庫帳密是既定的靜態資訊，必須從外部「注入」到 Kubernetes 中。

## 1. 主流解決方案

目前業界有兩種標準做法來處理 Git 中的 Secret：

### 方案 A：Sealed Secrets (加密後放入 Git)
這是最符合 GitOps 精神且門檻較低的做法，適合地端或不依賴特定雲端的環境。
*   **工具**：Bitnami Sealed Secrets
*   **核心概念**：**非對稱加密**。
    *   **公鑰 (Public Key)**：開發者持有，用於將明碼 YAML 加密成亂碼的 `SealedSecret`。
    *   **私鑰 (Private Key)**：僅存在於 K8s 叢集內部的 Controller 中，用於解密。
*   **流程**：
    1.  開發者在本地使用 `kubeseal` 指令加密 YAML。
    2.  將加密後的 `SealedSecret` 提交至 Git。
    3.  ArgoCD 同步至叢集，Controller 自動解密並還原為標準 Secret。

### 方案 B：External Secrets Operator (外部金庫參照)
這是企業級、雲端原生的標準做法。
*   **工具**：External Secrets Operator (ESO)
*   **核心概念**：Git 裡只存「索引 (Reference)」，真正的秘密鎖在雲端保險箱。
*   **流程**：
    1.  將帳密存放在 **AWS Secrets Manager**, **Azure Key Vault** 或 **HashiCorp Vault**。
    2.  Git 中提交 `ExternalSecret` YAML，內容僅包含：「請去 AWS 取出名為 `prod-db-password` 的資料」。
    3.  ESO 在叢集內驗證權限後，動態抓取密碼並建立 Secret。

## 2. 安全性分析：駭客能否進行「重放攻擊」？

**情境**：如果駭客從 Git 偷走了加密過的 `SealedSecret` YAML，原封不動地拿到他自己的 K8s 叢集執行，能否連線到資料庫？

**答案：不能。** 即使檔案被竊取，安全性依然受控，原因如下：

1.  **私鑰隔離 (Private Key Isolation)**
    *   解密需要特定的「私鑰」。這把私鑰是在您安裝 Sealed Secrets 時於**您的叢集內產生**的，且從未離開過您的內網環境。
    *   駭客的叢集沒有這把私鑰，因此無法將 `SealedSecret` 還原成明碼。

2.  **作用域綁定 (Strict Scope)**
    *   Sealed Secrets 預設採用 Strict Scope，加密時會將 **Namespace** 與 **Secret Name** 一併編碼。
    *   如果駭客試圖修改 YAML（例如改 namespace），解密就會失敗。

3.  **網路層防護**
    *   資料庫通常位於私有子網 (Private Subnet)，並透過 Security Group 限制僅允許您的 K8s Node IP 連線。

## 3. 深度探討：為何 HTTPS 憑證不應像資料庫密碼一樣加密入庫？

雖然 HTTPS 憑證與資料庫帳密都是機密資訊，但將 HTTPS 憑證加密後放入 Git，在現代雲原生維運中通常被視為**反模式 (Anti-pattern)**。

### A. 核心差異：生命週期 (Lifecycle)

| 比較項目 | 資料庫帳密 (DB Password) | HTTPS 憑證 (e.g., Let's Encrypt) |
| :--- | :--- | :--- |
| **性質** | **靜態 (Static)** | **動態 (Dynamic)** |
| **變動頻率** | 極低 (數月或數年一次) | 極高 (每 60-90 天過期) |
| **GitOps 策略** | **加密入庫** (Sealed Secrets) | **不入庫** (使用 cert-manager) |
| **原因** | 因為很少改，人工加密一次放入 Git 很方便。 | 因為常過期，若放入 Git 需每兩個月手動更新加密檔，維運成本過高且易造成服務中斷。 |

### B. 自動化與 GitOps 哲學的衝突

將短效期憑證放入 Git 會破壞自動化流程。

**情境模擬：如果將 Let's Encrypt 憑證加密入 Git**
1.  **Day 1**：您申請了憑證，加密後 Commit 進 Git。ArgoCD 部署成功。
2.  **Day 60**：憑證即將過期。
    *   **正確做法 (cert-manager)**：叢集內的 Controller 自動偵測，自動向 CA 續約，自動更新 Secret。**Git 完全不需要變更**，因為您的「意圖」（我要 HTTPS）沒變。
    *   **錯誤做法 (加密入 Git)**：您必須人工介入：申請 -> 下載 -> 加密 -> Commit -> Push。如果沒改 Git，ArgoCD 會持續確保叢集裡使用的是那張「舊的、加密過的」憑證。
3.  **Day 91**：若忘記人工更新，憑證過期，網站出現安全性警告。

**結論**：將短效期憑證「靜態化」存入版控，是將自動化流程「降級」為人工維運，增加了人為疏失的風險。

### C. 唯一例外：什麼時候可以比照辦理？

只有在以下特殊場景，將憑證加密入 Git 才是合理的選擇：

*   **場景**：**完全隔離的內網環境 (Air-gapped)** 且使用 **長效期商業憑證**。
    *   您的 K8s 叢集完全無法連上 Internet，無法使用 ACME 協定（如 Let's Encrypt）自動驗證。
    *   您購買的是傳統的 OV/EV 憑證，效期長達 1 年以上。
*   **理由**：在這種情況下，憑證的性質變得跟資料庫密碼一樣「靜態」。您確實可以將它視為一個固定的 Secret 檔案，加密後納入版控管理。

## 4. 總結建議

*   **資料庫帳密**：使用 **Sealed Secrets** (加密檔案) 或 **External Secrets** (外部金庫) 管理，確保 Git 中無明碼。
*   **HTTPS 憑證**：使用 **cert-manager** 自動化管理，Git 僅儲存 Ingress 設定 (意圖)，憑證實體由叢集動態產生。
# Jenkins 簡介與部署角色說明

## 1. 什麼是 Jenkins？

**Jenkins** 是一個基於 Java 開發的開源自動化伺服器，是目前 CI/CD（持續整合/持續交付）領域中最主流的工具之一。它透過龐大的插件（Plugins）生態系統，協助開發團隊實現軟體開發生命週期中的自動化流程。

### 核心功能
*   **自動化建置 (Build Automation)**：自動從版本控制系統（如 Git）拉取代碼並編譯。
*   **自動化測試 (Test Automation)**：執行單元測試、整合測試，確保代碼品質。
*   **持續整合 (Continuous Integration)**：頻繁將開發者的代碼合併到主線，並即時回饋建置結果。

---

## 2. Jenkins 可以處理部署 (Deployment) 嗎？

**是的，Jenkins 完全具備處理部署工作的能力。** 

Jenkins 的部署策略主要取決於目標環境是 **「虛擬機/實體機」** 還是 **「容器叢集 (K8s)」**。這兩者的運作邏輯截然不同。

---

## 3. 虛擬機 (VM) 與實體機的部署模式：Push-based

在傳統計算環境中（如 EC2, On-Premise Servers），Jenkins 通常扮演 **「指揮官」** 的角色，採用 **推播式 (Push-based)** 的方式直接控制目標伺服器。

### 運作方式
1.  **CI 階段**：Jenkins 完成編譯與測試，產出執行檔（Artifact，如 `.jar`, `.exe`, `.tar.gz`）。
2.  **傳輸階段**：Jenkins 透過 SSH 將檔案推送到目標伺服器。
3.  **執行階段**：Jenkins 下達指令（或呼叫 Ansible Playbook）停止舊服務、替換檔案、重啟服務。

### 常用工具搭配
*   **Shell Script + SSH**：最簡單直接，適合小型專案。
    *   *流程*：`scp` 上傳檔案 -> `ssh` 遠端執行 `systemctl restart`。
*   **Ansible / SaltStack**：適合中大型專案，具備冪等性 (Idempotency) 與組態管理能力。
    *   *流程*：Jenkins 呼叫 Ansible Playbook -> Ansible 負責所有伺服器的更新與重啟。

### 特點
*   **Jenkins 權限較大**：Jenkins 需要持有目標伺服器的 SSH Key 或連線憑證。
*   **即時性強**：部署指令下達後立即執行，Jenkins Console 可即時看到所有伺服器的回應日誌。

---

## 4. Kubernetes (K8s) 的部署模式：Pull-based (GitOps)

在容器化與 K8s 環境中，為了提升安全性與狀態一致性，業界傾向將 **CI** 與 **CD** 權責分離，採用 **拉取式 (Pull-based)** 模式。

### 運作方式
1.  **Jenkins 負責 CI**：
    *   程式碼掃描與測試。
    *   打包 Docker Image 並上傳至 Image Registry。
    *   **更新 Git 配置倉庫**（例如修改 Helm Chart 或 YAML 中的 Tag 為新版本）。
    *   *注意：Jenkins 不直接連線到 K8s Cluster。*
2.  **ArgoCD 負責 CD**：
    *   ArgoCD 監控 Git 配置倉庫的狀態。
    *   發現版本變更後，**主動拉取 (Pull)** 新設定並同步到 K8s 叢集。

### 特點
*   **安全性高**：Jenkins 不需要持有 K8s Cluster 的 Admin 權限 (Kubeconfig)。
*   **自我修復**：K8s 實際狀態永遠與 Git 保持一致，防止人為誤操作（Configuration Drift）。

---

## 5. 總結與建議選型

| 比較項目 | VM / 實體機部署 | Kubernetes (K8s) 部署 |
| :--- | :--- | :--- |
| **推薦模式** | **Jenkins + Ansible (Push)** | **Jenkins + ArgoCD (GitOps)** |
| **Jenkins 角色** | 全能型 (包辦 CI + CD) | 專注 CI (產出 Image + 更新 Git) |
| **部署發起者** | Jenkins 主動發起 | ArgoCD 偵測 Git 變更後發起 |
| **優點** | 架構簡單直觀，控制力強 | 安全性高，具備版本回滾與漂移偵測能力 |
| **適用場景** | 傳統應用架構、單體式應用 | 微服務架構、容器化應用 |
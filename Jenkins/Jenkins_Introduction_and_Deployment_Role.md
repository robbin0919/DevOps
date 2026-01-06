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

在傳統的自動化流程中，Jenkins 採用 **Push-based（推播式）** 模式來完成部署：

### 常見的部署方式：
*   **腳本部署**：透過 Shell Script、`scp`、`ssh` 等指令將編譯好的檔案傳送到伺服器並重啟服務。
*   **組態管理工具**：整合 Ansible、SaltStack 或 Terraform 等工具來自動化基礎設施的部署。
*   **容器化部署**：執行 `docker-compose up` 或 `kubectl apply` 來更新容器服務。

### 優點：
*   **極高靈活性**：支援各種環境（VM、實體機、雲端平台）。
*   **流程一體化**：從測試到部署都在同一個 Pipeline 中完成，易於追蹤。

---

## 3. 現代化架構中的職責分離：Jenkins + ArgoCD

雖然 Jenkins 可以執行部署，但在現代 Kubernetes 環境中，業界傾向將 **CI (持續整合)** 與 **CD (持續部署)** 權責分離，以提升安全性和穩定性。

### 職責分工：
1.  **Jenkins 負責 CI (持續整合)**：
    *   程式碼掃描與測試。
    *   打包 Docker Image 並上傳至 Image Registry。
    *   **更新 Git 配置倉庫**（例如修改 Helm Chart 或 YAML 中的 Tag）。
2.  **ArgoCD 負責 CD (持續部署)**：
    *   監控 Git 倉庫的狀態。
    *   採用 **Pull-based (拉取式 / GitOps)** 模式，主動同步設定到 K8s 叢集。

### 為什麼要分開？
*   **安全性**：Jenkins 不再需要持有目標環境（如生產環境 K8s）的最高存取權限。
*   **自我修復**：ArgoCD 會持續監控叢集狀態，若有人手動更改 K8s 設定，它會自動將其還原為 Git 上定義的狀態。

---

## 4. 總結與建議

*   **若部署目標為虛擬機 (VM) 或實體機**：Jenkins 是最直觀且強大的選擇，可透過 Pipeline 直接管理部署流程。
*   **若部署目標為 Kubernetes (K8s)**：建議將 Jenkins 定位為 CI 工具，負責產出 Artifact (Image)，並將部署的任務交給專門的 GitOps 工具（如 ArgoCD）處理。

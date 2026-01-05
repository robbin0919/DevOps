# ArgoCD 的監控對象與 CI/CD 職責劃分

在使用 ArgoCD 時，必須釐清一個核心觀念：**ArgoCD 監控的是「部署設定檔 (YAML)」，而非「應用程式原始碼 (Source Code)」。**

## 1. 職責劃分 (Separation of Concerns)

在現代 DevOps 流程中，通常會將「程式碼」與「部署設定」拆分開來：

| 類別 | 應用程式儲存庫 (App Repo) | 部署儲存庫 (Manifest/Config Repo) |
| :--- | :--- | :--- |
| **內容** | Java, Python, Go, Node.js 等原始碼 | K8s YAML, Helm Charts, Kustomize |
| **負責人** | 開發人員 (Developers) | 維運/SRE 人員 (DevOps) |
| **工具** | **CI 工具** (Jenkins, GitHub Actions) | **CD 工具 (ArgoCD)** |
| **產出物** | **Docker Image** (映像檔) | 運行中的 **Kubernetes 資源** |

## 2. 完整工作流程 (Workflow)

1.  **CI 階段 (由 CI Server 執行)**：
    *   開發者推送「程式碼」變動。
    *   CI Server 進行測試與編譯。
    *   CI Server 建立 Docker Image (例如 `myapp:v1.2`) 並推送到 Registry。
    *   **關鍵動作**：CI Server 自動更新「部署儲存庫」中的 YAML 檔案，將 Image Tag 從 `v1.1` 改為 `v1.2`。

2.  **CD 階段 (由 ArgoCD 執行)**：
    *   **ArgoCD 監控部署儲存庫**：發現 YAML 檔案中的 Image 版本有異動。
    *   **同步 (Sync)**：ArgoCD 向 Kubernetes 發出指令，要求更新資源狀態。
    *   **部署完成**：Kubernetes 拉取新版 Image，完成滾動更新。

## 3. 為什麼要這樣設計？

*   **安全性**：ArgoCD 只需要存取部署設定，不需要存取原始碼。
*   **清晰的審核軌跡**：從 Git 紀錄可以清楚看到「是誰在什麼時間點修改了部署版本」，而不僅僅是程式碼的變動。
*   **版本回滾**：若新版本有問題，只需在部署儲存庫執行 `git revert`，ArgoCD 就會立刻將 K8s 回退到上一個穩定的 YAML 狀態。

## 4. 總結
ArgoCD 的眼睛是盯著 **「描述基礎設施與部署狀態的 YAML」**。它是 GitOps 理念的實踐者，確保「Git 裡的 YAML」與「K8s 裡的現實」永遠保持一致。

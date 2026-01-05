# ArgoCD 學習筆記與實踐指南

本目錄收錄了關於 ArgoCD、GitOps 流程以及 Kubernetes 機密管理的相關文件與指南。

## 檔案列表與用途說明

### 基礎概念
*   **[ArgoCD_Introduction.md](./ArgoCD_Introduction.md)**
    *   **內容**：ArgoCD 的核心介紹、功能特點（聲明式 GitOps、漂移偵測）。
    *   **用途**：快速了解 ArgoCD 是什麼以及它解決了什麼問題。

*   **[ArgoCD_vs_VM_Deployment.md](./ArgoCD_vs_VM_Deployment.md)**
    *   **內容**：比較 Kubernetes (ArgoCD) 與傳統 VM 環境 (Ansible/Jenkins) 的部署差異。
    *   **用途**：釐清不同基礎設施下的自動化工具選擇與策略。

*   **[ArgoCD_Responsibilities_and_Workflow.md](./ArgoCD_Responsibilities_and_Workflow.md)**
    *   **內容**：詳解 CI (Jenkins/GitHub Actions) 與 CD (ArgoCD) 的職責劃分，以及 ArgoCD 監控「部署設定檔」而非「原始碼」的核心觀念。
    *   **用途**：建立正確的 DevOps 流水線架構觀念。

### 機密管理 (Secrets Management)
*   **[ArgoCD_Secrets_Management.md](./ArgoCD_Secrets_Management.md)**
    *   **內容**：GitOps 環境下機密管理的總體策略（Sealed Secrets vs External Secrets），以及為何 HTTPS 憑證不應加密入庫的深度探討。
    *   **用途**：制定安全的機密資訊處理規範。

*   **[SealedSecret_Implementation_Example.md](./SealedSecret_Implementation_Example.md)**
    *   **內容**：SealedSecret 的詳細原理、運作流程，以及「加密前 vs 加密後」的實戰 YAML 範例。
    *   **用途**：提供開發者具體的操作指南，將 Secret 安全地轉換為可提交的版本。

### HTTPS 憑證管理
*   **[ArgoCD_HTTPS_Management.md](./ArgoCD_HTTPS_Management.md)**
    *   **內容**：如何結合 Ingress 與 cert-manager 實現 HTTPS 憑證的自動化申請與續約。
    *   **用途**：實作零接觸（Zero-touch）的憑證維運流程，並了解憑證不入 Git 的安全原則。

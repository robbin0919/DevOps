# ArgoCD 介紹

**ArgoCD** 是一款專為 **Kubernetes** 設計的**聲明式（Declarative）GitOps 持續交付（CD）工具**。

## 核心特點

1.  **GitOps 核心理念**
    它將 Git 儲存庫（如 GitHub, GitLab）視為「單一事實來源（Single Source of Truth）」。開發者只需將 Kubernetes 的 YAML 設定檔推送到 Git，ArgoCD 就會自動確保叢集的實際狀態與 Git 中的描述一致。

2.  **自動化同步**
    ArgoCD 會持續監控 Git 上的版本。一旦發現 Git 中的設定有變動，它可以自動或手動地將這些變更應用到 Kubernetes 叢集中。

3.  **狀態視覺化**
    它提供了一個非常直觀的 Web UI，讓維運人員可以清楚看到應用程式的部署狀態（如：Healthy, Synced, OutOfSync）。

4.  **漂移偵測（Drift Detection）**
    如果有人手動修改了 Kubernetes 叢集的資源（非透過 Git），ArgoCD 會偵測到這種「狀態漂移」，並能自動將其修復回 Git 所定義的正確狀態。

5.  **多叢集管理**
    它具備管理多個 Kubernetes 叢集的能力，非常適合現代化的雲原生架構。

## 總結
ArgoCD 讓部署過程變得**透明、可追溯且易於回滾**，是實踐 K8s 自動化部署的主流選擇。

# Jenkins 知識庫

此目錄存放關於 Jenkins 自動化伺服器的相關說明、設定指南與最佳實踐。

## 目錄導覽

- [Jenkins 簡介與部署角色說明](./Jenkins_Introduction_and_Deployment_Role.md)：了解 Jenkins 的基本定義、部署模式（Push-based）以及與 ArgoCD 的協作架構。
- [Jenkins 針對 VM/實體機的安全部署策略](./Secure_VM_Deployment_Strategies.md)：探討如何在不給予 Jenkins 最高權限的情況下，安全地部署至 VM 或實體伺服器（包含最小權限 sudo、Pull 模式與中間協調者模式）。
- [Jenkins + ArgoCD：Kubernetes GitOps 部署流程](./Jenkins_ArgoCD_K8s_Workflow.md)：使用 Mermaid 時序圖詳解 Jenkins (CI) 與 ArgoCD (CD) 如何透過 Git 倉庫進行互動。
- [Jenkins + Ansible：VM/實體機自動化部署流程](./Jenkins_Ansible_VM_Workflow.md)：使用 Mermaid 時序圖說明 Jenkins 如何呼叫 Ansible Playbook 將應用程式部署至 SSH 目標伺服器。
- [Jenkins Push-based 模式權限風險分析](./Jenkins_Push_Privilege_Analysis.md)：深入探討為何不論直接指令或呼叫 Ansible，Jenkins 在 Push 模式下皆具備高權限風險。
- [Jenkins + Ansible Tower (AWX)：高安全性 VM 部署流程](./Jenkins_Ansible_Tower_VM_Workflow.md)：使用 Mermaid 時序圖說明如何透過 Ansible Tower 隔離金鑰，實現最安全的 VM 部署架構。

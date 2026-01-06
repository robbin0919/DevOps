# Jenkins + Ansible：VM/實體機自動化部署流程

此文件使用時序圖說明在虛擬機 (VM) 或實體伺服器環境中，如何結合 Jenkins (CI) 與 Ansible (CD) 實現自動化部署。

## 核心架構概念

*   **Jenkins (指揮官)**：負責排程、建置 Artifact，並保存部署所需的憑證 (SSH Keys) 透過 Credential Binding 注入環境變數。
*   **Ansible (執行者)**：負責實際連線到目標伺服器，執行冪等性 (Idempotent) 的部署腳本 (Playbook)。
*   **Inventory**：定義目標伺服器的清單檔案 (Hosts)。

---

## 部署時序圖 (Sequence Diagram)

```mermaid
sequenceDiagram
    autonumber
    participant Dev as 開發者 (Developer)
    participant Git as Git Repo
    participant Jenkins as Jenkins (Controller)
    participant Artifact as Artifact Repo<br/>(Nexus/S3/Local)
    participant Ansible as Ansible Process<br/>(on Jenkins Node)
    participant VM as Target VM (Production)

    Note over Dev, Jenkins: CI 階段 (持續整合)

    Dev->>Git: 1. Push Code
    Git->>Jenkins: 2. Webhook Trigger
    activate Jenkins
    Jenkins->>Jenkins: 3. Build & Test (編譯/測試)
    Jenkins->>Artifact: 4. Upload Artifact (v1.2.0.zip)
    
    Note over Jenkins, VM: CD 階段 (持續部署) 
    
    Jenkins->>Ansible: 5. Execute Ansible Playbook<br/>(Pass vars: Version=1.2.0)
    activate Ansible
    
    Note right of Ansible: Ansible 讀取 Inventory<br/>並使用 SSH Key 連線
    
    Ansible->>VM: 6. SSH Connect
    activate VM
    Ansible->>VM: 7. Stop Service (systemctl stop app)
    Ansible->>Artifact: 8. Download Artifact (v1.2.0.zip)
    Artifact-->>VM: Transfer File
    Ansible->>VM: 9. Extract & Update Config
    Ansible->>VM: 10. Start Service (systemctl start app)
    Ansible->>VM: 11. Health Check (curl localhost)
    
    VM-->>Ansible: Return Success
    deactivate VM
    
    Ansible-->>Jenkins: Playbook Finished (OK)
    deactivate Ansible
    
    Jenkins-->>Dev: 12. Notify Result (Slack/Email)
    deactivate Jenkins
```

---

## 流程詳細說明

### 1. CI 階段：產出部署包
*   Jenkins 拉取程式碼並進行編譯。
*   測試通過後，將編譯好的檔案（Artifact）上傳到儲存庫（如 Nexus, Artifactory, S3, 或僅暫存在 Jenkins Workspace）。
*   **關鍵點**：Jenkins 必須確定本次部署的「版本號」或「檔案路徑」。

### 2. 觸發 Ansible (Handover)
Jenkins 透過 `Ansible Plugin` 或直接執行 `sh "ansible-playbook ..."` 來啟動部署。
在此步驟，Jenkins 會將重要參數傳遞給 Ansible：
*   **Inventory**：告訴 Ansible 要部署到哪一台機器（例如 `-i hosts/prod`）。
*   **Extra Vars**：傳遞動態參數（例如 `-e "app_version=1.2.0"`）。
*   **Credentials**：Jenkins 負責解密 SSH Private Key 並注入到 Ansible 的執行環境中。

### 3. CD 階段：Ansible 執行部署
Ansible 根據 Playbook 的定義，依序對目標 VM 執行任務：
*   **管理服務**：確保應用程式停止，避免檔案鎖定。
*   **更新檔案**：使用 `get_url` 或 `copy` 模組，將新版 Artifact 下載到 VM 指定路徑。
*   **組態管理**：使用 `template` 模組更新設定檔（如 `app.conf`），這點是 Ansible 的強項。
*   **重啟與驗證**：重啟服務並透過 `uri` 模組檢查服務是否回應 200 OK。

---

## 常見 Jenkinsfile 整合範例

```groovy
pipeline {
    agent any
    environment {
        // 在 Jenkins 憑證管理中設定好的 SSH Key ID
        SSH_KEY = credentials('prod-ssh-key')
    }
    stages {
        stage('Build') {
            steps {
                sh './build.sh' // 產出 app-v1.0.jar
            }
        }
        stage('Deploy to VM') {
            steps {
                // 使用 Ansible Plugin 或 Shell
                sh """
                  ansible-playbook -i inventory/prod deploy.yml \
                  --private-key $SSH_KEY_USR \
                  -e "artifact_path=./target/app-v1.0.jar"
                """
            }
        }
    }
}
```
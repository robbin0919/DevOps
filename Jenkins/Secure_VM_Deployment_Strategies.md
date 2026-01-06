# Jenkins 針對 VM/實體機的安全部署策略

此文件說明如何在不賦予 Jenkins 目標伺服器最高權限 (Root/Admin) 的情況下，安全地將應用程式部署至虛擬機 (VM) 或實體伺服器。

## 策略一：最小權限原則 (Least Privilege)

這是最常見的改良方式。Jenkins 仍使用 SSH 推送 (Push)，但嚴格限制其執行權限。

### 實作步驟

1.  **建立受限帳號**：
    在目標 VM 上建立專用帳號（例如 `deploy-user`），**不要**將其加入 `wheel` 或 `admin` 群組。

2.  **設定 SSH 金鑰**：
    將 Jenkins 的公鑰 (Public Key) 放入 `/home/deploy-user/.ssh/authorized_keys`。

3.  **設定 Sudoers 白名單 (`/etc/sudoers`)**：
    使用 `visudo` 編輯，僅授權特定指令免密碼執行。

    ```bash
    # /etc/sudoers.d/jenkins-deploy
    
    # 允許 deploy-user 執行 cp 和 systemctl restart，且不需要輸入密碼
    deploy-user ALL=(root) NOPASSWD: /usr/bin/cp /tmp/artifact.jar /opt/app/, /bin/systemctl restart my-service
    ```

4.  **Jenkins Pipeline 行為**：
    ```groovy
    sh "scp artifact.jar deploy-user@target-vm:/tmp/"
    sh "ssh deploy-user@target-vm 'sudo cp /tmp/artifact.jar /opt/app/ && sudo systemctl restart my-service'"
    ```

### 優缺點
*   **優點**：架構變動小，易於理解。
*   **缺點**：Jenkins 仍需持有進入伺服器的 SSH Key。若設定不當（如開放 `chmod` 或 `mv`），仍有被提權的風險。

---

## 策略二：Pull-based 架構 (類 GitOps)

此策略完全切斷 Jenkins 對 VM 的連線需求。Jenkins 只負責產出，由 VM 主動更新。

### 實作步驟

1.  **Jenkins 職責**：
    *   編譯程式碼。
    *   將成品 (Artifact) 上傳至中間儲存區（如 Nexus, Artifactory, S3, 或 Azure Blob Storage）。
    *   更新版本資訊文件（如 `latest-version.txt`）。

2.  **VM 職責 (Agent/Cron)**：
    *   在 VM 上設定 Cron Job 或 Systemd Timer。
    *   執行腳本定期檢查版本資訊。

    ```bash
    #!/bin/bash
    # 範例檢查腳本
    CURRENT_VER=$(cat /opt/app/version)
    LATEST_VER=$(curl -s https://my-repo/latest-version.txt)

    if [ "$CURRENT_VER" != "$LATEST_VER" ]; then
        wget https://my-repo/app-$LATEST_VER.jar -O /opt/app/app.jar
        systemctl restart my-service
        echo "$LATEST_VER" > /opt/app/version
    fi
    ```

3.  **進階工具**：
    *   **Ansible-Pull**：讓 VM 定期從 Git 拉取 Playbook 並對自己執行 `localhost` 部署。

### 優缺點
*   **優點**：Jenkins 不需要任何 VM 的連線資訊或金鑰，防火牆可完全阻擋入站連線。
*   **缺點**：部署會有延遲（取決於輪詢頻率），難以即時得知部署結果（需搭配額外的監控回報）。

---

## 策略三：使用中間協調者 (Orchestrator)

引入專門的部署工具來管理金鑰與權限，將 CI (Jenkins) 與 CD (Deployer) 實體隔離。

### 常見工具
*   **Ansible Tower / AWX**
*   **Rundeck**

### 實作流程
1.  **金鑰管理**：將 VM 的 SSH Key 儲存在 Ansible Tower 的加密庫 (Vault) 中，Jenkins **不知道** 這些 Key。
2.  **觸發機制**：
    *   Jenkins 完成建置後，透過 API 呼叫 Ansible Tower 的 Job Template。
3.  **執行部署**：
    *   Ansible Tower 使用其管理的憑證連線至 VM 進行部署。

### 優缺點
*   **優點**：權限管理最完善，具備詳細的稽核紀錄 (Audit Log)。
*   **缺點**：需維護額外的一套系統 (Tower/AWX)，架構較複雜。

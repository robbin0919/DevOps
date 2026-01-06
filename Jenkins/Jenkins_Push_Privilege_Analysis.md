# Jenkins Push-based 模式權限風險分析

此文件深入探討在虛擬機 (VM) 或實體機環境中，採用推播式 (Push-based) 部署時，Jenkins 所面臨的高權限風險問題。

---

## 1. 核心問題：為什麼 Jenkins 權限較大？

在 Push-based 模式下，Jenkins 必須主動發起連線並控制目標伺服器。這意味著 Jenkins 伺服器必須具備 **「進入並操作目標機器」** 的憑證（通常是 SSH Private Key）。

不論您是透過 **直接執行 Shell 指令** 還是 **呼叫 Ansible Playbook**，權限本質上的風險是一致的：

### 情境對照表

| 部署方式 | Jenkins 持有的憑證 | 權限運作邏輯 |
| :--- | :--- | :--- |
| **直接指令 (Shell/SSH)** | SSH Private Key | Jenkins 透過 `ssh -i key.pem user@vm` 登入。 |
| **呼叫 Ansible** | SSH Private Key | Jenkins 將 Key 提供給 Ansible 程序，由 Ansible 發起 SSH 連線。 |

---

## 2. 深入分析：Ansible 整合模式的權限誤區

許多人認為使用 Ansible 可以提升安全性，但在 **Jenkins 呼叫 Ansible CLI** 的架構中，這是一個誤區：

1.  **憑證存儲點**：Ansible 執行所需的 Private Key 仍然儲存在 Jenkins 的 `Credentials Store` 中。
2.  **注入機制**：在 Pipeline 執行期間，Jenkins 會將該 Key 解密並存放在臨時目錄（或以環境變數注入），供 Ansible 使用。
3.  **攻擊路徑**：若 Jenkins 伺服器本身被攻破，駭客可以直接從 Jenkins 內存或配置中提取這些 SSH Key。

**結論：從資安角度來看，Jenkins 直接連線與 Jenkins 呼叫 Ansible 對於「金鑰洩漏」的風險等級是相同的。**

---

## 3. 權限洩漏的連鎖反應 (Blast Radius)

一旦 Jenkins 伺服器遭到入侵，高權限帶來的影響如下：

*   **橫向移動 (Lateral Movement)**：駭客可以利用 Jenkins 持有的 SSH Key 登入所有受控的 VM 伺服器。
*   **持久化後門**：駭客可以在目標 VM 上植入後門，即使 Jenkins 修復了，目標 VM 仍然受控。
*   **資料竊取**：由於部署帳號通常具備讀取設定檔的權限，駭客可以輕易取得資料庫連線字串或其他 API Key。

---

## 4. 緩解與資安建議

若必須使用 Push-based 模式，建議採取以下層次的防禦：

### A. 最小權限 (Least Privilege)
*   **Sudoers 限制**：目標 VM 的部署帳號絕對不應具備 `sudo ALL` 權限，必須嚴格限制僅能執行如 `systemctl restart` 等特定指令。

### B. 金鑰隔離 (Key Isolation)
*   **Ansible Tower / AWX**：將 SSH Key 存放在專門的部署控制器 (Tower) 中。Jenkins 僅持有「呼叫 API」的 Token。即使 Jenkins 被駭，駭客也拿不到 SSH Key。

### C. 網路層防護
*   **IP 白名單**：目標 VM 的 SSH 服務應僅允許來自 Jenkins 伺服器 (Fixed IP) 的連線。

### D. 審核日誌 (Audit Logs)
*   **操作稽核**：確保目標 VM 開啟詳細的 `auth.log`，紀錄所有 sudo 指令與登入行為，並將日誌同步到中央伺服器。

---

## 5. 總結

Push-based 模式讓 Jenkins 像是一位拿著各家門禁卡 (SSH Keys) 的 **「超級管家」**。

*   **優點**：架構簡單、反應快速、適合 VM 環境。
*   **缺點**：Jenkins 成為了單點故障 (Single Point of Failure) 與高價值攻擊目標。

**建議：在具備條件的情況下，盡可能轉向 Pull-based (如 Ansible-pull) 或使用中間層 (Ansible Tower) 來隔離權限。**

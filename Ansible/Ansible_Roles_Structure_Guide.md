# Ansible Roles 結構設計指南

**Ansible Roles** 是 Ansible 用來實現 **「模組化 (Modularity)」** 與 **「重用性 (Reusability)」** 的核心機制。它將 Playbook 中的 Task、Variable、File、Template 拆解並分類存放，是管理大型專案的最佳實踐。

## 1. 標準目錄結構 (Directory Structure)

Ansible 採用「約定優於配置 (Convention over Configuration)」的原則，會自動載入符合特定目錄結構的 `main.yml` 檔案。

假設我們建立一個安裝 Nginx 的 Role，名稱叫 `nginx_server`，其結構如下：

```text
roles/
└── nginx_server/
    ├── tasks/          <-- [核心] 所有的執行步驟 (Playbook 的 tasks 部分)
    │   └── main.yml
    ├── handlers/       <-- [觸發器] 服務重啟等動作 (Service restart)
    │   └── main.yml
    ├── templates/      <-- [模板] Jinja2 設定檔 (如 nginx.conf.j2)
    ├── files/          <-- [檔案] 靜態檔案 (不需要變數替換的檔案)
    ├── vars/           <-- [變數] Role 內部的高優先級變數 (通常不建議使用者修改)
    │   └── main.yml
    ├── defaults/       <-- [預設值] 預設變數 (優先級最低，方便使用者覆蓋)
    │   └── main.yml
    ├── meta/           <-- [中繼資料] 相依性設定 (例如依賴 common role)
    │   └── main.yml
    └── tests/          <-- [測試] 用來測試這個 Role 的腳本
```

---

## 2. 目錄職責詳解

| 目錄名稱 | 必要性 | 用途說明 |
| :--- | :---: | :--- |
| **tasks** | **必須** | Role 的主程式入口。所有的 `yum install`, `copy`, `service` 指令都寫在 `tasks/main.yml` 中。 |
| **handlers** | 選用 | 存放被觸發的動作。例如：當設定檔改變時 (`notify: restart nginx`)，觸發 `systemctl restart nginx`。 |
| **templates** | 選用 | 存放 `.j2` 結尾的模板檔。Ansible 會將變數填入後，產生最終設定檔。 |
| **files** | 選用 | 存放靜態檔案（憑證、二進位檔）。透過 `copy` 模組直接複製到目標機器。 |
| **defaults** | 選用 | **「建議變數」**。這裡定義的變數優先權最低。使用者可以在 Playbook 中輕鬆覆蓋這些值（例如 `port: 80` 改為 `8080`）。 |
| **vars** | 選用 | **「內部變數」**。這裡定義的變數優先權較高，通常用於定義 Role 內部運作所需的固定參數，不希望使用者隨意修改。 |
| **meta** | 選用 | 定義 Role 的依賴關係 (Dependencies)。例如安裝 Nginx 前必須先執行 `common` Role。 |

---

## 3. 如何使用 Role？

一旦定義好 Role，您的主 Playbook (`site.yml`) 就會變得非常乾淨且易讀：

```yaml
# site.yml
- hosts: web_servers
  roles:
    # 引用 common role，執行基礎設定
    - role: common
    
    # 引用 nginx_server role，並覆蓋預設變數
    - role: nginx_server
      vars:
        nginx_port: 8080    # 覆蓋 defaults/main.yml 中的預設值
```

---

## 4. 設計最佳實踐 (Best Practices)

### 單一職責原則 (Single Responsibility Principle)
一個 Role 只做一件事。
*   **Bad**: 建立一個 `setup_web_db_cache` Role。
*   **Good**: 拆分成 `nginx`, `mysql`, `redis` 三個獨立的 Role。

### 變數命名空間 (Namespacing)
為了避免變數名稱在全域衝突，Role 內的變數應加上前綴。
*   **Bad**: 在 `nginx` role 中定義變數 `port: 80`。
*   **Good**: 定義變數 `nginx_port: 80`。

### 善用 Defaults vs Vars
*   **defaults/main.yml**: 放置**「希望使用者可以修改」**的參數（如 Port, 安裝版本）。
*   **vars/main.yml**: 放置**「系統運作邏輯強相關」**的參數（如 OS 特定的套件名稱對照表），不建議使用者修改。

### 保持 Tasks 簡潔
如果 `tasks/main.yml` 超過 100 行，建議使用 `include_tasks` 或 `import_tasks` 將邏輯拆分到不同檔案中（例如 `install.yml`, `configure.yml`, `service.yml`）。

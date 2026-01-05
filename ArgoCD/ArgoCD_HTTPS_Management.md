# ArgoCD HTTPS 憑證自動化管理

在 Kubernetes 環境中，ArgoCD 的 HTTPS 憑證更新通常不建議手動處理，而是透過 **cert-manager** 結合 **Ingress Controller** 實作自動化生命週期管理。

## 1. 核心架構：TLS Termination
最標準的做法是在 **Ingress 層** 進行 TLS 終止（TLS Termination）：
*   **外部流量 (HTTPS)** -> **Ingress (解密/掛載憑證)** -> **內部流量 (HTTP)** -> **ArgoCD Server**

## 2. 自動化組件：cert-manager
**cert-manager** 是 K8s 憑證管理的標準工具，負責：
1.  **監控**：觀察 Ingress 設定。
2.  **申請**：向 CA（如 Let's Encrypt）自動申請憑證。
3.  **更新**：在過期前自動續約（Renew）並更新 K8s Secret。

## 3. 實作設定範例

### A. 定義 Issuer (憑證頒發者)
建立一個 `ClusterIssuer` 資源，設定 ACME 伺服器與驗證方式：

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: user@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

### B. 設定 ArgoCD Ingress
透過 Annotation 關聯 cert-manager，它會自動將簽發的憑證存入 `secretName` 指定的 Secret 中：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod" # 使用上述 Issuer
    nginx.ingress.kubernetes.io/ssl-passthrough: "false"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - argocd.yourdomain.com
    secretName: argocd-server-tls # cert-manager 自動管理此 Secret
  rules:
  - host: argocd.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 80
```

### C. 調整 ArgoCD Server (Insecure Mode)
由於 HTTPS 已在 Ingress 處理完成，需修改 ArgoCD Deployment 參數，讓 Server 以 HTTP 模式運行：
*   在 `argocd-server` 的啟動參數中加入 `--insecure`。

## 4. 維運優勢
*   **零手動介入**：憑證即將過期時，cert-manager 會自動處理續約。
*   **無縫更新**：Ingress Controller 會在 Secret 更新時自動重新載入，服務不中斷。
*   **集中管理**：所有憑證狀態皆可透過 `kubectl get certificate -A` 進行監控。

## 5. 安全性與 GitOps 注意事項 (重要)
**嚴格禁止**將產生的憑證檔案（`tls.crt`, `tls.key`）或含有明碼 Secret 的 YAML 提交至 Git 版本控制系統。

*   **安全性風險**：提交 Private Key 等同於將加密鑰匙公開，極易遭受中間人攻擊。
*   **與自動化衝突**：`cert-manager` 會定期自動輪替（Rotate）憑證。若 Git 中存在靜態的 Secret YAML，ArgoCD 可能會將過期的憑證強制覆蓋回叢集，導致服務中斷。

**正確做法**：
Git 僅儲存 `Ingress` 與 `Issuer` 的設定（意圖），實際的 `Secret` 由 `cert-manager` 在叢集內動態產生並管理（狀態），不應納入 Git 版控。
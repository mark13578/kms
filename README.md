# kms

**KMS 授權系統詳細設計文件**

---

### **1. 系統架構概述**

**1.1 系統組成：**

- **客戶端核心 DLL：** 負責硬體資訊收集、授權驗證、API 通訊。
- **授權伺服器 API：** 提供授權啟用、驗證、撤銷等服務。
- **管理後台：** 提供使用者管理、產品管理、授權管理等功能，基於 Vue-admin-template。
- **資料庫：** 儲存使用者資料、產品資訊、授權紀錄等。

**1.2 技術選型：**

- 後端：.NET Core Web API
- 前端：Vue-admin-template + Element UI
- 即時通訊：SignalR
- 認證機制：JWT Bearer
- 資料庫：MySQL

---

### **2. 硬體資訊綁定策略**

**2.1 綁定項目：**

- CPU ID
- 主機板序號（Motherboard Serial Number）
- 硬碟序號（Disk Serial Number）
- MAC Address
- BIOS UUID / System UUID
- 記憶體條序號（RAM Serial Number）

**2.2 Hardware ID 生成：**

- 收集上述硬體資訊後，進行 SHA-256 雜湊處理，並加上隨機 salt 增強安全性。

---

### **3. 授權處理流程**

**3.1 授權申請與核發：**

1. **使用者提交申請** （API：`POST /api/license/request`）：
   - 提供必要資訊（如硬體指紋、產品選擇、客戶資料等）。

2. **伺服器生成序號與憑證** （API：`POST /api/license/generate`）：
   - 勾選授權產品模組，產生唯一序號與加密授權憑證。

3. **憑證自動下載** （API：`GET /api/license/download/{licenseId}`）：
   - 伺服器發出已加密憑證，客戶端自動下載。

4. **客戶端授權驗證** （API：`POST /api/license/validate`）：
   - 客戶端核心 DLL 透過加密演算法解密憑證，驗證授權是否合法。
   - 驗證成功：授權使用功能；失敗則記錄異常 Log（API：`POST /api/license/log`）。

5. **短期憑證更新** （API：`POST /api/license/refresh`）：
   - 每 7 天自動聯網更新憑證。
   - 若超過 7 天未更新，提示：「⚠️ 您的客戶端已經斷線7天了，請重新聯網，以取得使用權。」

---

### **4. 授權金鑰設計**

**4.1 授權金鑰格式：**

- **格式範例：** `ABCDE-12345-FGHIJ-67890-KLMNO`
- **組成規則：** 英數混合（`A-Z` + `0-9`），共 25 位元，分為 5 組，每組 5 字元，以 `-` 分隔。
- **安全性設計：**
  - 基於硬體指紋與授權資訊加密後生成，避免使用者察覺硬體綁定。
  - 使用 AES-256 加密 + Base32 編碼。

**4.2 授權金鑰流程：**

1. **伺服器生成授權金鑰** （API：`POST /api/license/generate-key`）：
   - 包含硬體指紋與授權資訊，進行加密後轉換為 25 位元授權金鑰。

2. **客戶端輸入授權金鑰進行驗證** （API：`POST /api/license/validate-key`）：
   - 解密並比對硬體指紋，確保授權無法轉移至其他設備。

---

### **5. API 設計規範**

| **API 端點**                    | **HTTP 方法** | **描述**               |
|:---------------------------------|:--------------|:------------------------|
| `/api/license/request`          | POST          | 提交授權申請             |
| `/api/license/generate`         | POST          | 生成授權序號與憑證         |
| `/api/license/download/{id}`    | GET           | 下載授權憑證              |
| `/api/license/validate`         | POST          | 客戶端授權憑證驗證          |
| `/api/license/refresh`          | POST          | 更新短期憑證               |
| `/api/license/log`              | POST          | 記錄授權異常日誌            |
| `/api/license/generate-key`     | POST          | 生成授權金鑰               |
| `/api/license/validate-key`     | POST          | 驗證授權金鑰有效性            |

---

### **6. 資料庫設計**

**6.1 主要資料表：**

- `Users`：使用者帳號資訊
- `Products`：產品 SKU 與授權設定
- `Licenses`：授權碼、授權狀態、綁定硬體資訊、加密憑證
- `License_Logs`：授權啟用與異常記錄

**6.2 授權表調整：**

```sql
ALTER TABLE licenses
ADD COLUMN license_key VARCHAR(29) NOT NULL UNIQUE, -- 25 位元 + 4 個分隔符
ADD COLUMN certificate TEXT,
ADD COLUMN last_synced_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP;
```

**6.3 授權日誌表：**

```sql
CREATE TABLE IF NOT EXISTS license_logs (
    id CHAR(36) PRIMARY KEY,
    license_id CHAR(36) NOT NULL,
    log_type ENUM('ValidationFailure', 'NetworkDisconnected') NOT NULL,
    message TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (license_id) REFERENCES licenses(id) ON DELETE CASCADE
);
```

---

### **7. 安全性設計**

- **加密技術：** 採用 AES-256 + RSA 雙層加密，確保授權資料安全。
- **憑證簽章：** 使用非對稱加密進行數位簽章，防止偽造授權。
- **防破解機制：** 客戶端 DLL 代碼混淆，防止逆向工程。
- **異常偵測：** 當授權碼於多台設備啟用時，自動撤銷授權並記錄異常。

---

### **8. 未來擴展性**

- 與 VM 管理、帳務系統 API 界接，提供統一授權平台
- 支援多語系與國際化
- 提供 SDK，方便第三方應用整合

---

### **9. 參考網站**

- [Maskoid 授權管理系統](https://maskoid.com/license-key-management/)

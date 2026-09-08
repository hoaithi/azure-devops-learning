# Topic 02: Azure DevOps Secret Variables & Environment Mapping

## 1. Conceptual Map

- **What:** Cơ chế quản lý, bảo vệ và truy xuất biến bí mật (Secret Variables) trong Azure DevOps Pipelines.
- **Why:** 
  - Ngăn ngừa rò rỉ bí mật (credentials, API keys, connection strings) ra ngoài log thực thi (tự động mask thành `***`).
  - Chặn các script/công cụ độc hại của bên thứ 3 trong quá trình build tự động đánh cắp biến môi trường hệ thống.
- **When:** Bất kỳ khi nào pipeline cần làm việc với API Key, mật khẩu, Token cá nhân (PAT), SSH Key hoặc thông tin nhạy cảm từ Azure Key Vault.
- **Relationship:** Azure DevOps Pipeline (YAML) <-> Pipeline Variables / Variable Groups / Azure Key Vault <-> Agent Process Environment (`env:` block).
- **Microsoft Learn Keywords:** `Azure Pipelines secret variables`, `Explicit environment mapping`, `Masking secrets in logs`, `Variable groups Azure Key Vault`.

---

## 2. Technical Support Perspective

### Symptoms & Evidence
1. **Symptom 1:** Log hoặc HTTP request gửi đi mang giá trị `undefined` hoặc rỗng (ví dụ: `X-RapidAPI-Key: undefined`).
2. **Symptom 2:** Trong script in ra biến môi trường nhưng kết quả trả về khoảng trắng / null.
3. **Symptom 3:** Cố gán `VAR_NAME: $(SECRET_NAME)` trong khối `variables:` toàn cục nhưng khi chạy step thì `VAR_NAME` bị rỗng.

### Common Root Causes
- **Missing Explicit `env:` mapping:** Lập trình viên quen với biến thường (tự động inject vào môi trường hệ thống) nên không khai báo khối `env:` trong step.
- **Root `variables:` expansion limitation:** Secret variables không bao giờ được mở rộng (expand) bên trong khối `variables:` cấp cao nhất của pipeline.
- **Case-sensitivity / Typo:** Tên biến trong `$(MY_SECRET)` bị sai chính tả so với tên đã lưu trên giao diện Web / Variable Group.

### Resolution
- Khai báo explicit mapping trong khối `env:` của task/script:
  ```yaml
  - script: |
      npm run build
    env:
      REACT_APP_API_KEY: $(MY_SECRET_KEY)
  ```
- Nếu dùng task có sẵn (như `AzureWebApp@1`), có thể truyền trực tiếp `$(MY_SECRET_KEY)` vào các tham số `inputs:` của task.

---

## 3. Critical Feedback & Adjustments (What to adjust)

### Support Engineer Mental Model:
- **Nguyên tắc "Zero Trust" của CI/CD Agent:** Không bao giờ tin tưởng môi trường build. Chỉ cấp phát bí mật đúng nơi, đúng lúc (Least Privilege / Need-to-know basis).
- **Phân biệt 3 cú pháp truy cập biến trong Azure Pipelines:**
  1. `$(varName)`: **Macro syntax** (được thay thế trước khi task thực thi, áp dụng cho task inputs).
  2. `$env:VAR_NAME` / `$VAR_NAME`: **Runtime environment variable** (chỉ có khi được inject hoặc khai báo qua `env:`).
  3. `$[variables.varName]`: **Template / Runtime expression**.

---

## 4. Action Items & Next Steps
- [ ] Nắm vững cách giải thích cho khách hàng về lý do tại sao Secret Variable không tự động xuất hiện trong `$VAR`.
- [ ] Thực hành tích hợp Azure Key Vault với Azure DevOps Variable Group trong các bài lab tiếp theo.

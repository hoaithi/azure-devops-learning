# Case Study: Troubleshooting Azure App Service Deployment (404 ARM Blade vs 503 Service Unavailable)

## 1. Conceptual Map

- **What:** Sự cố triển khai ứng dụng React/SPA lên Azure App Service Linux (Node.js Runtime) thông qua Azure DevOps Pipelines dẫn đến lỗi HTTP 503 Service Unavailable và lỗi 404 NotFoundAssetBlade trong Azure Portal.
- **Why:** Phân biệt rõ ràng giữa 2 mặt phẳng: **Control Plane (Management Plane/ARM)** và **Data Plane (Runtime/Application Plane)**.
  - Control Plane quản lý vòng đời tài nguyên (Resource ID, Slots, RBAC).
  - Data Plane xử lý lưu lượng và thực thi mã nguồn (Container boot, Port binding, Web Server process).
- **When to Apply:** Khi pipeline CI/CD báo "Success" (Green) nhưng ứng dụng không thể truy cập hoặc Azure Portal báo lỗi không tìm thấy tài nguyên.
- **Relationship:** Azure DevOps Pipeline (`AzureRmWebAppDeployment@5`) -> Azure Resource Manager (ARM API) -> Azure App Service Linux (Oryx / PM2 container) -> React App (Static build).
- **Microsoft Learn Keywords:** `Azure App Service Linux startup command`, `PM2 serve SPA`, `Oryx build manifest`, `HTTP 503 Service Unavailable App Service`, `ENOENT package.json`.

---

## 2. Technical Support Perspective

### Symptoms & Evidence
1. **Control Plane Symptom:** `HubsExtension / NotFoundAssetBlade / Error 404`: Thường do URL/link điều hướng trỏ tới một Deployment Slot không tồn tại trong ARM resource tree (`.../slots/<slot-id>`).
2. **Data Plane Symptom:** Trình duyệt trả về `HTTP 503 Service Unavailable`.
3. **Log Stream Evidence:**
   ```text
   cd "/home/site/wwwroot"
   PATH="$PATH:/home/site/wwwroot" npm start
   npm error code ENOENT
   npm error enoent Could not read package.json: Error: ENOENT: no such file or directory, open '/package.json'
   ```

### Common Root Causes
- **Missing `package.json` in root:** Pipeline chỉ deploy thư mục `build/` (đầu ra của `npm run build`), bên trong không có file `package.json`. Node.js runtime mặc định chạy `npm start` nên crash ngay lập tức.
- **Wrong Server Paradigm for SPA:** Triển khai static frontend (React/Vite) lên một dynamic runtime container (Node.js) mà không có static server (như PM2, Nginx, serve).
- **3rd-Party API Rate Limit / Quota Exhaustion:** Gói miễn phí bên thứ 3 (RapidAPI) hết lượt gọi (`429 Too Many Requests / Quota Exceeded`), làm React loading spinner quay vô tận dù hạ tầng Azure hoàn toàn khỏe mạnh.

### Resolution
- Cấu hình **Startup Command** cho App Service:
  ```bash
  pm2 serve /home/site/wwwroot --no-daemon --spa
  ```
- Phân lập lỗi: Sử dụng **F12 Developer Tools** (Console / Network) để phân biệt lỗi Hạ tầng (Azure/Pipeline) và lỗi Dịch vụ ngoài (3rd-party API).
- Quản lý API Key bảo mật thông qua **Azure DevOps Pipeline Secret Variables**.

---

## 3. Critical Feedback & Adjustments (What to adjust)

### What was understood well:
- Nhận biết được vấn đề nằm ở tầng runtime / startup khi thấy lỗi 503.
- Biết cách lấy và đọc `Log stream` từ Azure Portal để cung cấp bằng chứng chính xác.
- Sử dụng thành thạo F12 DevTools để trích xuất payload lỗi từ Network response.

### Misconceptions & Gaps:
- **Nhầm lẫn giữa 404 ARM Resource Blade và 404 Web Not Found:** 
  - Lỗi `NotFoundAssetBlade` là của Azure Portal không tìm thấy ARM resource ID.
  - Lỗi web app không chạy là 503 Service Unavailable.
- **Hiểu lầm về hành vi của Node.js App Service:** Node.js App Service không tự động phục vụ file HTML tĩnh như IIS/Nginx nếu không được cấu hình lệnh phục vụ (server process).
- **Phân biệt Build-time vs Runtime Variables:** React SPA cần inject biến môi trường tại bước `npm run build` trên Agent, không thể đọc trực tiếp App Settings trên Azure Portal lúc runtime như Backend.

### Support Engineer Communication Rule:
- Luôn kiểm tra file thực tế được deploy tại `/home/site/wwwroot` qua SSH / Kudu console trước khi kết luận.
- Khi gặp lỗi UI không load dữ liệu, cô lập lỗi theo thứ tự: **Pipeline CI/CD -> Azure Infrastructure -> Web Server Container -> Client-side App -> 3rd-Party Dependencies**.
- Định hướng khách hàng lựa chọn dịch vụ tối ưu: Nếu là static frontend thuần, đề xuất **Azure Static Web Apps** thay vì Web App Linux để tiết kiệm chi phí và tối ưu hiệu năng.

---

## 4. Action Items & Next Steps
- [x] Sửa lỗi 503 bằng Startup Command `pm2 serve /home/site/wwwroot --no-daemon --spa`.
- [x] Bắt đúng lỗi 3rd-party API Quota Exceeded qua F12 Network tab.
- [ ] Đổi API Key mới trên RapidAPI / Google YouTube Data API v3.
- [ ] Truyền biến API Key qua Pipeline Secret Variables vào bước `npm run build`.
- [ ] Chuyển đổi Pipeline sang dạng Multi-stage CI/CD chuẩn production.

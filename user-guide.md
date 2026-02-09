# Hướng Dẫn Cho User

Bạn là người dùng muốn expose dịch vụ local của mình ra internet? Làm theo 3 bước sau:

## Bước 1: Đăng ký tài khoản

1. Truy cập vào địa chỉ server mà admin cung cấp (ví dụ: `http://tunnel.yourcompany.com:9000`)
2. Nhấn nút **"Create Account"**
3. Nhập username và password của bạn
4. Đăng ký thành công!

## Bước 2: Cấu hình Tunnel Mapping

Sau khi đăng nhập, bạn sẽ thấy giao diện **Remote Config**:

1. Nhấn **"+ Add New Rule"** để thêm mapping mới
2. Điền thông tin:
   - **Subdomain**: Tên miền con bạn muốn (vd: `myapp`)
   - **Local Target**: Địa chỉ dịch vụ đang chạy trên máy bạn (vd: `http://localhost:3000`)
3. Nhấn **"Apply Configuration"**

**Ví dụ:**
- Subdomain: `blog` → Local: `http://localhost:4000`
- Kết quả: Người khác có thể truy cập `http://blog.tunnel.yourcompany.com` để vào blog của bạn

## Bước 3: Chạy Agent

1. **Lấy token**: Sao chép **Agent Token** từ portal (hiển thị ở cuối trang dashboard)

2. **Tải Agent**: Download file `agent.exe` (Windows) hoặc `agent-linux` phù hợp với OS của bạn

3. **Chạy lệnh**:
   ```bash
   # Windows
   agent.exe -server=tunnel.yourcompany.com:8000 -token=YOUR_TOKEN_HERE

   # Linux/Mac  
   ./agent-linux -server=tunnel.yourcompany.com:8000 -token=YOUR_TOKEN_HERE
   ```

4. **Xác nhận**: Quay lại portal, status sẽ đổi từ "OFFLINE" sang **"AGENT ONLINE"** 🟢

## Truy cập dịch vụ của bạn

Giờ đây, dịch vụ local của bạn đã được expose! Người khác có thể truy cập qua:

```
http://<subdomain>.<base-domain>
```

Ví dụ: `http://myapp.tunnel.yourcompany.com`

---

## Thay đổi cấu hình

Bạn có thể thay đổi mappings bất cứ lúc nào:
1. Đăng nhập vào portal
2. Thêm/Xóa/Sửa rules
3. Nhấn "Apply Configuration"
4. **Khởi động lại agent** để áp dụng cấu hình mới

---

## Câu hỏi thường gặp

**Q: Agent báo "connection refused"?**  
A: Kiểm tra địa chỉ server và port có đúng không. Hỏi admin để xác nhận.

**Q: Tôi có cần mở port trên router không?**  
A: Không! Tunnel sử dụng kỹ thuật reverse connection, không cần mở port firewall.

**Q: Agent tắt thì sao?**  
A: Khi agent tắt, tunnel sẽ không hoạt động. Người truy cập sẽ gặp lỗi 502/503.

**Q: Tôi có thể dùng HTTPS không?**  
A: Liên hệ admin để được cấp subdomain với HTTPS.

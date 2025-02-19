# 📌 HTTP/2 - Tổng quan và Bảo mật  

## 🔍 1. HTTP/1.1 vs HTTP/2  
- **HTTP/1.1**: Sử dụng văn bản thuần (`text-based`), dễ đọc nhưng kém hiệu quả.  
- **HTTP/2**: Sử dụng giao thức **nhị phân** (`binary`), nhanh hơn và tối ưu hơn.  

## 🚀 2. Cấu trúc Request trong HTTP/2  
### **📌 Đặc điểm quan trọng của HTTP/2 Request**  
 HTTP/2 có **các pseudo-header** đặc biệt, tất cả đều bắt đầu bằng dấu 
   - Ví dụ: `:method`, `:path`, `:authority`, `:scheme`  
 Sau các pseudo-header, request có thể chứa các header thông thường như:  
   - `user-agent`, `content-length`, `accept-encoding`  

### **📌 Sự khác biệt trong cách phân tách Header và Body**  
- **HTTP/1.1**: Dùng `\r\n` để phân cách giữa header và body.  
- **HTTP/2**: Không dùng ký tự đặc biệt, mà chia request thành **các frame**.  
  - Mỗi frame có **trường độ dài (length field)** để xác định kích thước.  

## ⚠️ 3. HTTP Request Smuggling và HTTP/2  
### **❌ Lỗ hổng Request Smuggling trong HTTP/1.1**  
- Do sự **mơ hồ trong cách xác định kích thước request body**.  
- Có **2 cách chính** để xác định độ dài request trong HTTP/1.1:  
  1️⃣ `Content-Length`  
  2️⃣ `Transfer-Encoding: chunked`  

➡️ Nếu proxy và backend hiểu khác nhau về điểm kết thúc request, hacker có thể **chèn thêm request ẩn**.  

📌 **Ví dụ request smuggling trong HTTP/1.1**  
```http
POST / HTTP/1.1
Host: vulnerable.com
Content-Length: 13
Transfer-Encoding: chunked

0

SMUGGLED DATA

### 📌 HTTP/2 - Tổng quan

#### 🔹 HTTP/1.1 vs HTTP/2
- **HTTP/1.1**: Dùng văn bản thuần (`text-based`), chậm hơn.  
- **HTTP/2**: Dùng nhị phân (`binary`), tối ưu hơn.  

#### 🔹 Cấu trúc Request trong HTTP/2  
- Có **pseudo-headers** đặc biệt (`:method`, `:path`, `:authority`...)  
- Headers thông thường (`user-agent`, `content-length`...)  
- Không dùng `\r\n` để phân tách, mà dùng **frame** có độ dài xác định.  

---

### ⚠️ Request Smuggling và HTTP/2  
#### ❌ Lỗ hổng Request Smuggling trong HTTP/1.1  
- HTTP/1.1 có hai cách xác định độ dài request:  
 1. `Content-Length`  
 2. `Transfer-Encoding: chunked`  

📌 **Ví dụ Request Smuggling (HTTP/1.1)**  
```http
POST / HTTP/1.1  
Host: victim.com  
Content-Length: 13  
Transfer-Encoding: chunked  

0  
SMUGGLED DATA
```
- Để tránh mơ hồ HTTP/2 đính kèm **một trường kích thước** cho mỗi phần của request, giúp bộ phân tích(**parse**) biết chính xác mà đọc.
📌 Tại HTTP/2 vẫn có thể bị Request Smuggling?
HTTP/2 không sử dụng Content-Length hay Transfer-Encoding để xác định độ dài request.
Nhưng nếu load balancer/reverse proxy nhận HTTP/2 rồi chuyển thành HTTP/1.1, nó phải dịch request sang định dạng HTTP/1.1.




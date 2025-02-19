- Trước khi đi vào giải quyết các bài lab ta hãy tìm hiểu xem HTTP/2 là gì, và tại sao nó lại có thể bị lỗ hổng Request Smuggling.
### 📌 HTTP/2 - Tổng quan.  

**HTTP/1.1 vs HTTP/2**.  
- **HTTP/1.1**: Dùng văn bản thuần (`text-based`), chậm hơn.  
- **HTTP/2**: Dùng nhị phân (`binary`), tối ưu hơn.  

**Cấu trúc Request trong HTTP/2**.    
- Có **pseudo-headers** đặc biệt (`:method`, `:path`, `:authority`...).   
- Headers thông thường (`user-agent`, `content-length`...).  
- Không dùng `\r\n` để phân tách, mà dùng **frame** có độ dài xác định.  

---

### ⚠️ Request Smuggling và HTTP/2.    
#### ❌ Lỗ hổng Request Smuggling trong HTTP/1.1.  
- HTTP/1.1 có hai cách xác định độ dài request:  
 1. `Content-Length`  
 2. `Transfer-Encoding: chunked`  

**Ví dụ Request Smuggling (HTTP/1.1)**.    
```
    POST / HTTP/1.1  
    Host: victim.com  
    Content-Length: 13  
    Transfer-Encoding: chunked  
    
    0  
    SMUGGLED DATA
```
- Để tránh mơ hồ HTTP/2 đính kèm **một trường kích thước** cho mỗi phần của request, giúp bộ phân tích (**parse**) biết chính xác mà đọc.  
### 📌 Tại sao HTTP/2 vẫn có thể bị Request Smuggling?.    
- HTTP/2 không sử dụng `Content-Length` hay `Transfer-Encoding` để xác định độ dài request.  
- Nhưng nếu **load balancer/reverse proxy** nhận HTTP/2 rồi chuyển thành HTTP/1.1, nó phải dịch request sang định dạng HTTP/1.1.
### 📌 HTTP/2 Downgrading.  
- Khi một **reverse proxy** phục vụ nội dung cho users bằng **HTTP/2** (để kết nối frontend) nhưng lại yêu cầu dữ liệu từ backend bằng **HTTP/1.1**, thì  đó là **HTTP/2 downgrading**.  
![image](https://github.com/user-attachments/assets/c21470d4-c1b5-4077-a108-c00d6050d2f0).    
- **HTTP/2** thì có header :authority nó sẽ được chuyển thành header Host khi chuyển sang **HTTP/1.1**.  
### 📌 H2.CL – HTTP/2 Request Smuggling qua Content-Length.    
- Tại sao Content-Length vẫn có thể gây vấn đề trong HTTP/2?.  
- Nếu hệ thống downgrading từ HTTP/2 sang HTTP/1.1, proxy sẽ chuyển  request (bao gồm cả header Content-Length) sang HTTP/1.1.
### ⚠️ Tấn công Desync Attack qua H2.CL.  
- Ví dụ hacker gửi request HTTP/2:
  ```
     :method  POST
     :path  /api/login
     :authority  victim.com
     content-length  0
     
     HELLO
  ```
- Sau đó nó sẽ bị chuyển qua HTTP/1.1 để nhận dữ liệu.  
  ```
  POST /api/login HTTP/1.1
  Host: victim.com
  content-Length: 0
  
  HELLO
  ```
- Vì lúc này Content-Length = 0 nên Backend sẽ nghĩ là không có body ở request và chờ 1 request mới.
- Sau đó khi người dùng gửi request khác thì nó sẽ bị nối vào => gây ra lỗi Request Smuggling.  
### 📌 H2.TE HTTP/2 Request Smuggling qua Transfer-Encoding.  
- Về TE thì HTTP/2 không sử dụng nhưng nếu proxy/load balancer không kiểm tra đầu vào thì ta có thể thêm vào.
- Ví dụ hacker gửi request HTTP/2:
  ```
  :method  POST
  :path  /api/login
  :authority  victim.com
  transfer-encoding  chunked
  content-length  6
  
   5
   HELLO
   0
   ```
- Sau đó nó sẽ bị chuyển qua HTTP/1.1 để nhận dữ liệu.
   ```
   POST /api/login HTTP/1.1
   Host: victim.com
   Transfer-Encoding: chunked
   Content-Length: 6
   
   5
   HELLO
   0
   ```
- Nếu Backend ưu tiên TE thì nó sẽ bỏ qua CL và đọc tới số 0. Sau đó nó sẽ xử lí CL vẫn còn nữa.
- Thì nó sẽ chờ tiếp phản hồi request khác => bị ghép chung vào với request này. => gây ra **Smuggling**.
### 📌 CRLF Injection.  
- **CRLF là gì?**.  
- CRLF = Carriage Return (CR) + Line Feed (LF), tức là \r\n.  
- Các kí tự này dùng để phân tách trong HTTP/1.1.
- **Tại sao lại xảy ra lỗ hổng CRLF ở HTTP/2?**.  
- Đó là do HTTP/2 là nhị phân nên nó có thể bị chèn bất kỳ kí tự nào vào thẳng request.
- Nếu hacker chèn các kí tự này vào thì có thể làm cho khi drowngrade sang HTTP/1.1 bị sai dẫn đến lỗi **Smuggling**.
- Ví dụ hacker gửi request HTTP/2:
```
  :method  GET
  :path  /api/user
  :authority  victim.com
  x-custom-header  attack\r\nInjected-Header: evil
```
- X-custom-header là 1 head do người dùng tùy chỉnh, nó thường bắt đầu bằng (X).
- Backend sẽ bị hiểu nhầm do kí tự phân tách lúc này sẽ thành.
```
  GET /api/user HTTP/1.1
  Host: victim.com
  X-Custom-Header: attack
  Injected-Header: evil
```
- => Dẫn đến lỗi **Smuggling**.  
- ### **📌 LAB 1 - HTTP2/DESYNC**.
- Sau khi đã hiểu sơ qua về HTTP/2 và Smuggling giờ ta sẽ đi vào làm lab.
- Đầu tiên khi truy cập vào đường link giao diện hiện lên như sau.  
![image](https://github.com/user-attachments/assets/031a9e02-30d8-42db-bc38-6b94b61b428e)  
- Quan sát thấy có một bài viết, ta có thể chọn like hoặc dislike.  
- Ta sẽ xem cách ứng dụng hoạt động:
- Insecpt để kiểm tra thấy web lưu 1 sesion cookie để định danh người dùng.  
![image](https://github.com/user-attachments/assets/08536186-1b34-4980-aa53-86a0154aa94f).  
- Thử ấn like thì thấy 1 request GET HTTP/2 được gửi đi. 
![image](https://github.com/user-attachments/assets/b89886cb-9856-4614-bd58-bdfd72a08755).
- Trong đó có PostId ta có thể thử gửi 1 request với PostId khác để bắt người dùng like bài viết đó mà không hề hay biết.
- Ta sẽ thử tấn công qua Content-Length = 0 và HTTP/1.1.
 📌 **PAYLOAD**
 ```
   Content-Length: 0

   GET /post/like/12315198742342 HTTP/1.1
   X: F
 ```  
- Khi request này bị downgrade từ HTTP/2 xuống HTTP/1.1, reverse proxy có thể bị nhầm lẫn và coi đây là một request khác.
- x:F để tránh lỗi cú pháp HTTP/1.1, vì request phải có ít nhất một header.
![image](https://github.com/user-attachments/assets/e3671c23-d262-4552-800c-f69746e5d26e)
- Khi ta gửi request kèm với payload đi response sẽ trả về như trên.
- ⚠️ **Lưu ý** :  Không được để dòng trống sau X: f, vì nếu có dòng trống, request tiếp theo từ victim sẽ không bị nối vào mà trở thành một request riêng biệt.
- Sau đó sẽ chờ 1 thời gian để nạn nhân dính bẫy nối tiếp vào.
![image](https://github.com/user-attachments/assets/94e791e7-882c-42ec-9d50-42206a054a3a)
FLag đã xuất hiện : THM{my_name_is_a_flag}
- ### **📌 LAB 2 - HTTP/2 Request Tunneling.**.  
- **Ví dụ trong bài lab với HAProxy (CVE-2019-19330).**  
- Request Tunneling là một dạng HTTP Request Smuggling, nhưng khác với Desync Attack, nó xảy ra khi mỗi người dùng có một backend connection riêng thay vì chia sẻ một kết nối chung. Điều này có nghĩa là kẻ tấn công không thể tác động trực tiếp đến request của người dùng khác, nhưng vẫn có thể chèn request vào backend của chính mình để khai thác lỗ hổng trong ứng dụng.
![image](https://github.com/user-attachments/assets/820ea5e4-da72-48c0-a939-8641e85c1230).  
- Khi mở lên ta thấy có /admin click vào thì bị 403, nên ta sẽ click vào phần /hello.
![image](https://github.com/user-attachments/assets/748f958c-d594-4fa0-97fa-3f2bd23a6806).  
- Thử nhập vào ô tìm kiếm xong click search, ta thấy 1 request POST /hello với tham số q gửi đi.
![image](https://github.com/user-attachments/assets/87e23b15-db98-49f7-92df-1c1e4699dfd1)
- Kết quả hiện ra chính là giá trị của tham số q. ta cũng thấy có Content-Length: 8 mặc dù HTTP/2 sẽ bỏ qua.  
![image](https://github.com/user-attachments/assets/9b4666bf-4907-4ea7-a953-cf73f175a8af) 
- Hầu hết các trình duyệt sẽ thêm header này vào tất cả các yêu cầu HTTP/2 để phần sau vẫn sẽ nhận được header độ dài nội dung hợp lệ nếu xảy ra drowngrade HTTP.  
- Ta sẽ lợi dụng lỗ hổng trong HAProxy, cho phép chèn CRLF vào headers để rò rỉ các header nội bộ của backend.
- **CRLF (\r\n)** là ký tự kết thúc header trong HTTP.
- Khi một request có dòng header như:
```
   Foo: something\r\nX-Internal: secret-token\r\n
```
- HAProxy có thể hiểu đây là hai header khác nhau:
```
   Foo: something
   X-Internal: secret-token
```
Do đó ta sẽ chèn Payload này vào header tùy chỉnh (Foo) để tạo một request thứ hai:
```
   HOST: 10.10.152.146:8100
   POST /hello HTTP/1.1
   Content-Length: 300
   HOST: 10.10.152.146:8100
   Content-Type: application/x-www-form-urlencoded
   
   q = 
```

#Giải thích : 
- Request ban đầu gửi đến HAProxy. Proxy HAProxy không nhận ra sự bất thường vì HTTP/2 cho phép ký tự nhị phân trong header.  
- Tuy nhiên, khi chuyển request về backend (chạy HTTP/1.1), nó sẽ tách thành hai request do CRLF.
- Backend đọc request đầu tiên, thấy Content-Length: 0, nên hiểu rằng nó không có body và xử lý như request bình thường.
- Request thứ hai bắt đầu ngay sau đó, nhưng backend vẫn gán các header nội bộ vào nó trước khi xử lý body.
- Khi backend xử lý q=, nó nhận ra rằng body chứa luôn cả phần header nội bộ sẽ bị đẩy vào body do lỗi tách request.
- ![image](https://github.com/user-attachments/assets/99723650-65f3-4f2c-b62f-ad455ea8ec9e).
- Flag xuất hiện khi gửi : ![image](https://github.com/user-attachments/assets/24417e0a-8828-47c3-89b9-d03bf38fcaec)


  
  

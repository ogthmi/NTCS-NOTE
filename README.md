# Tài liệu nghiên cứu và thực hành an ninh mạng

Tài liệu này tổng hợp các nội dung học tập, nghiên cứu và hướng dẫn thực hành về an ninh ứng dụng web. Các chủ đề tập trung vào cơ chế hoạt động của trình duyệt, cách thức xử lý của máy chủ và các kỹ thuật khai thác, bypass an toàn thông tin.

## Danh sách các nhiệm vụ và nội dung chi tiết

### TASK 3: Cơ chế bảo mật trình duyệt và các lỗ hổng Client side

* **Task 3.1: SOP và CORS**
  Trình bày về chính sách đồng nguồn Same Origin Policy, cơ chế chia sẻ tài nguyên khác nguồn CORS, luồng xử lý Simple Request và Preflight OPTIONS, các lỗi cấu hình sai CORS phổ biến trong thực tế.

* **Task 3.2: XSS**
  Tổng quan về lỗ hổng Cross Site Scripting, cách thức trình duyệt phân tích dữ liệu HTML, DOM, JS Engine, phân loại Reflected, Stored, DOM based XSS và phương pháp phòng chống hiệu quả.

* **Task 3.3: XSS Port Swigger**
  Hướng dẫn từng bước giải quyết các bài tập thực hành về XSS trên nền tảng Web Security Academy của PortSwigger.

* **Task 3.4: XSS Filter**
  Phân tích các cơ chế mã hóa đầu ra, bộ lọc đầu vào và các kỹ thuật bypass phức tạp như Context Escape, Keyword/Tag Bypass, Parser Differential.

* **Task 3.5: CSRF**
  Cơ chế hoạt động của lỗ hổng giả mạo yêu cầu Cross Site Request Forgery, vai trò của Cookie, Session, Token và các biện pháp ngăn chặn như SameSite, Anti CSRF Token.

### TASK 4: Tải tệp tin và Điều hướng đường dẫn

* **Task 4.1: HTTP Multipart**
  Cấu trúc giao thức multipart/form data và quy trình truyền tải file của trình duyệt lên máy chủ PHP.

* **Task 4.2: PHP Core**
  Cơ chế hoạt động của PHP Engine, cách đọc dữ liệu từ superglobals và các hàm thao tác file hệ thống.

* **Task 4.3: DVWA File Upload**
  Hướng dẫn chi tiết thực hành và bypass bộ lọc upload file qua các cấp độ bảo mật của môi trường DVWA.

* **Task 4.4: Web Server File Processing**
  Phân tích quyết định thực thi hoặc tải xuống của web server đối với các loại file tĩnh và động, hướng dẫn cấu hình cấm thực thi php trong thư mục uploads.

* **Task 4.5: Docker Core**
  Lý thuyết về container, kiến trúc namespaces và cgroups, hướng dẫn Dockerfile, Docker Volumes, Docker Networking, Docker Compose và bảo mật Docker Daemon.

* **Task 4.6: Filter và Bypass file upload**
  Phân tích nguyên nhân gốc rễ của các bộ lọc file upload và hướng dẫn các kỹ thuật bypass thực tế như Double Extension, Null Byte, .htaccess, .user.ini.

* **Task 4.7: PortSwigger File Upload**
  Hướng dẫn từng bước giải quyết các bài lab thực hành an toàn file upload trên PortSwigger.

* **Task 4.8: Path Traversal**
  Lý thuyết về điều hướng đường dẫn để đọc hoặc ghi file tùy ý, các kỹ thuật bypass bộ lọc và biện pháp phòng chống triệt để dùng realpath.

* **Task 4.9: PortSwigger Path Traversal**
  Hướng dẫn thực hành và giải quyết trọn bộ sáu bài lab về Path Traversal của PortSwigger.

### TASK 5: Lỗ hổng nhúng file cục bộ và từ xa

* **Task 5.1: LFI RFI**
  Khái niệm, điều kiện khai thác, cơ chế hoạt động của PHP Stream Wrappers.

* **Task 5.2: LFI RFI Exploit Chains và Checklist**
  Các chuỗi khai thác RCE nâng cao như Log Poisoning, Session Poisoning, File Upload Chaining, PHP Filter Chain, PHP Session Upload Progress và danh sách kiểm tra lỗi cho kiểm thử viên.
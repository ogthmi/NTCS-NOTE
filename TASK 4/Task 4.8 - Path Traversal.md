# Path Traversal

> Bản chất lỗ hổng điều hướng đường dẫn để Đọc file và Ghi/Tải lên file tùy ý ngoài thư mục quy định.

## 1. Path Traversal là gì?

**Path Traversal** (hay còn gọi là Directory Traversal) là lỗ hổng bảo mật cho phép kẻ tấn công điều hướng qua các thư mục trên hệ thống tệp tin của máy chủ.

* **Ký tự cốt lõi**: Lỗ hổng này hoạt động dựa trên các ký tự đặc biệt dùng để di chuyển thư mục:
  * `../` (trên hệ thống Linux/Unix/macOS)
  * `..\` (trên hệ thống Windows)
* **Nguyên nhân gốc rễ**: Ứng dụng nhận dữ liệu đầu vào từ người dùng (như tên tệp hoặc đường dẫn tệp) và trực tiếp ghép chuỗi (concatenation) vào các hàm xử lý tệp của hệ thống mà không thực hiện kiểm tra, làm sạch hoặc chuẩn hóa đường dẫn.

## 2. Phân loại tác động của Path Traversal

Lỗ hổng Path Traversal được chia làm hai hướng tác động chính với mức độ nguy hiểm khác nhau:

### A. Arbitrary File Read (Đọc file tùy ý)
* **Khái niệm**: Kẻ tấn công lợi dụng tham số đường dẫn để đọc các tệp tin nhạy cảm trên máy chủ mà thông thường không thể truy cập trực tiếp qua URL tĩnh.
* **Mục tiêu phổ biến**:
  * File hệ thống: `/etc/passwd`, `/etc/shadow` (Linux), `C:\Windows\win.ini` (Windows).
  * File cấu hình ứng dụng (chứa thông tin đăng nhập database, API keys): `config.php`, `settings.py`, `.env`.
  * Mã nguồn của chính ứng dụng Web để tìm thêm các lỗ hổng khác.
* **Mã nguồn lỗi minh họa (PHP)**:
  ```php
  $page = $_GET['page']; // ví dụ: "about.html"
  include("/var/www/html/templates/" . $page); 
  // hoặc
  echo file_get_contents("/var/www/html/static/" . $page);
  ```
* **Cách khai thác**:
  Kẻ tấn công gửi request với payload lùi thư mục:
  ```http
  GET /view.php?page=../../../../etc/passwd HTTP/1.1
  ```
  Hàm `file_get_contents` sẽ xử lý đường dẫn: `/var/www/html/static/../../../../etc/passwd`.
  Do có các chuỗi `../` điều hướng lùi về thư mục cha, đường dẫn thực tế sẽ trỏ thẳng tới `/etc/passwd` và trả nội dung về cho kẻ tấn công.

### B. Arbitrary File Write / Upload (Ghi/Tải lên file tùy ý)
* **Khái niệm**: Lợi dụng chức năng lưu tệp tin (như File Upload hoặc tạo file mới) để lưu tệp độc hại ngoài thư mục chỉ định.
* **Tầm quan trọng đặc biệt trong File Upload**:
  * Trong thực tế, nhiều hệ thống cấu hình rất chặt chẽ: Thư mục `/var/www/html/uploads/` được cấu hình **tắt thực thi PHP Engine** (như đã học trong [Task 4.4 - Web Server File Processing.md](file:///d:/NINHTHANH_CYBERSEC/DOCUMENTATION/TASK%204/Task%204.4%20-%20Web%20Server%20File%20Processing.md#L264>)). Mọi tệp tải lên đây đều chỉ được xử lý như tệp tĩnh thông thường, không thể chạy webshell.
  * Lúc này, kẻ tấn công sử dụng Path Traversal lồng vào **tên file** (filename) khi gửi yêu cầu tải lên. Mục tiêu là ghi tệp webshell ngược ra thư mục cha (ví dụ thư mục Web Root `/var/www/html/` - nơi có quyền thực thi PHP).
* **Mã nguồn lỗi minh họa (PHP)**:
  ```php
  $filename = $_FILES['file']['name']; // Ví dụ client gửi lên: "../shell.php"
  $target_path = "/var/www/html/uploads/" . $filename;
  
  move_uploaded_file($_FILES['file']['tmp_name'], $target_path);
  ```
* **Cách khai thác**:
  Khi tải file lên, kẻ tấn công dùng Burp Suite sửa tên file trong phần tiêu đề Multipart:
  ```http
  Content-Disposition: form-data; name="file"; filename="../shell.php"
  ```
  Hàm `move_uploaded_file` sẽ nối chuỗi thành đường dẫn lưu file: `/var/www/html/uploads/../shell.php`.
  Đường dẫn này được hệ điều hành chuẩn hóa thành `/var/www/html/shell.php`. Webshell đã được ghi đè ra ngoài thư mục `/uploads/` bị cấm và nằm ở Web Root, cho phép kẻ tấn công kích hoạt RCE thành công.

## 3. Các kỹ thuật Bypass bộ lọc Path Traversal

Khi lập trình viên cố gắng tự xây dựng bộ lọc bằng các hàm xử lý chuỗi cơ bản, kẻ tấn công thường vượt qua bằng các kỹ thuật sau:

### 3.1. Lọc không đệ quy (Non-recursive stripping)
* **Bộ lọc yếu**: Chỉ thay thế chuỗi `../` thành chuỗi rỗng một lần duy nhất bằng các hàm như `str_replace()` trong PHP.
  ```php
  $filename = str_replace("../", "", $_POST['filename']);
  ```
* **Kỹ thuật bypass**: Sử dụng chuỗi lồng nhau `....//` hoặc `..././`.
  * Khi bộ lọc quét qua, nó sẽ tìm thấy cụm `../` ở giữa chuỗi `....//` (tức là `..[../]/`) và xóa nó đi.
  * Các ký tự còn lại ở hai bên sẽ chập lại với nhau, tự động tạo thành chuỗi điều hướng hợp lệ mới: `../`.

### 3.2. URL Encoding & Double URL Encoding
* **Bộ lọc yếu**: Máy chủ web tự động giải mã URL trước khi truyền dữ liệu vào ứng dụng, hoặc ứng dụng chỉ kiểm tra chuỗi ký tự thô `../` mà không tính đến các biến thể được mã hóa.
* **Kỹ thuật bypass**:
  * **Single URL Encode**: 
    * Mã hóa dấu `/` thành `%2f` hoặc dấu `.` thành `%2e`.
    * Payload: `..%2f` hoặc `%2e%2e/` hoặc `%2e%2e%2f`
  * **Double URL Encode** (Khi hệ thống thực hiện giải mã 2 lần trước khi gọi hàm hệ thống):
    * Ký tự `%` trong `%2f` sẽ được mã hóa tiếp thành `%25`.
    * Payload: `..%252f` (Giải mã lần 1 -> `..%2f` -> Giải mã lần 2 -> `../`).

### 3.3. Sử dụng đường dẫn tuyệt đối trực tiếp (Absolute Path)
* **Bộ lọc yếu**: Bộ lọc chỉ tập trung tìm kiếm các ký tự điều hướng lùi thư mục `..`.
* **Kỹ thuật bypass**: Bỏ qua hoàn toàn việc lùi thư mục và chỉ định trực tiếp đường dẫn tuyệt đối bắt đầu bằng gốc thư mục gốc (nếu ứng dụng không tự động chèn thư mục gốc tĩnh vào đầu).
  * Ví dụ: `/etc/passwd` thay vì `../../../../etc/passwd`.

### 3.4. Giữ nguyên thư mục bắt đầu bắt buộc (Start of path validation)
* **Bộ lọc yếu**: Ứng dụng kiểm tra xem đường dẫn tệp tin do người dùng nhập vào có bắt đầu bằng một thư mục tĩnh chỉ định trước hay không (ví dụ: yêu cầu phải bắt đầu bằng `/var/www/images/`).
* **Kỹ thuật bypass**: Kẻ tấn công cung cấp thư mục bắt đầu bắt buộc đó để thỏa mãn điều kiện kiểm tra của bộ lọc, sau đó sử dụng chuỗi `../` ngay phía sau để lùi ngược về gốc hệ thống và truy cập file nhạy cảm.
  * Ví dụ: `/var/www/images/../../../etc/passwd`

### 3.5. Kiểm tra đuôi file bằng Null Byte (Validation of file extension via Null Byte)
* **Bộ lọc yếu**: Ứng dụng kiểm tra nghiêm ngặt phần mở rộng của tệp tin ở cuối đường dẫn (ví dụ: chỉ cho phép đường dẫn kết thúc bằng `.jpg` hoặc `.png`).
* **Kỹ thuật bypass**: Kẻ tấn công sử dụng ký tự Null Byte `%00` (ở định dạng URL) để chèn vào trước phần mở rộng bắt buộc. Kỹ thuật này chủ yếu hoạt động trên các ngôn ngữ/phiên bản cũ (như PHP < 5.3.4) sử dụng các API hệ thống viết bằng C. Hàm kiểm tra của ứng dụng web vẫn nhìn thấy đuôi `.jpg` hợp lệ ở cuối chuỗi, nhưng khi hệ điều hành xử lý đường dẫn ở tầng dưới (C-style string), nó sẽ coi chuỗi kết thúc tại vị trí Null Byte và bỏ qua hoàn toàn phần mở rộng giả phía sau.
  * Ví dụ: `../../../etc/passwd%00.jpg`

## 4. Giải pháp phòng chống Path Traversal triệt để

Để ngăn chặn lỗ hổng Path Traversal (cả trong đọc file lẫn tải file), lập trình viên cần tuân thủ các nguyên tắc sau:

### 4.1. Sử dụng hàm basename()
Hàm [basename()](file:///d:/NINHTHANH_CYBERSEC/DOCUMENTATION/TASK%204/Task%204.2%20-%20PHP%20Core.md#L228>) trong PHP sẽ chỉ trích xuất phần tên file cuối cùng và loại bỏ hoàn toàn các thông tin về thư mục hay ký tự điều hướng đứng trước nó.

* **Ví dụ**:
  ```php
  echo basename("../../../etc/passwd"); // Kết quả: "passwd"
  echo basename("uploads/../shell.php"); // Kết quả: "shell.php"
  ```
* **Mã nguồn an toàn khi upload**:
  ```php
  $filename = basename($_FILES['file']['name']); // Đảm bảo chỉ có tên tệp tĩnh
  $target_path = "/var/www/html/uploads/" . $filename;
  ```

### 4.2. Chuẩn hóa và đối chiếu đường dẫn bằng realpath()
Hàm `realpath()` sẽ giải quyết toàn bộ các liên kết động, ký tự điều hướng `../` và trả về đường dẫn tuyệt đối chuẩn hóa duy nhất của tệp tin trên đĩa cứng (hoặc trả về `false` nếu tệp không tồn tại).

* **Cách hoạt động phòng chống**:
  1. Lấy đường dẫn do người dùng yêu cầu.
  2. Dùng `realpath()` để chuyển đổi nó thành đường dẫn tuyệt đối.
  3. Kiểm tra xem đường dẫn tuyệt đối này có bắt đầu bằng thư mục gốc được phép hay không (sử dụng hàm `str_starts_with()`).

* **Mã nguồn an toàn minh họa (Đọc file)**:
  ```php
  $base_dir = "/var/www/html/templates/";
  $user_input = $_GET['page'];
  
  // Tạo đường dẫn tuyệt đối được chuẩn hóa
  $real_path = realpath($base_dir . $user_input);
  
  // Xác minh xem đường dẫn thật có nằm trong thư mục gốc hợp lệ không
  if ($real_path !== false && str_starts_with($real_path, $base_dir)) {
      include($real_path);
  } else {
      die("Truy cập bị từ chối!");
  }
  ```

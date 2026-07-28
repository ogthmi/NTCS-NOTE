# Web Server và File Execution

> Upload file xong rồi, tại sao có file thì chạy được, có tệp tin thì chỉ tải xuống?

## Bố cục nội dung

```
Web Server & File Execution
│
├── Browser Request (Yêu cầu từ trình duyệt)
│
├── Web Server (Máy chủ web)
│   ├── Apache
│   ├── Nginx
│   ├── IIS
│   └── Kestrel
│
├── Static File vs Dynamic Resources (Tài nguyên tĩnh và động)
│   ├── Khái niệm tệp tĩnh
│   ├── Khái niệm kịch bản động
│   └── Các cơ chế thực thi mã động
│
├── Request Flow (Dòng chảy của một yêu cầu)
│   ├── Yêu cầu tệp động (PHP)
│   └── Yêu cầu tệp tĩnh (Ảnh)
│
├── Execute hay Download? (Thực thi hay tải xuống?)
│   ├── Quyết định phía Web Server
│   ├── Quyết định phía Trình duyệt
│   └── Cơ chế Fallback và MIME Sniffing
│
├── Mapping Extension (Cơ chế ánh xạ đuôi tệp)
│   ├── Ánh xạ trên Apache
│   ├── Ánh xạ trên Nginx
│   ├── Ánh xạ trên IIS
│   └── Cấm thực thi kịch bản trong thư mụcuploads
│
├── Docker Debug (Kiểm tra thực tế trong container)
│   ├── Lệnh truy cập Container
│   ├── Các lệnh kiểm tra cấu hình hữu ích
│   └── Bản chất quyền Đọc và quyền Thực thi
│
├── Lab (Thực hành quan sát cấu trúc phản hồi)
│   ├── Chuẩn bị các tệp kiểm thử
│   ├── Các bước tiến hành
│   ├── Bảng ghi nhận kết quả quan sát
│   └── Lưu ý bảo mật quan trọng về tệp SVG
│
└── Chuẩn bị cho Upload Bypass (Các kịch bản tấn công cơ bản)
    ├── Lợi dụng đa phần mở rộng (Multi-extension)
    ├── Lợi dụng tệp cấu hình cục bộ .htaccess
    └── Khai thác phần mở rộng thay thế
```

---

## Browser Request

### Tài liệu tham khảo:

* [MDN Web Docs: An overview of HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview)
* [MDN Web Docs: HTTP Headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers)

Trình duyệt web (Browser) giống như một người khách hàng đi vào cửa hiệu để mua sắm. Trình duyệt sẽ gửi một thông điệp yêu cầu bằng văn bản (gọi là HTTP Request) qua mạng Internet để hỏi xin máy chủ một tệp tin cụ thể nào đó.

Thông điệp HTTP Request này chứa tên tệp tin cần xin (ví dụ: `index.html` hoặc `avatar.png`) và các thông tin đi kèm để máy chủ hiểu cách phản hồi phù hợp.

Chi tiết về cấu trúc của một HTTP Request (bao gồm dòng yêu cầu Request Line, các tiêu đề quan trọng như Host, User-Agent, Accept, và phần thân Request Body) đã được giải thích chi tiết tại phần **Ôn tập về HTTP Request** của tài liệu [Task 4.1 - HTTP Multipart.md](<file:///d:/NINHTHANH_CYBERSEC/DOCUMENTATION/TASK%204/Task%204.1%20-%20HTTP%20Multipart.md>). Người học nên xem lại tài liệu đó để nắm vững cấu trúc yêu cầu trước khi tìm hiểu cách máy chủ xử lý tệp.

> Kể từ thời điểm trình duyệt gửi thông điệp đi, trình duyệt sẽ hoàn toàn ở trạng thái thụ động để chờ đợi phản hồi. Quyết định tệp tin có được chạy (thực thi) hay không hoàn toàn nằm ở máy chủ web chứ không phải ở trình duyệt.

---

## Web Server

### Tài liệu tham khảo:

* [Apache HTTP Server Documentation](https://httpd.apache.org/docs/)
* [Nginx Documentation](https://nginx.org/en/docs/)
* [Microsoft IIS Documentation](https://learn.microsoft.com/en-us/iis/)
* [Kestrel Web Server Overview](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel)

Máy chủ web (Web Server) là một phần mềm chuyên dụng chạy liên tục trên máy chủ vật lý hoặc máy ảo. Nhiệm vụ chính của nó là lắng nghe các yêu cầu kết nối từ trình duyệt gửi đến thông qua các cổng mạng tiêu chuẩn (mặc định là cổng 80 cho HTTP và cổng 443 cho HTTPS).

Các cổng mạng này hoạt động giống như các quầy tiếp tân mở sẵn cửa để đón khách vào giao dịch. Khi có yêu cầu gửi đến, máy chủ web sẽ phân tích đường dẫn tài nguyên để tự tìm tệp tin vật lý tương ứng trên đĩa cứng hoặc chuyển tiếp yêu cầu đến các bộ xử lý mã kịch bản động.

Dưới đây là bốn phần mềm máy chủ web phổ biến nhất được sử dụng trong thực tế:

* **Apache HTTP Server**:
  * Đây là một trong những phần mềm máy chủ web lâu đời, ổn định và phổ biến nhất trên thế giới.
  * Apache hoạt động dựa trên cấu trúc các khối chức năng độc lập (gọi là các modules) xếp chồng lên nhau để xử lý công việc.
  * Điểm đặc biệt của Apache là cho phép người dùng cấu hình riêng biệt cho từng thư mục bằng cách đặt tệp tin cấu hình cục bộ `.htaccess` trực tiếp bên trong thư mục đó mà không cần can thiệp vào tệp cấu hình chính của hệ thống.
* **Nginx**:
  * Nginx được thiết kế theo kiến trúc hướng sự kiện và bất đồng bộ (event-driven, asynchronous).
  * Cơ chế này hoạt động giống như một nhân viên phục vụ siêu năng lực, có thể tiếp nhận và xử lý yêu cầu của hàng vạn khách hàng cùng một lúc mà không bị quá tải.
  * Vì lý do tối ưu hóa tốc độ, Nginx không hỗ trợ tệp cấu hình cục bộ như `.htaccess` của Apache; mọi cấu hình bắt buộc phải viết trực tiếp trong tệp cấu hình chính của hệ thống.
* **Microsoft IIS (Internet Information Services)**:
  * Đây là máy chủ web độc quyền do Microsoft phát triển và được cài đặt sẵn trên các hệ điều hành Windows Server.
    *IIS được quản trị chủ yếu thông qua giao diện đồ họa (GUI) trực quan thay vì dòng lệnh thô, và lưu trữ cấu hình trong tệp tin XML mang tên `web.config`.
  * Máy chủ này hỗ trợ tối ưu nhất cho các ứng dụng viết bằng ngôn ngữ .NET (như ASP.NET) thông qua các module xử lý tích hợp sâu vào hệ thống.
* **Kestrel**:
  * Kestrel là một máy chủ web mã nguồn mở, siêu nhẹ và có tốc độ xử lý rất nhanh dành riêng cho các ứng dụng viết bằng ASP.NET Core.
  * Kestrel thường không được cấu hình đứng trực tiếp đối mặt với Internet bên ngoài; nó thường được đặt ẩn mình phía sau một máy chủ cổng chuyển tiếp (Reverse Proxy) như Nginx hoặc IIS để đảm bảo an toàn bảo mật.

Sơ đồ định tuyến cơ bản của một Web Server khi nhận yêu cầu:

```
Trình duyệt gửi yêu cầu
         │
         ▼
    Web Server
         │
         ├── Yêu cầu tệp tĩnh ──► Đọc trực tiếp tệp trên đĩa cứng và gửi đi
         │
         └── Yêu cầu kịch bản động ──► Chuyển tiếp cho PHP / ASP.NET / JSP xử lý
```

---

## Static File vs Dynamic Resources

### Tài liệu tham khảo:

* [RFC 3875: The Common Gateway Interface (CGI) Version 1.1](https://datatracker.ietf.org/doc/html/rfc3875)
* [PHP-FPM (FastCGI Process Manager) Documentation](https://www.php.net/manual/en/install.fpm.php)

Để hiểu rõ bản chất của tài nguyên tĩnh và kịch bản động, hãy tưởng tượng Web Server là người phục vụ bàn ở một nhà hàng, còn bộ thông dịch (ví dụ: PHP) là người đầu bếp trong bếp.

### Tệp tĩnh (Static file)

Tệp tĩnh là những tệp tin có nội dung cố định, không thay đổi theo thời gian hay theo người dùng (ví dụ: hình ảnh `.jpg`, tệp văn bản `.txt`, tệp thiết kế `.css` hoặc mã nguồn `.js` chạy trên trình duyệt).
Tệp tĩnh giống như một chai nước đóng chai có sẵn trên kệ của nhà hàng. Khi khách hàng yêu cầu chai nước, người phục vụ bàn (Web Server) chỉ cần ra kệ lấy đúng chai nước đó từ đĩa cứng và mang nguyên vẹn ra giao cho khách hàng mà không cần chế biến gì thêm.

Sơ đồ quy trình xử lý tệp tĩnh:

```
Trình duyệt ──(Xin tệp img.png)──► Web Server ──(Đọc đĩa cứng)──► Byte dữ liệu ảnh ──► Trình duyệt
```

### Kịch bản động (Dynamic script)

Kịch bản động là các tệp tin chứa mã nguồn lập trình chạy trên máy chủ (như ngôn ngữ PHP, ASP.NET hoặc JSP) cần được thực thi để tạo ra kết quả phù hợp cho từng yêu cầu.
Kịch bản động giống như một món ăn cần nấu theo yêu cầu riêng của thực khách. Người phục vụ bàn (Web Server) không thể tự làm, mà phải chuyển thực đơn xuống bếp cho người đầu bếp (bộ thông dịch PHP-FPM) chế biến. Đầu bếp sẽ thực thi công thức nấu ăn (chạy mã nguồn PHP, lấy nguyên liệu từ cơ sở dữ liệu) để tạo ra đĩa thức ăn hoàn chỉnh dạng mã HTML rồi giao lại cho người phục vụ bàn để mang ra cho khách.
Chi tiết về mô hình hoạt động của bộ dịch PHP và cách máy chủ chuyển tiếp yêu cầu qua các biến siêu toàn cục (superglobals) có thể tham khảo tại phần **PHP hoạt động như thế nào** của tài liệu [Task 4.2 - PHP Core.md](<file:///d:/NINHTHANH_CYBERSEC/DOCUMENTATION/TASK%204/Task%204.2%20-%20PHP%20Core.md>).

Sơ đồ quy trình xử lý kịch bản động:

```
Trình duyệt ──(Xin tệp file.php)──► Web Server ──(Chuyển tiếp)──► Bộ dịch PHP ──(Trả về HTML)──► Trình duyệt
```

### Các cơ chế thực thi mã động của máy chủ

- **CGI (Common Gateway Interface)**: Là cơ chế cổ điển. Mỗi khi có yêu cầu mới gửi đến, máy chủ sẽ khởi tạo một tiến trình hệ thống mới để xử lý; cơ chế này cực kỳ ngốn tài nguyên hệ thống và tốc độ phản hồi rất chậm.
- **Module tích hợp (mod_php)**: Nhúng trực tiếp bộ dịch PHP vào bên trong tiến trình hoạt động của Apache; tốc độ xử lý nhanh hơn nhưng làm máy chủ trở nên nặng nề và kém an toàn do dùng chung quyền hạn hệ thống.
- **FastCGI / PHP-FPM (FastCGI Process Manager)**: Bộ dịch PHP chạy như một dịch vụ độc lập bên ngoài máy chủ web với các tiến trình xử lý chạy ngầm được tối ưu sẵn; đây là chuẩn mực cấu hình hiện đại nhờ hiệu năng cao và độ an toàn bảo mật vượt trội.

---

## Request Flow

Việc hiểu rõ dòng chảy của yêu cầu giúp người học giải thích được vì sao trình duyệt của người dùng không bao giờ nhìn thấy mã nguồn PHP thô của ứng dụng mà chỉ thấy mã HTML được sinh ra sau quá trình xử lý.

### Dòng chảy yêu cầu tệp động (Ví dụ: GET /shell.php)

Khi người dùng yêu cầu một tệp kịch bản động, quy trình diễn ra từng bước như sau:

```
Trình duyệt ──(Yêu cầu GET /shell.php)──► Máy chủ Apache ──(Chuyển tiếp tệp)──► PHP Engine
                                                                                   │
                                                                           (Thực thi mã nguồn:
                                                                            Chạy lệnh, query DB)
                                                                                   │
                                                                                   ▼
Trình duyệt ◄──(Nhận mã HTML đã sinh)◄── Máy chủ Apache ◄──(Gửi lại kết quả HTML)─┘
```

Trình duyệt chỉ nhận lại kết quả dạng HTML cuối cùng được sinh ra sau quá trình xử lý trên máy chủ. Mã nguồn PHP thô được giữ an toàn trên server.

### Dòng chảy yêu cầu tệp tĩnh (Ví dụ: GET /cat.jpg)

Khi người dùng yêu cầu một tệp tĩnh, quy trình đơn giản hơn rất nhiều:

```
Trình duyệt ──(Yêu cầu GET /cat.jpg)──► Máy chủ Apache ──(Đọc tệp trực tiếp)──► Ổ đĩa cứng
                                                                                   │
Trình duyệt ◄──(Nhận dữ liệu nhị phân ảnh)◄────────────────────────────────────────┘
```

Không có bộ xử lý ngôn ngữ nào tham gia vào quá trình này. Máy chủ chỉ đóng vai trò truyền tải tệp từ ổ đĩa về máy khách.

---

## Execute hay Download?

### Tài liệu tham khảo:

* [MDN Web Docs: Common MIME types](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types/Common_types)
* [MDN Web Docs: X-Content-Type-Options](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Content-Type-Options)

Hành vi xử lý một tệp tin là kết quả của sự phối hợp chặt chẽ giữa hai quyết định độc lập từ phía máy chủ và trình duyệt.

Sơ đồ quy trình quyết định hai bước:

```
Yêu cầu HTTP từ trình duyệt
             │
             ▼
        Web Server
[Quyết định 1: Có chạy mã động (Execute) hay chỉ đọc nội dung tệp tĩnh (Read file)?]
             │
             ▼
HTTP Response (Gửi kèm Headers: Content-Type, Content-Disposition)
             │
             ▼
        Trình duyệt
[Quyết định 2: Dựng giao diện (Render), hiển thị thô (Display) hay tải về máy (Download)?]
```

### Quyết định 1: Phía Web Server (Execute hay Read file)

Khi nhận được yêu cầu từ trình duyệt, máy chủ web quyết định xem có chạy tệp tin đó dưới dạng mã nguồn động hay chỉ đọc nội dung tĩnh.

* Nếu máy chủ nhận diện được phần mở rộng và cấu hình tương ứng (ví dụ đuôi `.php` khớp với cấu hình PHP Engine), máy chủ sẽ thực thi mã nguồn tệp tin (Execute).
* Nếu không có cấu hình tương ứng, máy chủ xử lý tệp đó như một tệp tĩnh bình thường, đọc nội dung byte thô từ đĩa cứng và gửi về cho trình duyệt (Read file).

### Quyết định 2: Phía Trình duyệt (Render, Display hay Download)

Sau khi nhận được phản hồi HTTP từ máy chủ, trình duyệt dựa vào hai tiêu đề phản hồi chính để quyết định cách hiển thị:

* **Content-Type**: Tiêu đề này khai báo kiểu dữ liệu của tệp tin gửi về (ví dụ `text/html` để dựng trang, `image/jpeg` để hiển thị ảnh, `application/octet-stream` để tải xuống).
* **Content-Disposition**: Tiêu đề này chỉ thị trực tiếp cho trình duyệt cách xử lý tệp tin. Nếu tiêu đề này được thiết lập giá trị là `attachment` (đính kèm), trình duyệt sẽ bỏ qua mọi khai báo `Content-Type` và lập tức lưu tệp xuống đĩa cứng thay vì hiển thị trực tiếp.

Bảng tổng hợp hành vi của Web Server và trình duyệt đối với các tệp phổ biến:

| Yêu cầu URL | Web Server làm gì          | Browser nhận được    | Hành vi của trình duyệt           |
| :------------ | :--------------------------- | :----------------------- | :------------------------------------ |
| `file.php`  | Thực thi mã PHP (Execute)  | HTML                     | Dựng giao diện trang web (Render).  |
| `file.jpg`  | Đọc tệp tĩnh (Read file) | image/jpeg               | Hiển thị hình ảnh (Display).      |
| `file.txt`  | Đọc tệp tĩnh (Read file) | text/plain               | Hiển thị văn bản thô (Display).  |
| `file.html` | Đọc tệp tĩnh (Read file) | text/html                | Dựng giao diện trang web (Render).  |
| `file.xyz`  | Dùng cấu hình mặc định | application/octet-stream | Mở hộp thoại lưu tệp (Download). |

### Cấu hình mặc định (Fallback) và tính năng bảo mật

- **Fallback**: Nếu Web Server không nhận diện được phần mở rộng của tệp, nó sử dụng MIME Type mặc định là `application/octet-stream`, trình duyệt khi nhận được sẽ bắt buộc tải file xuống.
- **MIME Sniffing**: Trình duyệt có thể tự ý đoán định dạng file bằng cách đọc các byte đầu tiên nếu server cấu hình thiếu; quản trị viên bắt buộc phải cấu hình tiêu đề `X-Content-Type-Options: nosniff` để vô hiệu hóa tính năng này nhằm tránh rủi ro bảo mật.

---

## Mapping Extension

### Tài liệu tham khảo:

* [Apache Module mod_mime: Directives](https://httpd.apache.org/docs/current/mod/mod_mime.html)
* [Nginx Config: Server blocks and Location](https://nginx.org/en/docs/http/ngx_http_core_module.html#location)
* [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Security_Cheat_Sheet.html)

Mỗi máy chủ web sử dụng các cơ chế cấu hình cấu trúc đường dẫn và phần mở rộng khác nhau để liên kết tệp tin với bộ xử lý tương ứng.

### Cấu hình ánh xạ trên Apache

Apache sử dụng các cấu hình trong tệp chính hoặc tệp tin cục bộ `.htaccess` để chỉ định cách xử lý đuôi tệp:

* **`AddType`**: Khai báo kiểu MIME cho đuôi file. Ví dụ: `AddType application/x-httpd-php .php`.
* **`AddHandler`**: Chỉ định Handler xử lý đuôi file. Ví dụ: `AddHandler application/x-httpd-php .php`.
* **`mod_php`**: Module nhúng trực tiếp bộ thông dịch PHP vào tiến trình Apache.
* **`PHP-FPM`**: Chuyển tiếp yêu cầu xử lý PHP sang dịch vụ FPM chạy độc lập qua các chỉ thị proxy.
* **`FilesMatch`**: So khớp tên tệp bằng Regex để áp dụng cấu hình đặc biệt.
* **Nguy cơ bảo mật**: Apache có cơ chế xử lý đa phần mở rộng (Multi-extension). Nếu tệp có tên `shell.php.jpg`, Apache quét từ phải qua trái. Do không có cấu hình đặc biệt cho `.jpg`, Apache tiếp tục quét sang trái gặp đuôi `.php` và thực thi tệp này dưới dạng mã PHP.

### Cấu hình ánh xạ trên Nginx

Nginx ánh xạ tệp tin dựa trên cấu trúc các khối lệnh so khớp đường dẫn:

* **`location`**: Sử dụng biểu thức chính quy (Regex) để bắt các yêu cầu. Ví dụ: `location ~ \.php$ { ... }` chỉ bắt các tệp kết thúc bằng đúng đuôi `.php`.
* **`try_files`**: Kiểm tra sự tồn tại của tệp trên đĩa cứng trước khi chuyển tiếp cho backend xử lý, tránh lỗi xử lý tệp ảo.
* **`fastcgi_pass`**: Chỉ định Socket hoặc địa chỉ IP/Cổng của PHP-FPM để chuyển tiếp xử lý.
* **`fastcgi_param`**: Truyền các tham số môi trường quan trọng (như đường dẫn tệp thực tế `SCRIPT_FILENAME`) cho PHP-FPM.
* **Nguy cơ bảo mật**: Nginx so khớp toàn bộ tên tệp bằng Regex. Tệp `shell.php.jpg` sẽ không khớp với đuôi `.php$` nên chỉ được xử lý như tệp ảnh tĩnh, giúp Nginx an toàn hơn trước cơ chế đa phần mở rộng.

### Cấu hình ánh xạ trên IIS

IIS quản lý cấu hình thông qua giao diện quản trị hoặc tệp tin cấu hình cấu trúc XML `web.config`:

* **`Handler Mapping`**: Định nghĩa module hoặc ứng dụng xử lý cho từng đuôi tệp (ví dụ bản đồ xử lý tệp `.php` qua module FastCGI).
* **`Request Filtering`**: Lọc và từ chối các yêu cầu chứa ký tự lạ, thư mục ẩn hoặc các phần mở rộng nguy hiểm.
* **`MIME Mapping`**: Quản lý danh sách các kiểu dữ liệu MIME trả về cho trình duyệt.

### Cấu hình chặn thực thi kịch bản trong thư mục Uploads

Để bảo vệ thư mục tải lên khỏi các cuộc tấn công tải tệp độc hại, cần vô hiệu hóa hoàn toàn quyền thực thi mã động trong thư mục này.

#### Cấu hình cấm chạy PHP trên Apache (Viết vào .htaccess trong thư mục uploads)

```apache
# Tắt engine xử lý PHP
php_flag engine off

# Hoặc cấm truy cập trực tiếp vào các định dạng script
<FilesMatch "\.(php|phtml|php3|php4|php5|php7|phar)$">
    Order Deny,Allow
    Deny from all
</FilesMatch>
```

#### Cấu hình cấm chạy PHP trên Nginx (Viết vào tệp cấu hình máy chủ chính)

```nginx
location ^~ /uploads/ {
    # Chỉ cho phép đọc tệp tĩnh, cấm truy cập tệp script PHP
    location ~* \.(php|phtml|php3|php4|php5|phar)$ {
        deny all;
    }
}
```

---

## Docker Debug (Phân tích cấu hình thực tế trên container)

### Tài liệu tham khảo:

* [Docker Command Line: exec](https://docs.docker.com/engine/reference/commandline/exec/)

Trong kiểm thử bảo mật, việc kết nối trực tiếp vào môi trường chạy ứng dụng để phân tích cấu hình thực tế là kỹ năng rất quan trọng.

### Lệnh truy cập vào Container

Sử dụng lệnh sau để truy cập vào giao diện dòng lệnh của container đang hoạt động:

```bash
docker exec -it <container_id_hoac_ten> /bin/bash
```

(Nếu container sử dụng Alpine Linux không có `/bin/bash`, hãy thay thế bằng `/bin/sh`).

### Các lệnh kiểm tra cấu hình hữu ích cho Pentester

Khi đã ở trong container, có thể dùng các lệnh sau để trích xuất cấu hình:

* **Kiểm tra Apache**:
  * `apachectl -M`: Liệt kê tất cả các module đang được nạp (kiểm tra xem `php_module` hay `rewrite_module` có hoạt động hay không).
  * `apachectl -S`: Hiển thị danh sách các cấu hình VirtualHost đang chạy trên máy chủ cùng cổng kết nối tương ứng.
* **Kiểm tra Nginx**:
  * `nginx -T`: Trích xuất toàn bộ nội dung cấu hình đang hoạt động của Nginx (bao gồm tất cả các tệp tin cấu hình phụ được nhúng vào tệp cấu hình chính).
* **Kiểm tra cấu hình PHP**:
  * `php -i`: Hiển thị toàn bộ thông tin cấu hình chi tiết của PHP (tương đương hàm `phpinfo()` nhưng chạy trên dòng lệnh).
  * `php -m`: Liệt kê tất cả các PHP Modules đang được nạp để xử lý mã nguồn.

### Quyền hạn trên hệ điều hành và quyền thực thi mã kịch bản

* Đối với các tệp tin chạy bằng ngôn ngữ thông dịch (như PHP), bộ dịch PHP chỉ cần quyền **Đọc (Read)** đối với tệp tin để có thể biên dịch và thực thi.
* Tệp tin PHP không cần được phân quyền thực thi ở cấp độ hệ điều hành (như quyền Execute - ví dụ phân quyền `chmod +x` hoặc chỉ số `755`) để có thể chạy trên trình duyệt web.
* Kiểm tra quyền sở hữu và phân quyền chi tiết các tệp tin trong thư mục tải lên bằng lệnh:
  ```bash
  ls -la /var/www/html/uploads/
  ```

---

## Lab

### Tài liệu tham khảo:
* [Chrome DevTools: Network Tab](https://developer.chrome.com/docs/devtools/network/)

### Mục tiêu
Thực hành tải lên các định dạng tệp tin khác nhau và trực tiếp quan sát các tiêu đề phản hồi (Headers) và nội dung phản hồi (Response Body) từ Web Server thông qua công cụ nhà phát triển để hiểu cơ chế xử lý của trình duyệt.

### Chuẩn bị các tệp kiểm thử
Tải các tệp tin có nội dung cụ thể dưới đây lên thư mục `/var/www/html/uploads/` trên máy chủ web thử nghiệm:

1. **`test.php`**
   ```php
   <?php echo "Mã PHP hoạt động!"; ?>
   ```
2. **`test.txt`**
   ```text
   Nội dung văn bản tĩnh.
   ```
3. **`test.html`**
   ```html
   <h1>HTML Rendered</h1>
   ```
4. **`test.svg`**
   ```xml
   <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 100 100" width="100" height="100">
       <script>alert('XSS kích hoạt!');</script>
       <rect width="100" height="100" fill="red" />
   </svg>
   ```
5. **`test.xyz`** (Chứa nội dung bất kỳ để kiểm tra định dạng lạ).

### Các bước tiến hành
1. Mở trình duyệt web, nhấn phím F12 để mở Công cụ nhà phát triển và chọn thẻ **Network (Mạng)**.
2. Truy cập trực tiếp đường dẫn của từng tệp tin vừa tải lên (ví dụ: `http://localhost/uploads/test.php`).
3. Quan sát hành vi thực tế của trình duyệt, sau đó bấm vào tên yêu cầu tương ứng trong tab Network.
4. Kiểm tra các thông số trong phần Headers bao gồm: `Status Code`, `Content-Type`, `Content-Disposition`, và `Content-Length`.
5. Chuyển sang tab **Response** trong DevTools để quan sát cấu trúc của Response Body trả về.

### Bảng ghi nhận kết quả quan sát
Kết quả kiểm thử thực tế thu được trên máy chủ cấu hình mặc định (chưa áp dụng cấu hình chặn thực thi script):

| Tên tệp tin | HTTP Status | Header Content-Type | Response Body nhận được | Hành vi trên trình duyệt | Phân tích bản chất |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `test.php` | 200 OK | `text/html` | `Mã PHP hoạt động!` (Đoạn mã `<?php ... ?>` đã biến mất) | Hiển thị dòng chữ: "Mã PHP hoạt động!" | Máy chủ đã thực thi mã PHP và chỉ gửi kết quả đầu ra (Response Body) dạng HTML về cho trình duyệt. Đây là bằng chứng cho thấy mã nguồn thô chạy hoàn toàn trên server. |
| `test.txt` | 200 OK | `text/plain` | `Nội dung văn bản tĩnh.` | Hiển thị nguyên văn văn bản thô. | Máy chủ đọc trực tiếp tệp tĩnh từ đĩa cứng và gửi đi, trình duyệt hiển thị ký tự đơn giản. |
| `test.html` | 200 OK | `text/html` | `<h1>HTML Rendered</h1>` | Dựng giao diện hoàn chỉnh (Hiển thị tiêu đề h1 đậm). | Trình duyệt nhận mã HTML, dựng cấu trúc trang và hiển thị giao diện đồ họa. |
| `test.svg` | 200 OK | `image/svg+xml` | Toàn bộ mã nguồn XML của tệp SVG | Hiện hình vuông màu đỏ và hiển thị hộp thoại alert JavaScript. | Trình duyệt vẽ ảnh vector nhưng do định dạng XML của SVG cho phép chạy script, mã JavaScript bên trong được kích hoạt. |
| `test.xyz` | 200 OK | `application/octet-stream` | Nội dung thô của tệp tin | Tự động tải tệp tin xuống máy. | Máy chủ không nhận dạng được đuôi `.xyz` nên dùng kiểu mặc định, trình duyệt bắt buộc tải về. |

### Lưu ý bảo mật quan trọng về tệp SVG
* Không phải mọi trường hợp tải lên tệp SVG đều tự động kích hoạt được lỗi XSS thành công.
* Hành vi thực thi mã JavaScript bên trong tệp SVG phụ thuộc hoàn toàn vào cách ứng dụng web hiển thị tệp tin đó:
  * Nếu người dùng truy cập trực tiếp URL của tệp SVG trên thanh địa chỉ hoặc nhúng tệp qua các thẻ `<iframe>` hoặc `<object>`: Trình duyệt **sẽ thực thi** mã JavaScript bên trong.
  * Nếu tệp SVG được nhúng dưới dạng nguồn ảnh tĩnh bằng thẻ `<img src="test.svg">` hoặc thuộc tính CSS `background-image`: Trình duyệt **sẽ vô hiệu hóa hoàn toàn** việc chạy script để bảo vệ hệ thống.
  * Việc thực thi mã JavaScript cũng bị kiểm soát chặt chẽ bởi các chính sách bảo mật nội dung CSP (Content Security Policy) được cấu hình trên website.

---

## Chuẩn bị cho Upload Bypass (Liên hệ đến các kỹ thuật bypass cấu hình lỗi)

### Tài liệu tham khảo:
* [PortSwigger Web Security Academy: File upload vulnerabilities](https://portswigger.net/web-security/file-upload)
* [OWASP WSTG: Test File Upload](https://owasp.org/www-project-web-security-testing-guide/v42/4-Web_Application_Security_Testing/10-Business_Logic_Testing/01-Test_Business_Logic_Data_Validation)

Lỗ hổng thực thi mã từ xa (RCE) xảy ra khi kẻ tấn công tìm cách lách qua các bộ lọc của ứng dụng và tận dụng lỗi cấu hình máy chủ web.

### Kịch bản lợi dụng cơ chế đa phần mở rộng (Multi-extension) trên Apache
Trong Apache cấu hình lỏng lẻo, tệp `shell.php.jpg` có thể được xử lý như một tệp PHP do cơ chế quét từ phải qua trái.
Ứng dụng kiểm tra thấy đuôi cuối cùng là `.jpg` nên cho phép tải lên.
Tuy nhiên, khi kẻ tấn công truy cập trực tiếp, Apache vẫn chuyển tệp này cho PHP Engine thực thi.

### Kịch bản lợi dụng tệp cấu hình cục bộ .htaccess của Apache
Nếu máy chủ Apache cho phép ghi đè cấu hình (`AllowOverride All`), kẻ tấn công có thể tải lên tệp `.htaccess` tự chế.
Tệp `.htaccess` này cấu hình cho phép chạy các đuôi thông thường như `.txt` dưới dạng PHP:
```apache
AddType application/x-httpd-php .txt
```
Kẻ tấn công tải lên tiếp tệp `shell.txt` chứa mã PHP và truy cập để kích hoạt RCE trên hệ thống.

### Kịch bản khai thác các phần mở rộng thay thế được máy chủ hỗ trợ
Nếu bộ lọc của ứng dụng chỉ chặn đuôi `.php` theo cơ chế danh sách đen (Blacklist), kẻ tấn công có thể thử các đuôi thay thế.
Các đuôi tệp như `.phtml`, `.php5` hay `.phar` thường vẫn được máy chủ web cấu hình mặc định để dịch bằng PHP Engine.
Tận dụng điều này giúp vượt qua bộ lọc của ứng dụng và thực thi mã độc thành công.

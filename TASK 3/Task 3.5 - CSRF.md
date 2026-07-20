# CSRF

## Tài liệu tham khảo

[viblo.asia/p/ky-thuat-tan-cong-csrf-va-cach-phong-chong-amoG84bOGz8P](https://viblo.asia/p/ky-thuat-tan-cong-csrf-va-cach-phong-chong-amoG84bOGz8P)

[portswigger.net/web-security/csrf](https://portswigger.net/web-security/csrf)

[developer.mozilla.org/en-US/docs/Web/Security/Attacks/CSRF](https://developer.mozilla.org/en-US/docs/Web/Security/Attacks/CSRF)

[owasp.org/www-community/attacks/csrf](https://owasp.org/www-community/attacks/csrf)

## Bố cục nội dung

```
CSRF (Cross-Site Request Forgery)

├── 1. Tổng quan
│   ├── CSRF là gì?
│   ├── Vì sao CSRF xảy ra?
│   ├── Mối liên hệ với Browser, SOP và CORS
│   ├── Phân biệt Cookie, Session, Token và Credentials
│   └── Luồng hình thành CSRF
│
├── 2. Browser gửi Request như thế nào?
│   ├── Cookie và Session
│   ├── Credentials
│   ├── HTML Form
│   ├── IMG, SCRIPT, LINK, IFRAME
│   ├── Fetch API
│   └── XMLHttpRequest
│
├── 3. Điều kiện khai thác
│   ├── Cookie-based Authentication
│   ├── Không có CSRF Token
│   ├── SameSite Cookie
│   ├── Origin
│   ├── Referer
│   └── User Interaction
│
├── 4. Các loại CSRF
│   ├── GET CSRF
│   ├── POST CSRF
│   ├── Login CSRF
│   ├── JSON CSRF
│   └── Multi-step CSRF
│
├── 5. Phân tích an ninh
│   ├── Điều kiện khai thác
│   ├── Tác động
│   ├── Khi nào CSRF không xảy ra?
│   └── Ví dụ thực tế
│
└── 6. Phòng chống
    ├── Synchronizer Token
    ├── Double Submit Cookie
    ├── SameSite Cookie
    ├── Origin Validation
    ├── Referer Validation
    ├── Re-authentication
    └── Mitigation Checklist
```

## 1. Tổng quan

### CSRF là gì?

CSRF là viết tắt của Cross Site Request Forgery, tức tấn công giả mạo yêu cầu từ một trang web khác. Mục tiêu của kỹ thuật này là lợi dụng trình duyệt để gửi yêu cầu đến một website mà người dùng đã đăng nhập, mà người dùng không nhận ra hoặc không có ý định thực hiện.

Khi tấn công thành công, hệ thống sẽ hiểu rằng đây là một yêu cầu hợp lệ vì nó đến từ trình duyệt có cookie hợp lệ. Nói cách khác, attacker không cần đánh cắp mật khẩu hay cookie trực tiếp. Thay vào đó, attacker chỉ cần khiến người dùng thực hiện một hành động trong trình duyệt.

### Vì sao CSRF xảy ra?

CSRF xảy ra khi trình duyệt tự động gửi thông tin xác thực đến website mà người dùng đã đăng nhập. Nếu server chỉ dựa vào thông tin đó để xác thực và không kiểm tra nguồn gốc của request hoặc không yêu cầu CSRF token, attacker có thể lợi dụng trình duyệt của nạn nhân để gửi yêu cầu giả mạo.

Browser đóng vai trò rất quan trọng trong vụ việc này. Nó không chỉ gửi cookie mà còn gửi các thông tin xác thực theo đúng quy tắc của trình duyệt.

Sơ đồ đơn giản như sau:

```text
Attacker
   |
   | mở website độc hại
   v
Victim Browser
   |
   | tự động gửi Cookie
   v
Bank Server
   |
   | Cookie hợp lệ
   v
Thực hiện hành động
```

### Mối liên hệ với Browser, SOP và CORS

Browser là trung tâm của vụ việc này. Browser chịu trách nhiệm lưu cookie, quản lý session và gửi request cho các website. Trong khi đó, SOP là nguyên tắc ngăn các script ở một origin khác đọc dữ liệu của origin khác. CORS là cơ chế cho phép một origin khác được phép chia sẻ tài nguyên trong một số trường hợp.

CSRF thường lợi dụng điểm yếu sau đây:

1. Browser vẫn gửi cookie cho request đến domain đích.
2. Server không kiểm tra đủ mạnh xem request có phải do người dùng chủ động tạo hay không.
3. Một số kỹ thuật như form tự submit, ảnh, iframe hoặc script có thể kích hoạt request mà không cần người dùng click chuột.

### Phân biệt Cookie, Session, Token và Credentials

Bảng dưới đây giúp phân biệt các khái niệm thường bị nhầm lẫn trong CSRF:

| Khái niệm | Ý nghĩa                                                     | Vai trò trong CSRF                                                                                     |
| ----------- | ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| Cookie      | Dữ liệu trình duyệt lưu lại cho một website            | Thường là loại credentials dễ bị lợi dụng nhất vì browser tự động gửi lại                |
| Session     | Trạng thái đăng nhập được server ghi nhớ             | Thường được gắn với cookie hoặc session ID để nhận diện người dùng                       |
| Token       | Chuỗi giá trị dùng để xác thực hoặc chống giả mạo | CSRF token dùng để kiểm tra request có hợp lệ hay không                                         |
| Credentials | Thông tin xác thực đi kèm request                        | Trong CSRF, credentials chủ yếu là cookie, còn Bearer token thì không tự động gửi như cookie |

Một cách đơn giản để nhớ:

- Cookie là vật mang thông tin
- Session là trạng thái đăng nhập
- Token là chứng chỉ để xác thực
- Credentials là tập hợp các thông tin xác thực đi cùng request

### CSRF vs XSS

| Khía cạnh                                | CSRF                                                  | XSS                                            |
| ------------------------------------------ | ----------------------------------------------------- | ---------------------------------------------- |
| Mục tiêu chính                          | Kích hoạt request giả mạo                         | Chạy mã độc trên trang web mục tiêu     |
| Nguyên lý                                | Lợi dụng browser gửi cookie và request tự động | Lợi dụng JavaScript chạy trong cùng origin |
| Có cần chạy script trên victim không  | Không                                                | Có                                            |
| Có thể đọc dữ liệu của trang không | Thường không                                       | Có                                            |
| Thường bị tấn công bởi               | Form, ảnh, iframe, script                            | Script độc hại chèn vào trang             |

### CSRF token vs Session cookie

| Khái niệm    | Vai trò                                                                  | Mức độ ảnh hưởng đến CSRF                                               |
| -------------- | ------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| Session cookie | Dùng để nhận diện người dùng đã đăng nhập                    | Rất quan trọng, vì đây thường là credential bị lợi dụng              |
| CSRF token     | Dùng để xác nhận request có hợp lệ và do trang web đó tạo     | Giúp phòng chống CSRF hiệu quả                                             |
| Mối quan hệ  | Cookie thường mang session, token có thể nằm trong form hoặc header | Cookie không đủ để chống CSRF nếu không có token hoặc kiểm tra khác |

### Luồng hình thành CSRF

Sơ đồ đơn giản như sau:

```text
Người dùng đăng nhập vào website A
        |
        v
Trình duyệt lưu cookie cho website A
        |
        v
Người dùng mở trang độc hại B
        |
        v
Trang B gửi request đến website A
        |
        v
Website A nhận request và xử lý như một request hợp lệ
```

Ví dụ thực tiễn: người dùng đang đăng nhập ngân hàng trực tuyến. Một trang web độc hại gửi một request đổi mật khẩu hoặc chuyển tiền đến ngân hàng. Vì trình duyệt vẫn mang cookie của ngân hàng, hệ thống có thể chấp nhận request đó.

## 2. Browser gửi Request như thế nào?

### Cookie và Session

Cookie là dữ liệu trình duyệt lưu lại cho một website. Session là cách server ghi nhớ trạng thái của người dùng dựa trên cookie hoặc token.

Khi người dùng đăng nhập, server có thể tạo session và lưu session id trong cookie. Lúc đó, mỗi lần gửi request đến server, trình duyệt sẽ tự động gửi cookie đó đi. Đây là điều khiến CSRF trở nên nguy hiểm.

### Credentials

Trong bối cảnh CSRF, credentials chủ yếu là cookie được trình duyệt tự động gửi. Đây là loại thông tin xác thực dễ bị lợi dụng nhất vì browser sẽ mang nó theo request mà không cần người dùng nhận ra.

Trong một số trường hợp khác, credentials có thể là TLS client certificate hoặc HTTP authentication. Tuy nhiên, những thông tin như Authorization Bearer token thường không bị CSRF tấn công trực tiếp vì browser không tự động thêm chúng. JavaScript phải tự thêm header đó, nên attacker khó lợi dụng được.

### HTML Form

HTML form là một cách rất phổ biến để tạo request. Khi người dùng submit form, browser sẽ gửi dữ liệu đến action của form. Nếu attacker tạo một form ẩn và tự động submit, người dùng có thể vô tình gửi request mà không nhận ra.

Ví dụ:

```html
<form action="https://bank.example.com/transfer" method="POST">
  <input type="hidden" name="amount" value="1000000">
  <input type="hidden" name="to" value="attacker">
  <input type="submit" value="Submit">
</form>
<script>
  document.forms[0].submit();
</script>
```

### IMG, SCRIPT, LINK, IFRAME

Một số thẻ HTML có thể khiến browser gửi request mà không cần người dùng click. Ví dụ:

```html
<img src="https://bank.example.com/transfer?amount=1000&to=attacker">
```

Khi trình duyệt tải ảnh, nó sẽ tạo một request GET đến URL đó. Nếu server chấp nhận hành động này mà không kiểm tra token, request có thể bị thực thi.

Trong thực tế:

- img tạo request GET
- link tạo request GET
- script src tạo request GET
- iframe cũng chủ yếu tạo request GET

Những cách này thường không dùng để gửi POST.

### Fetch API

Fetch API cho phép JavaScript gửi request đến server. Fetch vẫn có thể gửi request cross-origin. Tuy nhiên, việc gửi cookie phụ thuộc vào thuộc tính credentials và chính sách SameSite của cookie. Ngoài ra, JavaScript trên trang nguồn sẽ không thể đọc response nếu server không cho phép thông qua CORS.

### XMLHttpRequest

XMLHttpRequest cũng có thể tạo request từ JavaScript. Request vẫn có thể được gửi cross-origin, nhưng response bị chặn nếu server không cho phép qua CORS. Đây là điểm khác biệt quan trọng so với việc request có được gửi hay không.

## 3. Điều kiện khai thác

### Cookie based Authentication

CSRF dễ khai thác khi hệ thống dựa vào cookie để xác thực người dùng. Nếu server chỉ cần cookie hợp lệ mà không yêu cầu thêm bằng chứng nào khác, attacker có thể lợi dụng điều này.

### Không có CSRF Token

CSRF token là một giá trị ngẫu nhiên được server tạo ra cho mỗi session hoặc mỗi request quan trọng. Nếu server không yêu cầu token cho các hành động thay đổi trạng thái như đổi mật khẩu, chỉnh hồ sơ, chuyển tiền thì attacker có thể lợi dụng.

### SameSite Cookie

SameSite là thuộc tính của cookie. Nếu cookie được đặt với SameSite=Lax hoặc Strict thì trình duyệt sẽ ít gửi cookie trong các request cross-site. SameSite=Strict gần như không gửi cookie trong các request cross-site. SameSite=Lax vẫn cho phép một số trường hợp top-level navigation bằng GET. Điều này làm giảm khả năng CSRF. Nếu cookie dùng SameSite=None thì nguy cơ tăng lên, đặc biệt khi trình duyệt hỗ trợ chế độ cross-site.

### Origin

Origin là nơi bắt nguồn của request. Server có thể kiểm tra Origin header để xác định request đến từ website nào. Nếu Origin không khớp với domain mong đợi, server có thể bỏ qua request. Origin thường đáng tin cậy hơn Referer vì Referer có thể bị xóa, bị cắt ngắn hoặc bị policy ảnh hưởng.

### Referer

Referer là thông tin cho biết request đến từ trang nào. Server có thể dùng nó để kiểm tra liệu request có xuất phát từ một trang hợp lệ trong cùng hệ thống hay không.

### User Interaction

CSRF thường không cần người dùng biết. Tuy nhiên, trong nhiều trường hợp người dùng vẫn cần thực hiện một thao tác như mở trang, nhấp vào liên kết hoặc truy cập một trang web độc hại. Đây là yếu tố làm cho tấn công có thể xảy ra trong thực tế.

## 4. Các loại CSRF

### GET CSRF

Đây là kiểu đơn giản nhất. Attacker tạo một đường dẫn khiến browser gửi request GET tới server. Ví dụ:

```html
<img src="https://example.com/change-email?email=attacker@example.com">
```

Nếu server chấp nhận thao tác này qua GET và không kiểm tra bảo mật, hành động có thể diễn ra.

### POST CSRF

Kiểu này sử dụng form hoặc JavaScript để gửi request POST. Đây là phương thức phổ biến hơn vì nhiều thao tác quan trọng dùng POST.

```html
<form action="https://example.com/change-password" method="POST">
  <input type="hidden" name="newPassword" value="123456">
</form>
<script>document.forms[0].submit();</script>
```

### Login CSRF

Attackers có thể buộc người dùng đăng nhập vào một tài khoản do attacker kiểm soát. Sau đó, khi người dùng thực hiện các hành động trên hệ thống, hành động đó sẽ được ghi nhận dưới tài khoản của attacker.

Ví dụ kinh điển:

```html
<form action="https://example.com/login" method="POST">
  <input type="hidden" name="username" value="attacker">
  <input type="hidden" name="password" value="123456">
</form>
<script>document.forms[0].submit();</script>
```

Sau khi nạn nhân bị buộc đăng nhập vào tài khoản attacker, bất kỳ hành động nào như upload file, gửi dữ liệu hoặc thay đổi hồ sơ cũng có thể được lưu dưới tài khoản attacker.

### JSON CSRF

JSON CSRF hiếm hơn POST Form CSRF. Vì Content-Type: application/json không phải simple request, trình duyệt sẽ gửi preflight. Nếu server không cho phép hoặc cấu hình không đủ an toàn, cuộc tấn công sẽ khó thực hiện. JSON CSRF thường chỉ xảy ra khi server parse sai Content-Type, framework chấp nhận nhiều kiểu Content-Type hoặc có cấu hình không an toàn.

### Multi step CSRF

Đây là dạng tấn công nhiều bước. Attacker không chỉ gửi một request đơn lẻ mà liên tiếp nhiều request để thực hiện một chuỗi hành động. Ví dụ: tạo tài khoản, đổi quyền, thiết lập mật khẩu, sau đó đăng nhập.

## 5. Phân tích an ninh

### Điều kiện khai thác

CSRF không phải là tấn công đánh cắp dữ liệu trực tiếp. Nó thường hoạt động khi:

1. Server sử dụng session hoặc cookie để xác thực.
2. Request thay đổi trạng thái như đổi mật khẩu, chuyển tiền, chỉnh hồ sơ.
3. Server không có cơ chế phòng chống đủ mạnh.

### Tác động

Tác động của CSRF có thể rất nghiêm trọng. Nó có thể dẫn đến:

1. Chuyển tiền mà không có ý định của người dùng.
2. Đổi mật khẩu hoặc thay đổi email.
3. Thêm tài khoản quản trị hoặc thay đổi quyền.
4. Gửi dữ liệu nhạy cảm không mong muốn.

### Khi nào CSRF không xảy ra?

CSRF không xảy ra hoặc ít xảy ra khi:

1. Server yêu cầu CSRF token cho mọi thao tác quan trọng.
2. Cookie được cấu hình SameSite=Lax hoặc Strict.
3. Server kiểm tra Origin hoặc Referer.
4. Các hành động quan trọng yêu cầu xác minh lại bằng mật khẩu hoặc MFA.
5. Ứng dụng dùng JWT trong Authorization header thay vì cookie. Browser không tự gửi JWT này, nên attacker khó lợi dụng để tạo CSRF. Tuy nhiên, đây lại là vùng nguy hiểm khác là XSS.

### Ví dụ thực tế

Giả sử một người dùng đang đăng nhập vào hệ thống quản trị nội bộ. Một trang web độc hại chứa một form ẩn gửi request đổi quyền cho một tài khoản. Nếu hệ thống không có CSRF token và không kiểm tra Origin, request có thể được thực hiện thành công. Kết quả là tài khoản bị nâng quyền mà không hề có sự đồng ý từ người dùng.

### CSRF vs XSS

CSRF và XSS đều là lỗ hổng web nhưng có cách hoạt động khác nhau:

- CSRF lợi dụng browser gửi cookie và request tự động.
- XSS cho phép JavaScript chạy trên trang mục tiêu.
- CSRF không cần chạy script trên victim browser.
- XSS có thể đọc dữ liệu, đánh cắp cookie hoặc token vì code chạy trong cùng origin.

Nói ngắn gọn, CSRF là tấn công bằng cách khiến browser gửi request giả mạo, còn XSS là tấn công bằng cách cho code độc hại chạy trên trang web mục tiêu.

## 6. Phòng chống

### Synchronizer Token

Đây là phương pháp phổ biến nhất. Server tạo một token ngẫu nhiên cho mỗi session và chèn token vào form hoặc header. Khi nhận request, server kiểm tra token đó có khớp với session hay không.

Ưu điểm của phương pháp này là hiệu quả khi áp dụng đúng cách. Nhược điểm là cần triển khai đúng ở cả phía server và phía client.

### Double Submit Cookie

Phương pháp này dùng một token trong cookie và một token trong request. Server chỉ chấp nhận khi cả hai giá trị khớp nhau. Cách này giúp giảm rủi ro vì attacker khó chèn được giá trị đúng vào cả cookie và request.

### SameSite Cookie

Việc thiết lập SameSite cho cookie là một biện pháp phòng chống rất quan trọng. SameSite=Lax hoặc Strict giúp giới hạn việc trình duyệt gửi cookie trong các request cross site.

### Origin Validation

Server nên kiểm tra Origin header của request. Nếu Origin không khớp với domain được phép, request có thể bị từ chối.

### Referer Validation

Referer cũng có thể được dùng như một lớp kiểm tra bổ sung. Tuy nhiên, vì Referer có thể bị xóa hoặc bị thay đổi trên một số trình duyệt hoặc môi trường, cách này nên dùng kết hợp với các cơ chế khác.

### Re authentication

Đối với các thao tác rất nhạy cảm như chuyển tiền lớn, đổi mật khẩu chính, xóa tài khoản, hệ thống nên yêu cầu xác minh lại bằng mật khẩu hoặc MFA trước khi thực hiện.

### Mitigation Checklist

Một checklist phòng chống CSRF có thể bao gồm:

1. Không dùng GET cho các thao tác thay đổi trạng thái.
2. Thêm CSRF token cho các form và request quan trọng.
3. Đặt SameSite cho cookie ở mức phù hợp.
4. Kiểm tra Origin và Referer khi có thể.
5. Yêu cầu xác minh lại cho các hành động nhạy cảm.
6. Thực hiện kiểm thử thủ công và tự động để đảm bảo không còn lỗ hổng.

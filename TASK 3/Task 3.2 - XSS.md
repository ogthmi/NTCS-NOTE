# Tấn công XSS (Cross-site Scripting)

## Tài liệu tham khảo

[Viblo: Kỹ thuật tấn công XSS và cách ngăn chặn](https://viblo.asia/p/ky-thuat-tan-cong-xss-va-cach-ngan-chan-YWOZr0Py5Q0)

[portswigger.net: XSS and prevention](https://portswigger.net/web-security/cross-site-scripting)

[MDN: XSS](https://developer.mozilla.org/en-US/docs/Web/Security/Attacks/XSS)

[OWASP: XSS](https://owasp.org/www-community/attacks/xss/)

## Bố cục nội dung

```
XSS (Cross-Site Scripting)

├── 1. Tổng quan
│   ├── XSS là gì?
│   ├── Vì sao XSS xảy ra?
│   ├── Mối liên hệ với Browser và SOP
│   └── Luồng hình thành XSS
│
├── 2. Browser xử lý dữ liệu như thế nào?
│   ├── HTML Parser
│   ├── DOM Tree
│   ├── JavaScript Engine
│   ├── Rendering Flow
│   └── Khi nào JavaScript được thực thi?
│
├── 3. Context Injection
│   ├── HTML Context
│   ├── HTML Attribute Context
│   ├── JavaScript Context
│   ├── CSS Context
│   └── URL Context
│
├── 4. Sources và Sinks
│   ├── Source
│   ├── Sink
│   ├── Dangerous Sink
│   └── Safe Sink
│
├── 5. Các loại XSS
│   ├── Reflected XSS
│   ├── Stored XSS
│   └── DOM-based XSS
│
├── 6. Phân tích an ninh
│   ├── Điều kiện khai thác
│   ├── Tác động
│   ├── Kiểm thử
│   └── Ví dụ thực tế
│
└── 7. Phòng chống XSS
    ├── Output Encoding
    ├── HTML Sanitization
    ├── Input Validation
    ├── CSP
    ├── HttpOnly Cookie
    ├── Trusted Types
    └── Mitigation Checklist
```

## Tổng quan về XSS

### XSS là gì?

XSS (Cross-Site Scripting) là một lỗ hổng bảo mật ứng dụng web xảy ra khi dữ liệu không đáng tin cậy được đưa vào một context có khả năng thực thi trong trình duyệt, khiến JavaScript được chạy dưới quyền của website bị ảnh hưởng.

### Vì sao XSS xảy ra?

XSS xảy ra khi dữ liệu không đáng tin cậy được đưa trở lại trang web mà không được xử lý đúng ngữ cảnh. Nếu dữ liệu từ URL, form, API hoặc cơ sở dữ liệu được chèn trực tiếp vào HTML, JavaScript, attribute hoặc URL, trình duyệt có thể hiểu nó như mã thực thi.

### Mối liên hệ giữa XSS và SOP

SOP ngăn JavaScript ở một nguồn khác đọc dữ liệu của một nguồn khác. XSS không vượt qua Same-Origin Policy mà lợi dụng việc mã JavaScript độc hại được thực thi bên trong origin hợp lệ của ứng dụng. Vì vậy trình duyệt coi đoạn mã này là một phần của website và cấp quyền tương ứng với origin đó.

### Luồng hình thành XSS

1. Người dùng hoặc attacker đưa dữ liệu vào hệ thống.
2. Dữ liệu được lưu hoặc phản hồi trực tiếp từ server.
3. Browser render dữ liệu vào một context có thể thực thi mã.
4. Payload được trình duyệt parse và thực thi.
5. Hành vi độc hại xảy ra trên trang đang mở.

## Cơ chế biên dịch của trình duyệt

### HTML Parser và JavaScript Parser

Browser xử lý tài liệu web theo từng giai đoạn. HTML Parser tạo DOM từ markup, JavaScript Parser phân tích và thực thi script. Một chuỗi ký tự có thể là dữ liệu bình thường trong một nơi, nhưng lại trở thành mã thực thi ở nơi khác.

Ví dụ:

```html
<div><script>alert(1)</script></div>
```

```
HTTP Response
      |
      v
HTML Parser
      |
      v
DOM Tree
      |
      +---- CSS Parser
      |
      +---- JavaScript Engine
      |
      v
Render Tree
      |
      v
Page Display
```

Ở đây, browser hiểu phần bên trong là HTML và script thực thi.

### DOM (Document Object Model)

DOM là cấu trúc cây biểu diễn trang web. JavaScript có thể đọc và sửa DOM. Nếu dữ liệu độc hại được chèn qua các API như innerHTML, outerHTML hoặc document.write, browser sẽ tạo lại cây DOM và có thể thực thi payload.

Ví dụ:

```js
const x = location.hash.slice(1);
document.body.innerHTML = x;
```

Nếu URL chứa `<img src=x onerror=alert(1)>`, payload có thể bị thực thi.

## Context (bối cảnh thực thi)

### Các loại context

Một chuỗi ký tự có thể an toàn ở một context nhưng nguy hiểm ở context khác. Đây là điểm mấu chốt của XSS.

| Context    | Nơi dặt dữ liệu                                              | Ví dụ                  | Rủi ro            |
| ---------- | ---------------------------------------------------------------- | ------------------------ | ------------------ |
| HTML       | dữ liệu được chèn vào nội dung trang                     | `<div>USER</div>`      | Chèn HTML tag     |
| Attribute  | dữ liệu được đặt trong thuộc tính như href, src, value | `<input value="USER">` | Escape attribute   |
| JavaScript | dữ liệu được đưa vào đoạn mã JS                       | `var x="USER"`         | Break string       |
| CSS        | dữ liệu được đưa vào style sheet                         | `style="USER"`         | CSS injection      |
| URL        | dữ liệu được đưa vào URL hoặc tham số link             | `<a href="USER">`      | javascript: scheme |

Ví dụ:

```html
<a href="javascript:alert(1)">Click</a>
```

Đây là payload nguy hiểm vì browser hiểu javascript: như một lệnh thực thi.

## Cơ chế nguồn và đích dữ liệu (Sources và Sinks)

### Source là gì

Source là nơi dữ liệu không đáng tin cậy được thu thập. Ví dụ: query parameter, form input, cookie, API response, localStorage, postMessage, URL fragment.

Ví dụ:

```js
const name = new URLSearchParams(location.search).get('name');
```

`name` là một source.

### Sink là gì

Sink là nơi dữ liệu bị browser hiểu như mã hoặc được chèn vào DOM theo cách có thể thực thi. Các sink nguy hiểm thường là các API dùng để tạo nội dung động.

Ví dụ:

```js
document.getElementById('output').innerHTML = name;
```

Đây là sink nguy hiểm vì dữ liệu được chèn trực tiếp vào HTML.

### Sink an toàn và sink nguy hiểm

- Sink nguy hiểm: innerHTML, outerHTML, document.write, eval, Function, setTimeout(string), setInterval(string).
- Sink an toàn hơn: textContent, innerText, value, DOM APIs dùng để gán giá trị thuần túy.

### Ví dụ:

```js
document.body.textContent = name; // an toàn hơn
```

## Một số dạng Payload XSS phổ biến

> Các payload chỉ có ý nghĩa khi tồn tại context phù hợp.

### Script tag

```html
<script>alert(1)</script>
```

Payload này có thể hoạt động khi context cho phép chèn thẻ script vào trang.

### Event Handler

```html
<img src=x onerror=alert(1)>
```

Khi trình duyệt tải ảnh thất bại, handler onerror được thực thi.

### SVG

```html
<svg onload=alert(1)>
```

SVG có thể bị lợi dụng để thực thi mã khi được chèn vào DOM.

### IMG Error

```html
<img src=1 onerror=alert(1)>
```

Một payload rất phổ biến vì dễ chèn vào các chỗ hiển thị nội dung người dùng.

### JavaScript URL

```html
<a href="javascript:alert(1)">Click</a>
```

Payload này nguy hiểm trong context của thuộc tính URL.

## Các kiểu tấn công XSS

### Reflected XSS

Reflected XSS xảy ra khi payload được gửi từ client và phản hồi ngay lại trong response. Payload thường đi qua URL hoặc form và được server trả về trực tiếp.

Ví dụ:

```text
/search?q=<script>alert(1)</script>
```

Nếu ứng dụng phản ánh lại `q` vào trang, payload sẽ được thực thi.

Flow tấn công

```
Attacker
   |
URL payload
   |
Server
   |
HTML Response
   |
Victim Browser
   |
Execute
```

### Stored XSS

Stored XSS xảy ra khi payload được lưu vào cơ sở dữ liệu, file, cache hoặc cấu hình và sau đó được trình bày lại cho nhiều người dùng khác.

Ví dụ: bình luận chứa `<img src=x onerror=alert(1)>`, khi được hiển thị trên trang, payload sẽ chạy cho tất cả người xem.

Flow tấn công

```
Attacker
   |
Payload
   |
Database
   |
Victim request
   |
Browser execute
```

### DOM-based XSS

DOM-based XSS không cần server phản ánh payload. Mã độc được tạo và thực thi hoàn toàn ở phía client bằng JavaScript.

Ví dụ:

```js
const user = new URLSearchParams(location.search).get('user');
document.getElementById('name').innerHTML = user;
```

Nếu URL chứa payload, browser sẽ thực thi nó ngay trên trang.

Flow tấn công

```
URL
 |
JavaScript Source
 |
Dangerous Sink
 |
Execute
```

## Phân tích an ninh mạng

### Điều kiện khai thác

XSS chỉ thành công khi có đủ ba điều kiện: dữ liệu không đáng tin cậy đi vào hệ thống, dữ liệu được render vào một context nguy hiểm, và browser cho phép payload được thực thi.

### Tác động

XSS có thể dẫn đến

- đánh cắp cookie
- lấy thông tin phiên đăng nhập
- chỉnh sửa DOM
- giả mạo giao diện
- gửi request trái phép hoặc lấy dữ liệu nhạy cảm khỏi trang đang mở.

### Kiểm thử

Quá trình kiểm thử XSS thường bắt đầu bằng việc tìm các điểm nhập dữ liệu rồi thử chèn payload vào các context khác nhau. Cần kiểm tra cả URL, form, header, comment, search box và các endpoint trả dữ liệu động.

### Ví dụ thực tế

Một hệ thống cho phép người dùng đăng bình luận. Nếu nội dung bình luận được render bằng innerHTML mà không lọc ký tự, attacker có thể chèn một hình ảnh không tồn tại để kích hoạt onerror và chạy mã độc cho tất cả người xem.

## Phòng chống XSS

### Output Encoding

Encoding là cách chuyển đổi ký tự đặc biệt thành dạng an toàn trước khi chèn vào HTML, attribute, URL hoặc JavaScript. Mục tiêu là ngăn browser hiểu dữ liệu đó như mã thực thi.

Ví dụ: `<` cần được encode thành `&lt;` khi render vào HTML.

### HTML Sanitization

Sanitization là lọc nội dung người dùng để chỉ giữ các tag và thuộc tính an toàn. Đây là phương pháp phổ biến cho các hệ thống cho phép người dùng nhập rich text.

### Input Validation

Input validation giúp giới hạn loại dữ liệu hợp lệ. Tuy nhiên, đây không phải lớp phòng thủ duy nhất vì nhiều input hợp lệ vẫn có thể gây hại nếu không được encode đúng context.

### CSP

Content Security Policy giới hạn nguồn tài nguyên mà browser được phép tải và thực thi. CSP có thể ngăn chặn inline script, external script không được phép và giảm phạm vi tấn công XSS.

### HttpOnly Cookie

Cookie nên được thiết lập HttpOnly để JavaScript không thể đọc giá trị cookie bằng document.cookie. Điều này làm giảm khả năng đánh cắp session bằng XSS.

### Trusted Types

Trusted Types buộc các API DOM phải nhận dữ liệu được đánh dấu là an toàn. Đây là một cơ chế hiện đại giúp giảm nguy cơ XSS trong các ứng dụng JavaScript phức tạp.

### Mitigation Checklist

- Tránh dùng innerHTML cho dữ liệu không đáng tin cậy.
- Dùng textContent cho nội dung văn bản.
- Encode dữ liệu theo đúng context.
- Sanitize nội dung do người dùng nhập.
- Triển khai CSP phù hợp.
- Đặt HttpOnly, Secure và SameSite cho cookie.
- Kiểm thử payload trên các điểm nhập dữ liệu khác nhau.

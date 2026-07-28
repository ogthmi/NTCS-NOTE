# PHP Core cho Pentester

## PHP hoạt động như thế nào?

Browser không chạy PHP. Luồng thực tế như sau:

```text
Browser
   │
HTTP Request
   │
   ▼
Apache / Nginx
   │
   ▼
PHP Engine
   │
   ▼
index.php chạy
   │
   ▼
HTML / JSON / File
   │
Browser
```

Tức là:

1. Browser gửi request tới server
2. Web server chuyển request cho PHP xử lý
3. PHP trả kết quả về cho browser

> Với pentester, điều quan trọng là hiểu: mọi dữ liệu đầu vào từ phía client đều phải được đọc từ các biến đặc biệt của PHP.

## Nguồn dữ liệu đầu vào

PHP không tự hỏi browser. Mọi dữ liệu đều được lấy từ các biến đặc biệt gọi là superglobals.

### Các biến thường gặp

- `$_GET`: dữ liệu từ query string, ví dụ `?id=5`
- `$_POST`: dữ liệu từ form hoặc request body
- `$_COOKIE`: dữ liệu từ cookie
- `$_FILES`: dữ liệu file upload
- `$_SERVER`: thông tin request, header, method, đường dẫn...
- `$_SESSION`: dữ liệu session
- `$_REQUEST`: gom dữ liệu từ `GET`, `POST`, `COOKIE`

Ví dụ:

```php
$id = $_GET['id'];
$username = $_POST['username'];
$session_id = $_COOKIE['session'];
```

### Upload file

```html
<form action="/upload" method="POST" enctype="multipart/form-data">
    <input type="file" name="avatar">
    <button>Upload</button>
</form>
```

Khi upload, PHP sẽ lưu thông tin vào `$_FILES`:

```php
$avatar = $_FILES['avatar'];
```

Thông thường sẽ có các key như:

- `name`: tên file do client gửi
- `type`: MIME type
- `tmp_name`: đường dẫn file tạm trên server
- `error`: mã lỗi
- `size`: kích thước file

## Cấu trúc cơ bản của PHP

### Biến, kiểu dữ liệu và phép gán

PHP là ngôn ngữ động, nên biến không cần khai báo kiểu trước. Đây là cách PHP lưu dữ liệu để xử lý và truyền giữa các đoạn code.

```php
$a = 1;
$b = "hello";
$c = true;
```

Ví dụ cơ bản:

```php
$a = 1;
$a = $a + 5; // 1 + 5 = 6
echo $a; // in ra 6
```

Giải thích: biến `$a` được gán giá trị rồi thay đổi, sau đó được in ra màn hình.

### Một số kiểu dữ liệu thường gặp

- `string`: chuỗi
- `int`: số nguyên
- `float`: số thực
- `bool`: đúng/sai
- `array`: mảng
- `null`: không có giá trị

### Câu điều kiện

Câu điều kiện dùng để kiểm tra một điều kiện rồi quyết định chạy đoạn code nào.

```php
if ($age >= 18) {
    echo "Adult";
} else {
    echo "Minor";
}
```

Ví dụ này nghĩa là: nếu tuổi lớn hơn hoặc bằng 18 thì in "Adult", ngược lại in "Minor".

```php
switch ($day) {
    case 1:
        echo "Monday";
        break;
    case 2:
        echo "Tuesday";
        break;
    default:
        echo "Other";
}
```

`switch` thường dùng khi có nhiều trường hợp khác nhau cần xét.

### Vòng lặp

Vòng lặp giúp lặp lại một khối code nhiều lần mà không cần viết lại nhiều lần.

```php
for ($i = 0; $i < 3; $i++) {
    echo $i;
}
```

Ví dụ này sẽ in ra các giá trị 0, 1, 2.

```php
while ($x < 5) {
    echo $x;
    $x++;
}
```

`while` lặp đi lặp lại miễn là điều kiện còn đúng.

### Hàm

Hàm dùng để đóng gói một đoạn logic thành một khối có thể gọi lại nhiều lần.

```php
function add($a, $b) {
    return $a + $b;
}

echo add(2, 3);
```

Ví dụ này định nghĩa hàm `add()` để cộng hai số lại với nhau rồi in kết quả.

### Mảng và chuỗi

Mảng dùng để lưu nhiều giá trị trong một biến; chuỗi dùng để lưu text.

```php
$colors = array("red", "blue", "green");
echo $colors[0];
```

Ví dụ này cho thấy mảng có thể lưu nhiều phần tử và truy cập bằng chỉ số.

```php
$user = [
    'name' => 'Minh',
    'age' => 20
];

echo $user['name'];
```

Đây là mảng dạng key-value, thường dùng để lưu thông tin có cấu trúc rõ ràng.

```php
$name = "Minh";
echo "Hello " . $name;
```

Dấu `.` dùng để nối chuỗi lại với nhau.

### Include / Require

`include` và `require` dùng để chèn nội dung của một file khác vào file hiện tại.

```php
include 'header.php';
require 'config.php';
```

Đây là điểm quan trọng trong pentest, vì nếu input người dùng quyết định file cần include thì có thể gây Local File Inclusion.

---

## Các hàm xử lý tệp, chuỗi và ảnh thông dụng

### Các hàm xử lý chuỗi (String Functions)
*   **substr()**: Trích xuất một phần của chuỗi dựa vào vị trí bắt đầu và độ dài. Hàm này thường được dùng để lấy đuôi mở rộng của tệp tin tải lên.
*   **strrpos()**: Tìm vị trí xuất hiện cuối cùng của một ký tự trong chuỗi (ví dụ tìm dấu chấm cuối cùng của tên tệp để xác định vị trí phần mở rộng).
*   **strtolower()**: Chuyển đổi toàn bộ chuỗi ký tự thành chữ thường. Giúp đồng bộ hóa phần mở rộng trước khi kiểm tra (tránh việc bypass bằng cách viết hoa như `.pHp` hay `.PnG`).
*   **trim()**: Loại bỏ các khoảng trắng và ký tự đặc biệt ở đầu và cuối chuỗi.
*   **bin2hex()**: Chuyển đổi dữ liệu nhị phân sang dạng chuỗi thập lục phân (hexadecimal string). Thường dùng để sinh chuỗi ngẫu nhiên không chứa ký tự đặc biệt.
*   **random_bytes()**: Tạo ra các byte nhị phân ngẫu nhiên an toàn về mặt mật mã để sinh tên tệp mới.

### Các hàm xử lý tệp tin và đường dẫn (File & Path Functions)
*   **basename()**: Lấy ra tên tệp tin từ một đường dẫn đầy đủ (ví dụ `/var/www/uploads/shell.php` $\rightarrow$ `shell.php`). Giúp loại bỏ lỗi Path Traversal khi kẻ tấn công chèn `../`.
*   **move_uploaded_file()**: Di chuyển tệp tin tải lên từ thư mục tạm thời của hệ thống sang thư mục lưu trữ chính thức. Hàm này tự động kiểm tra xem tệp tin có thực sự được tải lên từ client hay không để đảm bảo an toàn.
*   **is_uploaded_file()**: Kiểm tra xem một tệp tin có thực sự được tải lên bằng giao thức HTTP POST hay không. Hàm này giúp chặn lỗi giả mạo đường dẫn tệp cục bộ trên máy chủ.
*   **pathinfo()**: Phân tích đường dẫn tệp tin và trả về thông tin chi tiết (như thư mục, tên tệp tin, phần mở rộng). Đây là giải pháp an toàn hơn so với việc cắt chuỗi thủ công để lấy phần mở rộng tệp.
*   **file_get_contents()**: Đọc toàn bộ nội dung của một tệp tin thành một chuỗi văn bản. Hàm này có nguy cơ gây lỗi Local/Remote File Inclusion (LFI/RFI) nếu kẻ tấn công kiểm soát được tham số đường dẫn tệp tin.
*   **file_put_contents()**: Ghi một chuỗi dữ liệu vào tệp tin. Nếu tên tệp tin và nội dung tệp tin do người dùng kiểm soát, nó sẽ cho phép kẻ tấn công ghi đè hoặc tạo các tệp kịch bản động nguy hiểm trên máy chủ.
*   **rename()**: Đổi tên hoặc di chuyển tệp tin. Khác với `move_uploaded_file`, hàm này hoạt động trên bất kỳ tệp tin nào trong hệ thống.
*   **unlink()**: Xóa một tệp tin vật lý khỏi đĩa cứng. Thường dùng để dọn dẹp các tệp tạm thời hoặc tệp bị kiểm tra lỗi.
*   **file_exists()**: Kiểm tra xem một tệp tin hoặc thư mục có tồn tại trên máy chủ hay không.
*   **getcwd()**: Trả về đường dẫn của thư mục làm việc hiện tại của ứng dụng.
*   **chmod()**: Thay đổi quyền hạn (phân quyền đọc/ghi/thực thi) của một tệp tin hoặc thư mục trên hệ điều hành của máy chủ.
*   **ini_get()**: Đọc giá trị cấu hình hệ thống từ tệp tin `php.ini` (ví dụ lấy thư mục tạm thời `upload_tmp_dir`).
*   **sys_get_temp_dir()**: Trả về đường dẫn thư mục tạm mặc định của hệ điều hành.

### Các hàm kiểm tra và xử lý hình ảnh (Image Functions)
*   **getimagesize()**: Lấy kích thước và định dạng thực tế của ảnh. Nếu tệp tin truyền vào không phải là ảnh hợp lệ, hàm sẽ trả về giá trị `false`. Attacker có thể chèn mã PHP vào phần comment của ảnh (Polyglot) để vượt qua hàm này.
*   **imagecreatefromjpeg() / **imagecreatefrompng()**: Khởi tạo một đối tượng hình ảnh mới từ tệp tin ảnh JPEG hoặc PNG có sẵn. Cơ chế này vẽ lại ảnh, giúp loại bỏ toàn bộ dữ liệu phụ (metadata) chứa mã độc.
*   **imagejpeg()** / **imagepng()**: Xuất dữ liệu hình ảnh từ đối tượng ảnh ra tệp tin vật lý hoặc màn hình.
*   **imagedestroy()**: Giải phóng bộ nhớ RAM hệ thống sau khi xử lý xong đối tượng hình ảnh.

---

## Xử lý input và debug

### Kiểm tra input

```php
if (isset($_POST['username'])) {
    echo "Đã nhận username";
}

if (!empty($_GET['id'])) {
    echo "ID không rỗng";
}

$name = trim($_POST['name']);
```

Ý nghĩa:

- `isset()`: kiểm tra biến có tồn tại và khác `null`
- `empty()`: kiểm tra biến có rỗng hay không
- `trim()`: loại bỏ khoảng trắng ở đầu và cuối chuỗi

### Debug bằng biến

```php
var_dump($_POST);
print_r($_FILES);
```

Các hàm này giúp xem nội dung biến một cách trực tiếp, rất hữu ích khi phân tích code PHP lỗi hoặc lab.

## Những điểm cần chú ý cho pentester

### Luôn coi dữ liệu từ client là không đáng tin

```php
$input = $_GET['id'];
echo $input;
```

Dữ liệu từ `$_GET`, `$_POST`, `$_COOKIE`, `$_FILES` đều có thể bị attacker kiểm soát.

### Hàm nguy hiểm

Một số hàm thường bị khai thác nếu dùng sai:

- `eval()`
- `system()`
- `exec()`
- `shell_exec()`
- `passthru()`
- `include` / `require`
- `file_get_contents()`

### Upload file cần kiểm tra kỹ

```php
$tmp = $_FILES['avatar']['tmp_name'];
$target = '/var/www/uploads/' . basename($_FILES['avatar']['name']);
move_uploaded_file($tmp, $target);
```

Nếu không kiểm tra extension, MIME type, path hoặc sanitize tên file, upload có thể bị bypass.

### Cách phân biệt code safe và vulnerable

Code có nguy cơ nếu:

- tin tưởng trực tiếp dữ liệu từ `$_GET`, `$_POST`, `$_FILES`
- không kiểm tra extension, MIME type, path
- dùng `include`, `eval`, `system` với input người dùng
- không sanitize đầu vào trước khi dùng trong SQL, HTML hoặc file system

Code safer thì thường:

- kiểm tra input trước khi dùng
- dùng whitelist thay vì blacklist
- escape hoặc bind parameter trước khi query database
- không cho user trực tiếp quyết định file hoặc path

## Cách đọc code PHP như pentester

Khi phân tích một đoạn PHP, hãy luôn hỏi 4 câu sau:

1. Input đến từ đâu?Ví dụ: `$_GET`, `$_POST`, `$_COOKIE`, `$_FILES`, `$_SERVER`
2. Có kiểm tra gì trước khi dùng không?Ví dụ: `isset()`, `empty()`, `trim()`, `basename()`, `getimagesize()`
3. Có dùng hàm nguy hiểm không?Ví dụ: `eval()`, `system()`, `include`, `file_get_contents()`
4. Kết quả cuối cùng được dùng để làm gì?
   Ví dụ: lưu file, hiển thị ra trang, truy vấn database, redirect, include file

### Ví dụ nâng cao

```php
$file = $_GET['file'];
include($file);
```

Nếu `file` được kiểm soát bởi người dùng, đây có thể trở thành Local File Inclusion.

## Luyện tập phân tích đoạn code mẫu PHP

> Phần này để bạn tự làm sau khi đã đọc xong nội dung trên. Mục tiêu là luyện cách nhìn ra input, điều kiện, hàm và logic xử lý.

### Bài tập 1

```php
<?php

$change = false;
$request_type = "html";
$return_message = "Request Failed";

if ($_SERVER['REQUEST_METHOD'] == "POST" && array_key_exists("CONTENT_TYPE", $_SERVER) && $_SERVER['CONTENT_TYPE'] == "application/json") {
    $data = json_decode(file_get_contents('php://input'), true);
    $request_type = "json";
    if (array_key_exists("HTTP_USER_TOKEN", $_SERVER) &&
        array_key_exists("password_new", $data) &&
        array_key_exists("password_conf", $data) &&
        array_key_exists("Change", $data)) {
        $token = $_SERVER['HTTP_USER_TOKEN'];
        $pass_new = $data["password_new"];
        $pass_conf = $data["password_conf"];
        $change = true;
    }
} else {
    if (array_key_exists("user_token", $_REQUEST) &&
        array_key_exists("password_new", $_REQUEST) &&
        array_key_exists("password_conf", $_REQUEST) &&
        array_key_exists("Change", $_REQUEST)) {
        $token = $_REQUEST["user_token"];
        $pass_new = $_REQUEST["password_new"];
        $pass_conf = $_REQUEST["password_conf"];
        $change = true;
    }
}

if ($change) {
    checkToken($token, $_SESSION['session_token'], 'index.php');

    if ($pass_new == $pass_conf) {
        $pass_new = mysqli_real_escape_string($GLOBALS["___mysqli_ston"], $pass_new);
        $pass_new = md5($pass_new);

        $current_user = dvwaCurrentUser();
        $insert = "UPDATE `users` SET password = '" . $pass_new . "' WHERE user = '" . $current_user . "';";
        $result = mysqli_query($GLOBALS["___mysqli_ston"], $insert);

        $return_message = "Password Changed.";
    } else {
        $return_message = "Passwords did not match.";
    }

    mysqli_close($GLOBALS["___mysqli_ston"]);

    if ($request_type == "json") {
        generateSessionToken();
        header("Content-Type: application/json");
        print json_encode(array("Message" => $return_message));
        exit;
    } else {
        $html .= "<pre>" . $return_message . "</pre>";
    }
}

generateSessionToken();
?>
```

### Bài tập 2

```php
<?php

if (isset($_POST['Upload'])) {
    $target_path = DVWA_WEB_PAGE_TO_ROOT . "hackable/uploads/";
    $target_path .= basename($_FILES['uploaded']['name']);

    $uploaded_name = $_FILES['uploaded']['name'];
    $uploaded_ext = substr($uploaded_name, strrpos($uploaded_name, '.') + 1);
    $uploaded_size = $_FILES['uploaded']['size'];
    $uploaded_tmp = $_FILES['uploaded']['tmp_name'];

    if ((strtolower($uploaded_ext) == "jpg" || strtolower($uploaded_ext) == "jpeg" || strtolower($uploaded_ext) == "png") &&
        ($uploaded_size < 100000) &&
        getimagesize($uploaded_tmp)) {

        if (!move_uploaded_file($uploaded_tmp, $target_path)) {
            $html .= '<pre>Your image was not uploaded.</pre>';
        } else {
            $html .= "<pre>{$target_path} succesfully uploaded!</pre>";
        }
    } else {
        $html .= '<pre>Your image was not uploaded. We can only accept JPEG or PNG images.</pre>';
    }
}
?>
```

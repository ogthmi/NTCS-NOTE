# Docker Core, Dockerfile và Docker Compose

> Tại sao lại cần Docker? Docker giải quyết vấn đề gì và hoạt động như thế nào?

## Bố cục nội dung

```
Docker Core & Compose
│
├── Giới thiệu về Docker Core (Khái niệm cơ bản)
│   ├── Vấn đề "Nó chạy trên máy tôi"
│   ├── Khái niệm Máy ảo (Virtual Machine) vs Container
│   └── Khái niệm Image (Hình ảnh) vs Container (Thực thể)
│
├── Kiến trúc Docker và Các cơ chế lõi
│   ├── Docker Daemon, Client và Registry
│   ├── Linux Namespaces (Cô lập tài nguyên)
│   └── Linux Control Groups - cgroups (Giới hạn tài nguyên)
│
├── Dockerfile (Xây dựng hình ảnh container)
│   ├── Định nghĩa và Cú pháp cơ bản
│   ├── Cơ chế Docker Layers và Layer Caching (Bộ nhớ đệm)
│   └── Ví dụ Dockerfile cho ứng dụng PHP
│
├── Docker Volumes (Lưu trữ dữ liệu)
│   ├── Bản chất lưu trữ tạm thời của container
│   ├── Phân biệt Bind Mounts và Volumes
│   └── Sơ đồ cấu trúc lưu trữ dữ liệu
│
├── Mạng trong Docker (Docker Networking)
│   ├── Các chế độ Bridge, Host, None
│   └── Cơ chế Port Mapping (Ánh xạ cổng)
│
├── Docker Compose (Quản lý đa container)
│   ├── Định nghĩa và Sự cần thiết
│   └── Ví dụ docker-compose.yml thực tế (PHP + MySQL)
│
├── Các lệnh Docker thường dùng (Docker CLI Commands)
│   ├── Nhóm lệnh quản lý Container (run, ps, stop, rm, exec, logs)
│   ├── Nhóm lệnh quản lý Image (build, images, rmi)
│   ├── Nhóm lệnh kiểm tra tài nguyên và gỡ lỗi (inspect, volume, network)
│   └── Nhóm lệnh Docker Compose (up, down, ps, logs)
│
└── Lưu ý bảo mật Docker (Docker Security)
    ├── Rủi ro chạy quyền Root mặc định
    └── Lỗ hổng gắn kết Docker Socket (/var/run/docker.sock)
```

## Giới thiệu về Docker Core

### Tài liệu tham khảo:
* [Docker Overview](https://docs.docker.com/get-started/overview/)
* [Docker Glossary: Image and Container](https://docs.docker.com/glossary/)

Trong quy trình phát triển phần mềm truyền thống, lập trình viên thường xuyên gặp phải tình trạng ứng dụng hoạt động hoàn hảo trên máy tính cá nhân nhưng lại gặp lỗi nghiêm trọng khi triển khai lên máy chủ. Sự khác biệt về phiên bản hệ điều hành, cấu hình môi trường hoặc các thư viện dùng chung là nguyên nhân chính dẫn đến sự cố này.

Docker được tạo ra để giải quyết vấn đề trên bằng cách đóng gói toàn bộ ứng dụng cùng môi trường hoạt động của nó vào một gói duy nhất gọn nhẹ gọi là container.

### Khái niệm Máy ảo (Virtual Machine) vs Container
Máy ảo (Virtual Machine - VM) hoạt động bằng cách sử dụng một lớp ảo hóa phần cứng (gọi là Hypervisor) để chạy toàn bộ hệ điều hành khách (Guest OS) độc lập trên nền hệ điều hành chủ (Host OS). Cơ chế này đòi hỏi tài nguyên phần cứng lớn, chiếm nhiều dung lượng ổ cứng và mất nhiều thời gian để khởi động toàn bộ hệ điều hành.

Container không chạy hệ điều hành riêng biệt mà sử dụng chung nhân hệ điều hành của máy chủ (Host Kernel) và chỉ cô lập các tiến trình ứng dụng. Nhờ đó, container có dung lượng cực kỳ nhỏ nhẹ, khởi động tức thì và không làm hao phí tài nguyên phần cứng của máy chủ.

Sơ đồ so sánh kiến trúc giữa máy ảo và container:
```
[Virtual Machine - Máy ảo]          [Docker Container]
┌─────────────────────────────────┐ ┌─────────────────────────────────┐
│ App 1   │ App 2   │ App 3       │ │ App 1   │ App 2   │ App 3       │
├─────────┼─────────┼─────────────┤ ├─────────┼─────────┼─────────────┤
│ Guest OS│ Guest OS│ Guest OS    │ │   Bin   │   Bin   │   Bin       │
├─────────┴─────────┴─────────────┤ ├─────────┴─────────┴─────────────┤
│            Hypervisor           │ │          Docker Engine          │
├─────────────────────────────────┤ ├─────────────────────────────────┤
│             Host OS             │ │             Host OS             │
└─────────────────────────────────┘ └─────────────────────────────────┘
```

### Khái niệm Image (Hình ảnh) vs Container (Thực thể)
Image là một bản thiết kế chỉ đọc (Read-only template) chứa mã nguồn, thư viện cài đặt và các cấu hình môi trường cần thiết để ứng dụng hoạt động. Hãy tưởng tượng Image giống như một bản vẽ kỹ thuật trên giấy hoặc một khuôn đúc bánh tĩnh lặng, không thể tự chạy được.

Container là một thực thể hoạt động được khởi tạo từ Image, giống như ngôi nhà thật được xây dựng từ bản vẽ hoặc chiếc bánh thật được đúc ra từ khuôn. Mỗi container hoạt động độc lập và chứa thêm một lớp ghi dữ liệu tạm thời (Writable layer) xếp chồng lên trên các lớp chỉ đọc của Image để lưu trữ dữ liệu phát sinh khi chạy.

## Kiến trúc Docker và Các cơ chế lõi

### Tài liệu tham khảo:
* [Docker Architecture](https://docs.docker.com/engine/architecture/)
* [Linux Namespaces: Concepts](https://man7.org/linux/man-pages/man7/namespaces.7.html)
* [Linux Control Groups: cgroups](https://man7.org/linux/man-pages/man7/cgroups.7.html)

Docker hoạt động theo mô hình máy khách - máy chủ (Client - Server) và sử dụng hai tính năng bảo mật tích hợp sâu trong nhân hệ điều hành Linux để cô lập container.

### Docker Daemon, Client và Registry
* **Docker Client**: Là công cụ giao diện dòng lệnh (CLI) cho phép người dùng nhập lệnh để tương tác với Docker (ví dụ: `docker run`).
* **Docker Daemon (dockerd)**: Hoạt động như một người quản lý chạy ngầm trên máy chủ, tiếp nhận lệnh từ client để thực hiện tạo, chạy và quản lý các container, image, network và volume.
* **Docker Registry**: Là một thư viện lưu trữ dùng để chia sẻ các Docker Image, trong đó Docker Hub là kho lưu trữ trực tuyến mặc định lớn nhất.

### Linux Namespaces (Cô lập tài nguyên)
Nhân hệ điều hành Linux sử dụng công nghệ Namespaces để tạo ra các phòng cách ly riêng biệt cho từng container, khiến container tin rằng nó đang là hệ điều hành duy nhất trên máy.
* **PID Namespace**: Cách ly mã định danh tiến trình (Process ID), khiến tiến trình chính bên trong container được gán số PID là 1 mà không ảnh hưởng đến hệ thống bên ngoài.
* **NET Namespace**: Cách ly cấu hình mạng, cho phép mỗi container sở hữu địa chỉ IP riêng biệt và bảng định tuyến độc lập.
* **MNT Namespace**: Cách ly các điểm gắn kết hệ thống tệp (Mount points), giúp container chỉ nhìn thấy các thư mục được cấp phép hoạt động.
* **UTS Namespace**: Cách ly tên máy chủ (Hostname) và tên miền (Domain name) của container.
* **IPC Namespace**: Ngăn chặn các tiến trình của các container khác nhau truyền thông điệp trực tiếp qua bộ nhớ dùng chung để đảm bảo an toàn.
* **USER Namespace**: Cô lập người dùng, cho phép tài khoản quản trị root bên trong container chỉ tương đương với một tài khoản người dùng bình thường ngoài máy chủ.

### Linux Control Groups - cgroups (Giới hạn tài nguyên)
Trong khi Namespaces thực hiện nhiệm vụ cô lập môi trường xung quanh, thì Control Groups (cgroups) chịu trách nhiệm giới hạn hạn mức tài nguyên phần cứng.
Cgroups hoạt động giống như một thiết bị giới hạn lượng điện nước tiêu thụ, ngăn chặn tình trạng một container bị lỗi chiếm dụng toàn bộ tài nguyên CPU, dung lượng RAM hoặc băng thông ổ đĩa (I/O) của máy chủ.

## Dockerfile (Xây dựng hình ảnh container)

### Tài liệu tham khảo:
* [Dockerfile Reference Guide](https://docs.docker.com/engine/reference/builder/)
* [Best practices for writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

Dockerfile là một tệp văn bản đơn giản chứa tập hợp tuần tự các chỉ dẫn dòng lệnh giúp Docker tự động biên dịch và tạo ra một Image mới.

### Các chỉ thị cơ bản trong Dockerfile
* **`FROM`**: Định nghĩa hình ảnh nền (Base Image) để bắt đầu xây dựng (ví dụ: `FROM php:8.1-apache`).
* **`RUN`**: Thực thi các lệnh cài đặt gói phần mềm trong quá trình xây dựng Image (ví dụ: `RUN apt-get update && apt-get install -y git`).
* **`COPY` / `ADD`**: Sao chép các tệp tin từ máy chủ vật lý vào bên trong cấu trúc thư mục của Image.
* **`WORKDIR`**: Thiết lập thư mục làm việc mặc định cho các chỉ thị tiếp theo hoạt động bên trong container.
* **`ENV`**: Thiết lập các biến môi trường cấu hình cho ứng dụng.
* **`EXPOSE`**: Khai báo cổng mạng mà container sẽ lắng nghe khi chạy.
* **`CMD` / `ENTRYPOINT`**: Định nghĩa lệnh mặc định sẽ được tự động thực thi khi container được khởi tạo.

### Cơ chế Docker Layers và Layer Caching (Bộ nhớ đệm)
Mỗi dòng chỉ thị trong Dockerfile sẽ tạo ra một lớp dữ liệu chỉ đọc (Read-only layer) mới xếp chồng lên lớp dữ liệu trước đó.
Khi tiến hành biên dịch lại Image, Docker sẽ kiểm tra và tái sử dụng các lớp dữ liệu cũ từ bộ nhớ đệm (Layer Caching) nếu nội dung chỉ thị và các tệp tin liên quan không có bất kỳ thay đổi nào để tiết kiệm thời gian.

Sơ đồ cấu trúc lớp (UnionFS) trong Docker Image và Container:
```
┌──────────────────────────────────────┐
│ Writable Layer (Lớp ghi của Container)│ ◄── Cho phép ghi dữ liệu tạm thời khi chạy
├──────────────────────────────────────┤
│ Read-only Layer: COPY . /var/www/    │ ◄── Lớp chỉ đọc tạo từ lệnh COPY
├──────────────────────────────────────┤
│ Read-only Layer: RUN apt-get install │ ◄── Lớp chỉ đọc tạo từ lệnh RUN
├──────────────────────────────────────┤
│ Base Image: php:8.1-apache           │ ◄── Lớp nền ban đầu
└──────────────────────────────────────┘
```

### Ví dụ Dockerfile cho ứng dụng PHP
Dưới đây là một tệp `Dockerfile` mẫu để dựng máy chủ Apache chạy PHP:
```dockerfile
# Sử dụng base image PHP 8.1 kèm máy chủ Apache có sẵn
FROM php:8.1-apache

# Thiết lập thư mục làm việc mặc định trong container
WORKDIR /var/www/html

# Cài đặt các thư viện cần thiết cho PHP
RUN apt-get update && apt-get install -y \
    libpng-dev \
    && docker-php-ext-install pdo_mysql gd

# Sao chép toàn bộ mã nguồn từ thư mục src ngoài máy vào container
COPY src/ /var/www/html/

# Phân quyền cho máy chủ Apache có quyền đọc ghi dữ liệu trên thư mục
RUN chown -R www-data:www-data /var/www/html

# Khai báo cổng mạng hoạt động mặc định
EXPOSE 80
```

## Docker Volumes (Lưu trữ dữ liệu)

### Tài liệu tham khảo:
* [Docker Storage Overview](https://docs.docker.com/storage/)
* [Volumes Guide](https://docs.docker.com/storage/volumes/)

Hệ thống tệp tin mặc định bên trong container có tính chất tạm thời (Ephemeral). Mọi dữ liệu phát sinh hoặc thay đổi trong quá trình container hoạt động sẽ lập tức bị xóa sạch hoàn toàn khi container bị xóa bỏ.

### Phân biệt Bind Mounts và Volumes
Để lưu trữ dữ liệu bền vững qua nhiều thế hệ container, Docker cung cấp hai cơ chế gắn kết bộ nhớ chính:
* **Bind Mounts**: Gắn kết trực tiếp một đường dẫn thư mục cụ thể ngoài máy chủ vật lý vào trong container. Cơ chế này rất phù hợp trong môi trường phát triển ứng dụng (Development) vì lập trình viên có thể chỉnh sửa mã nguồn ở máy ngoài và lập tức thấy kết quả cập nhật bên trong container.
* **Volumes**: Được Docker tự động tạo ra và quản lý hoàn toàn bên trong một thư mục đặc biệt trên đĩa cứng máy chủ. Volumes có hiệu suất truy xuất dữ liệu cao hơn, an toàn hơn và dễ dàng sao lưu hơn so với Bind Mounts.

Sơ đồ vị trí lưu trữ dữ liệu của Volumes và Bind Mounts trên máy chủ:
```
      ┌──────────────────────────────────────────────┐
      │               Máy chủ Vật lý                 │
      │                                              │
      │   ┌──────────────────────────────────────┐   │
      │   │           Thư mục của Docker         │   │
      │   │  (C:/ProgramData/docker/volumes/)    │   │
      │   │                  │                   │   │
      │   │                  ▼                   │   │
      │   │            Docker Volume             │   │
      │   └──────────────────┬───────────────────┘   │
      │                      │                       │
      │   ┌──────────────────┼───────────────────┐   │
      │   │                  ▼                   │   │
      │   │          Thư mục trong Container     │   │
      │   │         (/var/www/html/uploads)      │   │
      │   └──────────────────▲───────────────────┘   │
      │                      │                       │
      │   ┌──────────────────┴───────────────────┐   │
      │   │           Thư mục trên Máy chủ       │   │
      │   │            (D:/my_project/src)       │   │
      │   │            (Cơ chế Bind Mount)       │   │
      │   └──────────────────────────────────────┘
      └──────────────────────────────────────────────┘
```

## Mạng trong Docker (Docker Networking)

### Tài liệu tham khảo:
* [Docker Networking Drivers](https://docs.docker.com/network/)
* [Container Network Interface (CNI)](https://docs.docker.com/network/bridge/)

Docker cung cấp các trình điều khiển mạng (Network Drivers) tích hợp sẵn để quản lý giao tiếp giữa các container với nhau và với thế giới Internet bên ngoài.

### Các chế độ mạng phổ biến
* **Bridge**: Là chế độ mạng mặc định. Docker tạo ra một cầu nối mạng ảo (ví dụ `docker0`), mỗi container tham gia mạng này sẽ được cấp một dải IP nội bộ riêng để tự động giao tiếp với nhau thông qua tên container.
* **Host**: Container sử dụng chung giao diện mạng của máy chủ vật lý mà không có sự cô lập nào. Nếu container chạy ứng dụng ở cổng 80, ứng dụng đó sẽ trực tiếp mở cổng 80 trên máy chủ vật lý bên ngoài.
* **None**: Vô hiệu hóa hoàn toàn mạng của container, khiến container không thể kết nối Internet hoặc giao tiếp với bất kỳ container nào khác để đảm bảo an toàn tuyệt đối.

### Cơ chế Port Mapping (Ánh xạ cổng)
Mặc dù container hoạt động độc lập trên mạng Bridge nội bộ, người dùng từ máy tính bên ngoài không thể truy cập trực tiếp các cổng nội bộ này.
Cơ chế ánh xạ cổng (Port Mapping) sử dụng tùy chọn `-p <host_port>:<container_port>` để mở đường dẫn truyền dữ liệu từ cổng máy chủ vật lý vào cổng tương ứng của container.

## Docker Compose (Quản lý đa container)

### Tài liệu tham khảo:
* [Docker Compose Overview](https://docs.docker.com/compose/)
* [Compose file specification](https://docs.docker.com/compose/compose-file/)

Một ứng dụng web thực tế thường bao gồm nhiều thành phần hoạt động cùng nhau (ví dụ: Web Server, Database, Cache Server).
Việc quản lý và khởi chạy thủ công từng container bằng các lệnh `docker run` đơn lẻ rất phức tạp và dễ gây ra sai sót cấu hình mạng.

Docker Compose là công cụ giúp định nghĩa và vận hành hệ thống đa container bằng cách mô tả toàn bộ kiến trúc trong một tệp tin cấu hình YAML duy nhất có tên là `docker-compose.yml`.

### Ví dụ docker-compose.yml thực tế (PHP + MySQL)
Dưới đây là tệp tin cấu hình dựng hệ thống Web PHP kết nối tới cơ sở dữ liệu MySQL:
```yaml
version: '3.8'

services:
  # Cấu hình dịch vụ Web Server chạy PHP
  web:
    build: .
    ports:
      - "8080:80"
    volumes:
      - ./src:/var/www/html
    depends_on:
      - db
    networks:
      - app-network

  # Cấu hình dịch vụ cơ sở dữ liệu MySQL
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: app_db
    volumes:
      - db-data:/var/lib/mysql
    networks:
      - app-network

volumes:
  db-data:

networks:
  app-network:
    driver: bridge
```

## Các lệnh Docker thường dùng (Docker CLI Commands)

### Tài liệu tham khảo:
* [Docker Command Line Reference](https://docs.docker.com/engine/reference/commandline/cli/)
* [Docker Compose CLI Reference](https://docs.docker.com/compose/reference/)

Việc sử dụng thành thạo các câu lệnh CLI giúp quản lý container và khắc phục sự cố hệ thống nhanh chóng hơn.

### Nhóm lệnh quản lý Container
* **`docker run`**: Khởi tạo và chạy một container mới từ một Image có sẵn.
  * Ví dụ: `docker run -d -p 80:80 --name my_web php:8.1-apache` (Chạy ngầm với tham số `-d`, đặt tên bằng `--name` và ánh xạ cổng bằng `-p`).
* **`docker ps`**: Liệt kê các container đang hoạt động trên hệ thống.
  * Sử dụng tham số `docker ps -a` để liệt kê toàn bộ các container bao gồm cả các container đã dừng hoạt động.
* **`docker stop`**: Dừng một container đang chạy bằng cách gửi tín hiệu SIGTERM.
  * Ví dụ: `docker stop my_web`.
* **`docker rm`**: Xóa một container đã dừng hoạt động khỏi đĩa cứng máy chủ.
  * Ví dụ: `docker rm my_web` (Thêm tham số `-f` để ép buộc xóa container đang chạy).
* **`docker exec`**: Thực thi một câu lệnh mới bên trong một container đang hoạt động.
  * Ví dụ: `docker exec -it my_web /bin/bash` (Tham số `-it` mở cửa sổ tương tác dòng lệnh).
* **`docker logs`**: Trích xuất toàn bộ nhật ký ghi nhận lỗi (Logs) đầu ra của container.
  * Ví dụ: `docker logs -f my_web` (Tham số `-f` để theo dõi nhật ký thời gian thực).

### Nhóm lệnh quản lý Image
* **`docker build`**: Biên dịch và tạo ra một Image mới từ các chỉ dẫn trong tệp Dockerfile.
  * Ví dụ: `docker build -t my_custom_php:1.0 .` (Tham số `-t` để đặt tên nhãn và dấu chấm `.` chỉ đường dẫn thư mục hiện tại).
* **`docker images`**: Liệt kê danh sách tất cả các Image đang được lưu trữ cục bộ trên máy chủ.
* **`docker rmi`**: Xóa một Image cục bộ khỏi máy chủ vật lý để giải phóng bộ nhớ.
  * Ví dụ: `docker rmi my_custom_php:1.0`.

### Nhóm lệnh kiểm tra tài nguyên và gỡ lỗi
* **`docker inspect`**: Hiển thị toàn bộ thông tin cấu hình chi tiết dưới dạng JSON của container hoặc image.
  * Ví dụ: `docker inspect my_web` (Hữu ích để tìm địa chỉ IP nội bộ của container).
* **`docker volume ls`**: Liệt kê các Volume đang được Docker quản lý.
* **`docker network ls`**: Liệt kê danh sách các cấu hình mạng đang hoạt động.

### Nhóm lệnh Docker Compose
* **`docker-compose up`**: Khởi động toàn bộ các dịch vụ được mô tả trong tệp cấu hình `docker-compose.yml`.
  * Ví dụ: `docker-compose up -d` (Khởi động và chạy ngầm tất cả các container).
* **`docker-compose down`**: Dừng và xóa sạch các container, mạng nội bộ và tài nguyên được tạo ra bởi lệnh `up`.
* **`docker-compose ps`**: Kiểm tra trạng thái hoạt động của các container thuộc dự án hiện tại.
* **`docker-compose logs`**: Xem nhật ký hoạt động tổng hợp của tất cả các container trong cấu hình.

## Lưu ý bảo mật Docker (Docker Security)

### Tài liệu tham khảo:
* [Docker Security Practices](https://docs.docker.com/engine/security/)
* [OWASP Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)

Việc cấu hình sai các đặc quyền của Docker có thể tạo điều kiện cho kẻ tấn công thực hiện kỹ thuật vượt rào container (Container Escape) để chiếm quyền điều khiển toàn bộ máy chủ vật lý.

### Rủi ro chạy quyền Root mặc định
* Theo mặc định, nếu không được cấu hình cụ thể trong Dockerfile, các tiến trình bên trong container sẽ chạy dưới quyền người dùng root.
* Nếu ứng dụng web bị tấn công chiếm quyền điều khiển (như lỗi RCE), kẻ tấn công sẽ lập tức có quyền root bên trong container đó.
* Quản trị viên nên khai báo một người dùng không có đặc quyền (Non-root user) ở cuối Dockerfile bằng chỉ thị `USER` để giảm thiểu thiệt hại khi xảy ra sự cố bảo mật.

### Lỗ hổng gắn kết Docker Socket (/var/run/docker.sock)
* Đường dẫn `/var/run/docker.sock` là cổng giao tiếp Unix socket chính của Docker Daemon, cho phép tương tác trực tiếp với nhân điều khiển hệ thống Docker.
* Nếu quản trị viên cấu hình gắn kết (mount) tệp tin này vào trong một container thông thường để quản trị, kẻ tấn công có quyền kiểm soát container đó sẽ có khả năng gửi lệnh trực tiếp tới Docker Daemon trên máy chủ vật lý.
* Từ đó, kẻ tấn công có thể khởi tạo một container mới có đặc quyền cao (`--privileged`) chứa toàn bộ đĩa cứng của máy chủ vật lý để chiếm quyền root của hệ thống máy chủ vật lý bên ngoài.

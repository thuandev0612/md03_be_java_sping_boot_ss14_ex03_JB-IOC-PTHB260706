# Session14 - Bài 1 đến 3: Đăng nhập JWT, bảo vệ API và JSON lỗi

Kế thừa Session13: User, UserRepository, UserPrincipal, UserDetailServiceCustom và BCrypt. Java 21, Spring Boot và PostgreSQL.

## Chạy dự án

1. Mở project bằng IntelliJ, chọn JDK 21 và tải lại Gradle.
2. Chạy `BaitapSession14Application`.
3. Máy hiện tại đã có database `security_session14` và cấu hình riêng tại `../.local-config/baitapSession14.properties`.

Máy khác tạo database bằng `CREATE DATABASE security_session14;`, rồi tạo `config/application-local.properties` (file này được bỏ qua bởi Git):

```properties
DB_USERNAME=postgres
DB_PASSWORD=mat_khau_database_cua_may
JWT_SECRET=thay_bang_chuoi_base64_tao_ben_duoi
```

Tạo JWT_SECRET bằng PowerShell, dán kết quả vào file riêng:

```powershell
$jwtBytes = New-Object byte[] 32
$rng = [System.Security.Cryptography.RandomNumberGenerator]::Create()
$rng.GetBytes($jwtBytes)
$rng.Dispose()
[Convert]::ToBase64String($jwtBytes)
```

`application.properties` đọc secret qua `${JWT_SECRET}`; `jwt.expiration-ms=86400000` là thời hạn 1 ngày. Có thể đặt DB_URL để thay địa chỉ database.

## Thử đăng nhập bằng Postman

```text
POST http://localhost:8080/api/v1/auth/login
Authorization: No Auth
Content-Type: application/json
```

```json
{"username": "admin", "password": "Admin@123456"}
```

Tài khoản trên chỉ dùng cho database bài tập. Máy hiện tại đã được chuẩn bị tài khoản này. Máy khác chạy ứng dụng một lần để tạo bảng users, sau đó chạy SQL bên dưới trong database `security_session14` để tạo tài khoản thử với mật khẩu BCrypt:

```sql
-- Username: admin; mật khẩu thử: Admin@123456
INSERT INTO users (username, password, role, enabled)
SELECT 'admin', '$2a$10$.BGmcS4ugTZdUBpDJi90eOPifUGgAdz.0N5CM9MldS9I3i8Qrkl1q', 'ADMIN', true
WHERE NOT EXISTS (SELECT 1 FROM users WHERE username = 'admin');
```

Thành công trả HTTP 200 với đúng 3 trường:

```json
{"accessToken": "header.payload.signature", "type": "Bearer", "username": "admin"}
```

Token thực tế có 3 phần ngăn bằng dấu chấm. Decode token thử trên jwt.io sẽ thấy `sub=admin`, `role=ADMIN`, `iat` và `exp`; `exp - iat = 86400` giây. Không cần đưa secret lên trang đó để đọc payload.

- Sai mật khẩu, user không tồn tại hoặc bị vô hiệu hóa: HTTP 401.
- Thiếu/rỗng username hoặc password, JSON sai: HTTP 400.
- API public cũ: GET/POST `/api/v1/auth/test`.

## Luồng xử lý

AuthController → AuthenticationManager → UserDetailServiceCustom → UserRepository → BCrypt kiểm tra mật khẩu → JwtProvider ký HS256 → LoginResponse.

Token chứa username, role, thời điểm tạo và hết hạn; không chứa mật khẩu. Giữ đúng dependency `io.jsonwebtoken:jjwt:0.9.1` theo đề và thêm JAXB để chạy với Java 21. Tham khảo [tài liệu JJWT 0.9.1](https://github.com/jwtk/jjwt/blob/0.9.1/README.md).

## Bài 2: Gọi API nhân viên bằng token

Sau khi đăng nhập ở Bài 1, copy giá trị `accessToken`. Trong Postman tạo request:

```text
GET http://localhost:8080/api/v1/employees
Authorization: Bearer <accessToken_vua_nhan>
```

Có thể chọn tab Authorization → Type **Bearer Token**, rồi dán token vào ô Token (không thêm chữ Bearer trong ô này).

- Token hợp lệ và tài khoản còn hoạt động: HTTP 200, trả danh sách 3 nhân viên.
- Không có token, token hết hạn, sai chữ ký hoặc sai định dạng: HTTP 401 kèm JSON theo Bài 3.
- User đã bị xóa hoặc vô hiệu hóa trong DB: HTTP 401 dù token chưa hết hạn.
- `/api/v1/auth/login` và `/api/v1/auth/test` vẫn public. Đăng nhập sai thông tin vẫn trả 401 như Bài 1.

`JwtAuthenticationFilter` extends `OncePerRequestFilter`, đọc Authorization, gọi `JwtProvider.validateAndGetUsername()` để kiểm tra chữ ký/hết hạn, load UserDetails từ DB và đặt Authentication vào SecurityContext. Quyền được lấy từ dữ liệu hiện tại trong DB.

SecurityConfig đăng ký filter trước `UsernamePasswordAuthenticationFilter` bằng `addFilterBefore()`, đặt session STATELESS để mỗi request private đều cần token. Filter chỉ đăng ký trong SecurityFilterChain, không dùng @Component để tránh bị đăng ký thêm ở Servlet container. Tham khảo [Spring Security Filter Architecture](https://docs.spring.io/spring-security/reference/servlet/architecture.html).

## Bài 3: JSON lỗi xác thực và quyền hạn

`JwtAuthenticationEntryPoint` implements `AuthenticationEntryPoint`, dùng ObjectMapper của Spring Boot để ghi JSON UTF-8 với HTTP 401:

```json
{
  "status": 401,
  "message": "Unauthorized: Token không hợp lệ hoặc đã hết hạn"
}
```

Filter bắt riêng `ExpiredJwtException` và `MalformedJwtException`, đồng thời xử lý các lỗi chữ ký/token còn lại. Khi token lỗi, filter xóa SecurityContext, gọi EntryPoint rồi kết thúc request, không chạy Controller. Trường hợp thiếu token được Spring Security chuyển tới cùng EntryPoint đã đăng ký trong `http.exceptionHandling()`.

Bài 3 thay phản hồi 403 ở Bài 2 bằng 401 cho lỗi chưa xác thực. API `/api/v1/auth/**` vẫn public và bỏ qua filter JWT, nên có thể đăng nhập lại khi token cũ hết hạn. Đăng nhập sai username/password vẫn trả thông báo của Bài 1.

Phân biệt:

- **AuthenticationEntryPoint → 401:** chưa xác thực, token sai hoặc hết hạn.
- **AccessDeniedHandler → 403:** đã xác thực nhưng thiếu quyền truy cập. Đã cấu hình handler trả JSON `{"status":403,"message":"Forbidden: Bạn không có quyền truy cập"}`.

Hiện `/api/v1/employees` chỉ yêu cầu đăng nhập; chưa đặt giới hạn theo role. Gửi token hợp lệ vẫn nhận HTTP 200 và danh sách nhân viên như Bài 2. Response lỗi không trả stack trace hoặc thông tin nội bộ.

Build/cache và file kiểm tra đặt ngoài thư mục nộp. Nếu IntelliJ tạo thêm build/.gradle/.idea, bỏ các thư mục này khi kéo thả GitHub.

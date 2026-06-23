# BÀI TẬP 2 - BẢO MẬT API: TÙY CHỈNH SECURITYFILTERCHAIN CHO REST API VÀ CSRF

## Phần 1: Phân tích logic

### 1. Sự khác biệt giữa cơ chế bảo vệ CSRF cho ứng dụng Web truyền thống và REST API

#### Ứng dụng Web truyền thống (Session-Based)

Trong ứng dụng web truyền thống, sau khi người dùng đăng nhập thành công, server sẽ tạo một Session và lưu Session ID trong Cookie của trình duyệt.

Mỗi khi người dùng gửi request, trình duyệt sẽ tự động đính kèm Cookie chứa Session ID vào request.

Điều này tạo ra nguy cơ tấn công CSRF (Cross-Site Request Forgery), bởi vì:

* Kẻ tấn công có thể dụ người dùng truy cập một trang web độc hại.
* Trang web đó gửi request tới hệ thống đích.
* Trình duyệt tự động gửi Cookie xác thực.
* Server hiểu nhầm request đó đến từ người dùng hợp lệ.

Để ngăn chặn điều này, Spring Security sử dụng CSRF Token:

* Server sinh ra một token ngẫu nhiên.
* Token được gửi tới client.
* Client phải gửi lại token này trong các request thay đổi dữ liệu (POST, PUT, DELETE,...).
* Server kiểm tra token trước khi xử lý request.

---

#### REST API (Stateless / Token-Based)

Trong REST API hiện đại:

* Server không lưu Session.
* Người dùng đăng nhập và nhận JWT Token hoặc Access Token.
* Token được gửi trong Authorization Header.

Ví dụ:

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

Khác với Cookie, trình duyệt không tự động gửi Authorization Header đến website khác.

Do đó:

* Hacker không thể lợi dụng trình duyệt tự động gửi JWT.
* Nguy cơ CSRF gần như không tồn tại khi token được lưu và gửi thủ công.

Vì vậy:

* Các REST API sử dụng JWT Stateless thường vô hiệu hóa CSRF.
* Thay vào đó tập trung bảo vệ bằng Authentication Token và Authorization.

---

### 2. Tại sao không nên vô hiệu hóa CSRF một cách mù quáng?

Nếu vô hiệu hóa CSRF cho toàn bộ ứng dụng web truyền thống:

```java
csrf.disable();
```

thì các request POST, PUT, DELETE sẽ không còn được kiểm tra nguồn gốc.

Khi đó:

1. Người dùng đăng nhập vào hệ thống.
2. Session Cookie vẫn còn hiệu lực.
3. Người dùng truy cập website độc hại.
4. Website độc hại gửi request đến hệ thống thật.
5. Trình duyệt tự động gửi Session Cookie.
6. Server chấp nhận request.

Hậu quả:

* Thay đổi mật khẩu trái phép.
* Chuyển tiền trái phép.
* Xóa dữ liệu.
* Thay đổi thông tin tài khoản.

Do đó:

* Chỉ nên vô hiệu hóa CSRF đối với REST API sử dụng Stateless Authentication.
* Không nên vô hiệu hóa CSRF cho các ứng dụng Web truyền thống sử dụng Session và Cookie.

---

## Phần 2: Thực thi

### Cấu hình SecurityFilterChain cho REST API

```java
package com.globalconnect.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http)
            throws Exception {

        http

            // Tắt CSRF cho REST API Stateless
            .csrf(csrf -> csrf.disable())

            // Tắt form login mặc định
            .formLogin(form -> form.disable())

            // Cấu hình phân quyền
            .authorizeHttpRequests(auth -> auth

                // Public APIs
                .requestMatchers(
                        "/api/auth/register",
                        "/api/auth/login"
                ).permitAll()

                // Các API còn lại yêu cầu xác thực
                .requestMatchers("/api/**").authenticated()

                // Request khác cho phép truy cập
                .anyRequest().permitAll()
            )

            // Basic configuration
            .httpBasic(Customizer.withDefaults());

        return http.build();
    }
}
```

---

### Cấu hình CSRF bằng CookieCsrfTokenRepository

Trong trường hợp hệ thống muốn vẫn duy trì CSRF Protection cho REST API, có thể sử dụng CookieCsrfTokenRepository.

```java
package com.globalconnect.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.csrf.CookieCsrfTokenRepository;

@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http)
            throws Exception {

        http

            .csrf(csrf -> csrf
                    .csrfTokenRepository(
                            CookieCsrfTokenRepository.withHttpOnlyFalse()
                    )
            )

            .formLogin(form -> form.disable())

            .authorizeHttpRequests(auth -> auth
                    .requestMatchers(
                            "/api/auth/register",
                            "/api/auth/login"
                    ).permitAll()
                    .requestMatchers("/api/**").authenticated()
                    .anyRequest().permitAll()
            );

        return http.build();
    }
}
```

### Giải thích

`CookieCsrfTokenRepository.withHttpOnlyFalse()`:

* Lưu CSRF Token trong Cookie.
* JavaScript phía client có thể đọc token.
* Client gửi token lại qua Header:

```http
X-XSRF-TOKEN: <csrf-token>
```

* Spring Security sẽ xác thực token trước khi xử lý request.

Giải pháp này phù hợp với:

* SPA (React, Angular, Vue).
* REST API có sử dụng Cookie.
* Hệ thống cần bảo vệ khỏi CSRF nhưng vẫn muốn hoạt động theo mô hình Stateless.

---

## Kết luận

* Web truyền thống sử dụng Session/Cookie cần bật CSRF Protection.
* REST API sử dụng JWT Stateless thường có thể vô hiệu hóa CSRF an toàn.
* Không nên tắt CSRF cho toàn bộ hệ thống nếu vẫn sử dụng Session Authentication.
* SecurityFilterChain cần:

  * Permit `/api/auth/register`
  * Permit `/api/auth/login`
  * Authenticate `/api/**`
  * Disable formLogin()
  * Cấu hình CSRF phù hợp với kiến trúc REST API.

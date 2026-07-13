---
title: "Ngày 5 (Sáng) — Spring Security + JWT Authentication"
track: java-spring
day: 5
topic: security-jwt
tags: [spring-security, jwt, authentication, authorization]
source_goc: Ngay5_TaiLieu.md (buổi sáng)
---

# Ngày 5 (Sáng) — Spring Security + JWT Authentication

# BUỔI SÁNG — Spring Security + JWT Authentication

### 1. Authentication vs Authorization — 2 khái niệm hay bị lẫn lộn

- **Authentication (Xác thực)** = "Bạn là ai?" — kiểm tra username/password đúng không, xác nhận danh tính
- **Authorization (Phân quyền)** = "Bạn được làm gì?" — đã biết bạn là ai rồi, nhưng bạn có quyền xóa bài viết này không, có phải admin không

Ẩn dụ: vào công ty, bảo vệ kiểm tra thẻ nhân viên (**Authentication** — xác nhận bạn đúng là nhân viên công ty), nhưng thẻ đó chỉ cho vào được tầng 1-3, muốn vào phòng server tầng 10 cần thẻ đặc biệt khác (**Authorization** — có quyền hay không).

### 2. Vì sao dùng JWT thay vì Session truyền thống?

**Cách cũ (Session):** Server tạo 1 "phiên làm việc" (session) sau khi đăng nhập, lưu trong bộ nhớ server, trả về cho client 1 `sessionId`. Mỗi request sau đó, server phải **tra cứu lại session đó trong bộ nhớ** để biết đây là ai.

**Vấn đề:** nếu có nhiều server (scale ngang), server B không biết session được tạo ở server A → phải đồng bộ session giữa các server (phức tạp, tốn tài nguyên).

**Cách mới (JWT — JSON Web Token):** Server không lưu gì cả (**stateless**). Sau khi đăng nhập, server tạo ra 1 "token" chứa sẵn thông tin người dùng **đã được mã hóa và ký số**, trả về cho client. Mỗi request sau đó, client tự gửi kèm token này, server chỉ cần **kiểm tra chữ ký** để biết token có hợp lệ không — không cần tra cứu ở đâu cả.

```
SESSION:  Client gửi sessionId  --> Server tra cứu trong bộ nhớ  --> Biết đây là ai
JWT:      Client gửi token      --> Server chỉ cần kiểm tra chữ ký  --> Biết đây là ai (không cần tra cứu)
```

### 3. Cấu trúc 1 JWT token

Token thực tế trông giống thế này: `eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhZG1pbiJ9.SflKxwRJSMeKKF2QT4f...`

Gồm 3 phần, ngăn cách bởi dấu `.`:

```
HEADER . PAYLOAD . SIGNATURE
```

- **Header**: thuật toán mã hóa dùng (ví dụ `HS256`)
- **Payload**: dữ liệu thực tế — username, quyền hạn, thời gian hết hạn (`exp`). **Lưu ý: phần này chỉ mã hóa Base64, KHÔNG bí mật** — ai cũng đọc được nội dung nếu có token, nên **không bao giờ** nhét mật khẩu hay dữ liệu nhạy cảm vào payload.
- **Signature**: chữ ký, tạo ra bằng cách mã hóa (Header + Payload) với 1 **secret key** chỉ server biết → đây là phần đảm bảo token không bị giả mạo/sửa đổi.

**Vì sao Signature quan trọng?** Nếu ai đó cố sửa payload (ví dụ đổi `"role": "user"` thành `"role": "admin"`), chữ ký sẽ **không còn khớp** với nội dung mới → server phát hiện ngay và từ chối token.

### 4. Spring Security hoạt động thế nào — luồng 1 request đi qua

```
Request đến
    │
    ▼
[Security Filter Chain] — 1 chuỗi các "trạm kiểm soát" request phải đi qua
    │
    ├── JwtAuthFilter: đọc token từ header "Authorization: Bearer xxx"
    │                  kiểm tra chữ ký hợp lệ không, còn hạn không
    │                  nếu hợp lệ -> đánh dấu "đã xác thực" cho request này
    │
    ▼
[Controller] — chỉ chạy nếu vượt qua được Filter Chain (hoặc endpoint đó cho phép public)
```

### 5. Code thực tế — xây API đăng nhập trả về JWT

**a) Thêm dependency (`pom.xml`):**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.3</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
```

**b) Class tạo & kiểm tra token:**
```java
@Component
public class JwtUtil {
    private final SecretKey secretKey = Jwts.SIG.HS256.key().build();
    private final long thoiHan = 1000 * 60 * 60; // 1 giờ

    public String taoToken(String username) {
        return Jwts.builder()
                .subject(username)
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + thoiHan))
                .signWith(secretKey)
                .compact();
    }

    public String layUsername(String token) {
        return Jwts.parser().verifyWith(secretKey).build()
                .parseSignedClaims(token).getPayload().getSubject();
    }

    public boolean hopLe(String token) {
        try {
            Jwts.parser().verifyWith(secretKey).build().parseSignedClaims(token);
            return true;
        } catch (Exception e) {
            return false; // hết hạn hoặc chữ ký sai
        }
    }
}
```

**c) API đăng nhập:**
```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {
    private final JwtUtil jwtUtil;
    private final PasswordEncoder passwordEncoder;

    public AuthController(JwtUtil jwtUtil, PasswordEncoder passwordEncoder) {
        this.jwtUtil = jwtUtil;
        this.passwordEncoder = passwordEncoder;
    }

    @PostMapping("/login")
    public ResponseEntity<?> dangNhap(@Valid @RequestBody LoginDTO dto) {
        // Thực tế: tìm user trong database theo username
        // So sánh password đã hash bằng passwordEncoder.matches(dto.getPassword(), userTrongDb.getPasswordHash())
        boolean dungMatKhau = kiemTraDangNhap(dto.getUsername(), dto.getPassword());

        if (!dungMatKhau) {
            return ResponseEntity.status(401).body("Sai tài khoản hoặc mật khẩu");
        }
        String token = jwtUtil.taoToken(dto.getUsername());
        return ResponseEntity.ok(Map.of("token", token));
    }
}
```

**Vì sao phải hash password, không lưu password thô?** Vì nếu database bị lộ (hack), password thô sẽ lộ theo — hàng loạt vụ rò rỉ dữ liệu thực tế xảy ra vì lưu password không mã hóa. `PasswordEncoder` (thường dùng `BCryptPasswordEncoder`) băm password thành chuỗi **không thể giải mã ngược lại**, chỉ có thể **so sánh** (hash lại password người dùng nhập, so với hash đã lưu).

### Bài tập buổi sáng

**Bài 1:** Viết `JwtUtil` như trên, viết API `/api/auth/login` nhận `username`/`password` cố định (`admin`/`123456`, có thể hardcode tạm), trả về token nếu đúng.

**Bài 2:** Viết 1 API bảo vệ đơn giản `/api/profile` chỉ trả dữ liệu nếu request có header `Authorization: Bearer <token hợp lệ>` (gợi ý: dùng `JwtUtil.hopLe()` kiểm tra thủ công trong Controller trước, chưa cần viết Filter phức tạp — Filter đầy đủ có thể tìm hiểu thêm ở link tham khảo).

---

## 💻 Code Example (repo `Code`)

> Nguồn: [`SangLeSoftZ/Code`](https://github.com/SangLeSoftZ/Code)

**Tạo & xác thực JWT** — [`jwt/JwtUtil.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/jwt/JwtUtil.java):

```java
package com.example.demo.jwt;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.JwtException;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.util.Date;

@Component
public class JwtUtil {

    private static final String SECRET = "softz-secret-key-must-be-32chars!!";
    private static final long   EXPIRATION_MS = 1000 * 60 * 60; // 1 giờ

    private SecretKey getKey() {
        return Keys.hmacShaKeyFor(SECRET.getBytes(StandardCharsets.UTF_8));
    }

    // Tạo token với username và role
    public String taoToken(String username, String role) {
        return Jwts.builder()
                .subject(username)
                .claim("role", role)                            // thêm role vào payload
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + EXPIRATION_MS))
                .signWith(getKey())
                .compact();
    }

    // Kiểm tra token hợp lệ: chữ ký đúng + chưa hết hạn
    public boolean hopLe(String token) {
        try {
            Jwts.parser()
                .verifyWith(getKey())
                .build()
                .parseSignedClaims(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }

    // Đọc username từ token
    public String layUsername(String token) {
        return getClaims(token).getSubject();
    }

    // Đọc role từ token
    public String layRole(String token) {
        return getClaims(token).get("role", String.class);
    }

    // Helper: parse và trả Claims
    private Claims getClaims(String token) {
        return Jwts.parser()
                .verifyWith(getKey())
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }
}
```

**API đăng nhập trả JWT** — [`jwt/controller/LoginController.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/jwt/controller/LoginController.java):

```java
package com.example.demo.jwt.controller;

import com.example.demo.jwt.JwtUtil;
import com.example.demo.jwt.dto.LoginRequest;
import com.example.demo.jwt.dto.LoginResponse;
import com.example.demo.jwt.entity.User;
import com.example.demo.jwt.service.UserService;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.Map;
import java.util.Optional;

@RestController
@RequestMapping("/api/auth")
public class LoginController {

    private final JwtUtil jwtUtil;
    private final UserService userService;

    public LoginController(JwtUtil jwtUtil, UserService userService) {
        this.jwtUtil = jwtUtil;
        this.userService = userService;
    }

    /**
     * POST /api/auth/login
     *
     * Request body:  { "username": "admin", "password": "123456" }
     *
     * Response 200:  { "id": 1, "username": "admin", "email": "admin@softz.com",
     *                  "role": "ADMIN", "active": true, "token": "eyJ..." }
     * Response 401:  { "error": "Sai username hoặc password" }
     * Response 403:  { "error": "Tài khoản đã bị khóa" }
     */
    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody LoginRequest request) {

        System.out.println("=== LOGIN DEBUG ===");
        System.out.println("Username: [" + request.getUsername() + "]");
        System.out.println("Password: [" + request.getPassword() + "]");

        // Bước 1: Tìm user theo username
        Optional<User> userOpt = userService.timTheoUsername(request.getUsername());
        System.out.println("Tìm thấy user: " + userOpt.isPresent());

        if (userOpt.isEmpty()) {
            return ResponseEntity.status(401)
                    .body(Map.of("error", "Sai username hoặc password"));
        }

        User user = userOpt.get();

        // Bước 2: Kiểm tra active
        if (!user.getActive()) {
            return ResponseEntity.status(403)
                    .body(Map.of("error", "Tài khoản đã bị khóa"));
        }

        // Bước 3: Kiểm tra password BCrypt
        boolean matKhauDung = userService.kiemTraPassword(
                request.getPassword(),
                user.getPassword()
        );
        System.out.println("Mật khẩu đúng: " + matKhauDung);
        System.out.println("===================");

        if (!matKhauDung) {
            return ResponseEntity.status(401)
                    .body(Map.of("error", "Sai username hoặc password"));
        }

        // Bước 4: Tạo JWT và trả LoginResponse đầy đủ cho Flutter
        String token = jwtUtil.taoToken(user.getUsername(), user.getRole());

        LoginResponse response = new LoginResponse(
                user.getId(),
                user.getUsername(),
                user.getEmail(),
                user.getRole(),
                user.getActive(),
                token
        );

        return ResponseEntity.ok(response);
    }
}
```

Các file liên quan: [`jwt/entity/User.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/jwt/entity/User.java),
[`jwt/service/impl/UserServiceImpl.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/jwt/service/impl/UserServiceImpl.java) (BCrypt password check),
[`jwt/controller/ProfileController.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/jwt/controller/ProfileController.java) (API cần token mới gọi được).

---
title: "Ngày 9 (Tuần 2 - Ngày 2, Tối) — @PreAuthorize: Role-based Authorization"
track: java-spring
day: 9
topic: preauthorize-role-based-authorization
tags: [spring-boot, spring-security, preauthorize, enablemethodsecurity, hasrole, hasauthority]
source_goc: Tuan2_Ngay2_TaiLieu.md
---

# Ngày 9 (Tuần 2 - Ngày 2, Tối) — @PreAuthorize: Role-based Authorization

## 1. `@EnableMethodSecurity` — lưu ý quan trọng nhất bài này

Đây là lỗi hay gặp nhất: viết `@PreAuthorize("hasRole('ADMIN')")` lên method nhưng
**annotation im lặng, không có tác dụng gì cả** — USER thường vẫn gọi được bình thường,
không có exception, không có lỗi hiện ra. Nguyên nhân luôn là 1 trong 2:

1. Thiếu `@EnableMethodSecurity` trong file cấu hình Security.
2. `SecurityContext` chưa được set (không ai xác thực token trước khi `@PreAuthorize` chạy).

Code thật trong repo — `config/SecurityConfig.java`:

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity // Bật @PreAuthorize — THIẾU dòng này thì annotation không có tác dụng!
public class SecurityConfig {
    ...
}
```

`@EnableWebSecurity` chỉ bật Spring Security ở tầng **request/URL** (`.authorizeHttpRequests`).
`@PreAuthorize` là bảo vệ ở tầng **method** (chạy trước khi method thực thi) — 2 cơ chế
khác nhau, cần bật riêng bằng `@EnableMethodSecurity`. Đây chính là lý do annotation "im
lặng": Spring Security vẫn chạy bình thường ở tầng URL, chỉ có phần method-level là bị bỏ
qua hoàn toàn nếu thiếu annotation này.

## 2. Vì sao `JwtAuthFilter` cũng bắt buộc phải đúng thứ tự

`@PreAuthorize` kiểm tra role dựa trên `SecurityContextHolder` — nếu không có ai "đăng
ký" thông tin user (username + authorities) vào đó trước, `@PreAuthorize` sẽ luôn coi như
**chưa xác thực**, dẫn tới 403 dù token đúng và đúng role. Trích comment thật trong
`config/JwtAuthFilter.java`:

```java
/**
 * Chạy TRƯỚC mọi request, đọc Authorization header, parse JWT và
 * set SecurityContext — để @PreAuthorize biết user hiện tại là ai.
 *
 * Nếu không set SecurityContext thì @PreAuthorize không biết role
 * → luôn trả 403 dù đúng role.
 */
```

Và trong `SecurityConfig.java`, thứ tự filter được khai báo rõ ràng:

```java
// Thêm JwtAuthFilter TRƯỚC UsernamePasswordAuthenticationFilter
// để SecurityContext được set trước khi @PreAuthorize chạy
.addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
```

Tóm lại **2 điều kiện bắt buộc để `@PreAuthorize` hoạt động đúng**:
1. `@EnableMethodSecurity` có mặt trong `SecurityConfig`.
2. Có 1 filter (ở đây là `JwtAuthFilter`) chạy trước, set `SecurityContextHolder` với
   đúng username + authorities lấy từ token.

## 3. `hasRole` vs `hasAuthority` — khác nhau ở tiền tố `ROLE_`

Trong `JwtAuthFilter.java`, role lấy từ token (`"ADMIN"`, `"USER"`, `"MANAGER"`) được bọc
lại trước khi đưa vào `SecurityContext`:

```java
String role = jwtUtil.layRole(token); // "ADMIN", "USER", "MANAGER"

// Spring Security convention: hasRole('ADMIN') kiểm tra authority "ROLE_ADMIN"
// Nên cần thêm prefix "ROLE_" vào trước role từ token
SimpleGrantedAuthority authority = new SimpleGrantedAuthority("ROLE_" + role);
```

Đây là quy ước riêng của Spring Security, không phải ngẫu nhiên:

| | `hasRole('ADMIN')` | `hasAuthority('ADMIN')` |
|---|---|---|
| Spring tự thêm `ROLE_` trước khi so sánh? | **Có** — thực chất so sánh authority `"ROLE_ADMIN"` | **Không** — so sánh chính xác chuỗi `"ADMIN"` |
| Dùng khi nào | Khi authority của bạn lưu theo convention `ROLE_xxx` (trường hợp phổ biến, đúng như code trong repo) | Khi bạn có các quyền chi tiết không theo dạng "role" (vd `"PERMISSION_DELETE_USER"`) |

→ Nếu code set authority là `"ROLE_ADMIN"` (như trong repo) mà lại viết
`@PreAuthorize("hasAuthority('ADMIN')")` (thiếu tiền tố `ROLE_`) thì **sẽ luôn fail** dù
đúng role — vì `hasAuthority` so khớp chuỗi chính xác, không tự thêm `ROLE_` giúp bạn.

Code thật — `controller/AdminController.java`:

```java
@PreAuthorize("hasRole('ADMIN')")
@GetMapping("/users")
public ResponseEntity<List<User>> layTatCaUser() { ... }

@PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
@GetMapping("/dashboard")
public ResponseEntity<?> xemDashboard(...) { ... }

@PreAuthorize("isAuthenticated()")
@GetMapping("/me")
public ResponseEntity<?> xemThongTinToi(...) { ... }
```

`isAuthenticated()` không kiểm tra role cụ thể nào — chỉ cần đã xác thực thành công (có
token hợp lệ) là gọi được, dùng cho endpoint mọi user đã đăng nhập đều có quyền xem.

## 4. 401 vs 403 — 2 lỗi rất hay bị nhầm

| | 401 Unauthorized | 403 Forbidden |
|---|---|---|
| Ý nghĩa | **Chưa xác thực được** — không có token, token sai, hoặc hết hạn | **Đã xác thực thành công nhưng không đủ quyền** — biết bạn là ai rồi, nhưng role không cho phép |
| Ví dụ trong repo | Gọi API mà không gắn header `Authorization: Bearer ...`, hoặc token hết hạn | Đăng nhập bằng tài khoản `role: USER` nhưng gọi `/api/admin/users` (chỉ `ADMIN`) |
| Exception Spring ném ra | `AuthenticationException` | `AccessDeniedException` (`@PreAuthorize` fail sẽ ném lỗi này) |

Code thật — `exception/GlobalExceptionHandler.java` xử lý đúng 2 loại riêng biệt:

```java
// 403 — đã đăng nhập nhưng không đủ quyền (@PreAuthorize fail)
@ExceptionHandler(AccessDeniedException.class)
public ResponseEntity<ApiError> handleAccessDenied(...) {
    return ResponseEntity.status(HttpStatus.FORBIDDEN)
            .body(new ApiError(403, "Forbidden",
                    "Bạn không có quyền thực hiện thao tác này", ...));
}

// 401 — chưa đăng nhập hoặc token sai
@ExceptionHandler(AuthenticationException.class)
public ResponseEntity<ApiError> handleAuthentication(...) {
    return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
            .body(new ApiError(401, "Unauthorized",
                    "Chưa xác thực — vui lòng đăng nhập", ...));
}
```

Ghi nhớ ngắn gọn: **401 = "tôi không biết bạn là ai"**, **403 = "tôi biết bạn là ai rồi,
nhưng bạn không được phép làm việc này"**.

## 5. Bài tập

**Bài 5 (D09_dap-an-checklist.md):** Giải thích bằng lời (không cần code): nếu xoá dòng
`@EnableMethodSecurity` khỏi `SecurityConfig`, gọi `/api/admin/users` bằng tài khoản
`role: USER` sẽ trả về kết quả gì? Vì sao?

**Bài 6:** Thêm 1 endpoint mới `GET /api/admin/reports` trong `AdminController`, chỉ cho
phép `MANAGER` truy cập (không cho `ADMIN` lẫn `USER`). Viết đúng `@PreAuthorize` cho
endpoint này.

##  Code Example (repo `java.git`)

> Xem nguyên các file tại branch `tuan2`:
> [`SecurityConfig.java`](https://github.com/SangLeSoftZ/java/blob/tuan2/src/main/java/com/example/demo/config/SecurityConfig.java) ·
> [`JwtAuthFilter.java`](https://github.com/SangLeSoftZ/java/blob/tuan2/src/main/java/com/example/demo/config/JwtAuthFilter.java) ·
> [`AdminController.java`](https://github.com/SangLeSoftZ/java/blob/tuan2/src/main/java/com/example/demo/controller/AdminController.java) ·
> [`GlobalExceptionHandler.java`](https://github.com/SangLeSoftZ/java/blob/tuan2/src/main/java/com/example/demo/exception/GlobalExceptionHandler.java)

##  Tài liệu tham khảo

- [Spring Security — Method Security](https://docs.spring.io/spring-security/reference/servlet/authorization/method-security.html)
- [Spring Security — Authorize HttpServletRequests](https://docs.spring.io/spring-security/reference/servlet/authorization/authorize-http-requests.html)
- [Spring Security — Expression-Based Access Control (hasRole/hasAuthority)](https://docs.spring.io/spring-security/reference/servlet/authorization/expression-based.html)


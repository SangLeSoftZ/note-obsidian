
# Ngày 8 / Tuần 2 - Ngày 1 (Tối, 30%) — Spring Boot: Refresh Token

# BUỔI TỐI (30%) — Refresh Token

## 1. Vấn đề: Access Token nên có hạn ngắn, nhưng ngắn quá thì phiền

Ở `D05_spring-security-jwt.md`, token JWT có hạn 1 giờ. Nếu để hạn **dài** (VD 30 ngày) để đỡ đăng nhập lại — token bị lộ thì rủi ro 30 ngày. Nếu để hạn **ngắn** (1 giờ) cho an toàn — user bị đăng xuất liên tục.

**Giải pháp: 2 loại token riêng biệt:**

| | Access Token | Refresh Token |
|---|---|---|
| Thời hạn | Ngắn (15-60 phút) | Dài (7-30 ngày) |
| Dùng để | Gửi kèm mọi API request | Chỉ để xin cấp Access Token mới |
| Gửi đi đâu | Header `Authorization` mỗi request | Chỉ gửi tới `/api/auth/refresh` |
| Nếu bị lộ | Rủi ro thấp hơn | Rủi ro cao hơn — cần lưu an toàn |

## 2. Luồng hoạt động

```
1. Dang nhap -> Server tra ve CA 2: accessToken (15 phut) + refreshToken (7 ngay)
2. Moi request binh thuong -> gui kem accessToken
3. Sau 15 phut, accessToken het han -> request bi 401
4. App tu dong goi /api/auth/refresh, gui kem refreshToken
5. Server kiem tra refreshToken con hop le -> cap accessToken MOI
6. Chi khi refreshToken cung het han -> moi bat dang nhap lai that su
```

## 3. Code — mở rộng `JwtUtil`

```java
@Component
public class JwtUtil {
    private final SecretKey secretKey = Jwts.SIG.HS256.key().build();
    private final long accessTokenThoiHan = 1000 * 60 * 15;
    private final long refreshTokenThoiHan = 1000L * 60 * 60 * 24 * 7;

    public String taoAccessToken(String username) {
        return Jwts.builder()
                .subject(username)
                .claim("type", "access")
                .expiration(new Date(System.currentTimeMillis() + accessTokenThoiHan))
                .signWith(secretKey).compact();
    }

    public String taoRefreshToken(String username) {
        return Jwts.builder()
                .subject(username)
                .claim("type", "refresh")
                .expiration(new Date(System.currentTimeMillis() + refreshTokenThoiHan))
                .signWith(secretKey).compact();
    }
}
```

`claim("type", ...)` để phân biệt 2 loại token — tránh gửi nhầm `refreshToken` vào chỗ cần `accessToken`.

## 4. API `/api/auth/refresh`

```java
@PostMapping("/refresh")
public ResponseEntity<?> refresh(@RequestBody Map<String, String> body) {
    String refreshToken = body.get("refreshToken");

    if (!jwtUtil.hopLe(refreshToken)) {
        return ResponseEntity.status(401).body("Refresh token khong hop le hoac da het han");
    }

    Claims claims = jwtUtil.layClaims(refreshToken);
    if (!"refresh".equals(claims.get("type"))) {
        return ResponseEntity.status(401).body("Token khong dung loai");
    }

    String accessTokenMoi = jwtUtil.taoAccessToken(claims.getSubject());
    return ResponseEntity.ok(Map.of("accessToken", accessTokenMoi));
}
```

> Nâng cao (tham khảo, chưa bắt buộc): hệ thống thực tế lưu refresh token vào database (kèm trạng thái còn hiệu lực/đã thu hồi) để chủ động vô hiệu hóa khi cần (VD "Đăng xuất khỏi mọi thiết bị").

## Bài tập

**Bài 6:** Thêm `taoRefreshToken()`, API login trả về cả 2 token. Viết `/api/auth/refresh`, test Postman: login lấy refreshToken → giả lập accessToken hết hạn → gọi `/refresh` lấy accessToken mới.

**Bài 7:** Test lỗi: gửi `accessToken` (không phải `refreshToken`) vào `/api/auth/refresh` — xác nhận bị từ chối đúng nhờ kiểm tra `claim("type")`.

**Bài 8 (nâng cao):** Lưu refreshToken vào database (bảng `refresh_tokens`: `token`, `username`, `con_hieu_luc`), viết API `/api/auth/logout` thu hồi 1 refresh token cụ thể.

---

## Code Example (repo `Code`)

> Nguồn: [`SangLeSoftZ/java`](https://github.com/SangLeSoftZ/java)
>  Dã triển khai — vị trí dự kiến theo quy ước (`QUY_UOC_DAT_TEN.md`, mục 2.2): util\JwtUtil.java, `controller\AuthController.java` (bổ sung method `refresh`). Khi đẩy code lên, thêm link + snippet vào đúng mục này.

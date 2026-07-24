---
title: "Ngày 18 (Tuần 4 - Ngày 2, Sáng) — flutter_secure_storage: Lưu token đúng chuẩn bảo mật"
track: flutter
day: 18
topic: secure-storage-token
tags: [flutter, secure-storage, keystore, keychain, token]
source_goc: Tuan4-Ngay2-SecureStorage-DioInterceptor-Scheduled-Async.md
---

# Ngày 18 (Tuần 4 - Ngày 2, Sáng) — flutter_secure_storage: Lưu token đúng chuẩn bảo mật

> Lưu ý về số ngày: tài liệu gốc ghi "Tuần 4 - Ngày 2", nhưng code thật nằm ở thư mục
> `tuan3_ngay8` (nối tiếp `tuan3_ngay7` — D17). Ghi chú dùng số **D18** theo đúng thứ tự
> tiếp theo trong vault.

## 1. Vì sao Hive/shared_preferences không đủ an toàn cho token

Cả `Hive` và `shared_preferences` đều lưu dữ liệu dưới dạng **file thông thường** trên bộ
nhớ thiết bị — trên thiết bị đã root/jailbreak, những file này có thể bị đọc trực tiếp.
Nếu access token/refresh token nằm trong đó ở dạng gần như văn bản thường, kẻ tấn công có
thể lấy token và **mạo danh người dùng**.

`flutter_secure_storage` không tự phát minh cơ chế mã hóa riêng — nó là 1 **lớp bọc
(wrapper)** gọi tới cơ chế lưu trữ bảo mật **có sẵn của hệ điều hành**: **Keychain** trên
iOS, **Keystore/EncryptedSharedPreferences** trên Android.

## 2. Phát hiện quan trọng: `AuthLocalDataSource` thật đã dùng SecureStorage từ trước

Khác với kịch bản "migrate từ Hive sang SecureStorage" mô tả trong tài liệu lý thuyết
gốc, code thật cho thấy `AuthLocalDataSource` (viết từ Tuần 1) **đã dùng
`flutter_secure_storage` sẵn từ đầu** — không có đoạn code Hive/`shared_preferences` nào
cần migrate. Comment ngay đầu file `bai1_secure_storage_screen.dart` xác nhận điều này:

```dart
// AuthLocalDataSource HIỆN TẠI đã dùng SecureStorage — không cần migrate
// File này demo trực tiếp cách hoạt động của SecureStorage
```

Code thật — `login/data/datasources/auth_local_datasource.dart`:

```dart
class AuthLocalDataSource {
  final FlutterSecureStorage _storage;

  AuthLocalDataSource({FlutterSecureStorage? storage})
    : _storage = storage ?? const FlutterSecureStorage(
        aOptions: AndroidOptions(encryptedSharedPreferences: true),
      );

  Future<void> saveAuthInfo({
    required String token,
    required String username,
    required String role,
    required String userId,
    String? refreshToken,
  }) async {
    await Future.wait([
      _storage.write(key: kTokenKey, value: token),
      // Lưu refresh token: nếu có riêng thì dùng riêng, không thì dùng chính token
      _storage.write(key: kRefreshTokenKey, value: refreshToken ?? token),
      _storage.write(key: kUsernameKey, value: username),
      _storage.write(key: kRoleKey, value: role),
      _storage.write(key: kUserIdKey, value: userId),
    ]);
  }

  Future<String?> getToken() => _storage.read(key: kTokenKey);
  Future<String?> getRefreshToken() => _storage.read(key: kRefreshTokenKey);

  // ── XÓA CHỈ access token — giữ refresh token (để test 401) ───
  Future<void> deleteAccessTokenOnly() => _storage.delete(key: kTokenKey);

  // ── KIỂM TRA đã đăng nhập chưa (dựa vào refresh token còn không)
  // Dùng refresh token vì access token có thể đã hết hạn
  Future<bool> isLoggedIn() async {
    final refreshToken = await getRefreshToken();
    return refreshToken != null && refreshToken.isNotEmpty;
  }
}
```

**Chi tiết đáng chú ý không có trong tài liệu gốc:**
- `saveAuthInfo` dùng `Future.wait([...])` để ghi **song song** 5 giá trị cùng lúc thay vì
  `await` tuần tự từng dòng — nhanh hơn vì các thao tác ghi độc lập, không phụ thuộc nhau.
- `refreshToken ?? token` — nếu Spring Boot chưa trả `refreshToken` riêng (một số API cũ),
  tự động dùng chung `token` cho cả 2 khóa, đảm bảo tương thích ngược.
- `isLoggedIn()` kiểm tra dựa trên **refresh token còn không**, không phải access token —
  vì access token có thể đã hết hạn (ngắn hạn) trong khi người dùng vẫn "coi như" đăng
  nhập nếu refresh token còn hạn (dài hạn).
- `deleteAccessTokenOnly()` — hàm được viết riêng **chỉ để phục vụ việc test** (buổi
  chiều): xóa access token nhưng giữ refresh token, mô phỏng đúng tình huống "access token
  hết hạn" mà không cần đợi thời gian thật hoặc chỉnh cấu hình server.

## 3. `encryptedSharedPreferences: true` — bắt buộc trên Android

```dart
const _storage = FlutterSecureStorage(
  aOptions: AndroidOptions(encryptedSharedPreferences: true),
);
```

Mặc định (không khai báo `AndroidOptions`), `flutter_secure_storage` trên Android cũ hơn
dùng cơ chế mã hóa đơn giản hơn kèm cảnh báo deprecated. `encryptedSharedPreferences:
true` yêu cầu dùng thư viện `androidx.security` chính thức của Google — mã hóa dữ liệu
bằng khóa được chính **Android Keystore** quản lý (khóa không bao giờ rời khỏi phần cứng
bảo mật của thiết bị).

**Toàn bộ API đều là `Future`** — vì việc đọc/ghi Keychain/Keystore đi qua **kênh giao
tiếp native** (Platform Channel, khái niệm đã học Tuần 3 Ngày 4) tới hệ điều hành, có độ
trễ nhỏ nhưng thực sự tồn tại.

## 4. Màn hình demo — so sánh song song `SecureTokenStorage` demo vs token thật của app

Code thật — `lib/tuan3_ngay8/bai1_secure_storage_screen.dart` — không chỉ demo 1
`SecureTokenStorage` viết riêng cho bài học, mà còn **đọc song song** trạng thái đăng nhập
thật từ `AuthLocalDataSource`:

```dart
Future<void> _docTrangThaiHienTai() async {
  final tokens = await _demo.layTatCa();           // SecureTokenStorage demo
  final loggedIn = await _authLocal.isLoggedIn();  // AuthLocalDataSource THẬT
  final token = await _authLocal.getToken();
  setState(() {
    _accessToken = tokens['access_token'];
    _refreshToken = tokens['refresh_token'];
    _isLoggedIn = loggedIn;
    _authToken = token;
  });
}
```

Cách trình bày này giúp người học phân biệt rõ: 1 bên là "tự tay thử nghiệm" ghi/đọc/xóa
token demo để hiểu cơ chế, 1 bên là quan sát trạng thái đăng nhập **thật** của app đang
chạy — cả 2 cùng dùng chung công nghệ nền (SecureStorage) nhưng phục vụ mục đích khác
nhau.

## Bài tập

**Bài 1 (đã có sẵn — `bai1_secure_storage_screen.dart`):** Chạy màn hình, nhập access
token/refresh token demo, bấm Lưu → Đọc lại → Xóa, quan sát log xác nhận đúng hành vi ghi/
đọc/xóa qua Keystore/Keychain.

**Bài 2 (không cần migrate — đã dùng SecureStorage từ đầu):** Xác nhận `AuthLocalDataSource.isLoggedIn()`
dựa trên refresh token còn hạn, không phải access token — đăng nhập thật, sau đó gọi
`deleteAccessTokenOnly()` (sẽ dùng ở bài chiều) để tự kiểm chứng ứng dụng vẫn coi là "đã
đăng nhập" dù access token đã bị xóa.

## Code Example (repo `flutter.git`)

> Xem nguyên các file tại branch `flutter_secure_storage+Dio-Interceptor`:
> [`bai1_secure_storage_screen.dart`](<https://github.com/SangLeSoftZ/flutter/blob/flutter_secure_storage+Dio-Interceptor/testflutter/lib/tuan3_ngay8/bai1_secure_storage_screen.dart>) ·
> [`auth_local_datasource.dart`](<https://github.com/SangLeSoftZ/flutter/blob/flutter_secure_storage+Dio-Interceptor/testflutter/lib/login/data/datasources/auth_local_datasource.dart>)


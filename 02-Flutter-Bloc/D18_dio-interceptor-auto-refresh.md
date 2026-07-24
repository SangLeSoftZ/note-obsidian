---
title: "Ngày 18 (Tuần 4 - Ngày 2, Chiều) — Dio Interceptor: Tự gắn token + Tự refresh khi 401"
track: flutter
day: 18
topic: dio-interceptor-auto-refresh
tags: [flutter, dio, queuedinterceptor, auto-refresh, 401]
source_goc: Tuan4-Ngay2-SecureStorage-DioInterceptor-Scheduled-Async.md
---

# Ngày 18 (Tuần 4 - Ngày 2, Chiều) — Dio Interceptor: Tự gắn token + Tự refresh khi 401

## 1. Vấn đề: refresh token đã có (Tuần 2) nhưng chưa được tự động dùng

Tuần 2 đã xây xong endpoint `/api/auth/refresh` phía Spring Boot — nhưng phía Flutter
chưa có cơ chế tự động gọi nó. Mục tiêu hôm nay: chỗ nào gọi `dio.get('/api/tasks')` vẫn
viết y hệt như cũ, nhưng phía sau — tự gắn token, tự phát hiện 401, tự refresh, tự gửi lại
request ban đầu — người dùng hoàn toàn không nhận ra có chuyện gì xảy ra.

## 2. `AuthInterceptor` — code thật, dùng luôn `AuthLocalDataSource` thay vì storage riêng

Code thật — `lib/tuan3_ngay8/bai3_auth_interceptor.dart` — khác ví dụ lý thuyết gốc (dùng
1 `SecureTokenStorage` viết riêng), code thật tái sử dụng thẳng `AuthLocalDataSource` đã
có từ Tuần 1:

```dart
class AuthInterceptor extends QueuedInterceptor {
  final Dio _dio;
  final AuthLocalDataSource _authLocal;
  bool _dangRefresh = false;
  static const String _refreshUrl = 'http://10.0.2.2:8080/api/auth/refresh';

  AuthInterceptor(this._dio, this._authLocal);

  @override
  Future<void> onRequest(RequestOptions options, RequestInterceptorHandler handler) async {
    final token = await _authLocal.getToken();
    if (token != null && token.isNotEmpty) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    handler.next(options);
  }

  @override
  Future<void> onError(DioException err, ErrorInterceptorHandler handler) async {
    if (err.response?.statusCode != 401) return handler.next(err);
    try {
      final tokenMoi = await _lamMoiToken();
      err.requestOptions.headers['Authorization'] = 'Bearer $tokenMoi';
      final response = await _dio.fetch(err.requestOptions); // gửi lại request GỐC
      handler.resolve(response);
    } catch (e) {
      await _authLocal.clearAuthInfo();
      handler.next(err);
    }
  }
}
```

`http://10.0.2.2:8080` — đúng địa chỉ Android Emulator dùng để gọi tới `localhost` của
máy host, cùng quy ước đã dùng ở `StompService` (Tuần 3 Ngày 2).

## 3. `_lamMoiToken()` — cờ `_dangRefresh` đảm bảo chỉ gọi refresh đúng 1 lần

```dart
Future<String> _lamMoiToken() async {
  // Nếu đã có request khác đang refresh -> chờ kết quả đó, không gọi thêm
  if (_dangRefresh) {
    while (_dangRefresh) {
      await Future.delayed(const Duration(milliseconds: 100));
    }
    final token = await _authLocal.getToken();
    if (token != null && token.isNotEmpty) return token;
    throw Exception('Refresh thất bại ở request khác');
  }

  _dangRefresh = true;
  try {
    final refreshToken = await _authLocal.getRefreshToken();
    if (refreshToken == null || refreshToken.isEmpty) {
      throw Exception('Không có refresh token');
    }

    // Dùng Dio RIÊNG — không gắn AuthInterceptor này, tránh vòng lặp
    final dioRieng = Dio();
    final info = await _authLocal.getAuthInfo();
    final response = await dioRieng.post(
      _refreshUrl,
      data: {'refreshToken': refreshToken},
      options: Options(sendTimeout: const Duration(seconds: 10), receiveTimeout: const Duration(seconds: 10)),
    );

    final accessTokenMoi = response.data['accessToken'] as String? ?? response.data['token'] as String?;
    if (accessTokenMoi == null) throw Exception('Response không có accessToken');
    final refreshTokenMoi = response.data['refreshToken'] as String?;

    await _authLocal.saveAuthInfo(
      token: accessTokenMoi,
      refreshToken: refreshTokenMoi, // null = giữ nguyên refresh token cũ
      username: response.data['username'] as String? ?? info['username'] ?? '',
      role: response.data['role'] as String? ?? info['role'] ?? '',
      userId: response.data['id']?.toString() ?? info['userId'] ?? '',
    );

    return accessTokenMoi;
  } finally {
    _dangRefresh = false;
  }
}
```

**Vì sao kế thừa `QueuedInterceptor` thay vì `Interceptor` thường?** `QueuedInterceptor`
tự động **xếp hàng** các request đang chờ xử lý trong `onError`/`onRequest` — nếu có nhiều
request cùng gặp 401 gần như đồng thời, chúng được xử lý **tuần tự**, không chồng chéo.
Kết hợp cờ `_dangRefresh` tự viết, đảm bảo dù có bao nhiêu request cùng lúc bị 401,
`/api/auth/refresh` **chỉ thực sự được gọi đúng 1 lần**.

**Vì sao dùng `Dio` riêng (`dioRieng`) để gọi refresh?** Nếu dùng lại `_dio` chính (đã gắn
`AuthInterceptor`), khi gọi `/api/auth/refresh`, chính interceptor này sẽ lại chạy
`onRequest` (cố gắn access token cũ đã hết hạn), có thể vô tình kích hoạt lại `onError` của
chính nó, tạo vòng lặp khó kiểm soát.

`response.data['accessToken'] as String? ?? response.data['token'] as String?` — chi tiết
thực tế không có trong ví dụ lý thuyết gốc: chấp nhận **2 khả năng tên field** khác nhau từ
response server (`accessToken` hoặc `token`), tăng độ chịu lỗi nếu API backend trả về tên
field không hoàn toàn thống nhất.

## 4. Đăng ký `AuthInterceptor` vào `Dio` chính

```dart
Dio taoDioChinh(AuthLocalDataSource authLocal) {
  final dio = Dio(BaseOptions(
    baseUrl: 'http://10.0.2.2:8080',
    connectTimeout: const Duration(seconds: 10),
    receiveTimeout: const Duration(seconds: 10),
    headers: {'Content-Type': 'application/json'},
  ));
  dio.interceptors.add(AuthInterceptor(dio, authLocal));
  return dio;
}
```

Từ giờ, mọi nơi trong app gọi API qua `Dio` được tạo từ `taoDioChinh(...)` đều tự động
được hưởng cơ chế gắn token + auto-refresh — không cần sửa lại bất kỳ Repository/
DataSource nào đã viết từ trước.

## 5. Bài 4+5 — kỹ thuật test hay nhất: nút "Giả lập hết hạn" thay vì đợi thời gian thật

Code thật — `lib/tuan3_ngay8/bai4_bai5_interceptor_demo_screen.dart` — thay vì đợi access
token hết hạn thật hoặc sửa cấu hình server (như tài liệu lý thuyết gốc gợi ý), màn hình
demo có 1 nút bấm **giả lập hết hạn ngay lập tức**:

```dart
// GIẢ LẬP HẾT HẠN: xóa CHỈ access token, giữ refresh token
// -> khi gọi API: không có token -> server trả 401
// -> interceptor dùng refresh token để lấy token mới
Future<void> _giamLapHetHan() async {
  await _authLocal.deleteAccessTokenOnly();
  setState(() => _accessTokenBiXoa = true);
}
```

Đây là kỹ thuật test thực tế và tiện lợi hơn nhiều so với chỉnh thời hạn token phía server
— chỉ cần 1 hàm xóa đúng 1 field (access token, giữ nguyên refresh token), lập tức tái tạo
được đúng tình huống "access token hết hạn nhưng refresh token còn dùng được" để test luồng
401 → refresh → retry bất cứ lúc nào, không phải chờ đợi.

**Bài 4 — gọi 1 API sau khi giả lập hết hạn:**

```dart
Future<void> _goiApi() async {
  final r = await _dio.get('/api/tasks');
  // THÀNH CÔNG dù access token đã bị xóa trước đó — interceptor đã tự refresh
}
```

**Bài 5 — gọi 3 API song song, kiểm chứng `/refresh` chỉ gọi đúng 1 lần:**

```dart
Future<void> _goiSongSong() async {
  final results = await Future.wait([
    _dio.get('/api/tasks'),
    _dio.get('/api/tasks'),
    _dio.get('/api/tasks'),
  ]);
  // Xem terminal: [AuthInterceptor] chỉ xuất hiện 1 lần "BẮT ĐẦU gọi /api/auth/refresh"
}
```

3 request cùng nhận 401 gần như đồng thời — nhờ `QueuedInterceptor` + cờ `_dangRefresh`,
chỉ request đầu tiên thực sự gọi `/api/auth/refresh`, 2 request còn lại chờ và dùng chung
kết quả.

## 6. Trường hợp refresh token cũng hết hạn

Khi `_lamMoiToken()` ném lỗi (refresh token cũng hết hạn/bị thu hồi),
`AuthInterceptor.onError` gọi `_authLocal.clearAuthInfo()` xóa sạch token — phần còn lại
là để UI lắng nghe tình huống này (VD qua `BlocListener` toàn cục ở gốc app) và tự điều
hướng về `/login` bằng `go_router` đã học Tuần 2 Ngày 2.

## Bài tập

**Bài 3 (đã có sẵn — `bai3_auth_interceptor.dart`):** Đối chiếu `AuthInterceptor` với
nguyên tắc `QueuedInterceptor` + cờ `_dangRefresh`, xác nhận đăng ký đúng vào `Dio` chính
qua `taoDioChinh(...)`.

**Bài 4 (đã có sẵn — `bai4_bai5_interceptor_demo_screen.dart`):** Đăng nhập trước, bấm
"Giả lập hết hạn", sau đó bấm "Bài 4: Gọi API" — xác nhận request vẫn thành công bình
thường, log xác nhận đúng 1 lần gọi `/api/auth/refresh`.

**Bài 5 (đã có sẵn):** Sau khi giả lập hết hạn, bấm "Bài 5: 3 API song song" — xác nhận cả
3 API đều thành công, và log chỉ hiện đúng 1 dòng "BẮT ĐẦU gọi /api/auth/refresh".

## Code Example (repo `flutter.git`)

> Xem nguyên các file tại branch `flutter_secure_storage+Dio-Interceptor`:
> [`bai3_auth_interceptor.dart`](<https://github.com/SangLeSoftZ/flutter/blob/flutter_secure_storage+Dio-Interceptor/testflutter/lib/tuan3_ngay8/bai3_auth_interceptor.dart>) ·
> [`bai4_bai5_interceptor_demo_screen.dart`](<https://github.com/SangLeSoftZ/flutter/blob/flutter_secure_storage+Dio-Interceptor/testflutter/lib/tuan3_ngay8/bai4_bai5_interceptor_demo_screen.dart>)

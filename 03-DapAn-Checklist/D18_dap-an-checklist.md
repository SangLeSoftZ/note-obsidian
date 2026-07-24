---
title: "Ngày 18 / Tuần 4 - Ngày 2 — Đáp án gợi ý & Checklist hoàn thành"
track: chung
day: 18
topic: dap-an-checklist
tags: [dap-an, checklist]
source_goc: Tuan4-Ngay2-SecureStorage-DioInterceptor-Scheduled-Async.md
---

# Ngày 18 / Tuần 4 - Ngày 2 — Đáp án gợi ý & Checklist hoàn thành

## ĐÁP ÁN GỢI Ý

<details><summary>Bài 1 — SecureTokenStorage: lưu/đọc/xóa token qua Keystore/Keychain (xem D18_secure-storage-token.md)</summary>

Đã có sẵn trong repo tại `lib/tuan3_ngay8/bai1_secure_storage_screen.dart`:

```dart
class SecureTokenStorage {
  final _storage = const FlutterSecureStorage(
    aOptions: AndroidOptions(encryptedSharedPreferences: true),
  );
  Future<void> luuTokens({required String accessToken, required String refreshToken}) async {
    await _storage.write(key: _khoaAccessToken, value: accessToken);
    await _storage.write(key: _khoaRefreshToken, value: refreshToken);
  }
}
```

Kiểm chứng: nhập token demo, bấm Lưu → Đọc lại → Xóa, quan sát log xác nhận đúng hành vi.
</details>

<details><summary>Bài 2 — Không cần migrate, AuthLocalDataSource đã dùng SecureStorage sẵn (xem D18_secure-storage-token.md)</summary>

Khác với kịch bản migrate trong tài liệu gốc, `login/data/datasources/auth_local_datasource.dart`
**đã dùng `flutter_secure_storage` từ trước**, không có code Hive/`shared_preferences` cần
chuyển đổi:

```dart
class AuthLocalDataSource {
  final FlutterSecureStorage _storage;
  Future<bool> isLoggedIn() async {
    final refreshToken = await getRefreshToken();
    return refreshToken != null && refreshToken.isNotEmpty;
  }
}
```

Kiểm chứng: đăng nhập thật, xác nhận `isLoggedIn()` trả `true` dựa trên refresh token còn
hạn — không phụ thuộc access token.
</details>

<details><summary>Bài 3 — AuthInterceptor kế thừa QueuedInterceptor (xem D18_dio-interceptor-auto-refresh.md)</summary>

Đã có sẵn trong repo tại `lib/tuan3_ngay8/bai3_auth_interceptor.dart`:

```dart
class AuthInterceptor extends QueuedInterceptor {
  @override
  Future<void> onRequest(RequestOptions options, RequestInterceptorHandler handler) async {
    final token = await _authLocal.getToken();
    if (token != null) options.headers['Authorization'] = 'Bearer $token';
    handler.next(options);
  }

  @override
  Future<void> onError(DioException err, ErrorInterceptorHandler handler) async {
    if (err.response?.statusCode != 401) return handler.next(err);
    final tokenMoi = await _lamMoiToken();
    err.requestOptions.headers['Authorization'] = 'Bearer $tokenMoi';
    handler.resolve(await _dio.fetch(err.requestOptions));
  }
}
```

Đăng ký qua `taoDioChinh(authLocal)`, gắn `dio.interceptors.add(AuthInterceptor(dio, authLocal))`.
</details>

<details><summary>Bài 4 — Test tự refresh sau khi giả lập hết hạn (xem D18_dio-interceptor-auto-refresh.md)</summary>

Đã có sẵn trong repo tại `bai4_bai5_interceptor_demo_screen.dart`:

```dart
Future<void> _giamLapHetHan() async {
  await _authLocal.deleteAccessTokenOnly(); // xóa CHỈ access token, giữ refresh token
}
```

Kiểm chứng: đăng nhập → bấm "Giả lập hết hạn" → bấm "Bài 4: Gọi API" — xác nhận request
vẫn thành công, log chỉ hiện đúng 1 lần gọi `/api/auth/refresh`.
</details>

<details><summary>Bài 5 — 3 API song song, refresh chỉ gọi 1 lần (xem D18_dio-interceptor-auto-refresh.md)</summary>

Đã có sẵn trong repo:

```dart
Future<void> _goiSongSong() async {
  final results = await Future.wait([
    _dio.get('/api/tasks'), _dio.get('/api/tasks'), _dio.get('/api/tasks'),
  ]);
}
```

Kiểm chứng: sau khi giả lập hết hạn, bấm "Bài 5: 3 API song song" — cả 3 đều thành công,
log terminal chỉ hiện đúng 1 dòng "BẮT ĐẦU gọi /api/auth/refresh" nhờ `QueuedInterceptor` +
cờ `_dangRefresh`.
</details>

<details><summary>Bài 6 — @Scheduled quét Task định kỳ (xem D18_scheduled-async-task-scheduler.md)</summary>

Đã có sẵn trong repo tại `scheduler/TaskQuaHanScheduler.java` (điều chỉnh theo schema
thật — quét theo `trangThai` thay vì `ngayHetHan`):

```java
@Scheduled(cron = "0 */1 * * * *") // test mỗi 1 phút, production đổi thành mỗi ngày
public void quetTaskChuaHoanThanh() {
    List<Task> chuaHoanThanh = taskRepository.findAll().stream()
        .filter(t -> !"HOAN_THANH".equals(t.getTrangThai())).toList();
    System.out.printf("=== [Scheduler] Quét định kỳ: có %d Task chưa hoàn thành%n", chuaHoanThanh.size());
}
```

Kiểm chứng: chạy server, đợi tối đa 1 phút, xác nhận log tự động xuất hiện không cần
request nào.
</details>

<details><summary>Bài 7 — @Async gửi thông báo, đo thời gian phản hồi (xem D18_scheduled-async-task-scheduler.md)</summary>

Đã có sẵn trong repo tại `ThongBaoService.guiThongBaoTaoTask()`, gọi từ
`TaskServiceImpl.create()` (đúng ví dụ KHÔNG self-invocation):

```java
@Async
public void guiThongBaoTaoTask(Task task) {
    Thread.sleep(2000); // giả lập gửi email
}
```

```java
// Trong TaskServiceImpl.create(), SAU khi save():
thongBaoService.guiThongBaoTaoTask(task); // gọi từ bean khác — @Async hoạt động đúng
return task; // trả về NGAY, không chờ 2 giây
```

Kiểm chứng: gọi `POST /api/tasks`, đo thời gian phản hồi — trả về nhanh, không đợi đủ 2
giây; log `[Async - ThongBao]` xuất hiện trễ hơn với tên thread khác.
</details>

<details><summary>Bài 8 — Tự kiểm chứng self-invocation làm @Async mất tác dụng (nâng cao, tự làm)</summary>

Chưa có sẵn trong repo (repo cố tình viết đúng, không viết ví dụ sai) — tự thử: thêm 1
method `@Async` ngay trong `TaskServiceImpl`, gọi bằng `this.method(...)` thay vì qua
`ThongBaoService`:

```java
@Async
public void guiThongBaoNoiBo(Task task) { ... }

public Task create(TaskRequest dto) {
    ...
    this.guiThongBaoNoiBo(task); // self-invocation — @Async KHÔNG có tác dụng
    return task;
}
```

Kiểm chứng: gọi `POST /api/tasks`, xác nhận response bị **chặn đủ 2 giây** dù có `@Async`
— do lời gọi `this.method()` không đi qua Spring AOP Proxy.
</details>

---

## ✅ CHECKLIST HOÀN THÀNH

- [ ] Access token/refresh token lưu qua `flutter_secure_storage` (Keystore/Keychain), không còn dạng plain text
- [ ] Hiểu vì sao `isLoggedIn()` nên dựa trên refresh token còn hạn, không phải access token
- [ ] `AuthInterceptor` tự gắn token vào mọi request, tự refresh khi 401, tự thử lại request cũ
- [ ] Xác nhận `/api/auth/refresh` chỉ gọi đúng 1 lần dù nhiều request cùng bị 401 đồng thời
- [ ] Biết cách tự tạo tình huống "access token hết hạn" để test mà không cần đợi thời gian thật
- [ ] Tác vụ `@Scheduled` chạy đúng lịch, biết cách điều chỉnh logic quét khi schema thật khác ví dụ lý thuyết
- [ ] `@Async` áp dụng đúng — gọi từ bean khác, không chặn response của request gốc
- [ ] Giải thích và tự kiểm chứng được vì sao `@Async` không hoạt động khi gọi từ trong cùng 1 class (self-invocation)
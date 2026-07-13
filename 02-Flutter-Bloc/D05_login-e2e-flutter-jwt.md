---
title: "Ngày 5 (Tối) — Kết nối đăng nhập Flutter gọi API JWT end-to-end"
track: flutter
day: 5
topic: auth-e2e
tags: [flutter, jwt, login, secure-storage, interceptor]
source_goc: Ngay5_TaiLieu.md (buổi tối)
---

# Ngày 5 (Tối) — Kết nối đăng nhập Flutter gọi API JWT end-to-end

# BUỔI TỐI — Kết nối đăng nhập Flutter gọi API JWT end-to-end

### Luồng đầy đủ khi người dùng bấm nút "Đăng nhập"

```
LoginScreen (UI)
     │ context.read<LoginCubit>().dangNhap(user, pass)
     ▼
LoginCubit (Presentation) — emit(LoginLoading())
     │ loginUseCase.thucThi(user, pass)
     ▼
LoginUseCase (Domain)
     │ authRepository.dangNhap(user, pass)
     ▼
AuthRepositoryImpl (Data) — implement interface ở Domain
     │ remoteDataSource.dangNhap(user, pass)
     ▼
AuthRemoteDataSource (Data) — dio.post('/auth/login', ...)
     │
     ▼
Spring Boot API /api/auth/login  →  trả về { "token": "eyJ..." }
     │
     ▼  (đi ngược lại từng tầng)
LoginCubit — lưu token, emit(LoginSuccess())
     ▼
LoginScreen — điều hướng qua màn hình chính
```

### Code `LoginCubit` hoàn chỉnh

```dart
class LoginCubit extends Cubit<LoginState> {
  final LoginUseCase loginUseCase;
  LoginCubit(this.loginUseCase) : super(LoginInitial());

  Future<void> dangNhap(String username, String password) async {
    emit(LoginLoading());
    try {
      final user = await loginUseCase.thucThi(username, password);
      emit(LoginSuccess(user));
    } catch (e) {
      emit(LoginFailure("Sai tài khoản hoặc mật khẩu"));
    }
  }
}
```

### Lưu token an toàn — `flutter_secure_storage`

**Vì sao không lưu token bằng biến thường (mất khi tắt app) hay `SharedPreferences` (lưu dạng text thô, không mã hóa)?** Vì token là dữ liệu nhạy cảm — nếu thiết bị bị root/jailbreak, dữ liệu `SharedPreferences` có thể đọc được dễ dàng. `flutter_secure_storage` lưu vào **Keychain (iOS)** / **Keystore (Android)** — nơi được hệ điều hành mã hóa bảo vệ.

```yaml
dependencies:
  flutter_secure_storage: ^9.0.0
```

```dart
class TokenStorage {
  final storage = FlutterSecureStorage();

  Future<void> luuToken(String token) async {
    await storage.write(key: 'jwt_token', value: token);
  }

  Future<String?> layToken() async {
    return await storage.read(key: 'jwt_token');
  }

  Future<void> xoaToken() async {
    await storage.delete(key: 'jwt_token');
  }
}
```

### Tự động gắn token vào mọi request sau đăng nhập — dùng lại Interceptor đã học Ngày 4

```dart
dio.interceptors.add(InterceptorsWrapper(
  onRequest: (options, handler) async {
    final token = await tokenStorage.layToken();
    if (token != null) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    return handler.next(options);
  },
));
```

Đây chính là lý do Interceptor được nhấn mạnh ở tài liệu đào sâu Ngày 4 — nó giải quyết **chính xác** vấn đề này: gắn token tự động, không cần lặp code ở từng nơi gọi API.

### Bài tập buổi tối (tổng hợp cả ngày — bài tập chính)

**Bài 5:** Ghép toàn bộ chuỗi lại: `LoginScreen` (nhập username/password, bấm nút) → `LoginCubit` (lấy từ `get_it`) → `LoginUseCase` → `AuthRepositoryImpl` → `AuthRemoteDataSource` (Dio) → gọi API `/api/auth/login` thật (đã viết ở buổi sáng). Khi đăng nhập thành công: lưu token bằng `flutter_secure_storage`, hiện thông báo/điều hướng màn hình.

**Bài 6 (kiểm tra hiểu bài):** Thử **cố tình** gọi trực tiếp `AuthRepositoryImpl` từ `LoginScreen` (bỏ qua `LoginCubit` và `LoginUseCase`) — code có chạy được không? Giải thích tại sao đây là cách làm **sai nguyên tắc** dù vẫn chạy được (gợi ý: nghĩ về việc test và tách trách nhiệm đã học).

---

## CHECKLIST HOÀN THÀNH NGÀY 5

- [ ] API `/api/auth/login` trả về JWT token hợp lệ khi đúng tài khoản
- [ ] Giải thích được vì sao JWT "stateless", khác Session ở điểm nào
- [ ] Hiểu đúng Dependency Rule của Clean Architecture: Domain không phụ thuộc Data/Presentation
- [ ] Setup `get_it` chạy được, inject đủ chuỗi Repository → UseCase → Cubit
- [ ] Đăng nhập từ Flutter gọi API thật thành công, nhận và lưu được token
- [ ] Token được tự động gắn vào request tiếp theo qua Interceptor

## LINK TÀI LIỆU THAM KHẢO

- Spring Security chính thức: https://docs.spring.io/spring-security/reference/index.html
- JWT giải thích trực quan (có thể tự tạo/giải mã token thử): https://jwt.io/introduction
- Baeldung — Spring Boot + JWT hướng dẫn đầy đủ: https://www.baeldung.com/spring-security-oauth-jwt
- Clean Architecture nguyên bản (Robert C. Martin — bài blog gốc): https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html
- Clean Architecture áp dụng cho Flutter (ví dụ thực tế rất chi tiết): https://resocoder.com/flutter-clean-architecture-tdd/
- get_it package: https://pub.dev/packages/get_it
- flutter_secure_storage: https://pub.dev/packages/flutter_secure_storage

## 💻 Code Example (repo `Code`)

> ⚠️ Repo [`SangLeSoftZ/Code`](https://github.com/SangLeSoftZ/Code) **hiện chỉ chứa code Java/Spring Boot**
> (package `com.example.demo`), chưa có project Flutter/Dart nào. Khi có repo code Flutter
> riêng, bổ sung link + snippet vào đúng mục này theo cùng quy ước (xem `QUY_UOC_DAT_TEN.md`).

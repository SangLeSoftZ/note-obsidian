---
title: "Ngày 5 (Chiều) — Clean Architecture Flutter + Repository Pattern + DI (get_it)"
track: flutter
day: 5
topic: clean-architecture
tags: [flutter, clean-architecture, repository, get_it, di]
source_goc: Ngay5_TaiLieu.md (buổi chiều)
---

# Ngày 5 (Chiều) — Clean Architecture Flutter + Repository Pattern + DI (get_it)

# BUỔI CHIỀU — Clean Architecture Flutter + Repository Pattern + DI (get_it)

### 1. Vấn đề Clean Architecture giải quyết

Nếu code gọi API trực tiếp ngay trong Widget (`http.get(...)` gõ thẳng trong `build()`), bạn gặp vấn đề: khó test, khó thay đổi nguồn dữ liệu (ví dụ đổi từ API sang cache local), code UI và logic nghiệp vụ trộn lẫn.

Clean Architecture giải quyết bằng cách chia thành **3 tầng độc lập**, tầng trong không biết gì về tầng ngoài:

```
┌─────────────────────────────────────┐
│  PRESENTATION (UI + Cubit/Bloc)      │  <- người dùng thấy, tương tác
├─────────────────────────────────────┤
│  DOMAIN (Entity + UseCase + Repository interface) │  <- "luật chơi" nghiệp vụ, KHÔNG phụ thuộc Flutter/API cụ thể
├─────────────────────────────────────┤
│  DATA (Model + DataSource + Repository implementation) │  <- lấy dữ liệu thật (API, database local)
└─────────────────────────────────────┘
```

**Nguyên tắc quan trọng nhất — Dependency Rule:** tầng **Domain** không được biết gì về tầng **Data** hay **Presentation**. Chỉ có chiều phụ thuộc đi từ ngoài vào trong (`Presentation` → `Domain` ← `Data`), không bao giờ ngược lại.

### 2. Domain Layer — "luật chơi" thuần túy, không dính Flutter/API

```dart
// domain/entities/user.dart — Entity: object nghiệp vụ thuần túy, KHÔNG có fromJson/toJson
class User {
  final String id;
  final String username;
  User({required this.id, required this.username});
}

// domain/repositories/auth_repository.dart — chỉ là INTERFACE (hợp đồng), chưa có logic thật
abstract class AuthRepository {
  Future<User> dangNhap(String username, String password);
}

// domain/usecases/login_usecase.dart — 1 "hành động nghiệp vụ" cụ thể
class LoginUseCase {
  final AuthRepository repository;
  LoginUseCase(this.repository);

  Future<User> thucThi(String username, String password) {
    return repository.dangNhap(username, password);
  }
}
```

**Vì sao cần `UseCase` riêng, không gọi thẳng `Repository` từ Cubit?** Vì `UseCase` đại diện cho **đúng 1 hành động nghiệp vụ** (ví dụ "đăng nhập"), có thể chứa thêm logic nghiệp vụ riêng (validate, kết hợp nhiều repository) mà không làm Cubit phình to. Với app nhỏ, đôi khi có thể bỏ qua UseCase và gọi thẳng Repository — nhưng biết cấu trúc chuẩn giúp bạn mở rộng dễ dàng khi app lớn lên.

### 3. Data Layer — nơi thực sự "nói chuyện" với API

```dart
// data/models/user_model.dart — Model: bản "mở rộng" của Entity, CÓ fromJson/toJson
class UserModel extends User {
  UserModel({required super.id, required super.username});

  factory UserModel.fromJson(Map<String, dynamic> json) {
    return UserModel(id: json['id'], username: json['username']);
  }
}

// data/datasources/auth_remote_datasource.dart — nơi gọi API thật
class AuthRemoteDataSource {
  final Dio dio;
  AuthRemoteDataSource(this.dio);

  Future<Map<String, dynamic>> dangNhap(String username, String password) async {
    final response = await dio.post('/auth/login', data: {
      'username': username, 'password': password,
    });
    return response.data;
  }
}

// data/repositories/auth_repository_impl.dart — hiện thực hóa interface ở Domain
class AuthRepositoryImpl implements AuthRepository {
  final AuthRemoteDataSource remoteDataSource;
  AuthRepositoryImpl(this.remoteDataSource);

  @override
  Future<User> dangNhap(String username, String password) async {
    final data = await remoteDataSource.dangNhap(username, password);
    return UserModel.fromJson(data);
  }
}
```

**Vì sao tách riêng `Entity` (Domain) và `Model` (Data)?** Vì `Entity` là "khái niệm nghiệp vụ thuần túy" (không quan tâm dữ liệu tới từ đâu), còn `Model` là "cách dữ liệu trông như thế nào khi tới từ API cụ thể" (có `fromJson`). Nếu ngày mai bạn đổi từ REST API sang GraphQL, chỉ cần sửa `Model`/`DataSource`, `Entity` và toàn bộ `Domain`/`Presentation` phía trên **không cần sửa gì**.

### 4. Dependency Injection với `get_it` — vì sao cần, dùng thế nào?

Ở phần Spring Boot bạn đã học DI qua `@Autowired`/constructor injection. Flutter không có "Container" tích hợp sẵn như Spring — `get_it` là 1 package phổ biến đóng vai trò tương tự IoC Container.

```dart
// injection_container.dart
final getIt = GetIt.instance;

void setupDependencies() {
  // Đăng ký từ tầng trong ra ngoài
  getIt.registerLazySingleton(() => Dio(BaseOptions(baseUrl: "http://10.0.2.2:8080/api")));
  getIt.registerLazySingleton(() => AuthRemoteDataSource(getIt()));
  getIt.registerLazySingleton<AuthRepository>(() => AuthRepositoryImpl(getIt()));
  getIt.registerLazySingleton(() => LoginUseCase(getIt()));
}
```

Gọi 1 lần trong `main()`:
```dart
void main() {
  setupDependencies();
  runApp(MyApp());
}
```

Dùng ở bất kỳ đâu (thường trong `BlocProvider`):
```dart
BlocProvider(
  create: (_) => LoginCubit(getIt<LoginUseCase>()),
  child: LoginScreen(),
)
```

**`registerLazySingleton` nghĩa là gì?** "Lazy" = chỉ thực sự tạo object khi **lần đầu tiên** có ai gọi `getIt<...>()`, không tạo ngay lúc `setupDependencies()` chạy. "Singleton" = chỉ tạo **1 lần duy nhất**, các lần gọi sau dùng lại cùng 1 instance — giống `singleton` scope đã học ở Spring Ngày 2.

### Bài tập buổi chiều

**Bài 3:** Tạo cấu trúc thư mục Clean Architecture cho tính năng "Login": `domain/entities`, `domain/repositories`, `domain/usecases`, `data/models`, `data/datasources`, `data/repositories`. Viết đủ các file như ví dụ trên (chưa cần chạy được, chỉ cần đúng cấu trúc và các class biên dịch được).

**Bài 4:** Setup `get_it`, đăng ký đủ chuỗi dependency cho `LoginUseCase` như ví dụ. Thử gọi `getIt<LoginUseCase>()` ở 1 chỗ bất kỳ, in ra kiểm tra không bị lỗi.

---

## 💻 Code Example (repo `Code`)

> ⚠️ Repo [`SangLeSoftZ/Code`](https://github.com/SangLeSoftZ/Code) **hiện chỉ chứa code Java/Spring Boot**
> (package `com.example.demo`), chưa có project Flutter/Dart nào. Khi có repo code Flutter
> riêng, bổ sung link + snippet vào đúng mục này theo cùng quy ước (xem `QUY_UOC_DAT_TEN.md`).

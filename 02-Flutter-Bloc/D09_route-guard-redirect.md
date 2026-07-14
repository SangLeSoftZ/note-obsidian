---
title: "Ngày 9 (Tuần 2 - Ngày 2, Chiều) — Route Guard: Cơ chế Redirect"
track: flutter
day: 9
topic: route-guard-redirect
tags: [flutter, go_router, route-guard, redirect, changenotifier, refreshlistenable]
source_goc: Tuan2_Ngay2_TaiLieu.md
---

# Ngày 9 (Tuần 2 - Ngày 2, Chiều) — Route Guard: Cơ chế Redirect

## 1. Vấn đề cần giải quyết

`app_router.dart` buổi sáng chưa có bảo vệ gì — ai chưa đăng nhập vẫn gõ thẳng `/home`
là vào được. Route Guard nghĩa là: **trước khi vào 1 route, kiểm tra điều kiện (đã login
chưa) — nếu không đạt thì tự động đá về route khác** (ở đây là `/login`).

## 2. Vì sao không thể `await` thẳng trong `redirect()`

Đây là điểm dễ vướng nhất khi mới học go_router. Trích đúng comment trong code thật
của bạn — `lib/tuan2_ngay2_guard/auth_state.dart`:

```dart
// Tại sao cần class này thay vì đọc thẳng SecureStorage?
//   - go_router redirect() là ĐỒNG BỘ (synchronous)
//   - SecureStorage.read() là BẤT ĐỒNG BỘ (async/Future)
//   - Không thể await trong redirect() → cần 1 biến bool trong RAM
```

Nói cách khác:
- Hàm `redirect` của `GoRouter` bắt buộc phải trả kết quả **ngay lập tức** (không phải
  `Future`), vì nó chạy trong quá trình build lại giao diện.
- Nhưng việc đọc token thường phải qua `SecureStorage`/`SharedPreferences` — đây là thao
  tác **bất đồng bộ** (mất thời gian đọc từ đĩa).
- Giải pháp: đọc token **một lần duy nhất lúc khởi động app**, lưu kết quả vào 1 biến
  `bool` nằm sẵn trong RAM (`_daDangNhap`). Từ đó `redirect()` chỉ cần đọc biến RAM này —
  luôn đồng bộ, không cần `await`.

## 3. `AuthState` — cầu nối giữa async storage và sync redirect

```dart
class AuthState extends ChangeNotifier {
  bool _daDangNhap = false;
  bool get daDangNhap => _daDangNhap;

  // Gọi khi đọc token từ SecureStorage xong (lúc khởi động)
  void khoiTao(bool daCoToken) {
    _daDangNhap = daCoToken;
    // Không notifyListeners() ở đây — GoRouter chưa init xong
  }

  // Gọi sau login thành công hoặc logout
  void capNhat(bool giaTri) {
    _daDangNhap = giaTri;
    notifyListeners(); // → go_router tự chạy lại redirect()
  }
}

final authState = AuthState(); // Singleton — dùng chung toàn app
```

`AuthState` kế thừa `ChangeNotifier` — đúng class nền tảng bạn đã học ở phần state
management cơ bản (trước cả Bloc/Cubit). Ở đây nó không dùng để build UI, mà dùng làm
**"cái chuông báo"**: mỗi khi trạng thái đăng nhập đổi (`capNhat()` được gọi),
`notifyListeners()` bắn tín hiệu ra ngoài.

## 4. `refreshListenable` — go_router lắng nghe cái chuông đó

```dart
final GoRouter appRouterGuard = GoRouter(
  initialLocation: '/home', // thử vào /home thẳng → bị redirect về /login
  refreshListenable: authState, // lắng nghe AuthState
  redirect: (context, state) {
    final daDangNhap = authState.daDangNhap;
    final dangOManLogin = state.matchedLocation == '/login';

    if (!daDangNhap && !dangOManLogin) return '/login'; // chưa login → đá về /login
    if (daDangNhap && dangOManLogin) return '/home';    // đã login mà còn ở /login → vào /home
    return null; // không chặn gì cả — cho đi tiếp
  },
  routes: [ /* ...giống buổi sáng... */ ],
);
```

Luồng hoạt động đầy đủ:

1. App khởi động → đọc token từ SecureStorage (async) → gọi `authState.khoiTao(...)`.
2. `GoRouter` được tạo với `refreshListenable: authState` — nghĩa là **mỗi khi `authState`
   gọi `notifyListeners()`, go_router tự động chạy lại hàm `redirect()`** mà không cần ai
   gọi thủ công.
3. Người dùng đăng nhập thành công → `login_screen_guard.dart` gọi `authState.capNhat(true)`
   (thay vì `context.go('/home')` như buổi sáng) → `notifyListeners()` bắn ra →
   `redirect()` chạy lại → thấy `daDangNhap == true` và đang ở `/login` → tự trả về `/home`.
4. Ngược lại khi logout, gọi `authState.capNhat(false)` → `redirect()` chạy lại → đá thẳng
   về `/login`.

Đây chính là điểm khác biệt quan trọng nhất so với router buổi sáng — trích comment
trong `login_screen_guard.dart`:

```dart
// KHÁC với tuan2_ngay2/login_screen.dart (buổi sáng):
//   Buổi sáng: context.go('/home') trực tiếp
//   Buổi chiều: authState.capNhat(true) → go_router tự redirect
//     Không cần gọi context.go() nữa — router tự xử lý
```

## 5. Vì sao thiết kế theo cách này (thay vì tự `if` kiểm tra token ở mỗi màn hình)

- **Tập trung 1 chỗ**: toàn bộ logic "ai được vào đâu" nằm trong `redirect()`, không rải
  rác `if (!isLoggedIn) Navigator.push(LoginScreen())` ở từng màn hình.
- **Tự động đồng bộ mọi nơi**: chỉ cần gọi `authState.capNhat(...)` ở đúng 1 chỗ (sau
  login/logout), toàn bộ router tự cập nhật theo — không cần nhớ gọi redirect thủ công ở
  từng nơi có thể thay đổi trạng thái đăng nhập.
- **Không có khoảng trống bảo mật do quên check**: kể cả khi người dùng cố tình gõ thẳng
  URL `/home` (trên web) hoặc app bị deep-link vào thẳng 1 route được bảo vệ, `redirect()`
  luôn chạy trước khi build màn hình đó.

## 6. Bài tập

**Bài 3 (D09_dap-an-checklist.md):** Thêm route `/tasks/:id` vào danh sách "cần đăng
nhập mới được vào" (hiện tại route này không bị chặn — kiểm tra lại điều kiện `redirect`
để xác nhận đúng/sai, sửa nếu cần).

**Bài 4:** Giải thích bằng lời: điều gì xảy ra nếu quên không gọi `notifyListeners()`
trong hàm `capNhat()`? Người dùng đăng nhập xong có tự động được chuyển sang `/home`
không?

##  Code Example (repo `flutter.git`)

> Xem nguyên các file tại branch `tuan2`:
> [`app_router_guard.dart`](https://github.com/SangLeSoftZ/flutter/blob/tuan2/testflutter/lib/tuan2_ngay2_guard/app_router_guard.dart) ·
> [`auth_state.dart`](https://github.com/SangLeSoftZ/flutter/blob/tuan2/testflutter/lib/tuan2_ngay2_guard/auth_state.dart) ·
> [`login_screen_guard.dart`](https://github.com/SangLeSoftZ/flutter/blob/tuan2/testflutter/lib/tuan2_ngay2_guard/login_screen_guard.dart)

##  Tài liệu tham khảo

- [go_router — Redirection](https://pub.dev/documentation/go_router/latest/topics/Redirection-topic.html)
- [ChangeNotifier — Flutter API docs](https://api.flutter.dev/flutter/foundation/ChangeNotifier-class.html)
- [go_router — Understanding refreshListenable (GitHub wiki thảo luận)](https://github.com/flutter/flutter/issues/95034)


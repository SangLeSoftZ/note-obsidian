---
title: "Ngày 9 (Tuần 2 - Ngày 2, Sáng) — go_router: Declarative Navigation"
track: flutter
day: 9
topic: go_router-declarative-navigation
tags: [flutter, go_router, navigation, declarative]
source_goc: Tuan2_Ngay2_TaiLieu.md
---

# Ngày 9 (Tuần 2 - Ngày 2, Sáng) — go_router: Declarative Navigation

## 1. Vấn đề của Navigator mặc định

Navigator gốc của Flutter (imperative) dùng `Navigator.push(context, MaterialPageRoute(...))`
ở bất kỳ đâu trong code muốn điều hướng. Nhược điểm:

- Bản đồ route (route map) không nằm ở 1 chỗ — muốn biết app có bao nhiêu màn hình phải
  tìm rải rác trong toàn bộ code.
- Không hỗ trợ tốt deep link / URL (quan trọng với Flutter Web).
- Khó viết logic redirect tập trung (vd: chưa login thì luôn về trang Login) — đây là
  lý do chính dẫn tới bài **Route Guard** buổi chiều.

## 2. go_router giải quyết bằng cách nào

**Declarative** = bạn khai báo "app có những route nào, path nào" ở 1 nơi duy nhất
(`GoRouter`), Flutter tự lo phần điều hướng thực tế thay vì bạn tự tay `push` từng bước.

Code thật trong repo — `lib/tuan2_ngay2/app_router.dart`:

```dart
final GoRouter appRouter = GoRouter(
  initialLocation: '/login',
  routes: [
    GoRoute(
      path: '/login',
      builder: (context, state) => const LoginScreen(),
    ),
    GoRoute(
      path: '/home',
      builder: (context, state) => const HomeScreen(),
    ),
    GoRoute(
      path: '/tasks/:id',
      builder: (context, state) {
        final id = state.pathParameters['id']!;
        // Nhận Task object từ extra nếu có (truyền từ HomeScreen)
        final task = state.extra as Task?;
        return TaskDetailScreen(taskId: id, task: task);
      },
    ),
  ],
);
```

Điểm quan trọng:
- **`initialLocation`**: route đầu tiên khi app khởi động — ở đây là `/login`.
- **`path: '/tasks/:id'`**: `:id` là **path parameter**, lấy ra bằng
  `state.pathParameters['id']`. Dùng cho dữ liệu bắt buộc phải có trong URL (để deep-link
  vẫn hoạt động đúng, vd mở thẳng `/tasks/42` từ 1 link ngoài app).
- **`state.extra`**: dùng để truyền nguyên 1 object Dart (ở đây là `Task`) mà **không** đi
  qua URL — tiện khi bạn đã có sẵn object trong RAM và không muốn parse lại từ ID. Nhược
  điểm: `extra` sẽ mất nếu người dùng vào thẳng route bằng deep link (vì lúc đó không ai
  set `extra` cả) — đây là lý do `task` được khai báo kiểu `Task?` (nullable), và
  `TaskDetailScreen` cần có logic tự fetch lại theo `taskId` nếu `task == null`.

Để `MaterialApp` dùng `GoRouter` này thay vì Navigator mặc định, khai báo trong `main.dart`:

```dart
MaterialApp.router(
  routerConfig: appRouter,
  // không dùng `home:` hay `routes:` của MaterialApp nữa
)
```

## 3. `go` vs `push` — khác nhau ở đâu

| | `context.go('/home')` | `context.push('/home')` |
|---|---|---|
| Cơ chế | **Thay thế** vị trí hiện tại trong stack điều hướng | **Thêm** 1 trang mới lên trên stack |
| Nút Back | Không quay lại được trang trước đó (vì đã bị thay) | Quay lại được trang trước |
| Khi nào dùng | Chuyển "trạng thái chính" của app — vd sau khi login xong chuyển sang Home (không muốn bấm Back quay lại màn Login) | Điều hướng "đào sâu" — vd từ danh sách Task bấm vào 1 Task để xem chi tiết (bấm Back phải quay lại được danh sách) |
| Ảnh hưởng URL (web) | Thay URL, không cộng dồn lịch sử | Thay URL, có cộng dồn lịch sử |

Quy tắc ghi nhớ nhanh: **"go" = đi tới nơi khác hẳn (không cần nhớ đường về)**,
**"push" = đi sâu thêm 1 bước (vẫn cần đường quay lại)**.

Trong repo, `HomeScreen` → `TaskDetailScreen` nên dùng `push` (người dùng cần bấm Back
để quay lại danh sách), còn `LoginScreen` → `HomeScreen` sau khi đăng nhập thành công nên
dùng `go` (không muốn Back quay lại màn hình Login).

## 4. Bài tập

**Bài 1 (D09_dap-an-checklist.md):** Thêm 1 route mới `/profile` vào `appRouter`, tạo
`ProfileScreen` đơn giản chỉ hiện `Text('Profile')`. Từ `HomeScreen` thêm 1 nút điều
hướng tới `/profile` bằng `context.push(...)`.

**Bài 2:** Giải thích bằng lời (không cần code): nếu đổi `context.push('/tasks/:id')`
trong `HomeScreen` thành `context.go('/tasks/:id')` thì hành vi nút Back sẽ thay đổi ra
sao? Vì sao không nên dùng `go` ở trường hợp này?

## Code Example (repo `flutter.git`)

> Xem nguyên file tại
> [`lib/tuan2_ngay2/app_router.dart`](https://github.com/SangLeSoftZ/flutter/blob/tuan2/testflutter/lib/tuan2_ngay2/app_router.dart)
> (branch `tuan2`) — cùng track với [[D09_route-guard-redirect]].

##  Tài liệu tham khảo

- [go_router — official docs](https://pub.dev/documentation/go_router/latest/)
- [go_router: Navigation — flutter.dev](https://docs.flutter.dev/ui/navigation)
- [go_router pub.dev package](https://pub.dev/packages/go_router)

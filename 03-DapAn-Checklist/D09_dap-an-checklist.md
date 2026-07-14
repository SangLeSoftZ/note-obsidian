---
title: "Ngày 9 / Tuần 2 - Ngày 2 — Đáp án gợi ý & Checklist hoàn thành"
track: chung
day: 9
topic: dap-an-checklist
tags: [dap-an, checklist]
source_goc: Tuan2_Ngay2_TaiLieu.md
---

# Ngày 9 / Tuần 2 - Ngày 2 — Đáp án gợi ý & Checklist hoàn thành

## ĐÁP ÁN GỢI Ý

<details><summary>Bài 1 — Thêm route /profile vào appRouter (xem D09_go_router-declarative-navigation.md)</summary>

```dart
GoRoute(
  path: '/profile',
  builder: (context, state) => const ProfileScreen(),
),
```

```dart
class ProfileScreen extends StatelessWidget {
  const ProfileScreen({super.key});
  @override
  Widget build(BuildContext context) => const Scaffold(
    body: Center(child: Text('Profile')),
  );
}
```

Trong `HomeScreen`, thêm nút điều hướng bằng `push` (vì đây là đi sâu thêm 1 bước, cần
quay lại được):

```dart
ElevatedButton(
  onPressed: () => context.push('/profile'),
  child: const Text('Xem Profile'),
),
```
</details>

<details><summary>Bài 2 — go vs push khi xem chi tiết Task (lý thuyết, không cần code)</summary>

Nếu đổi `context.push('/tasks/:id')` thành `context.go('/tasks/:id')`: `go` sẽ **thay
thế** vị trí hiện tại trong stack thay vì thêm mới, nên khi bấm nút Back ở
`TaskDetailScreen`, người dùng sẽ **không quay lại được** `HomeScreen` như mong đợi (có
thể thoát hẳn app hoặc về màn hình trước `HomeScreen` trong stack, tuỳ cấu hình). Vì việc
xem chi tiết Task là điều hướng "đào sâu" — cần giữ lại đường quay về danh sách — nên phải
dùng `push`, không dùng `go`.
</details>

<details><summary>Bài 3 — Bảo vệ route /tasks/:id trong Route Guard (xem D09_route-guard-redirect.md)</summary>

Route `/tasks/:id` hiện **đã được bảo vệ gián tiếp** bởi điều kiện chung trong `redirect`:

```dart
if (!daDangNhap && !dangOManLogin) return '/login';
```

Điều kiện này áp dụng cho **mọi route không phải `/login`**, nên `/tasks/:id` cũng bị chặn
nếu chưa đăng nhập — không cần sửa thêm gì. Đây là điểm mạnh của cách thiết kế tập trung:
thêm route mới vào `routes: [...]` không cần nhớ thêm logic bảo vệ riêng, vì `redirect()`
áp dụng chung cho toàn bộ router.
</details>

<details><summary>Bài 4 — Quên gọi notifyListeners() trong capNhat() (lý thuyết, không cần code)</summary>

Nếu bỏ `notifyListeners()` trong `capNhat()`, biến `_daDangNhap` vẫn được cập nhật đúng
giá trị trong RAM, nhưng `go_router` (đang lắng nghe qua `refreshListenable: authState`)
sẽ **không biết** có gì thay đổi — vì tín hiệu duy nhất để `go_router` tự chạy lại
`redirect()` chính là sự kiện `notifyListeners()` từ `ChangeNotifier`. Kết quả: sau khi
đăng nhập thành công, người dùng vẫn bị "kẹt" ở màn hình Login cho tới khi có 1 sự kiện
điều hướng khác (vd rebuild toàn bộ app) vô tình kích hoạt lại `redirect()`.
</details>

<details><summary>Bài 5 — Thiếu @EnableMethodSecurity, USER gọi /api/admin/users (lý thuyết, không cần code)</summary>

`@PreAuthorize` là bảo vệ ở tầng method, chỉ được Spring Security "bật" khi có
`@EnableMethodSecurity`. Nếu xoá annotation này, Spring sẽ **bỏ qua hoàn toàn** mọi
`@PreAuthorize` trong code — coi như nó không tồn tại. Vì `SecurityConfig` chỉ chặn theo
URL bằng `.authorizeHttpRequests(...anyRequest().authenticated())`, nên request tới
`/api/admin/users` với token hợp lệ (dù role là `USER`) vẫn **đi qua được tầng URL bình
thường**, method `layTatCaUser()` vẫn chạy — trả về **200 OK** với toàn bộ danh sách user,
thay vì 403 Forbidden như mong đợi. Đây chính là lỗi bảo mật nghiêm trọng nếu quên dòng
này.
</details>

<details><summary>Bài 6 — Endpoint /api/admin/reports chỉ cho MANAGER (xem D09_preauthorize-role-based-authorization.md)</summary>

```java
@Operation(
    summary     = "Báo cáo nội bộ [MANAGER only]",
    description = "403 nếu role không phải MANAGER",
    security    = @SecurityRequirement(name = "bearerAuth")
)
@PreAuthorize("hasRole('MANAGER')")
@GetMapping("/reports")
public ResponseEntity<?> xemBaoCao() {
    return ResponseEntity.ok(Map.of("message", "Báo cáo dành riêng cho MANAGER"));
}
```

Lưu ý dùng `hasRole('MANAGER')` (không phải `hasAnyRole`) vì chỉ 1 role duy nhất được
phép — kể cả `ADMIN` gọi endpoint này cũng sẽ nhận 403, đúng yêu cầu đề bài.
</details>

---

##  CHECKLIST HOÀN THÀNH

- [ ] Hiểu vì sao `GoRouter` khai báo route tập trung 1 chỗ thay vì gọi `Navigator.push` rải rác
- [ ] Phân biệt được `path parameter` (`:id`) và `extra` — khi nào dùng cái nào
- [ ] Giải thích được khác biệt `go` vs `push` bằng ví dụ cụ thể trong app (Login→Home vs Home→TaskDetail)
- [ ] Giải thích được vì sao `redirect()` của go_router phải đồng bộ, không thể `await`
- [ ] Vẽ lại được luồng: login → `authState.capNhat(true)` → `notifyListeners()` → `redirect()` chạy lại → vào `/home`
- [ ] Biết chính xác 2 điều kiện bắt buộc để `@PreAuthorize` hoạt động (`@EnableMethodSecurity` + filter set `SecurityContext` đúng thứ tự)
- [ ] Phân biệt được `hasRole('ADMIN')` (tự thêm `ROLE_`) và `hasAuthority('ADMIN')` (so khớp chính xác)
- [ ] Phân biệt được khi nào trả 401, khi nào trả 403 — đúng theo `AuthenticationException` vs `AccessDeniedException`
- [ ] Đã tự tay thêm được 1 route Flutter mới và 1 endpoint Spring Boot mới có phân quyền


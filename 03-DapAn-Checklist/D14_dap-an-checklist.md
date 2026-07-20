---
title: "Ngày 14 / Tuần 3 - Ngày 3 — Đáp án gợi ý & Checklist hoàn thành"
track: chung
day: 14
topic: dap-an-checklist
tags: [dap-an, checklist]
source_goc: Tuan3-Ngay3-Generics-Mixin-Riverpod-JPAAuditing.md
---

# Ngày 14 / Tuần 3 - Ngày 3 — Đáp án gợi ý & Checklist hoàn thành

## ĐÁP ÁN GỢI Ý

<details><summary>Bài 1 — ApiResponse&lt;T&gt; thay cho response riêng từng loại (xem D16_generics-mixin-extension.md)</summary>

Đã có sẵn trong repo tại `lib/tuan3_ngay3/bai1_generics.dart`:

```dart
class ApiResponse<T> {
  final bool thanhCong;
  final T? duLieu;
  final String? loi;
  final int? statusCode;

  const ApiResponse.ok(this.duLieu) : thanhCong = true, loi = null, statusCode = 200;
  const ApiResponse.loi(this.loi, {this.statusCode}) : thanhCong = false, duLieu = null;
}
```

Áp dụng: `ApiResponse<Task>.ok(task)`, `ApiResponse<Task>.loi('Token hết hạn', statusCode: 401)`,
`ApiResponse<List<Task>>.ok([task1, task2])` — cùng 1 class cho mọi kiểu dữ liệu.
</details>

<details><summary>Bài 2 — Mixin dùng cho 2 class không liên quan (xem D16_generics-mixin-extension.md)</summary>

Đã có sẵn trong repo tại `lib/tuan3_ngay3/bai2_mixin.dart`:

```dart
mixin LoggerMixin {
  void logInfo(String msg) { /* in log kèm timestamp */ }
  void logLoi(String msg)  { /* in log lỗi kèm timestamp */ }
}

class TaskRepository with LoggerMixin { ... }
class LoginFormHelper with ValidatorMixin, LoggerMixin { ... }
class _MixinScreenState extends State<MixinScreen> with LoggerMixin { ... }
```

3 class hoàn toàn không liên quan gì (Repository, Helper, State) đều dùng chung được
`LoggerMixin` — đúng minh chứng cho "chia sẻ hành vi không cần kế thừa".
</details>

<details><summary>Bài 3 — 2+ Extension method cho Task/String/DateTime (xem D16_generics-mixin-extension.md)</summary>

Đã có sẵn trong repo tại `lib/tuan3_ngay3/bai3_extension.dart`:

```dart
extension TaskExtension on Task {
  String get trangThaiHienThi { ... }
  bool get tieuDeDai => tieuDe.length > 20;
}

extension StringExtension on String {
  bool get laEmailHopLe => RegExp(r'^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$').hasMatch(this);
  String get boDau { /* bỏ dấu tiếng Việt */ }
}
```

Áp dụng vào UI: `Text(task.trangThaiHienThi)`, `Icon(task.iconTrangThai, color: task.mauTrangThai)`.
</details>

<details><summary>Bài 4 — StateProvider&lt;int&gt; đơn giản (xem D16_riverpod-vs-cubit.md)</summary>

Đã có sẵn trong repo tại `lib/tuan3_ngay3/bai4_state_provider_screen.dart`:

```dart
final demTaskProvider = StateProvider<int>((ref) => 0);

// Đọc + rebuild
final soLuong = ref.watch(demTaskProvider);
// Ghi, không rebuild tại chỗ gọi
ref.read(demTaskProvider.notifier).state++;
```

Kiểm chứng: chạy `Bai4StateProviderScreen`, bấm Thêm/Bớt/Reset — số hiển thị cập nhật
đúng theo `ref.watch`.
</details>

<details><summary>Bài 5 — Notifier tương đương TaskCubit (xem D16_riverpod-vs-cubit.md)</summary>

Đã có sẵn trong repo tại `lib/tuan3_ngay3/bai5_notifier_screen.dart`:

```dart
class TaskListNotifier extends Notifier<List<Task>> {
  @override
  List<Task> build() => [];

  void themTask(Task task) => state = [...state, task];
}

final taskListProvider = NotifierProvider<TaskListNotifier, List<Task>>(TaskListNotifier.new);
```

Kiểm chứng: chạy `Bai5NotifierScreen`, thêm/xóa/toggle Task và load từ API — so sánh trải
nghiệm viết code với `TaskCubit` tương đương trong project chính.
</details>

<details><summary>Bài 6 — 3 điểm khác biệt Riverpod vs Cubit (đã có sẵn ngay trong UI, xem D16_riverpod-vs-cubit.md)</summary>

Đã có sẵn trong repo, hiển thị trực tiếp trong `Bai5NotifierScreen`:

1. Không cần `BlocProvider` bọc widget — `ProviderScope` 1 lần ở `main()`.
2. `state = [...]` trực tiếp thay vì gọi `emit()` — dễ quên tạo list mới (bẫy thực tế:
   viết nhầm `state.add(task)` vẫn biên dịch được nhưng không kích hoạt rebuild).
3. `ref` không cần `BuildContext` — dùng được trong hàm async bên ngoài widget.
</details>

<details><summary>Bài 7 — @EnableJpaAuditing + @CreatedDate/@LastModifiedDate (xem D16_jpa-auditing-optimistic-locking.md)</summary>

Đã có sẵn trong repo tại `JpaAuditingConfig.java` + `Task.java`:

```java
@Configuration
@EnableJpaAuditing
public class JpaAuditingConfig {}
```

```java
@EntityListeners(AuditingEntityListener.class)
public class Task {
    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
}
```

Kiểm chứng: gọi `POST /api/tasks`, xác nhận response chứa `createdAt`/`updatedAt` có giá
trị đúng mà không set tay trong request body.
</details>

<details><summary>Bài 8 — updatedAt đổi, createdAt giữ nguyên (xem D16_jpa-auditing-optimistic-locking.md)</summary>

Gọi `PUT /api/tasks/{id}` đổi `tieuDe` — do `createdAt` có `@Column(updatable = false)`
và không có setter trong entity, giá trị này **không thể** bị ghi đè dù có cố tình; chỉ
`updatedAt` được Spring tự cập nhật lại theo `@LastModifiedDate`.
</details>

<details><summary>Bài 9 — @Version + mô phỏng xung đột 409 (xem D16_jpa-auditing-optimistic-locking.md)</summary>

Đã có sẵn trong repo tại `Task.java` + `TaskServiceImpl.update()`:

```java
@Version
@Column(name = "phien_ban")
private Integer phienBan = 0;
```

```java
} catch (OptimisticLockingFailureException e) {
    throw new ResponseStatusException(
        HttpStatus.CONFLICT,
        "Task đã bị người khác cập nhật. Vui lòng tải lại và thử lại."
    );
}
```

Kiểm chứng: `GET /api/tasks/{id}` 2 lần (2 request riêng, cùng `phienBan`) → `PUT` request
1 trước (thành công, `phienBan` tăng lên) → `PUT` request 2 (đang cầm `phienBan` cũ) →
nhận **409 Conflict** thay vì ghi đè âm thầm.
</details>

---

## ✅ CHECKLIST HOÀN THÀNH

- [ ] Giải thích được khác biệt `extends` (kế thừa, 1 cha) vs `with` Mixin (nhiều hành vi trộn vào, kể cả với `State`)
- [ ] Áp dụng được ít nhất 1 Extension method và 1 Mixin vào code thật trong project
- [ ] Hiểu vì sao static method cần khai báo lại `<T>` riêng, không dùng chung `<T>` của class
- [ ] Viết được 1 `Notifier`/`StateProvider` Riverpod chạy đúng, tự so sánh được với Cubit tương đương
- [ ] Nêu được ít nhất 1 "bẫy" thực tế của Riverpod so với Cubit (VD: quên tạo list mới khi gán `state`)
- [ ] Task tự động ghi `createdAt`/`updatedAt` mà không cần set tay
- [ ] Hiểu vì sao entity không nên có setter cho field do Auditing/Optimistic Locking tự quản lý
- [ ] Tái hiện và xử lý đúng lỗi xung đột cập nhật đồng thời (409) nhờ `@Version`


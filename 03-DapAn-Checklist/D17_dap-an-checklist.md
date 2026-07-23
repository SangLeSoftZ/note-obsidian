---
title: "Ngày 17 / Tuần 4 - Ngày 1 — Đáp án gợi ý & Checklist hoàn thành"
track: chung
day: 17
topic: dap-an-checklist
tags: [dap-an, checklist]
source_goc: Tuan4-Ngay1-Dart3SealedClass-BlocSelector-N1Query.md
---

# Ngày 17 / Tuần 4 - Ngày 1 — Đáp án gợi ý & Checklist hoàn thành

## ĐÁP ÁN GỢI Ý

<details><summary>Bài 1 — sealed class TaskSealedState + switch pattern matching (xem D17_dart3-sealed-class-records.md)</summary>

Đã có sẵn trong repo tại `lib/tuan3_ngay7/bai1_sealed_state.dart` +
`bai2_sealed_demo_screen.dart`:

```dart
sealed class TaskSealedState {}
class TaskSealedInitial extends TaskSealedState {}
class TaskSealedLoading extends TaskSealedState {}
class TaskSealedLoaded extends TaskSealedState {
  final List<Task> danhSach;
  TaskSealedLoaded(this.danhSach);
}
class TaskSealedError extends TaskSealedState {
  final String thongDiep;
  TaskSealedError(this.thongDiep);
}
```

```dart
Widget _buildTheoState(TaskSealedState state) => switch (state) {
  TaskSealedInitial() => ...,
  TaskSealedLoading() => ...,
  TaskSealedLoaded(danhSach: final ds) => ...,
  TaskSealedError(thongDiep: final td) => ...,
};
```
</details>

<details><summary>Bài 2 — Thêm class con mới, quan sát lỗi biên dịch (xem D17_dart3-sealed-class-records.md)</summary>

Bỏ comment dòng có sẵn trong `bai1_sealed_state.dart`:

```dart
class TaskSealedEmpty extends TaskSealedState {}
```

Không sửa `switch` trong `bai2_sealed_demo_screen.dart` — xác nhận Dart báo lỗi biên dịch
ngay (thiếu nhánh xử lý `TaskSealedEmpty`), buộc phải thêm nhánh mới chạy được lại.
</details>

<details><summary>Bài 3 — Records: thongKeTask() + layTaskVaTongSoTrang() (xem D17_dart3-sealed-class-records.md)</summary>

Đã có sẵn trong repo tại `lib/tuan3_ngay7/bai3_records_screen.dart`:

```dart
({int tongSoTask, int soHoanThanh, int dangLam}) thongKeTask(List<Task> danhSach) { ... }

(List<Task>, int) layTaskVaTongSoTrang(List<Task> allTasks, int trang) { ... }
```

Sử dụng: `final thongKe = thongKeTask(_allTasks); print(thongKe.tongSoTask);` và
`final (trangHienTai, tongTrang) = layTaskVaTongSoTrang(_allTasks, _trang);`.
</details>

<details><summary>Bài 4 — buildWhen giảm số lần rebuild (xem D17_blocselector-buildwhen-listenwhen.md)</summary>

Đã có sẵn trong repo tại `lib/tuan3_ngay7/bai4_bai5_bai6_bloc_selector_screen.dart`:

```dart
BlocBuilder<TaskSealedCubit, TaskSealedState>(
  buildWhen: (old, current) => old.runtimeType != current.runtimeType,
  builder: (context, state) { ... },
)
```

Kiểm chứng: chạy `BlocSelectorScreen`, bấm Refresh nhiều lần, so sánh số đếm "❌ BlocBuilder
thô" vs "✅ buildWhen (Bài 4)" trên bảng thống kê ngay trong UI.
</details>

<details><summary>Bài 5 — BlocSelector cho giá trị đơn giản (xem D17_blocselector-buildwhen-listenwhen.md)</summary>

Đã có sẵn trong repo — dùng cho tiêu đề AppBar hiển thị số lượng Task:

```dart
title: BlocSelector<TaskSealedCubit, TaskSealedState, int>(
  selector: (state) => state is TaskSealedLoaded ? state.danhSach.length : 0,
  builder: (context, soLuong) => Text('Tasks ($soLuong)'),
),
```

Kiểm chứng: quan sát counter "✅ BlocSelector (Bài 5)" tăng ít hơn số lần state thực sự đổi.
</details>

<details><summary>Bài 6 — listenWhen tránh SnackBar lặp lại (xem D17_blocselector-buildwhen-listenwhen.md)</summary>

Đã có sẵn trong repo — bọc ngoài `Scaffold`:

```dart
BlocListener<TaskSealedCubit, TaskSealedState>(
  listenWhen: (old, current) => current is TaskSealedError,
  listener: (context, state) {
    if (state is TaskSealedError) {
      ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('Lỗi: ${state.thongDiep}')));
    }
  },
  child: Scaffold(...),
)
```

Kiểm chứng: bấm Refresh nhiều lần, xác nhận SnackBar chỉ hiện đúng khi có lỗi thật, không
lặp lại ở các lần đổi state khác.
</details>

<details><summary>Bài 7 — Tái hiện N+1, đếm số query trong log (xem D17_n-plus-1-query-entitygraph.md)</summary>

Đã có sẵn trong repo tại `NPlusOneController.demoNPlus1()`:

```java
@Transactional
@GetMapping("/n-plus-1")
public ResponseEntity<?> demoNPlus1() {
    List<Task> tasks = taskRepository.findAllForNPlusOne();
    // .getNguoiTao().getUsername() → mỗi dòng = 1 SELECT thêm
}
```

Kiểm chứng: bật `show-sql`, gọi `GET /api/demo/n-plus-1`, đếm số dòng `select` trong log
— xác nhận đúng **4 câu query** (1 Task + 3 User riêng lẻ, khớp 3 task seed sẵn).
</details>

<details><summary>Bài 8 — Sửa bằng @EntityGraph, xác nhận giảm còn 1 query (xem D17_n-plus-1-query-entitygraph.md)</summary>

Đã có sẵn trong repo tại `TaskRepository.findAllWithNguoiTao()` +
`NPlusOneController.demoFix()`:

```java
@EntityGraph(attributePaths = {"nguoiTao"})
@Query("SELECT t FROM Task t")
List<Task> findAllWithNguoiTao();
```

Kiểm chứng: gọi `GET /api/demo/fix-entity-graph`, đếm lại số dòng `select` — xác nhận chỉ
còn **đúng 1 câu**, kết quả trả về giống hệt Bài 7.
</details>

<details><summary>Bài 9 — Cartesian product khi có @OneToMany (nâng cao, không bắt buộc)</summary>

Chưa có sẵn trong repo — tự thử: thêm 1 quan hệ `@OneToMany` mới (VD `List<Comment>` cho
`Task`), dùng `@EntityGraph(attributePaths = {"nguoiTao", "comments"})` lấy kèm cả 2 quan
hệ cùng lúc — quan sát số dòng Task trả về có bị nhân bản theo số lượng comment không
(Cartesian product).
</details>

---

## ✅ CHECKLIST HOÀN THÀNH

- [ ] `TaskState` đã chuyển sang `sealed class`, UI dùng `switch` pattern matching thay `if-else`/`is`
- [ ] Tự kiểm chứng được lỗi biên dịch xuất hiện khi thêm state mới mà quên sửa `switch`
- [ ] Dùng được Named Record và Positional Record đúng chỗ, phân biệt khi nào dùng loại nào
- [ ] Giải thích được khác biệt `BlocSelector` vs `BlocBuilder + buildWhen`, và `buildWhen` vs `listenWhen`
- [ ] Đo được số lần rebuild giảm thực sự sau khi áp dụng `buildWhen`/`BlocSelector` (bằng counter, không chỉ cảm nhận)
- [ ] Tái hiện được N+1 Query bằng log SQL, hiểu đúng con số N+1 tương ứng với dataset thật
- [ ] Sửa N+1 bằng `@EntityGraph`, xác nhận số câu query giảm đúng như kỳ vọng
- [ ] Hiểu vì sao cần `@Transactional` để lazy loading hoạt động đúng khi demo N+1, và vì sao endpoint đã fix thì không cần
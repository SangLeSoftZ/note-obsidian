---
title: "Ngày 17 (Tuần 4 - Ngày 1, Sáng) — Dart 3: Sealed Class, Pattern Matching, Records"
track: flutter
day: 17
topic: dart3-sealed-class-records
tags: [flutter, dart3, sealed-class, pattern-matching, records, exhaustiveness]
source_goc: Tuan4-Ngay1-Dart3SealedClass-BlocSelector-N1Query.md
---

# Ngày 17 (Tuần 4 - Ngày 1, Sáng) — Dart 3: Sealed Class, Pattern Matching, Records

> Lưu ý về số ngày: tài liệu gốc ghi "Tuần 4 - Ngày 1", nhưng trong code thật của 2 repo,
> nội dung này nằm ở thư mục `tuan3_ngay7` (nối tiếp `tuan3_ngay6` — GetIt Lifecycle/Scope
> — vốn cũng chưa có note riêng trong vault). Ghi chú này dùng số ngày **D17** theo đúng
> thứ tự tiếp theo trong vault Obsidian hiện có (cao nhất đang là D16).

## 1. Vấn đề của cách viết `TaskState` kiểu cũ

Từ Tuần 1, `TaskState` thường viết bằng `abstract class` + nhiều class con kế thừa. Vấn đề
cốt lõi: `abstract class` **không giới hạn nơi kế thừa** — 1 class con "lạ" có thể xuất
hiện ở bất kỳ file nào, nên trình biên dịch **không thể biết** đã liệt kê đủ mọi trường
hợp trong 1 khối `if-else`/`is` hay chưa. Quên xử lý 1 nhánh (VD `TaskInitial`) vẫn biên
dịch được bình thường — lỗi chỉ lộ ra lúc chạy thực tế.

## 2. `sealed class` — code thật, Bài 1 và Bài 2 gộp chung 1 khái niệm để so sánh

Code thật — `lib/tuan3_ngay7/bai1_sealed_state.dart`:

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

// ── BÀI 2: Thêm class con mới để test exhaustiveness ─────────────
// Bỏ comment dòng dưới → switch trong bai2_sealed_demo_screen.dart
// sẽ báo lỗi biên dịch NGAY nếu chưa xử lý TaskSealedEmpty
// class TaskSealedEmpty extends TaskSealedState {}
```

**Chi tiết hay hơn ví dụ lý thuyết gốc:** code thật để sẵn 1 dòng `TaskSealedEmpty` đã
comment sẵn, kèm hướng dẫn ngay tại chỗ — chỉ cần bỏ comment dòng này là **tự tay kích
hoạt** đúng tình huống lỗi biên dịch cần quan sát cho Bài 2, không cần tự viết thêm code.

`sealed` khác `abstract` ở điểm: **toàn bộ** class con của `TaskSealedState` chỉ được
phép khai báo **trong cùng 1 file** (hoặc cùng thư viện) — không có class con "lạ" từ
file khác. Nhờ vậy, khi dùng `switch`, trình biên dịch **biết chính xác** có bao nhiêu
nhánh cần xử lý và **báo lỗi biên dịch ngay** nếu thiếu.

## 3. `switch` pattern matching — destructuring trong `bai2_sealed_demo_screen.dart`

```dart
Widget _buildTheoState(TaskSealedState state) {
  return switch (state) {
    TaskSealedInitial() => Center(/* ... */),
    TaskSealedLoading()  => const Center(child: CircularProgressIndicator()),
    // Destructuring: lấy danhSach ra biến ds trong 1 bước
    // Không cần: final ds = (state as TaskSealedLoaded).danhSach
    TaskSealedLoaded(danhSach: final ds) => ds.isEmpty
        ? const Center(child: Text('Không có task nào'))
        : ListView.builder(itemCount: ds.length, itemBuilder: (context, i) { ... }),
    TaskSealedError(thongDiep: final td) => Center(child: Text(td)),
  };
  // KHÔNG có "default:" — sealed đảm bảo đây là TẤT CẢ trường hợp có thể có
}
```

Không có `default:` — chính là điểm mấu chốt: nếu `TaskSealedEmpty` được thêm vào (bỏ
comment ở Bài 1) mà quên sửa `switch` này, Dart **báo lỗi biên dịch ngay lập tức**, buộc
xử lý đầy đủ trước khi chạy được app — tính "kiểm tra đầy đủ" (exhaustiveness checking)
không thể đạt được với `if-else`/`is` thông thường.

Màn hình demo (`SealedClassScreen`) có sẵn nút Refresh (gọi `taiDanhSach()`) và nút Reset
(gọi `reset()` về `TaskSealedInitial`) để tự chuyển qua đủ 4 trạng thái và quan sát UI
tương ứng — kèm hiển thị trực tiếp `state.runtimeType.toString()` trên màn hình để biết
đang ở đúng state nào.

## 4. Records — Bài 3, `thongKeTask()` và `layTaskVaTongSoTrang()` thật

Code thật — `lib/tuan3_ngay7/bai3_records_screen.dart`. Named Record — không cần tạo hẳn
1 class chỉ để gói 3 giá trị thống kê:

```dart
({int tongSoTask, int soHoanThanh, int dangLam}) thongKeTask(List<Task> danhSach) {
  final hoanThanh = danhSach.where((t) => t.trangThai == 'HOAN_THANH').length;
  final dangLam = danhSach.where((t) => t.trangThai == 'DANG_LAM').length;
  return (tongSoTask: danhSach.length, soHoanThanh: hoanThanh, dangLam: dangLam);
}
```

Positional Record (không tên field) — dùng cho phân trang, kèm logic cắt danh sách theo
trang thật, không phải ví dụ giả:

```dart
(List<Task>, int) layTaskVaTongSoTrang(List<Task> allTasks, int trang) {
  const perPage = 2;
  final tongTrang = (allTasks.length / perPage).ceil();
  final start = (trang - 1) * perPage;
  final end = (start + perPage).clamp(0, allTasks.length);
  return (allTasks.sublist(start, end), tongTrang);
}
```

Sử dụng — destructure ngay lúc gọi, đúng 2 kiểu Record cùng lúc trong 1 màn hình để so
sánh trực tiếp:

```dart
// Positional — truy cập theo vị trí khi destructure
final (trangHienTai, tongTrang) = layTaskVaTongSoTrang(_allTasks, _trang);

// Named — truy cập qua tên field, rõ nghĩa hơn
final thongKe = thongKeTask(_allTasks);
print(thongKe.tongSoTask); // không phải thongKe.$1
```

**Named vs Positional Record — khi nào dùng loại nào:** Positional phù hợp khi thứ tự giá
trị đã đủ rõ nghĩa và ngắn gọn (`(List<Task>, int)` — hiển nhiên là "danh sách rồi tới
tổng trang"); Named phù hợp khi có từ 3 giá trị trở lên hoặc thứ tự dễ gây nhầm lẫn
(`tongSoTask`/`soHoanThanh`/`dangLam` đều là `int`, nếu dùng positional dễ gán nhầm thứ
tự).

**Record khác class ở điểm cốt lõi:** Record là kiểu dữ liệu **không cần khai báo trước**
— Dart tự suy ra kiểu dựa vào giá trị bên trong. Phù hợp cho "gói tạm nhiều giá trị" trong
phạm vi hẹp (trả về từ 1 hàm, dùng nội bộ) — **không** thay thế hoàn toàn cho `class` khi
cần đặt tên rõ ràng, có method riêng, hoặc dùng rộng rãi xuyên suốt nhiều nơi (lúc đó
`TaskSealedLoaded` vẫn hợp lý hơn nhiều so với 1 Record ẩn danh).

## Bài tập

**Bài 1 (đã có sẵn — `bai1_sealed_state.dart` + `bai2_sealed_demo_screen.dart`):** Chạy
`SealedClassScreen`, bấm Refresh/Reset để chuyển qua đủ 4 trạng thái, đối chiếu với
`switch` trong `_buildTheoState`.

**Bài 2 (đã có sẵn — chỉ cần bỏ comment):** Bỏ comment dòng `class TaskSealedEmpty extends TaskSealedState {}`
trong `bai1_sealed_state.dart`, **không** sửa `switch` trong `bai2_sealed_demo_screen.dart`
— xác nhận Dart báo lỗi biên dịch ngay, buộc phải thêm nhánh xử lý mới chạy được lại.

**Bài 3 (đã có sẵn — `bai3_records_screen.dart`):** Chạy `RecordsScreen`, đối chiếu số
liệu Named Record (`thongKe.tongSoTask`...) và điều hướng phân trang bằng Positional
Record (`layTaskVaTongSoTrang`).

## Code Example (repo `flutter.git`)

> Xem nguyên các file tại branch `Dart3`:
> [`bai1_sealed_state.dart`](https://github.com/SangLeSoftZ/flutter/blob/Dart3/testflutter/lib/tuan3_ngay7/bai1_sealed_state.dart) ·
> [`bai2_sealed_demo_screen.dart`](https://github.com/SangLeSoftZ/flutter/blob/Dart3/testflutter/lib/tuan3_ngay7/bai2_sealed_demo_screen.dart) ·
> [`bai3_records_screen.dart`](https://github.com/SangLeSoftZ/flutter/blob/Dart3/testflutter/lib/tuan3_ngay7/bai3_records_screen.dart)
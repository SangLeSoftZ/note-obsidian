---
title: "Ngày 14 (Tuần 3 - Ngày 3, Sáng) — Generics, Mixin, Extension method trong Dart"
track: flutter
day: 14
topic: generics-mixin-extension
tags: [flutter, dart-advanced, generics, mixin, extension-method]
source_goc: Tuan3-Ngay3-Generics-Mixin-Riverpod-JPAAuditing.md
---

# Ngày 14 (Tuần 3 - Ngày 3, Sáng) — Generics, Mixin, Extension method trong Dart

## 1. Generics — `ApiResponse<T>` với named constructor thật

Code thật — `lib/tuan3_ngay3/bai1_generics.dart` — chi tiết hơn ví dụ lý thuyết gốc: có
thêm `statusCode` và 1 factory method dùng `try/catch`:

```dart
class ApiResponse<T> {
  final bool thanhCong;
  final T? duLieu;
  final String? loi;
  final int? statusCode;

  const ApiResponse.ok(this.duLieu)
      : thanhCong = true, loi = null, statusCode = 200;

  const ApiResponse.loi(this.loi, {this.statusCode})
      : thanhCong = false, duLieu = null;

  // Factory từ try/catch — dùng trong ApiClient thực tế
  static ApiResponse<T> from<T>(T Function() goi) {
    try {
      return ApiResponse.ok(goi());
    } catch (e) {
      return ApiResponse.loi(e.toString());
    }
  }
}
```

**`T` là "biến kiểu dữ liệu"** — giống tham số của hàm nhưng thay vì nhận **giá trị**, nó
nhận **kiểu dữ liệu**. Viết `ApiResponse<Task>`, Dart tự thay mọi chỗ `T` thành `Task`;
viết `ApiResponse<List<Task>>`, tự thay thành `List<Task>` — code gốc của class không đổi
1 dòng.

**Chi tiết đáng chú ý:** `static ApiResponse<T> from<T>(...)` khai báo `<T>` **2 lần** —
1 lần ở tên class (`ApiResponse<T>`), 1 lần ở chính method (`from<T>`). Vì đây là
**static method**, nó không "thấy" được `T` của instance (static method không gắn với 1
instance cụ thể nào) — nên phải khai báo lại `<T>` riêng cho chính method đó. Đây là điểm
dễ nhầm khi mới học generics với static method.

**Generics có ràng buộc — `<T extends Comparable<T>>`:**

```dart
T timLonNhat<T extends Comparable<T>>(List<T> ds) {
  assert(ds.isNotEmpty, 'Danh sách không được rỗng');
  var lonNhat = ds.first;
  for (final item in ds) {
    if (item.compareTo(lonNhat) > 0) lonNhat = item;
  }
  return lonNhat;
}

timLonNhat([3, 7, 2, 9, 1]);              // Dart tự suy ra <int>
timLonNhat(['apple', 'zebra', 'mango']);  // Dart tự suy ra <String>
```

`extends Comparable<T>` giới hạn: chỉ kiểu nào **so sánh được** (`int`, `String`,
`DateTime`...) mới truyền vào được — gọi `timLonNhat([Task(...), Task(...)])` sẽ báo lỗi
biên dịch ngay, vì `Task` không implement `Comparable`. Đây là lợi ích cốt lõi của
generics có ràng buộc: bắt lỗi **lúc biên dịch**, không phải đợi tới runtime mới crash.

**Generics đã dùng ngầm từ đầu khóa:** `List<Task>`, `Map<String, dynamic>`,
`Future<Task>`, `Stream<String>`, `BlocBuilder<TaskCubit, TaskState>` — tất cả đều là
Generics. Hôm nay là lúc hiểu cơ chế bên dưới thay vì chỉ dùng theo thói quen.

## 2. Mixin — áp dụng cả cho `State` class, không chỉ Repository

Code thật — `lib/tuan3_ngay3/bai2_mixin.dart` — mở rộng hơn ví dụ lý thuyết gốc, cho thấy
mixin dùng được **cả với class State của Flutter**, không chỉ các class tự viết:

```dart
mixin LoggerMixin {
  String get _tenClass => runtimeType.toString();
  void logInfo(String msg) { /* in log kèm timestamp + tên class */ }
  void logLoi(String msg)  { /* in log lỗi kèm timestamp + tên class */ }
}

mixin ValidatorMixin {
  String? validateTieuDe(String? value) { ... }
  String? validateEmail(String? value)  { ... }
}

// Repository dùng 1 mixin
class TaskRepository with LoggerMixin { ... }

// Helper dùng 2 mixin CÙNG LÚC
class LoginFormHelper with ValidatorMixin, LoggerMixin { ... }

// State CŨNG dùng được Mixin — không giới hạn ở class tự viết
class _MixinScreenState extends State<MixinScreen> with LoggerMixin {
  // Vừa extends State<MixinScreen> (bắt buộc, để có lifecycle Flutter)
  // Vừa with LoggerMixin (trộn thêm khả năng log) — 2 việc không xung đột
}
```

**`_tenClass => runtimeType.toString()`** là chi tiết hay: mixin dùng `runtimeType` để tự
biết nó đang được trộn vào class nào tại thời điểm chạy — log ra sẽ hiện đúng tên class
thực sự (`TaskRepository`, `LoginFormHelper`, `_MixinScreenState`...) dù `LoggerMixin`
chỉ viết 1 lần.

**Phân biệt cốt lõi:**
- `extends`: quan hệ **"là một"** (is-a). Chỉ được **1** class cha.
- `with` (mixin): quan hệ **"có khả năng"** (can-do). Được trộn **nhiều** mixin cùng lúc.

Ví dụ `_MixinScreenState` chứng minh rõ nhất: nó **là một** `State<MixinScreen>` (bắt
buộc `extends`, không có lựa chọn khác vì Flutter yêu cầu), nhưng **có thêm khả năng**
log giống `LoggerMixin` (không bắt buộc, chỉ là tiện ích cộng thêm) — 2 quan hệ khác bản
chất, dùng đúng từ khóa cho đúng mục đích.

## 3. Extension method — bộ tiện ích thật cho `Task`/`String`/`DateTime`

Code thật — `lib/tuan3_ngay3/bai3_extension.dart` — nhiều hơn hẳn ví dụ lý thuyết gốc,
gồm cả extension xử lý tiếng Việt:

```dart
extension TaskExtension on Task {
  String get trangThaiHienThi { /* switch theo trangThai → chuỗi tiếng Việt */ }
  Color get mauTrangThai      { /* switch → màu tương ứng */ }
  IconData get iconTrangThai  { /* switch → icon tương ứng */ }
  bool get tieuDeDai => tieuDe.length > 20;
}

extension StringExtension on String {
  String get vietHoaChuCai => isEmpty ? this : this[0].toUpperCase() + substring(1);
  bool get laEmailHopLe => RegExp(r'^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$').hasMatch(this);
  String rutGon(int max) => length <= max ? this : '${substring(0, max)}...';
  String get boDau { /* bỏ dấu tiếng Việt bằng chuỗi RegExp thay thế từng nhóm ký tự */ }
}

extension DateTimeExtension on DateTime {
  String get dinhDangVN => '${day.toString().padLeft(2,'0')}/${month...}/$year';
  String get thoiGianTuongDoi {
    final diff = DateTime.now().difference(this);
    if (diff.inDays > 0) return '${diff.inDays} ngày trước';
    if (diff.inHours > 0) return '${diff.inHours} giờ trước';
    if (diff.inMinutes > 0) return '${diff.inMinutes} phút trước';
    return 'Vừa xong';
  }
}
```

**`String.boDau`** — chính hàm bỏ dấu tiếng Việt đã dùng ở `custom_widget_screen.dart`
(Tuần 2 Ngày 4) để lọc tìm kiếm không phân biệt dấu — giờ được "chính thức hoá" thành
extension method tái sử dụng được ở mọi nơi, thay vì viết hàm riêng lẻ trong từng màn hình
như trước.

**`DateTime.thoiGianTuongDoi`** — dạng hiển thị "X phút/giờ/ngày trước" rất phổ biến
trong UI thực tế (feed, thông báo, bình luận) — ví dụ điển hình cho lý do nên gắn logic
này vào `DateTime` qua extension thay vì viết hàm riêng lẻ ở từng nơi cần hiển thị.

**Cơ chế bên dưới (đúng như lý thuyết gốc):** extension **không thực sự sửa** class
`String`/`DateTime`/`Task` gốc — trình biên dịch chỉ "biết" gọi đúng hàm nào dựa vào kiểu
dữ liệu tại chỗ gọi (syntactic sugar: `task.trangThaiHienThi` thực chất tương đương
`TaskExtension(task).trangThaiHienThi`). Vì vậy extension method **không override được**
hành vi có sẵn, chỉ **thêm mới**.

## 4. Bảng quyết định nhanh

| Tình huống | Dùng gì |
|---|---|
| Cần viết class/hàm dùng chung cho nhiều kiểu dữ liệu khác nhau | **Generics** |
| Cần chia sẻ hành vi cho nhiều class không cùng dòng kế thừa (kể cả `State`) | **Mixin** |
| Cần thêm hàm tiện ích cho class có sẵn (kể cả class thư viện/Dart core) | **Extension method** |

## Bài tập

**Bài 1 (đã có sẵn — `bai1_generics.dart`):** Chạy `GenericsScreen`, đối chiếu
`ApiResponse<Task>`, `ApiResponse<Task>.loi(...)`, `ApiResponse<List<Task>>` — xác nhận
cùng 1 class dùng được cho cả object đơn lẻ, lỗi, và danh sách.

**Bài 2 (đã có sẵn — `bai2_mixin.dart`):** Chạy `MixinScreen`, thử nhập email/password sai
để kích hoạt `ValidatorMixin`, quan sát log console để thấy `LoggerMixin` hoạt động cả ở
`LoginFormHelper` lẫn `_MixinScreenState` (2 class không liên quan gì tới nhau).

**Bài 3 (đã có sẵn — `bai3_extension.dart`):** Chạy `ExtensionScreen`, đối chiếu từng dòng
kết quả với extension tương ứng (`TaskExtension`, `StringExtension`, `DateTimeExtension`).

##  Code Example (repo `flutter.git`)

> Xem nguyên các file tại branch `Generics,-Mixin,-Riverpod`:
> [`bai1_generics.dart`](<https://github.com/SangLeSoftZ/flutter/blob/Generics,-Mixin,-Riverpod/testflutter/lib/tuan3_ngay3/bai1_generics.dart>) ·
> [`bai2_mixin.dart`](<https://github.com/SangLeSoftZ/flutter/blob/Generics,-Mixin,-Riverpod/testflutter/lib/tuan3_ngay3/bai2_mixin.dart>) ·
> [`bai3_extension.dart`](<https://github.com/SangLeSoftZ/flutter/blob/Generics,-Mixin,-Riverpod/testflutter/lib/tuan3_ngay3/bai3_extension.dart>)


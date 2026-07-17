---
title: "Ngày 12 (Tuần 3 - Ngày 1, Sáng) — Isolates: Chạy tác vụ nặng không làm đơ UI"
track: flutter
day: 12
topic: isolates-compute
tags: [flutter, isolate, compute, concurrency, performance]
source_goc: Tuan3_Ngay1_TaiLieu.md
---

# Ngày 12 (Tuần 3 - Ngày 1, Sáng) — Isolates: Chạy tác vụ nặng không làm đơ UI

## 1. Vì sao `async/await` không cứu được — vấn đề CPU-bound

Dart chạy trên **1 luồng chính duy nhất** (main isolate) — vừa vẽ UI, vừa xử lý logic.
Điểm dễ nhầm nhất: `async/await`/`Future` (đã học Ngày 1 Tuần 1) chỉ giải quyết vấn đề
**chờ đợi** (I/O-bound — chờ mạng, chờ file) — trong lúc chờ, luồng chính **rảnh**. Nhưng
với tác vụ **tính toán nặng thực sự** (CPU-bound — CPU làm việc liên tục không ngừng),
`async/await` **không giúp được gì** — luồng chính vẫn bị chiếm dụng 100% dù có bọc
`Future` hay không, vì không có khoảnh khắc nào để "nhường" lại cho việc vẽ UI.

## 2. Isolate — bộ nhớ riêng, không chia sẻ (khác Thread ở Java)

**Isolate** = 1 luồng thực thi Dart hoàn toàn độc lập, **có bộ nhớ riêng**, không chia sẻ
bộ nhớ với isolate khác. Đây là khác biệt cốt lõi so với Thread trong Java — Thread Java
**chia sẻ chung** bộ nhớ (dễ gây race condition khi nhiều thread cùng sửa 1 biến). Vì
không chia sẻ bộ nhớ, 2 isolate muốn "nói chuyện" phải gửi dữ liệu qua lại (message
passing, dữ liệu được **copy** sang isolate kia) — an toàn hơn nhưng chỉ gửi được dữ liệu
có thể "sao chép" được (không gửi được instance phức tạp có tham chiếu vòng, hàm callback...).

## 3. Bài 1 — So sánh trực tiếp: luồng chính đơ vs `compute()` mượt

Code thật — `lib/tuan3_ngay1/bai1_isolate_demo_screen.dart`. Hàm tính toán nặng **bắt
buộc phải là top-level** (viết ngoài mọi class):

```dart
// PHẢI là top-level hoặc static
// Lý do: Isolate mới không có quyền truy cập bộ nhớ của isolate gốc
// → cần hàm "độc lập", không phụ thuộc `this` hay bất kỳ instance nào
int tinhTongNang(int soLuong) {
  int tong = 0;
  for (int i = 0; i < soLuong; i++) {
    tong += i;
  }
  return tong;
}
```

**1a — Chạy trực tiếp trên luồng chính (UI đơ):**

```dart
void _tinhTrucTiep() {
  setState(() { _dangTinh = true; _ketQua = 'Đang tính...'; });
  // Chạy thẳng trên main isolate → UI ĐÓNG BĂNG
  final ketQua = tinhTongNang(_soLuong); // 500 triệu số
  setState(() { _dangTinh = false; _ketQua = 'Kết quả: $ketQua'; });
}
```

**1b — Dùng `compute()` (UI mượt):**

```dart
Future<void> _tinhVoiCompute() async {
  setState(() { _dangTinh = true; _ketQua = 'Đang tính (isolate)...'; });
  // compute() chạy tinhTongNang trên Isolate riêng
  final ketQua = await compute(tinhTongNang, _soLuong);
  setState(() { _dangTinh = false; _ketQua = 'Kết quả: $ketQua'; });
}
```

Cách tự kiểm chứng rất trực quan trong code: màn hình có sẵn 1 nút **Counter** riêng biệt
— trong lúc `_tinhTrucTiep()` đang chạy, bấm Counter **không phản hồi gì** (UI đơ thật
sự); trong lúc `_tinhVoiCompute()` đang chạy, Counter **vẫn tăng bình thường** — bằng
chứng UI vẫn mượt dù đang tính 500 triệu số.

## 4. `compute(function, argument)` làm gì bên dưới

1. Tạo 1 Isolate mới với bộ nhớ riêng.
2. Gửi `argument` sang isolate đó (copy dữ liệu qua message passing).
3. Chạy `function` **hoàn toàn trên isolate riêng**.
4. Gửi **kết quả** trả về luồng chính.

Trong suốt quá trình, luồng chính **hoàn toàn rảnh** — đây là lý do UI không đơ.

## 5. Bài 2 — Ứng dụng thực tế: parse JSON lớn (50.000 Task)

Code thật — `lib/tuan3_ngay1/bai2_isolate_parse_screen.dart`:

```dart
// Hàm parse JSON — PHẢI là top-level
// Không thể là method của class vì Isolate không truy cập được `this`
List<TaskItem> parseJsonList(String jsonString) {
  final List<dynamic> data = jsonDecode(jsonString) as List<dynamic>;
  return data.map((item) {
    final map = item as Map<String, dynamic>;
    // Giả lập xử lý nặng cho mỗi item để tạo tải CPU thật sự
    var dummy = 0;
    for (var i = 0; i < 1000; i++) { dummy += i; }
    return TaskItem.fromJson(map);
  }).toList();
}
```

So với ví dụ lý thuyết gốc, code thật đo thời gian bằng `Stopwatch` để thấy rõ con số cụ
thể:

```dart
final stopwatch = Stopwatch()..start();
final result = await compute(parseJsonList, jsonString);
stopwatch.stop();
// Hiển thị: '${stopwatch.elapsedMilliseconds}ms (UI mượt)'
```

Và cũng có nút Counter riêng để tự tay kiểm chứng UI có đơ hay không trong lúc parse —
đúng cách kiểm tra thực nghiệm, không chỉ đọc lý thuyết suông.

## 6. Khi nào thực sự cần Isolate — đừng lạm dụng

**Dùng khi:** parse JSON/XML rất lớn, xử lý ảnh (resize, filter), mã hóa/giải mã, thuật
toán tính toán phức tạp — nói chung là **CPU-bound, tính toán liên tục**.

**KHÔNG cần khi:** gọi API (đã bất đồng bộ qua `Future`), đọc/ghi file nhỏ, phép tính đơn
giản — dùng Isolate cho việc nhẹ chỉ làm code phức tạp không cần thiết, vì **tạo Isolate
cũng tốn chi phí** (khởi tạo isolate mới, copy dữ liệu qua lại) — với việc nhẹ, chi phí này
còn lớn hơn lợi ích.

## Bài tập

**Bài 1 (đã có sẵn — `bai1_isolate_demo_screen.dart`):** Tự chạy cả 2 nút, dùng Counter để
tự kiểm chứng UI đơ vs UI mượt.

**Bài 2 (đã có sẵn — `bai2_isolate_parse_screen.dart`):** Đối chiếu thời gian
(`elapsedMilliseconds`) giữa parse trực tiếp và `compute()` — giải thích bằng lời: vì sao
tổng thời gian xử lý của `compute()` **có thể chậm hơn một chút** so với parse trực tiếp
(gợi ý: chi phí tạo isolate + copy dữ liệu), nhưng vẫn đáng dùng vì UI không bị đơ.

##  Code Example (repo `flutter.git`)

> Xem nguyên các file tại branch `CustomPainter` (đã gộp cả `Isolates`):
> [`bai1_isolate_demo_screen.dart`](https://github.com/SangLeSoftZ/flutter/blob/CustomPainter/testflutter/lib/tuan3_ngay1/bai1_isolate_demo_screen.dart) ·
> [`bai2_isolate_parse_screen.dart`](https://github.com/SangLeSoftZ/flutter/blob/CustomPainter/testflutter/lib/tuan3_ngay1/bai2_isolate_parse_screen.dart)

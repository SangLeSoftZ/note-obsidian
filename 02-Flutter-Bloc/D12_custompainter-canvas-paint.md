---
title: "Ngày 12 (Tuần 3 - Ngày 1, Chiều) — CustomPainter: Tự vẽ UI tùy biến"
track: flutter
day: 12
topic: custompainter-canvas-paint
tags: [flutter, custompainter, canvas, paint, shouldrepaint, animation]
source_goc: Tuan3_Ngay1_TaiLieu.md
---

# Ngày 12 (Tuần 3 - Ngày 1, Chiều) — CustomPainter: Tự vẽ UI tùy biến

## 1. Vấn đề: Widget có sẵn không đủ cho hình dạng tùy biến

Thư viện Widget của Flutter đủ dùng cho UI thông thường, nhưng không đủ khi cần **hình
dạng/hiệu ứng hoàn toàn tùy biến** — ví dụ vòng tròn tiến độ dạng riêng, biểu đồ tự vẽ.
`CustomPainter` cho phép "vẽ tay" trực tiếp lên `Canvas` — toàn quyền kiểm soát từng điểm
ảnh.

## 2. `VongTronTienDoPainter` — code thật, chi tiết hơn ví dụ lý thuyết gốc

Code thật — `lib/tuan3_ngay1/bai3_custom_painter_screen.dart`. So với ví dụ trong tài
liệu gốc, bản thật **tham số hóa đầy đủ** (màu nền, màu tiến độ, độ dày đều truyền vào
được) và **có vẽ thêm text %** ở giữa vòng tròn:

```dart
class VongTronTienDoPainter extends CustomPainter {
  final double phanTram;    // 0.0 → 1.0
  final Color mauNen;
  final Color mauTienDo;
  final double doDay;

  VongTronTienDoPainter({
    required this.phanTram,
    this.mauNen = const Color(0xFFE0E0E0),
    this.mauTienDo = Colors.blue,
    this.doDay = 10,
  });

  @override
  void paint(Canvas canvas, Size size) {
    final center = Offset(size.width / 2, size.height / 2);
    final radius = (size.width / 2) - doDay / 2; // trừ đi doDay/2 để nét vẽ không bị cắt mép

    final paintNen = Paint()
      ..color = mauNen
      ..style = PaintingStyle.stroke  // chỉ viền, không tô đầy
      ..strokeWidth = doDay;
    canvas.drawCircle(center, radius, paintNen);

    final paintTienDo = Paint()
      ..color = mauTienDo
      ..style = PaintingStyle.stroke
      ..strokeWidth = doDay
      ..strokeCap = StrokeCap.round;  // đầu cung bo tròn, không vuông cạnh

    canvas.drawArc(
      Rect.fromCircle(center: center, radius: radius),
      -math.pi / 2,           // bắt đầu từ đỉnh trên (12 giờ)
      2 * math.pi * phanTram, // quét theo chiều kim đồng hồ
      false,                   // false = không nối tâm
      paintTienDo,
    );

    // Vẽ text % ở giữa — dùng TextPainter, không dùng Widget Text thông thường
    final textPainter = TextPainter(
      text: TextSpan(
        text: '${(phanTram * 100).round()}%',
        style: TextStyle(fontSize: size.width * 0.22, fontWeight: FontWeight.bold, color: mauTienDo),
      ),
      textDirection: TextDirection.ltr,
    )..layout();

    textPainter.paint(
      canvas,
      Offset(center.dx - textPainter.width / 2, center.dy - textPainter.height / 2),
    );
  }
  ...
}
```

**Vì sao vẽ chữ bằng `TextPainter` chứ không đặt 1 Widget `Text` chồng lên?** Vì bên
trong `paint()` bạn chỉ có `Canvas` để làm việc — không có cây Widget nào ở đây cả.
`TextPainter` là công cụ "vẽ chữ lên Canvas" tương ứng, cần gọi `.layout()` trước để nó
tính toán kích thước chữ, rồi mới `.paint()` để vẽ đúng vị trí (ở đây là canh giữa bằng
cách trừ đi nửa `width`/`height` của chữ).

## 3. `radius = (size.width / 2) - doDay / 2` — chi tiết dễ bỏ sót

Đây là điểm tinh tế không có trong ví dụ lý thuyết gốc (ví dụ gốc dùng thẳng
`radius = size.width / 2`): nếu không trừ đi `doDay / 2`, phần nét vẽ dày `doDay` sẽ bị
**cắt mép** ở rìa `CustomPaint` (vì `drawCircle` vẽ theo tâm đường viền, nét vẽ tỏa ra cả
2 phía của bán kính). Trừ đi nửa độ dày đảm bảo toàn bộ vòng tròn nằm gọn trong `size`
được cấp.

## 4. `shouldRepaint()` — Bài 4 chứng minh bằng số liệu thật, không chỉ lý thuyết

Code thật — `lib/tuan3_ngay1/bai4_should_repaint_demo_screen.dart` — thay vì chỉ giải
thích suông, bài này **đo đếm số lần `paint()` thực sự chạy** bằng `ValueNotifier`:

```dart
class _CirclePainter extends CustomPainter {
  final double phanTram;
  final bool alwaysRepaint;
  final ValueNotifier<int> paintCounter; // đếm bằng ValueNotifier, không setState

  @override
  void paint(Canvas canvas, Size size) {
    paintCounter.value++; // tăng counter mà không trigger rebuild của parent widget
    ...
  }

  @override
  bool shouldRepaint(_CirclePainter old) {
    if (alwaysRepaint) return true;          // ❌ luôn vẽ lại
    return old.phanTram != phanTram;         // ✅ chỉ vẽ lại khi % đổi
  }
}
```

**Vì sao dùng `ValueNotifier` thay vì `setState` ngay trong `paint()`?** Vì `paint()` chạy
trong quá trình render — gọi `setState()` ở đây sẽ kích hoạt rebuild lại chính Widget cha,
dẫn tới `paint()` chạy lại, dẫn tới `setState()` lại... tạo **vòng lặp vô hạn**. Dùng
`ValueNotifier` + `ValueListenableBuilder` chỉ rebuild đúng phần hiển thị con số, không
đụng tới toàn bộ cây Widget.

Màn hình đặt 2 `CustomPaint` chạy song song — 1 bên `alwaysRepaint: true` (đỏ), 1 bên
`alwaysRepaint: false` (xanh) — cùng nhận 1 giá trị `_phanTram`. Có nút **"Rebuild KHÔNG
đổi %"**: bấm nhiều lần, quan sát số đếm:

- Bên đỏ (`shouldRepaint` luôn `true`): số lần `paint()` **tăng theo mỗi lần rebuild**, dù
  `phanTram` không đổi — vẽ lại lãng phí.
- Bên xanh (`shouldRepaint` đúng): số lần `paint()` **không tăng thêm** khi rebuild mà
  `phanTram` không đổi — chỉ vẽ lại khi kéo Slider (giá trị thực sự thay đổi).

Đây là bằng chứng trực quan, đo được bằng số cụ thể, cho nguyên tắc: **`shouldRepaint`
sai (luôn `true`) gây lãng phí CPU thật sự**, không chỉ là lý thuyết suông — liên hệ trực
tiếp tới phần tối ưu rebuild đã học ở Ngày 4 Tuần 2 (`const` constructor).

## 5. Kết hợp `CustomPainter` với `AnimationController` (explicit animation)

Trong `bai3_custom_painter_screen.dart`, vòng tròn chính chạy animation khi đổi task:

```dart
_controller = AnimationController(vsync: this, duration: const Duration(milliseconds: 800));
_animation = Tween<double>(begin: 0, end: 1).animate(CurvedAnimation(parent: _controller, curve: Curves.easeOut));
_controller.forward();

// Trong build():
AnimatedBuilder(
  animation: _animation,
  builder: (context, _) => CustomPaint(
    size: const Size(180, 180),
    painter: VongTronTienDoPainter(
      phanTram: _phanTram * _animation.value, // nhân với animation.value → vẽ dần từ 0 tới % thật
      mauTienDo: _mauTienDo,
      doDay: 14,
    ),
  ),
)
```

Đây là `AnimationController` "explicit" — khác `AnimatedContainer` (implicit, đã học
Tuần 2 Ngày 3) — bạn tự kiểm soát giá trị animation (`_animation.value` chạy từ 0 → 1) và
tự quyết định vẽ gì ở từng frame (`phanTram: _phanTram * _animation.value` khiến vòng
tròn "chạy" dần lên đúng % thay vì nhảy thẳng tới kết quả).

## Bài tập

**Bài 3 (đã có sẵn — `bai3_custom_painter_screen.dart`):** Tự chạy, bấm chọn giữa 3 task
category để xem vòng tròn tiến độ animate lại từ đầu mỗi lần chọn (`_controller.forward(from: 0)`).

**Bài 4 (đã có sẵn — `bai4_should_repaint_demo_screen.dart`):** Bấm nút "Rebuild KHÔNG
đổi %" 10 lần, ghi lại số đếm `paint()` của cả 2 bên (đỏ/xanh) — xác nhận bên đỏ tăng đúng
10, bên xanh không tăng. Kéo Slider 1 lần — xác nhận cả 2 bên đều tăng thêm đúng 1.

##  Code Example (repo `flutter.git`)

> Xem nguyên các file tại branch `CustomPainter`:
> [`bai3_custom_painter_screen.dart`](https://github.com/SangLeSoftZ/flutter/blob/CustomPainter/testflutter/lib/tuan3_ngay1/bai3_custom_painter_screen.dart) ·
> [`bai4_should_repaint_demo_screen.dart`](https://github.com/SangLeSoftZ/flutter/blob/CustomPainter/testflutter/lib/tuan3_ngay1/bai4_should_repaint_demo_screen.dart)

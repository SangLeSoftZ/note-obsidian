---
title: "Ngày 16 (Tuần 3 - Ngày 5, Sáng) — SpringSimulation: Animation vật lý phản hồi vận tốc"
track: flutter
day: 16
topic: springsimulation-physics-animation
tags: [flutter, springsimulation, physics, animationcontroller, curves]
source_goc: Tuan3-Ngay5-SpringSimulation-ShellRoute-GraphQL.md
---

# Ngày 16 (Tuần 3 - Ngày 5, Sáng) — SpringSimulation: Animation vật lý phản hồi vận tốc

## 1. Vấn đề: `Curves` cố định không phản ứng theo cách người dùng thao tác

Animation thông thường (`Curves.easeInOut`, đã học Tuần 2 Ngày 3) chạy theo 1 đường cong
**cố định, tính sẵn từ đầu** — dù người dùng vuốt nhanh hay chậm, animation vẫn chạy đúng
1 `duration` y hệt. `SpringSimulation` mô phỏng **vật lý lò xo thật** — tính toán liên tục
dựa trên vị trí, vận tốc, độ "nảy" (damping) hiện tại, tạo cảm giác tự nhiên hơn nhiều cho
tương tác kéo-thả, vuốt-bung.

## 2. Bài 1A — repo chọn cách vuốt trái/phải, có 3 preset để so sánh trực quan

Code thật — `lib/tuan3_ngay5/bai1a_spring_simulation_screen.dart`. Điểm khác đầu tiên so
với ví dụ lý thuyết gốc: dùng `AnimationController.unbounded`, không phải
`AnimationController` thường:

```dart
// AnimationController.unbounded: không giới hạn 0.0-1.0
// vì vị trí X có thể là bất kỳ giá trị nào (pixel), không phải tỉ lệ 0-1
late final AnimationController _controller;

@override
void initState() {
  super.initState();
  _controller = AnimationController.unbounded(vsync: this)
    ..addListener(() {
      setState(() => _viTriX = _controller.value);
    });
}
```

`AnimationController` mặc định luôn giới hạn giá trị trong khoảng `[0.0, 1.0]` — hợp lý
cho animation kiểu "tiến độ" (mờ dần, phóng to...), nhưng **không phù hợp** khi giá trị
cần theo dõi là **vị trí pixel thật** trên màn hình (có thể là -300, 150, 820...).
`.unbounded` bỏ giới hạn này đi.

**3 preset để tự cảm nhận khác biệt** — chi tiết hay hơn ví dụ lý thuyết gốc (chỉ có 1 bộ
tham số cố định):

```dart
final List<Map<String, dynamic>> _presets = [
  {'name': 'Lò xo mềm\n(nảy nhiều)', 'mass': 1.0, 'stiffness': 50.0,  'damping': 3.0,  'color': Colors.red},
  {'name': 'Cân bằng\n(tự nhiên)',   'mass': 1.0, 'stiffness': 200.0, 'damping': 15.0, 'color': Colors.blue},
  {'name': 'Lò xo cứng\n(về nhanh)', 'mass': 1.0, 'stiffness': 500.0, 'damping': 40.0, 'color': Colors.green},
];
```

3 tham số cốt lõi của lò xo:
- `mass` (khối lượng) — lò xo "nặng" hay "nhẹ", ảnh hưởng quán tính.
- `stiffness` (độ cứng) — càng cao, lò xo càng "cứng", kéo về vị trí gốc nhanh, dứt khoát.
- `damping` (độ cản/giảm chấn) — càng thấp, càng "nảy" nhiều lần trước khi dừng hẳn; càng
  cao, dừng nhanh, gần như không nảy.

So 3 preset: "Lò xo mềm" (`stiffness: 50, damping: 3`) sẽ nảy qua lại nhiều lần trước khi
dừng; "Lò xo cứng" (`stiffness: 500, damping: 40`) gần như bật thẳng về vị trí gốc, hầu
như không nảy. Đổi qua đổi lại 3 preset ngay trên UI (không cần sửa code) là cách tốt
nhất để tự cảm nhận vai trò của từng tham số.

## 3. `_khiThaTay` — vận tốc thả tay chính là đầu vào khác biệt cốt lõi

```dart
void _khiThaTay(DragEndDetails details) {
  final preset = _presets[_selectedPreset];
  final springDesc = SpringDescription(
    mass: preset['mass'] as double,
    stiffness: preset['stiffness'] as double,
    damping: preset['damping'] as double,
  );

  final simulation = SpringSimulation(
    springDesc,
    _viTriX,                             // vị trí bắt đầu (đang ở đó khi thả tay)
    0,                                    // vị trí đích (về gốc = 0)
    details.velocity.pixelsPerSecond.dx,  // vận tốc thả tay — điểm khác biệt vs Curves
  );

  // animateWith: chạy theo simulation vật lý, KHÔNG phải duration cố định
  _controller.animateWith(simulation);
}
```

**Chi tiết khác với ví dụ lý thuyết gốc:** ví dụ gốc chia `details.velocity.pixelsPerSecond.dx`
cho `1000` trước khi truyền vào; code thật truyền **nguyên giá trị** không chia. Cả 2 đều
hợp lệ về mặt kỹ thuật — khác biệt chỉ là đơn vị/độ nhạy cảm nhận được (chia nhỏ hơn khiến
lò xo "phản ứng" nhẹ nhàng hơn với cùng 1 lực vuốt). Không có con số "đúng tuyệt đối" —
đây là tham số cần tự chỉnh theo cảm giác UX mong muốn, giống việc chỉnh `stiffness`/
`damping`.

**Điểm mấu chốt:** `SpringSimulation` nhận **vận tốc thực tế lúc buông tay**
(`details.velocity`) làm đầu vào — vuốt nhanh thì lò xo "bật" mạnh và nhanh hơn (có thể
"vọt qua" vị trí 0 rồi mới quay lại), vuốt chậm thì nhẹ nhàng hơn — đúng cảm giác vật lý
thật, điều mà `Curves.easeInOut` cố định **không** làm được.

## 4. So sánh nhanh khi nào dùng SpringSimulation, khi nào dùng Lottie

| | SpringSimulation | Lottie |
|---|---|---|
| Phù hợp cho | Tương tác **phản hồi cử chỉ người dùng** (kéo, vuốt, buông tay) | Hiệu ứng **trang trí phức tạp có sẵn**, không cần tương tác vật lý thực |
| Ai tạo animation | Lập trình viên tự viết bằng code | Designer làm sẵn bằng After Effects, lập trình viên chỉ nhúng vào |
| Độ phức tạp hình ảnh | Đơn giản (di chuyển, scale 1 widget) | Có thể rất phức tạp (nhân vật, nhiều lớp, màu sắc) |

Repo hôm nay chọn **Lựa chọn A (SpringSimulation)** cho Bài 1 — chưa có file Lottie nào
trong nhánh code; nếu muốn thử Lựa chọn B, tự làm theo hướng dẫn tài liệu gốc (tải file
`.json` từ LottieFiles, thêm package `lottie`, hiển thị khi Task được đánh dấu hoàn
thành).

## Bài tập

**Bài 1 (đã có sẵn — `bai1a_spring_simulation_screen.dart`):** Chạy màn hình, vuốt thẻ
sang trái/phải rồi thả tay ở cả 3 preset (mềm/cân bằng/cứng) — quan sát khác biệt về số
lần nảy và tốc độ về vị trí gốc. Thử vuốt rất nhanh so với vuốt chậm ở cùng 1 preset để
cảm nhận vai trò của vận tốc thả tay.

## Code Example (repo `flutter.git`)

> Xem nguyên file tại branch `NestedNavigator--ShellRoute` (đã gộp cả `SpringSimulation`):
> [`bai1a_spring_simulation_screen.dart`](https://github.com/SangLeSoftZ/flutter/blob/NestedNavigator--ShellRoute/testflutter/lib/tuan3_ngay5/bai1a_spring_simulation_screen.dart)


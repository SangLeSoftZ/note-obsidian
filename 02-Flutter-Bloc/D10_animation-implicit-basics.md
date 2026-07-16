---
title: "Ngày 10 / Tuần 2 - Ngày 3 (Sáng) — Flutter: Animation cơ bản (Implicit Animation)"
track: flutter
day: 10
topic: animation-implicit-basics
tags: [flutter, animation, animatedcontainer, animatedopacity, animatedswitcher]
source_goc: Tuan2-Ngay3-Animation-Hero-SpecificationAPI.md (buổi sáng)
---

# Ngày 10 / Tuần 2 - Ngày 3 (Sáng) — Flutter: Animation cơ bản (Implicit Animation)

# BUỔI SÁNG — Animation cơ bản (Implicit Animation)

## 1. Vấn đề thực tế Animation giải quyết

Khi 1 giá trị UI đổi đột ngột (VD: `width` từ 100 → 200, `opacity` từ 0 → 1), Flutter mặc định vẽ lại ngay lập tức trong 1 frame — người dùng thấy "giật cục". **Implicit Animation** cho phép chỉ cần đổi giá trị đích, Flutter tự động nội suy mượt qua các frame trung gian, không cần tự viết `AnimationController` thủ công.

**Phân biệt Implicit vs Explicit animation:**
- **Implicit** (`AnimatedContainer`, `AnimatedOpacity`, `AnimatedAlign`...): chỉ cần đổi state → widget tự chạy animation đến giá trị mới. Đủ dùng cho 80% trường hợp UI thông thường.
- **Explicit** (`AnimationController` + `Tween` + `AnimatedBuilder`): tự kiểm soát chi tiết (start/stop/reverse/lặp lại) — phức tạp hơn, dùng khi cần điều khiển tinh vi (chưa nằm trong phạm vi hôm nay).

## 2. `AnimatedContainer` — animation "tất cả trong 1"

```dart
class AnimatedBoxDemo extends StatefulWidget {
  const AnimatedBoxDemo({super.key});
  @override
  State<AnimatedBoxDemo> createState() => _AnimatedBoxDemoState();
}

class _AnimatedBoxDemoState extends State<AnimatedBoxDemo> {
  bool _phongTo = false;

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: () => setState(() => _phongTo = !_phongTo),
      child: AnimatedContainer(
        duration: const Duration(milliseconds: 300),
        curve: Curves.easeInOut,
        width: _phongTo ? 200 : 100,
        height: _phongTo ? 200 : 100,
        decoration: BoxDecoration(
          color: _phongTo ? Colors.blue : Colors.red,
          borderRadius: BorderRadius.circular(_phongTo ? 40 : 8),
        ),
      ),
    );
  }
}
```

**Cơ chế bên dưới:** mỗi khi `build()` chạy lại (do `setState`), `AnimatedContainer` so sánh giá trị thuộc tính **cũ** với giá trị **mới** — nếu khác nhau, nó tự tạo 1 `Tween` ngầm và animate từ cũ → mới trong đúng `duration`. Không cần gọi `.animateTo()` hay quản lý `AnimationController`.

**`curve` là gì?** Hàm mô tả tốc độ animation theo thời gian. `Curves.easeInOut` (chậm → nhanh → chậm) tạo cảm giác tự nhiên hơn `Curves.linear` (tốc độ đều, cứng).

## 3. `AnimatedOpacity` — hiệu ứng ẩn/hiện mượt

```dart
class FadeDemo extends StatefulWidget {
  const FadeDemo({super.key});
  @override
  State<FadeDemo> createState() => _FadeDemoState();
}

class _FadeDemoState extends State<FadeDemo> {
  bool _hienThi = true;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        AnimatedOpacity(
          opacity: _hienThi ? 1.0 : 0.0,
          duration: const Duration(milliseconds: 400),
          child: const Icon(Icons.favorite, size: 80, color: Colors.pink),
        ),
        TextButton(
          onPressed: () => setState(() => _hienThi = !_hienThi),
          child: const Text('Ẩn/Hiện'),
        ),
      ],
    );
  }
}
```

**Lưu ý dễ nhầm:** `AnimatedOpacity` chỉ đổi **độ trong suốt** — widget vẫn **chiếm chỗ** trong layout kể cả khi `opacity: 0` (chỉ "vô hình", không "biến mất"). Muốn không chiếm không gian khi ẩn, cần kết hợp thêm `Visibility` — lỗi hay gặp: tưởng `opacity: 0` tự co layout nhưng khoảng trống vẫn còn.

## 4. Áp dụng thực tế: chuyển cảnh khi load dữ liệu

```dart
AnimatedSwitcher(
  duration: const Duration(milliseconds: 300),
  child: dangTai
      ? const CircularProgressIndicator(key: ValueKey('loading'))
      : Text('Dữ liệu đã tải xong', key: const ValueKey('content')),
)
```

`AnimatedSwitcher` tự động fade-out widget cũ + fade-in widget mới khi `child` đổi loại/`key`. Bắt buộc mỗi trạng thái con phải có `key` riêng, nếu không `AnimatedSwitcher` không biết đâu là widget "mới" để animate.

## Bài tập

**Bài 1:** Tạo 1 nút bấm dùng `AnimatedContainer` đổi màu + bo góc khi nhấn, `duration` 300ms, `curve: Curves.easeInOut`.

**Bài 2:** Áp dụng `AnimatedOpacity` cho 1 thông báo lỗi trong màn hình Login đã có — chỉ hiện khi có lỗi, fade mượt thay vì xuất hiện/biến mất đột ngột.

---

##  Code Example (repo `flutter`)

>  Repo code Flutter riêng (`SangLeSoftZ/flutter`) — bổ sung link khi đã đẩy code thật. 
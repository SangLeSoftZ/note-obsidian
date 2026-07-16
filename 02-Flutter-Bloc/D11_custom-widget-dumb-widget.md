---
title: "Ngày 11 (Tuần 2 - Ngày 4, Sáng) — Custom Widget tái sử dụng (DRY + Dumb Widget)"
track: flutter
day: 11
topic: custom-widget-dumb-widget
tags: [flutter, custom-widget, dry, widget-test, const-constructor]
source_goc: Tuan2_Ngay4_TaiLieu.md
---

# Ngày 11 (Tuần 2 - Ngày 4, Sáng) — Custom Widget tái sử dụng (DRY + Dumb Widget)

## 1. Vấn đề thật trong code: `build()` lặp lại cấu trúc `Card` + `Icon` + `Text`

Trước khi tách, mỗi item trong danh sách Task đều phải viết lại y hệt cấu trúc UI. Đây
đúng là nguyên tắc **DRY (Don't Repeat Yourself)** áp dụng cho UI — cùng lý do vì sao
tách Service/Repository ở Spring Boot Tuần 1: **sửa 1 chỗ, áp dụng mọi nơi**.

Repo tách ra 2 widget riêng: `TaskCard` (hiển thị 1 task) và `TaskSearchBar` (thanh tìm
kiếm) — dùng lại được ở bất kỳ màn hình nào cần.

## 2. `TaskCard` — "dumb widget" thật sự trong code

Code thật — `lib/tuan2_ngay4/task_card.dart`:

```dart
class TaskCard extends StatelessWidget {
  final Task task;
  final VoidCallback? onXoa;       // nullable — không phải màn hình nào cũng có nút xóa
  final VoidCallback? onTap;       // nullable — không phải màn hình nào cũng navigate
  final bool showDeleteButton;

  const TaskCard({
    super.key,
    required this.task,
    this.onXoa,
    this.onTap,
    this.showDeleteButton = true,
  });

  Color get _statusColor {
    switch (task.trangThai) {
      case 'HOAN_THANH': return Colors.green;
      case 'DANG_LAM':   return Colors.orange;
      default:           return Colors.grey;
    }
  }
  // ...tương tự _statusLabel, _statusIcon
}
```

Điểm đáng chú ý so với thiết kế "tối thiểu" trong tài liệu gốc:

- **`onXoa`/`onTap` đều là `VoidCallback?` (nullable)**, không bắt buộc (`required`) — vì
  không phải màn hình nào dùng `TaskCard` cũng cần nút xóa hay cần điều hướng khi tap
  (ví dụ 1 màn hình chỉ hiển thị task, không cho sửa/xóa). Đây là cách làm widget **linh
  hoạt hơn** so với ví dụ cơ bản chỉ có 1 `onXoa` bắt buộc.
- **`showDeleteButton`**: cho phép màn hình cha tắt hẳn nút xóa mà không cần truyền
  `onXoa: null` (2 cờ độc lập — có thể có `onXoa` nhưng ẩn nút, hoặc không có `onXoa` gì
  cả).
- Widget tự tính `_statusColor`/`_statusLabel`/`_statusIcon` dựa theo `task.trangThai`
  — đây vẫn là "dumb" đúng nghĩa vì logic này chỉ là **cách trình bày** dữ liệu đã có
  sẵn trong `task`, không phải logic nghiệp vụ (không gọi API, không gọi Cubit).

**Nguyên tắc cốt lõi (không đổi):** `TaskCard` không tự gọi `context.read<TaskCubit>()`.
Nó chỉ nhận dữ liệu qua constructor → **tái sử dụng được ở bất kỳ đâu** (kể cả màn hình
không liên quan `TaskCubit`), và **dễ test** (không cần setup Cubit giả).

## 3. `TaskSearchBar` — callback báo ra ngoài, không tự lọc

Code thật — `lib/tuan2_ngay4/task_search_bar.dart`:

```dart
class TaskSearchBar extends StatefulWidget {
  final ValueChanged<String> onSearch; // callback khi text thay đổi
  final String hintText;
  final VoidCallback? onClear;
  ...
}
```

Lưu ý: `TaskSearchBar` là `StatefulWidget` (khác `TaskCard` là `StatelessWidget`) — vì nó
cần tự quản lý `TextEditingController` và trạng thái "có nội dung hay không" (để hiện/ẩn
nút clear). Nhưng **logic tìm kiếm thật sự** (lọc danh sách) vẫn **không** nằm trong
widget này — nó chỉ gọi `widget.onSearch(text)` để "báo" ra ngoài, còn màn hình cha
(`CustomWidgetScreen`) mới là nơi quyết định lọc như thế nào. Đây vẫn đúng tinh thần
"dumb widget": có state nội bộ (UI state) nhưng không có business logic.

## 4. Màn hình cha `CustomWidgetScreen` — chỉ lo dữ liệu, không lo UI chi tiết

Code thật cho thấy rõ sự phân chia trách nhiệm:

```dart
// Logic lọc nằm ở màn hình cha — không nằm trong TaskSearchBar
// Hỗ trợ: không phân biệt hoa/thường + không phân biệt dấu tiếng Việt
void _onSearch(String tuKhoa) {
  setState(() {
    _tuKhoa = tuKhoa;
    _filteredTasks = tuKhoa.isEmpty
        ? _allTasks
        : _allTasks
            .where((t) =>
                _boDau(t.tieuDe).contains(_boDau(tuKhoa)) ||
                _boDau(t.moTa).contains(_boDau(tuKhoa)))
            .toList();
  });
}
```

Chi tiết thú vị: hàm `_boDau()` bỏ dấu tiếng Việt trước khi so sánh, để gõ `"hoc"` vẫn
tìm ra `"Học Spring Boot"`. Đây không nằm trong tài liệu lý thuyết gốc nhưng là ví dụ thực
tế tốt cho việc **tách đúng trách nhiệm**: `TaskSearchBar` không cần biết gì về tiếng Việt
có dấu hay không — nó chỉ đưa `String` ra ngoài, còn xử lý "thế nào là khớp" là việc của
màn hình cha.

`ListView.builder` dùng lại `TaskCard`:

```dart
ListView.builder(
  itemCount: _filteredTasks.length,
  itemBuilder: (context, index) => TaskCard(
    task: _filteredTasks[index],
    onXoa: () => _onXoa(_filteredTasks[index]),
    onTap: () { /* show SnackBar demo */ },
  ),
)
```

## 5. `const` constructor — không chỉ là thói quen

```dart
const TaskCard({super.key, required this.task, this.onXoa, this.onTap, this.showDeleteButton = true});
```

Khi Widget đánh dấu `const` và tham số truyền vào không đổi giữa các lần rebuild, Flutter
**biết chắc** widget đó không cần build lại — bỏ qua hoàn toàn bước build, tăng hiệu năng
khi danh sách dài. Đây là nội dung "tối ưu hiệu năng" từ Tuần 1, áp dụng cụ thể khi tách
Custom Widget.

## 6. Widget Test cho "dumb widget" — dễ vì không cần Cubit/Provider giả

Code thật — `test/tuan2_ngay4/task_card_test.dart` minh chứng đúng lý do "dumb widget dễ
test":

```dart
const taskMau = Task(
  id: 1,
  tieuDe: 'Học Flutter Widget Test',
  moTa: 'Viết test cho TaskCard',
  trangThai: 'DANG_LAM',
);

Widget buildTaskCard({required Task task, VoidCallback? onXoa, VoidCallback? onTap, bool showDeleteButton = true}) {
  return MaterialApp(
    home: Scaffold(body: TaskCard(task: task, onXoa: onXoa, onTap: onTap, showDeleteButton: showDeleteButton)),
  );
}

testWidgets('gọi onXoa khi nhấn nút xóa', (tester) async {
  bool daGoi = false;
  await tester.pumpWidget(buildTaskCard(task: taskMau, onXoa: () => daGoi = true));
  await tester.tap(find.byIcon(Icons.delete_outline));
  await tester.pump();
  expect(daGoi, isTrue);
});
```

Không có dòng nào setup `BlocProvider`, `GetIt`, hay mock API — toàn bộ test chỉ xoay
quanh: truyền `Task` giả → kiểm tra UI hiển thị đúng, hoặc giả lập tap → xác nhận callback
được gọi. Đây chính là lợi ích thực tế (không chỉ lý thuyết) của việc tách "dumb widget".

## Bài tập

**Bài 1 (đã có sẵn trong repo — `task_card.dart` + `task_search_bar.dart`):** Đối chiếu
lại 2 widget này với nguyên tắc "dumb widget" — xác nhận đúng là cả 2 đều không tự gọi
Cubit/API bên trong.

**Bài 2 (đã có sẵn trong repo — `task_card_test.dart`):** Đọc lại 8 test case đã viết,
thử tự thêm 1 test case mới: kiểm tra `TaskCard` **không** hiện `Chip` trạng thái sai khi
`trangThai` là 1 giá trị lạ không khớp `HOAN_THANH`/`DANG_LAM` (rơi vào nhánh `default`).

##  Code Example (repo `flutter.git`)

> Xem nguyên các file tại branch `tuan2`:
> [`task_card.dart`](https://github.com/SangLeSoftZ/flutter/blob/tuan2/testflutter/lib/tuan2_ngay4/task_card.dart) ·
> [`task_search_bar.dart`](https://github.com/SangLeSoftZ/flutter/blob/tuan2/testflutter/lib/tuan2_ngay4/task_search_bar.dart) ·
> [`custom_widget_screen.dart`](https://github.com/SangLeSoftZ/flutter/blob/tuan2/testflutter/lib/tuan2_ngay4/custom_widget_screen.dart) ·
> [`task_card_test.dart`](https://github.com/SangLeSoftZ/flutter/blob/tuan2/testflutter/test/tuan2_ngay4/task_card_test.dart)

---
title: "Ngày 17 (Tuần 4 - Ngày 1, Chiều) — BlocSelector, buildWhen, listenWhen"
track: flutter
day: 17
topic: blocselector-buildwhen-listenwhen
tags: [flutter, bloc, blocselector, buildwhen, listenwhen, rebuild-optimization]
source_goc: Tuan4-Ngay1-Dart3SealedClass-BlocSelector-N1Query.md
---

# Ngày 17 (Tuần 4 - Ngày 1, Chiều) — BlocSelector, buildWhen, listenWhen

## 1. Cách dạy hay trong repo: đo bằng số đếm thật, không chỉ nói lý thuyết

Code thật — `lib/tuan3_ngay7/bai4_bai5_bai6_bloc_selector_screen.dart` — dùng 4
`ValueNotifier<int>` độc lập để **đếm chính xác** số lần mỗi cơ chế thực sự được gọi,
hiển thị song song ngay trên UI — đúng tinh thần đo lường bằng số liệu đã thấy ở
`shouldRepaint` (Tuần 3 Ngày 1) và debounce (Tuần 3 Ngày 2):

```dart
// ValueNotifier: tăng giá trị mà KHÔNG trigger rebuild toàn widget
// ValueListenableBuilder bên dưới chỉ rebuild đúng Text hiển thị số
final _cntTho = ValueNotifier(0);        // BlocBuilder thô
final _cntSelector = ValueNotifier(0);   // BlocSelector (Bài 5)
final _cntBuildWhen = ValueNotifier(0);  // buildWhen (Bài 4)
final _cntListener = ValueNotifier(0);   // listenWhen calls (Bài 6)
```

## 2. Bài 4 — `buildWhen`: chỉ rebuild khi TYPE state đổi

```dart
BlocBuilder<TaskSealedCubit, TaskSealedState>(
  // BÀI 4: chỉ rebuild khi TYPE state đổi
  // Nếu emit TaskLoaded 2 lần liên tiếp → KHÔNG rebuild lần 2
  buildWhen: (old, current) {
    return old.runtimeType != current.runtimeType;
  },
  builder: (context, state) {
    WidgetsBinding.instance.addPostFrameCallback((_) {
      _cntBuildWhen.value++;
    });
    return _buildContent(state);
  },
)
```

**Chi tiết kỹ thuật đáng chú ý:** đếm số lần build bằng
`WidgetsBinding.instance.addPostFrameCallback((_) { ... })` thay vì tăng trực tiếp trong
`builder`. Lý do: `builder` chạy trong quá trình Flutter đang build lại cây widget — nếu
gọi thẳng `_cntBuildWhen.value++` ở đây, `ValueNotifier` sẽ bắn thông báo thay đổi
**ngay trong lúc build đang diễn ra**, có thể gây lỗi hoặc hành vi không xác định.
`addPostFrameCallback` hoãn việc tăng số đếm tới **ngay sau khi frame hiện tại build xong**
— an toàn hơn nhiều.

`buildWhen` so sánh `old.runtimeType != current.runtimeType` — chỉ rebuild khi **loại**
state đổi (VD từ `TaskSealedLoading` sang `TaskSealedLoaded`), bỏ qua rebuild nếu cùng
loại state được emit lại (VD `TaskSealedLoaded` với danh sách khác nhưng cấu trúc UI
không cần vẽ lại toàn bộ).

## 3. Bài 5 — `BlocSelector`: chỉ lấy đúng 1 giá trị cần thiết từ state

```dart
// BÀI 5: BlocSelector — title CHỈ rebuild khi số lượng task đổi
title: BlocSelector<TaskSealedCubit, TaskSealedState, int>(
  selector: (state) {
    _cntSelector.value++;
    return state is TaskSealedLoaded ? state.danhSach.length : 0;
  },
  builder: (context, soLuong) => Text('Tasks ($soLuong)'),
),
```

`BlocSelector<Cubit, State, T>` — tham số kiểu thứ 3 (`int`) là kiểu giá trị **đã chọn
lọc** ra từ state, không phải cả `TaskSealedState`. `selector` chạy **mỗi khi state đổi**
để tính lại giá trị `int` này, nhưng `builder` (phần dựng UI thật) **chỉ chạy khi giá trị
`int` trả về khác với lần trước** — nếu state đổi nhưng `danhSach.length` vẫn y hệt (VD
chỉ đổi `trangThai` của 1 Task mà không đổi tổng số lượng), `builder` **không chạy lại**,
dù `selector` vẫn chạy.

**`BlocSelector` khác `BlocBuilder + buildWhen` ở đâu?** `buildWhen` so sánh **toàn bộ
state cũ vs mới** (thường theo `runtimeType` hoặc `==`) để quyết định build hay không, còn
`BlocSelector` **trích xuất trước 1 giá trị cụ thể** rồi mới so sánh đúng giá trị đó —
phù hợp hơn khi UI chỉ cần hiển thị **1 con số/chuỗi đơn giản** suy ra từ state phức tạp
(ở đây là đếm số Task), không cần biết toàn bộ state đổi ra sao.

## 4. Bài 6 — `listenWhen`: chỉ phản ứng đúng lúc cần, tránh SnackBar lặp

```dart
BlocListener<TaskSealedCubit, TaskSealedState>(
  // BÀI 6: listenWhen — chỉ phản ứng khi state MỚI là Error
  listenWhen: (old, current) {
    _cntListener.value++;
    return current is TaskSealedError;
  },
  listener: (context, state) {
    if (state is TaskSealedError) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Lỗi: ${state.thongDiep}'), backgroundColor: Colors.red),
      );
    }
  },
  child: Scaffold(...),
)
```

**Vì sao cần `listenWhen` ở đây dù `listener` đã có `if (state is TaskSealedError)`?**
Không có `listenWhen`, hàm `listener` vẫn được **gọi mỗi khi state đổi** (dù có kiểm tra
`if` bên trong) — `_cntListener.value++` trong `listenWhen` tăng ở **mọi lần** state đổi,
trong khi `listener` (hiện SnackBar) chỉ thực sự làm gì khi đúng `TaskSealedError`. Đặt
điều kiện lọc ngay ở `listenWhen` giúp tránh trường hợp phức tạp hơn: nếu logic bên trong
`listener` nặng hơn (gọi thêm API, ghi log...), lọc sớm ở `listenWhen` tránh chạy logic đó
không cần thiết mỗi lần state đổi bất kỳ.

## 5. Giao diện so sánh song song 2 cột — cách kiểm chứng trực quan nhất

Màn hình chia 2 cột `BlocBuilder` thô (không `buildWhen`) và `BlocBuilder + buildWhen`,
chạy **cùng lúc, cùng 1 Cubit** — bấm Refresh nhiều lần, quan sát bảng đếm phía trên:

```dart
'Nhấn Refresh nhiều lần:\n'
'→ Thô: tăng nhiều hơn\n'
'→ buildWhen/Selector: tăng ÍT hơn khi state không đổi loại',
```

Đây là cách kiểm chứng mạnh hơn nhiều so với chỉ đọc lý thuyết: 2 `BlocBuilder` cùng nhận
1 state, nhưng số lần `builder()` thực sự chạy **khác nhau rõ rệt** — cột "❌ BlocBuilder
thô" luôn tăng đúng bằng số lần state đổi, cột "✅ buildWhen" chỉ tăng khi loại state thực
sự khác trước.

## 6. Tổng kết khi nào dùng cái nào

| Cơ chế | Dùng khi |
|---|---|
| `BlocBuilder` thường | UI cần hiển thị đầy đủ theo mọi lần đổi state |
| `BlocBuilder` + `buildWhen` | Vẫn cần build cả UI phức tạp, nhưng muốn bỏ qua 1 số lần đổi state không cần vẽ lại |
| `BlocSelector` | Chỉ cần hiển thị **1 giá trị đơn giản** suy ra từ state (đếm, tên, boolean...) |
| `BlocListener` + `listenWhen` | Cần phản ứng 1 lần (SnackBar, điều hướng, log) đúng lúc, tránh phản ứng lặp lại không cần thiết |

## Bài tập

**Bài 4 (đã có sẵn — `bai4_bai5_bai6_bloc_selector_screen.dart`):** Chạy
`BlocSelectorScreen`, bấm Refresh nhiều lần liên tiếp — so sánh số đếm "❌ BlocBuilder thô"
vs "✅ buildWhen (Bài 4)" trên bảng thống kê, xác nhận buildWhen tăng ít hơn.

**Bài 5 (đã có sẵn, xem `title: BlocSelector<...>`):** Quan sát counter "✅ BlocSelector
(Bài 5)" — so sánh với số lần state thực sự đổi để thấy `builder` của `BlocSelector` được
gọi ít hơn `selector`.

**Bài 6 (đã có sẵn, xem `BlocListener` bọc ngoài `Scaffold`):** Bấm Refresh nhiều lần
(một số lần trả lỗi giả lập nếu API lỗi), quan sát SnackBar chỉ hiện đúng khi có lỗi thật
sự, không lặp lại vô ích ở các lần đổi state khác.

## Code Example (repo `flutter.git`)
> Xem nguyên file tại branch `BlocSelector--buildWhen-listenWhen`:
> [`bai4_bai5_bai6_bloc_selector_screen.dart`](<https://github.com/SangLeSoftZ/flutter/blob/BlocSelector--buildWhen-listenWhen/testflutter/lib/tuan3_ngay7/bai4_bai5_bai6_bloc_selector_screen.dart>)

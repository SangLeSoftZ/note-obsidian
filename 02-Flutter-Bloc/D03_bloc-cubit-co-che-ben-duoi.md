---
title: "Ngày 3 — flutter_bloc: Cơ chế bên dưới Stream & InheritedWidget"
track: flutter
day: 3
topic: bloc-internals
tags: [flutter, bloc, cubit, stream, inheritedwidget, bloc_test]
source_goc: Ngay3_LyThuyetSau.md (Phần 2)
---

# Ngày 3 — flutter_bloc: Cơ chế bên dưới Stream & InheritedWidget

# PHẦN 2 — flutter_bloc: CƠ CHẾ BÊN DƯỚI STREAM & INHERITEDWIDGET

### `Cubit` thực chất là gì, xây trên nền tảng nào?

Bạn đã học Stream ở Ngày 1. Sự thật: **`Cubit<State>` bản chất là 1 lớp bọc quanh `StreamController<State>`**. Khi bạn gọi `emit(stateMoi)`, bên trong nó gọi `streamController.add(stateMoi)` — đúng cơ chế Stream bạn đã học, không có gì "ma thuật" cả.

```dart
// Cách BlocBuilder "nghe" Cubit, đơn giản hóa để hiểu bản chất — tương tự thế này:
StreamBuilder<TaskState>(
  stream: cubit.stream,          // Cubit expose ra 1 Stream
  initialData: cubit.state,
  builder: (context, snapshot) {
    return VeGiaoDienTheo(snapshot.data);
  },
)
```

`BlocBuilder` thực tế làm chính xác việc này (kèm tối ưu hiệu năng, so sánh state cũ/mới để tránh rebuild thừa) — nó **không phải 1 khái niệm hoàn toàn mới**, mà là `StreamBuilder` được đóng gói lại cho tiện dùng.

### `BlocProvider` hoạt động dựa trên cơ chế gì của Flutter?

`BlocProvider` (và cả package `provider` nói chung) dựa trên **`InheritedWidget`** — 1 cơ chế có sẵn của Flutter cho phép "truyền dữ liệu xuống toàn bộ cây Widget con" mà không cần truyền tay qua từng constructor.

```
BlocProvider(create: (_) => TaskCubit())
   │
   ├── Widget con A
   │      └── Widget cháu (vẫn lấy được TaskCubit dù A không truyền tay)
   │
   └── Widget con B (cũng lấy được TaskCubit)
```

Đây chính là lý do bạn gọi được `context.read<TaskCubit>()` ở **bất kỳ Widget con nào** bên dưới `BlocProvider`, dù nó nằm sâu bao nhiêu tầng — vì `InheritedWidget` cho phép "tra cứu ngược lên cây Widget" tới `BlocProvider` gần nhất.

**Hệ quả thực tế quan trọng:** nếu bạn gọi `context.read<TaskCubit>()` ở 1 Widget nằm **ngoài** (phía trên) `BlocProvider`, bạn sẽ gặp lỗi runtime `ProviderNotFoundException` — vì widget đó không nằm trong "cây con" của `BlocProvider`, không tìm thấy Cubit nào để trả về.

### Vòng đời (lifecycle) của Cubit — vì sao phải `close()`?

`Cubit` giữ 1 `StreamController` bên trong — tài nguyên này **không tự giải phóng**. Nếu Widget bị hủy (dispose) mà Cubit không được đóng, sẽ gây **memory leak** (rò rỉ bộ nhớ).

May mắn là `BlocProvider` tự động gọi `close()` khi Widget cha bị dispose — **miễn bạn dùng `create:` chứ không truyền sẵn 1 instance có sẵn**:

```dart
// ĐÚNG - BlocProvider tự quản lý và tự close khi không cần nữa
BlocProvider(create: (_) => TaskCubit())

// SAI (tiềm ẩn rủi ro) - nếu instance này được tái sử dụng ở nơi khác,
// BlocProvider vẫn có thể tự đóng nó dù nơi khác còn cần dùng
BlocProvider.value(value: taskCubitDaTonTai)
```

`BlocProvider.value` chỉ nên dùng khi bạn **chắc chắn** Cubit đó được quản lý vòng đời ở nơi khác (ví dụ truyền qua route mới nhưng Cubit gốc vẫn cần sống tiếp).

### Test Cubit bằng `bloc_test` — vì sao dễ test hơn nhiều so với logic nhét trong Widget?

```dart
blocTest<TaskCubit, TaskState>(
  'emit [Loading, Loaded] khi taiDuLieu thành công',
  build: () => TaskCubit(),
  act: (cubit) => cubit.taiDuLieu(),
  expect: () => [isA<TaskLoading>(), isA<TaskLoaded>()],
);
```

**Vì sao test được dễ dàng thế này?** Vì Cubit **hoàn toàn tách biệt khỏi UI** — bạn không cần dựng lên 1 màn hình, không cần giả lập việc bấm nút, chỉ cần gọi trực tiếp hàm và kiểm tra state phát ra đúng thứ tự hay không. Đây chính là "phần thưởng" thực tế của việc tách logic ra khỏi Widget đã nói ở lý thuyết trước — không phải chỉ để "code đẹp" mà để **test được** mà không cần chạy cả app.

### Bài tập đào sâu

**Bài C:** Thử tạo tình huống lỗi: gọi `context.read<TaskCubit>()` ở 1 widget nằm **ngoài** phạm vi `BlocProvider` bao quanh nó. Chụp lại lỗi Flutter báo ra, đọc hiểu thông báo lỗi đó nói gì.

**Bài D:** Viết 1 test đơn giản bằng `bloc_test` cho `FavoriteCubit` (bài tập cũ ở Ngày 3): kiểm tra gọi `toggle()` 2 lần liên tiếp thì state cuối cùng phải quay lại `false`.

---

## 💻 Code Example (repo `Code`)

> ⚠️ Repo [`SangLeSoftZ/Code`](https://github.com/SangLeSoftZ/Code) **hiện chỉ chứa code Java/Spring Boot**
> (package `com.example.demo`), chưa có project Flutter/Dart nào. Khi có repo code Flutter
> riêng, bổ sung link + snippet vào đúng mục này theo cùng quy ước (xem `QUY_UOC_DAT_TEN.md`).

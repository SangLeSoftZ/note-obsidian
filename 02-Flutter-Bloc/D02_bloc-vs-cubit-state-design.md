---
title: "Ngày 2 (Tối, phần Flutter) — Bloc vs Cubit, thiết kế State"
track: flutter
day: 2
topic: bloc-vs-cubit
tags: [flutter, bloc, cubit, state-design]
source_goc: Ngay2_OnTapChiTiet.md (buổi tối - phần lý thuyết BLoC, dòng 221-275 trong file gốc)
---

# Ngày 2 (Tối, phần Flutter) — Bloc vs Cubit, thiết kế State

# BUỔI TỐI — LÝ THUYẾT BLoC/CUBIT + BẮT ĐẦU API CRUD

## 1. Bloc vs Cubit — khác nhau ở đâu?

Cubit là **bản đơn giản hóa** của Bloc. Bloc thêm 1 lớp trung gian là **Event**, giúp code rõ ràng hơn khi logic phức tạp (nhiều nguồn gọi cùng 1 hành động).

```
CUBIT:   UI gọi hàm trực tiếp  --->  Cubit xử lý  --->  emit(State mới)

BLOC:    UI add(Event)  --->  Bloc nhận Event  --->  xử lý  --->  emit(State mới)
```

Ví dụ cùng 1 chức năng "tăng đếm":

```dart
// CUBIT — gọi hàm trực tiếp
class CounterCubit extends Cubit<int> {
  CounterCubit() : super(0);
  void tang() => emit(state + 1);
}
// UI gọi: context.read<CounterCubit>().tang();

// BLOC — phải định nghĩa Event
abstract class CounterEvent {}
class TangEvent extends CounterEvent {}

class CounterBloc extends Bloc<CounterEvent, int> {
  CounterBloc() : super(0) {
    on<TangEvent>((event, emit) => emit(state + 1));
  }
}
// UI gọi: context.read<CounterBloc>().add(TangEvent());
```

**Khi nào chọn cái nào:** Cubit cho tính năng đơn giản (form, toggle, counter). Bloc cho tính năng phức tạp cần log lại lịch sử Event, hoặc nhiều loại hành động tác động lên cùng 1 State (VD: giỏ hàng có Event ThemSanPham, XoaSanPham, ApDungMaGiamGia...).

## 2. State nên thiết kế thế nào?

Thay vì chỉ 1 kiểu dữ liệu đơn giản (`int`, `String`), thực tế nên có 1 class `State` đại diện đủ trạng thái: đang tải / thành công / lỗi.

```dart
abstract class ProductState {}
class ProductLoading extends ProductState {}
class ProductLoaded extends ProductState {
  final List<String> danhSach;
  ProductLoaded(this.danhSach);
}
class ProductError extends ProductState {
  final String message;
  ProductError(this.message);
}
```

UI sẽ dùng `if (state is ProductLoading) ... else if (state is ProductLoaded) ...` để hiển thị đúng giao diện cho từng trạng thái — đây chính là mô hình sẽ dùng khi kết nối API thật ở Ngày 4.

## 💻 Code Example (repo `Code`)

> ⚠️ Repo [`SangLeSoftZ/Code`](https://github.com/SangLeSoftZ/Code) **hiện chỉ chứa code Java/Spring Boot**
> (package `com.example.demo`), chưa có project Flutter/Dart nào. Khi có repo code Flutter
> riêng, bổ sung link + snippet vào đúng mục này theo cùng quy ước (xem `QUY_UOC_DAT_TEN.md`).


# Ngày 8 / Tuần 2 - Ngày 1 (Sáng) — Flutter: BlocObserver

# BUỔI SÁNG — BlocObserver: theo dõi mọi thay đổi state toàn app

## 1. Vấn đề thực tế BlocObserver giải quyết

Khi app có hàng chục Cubit/Bloc khác nhau, rất khó biết **chính xác** state nào đang đổi ở đâu, khi nào — đặc biệt lúc debug 1 lỗi khó tái hiện. `BlocObserver` là 1 "trạm quan sát toàn cục" — nhận được thông báo **mọi khi bất kỳ Cubit/Bloc nào trong app** thay đổi state, không cần tự thêm log vào từng Cubit riêng lẻ.

## 2. Vòng đời (lifecycle) mà BlocObserver theo dõi được

```dart
class AppBlocObserver extends BlocObserver {
  @override
  void onCreate(BlocBase bloc) {
    super.onCreate(bloc);
    print('Tao moi: ${bloc.runtimeType}');
  }

  @override
  void onChange(BlocBase bloc, Change change) {
    super.onChange(bloc, change);
    print('${bloc.runtimeType} doi state: ${change.currentState} -> ${change.nextState}');
  }

  @override
  void onTransition(Bloc bloc, Transition transition) {
    super.onTransition(bloc, transition);
    print('${bloc.runtimeType} nhan Event: ${transition.event}');
  }

  @override
  void onError(BlocBase bloc, Object error, StackTrace stackTrace) {
    print('Loi trong ${bloc.runtimeType}: $error');
    super.onError(bloc, error, stackTrace);
  }

  @override
  void onClose(BlocBase bloc) {
    super.onClose(bloc);
    print('Dong: ${bloc.runtimeType}');
  }
}
```

**Phân biệt `onChange` và `onTransition`:**
- `onChange` hoạt động cho **cả Cubit lẫn Bloc** — chỉ báo "state cũ" → "state mới"
- `onTransition` **chỉ hoạt động với Bloc** (không phải Cubit) — báo thêm cả **Event nào** gây ra sự thay đổi đó, vì Cubit không có Event (xem `D02_bloc-vs-cubit-state-design.md`).

## 3. Đăng ký BlocObserver — chỉ cần làm 1 lần trong `main()`

```dart
void main() {
  Bloc.observer = AppBlocObserver();
  runApp(MyApp());
}
```

## 4. Ứng dụng thực tế ngoài debug/log

- **Analytics:** trong `onChange`, gửi sự kiện lên Firebase Analytics/Mixpanel mỗi khi có 1 loại state quan trọng (VD: `LoginSuccess`)
- **Bắt lỗi tập trung:** trong `onError`, gửi lỗi lên Sentry/Crashlytics cho **toàn bộ** Cubit/Bloc, không cần try-catch riêng từng nơi

> Lưu ý: BlocObserver dùng để **quan sát**, không nên dùng để **thay đổi luồng xử lý** (không gọi ngược vào Cubit từ trong Observer) — tránh phá vỡ nguyên tắc tách biệt trách nhiệm.

## Bài tập

**Bài 1:** Viết `AppBlocObserver` như ví dụ, đăng ký vào `main()`. Chạy app, thao tác vài chức năng, quan sát console — xác nhận thấy đủ log `onCreate`/`onChange`/`onClose` cho từng Cubit.

**Bài 2:** Cố tình gây lỗi trong 1 Cubit (VD chia cho 0) — quan sát `onError` có bắt được không.

**Bài 3 (nâng cao):** Sửa `onChange` để chỉ log Cubit có tên chứa `"Task"` (`bloc.runtimeType.toString().contains('Task')`).

---

##  Code Example (repo `flutter`)

>  Repo code Flutter riêng (`SangLeSoftZ/flutter`) — khi đã đẩy code thật lên, bổ sung link cụ thể vào đúng mục này theo quy ước (xem `QUY_UOC_DAT_TEN.md`, mục 2.1). Vị trí dự kiến: `lib\tuan2_ngay1\app_bloc_observer.dart`, khởi tạo tại `lib/main.dart`.

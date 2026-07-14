# Ngày 8 / Tuần 2 - Ngày 1 (Chiều) — Flutter: HydratedBloc

# BUỔI CHIỀU — HydratedBloc: tự động lưu/khôi phục state

## 1. Vấn đề thực tế: state "biến mất" khi tắt app

Nếu người dùng chọn Dark Mode, tắt app, mở lại → state của `ThemeCubit` **reset về ban đầu** (Cubit chỉ sống trong RAM). Cách thông thường là tự dùng `SharedPreferences` lưu/đọc thủ công (xem `D06_local-storage-hive-sharedpreferences.md` nếu đã có) — `HydratedBloc` **tự động hóa** việc này.

## 2. Setup

```yaml
dependencies:
  hydrated_bloc: ^9.1.5
  path_provider: ^2.1.0
```

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  final storage = await HydratedStorage.build(
    storageDirectory: HydratedStorageDirectory(
      (await getApplicationDocumentsDirectory()).path,
    ),
  );
  HydratedBloc.storage = storage;

  runApp(MyApp());
}
```

## 3. Chuyển `ThemeCubit` thường sang `HydratedCubit`

```dart
// TRUOC - Cubit thuong, mat state khi tat app
class ThemeCubit extends Cubit<bool> {
  ThemeCubit() : super(false);
  void toggle() => emit(!state);
}

// SAU - HydratedCubit, tu dong luu/khoi phuc
class ThemeCubit extends HydratedCubit<bool> {
  ThemeCubit() : super(false);
  void toggle() => emit(!state);

  @override
  bool fromJson(Map<String, dynamic> json) => json['darkMode'] as bool;

  @override
  Map<String, dynamic> toJson(bool state) => {'darkMode': state};
}
```

Chỉ cần đổi 2 điểm: kế thừa `HydratedCubit` thay vì `Cubit`, và thêm `fromJson`/`toJson` — không đổi gì khác trong logic `toggle()` hay cách dùng ở UI.

## 4. Cơ chế bên dưới

Mỗi khi `emit(...)` được gọi, `HydratedCubit` tự động gọi `toJson()`, lưu xuống ổ đĩa (dùng Hive bên trong làm storage engine). Lúc khởi động lại, tự đọc và gọi `fromJson()` để khôi phục state **trước khi** Widget đầu tiên được build.

> Không dùng HydratedBloc cho dữ liệu nhạy cảm (token) — dùng `flutter_secure_storage` riêng.

## Bài tập

**Bài 4:** Chuyển `ThemeCubit` (hoặc 1 Cubit đơn giản đang có) sang `HydratedCubit`. Test: đổi theme → tắt hẳn app → mở lại → xác nhận theme vẫn giữ đúng.

**Bài 5:** Thử chuyển tiếp 1 Cubit có state phức tạp hơn (union type `Loading/Loaded/Error`) sang `HydratedCubit`. Gợi ý: chỉ nên lưu state `Loaded`, khi khôi phục mà không có dữ liệu thì trả về `Loading`/`Initial`.

---

##  Code Example (repo `flutter`)

> Repo code Flutter riêng (`SangLeSoftZ/flutter`) — bổ sung link cụ thể khi đã đẩy code thật. Vị trí dự kiến theo quy ước: `lib\tuan2_ngay1\theme_cubit.dart`.

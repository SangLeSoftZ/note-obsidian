---
title: "Ngày 6 — Test bài 3: MultiBlocProvider (State Management)"
track: flutter
day: 6
topic: state-management-multiblocprovider
tags: [flutter, multiblocprovider, state-management]
source_goc: TEST_BAI3_MULTIBLOCPROVIDER.md
---

# Ngày 6 — Test bài 3: MultiBlocProvider (State Management)

# TEST BÀI 3 — MultiBlocProvider (State Management)

## Mục đích test
Hiểu MultiBlocProvider **KHÔNG lưu xuống ổ đĩa**, state chỉ sống trong RAM.

---

## Chuẩn bị
Mở `lib/main.dart`, đổi home:
```dart
home: const MultiBlocDemoScreen(),
```

Chạy app:
```bash
flutter run
```

---

## Kịch bản 1: Test 3 Cubit cùng lúc

**Màn hình có 3 section:**

### A. CounterCubit (số đếm)
**1. Nhấn ➕ nhiều lần**
- Số tăng: 0 → 1 → 2 → 3...
- State lưu trong **RAM**

**2. Nhấn ➖**
- Số giảm nhưng không xuống dưới 0

### B. ThemeCubit (dark/light)
**1. Nhấn nút 🌙 ở AppBar**
- Nền đổi sang đen (dark mode)
- ThemeCubit: `emit(true)`

**2. Nhấn lại → đổi về sáng**

### C. FavoriteCubit (yêu thích)
**1. Nhấn "Thêm thích"**
- Icon đổi sang ❤️ đỏ
- FavoriteCubit: `emit(true)`

**2. Nhấn "Bỏ thích" → icon về 🤍 trắng**

---

## Kịch bản 2: TẮT APP → State mất hết (QUAN TRỌNG)

**1. Thao tác:**
- Đếm lên 10
- Bật dark mode
- Thêm vào yêu thích

**2. TẮT APP hoàn toàn** (Stop trong VS Code)

**3. MỞ LẠI APP:**
```bash
flutter run
```

**4. Kỳ vọng:**
- Số đếm về 0 (CounterCubit reset)
- Nền sáng trở lại (ThemeCubit reset)
- Icon trắng (FavoriteCubit reset)

**✅ KẾT LUẬN:** State Management **KHÔNG lưu** xuống ổ đĩa. Tắt app = mất hết state.

---

## Kịch bản 3: So sánh với Hive/SharedPreferences

### Thử nghiệm kết hợp

**Nếu muốn số đếm còn lại sau khi tắt app:**

❌ **SAI:** Chỉ dùng Cubit
```dart
class CounterCubit extends Cubit<int> {
  void increment() => emit(state + 1);
  // ← chỉ emit, không ghi xuống ổ đĩa
}
```

✅ **ĐÚNG:** Cubit + SharedPreferences
```dart
class CounterCubit extends Cubit<int> {
  Future<void> increment() async {
    final newValue = state + 1;
    emit(newValue);
    // Ghi xuống ổ đĩa
    final prefs = await SharedPreferences.getInstance();
    await prefs.setInt('counter', newValue);
  }

  Future<void> loadInitial() async {
    final prefs = await SharedPreferences.getInstance();
    final saved = prefs.getInt('counter') ?? 0;
    emit(saved); // đọc lại khi khởi động
  }
}
```

---

## Kịch bản 4: Test MultiBlocProvider (mục đích chính của bài 3)

**🔍 Xem code để hiểu:**

File: `lib/bai5_storage/bai3_multi_bloc/multi_bloc_demo_screen.dart`

**TRƯỚC (lồng nhau — khó đọc):**
```dart
BlocProvider<CounterCubit>(
  create: (_) => CounterCubit(),
  child: BlocProvider<ThemeCubit>(
    create: (_) => ThemeCubit(),
    child: BlocProvider<FavoriteCubit>(
      create: (_) => FavoriteCubit(),
      child: MyScreen(), // ← 3 lớp lồng nhau
    ),
  ),
)
```

**SAU (MultiBlocProvider — dễ đọc):**
```dart
MultiBlocProvider(
  providers: [
    BlocProvider<CounterCubit>(create: (_) => CounterCubit()),
    BlocProvider<ThemeCubit>(create: (_) => ThemeCubit()),
    BlocProvider<FavoriteCubit>(create: (_) => FavoriteCubit()),
  ],
  child: MyScreen(), // ← phẳng, dễ thêm/bớt provider
)
```

**Test thực tế:**
1. Thử thêm CounterCubit thứ 2 vào `providers`
2. Dùng cả 2 counter trong UI
3. Thấy cả 2 đều hoạt động độc lập

---

## So sánh tổng hợp 3 bài

### Bài 1: SharedPreferences
```dart
final prefs = await SharedPreferences.getInstance();
await prefs.setBool('key', true);
// ✅ Tắt app → mở lại → còn
```

### Bài 2: Hive
```dart
final box = Hive.box<TaskHiveModel>('box');
await box.add(TaskHiveModel(...));
// ✅ Tắt app → mở lại → còn
// ✅ ValueListenableBuilder → auto rebuild
```

### Bài 3: MultiBlocProvider (State Management)
```dart
MultiBlocProvider(
  providers: [
    BlocProvider<CounterCubit>(...),
    BlocProvider<ThemeCubit>(...),
  ],
  child: Screen(),
)
// ❌ Tắt app → mở lại → MẤT HẾT state
// ✅ Chỉ dùng để quản lý state trong phiên làm việc hiện tại
```

---

## Bảng tổng hợp cuối cùng

| | State Mgmt (Bloc/Cubit) | SharedPreferences | Hive |
|---|---|---|---|
| **Lưu ở đâu** | RAM | Ổ đĩa (XML/JSON) | Ổ đĩa (binary) |
| **Tắt app** | ❌ Mất | ✅ Còn | ✅ Còn |
| **Dùng cho** | UI state tạm | Cài đặt đơn giản | Dữ liệu có cấu trúc |
| **Lưu object** | ✅ (trong RAM) | ❌ | ✅ (cần TypeAdapter) |
| **Auto rebuild** | ✅ (BlocBuilder) | ❌ | ✅ (ValueListenableBuilder) |
| **Ví dụ** | Loading, filter | Onboarding, theme | Task list, cart |

---

## Kết luận thực tế

**Khi coding app thật, bạn sẽ dùng CẢ BA:**

```dart
// State Management — UI state trong phiên hiện tại
BlocProvider<LoadingCubit>(...)

// SharedPreferences — cài đặt nhỏ
final isDark = prefs.getBool('dark_mode') ?? false;

// Hive — dữ liệu phức tạp offline
final cart = Hive.box<CartItem>('cart').values.toList();
```

Chúng **BỔ SUNG** cho nhau, không thay thế.

## 💻 Code Example (repo `Code`)

> ⚠️ Repo [`SangLeSoftZ/Code`](https://github.com/SangLeSoftZ/Code) **hiện chỉ chứa code Java/Spring Boot**
> (package `com.example.demo`), chưa có project Flutter/Dart nào. Khi có repo code Flutter
> riêng, bổ sung link + snippet vào đúng mục này theo cùng quy ước (xem `QUY_UOC_DAT_TEN.md`).

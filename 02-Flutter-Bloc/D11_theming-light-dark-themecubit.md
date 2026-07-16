---
title: "Ngày 11 (Tuần 2 - Ngày 4, Chiều) — Theming Light/Dark kết nối ThemeCubit"
track: flutter
day: 11
topic: theming-light-dark-themecubit
tags: [flutter, theming, colorscheme, material3, themecubit]
source_goc: Tuan2_Ngay4_TaiLieu.md
---

# Ngày 11 (Tuần 2 - Ngày 4, Chiều) — Theming Light/Dark kết nối ThemeCubit

## 1. `lightTheme`/`darkTheme` — dùng `ColorScheme.fromSeed` theo Material 3

Code thật — `lib/tuan2_ngay4_theme/app_themes.dart`:

```dart
final lightTheme = ThemeData(
  brightness: Brightness.light,
  colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue),
  useMaterial3: true,
  textTheme: const TextTheme(
    titleLarge: TextStyle(fontWeight: FontWeight.bold, fontSize: 20),
    bodyMedium: TextStyle(fontSize: 14),
  ),
  appBarTheme: const AppBarTheme(
    backgroundColor: Colors.white,
    foregroundColor: Colors.black,
    elevation: 1,
  ),
  cardTheme: CardThemeData(
    elevation: 2,
    shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
  ),
);

final darkTheme = ThemeData(
  brightness: Brightness.dark,
  colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue, brightness: Brightness.dark),
  useMaterial3: true,
  textTheme: const TextTheme(
    titleLarge: TextStyle(fontWeight: FontWeight.bold, fontSize: 20),
    bodyMedium: TextStyle(fontSize: 14),
  ),
  appBarTheme: const AppBarTheme(elevation: 1),
  cardTheme: CardThemeData(
    elevation: 2,
    shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
  ),
);
```

**`ColorScheme.fromSeed` là gì?** Thay vì tự định nghĩa từng màu (primary, secondary,
surface, error...), chỉ cần đưa 1 màu "hạt giống" (`seedColor: Colors.blue`), Flutter tự
sinh ra 1 bộ màu hài hòa theo chuẩn Material 3 — tiết kiệm công sức so với tự phối màu
tay, và đảm bảo tương phản đủ tốt cho khả năng đọc (accessibility). `darkTheme` dùng
**cùng seed color** nhưng thêm `brightness: Brightness.dark` — Flutter tự tạo bộ màu tối
tương ứng, không cần chọn lại từng màu cho bản dark.

Lưu ý thêm so với ví dụ lý thuyết gốc: code thật còn khai báo `cardTheme` chung cho cả 2
theme (bo góc 12px, elevation 2) — để mọi `Card` trong app (bao gồm `TaskCard` ở buổi
sáng) tự động đồng bộ style mà không cần set riêng lẻ từng nơi.

## 2. Kết nối `ThemeCubit` (đã có từ Ngày 1) với `MaterialApp` — đúng code trong `main.dart`

```dart
return BlocProvider(
  create: (_) => ThemeCubit(),
  child: BlocBuilder<ThemeCubit, bool>(
    builder: (context, isDark) {
      return MaterialApp(
        debugShowCheckedModeBanner: false,
        // Kết nối lightTheme/darkTheme với ThemeCubit
        theme: lightTheme,
        darkTheme: darkTheme,
        themeMode: isDark ? ThemeMode.dark : ThemeMode.light,
        home: const ThemedTaskScreen(),
      );
    },
  ),
);
```

**Vì sao `BlocBuilder` bọc ngay `MaterialApp`, không bọc từng widget con riêng lẻ?** Vì
`theme`/`darkTheme`/`themeMode` là thuộc tính của chính `MaterialApp` — thay đổi ở đây tự
động áp dụng xuống **toàn bộ cây widget con**, thông qua `Theme.of(context)` mà mọi
widget Material tự động đọc. Không cần truyền `isDark` thủ công xuống từng màn hình.

`ThemeCubit` ở đây chính là `HydratedCubit` đã làm ở Ngày 1 — nên lựa chọn Light/Dark
**tự động được giữ lại** sau khi tắt/mở lại app, không cần thêm code gì nữa (thừa hưởng
toàn bộ cơ chế Hive đã học).

## 3. `ThemedTaskScreen` — dùng `Theme.of(context)` thay vì hardcode, có nút đổi theme thật

Code thật — `lib/tuan2_ngay4_theme/themed_task_screen.dart`:

```dart
final theme = Theme.of(context);
final colors = theme.colorScheme;

Scaffold(
  appBar: AppBar(
    title: Text('Tuần 2 Ngày 4 — Theming', style: theme.textTheme.titleLarge),
    actions: [
      IconButton(
        icon: Icon(context.watch<ThemeCubit>().state ? Icons.light_mode : Icons.dark_mode),
        onPressed: () => context.read<ThemeCubit>().toggle(),
        tooltip: 'Đổi Light/Dark',
      ),
      ...
    ],
  ),
  backgroundColor: colors.surface,
  ...
)
```

Đây là chi tiết quan trọng không có trong ví dụ lý thuyết gốc: `context.watch<ThemeCubit>()`
dùng để **rebuild lại icon** (mặt trời/mặt trăng) mỗi khi theme đổi, còn
`context.read<ThemeCubit>().toggle()` dùng trong `onPressed` để **gọi hành động** mà
không cần rebuild ngay tại đó. Đây đúng phân biệt `watch` (đọc + lắng nghe thay đổi) vs
`read` (chỉ đọc 1 lần, dùng trong callback) mà bạn đã học ở phần Bloc/Cubit cơ bản.

## 4. Từng chỗ hardcode được thay bằng Theme — đối chiếu cụ thể trong `_ThemedTaskCard`

```dart
switch (task.trangThai) {
  case 'HOAN_THANH':
    statusColor = colors.tertiary;   // KHÔNG dùng Colors.green cố định
    ...
  case 'DANG_LAM':
    statusColor = colors.secondary;  // KHÔNG dùng Colors.orange cố định
    ...
  default:
    statusColor = colors.outline;
}
```

So sánh với `TaskCard` ở buổi sáng (hardcode `Colors.green`/`Colors.orange`/`Colors.grey`)
— `_ThemedTaskCard` lấy màu từ `colorScheme.tertiary`/`secondary`/`outline`. Đây là ví dụ
thực tế rất rõ cho nguyên tắc trong tài liệu gốc:

```dart
// KHÔNG NÊN — hardcode màu, không tự đổi theo Light/Dark
Text('Tiêu đề', style: TextStyle(color: Colors.black))

// NÊN — lấy từ Theme hiện tại
Text('Tiêu đề', style: theme.textTheme.titleLarge)
```

Nếu hardcode màu, khi chuyển Dark Mode, đoạn UI đó **không đổi theo** (ví dụ chữ đen trên
nền đen, không đọc được) — lỗi UI rất hay gặp khi mới làm Theming. Trong code thật,
`_ThemedTaskCard` còn có hẳn 1 khối UI "Hardcode vs Theme" minh họa trực quan sự khác biệt
này ngay trong app, để người dùng vừa học vừa nhìn thấy kết quả.

## Bài tập

**Bài 3 (đã có sẵn trong repo):** `lightTheme`/`darkTheme` đã định nghĩa và kết nối với
`ThemeCubit` trong `main.dart`. Tự kiểm chứng: bấm icon ☀️/🌙 trên `ThemedTaskScreen`,
xác nhận toàn bộ app đổi giao diện; tắt/mở lại app, xác nhận lựa chọn Dark/Light được giữ
nguyên (nhờ `HydratedCubit` từ Ngày 1).

**Bài 4 (đã có sẵn trong repo — `themed_task_screen.dart`):** Đối chiếu `_ThemedTaskCard`
với `TaskCard` (buổi sáng) — liệt kê ra tất cả những chỗ `_ThemedTaskCard` đã thay hardcode
bằng `Theme.of(context)`. Thử đổi ngược 1 chỗ (vd `colors.tertiary` → `Colors.green`) rồi
bật Dark Mode để tự quan sát lỗi tương phản.

##  Code Example (repo `flutter.git`)

> Xem nguyên các file tại branch `tuan2`:
> [`app_themes.dart`](https://github.com/SangLeSoftZ/flutter/blob/tuan2/testflutter/lib/tuan2_ngay4_theme/app_themes.dart) ·
> [`themed_task_screen.dart`](https://github.com/SangLeSoftZ/flutter/blob/tuan2/testflutter/lib/tuan2_ngay4_theme/themed_task_screen.dart) ·
> [`main.dart`](https://github.com/SangLeSoftZ/flutter/blob/tuan2/testflutter/lib/main.dart) (phần `MyApp.build`)

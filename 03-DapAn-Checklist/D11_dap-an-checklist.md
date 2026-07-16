---
title: "Ngày 11 / Tuần 2 - Ngày 4 — Đáp án gợi ý & Checklist hoàn thành"
track: chung
day: 11
topic: dap-an-checklist
tags: [dap-an, checklist]
source_goc: Tuan2_Ngay4_TaiLieu.md
---

# Ngày 11 / Tuần 2 - Ngày 4 — Đáp án gợi ý & Checklist hoàn thành

## ĐÁP ÁN GỢI Ý

<details><summary>Bài 1 — Tách TaskCard + TaskSearchBar (xem D11_custom-widget-dumb-widget.md)</summary>

Đã có sẵn trong repo tại `lib/tuan2_ngay4/task_card.dart` và
`lib/tuan2_ngay4/task_search_bar.dart`. Điểm cần đối chiếu để xác nhận đúng nguyên tắc
"dumb widget":

```dart
// TaskCard chỉ nhận dữ liệu qua constructor — KHÔNG có dòng nào gọi context.read<TaskCubit>()
class TaskCard extends StatelessWidget {
  final Task task;
  final VoidCallback? onXoa;
  final VoidCallback? onTap;
  final bool showDeleteButton;
  const TaskCard({super.key, required this.task, this.onXoa, this.onTap, this.showDeleteButton = true});
}
```

`TaskSearchBar` cũng vậy — chỉ gọi `widget.onSearch(text)` để báo ra ngoài, không tự lọc
danh sách bên trong widget.
</details>

<details><summary>Bài 2 — Widget Test cho TaskCard (xem D11_custom-widget-dumb-widget.md)</summary>

Đã có sẵn trong repo tại `test/tuan2_ngay4/task_card_test.dart` với 8 test case, bao gồm:

```dart
testWidgets('gọi onXoa khi nhấn nút xóa', (tester) async {
  bool daGoi = false;
  await tester.pumpWidget(buildTaskCard(task: taskMau, onXoa: () => daGoi = true));
  await tester.tap(find.byIcon(Icons.delete_outline));
  await tester.pump();
  expect(daGoi, isTrue);
});
```

Test case mở rộng gợi ý (trạng thái lạ rơi vào nhánh `default`):

```dart
testWidgets('hiển thị label mặc định khi trangThai không xác định', (tester) async {
  const taskLa = Task(id: 3, tieuDe: 'Task lạ', moTa: '', trangThai: 'KHONG_XAC_DINH');
  await tester.pumpWidget(buildTaskCard(task: taskLa));
  expect(find.text('Chưa làm'), findsOneWidget);
});
```
</details>

<details><summary>Bài 3 — lightTheme/darkTheme kết nối ThemeCubit (xem D11_theming-light-dark-themecubit.md)</summary>

Đã có sẵn trong repo tại `lib/tuan2_ngay4_theme/app_themes.dart` và đã kết nối trong
`main.dart`:

```dart
return BlocProvider(
  create: (_) => ThemeCubit(),
  child: BlocBuilder<ThemeCubit, bool>(
    builder: (context, isDark) => MaterialApp(
      theme: lightTheme,
      darkTheme: darkTheme,
      themeMode: isDark ? ThemeMode.dark : ThemeMode.light,
      home: const ThemedTaskScreen(),
    ),
  ),
);
```

Kiểm chứng thủ công: bấm icon ☀️/🌙 trên `ThemedTaskScreen` → toàn app đổi giao diện; tắt
mở lại app → do `ThemeCubit` đã là `HydratedCubit` từ Ngày 1, lựa chọn Dark/Light vẫn giữ
nguyên.
</details>

<details><summary>Bài 4 — Thay hardcode bằng Theme.of(context) (xem D11_theming-light-dark-themecubit.md)</summary>

Đã có sẵn trong repo tại `_ThemedTaskCard` (trong `themed_task_screen.dart`) — toàn bộ
màu status lấy từ `colorScheme` thay vì `Colors.xxx` cố định:

```dart
switch (task.trangThai) {
  case 'HOAN_THANH': statusColor = colors.tertiary; break;
  case 'DANG_LAM':   statusColor = colors.secondary; break;
  default:           statusColor = colors.outline;
}
```

Kiểm chứng lỗi tương phản: đổi tạm `colors.tertiary` thành `Colors.green` rồi bật Dark
Mode — quan sát xem có khó đọc hơn không so với dùng màu từ theme.
</details>

<details><summary>Bài 5 — @Cacheable cho getAll()/getById() (xem D11_cacheable-cacheevict-task.md)</summary>

Đã có sẵn trong repo tại `TaskServiceImpl.java`:

```java
@Override
@Cacheable("tasks")
public List<Task> getAll() {
    System.out.println("=== [DB] getAll — đang truy vấn database...");
    return repository.findAll();
}
```

Kiểm chứng: gọi `GET /api/tasks` 2 lần liên tiếp qua Postman — dòng log
`"=== [DB] getAll..."` chỉ in ra đúng 1 lần ở lần gọi đầu.

**Lưu ý bắt buộc:** phải có `@Bean CacheManager` khai báo thủ công trong
`SecurityConfig.java` (`ConcurrentMapCacheManager("tasks", "taskById")`) — thiếu bean này
app sẽ lỗi khi khởi động, vì phiên bản Spring Boot đang dùng không tự động tạo
`CacheManager` như tài liệu cũ mô tả.
</details>

<details><summary>Bài 6 — @CacheEvict khi tạo/sửa/xóa Task (xem D11_cacheable-cacheevict-task.md)</summary>

Đã có sẵn trong repo tại `TaskServiceImpl.java`:

```java
@Override
@CacheEvict(value = "tasks", allEntries = true)
public Task create(TaskRequest dto) {
    System.out.println("=== [DB] create — xóa cache 'tasks'");
    ...
}
```

Kiểm chứng đầy đủ luồng: `GET /api/tasks` (cache lại) → `POST /api/tasks` tạo task mới →
`GET /api/tasks` lại → xác nhận task mới xuất hiện ngay, không cần đợi cache hết hạn.
</details>

---

##  CHECKLIST HOÀN THÀNH

- [ ] Tách được `TaskCard` + `TaskSearchBar`, cả 2 chỉ nhận dữ liệu/callback qua constructor
- [ ] Hiểu vì sao "dumb widget" dễ test — không cần setup Cubit/Provider/GetIt trong test
- [ ] Chạy được `task_card_test.dart`, hiểu từng test case kiểm tra gì
- [ ] `lightTheme`/`darkTheme` dùng `ColorScheme.fromSeed`, kết nối đúng với `ThemeCubit` qua `BlocBuilder` bọc `MaterialApp`
- [ ] Xác nhận Dark/Light giữ nguyên sau khi tắt/mở app (nhờ `HydratedCubit` Ngày 1)
- [ ] Không còn `Colors.xxx`/`TextStyle` hardcode gây lỗi tương phản khi đổi Dark Mode
- [ ] Biết chính xác lý do phải tự khai báo `@Bean CacheManager` (không tự động như tài liệu cũ)
- [ ] `@Cacheable` hoạt động đúng — log xác nhận lần gọi sau không truy vấn database
- [ ] `@CacheEvict` xóa đúng cache khi tạo/sửa/xóa Task — không trả về dữ liệu cũ
- [ ] Giải thích được vì sao `timKiem()` (Specification) không nên `@Cacheable`

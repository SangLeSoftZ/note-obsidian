---
title: "Ngày 6 — Test bài 2: Hive Local Storage"
track: flutter
day: 6
topic: local-storage-hive
tags: [flutter, hive, local-storage, list-object]
source_goc: TEST_BAI2_HIVE.md
---

# Ngày 6 — Test bài 2: Hive Local Storage

# TEST BÀI 2 — Hive Local Storage

## Mục đích test
Hiểu Hive lưu **List<Object>**, có **auto-rebuild UI**, dữ liệu **còn sau khi tắt app**.

---

## Chuẩn bị
Mở `lib/main.dart`, đổi home:
```dart
home: const HiveTaskScreen(),
```

Chạy app:
```bash
flutter run
```

---

## Kịch bản 1: THÊM task

**1. Nhập vào ô text:** "Học Flutter"
**2. Nhấn nút "Thêm"**

**Kỳ vọng:**
- Task xuất hiện ngay trong danh sách (không cần reload)
- Có icon ⏳, trạng thái "Chưa xong"

**🔍 Kiểm tra code:**
File: `lib/bai5_storage/bai2_hive/hive_task_screen.dart`
```dart
Future<void> _themTask() async {
  // ...
  await _box.add(TaskHiveModel(tieuDe: tieuDe)); // ← Hive lưu object
  // KHÔNG cần setState — ValueListenableBuilder tự rebuild
}
```

**3. Thêm thêm vài task nữa:**
- "Học Bloc"
- "Học Clean Architecture"

---

## Kịch bản 2: ĐỔI trạng thái (UPDATE)

**1. Nhấn vào card "Học Flutter"**

**Kỳ vọng:**
- Trạng thái đổi: Chưa xong → Đang làm → Hoàn thành → Chưa xong (vòng lặp)
- Icon và màu thay đổi theo

**🔍 Kiểm tra code:**
```dart
Future<void> _doiTrangThai(int index) async {
  final task = _box.getAt(index)!;
  task.trangThai = 'HOAN_THANH'; // ← sửa field
  await task.save();             // ← HiveObject.save() tự ghi vào Box
}
```

**2. Đổi trạng thái 2-3 task để thấy rõ sự khác biệt**

---

## Kịch bản 3: XÓA task

**1. Nhấn nút 🗑️ ở task "Học Bloc"**

**Kỳ vọng:**
- Task biến mất ngay
- SnackBar hiện "Đã xóa task"

**🔍 Kiểm tra code:**
```dart
Future<void> _xoaTask(int index) async {
  await _box.deleteAt(index); // ← Xóa khỏi Box
  // KHÔNG cần setState — ValueListenableBuilder tự rebuild
}
```

---

## Kịch bản 4: Tắt app → mở lại (QUAN TRỌNG NHẤT)

**1. TẮT APP hoàn toàn** (Stop trong VS Code)

**2. MỞ LẠI APP:**
```bash
flutter run
```

**3. Kỳ vọng:**
- Tất cả task vẫn còn nguyên (kể cả trạng thái đã đổi)
- Thứ tự vẫn giữ nguyên

**✅ KẾT LUẬN:** Hive đã lưu List<TaskHiveModel> xuống file nhị phân, đọc lại được sau khi tắt app.

---

## Kịch bản 5: So sánh với State Management

**Thử nghiệm phản chứng:**

Giả sử dùng Cubit thay Hive:
```dart
class TaskCubit extends Cubit<List<TaskModel>> {
  TaskCubit() : super([]);
  void addTask(TaskModel task) => emit([...state, task]);
}
```

→ Thêm 5 task → danh sách hiện đủ 5 task
→ **Tắt app** → mở lại → **danh sách trống** (state mất)

**Hive khác ở chỗ:**
→ Thêm 5 task → `box.add()` ghi xuống file
→ **Tắt app** → mở lại → `box.values.toList()` đọc lại được 5 task

---

## Điểm đặc biệt: ValueListenableBuilder

**Test này:**

1. Mở app, thêm 1 task
2. **KHÔNG TẮT APP**, để nguyên
3. Mở DevTools → tìm `_box` trong widget tree
4. Thêm 1 task nữa
5. Quan sát UI tự rebuild **không cần setState**

**🔍 Kiểm tra code:**
```dart
ValueListenableBuilder<Box<TaskHiveModel>>(
  valueListenable: _box.listenable(), // ← tự lắng nghe Box thay đổi
  builder: (context, box, _) {
    final tasks = box.values.toList();
    return ListView.builder(...); // rebuild khi add/delete/update
  },
)
```

SharedPreferences **KHÔNG có** tính năng này — bạn phải `setState` thủ công.

---

## So sánh Hive vs SharedPreferences

| Tính năng | SharedPreferences | Hive |
|---|---|---|
| Lưu bool, int, String | ✅ | ✅ |
| Lưu List<Object> | ❌ | ✅ |
| Auto rebuild UI | ❌ | ✅ (ValueListenableBuilder) |
| CRUD từng item | ❌ | ✅ (add/getAt/deleteAt/save) |
| Cần TypeAdapter | ❌ | ✅ |

---

## Tóm tắt

**SharedPreferences** = lưu vài giá trị đơn lẻ (onboarding, theme, token)
**Hive** = lưu danh sách có cấu trúc (task, giỏ hàng, lịch sử)

Cả hai đều **tồn tại sau khi tắt app** — khác hoàn toàn với Bloc/Cubit.

## 💻 Code Example (repo `Code`)

> ⚠️ Repo [`SangLeSoftZ/Code`](https://github.com/SangLeSoftZ/Code) **hiện chỉ chứa code Java/Spring Boot**
> (package `com.example.demo`), chưa có project Flutter/Dart nào. Khi có repo code Flutter
> riêng, bổ sung link + snippet vào đúng mục này theo cùng quy ước (xem `QUY_UOC_DAT_TEN.md`).

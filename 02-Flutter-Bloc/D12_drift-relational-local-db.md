---
title: "Ngày 12 / Tuần 2 - Ngày 5 (Sáng) — Flutter: Drift (Local DB quan hệ)"
track: flutter
day: 12
topic: drift-relational-local-db
tags: [flutter, drift, sqlite, local-database, dao]
source_goc: Tuan2_Ngay5_TaiLieu.md (buổi sáng)
---

# Ngày 12 / Tuần 2 - Ngày 5 (Sáng) — Flutter: Drift (Local DB quan hệ)

# BUỔI SÁNG — Setup Drift, thiết kế bảng local có quan hệ

## 1. Vì sao cần Drift, Hive (D06) chưa đủ sao?

Hive là NoSQL key-value — nhanh, đơn giản, nhưng **không hỗ trợ quan hệ thật** giữa các bảng. Drift (nền SQLite) là database **quan hệ** thật sự — hỗ trợ khóa ngoại, JOIN, transaction — đúng tư duy đã quen với MySQL/PostgreSQL, chạy ngay trên điện thoại.

**Chọn nhanh:** dữ liệu đơn giản, không cần join → Hive. Dữ liệu có quan hệ rõ ràng, truy vấn phức tạp → Drift.

## 2. Setup

```yaml
dependencies:
  drift: ^2.20.0
  sqlite3_flutter_libs: ^0.5.0
  path_provider: ^2.1.0
  path: ^1.9.0

dev_dependencies:
  drift_dev: ^2.20.0
  build_runner: ^2.4.0
```

## 3. Định nghĩa bảng — giống tư duy Entity ở Spring Boot

```dart
import 'package:drift/drift.dart';
part 'database.g.dart';

class Categories extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get ten => text()();
}

class Tasks extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get tieuDe => text()();
  BoolColumn get hoanThanh => boolean().withDefault(const Constant(false))();
  IntColumn get categoryId => integer().references(Categories, #id)(); // tuong duong @ManyToOne
}

@DriftDatabase(tables: [Categories, Tasks])
class AppDatabase extends _$AppDatabase {
  AppDatabase() : super(_openConnection());
  @override
  int get schemaVersion => 1;
}
```

`integer().references(Categories, #id)()` **chính là** khái niệm `@ManyToOne` + `@JoinColumn` bên Spring Boot (xem `D04_spring-data-jpa` nếu có) — cùng tư duy, khác cú pháp.

Sinh code:
```powershell
dart run build_runner build --delete-conflicting-outputs
```

## 4. DAO — tương đương Repository bên Spring Boot

```dart
extension TaskQueries on AppDatabase {
  Future<List<Task>> layTaskTheoCategory(int categoryId) {
    return (select(tasks)..where((t) => t.categoryId.equals(categoryId))).get();
  }
  Future<int> themTask(TasksCompanion task) => into(tasks).insert(task);
  Future<bool> capNhatTask(Task task) => update(tasks).replace(task);
  Future<int> xoaTask(int id) => (delete(tasks)..where((t) => t.id.equals(id))).go();
}
```

`select(tasks)..where(...)` là query builder — dịch sang SQL thật lúc chạy, **type-safe** (gõ sai tên cột → lỗi ngay lúc code, không đợi tới runtime).

## Bài tập

**Bài 1:** Định nghĩa 2 bảng `Categories`/`Tasks` với quan hệ khóa ngoại. Chạy `build_runner`, xác nhận không lỗi.

**Bài 2:** Viết DAO đủ CRUD cho `Tasks` + `layTaskTheoCategory`. Test: thêm 2 Category, vài Task thuộc từng Category, xác nhận lọc đúng.

---

## 💻 Code Example (repo `flutter`)

> ⚠️ Chưa có link cụ thể — đang chờ đường dẫn file thật trong [`SangLeSoftZ/flutter`](https://github.com/SangLeSoftZ/flutter.git). Vị trí dự kiến: `lib/core/database/app_database.dart`, `lib/core/database/task_dao.dart`.

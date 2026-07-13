---
title: "Ngày 4 — Code Generation: json_serializable & freezed"
track: flutter
day: 4
topic: json-codegen
tags: [flutter, json_serializable, freezed, code-generation]
source_goc: Ngay4_LyThuyetSau.md (Phần 3)
---

# Ngày 4 — Code Generation: json_serializable & freezed

# PHẦN 3 — CODE GENERATION: VÌ SAO `json_serializable` NHANH HƠN REFLECTION

### 2 cách để "tự động" parse JSON — và vì sao Dart chọn cách khác Java

Ở Java/Spring, việc parse JSON dùng thư viện Jackson dựa trên **Reflection** (đọc cấu trúc class **lúc chương trình đang chạy**). Nhưng Dart/Flutter với `json_serializable` lại chọn cách khác: **Code Generation** (sinh code **trước khi** chạy, ngay lúc build).

```
Reflection (cách Java/Jackson dùng):
  Lúc APP ĐANG CHẠY → đọc cấu trúc class → tự map field → chậm hơn 1 chút, nhưng linh hoạt

Code Generation (cách Dart/json_serializable dùng):
  Lúc BUILD (trước khi chạy) → sinh sẵn code Dart thật (file .g.dart) → lúc chạy chỉ gọi code có sẵn → nhanh hơn
```

**Vì sao Dart/Flutter ưu tiên Code Generation?** Vì Flutter biên dịch ra **native code** (chạy trực tiếp trên CPU điện thoại, không qua máy ảo như Java Reflection cần), và hiệu năng trên di động (đặc biệt máy yếu) quan trọng hơn nhiều so với server. Việc sinh code trước giúp lúc chạy thực tế **không tốn thời gian "suy nghĩ" cấu trúc class** — mọi thứ đã được viết sẵn thành code Dart thuần túy trong file `.g.dart`.

**Vì sao phải chạy `build_runner` lại mỗi khi sửa model?** Vì file `.g.dart` là code **đã sinh sẵn từ trước** — nó không tự cập nhật khi bạn thêm/sửa field trong class gốc. Đây là lý do rất hay gặp lỗi: sửa model xong, quên chạy lại `build_runner`, code vẫn chạy nhưng field mới **không hề được parse** (vì file `.g.dart` cũ chưa biết tới field đó).

```powershell
# Chạy 1 lần
dart run build_runner build --delete-conflicting-outputs

# Hoặc chạy chế độ "theo dõi" - tự sinh lại mỗi khi bạn lưu file model
dart run build_runner watch --delete-conflicting-outputs
```

Khi phát triển lâu dài (sửa model liên tục), nên dùng `watch` thay vì gõ `build` lại tay mỗi lần.

### `freezed` Union Type — pattern matching thay cho if/else dài dòng

```dart
@freezed
class TaskState with _$TaskState {
  const factory TaskState.loading() = TaskLoading;
  const factory TaskState.loaded(List<Task> danhSach) = TaskLoaded;
  const factory TaskState.error(String message) = TaskError;
}
```

Thay vì viết `if (state is TaskLoading) ... else if (state is TaskLoaded) ...` (dễ **quên xử lý 1 trường hợp** mà compiler không hề cảnh báo bạn), `freezed` cho phép dùng `when` — **bắt buộc** phải xử lý đủ tất cả các trường hợp, nếu thiếu 1 case, code **báo lỗi ngay lúc biên dịch**:

```dart
state.when(
  loading: () => CircularProgressIndicator(),
  loaded: (danhSach) => ListView(...),
  error: (message) => Text("Lỗi: $message"),
  // Nếu bạn thêm 1 loại state mới (ví dụ "empty") mà quên xử lý ở đây,
  // code sẽ KHÔNG BUILD ĐƯỢC cho tới khi bạn thêm case "empty" -> an toàn hơn if/else nhiều
);
```

**Đây chính là giá trị thực sự của union type:** nó biến lỗi "quên xử lý 1 trường hợp" từ **lỗi runtime** (app crash khi chạy, người dùng gặp phải) thành **lỗi compile-time** (bạn thấy ngay khi code, sửa trước khi build app) — an toàn hơn rất nhiều trong dự án thực tế.

### Bài tập đào sâu

**Bài D:** Thêm 1 field mới vào model `Task` (ví dụ `doUuTien: int`), **cố tình không chạy lại** `build_runner`, chạy app và gửi JSON có field mới này từ backend. Quan sát: field mới có xuất hiện trong object Dart không? Sau đó chạy lại `build_runner`, thử lại — so sánh kết quả.

**Bài E:** Chuyển `TaskState` (đã viết tay ở Ngày 3 bằng `abstract class` + class con) sang dùng `freezed` union type với `when`. So sánh độ dài code và cảm nhận mức độ "an toàn" khi thêm 1 state mới.

---

## 💻 Code Example (repo `Code`)

> ⚠️ Repo [`SangLeSoftZ/Code`](https://github.com/SangLeSoftZ/Code) **hiện chỉ chứa code Java/Spring Boot**
> (package `com.example.demo`), chưa có project Flutter/Dart nào. Khi có repo code Flutter
> riêng, bổ sung link + snippet vào đúng mục này theo cùng quy ước (xem `QUY_UOC_DAT_TEN.md`).

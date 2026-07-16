---
title: "Ngày 10 / Tuần 2 - Ngày 3 (Chiều) — Flutter: Hero Animation"
track: flutter
day: 10
topic: hero-animation-transition
tags: [flutter, animation, hero, navigation]
source_goc: Tuan2-Ngay3-Animation-Hero-SpecificationAPI.md (buổi chiều)
---

# Ngày 10 / Tuần 2 - Ngày 3 (Chiều) — Flutter: Hero Animation

# BUỔI CHIỀU — Hero Animation

## 1. Vấn đề thực tế Hero giải quyết

Khi chuyển từ màn hình Danh sách Task sang màn hình Chi tiết Task, nếu dùng `Navigator.push` bình thường, ảnh/icon đại diện sẽ "biến mất" ở màn cũ rồi "xuất hiện" đột ngột ở màn mới. `Hero` tạo hiệu ứng widget đó **"bay" liền mạch** từ vị trí/kích thước ở màn A sang đúng vị trí/kích thước ở màn B — cảm giác là **cùng 1 widget di chuyển**, không phải 2 widget riêng biệt.

## 2. Cách hoạt động: `tag` phải khớp

```dart
// Man hinh Danh sach - moi item boc trong Hero
class TaskListItem extends StatelessWidget {
  final Task task;
  const TaskListItem({super.key, required this.task});

  @override
  Widget build(BuildContext context) {
    return ListTile(
      leading: Hero(
        tag: 'task-icon-${task.id}', // PHAI duy nhat cho tung item
        child: CircleAvatar(
          backgroundColor: task.hoanThanh ? Colors.green : Colors.orange,
          child: Icon(task.hoanThanh ? Icons.check : Icons.circle_outlined),
        ),
      ),
      title: Text(task.tieuDe),
      onTap: () => context.push('/tasks/${task.id}', extra: task),
    );
  }
}

// Man hinh Chi tiet - Hero voi CUNG tag
class TaskDetailScreen extends StatelessWidget {
  final Task task;
  const TaskDetailScreen({super.key, required this.task});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Hero(
          tag: 'task-icon-${task.id}', // KHOP chinh xac voi tag ben tren
          child: CircleAvatar(
            radius: 60, // kich thuoc khac nhau van animate muot
            backgroundColor: task.hoanThanh ? Colors.green : Colors.orange,
            child: Icon(task.hoanThanh ? Icons.check : Icons.circle_outlined, size: 50),
          ),
        ),
      ),
    );
  }
}
```

**Điểm mấu chốt (hay gây lỗi nhất):** `tag` ở 2 màn hình phải **khớp tuyệt đối**. Nếu 1 danh sách hiển thị nhiều Task, `tag` phải **duy nhất theo từng item** (VD `'task-icon-${task.id}'`), tuyệt đối **không** dùng 1 tag cố định chung cho tất cả item — nếu không Flutter báo lỗi `Multiple heroes that share the same tag` ngay khi build màn danh sách.

## 3. Flutter tự động làm gì bên dưới

Khi `context.push` được gọi, Flutter tìm tất cả cặp `Hero` có `tag` trùng nhau giữa route cũ và route mới, tự tính toán đường bay (kích thước, vị trí, hình dạng) và animate trong lúc chuyển route — không cần tự viết logic tính toán vị trí.

**Lưu ý khi nào Hero KHÔNG hoạt động:**
- Thiếu `Hero` ở 1 trong 2 màn hình → không animate, chuyển màn bình thường
- `tag` không khớp (khác kiểu String vs int, sai chính tả) → im lặng bỏ qua animation, không báo lỗi rõ ràng
- Dùng `context.go()` thay vì `context.push()` (xem `D09_go-router-declarative-navigation.md`) → vì `go` thay thế toàn bộ stack, Hero animation có thể không mượt hoặc không xảy ra — ưu tiên `push` cho luồng có Hero

## 4. Kết hợp Hero với Animation buổi sáng

`Hero` lo phần "bay" giữa 2 màn hình, còn nội dung **bên trong** màn Chi tiết (VD: mô tả Task hiện dần sau khi bay xong) dùng `AnimatedOpacity`/`AnimatedContainer` riêng (xem `D10_animation-implicit-basics.md`) — 2 kỹ thuật không xung đột, bổ trợ nhau.

## Bài tập

**Bài 3:** Bọc icon/avatar đại diện của mỗi Task trong danh sách bằng `Hero` với `tag` duy nhất theo `id`. Bọc đúng widget tương ứng ở màn Chi tiết với cùng `tag`. Test: bấm vào 1 item, quan sát hiệu ứng bay mượt.

**Bài 4 (nâng cao):** Thử cố ý đặt `tag` sai khớp giữa 2 màn hình (hoặc dùng tag trùng cho nhiều item) để tự quan sát lỗi/hiện tượng xảy ra.

---

##  Code Example (repo `flutter`)

>  Repo code Flutter riêng (`SangLeSoftZ/flutter`) — bổ sung link khi đã đẩy code thật.

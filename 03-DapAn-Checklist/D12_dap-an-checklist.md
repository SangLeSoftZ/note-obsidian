---
title: "Ngày 12 / Tuần 3 - Ngày 1 — Đáp án gợi ý & Checklist hoàn thành"
track: chung
day: 12
topic: dap-an-checklist
tags: [dap-an, checklist]
source_goc: Tuan3_Ngay1_TaiLieu.md
---

# Ngày 12 / Tuần 3 - Ngày 1 — Đáp án gợi ý & Checklist hoàn thành

## ĐÁP ÁN GỢI Ý

<details><summary>Bài 1 — Tính tổng 500 triệu số: luồng chính vs compute() (xem D14_isolates-compute.md)</summary>

Đã có sẵn trong repo tại `lib/tuan3_ngay1/bai1_isolate_demo_screen.dart`:

```dart
int tinhTongNang(int soLuong) {
  int tong = 0;
  for (int i = 0; i < soLuong; i++) { tong += i; }
  return tong;
}

// Trực tiếp — UI đơ
void _tinhTrucTiep() {
  final ketQua = tinhTongNang(_soLuong);
}

// compute() — UI mượt
Future<void> _tinhVoiCompute() async {
  final ketQua = await compute(tinhTongNang, _soLuong);
}
```

Kiểm chứng: bấm nút Counter trong lúc `_tinhTrucTiep()` chạy → không phản hồi (UI đơ);
bấm Counter trong lúc `_tinhVoiCompute()` chạy → vẫn tăng bình thường (UI mượt).
</details>

<details><summary>Bài 2 — compute() cho parse JSON 50.000 Task (xem D14_isolates-compute.md)</summary>

Đã có sẵn trong repo tại `lib/tuan3_ngay1/bai2_isolate_parse_screen.dart`:

```dart
List<TaskItem> parseJsonList(String jsonString) {
  final List<dynamic> data = jsonDecode(jsonString) as List<dynamic>;
  return data.map((item) => TaskItem.fromJson(item as Map<String, dynamic>)).toList();
}

final result = await compute(parseJsonList, jsonString);
```

Kiểm chứng bằng `Stopwatch` đã có sẵn trong code — so sánh `elapsedMilliseconds` giữa parse
trực tiếp và `compute()`, đồng thời quan sát Counter để xác nhận UI không đơ khi dùng
`compute()`.
</details>

<details><summary>Bài 3 — CustomPainter vòng tròn tiến độ Task (xem D14_custompainter-canvas-paint.md)</summary>

Đã có sẵn trong repo tại `lib/tuan3_ngay1/bai3_custom_painter_screen.dart`:

```dart
class VongTronTienDoPainter extends CustomPainter {
  final double phanTram;
  @override
  void paint(Canvas canvas, Size size) {
    final center = Offset(size.width / 2, size.height / 2);
    final radius = (size.width / 2) - doDay / 2;
    canvas.drawCircle(center, radius, paintNen);
    canvas.drawArc(Rect.fromCircle(center: center, radius: radius),
        -math.pi / 2, 2 * math.pi * phanTram, false, paintTienDo);
  }
  @override
  bool shouldRepaint(VongTronTienDoPainter oldDelegate) => oldDelegate.phanTram != phanTram;
}
```

Áp dụng cho màn hình "% Task đã hoàn thành" — đã có sẵn với 3 category demo (Học Flutter,
Spring Boot, Clean Code), kèm animation khi chuyển đổi giữa các category.
</details>

<details><summary>Bài 4 — shouldRepaint luôn true, quan sát vẽ lại lãng phí (xem D14_custompainter-canvas-paint.md)</summary>

Đã có sẵn trong repo tại `lib/tuan3_ngay1/bai4_should_repaint_demo_screen.dart` — thay vì
chỉ dùng DevTools, code tự đếm số lần `paint()` bằng `ValueNotifier`:

```dart
bool shouldRepaint(_CirclePainter old) {
  if (alwaysRepaint) return true;          // ❌ luôn vẽ lại
  return old.phanTram != phanTram;         // ✅ chỉ vẽ lại khi % đổi
}
```

Kiểm chứng: bấm nút "Rebuild KHÔNG đổi %" nhiều lần — số đếm bên "❌ shouldRepaint = true"
tăng liên tục dù `phanTram` không đổi, còn bên "✅ shouldRepaint đúng" giữ nguyên số đếm.
</details>

<details><summary>Bài 5 — TaskCreatedEvent + publish trong taoTask (xem D14_spring-events-eventlistener-async.md)</summary>

Đã có sẵn trong repo tại `event/TaskCreatedEvent.java` + `TaskServiceImpl.java`:

```java
public class TaskCreatedEvent {
    private final Task task;
    public TaskCreatedEvent(Task task) { this.task = task; }
    public Task getTask() { return task; }
}

// Trong TaskServiceImpl.create(), SAU khi repository.save():
eventPublisher.publishEvent(new TaskCreatedEvent(task));
```

```java
@Component
public class TaskEventListener {
    @EventListener
    public void ghiLog(TaskCreatedEvent event) {
        System.out.println("=== [EVENT - Log] Task mới được tạo | id=" + event.getTask().getId());
    }
}
```

Kiểm chứng: gọi `POST /api/tasks` qua Postman, quan sát dòng log `[EVENT - Log]` xuất
hiện trong console.
</details>

<details><summary>Bài 6 — Listener thứ 2: cập nhật thống kê (xem D14_spring-events-eventlistener-async.md)</summary>

Đã có sẵn trong repo — `capNhatThongKe()` trong cùng `TaskEventListener`:

```java
@EventListener
public void capNhatThongKe(TaskCreatedEvent event) {
    System.out.println("=== [EVENT - ThongKe] Cập nhật số liệu | tổng task mới hôm nay +1");
}
```

Kiểm chứng: gọi `POST /api/tasks` 1 lần, xác nhận **cả 2 dòng log** (`Log` và `ThongKe`)
đều xuất hiện — `TaskServiceImpl` không hề thay đổi gì để thêm Listener thứ 2 này.
</details>

<details><summary>Bài 7 — @Async cho Listener gửi email (xem D14_spring-events-eventlistener-async.md)</summary>

Đã có sẵn trong repo — `giamapEmailThongBao()`:

```java
@EventListener
@Async
public void giamapEmailThongBao(TaskCreatedEvent event) throws InterruptedException {
    System.out.println("=== [EVENT - Email] Bắt đầu gửi email | thread=" + Thread.currentThread().getName());
    Thread.sleep(2000);
    System.out.println("=== [EVENT - Email] Đã gửi email thông báo cho admin");
}
```

Cần `@EnableAsync` ở `DemoApplication` (đã có sẵn). Kiểm chứng: gọi `POST /api/tasks`, xác
nhận response trả về gần như ngay lập tức (không đợi 2 giây), và tên thread in ra ở log
`[EVENT - Email]` khác với tên thread ở `[EVENT - Log]`/`[EVENT - ThongKe]`.
</details>

---

## ✅ CHECKLIST HOÀN THÀNH

- [ ] Phân biệt được khi nào cần Isolate (`compute()`) và khi nào chỉ cần `async/await` (I/O-bound vs CPU-bound)
- [ ] Giải thích được vì sao hàm chạy trong Isolate phải là top-level/static
- [ ] Chạy được 1 tác vụ nặng qua Isolate, tự kiểm chứng UI không đơ bằng nút Counter
- [ ] Vẽ được vòng tròn tiến độ bằng `CustomPainter`, hiểu vai trò `Canvas`/`Paint`/`TextPainter`
- [ ] Hiểu và tự đo được bằng số liệu (`ValueNotifier`) hiệu quả của `shouldRepaint` đúng
- [ ] Kết hợp được `CustomPainter` với `AnimationController` (explicit animation)
- [ ] `TaskServiceImpl` không còn gọi trực tiếp logic phụ, tách qua `ApplicationEventPublisher`
- [ ] Có ít nhất 2 Listener độc lập cùng lắng nghe `TaskCreatedEvent`
- [ ] Hiểu và kiểm chứng được bằng thread name vì sao `@Async` giúp Listener không làm chậm response chính
- [ ] Biết chính xác cần `@EnableAsync` — thiếu thì `@Async` im lặng không có tác dụng (giống mẫu lỗi `@EnableCaching`/`@EnableMethodSecurity`)

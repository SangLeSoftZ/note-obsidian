---
title: "Ngày 12 (Tuần 3 - Ngày 1, Tối) — Spring Events: Tách hành động phụ khỏi luồng chính"
track: java-spring
day: 12
topic: spring-events-eventlistener-async
tags: [spring-boot, spring-events, eventlistener, applicationeventpublisher, async]
source_goc: Tuan3_Ngay1_TaiLieu.md
---

# Ngày 12 (Tuần 3 - Ngày 1, Tối) — Spring Events: Tách hành động phụ khỏi luồng chính

## 1. Vấn đề: Service "gánh" quá nhiều trách nhiệm phụ

Nếu `TaskServiceImpl.create()` tự gọi trực tiếp log, gửi email, cập nhật thống kê... thì
mỗi lần cần thêm 1 hành động phụ mới lại phải sửa `TaskServiceImpl` — vi phạm nguyên tắc
tách biệt trách nhiệm (Ngày 2 Tuần 1). Spring Events giải quyết bằng mô hình
**publish–subscribe**: Service chỉ "phát tín hiệu", không cần biết ai đang lắng nghe.

**Ẩn dụ:** `TaskServiceImpl` như 1 phát thanh viên — chỉ "phát sóng" thông báo mà **không
cần biết** ai đang nghe. Các "đài thu" (Listener) tự đăng ký và tự quyết định phản ứng.

## 2. `TaskCreatedEvent` — Event object chỉ chứa dữ liệu

Code thật — `event/TaskCreatedEvent.java`:

```java
/**
 * Event object — chỉ lưu dữ liệu, không có logic gì.
 * TaskService "phát" event này sau khi tạo Task thành công.
 * Các Listener "nhận" event này và tự xử lý — TaskService không biết ai nhận.
 */
public class TaskCreatedEvent {
    private final Task task;
    public TaskCreatedEvent(Task task) { this.task = task; }
    public Task getTask() { return task; }
}
```

Đúng như tài liệu gốc — Event chỉ là 1 class dữ liệu đơn giản, không kế thừa gì đặc biệt,
không có logic xử lý bên trong.

## 3. Publish Event — đúng vị trí: SAU KHI lưu DB thành công

Code thật — `service/impl/TaskServiceImpl.java`:

```java
private final TaskRepository            repository;
private final ApplicationEventPublisher eventPublisher;

public TaskServiceImpl(TaskRepository repository, ApplicationEventPublisher eventPublisher) {
    this.repository     = repository;
    this.eventPublisher = eventPublisher;
}

public Task create(TaskRequest dto) {
    ...
    task = repository.save(task);

    // Publish event SAU KHI lưu DB thành công
    // TaskService chỉ "phát tín hiệu" — không biết ai đang lắng nghe
    eventPublisher.publishEvent(new TaskCreatedEvent(task));

    return task;
}
```

Điểm quan trọng dễ bỏ sót: `publishEvent()` được gọi **sau** `repository.save(task)`,
không phải trước. Nếu publish trước khi lưu, các Listener có thể nhận được `Task` chưa có
`id` (chưa được database sinh ra), hoặc tệ hơn — nếu `save()` sau đó thất bại (lỗi
validation, trùng tiêu đề...), Listener đã "tưởng" Task được tạo thành công trong khi thực
tế không có gì trong database cả.

`ApplicationEventPublisher` được **inject qua constructor** — giống hệt cách
`TaskRepository` được inject — đây là dependency injection tiêu chuẩn của Spring, không
có gì đặc biệt so với các Service khác đã học.

## 4. `TaskEventListener` — nhiều Listener độc lập, `TaskService` không biết chúng tồn tại

Code thật — `event/TaskEventListener.java`, đủ cả 3 bài (5, 6, 7) trong 1 class:

```java
@Component
public class TaskEventListener {

    // Bài 5 — Listener 1: Ghi log
    // Chạy ĐỒNG BỘ trên cùng thread với request
    // → TaskService phải chờ method này xong mới tiếp tục
    @EventListener
    public void ghiLog(TaskCreatedEvent event) {
        System.out.println("=== [EVENT - Log] Task mới được tạo"
            + " | id=" + event.getTask().getId()
            + " | tiêu đề=" + event.getTask().getTieuDe()
            + " | thread=" + Thread.currentThread().getName());
    }

    // Bài 6 — Listener 2: Cập nhật thống kê
    // Vẫn đồng bộ — TaskService vẫn không biết listener này tồn tại
    // Nhưng cả 2 đều chạy khi tạo Task mới
    @EventListener
    public void capNhatThongKe(TaskCreatedEvent event) {
        System.out.println("=== [EVENT - ThongKe] Cập nhật số liệu"
            + " | tổng task mới hôm nay +1"
            + " | thread=" + Thread.currentThread().getName());
    }
}
```

Cả `ghiLog` và `capNhatThongKe` cùng nằm trong 1 class `@Component` — nhưng về nguyên
tắc, chúng hoàn toàn có thể nằm ở 2 class khác nhau, thậm chí 2 package khác nhau — điểm
mấu chốt là **mỗi Listener độc lập, không biết Listener khác tồn tại**, và cả hai chỉ cần
đăng ký lắng nghe đúng kiểu Event (`TaskCreatedEvent`) là tự động được gọi khi có event
đó phát ra — không cần đăng ký thủ công ở đâu khác (`@EventListener` + `@Component` là đủ
để Spring tự quét và kết nối).

## 5. `@Async` — Listener 3 (gửi email) chạy trên thread khác

```java
// Bài 7 — Listener 3: Gửi email (giả lập) — ASYNC
// @Async: chạy trên thread KHÁC → request trả về ngay, không chờ
// Quan sát: thread name sẽ khác với 2 listener trên
@EventListener
@Async
public void giamapEmailThongBao(TaskCreatedEvent event) throws InterruptedException {
    System.out.println("=== [EVENT - Email] Bắt đầu gửi email"
        + " | thread=" + Thread.currentThread().getName());

    Thread.sleep(2000); // giả lập gửi email mất 2 giây

    System.out.println("=== [EVENT - Email] Đã gửi email thông báo cho admin"
        + " | task id=" + event.getTask().getId());
}
```

**Vì sao `@Async` quan trọng?** Không có `@Async`, `giamapEmailThongBao` chạy **đồng bộ
ngay trong request** — người dùng gọi `POST /api/tasks` phải **chờ đủ 2 giây** (thời gian
giả lập gửi email) mới nhận được response, dù việc gửi email không liên quan trực tiếp
tới việc "tạo Task thành công hay chưa". Có `@Async`, Listener này chạy "ngầm" ở 1 thread
khác — API trả response ngay sau khi lưu Task + publish event, không chờ email gửi xong.

Cần bật `@EnableAsync` ở `DemoApplication` — code thật:

```java
@SpringBootApplication
@EnableCaching // Bài 5: bật cơ chế cache
@EnableAsync   // Bài 7: bật @Async cho Listener chạy trên thread riêng
public class DemoApplication { ... }
```

Đúng mẫu lỗi quen thuộc đã gặp ở `@EnableMethodSecurity` (Ngày 2) và `@EnableCaching`
(Ngày 4): thiếu `@EnableAsync` thì `@Async` **im lặng không có tác dụng** — Listener vẫn
chạy, nhưng chạy đồng bộ như bình thường, không có exception báo lỗi.

**Cách kiểm chứng `@Async` hoạt động đúng:** so sánh `Thread.currentThread().getName()`
được in ra ở cả 3 Listener. `ghiLog` và `capNhatThongKe` (không `@Async`) sẽ in cùng 1 tên
thread với thread đang xử lý HTTP request (thường có dạng `http-nio-8080-exec-N`), còn
`giamapEmailThongBao` (có `@Async`) sẽ in ra 1 tên thread khác hẳn (thường có dạng
`task-N`, do Spring's `SimpleAsyncTaskExecutor` hoặc `ThreadPoolTaskExecutor` quản lý).

## Bài tập

**Bài 5 (đã có sẵn trong repo):** `TaskCreatedEvent` + publish trong `create()` +
`ghiLog()` đã có sẵn. Gọi `POST /api/tasks` qua Postman, quan sát log console xác nhận
dòng `"=== [EVENT - Log] Task mới được tạo..."` xuất hiện.

**Bài 6 (đã có sẵn trong repo):** `capNhatThongKe()` đã có sẵn — Listener thứ 2 lắng nghe
cùng `TaskCreatedEvent`. Xác nhận cả 2 dòng log (`Log` và `ThongKe`) đều xuất hiện khi tạo
1 Task, chứng minh nhiều Listener độc lập cùng phản ứng với 1 Event.

**Bài 7 (đã có sẵn trong repo):** `giamapEmailThongBao()` đã có `@Async`. Gọi
`POST /api/tasks`, đo thời gian phản hồi (response nên trả về gần như ngay lập tức, không
đợi đủ 2 giây), rồi quan sát log xuất hiện trễ hơn ~2 giây sau đó với tên thread khác 2
Listener còn lại.

##  Code Example (repo `java.git`)

> Xem nguyên các file tại branch `Spring-Events`:
> [`TaskCreatedEvent.java`](https://github.com/SangLeSoftZ/java/blob/Spring-Events/src/main/java/com/example/demo/event/TaskCreatedEvent.java) ·
> [`TaskEventListener.java`](https://github.com/SangLeSoftZ/java/blob/Spring-Events/src/main/java/com/example/demo/event/TaskEventListener.java) ·
> [`TaskServiceImpl.java`](https://github.com/SangLeSoftZ/java/blob/Spring-Events/src/main/java/com/example/demo/service/impl/TaskServiceImpl.java) ·
> [`DemoApplication.java`](https://github.com/SangLeSoftZ/java/blob/Spring-Events/src/main/java/com/example/demo/DemoApplication.java)


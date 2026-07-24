---
title: "Ngày 18 (Tuần 4 - Ngày 2, Tối) — @Scheduled và @Async: Tác vụ định kỳ + không chặn luồng"
track: java-spring
day: 18
topic: scheduled-async-task-scheduler
tags: [spring-boot, scheduled, async, cron, self-invocation]
source_goc: Tuan4-Ngay2-SecureStorage-DioInterceptor-Scheduled-Async.md
---

# Ngày 18 (Tuần 4 - Ngày 2, Tối) — @Scheduled và @Async: Tác vụ định kỳ + không chặn luồng

## 1. Vấn đề: cần chạy tác vụ mà không có request nào "gọi" nó

Toàn bộ code từ đầu khóa chạy theo mô hình: có request HTTP tới → controller xử lý → trả
response. Nhưng có những việc **không** gắn với request nào của người dùng — VD: định kỳ
quét toàn bộ Task để thống kê/xử lý. Không ai "bấm nút" để kích hoạt — nó cần tự chạy theo
**lịch cố định**.

## 2. Bật cả 3 cơ chế cùng lúc trong `DemoApplication`

Code thật — `DemoApplication.java` — đến hôm nay đã tích lũy đủ 3 annotation `@Enable...`
quen thuộc từ các bài trước:

```java
@SpringBootApplication
@EnableCaching    // Tuần 2 — Bài 5: bật cache
@EnableAsync      // Tuần 2+3: bật @Async — chạy trên thread riêng
@EnableScheduling // Tuần 3 — Bài 6: bật @Scheduled — chạy tác vụ định kỳ
public class DemoApplication {
    public static void main(String[] args) { SpringApplication.run(DemoApplication.class, args); }
}
```

Đúng mẫu lỗi quen thuộc xuyên suốt khóa học (`@EnableMethodSecurity`,
`@EnableJpaAuditing`...): thiếu `@EnableScheduling` thì `@Scheduled` **im lặng không chạy
gì cả**, không có exception báo lỗi.

## 3. `TaskQuaHanScheduler` — điều chỉnh thực tế so với tài liệu lý thuyết gốc

**Khác biệt quan trọng:** tài liệu lý thuyết gốc giả định `Task` có sẵn field
`ngayHetHan` và `quaHan`, dùng
`taskRepository.findByNgayHetHanBeforeAndHoanThanh(LocalDateTime.now(), false)`. Nhưng
entity `Task` thật (đã xây từ Tuần 3 Ngày 3 — JPA Auditing) **không có** 2 field này. Code
thật điều chỉnh lại cho khớp đúng schema hiện có — quét theo `trangThai` thay vì
`ngayHetHan`:

```java
@Component
public class TaskQuaHanScheduler {

    private final TaskRepository taskRepository;

    public TaskQuaHanScheduler(TaskRepository taskRepository) {
        this.taskRepository = taskRepository;
    }

    // Cron để test: "0 */1 * * * *" = chạy mỗi 1 phút (không phải chờ cả ngày)
    // Production nên đổi lại: "0 0 0 * * *" = chạy 00:00:00 mỗi ngày
    @Scheduled(cron = "0 */1 * * * *")
    public void quetTaskChuaHoanThanh() {
        List<Task> chuaHoanThanh = taskRepository.findAll()
                .stream()
                .filter(t -> !"HOAN_THANH".equals(t.getTrangThai()))
                .toList();

        System.out.printf(
            "=== [Scheduler] Quét định kỳ: có %d Task chưa hoàn thành | thread=%s%n",
            chuaHoanThanh.size(),
            Thread.currentThread().getName()
        );

        // Nếu có Task CHUA_LAM quá lâu, có thể tự động chuyển sang DANG_LAM...
        // Bài học chỉ demo log — logic thật thêm sau theo yêu cầu
    }

    // Ví dụ fixedRate: chạy mỗi 30 giây (tính từ lúc lần trước BẮT ĐẦU)
    // @Scheduled(fixedRate = 30000)
    // public void kiemTraHeThong() { ... }

    // Ví dụ fixedDelay: chờ 30 giây SAU KHI lần trước KẾT THÚC mới chạy
    // @Scheduled(fixedDelay = 30000)
    // public void dongBoData() { ... }
}
```

**Bài học thực tế quan trọng:** khi áp dụng lý thuyết vào code có sẵn, không phải lúc nào
schema cũng khớp 100% với ví dụ minh họa — cần **điều chỉnh logic cho phù hợp dữ liệu thật
đang có**, thay vì thêm field mới chỉ để khớp đúng ví dụ. Ở đây, "quá hạn" được diễn giải
lại thành "chưa hoàn thành" (`trangThai != HOAN_THANH`) — vẫn giữ đúng tinh thần bài học
(quét định kỳ, không cần request kích hoạt), chỉ khác tiêu chí lọc.

`cron = "0 */1 * * * *"` dùng để **test nhanh** (mỗi 1 phút) thay vì phải đợi cả ngày mới
thấy tác vụ chạy — comment ghi rõ khi lên production cần đổi lại thành
`"0 0 0 * * *"` (00:00:00 mỗi ngày). 2 ví dụ `fixedRate`/`fixedDelay` được để lại dạng
comment tham khảo, không kích hoạt, cho người học tự thử khi cần.

`Thread.currentThread().getName()` được in ra mỗi lần chạy — cách kiểm chứng trực tiếp
rằng tác vụ `@Scheduled` chạy trên **thread riêng của bộ lập lịch**, tách biệt khỏi các
thread xử lý HTTP request.

## 4. `ThongBaoService.guiThongBaoTaoTask()` — `@Async` áp dụng đúng vào luồng tạo Task thật

Code thật — `service/ThongBaoService.java`:

```java
/**
 * QUAN TRỌNG: @Async chỉ hoạt động khi gọi từ CLASS KHÁC.
 * Nếu TaskServiceImpl tự gọi method @Async của chính nó (self-invocation)
 * → Spring AOP Proxy bỏ qua annotation → chạy đồng bộ như bình thường.
 */
@Service
public class ThongBaoService {

    @Async
    public void guiThongBaoTaoTask(Task task) {
        System.out.printf(
            "=== [Async - ThongBao] Bắt đầu gửi thông báo | task=%s | thread=%s%n",
            task.getTieuDe(), Thread.currentThread().getName()
        );
        try {
            Thread.sleep(2000); // giả lập gọi dịch vụ email bên ngoài mất 2 giây
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        System.out.printf("=== [Async - ThongBao] Đã gửi thông báo cho Task: %s%n", task.getTieuDe());
    }
}
```

Không giống ví dụ đơn lẻ trong tài liệu lý thuyết gốc, `@Async` ở đây được **thêm thẳng
vào luồng tạo Task thật** — `TaskServiceImpl.create()`:

```java
public Task create(TaskRequest dto) {
    ...
    task = repository.save(task);

    // Tuần 2: publish event (WebSocket, Email listener...)
    eventPublisher.publishEvent(new TaskCreatedEvent(task));

    // Tuần 3 — Bài 7: gọi @Async từ class khác — response trả về ngay,
    // thongBaoService.guiThongBaoTaoTask() chạy ngầm trên thread riêng
    thongBaoService.guiThongBaoTaoTask(task);

    return task;
}
```

**Điểm quan trọng nhất — đây chính là ví dụ ĐÚNG của Bài 8, không phải ví dụ sai:**
`thongBaoService` là 1 **bean khác** (`ThongBaoService`), được inject qua constructor vào
`TaskServiceImpl` — gọi `thongBaoService.guiThongBaoTaoTask(task)` là gọi **từ class khác**
tới **bean khác** (đi qua Spring AOP Proxy), nên `@Async` hoạt động đúng: `create()` trả
`task` về **ngay lập tức** sau dòng gọi này, không chờ 2 giây giả lập bên trong
`guiThongBaoTaoTask`.

## 5. Vì sao self-invocation làm `@Async` mất tác dụng — cơ chế Proxy

Nếu `TaskServiceImpl` tự viết thêm 1 method `@Async` **ngay trong chính nó** rồi tự gọi
bằng `this.guiThongBaoNoiBo(task)`:

```java
@Service
public class TaskServiceImpl {
    @Async
    public void guiThongBaoNoiBo(Task task) { ... } // method @Async trong CHÍNH class này

    public Task create(TaskRequest dto) {
        ...
        this.guiThongBaoNoiBo(task); // GỌI NỘI BỘ — @Async sẽ KHÔNG có tác dụng!
        return task;
    }
}
```

**Cơ chế bên dưới:** Spring không chạy trực tiếp instance `TaskServiceImpl` bạn viết — nó
tạo ra 1 **Proxy bao quanh** instance đó. Khi code **bên ngoài** (VD Controller) gọi
`taskServiceImpl.create(...)`, lời gọi đi **qua Proxy trước**, Proxy kiểm tra thấy method
đích có `@Async` thì mới "chuyển việc" sang thread khác. Nhưng khi `create()` tự gọi
`this.guiThongBaoNoiBo(...)` — `this` ở đây là **instance gốc**, không phải Proxy — lời
gọi đi **thẳng, bỏ qua Proxy hoàn toàn**, nên `@Async` (và mọi annotation dựa trên AOP
Proxy khác như `@Transactional`, `@Cacheable`) đều **không có tác dụng** trong tình huống
self-invocation này.

**Đây là lý do repo thiết kế tách hẳn `ThongBaoService` thành 1 class/bean riêng** thay vì
viết method `@Async` ngay trong `TaskServiceImpl` — không chỉ vì tách biệt trách nhiệm
(Single Responsibility), mà còn là cách **duy nhất đảm bảo `@Async` chắc chắn hoạt động**.

## 6. So sánh nhanh

| | `@Scheduled` | `@Async` |
|---|---|---|
| Kích hoạt bởi | Thời gian (lịch cố định) | Lời gọi hàm từ **class khác** |
| Mục đích chính | Chạy tác vụ định kỳ, không cần request nào | Chạy tác vụ không chặn luồng đang xử lý |
| Ví dụ trong repo | Quét Task chưa hoàn thành mỗi phút (test) / mỗi đêm (production) | Gửi thông báo sau khi tạo Task |
| Bẫy thường gặp | Quên `@EnableScheduling` | Self-invocation (gọi `this.method()` trong cùng class) |

## Bài tập

**Bài 6 (đã có sẵn — `TaskQuaHanScheduler.quetTaskChuaHoanThanh()`):** Chạy server, đợi
tối đa 1 phút, xác nhận log `[Scheduler] Quét định kỳ...` tự động xuất hiện mà không có
request nào được gọi.

**Bài 7 (đã có sẵn — `ThongBaoService` + `TaskServiceImpl.create()`):** Gọi
`POST /api/tasks` qua Postman, đo thời gian phản hồi — xác nhận response trả về **nhanh**
(không đợi đủ 2 giây), sau đó quan sát log `[Async - ThongBao]` xuất hiện trễ hơn với tên
thread khác thread xử lý request.

**Bài 8 (nâng cao, không có sẵn — tự làm để kiểm chứng):** Thử tự thêm 1 method `@Async`
**ngay trong `TaskServiceImpl`** rồi gọi bằng `this.method(...)` thay vì qua
`ThongBaoService` — quan sát response bị **chặn đủ 2 giây** dù có `@Async`, từ đó tự xác
nhận đúng giới hạn self-invocation đã học ở mục 5.

## Code Example (repo `java.git`)

> Xem nguyên các file tại branch `@Scheduled-@Async`:
> [`DemoApplication.java`](<https://github.com/SangLeSoftZ/java/blob/@Scheduled-@Async/src/main/java/com/example/demo/DemoApplication.java>) ·
> [`TaskQuaHanScheduler.java`](<https://github.com/SangLeSoftZ/java/blob/@Scheduled-@Async/src/main/java/com/example/demo/scheduler/TaskQuaHanScheduler.java>) ·
> [`ThongBaoService.java`](<https://github.com/SangLeSoftZ/java/blob/@Scheduled-@Async/src/main/java/com/example/demo/service/ThongBaoService.java>) ·
> [`TaskServiceImpl.java`](<https://github.com/SangLeSoftZ/java/blob/@Scheduled-@Async/src/main/java/com/example/demo/service/impl/TaskServiceImpl.java>)
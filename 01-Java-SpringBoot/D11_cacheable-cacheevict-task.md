---
title: "Ngày 11 (Tuần 2 - Ngày 4, Tối) — @Cacheable: Giảm tải database cho Task"
track: java-spring
day: 11
topic: cacheable-cacheevict-task
tags: [spring-boot, caching, cacheable, cacheevict, cachemanager]
source_goc: Tuan2_Ngay4_TaiLieu.md
---

# Ngày 11 (Tuần 2 - Ngày 4, Tối) — @Cacheable: Giảm tải database cho Task

## 1. Bật Caching — `@EnableCaching`

Code thật — `DemoApplication.java`:

```java
@SpringBootApplication
@EnableCaching // Tuần 2 — Bài 5: bật cơ chế cache
               // Thiếu dòng này thì @Cacheable/@CacheEvict không có tác dụng!
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

Giống hệt lỗi kinh điển của `@EnableMethodSecurity` (Ngày 2) — thiếu `@EnableCaching` thì
`@Cacheable`/`@CacheEvict` **im lặng, không có tác dụng gì**, không có exception, không có
cảnh báo.

## 2. Lưu ý thực tế quan trọng: phải tự khai báo `CacheManager` (khác với lý thuyết "mặc định")

Tài liệu lý thuyết thường nói: "mặc định Spring Boot tự dùng
`ConcurrentMapCacheManager`, không cần cấu hình gì thêm". Nhưng code thật trong repo có
1 comment rất đáng chú ý — `config/SecurityConfig.java`:

```java
// Tuần 2 — Bài 5: Spring Boot 4 không tự tạo CacheManager
// Phải khai báo thủ công — ConcurrentMapCacheManager lưu trong RAM, không cần Redis
@Bean
public CacheManager cacheManager() {
    return new ConcurrentMapCacheManager("tasks", "taskById");
}
```

**Đây là điểm khác biệt thực tế bạn nên nhớ:** ở phiên bản Spring Boot bạn đang dùng,
việc auto-configure `CacheManager` **không còn tự động xảy ra** như tài liệu cũ mô tả —
phải tự khai báo 1 `@Bean CacheManager` thủ công, và liệt kê rõ tên các cache sẽ dùng
(`"tasks"`, `"taskById"`) ngay trong constructor của `ConcurrentMapCacheManager`. Nếu chỉ
thêm `@Cacheable("tasks")` mà không có `@Bean` này, Spring sẽ báo lỗi không tìm thấy
`CacheManager` khi khởi động app.

## 3. `@Cacheable` cho `getAll()` và `getById()` — đúng code trong `TaskServiceImpl`

```java
// Bài 5: @Cacheable
// Lần đầu: chạy method thật, lưu kết quả vào cache "tasks"
// Lần sau: KHÔNG chạy method, trả thẳng từ cache — không có SQL nào
// Quan sát log: dòng "=== [DB] getAll" chỉ in đúng 1 lần dù gọi nhiều lần
@Override
@Cacheable("tasks")
public List<Task> getAll() {
    System.out.println("=== [DB] getAll — đang truy vấn database...");
    return repository.findAll();
}

// key="#id": mỗi id có 1 ô cache riêng
// getById(1) và getById(2) là 2 ô nhớ khác nhau, không lẫn lộn
@Override
@Cacheable(value = "taskById", key = "#id")
public Task getById(Long id) {
    System.out.println("=== [DB] getById(" + id + ") — đang truy vấn database...");
    return repository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Task", id));
}
```

Cách quan sát cache có hoạt động đúng hay không rất đơn giản: gọi `GET /api/tasks` (hoặc
`GET /api/tasks/{id}`) 2 lần liên tiếp qua Postman, nhìn log console — dòng
`"=== [DB] getAll — đang truy vấn database..."` chỉ được in ra **đúng 1 lần** ở lần gọi
đầu tiên. Lần gọi thứ 2 trở đi, Spring chặn method lại từ trước khi nó chạy, trả thẳng kết
quả cũ trong cache — log không in gì thêm, chứng tỏ không có câu SQL nào chạy.

## 4. `@CacheEvict` — dọn dẹp cache khi dữ liệu Task thay đổi

Nếu chỉ có `@Cacheable` mà không dọn dẹp: admin tạo Task mới, nhưng `GET /api/tasks` vẫn
trả về danh sách **cũ** (thiếu Task vừa tạo) cho tới khi app restart. Code thật xử lý
đúng 3 trường hợp thay đổi dữ liệu — tạo, sửa, xóa:

```java
@Override
@CacheEvict(value = "tasks", allEntries = true)
public Task create(TaskRequest dto) {
    System.out.println("=== [DB] create — xóa cache 'tasks'");
    ...
}

// Xóa cả 2 cache: list + item cụ thể
@Override
@CacheEvict(value = {"tasks", "taskById"}, allEntries = true)
public Task update(Long id, TaskRequest dto) {
    System.out.println("=== [DB] update(" + id + ") — xóa cache 'tasks' + 'taskById'");
    ...
}

@Override
@CacheEvict(value = {"tasks", "taskById"}, allEntries = true)
public void delete(Long id) {
    System.out.println("=== [DB] delete(" + id + ") — xóa cache 'tasks' + 'taskById'");
    ...
}
```

Điểm cần chú ý:
- **`create()`** chỉ cần xóa cache `"tasks"` (danh sách) — vì Task mới chưa từng có
  trong `"taskById"`, không có gì để xóa ở đó.
- **`update()`/`delete()`** phải xóa **cả 2 cache** (`"tasks"` và `"taskById"`) — vì Task
  bị sửa/xóa có thể đã được cache riêng lẻ qua `getById()` trước đó, nếu không xóa,
  `GET /api/tasks/{id}` sẽ tiếp tục trả dữ liệu cũ dù danh sách chung đã đúng.
- **`allEntries = true`**: xóa **toàn bộ** entry trong cache đó (mọi id), không chỉ 1 id
  cụ thể — cách đơn giản, đảm bảo an toàn (dù kém tối ưu hơn chỉ xóa đúng 1 entry), phù
  hợp với quy mô bài học.

## 5. Vì sao `timKiem()` (Specification API — Ngày 3) không được cache

Code thật để nguyên comment rất rõ ràng:

```java
// Specification — không cache vì kết quả thay đổi theo filter
@Override
public List<Task> timKiem(String tuKhoa, String trangThai,
                           LocalDateTime tuNgay, LocalDateTime denNgay) {
    ...
}
```

Đây là nguyên tắc quan trọng khi quyết định method nào nên `@Cacheable`: chỉ cache những
method có **kết quả ổn định theo cùng 1 bộ tham số** và **được gọi lại nhiều lần với
cùng tham số**. `timKiem()` nhận 4 tham số filter khác nhau mỗi lần gọi (tuKhoa, trangThai,
khoảng ngày) — số lượng tổ hợp tham số gần như vô hạn, cache sẽ phình to vô ích mà tỷ lệ
"trúng cache" (cache hit) lại rất thấp. Ngược lại, `getAll()` không có tham số và
`getById(id)` có tập tham số hữu hạn (số lượng Task) — phù hợp để cache.

## Bài tập

**Bài 5 (đã có sẵn trong repo):** `@EnableCaching` + `@Cacheable("tasks")` +
`@Cacheable(value = "taskById", key = "#id")` đã có sẵn trong `TaskServiceImpl`. Tự kiểm
chứng: gọi `GET /api/tasks` 2 lần liên tiếp qua Postman/Swagger, quan sát log console —
xác nhận dòng `"=== [DB] getAll..."` chỉ in ra đúng 1 lần.

**Bài 6 (đã có sẵn trong repo):** `@CacheEvict` đã áp cho `create`/`update`/`delete`. Tự
kiểm chứng đầy đủ luồng: gọi `GET /api/tasks` (cache lại) → `POST /api/tasks` tạo task mới
→ gọi lại `GET /api/tasks` → xác nhận thấy task mới xuất hiện ngay lập tức, không phải đợi
cache hết hạn hay restart app.

## Code Example (repo `java.git`)

> Xem nguyên các file tại branch `tuan2`:
> [`DemoApplication.java`](https://github.com/SangLeSoftZ/java/blob/tuan2/src/main/java/com/example/demo/DemoApplication.java) ·
> [`SecurityConfig.java`](https://github.com/SangLeSoftZ/java/blob/tuan2/src/main/java/com/example/demo/config/SecurityConfig.java) (phần `@Bean CacheManager`) ·
> [`TaskServiceImpl.java`](https://github.com/SangLeSoftZ/java/blob/tuan2/src/main/java/com/example/demo/service/impl/TaskServiceImpl.java) ·
> [`TaskController.java`](https://github.com/SangLeSoftZ/java/blob/tuan2/src/main/java/com/example/demo/controller/TaskController.java)



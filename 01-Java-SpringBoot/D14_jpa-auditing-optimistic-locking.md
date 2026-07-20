---
title: "Ngày 14 (Tuần 3 - Ngày 3, Tối) — JPA Auditing + Optimistic Locking cho Task"
track: java-spring
day: 14
topic: jpa-auditing-optimistic-locking
tags: [spring-boot, jpa, auditing, createddate, lastmodifieddate, version, optimistic-locking]
source_goc: Tuan3-Ngay3-Generics-Mixin-Riverpod-JPAAuditing.md
---

# Ngày 16 (Tuần 3 - Ngày 3, Tối) — JPA Auditing + Optimistic Locking cho Task

## 1. JPA Auditing — bật cơ chế, không code tay từng chỗ `save()`

Cách làm thủ công (dễ quên, dễ sai): mỗi lần tạo/sửa Task đều phải tự set
`task.setNgayTao(LocalDateTime.now())` ở đúng mọi chỗ gọi `save()` — quên 1 chỗ là dữ
liệu sai/thiếu. JPA Auditing để Spring **tự động** gán giá trị này.

Code thật — `config/JpaAuditingConfig.java`:

```java
/**
 * @EnableJpaAuditing — bật cơ chế tự động ghi ngày tạo/sửa.
 * THIẾU annotation này thì @CreatedDate/@LastModifiedDate luôn là null
 * mà không báo lỗi gì — rất dễ gây nhầm lẫn khi debug.
 */
@Configuration
@EnableJpaAuditing
public class JpaAuditingConfig {
    // Không cần code thêm — annotation lo hết
}
```

Đây đúng mẫu lỗi quen thuộc đã gặp nhiều lần trong khóa học (`@EnableMethodSecurity`,
`@EnableCaching`, `@EnableAsync`): thiếu `@Enable...` thì cơ chế liên quan **im lặng
không hoạt động**, không có exception báo lỗi rõ ràng.

## 2. Entity `Task` — cả Auditing lẫn Optimistic Locking trong cùng 1 lần cập nhật

Code thật — `entity/Task.java` (khác với ví dụ lý thuyết gốc ở chỗ: đây là bản đầy đủ,
gộp cả 3 nội dung Bài 7, 8, 9 vào cùng 1 entity thật đang dùng cho toàn bộ CRUD Task từ
đầu khóa, không phải entity demo riêng):

```java
@Entity
@Table(name = "tasks")
@EntityListeners(AuditingEntityListener.class) // BẮT BUỘC để Auditing hoạt động
public class Task {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "tieu_de", nullable = false, unique = true)
    private String tieuDe;

    // ── Bài 7: JPA Auditing ───────────────────────────────────────────
    @CreatedDate
    @Column(name = "created_at", updatable = false) // chặn sửa dù có lỡ gọi setter
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    // ── Bài 9: Optimistic Locking ─────────────────────────────────────
    // Hibernate tự tăng phienBan mỗi lần UPDATE
    // Nếu 2 người cùng sửa 1 Task: người sửa sau nhận OptimisticLockException
    // → trả về 409 Conflict thay vì ghi đè âm thầm (lost update)
    @Version
    @Column(name = "phien_ban")
    private Integer phienBan = 0;

    // Lưu ý: getter cho createdAt/updatedAt/phienBan — KHÔNG có setter
    // Không cho phép set tay 3 field này — đúng đắn vì đây là dữ liệu
    // Spring/Hibernate tự quản lý hoàn toàn
}
```

**Chi tiết dễ bỏ sót:** cả 3 field `createdAt`, `updatedAt`, `phienBan` đều **chỉ có
getter, không có setter** trong code thật. Đây là thiết kế cố ý — nếu có setter, lập
trình viên (hoặc chính bạn lúc code nhanh, quên mất) có thể vô tình gọi
`task.setUpdatedAt(...)` ghi đè giá trị Spring vừa tự động gán, làm hỏng toàn bộ ý nghĩa
của Auditing/Optimistic Locking. Không có setter là cách "khóa" ở tầng compile-time,
chắc chắn hơn nhiều so với chỉ dựa vào `@Column(updatable = false)` (chỉ chặn ở tầng DB).

**Cập nhật `schema.sql` tương ứng:**

```sql
CREATE TABLE IF NOT EXISTS tasks (
    ...
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,  -- Bài 8: JPA Auditing ngaySua
    phien_ban  INT       NOT NULL DEFAULT 0           -- Bài 9: Optimistic Locking
);
```

`phien_ban INT NOT NULL DEFAULT 0` khớp đúng với `private Integer phienBan = 0;` ở phía
Java — Task mới tạo luôn bắt đầu ở phiên bản 0, tăng dần 1 mỗi lần `UPDATE` thành công.

## 3. Cơ chế bên dưới — Hibernate tự thêm điều kiện `WHERE ... AND phien_ban = ?`

```sql
-- Không có @Version, UPDATE bình thường:
UPDATE task SET tieu_de = 'Học Flutter nâng cao' WHERE id = 5;

-- Có @Version, Hibernate tự động thêm điều kiện:
UPDATE task SET tieu_de = 'Học Flutter nâng cao', phien_ban = 2
WHERE id = 5 AND phien_ban = 1; -- 1 là phiên bản lúc entity được đọc lên
-- Nếu phien_ban trong DB đã là 2 (người khác sửa trước) → câu lệnh này khớp 0 dòng
-- Hibernate phát hiện 0 dòng bị ảnh hưởng → ném OptimisticLockException
```

## 4. Xử lý xung đột trong `TaskServiceImpl.update()` — bắt đúng exception của Spring

Code thật — `service/impl/TaskServiceImpl.java`:

```java
@Override
@CacheEvict(value = {"tasks", "taskById"}, allEntries = true)
public Task update(Long id, TaskRequest dto) {
    System.out.println("=== [DB] update(" + id + ") — xóa cache 'tasks' + 'taskById'");
    try {
        Task task = repository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Task", id));

        boolean tieuDeThayDoi = !task.getTieuDe().equals(dto.getTieuDe());
        if (tieuDeThayDoi && repository.existsByTieuDe(dto.getTieuDe())) {
            throw new BusinessException("tieuDe", dto.getTieuDe(),
                    "Task với tiêu đề '" + dto.getTieuDe() + "' đã tồn tại");
        }

        task.setTieuDe(dto.getTieuDe());
        task.setMoTa(dto.getMoTa());
        if (dto.getTrangThai() != null && !dto.getTrangThai().isBlank()) {
            task.setTrangThai(dto.getTrangThai());
        }
        return repository.save(task);

    } catch (OptimisticLockingFailureException e) {
        // Xảy ra khi 2 người cùng sửa 1 Task — người sửa sau thua
        // Hibernate phát hiện phienBan trong DB đã thay đổi → ném exception
        throw new ResponseStatusException(
            HttpStatus.CONFLICT,
            "Task đã bị người khác cập nhật. Vui lòng tải lại và thử lại."
        );
    }
}
```

**Điểm khác với ví dụ lý thuyết gốc:** code thật bắt `OptimisticLockingFailureException`
(package `org.springframework.dao`) — đây là **class cha tổng quát hơn** của
`ObjectOptimisticLockingFailureException` mà tài liệu gốc dùng làm ví dụ.
`ObjectOptimisticLockingFailureException` kế thừa từ `OptimisticLockingFailureException`
— bắt class cha vẫn bắt được đúng lỗi này (và cả các biến thể tương tự khác nếu có), nên
đây là lựa chọn **an toàn hơn**, không hẹp hơn.

Toàn bộ khối `try/catch` bao quanh cả logic kiểm tra trùng tiêu đề (`BusinessException`)
lẫn logic cập nhật — nghĩa là `update()` giờ xử lý gọn cả 2 loại lỗi nghiệp vụ trong cùng
1 method, đúng tinh thần Exception Handling đã học từ Tuần 1.

**Vì sao trả HTTP 409 (Conflict) thay vì 400/500?** 409 đúng ngữ nghĩa nhất cho tình huống
"request hợp lệ về cú pháp/dữ liệu, nhưng xung đột với trạng thái hiện tại của tài nguyên
trên server" — giúp phía Flutter phân biệt được đây là lỗi **xung đột đồng thời** (nên
hiển thị "dữ liệu đã thay đổi, tải lại đi") khác hẳn lỗi validate (400) hay lỗi server
(500).

## 5. Optimistic vs Pessimistic Locking — biết để phân biệt

- **Optimistic (lạc quan)** — hôm nay đang học: **không khóa** dữ liệu khi đọc, chỉ
  **kiểm tra** lúc ghi. Giả định xung đột **hiếm** xảy ra. Phù hợp hệ thống đọc nhiều, ghi
  ít/hiếm xung đột (như Task quản lý cá nhân).
- **Pessimistic (bi quan)** — **khóa** dữ liệu ngay khi đọc (`SELECT ... FOR UPDATE`),
  người khác phải **đợi** tới khi khóa được giải phóng. Phù hợp hệ thống xung đột xảy ra
  **thường xuyên** (VD hệ thống đặt vé, tồn kho số lượng giới hạn).

## Bài tập

**Bài 7 (đã có sẵn — `JpaAuditingConfig.java` + `Task.java`):** Tạo 1 Task mới qua
`POST /api/tasks`, xác nhận `createdAt`/`updatedAt` tự động có giá trị đúng mà không set
tay ở đâu trong request body.

**Bài 8 (đã có sẵn):** Cập nhật lại Task đó (`PUT /api/tasks/{id}`, đổi `tieuDe`), xác
nhận `updatedAt` tự động đổi còn `createdAt` giữ nguyên giá trị ban đầu.

**Bài 9 (đã có sẵn — `TaskServiceImpl.update()`):** Mô phỏng xung đột: mở 2 request
Postman `GET /api/tasks/{id}` lấy cùng 1 Task (cùng `phienBan`), sửa và lưu request 1
trước (`PUT`) — sau đó thử lưu request 2 (đang cầm `phienBan` cũ) — xác nhận nhận được lỗi
**409** với thông báo "Task đã bị người khác cập nhật..." thay vì ghi đè âm thầm.

##  Code Example (repo `java.git`)

> Xem nguyên các file tại branch `JPA-Auditing--Optimistic-Locking`:
> [`JpaAuditingConfig.java`](https://github.com/SangLeSoftZ/java/blob/JPA-Auditing--Optimistic-Locking/src/main/java/com/example/demo/config/JpaAuditingConfig.java) ·
> [`Task.java`](https://github.com/SangLeSoftZ/java/blob/JPA-Auditing--Optimistic-Locking/src/main/java/com/example/demo/entity/Task.java) ·
> [`TaskServiceImpl.java`](https://github.com/SangLeSoftZ/java/blob/JPA-Auditing--Optimistic-Locking/src/main/java/com/example/demo/service/impl/TaskServiceImpl.java) ·
> [`schema.sql`](https://github.com/SangLeSoftZ/java/blob/JPA-Auditing--Optimistic-Locking/src/main/resources/schema.sql)

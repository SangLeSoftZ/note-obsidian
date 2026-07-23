---
title: "Ngày 17 (Tuần 4 - Ngày 1, Tối) — N+1 Query Problem và cách fix bằng @EntityGraph"
track: java-spring
day: 17
topic: n-plus-1-query-entitygraph
tags: [spring-boot, jpa, n-plus-1, entitygraph, joinfetch, lazy-loading]
source_goc: Tuan4-Ngay1-Dart3SealedClass-BlocSelector-N1Query.md
---

# Ngày 17 (Tuần 4 - Ngày 1, Tối) — N+1 Query Problem và cách fix bằng @EntityGraph

## 1. Thêm quan hệ `@ManyToOne` vào `Task` — dùng lại đúng entity đã xây từ Ngày 3

Code thật — `entity/Task.java` — không tạo entity demo riêng, mà thêm thẳng quan hệ mới
vào `Task` đã có sẵn `@CreatedDate`/`@LastModifiedDate`/`@Version` từ Tuần 3 Ngày 3 (JPA
Auditing + Optimistic Locking):

```java
// ── N+1 Demo ─────────────────────────────────────────────────────
// FetchType.LAZY (mặc định của @ManyToOne):
//   → Hibernate KHÔNG lấy User khi lấy Task
//   → Mỗi lần gọi nguoiTao.getUsername() = 1 câu SQL riêng
//   → 3 Task = 4 câu query (Bài 7: quan sát trong log)
//
// Sau khi sửa bằng @EntityGraph (Bài 8):
//   → Chỉ còn 1 câu SQL duy nhất (JOIN FETCH)
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "nguoi_tao_id")
private User nguoiTao;
```

**Lưu ý số liệu thực tế:** dataset học tập chỉ có **3 Task** seed sẵn (không phải 20 như
ví dụ minh họa trong tài liệu gốc) — nên hiện tượng N+1 ở đây là **3 + 1 = 4 câu query**,
không phải 21 câu. Con số nhỏ này vẫn đủ để quan sát rõ trong log, và giúp hình dung dễ
hơn: N+1 không phụ thuộc vào việc dataset lớn hay nhỏ, mà phụ thuộc vào **số bản ghi có
quan hệ cần load riêng** — với dataset thật hàng chục nghìn dòng, cùng cơ chế này sẽ gây
chậm nghiêm trọng.

`schema.sql` cập nhật tương ứng — thêm cột khóa ngoại:

```sql
CREATE TABLE IF NOT EXISTS tasks (
    ...
    nguoi_tao_id BIGINT,  -- N+1 Demo: FK tới users
    FOREIGN KEY (nguoi_tao_id) REFERENCES users(id)
);
```

`DataInitializer.java` gán `admin` làm `nguoiTao` cho toàn bộ Task đã seed sẵn — chạy
sau khi user đã được tạo (`userRepository.findByUsername("admin")`), đảm bảo có dữ liệu
`nguoiTao` thật để demo N+1, không phải giá trị `null`.

## 2. `TaskRepository` — tách riêng 2 method, không tái sử dụng `findAll()` đã cache

Code thật — `repository/TaskRepository.java`:

```java
// ── N+1 Demo (Bài 7): LAZY load — mỗi getNguoiTao() = 1 SELECT riêng ──
// Không dùng @EntityGraph → sẽ tạo ra N+1 queries
// Không cache để Hibernate session còn mở, lazy load mới hoạt động
@Query("SELECT t FROM Task t")
List<Task> findAllForNPlusOne();

// ── Fix N+1 (Bài 8): @EntityGraph → JOIN FETCH → 1 query duy nhất ──
@EntityGraph(attributePaths = {"nguoiTao"})
@Query("SELECT t FROM Task t")
List<Task> findAllWithNguoiTao();
```

**Chi tiết quan trọng, không nằm trong tài liệu lý thuyết gốc:** repo **cố tình không**
tái sử dụng `findAll()` gốc (vốn đã có `@Cacheable("tasks")` từ Tuần 2 Ngày 4) để demo
N+1 — comment giải thích rõ lý do: "Không cache để Hibernate session còn mở, lazy load
mới hoạt động". Nếu dùng `findAll()` đã cache, lần gọi thứ 2 trở đi sẽ trả thẳng từ cache
(không chạm database), khiến không thể quan sát lặp lại hiện tượng N+1 mỗi lần gọi API —
tạo 1 method riêng `findAllForNPlusOne()` đảm bảo mỗi lần gọi đều thực sự chạm database,
phục vụ đúng mục đích học tập.

## 3. `NPlusOneController` — 2 endpoint song song, cùng kết quả khác số query

Code thật — `controller/NPlusOneController.java`:

```java
@RestController
@RequestMapping("/api/demo")
public class NPlusOneController {

    // ── Bài 7: CÓ N+1 ──────────────────────────────────────────────
    @Transactional
    @GetMapping("/n-plus-1")
    public ResponseEntity<?> demoNPlus1() {
        System.out.println("\n=== [N+1 Demo] BẮT ĐẦU — CÓ N+1 ===");

        List<Task> tasks = taskRepository.findAllForNPlusOne();

        List<Map<String, Object>> result = tasks.stream().map(task -> {
            String tenNguoiTao = task.getNguoiTao() != null
                    ? task.getNguoiTao().getUsername() // ← mỗi dòng này = 1 SELECT thêm
                    : "Không có";
            ...
        }).collect(Collectors.toList());

        System.out.println("=== [N+1 Demo] KẾT THÚC — đếm số SELECT trong log trên ===\n");
        return ResponseEntity.ok(result);
    }

    // ── Bài 8: ĐÃ FIX bằng @EntityGraph ─────────────────────────────
    @GetMapping("/fix-entity-graph")
    public ResponseEntity<?> demoFix() {
        List<Task> tasks = taskRepository.findAllWithNguoiTao();
        // ... map giống hệt, nhưng .getNguoiTao().getUsername() KHÔNG tạo thêm query
        return ResponseEntity.ok(result);
    }
}
```

**Vì sao `demoNPlus1()` cần `@Transactional` mà `demoFix()` thì không?** Đây là chi tiết
kỹ thuật quan trọng nhất, không nằm trong tài liệu lý thuyết gốc. `nguoiTao` khai báo
`FetchType.LAZY` — nghĩa là Hibernate chỉ thực sự chạy câu SQL lấy `User` **tại đúng thời
điểm** code gọi `task.getNguoiTao().getUsername()`. Việc này đòi hỏi **session Hibernate
(persistence context) vẫn còn mở** tại thời điểm đó. Nếu không có `@Transactional` bao
quanh, session có thể đã đóng ngay sau khi `taskRepository.findAllForNPlusOne()` trả về
(kết thúc method repository) — gọi `getNguoiTao()` lúc đó sẽ ném
`LazyInitializationException` thay vì tạo thêm 1 câu query như mong đợi. `@Transactional`
giữ session mở xuyên suốt cả method controller, cho phép lazy loading "kịp" chạy đúng lúc
cần. Ngược lại, `demoFix()` không cần `@Transactional` vì `@EntityGraph` đã lấy sẵn
`nguoiTao` **cùng lúc** với `Task` trong 1 câu query — không còn thao tác lazy loading nào
xảy ra sau đó nữa.

## 4. Cách quan sát: đếm số dòng SELECT trong log

Bật `show-sql`/`format_sql`/`logging.level.org.hibernate.SQL=DEBUG` (đã học Tuần 1), gọi
lần lượt 2 endpoint và so sánh:

- `GET /api/demo/n-plus-1` → log hiện **1 dòng `select ... from tasks`**, sau đó
  **thêm 3 dòng `select ... from users where id = ?`** riêng lẻ (mỗi Task 1 dòng) —
  tổng cộng **4 câu query**.
- `GET /api/demo/fix-entity-graph` → log chỉ hiện **đúng 1 dòng** `select t.*, u.* from
  tasks t left join users u on t.nguoi_tao_id = u.id` — toàn bộ dữ liệu lấy trong 1 lần.

## 5. `@EntityGraph` vs `JOIN FETCH` viết tay — khi nào dùng cái nào

| | `JOIN FETCH` (viết JPQL tay) | `@EntityGraph` (khai báo qua annotation) |
|---|---|---|
| Cú pháp | Phải viết câu JPQL đầy đủ | Chỉ cần liệt kê tên field quan hệ cần lấy kèm |
| Phù hợp khi | Query đã tùy biến sẵn (`WHERE`, `ORDER BY` phức tạp) | Chỉ cần thêm "lấy kèm quan hệ X" cho method có sẵn |
| Lấy nhiều quan hệ cùng lúc | Viết nhiều `JOIN FETCH` nối tiếp trong JPQL | Liệt kê nhiều field trong `attributePaths = {"nguoiTao", "category"}` |

Code thật kết hợp cả 2 trong cùng 1 method — `@EntityGraph` áp lên trên 1 `@Query` đã có
JPQL riêng (`findAllWithNguoiTao`), cho thấy 2 cách có thể **dùng chung**, không nhất thiết
chọn 1 trong 2.

## 6. Lưu ý mở rộng: Cartesian product khi có nhiều quan hệ `@OneToMany`

`JOIN FETCH`/`@EntityGraph` cho nhiều quan hệ **dạng danh sách** cùng lúc (không phải
`@ManyToOne` đơn giản như hôm nay) có thể gây "Cartesian product" — nhân bản dòng dữ liệu
(VD 1 Task có 3 Comment và 2 Tag sẽ nhân thành 6 dòng trùng lặp Task). Phạm vi hôm nay tập
trung vào trường hợp cơ bản (`@ManyToOne`); vấn đề Cartesian product là kiến thức mở rộng
nên biết, không bắt buộc xử lý ngay.

## Bài tập

**Bài 7 (đã có sẵn — `NPlusOneController.demoNPlus1()`):** Bật `show-sql`, gọi
`GET /api/demo/n-plus-1`, đếm số dòng `select` trong log — xác nhận đúng **4 câu query**
(1 cho Task + 3 cho từng User riêng lẻ).

**Bài 8 (đã có sẵn — `NPlusOneController.demoFix()`):** Gọi `GET /api/demo/fix-entity-graph`,
đếm lại số dòng `select` — xác nhận giảm còn **đúng 1 câu**, kết quả trả về giống hệt Bài 7.

**Bài 9 (nâng cao, không bắt buộc):** Nếu muốn tự trải nghiệm Cartesian product, thử thêm
1 quan hệ `@OneToMany` mới (VD danh sách comment cho Task), rồi dùng `@EntityGraph` lấy
kèm cả `nguoiTao` (`@ManyToOne`) lẫn quan hệ `@OneToMany` mới cùng lúc — quan sát số dòng
dữ liệu trả về có bị nhân bản bất thường không.

## Code Example (repo `java.git`)
> Xem nguyên các file tại branch `N+1-Query`:
> [`Task.java`](<https://github.com/SangLeSoftZ/java/blob/N+1-Query/src/main/java/com/example/demo/entity/Task.java>) ·
> [`TaskRepository.java`](<https://github.com/SangLeSoftZ/java/blob/N+1-Query/src/main/java/com/example/demo/repository/TaskRepository.java>) ·
> [`NPlusOneController.java`](<https://github.com/SangLeSoftZ/java/blob/N+1-Query/src/main/java/com/example/demo/controller/NPlusOneController.java>) ·
> [`DataInitializer.java`](<https://github.com/SangLeSoftZ/java/blob/N+1-Query/src/main/java/com/example/demo/config/DataInitializer.java>)

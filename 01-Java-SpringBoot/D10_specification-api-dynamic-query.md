---
title: "Ngày 10 / Tuần 2 - Ngày 3 (Tối, 30%) — Spring Boot: Specification API"
track: java-spring
day: 10
topic: specification-api-dynamic-query
tags: [spring-boot, jpa, specification, criteria-api, dynamic-query]
source_goc: Tuan2-Ngay3-Animation-Hero-SpecificationAPI.md (buổi tối)
---

# Ngày 10 / Tuần 2 - Ngày 3 (Tối, 30%) — Spring Boot: Specification API

# BUỔI TỐI (30%) — Specification API cho Task

## 1. Vấn đề thực tế: tìm kiếm theo NHIỀU điều kiện, có thể có hoặc không

Cách viết Repository thông thường (`findByTieuDeContaining`, `findByHoanThanh`...) chỉ xử lý được **1 kiểu điều kiện cố định**. Thực tế cần lọc **kết hợp linh hoạt**: có lúc chỉ tìm theo từ khóa, có lúc chỉ lọc trạng thái, có lúc cả 2, có lúc không lọc gì — nếu viết method riêng cho từng tổ hợp thì số lượng method **bùng nổ theo cấp số nhân** khi thêm điều kiện mới.

**Giải pháp:** `Specification` — viết điều kiện `WHERE` linh hoạt bằng code Java, **kết hợp động** tùy điều kiện nào thực sự có giá trị lúc runtime.

## 2. Chuẩn bị: Repository kế thừa thêm `JpaSpecificationExecutor`

```java
public interface TaskRepository extends JpaRepository<Task, Long>,
        JpaSpecificationExecutor<Task> {
}
```

**Vì sao cần thêm?** `JpaRepository` mặc định chỉ hỗ trợ query cố định — không có cách truyền "điều kiện linh hoạt lúc runtime". `JpaSpecificationExecutor` bổ sung method nhận vào `Specification<T>` — "khe cắm" để đưa điều kiện động vào.

## 3. Viết `Specification` cho từng điều kiện riêng lẻ

```java
public class TaskSpecification {

    public static Specification<Task> tieuDeChua(String tuKhoa) {
        return (root, query, cb) -> {
            if (tuKhoa == null || tuKhoa.isBlank()) {
                return cb.conjunction(); // dieu kien "luon dung", tuong duong khong loc
            }
            return cb.like(cb.lower(root.get("tieuDe")), "%" + tuKhoa.toLowerCase() + "%");
        };
    }

    public static Specification<Task> hoanThanhBang(Boolean hoanThanh) {
        return (root, query, cb) -> {
            if (hoanThanh == null) {
                return cb.conjunction();
            }
            return cb.equal(root.get("hoanThanh"), hoanThanh);
        };
    }
}
```

**Giải thích 3 tham số của lambda:** `root` — đại diện bảng đang truy vấn, dùng `root.get("tenField")` để trỏ cột. `query` — đối tượng query tổng thể (hữu ích cho `distinct()`, `orderBy`). `cb` (`CriteriaBuilder`) — "hộp công cụ" tạo điều kiện: `equal`, `like`, `greaterThan`, `and`, `or`...

**`cb.conjunction()` là gì?** Điều kiện **luôn đúng** (tương đương `WHERE 1=1`) — dùng làm "điều kiện rỗng" khi không truyền giá trị lọc đó. Kỹ thuật giúp API lọc hoặc không lọc từng điều kiện độc lập.

## 4. Kết hợp nhiều Specification lại

```java
@Service
public class TaskService {
    private final TaskRepository taskRepository;

    public List<Task> timKiem(String tuKhoa, Boolean hoanThanh) {
        Specification<Task> spec = Specification
                .where(TaskSpecification.tieuDeChua(tuKhoa))
                .and(TaskSpecification.hoanThanhBang(hoanThanh));

        return taskRepository.findAll(spec);
    }
}
```

**Vì sao gọi là "kết hợp động"?** `.and(...)` chỉ ghép SQL `WHERE` bằng `AND` — nhờ `cb.conjunction()`, nếu 1 điều kiện không truyền, nó tự "biến mất" khỏi câu SQL cuối (vì `AND 1=1` không ảnh hưởng kết quả). Cùng 1 dòng code xử lý được cả 4 tình huống mà không cần `if-else` rẽ nhánh.

## 5. Controller — expose endpoint tìm kiếm

```java
@RestController
@RequestMapping("/api/tasks")
public class TaskController {
    private final TaskService taskService;

    @GetMapping("/search")
    public List<Task> timKiemTask(
            @RequestParam(required = false) String tuKhoa,
            @RequestParam(required = false) Boolean hoanThanh) {
        return taskService.timKiem(tuKhoa, hoanThanh);
    }
}
```

Test bằng Postman:
- `GET /api/tasks/search` → tất cả Task
- `GET /api/tasks/search?tuKhoa=hoc` → lọc theo tiêu đề
- `GET /api/tasks/search?hoanThanh=true` → lọc trạng thái
- `GET /api/tasks/search?tuKhoa=hoc&hoanThanh=false` → kết hợp cả 2

## 6. Khi nào dùng Specification, khi nào dùng derived method thường

| | Derived method | Specification |
|---|---|---|
| Số điều kiện cố định | ✅ Phù hợp, ngắn gọn | Hơi "thừa công cụ" nếu chỉ 1 điều kiện |
| Điều kiện có/không, kết hợp tùy ý | ❌ Bùng nổ số method | ✅ Đúng mục đích thiết kế |
| Query rất phức tạp (join, subquery) | ❌ Khó/không viết được | ✅ Linh hoạt hơn `@Query` tĩnh |

## Bài tập

**Bài 5:** Cho `TaskRepository` kế thừa thêm `JpaSpecificationExecutor<Task>`. Viết `TaskSpecification` với 2 method `tieuDeChua` và `hoanThanhBang`.

**Bài 6:** Viết `TaskService.timKiem()` kết hợp 2 Specification bằng `.and()`. Expose qua `GET /api/tasks/search`, test đủ 4 trường hợp bằng Postman.

**Bài 7 (nâng cao):** Thêm Specification thứ 3 lọc theo khoảng ngày tạo (`cb.between(...)`), ghép tiếp vào `.and()` — kiểm chứng việc mở rộng không cần sửa lại 2 Specification cũ.

---

##  Code Example (repo `Code`)

> Nguồn: [`SangLeSoftZ/java`](https://github.com/SangLeSoftZ/
java)
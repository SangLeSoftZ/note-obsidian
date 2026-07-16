---
title: "Ngày 10 / Tuần 2 - Ngày 3 — Đáp án gợi ý & Checklist hoàn thành"
track: chung
day: 10
topic: dap-an-checklist
tags: [dap-an, checklist]
source_goc: Tuan2-Ngay3-Animation-Hero-SpecificationAPI.md
---

# Ngày 10 / Tuần 2 - Ngày 3 — Đáp án gợi ý & Checklist hoàn thành

## ĐÁP ÁN GỢI Ý

<details><summary>Bài 1 — Nút bấm AnimatedContainer (xem D10_animation-implicit-basics.md)</summary>

```dart
AnimatedContainer(
  duration: const Duration(milliseconds: 300),
  curve: Curves.easeInOut,
  width: _phongTo ? 200 : 100,
  height: _phongTo ? 200 : 100,
  decoration: BoxDecoration(
    color: _phongTo ? Colors.blue : Colors.red,
    borderRadius: BorderRadius.circular(_phongTo ? 40 : 8),
  ),
)
```
</details>

<details><summary>Bài 3 — Hero cho danh sách Task (xem D10_hero-animation-transition.md)</summary>

```dart
// Danh sach
Hero(
  tag: 'task-icon-${task.id}',
  child: CircleAvatar(/* ... */),
)

// Chi tiet - CUNG tag
Hero(
  tag: 'task-icon-${task.id}',
  child: CircleAvatar(radius: 60 /* ... */),
)
```
</details>

<details><summary>Bài 5-6 — Specification kết hợp (xem D10_specification-api-dynamic-query.md)</summary>

```java
public interface TaskRepository extends JpaRepository<Task, Long>,
        JpaSpecificationExecutor<Task> {}

public class TaskSpecification {
    public static Specification<Task> tieuDeChua(String tuKhoa) {
        return (root, query, cb) -> tuKhoa == null || tuKhoa.isBlank()
                ? cb.conjunction()
                : cb.like(cb.lower(root.get("tieuDe")), "%" + tuKhoa.toLowerCase() + "%");
    }
    public static Specification<Task> hoanThanhBang(Boolean hoanThanh) {
        return (root, query, cb) -> hoanThanh == null
                ? cb.conjunction()
                : cb.equal(root.get("hoanThanh"), hoanThanh);
    }
}

// Service
public List<Task> timKiem(String tuKhoa, Boolean hoanThanh) {
    Specification<Task> spec = Specification
            .where(TaskSpecification.tieuDeChua(tuKhoa))
            .and(TaskSpecification.hoanThanhBang(hoanThanh));
    return taskRepository.findAll(spec);
}
```
</details>

---

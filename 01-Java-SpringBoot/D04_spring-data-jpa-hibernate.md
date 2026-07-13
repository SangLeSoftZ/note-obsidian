---
title: "Ngày 4 — Spring Data JPA & Hibernate: Bên dưới lớp vỏ JpaRepository"
track: java-spring
day: 4
topic: jpa-hibernate
tags: [jpa, hibernate, orm, lazy-loading, transactional]
source_goc: Ngay4_LyThuyetSau.md (Phần 1)
---

# Ngày 4 — Spring Data JPA & Hibernate: Bên dưới lớp vỏ JpaRepository

# PHẦN 1 — JPA & HIBERNATE: BÊN DƯỚI LỚP VỎ `JpaRepository`

### JPA là gì, Hibernate là gì — 2 cái khác nhau như thế nào?

Đây là điểm gây nhầm lẫn nhiều nhất khi mới học: **JPA là 1 đặc tả (specification)** — tức là "1 bộ quy tắc/interface chuẩn" quy định ORM (Object-Relational Mapping) nên hoạt động thế nào, nhưng JPA **không tự chạy được**. **Hibernate là 1 implementation cụ thể** thực hiện đúng bộ quy tắc đó (giống như `interface` và `class implements` trong Java — JPA là interface, Hibernate là class hiện thực hóa nó).

Khi bạn dùng Spring Boot với `spring-boot-starter-data-jpa`, mặc định Spring **chọn Hibernate** làm implementation chạy phía sau — đây là lý do bạn thấy log lúc chạy app có nhắc tới "Hibernate", dù code bạn viết chỉ dùng annotation `@Entity`, `@Id` (là annotation của JPA, không phải của Hibernate).

### `JpaRepository.save()` thực sự làm gì bên trong?

```java
repository.save(task);
```

Bên dưới, Hibernate phải quyết định: đây là **INSERT** (tạo mới) hay **UPDATE** (cập nhật)? Nó quyết định dựa trên: **object có `id` hay chưa?**

- Nếu `task.getId() == null` → Hibernate hiểu đây là bản ghi mới → chạy `INSERT INTO tasks...`
- Nếu `task.getId() != null` → Hibernate hiểu đây là bản ghi đã tồn tại → chạy `UPDATE tasks SET ... WHERE id = ?`

**Hệ quả thực tế hay gặp lỗi:** nếu bạn vô tình set `id` cho 1 object mà bạn nghĩ là "tạo mới" (ví dụ do gán nhầm từ DTO có field `id` không nên có), Hibernate sẽ hiểu nhầm thành **UPDATE** một bản ghi đã tồn tại — đây chính là 1 trong những lý do thực tế (ngoài lý do bảo mật) vì sao DTO tạo mới **không nên có field `id`**.

### Lazy Loading vs Eager Loading — vấn đề N+1 Query nổi tiếng

```java
@Entity
public class Category {
    @OneToMany(mappedBy = "category", fetch = FetchType.LAZY)  // mặc định của @OneToMany là LAZY
    private List<Product> danhSachSanPham;
}
```

- **LAZY**: Khi lấy `Category` ra, Hibernate **chưa** lấy `danhSachSanPham` ngay — chỉ khi bạn thực sự gọi `category.getDanhSachSanPham()` thì nó **mới** chạy thêm 1 câu SQL để lấy.
- **EAGER**: Lấy `Category` là lấy luôn `danhSachSanPham` ngay lập tức, dù bạn có dùng tới hay không.

**Vấn đề N+1 Query — lỗi hiệu năng kinh điển:**

```java
List<Category> danhSach = categoryRepository.findAll(); // 1 câu SQL lấy N category

for (Category c : danhSach) {
    System.out.println(c.getDanhSachSanPham().size()); // MỖI vòng lặp lại chạy thêm 1 câu SQL riêng!
}
// Tổng cộng: 1 (lấy category) + N (lấy sản phẩm từng category) = N+1 câu SQL
```

Nếu có 1000 category, đoạn code trên chạy **1001 câu SQL** thay vì có thể gộp lại chỉ 1-2 câu — đây là nguyên nhân phổ biến khiến app chạy chậm dần khi dữ liệu tăng lên mà lập trình viên mới không nhận ra vì lúc test với ít dữ liệu vẫn thấy "chạy nhanh bình thường".

**Cách khắc phục cơ bản (biết để nhận diện vấn đề, không cần thành thạo ngay):** dùng `JOIN FETCH` trong câu query tùy chỉnh để lấy cả 2 bảng trong 1 lần:
```java
@Query("SELECT c FROM Category c LEFT JOIN FETCH c.danhSachSanPham")
List<Category> findAllWithProducts();
```

### `@Transactional` — vì sao cần, dùng khi nào?

```java
@Service
public class DonHangService {

    @Transactional
    public void datHang(Long userId, List<Long> sanPhamIds) {
        Order order = new Order(userId);
        orderRepository.save(order);          // Bước 1

        for (Long id : sanPhamIds) {
            Product p = productRepository.findById(id).orElseThrow();
            p.setSoLuongTon(p.getSoLuongTon() - 1);
            productRepository.save(p);         // Bước 2 (nhiều lần)
        }
    }
}
```

**Vấn đề nếu không có `@Transactional`:** giả sử Bước 1 chạy xong (đơn hàng đã lưu), nhưng Bước 2 bị lỗi ở sản phẩm thứ 3 (ví dụ hết hàng) → bạn sẽ có 1 đơn hàng đã lưu nhưng dữ liệu tồn kho bị cập nhật **nửa vời** (2/5 sản phẩm đã trừ kho, 3 sản phẩm còn lại chưa) → dữ liệu không nhất quán.

`@Transactional` đảm bảo: **hoặc tất cả các bước bên trong đều thành công, hoặc nếu có bất kỳ lỗi nào xảy ra, mọi thay đổi được "hoàn tác" (rollback) về trạng thái ban đầu** — giống như "tất cả hoặc không có gì" (all-or-nothing). Đây là khái niệm **ACID transaction** trong database mà gần như mọi hệ thống thực tế xử lý tiền/đơn hàng đều cần.

### Bài tập đào sâu

**Bài A:** Tạo tình huống N+1 query thật: có `Category` với vài `Product`, viết đoạn code lặp qua danh sách category và gọi `getDanhSachSanPham()`, bật `show-sql: true` trong `application.yml`, đếm xem có bao nhiêu câu SQL thực sự được in ra console.

**Bài B:** Thử bỏ `@Transactional` khỏi 1 method có nhiều bước lưu dữ liệu, cố tình cho bước giữa bị lỗi (throw exception), kiểm tra xem dữ liệu trước đó có bị lưu "nửa vời" không. Sau đó thêm lại `@Transactional`, lặp lại, so sánh kết quả.

---

## 💻 Code Example (repo `Code`)

> Nguồn: [`SangLeSoftZ/Code`](https://github.com/SangLeSoftZ/Code)

**`JpaRepository` thực tế** — [`baiA/repository/BaiVietARepository.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/baiA/repository/BaiVietARepository.java):

```java
package com.example.demo.baiA.repository;

import com.example.demo.baiA.entity.BaiVietA;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface BaiVietARepository extends JpaRepository<BaiVietA, Long> {

    // Spring Data JPA tự sinh SQL: SELECT * FROM bai_viet WHERE loai = ?
    List<BaiVietA> findByLoai(String loai);
}
```

Entity tương ứng (annotation `@Entity`, `@Id`, `@GeneratedValue`) xem tại [`baiA/entity/BaiVietA.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/baiA/entity/BaiVietA.java).

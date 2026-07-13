---
title: "Ngày 2 (Chiều) — Spring Core: IoC, DI, Bean, application.yml"
track: java-spring
day: 2
topic: spring-core
tags: [spring, ioc, di, bean, application-yml]
source_goc: Ngay2_OnTapChiTiet.md (buổi chiều)
---

# Ngày 2 (Chiều) — Spring Core: IoC, DI, Bean, application.yml

# BUỔI CHIỀU — SPRING CORE (IoC, DI, Bean, Annotation, application.yml)

## 1. IoC Container hoạt động thế nào?

Khi Spring Boot khởi động, nó quét (scan) toàn bộ code, tìm các class có đánh dấu `@Component`/`@Service`/`@Repository`/`@Controller`, tạo object cho từng class đó (gọi là **Bean**), rồi lưu vào 1 "cái hộp" gọi là **IoC Container**. Khi 1 class cần dùng class khác, Spring lấy Bean có sẵn trong hộp đó ra đưa cho — đây gọi là **Dependency Injection**.

```
[Spring Boot khởi động]
        │
        ▼
[Quét code tìm @Component/@Service/@Repository/@Controller]
        │
        ▼
[Tạo Bean cho từng class, lưu vào IoC Container]
        │
        ▼
[Khi Class A cần Class B → Spring inject Bean B có sẵn vào Class A]
```

## 2. 3 cách Dependency Injection

```java
// Cách 1: Constructor Injection (KHUYÊN DÙNG — dễ test, đảm bảo không null)
@Service
public class OrderService {
    private final PaymentService paymentService;
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}

// Cách 2: Field Injection (dễ viết nhưng khó test, không nên dùng cho project thật)
@Service
public class OrderService {
    @Autowired
    private PaymentService paymentService;
}

// Cách 3: Setter Injection (ít dùng, phù hợp dependency optional)
@Service
public class OrderService {
    private PaymentService paymentService;
    @Autowired
    public void setPaymentService(PaymentService p) { this.paymentService = p; }
}
```

## 3. Bean Scope (phạm vi sống của Bean)

| Scope | Ý nghĩa |
|---|---|
| `singleton` (mặc định) | Cả app chỉ có **1** instance duy nhất, dùng chung |
| `prototype` | Mỗi lần cần là tạo **1 instance mới** |

```java
@Service
@Scope("prototype")
public class ReportGenerator { ... }
```

## 4. application.yml chi tiết hơn

```yaml
server:
  port: 8080

spring:
  application:
    name: demo
  datasource:
    url: jdbc:mysql://localhost:3306/demo_db
    username: root
    password: 123456
  jpa:
    hibernate:
      ddl-auto: update      # tự tạo/cập nhật bảng theo Entity (chỉ nên dùng khi dev)
    show-sql: true          # in ra câu SQL thực thi (hữu ích để debug)

# Có thể tách môi trường dev/prod bằng profile:
---
spring:
  config:
    activate:
      on-profile: prod
  jpa:
    hibernate:
      ddl-auto: validate    # ở production KHÔNG tự sửa bảng, chỉ kiểm tra khớp
```

`ddl-auto: update` rất tiện khi học/dev vì Spring tự tạo bảng theo Entity, nhưng **không dùng ở production** vì có thể làm mất dữ liệu khi cấu trúc bảng thay đổi.

## Bài tập buổi chiều

**Bài 4:** Tạo interface `KhoService` với method `kiemTraTonKho(String maSanPham)`. Viết class `KhoServiceImpl implements KhoService` (giả lập trả về `true`). Inject `KhoService` vào `DonHangService` bằng **Constructor Injection**, viết method `datHang(String maSanPham)`: nếu còn hàng thì in `"Đặt hàng thành công"`, ngược lại in `"Hết hàng"`.

**Bài 5:** Viết 1 file `application.yml` cho project có: port chạy ở `9090`, kết nối tới database PostgreSQL tên `shop_db`, bật `show-sql`.

---

## 💻 Code Example (repo `Code`)

> Nguồn: [`SangLeSoftZ/Code`](https://github.com/SangLeSoftZ/Code)

**Spring Bean qua `@Service` (IoC/DI thực tế)** — [`bai4/KhoService.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/bai4/KhoService.java) + [`KhoServiceImpl.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/bai4/KhoServiceImpl.java):

```java
package com.example.demo.bai4;

public interface KhoService {

    /**
     * Kiểm tra xem sản phẩm còn tồn kho không.
     *
     * @param maSanPham mã sản phẩm cần kiểm tra
     * @return true nếu còn hàng, false nếu hết hàng
     */
    boolean kiemTraTonKho(String maSanPham);
}
```

```java
package com.example.demo.bai4;

import org.springframework.stereotype.Service;

// @Service đánh dấu class này là một Spring Bean
// Spring IoC Container sẽ tự động tạo và quản lý object này
@Service
public class KhoServiceImpl implements KhoService {

    @Override
    public boolean kiemTraTonKho(String maSanPham) {
        // Giả lập: thực tế sẽ truy vấn database
        System.out.println("  [KhoService] Kiểm tra tồn kho cho sản phẩm: " + maSanPham);
        return true;
    }
}
```

Xem thêm `DonHangService.java` cùng thư mục để thấy constructor injection dùng `KhoService`.

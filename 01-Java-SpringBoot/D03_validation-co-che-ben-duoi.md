---
title: "Ngày 3 — Validation: Cơ chế thực sự bên dưới (@NotBlank, @Valid...)"
track: java-spring
day: 3
topic: validation
tags: [spring, validation, bean-validation, jsr-380]
source_goc: Ngay3_LyThuyetSau.md (Phần 1)
---

# Ngày 3 — Validation: Cơ chế thực sự bên dưới (@NotBlank, @Valid...)

# PHẦN 1 — VALIDATION: CƠ CHẾ THỰC SỰ BÊN DƯỚI

### Câu hỏi cốt lõi: `@NotBlank` chỉ là 1 cái "nhãn dán" — ai là người thực sự đọc nhãn đó và làm gì với nó?

Khi bạn viết:
```java
@PostMapping
public Task taoMoi(@Valid @RequestBody TaskDTO dto) { ... }
```

Chuyện thực sự xảy ra theo thứ tự:

1. Request HTTP đến → Spring's `DispatcherServlet` nhận request, xác định method `taoMoi` sẽ xử lý
2. Trước khi **gọi** method `taoMoi`, Spring thấy tham số có `@RequestBody` → nó chuyển JSON thành object `TaskDTO` (bước này gọi là **deserialize**, dùng thư viện Jackson)
3. Spring thấy tiếp có `@Valid` đứng trước `@RequestBody` → nó gọi tới 1 bean có sẵn tên `LocalValidatorFactoryBean` (dựa trên chuẩn **Bean Validation / JSR-380**), bean này dùng **Reflection** để quét toàn bộ field của `TaskDTO`, tìm annotation nào bắt đầu bằng validation (`@NotBlank`, `@Size`...), rồi kiểm tra giá trị thực tế so với ràng buộc
4. Nếu có field vi phạm → Spring **ném ra** `MethodArgumentNotValidException` **trước khi** method `taoMoi()` được gọi (method của bạn hoàn toàn không chạy)
5. Exception này bay lên tới `DispatcherServlet`, và nếu bạn có `@RestControllerAdvice` bắt đúng loại exception này → trả về response bạn tự định nghĩa; nếu không có → Spring dùng handler mặc định, trả JSON lỗi 400 kèm cấu trúc mặc định (khó đọc hơn)

**Điểm mấu chốt hay bị hiểu nhầm:** `@Valid` không "tự chạy code kiểm tra" — nó chỉ là **tín hiệu** để Spring biết "hãy gọi bộ máy Validator có sẵn cho tham số này". Nếu bạn quên `@Valid`, các annotation như `@NotBlank` vẫn nằm đó nhưng **không có tác dụng gì cả** — đây là lỗi rất hay gặp: viết đủ annotation nhưng quên `@Valid`, rồi thắc mắc "sao validation không hoạt động?".

### Validate lồng nhau (nested object)

```java
public class OrderDTO {
    @NotBlank
    private String tenKhachHang;

    @Valid  // <-- BẮT BUỘC phải có thêm @Valid ở đây để Spring "đi sâu" vào validate object con
    private DiaChiDTO diaChiGiaoHang;
}

public class DiaChiDTO {
    @NotBlank
    private String duong;
}
```

Nếu quên `@Valid` ở field `diaChiGiaoHang`, Spring **chỉ validate `OrderDTO` ở tầng ngoài cùng**, không tự động "lặn" xuống kiểm tra bên trong `DiaChiDTO` — dù `DiaChiDTO` có khai báo `@NotBlank` cũng vô nghĩa nếu thiếu `@Valid` lồng.

### Validation ở tầng Java vs Validation ở tầng Database — khác nhau và cần cả 2

```java
@Entity
public class Task {
    @Column(nullable = false, length = 100)  // ràng buộc TẦNG DATABASE
    private String tieuDe;
}

public class TaskDTO {
    @NotBlank  // ràng buộc TẦNG APPLICATION (Java)
    private String tieuDe;
}
```

**Vì sao cần cả 2, không phải chỉ 1 cái là đủ?**
- `@NotBlank` (tầng Java) phản hồi lỗi **ngay lập tức, thân thiện** cho client trước khi chạm database — trải nghiệm tốt hơn nhiều
- `@Column(nullable=false)` (tầng database) là **lớp phòng thủ cuối cùng** — phòng trường hợp có đoạn code khác (script, batch job, hay chính bạn quên `@Valid` ở đâu đó) cố tình ghi dữ liệu sai mà không đi qua Controller. Database luôn là chốt chặn cuối, không tin tưởng 100% vào tầng ứng dụng.

### Bài tập đào sâu

**Bài A:** Thử cố tình **xóa** `@Valid` khỏi Controller (giữ nguyên `@NotBlank` trong DTO), gửi `tieuDe: ""` qua Postman. Quan sát: request có bị chặn không? Vì sao?

**Bài B:** Viết 1 Custom Validator đơn giản: annotation `@KhongChuaTuCam` áp cho field `String`, không cho phép chứa từ "spam". (Gợi ý tìm hiểu: `ConstraintValidator` interface — đây là cách tự mở rộng validation khi các annotation có sẵn không đủ dùng).

---

## 💻 Code Example (repo `Code`)

> Nguồn: [`SangLeSoftZ/Code`](https://github.com/SangLeSoftZ/Code)

**Custom Validation Annotation** — [`baiB/validator/KhongChuaTuCam.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/baiB/validator/KhongChuaTuCam.java) + [`KhongChuaTuCamValidator.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/baiB/validator/KhongChuaTuCamValidator.java):

```java
package com.example.demo.baiB.validator;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;
import java.lang.annotation.*;

@Constraint(validatedBy = KhongChuaTuCamValidator.class)
@Target({ ElementType.FIELD })
@Retention(RetentionPolicy.RUNTIME)
public @interface KhongChuaTuCam {
    String message() default "Nội dung chứa từ bị cấm";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

```java
package com.example.demo.baiB.validator;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;

public class KhongChuaTuCamValidator implements ConstraintValidator<KhongChuaTuCam, String> {

    private static final String TU_CAM = "spam";

    @Override
    public void initialize(KhongChuaTuCam annotation) {}

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null) return true;
        return !value.toLowerCase().contains(TU_CAM);
    }
}
```

Áp dụng thực tế trong DTO xem tại [`baiB/dto/BaiVietBDTO.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/baiB/dto/BaiVietBDTO.java).

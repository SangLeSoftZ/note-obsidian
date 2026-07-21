---
title: "Ngày 15 (Tuần 3 - Ngày 4, Tối) — Custom Validator Annotation nâng cao"
track: java-spring
day: 15
topic: custom-validator-annotation
tags: [spring-boot, bean-validation, constraintvalidator, custom-annotation, cross-field-validation]
source_goc: Tuan3_Ngay4_TaiLieu.md
---

# Ngày 15 (Tuần 3 - Ngày 4, Tối) — Custom Validator Annotation nâng cao

## 1. Vấn đề: annotation có sẵn không đủ cho logic phức tạp

Đã dùng `@NotBlank`, `@Size`, `@Min`/`@Max` (Tuần 1) — đủ cho validation đơn giản. Với
logic phức tạp hơn (mật khẩu cần 1 chữ hoa, 1 số, 1 ký tự đặc biệt), không annotation có
sẵn nào làm được — cần tự tạo annotation riêng.

## 2. `@ValidPassword` — Phần 1: định nghĩa annotation

Code thật — `validator/ValidPassword.java`:

```java
/**
 * Kiểm tra password đủ mạnh:
 *   - Ít nhất 8 ký tự
 *   - Ít nhất 1 chữ hoa
 *   - Ít nhất 1 chữ số
 *   - Ít nhất 1 ký tự đặc biệt (@#$%^&+=!)
 */
@Target({ ElementType.FIELD, ElementType.PARAMETER })
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = ValidPasswordValidator.class)
@Documented
public @interface ValidPassword {
    String message() default "Mật khẩu phải có ít nhất 8 ký tự, 1 chữ hoa, 1 chữ số và 1 ký tự đặc biệt (@#$%^&+=!)";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

- `@Target`: annotation chỉ dùng trên field/parameter.
- `@Retention(RUNTIME)`: annotation tồn tại lúc chạy — bắt buộc để Spring đọc được.
- `@Constraint(validatedBy = ...)`: chỉ định class chứa logic kiểm tra thật.
- `message()/groups()/payload()`: 3 thuộc tính bắt buộc theo chuẩn Bean Validation
  (JSR 380, đã học Tuần 1).
- `@Documented`: chi tiết nhỏ không có trong ví dụ lý thuyết gốc — giúp annotation này
  xuất hiện trong Javadoc sinh ra tự động, không ảnh hưởng logic runtime.

## 3. Phần 2 — Class xử lý logic, đã làm luôn cả Bài 8 (mở rộng)

Code thật — `validator/ValidPasswordValidator.java` — không dừng ở việc trả `true`/`false`
đơn giản như ví dụ lý thuyết gốc, mà đã cài đặt luôn **message linh động** (Bài 8):

```java
public class ValidPasswordValidator implements ConstraintValidator<ValidPassword, String> {

    private static final String PATTERN =
            "^(?=.*[0-9])(?=.*[a-z])(?=.*[A-Z])(?=.*[@#$%^&+=!]).{8,}$";

    @Override
    public boolean isValid(String password, ConstraintValidatorContext context) {
        if (password == null) return false;

        boolean valid = password.matches(PATTERN);

        // Bài 8: message linh động — chỉ rõ thiếu điều kiện nào
        if (!valid) {
            context.disableDefaultConstraintViolation();

            StringBuilder msg = new StringBuilder("Mật khẩu chưa đủ mạnh:");
            if (password.length() < 8)
                msg.append(" thiếu độ dài (min 8 ký tự);");
            if (!password.matches(".*[A-Z].*"))
                msg.append(" thiếu chữ hoa;");
            if (!password.matches(".*[0-9].*"))
                msg.append(" thiếu chữ số;");
            if (!password.matches(".*[@#$%^&+=!].*"))
                msg.append(" thiếu ký tự đặc biệt (@#$%^&+=!);");

            context.buildConstraintViolationWithTemplate(msg.toString())
                   .addConstraintViolation();
        }

        return valid;
    }
}
```

**`ConstraintValidator<ValidPassword, String>`:** interface generic (cùng tư duy Generics
đã học sáng nay — Tuần 3 Ngày 3) — tham số 1 là annotation áp dụng, tham số 2 là kiểu dữ
liệu kiểm tra.

**Cơ chế `disableDefaultConstraintViolation()` + `buildConstraintViolationWithTemplate()`:**
mặc định, nếu `isValid()` trả `false`, Spring tự dùng `message()` khai báo trong
annotation (1 câu chung chung "Mật khẩu phải có ít nhất 8 ký tự..."). Gọi
`disableDefaultConstraintViolation()` **tắt** thông báo mặc định này đi, rồi
`buildConstraintViolationWithTemplate(msg)` tạo 1 thông báo **mới, cụ thể hơn** — ở đây
là liệt kê chính xác **thiếu điều kiện nào** (thiếu độ dài, thiếu chữ hoa, thiếu chữ
số...) thay vì chỉ nói chung chung "chưa đủ mạnh". Đây là trải nghiệm tốt hơn nhiều cho
người dùng cuối — họ biết chính xác cần sửa gì, không phải đoán.

## 4. `@PasswordMatches` — kiểm tra chéo 2 field, đặt ở cấp CLASS

Code thật — `validator/PasswordMatches.java`:

```java
/**
 * Tại sao đặt ở cấp TYPE thay vì FIELD?
 * Vì cần truy cập cả 2 field password + confirmPassword cùng lúc —
 * annotation cấp field chỉ thấy được 1 field tại 1 thời điểm.
 */
@Target({ ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PasswordMatchesValidator.class)
@Documented
public @interface PasswordMatches {
    String message() default "Mật khẩu xác nhận không khớp";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

Validator — `validator/PasswordMatchesValidator.java`, có 1 chi tiết thực tế quan trọng
không nằm trong ví dụ lý thuyết gốc:

```java
public class PasswordMatchesValidator
        implements ConstraintValidator<PasswordMatches, RegisterRequest> {

    @Override
    public boolean isValid(RegisterRequest dto, ConstraintValidatorContext context) {
        if (dto.getPassword() == null || dto.getConfirmPassword() == null) return false;

        boolean khop = dto.getPassword().equals(dto.getConfirmPassword());

        if (!khop) {
            // Gắn lỗi vào đúng field confirmPassword — Flutter biết field nào sai
            context.disableDefaultConstraintViolation();
            context.buildConstraintViolationWithTemplate("Mật khẩu xác nhận không khớp")
                   .addPropertyNode("confirmPassword")
                   .addConstraintViolation();
        }
        return khop;
    }
}
```

**`addPropertyNode("confirmPassword")` là chi tiết dễ bỏ sót nhưng rất quan trọng:**
annotation `@PasswordMatches` đặt ở **cấp class** (`RegisterRequest`), nên nếu không có
dòng này, lỗi trả về mặc định sẽ gắn vào **toàn bộ object**, không rõ field nào sai —
phía Flutter khó biết nên tô đỏ ô nhập nào. `addPropertyNode("confirmPassword")` "chuyển
hướng" lỗi này gắn cụ thể vào field `confirmPassword` trong response JSON lỗi, giúp
Flutter tô đúng đúng ô `confirmPassword` là sai, không phải hiển thị lỗi chung chung ở
đầu form.

## 5. Áp dụng vào `RegisterRequest` — annotation cấp field + cấp class cùng lúc

Code thật — `dto/RegisterRequest.java`:

```java
@PasswordMatches // đặt ở CẤP CLASS
public class RegisterRequest {

    @NotBlank(message = "Username không được để trống")
    @Size(min = 3, max = 50, message = "Username phải từ 3 đến 50 ký tự")
    private String username;

    @ValidPassword // đặt ở CẤP FIELD — dùng y hệt @NotBlank có sẵn
    private String password;

    @NotBlank(message = "Xác nhận mật khẩu không được để trống")
    private String confirmPassword;

    @Email(message = "Email không đúng định dạng")
    private String email;
    // getter/setter...
}
```

`@ValidPassword` hoạt động giống hệt annotation có sẵn — Controller
(`AuthController.register()`) vẫn chỉ cần `@Valid @RequestBody RegisterRequest request`
như đã quen, không cần code xử lý gì thêm. Cả 2 annotation tự viết (`@ValidPassword` cấp
field, `@PasswordMatches` cấp class) hoạt động song song, độc lập, mỗi cái lo đúng phần
việc của mình.

## Bài tập

**Bài 6 (đã có sẵn — `ValidPassword.java` + `ValidPasswordValidator.java`):** Gọi
`POST /api/auth/register` qua Postman với mật khẩu yếu (VD `"123"`) — xác nhận nhận đúng
message chi tiết liệt kê thiếu điều kiện nào (không phải message chung chung).

**Bài 7 (đã có sẵn — `PasswordMatches.java` + `PasswordMatchesValidator.java`):** Gửi
`password` và `confirmPassword` khác nhau — xác nhận bị từ chối, lỗi gắn đúng vào field
`confirmPassword`.

**Bài 8 (đã có sẵn trong `ValidPasswordValidator.isValid()`):** Đối chiếu code thật với
việc dùng `context.buildConstraintViolationWithTemplate(...)` thay vì chỉ dùng `message()`
mặc định — thử tự đổi 1 mật khẩu chỉ thiếu đúng 1 điều kiện (VD thiếu ký tự đặc biệt) để
xác nhận message chỉ liệt kê đúng điều kiện đang thiếu, không liệt kê thừa.

## Code Example (repo `java.git`)

> Xem nguyên các file tại branch `Custom-Validator-Annotation`:
> [`ValidPassword.java`](https://github.com/SangLeSoftZ/java/blob/Custom-Validator-Annotation/src/main/java/com/example/demo/validator/ValidPassword.java) ·
> [`ValidPasswordValidator.java`](https://github.com/SangLeSoftZ/java/blob/Custom-Validator-Annotation/src/main/java/com/example/demo/validator/ValidPasswordValidator.java) ·
> [`PasswordMatches.java`](https://github.com/SangLeSoftZ/java/blob/Custom-Validator-Annotation/src/main/java/com/example/demo/validator/PasswordMatches.java) ·
> [`PasswordMatchesValidator.java`](https://github.com/SangLeSoftZ/java/blob/Custom-Validator-Annotation/src/main/java/com/example/demo/validator/PasswordMatchesValidator.java) ·
> [`RegisterRequest.java`](https://github.com/SangLeSoftZ/java/blob/Custom-Validator-Annotation/src/main/java/com/example/demo/dto/RegisterRequest.java)


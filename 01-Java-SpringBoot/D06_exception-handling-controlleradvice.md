---
title: "Ngày 6 — Exception Handling tập trung (@RestControllerAdvice)"
track: java-spring
day: 6
topic: exception-handling
tags: [spring, exceptionhandler, controlleradvice, apierror]
source_goc: "Không có trong vault Obsidian gốc — bổ sung từ repo Code (package exception_handling)"
---

# Ngày 6 — Exception Handling tập trung (@RestControllerAdvice)

> File này lấp khoảng trống: kế hoạch gốc yêu cầu học "Exception Handling & Validation
> (@ControllerAdvice)" ở Ngày 6 nhưng vault Obsidian chưa có ghi chú lý thuyết. Nội dung
> dưới đây được viết lại dựa trên code thực tế trong repo `Code`.

## Vì sao cần Exception Handling tập trung?

Nếu mỗi Controller tự `try/catch` riêng, response lỗi sẽ không đồng nhất (chỗ trả JSON
này, chỗ trả JSON khác) và code bị lặp lại nhiều lần. `@RestControllerAdvice` cho phép
gom toàn bộ logic xử lý exception vào **một nơi duy nhất**, lắng nghe exception ném ra
từ **tất cả** Controller trong ứng dụng, rồi trả về một cấu trúc lỗi (`ApiError`) thống
nhất cho mọi API.

## 3 loại exception thường gặp

| Loại | Khi nào ném | HTTP Status |
|---|---|---|
| `ResourceNotFoundException` | Tìm 1 bản ghi theo ID nhưng không có trong DB | 404 Not Found |
| `BusinessException` | Vi phạm luật nghiệp vụ (vd: tiêu đề bị trùng) | 409 Conflict |
| `MethodArgumentNotValidException` | `@Valid` phát hiện field sai (`@NotBlank`, `@Size`...) | 400 Bad Request |
| (bất kỳ `Exception` nào khác) | Lỗi hệ thống không lường trước | 500 Internal Server Error |

## Cấu trúc response lỗi thống nhất — `ApiError`

Mọi lỗi trả về client đều theo đúng 1 format, giúp Flutter (hoặc bất kỳ client nào)
xử lý lỗi dễ dàng bằng 1 đoạn code chung, không cần biết trước lỗi gì:

```json
{
  "status": 404,
  "error": "Not Found",
  "message": "Task không tìm thấy với ID: 99",
  "path": "/api/tasks/99",
  "timestamp": "2026-07-13T10:00:00"
}
```

## 💻 Code Example (repo `Code`)

> Nguồn: [`SangLeSoftZ/Code`](https://github.com/SangLeSoftZ/Code) — package `exception_handling`

**Global handler — bắt exception từ mọi Controller** — [`exception_handling/exception/GlobalExceptionHandler.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/exception_handling/exception/GlobalExceptionHandler.java):

```java
package com.example.demo.exception_handling.exception;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.HashMap;
import java.util.Map;

// @RestControllerAdvice: lắng nghe exception từ TẤT CẢ controller
// Bắt exception → trả ApiError dạng JSON chuẩn
@RestControllerAdvice
public class GlobalExceptionHandler {

    // ===================== BÀI 4 =====================
    // Bắt ResourceNotFoundException → HTTP 404
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(
            ResourceNotFoundException ex,
            HttpServletRequest request) {

        ApiError error = new ApiError(
                404,
                "Not Found",
                ex.getMessage(),          // "Task không tìm thấy với ID: 99"
                request.getRequestURI()   // "/api/tasks/99"
        );

        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    // ===================== BÀI 5 =====================
    // Bắt BusinessException → HTTP 409 Conflict
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ApiError> handleBusiness(
            BusinessException ex,
            HttpServletRequest request) {

        ApiError error = new ApiError(
                409,
                "Conflict",
                ex.getMessage(),         // "Task với tiêu đề '...' đã tồn tại"
                request.getRequestURI()  // "/api/tasks"
        );

        return ResponseEntity.status(HttpStatus.CONFLICT).body(error);
    }

    // ===================== VALIDATION =====================
    // Bắt lỗi @Valid (@NotBlank, @Size...) → HTTP 400
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> handleValidation(
            MethodArgumentNotValidException ex,
            HttpServletRequest request) {

        // Gom tất cả field lỗi vào Map
        Map<String, String> fieldErrors = new HashMap<>();
        ex.getBindingResult()
          .getFieldErrors()
          .forEach(err -> fieldErrors.put(err.getField(), err.getDefaultMessage()));

        Map<String, Object> body = new HashMap<>();
        body.put("status",  400);
        body.put("error",   "Bad Request");
        body.put("message", "Dữ liệu không hợp lệ");
        body.put("path",    request.getRequestURI());
        body.put("fields",  fieldErrors); // chi tiết từng field lỗi

        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(body);
    }

    // ===================== FALLBACK =====================
    // Bắt tất cả exception không xử lý ở trên → HTTP 500
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handleGeneral(
            Exception ex,
            HttpServletRequest request) {

        ApiError error = new ApiError(
                500,
                "Internal Server Error",
                "Đã xảy ra lỗi hệ thống",
                request.getRequestURI()
        );

        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

**Custom exceptions & cấu trúc lỗi chuẩn** — [`ApiError.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/exception_handling/exception/ApiError.java), [`BusinessException.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/exception_handling/exception/BusinessException.java), [`ResourceNotFoundException.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/exception_handling/exception/ResourceNotFoundException.java):

```java
package com.example.demo.exception_handling.exception;

import java.time.LocalDateTime;

// Cấu trúc response lỗi chuẩn — trả về cho client khi có exception
// Mọi lỗi đều có cùng format này → client dễ xử lý
public class ApiError {

    private int status;           // HTTP status code: 404, 400, 500...
    private String error;         // tên lỗi: "Not Found", "Bad Request"...
    private String message;       // mô tả lỗi chi tiết
    private String path;          // URL bị lỗi: "/api/tasks/99"
    private LocalDateTime timestamp; // thời điểm xảy ra lỗi

    public ApiError(int status, String error, String message, String path) {
        this.status    = status;
        this.error     = error;
        this.message   = message;
        this.path      = path;
        this.timestamp = LocalDateTime.now();
    }

    // Getters
    public int getStatus()             { return status; }
    public String getError()           { return error; }
    public String getMessage()         { return message; }
    public String getPath()            { return path; }
    public LocalDateTime getTimestamp(){ return timestamp; }
}
package com.example.demo.exception_handling.exception;

// Bài 5: Ném khi vi phạm logic nghiệp vụ → HTTP 409 Conflict
// Ví dụ: tạo Task có tiêu đề trùng với task đã tồn tại
public class BusinessException extends RuntimeException {

    private final String field;   // field bị vi phạm: "tieuDe"
    private final Object value;   // giá trị bị trùng: "Học Spring Boot"

    public BusinessException(String field, Object value, String message) {
        super(message);
        this.field = field;
        this.value = value;
    }

    public String getField() { return field; }
    public Object getValue() { return value; }
}
package com.example.demo.exception_handling.exception;

// Bài 4: Ném khi tìm không thấy resource trong DB → HTTP 404
// Extends RuntimeException: không cần khai báo throws ở mọi method
public class ResourceNotFoundException extends RuntimeException {

    private final String resourceName; // "Task", "User"...
    private final Long   resourceId;   // id không tìm thấy

    public ResourceNotFoundException(String resourceName, Long id) {
        // Message tự động: "Task không tìm thấy với ID: 99"
        super(resourceName + " không tìm thấy với ID: " + id);
        this.resourceName = resourceName;
        this.resourceId   = id;
    }

    public String getResourceName() { return resourceName; }
    public Long   getResourceId()   { return resourceId; }
}
```

Toàn bộ luồng CRUD dùng các exception này: xem [`exception_handling/service/impl/TaskServiceImpl.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/exception_handling/service/impl/TaskServiceImpl.java)
và [`exception_handling/controller/TaskController.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/exception_handling/controller/TaskController.java).

## Checklist hiểu bài

- [ ] Giải thích được vì sao dùng `@RestControllerAdvice` thay vì `try/catch` trong từng Controller
- [ ] Tạo được 1 custom exception mới (extends `RuntimeException`) và bắt nó trong `GlobalExceptionHandler`
- [ ] Biết `MethodArgumentNotValidException` được Spring tự ném ra khi nào (gợi ý: liên quan đến `@Valid`)

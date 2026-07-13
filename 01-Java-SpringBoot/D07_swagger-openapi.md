---
title: "Ngày 7 — Document API với Swagger/OpenAPI"
track: java-spring
day: 7
topic: swagger-openapi
tags: [swagger, openapi, springdoc, jwt-bearer]
source_goc: "Không có trong vault Obsidian gốc — bổ sung từ repo Code (config/SwaggerConfig.java)"
---

# Ngày 7 — Document API với Swagger/OpenAPI

> File này lấp khoảng trống: kế hoạch gốc yêu cầu "Swagger/OpenAPI" ở Ngày 7 nhưng vault
> Obsidian chưa có ghi chú. Nội dung dưới đây viết lại từ code thực tế trong repo `Code`.

## Vì sao cần Swagger?

Sau khi backend xong, đội Flutter cần biết: API nào, method gì, body ra sao, trả về gì.
Thay vì viết tài liệu tay (dễ lỗi thời khi code đổi), Swagger/OpenAPI **tự sinh trang
tài liệu tương tác** từ chính annotation trong code — luôn khớp với code thật, và cho
phép **gọi thử API ngay trên trình duyệt** (kể cả API cần JWT).

## Cấu hình JWT Bearer trong Swagger

Vì dự án có API cần đăng nhập (JWT), Swagger cần biết cách nhận token để hiện nút
**"Authorize"** cho phép nhập `Bearer <token>` một lần, dùng chung cho mọi API test thử.

## 💻 Code Example (repo `Code`)

> Nguồn: [`SangLeSoftZ/Code`](https://github.com/SangLeSoftZ/Code)

**Cấu hình OpenAPI + Bearer JWT** — [`config/SwaggerConfig.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/config/SwaggerConfig.java):

```java
package com.example.demo.config;

import io.swagger.v3.oas.annotations.OpenAPIDefinition;
import io.swagger.v3.oas.annotations.enums.SecuritySchemeType;
import io.swagger.v3.oas.annotations.info.Info;
import io.swagger.v3.oas.annotations.security.SecurityScheme;
import org.springframework.context.annotation.Configuration;

// Khai báo Swagger dùng Bearer JWT để xác thực
@Configuration
@OpenAPIDefinition(
    info = @Info(
        title = "SoftZ Demo API",
        version = "1.0",
        description = "API cho bài học Spring Boot — JWT, Exception Handling, Validation"
    )
)
@SecurityScheme(
    name = "bearerAuth",               // tên scheme — dùng trong @SecurityRequirement
    type = SecuritySchemeType.HTTP,
    scheme = "bearer",                 // kiểu: bearer token
    bearerFormat = "JWT"               // format: JWT
)
public class SwaggerConfig {
    // Không cần viết thêm gì — annotation lo hết
}
```

Sau khi chạy app, mở `http://localhost:8080/swagger-ui.html` để xem toàn bộ API
(bao gồm cả `baiA`, `baiB`, `exception_handling`, `jwt`...) và test thử trực tiếp.

## Checklist hiểu bài

- [ ] Biết `@OpenAPIDefinition` dùng để khai báo thông tin chung (title, version...)
- [ ] Biết `@SecurityScheme` dùng để khai báo cách xác thực (ở đây: Bearer JWT)
- [ ] Test thử được 1 API cần JWT ngay trên Swagger UI bằng nút "Authorize"

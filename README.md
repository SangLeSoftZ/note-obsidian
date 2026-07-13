---
title: "MỤC LỤC TỔNG — Tài liệu tự học Flutter (BLoC/Cubit) + Java Spring Boot"
---

# 📚 Mục lục tổng — Tài liệu tự học

Repo này được tổ chức lại từ vault Obsidian gốc (`SangLeSoftZ/obsidian`) theo đúng
**Kế hoạch tự học Flutter + Spring Boot 7 ngày**, tách riêng 2 chủ đề (Java/Spring Boot
và Flutter/BLoC) để dễ tra cứu (lookup) lại từng đầu mục đã học.

## 🗂 Cấu trúc thư mục

```
├── README.md                     ← file này (mục lục tổng)
├── QUY_UOC_DAT_TEN.md             ← quy ước đặt tên file tài liệu & code
├── 01-Java-SpringBoot/            ← toàn bộ lý thuyết Java + Spring Boot
├── 02-Flutter-Bloc/               ← toàn bộ lý thuyết Flutter + BLoC/Cubit
└── 03-DapAn-Checklist/            ← đáp án bài tập + checklist theo từng ngày (dùng chung 2 track)
```

## 🔎 Bảng lookup: Ngày học → Chủ đề → File

| Ngày | Buổi | Chủ đề | Track | File |
|---|---|---|---|---|
| 2 | Sáng | Java Core (OOP, Collection, Stream, Lambda) | Java | `01-Java-SpringBoot/D02_java-core-oop-collection-stream.md` |
| 2 | Chiều | Spring Core (IoC, DI, Bean, application.yml) | Java | `01-Java-SpringBoot/D02_spring-core-ioc-di-bean.md` |
| 2 | (ghi chú thêm) | Java cú pháp cơ bản | Java | `01-Java-SpringBoot/D02_ghi-chu-nhanh-java-syntax-co-ban.md` |
| 2 | (ghi chú thêm) | Tổng quan Spring Framework | Java | `01-Java-SpringBoot/D02_spring-framework-tong-quan-buoi4.md` |
| 2 | Tối | Bloc vs Cubit, thiết kế State | Flutter | `02-Flutter-Bloc/D02_bloc-vs-cubit-state-design.md` |
| 2 | (ghi chú thêm) | BLoC lý thuyết & thực hành cơ bản | Flutter | `02-Flutter-Bloc/D02_bloc-cubit-ly-thuyet-thuc-hanh.md` |
| 3 | Tối (nối Ngày 2) | REST API CRUD — Controller + DTO cho Task | Java | `01-Java-SpringBoot/D03_spring-mvc-rest-api-crud.md` |
| 3 | Đào sâu | Validation — cơ chế bên dưới (@NotBlank, @Valid) | Java | `01-Java-SpringBoot/D03_validation-co-che-ben-duoi.md` |
| 3 | Đào sâu | HTTP Status Code & Idempotency | Java | `01-Java-SpringBoot/D03_http-status-code-idempotency.md` |
| 3 | Đào sâu | flutter_bloc — cơ chế Stream & InheritedWidget | Flutter | `02-Flutter-Bloc/D03_bloc-cubit-co-che-ben-duoi.md` |
| 4 | Đào sâu | Spring Data JPA & Hibernate (bên dưới JpaRepository) | Java | `01-Java-SpringBoot/D04_spring-data-jpa-hibernate.md` |
| 4 | Đào sâu | Dio — Interceptor & xử lý lỗi (DioException) | Flutter | `02-Flutter-Bloc/D04_dio-interceptor-xu-ly-loi.md` |
| 4 | Đào sâu | Code Generation — json_serializable & freezed | Flutter | `02-Flutter-Bloc/D04_json-serializable-freezed-codegen.md` |
| 5 | Sáng | Spring Security + JWT Authentication | Java | `01-Java-SpringBoot/D05_spring-security-jwt.md` |
| 5 | Chiều | Clean Architecture Flutter + Repository + DI (get_it) | Flutter | `02-Flutter-Bloc/D05_clean-architecture-repository-di.md` |
| 5 | Tối | Login end-to-end: Flutter gọi API JWT | Flutter | `02-Flutter-Bloc/D05_login-e2e-flutter-jwt.md` |
| 6 | Test | SharedPreferences — Onboarding | Flutter | `02-Flutter-Bloc/D06_test-bai1-sharedpreferences-onboarding.md` |
| 6 | Test | Hive — Local Storage | Flutter | `02-Flutter-Bloc/D06_test-bai2-hive-local-storage.md` |
| 6 | Test | MultiBlocProvider — State Management | Flutter | `02-Flutter-Bloc/D06_test-bai3-multiblocprovider.md` |
| 2 | Đáp án | Đáp án bài tập + Checklist Ngày 2 | Chung | `03-DapAn-Checklist/D02_dap-an-checklist.md` |
| 3 | Đáp án | Checklist hiểu sâu + link tham khảo Ngày 3 | Chung | `03-DapAn-Checklist/D03_dap-an-checklist.md` |
| 4 | Đáp án | Checklist hiểu sâu + link tham khảo Ngày 4 | Chung | `03-DapAn-Checklist/D04_dap-an-checklist.md` |
| 6 | Đào sâu | Exception Handling tập trung (@RestControllerAdvice) — **mới bổ sung** | Java | `01-Java-SpringBoot/D06_exception-handling-controlleradvice.md` |
| 7 | Đào sâu | Unit Test JUnit 5 + Mockito — **mới bổ sung** | Java | `01-Java-SpringBoot/D07_unit-test-junit-mockito.md` |
| 7 | Đào sâu | Swagger/OpenAPI — **mới bổ sung** | Java | `01-Java-SpringBoot/D07_swagger-openapi.md` |

> ⚠️ **Còn thiếu (gap) so với kế hoạch gốc:** Ngày 1 (setup môi trường, Dart core cơ bản)
> và buổi tích hợp/demo cuối Ngày 7 **vẫn chưa có ghi chú trong vault gốc lẫn code mẫu**.
> Riêng Exception Handling / Unit Test / Swagger (thiếu ở lần soát trước) **đã được bổ
> sung** dựa trên code thực tế trong repo [`SangLeSoftZ/Code`](https://github.com/SangLeSoftZ/Code).

## 💻 Code Example — repo `SangLeSoftZ/Code`

Mỗi file lý thuyết đều có mục **`## 💻 Code Example`** ở cuối, trích code thật kèm link
trực tiếp tới file trên GitHub ([`https://github.com/SangLeSoftZ/Code`](https://github.com/SangLeSoftZ/Code)).

> ⚠️ **Lưu ý quan trọng:** repo `Code` hiện tại **chỉ chứa code Java/Spring Boot**
> (package `com.example.demo`: `bai1`–`bai4`, `baiA`, `baiB`, `jwt`, `exception_handling`,
> `config`, và test `bai6`/`bai7`). **Chưa có project Flutter/Dart nào** trong repo này —
> nên toàn bộ file trong `02-Flutter-Bloc/` hiện chỉ có ghi chú "đang chờ bổ sung" ở mục
> Code Example. Khi có repo code Flutter, cập nhật theo đúng `QUY_UOC_DAT_TEN.md`.

## 🌳 Sơ đồ cây thư mục

Graph view gốc của Obsidian (các node rời rạc, tên file như `Ngay4_LyThuyetSau`,
`baocaobuoi2`...) không cho biết node nào là Java, node nào là Flutter. Cấu trúc mới
thay bằng **2 cây thư mục tách riêng theo track**, nhóm theo ngày học (xem sơ đồ trực
quan `01-Java-SpringBoot/` và `02-Flutter-Bloc/` đã gửi kèm trong hội thoại tạo tài liệu này).

## 🏷 Cách tra cứu nhanh

- **Tra theo ngày:** dùng bảng trên, tìm số Ngày (D0x) → mở đúng file.
- **Tra theo chủ đề riêng lẻ (không nhớ ngày nào):** mở thư mục `01-Java-SpringBoot/`
  hoặc `02-Flutter-Bloc/`, tên file đã có sẵn từ khoá chủ đề (vd: `jpa-hibernate`,
  `bloc-vs-cubit`, `dio-interceptor`...).
- **Tra theo tag:** mỗi file có khối `---` (frontmatter) ở đầu ghi rõ `day`, `topic`, `tags`
  — nếu dùng Obsidian, có thể search theo các tag này (`tags: [jwt, spring-security]`...).
- Xem thêm `QUY_UOC_DAT_TEN.md` để hiểu quy ước đặt tên file tài liệu và cách nó khớp
  với tên file/class trong code.

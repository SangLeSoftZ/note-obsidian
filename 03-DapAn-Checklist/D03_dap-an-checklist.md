---
title: "Ngày 3 — Checklist hiểu sâu & Link đào sâu thêm"
track: chung
day: 3
topic: dap-an-checklist
tags: [checklist, tham-khao]
source_goc: Ngay3_LyThuyetSau.md
---

# Ngày 3 — Checklist hiểu sâu & Link đào sâu thêm

## CHECKLIST HIỂU SÂU NGÀY 3

- [ ] Giải thích được thứ tự chính xác: JSON → deserialize → `@Valid` kiểm tra → method Controller chạy (hoặc bị chặn)
- [ ] Biết khi nào cần thêm `@Valid` lồng cho nested object
- [ ] Giải thích được `Cubit` thực chất là 1 `StreamController` được bọc lại
- [ ] Hiểu vì sao `context.read` chỉ hoạt động bên trong phạm vi `BlocProvider`
- [ ] Phân biệt được ý nghĩa `4xx` vs `5xx`, và khái niệm idempotency áp dụng vào request nào

## LINK ĐÀO SÂU THÊM

- Bean Validation spec (JSR 380) chi tiết: https://beanvalidation.org/2.0/spec/
- Custom Validator trong Spring: https://www.baeldung.com/spring-mvc-custom-validator
- flutter_bloc — Architecture (giải thích cách Cubit dùng Stream): https://bloclibrary.dev/bloc-concepts/
- HTTP Status Code đầy đủ: https://developer.mozilla.org/en-US/docs/Web/HTTP/Status
- Idempotency trong REST API (MDN): https://developer.mozilla.org/en-US/docs/Glossary/Idempotent

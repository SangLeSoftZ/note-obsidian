---
title: "Ngày 4 — Checklist hiểu sâu & Link đào sâu thêm"
track: chung
day: 4
topic: dap-an-checklist
tags: [checklist, tham-khao]
source_goc: Ngay4_LyThuyetSau.md
---

# Ngày 4 — Checklist hiểu sâu & Link đào sâu thêm

## CHECKLIST HIỂU SÂU NGÀY 4

- [ ] Phân biệt được JPA (đặc tả) và Hibernate (implementation)
- [ ] Giải thích được Hibernate quyết định INSERT hay UPDATE dựa trên điều gì
- [ ] Hiểu được vấn đề N+1 Query là gì và vì sao nó nguy hiểm khi dữ liệu lớn
- [ ] Giải thích được `@Transactional` bảo vệ dữ liệu khỏi trạng thái "nửa vời" như thế nào
- [ ] Hiểu Interceptor của Dio giải quyết vấn đề gì (không phải chỉ để log)
- [ ] Giải thích được vì sao Dart chọn Code Generation thay vì Reflection như Java

## LINK ĐÀO SÂU THÊM

- Hibernate — N+1 Query Problem giải thích chi tiết: https://vladmihalcea.com/n-plus-1-query-problem/
- Spring `@Transactional` — cách hoạt động thực sự: https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html
- Dio Interceptors — tài liệu chính thức: https://pub.dev/documentation/dio/latest/dio/Interceptor-class.html
- json_serializable — cách hoạt động (code generation): https://pub.dev/packages/json_serializable
- freezed — Union Types & pattern matching: https://pub.dev/packages/freezed

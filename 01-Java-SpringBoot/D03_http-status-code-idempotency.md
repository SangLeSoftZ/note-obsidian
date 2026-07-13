---
title: "Ngày 3 — HTTP Status Code & Idempotency"
track: java-spring
day: 3
topic: http-api-design
tags: [http, status-code, idempotency, rest]
source_goc: Ngay3_LyThuyetSau.md (Phần 3)
---

# Ngày 3 — HTTP Status Code & Idempotency

# PHẦN 3 — HTTP STATUS CODE & IDEMPOTENCY (kiến thức hay bị bỏ qua nhưng dùng hằng ngày)

### Status code không phải "số ngẫu nhiên" — mỗi nhóm mang 1 ý nghĩa

| Nhóm | Ý nghĩa | Ví dụ hay gặp |
|---|---|---|
| `2xx` | Thành công | `200 OK`, `201 Created` (khi POST tạo thành công) |
| `4xx` | Lỗi **do client** gửi sai | `400 Bad Request` (validation), `404 Not Found` (không tìm thấy id), `401 Unauthorized` (chưa đăng nhập) |
| `5xx` | Lỗi **do server** | `500 Internal Server Error` (bug trong code, exception không bắt) |

**Vì sao phân biệt 4xx và 5xx quan trọng trong thực tế?** Khi debug, nếu thấy lỗi `4xx` → nhìn vào dữ liệu **client gửi lên** trước (có đúng định dạng không, có thiếu field không). Nếu thấy `5xx` → lỗi nằm ở **code backend** (thường là NullPointerException hoặc exception chưa được xử lý) — 2 hướng debug hoàn toàn khác nhau, biết phân biệt giúp bạn không mất thời gian tìm sai chỗ.

### Idempotency — vì sao PUT và DELETE "an toàn" hơn POST khi gọi lại nhiều lần?

**Idempotent** = gọi 1 lần hay gọi 100 lần, kết quả cuối cùng vẫn giống nhau.

- `GET /api/tasks/5` gọi 100 lần → vẫn chỉ trả về đúng task đó, không đổi gì → idempotent
- `PUT /api/tasks/5` (update toàn bộ thành data X) gọi 100 lần → task vẫn có giá trị = X sau cùng → idempotent
- `DELETE /api/tasks/5` gọi 100 lần → sau lần đầu đã xóa, các lần sau "xóa cái không tồn tại" nhưng kết quả cuối cùng vẫn là "task 5 không còn tồn tại" → idempotent
- `POST /api/tasks` gọi 100 lần → tạo ra **100 task mới khác nhau** → **KHÔNG** idempotent

**Ứng dụng thực tế:** đây là lý do vì sao khi mạng chập chờn, app di động thường tự động **retry** (gửi lại) request `GET`/`PUT`/`DELETE` mà không sợ hậu quả, nhưng **không nên tự động retry `POST`** nếu không có cơ chế chống trùng — nếu không, người dùng bấm "Đặt hàng" 1 lần nhưng do mạng lag, app retry ngầm → có thể tạo ra 2-3 đơn hàng trùng.

---

## 💻 Code Example (repo `Code`)

> Ví dụ Controller minh hoạ đúng các status code (200 OK, 201 Created, 204 No Content)
> xem [`baiA/controller/BaiAController.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/baiA/controller/BaiAController.java)
> (đã trích dẫn đầy đủ trong `D03_spring-mvc-rest-api-crud.md`) — chú ý `ResponseEntity.status(HttpStatus.CREATED)` ở POST và `noContent()` ở DELETE.

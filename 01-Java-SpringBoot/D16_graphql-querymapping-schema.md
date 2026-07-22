---
title: "Ngày 16 (Tuần 3 - Ngày 5, Tối) — GraphQL cơ bản với Spring for GraphQL"
track: java-spring
day: 16
topic: graphql-querymapping-schema
tags: [spring-boot, graphql, querymapping, schema, graphiql]
source_goc: Tuan3-Ngay5-SpringSimulation-ShellRoute-GraphQL.md
---

# Ngày 16 (Tuần 3 - Ngày 5, Tối) — GraphQL cơ bản với Spring for GraphQL

## 1. Vấn đề: REST cứng nhắc về field trả về

Với REST (`GET /api/tasks/5`), server **luôn trả về toàn bộ field** đã định nghĩa sẵn
trong response DTO — dù client chỉ cần đúng `tieuDe` để hiển thị trong 1 danh sách rút
gọn, response vẫn kèm cả `moTa`, `createdAt`, `phienBan`... gây lãng phí băng thông.
**GraphQL** đảo ngược cách tiếp cận: **client tự khai báo chính xác field mình cần**
trong 1 query, server chỉ trả về **đúng những field đó** — 1 endpoint duy nhất phục vụ
được mọi nhu cầu hiển thị khác nhau.

## 2. Schema — dùng đúng field thật của `Task` (không phải bản rút gọn)

Code thật — `src/main/resources/graphql/schema.graphqls`. Điểm khác quan trọng so với ví
dụ lý thuyết gốc: type `Task` khai báo **đúng các field thật** đã xây dựng từ Tuần 3
Ngày 3 (JPA Auditing + Optimistic Locking) — `createdAt`, `updatedAt`, `phienBan` — không
phải bản rút gọn chỉ có `hoanThanh: Boolean!` như ví dụ minh họa trong tài liệu gốc:

```graphql
type Task {
    id: ID!
    tieuDe: String!
    moTa: String
    trangThai: String!
    createdAt: String
    updatedAt: String
    phienBan: Int
}

# Bài 7: type thống kê tổng hợp
type ThongKe {
    tongSoTask: Int!
    soTaskHoanThanh: Int!
    soTaskChuaLam: Int!
    soTaskDangLam: Int!
}

type Query {
    tatCaTasks: [Task!]!
    taskTheoId(id: ID!): Task
    thongKeTask: ThongKe!  # Bài 7
}
```

`String!`, `Boolean!`, `ID!` — dấu `!` nghĩa là field **bắt buộc có giá trị**, không được
`null`. Không có `!` (như `moTa: String`, `createdAt: String`) nghĩa là field **có thể
`null`** — khớp đúng thực tế: `moTa` là optional khi tạo Task, còn `createdAt` có thể
chưa có giá trị nếu (giả định) 1 bản ghi cũ chưa qua Auditing.

## 3. `TaskGraphQLController` — Bài 6 và Bài 7 viết chung 1 Controller

Code thật — `controller/TaskGraphQLController.java`:

```java
@Controller // LƯU Ý: KHÔNG phải @RestController
public class TaskGraphQLController {

    private final TaskRepository taskRepository;

    public TaskGraphQLController(TaskRepository taskRepository) {
        this.taskRepository = taskRepository;
    }

    // Tên method = tên field trong type Query của schema
    @QueryMapping
    public List<Task> tatCaTasks() {
        return taskRepository.findAll();
    }

    // @Argument ánh xạ tham số "id" từ GraphQL query vào Long id
    @QueryMapping
    public Task taskTheoId(@Argument Long id) {
        return taskRepository.findById(id).orElse(null);
    }
}
```

`@Controller` (không phải `@RestController`) — vì response GraphQL không được Spring MVC
serialize trực tiếp theo cách REST thông thường; framework GraphQL tự xử lý phần đóng gói
JSON theo đúng cấu trúc `{ "data": {...} }`.

## 4. Bài 7 — mở rộng Schema không ảnh hưởng Query cũ, và 1 kỹ thuật hay: trả `Map`

Code thật cho `thongKeTask()` — chi tiết thú vị không nằm trong tài liệu lý thuyết gốc:
trả về **`Map<String, Integer>`**, không phải 1 class Java riêng tên `ThongKe`:

```java
// Trả về Map — Spring GraphQL tự map sang type ThongKe trong schema
// Mở rộng schema không ảnh hưởng 2 query cũ ở trên — đây là điểm mạnh GraphQL
@QueryMapping
public Map<String, Integer> thongKeTask() {
    List<Task> tatCa = taskRepository.findAll();

    int tongSo    = tatCa.size();
    int hoanThanh = (int) tatCa.stream().filter(t -> "HOAN_THANH".equals(t.getTrangThai())).count();
    int chuaLam   = (int) tatCa.stream().filter(t -> "CHUA_LAM".equals(t.getTrangThai())).count();
    int dangLam   = (int) tatCa.stream().filter(t -> "DANG_LAM".equals(t.getTrangThai())).count();

    return Map.of(
        "tongSoTask",      tongSo,
        "soTaskHoanThanh", hoanThanh,
        "soTaskChuaLam",   chuaLam,
        "soTaskDangLam",   dangLam
    );
}
```

**Vì sao không cần tạo class `ThongKe.java` riêng?** Spring for GraphQL tự khớp **tên
key** trong `Map` trả về với **tên field** khai báo trong `type ThongKe` của schema —
không yêu cầu Java trả về đúng 1 class có tên/cấu trúc khớp 100% với type GraphQL. Đây là
cách viết gọn cho các query "tổng hợp/thống kê" đơn giản, không cần định nghĩa thêm 1
entity/DTO chỉ để phục vụ 1 query duy nhất — miễn là tên key trong `Map` khớp đúng tên
field trong schema.

**Bài học về khả năng mở rộng:** thêm `type ThongKe` + 1 dòng `thongKeTask: ThongKe!` vào
`type Query`, thêm 1 method `@QueryMapping` mới — 2 query cũ (`tatCaTasks`, `taskTheoId`)
**không cần sửa gì**, vẫn hoạt động y như trước. Đây chính là điểm mạnh thực tế của
GraphQL khi hệ thống phát triển thêm nhu cầu mới, không giống REST nhiều khi phải cân nhắc
kỹ trước khi đổi cấu trúc response của endpoint đang dùng (dễ vỡ hợp đồng với client cũ).

## 5. Bật GraphiQL — UI test trực quan có sẵn

Code thật — `application-hoc.properties`:

```properties
# Bật GraphiQL UI để test tại http://localhost:9090/graphiql
spring.graphql.graphiql.enabled=true
spring.graphql.graphiql.path=/graphiql
```

**Lưu ý về cổng:** profile `hoc` (dùng khi học/test độc lập) chạy ở cổng `9090`, khác với
profile `flutter` (cổng `8080`) đã dùng khi kết nối từ app Flutter thật (VD `StompService`
ở Tuần 3 Ngày 2 dùng `10.0.2.2:8080`) — cần chú ý xem đang chạy Spring Boot với profile
nào để test đúng cổng tương ứng.

## 6. Client gửi query — thử tự tay trên GraphiQL

```graphql
# Client chỉ cần tieuDe và trangThai -> server chỉ trả về đúng 2 field này
query {
  tatCaTasks {
    tieuDe
    trangThai
  }
}
```

```graphql
# Bài 7: gọi thống kê tổng hợp
query {
  thongKeTask {
    tongSoTask
    soTaskHoanThanh
  }
}
```

**Vì sao KHÔNG cần viết nhiều hàm trả về DTO rút gọn khác nhau như REST?** Vì Spring for
GraphQL tự động **chỉ serialize (chuyển thành JSON) đúng những field mà client yêu cầu**
trong query — dù hàm Java trả về `Task` đầy đủ (bao gồm `createdAt`, `phienBan`...), phần
field không được client hỏi tới sẽ **không xuất hiện** trong response cuối cùng. Đây là
cơ chế cốt lõi giải quyết vấn đề "REST cứng nhắc" — không phải do lập trình viên tự lọc
field thủ công, mà do bản chất cơ chế thực thi GraphQL.

## 7. So sánh trực tiếp với REST đã quen thuộc

| | REST (`GET /api/tasks`) | GraphQL (`POST /graphql`) |
|---|---|---|
| Field trả về | Cố định theo DTO server định nghĩa sẵn | Client tự chọn field mỗi lần gọi |
| Số endpoint cần cho nhiều nhu cầu hiển thị | Nhiều (`/tasks`, `/tasks/summary`...) | 1 endpoint duy nhất (`/graphql`) |
| HTTP method | Nhiều (`GET`/`POST`/`PUT`/`DELETE`) | Chủ yếu `POST` (kể cả cho Query đọc dữ liệu) |
| Phù hợp | API đơn giản, ít nhu cầu tùy biến field | Client đa dạng (web, mobile, đối tác thứ 3) có nhu cầu field khác nhau nhiều |

**Lưu ý phạm vi hôm nay:** chỉ học `Query` (đọc dữ liệu) ở mức cơ bản — `Mutation` (ghi
dữ liệu) và `DataLoader` (chống N+1 query khi có quan hệ lồng nhau) chưa nằm trong phạm vi
hôm nay.

## Bài tập

**Bài 5 (đã có sẵn — `schema.graphqls` + `pom.xml`):** Xác nhận dependency
`spring-boot-starter-graphql` đã có, schema khai báo đúng `type Task` + `type Query`.

**Bài 6 (đã có sẵn — `TaskGraphQLController.tatCaTasks/taskTheoId`):** Mở
`http://localhost:9090/graphiql` (đúng profile `hoc`), gửi query chỉ lấy `tieuDe` +
`trangThai`, sau đó thử lại với đầy đủ field hơn (`id`, `createdAt`) — quan sát response
thay đổi tương ứng theo đúng field yêu cầu.

**Bài 7 (đã có sẵn — `type ThongKe` + `thongKeTask()`):** Gửi query `thongKeTask` trên
GraphiQL, xác nhận số liệu tổng hợp đúng với dữ liệu Task hiện có, đồng thời xác nhận 2
query cũ vẫn hoạt động bình thường không bị ảnh hưởng.

## Code Example (repo `java.git`)

> Xem nguyên các file tại branch `GraphQL`:
> [`schema.graphqls`](https://github.com/SangLeSoftZ/java/blob/GraphQL/src/main/resources/graphql/schema.graphqls) ·
> [`TaskGraphQLController.java`](https://github.com/SangLeSoftZ/java/blob/GraphQL/src/main/java/com/example/demo/controller/TaskGraphQLController.java) ·
> [`application-hoc.properties`](https://github.com/SangLeSoftZ/java/blob/GraphQL/src/main/resources/application-hoc.properties)

---
title: "Ngày 16 / Tuần 3 - Ngày 5 — Đáp án gợi ý & Checklist hoàn thành"
track: chung
day: 16
topic: dap-an-checklist
tags: [dap-an, checklist]
source_goc: Tuan3-Ngay5-SpringSimulation-ShellRoute-GraphQL.md
---

# Ngày 16 / Tuần 3 - Ngày 5 — Đáp án gợi ý & Checklist hoàn thành

## ĐÁP ÁN GỢI Ý

<details><summary>Bài 1 — SpringSimulation: thẻ vuốt-thả nảy về vị trí gốc (xem D16_springsimulation-physics-animation.md)</summary>

Đã có sẵn trong repo tại `lib/tuan3_ngay5/bai1a_spring_simulation_screen.dart` (chọn
Lựa chọn A):

```dart
final springDesc = SpringDescription(mass: 1.0, stiffness: 200.0, damping: 15.0);
final simulation = SpringSimulation(
  springDesc, _viTriX, 0, details.velocity.pixelsPerSecond.dx,
);
_controller.animateWith(simulation);
```

Có sẵn 3 preset (mềm/cân bằng/cứng) để đổi qua đổi lại. Kiểm chứng: vuốt thẻ nhanh vs
chậm ở cùng 1 preset, và so sánh giữa 3 preset khác nhau để cảm nhận vai trò
`stiffness`/`damping`.
</details>

<details><summary>Bài 2 — ShellRoute thường: quan sát mất state khi đổi tab (xem D16_shellroute-statefulshellroute.md)</summary>

Đã có sẵn trong repo tại `lib/tuan3_ngay5/bai2_shell_route_screen.dart`:

```dart
ShellRoute(
  builder: (context, state, child) => _ScaffoldWithBottomNav(child: child),
  routes: [ /* /shell/tasks, /shell/tasks/:id, /shell/profile */ ],
)
```

Kiểm chứng: vào chi tiết 1 Task, chuyển tab Profile, quay lại Task — xác nhận **bị reset**
về danh sách (đúng hành vi giới hạn của `ShellRoute` thường, sẽ sửa ở Bài 3).
</details>

<details><summary>Bài 3 — StatefulShellRoute.indexedStack: giữ state đúng mỗi tab (xem D16_shellroute-statefulshellroute.md)</summary>

Đã có sẵn trong repo tại `lib/tuan3_ngay5/bai3_stateful_shell_route_screen.dart`:

```dart
StatefulShellRoute.indexedStack(
  builder: (context, state, navigationShell) => _StatefulScaffold(navigationShell: navigationShell),
  branches: [
    StatefulShellBranch(routes: [/* /stateful/tasks, /stateful/tasks/:id */]),
    StatefulShellBranch(routes: [/* /stateful/profile */]),
  ],
)
```

Danh sách Task được nhân lên 10x (`List.generate(t.length * 10, ...)`) kèm
`ScrollController` để test được cả vị trí cuộn. Kiểm chứng: cuộn xuống giữa danh sách, vào
chi tiết 1 Task, chuyển tab Profile, quay lại Task — xác nhận **vẫn ở đúng màn hình chi
tiết đó**, và nếu back về danh sách, vẫn giữ đúng vị trí cuộn cũ.
</details>

<details><summary>Bài 4 — bấm lại tab đang đứng sẵn → về màn gốc (xem D16_shellroute-statefulshellroute.md)</summary>

Đã có sẵn trong repo tại `_StatefulScaffold`:

```dart
onDestinationSelected: (index) => navigationShell.goBranch(
  index,
  initialLocation: index == navigationShell.currentIndex,
),
```

Kiểm chứng: vào sâu 1 màn Task con, bấm lại đúng icon tab Task (đang đứng sẵn ở tab đó) —
xác nhận quay về đúng danh sách gốc.
</details>

<details><summary>Bài 5 — spring-boot-starter-graphql + schema.graphqls (xem D16_graphql-querymapping-schema.md)</summary>

Đã có sẵn trong repo tại `pom.xml` + `src/main/resources/graphql/schema.graphqls`:

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

type Query {
    tatCaTasks: [Task!]!
    taskTheoId(id: ID!): Task
}
```
</details>

<details><summary>Bài 6 — TaskGraphQLController với 2 @QueryMapping (xem D16_graphql-querymapping-schema.md)</summary>

Đã có sẵn trong repo tại `controller/TaskGraphQLController.java`:

```java
@Controller
public class TaskGraphQLController {
    @QueryMapping
    public List<Task> tatCaTasks() { return taskRepository.findAll(); }

    @QueryMapping
    public Task taskTheoId(@Argument Long id) { return taskRepository.findById(id).orElse(null); }
}
```

Kiểm chứng: mở `http://localhost:9090/graphiql` (profile `hoc`), gửi query chỉ lấy
`tieuDe` + `trangThai`, sau đó thử lại với đầy đủ field hơn — quan sát response thay đổi
tương ứng.
</details>

<details><summary>Bài 7 — Mở rộng schema với ThongKe, không ảnh hưởng Query cũ (xem D16_graphql-querymapping-schema.md)</summary>

Đã có sẵn trong repo:

```graphql
type ThongKe {
    tongSoTask: Int!
    soTaskHoanThanh: Int!
    soTaskChuaLam: Int!
    soTaskDangLam: Int!
}
```

```java
@QueryMapping
public Map<String, Integer> thongKeTask() {
    // trả về Map — Spring GraphQL tự map sang type ThongKe
    return Map.of("tongSoTask", tongSo, "soTaskHoanThanh", hoanThanh, ...);
}
```

Kiểm chứng: gửi query `thongKeTask` trên GraphiQL, xác nhận số liệu đúng và 2 query cũ
(`tatCaTasks`, `taskTheoId`) vẫn hoạt động bình thường.
</details>

---

## ✅ CHECKLIST HOÀN THÀNH

- [ ] Giải thích được vì sao `SpringSimulation` phản hồi vận tốc thực tế còn `Curves` cố định thì không
- [ ] Hiểu vai trò 3 tham số `mass`/`stiffness`/`damping`, tự cảm nhận qua ít nhất 2 preset khác nhau
- [ ] Tự quan sát được giới hạn của `ShellRoute` thường (mất state khi đổi tab) trước khi chuyển sang `StatefulShellRoute`
- [ ] Bottom Navigation Bar dùng `StatefulShellRoute.indexedStack`, mỗi tab giữ đúng stack điều hướng + vị trí cuộn riêng
- [ ] Giải thích được khác biệt `goBranch` vs `context.go` thông thường
- [ ] Query GraphQL trả về đúng, chỉ đúng field mà client yêu cầu — không thừa, không thiếu
- [ ] Hiểu vì sao dùng `@Controller` (không phải `@RestController`) cho GraphQL Controller
- [ ] Mở rộng được Schema (thêm type/query mới) mà không ảnh hưởng query cũ đã có


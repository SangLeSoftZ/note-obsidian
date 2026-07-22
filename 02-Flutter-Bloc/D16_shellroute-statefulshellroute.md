---
title: "Ngày 16 (Tuần 3 - Ngày 5, Chiều) — ShellRoute vs StatefulShellRoute (Bottom Nav)"
track: flutter
day: 16
topic: shellroute-statefulshellroute
tags: [flutter, go_router, shellroute, statefulshellroute, bottom-navigation]
source_goc: Tuan3-Ngay5-SpringSimulation-ShellRoute-GraphQL.md
---

# Ngày 16 (Tuần 3 - Ngày 5, Chiều) — ShellRoute vs StatefulShellRoute (Bottom Nav)

## 1. Cách dạy hay trong repo: viết CẢ 2 bản để tự thấy sự khác biệt

Khác với tài liệu lý thuyết gốc (chỉ đưa thẳng `StatefulShellRoute.indexedStack` là giải
pháp đúng), code thật viết **2 màn hình độc lập** — Bài 2 dùng `ShellRoute` thường (có
giới hạn), Bài 3 dùng `StatefulShellRoute.indexedStack` (giải pháp đúng) — để tự trải
nghiệm được vấn đề **trước khi** thấy cách giải quyết, không chỉ đọc lý thuyết suông.

## 2. Bài 2 — `ShellRoute` thường: cố ý để lộ giới hạn "mất state khi đổi tab"

Code thật — `lib/tuan3_ngay5/bai2_shell_route_screen.dart`:

```dart
final shellRouter = GoRouter(
  initialLocation: '/shell/tasks',
  routes: [
    ShellRoute(
      // builder nhận "child" = nội dung route con đang active
      builder: (context, state, child) => _ScaffoldWithBottomNav(child: child),
      routes: [
        GoRoute(
          path: '/shell/tasks',
          builder: (context, state) => const _TaskListTab(),
          routes: [
            GoRoute(
              path: ':id',
              builder: (context, state) => _TaskDetailTab(taskId: state.pathParameters['id']!),
            ),
          ],
        ),
        GoRoute(path: '/shell/profile', builder: (context, state) => const _ProfileTab()),
      ],
    ),
  ],
);
```

Comment trong code tự nói rõ vấn đề, ngay trong UI hiển thị cho người học:

```dart
child: const Text(
  '📌 Bài 2: ShellRoute\n'
  'Nhấn vào task → vào chi tiết → chuyển tab Profile\n'
  '→ Quay lại Task: BỊ reset về danh sách (xem Bài 3 để fix)',
),
```

**Vì sao lại bị reset?** `ShellRoute` (không có `Stateful`) chỉ có **1 Navigator stack
dùng chung** cho toàn bộ route con — dù khung ngoài (`_ScaffoldWithBottomNav`, chứa
`BottomNavigationBar`) không bị rebuild khi đổi tab, phần route con (`child`) vẫn bị thay
thế hoàn toàn mỗi lần chuyển route, không giữ lại vị trí đã điều hướng sâu trước đó trong
tab kia.

## 3. Bài 3 — `StatefulShellRoute.indexedStack`: mỗi branch có stack riêng thật sự

Code thật — `lib/tuan3_ngay5/bai3_stateful_shell_route_screen.dart`:

```dart
final statefulShellRouter = GoRouter(
  initialLocation: '/stateful/tasks',
  routes: [
    StatefulShellRoute.indexedStack(
      builder: (context, state, navigationShell) =>
          _StatefulScaffold(navigationShell: navigationShell),
      branches: [
        // Branch 1: Tab Task (có navigator stack riêng)
        StatefulShellBranch(
          routes: [
            GoRoute(
              path: '/stateful/tasks',
              builder: (context, state) => const _StatefulTaskList(),
              routes: [
                GoRoute(
                  path: ':id',
                  builder: (context, state) => _StatefulTaskDetail(taskId: state.pathParameters['id']!),
                ),
              ],
            ),
          ],
        ),
        // Branch 2: Tab Profile (navigator stack riêng, độc lập)
        StatefulShellBranch(
          routes: [GoRoute(path: '/stateful/profile', builder: (context, state) => const _StatefulProfile())],
        ),
      ],
    ),
  ],
);
```

**Khác với ví dụ lý thuyết gốc:** tài liệu gốc khai báo `navigatorKey` riêng cho từng
`StatefulShellBranch` (`_shellNavigatorKeyTask`, `_shellNavigatorKeyProfile`) — code thật
**không khai báo** `navigatorKey` tường minh. Điều này vẫn hoạt động đúng vì go_router tự
động sinh 1 `GlobalKey<NavigatorState>` riêng cho mỗi branch nếu không truyền vào — chỉ
cần tự khai báo `navigatorKey` khi cần **truy cập trực tiếp** `NavigatorState` của 1
branch cụ thể từ bên ngoài (ít gặp trong bài học cơ bản này).

```dart
class _StatefulScaffold extends StatelessWidget {
  final StatefulNavigationShell navigationShell;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      // navigationShell thay cho child — mỗi branch là IndexedStack riêng
      body: navigationShell,
      bottomNavigationBar: NavigationBar(
        selectedIndex: navigationShell.currentIndex,
        onDestinationSelected: (index) => navigationShell.goBranch(
          index,
          // bấm lại tab đang ở → về root của tab đó
          initialLocation: index == navigationShell.currentIndex,
        ),
        destinations: const [...],
      ),
    );
  }
}
```

## 4. Chứng minh bằng thực nghiệm: nhân danh sách Task lên 10x + `ScrollController`

Chi tiết hay nhất, không có trong tài liệu lý thuyết gốc — code thật **cố ý** tạo danh
sách dài để kiểm chứng được cả vị trí cuộn, không chỉ màn hình đang xem:

```dart
final _scrollController = ScrollController();

Future<void> _load() async {
  setState(() => _loading = true);
  final t = await _api.layDanhSachTask();
  // Nhân task lên 10x để danh sách đủ dài để cuộn
  setState(() => _tasks = List.generate(t.length * 10, (i) => t[i % t.length]));
  setState(() => _loading = false);
}
```

Kịch bản test được viết thẳng trong UI: cuộn xuống giữa danh sách → nhấn vào 1 Task → vào
chi tiết → chuyển tab Profile → quay lại tab Task → **vẫn ở đúng màn hình chi tiết, và
nếu bấm Back về danh sách, vẫn ở đúng vị trí cuộn cũ**. Đây là mức độ kiểm chứng cao hơn
nhiều so với chỉ kiểm tra "còn ở đúng route hay không" — còn xác nhận cả `ScrollController`
(gắn với `State` của widget) không bị huỷ và tạo lại.

## 5. `goBranch` vs `context.go` — lỗi rất hay gặp

**`navigationShell.goBranch(index)` khác `context.go('/tasks')` như thế nào?** `goBranch`
chuyển giữa các **nhánh đã được quản lý sẵn**, giữ nguyên state/stack của từng nhánh
(giống chuyển app khác rồi quay lại, app cũ vẫn ở đúng chỗ đang dừng). Nếu dùng
`context.go('/stateful/tasks')` thông thường sẽ **reset** về route gốc của nhánh đó, mất
lịch sử điều hướng bên trong tab — đây là lỗi rất hay gặp khi mới chuyển từ Bottom Nav
thường sang `ShellRoute`.

**Vì sao cần `initialLocation: index == navigationShell.currentIndex`?** Đây là hành vi
UX quen thuộc của hầu hết app (Instagram, Facebook...): bấm vào tab **đang đứng sẵn** ở đó
sẽ **quay về màn hình gốc** của tab (thay vì giữ nguyên màn hình con đang xem dở) — dòng
điều kiện này match đúng hành vi đó.

## 6. `ShellRoute` thường — khi nào vẫn nên dùng

`ShellRoute` (không `Stateful`) phù hợp khi chỉ cần 1 khung UI **dùng chung** (VD `AppBar`
cố định, `Drawer` cố định) bao quanh nhiều route, nhưng **không cần** giữ nhiều stack song
song như Bottom Nav — đơn giản hơn `StatefulShellRoute` nhưng không tách biệt state từng
nhánh. Bài 2 trong repo là ví dụ chính xác cho trường hợp này — dùng đúng khi chấp nhận
được việc mất state khi đổi "khu vực", không dùng khi cần Bottom Nav giữ trạng thái nhiều
tab như Bài 3.

## Bài tập

**Bài 2 (đã có sẵn — `bai2_shell_route_screen.dart`):** Chạy màn hình, vào chi tiết 1
Task, chuyển sang Profile, quay lại Task — tự xác nhận đúng hiện tượng bị reset về danh
sách như comment trong code đã báo trước.

**Bài 3 (đã có sẵn — `bai3_stateful_shell_route_screen.dart`):** Lặp lại đúng kịch bản
trên với `StatefulShellRoute` — cuộn xuống giữa danh sách, vào chi tiết, đổi tab, quay
lại — xác nhận vẫn đúng màn hình và vị trí cuộn cũ.

**Bài 4 (đã có sẵn qua `initialLocation: index == navigationShell.currentIndex`):** Vào
sâu 1 màn Task con, bấm lại đúng icon tab Task (đang đứng sẵn ở tab đó) — xác nhận quay về
đúng danh sách gốc, không giữ nguyên màn con đang xem dở.

## Code Example (repo `flutter.git`)

> Xem nguyên các file tại branch `NestedNavigator--ShellRoute`:
> [`bai2_shell_route_screen.dart`](https://github.com/SangLeSoftZ/flutter/blob/NestedNavigator--ShellRoute/testflutter/lib/tuan3_ngay5/bai2_shell_route_screen.dart) ·
> [`bai3_stateful_shell_route_screen.dart`](https://github.com/SangLeSoftZ/flutter/blob/NestedNavigator--ShellRoute/testflutter/lib/tuan3_ngay5/bai3_stateful_shell_route_screen.dart)


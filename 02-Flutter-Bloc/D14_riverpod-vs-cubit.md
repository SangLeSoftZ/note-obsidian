---
title: "Ngày 14 (Tuần 3 - Ngày 3, Chiều) — Riverpod: So sánh trực tiếp với Cubit"
track: flutter
day: 14
topic: riverpod-vs-cubit
tags: [flutter, riverpod, stateprovider, notifier, cubit, state-management]
source_goc: Tuan3-Ngay3-Generics-Mixin-Riverpod-JPAAuditing.md
---

# Ngày 14 (Tuần 3 - Ngày 3, Chiều) — Riverpod: So sánh trực tiếp với Cubit

## 1. Vấn đề Riverpod giải quyết — không cần `BuildContext`

Cubit/Bloc cần `BuildContext` để truy cập (`context.read<T>()`, `BlocProvider`) — nghĩa là
**phải nằm trong cây widget** mới dùng được. Điều này gây khó khi cần đọc/gọi state
**ngoài widget tree** (VD trong 1 hàm xử lý nền, trong interceptor của Dio, trong unit
test không dựng UI). Riverpod tách hoàn toàn state ra khỏi widget tree — `ref` hoạt động
cả ngoài widget tree.

**Không có nghĩa Riverpod "tốt hơn" Bloc tuyệt đối** — đây là bài học mở rộng góc nhìn,
không nhất thiết phải đổi hẳn project đang làm (project chính vẫn tiếp tục dùng Cubit).

## 2. Bài 4 — `StateProvider<int>`, đối chiếu song song với Cubit ngay trong UI

Code thật — `lib/tuan3_ngay3/bai4_state_provider_screen.dart` — điểm hay: màn hình tự
hiển thị bảng so sánh Cubit vs Riverpod ngay trong UI, không chỉ ở comment:

```dart
// Provider khai báo ở top-level — không cần class, không cần Provider widget
// Tương đương: class DemCubit extends Cubit<int> { DemCubit() : super(0); }
final demTaskProvider = StateProvider<int>((ref) => 0);

class Bai4StateProviderScreen extends ConsumerWidget {
  const Bai4StateProviderScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // ref.watch() → đọc + tự rebuild khi state đổi
    // Tương đương: context.watch<DemCubit>().state
    final soLuong = ref.watch(demTaskProvider);
    ...
  }
}
```

```dart
// ref.read() → chỉ gọi action, không cần rebuild
// Tương đương: context.read<DemCubit>().decrement()
onPressed: soLuong > 0 ? () => ref.read(demTaskProvider.notifier).state-- : null,
```

**`ConsumerWidget`** — tương đương `StatelessWidget` nhưng `build()` nhận thêm
`WidgetRef ref` — đây là "cửa" duy nhất để đọc/ghi provider, thay cho `BuildContext`
trong cách làm của Bloc. `demTaskProvider.notifier` trỏ tới đối tượng điều khiển state
(tương đương chính instance `Cubit`), còn `demTaskProvider` (không có `.notifier`) trỏ
tới **giá trị state hiện tại**.

## 3. Bài 5 — `TaskListNotifier`, đối chiếu dòng-với-dòng với `TaskCubit`

Code thật — `lib/tuan3_ngay3/bai5_notifier_screen.dart`, comment đầu file liệt kê đúng
bảng tương ứng:

```dart
// TaskCubit:                    TaskListNotifier:
// extends Cubit<List<Task>>     extends Notifier<List<Task>>
// Cubit() : super([])           List<Task> build() => []
// emit([...state, task])        state = [...state, task]
// BlocProvider(create: ...)     NotifierProvider(TaskListNotifier.new)
// context.watch<TaskCubit>()    ref.watch(taskListProvider)
// context.read<TaskCubit>()     ref.read(taskListProvider.notifier)

class TaskListNotifier extends Notifier<List<Task>> {
  @override
  List<Task> build() => []; // = constructor + state ban đầu của Cubit

  void themTask(Task task) {
    // PHẢI tạo list mới — không sửa trực tiếp state cũ
    // Giống emit([...state, task]) trong Cubit
    state = [...state, task];
  }

  void xoaTask(int id) {
    state = state.where((t) => t.id != id).toList();
  }

  void toggleHoanThanh(Task task) {
    state = state.map((t) => t.id != task.id ? t : Task(
      id: t.id, tieuDe: t.tieuDe, moTa: t.moTa,
      trangThai: t.trangThai == 'HOAN_THANH' ? 'DANG_LAM' : 'HOAN_THANH',
    )).toList();
  }

  // Load từ API — giống hàm async trong Cubit
  Future<void> taiTuApi() async {
    try {
      final tasks = await ApiClient().layDanhSachTask();
      state = tasks; // gán trực tiếp, không cần emit()
    } catch (e) { /* error handling */ }
  }
}

final taskListProvider =
    NotifierProvider<TaskListNotifier, List<Task>>(TaskListNotifier.new);
```

Màn hình demo có đủ CRUD thật: thêm Task qua dialog, toggle hoàn thành qua `Checkbox`, xóa
qua `IconButton`, và load từ API thật qua nút tải — đủ để so sánh trải nghiệm viết code
với 1 `TaskCubit` tương đương đã quen trong project chính.

## 4. Bảng so sánh trực tiếp

| Tiêu chí | Cubit (bloc) | Riverpod |
|---|---|---|
| Cần `BuildContext` để đọc state? | Có (`context.read`/`context.watch`) | Không (`ref.read`/`ref.watch` hoạt động cả ngoài widget) |
| Cách cung cấp cho widget tree | `BlocProvider` bọc từng nơi cần | `ProviderScope` bọc **1 lần** ở gốc app |
| Phát hiện lỗi thiếu Provider | Lúc **runtime** | Nhiều lỗi phát hiện được lúc **compile-time** |
| Debug/log toàn cục | `BlocObserver` (Tuần 2 Ngày 1) | `ProviderObserver` (cơ chế tương tự) |
| Tự động huỷ khi không dùng nữa | Phải tự quản lý (`close()`) | Có `autoDispose` tự động dọn dẹp |

## 5. Bài 6 — 3 điểm khác biệt tự nhận thấy (đã có sẵn ngay trong UI của repo)

Code thật để sẵn phần trả lời Bài 6 ngay trong `Bai5NotifierScreen`, không phải chỉ là
bài tập bỏ ngỏ:

```dart
Text(
  '1. Không cần BlocProvider bọc widget — ProviderScope 1 lần ở main()\n'
  '2. state = [...] trực tiếp thay vì gọi emit() — dễ quên tạo list mới\n'
  '3. ref không cần BuildContext — dùng được trong hàm async bên ngoài widget',
)
```

Điểm số 2 đáng chú ý nhất về mặt thực hành: `emit()` của Cubit là 1 lời gọi hàm tường
minh, khó quên; còn `state = [...]` trong Riverpod chỉ là 1 phép **gán biến** thông
thường — dễ vô tình viết `state.add(task)` (sửa trực tiếp list cũ, không tạo list mới)
mà code vẫn biên dịch được, chỉ là **không kích hoạt rebuild** đúng cách. Đây là 1 bẫy
thực tế của Riverpod mà Cubit ít gặp hơn nhờ `emit()` buộc phải truyền tham số rõ ràng.

**Kết luận thực tế:** không cần chọn phe hay migrate cả project — mục tiêu là hiểu đủ để
biết Riverpod tồn tại, giải quyết vấn đề gì, để sau này nếu gặp dự án dùng Riverpod hoặc
cần quyết định cho dự án mới thì có đủ thông tin so sánh.

## Bài tập

**Bài 4 (đã có sẵn — `bai4_state_provider_screen.dart`):** Chạy màn hình, bấm Thêm/Bớt/
Reset — đối chiếu từng dòng code với bảng so sánh `ref.watch`/`ref.read` hiển thị ngay
trong UI.

**Bài 5 (đã có sẵn — `bai5_notifier_screen.dart`):** Chạy màn hình, thêm 1 Task mới, toggle
hoàn thành, xóa 1 Task, và bấm nút tải để load từ API thật — đối chiếu với 1 `TaskCubit`
tương đương trong project chính.

**Bài 6 (đã có sẵn ngay trong UI — xem mục 5 phía trên):** Đọc lại 3 điểm khác biệt đã
liệt kê, thử tự bổ sung thêm ít nhất 1 điểm khác biệt của riêng bạn dựa trên trải nghiệm
thực tế vừa làm.

##  Code Example (repo `flutter.git`)

> Xem nguyên các file tại branch `Generics,-Mixin,-Riverpod`:
> [`bai4_state_provider_screen.dart`](<https://github.com/SangLeSoftZ/flutter/blob/Generics,-Mixin,-Riverpod/testflutter/lib/tuan3_ngay3/bai4_state_provider_screen.dart>) ·
> [`bai5_notifier_screen.dart`](<https://github.com/SangLeSoftZ/flutter/blob/Generics,-Mixin,-Riverpod/testflutter/lib/tuan3_ngay3/bai5_notifier_screen.dart>)

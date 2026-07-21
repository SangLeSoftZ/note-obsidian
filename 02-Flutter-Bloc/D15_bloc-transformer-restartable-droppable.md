---
title: "Ngày 15 (Tuần 3 - Ngày 4, Chiều) — Bloc Transformer: restartable/sequential/droppable"
track: flutter
day: 15
topic: bloc-transformer-restartable-droppable
tags: [flutter, bloc, bloc_concurrency, restartable, droppable, sequential]
source_goc: Tuan3_Ngay4_TaiLieu.md
---

# Ngày 15 (Tuần 3 - Ngày 4, Chiều) — Bloc Transformer: restartable/sequential/droppable

## 1. Vấn đề: response cũ về sau response mới (race condition)

Mỗi ký tự gõ vào ô tìm kiếm có thể trigger 1 lần gọi API. Gõ nhanh "học flutter" bắn ra
nhiều request gần cùng lúc — vì mạng không đảm bảo thứ tự phản hồi, **request cũ có thể
trả về SAU** request mới, khiến UI hiển thị nhầm kết quả từ khóa cũ.

## 2. Bài 3+4 — 2 Bloc song song trong cùng 1 file để so sánh trực tiếp

Code thật — `lib/tuan3_ngay4/bai3_bai4_search_bloc.dart` — thay vì chỉ đổi transformer
qua lại trên 1 Bloc, repo viết **2 class Bloc riêng biệt** để chạy so sánh cùng lúc trên
2 tab:

```dart
// ── Bloc dùng restartable() — Bài 3 ──────────────────────────────
class SearchBlocRestartable extends Bloc<SearchEvent, SearchState> {
  final ApiClient _api;
  int _requestCount = 0;

  SearchBlocRestartable(this._api) : super(SearchInitial()) {
    on<TimKiemThayDoi>(
      _onTimKiem,
      // restartable(): Event mới đến → HỦY Future đang chạy
      // Chỉ kết quả của Event cuối cùng được emit
      transformer: restartable(),
    );
  }

  Future<void> _onTimKiem(TimKiemThayDoi event, Emitter<SearchState> emit) async {
    if (event.tuKhoa.isEmpty) { emit(SearchInitial()); return; }
    _requestCount++;
    final soNay = _requestCount;
    emit(SearchLoading(event.tuKhoa));
    try {
      // Giả lập network delay KHÁC NHAU để cố ý tạo race condition có thể quan sát
      await Future.delayed(Duration(milliseconds: 200 + (soNay % 3) * 300));
      final tasks = await _api.timKiemTask(event.tuKhoa);
      // Với restartable(): nếu đã có Event mới → dòng này không chạy
      emit(SearchLoaded(tasks, event.tuKhoa, soNay));
    } catch (e) {
      emit(SearchError(e.toString()));
    }
  }
}

// ── Bloc dùng sequential() — Bài 4 ───────────────────────────────
class SearchBlocSequential extends Bloc<SearchEvent, SearchState> {
  // Cấu trúc giống hệt SearchBlocRestartable, chỉ khác:
  // transformer: sequential()
}
```

**Chi tiết quan trọng không có trong tài liệu lý thuyết gốc:** dòng
`Duration(milliseconds: 200 + (soNay % 3) * 300)` **cố tình** cho mỗi request 1 độ trễ
mạng khác nhau (200ms, 500ms, hoặc 800ms tuỳ số dư chia 3) — mô phỏng đúng thực tế mạng
không ổn định, để bài học **thực sự quan sát được** race condition xảy ra, thay vì chỉ
tin vào lý thuyết suông.

`SearchState.requestSo` — đếm số thứ tự request — cũng là chi tiết thực tế giúp nhìn thấy
bằng mắt "request nào đang hiển thị kết quả", so sánh trực tiếp với "request nào lẽ ra
phải hiển thị" (request có `tuKhoa` khớp ô tìm kiếm hiện tại).

## 3. Màn hình so sánh trực tiếp bằng Tab — không cần đọc code cũng thấy khác biệt

Code thật — `lib/tuan3_ngay4/bai3_bai4_search_screen.dart`:

```dart
DefaultTabController(
  length: 2,
  child: Scaffold(
    appBar: AppBar(bottom: const TabBar(tabs: [
      Tab(text: 'Bài 3: restartable()', icon: Icon(Icons.restart_alt)),
      Tab(text: 'Bài 4: sequential()', icon: Icon(Icons.queue)),
    ])),
    body: TabBarView(children: [
      _SearchTab(bloc: SearchBlocRestartable(ApiClient()), ...),
      _SearchTab(bloc: SearchBlocSequential(ApiClient()), ...),
    ]),
  ),
)
```

Mỗi tab hiển thị **số ký tự đã gõ** và **số request thực sự** ngay trên UI (giống cách đo
lường debounce ở Tuần 3 Ngày 2) — gõ nhanh vào cả 2 tab, so sánh trực tiếp: tab
`restartable()` luôn hiển thị đúng kết quả khớp với ô tìm kiếm hiện tại; tab
`sequential()` có thể hiển thị kết quả của từ khóa **cũ hơn** trong 1 khoảnh khắc, do phải
xử lý tuần tự hết các request trước đó.

## 4. Bảng 4 chiến lược

| Transformer | Hành vi | Dùng khi nào |
|---|---|---|
| `restartable()` | Có Event mới, **hủy ngay** xử lý Event cũ, chỉ giữ kết quả mới nhất | Tìm kiếm — chỉ quan tâm kết quả mới nhất |
| `droppable()` | Đang xử lý, **bỏ qua** mọi Event mới tới khi xong | Nút "Submit" — tránh submit trùng |
| `sequential()` (mặc định) | Xử lý lần lượt theo thứ tự, Event sau chờ Event trước | Thao tác cần đảm bảo thứ tự |
| `concurrent()` | Xử lý tất cả song song, không quan tâm thứ tự hoàn thành | Tác vụ độc lập nhau |

## 5. Bài 5 — `droppable()` áp dụng vào luồng Login thật (không phải demo giả)

Code thật — `lib/tuan3_ngay4/bai5_droppable_login_bloc.dart` — điểm đáng chú ý nhất: đây
**không phải** 1 Bloc login giả lập, mà tái sử dụng **đúng luồng login thật** từ Tuần 1
(`LoginUseCase`, `AuthLocalDataSource`):

```dart
class LoginTransformerBloc extends Bloc<LoginTransformerEvent, LoginTransformerState> {
  int _soLanBam = 0;
  int _soRequestThucSu = 0;

  LoginTransformerBloc() : super(LoginTransformerInitial()) {
    on<NhanDangNhap>(
      _onDangNhap,
      // droppable(): đang xử lý → BỎ QUA Event mới
      // Bấm Login 5 lần → chỉ lần đầu được xử lý
      transformer: droppable(),
    );
    on<DangXuat>(_onDangXuat);
  }

  Future<void> _onDangNhap(NhanDangNhap event, Emitter<LoginTransformerState> emit) async {
    _soLanBam++;
    _soRequestThucSu++;
    final soNay = _soRequestThucSu;

    emit(LoginTransformerLoading(_soLanBam));
    print('[droppable] Bắt đầu xử lý request #$soNay (username: ${event.username})');

    try {
      final loginUseCase = getIt<LoginUseCase>(); // đúng use case thật của Tuần 1
      final (user, failure) = await loginUseCase(
        LoginParams(username: event.username, password: event.password),
      );
      if (failure != null) { emit(LoginTransformerError(failure.message)); return; }

      final authLocal = AuthLocalDataSource();
      await authLocal.saveAuthInfo(
        token: user!.token, username: user.username, role: user.role, userId: user.id,
      );

      print('[droppable] Request #$soNay hoàn thành');
      emit(LoginTransformerSuccess(user.username, soNay));
    } catch (e) {
      emit(LoginTransformerError(e.toString()));
    }
  }
}
```

**`_soLanBam` (số lần bấm) tách riêng khỏi `_soRequestThucSu` (số request thực sự xử
lý)** — đây chính là cách đo lường trực tiếp hiệu quả của `droppable()`: nếu bấm nút Login
5 lần liên tiếp, `_soLanBam` sẽ là 5, nhưng nhờ `droppable()`, `_onDangNhap` chỉ **thực sự
chạy đúng 1 lần** (`_soRequestThucSu` dừng ở 1) — 4 lần bấm còn lại bị Bloc âm thầm bỏ
qua vì Event đầu tiên vẫn đang xử lý.

**Vì sao dùng `getIt<LoginUseCase>()` (Clean Architecture, dependency injection) thay vì
gọi thẳng `ApiClient`?** Vì đây là ứng dụng transformer vào đúng luồng nghiệp vụ đăng nhập
thật của project — không phải bài học riêng lẻ, mà là ví dụ cho thấy `droppable()` bảo vệ
đúng nút bấm quan trọng nhất trong app (Login) khỏi bị submit trùng, dùng chính code sản
xuất thực tế chứ không phải bản rút gọn.

## Bài tập

**Bài 3 (đã có sẵn — `SearchBlocRestartable` trong `bai3_bai4_search_bloc.dart`):** Chạy
`SearchTransformerScreen`, chuyển sang tab "Bài 3: restartable()", gõ thật nhanh 1 từ dài
— xác nhận kết quả cuối cùng luôn khớp đúng từ khóa mới nhất, dù có nhiều request chạy
"chồng" nhau.

**Bài 4 (đã có sẵn — `SearchBlocSequential`):** Chuyển sang tab "Bài 4: sequential()", gõ
nhanh cùng 1 từ — quan sát hiện tượng kết quả bị "trễ" hoặc hiển thị sai từ khóa trong
khoảnh khắc, do phải xử lý tuần tự hết các request cũ trước.

**Bài 5 (đã có sẵn — `bai5_droppable_login_bloc.dart`):** Chạy `DroppableLoginScreen`, bấm
nút Login liên tục nhiều lần thật nhanh — xác nhận qua log/`_soRequestThucSu` chỉ có
**đúng 1** request thực sự được xử lý, dù đã bấm nhiều lần.

## Code Example (repo `flutter.git`)

> Xem nguyên các file tại branch `Bloc-Transformer`:
> [`bai3_bai4_search_bloc.dart`](https://github.com/SangLeSoftZ/flutter/blob/Bloc-Transformer/testflutter/lib/tuan3_ngay4/bai3_bai4_search_bloc.dart) ·
> [`bai3_bai4_search_screen.dart`](https://github.com/SangLeSoftZ/flutter/blob/Bloc-Transformer/testflutter/lib/tuan3_ngay4/bai3_bai4_search_screen.dart) ·
> [`bai5_droppable_login_bloc.dart`](https://github.com/SangLeSoftZ/flutter/blob/Bloc-Transformer/testflutter/lib/tuan3_ngay4/bai5_droppable_login_bloc.dart) ·
> [`bai5_droppable_login_screen.dart`](https://github.com/SangLeSoftZ/flutter/blob/Bloc-Transformer/testflutter/lib/tuan3_ngay4/bai5_droppable_login_screen.dart)

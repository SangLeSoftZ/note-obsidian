---
title: "Ngày 13(Tuần 3 - Ngày 2, Sáng) — Debounce cho ô tìm kiếm Task"
track: flutter
day: 13
topic: debounce-search-timer
tags: [flutter, stream, debounce, timer, cubit, search]
source_goc: Tuan3-Ngay2-Debounce-WebSocketClient-STOMPServer.md
---

# Ngày 13 (Tuần 3 - Ngày 2, Sáng) — Debounce cho ô tìm kiếm Task

## 1. Vấn đề thật: gọi API dồn dập khi gõ nhanh

Nếu ô tìm kiếm gọi API ngay mỗi khi `onChanged` bắn ra, gõ từ "học" (3 ký tự) tạo ra
**3 request** liên tiếp trong vài trăm mili-giây — hầu hết bị lãng phí (request cho "h",
"hơ" gần như vô nghĩa vì người dùng gõ tiếp ngay). Vấn đề nặng hơn: do độ trễ mạng không
đều, response của request gửi trước có thể **về sau** response gửi sau → hiển thị sai kết
quả. `debounce` giải quyết: chỉ thực sự gọi API khi người dùng đã **ngừng gõ** đủ lâu.

## 2. Lựa chọn thật trong repo: `Timer` tự viết, không dùng `rxdart`

Tài liệu lý thuyết đưa ra 2 cách (rxdart `debounceTime` hoặc tự viết `Timer`) — code thật
trong repo **chọn cách `Timer`**, đúng như gợi ý "ưu tiên Timer nếu chỉ cần debounce đơn
giản, không muốn thêm dependency". Code thật —
`lib/tuan3_ngay2/bai1_debounce_search_cubit.dart`:

```dart
class DebounceSearchCubit extends Cubit<DebounceSearchState> {
  final ApiClient _api;
  Timer? _debounceTimer;
  int _soLanGoiApi = 0; // đếm số lần thực sự gọi API — để demo trực quan trên UI

  static const Duration _debounceDuration = Duration(milliseconds: 400);

  DebounceSearchCubit(this._api) : super(DebounceSearchInitial());

  void onTuKhoaThayDoi(String tuKhoa) {
    // Hủy timer cũ nếu đang chạy (người dùng tiếp tục gõ)
    _debounceTimer?.cancel();

    if (tuKhoa.isEmpty) {
      emit(DebounceSearchInitial());
      return;
    }

    emit(DebounceSearchLoading());

    // Đặt timer mới — chỉ kích hoạt sau 400ms không có input mới
    _debounceTimer = Timer(_debounceDuration, () {
      _goiApi(tuKhoa);
    });
  }

  Future<void> _goiApi(String tuKhoa) async {
    _soLanGoiApi++;
    debugLog('API call #$_soLanGoiApi — tìm: "$tuKhoa"');
    try {
      final ketQua = await _api.timKiemTask(tuKhoa);
      emit(DebounceSearchLoaded(ketQua, tuKhoa, _soLanGoiApi));
    } catch (e) {
      emit(DebounceSearchError(e.toString()));
    }
  }

  @override
  Future<void> close() {
    // BẮT BUỘC cancel timer khi Cubit bị hủy — tránh memory leak
    _debounceTimer?.cancel();
    return super.close();
  }
}
```

**Cơ chế debounce nhìn từ code thật:** mỗi lần gõ, dòng đầu tiên trong
`onTuKhoaThayDoi()` luôn là `_debounceTimer?.cancel()` — hủy đồng hồ đếm ngược cũ (nếu
đang chạy), rồi đặt 1 đồng hồ **mới** 400ms. Chỉ khi hết đúng 400ms mà không có lần gõ nào
khác xảy ra (không ai gọi `cancel()` nữa), `Timer` mới thực sự kích hoạt và gọi
`_goiApi()`. Đây chính là bản chất "reset đồng hồ đếm ngược mỗi khi có giá trị mới" mà
tài liệu lý thuyết mô tả — chỉ khác là viết tay bằng `Timer` thay vì dùng operator
`debounceTime` có sẵn của `rxdart`.

**`_soLanGoiApi`** là chi tiết hay: không có trong ví dụ lý thuyết gốc, nhưng giúp
**đo được bằng số** hiệu quả debounce — so sánh trực tiếp "số ký tự đã gõ" với "số lần
thực sự gọi API" ngay trên UI, thay vì chỉ đọc log.

## 3. Vì sao `.close()` bắt buộc — đúng như tài liệu gốc cảnh báo

```dart
@override
Future<void> close() {
  _debounceTimer?.cancel();
  return super.close();
}
```

`Timer` đang chờ (chưa kích hoạt) vẫn giữ 1 tham chiếu tới callback của nó — nếu Cubit đã
bị `dispose`/`close()` (người dùng rời màn hình) nhưng `Timer` vẫn "sống" và sau đó kích
hoạt, nó sẽ cố gọi `_goiApi()` rồi `emit()` lên 1 Cubit **đã đóng** → lỗi runtime, hoặc
nhẹ hơn là rò rỉ bộ nhớ (tài nguyên `Timer` không được giải phóng đúng lúc). Gọi
`_debounceTimer?.cancel()` trong `close()` đảm bảo không còn callback nào "lơ lửng" sau
khi Cubit đã bị hủy.

## 4. Màn hình đo lường trực quan: ký tự gõ vs số lần gọi API

Code thật — `lib/tuan3_ngay2/bai1_debounce_search_screen.dart` — hiển thị 3 chỉ số song
song để tự kiểm chứng debounce hoạt động đúng, không cần mở log:

```dart
_StatChip(label: 'Ký tự đã gõ', value: '$_soKyTuDaGo', color: Colors.orange),
_StatChip(label: 'Lần gọi API', value: '$apiCount', color: Colors.green),
_StatChip(label: 'Tiết kiệm', value: '${_soKyTuDaGo - apiCount} request', color: Colors.blue),
```

Gõ càng nhanh và càng dài, số "Tiết kiệm" càng lớn — đây là cách trực quan nhất để thấy
lợi ích thực tế của debounce, thay vì chỉ tin vào lý thuyết.

## 5. Bài 2 — thử tắt debounce để thấy khác biệt

Code thật để sẵn 1 dòng comment hướng dẫn ngay trong `_debounceDuration`:

```dart
// Bài 2: đổi giá trị này về 0 để thấy gọi API dồn dập
static const Duration _debounceDuration = Duration(milliseconds: 400);
```

Đổi `400` → `0` khiến mỗi ký tự gõ gần như lập tức kích hoạt gọi API riêng — số "Lần gọi
API" sẽ tiến sát bằng số "Ký tự đã gõ", trực tiếp chứng minh vai trò của debounce.

## Bài tập

**Bài 1 (đã có sẵn — `bai1_debounce_search_cubit.dart` + `bai1_debounce_search_screen.dart`):**
Chạy màn hình, gõ nhanh liên tục 1 từ, xác nhận qua chỉ số "Lần gọi API" trên UI (hoặc log
`[Debounce] API call #...`) chỉ có đúng 1 lần gọi sau khi ngừng gõ.

**Bài 2 (nâng cao, không bắt buộc):** Đổi `_debounceDuration` về `Duration(milliseconds: 0)`
trong `bai1_debounce_search_cubit.dart`, so sánh số "Lần gọi API" trước/sau để tự cảm
nhận rõ khác biệt.

## Code Example (repo `flutter.git`)

> Xem nguyên các file tại branch `WebSocket-Client-Stomp`:
> [`bai1_debounce_search_cubit.dart`](https://github.com/SangLeSoftZ/flutter/blob/WebSocket-Client-Stomp/testflutter/lib/tuan3_ngay2/bai1_debounce_search_cubit.dart) ·
> [`bai1_debounce_search_screen.dart`](https://github.com/SangLeSoftZ/flutter/blob/WebSocket-Client-Stomp/testflutter/lib/tuan3_ngay2/bai1_debounce_search_screen.dart)

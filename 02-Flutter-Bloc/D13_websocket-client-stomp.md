---
title: "Ngày 13 (Tuần 3 - Ngày 2, Chiều) — WebSocket Client: web_socket_channel + STOMP"
track: flutter
day: 13
topic: websocket-client-stomp
tags: [flutter, websocket, web_socket_channel, stomp_dart_client, stream, sink]
source_goc: Tuan3-Ngay2-Debounce-WebSocketClient-STOMPServer.md
---

# Ngày 13 (Tuần 3 - Ngày 2, Chiều) — WebSocket Client: web_socket_channel + STOMP

## 1. Vì sao không dùng polling — WebSocket là kết nối 2 chiều giữ liên tục

Muốn có "thông báo real-time" (VD: có Task mới do người khác tạo), cách đơn giản nhất là
**polling** — gọi API lặp lại mỗi vài giây. Vấn đề: lãng phí tài nguyên (đa số lần hỏi
không có gì mới), độ trễ, tốn băng thông/pin. **WebSocket** mở 1 kết nối 2 chiều, giữ
liên tục — server có thể **chủ động đẩy** dữ liệu xuống client ngay khi có sự kiện.

| | HTTP thường | WebSocket |
|---|---|---|
| Vòng đời | Mỗi request là 1 "cuộc gọi" độc lập, ngắn hạn | Bắt tay 1 lần (handshake qua HTTP) → nâng cấp thành kết nối giữ nguyên |
| Ai chủ động gửi | Chỉ client hỏi, server chỉ trả lời | Cả 2 phía gửi dữ liệu qua lại bất cứ lúc nào |

## 2. Bài 3 — `channel.stream` (tai nghe) vs `channel.sink` (miệng nói)

Code thật — `lib/tuan3_ngay2/bai3_websocket_echo_screen.dart`, kết nối tới echo server
công khai để test không cần tự dựng server:

```dart
static const String _echoUrl = 'wss://echo.websocket.org';

Future<void> _ketNoi() async {
  _channel = WebSocketChannel.connect(Uri.parse(_echoUrl));

  // Lắng nghe message từ server qua stream — "tai nghe"
  _channel!.stream.listen(
    (message) {
      setState(() => _messages.add(_Message(noidung: message.toString(), tuServer: true)));
    },
    onError: (error) => setState(() { _trangThai = 'Lỗi: $error'; _daKetNoi = false; }),
    onDone: () => setState(() { _trangThai = 'Kết nối đã đóng'; _daKetNoi = false; }),
  );
}

void _guiMessage() {
  final text = _msgCtrl.text.trim();
  if (text.isEmpty || !_daKetNoi) return;
  // Gửi message lên server qua sink — "miệng nói"
  _channel!.sink.add(text);
  setState(() => _messages.add(_Message(noidung: text, tuServer: false)));
}

@override
void dispose() {
  _channel?.sink.close(); // BẮT BUỘC đóng kết nối khi rời màn hình
  super.dispose();
}
```

**Ghi nhớ:** `stream` = "tai nghe" (nhận từ server), `sink` = "miệng nói" (gửi lên
server) — nhầm giữa 2 khái niệm này là lỗi rất phổ biến khi mới học. Màn hình thật hiển
thị 2 loại message theo 2 phía (`Align.centerLeft` cho message từ server, `centerRight`
cho message tự gửi) — giống giao diện chat, giúp phân biệt trực quan `stream` vs `sink`
ngay khi nhìn UI.

**`onError`/`onDone`** là 2 callback quan trọng không có trong ví dụ lý thuyết gốc (ví dụ
gốc chỉ dùng `StreamBuilder` đơn giản): `onDone` được gọi khi kết nối bị đóng (do server
đóng, hoặc do gọi `sink.close()`), `onError` khi có lỗi mạng — cả 2 đều cần cập nhật lại
`_daKetNoi = false` để UI phản ánh đúng trạng thái thực tế, tránh tình trạng UI vẫn hiện
"Đã kết nối" trong khi kết nối đã chết.

## 3. Bài 4 — `StompService`: chuẩn bị kết nối STOMP tới Spring Boot

Code thật — `lib/tuan3_ngay2/bai4_stomp_service.dart`:

```dart
class StompService {
  StompClient? _stompClient;
  bool _daKetNoi = false;

  // Android Emulator dùng 10.0.2.2 thay cho localhost
  static const String _wsUrl = 'ws://10.0.2.2:8080/ws';

  OnTaskReceived? onTaskReceived;

  void ketNoi({OnTaskReceived? onTask}) {
    onTaskReceived = onTask;
    _stompClient = StompClient(
      config: StompConfig(
        url: _wsUrl,
        onConnect: _onConnect,
        onDisconnect: (frame) { _daKetNoi = false; },
        onWebSocketError: (error) { /* log lỗi */ },
        onStompError: (frame) { /* log lỗi STOMP riêng biệt */ },
        reconnectDelay: const Duration(seconds: 5), // tự động reconnect nếu mất kết nối
      ),
    );
    _stompClient!.activate();
  }

  void _onConnect(StompFrame frame) {
    _daKetNoi = true;
    _stompClient!.subscribe(
      destination: '/topic/tasks',
      callback: (frame) {
        final body = frame.body ?? '';
        onTaskReceived?.call(body); // báo ra ngoài qua callback, không tự update UI ở đây
      },
    );
  }

  void ngatKetNoi() => _stompClient?.deactivate();
}
```

3 chi tiết thực tế không có trong tài liệu lý thuyết gốc:

- **`reconnectDelay: Duration(seconds: 5)`** — tự động thử kết nối lại sau 5 giây nếu mất
  kết nối (VD mất mạng tạm thời, server restart) — không cần người dùng tự bấm kết nối lại.
- **`onStompError` tách riêng khỏi `onWebSocketError`** — lỗi ở tầng WebSocket (mất kết
  nối mạng) và lỗi ở tầng STOMP (VD sai định dạng frame, subscribe sai destination) là 2
  loại lỗi khác nhau, nên xử lý/log riêng để dễ debug hơn.
- **`OnTaskReceived` là callback (`typedef`), không tự cập nhật UI trong `StompService`**
  — đúng nguyên tắc "dumb service" giống "dumb widget" đã học Tuần 2 Ngày 4:
  `StompService` chỉ lo kết nối + nhận dữ liệu, còn màn hình (`StompDemoScreen`) mới
  quyết định nhận dữ liệu xong thì làm gì (parse JSON, cập nhật danh sách...).

## 4. Vì sao cần STOMP thay vì WebSocket thuần

WebSocket "trần" (như Bài 3) chỉ định nghĩa cách gửi/nhận byte qua lại — không có khái
niệm "kênh" (channel/topic), không có cơ chế "đăng ký nhận đúng loại tin". Nếu nhiều tính
năng real-time khác nhau (thông báo Task mới, cập nhật trạng thái, chat...) đều dùng
chung 1 kết nối WebSocket thuần, code phía client phải tự phân loại message thủ công rất
phức tạp. **STOMP** thêm khái niệm "destination" (`/topic/tasks`) giúp client chỉ
**subscribe đúng kênh** mình cần.

## 5. `StompDemoScreen` — parse JSON và xử lý khi Spring Boot chưa sẵn sàng

Code thật — `lib/tuan3_ngay2/bai4_stomp_demo_screen.dart`:

```dart
_stompService.ketNoi(
  onTask: (jsonBody) {
    if (mounted) {
      setState(() {
        try {
          final data = jsonDecode(jsonBody);
          _taskLogs.insert(0, '🆕 Task mới: ${data['tieuDe'] ?? jsonBody}');
        } catch (_) {
          _taskLogs.insert(0, '📨 Nhận: $jsonBody');
        }
      });
    }
  },
);
```

Có `try/catch` khi `jsonDecode` — phòng trường hợp server gửi dữ liệu không đúng định
dạng JSON mong đợi (fallback: vẫn hiển thị nguyên văn thay vì crash app). Comment trong
code cũng ghi rõ: chạy màn hình này **trước khi** Spring Boot có WebSocket endpoint sẽ
thấy "Lỗi kết nối" — đây là hành vi **đúng, không phải bug** — cần hoàn thành phần Tối
(Spring Boot) trước để test được trọn vẹn.

## Bài tập

**Bài 3 (đã có sẵn — `bai3_websocket_echo_screen.dart`):** Chạy màn hình, kết nối tới
`wss://echo.websocket.org`, gửi 1 message, xác nhận nhận lại đúng message đó (xuất hiện
2 lần trong danh sách — 1 lần "→ Gửi (sink)", 1 lần "← Server (stream)").

**Bài 4 (đã có sẵn — `bai4_stomp_service.dart` + `bai4_stomp_demo_screen.dart`):** Chạy
`StompDemoScreen` trước khi Spring Boot có WebSocket (phần Tối) — xác nhận thấy đúng
trạng thái lỗi kết nối, không phải app crash. Sau khi hoàn thành phần Tối, quay lại chạy
tiếp để thấy kết nối thành công.

##  Code Example (repo `flutter.git`)

> Xem nguyên các file tại branch `WebSocket-Client-Stomp`:
> [`bai3_websocket_echo_screen.dart`](https://github.com/SangLeSoftZ/flutter/blob/WebSocket-Client-Stomp/testflutter/lib/tuan3_ngay2/bai3_websocket_echo_screen.dart) ·
> [`bai4_stomp_service.dart`](https://github.com/SangLeSoftZ/flutter/blob/WebSocket-Client-Stomp/testflutter/lib/tuan3_ngay2/bai4_stomp_service.dart) ·
> [`bai4_stomp_demo_screen.dart`](https://github.com/SangLeSoftZ/flutter/blob/WebSocket-Client-Stomp/testflutter/lib/tuan3_ngay2/bai4_stomp_demo_screen.dart)

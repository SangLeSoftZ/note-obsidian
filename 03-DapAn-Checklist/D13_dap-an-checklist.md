---
title: "Ngày 13 / Tuần 3 - Ngày 2 — Đáp án gợi ý & Checklist hoàn thành"
track: chung
day: 13
topic: dap-an-checklist
tags: [dap-an, checklist]
source_goc: Tuan3-Ngay2-Debounce-WebSocketClient-STOMPServer.md
---

# Ngày 13 / Tuần 3 - Ngày 2 — Đáp án gợi ý & Checklist hoàn thành

## ĐÁP ÁN GỢI Ý

<details><summary>Bài 1 — Debounce 400ms cho ô tìm kiếm Task (xem D13_debounce-search-timer.md)</summary>

Đã có sẵn trong repo tại `lib/tuan3_ngay2/bai1_debounce_search_cubit.dart`:

```dart
void onTuKhoaThayDoi(String tuKhoa) {
  _debounceTimer?.cancel();
  if (tuKhoa.isEmpty) { emit(DebounceSearchInitial()); return; }
  emit(DebounceSearchLoading());
  _debounceTimer = Timer(_debounceDuration, () => _goiApi(tuKhoa));
}
```

Kiểm chứng: gõ nhanh liên tục 1 từ trong `DebounceSearchScreen`, quan sát chỉ số "Lần gọi
API" trên UI — chỉ tăng đúng 1 sau khi ngừng gõ, dù "Ký tự đã gõ" tăng nhiều lần.
</details>

<details><summary>Bài 2 — So sánh debounceTime 0ms vs 400ms (lý thuyết + tự thử, không bắt buộc)</summary>

Đổi trong `bai1_debounce_search_cubit.dart`:

```dart
static const Duration _debounceDuration = Duration(milliseconds: 0); // thử về 0
```

Kết quả: chỉ số "Lần gọi API" sẽ tiến sát bằng "Ký tự đã gõ" (gần như mỗi ký tự đều gọi
API riêng) — chứng minh trực tiếp vai trò giảm số lần gọi API của debounce khi đặt đúng
giá trị (400ms).
</details>

<details><summary>Bài 3 — Kết nối wss://echo.websocket.org, gửi/nhận message (xem D13_websocket-client-stomp.md)</summary>

Đã có sẵn trong repo tại `lib/tuan3_ngay2/bai3_websocket_echo_screen.dart`:

```dart
_channel = WebSocketChannel.connect(Uri.parse('wss://echo.websocket.org'));
_channel!.stream.listen((message) { /* hiển thị message nhận được */ });
_channel!.sink.add('Xin chao tu Flutter');
```

Kiểm chứng: bấm "Kết nối", gửi 1 message qua ô nhập — xác nhận thấy đúng message đó xuất
hiện lại trong danh sách (đánh dấu "← Server (stream)"), do server echo tự gửi lại nguyên
văn.
</details>

<details><summary>Bài 4 — StompService chuẩn bị kết nối (xem D13_websocket-client-stomp.md)</summary>

Đã có sẵn trong repo tại `lib/tuan3_ngay2/bai4_stomp_service.dart` — `url` đã cấu hình sẵn
cho Android Emulator:

```dart
static const String _wsUrl = 'ws://10.0.2.2:8080/ws';
```

Nếu test bằng thiết bị thật, đổi thành `ws://<IP_máy>:8080/ws`; nếu test bằng Web/iOS
Simulator, đổi thành `ws://localhost:8080/ws`. Trước khi hoàn thành phần Tối (Spring
Boot), chạy `StompDemoScreen` sẽ thấy "Lỗi kết nối" — đây là hành vi đúng, không phải bug.
</details>

<details><summary>Bài 5 — spring-boot-starter-websocket + WebSocketConfig (xem D13_websocket-server-stomp.md)</summary>

Đã có sẵn trong repo tại `config/WebSocketConfig.java`:

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws").setAllowedOriginPatterns("*");
        registry.addEndpoint("/ws-sockjs").setAllowedOriginPatterns("*").withSockJS();
    }
    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic", "/queue");
        registry.setApplicationDestinationPrefixes("/app");
    }
}
```

Kiểm chứng: chạy server, mở `http://localhost:8080/ws-test.html` trong trình duyệt, bấm
"Kết nối WebSocket" — xác nhận log hiện trạng thái kết nối thành công.
</details>

<details><summary>Bài 6 — TaskWebSocketNotifier broadcast qua /topic/tasks (xem D13_websocket-server-stomp.md)</summary>

Đã có sẵn trong repo tại `websocket/TaskWebSocketNotifier.java` — tái sử dụng đúng
`TaskCreatedEvent` từ Tuần 3 Ngày 1:

```java
@EventListener
@Async
public void khiTaskDuocTao(TaskCreatedEvent event) {
    Task task = event.getTask();
    messagingTemplate.convertAndSend("/topic/tasks", task);
}
```

Kiểm chứng: từ `ws-test.html`, bấm "Tạo Task (REST)" — xác nhận log hiện message nhận
được ngay lập tức từ `/topic/tasks`, chứa đúng Task vừa tạo.
</details>

<details><summary>Bài 7 — Nối trọn 3 buổi: Flutter nhận real-time từ Spring Boot</summary>

Chạy `StompDemoScreen` (Flutter, đã hoàn thành phần Tối trước đó), subscribe
`/topic/tasks`, sau đó tạo 1 Task mới qua `ws-test.html` hoặc Postman. Kiểm chứng: màn
hình Flutter tự cập nhật danh sách `_taskLogs` với dòng `🆕 Task mới: ...` **không cần
bấm refresh** — xác nhận trọn vẹn luồng real-time từ Debounce Search (Sáng) → WebSocket
Client (Chiều) → WebSocket Server (Tối).
</details>

---

## ✅ CHECKLIST HOÀN THÀNH

- [ ] Ô tìm kiếm Task chỉ gọi API 1 lần sau khi ngừng gõ 400ms, không dồn dập theo từng ký tự
- [ ] Giải thích được vì sao `Timer` debounce phải `cancel()` trong `close()` của Cubit để tránh memory leak
- [ ] Flutter kết nối được tới 1 WebSocket endpoint qua `web_socket_channel`, phân biệt đúng `stream` (nhận) vs `sink` (gửi)
- [ ] Hiểu vì sao cần STOMP thay vì WebSocket thuần khi có nhiều tính năng real-time
- [ ] Server STOMP chạy được, `/topic/tasks` broadcast đúng khi có Task mới
- [ ] Giải thích được vì sao dùng Spring Events (Ngày 1) làm cầu nối giữa `TaskServiceImpl` và `TaskWebSocketNotifier` thay vì gọi trực tiếp
- [ ] Biết vì sao `SecurityConfig` tạm đổi sang `anyRequest().permitAll()` khi thêm WebSocket, và đây là điểm cần siết lại trước khi lên production


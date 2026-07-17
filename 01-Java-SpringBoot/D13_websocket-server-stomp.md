---
title: "Ngày 13 (Tuần 3 - Ngày 2, Tối) — WebSocket Server bằng STOMP (Spring Boot)"
track: java-spring
day: 13
topic: websocket-server-stomp
tags: [spring-boot, websocket, stomp, simpmessagingtemplate, spring-events, async]
source_goc: Tuan3-Ngay2-Debounce-WebSocketClient-STOMPServer.md
---

# Ngày 13 (Tuần 3 - Ngày 2, Tối) — WebSocket Server bằng STOMP (Spring Boot)

## 1. Vấn đề: REST chỉ là request–response, không đẩy được dữ liệu chủ động

Toàn bộ API đã viết từ Tuần 1-2 đều theo mô hình: client hỏi → server trả lời → xong.
Không có cách nào để server **tự đẩy** dữ liệu xuống client khi có sự kiện xảy ra ở phía
server (VD: người dùng A tạo Task mới → người dùng B đang mở app cần **thấy ngay**).
`spring-boot-starter-websocket` kèm giao thức STOMP giải quyết đúng bài toán này.

## 2. `WebSocketConfig` — 2 endpoint song song, không chỉ 1 như lý thuyết gốc

Code thật — `config/WebSocketConfig.java`:

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        // Endpoint không có SockJS — dùng cho Postman và Flutter stomp_dart_client
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("*");

        // Endpoint có SockJS fallback — dùng cho browser test
        registry.addEndpoint("/ws-sockjs")
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        // /topic → SimpleBroker: server push tới client
        // /queue → cho private message (1 user cụ thể, chưa dùng hôm nay)
        registry.enableSimpleBroker("/topic", "/queue");

        // /app → prefix cho @MessageMapping (client gửi lên server)
        registry.setApplicationDestinationPrefixes("/app");
    }
}
```

**Khác biệt so với tài liệu lý thuyết gốc:** thay vì chỉ 1 endpoint `/ws` với
`.withSockJS()`, code thật tách thành **2 endpoint riêng**:
- `/ws` — không có SockJS, dùng cho Postman và `stomp_dart_client` (Flutter) — các client
  này hỗ trợ WebSocket thuần trực tiếp, không cần lớp fallback.
- `/ws-sockjs` — có `.withSockJS()`, dùng riêng cho test bằng trình duyệt (file
  `ws-test.html` dùng thư viện JS `sockjs-client` + `stompjs`).

Lý do tách: SockJS thêm 1 lớp giao thức phụ (đóng gói frame theo cách riêng) — nếu bắt
`stomp_dart_client` phải nói chuyện qua lớp này trong khi nó vốn hỗ trợ WebSocket thuần
tốt, chỉ thêm phức tạp không cần thiết. Tách endpoint giúp mỗi loại client dùng đúng kênh
phù hợp với khả năng thực tế của nó.

`enableSimpleBroker("/topic", "/queue")` — code thật đăng ký sẵn cả `/queue` (dành cho
tin nhắn riêng tới 1 user cụ thể) dù bài học hôm nay chỉ dùng `/topic` — chuẩn bị trước
cho tính năng mở rộng sau này (VD thông báo cá nhân) mà không cần sửa lại config.

## 3. `TaskWebSocketNotifier` — tái sử dụng đúng `TaskCreatedEvent` từ Ngày 1

Điểm quan trọng nhất: code thật **không tạo Event mới** (`TaskDaTaoEvent` như ví dụ lý
thuyết gốc gợi ý) — mà **dùng lại chính xác** `TaskCreatedEvent` đã viết ở Tuần 3 Ngày 1.
Code thật — `websocket/TaskWebSocketNotifier.java`:

```java
/**
 * Lắng nghe TaskCreatedEvent (đã có từ Tuần 3 Ngày 1)
 * → broadcast Task mới tới TẤT CẢ client đang subscribe /topic/tasks
 *
 * TaskService không biết class này tồn tại — đúng nguyên tắc tách rời.
 * Muốn bỏ real-time → xóa class này, TaskService không cần sửa.
 */
@Component
public class TaskWebSocketNotifier {

    private final SimpMessagingTemplate messagingTemplate;

    public TaskWebSocketNotifier(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    // @Async: không chặn luồng chính — WebSocket push chạy ngầm
    @EventListener
    @Async
    public void khiTaskDuocTao(TaskCreatedEvent event) {
        Task task = event.getTask();

        System.out.println("=== [WebSocket] Broadcast Task mới"
            + " | id=" + task.getId()
            + " | tiêu đề=" + task.getTieuDe()
            + " | tới /topic/tasks"
            + " | thread=" + Thread.currentThread().getName());

        // Gửi Task object → Spring tự serialize thành JSON
        messagingTemplate.convertAndSend("/topic/tasks", task);
    }
}
```

Đây là bằng chứng thực tế rõ ràng nhất cho nguyên tắc tách biệt trách nhiệm bằng Spring
Events: `TaskServiceImpl.create()` (viết từ Ngày 1) **không cần sửa lại một dòng nào** để
thêm tính năng real-time — chỉ cần thêm 1 `@Component` Listener mới
(`TaskWebSocketNotifier`) lắng nghe đúng `TaskCreatedEvent` đã tồn tại sẵn.

**`@Async` ở đây quan trọng vì sao?** Việc gửi message qua WebSocket (dù thường nhanh)
vẫn là 1 thao tác I/O — không nên chặn response của `POST /api/tasks` để chờ
`messagingTemplate.convertAndSend()` hoàn tất. Có `@Async`, việc broadcast chạy trên 1
thread riêng (`@EnableAsync` đã bật từ Ngày 1), API tạo Task trả response ngay sau khi lưu
DB + publish event, không chờ WebSocket push xong.

## 4. Thay đổi quan trọng trong `SecurityConfig` — cần biết để không nhầm là lỗi

Code thật có 1 thay đổi đáng chú ý ở `config/SecurityConfig.java` khi thêm WebSocket:

```java
// TRƯỚC (Ngày 2 Tuần 2):
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/auth/**", "/api/tasks/**", ...).permitAll()
    .anyRequest().authenticated()
)
.sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))

// SAU (thêm WebSocket, Tuần 3 Ngày 2):
.authorizeHttpRequests(auth -> auth
    .anyRequest().permitAll()  // permit all — JWT check thủ công trong controller
)
.cors(cors -> cors.disable())
```

**Đây là 1 sự đơn giản hoá tạm thời, không phải "đúng chuẩn production":** handshake của
STOMP/SockJS qua HTTP không dễ mang theo JWT header giống 1 request REST thông thường
(trình duyệt/thư viện STOMP có cách gửi header khác nhau tuỳ client) — để không mất thời
gian debug việc "tại sao WebSocket handshake bị 401" ngay trong buổi học, code thật tạm
thời mở `anyRequest().permitAll()` ở tầng URL, chấp nhận việc kiểm tra JWT thủ công trong
từng Controller nếu cần (comment ghi rõ điều này). `@PreAuthorize` ở tầng method (Tuần 2
Ngày 2) **vẫn hoạt động độc lập** với thay đổi này — vì nó không phụ thuộc vào
`authorizeHttpRequests`.

**Cần lưu ý khi làm dự án thật:** đây là điểm cần quay lại siết chặt hơn trước khi lên
production — ví dụ giới hạn lại `permitAll()` chỉ cho đúng endpoint `/ws`, `/ws-sockjs`,
giữ nguyên `authenticated()` cho các API REST khác, thay vì mở toàn bộ.

## 5. Test nhanh không cần chờ Flutter xong

Repo có sẵn `src/main/resources/static/ws-test.html` — trang HTML test bằng thư viện JS
`sockjs-client` + `stompjs`, kết nối tới `/ws-sockjs`, subscribe `/topic/tasks`, và có sẵn
form gọi `POST /api/tasks` ngay trong cùng trang để tự kiểm tra vòng lặp đầy đủ mà không
cần mở Postman riêng.

## 6. Luồng hoạt động đầy đủ (tổng kết cả 3 buổi)

```
1. Flutter (Chiều) mở kết nối WebSocket tới Spring Boot Server, subscribe "/topic/tasks"
2. Người dùng A gọi POST /api/tasks (REST bình thường) → TaskServiceImpl lưu Task
3. TaskServiceImpl phát ApplicationEvent (TaskCreatedEvent) — không gọi trực tiếp WebSocket
4. TaskWebSocketNotifier (listener, @Async) nhận Event → gọi messagingTemplate.convertAndSend("/topic/tasks", task)
5. Spring gửi message qua kết nối WebSocket tới TẤT CẢ client đang subscribe kênh đó
6. Flutter của người dùng B (đã subscribe từ Chiều) nhận được message ngay lập tức
7. Flutter cập nhật UI (thêm Task mới vào danh sách) — KHÔNG cần người dùng B tự bấm refresh
```

## Bài tập

**Bài 5 (đã có sẵn — `WebSocketConfig.java`):** Chạy server, mở `ws-test.html` trong
trình duyệt (`http://localhost:8080/ws-test.html`), bấm "Kết nối WebSocket" — xác nhận
log hiện trạng thái kết nối thành công.

**Bài 6 (đã có sẵn — `TaskWebSocketNotifier.java`):** Từ `ws-test.html`, bấm "Tạo Task
(REST)" — xác nhận log hiện message nhận được ngay lập tức từ `/topic/tasks`, chứa đúng
Task vừa tạo.

**Bài 7 (nối 3 buổi lại):** Chạy `StompDemoScreen` (Flutter, Chiều), subscribe
`/topic/tasks`, tạo 1 Task mới qua `ws-test.html` hoặc Postman — xác nhận Flutter app
**nhận được thông báo real-time** không cần refresh màn hình.

##  Code Example (repo `java.git`)

> Xem nguyên các file tại branch `WebSocket-Server`:
> [`WebSocketConfig.java`](https://github.com/SangLeSoftZ/java/blob/WebSocket-Server/src/main/java/com/example/demo/config/WebSocketConfig.java) ·
> [`TaskWebSocketNotifier.java`](https://github.com/SangLeSoftZ/java/blob/WebSocket-Server/src/main/java/com/example/demo/websocket/TaskWebSocketNotifier.java) ·
> [`SecurityConfig.java`](https://github.com/SangLeSoftZ/java/blob/WebSocket-Server/src/main/java/com/example/demo/config/SecurityConfig.java) ·
> [`ws-test.html`](https://github.com/SangLeSoftZ/java/blob/WebSocket-Server/src/main/resources/static/ws-test.html)


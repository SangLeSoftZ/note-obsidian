---
title: "Ngày 15 / Tuần 3 - Ngày 4 — Đáp án gợi ý & Checklist hoàn thành"
track: chung
day: 15
topic: dap-an-checklist
tags: [dap-an, checklist]
source_goc: Tuan3_Ngay4_TaiLieu.md
---

# Ngày 15 / Tuần 3 - Ngày 4 — Đáp án gợi ý & Checklist hoàn thành

## ĐÁP ÁN GỢI Ý

<details><summary>Bài 1 — MethodChannel lấy mức pin từ native (xem D15_platform-channel-methodchannel.md)</summary>

Đã có sẵn trong repo — `lib/tuan3_ngay4/bai1_platform_channel_screen.dart` +
`android/.../MainActivity.kt`:

```dart
class BatteryService {
  static const _channel = MethodChannel('com.example.testflutter/battery');
  Future<int> layMucPin() async => await _channel.invokeMethod('getBatteryLevel');
}
```

```kotlin
"getBatteryLevel" -> {
    val mucPin = layMucPin()
    if (mucPin != -1) result.success(mucPin)
    else result.error("UNAVAILABLE", "Không lấy được mức pin", null)
}
```

Kiểm chứng: chạy trên thiết bị/emulator Android thật, bấm "Mức pin (Bài 1)" — xác nhận
nhận đúng % pin thật.
</details>

<details><summary>Bài 2 — Gọi method không tồn tại, quan sát PlatformException (xem D15_platform-channel-methodchannel.md)</summary>

Đã có sẵn trong repo:

```dart
Future<String> goiMethodKhongTonTai() async {
  final String ketQua = await _channel.invokeMethod('methodKhongCoThatSu');
  return ketQua;
}
```

```kotlin
else -> result.notImplemented()
```

Kiểm chứng: bấm "Method sai (Bài 2)" — xác nhận nhận `PlatformException` với
`code: not_implemented`.
</details>

<details><summary>Bài 3 — restartable() cho Bloc tìm kiếm Task (xem D15_bloc-transformer-restartable-droppable.md)</summary>

Đã có sẵn trong repo — `lib/tuan3_ngay4/bai3_bai4_search_bloc.dart`:

```dart
class SearchBlocRestartable extends Bloc<SearchEvent, SearchState> {
  SearchBlocRestartable(this._api) : super(SearchInitial()) {
    on<TimKiemThayDoi>(_onTimKiem, transformer: restartable());
  }
}
```

Kiểm chứng: chạy `SearchTransformerScreen`, tab "Bài 3", gõ nhanh 1 từ dài — kết quả cuối
luôn khớp đúng từ khóa mới nhất.
</details>

<details><summary>Bài 4 — Đổi thành sequential(), quan sát kết quả sai lệch (xem D15_bloc-transformer-restartable-droppable.md)</summary>

Đã có sẵn trong repo — `SearchBlocSequential` trong cùng file:

```dart
class SearchBlocSequential extends Bloc<SearchEvent, SearchState> {
  SearchBlocSequential(this._api) : super(SearchInitial()) {
    on<TimKiemThayDoi>(_onTimKiem, transformer: sequential());
  }
}
```

Kiểm chứng: chuyển sang tab "Bài 4: sequential()", gõ nhanh — quan sát hiện tượng kết quả
hiển thị bị trễ/sai từ khóa trong khoảnh khắc, do phải xử lý tuần tự hết các request cũ.
</details>

<details><summary>Bài 5 — droppable() cho nút Đăng nhập (xem D15_bloc-transformer-restartable-droppable.md)</summary>

Đã có sẵn trong repo — `lib/tuan3_ngay4/bai5_droppable_login_bloc.dart`, dùng đúng
`LoginUseCase` thật từ Tuần 1:

```dart
LoginTransformerBloc() : super(LoginTransformerInitial()) {
  on<NhanDangNhap>(_onDangNhap, transformer: droppable());
}
```

Kiểm chứng: chạy `DroppableLoginScreen`, bấm Login liên tục nhiều lần thật nhanh — xác
nhận qua `_soRequestThucSu`/log chỉ đúng **1** request thực sự được xử lý.
</details>

<details><summary>Bài 6 — @ValidPassword đầy đủ, áp dụng vào RegisterRequest (xem D15_custom-validator-annotation.md)</summary>

Đã có sẵn trong repo — `validator/ValidPassword.java` + `ValidPasswordValidator.java`:

```java
@Constraint(validatedBy = ValidPasswordValidator.class)
public @interface ValidPassword {
    String message() default "Mật khẩu phải có ít nhất 8 ký tự, 1 chữ hoa, 1 chữ số và 1 ký tự đặc biệt (@#$%^&+=!)";
}
```

Kiểm chứng: `POST /api/auth/register` với mật khẩu yếu (VD `"123"`) — xác nhận nhận đúng
message lỗi chi tiết.
</details>

<details><summary>Bài 7 — @PasswordMatches ở cấp class (xem D15_custom-validator-annotation.md)</summary>

Đã có sẵn trong repo — `validator/PasswordMatches.java` + `PasswordMatchesValidator.java`,
áp dụng trên `RegisterRequest`:

```java
@PasswordMatches
public class RegisterRequest {
    @ValidPassword private String password;
    private String confirmPassword;
}
```

Kiểm chứng: gửi `password`/`confirmPassword` khác nhau — xác nhận bị từ chối, lỗi gắn
đúng vào field `confirmPassword` (nhờ `addPropertyNode("confirmPassword")`).
</details>

<details><summary>Bài 8 — Message lỗi linh động qua buildConstraintViolationWithTemplate (xem D15_custom-validator-annotation.md)</summary>

Đã có sẵn trong repo — trong `ValidPasswordValidator.isValid()`:

```java
if (!valid) {
    context.disableDefaultConstraintViolation();
    StringBuilder msg = new StringBuilder("Mật khẩu chưa đủ mạnh:");
    if (password.length() < 8) msg.append(" thiếu độ dài (min 8 ký tự);");
    if (!password.matches(".*[A-Z].*")) msg.append(" thiếu chữ hoa;");
    // ...
    context.buildConstraintViolationWithTemplate(msg.toString()).addConstraintViolation();
}
```

Kiểm chứng: thử mật khẩu chỉ thiếu đúng 1 điều kiện (VD `"Abcdefgh1"` — thiếu ký tự đặc
biệt) — xác nhận message chỉ liệt kê đúng điều kiện đang thiếu.
</details>

---

## ✅ CHECKLIST HOÀN THÀNH

- [ ] Gọi được ít nhất 1 hàm native qua Platform Channel, hiểu luồng giao tiếp 2 chiều
- [ ] Xử lý được lỗi khi gọi sai method (`PlatformException`, `code: not_implemented`) hoặc native báo lỗi (`result.error(...)`)
- [ ] Áp dụng đúng `restartable()` cho Bloc tìm kiếm, không còn kết quả cũ "đè" kết quả mới
- [ ] Tự quan sát được sự khác biệt thực tế giữa `restartable()` và `sequential()` khi gõ nhanh
- [ ] Phân biệt được khi nào dùng `restartable`/`droppable`/`sequential`/`concurrent`
- [ ] Viết được 1 Custom Validator Annotation hoàn chỉnh (annotation + Validator class)
- [ ] Hiểu vì sao validator kiểm tra chéo nhiều field phải đặt ở cấp class (`ElementType.TYPE`)
- [ ] Biết cách gắn lỗi vào đúng field cụ thể bằng `addPropertyNode(...)` thay vì lỗi chung chung ở object

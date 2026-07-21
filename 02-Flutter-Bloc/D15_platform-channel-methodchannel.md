---
title: "Ngày 15 (Tuần 3 - Ngày 4, Sáng) — Platform Channel: Gọi code native từ Flutter"
track: flutter
day: 15
topic: platform-channel-methodchannel
tags: [flutter, platform-channel, methodchannel, kotlin, android, platformexception]
source_goc: Tuan3_Ngay4_TaiLieu.md
---

# Ngày 15 (Tuần 3 - Ngày 4, Sáng) — Platform Channel: Gọi code native từ Flutter

## 1. Khi nào cần Platform Channel — luôn tìm package trước

Flutter cung cấp sẵn nhiều tính năng qua package, nhưng đôi khi cần **1 tính năng đặc thù
hệ điều hành** chưa có package nào hỗ trợ. Platform Channel là "cây cầu" cho Dart giao
tiếp qua lại với code native (Kotlin/Java Android, Swift/Objective-C iOS). **Lưu ý quan
trọng:** đây là giải pháp **cuối cùng** — ưu tiên tìm package có sẵn trên pub.dev trước.

## 2. `BatteryService` — tách riêng khỏi Widget, đúng nguyên tắc "dumb service"

Code thật — `lib/tuan3_ngay4/bai1_platform_channel_screen.dart`:

```dart
class BatteryService {
  // Tên kênh phải khớp chính xác với Kotlin (như hẹn tần số radio)
  static const _channel = MethodChannel('com.example.testflutter/battery');

  // Bài 1: gọi method có tồn tại bên native
  Future<int> layMucPin() async {
    final int mucPin = await _channel.invokeMethod('getBatteryLevel');
    return mucPin;
  }

  // Bài 2: gọi method KHÔNG tồn tại → PlatformException(not_implemented)
  Future<String> goiMethodKhongTonTai() async {
    final String ketQua = await _channel.invokeMethod('methodKhongCoThatSu');
    return ketQua;
  }

  Future<String> layThongTinThietBi() async {
    final String info = await _channel.invokeMethod('getDeviceInfo');
    return info;
  }
}
```

`MethodChannel('com.example.testflutter/battery')` — tên kênh phải **khớp chính xác**
giữa Dart và native. `BatteryService` được viết thành 1 class riêng (không đặt
`MethodChannel` thẳng trong `State`) — cùng tinh thần "dumb service" đã học ở
`StompService` (Tuần 3 Ngày 2): tách phần giao tiếp kỹ thuật ra khỏi widget, dễ tái sử
dụng và dễ test hơn.

## 3. Phía Android (Kotlin) — `MainActivity.kt` thật, có xử lý cả Android cũ

Code thật — `android/app/src/main/kotlin/com/example/testflutter/MainActivity.kt`:

```kotlin
class MainActivity : FlutterActivity() {
    private val CHANNEL = "com.example.testflutter/battery" // khớp CHÍNH XÁC với Dart

    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)
        MethodChannel(flutterEngine.dartExecutor.binaryMessenger, CHANNEL)
            .setMethodCallHandler { call, result ->
                when (call.method) {
                    "getBatteryLevel" -> {
                        val mucPin = layMucPin()
                        if (mucPin != -1) result.success(mucPin)
                        else result.error("UNAVAILABLE", "Không lấy được mức pin", null)
                    }
                    "getDeviceInfo" -> {
                        val info = "Thiết bị: ${Build.MANUFACTURER} ${Build.MODEL}\n" +
                                "Android: ${Build.VERSION.RELEASE} (API ${Build.VERSION.SDK_INT})"
                        result.success(info)
                    }
                    // Bài 2: method không có case nào khớp → notImplemented()
                    else -> result.notImplemented()
                }
            }
    }

    private fun layMucPin(): Int {
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            val batteryManager = getSystemService(Context.BATTERY_SERVICE) as BatteryManager
            batteryManager.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)
        } else {
            // Fallback cho Android cũ (trước Lollipop) — API BatteryManager mới chưa có
            val intent = registerReceiver(null, IntentFilter(Intent.ACTION_BATTERY_CHANGED))
            val level = intent?.getIntExtra(BatteryManager.EXTRA_LEVEL, -1) ?: -1
            val scale = intent?.getIntExtra(BatteryManager.EXTRA_SCALE, -1) ?: -1
            if (level == -1 || scale == -1) -1 else (level * 100 / scale.toFloat()).toInt()
        }
    }
}
```

**Chi tiết hay không có trong ví dụ lý thuyết gốc:** hàm `layMucPin()` xử lý **2 nhánh**
tùy phiên bản Android — từ Lollipop (API 21) trở lên dùng thẳng
`BatteryManager.BATTERY_PROPERTY_CAPACITY`; với Android cũ hơn phải "nghe" broadcast
`ACTION_BATTERY_CHANGED` rồi tự tính phần trăm bằng `level * 100 / scale`. Đây là ví dụ
thực tế cho việc code native đôi khi phức tạp hơn nhiều so với 1 dòng gọi API đơn giản,
dù phía Dart gọi `invokeMethod('getBatteryLevel')` chỉ là 1 dòng.

**Không cần thành thạo Kotlin** — điều quan trọng là nắm luồng giao tiếp 2 chiều: Dart
gọi tên method (`call.method`), native `when` để khớp đúng tên và trả kết quả qua
`result.success(...)`, hoặc `result.error(...)` nếu có lỗi xử lý được, hoặc rơi vào
`else -> result.notImplemented()` nếu tên method không khớp case nào.

## 4. Xử lý `PlatformException` — 3 trường quan trọng

Code thật xử lý đủ cả `PlatformException` lẫn lỗi không xác định:

```dart
try {
  ...
} on PlatformException catch (e) {
  // e.code    → mã lỗi (VD: "not_implemented", "UNAVAILABLE")
  // e.message → mô tả lỗi từ native
  // e.details → thông tin bổ sung (có thể null)
  final msg = 'PlatformException\ncode: ${e.code}\nmessage: ${e.message}';
} catch (e) {
  // Lỗi không xác định — không phải PlatformException
}
```

`PlatformException` tương tự cách phân loại `DioException` đã học Tuần 2 Ngày 4 — phân
loại đúng giúp xử lý UI phù hợp. Khi gọi `methodKhongCoThatSu` (Bài 2), native rơi vào
nhánh `else -> result.notImplemented()`, Flutter nhận `PlatformException` với
`e.code == 'not_implemented'` — đây chính là cách kiểm chứng bằng thực nghiệm, không chỉ
đọc lý thuyết.

Màn hình demo có sẵn 3 nút (mức pin, thiết bị, method sai) và khung log liệt kê từng lần
gọi kèm kết quả/lỗi — giúp so sánh trực quan phản hồi thành công vs `PlatformException`
ngay trên UI.

## Bài tập

**Bài 1 (đã có sẵn — `bai1_platform_channel_screen.dart` + `MainActivity.kt`):** Chạy
app trên **thiết bị/emulator Android thật** (Platform Channel không chạy được trên
Chrome/Web), bấm "Mức pin" — xác nhận nhận đúng % pin thật của thiết bị.

**Bài 2 (đã có sẵn):** Bấm "Method sai (Bài 2)" — gọi `methodKhongCoThatSu` không tồn tại
bên native — xác nhận nhận `PlatformException` với `code: not_implemented`.

## Code Example (repo `flutter.git`)

> Xem nguyên các file tại branch `Platform-Channel`:
> [`bai1_platform_channel_screen.dart`](https://github.com/SangLeSoftZ/flutter/blob/Platform-Channel/testflutter/lib/tuan3_ngay4/bai1_platform_channel_screen.dart) ·
> [`MainActivity.kt`](https://github.com/SangLeSoftZ/flutter/blob/Platform-Channel/testflutter/android/app/src/main/kotlin/com/example/testflutter/MainActivity.kt)


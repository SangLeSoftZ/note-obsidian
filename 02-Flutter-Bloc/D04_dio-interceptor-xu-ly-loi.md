---
title: "Ngày 4 — Dio: Cơ chế Interceptor & xử lý lỗi (DioException)"
track: flutter
day: 4
topic: dio-networking
tags: [flutter, dio, interceptor, dioexception, http]
source_goc: Ngay4_LyThuyetSau.md (Phần 2)
---

# Ngày 4 — Dio: Cơ chế Interceptor & xử lý lỗi (DioException)

# PHẦN 2 — DIO: CƠ CHẾ INTERCEPTOR & XỬ LÝ LỖI THỰC TẾ

### Interceptor hoạt động như "trạm kiểm soát" cho mọi request

```dart
_dio.interceptors.add(InterceptorsWrapper(
  onRequest: (options, handler) {
    print("Sắp gửi request tới: ${options.path}");
    options.headers['Authorization'] = 'Bearer $token'; // tự động gắn token vào MỌI request
    return handler.next(options); // cho phép request tiếp tục đi
  },
  onResponse: (response, handler) {
    print("Nhận response: ${response.statusCode}");
    return handler.next(response);
  },
  onError: (error, handler) {
    if (error.response?.statusCode == 401) {
      print("Token hết hạn, cần đăng nhập lại");
    }
    return handler.next(error);
  },
));
```

**Vì sao Interceptor quan trọng trong thực tế, không chỉ để "log cho vui"?** Vì nó cho phép bạn xử lý **1 lần duy nhất** những việc lẽ ra phải lặp lại ở **mọi** API call: gắn token xác thực, log debug, xử lý khi token hết hạn (tự động refresh token rồi gọi lại request cũ). Nếu không có Interceptor, bạn phải tự viết code gắn token vào từng hàm gọi API riêng lẻ — dễ quên sót, khó bảo trì khi có hàng chục API khác nhau.

### `DioException` — phân loại lỗi để xử lý đúng cách

```dart
try {
  final response = await _dio.get("/tasks");
} on DioException catch (e) {
  switch (e.type) {
    case DioExceptionType.connectionTimeout:
      print("Kết nối quá lâu, kiểm tra mạng");
      break;
    case DioExceptionType.badResponse:
      print("Server trả lỗi: ${e.response?.statusCode}");
      break;
    case DioExceptionType.connectionError:
      print("Không kết nối được server, kiểm tra địa chỉ IP/port");
      break;
    default:
      print("Lỗi khác: ${e.message}");
  }
}
```

**Vì sao cần phân loại thay vì bắt lỗi chung chung?** Vì cách xử lý UI cho từng loại lỗi nên khác nhau: mất mạng → hiện "Kiểm tra kết nối mạng"; server lỗi 500 → hiện "Hệ thống đang bảo trì"; timeout → có thể tự động thử lại. Bắt lỗi chung chung (`catch (e)`) khiến người dùng luôn thấy 1 thông báo mơ hồ như "Có lỗi xảy ra", không giúp họ biết nên làm gì tiếp theo.

### Bài tập đào sâu

**Bài C:** Thêm 1 Interceptor log toàn bộ request/response của `ApiClient` bạn đã viết ở Ngày 4. Cố tình tắt server Spring Boot, gọi API từ Flutter, quan sát chính xác `DioExceptionType` nào được trả về.

---

## 💻 Code Example (repo `Code`)

> ⚠️ Repo [`SangLeSoftZ/Code`](https://github.com/SangLeSoftZ/Code) **hiện chỉ chứa code Java/Spring Boot**
> (package `com.example.demo`), chưa có project Flutter/Dart nào. Khi có repo code Flutter
> riêng, bổ sung link + snippet vào đúng mục này theo cùng quy ước (xem `QUY_UOC_DAT_TEN.md`).

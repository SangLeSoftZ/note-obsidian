---
title: "Ngày 8 / Tuần 2 - Ngày 1 — Đáp án gợi ý & Checklist hoàn thành"
track: chung
day: 8
topic: dap-an-checklist
tags: [dap-an, checklist]
source_goc: Tuan2_Ngay1_TaiLieu.md
---

# Ngày 8 / Tuần 2 - Ngày 1 — Đáp án gợi ý & Checklist hoàn thành

## ĐÁP ÁN GỢI Ý

<details><summary>Bài 1 — AppBlocObserver cơ bản (xem D08_bloc-observer-logging.md)</summary>

```dart
class AppBlocObserver extends BlocObserver {
  @override
  void onChange(BlocBase bloc, Change change) {
    super.onChange(bloc, change);
    print('${bloc.runtimeType} doi state: ${change.currentState} -> ${change.nextState}');
  }
}

void main() {
  Bloc.observer = AppBlocObserver();
  runApp(MyApp());
}
```
</details>

<details><summary>Bài 3 — Lọc log chỉ Cubit chứa "Task"</summary>

```dart
@override
void onChange(BlocBase bloc, Change change) {
  super.onChange(bloc, change);
  if (bloc.runtimeType.toString().contains('Task')) {
    print('${bloc.runtimeType} doi state: ${change.currentState} -> ${change.nextState}');
  }
}
```
</details>

<details><summary>Bài 4 — ThemeCubit chuyển sang HydratedCubit (xem D08_hydrated-bloc-local-storage.md)</summary>

```dart
class ThemeCubit extends HydratedCubit<bool> {
  ThemeCubit() : super(false);
  void toggle() => emit(!state);

  @override
  bool fromJson(Map<String, dynamic> json) => json['darkMode'] as bool;

  @override
  Map<String, dynamic> toJson(bool state) => {'darkMode': state};
}
```
</details>

<details><summary>Bài 6 — API /api/auth/refresh (xem D08_refresh-token-jwt.md)</summary>

```java
@PostMapping("/refresh")
public ResponseEntity<?> refresh(@RequestBody Map<String, String> body) {
    String refreshToken = body.get("refreshToken");
    if (!jwtUtil.hopLe(refreshToken)) {
        return ResponseEntity.status(401).body("Refresh token khong hop le hoac da het han");
    }
    Claims claims = jwtUtil.layClaims(refreshToken);
    if (!"refresh".equals(claims.get("type"))) {
        return ResponseEntity.status(401).body("Token khong dung loai");
    }
    String accessTokenMoi = jwtUtil.taoAccessToken(claims.getSubject());
    return ResponseEntity.ok(Map.of("accessToken", accessTokenMoi));
}
```
</details>

---

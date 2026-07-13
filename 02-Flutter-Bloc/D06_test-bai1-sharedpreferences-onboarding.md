---
title: "Ngày 6 — Test bài 1: SharedPreferences Onboarding"
track: flutter
day: 6
topic: local-storage-sharedpreferences
tags: [flutter, sharedpreferences, onboarding, local-storage]
source_goc: TEST_BAI1_ONBOARDING.md
---

# Ngày 6 — Test bài 1: SharedPreferences Onboarding

# TEST BÀI 1 — SharedPreferences Onboarding

## Mục đích test
Hiểu SharedPreferences lưu **bool đơn giản**, dữ liệu **còn sau khi tắt app**.

---

## Chuẩn bị
Mở `lib/main.dart`, đảm bảo dòng này:
```dart
home: const AppStartup(),
```

---

## Kịch bản 1: Lần đầu mở app (chưa có dữ liệu)

**1. Chạy app:**
```bash
flutter run
```

**2. Kỳ vọng thấy:**
- Màn hình **màu tím** với chữ "Chào mừng!" (OnboardingScreen)
- Nút "Bắt đầu ngay!"

**3. Nhấn "Bắt đầu ngay!"**
- App ghi `SharedPreferences.setBool('onboarding_done', true)`
- Chuyển sang **HomeScreen** (màn trắng có icon nhà)

**🔍 Kiểm tra code:**
File: `lib/bai5_storage/bai1_onboarding/onboarding_screen.dart`
```dart
Future<void> _hoanThanhOnboarding(BuildContext context) async {
  final prefs = await SharedPreferences.getInstance();
  await prefs.setBool(kOnboardingDone, true); // ← GHI vào local storage
  // ...
}
```

---

## Kịch bản 2: Tắt app rồi mở lại (dữ liệu đã tồn tại)

**1. TẮT APP hoàn toàn** (không phải hot reload):
- Nhấn Stop trong VS Code/Android Studio
- Hoặc vuốt tắt app trên thiết bị

**2. MỞ LẠI APP:**
```bash
flutter run
```

**3. Kỳ vọng thấy:**
- App **KHÔNG hiện** OnboardingScreen màu tím nữa
- Vào thẳng **HomeScreen** (màn trắng có icon nhà)

**🔍 Kiểm tra code:**
File: `lib/bai5_storage/bai1_onboarding/app_startup.dart`
```dart
Future<void> _kiemTraOnboarding() async {
  final prefs = await SharedPreferences.getInstance();
  final done = prefs.getBool(kOnboardingDone) ?? false; // ← ĐỌC từ local storage
  setState(() => _onboardingDone = done);
}
```

**✅ KẾT LUẬN:** SharedPreferences đã lưu `bool` xuống ổ đĩa, còn nguyên sau khi tắt app.

---

## Kịch bản 3: Reset để test lại

**1. Trong HomeScreen, nhấn nút 🔄 ở góc trên phải**

**2. Thấy SnackBar:** "Đã reset! Đóng và mở lại app..."

**3. Tắt app → mở lại → sẽ thấy OnboardingScreen màu tím lại**

**🔍 Kiểm tra code:**
File: `lib/bai5_storage/bai1_onboarding/home_screen.dart`
```dart
Future<void> _resetOnboarding(BuildContext context) async {
  final prefs = await SharedPreferences.getInstance();
  await prefs.setBool(kOnboardingDone, false); // ← GHI lại false
}
```

---

## So sánh với State Management (Bloc/Cubit)

**Thử nghiệm phản chứng:**

Nếu dùng Cubit thay SharedPreferences:
```dart
// Giả sử dùng Cubit
class OnboardingCubit extends Cubit<bool> {
  OnboardingCubit() : super(false);
  void markDone() => emit(true);
}
```

→ Nhấn "Bắt đầu" → `emit(true)` (chỉ lưu trong RAM)
→ Tắt app → mở lại → **state mất** → lại hiện OnboardingScreen

**SharedPreferences khác ở chỗ:**
→ Nhấn "Bắt đầu" → `setBool(true)` (ghi xuống ổ đĩa)
→ Tắt app → mở lại → **đọc lại được** → bỏ qua OnboardingScreen

---

## Tóm tắt

| | State Management | SharedPreferences |
|---|---|---|
| Tắt app | ❌ Mất | ✅ Còn |
| Dùng cho | UI state tạm | Cài đặt lâu dài |
| Ví dụ | Loading, counter | Onboarding, theme |

## 💻 Code Example (repo `Code`)

> ⚠️ Repo [`SangLeSoftZ/Code`](https://github.com/SangLeSoftZ/Code) **hiện chỉ chứa code Java/Spring Boot**
> (package `com.example.demo`), chưa có project Flutter/Dart nào. Khi có repo code Flutter
> riêng, bổ sung link + snippet vào đúng mục này theo cùng quy ước (xem `QUY_UOC_DAT_TEN.md`).

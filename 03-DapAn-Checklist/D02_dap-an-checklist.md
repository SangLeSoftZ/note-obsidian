---
title: "Ngày 2 — Đáp án gợi ý & Checklist hoàn thành"
track: chung
day: 2
topic: dap-an-checklist
tags: [dap-an, checklist]
source_goc: Ngay2_OnTapChiTiet.md
---

# Ngày 2 — Đáp án gợi ý & Checklist hoàn thành

## ĐÁP ÁN GỢI Ý

<details><summary>Bài 1 — NhanVien</summary>

```java
abstract class NhanVien {
    String ten; double luongCoBan;
    NhanVien(String ten, double luongCoBan) { this.ten = ten; this.luongCoBan = luongCoBan; }
    abstract double tinhLuong();
}
class NhanVienChinhThuc extends NhanVien {
    NhanVienChinhThuc(String ten, double luongCoBan) { super(ten, luongCoBan); }
    double tinhLuong() { return luongCoBan + 2_000_000; }
}
class NhanVienThuViec extends NhanVien {
    NhanVienThuViec(String ten, double luongCoBan) { super(ten, luongCoBan); }
    double tinhLuong() { return luongCoBan * 0.85; }
}
```
</details>

<details><summary>Bài 2 — Set loại trùng</summary>

```java
List<String> hoTen = List.of("An", "Binh", "An", "Chi", "Binh");
Set<String> khongTrung = new HashSet<>(hoTen);
System.out.println(khongTrung);
```
</details>

<details><summary>Bài 3 — Stream tuổi trung bình</summary>

```java
List<Integer> tuoi = List.of(15, 22, 17, 30, 16, 45);
double tb = tuoi.stream()
        .filter(t -> t >= 18)
        .mapToInt(t -> t)
        .average()
        .orElse(0);
System.out.println(tb);
```
</details>

<details><summary>Bài 4 — DI Kho/DonHang</summary>

```java
interface KhoService { boolean kiemTraTonKho(String maSanPham); }

@Service
class KhoServiceImpl implements KhoService {
    public boolean kiemTraTonKho(String maSanPham) { return true; }
}

@Service
class DonHangService {
    private final KhoService khoService;
    public DonHangService(KhoService khoService) { this.khoService = khoService; }
    public void datHang(String maSanPham) {
        System.out.println(khoService.kiemTraTonKho(maSanPham) ? "Đặt hàng thành công" : "Hết hàng");
    }
}
```
</details>

<details><summary>Bài 5 — application.yml</summary>

```yaml
server:
  port: 9090
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/shop_db
    username: postgres
    password: 123456
  jpa:
    show-sql: true
```
</details>

<details><summary>Bài 7 — LoginCubit</summary>

```dart
abstract class LoginState {}
class LoginInitial extends LoginState {}
class LoginLoading extends LoginState {}
class LoginSuccess extends LoginState {}
class LoginFailure extends LoginState {
  final String loi;
  LoginFailure(this.loi);
}

class LoginCubit extends Cubit<LoginState> {
  LoginCubit() : super(LoginInitial());

  Future<void> dangNhap(String user, String pass) async {
    emit(LoginLoading());
    await Future.delayed(Duration(seconds: 1));
    if (user == "admin" && pass == "123") {
      emit(LoginSuccess());
    } else {
      emit(LoginFailure("Sai tài khoản"));
    }
  }
}
```
</details>

<details><summary>Bài 8 — Note CRUD (cấu trúc gợi ý)</summary>

```java
public class NoteDTO { private String noiDung; /* getter, setter */ }

public class Note {
    private Long id; private String noiDung;
    // constructor, getter, setter
}

@RestController
@RequestMapping("/api/notes")
public class NoteController {
    private final List<Note> danhSach = new ArrayList<>();
    private Long dem = 1L;

    @GetMapping
    public List<Note> layTatCa() { return danhSach; }

    @PostMapping
    public Note taoMoi(@RequestBody NoteDTO dto) {
        Note n = new Note(dem++, dto.getNoiDung());
        danhSach.add(n);
        return n;
    }

    @PutMapping("/{id}")
    public Note capNhat(@PathVariable Long id, @RequestBody NoteDTO dto) {
        for (Note n : danhSach) {
            if (n.getId().equals(id)) { n.setNoiDung(dto.getNoiDung()); return n; }
        }
        throw new RuntimeException("Không tìm thấy note");
    }

    @DeleteMapping("/{id}")
    public void xoa(@PathVariable Long id) { danhSach.removeIf(n -> n.getId().equals(id)); }
}
```
</details>

---

## CHECKLIST HOÀN THÀNH NGÀY 2

- [ ] Giải thích được 4 tính chất OOP bằng ví dụ của riêng bạn (không nhìn tài liệu)
- [ ] Làm xong Bài 1–3 (Java core)
- [ ] Vẽ lại được sơ đồ IoC Container hoạt động thế nào (nói miệng, không cần viết)
- [ ] Làm xong Bài 4–5 (Spring Core)
- [ ] Giải thích được khi nào dùng Bloc, khi nào dùng Cubit
- [ ] Làm xong Bài 6–8 (Bloc/Cubit + CRUD), test đủ 4 API bằng Postman cho bài Note

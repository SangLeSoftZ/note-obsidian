---
title: "Ngày 2 (Sáng) — Java Core: OOP, Collection, Stream, Lambda"
track: java-spring
day: 2
topic: java-core
tags: [java, oop, collection, stream, lambda]
source_goc: Ngay2_OnTapChiTiet.md (buổi sáng)
---

# Ngày 2 (Sáng) — Java Core: OOP, Collection, Stream, Lambda

# BUỔI SÁNG — JAVA CORE (OOP, Collection, Stream, Lambda)

## 1. OOP — 4 tính chất cốt lõi

### a) Tính đóng gói (Encapsulation)
Giấu dữ liệu bên trong class, chỉ cho truy cập qua getter/setter.

```java
public class TaiKhoan {
    private double soDu; // private = giấu, không cho truy cập trực tiếp từ ngoài

    public double getSoDu() { return soDu; }
    public void napTien(double soTien) {
        if (soTien > 0) soDu += soTien; // kiểm soát được logic khi thay đổi dữ liệu
    }
}
```

### b) Tính kế thừa (Inheritance)
```java
class PhuongTien {
    protected String bienSo;
    void diChuyen() { System.out.println(bienSo + " đang di chuyển"); }
}

class OTo extends PhuongTien {
    int soCho;
    // OTo tự động có bienSo và diChuyen() từ PhuongTien
}
```

### c) Tính đa hình (Polymorphism)
Cùng 1 method nhưng hành vi khác nhau tùy vào class con.

```java
class PhuongTien {
    void diChuyen() { System.out.println("Di chuyển..."); }
}
class Thuyen extends PhuongTien {
    @Override
    void diChuyen() { System.out.println("Thuyền đang chèo trên sông"); }
}

void chay(PhuongTien pt) {
    pt.diChuyen(); // gọi đúng phiên bản override tùy vào object thực tế truyền vào
}
```

### d) Tính trừu tượng (Abstraction) — Abstract class vs Interface

```java
abstract class HinhHoc {
    abstract double dienTich();      // bắt buộc class con phải viết
    void inThongTin() {              // có thể có method đã cài đặt sẵn
        System.out.println("Diện tích: " + dienTich());
    }
}

interface CoTheVe {
    void ve();  // interface: 100% là "hợp đồng", không chứa logic (trừ default method)
}
```

**Khi nào dùng abstract class, khi nào dùng interface?**
- Abstract class: các class con có chung 1 số logic/thuộc tính (VD: `PhuongTien` có sẵn `bienSo`)
- Interface: chỉ cần cam kết "phải có method này", không quan tâm class đó là gì (1 class có thể implement nhiều interface nhưng chỉ extends 1 class)

## 2. Collection — Khi nào dùng List, Set, Map?

| Loại | Đặc điểm | Dùng khi nào |
|---|---|---|
| `List` | Có thứ tự, cho phép trùng | Danh sách sản phẩm, danh sách học sinh theo thứ tự nhập |
| `Set` | Không trùng, (thường) không thứ tự | Danh sách email duy nhất, tag không lặp |
| `Map<K,V>` | Cặp khóa-giá trị | Tra cứu nhanh theo id, đếm số lần xuất hiện |

```java
List<String> ds = new ArrayList<>();
Set<String> email = new HashSet<>();
Map<Integer, String> hocSinhTheoId = new HashMap<>();
```

## 3. Lambda & Stream API

Lambda = 1 hàm viết gọn không cần đặt tên, hay dùng làm tham số truyền vào Stream.

```java
// Cú pháp: (tham số) -> biểu thức
Comparator<String> soSanhDoDai = (a, b) -> a.length() - b.length();
```

Stream API — các method hay dùng nhất:

```java
List<Integer> so = List.of(5, 2, 8, 1, 9, 3);

so.stream().filter(x -> x > 3).forEach(System.out::println);      // lọc
List<Integer> gapDoi = so.stream().map(x -> x * 2).toList();      // biến đổi
int tong = so.stream().mapToInt(x -> x).sum();                     // tổng
List<Integer> sapXep = so.stream().sorted().toList();              // sắp xếp
boolean coSoLon = so.stream().anyMatch(x -> x > 8);                // kiểm tra tồn tại
```

**Ghi nhớ nhanh:** `filter` = lọc, `map` = biến đổi từng phần tử, `sorted` = sắp xếp, `collect`/`toList` = gom kết quả lại thành List.

## Bài tập buổi sáng

**Bài 1 (OOP):** Tạo abstract class `NhanVien` có `ten`, `luongCoBan`, và abstract method `tinhLuong()`. Viết 2 class con `NhanVienChinhThuc` (lương = luongCoBan + 2 triệu thưởng) và `NhanVienThuViec` (lương = luongCoBan * 0.85).

**Bài 2 (Collection):** Có `List<String> hoTen = ["An", "Binh", "An", "Chi", "Binh"]`. Dùng `Set` để lấy ra danh sách tên **không trùng lặp**.

**Bài 3 (Stream + Lambda):** Có `List<Integer> tuoi = [15, 22, 17, 30, 16, 45]`. Dùng Stream lọc ra những người từ 18 tuổi trở lên, rồi tính tuổi trung bình của nhóm đó (gợi ý: `mapToInt(...).average()`).

---

## 💻 Code Example (repo `Code`)

> Nguồn: [`SangLeSoftZ/Code`](https://github.com/SangLeSoftZ/Code)

**Abstract class + kế thừa (OOP)** — [`bai1/NhanVien.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/bai1/NhanVien.java) (xem thêm `NhanVienChinhThuc.java`, `NhanVienThuViec.java` cùng thư mục để thấy override `tinhLuong()`):

```java
package com.example.demo.bai1;

// Abstract class đại diện cho nhân viên
public abstract class NhanVien {

    protected String ten;
    protected double luongCoBan;

    public NhanVien(String ten, double luongCoBan) {
        this.ten = ten;
        this.luongCoBan = luongCoBan;
    }

    // Abstract method - mỗi loại nhân viên tự tính lương theo cách riêng
    public abstract double tinhLuong();

    public void inThongTin() {
        System.out.printf("%-20s | Lương: %,.0f VNĐ%n", ten, tinhLuong());
    }
}
```

**Stream API — filter + average** — [`bai3/TinhTuoiTrungBinh.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/bai3/TinhTuoiTrungBinh.java):

```java
package com.example.demo.bai3;

import java.util.List;

public class TinhTuoiTrungBinh {

    public static void main(String[] args) {
        List<Integer> tuoi = List.of(15, 22, 17, 30, 16, 45);

        // Lọc từ 18 tuổi trở lên, rồi tính trung bình
        double tuoiTrungBinh = tuoi.stream()
                .filter(t -> t >= 18)           // Chỉ lấy tuổi >= 18
                .mapToInt(Integer::intValue)    // Chuyển sang IntStream
                .average()                       // Tính trung bình
                .orElse(0.0);                   // Nếu không có ai, trả về 0

        System.out.println("Danh sách tuổi: " + tuoi);
        System.out.printf("Tuổi trung bình (>= 18): %.1f%n", tuoiTrungBinh);
    }
}
```

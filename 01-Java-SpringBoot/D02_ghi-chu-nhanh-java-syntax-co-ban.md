---
title: "Ghi chú nhanh — Java cú pháp cơ bản (Buổi 2 cá nhân)"
track: java-spring
day: 2
topic: java-basic-syntax
tags: [java, basic, io, casting]
source_goc: baocaobuoi2.md
---

# Ghi chú nhanh — Java cú pháp cơ bản (Buổi 2 cá nhân)

#Java
+Biết được các cách cài đặt định nghĩa  cấu trúc cơ và các câu lệnh cơ bản:
 + Xuất dữ liệu : system.out.pritnl('');
 + Nhập dữ liệu : Scanner (ten bien) = new Scanner(system.in) # tạo một Scanner để lưu dữ liệu cần nhập (tùy vào kiểu dữ liệu  có thể ghi sẽ có cách luu riêng )
+Các cách có thể ghi chú trong java
 Cách ép kiểu để chuyển đổi kiểu dữ liệu
3 dạng chuyển đổi kiểu dữ liệu
  + Chuyển đổi cho các kiểu dữ liệu cơ bản
         (New Type) value;

  + Chuyển đổi kiểu cho các đối tượng: các lớp chuyển đổi phải kế thừa nhau
        (New Class) object;

 +  Chuyển đổi cho các kiểu dữ liệu cơ bản sang đối tượng và ngược lại:
    Chỉ chuyển đổi giữa các đối tượng bó sẵn trong gói java.lang tương ứng với các dữ liệu nguyên thủy

       int intObject = new Integer(32);
+ Biết được if, else , swich...case, for trong java 
#fluter
Tìm hiểu thêm về State là tình trạng của các widget cũng envent là các sự kiện để dẫn đến state của một trang hình cubit là một bloc vì nó không cần qua envent Các trạng thái là kết quả đầu ra của một hàm Cubit và đại diện cho một phần trạng thái của ứng dụng. 
Các thành phần giao diện người dùng có thể được thông báo về các trạng thái và vẽ lại các phần của chính chúng dựa trên trạng thái hiện tại.
Cũng như một số cách tạo và các đối tượng cubit biết được nếu muốn thay đổi trạng thái trong cubit có thể sủ dụng emit

## 💻 Code Example (repo `Code`)

> Xem ví dụ I/O + Scanner cơ bản áp dụng trong các bài tập nhỏ ở [`bai2/LocTenKhongTrung.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/bai2/LocTenKhongTrung.java) (dùng `List`, `Set`) — cùng track với `D02_java-core-oop-collection-stream.md`.

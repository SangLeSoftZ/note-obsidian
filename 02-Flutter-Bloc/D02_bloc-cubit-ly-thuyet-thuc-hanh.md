---
title: "Buổi 3 (cá nhân) — BLoC: Lý thuyết & thực hành cơ bản"
track: flutter
day: 2
topic: bloc-basics
tags: [flutter, bloc, cubit, blocbuilder, blocprovider]
source_goc: BaoCaoBuoi3.md
---

# Buổi 3 (cá nhân) — BLoC: Lý thuyết & thực hành cơ bản

# Bloc
Bloc dùng để tách lớp nghiệp vụ và giao diện một cách riên biệt  giúp dễ dàng sử dụng xử lí 
các state một cách gọn gàng và có thể mở rộng thông qua các event truyền đến bloc sau đó trả về state tương ưng 
# Học được cách tạo bloc bằng cách:
 1. Tạo state của màn hình dùng bien bằng cách dùng các biến và các constructor cần thiết
 2. Tạo event cần dùng  
 3. Tạo Bloc đưa các event và state vào hàm bloc sau đó gán state ban đầu
    tiêp đến thông qua các envent cập nhật lại các state
  
# Biết được các hàm được flutter hổ trợ như 
BlocBuilder dùng để xây dựng Ui người dùng khi state thay đổi
BlocSelector: Dùng giống BlocBuilder nhưng chọn lọc khi đúng điều kiện 
	      thì mới xây dựng UI người dùng
BlocProvider: cung cấp một bloc cho các widget con của nó 
bloc builder:  không cần phải tham truyền số của bloc  
MultiBlocProvider: dùng để làm nhiều bloc trên cùng một widget
                   mà ko phải lồng vào nhau
BlocListener : lắng nghe sự thay đổi trạng thái của bloc để thực
               thi phản hồi lại những thay đổi đó 
MultiBlocListener: gộp nhiều blocListener thành một 
BlocConsumer: xây dựng lại giao diện người dùng
              khi muốn kết hợp BlocBuilder và BlocListener.
Repository Provider: tạo repository (lưu trữ các thứ dã được lấy từ API)cho các lớp 
MultiRepository Provider: làm nhiều repository không phải lồng
                         vào nhau có thể làm trên một danh sách 
#Cubit
Là tập hợp con của bloc không phụ thuộc vào các event 
mà sẽ dùng các phương thức,khác khác để có các state thay đổi tương ứng 
cho các giao diện của người dùng
(có nghĩa chỉ hiện state thay đổi chứ không biết dược do hành động nào)
#Cách dùng cubit:
 1. Tạo state của màn hình dùng bien bằng cách dùng các biến và các constructor cần thiết
 2. Tạo event cần dùng 
 3. Tạo Cubit đưa các và state vào CuBit  sau đó gán state ban đầu
    tiêp đến thông qua các hàm được tạo để thay dổi các state  
#Search / Filter ListView tên Cubit
1. Tạo các state để all danh sach và sau khi đã lọc viết các hàm để có được các cubit chưa các state đã thay đổi
2. Tạo Bloc Provider cho cubit này 
3. Tạo Bloc Builder đê chạy khi không có viết gì  và lọc để có các  state tương ứng
4. điểu khiển khi chổ searh có tên hợp với điều kiện thì hiện danh sách đã lọc
# Cùng với đó có tìm hiểu thêm vê Image Picker, Add/Remove Dynamic Items From ListView ,|nternet Connectivity Check - Continuous Listen for Connection

## 💻 Code Example (repo `Code`)

> ⚠️ Repo [`SangLeSoftZ/Code`](https://github.com/SangLeSoftZ/Code) **hiện chỉ chứa code Java/Spring Boot**
> (package `com.example.demo`), chưa có project Flutter/Dart nào. Khi có repo code Flutter
> riêng, bổ sung link + snippet vào đúng mục này theo cùng quy ước (xem `QUY_UOC_DAT_TEN.md`).

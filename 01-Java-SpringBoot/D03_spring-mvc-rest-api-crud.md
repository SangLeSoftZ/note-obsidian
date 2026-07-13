---
title: "Ngày 2 (Tối, phần Spring) — Bắt đầu REST API CRUD: Controller + DTO cho Task"
track: java-spring
day: 3
topic: rest-api-crud
tags: [spring-mvc, controller, dto, rest, crud]
source_goc: Ngay2_OnTapChiTiet.md (buổi tối - phần API, không gồm phần BLoC)
---

# Ngày 2 (Tối, phần Spring) — Bắt đầu REST API CRUD: Controller + DTO cho Task

# BUỔI TỐI — LÝ THUYẾT BLoC/CUBIT + BẮT ĐẦU API CRUD

## 1. Bloc vs Cubit — khác nhau ở đâu?

Cubit là **bản đơn giản hóa** của Bloc. Bloc thêm 1 lớp trung gian là **Event**, giúp code rõ ràng hơn khi logic phức tạp (nhiều nguồn gọi cùng 1 hành động).

```
CUBIT:   UI gọi hàm trực tiếp  --->  Cubit xử lý  --->  emit(State mới)

BLOC:    UI add(Event)  --->  Bloc nhận Event  --->  xử lý  --->  emit(State mới)
```

Ví dụ cùng 1 chức năng "tăng đếm":

```dart
// CUBIT — gọi hàm trực tiếp
class CounterCubit extends Cubit<int> {
  CounterCubit() : super(0);
  void tang() => emit(state + 1);
}
// UI gọi: context.read<CounterCubit>().tang();

// BLOC — phải định nghĩa Event
abstract class CounterEvent {}
class TangEvent extends CounterEvent {}

class CounterBloc extends Bloc<CounterEvent, int> {
  CounterBloc() : super(0) {
    on<TangEvent>((event, emit) => emit(state + 1));
  }
}
// UI gọi: context.read<CounterBloc>().add(TangEvent());
```

**Khi nào chọn cái nào:** Cubit cho tính năng đơn giản (form, toggle, counter). Bloc cho tính năng phức tạp cần log lại lịch sử Event, hoặc nhiều loại hành động tác động lên cùng 1 State (VD: giỏ hàng có Event ThemSanPham, XoaSanPham, ApDungMaGiamGia...).

## 2. State nên thiết kế thế nào?

Thay vì chỉ 1 kiểu dữ liệu đơn giản (`int`, `String`), thực tế nên có 1 class `State` đại diện đủ trạng thái: đang tải / thành công / lỗi.

```dart
abstract class ProductState {}
class ProductLoading extends ProductState {}
class ProductLoaded extends ProductState {
  final List<String> danhSach;
  ProductLoaded(this.danhSach);
}
class ProductError extends ProductState {
  final String message;
  ProductError(this.message);
}
```

UI sẽ dùng `if (state is ProductLoading) ... else if (state is ProductLoaded) ...` để hiển thị đúng giao diện cho từng trạng thái — đây chính là mô hình sẽ dùng khi kết nối API thật ở Ngày 4.

## 3. Bắt đầu API CRUD — Controller + DTO cho "Task"

Mục tiêu tối nay: viết xong Controller CRUD cơ bản (chưa cần kết nối database thật, có thể dùng danh sách tạm trong bộ nhớ để tập trung vào cấu trúc Controller/DTO trước).

```java
// DTO nhận dữ liệu từ client
public class TaskDTO {
    private String tieuDe;
    private boolean hoanThanh;
    // getter, setter
}

// Model tạm giữ trong bộ nhớ (chưa cần Entity/Database tối nay)
public class Task {
    private Long id;
    private String tieuDe;
    private boolean hoanThanh;
    // constructor, getter, setter
}

@RestController
@RequestMapping("/api/tasks")
public class TaskController {
    private final List<Task> danhSachTask = new ArrayList<>();
    private Long dem = 1L;

    @GetMapping
    public List<Task> layTatCa() {
        return danhSachTask;
    }

    @PostMapping
    public Task taoMoi(@RequestBody TaskDTO dto) {
        Task task = new Task(dem++, dto.getTieuDe(), false);
        danhSachTask.add(task);
        return task;
    }

    @PutMapping("/{id}")
    public Task capNhat(@PathVariable Long id, @RequestBody TaskDTO dto) {
        for (Task t : danhSachTask) {
            if (t.getId().equals(id)) {
                t.setTieuDe(dto.getTieuDe());
                t.setHoanThanh(dto.isHoanThanh());
                return t;
            }
        }
        throw new RuntimeException("Không tìm thấy task");
    }

    @DeleteMapping("/{id}")
    public void xoa(@PathVariable Long id) {
        danhSachTask.removeIf(t -> t.getId().equals(id));
    }
}
```

> Ghi chú: Đây là bản CRUD "tạm" lưu trong `List` (mất dữ liệu khi restart app) — mục đích tối nay là làm quen cấu trúc Controller/DTO/HTTP method. Ngày 4 sẽ thay `List` này bằng `JpaRepository` kết nối database thật.

## Bài tập buổi tối

**Bài 6 (Lý thuyết Bloc/Cubit):** Vẽ lại (bằng sơ đồ chữ, không cần code) luồng xử lý khi người dùng bấm nút "Yêu thích" 1 sản phẩm, dùng mô hình Bloc với Event `ToggleYeuThichEvent`.

**Bài 7 (Cubit với State phức tạp):** Viết `LoginCubit extends Cubit<LoginState>` với `LoginState` gồm 3 loại: `LoginInitial`, `LoginLoading`, `LoginSuccess`, `LoginFailure(String loi)`. Viết hàm `dangNhap(String user, String pass)`: emit `LoginLoading` trước, chờ giả lập 1 giây, rồi emit `LoginSuccess` nếu đúng `admin/123`, ngược lại `LoginFailure("Sai tài khoản")`.

**Bài 8 (CRUD):** Dựa theo ví dụ `Task` ở trên, tự viết CRUD tương tự cho `Note` (`id`, `noiDung`). Viết đủ 4 API: GET all, POST tạo mới, PUT cập nhật theo id, DELETE theo id. Test bằng Postman đủ cả 4 request.

---

## 💻 Code Example (repo `Code`)

> Nguồn: [`SangLeSoftZ/Code`](https://github.com/SangLeSoftZ/Code)

**REST Controller CRUD đầy đủ** — [`baiA/controller/BaiAController.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/baiA/controller/BaiAController.java):

```java
package com.example.demo.baiA.controller;

import com.example.demo.baiA.dto.BaiVietADTO;
import com.example.demo.baiA.entity.BaiVietA;
import com.example.demo.baiA.service.BaiVietAService;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/baiA")
public class BaiAController {

    private final BaiVietAService service;

    public BaiAController(BaiVietAService service) {
        this.service = service;
    }

    // GET /baiA — lấy tất cả bài loại A từ DB
    @GetMapping
    public ResponseEntity<List<BaiVietA>> getAll() {
        return ResponseEntity.ok(service.getAll());
    }

    // GET /baiA/{id}
    @GetMapping("/{id}")
    public ResponseEntity<BaiVietA> getById(@PathVariable Long id) {
        return ResponseEntity.ok(service.getById(id));
    }

    // POST /baiA — CÓ @Valid → @NotBlank hoạt động
    @PostMapping
    public ResponseEntity<BaiVietA> create(@Valid @RequestBody BaiVietADTO dto) {
        return ResponseEntity.status(HttpStatus.CREATED).body(service.create(dto));
    }

    // POST /baiA/khong-valid — KHÔNG có @Valid → @NotBlank vô hiệu
    @PostMapping("/khong-valid")
    public ResponseEntity<BaiVietA> createKhongValid(@RequestBody BaiVietADTO dto) {
        return ResponseEntity.status(HttpStatus.CREATED).body(service.create(dto));
    }

    // PUT /baiA/{id}
    @PutMapping("/{id}")
    public ResponseEntity<BaiVietA> update(
            @PathVariable Long id,
            @Valid @RequestBody BaiVietADTO dto) {
        return ResponseEntity.ok(service.update(id, dto));
    }

    // DELETE /baiA/{id}
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        service.delete(id);
        return ResponseEntity.noContent().build(); // HTTP 204
    }
}
```

Toàn bộ layer đi kèm (đúng chuẩn Controller → Service → Repository → Entity/DTO):
- [`baiA/dto/BaiVietADTO.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/baiA/dto/BaiVietADTO.java)
- [`baiA/entity/BaiVietA.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/baiA/entity/BaiVietA.java)
- [`baiA/repository/BaiVietARepository.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/baiA/repository/BaiVietARepository.java)
- [`baiA/service/BaiVietAService.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/baiA/service/BaiVietAService.java) + [`impl/BaiVietAServiceImpl.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/baiA/service/impl/BaiVietAServiceImpl.java)

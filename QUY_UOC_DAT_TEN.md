---
title: "Quy ước đặt tên — Tài liệu & Code"
---

# 📐 Quy ước đặt tên file (Naming Convention)

Mục tiêu: đặt tên sao cho **tài liệu (docs)** và **code** luôn tra cứu chéo được với nhau
theo Ngày học (Dxx) và theo Chủ đề (topic-slug), không cần nhớ vị trí.

---

## 1. Tài liệu (Markdown trong repo ghi chú)

```
D<NN>_<topic-slug-tieng-anh-khong-dau>.md
```

- **`D<NN>`**: số ngày học, luôn 2 chữ số, khớp với "Lịch học chi tiết theo ngày"
  trong kế hoạch (D01 → D07). Nếu 1 ngày có nhiều buổi thuộc **cùng 1 chủ đề lớn**,
  vẫn dùng chung 1 số ngày, phân biệt bằng slug (không thêm hậu tố Sáng/Chiều/Tối vào tên file
  — nội dung buổi nào ghi rõ trong H1/H2 bên trong file).
- **`topic-slug`**: tên chủ đề viết thường, nối bằng dấu `-`, không dấu tiếng Việt,
  đúng thuật ngữ kỹ thuật gốc (vd: `spring-security-jwt`, `bloc-vs-cubit`, `dio-interceptor`).
- File **đáp án/checklist** dùng chung cho cả 2 track thì để trong `03-DapAn-Checklist/`
  với tên `D<NN>_dap-an-checklist.md`.
- Mỗi file **bắt buộc có frontmatter** ở đầu:

```yaml
---
title: "Tên đầy đủ, dễ đọc"
track: java-spring | flutter | chung
day: <số ngày>
topic: <topic-slug>
tags: [tag1, tag2, ...]
source_goc: <tên file gốc trong vault Obsidian, nếu có>
---
```

Frontmatter là thứ giúp bạn **lookup nhanh** bằng search (Ctrl+F trên `tags:`/`topic:`)
mà không cần mở từng file.

### Ví dụ
| Sai (mơ hồ) | Đúng (theo quy ước) |
|---|---|
| `Ngay4_LyThuyetSau.md` (gộp 3 chủ đề khác nhau) | `D04_spring-data-jpa-hibernate.md` + `D04_dio-interceptor-xu-ly-loi.md` + `D04_json-serializable-freezed-codegen.md` |
| `baocaobuoi2.md` | `D02_ghi-chu-nhanh-java-syntax-co-ban.md` |
| `TEST_BAI2_HIVE.md` | giữ nguyên ý nhưng chuẩn hoá: `D06_test-bai2-hive-local-storage.md` |

---

## 2. Code (Flutter & Spring Boot)

Quy ước đặt tên **file/thư mục code** nên phản ánh đúng **layer** (Clean Architecture)
và **feature**, để khi đọc lại tài liệu ngày nào, biết ngay cần mở code ở đâu.

### 2.1 Flutter (Dart)

```
lib/
├── features/
│   └── <feature_name>/              # snake_case, số ít, vd: auth, task
│       ├── data/
│       │   ├── models/
│       │   │   └── <feature>_model.dart
│       │   ├── datasources/
│       │   │   └── <feature>_remote_data_source.dart
│       │   └── repositories/
│       │       └── <feature>_repository_impl.dart
│       ├── domain/
│       │   ├── entities/
│       │   │   └── <feature>_entity.dart
│       │   ├── repositories/
│       │   │   └── <feature>_repository.dart
│       │   └── usecases/
│       │       └── <action>_<feature>_usecase.dart   # vd: login_usecase.dart
│       └── presentation/
│           ├── cubit/  (hoặc bloc/)
│           │   ├── <feature>_cubit.dart
│           │   └── <feature>_state.dart
│           └── pages/
│               └── <feature>_page.dart
```

- Tên **class** = PascalCase khớp tên file: `auth_cubit.dart` → `class AuthCubit`.
- Tên **test** đi kèm: `test/features/<feature>/.../<ten_file>_test.dart`
  (mirror y hệt cấu trúc `lib/`).
- Khi 1 file code triển khai đúng nội dung đã học ở tài liệu ngày nào, **ghi 1 dòng comment
  đầu file** trỏ ngược lại tài liệu, ví dụ:
  ```dart
  // Áp dụng: 02-Flutter-Bloc/D05_clean-architecture-repository-di.md
  ```

### 2.2 Java / Spring Boot

```
src/main/java/com/<project>/
├── <feature>/                       # vd: task, auth
│   ├── <Feature>Controller.java
│   ├── <Feature>Service.java
│   ├── <Feature>Repository.java
│   ├── dto/
│   │   ├── <Feature>RequestDTO.java
│   │   └── <Feature>ResponseDTO.java
│   └── entity/
│       └── <Feature>.java
├── config/
│   └── SecurityConfig.java, JwtConfig.java ...
└── common/
    └── exception/
        └── GlobalExceptionHandler.java
```

- Tên class = PascalCase, tên file = tên class (chuẩn Java bắt buộc).
- Cũng thêm Javadoc/comment trỏ về tài liệu tương ứng:
  ```java
  // Áp dụng: 01-Java-SpringBoot/D05_spring-security-jwt.md
  ```

### 2.3 Quy ước Git commit (khớp theo ngày học)

```
<Dxx>: <mô tả ngắn theo topic-slug>
```
Ví dụ: `D05: implement spring-security-jwt login API`, `D06: add hive local-storage test`.
→ Khi cần lookup "code ngày 5 làm gì", chỉ cần `git log --grep "D05"`.

---

## 3. Tóm tắt nguyên tắc chung

1. **Số ngày (Dxx)** luôn đi đầu — dùng chung giữa tài liệu và commit message.
2. **Topic-slug tiếng Anh, không dấu, có dấu gạch ngang** — dùng chung giữa tên file
   tài liệu, tên thư mục/class code, và comment trỏ ngược.
3. **Tách riêng thư mục theo track** (`01-Java-SpringBoot/`, `02-Flutter-Bloc/`) —
   không gộp 2 chủ đề vào 1 file như vault gốc.
4. **Đáp án/checklist tách riêng**, không lẫn vào file lý thuyết, để lý thuyết luôn
   sạch khi đọc lại.

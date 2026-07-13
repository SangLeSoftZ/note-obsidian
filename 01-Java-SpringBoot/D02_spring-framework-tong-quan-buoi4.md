---
title: "Ghi chú — Tổng quan Spring Framework (Buổi 4 cá nhân)"
track: java-spring
day: 2
topic: spring-overview
tags: [spring, di, ioc, aop, spring-security, spring-data, spring-boot]
source_goc: BaoCaoBuoi4.md
---

# Ghi chú — Tổng quan Spring Framework (Buổi 4 cá nhân)

#Spring framework
Tìm hiểu về spring  được sử dụng rất nhiều do nó rất tiện dụng cũng như các tính năng cung cấp rất đầy đủ bao gồm một số thành phần: 
 + Spring framework core:
  +DI (Dependency Injection) & IoC (Inversion of Control)
      
    +Dependency (sự phụ thuộc): là khi một class A cần dùng một class B để hoạt động → A "phụ thuộc" vào B.
    +Injection (tiêm): là hành động đưa (cung cấp) dependency đó vào class từ bên ngoài, thay vì class tự tạo ra nó. 
    +Dependency Injection: là một kỹ thuật thiết kế phần mềm, trong đó một object nhận các dependency của nó từ bên ngoài thay vì tự khởi tạo mới bên trong chính nó.
   + AOP (Aspect Oriented Programming)
    
  + Spring security
  + Spring data

 + Spring boot: giúp tạo các dự án bằng spring nhanh hơn chỉ cần một vài 
 live code thì web services đã có thể hoạt động  phải học về các :
  + Micoservices(dịch vụ):
    
   + openFegin: là http client giúp giao tiếp tốt hơn giưa các 
    sytem và các cũng như các microservices bên trong của hệ thống bằng các giao thức http
  
   + Spring cloud gateway: giống như một cổng để đi vào microservices 
  
  + Micrometer: bộ công cụ để do lường thu thập các vấn đề trong springboot bằng cách làm được cấc việc:
     + Logging :Ghi lại từng sự kiện xảy ra (request, lỗi, dữ liệu thay đổi), 
      để lần theo 1 request qua nhiều service

     + Metrics: Đo các con số hệ thống theo thời gian: số request/giây, CPU,
      độ trễ trung bình... xuất ra Prometheus, vẽ dashboard bằng Grafana
   
     + Tracing Vẽ timeline chi tiết 1 request cụ thể đi 
      qua bao nhiêu service, mất bao lâu ở từng bước (dùng Zipkin/Jaeger)

     + Docker: Dùng để gói mỗi service  thành 1 image thống nhất, 
      chạy giống hệt nhau trên mọi máy — giải quyết vấn đề "chạy máy tôi thì được".
  + Testing: 

    + SpringBootTest:  sẽ khởi động toàn bộ (hoặc một phần) Spring Application Context giống như lúc ứng dụng chạy thật,
    để bạn test tích hợp (Integration Test ) — nhiều Bean phối hợp với nhau có hoạt động đúng không.
    
    + MockMVC : Bạn có thể kiểm tra xem điểm cuối REST API ( @RestController) để biết yêu cầu, phản hồi, mã trạng thái — nhưng không muốn phải khởi động 1 máy chủ HTTP thật 

    + Mocking: Khi test 1 class, class đó thường phụ thuộc vào class khácNếu test thật với database, test sẽ chậm, phức tạp, và không còn là "unit test" nữa — vì bạn đang test luôn cả database

    + Testcontainers : Testcontainers tự động khởi động 1 Docker container thật (PostgreSQL thật, Kafka thật, Redis thật...) chỉ trong lúc chạy test, rồi tự hủy sau khi test xong. Bạn test với database/service giống hệt production, không phải bản giả lập.

  + Spring for Apache Kafka:
   + Đây là một dự án con của Spring Framework (thường gọi tắt là Spring Kafka), giúp tích hợp Apache Kafka (hệ thống message broker/streaming   phân tán) vào các ứng dụng Java sử dụng Spring.
   +  Nó cung cấp các abstraction quen thuộc kiểu Spring (như KafkaTemplate, @KafkaListener) để bạn không phải làm việc trực tiếp với Kafka  Producer/Consumer API thô của Kafka (vốn khá phức tạp).

## 💻 Code Example (repo `Code`)

> Ví dụ minh hoạ trực quan cho DI/IoC đã nêu ở phần lý thuyết: xem [`bai4/KhoServiceImpl.java`](https://github.com/SangLeSoftZ/Code/blob/main/src/main/java/com/example/demo/bai4/KhoServiceImpl.java)
> (đã trích dẫn đầy đủ trong `D02_spring-core-ioc-di-bean.md`).

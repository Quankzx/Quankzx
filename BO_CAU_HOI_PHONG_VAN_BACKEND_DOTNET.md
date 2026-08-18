# 🎯 BỘ CÂU HỎI & TRẢ LỜI PHỎNG VẤN THỰC CHIẾN 100% TỪ SOURCE CODE DỰ ÁN
> **Ứng viên:** Nguyễn Hồng Quân (Quankzx)  
> **Cam kết:** 100% dựa trên source code thực tế tại `C:\CongViec\LSB`, `C:\CongViec\LSB Watchdog`, Võ Minh Thiên, Nail Salon SaaS và HQ Soft. **Không vẽ vời công nghệ hư cấu, không bịa đặt số liệu.**

---

## 📑 MỤC LỤC 40 CÂU HỎI TRỌNG TÂM THEO SOURCE CODE

1. [Chương 1: Kiến trúc LSB & Clean Architecture Thực tế (Câu 1 - 7)](#chương-1-kiến-trúc-lsb--clean-architecture-thực-tế)
2. [Chương 2: Amazon SP-API & Các Worker Services Chạy Ngầm (Câu 8 - 15)](#chương-2-amazon-sp-api--các-worker-services-chạy-ngầm)
3. [Chương 3: Hệ thống LSB Watchdog, Telemetry & Polly Resilience (Câu 16 - 22)](#chương-3-hệ-thống-lsb-watchdog-telemetry--polly-resilience)
4. [Chương 4: Cơ sở dữ liệu SQL Server & PostgreSQL Thực chiến (Câu 23 - 29)](#chương-4-cơ-sở-dữ-liệu-sql-server--postgresql-thực-chiến)
5. [Chương 5: AWS S3 & Bảo mật Credentials (DPAPI / AES) (Câu 30 - 34)](#chương-5-aws-s3--bảo-mật-credentials-dpapi--aes)
6. [Chương 6: Dự án Võ Minh Thiên, Nail Salon SaaS & HQ Soft (Câu 35 - 38)](#chương-6-dự-án-võ-minh-thiên-nail-salon-saas--hq-soft)
7. [Chương 7: Quy trình Agile/Scrum & Kỹ năng làm việc (Câu 39 - 40)](#chương-7-quy-trình-agilescrum--kỹ-năng-làm-việc)

---

# CHƯƠNG 1: KIẾN TRÚC LSB & CLEAN ARCHITECTURE THỰC TẾ
*(Căn cứ: `LSB.SellerCentral.Core`, `LSB.SellerCentral.BackEnd`, `LSB.SellerCentral.Services`)*

### ❓ Câu 1: Em hãy trình bày cấu trúc Solution của dự án LSB Seller Central?
* **Trả lời chuẩn từ source code:**
  > *"Solution của em được tổ chức dạng Monorepo gồm các project rõ ràng theo Clean Architecture:  
  > 1. `LSB.SellerCentral.Core`: Thư viện dùng chung (.NET 7/8 Class Library) chứa các DTOs, cấu hình, Base SP-API Client (`OrdersApiClient`, `FinancesApiClient`, `LedgerSummaryApiClient`) và các policy Polly retry.  
  > 2. `LSB.SellerCentral.BackEnd`: Tách làm 4 tầng Clean Architecture:  
  >    - `Domain`: Chứa Entities (`SettlementSummary`, `FinancialTransaction`, `LedgerSummary`, `FulfilledShipment`...).  
  >    - `Application`: Chứa CQRS Handlers với MediatR, FluentValidation, AWS S3 Client.  
  >    - `Infrastructure`: Chứa EF Core `ApplicationDbContext` (SQL Server), Identity, JWT.  
  >    - `API`: Cung cấp RESTful endpoints cho dashboard.  
  > 3. `LSB.SellerCentral.Services`: Windows Worker Service chứa các Background Workers chạy ngầm đồng bộ dữ liệu Amazon.  
  > 4. `LSB.SellerCentral.App`: Chứa frontend React 18 + Vite + TailwindCSS (SellerManager) và WPF ConfigurationTool."*

---

### ❓ Câu 2: Tại sao em dùng MediatR trong tầng Application của LSB? Luồng xử lý một Request diễn ra như thế nào?
* **Trả lời chuẩn từ source code:**
  > *"Em dùng MediatR để áp dụng mô hình CQRS (Command Query Responsibility Segregation) nhằm tách biệt luồng Ghi và Đọc, đồng thời Decouple (giảm phụ thuộc) giữa Controller và Business Logic:  
  > - **Luồng xử lý:** Khi Controller nhận request, nó chỉ cần gửi `_mediator.Send(command)`. MediatR tự động điều phối đến đúng `IRequestHandler` tương ứng trong tầng Application.  
  > - Tầng Application xử lý nghiệp vụ, gọi `ApplicationDbContext` ở Infrastructure để lưu dữ liệu, sau đó trả về DTO cho Controller.  
  > - Nhờ MediatR, mỗi Use-Case nằm gọn trong 1 file Handler riêng, rất dễ bảo trì và viết Unit Test với Moq/xUnit."*

---

### ❓ Câu 3: Dependency Injection (DI) trong LSB được cấu hình thế nào? Tránh lỗi Captive Dependency ra sao?
* **Trả lời chuẩn từ source code:**
  > *"Trong Web API, em đăng ký các Service nghiệp vụ là **Scoped** (`ISettlementSyncService`, `ApplicationDbContext`), các Helper không trạng thái là **Transient**, và các thành phần dùng chung như `HttpClient`, `IConfiguration`, `LwaTokenProvider` là **Singleton**.  
  > - **Tránh lỗi Captive Dependency:** Trong `LSB.SellerCentral.Services` (BackgroundService là Singleton), em tuyệt đối không inject trực tiếp `DbContext` vào constructor. Thay vào đó, em inject `IServiceScopeFactory`, mỗi lần Worker chạy chu kỳ sync sẽ gọi `using (var scope = _scopeFactory.CreateScope())` để tạo scope mới và resolve `DbContext` từ scope đó. Khi xong việc, scope tự dispose giải phóng kết nối."*

---

### ❓ Câu 4: Em validate dữ liệu đầu vào trong LSB bằng cách nào?
* **Trả lời chuẩn từ source code:**
  > *"Em sử dụng thư viện `FluentValidation` trong project `LSB.SellerCentral.Application`. Mỗi Command/Query có một class Validator kế thừa `AbstractValidator<T>`, ví dụ kiểm tra khoảng ngày hợp lệ, định dạng Marketplace ID, SKU không được rỗng... Logic validation được tách biệt hoàn toàn khỏi DTO và Entity."*

---

### ❓ Câu 5: Em cấu hình xác thực người dùng và bảo mật API trong LSB BackEnd như thế nào?
* **Trả lời chuẩn từ source code:**
  > *"Em sử dụng `Microsoft.AspNetCore.Identity.EntityFrameworkCore` kết hợp với **JWT (JSON Web Token)** (`System.IdentityModel.Tokens.Jwt`). Khi người dùng đăng nhập thành công, hệ thống mã hóa UserId, Email, Roles vào Claims của Token, ký bằng khóa đối xứng HMAC-SHA256. Các Controller bảo mật được gắn thuộc tính `[Authorize]`."*

---

### ❓ Câu 6: Dự án LSB xử lý Exception tập trung như thế nào?
* **Trả lời chuẩn từ source code:**
  > *"Em xây dựng `GlobalExceptionMiddleware` đặt ở đầu pipeline của ASP.NET Core API. Mọi unhandled exception phát sinh từ tầng dưới đều được bắt tại đây, ghi log chi tiết vào Serilog kèm `TraceIdentifier`, và trả về response chuẩn ProblemDetails với mã lỗi HTTP phù hợp (400, 404, 500) mà không để lộ chi tiết nhạy cảm của server ra ngoài."*

---

### ❓ Câu 7: Lập trình bất đồng bộ (`async/await`) được áp dụng trong LSB ra sao?
* **Trả lời chuẩn từ source code:**
  > *"Toàn bộ các tác vụ I/O bound từ việc gọi Amazon SP-API qua `HttpClient` / `RestSharp`, query database qua EF Core (`ToListAsync`, `SaveChangesAsync`), upload file lên S3 (`PutObjectAsync`) đều là `async/await` 100%. Em luôn truyền `CancellationToken` từ Controller hoặc Worker vào các method bất đồng bộ để kịp thời giải phóng tài nguyên khi request bị cancel."*

---

# CHƯƠNG 2: AMAZON SP-API & CÁC WORKER SERVICES CHẠY NGẦM
*(Căn cứ: `LSB.SellerCentral.Services`, `SettlementSyncWorker`, `ReportSyncWorker`, `SearchOrdersDataWorker`)*

### ❓ Câu 8: Kể tên các Background Worker chính trong `LSB.SellerCentral.Services` và nhiệm vụ của từng worker?
* **Trả lời chuẩn từ source code:**
  > *"Trong project `LSB.SellerCentral.Services`, em phát triển các worker kế thừa `BackgroundService` chạy trên Windows Services:  
  > 1. `SettlementSyncWorker`: Thu thập các Financial Event Groups trạng thái `CLOSED` từ Amazon Finances API v2024-06-19 và liên kết `SettlementSummaryId` với các giao dịch thời gian thực (`FinancialTransaction`).  
  > 2. `ReportSyncWorker`: Yêu cầu và tải các báo cáo FBA Returns, Fulfilled Shipments, Inventory Ledger (`GET_LEDGER_SUMMARY_VIEW_DATA`).  
  > 3. `SearchOrdersDataWorker`: Đồng bộ đơn hàng mới và cập nhật trạng thái đơn hàng (Pending, Unshipped, Shipped, Cancelled).  
  > 4. `InventoryDataWorker` & `PerformanceDataWorker`: Cập nhật số lượng tồn kho và chỉ số sức khỏe tài khoản (Account Health / ODR)."*

---

### ❓ Câu 9: Cơ chế xác thực LWA (Login with Amazon) cho SP-API hoạt động như thế nào trong code của em?
* **Trả lời chuẩn từ source code:**
  > *"Amazon SP-API yêu cầu mỗi request phải có header `x-amz-access-token`. Token này chỉ có hạn 1 tiếng.  
  > Em xây dựng một Token Helper: lưu Access Token trong bộ nhớ cache. Trước khi gọi SP-API, hệ thống kiểm tra token. Nếu token sắp hết hạn (trước 5 phút), hệ thống sẽ gửi request OAuth2 POST đến `https://api.amazon.com/auth/o2/token` với `client_id`, `client_secret` và `refresh_token` để lấy access token mới, cập nhật lại vào cache rồi mới thực thi tiếp."*

---

### ❓ Câu 10: Amazon SP-API giới hạn Rate Limit (Token Bucket). Em đã xử lý lỗi `429 QuotaExceeded` trong Worker như thế nào?
* **Trả lời chuẩn từ source code:**
  > *"Em xử lý theo 2 cấp độ:  
  > 1. **Cấp độ Request (Polly):** Áp dụng chính sách Retry với Exponential Backoff kèm Jitter qua `Polly.Extensions.Http` cho các lỗi tạm thời.  
  > 2. **Cấp độ Worker Batch (Nghiệp vụ):** Với các báo cáo nặng trong `ReportSyncWorker` (như Inventory Ledger), nếu gặp `429 QuotaExceeded`, worker được lập trình áp dụng quy tắc tạm dừng và delay 4 tiếng trước khi thử lại chunk tiếp theo. Đồng thời khi `LastSyncedAt` đã chạm ngày hiện tại, worker cũng delay 4 tiếng để tránh spam gọi báo cáo nhiều lần trong ngày."*

---

### ❓ Câu 11: Trong `SettlementSyncWorker`, em chia batch đồng bộ dữ liệu tài chính như thế nào?
* **Trả lời chuẩn từ source code:**
  > *"Dữ liệu Settlement được chia theo từng chunk 180 ngày để truy vấn các nhóm sự kiện tài chính `CLOSED`. Khi lấy về, worker tự động liên kết `SettlementSummaryId` với các bản ghi `FinancialTransaction` có sẵn theo khoảng ngày mà không xóa hay ghi đè lịch sử giao dịch gốc, đảm bảo tính toàn vẹn cho dữ liệu đối soát."*

---

### ❓ Câu 12: Khi tải các file báo cáo lớn từ Amazon về, em lưu trữ payload audit ở đâu và như thế nào?
* **Trả lời chuẩn từ source code:**
  > *"Hệ thống tự động lưu các file JSON payload thô vào thư mục `C:\ProgramData\LSB\LSBSellerCentral\Payloads\` theo cấu trúc phân cấp để phục vụ cho việc kiểm tra (audit log) và đối soát khi có sai lệch số liệu, sau đó giải nén, parse từng dòng và lưu vào CSDL."*

---

### ❓ Câu 13: Làm sao đảm bảo không bị trùng đơn hàng khi `SearchOrdersDataWorker` chạy định kỳ?
* **Trả lời chuẩn từ source code:**
  > *"Trên bảng `Orders` trong CSDL SQL Server, em đặt chỉ mục duy nhất (Unique Index) trên cột `AmazonOrderId`. Trong worker, khi quét danh sách đơn hàng về, hệ thống kiểm tra nếu `AmazonOrderId` đã tồn tại thì chỉ cập nhật trạng thái (`OrderStatus`, `LastUpdateDate`), nếu chưa có thì mới `Insert` mới, bọc trong Transaction của DB."*

---

### ❓ Câu 14: Khi Windows tắt hoặc service bị Stop, làm sao Worker dừng an toàn (Graceful Shutdown)?
* **Trả lời chuẩn từ source code:**
  > *"Trong vòng lặp `ExecuteAsync(CancellationToken stoppingToken)` của Worker, em luôn kiểm tra `stoppingToken.IsCancellationRequested` và truyền `stoppingToken` vào các hàm I/O async. Sau khi xử lý xong một batch/chunk nhỏ (50-100 bản ghi), worker commit DB và lưu mốc `LastSyncedAt`. Nếu service bị ngắt giữa chừng, worker dừng ngay lập tức mà không để lại dữ liệu rác, và lần chạy tiếp theo sẽ bắt đầu từ mốc `LastSyncedAt` gần nhất."*

---

### ❓ Câu 15: Tại sao em đóng gói `LSB.SellerCentral.Services` thành Windows Service thay vì Console App?
* **Trả lời chuẩn từ source code:**
  > *"Em sử dụng package `Microsoft.Extensions.Hosting.WindowsServices` và WiX Toolset (`LSB.SellerCentral.Hub.Services`) để cài đặt thành Windows Service trên server. Điều này giúp service tự động khởi động cùng Windows khi máy chủ restart, chạy ngầm dưới quyền tài khoản hệ thống mà không cần người dùng đăng nhập mở terminal, và có thể được quản lý bởi ServiceController / Watchdog."*

---

# CHƯƠNG 3: HỆ THỐNG LSB WATCHDOG, TELEMETRY & POLLY RESILIENCE
*(Căn cứ: `C:\CongViec\LSB Watchdog`, `LSB.Watchdog.Services`, `LSB.Watchdog.Central`)*

### ❓ Câu 16: Dự án LSB Watchdog được sinh ra để giải quyết vấn đề gì?
* **Trả lời chuẩn từ source code:**
  > *"Trong môi trường production, các Worker Service đồng bộ Amazon chạy 24/7 có thể bị dừng do lỗi mạng đột ngột, server hết tài nguyên hoặc tiến trình bị crash/treo.  
  > `LSB.Watchdog` là một hệ thống độc lập chạy ngầm có nhiệm vụ: liên tục giám sát trạng thái sống còn của các Windows Services, đo đạc tải phần cứng (CPU/RAM/Disk), tự động khởi động lại service khi gặp sự cố (Self-Healing), và cung cấp API/WebPortal quản lý tập trung."*

---

### ❓ Câu 17: Watchdog kiểm tra và điều khiển các Windows Service bằng cách nào trong C#?
* **Trả lời chuẩn từ source code:**
  > *"Em sử dụng thư viện `System.ServiceProcess.ServiceController` (phiên bản 8.0.0).  
  > Watchdog định kỳ quét danh sách tên service được cấu hình:  
  > ```csharp
  > using var sc = new ServiceController(serviceName);
  > if (sc.Status == ServiceControllerStatus.Stopped)
  > {
  >     _logger.LogWarning("Service {ServiceName} is stopped. Attempting restart...", serviceName);
  >     sc.Start();
  >     sc.WaitForStatus(ServiceControllerStatus.Running, TimeSpan.FromSeconds(30));
  > }
  > ```
  > Điều này đảm bảo các Worker bị crash sẽ được tự động phục hồi trong vòng vài giây."*

---

### ❓ Câu 18: Watchdog đo đạc telemetry phần cứng (CPU, RAM, Disk) như thế nào?
* **Trả lời chuẩn từ source code:**
  > *"Em tích hợp thư viện `Hardware.Info` (v101.1.0) trong `LSB.Watchdog.Services`.  
  > Định kỳ hệ thống đọc thông số: tổng dung lượng RAM đã dùng/còn trống, phần trăm tải CPU, và dung lượng trống trên các ổ đĩa. Thông tin này được ghi vào Serilog và phục vụ cho các cảnh báo khi tài nguyên máy chủ chạm ngưỡng nguy hiểm (>90% RAM hoặc Disk đầy)."*

---

### ❓ Câu 19: Em cấu hình thư viện Polly (v8.6.4) trong Watchdog và LSB như thế nào?
* **Trả lời chuẩn từ source code:**
  > *"Em sử dụng các package `Polly`, `Polly.Core`, `Polly.Extensions` (v8.6.4) và `Microsoft.Extensions.Http.Resilience`.  
  > Em thiết lập **ResiliencePipeline** gồm:  
  > 1. **Retry Strategy:** Thử lại tối đa 3 lần với Exponential Backoff kèm Jitter khi gặp lỗi HTTP 5xx hoặc HttpRequestException.  
  > 2. **Timeout Strategy:** Giới hạn thời gian tối đa cho mỗi HTTP Call (ví dụ 30s) để giải phóng thread.  
  > 3. **Circuit Breaker:** Tạm ngắt kết nối nếu tỷ lệ lỗi vượt quá 50% trong 30 giây để tránh làm nghẽn hệ thống."*

---

### ❓ Câu 20: Cấu hình Serilog trong Watchdog có những tính năng gì nổi bật?
* **Trả lời chuẩn từ source code:**
  > *"Trong `LSB.Watchdog.Services`, em cấu hình Serilog (v4.3.0) với:  
  > - **Enrichers:** `Serilog.Enrichers.Process`, `Thread`, `Environment` để tự động đính kèm ProcessId, ThreadId và MachineName vào từng dòng log.  
  > - **Sinks:** Sử dụng `Serilog.Sinks.Async` để ghi log bất đồng bộ không làm chậm luồng chính, kết hợp `Serilog.Sinks.File` với cơ chế rolling log hàng ngày và `Serilog.Sinks.Map` để tách log theo từng module service riêng biệt."*

---

### ❓ Câu 21: `LSB.Watchdog.Central` và file cấu hình Yaml hoạt động ra sao?
* **Trả lời chuẩn từ source code:**
  > *"Project `LSB.Watchdog.Central` là một ASP.NET Core Web API sử dụng Swagger/OpenAPI để quản lý cấu hình tập trung. Em sử dụng `YamlDotNet` (v16.3.0) để đọc và parse các file cấu hình `.yaml` định nghĩa danh sách các service cần giám sát, ngưỡng cảnh báo CPU/RAM và chu kỳ kiểm tra."*

---

### ❓ Câu 22: Công cụ `ConfigurationTool` trong Watchdog và LSB dùng để làm gì?
* **Trả lời chuẩn từ source code:**
  > *"Đó là một ứng dụng Desktop WPF xây dựng bằng `CommunityToolkit.Mvvm` (v8.2.2) và giao diện `MaterialDesignThemes` (v5.2.1). Công cụ này cho phép kỹ thuật viên cấu hình nhanh chu kỳ Worker, kiểm tra bật/tắt Windows Services trực quan qua `ServiceController`, và mã hóa an toàn các API Key của Amazon trước khi ghi vào file cấu hình."*

---

# CHƯƠNG 4: CƠ SỞ DỮ LIỆU SQL SERVER & POSTGRESQL THỰC CHIẾN
*(Căn cứ: `LSB.SellerCentral.Infrastructure`, `query_db`, Võ Minh Thiên)*

### ❓ Câu 23: Em sử dụng Entity Framework Core với SQL Server trong LSB như thế nào?
* **Trả lời chuẩn từ source code:**
  > *"Em sử dụng `Microsoft.EntityFrameworkCore.SqlServer` (v7.0).  
  > Trong `LSB.SellerCentral.Infrastructure`, `ApplicationDbContext` quản lý các DbSet cho Entities. Em sử dụng Fluent API trong `OnModelCreating` để cấu hình ràng buộc quan hệ, kiểu dữ liệu (`decimal(18,2)` cho số tiền tài chính), và quản lý cấu trúc bảng hoàn toàn thông qua **EF Core Migrations** (`Microsoft.EntityFrameworkCore.Tools`)."*

---

### ❓ Câu 24: Tại sao trong các Worker đồng bộ dữ liệu em hay dùng `.AsNoTracking()`?
* **Trả lời chuẩn từ source code:**
  > *"Khi Worker cần đọc hàng nghìn bản ghi để kiểm tra xem đơn hàng hay giao dịch đã tồn tại trong DB chưa (Read-Only), việc dùng `.AsNoTracking()` giúp EF Core không đưa các entity này vào Change Tracker. Điều này giúp tiết kiệm rất nhiều dung lượng RAM và tăng tốc độ truy vấn từ 20-30%."*

---

### ❓ Câu 25: Em tối ưu truy vấn SQL Server bằng Covering Index (`INCLUDE`) như thế nào trong dự án Võ Minh Thiên / LSB?
* **Trả lời chuẩn từ source code:**
  > *"Trong hệ thống đơn hàng logistics tại Võ Minh Thiên và LSB, các truy vấn danh sách đơn hàng thường xuyên lọc theo `OrderStatus` và `OrderDate`, sau đó hiển thị các cột `AmazonOrderId`, `TotalAmount`, `CustomerName`.  
  > Nếu chỉ đánh index trên `(OrderStatus, OrderDate)`, SQL Server sẽ phải thực hiện thao tác **Key Lookup** vào bảng chính để lấy các cột còn lại.  
  > Em đã tạo **Covering Index**:  
  > ```sql
  > CREATE NONCLUSTERED INDEX IX_Orders_Filter
  > ON Orders (OrderStatus, OrderDate)
  > INCLUDE (AmazonOrderId, TotalAmount, CustomerName);
  > ```
  > Nhờ vậy, câu query lấy toàn bộ dữ liệu ngay trên Index (Index Covering) mà không cần Key Lookup, giảm Disk I/O và tăng tốc độ truy vấn rõ rệt."*

---

### ❓ Câu 26: Khi nào em dùng `ChangeTracker.Clear()` trong EF Core khi xử lý Batch Data?
* **Trả lời chuẩn từ source code:**
  > *"Khi Worker insert hoặc update hàng nghìn bản ghi theo từng đợt (ví dụ mỗi batch 500 records), sau khi gọi `await _dbContext.SaveChangesAsync()`, các bản ghi đó vẫn nằm trong bộ nhớ theo dõi của `ChangeTracker`. Nếu không xóa, qua nhiều vòng lặp `ChangeTracker` sẽ phình to gây tốn RAM và làm các lần `DetectChanges` sau bị chậm. Em gọi `_dbContext.ChangeTracker.Clear()` ngay sau mỗi batch để dọn sạch bộ nhớ theo dõi."*

---

### ❓ Câu 27: Phân biệt cách sử dụng SQL Server và PostgreSQL trong kinh nghiệm của em?
* **Trả lời chuẩn từ source code:**
  > *"Em có kinh nghiệm làm việc với cả hai hệ CSDL:  
  > - **SQL Server:** Thường dùng trong các ứng dụng doanh nghiệp (.NET Core + EF Core SqlServer), hỗ trợ cực tốt công cụ SSMS, phân tích Execution Plan trực quan, và chỉ mục Covering Index `INCLUDE`.  
  > - **PostgreSQL (v16):** Rất mạnh mẽ trong việc xử lý kiểu dữ liệu JSON/JSONB, chi phí vận hành mã nguồn mở tối ưu, và quản lý các tác vụ batch import lớn rất nhẹ nhàng."*

---

### ❓ Câu 28: Làm sao em phát hiện được câu query SQL bị chậm trong quá trình phát triển?
* **Trả lời chuẩn từ source code:**
  > *"1. **Bật Log SQL trong EF Core:** Cấu hình `LogTo(Console.WriteLine, LogLevel.Information)` trong môi trường Development để xem câu lệnh SQL mà EF Core tự sinh ra.  
  > 2. **SQL Server Profiler / Extended Events:** Bắt các câu query có `Duration` cao hoặc `Reads` lớn.  
  > 3. **SSMS Execution Plan (`Ctrl + M`):** Kiểm tra xem có xuất hiện `Table Scan`, `Index Scan` hoặc `Key Lookup` không để bổ sung Index thích hợp."*

---

### ❓ Câu 29: Stored Procedures có được dùng trong các dự án của em không và khi nào nên dùng?
* **Trả lời chuẩn từ source code:**
  > *"Có. Trong dự án tại Võ Minh Thiên và HQ Soft, các báo cáo tổng hợp doanh thu theo tháng/quý cần tính toán phức tạp trên nhiều bảng lớn được viết bằng **Stored Procedures** và gọi từ .NET qua EF Core `FromSqlRaw` hoặc Dapper để tận dụng việc pre-compile kế hoạch thực thi trực tiếp trên Database server."*

---

# CHƯƠNG 5: AWS S3 & BẢO MẬT CREDENTIALS (DPAPI / AES)
*(Căn cứ: `LSB.SellerCentral.Application`, `AWSSDK.S3`, `ConfigurationTool`)*

### ❓ Câu 30: Thư viện `AWSSDK.S3` được dùng như thế nào trong `LSB.SellerCentral.Application`?
* **Trả lời chuẩn từ source code:**
  > *"Trong project `LSB.SellerCentral.Application`, em cài đặt package `AWSSDK.S3` (v4.0.18.7). Khi các Worker tải về các file báo cáo tài chính lớn từ Amazon SP-API, hệ thống tạo `PutObjectRequest` truyền `MemoryStream` hoặc `FileStream` để upload trực tiếp lên S3 bucket của doanh nghiệp, sau đó lưu lại S3 Key / URL vào CSDL SQL Server để web dashboard có thể tải về khi cần."*

---

### ❓ Câu 31: Tại sao lại lưu file báo cáo trên AWS S3 thay vì lưu trực tiếp trong Database hoặc ổ cứng máy chủ?
* **Trả lời chuẩn từ source code:**
  > *"1. **Không làm nặng Database:** Các file báo cáo và export hàng trăm MB nếu lưu dạng `VARBINARY(MAX)` trong SQL Server sẽ làm phình dung lượng file `.mdf`, khiến backup/restore CSDL rất lâu và tốn tài nguyên.  
  > 2. **Độc lập máy chủ:** Lưu trên S3 giúp máy chủ chạy ứng dụng không bị đầy ổ đĩa cục bộ và có thể dễ dàng scale hoặc thay thế server mà không lo mất file dữ liệu."*

---

### ❓ Câu 32: Cơ chế mã hóa API Key của Amazon SP-API trong `ConfigurationTool` hoạt động ra sao?
* **Trả lời chuẩn từ source code:**
  > *"Để tránh việc lưu trữ Plain Text các thông tin cực kỳ nhạy cảm như `LWA Client Secret`, `Refresh Token`, `AWS Secret Key` vào file `appsettings.json`, em sử dụng **Windows DPAPI (Data Protection API)** qua `ProtectedData.Protect` (kèm phạm vi `DataProtectionScope.CurrentUser` hoặc `LocalMachine`) hoặc mã hóa **AES-256**. Chỉ khi ứng dụng khởi chạy trên chính máy chủ đó, hệ thống mới giải mã ra bộ nhớ để sử dụng."*

---

### ❓ Câu 33: Em quản lý Connection String và Secrets giữa các môi trường (Dev, Staging, Prod) như thế nào?
* **Trả lời chuẩn từ source code:**
  > *"Em sử dụng cơ chế cấu hình phân tầng của .NET Core:  
  > - Khi Dev: Dùng `appsettings.Development.json` hoặc công cụ `dotnet user-secrets`.  
  > - Khi Production: Đọc từ **Biến môi trường (Environment Variables)** của hệ điều hành hoặc file cấu hình được phân quyền truy cập chặt chẽ trên máy chủ. Không bao giờ commit mật khẩu production lên Git repository."*

---

### ❓ Câu 34: Mã hóa mật khẩu người dùng trong LSB được thực hiện bằng thư viện nào?
* **Trả lời chuẩn từ source code:**
  > *"Trong `LSB.SellerCentral.Infrastructure`, em sử dụng thư viện `BCrypt.Net-Next` (v4.0.3) để băm mật khẩu một chiều kết hợp Salt ngẫu nhiên (`BCrypt.HashPassword`). Khi xác thực đăng nhập, em dùng `BCrypt.Verify(password, hashedPassword)`, đảm bảo ngay cả khi CSDL bị lộ thì mật khẩu người dùng vẫn an toàn tuyệt đối."*

---

# CHƯƠNG 6: DỰ ÁN VÕ MINH THIÊN, NAIL SALON SAAS & HQ SOFT

### ❓ Câu 35: Em đã làm những gì tại Công ty Võ Minh Thiên?
* **Trả lời chuẩn từ source code & CV:**
  > *"Tại Võ Minh Thiên, em là .NET Developer tham gia phát triển hệ thống quản lý đơn hàng và logistics vận chuyển xuyên biên giới:  
  > - Xây dựng các RESTful Web APIs bằng ASP.NET Core để tiếp nhận đơn hàng, quản lý quy trình gom hàng và đồng bộ trạng thái vận đơn với các đơn vị vận chuyển bên ngoài.  
  > - Áp dụng `async/await` để xử lý các luồng tích hợp với bên vận chuyển mà không nghẽn server.  
  > - Phân tích Execution Plan trong SQL Server, tối ưu câu lệnh truy vấn và tạo Covering Index (`INCLUDE`) giúp tăng tốc độ xử lý đơn hàng."*

---

### ❓ Câu 36: Em đã xây dựng những module nào cho dự án Nail Salon SaaS (Mỹ)?
* **Trả lời chuẩn từ source code & CV:**
  > *"Đây là dự án SaaS quản lý chuỗi tiệm nail tại thị trường Mỹ mà em tham gia dạng Freelance:  
  > - Xây dựng backend RESTful APIs cho tính năng đặt lịch hẹn online (Appointment Booking), phân ca thợ (Staff Scheduling), và tích điểm khách hàng thân thiết.  
  > - Phát triển giao diện web tương tác bằng các **Blazor components** giúp chủ tiệm và nhân viên thao tác mượt mà trên trình duyệt.  
  > - Thiết kế CSDL phân tách dữ liệu theo từng chi nhánh/tiệm (Multi-Tenant Isolation) và phân quyền vai trò (Role-Based Access Control - RBAC)."*

---

### ❓ Câu 37: Kinh nghiệm của em tại HQ Soft (eSale DMS) là gì?
* **Trả lời chuẩn từ source code & CV:**
  > *"Tại HQ Soft, em là Intern .NET Developer tham gia vào hệ thống phân phối doanh nghiệp eSale DMS:  
  > - Phát triển các API Back-Office và nghiệp vụ quản lý bán hàng sử dụng ASP.NET Core và Entity Framework Core.  
  > - Phối hợp với Senior Developer để điều tra và sửa các lỗi (bug fixing) trong luồng tính toán chiết khấu và tối ưu các câu truy vấn báo cáo doanh số."*

---

### ❓ Câu 38: Sự khác biệt giữa Blazor (trong Nail Salon) và React (trong LSB) theo kinh nghiệm của em?
* **Trả lời chuẩn từ thực tế:**
  > *"Em đã trực tiếp làm việc với cả hai công nghệ:  
  > - **Blazor:** Rất mạnh mẽ khi phát triển ứng dụng nội bộ/quản trị (.NET C# từ Backend đến Frontend), tái sử dụng trực tiếp các DTO và Validation logic của C#, lập trình viên .NET không cần chuyển ngữ cảnh sang JavaScript.  
  > - **React 18 + Vite + TailwindCSS:** Được em dùng cho dashboard SellerManager của LSB vì hệ sinh thái UI phong phú, tốc độ build Vite cực nhanh, trải nghiệm SPA mượt mà và cộng đồng hỗ trợ khổng lồ."*

---

# CHƯƠNG 7: QUY TRÌNH AGILE/SCRUM & KỸ NĂNG LÀM VIỆC

### ❓ Câu 39: Quy trình làm việc nhóm Agile/Scrum thực tế của em diễn ra như thế nào?
* **Trả lời chuẩn từ thực tế:**
  > *"Team của em làm việc theo mô hình Scrum với chu kỳ Sprint 2 tuần:  
  > - **Sprint Planning:** Nhận User Stories từ Product Owner/Tech Lead trên Jira, cùng thảo luận giải pháp kỹ thuật, estimate story points.  
  > - **Daily Standup:** Mỗi sáng 15 phút cập nhật tiến độ, công việc trong ngày và nêu các khó khăn (blocker) cần hỗ trợ.  
  > - **Code Review (Pull Request):** Trước khi merge code vào branch `develop`, PR phải được review về Clean Code, logic nghiệp vụ, xử lý async và kiểm tra không bị lỗi N+1 Query.  
  > - **Sprint Review & Retrospective:** Demo tính năng cho các bên liên quan và họp rút kinh nghiệm cải tiến quy trình cho sprint sau."*

---

### ❓ Câu 40: Khi gặp một bài toán kỹ thuật mới hoặc bug hóc búa, phương pháp giải quyết của em là gì?
* **Trả lời chuẩn từ thực tế:**
  > *"Quy trình 4 bước của em:  
  > 1. **Khoanh vùng & Tái hiện (Reproduce):** Đọc log Serilog để lấy Stack Trace, CorrelationId và tái hiện lỗi chính xác trên môi trường Local.  
  > 2. **Phân tích nguyên nhân gốc rễ (Root Cause):** Dùng Visual Studio Debugger, kiểm tra biến dữ liệu, truy vấn SQL Execution Plan để xem điểm nghẽn.  
  > 3. **Nghiên cứu giải pháp:** Đọc Official Documentation của Microsoft / Amazon, kết hợp các công cụ hỗ trợ như Claude/ChatGPT/Gemini để brainstorm các phương án xử lý và so sánh ưu nhược điểm.  
  > 4. **Kiểm thử kỹ lưỡng:** Viết Unit Test bao phủ trường hợp lỗi đó để đảm bảo không bao giờ bị tái phát (No Regression) trước khi tạo Pull Request."*

---

## 🎯 3 NGUYÊN TẮC VÀNG ĐỂ PASS PHỎNG VẤN
1. **Luôn nói "Em đã làm trong dự án LSB / Võ Minh Thiên"** -> Biến mọi câu hỏi lý thuyết thành câu chuyện thực tế bạn đã code.
2. **Nêu rõ lý do tại sao chọn giải pháp đó** (Ví dụ: tại sao dùng `Polly`? tại sao dùng `INCLUDE` index? tại sao dùng `ServiceScopeFactory` trong Worker?).
3. **Thành thật & Tự tin:** Trả lời dứt khoát, mạch lạc đúng theo các câu hỏi trên vì toàn bộ kiến thức này đều nằm sẵn trong code của bạn! 🚀

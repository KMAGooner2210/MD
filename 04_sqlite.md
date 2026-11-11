# CHƯƠNG 4. SQLITE – CƠ SỞ DỮ LIỆU NHÚNG CHO HỆ THỐNG KIOSK

## Mục tiêu của chương

Sau khi học xong chương này, người đọc cần nắm được:

- SQLite là gì, khác các hệ quản trị cơ sở dữ liệu kiểu máy chủ như MySQL hay PostgreSQL ở điểm nào, và phù hợp với loại ứng dụng nào.
- Các khái niệm nền tảng: mô hình quan hệ, khóa chính, khóa ngoại, ràng buộc, kiểu dữ liệu động của SQLite.
- Cơ chế giao dịch và tính chất ACID, các loại khóa của SQLite, chế độ nhật ký WAL, và cách nhiều tiến trình cùng dùng một tệp cơ sở dữ liệu an toàn.
- Cách làm việc với SQLite từ Python qua mô-đun `sqlite3`: kết nối, truy vấn có tham số, quản lý giao dịch.
- Lý do dự án chọn SQLite.
- Cách SQLite được áp dụng trong dự án: thiết kế lược đồ, vòng đời đơn hàng, chốt đơn và trừ kho trong một giao dịch, cập nhật có điều kiện, mã OTP, báo cáo doanh thu, kiểm thử.
- Các kỹ thuật đã áp dụng, lỗi đã gặp và những hạn chế còn tồn tại, đủ để trình bày và trả lời câu hỏi trước hội đồng.

**Vị trí trong dự án.** Toàn bộ dữ liệu kinh doanh nằm trong một tệp duy nhất `kiosk.db` trên Raspberry Pi 4:
- thực đơn, nguyên liệu, công thức;
- đơn hàng;
- thông tin cửa hàng, mật khẩu quản trị.

| Thành phần | Tệp mã nguồn | Cách dùng cơ sở dữ liệu |
|---|---|---|
| Tầng truy cập dữ liệu | `Smart_Kiosk_Backup/backend/database.py` | Mọi câu lệnh SQL của hệ thống nằm tại đây |
| Ứng dụng kiosk | `Smart_Kiosk_Backup/kiosk_app/bridge.py` | Tạo đơn chờ, chốt đơn, đóng đơn; gọi trực tiếp `database.py` |
| Smart Kiosk API | `Smart_Kiosk_Backup/servers/api_server.py` | Phục vụ quản trị, màn hình bếp, trợ lý giọng nói; gọi `database.py` |
| Dữ liệu mẫu | `Smart_Kiosk_Backup/backend/seed_inventory.py` | Nạp nguyên liệu và công thức mẫu |
| Kiểm thử | `Smart_Kiosk_Backup/tests/test_order_lifecycle.py`, `test_database_path_config.py` | Kiểm thử vòng đời đơn và cấu hình đường dẫn |

Các thành phần khác (phần mềm quản trị trên laptop, màn hình bếp, Kitchen MCP) **không** mở tệp cơ sở dữ liệu. Chúng đi qua Smart Kiosk API bằng HTTP.

Phiên bản: thư viện SQLite 3.37 trở lên (đi kèm Python 3.10 trên Ubuntu 22.04; Raspberry Pi OS Bookworm đi kèm bản mới hơn), mô-đun `sqlite3` của thư viện chuẩn Python.

---

## 4.1. Khái niệm và đặc điểm

### 4.1.1. Cơ sở dữ liệu quan hệ

Trước khi nói về SQLite, cần nhắc lại một số khái niệm nền tảng.

**Cơ sở dữ liệu** là tập hợp dữ liệu có tổ chức, được lưu trữ bền vững và truy xuất được một cách hiệu quả. **Hệ quản trị cơ sở dữ liệu** (DBMS) là phần mềm quản lý tập dữ liệu đó: lưu trữ, truy vấn, bảo đảm toàn vẹn, xử lý truy cập đồng thời và phục hồi sau sự cố.

**Mô hình quan hệ** do Edgar F. Codd (IBM) đề xuất năm 1970. Trong mô hình này, dữ liệu được tổ chức thành các **bảng** (quan hệ):
- mỗi bảng gồm nhiều **dòng** (bản ghi);
- mỗi dòng gồm nhiều **cột** (thuộc tính).

Các bảng liên kết với nhau qua giá trị dữ liệu chứ không qua con trỏ. Ngôn ngữ dùng để làm việc với cơ sở dữ liệu quan hệ là **SQL** (*Structured Query Language*), gồm các nhóm lệnh:
- định nghĩa dữ liệu: `CREATE`, `ALTER`, `DROP`;
- thao tác dữ liệu: `SELECT`, `INSERT`, `UPDATE`, `DELETE`;
- điều khiển giao dịch: `BEGIN`, `COMMIT`, `ROLLBACK`.

Một số khái niệm cần nắm chắc:

| Khái niệm | Ý nghĩa | Ví dụ trong dự án |
|---|---|---|
| **Khóa chính** (*primary key*) | Cột hoặc nhóm cột xác định duy nhất mỗi dòng | `orders.order_id` = `DH12345678` |
| **Khóa chính phức hợp** | Khóa chính gồm nhiều cột | `recipe(product_id, ingredient_id)`: mỗi món chỉ có một định mức cho mỗi nguyên liệu |
| **Khóa ngoại** (*foreign key*) | Cột tham chiếu tới khóa chính của bảng khác, bảo đảm không trỏ tới dòng không tồn tại | `recipe.ingredient_id` → `ingredient.id` |
| **Ràng buộc** (*constraint*) | Quy tắc dữ liệu phải thỏa mãn: `NOT NULL`, `UNIQUE`, `CHECK`, `DEFAULT` | `settings.id CHECK (id = 1)` |
| **Phép nối** (*join*) | Ghép dòng của hai bảng theo điều kiện | Lấy định mức kèm tên nguyên liệu: `recipe JOIN ingredient` |
| **Chỉ mục** (*index*) | Cấu trúc phụ giúp tìm kiếm nhanh, đổi lại tốn chỗ và làm chậm thao tác ghi | Khóa chính tự có chỉ mục |

### 4.1.2. SQLite là gì

**SQLite** là một hệ quản trị cơ sở dữ liệu quan hệ **nhúng** (*embedded*), viết bằng ngôn ngữ C. Dự án do D. Richard Hipp khởi xướng năm 2000; phần mã nguồn được đặt vào **phạm vi công cộng** (*public domain*), nên ai cũng có thể dùng cho bất kỳ mục đích nào mà không cần giấy phép.

Điểm khác biệt cốt lõi của SQLite nằm ở chữ "nhúng":

```
  MySQL / PostgreSQL (kiểu máy chủ)            SQLite (kiểu nhúng)

  Ứng dụng ──TCP/socket──► Tiến trình máy chủ    Ứng dụng
                           cơ sở dữ liệu          └─ thư viện SQLite (chạy trong
                              │                      cùng tiến trình ứng dụng)
                              ▼                          │ đọc/ghi trực tiếp
                           Các tệp dữ liệu               ▼
                                                     kiosk.db (một tệp duy nhất)
```

Với MySQL hay PostgreSQL, cơ sở dữ liệu là **một chương trình máy chủ riêng**, luôn chạy nền. Ứng dụng gửi câu lệnh SQL qua mạng hoặc socket và nhận kết quả về. Với SQLite **không có máy chủ nào cả**: SQLite là một thư viện được liên kết vào chính ứng dụng, đọc và ghi trực tiếp lên **một tệp duy nhất** trên đĩa. Chính tác giả mô tả: SQLite không cạnh tranh với MySQL hay PostgreSQL, mà cạnh tranh với lệnh `fopen()`, tức là thay thế cho việc tự lưu dữ liệu vào tệp.

### 4.1.3. Đặc điểm nổi bật

**Không cần máy chủ, không cần cài đặt, không cần cấu hình.** Không có tiến trình nền, không có tài khoản, không có cổng mạng. Python có sẵn mô-đun `sqlite3` trong thư viện chuẩn: chỉ cần `import sqlite3` là dùng được.

**Một tệp chứa toàn bộ cơ sở dữ liệu.** Mọi bảng, chỉ mục và dữ liệu nằm trong một tệp. Sao lưu là chép tệp, chuyển hệ thống sang máy khác cũng là chép tệp. Định dạng tệp ổn định và tương thích giữa các nền tảng (x86, ARM, 32 hay 64 bit), được cam kết hỗ trợ đến năm 2050. Tệp `kiosk.db` tạo trên laptop có thể chép sang Raspberry Pi và dùng ngay.

**Giao dịch đầy đủ tính chất ACID.** Kể cả khi mất điện giữa chừng, cơ sở dữ liệu không bao giờ ở trạng thái "ghi được một nửa" (mục 4.2.2).

**Nhỏ gọn và cực kỳ phổ biến.** Thư viện chỉ khoảng 600–900 KB. SQLite có mặt trong mọi điện thoại Android và iPhone, trong các trình duyệt Chrome và Firefox, trong Python và PHP, trong máy bay, ô tô và vô số thiết bị nhúng. Đây được xem là phần mềm cơ sở dữ liệu được triển khai nhiều nhất thế giới.

**Độ tin cậy rất cao.** Bộ kiểm thử của SQLite lớn hơn mã nguồn hàng trăm lần và đạt độ bao phủ nhánh 100%. SQLite được dùng trong các hệ thống hàng không, nơi yêu cầu độ tin cậy khắt khe nhất.

**Kiểu dữ liệu động.** Khác với hầu hết hệ quản trị khác, kiểu dữ liệu trong SQLite gắn với **giá trị** chứ không gắn chặt với **cột** (mục 4.2.1).

Bên cạnh đó, SQLite có những **hạn chế** cần biết:
- **Chỉ một bên ghi tại một thời điểm.** Nhiều tiến trình đọc được cùng lúc, nhưng chỉ một tiến trình được ghi tại một thời điểm. Ứng dụng có hàng nghìn người dùng ghi đồng thời không phù hợp.
- **Không truy cập qua mạng.** Tệp cơ sở dữ liệu phải nằm trên ổ đĩa cục bộ của máy chạy ứng dụng. Đặt tệp trên ổ mạng chia sẻ (NFS, SMB) dễ gây hỏng dữ liệu, vì cơ chế khóa tệp qua mạng không đáng tin cậy.
- **Không có quản lý người dùng và phân quyền.** Ai đọc được tệp là đọc được toàn bộ dữ liệu; bảo mật dựa vào quyền truy cập tệp của hệ điều hành.
- **`ALTER TABLE` hạn chế.** Thêm cột thì dễ; các thay đổi khác thường phải tạo bảng mới rồi chép dữ liệu sang.

| Đặc điểm | Ý nghĩa với dự án |
|---|---|
| Nhúng, không máy chủ | Không phải cài và vận hành thêm dịch vụ trên Raspberry Pi |
| Một tệp duy nhất | Sao lưu, chuyển từ laptop sang Pi chỉ bằng một lệnh `scp` |
| ACID | Chốt đơn và trừ kho không bao giờ "nửa vời" khi mất điện |
| Nhẹ | Không tranh tài nguyên với giao diện kiosk |
| Một bên ghi tại một thời điểm | Đủ cho một quầy bán hàng; xử lý bằng WAL và chờ khóa |
| Chỉ dùng cục bộ | Các máy khác truy cập qua Smart Kiosk API chứ không mở tệp trực tiếp |

---

## 4.2. Nguyên lý hoạt động

### 4.2.1. Kiểu dữ liệu và lớp lưu trữ

Trong MySQL, một cột khai báo `INT` chỉ chứa được số nguyên. SQLite linh hoạt hơn: mỗi **giá trị** thuộc một trong năm **lớp lưu trữ** (*storage class*):

| Lớp lưu trữ | Nội dung |
|---|---|
| `NULL` | Giá trị rỗng |
| `INTEGER` | Số nguyên có dấu, 1–8 byte tùy độ lớn |
| `REAL` | Số thực dấu phẩy động 8 byte (IEEE 754) |
| `TEXT` | Chuỗi văn bản (UTF-8) |
| `BLOB` | Dữ liệu nhị phân, lưu nguyên dạng |

Kiểu khai báo của cột chỉ tạo ra một **ái lực kiểu** (*type affinity*), tức là "kiểu ưu tiên". SQLite cố đổi giá trị sang kiểu ưu tiên khi có thể, nhưng không bắt buộc. Ví dụ, cột khai báo `INTEGER` nhận chuỗi `'35000'` sẽ tự đổi thành số 35000, còn nhận chuỗi `'abc'` thì vẫn lưu nguyên chuỗi.

Hệ quả thực tế có hai điểm:
- SQLite **không có kiểu `BOOLEAN` và `DATETIME` riêng**. Cột khai báo `BOOLEAN` thực chất lưu số nguyên 0 hoặc 1. Thời gian lưu dưới dạng chuỗi `'YYYY-MM-DD HH:MM:SS'`, số thực (ngày Julius) hoặc số nguyên (giây Unix). Dự án dùng dạng chuỗi, và xử lý bằng các hàm `datetime()`, `date()`, `strftime()` của SQLite.
- Kiểm tra kiểu dữ liệu đầu vào là **trách nhiệm của ứng dụng**. Đây là lý do tầng dữ liệu của dự án có hàm chuẩn hóa đầu vào riêng (mục 4.4.3).

Từ phiên bản 3.37, SQLite hỗ trợ **bảng nghiêm ngặt** (*STRICT table*), kiểm tra kiểu như các hệ quản trị khác. Dự án chưa dùng tính năng này.

### 4.2.2. Giao dịch và tính chất ACID

**Giao dịch** (*transaction*) là một nhóm thao tác được thực hiện như **một khối không thể chia cắt**: hoặc tất cả cùng thành công, hoặc như chưa có gì xảy ra. Giao dịch bắt đầu bằng `BEGIN`, kết thúc bằng `COMMIT` (ghi nhận) hoặc `ROLLBACK` (hủy bỏ).

Ví dụ kinh điển trong dự án: khi khách thanh toán xong, hệ thống phải (1) đổi trạng thái đơn thành đã thanh toán và (2) trừ nguyên liệu theo công thức. Nếu bước 1 xong mà mất điện trước bước 2, kho sẽ sai. Nếu bước 2 xong mà bước 1 lỗi, kho bị trừ cho một đơn không tồn tại. Đặt cả hai bước trong một giao dịch bảo đảm điều này không bao giờ xảy ra.

Một hệ quản trị đáng tin cậy bảo đảm bốn tính chất, viết tắt là **ACID**:

| Tính chất | Ý nghĩa | Trong dự án |
|---|---|---|
| **A**tomicity – Nguyên tử | Tất cả hoặc không có gì | Chốt đơn và trừ kho cùng thành công hoặc cùng bị hủy |
| **C**onsistency – Nhất quán | Giao dịch đưa dữ liệu từ trạng thái hợp lệ sang trạng thái hợp lệ, không vi phạm ràng buộc | Không có định mức trỏ tới nguyên liệu không tồn tại (khóa ngoại) |
| **I**solation – Cô lập | Các giao dịch đồng thời không thấy trạng thái dở dang của nhau | API không đọc được một đơn "đã PAID nhưng chưa trừ kho" |
| **D**urability – Bền vững | Đã `COMMIT` thì dữ liệu còn nguyên kể cả khi mất điện ngay sau đó | Đơn đã báo thành công là không bao giờ mất |

### 4.2.3. Cơ chế khóa và các loại giao dịch

SQLite khóa ở mức **toàn bộ tệp cơ sở dữ liệu**, không khóa từng dòng như PostgreSQL. Ở chế độ nhật ký mặc định, một kết nối đi qua các mức khóa:

```
 KHÔNG KHÓA ──đọc──► SHARED ──định ghi──► RESERVED ──chuẩn bị commit──► EXCLUSIVE
 (nhiều kết nối      (nhiều kết nối       (chỉ MỘT kết nối giữ;        (chỉ một; không ai
  cùng lúc)           cùng đọc)            người khác vẫn đọc được)      khác được đọc)
```

Câu lệnh `BEGIN` có ba biến thể, khác nhau ở thời điểm lấy khóa:

| Lệnh | Lấy khóa khi nào | Đặc điểm |
|---|---|---|
| `BEGIN DEFERRED` (mặc định) | Chưa lấy khóa; lấy khóa đọc khi đọc lần đầu, khóa ghi khi ghi lần đầu | Có thể gặp lỗi "database is locked" **giữa chừng** khi đang đọc mà muốn chuyển sang ghi |
| `BEGIN IMMEDIATE` | Lấy **ngay** khóa ghi (RESERVED) | Từ lúc đó không ai khác ghi được; người khác vẫn đọc được |
| `BEGIN EXCLUSIVE` | Lấy ngay khóa độc quyền | Không ai khác đọc hay ghi được (ở chế độ WAL tương đương IMMEDIATE) |

Để hiểu vì sao dự án dùng `BEGIN IMMEDIATE`, xét mẫu **đọc – kiểm tra – ghi** (*read – check – write*), ví dụ "đọc trạng thái đơn, nếu còn PENDING thì chốt PAID". Với `BEGIN DEFERRED` có thể xảy ra tình huống sau:

```
 Kết nối A (webhook báo tiền về)        Kết nối B (khách bấm hủy)
 BEGIN (deferred)
 SELECT status → PENDING                 BEGIN (deferred)
                                         SELECT status → PENDING
 UPDATE … SET PAID   ← lấy khóa ghi
                                         UPDATE … SET CANCELLED → chờ khóa…
 COMMIT
                                         → hoặc lỗi "database is locked",
                                           hoặc ghi đè dựa trên dữ liệu đã cũ
```

Cả hai kết nối đều "thấy" đơn đang PENDING và đều ra quyết định dựa trên thông tin đó. Đây gọi là **tình trạng tranh chấp** (*race condition*). Với `BEGIN IMMEDIATE`, kết nối nào bắt đầu trước giữ khóa ghi ngay từ đầu. Kết nối kia phải **chờ đến khi giao dịch đầu kết thúc** mới bắt đầu được, và khi đó nó đọc được trạng thái mới nhất. Việc đọc – kiểm tra – ghi trở thành một khối nguyên tử thực sự.

### 4.2.4. Chế độ nhật ký WAL

Để bảo đảm tính nguyên tử và bền vững, SQLite cần một **nhật ký** (*journal*) phục vụ phục hồi khi sự cố. Có hai cơ chế chính.

**Nhật ký hoàn tác** (*rollback journal*, mặc định). Trước khi sửa một trang dữ liệu, SQLite chép **bản gốc** của trang sang tệp nhật ký `kiosk.db-journal`, rồi ghi đè trực tiếp lên tệp chính. Nếu mất điện giữa chừng, lần mở tiếp theo SQLite dùng nhật ký để chép bản gốc trở lại. Nhược điểm: trong lúc ghi, tệp chính đang bị sửa nên **người đọc phải chờ**; đọc và ghi chặn nhau.

**Nhật ký ghi trước** (*Write-Ahead Logging – WAL*). Cơ chế đảo ngược: thay đổi **không** ghi vào tệp chính mà được **nối thêm** vào tệp `kiosk.db-wal`.

```
                   ┌──────────────┐
 Bên ghi ─append──►│ kiosk.db-wal │  thay đổi mới nối vào cuối
                   └──────┬───────┘
                          │ checkpoint (định kỳ): chép các trang mới về tệp chính
                          ▼
                   ┌──────────────┐
 Bên đọc ─────────►│  kiosk.db    │  + đọc thêm phần đã commit trong tệp WAL
                   └──────────────┘
   kiosk.db-shm: bộ nhớ dùng chung, làm chỉ mục cho tệp WAL
```

Người đọc đọc tệp chính kết hợp phần đã commit trong tệp WAL, tại **một thời điểm nhất định**: mỗi người đọc thấy một "ảnh chụp" nhất quán của dữ liệu. Định kỳ, SQLite thực hiện **checkpoint**, chép các trang đã commit từ tệp WAL về tệp chính. Mặc định việc này diễn ra khi tệp WAL đạt khoảng 1000 trang.

| | Rollback journal | WAL |
|---|---|---|
| Đọc trong lúc có người ghi | Phải chờ | **Đọc song song được** |
| Số bên ghi đồng thời | Một | Một |
| Tốc độ ghi | Chậm hơn (ghi hai nơi, đồng bộ đĩa nhiều lần) | Nhanh hơn (chủ yếu nối tiếp vào cuối tệp) |
| Tệp phụ | `-journal`, chỉ tồn tại khi đang ghi | `-wal` và `-shm`, tồn tại khi có kết nối mở |
| Yêu cầu | – | Mọi tiến trình trên **cùng một máy**, không dùng qua ổ mạng |

Câu lệnh `PRAGMA journal_mode=WAL` chuyển cơ sở dữ liệu sang chế độ WAL. Thiết lập này **được lưu vào tệp**, nên chỉ cần đặt một lần, nhưng đặt lại mỗi lần kết nối cũng vô hại.

Lưu ý khi sao lưu: ở chế độ WAL, một phần dữ liệu mới nhất có thể còn nằm trong tệp `-wal`. Chép tệp `.db` khi ứng dụng đang chạy có thể **thiếu dữ liệu**. Cách an toàn là dừng ứng dụng trước khi chép, hoặc dùng lệnh sao lưu của SQLite (`sqlite3 kiosk.db ".backup ban_sao.db"`).

### 4.2.5. Chờ khóa và lỗi "database is locked"

Khi một kết nối muốn ghi mà kết nối khác đang giữ khóa ghi, SQLite mặc định báo lỗi **ngay lập tức**: `database is locked`. Thiết lập **thời gian chờ bận** (*busy timeout*) yêu cầu SQLite tự thử lại trong một khoảng thời gian trước khi bỏ cuộc:

```sql
PRAGMA busy_timeout = 10000;   -- chờ tối đa 10 giây
```

Trong Python, tham số `timeout` của `sqlite3.connect()` có tác dụng tương tự. Vì các giao dịch của dự án chỉ kéo dài vài mili giây, khoảng chờ 10 giây gần như loại bỏ hoàn toàn lỗi khóa khi ứng dụng kiosk và Smart Kiosk API cùng ghi.

### 4.2.6. Khóa ngoại trong SQLite

Vì lý do tương thích ngược, SQLite **mặc định không kiểm tra khóa ngoại**: khai báo `FOREIGN KEY` chỉ được ghi nhận chứ không được thi hành. Mỗi kết nối phải tự bật:

```sql
PRAGMA foreign_keys = ON;
```

Khi đã bật, SQLite từ chối thêm định mức trỏ tới nguyên liệu không tồn tại. SQLite cũng từ chối **xóa** một dòng đang được dòng khác tham chiếu, trừ khi khóa ngoại khai báo hành vi kèm theo như `ON DELETE CASCADE` (xóa theo) hoặc `ON DELETE SET NULL`. Mục 4.5.3 sẽ chỉ ra một lỗi thực tế liên quan đến chi tiết này.

### 4.2.7. Làm việc với SQLite từ Python

Mô-đun `sqlite3` của thư viện chuẩn Python tuân theo đặc tả DB-API 2.0 (PEP 249). Ví dụ tối giản:

```python
import sqlite3

conn = sqlite3.connect("demo.db", timeout=10)    # mở (hoặc tạo) tệp
conn.row_factory = sqlite3.Row                   # dòng trả về truy cập được theo tên cột
conn.execute("PRAGMA foreign_keys=ON")

conn.execute("CREATE TABLE IF NOT EXISTS mon (id INTEGER PRIMARY KEY, ten TEXT, gia INTEGER)")
conn.execute("INSERT INTO mon (ten, gia) VALUES (?, ?)", ("Bánh mì", 35000))   # tham số ?
conn.commit()                                    # ghi nhận giao dịch

for row in conn.execute("SELECT * FROM mon WHERE gia >= ?", (30000,)):
    print(row["ten"], row["gia"])                # nhờ sqlite3.Row
conn.close()
```

Có bốn điểm quan trọng.

**Truy vấn có tham số.** Giá trị luôn được truyền qua dấu `?` và bộ tham số, **không bao giờ** ghép chuỗi. Giả sử viết `f"SELECT * FROM orders WHERE order_id = '{ma}'"` và người dùng nhập `' OR '1'='1`. Câu lệnh trở thành `... WHERE order_id = '' OR '1'='1'` và trả về **mọi** đơn hàng. Đây là lỗ hổng **chèn SQL** (*SQL injection*), một trong những lỗ hổng phổ biến và nguy hiểm nhất của ứng dụng web. Với tham số `?`, giá trị được gửi tách biệt khỏi câu lệnh, nên luôn được hiểu là dữ liệu, không bao giờ là mã SQL.

**`sqlite3.Row`.** Mặc định mỗi dòng trả về là một bộ (*tuple*), chỉ truy cập được theo vị trí `row[0]`. Đặt `row_factory = sqlite3.Row` cho phép viết `row["payment_status"]` dễ đọc hơn, và đổi sang từ điển bằng `dict(row)` để trả về JSON.

**Quản lý giao dịch ngầm định.** Ở chế độ mặc định, Python **tự động chèn `BEGIN`** (kiểu DEFERRED) trước mỗi câu lệnh `INSERT`, `UPDATE` hay `DELETE` nếu chưa có giao dịch nào mở. Giao dịch kéo dài đến khi gọi `commit()` hoặc `rollback()`. Hai hệ quả:
- **Quên `commit()` thì mọi thay đổi bị mất** khi đóng kết nối.
- Muốn dùng `BEGIN IMMEDIATE` thì phải **tự gọi** `conn.execute("BEGIN IMMEDIATE")` trước câu lệnh đầu tiên, như dự án đã làm.

**`rowcount`.** Sau `UPDATE` hay `DELETE`, thuộc tính `cursor.rowcount` cho biết số dòng bị ảnh hưởng. Đây là công cụ then chốt của kỹ thuật **cập nhật có điều kiện**: câu lệnh `UPDATE ... WHERE id = ? AND trang_thai = 'PENDING'` chỉ đổi được dòng nếu trạng thái còn đúng. `rowcount == 1` nghĩa là "tôi là người thắng", `rowcount == 0` nghĩa là "trạng thái đã bị người khác thay đổi trước".

---

## 4.3. Lý do lựa chọn

Có ba hướng chính để lưu dữ liệu cho hệ thống kiosk:

| Tiêu chí | SQLite | MySQL / PostgreSQL | Tệp JSON / CSV tự quản lý |
|---|---|---|---|
| Cài đặt, vận hành | Không cần | Cài máy chủ, tạo tài khoản, cấu hình, theo dõi dịch vụ | Không cần |
| Tài nguyên trên Raspberry Pi | Rất nhẹ | Nặng hơn đáng kể (máy chủ chạy thường trực) | Nhẹ |
| Giao dịch ACID | Có | Có | Không: mất điện giữa lúc ghi là hỏng tệp |
| Truy vấn, lọc, thống kê | SQL đầy đủ | SQL đầy đủ | Tự viết bằng tay |
| Nhiều tiến trình cùng dùng | Được (một bên ghi tại một thời điểm) | Được, ghi đồng thời mức dòng | Rất dễ ghi đè lẫn nhau |
| Truy cập từ máy khác | Không trực tiếp | Có, qua mạng | Không |
| Sao lưu, di chuyển | Chép một tệp | Công cụ xuất, nhập dữ liệu | Chép tệp |

Dự án chọn **SQLite** vì năm lý do.

*Thứ nhất, quy mô phù hợp.* Hệ thống phục vụ **một cửa hàng với một quầy kiosk**: vài chục đến vài trăm đơn mỗi ngày, mỗi thao tác ghi kéo dài vài mili giây. Giới hạn "một bên ghi tại một thời điểm" của SQLite không bao giờ chạm tới ở mức tải này. Một hệ quản trị kiểu máy chủ sẽ là dùng "dao mổ trâu để giết gà".

*Thứ hai, không thêm gánh nặng vận hành.* Chủ quán không phải là người quản trị hệ thống. SQLite không có dịch vụ nào cần khởi động, giám sát hay cập nhật bảo mật, không có mật khẩu cơ sở dữ liệu cần quản lý.

*Thứ ba, tiết kiệm tài nguyên Raspberry Pi.* Pi 4 đã phải chạy giao diện kiosk viết bằng Qt, Smart Kiosk API và webhook. SQLite chạy ngay trong các tiến trình đó, không tốn thêm tiến trình nào.

*Thứ tư, vẫn đầy đủ sức mạnh của cơ sở dữ liệu quan hệ.* Dự án cần giao dịch ACID cho nghiệp vụ chốt đơn và trừ kho, cần phép nối giữa công thức và nguyên liệu, cần thống kê doanh thu theo ngày, tháng, năm. Lưu bằng tệp JSON tự quản lý sẽ phải tự viết lại tất cả những thứ này, và không an toàn khi mất điện.

*Thứ năm, dễ triển khai và sao lưu.* Chuyển hệ thống từ laptop sang Raspberry Pi chỉ cần chép tệp `kiosk.db`. Trong quá trình phát triển, việc này đã thực hiện đúng bằng một lệnh `scp`.

Giới hạn "chỉ dùng cục bộ" được giải quyết ở mức kiến trúc. Tệp cơ sở dữ liệu nằm trên Raspberry Pi; chỉ hai tiến trình trên Pi là ứng dụng kiosk và Smart Kiosk API mở tệp trực tiếp. Mọi thành phần trên máy khác đều đi qua **Smart Kiosk API**. Thiết kế này còn có lợi về bảo mật: các thành phần bên ngoài không bao giờ chạm trực tiếp vào dữ liệu, mọi truy cập đều qua lớp xác thực bằng khóa API (chương Flask).

**Khi nào nên chuyển sang PostgreSQL?** Khi hệ thống mở rộng thành nhiều chi nhánh dùng chung một cơ sở dữ liệu, hoặc nhiều kiosk ghi đồng thời với lưu lượng lớn. Nhờ toàn bộ câu lệnh SQL tập trung trong một tệp `database.py`, việc chuyển đổi chỉ ảnh hưởng tới tầng dữ liệu.

---

## 4.4. Ứng dụng và code trong dự án

### 4.4.1. Kết nối và cấu hình

Mọi hàm trong tầng dữ liệu đều lấy kết nối qua một hàm chung, bảo đảm mọi kết nối có cùng cấu hình:

```python
# Smart_Kiosk_Backup/backend/database.py
def resolve_database_path():
    configured_path = os.environ.get("KIOSK_DB_PATH")
    if configured_path is None:
        return os.path.join(CURRENT_DIR, "kiosk.db")       # mặc định: backend/kiosk.db
    normalized_path = configured_path.strip()
    if not normalized_path:
        raise RuntimeError("KIOSK_DB_PATH must not be empty")
    if not os.path.isabs(normalized_path):
        raise RuntimeError("KIOSK_DB_PATH must be an absolute path")
    if not os.path.isdir(os.path.dirname(normalized_path)):
        raise RuntimeError("KIOSK_DB_PATH parent directory does not exist")
    return os.path.realpath(normalized_path)


DB_PATH = resolve_database_path()

def get_connection():
    conn = sqlite3.connect(DB_PATH, timeout=10)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA journal_mode=WAL")       # đọc song song với ghi
    conn.execute("PRAGMA busy_timeout=10000")     # chờ khóa tối đa 10 giây
    conn.execute("PRAGMA foreign_keys=ON")        # thi hành khóa ngoại
    return conn
```

**Đường dẫn cơ sở dữ liệu cấu hình được nhưng kiểm tra chặt.** Biến môi trường `KIOSK_DB_PATH` cho phép đặt tệp ở vị trí khác, ví dụ một thẻ nhớ riêng. Hàm từ chối:
- chuỗi rỗng;
- đường dẫn tương đối, vì nó phụ thuộc thư mục làm việc hiện tại. Ứng dụng kiosk và API khởi động từ hai thư mục khác nhau sẽ vô tình mở hai tệp khác nhau, và dữ liệu bị "chia đôi" mà không ai hay;
- thư mục cha không tồn tại.

Nguyên tắc ở đây là **báo lỗi sớm và rõ ràng** (*fail fast*): cấu hình sai thì dừng ngay lúc khởi động, thay vì chạy với một tệp trống mới tạo.

**Ba câu lệnh PRAGMA** áp dụng đúng các cơ chế ở mục 4.2:
- **WAL** cho phép Smart Kiosk API đọc thực đơn hay đơn chờ ngay cả khi ứng dụng kiosk đang ghi đơn.
- **Busy timeout** cho phép hai tiến trình cùng ghi mà không báo lỗi khóa.
- **Khóa ngoại** phải bật lại ở mỗi kết nối, vì đây là thiết lập theo kết nối chứ không lưu vào tệp.

Dự án mở **kết nối mới cho mỗi thao tác** rồi đóng ngay. Với SQLite, mở kết nối rất rẻ vì chỉ là mở tệp, không có bắt tay mạng như MySQL. Cách làm này cũng tránh được rắc rối khi dùng chung kết nối giữa nhiều luồng: mô-đun `sqlite3` mặc định không cho một kết nối được dùng từ luồng khác luồng đã tạo ra nó.

### 4.4.2. Thiết kế lược đồ

Cơ sở dữ liệu gồm năm bảng:

```
 product (thực đơn)               recipe (định mức)                ingredient (kho)
 ┌──────────────────┐            ┌──────────────────────┐         ┌──────────────────────┐
 │ id          PK   │◄───────────│ product_id     PK,FK │         │ id              PK   │
 │ name             │            │ ingredient_id  PK,FK │────────►│ name                 │
 │ price  (INTEGER) │            │ qty_per_unit  (REAL) │         │ unit                 │
 │ image, category  │            └──────────────────────┘         │ quantity      (REAL) │
 │ status (0/1)     │                                             │ low_stock_threshold  │
 └──────────────────┘                                             └──────────────────────┘

 orders (đơn hàng)                                 settings (cấu hình, đúng MỘT dòng)
 ┌───────────────────────────────────────┐        ┌──────────────────────────────────────┐
 │ order_id      PK  (DH + 8 chữ số)     │        │ id PK CHECK (id = 1)                 │
 │ total_amount, paid_amount  (INTEGER)  │        │ store_name, store_address, …         │
 │ items_json    (TEXT: ảnh chụp món)    │        │ admin_password_hash                  │
 │ payment_status  PENDING/PAID/…        │        │ recovery_email                       │
 │ kitchen_status  pending/done          │        │ reset_otp_code, reset_otp_expires_at │
 │ created_at, closed_at                 │        │ reset_otp_attempts                   │
 └───────────────────────────────────────┘        └──────────────────────────────────────┘
```

Câu lệnh tạo bảng (trích):

```python
cursor.execute('''
    CREATE TABLE IF NOT EXISTS ingredient (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        unit TEXT NOT NULL,
        quantity REAL NOT NULL DEFAULT 0,
        low_stock_threshold REAL NOT NULL DEFAULT 0
    )
''')

cursor.execute('''
    CREATE TABLE IF NOT EXISTS recipe (
        product_id INTEGER NOT NULL,
        ingredient_id INTEGER NOT NULL,
        qty_per_unit REAL NOT NULL,
        PRIMARY KEY (product_id, ingredient_id),
        FOREIGN KEY (product_id) REFERENCES product(id),
        FOREIGN KEY (ingredient_id) REFERENCES ingredient(id)
    )
''')

cursor.execute('''
    CREATE TABLE IF NOT EXISTS settings (
        id INTEGER PRIMARY KEY CHECK (id = 1),
        store_name TEXT DEFAULT '',
        ...
        admin_password_hash TEXT
    )
''')
```

Các quyết định thiết kế đáng chú ý:

**Bảng `recipe` là bảng trung gian của quan hệ nhiều – nhiều.** Một món dùng nhiều nguyên liệu, và một nguyên liệu dùng cho nhiều món. Bảng trung gian giải quyết quan hệ này. Mỗi dòng có thêm thuộc tính `qty_per_unit`, là lượng nguyên liệu cho một phần món, ví dụ 1 ổ bánh mì, 30 g pate. Khóa chính phức hợp `(product_id, ingredient_id)` bảo đảm mỗi cặp món – nguyên liệu chỉ có một định mức.

**Tiền lưu bằng số nguyên.** Giá món, tổng tiền và số tiền thực nhận đều là `INTEGER` theo đơn vị đồng. Không bao giờ lưu tiền bằng số thực: số dấu phẩy động không biểu diễn chính xác được mọi giá trị thập phân. Ví dụ `0.1 + 0.2` cho ra `0.30000000000000004`, và cộng dồn nhiều lần sẽ sai lệch. Tiền Việt Nam không có phần lẻ, nên số nguyên là lựa chọn tự nhiên. Ngược lại, lượng nguyên liệu như 0,5 kg hay 30 ml dùng `REAL` vì cần phần thập phân và chấp nhận sai số nhỏ.

**Bảng cấu hình chỉ có một dòng.** Ràng buộc `CHECK (id = 1)` bảo đảm bảng `settings` không bao giờ có dòng thứ hai. Đây là cách cài đặt mẫu **singleton** ngay ở tầng cơ sở dữ liệu.

**Danh sách món của đơn lưu dạng JSON trong một cột văn bản.** Đây là một quyết định **phi chuẩn hóa** (*denormalization*) có chủ đích. Theo lý thuyết chuẩn hóa, lẽ ra cần thêm bảng `order_item`. Tuy nhiên, chi tiết đơn là một **ảnh chụp tại thời điểm bán**: tên món, giá, kích cỡ, lượng đá, ghi chú. Nếu tháng sau quản trị viên đổi giá hay đổi tên món, đơn cũ vẫn phải giữ nguyên giá và tên lúc bán. Lưu JSON giữ được ảnh chụp đó một cách tự nhiên, và mỗi lần đọc đơn chỉ cần một câu `SELECT`. Đổi lại, thống kê theo món phải phân tích JSON bằng Python (mục 4.4.7).

**Mật khẩu chỉ lưu dạng băm.** Cột `admin_password_hash` chứa chuỗi băm có muối do `werkzeug.security.generate_password_hash` tạo ra, không bao giờ chứa mật khẩu gốc.

**Nâng cấp lược đồ bằng `ALTER TABLE`.** Khi tính năng mới cần thêm cột, hàm khởi tạo thử thêm cột và bỏ qua lỗi nếu cột đã có:

```python
for alter_sql in [
    "ALTER TABLE settings ADD COLUMN recovery_email TEXT DEFAULT ''",
    "ALTER TABLE settings ADD COLUMN reset_otp_attempts INTEGER DEFAULT 0",
    "ALTER TABLE orders ADD COLUMN paid_amount INTEGER",
    "ALTER TABLE orders ADD COLUMN closed_at DATETIME",
    ...
]:
    try:
        cursor.execute(alter_sql)
        conn.commit()
    except sqlite3.OperationalError:
        pass  # Cột đã tồn tại từ lần khởi tạo trước
```

Nhờ vậy tệp `kiosk.db` cũ được tự động nâng cấp khi chạy phiên bản mã mới mà không mất dữ liệu. Đây là một dạng **di trú lược đồ** (*schema migration*) đơn giản; mục 4.5.3 bàn về cách làm bài bản hơn.

### 4.4.3. Vòng đời đơn hàng như một máy trạng thái

Trạng thái thanh toán của đơn được thiết kế như một **máy trạng thái hữu hạn**:

```
                 khách bấm thanh toán
                         │
                         ▼
                   ┌──────────┐   tiền về đủ khi khách còn chờ    ┌──────┐
                   │ PENDING  │──────────────────────────────────►│ PAID │──► trừ kho, gửi bếp
                   └──────────┘                                   └──────┘
           khách hủy │      │ quá 5 phút / kiosk khởi động lại
                     ▼      ▼
             ┌───────────┐ ┌─────────┐
             │ CANCELLED │ │ EXPIRED │
             └─────┬─────┘ └────┬────┘
                   └─────┬──────┘
                         │ tiền về SAU khi đơn đã đóng
                         ▼
                   ┌───────────┐
                   │ LATE_PAID │──► KHÔNG trừ kho, KHÔNG gửi bếp; quản trị viên hoàn tiền
                   └───────────┘
```

```python
# Smart_Kiosk_Backup/backend/database.py
ORDER_PENDING = "PENDING"
ORDER_PAID = "PAID"
ORDER_CANCELLED = "CANCELLED"
ORDER_EXPIRED = "EXPIRED"
ORDER_LATE_PAID = "LATE_PAID"
CLOSED_ORDER_STATUSES = (ORDER_CANCELLED, ORDER_EXPIRED)
```

Thiết kế này xuất phát từ một tình huống có thật của thanh toán chuyển khoản: khách quét mã QR, mở ứng dụng ngân hàng, nhưng chuyển tiền **sau khi** đã bấm hủy hoặc sau khi mã QR hết hạn. Tiền đã vào tài khoản cửa hàng, nhưng khách đã rời đi. Trong phiên bản đầu, khoản tiền này không được ghi nhận ở đâu cả. Với máy trạng thái mới:
- **Đơn được ghi vào cơ sở dữ liệu ngay khi tạo mã QR**, ở trạng thái PENDING, chưa trừ kho và chưa gửi bếp. Như vậy mọi khoản tiền về sau đều đối chiếu được với một đơn có thật.
- Nếu tiền về khi đơn đã đóng, đơn chuyển sang **LATE_PAID** để quản trị viên biết mà hoàn tiền, và tuyệt đối không được gửi xuống bếp.

Tạo đơn chờ dùng một câu `INSERT` đơn giản. Khóa chính `order_id` bảo đảm không có hai đơn trùng mã: chèn trùng sinh lỗi `IntegrityError`, được chuyển thành lý do `duplicate_order`.

```python
def create_pending_order(order_code, total_amount, items_json):
    result = {"inserted": False, "reason": ""}
    prepared = _normalize_order_input(order_code, total_amount, items_json, result)
    if prepared is None:
        return result
    order_code, total_amount, normalized_items = prepared
    conn = None
    try:
        conn = get_connection()
        conn.execute(
            """
            INSERT INTO orders (order_id, total_amount, items_json,
                                payment_status, kitchen_status, created_at)
            VALUES (?, ?, ?, 'PENDING', 'pending', datetime('now', 'localtime'))
            """,
            (order_code, total_amount, json.dumps(normalized_items, ensure_ascii=False))
        )
        conn.commit()
        result.update(inserted=True, reason="ok")
        return result
    except sqlite3.IntegrityError:
        result["reason"] = "duplicate_order"
        return result
    finally:
        if conn is not None:
            conn.close()
```

Trước khi ghi, hàm `_normalize_order_input` kiểm tra và chuẩn hóa dữ liệu:
- mã đơn không rỗng;
- tổng tiền là số dương;
- danh sách món là mảng JSON không rỗng;
- mỗi món có mã sản phẩm, số lượng dương và tên.

Vì SQLite có kiểu dữ liệu động (mục 4.2.1), việc kiểm tra này là tuyến phòng thủ chính chống dữ liệu sai. Khối `finally` bảo đảm kết nối luôn được đóng, kể cả khi có ngoại lệ.

**Đóng đơn bằng cập nhật có điều kiện:**

```python
def close_pending_order(order_code, new_status):
    # PENDING -> CANCELLED/EXPIRED. Chỉ đổi khi đơn còn PENDING nên
    # hủy và hết hạn chạy đua nhau thì trạng thái đến trước được giữ.
    if new_status not in CLOSED_ORDER_STATUSES:
        raise ValueError(f"Trạng thái đóng không hợp lệ: {new_status}")
    ...
    cursor = conn.execute(
        """
        UPDATE orders
        SET payment_status = ?, closed_at = datetime('now', 'localtime')
        WHERE order_id = ? AND payment_status = 'PENDING'
        """,
        (new_status, order_code)
    )
    conn.commit()
    return cursor.rowcount == 1
```

Điều kiện `AND payment_status = 'PENDING'` là cốt lõi. Giả sử khách bấm hủy đúng lúc đồng hồ đếm ngược hết giờ: cả hai sự kiện cùng gọi hàm này. Câu `UPDATE` là nguyên tử, nên chỉ lời gọi đầu tiên tìm thấy đơn còn PENDING và đổi được trạng thái (`rowcount == 1`). Lời gọi thứ hai không còn dòng nào khớp (`rowcount == 0`). Trạng thái đến trước được giữ nguyên, không bị ghi đè. Đây là kỹ thuật **khóa lạc quan** (*optimistic concurrency*) dựa trên điều kiện trạng thái: không cần khóa tường minh, chỉ cần điều kiện trong `WHERE`.

**Dọn đơn treo khi khởi động.** Nếu kiosk bị tắt đột ngột khi đang chờ thanh toán, đơn PENDING sẽ treo mãi. Lúc khởi động, kiosk gọi hàm đóng các đơn quá hạn:

```python
def expire_stale_pending_orders(max_age_minutes=10):
    ...
    cursor = conn.execute(
        """
        UPDATE orders
        SET payment_status = 'EXPIRED', closed_at = datetime('now', 'localtime')
        WHERE payment_status = 'PENDING'
          AND created_at <= datetime('now', 'localtime', ?)
        """,
        (f"-{int(max_age_minutes)} minutes",)
    )
```

Hàm `datetime('now', 'localtime', '-10 minutes')` của SQLite tính mốc thời gian ngay trong câu lệnh SQL. Chuỗi thời gian dạng `'YYYY-MM-DD HH:MM:SS'` so sánh được trực tiếp bằng `<=`, vì thứ tự chữ cái trùng với thứ tự thời gian. Đây là lý do định dạng này được chọn.

### 4.4.4. Chốt đơn và trừ kho trong một giao dịch

Hàm `save_new_order` là **giao dịch quan trọng nhất** của hệ thống: đổi đơn sang PAID và trừ kho theo công thức. Toàn bộ nằm trong một giao dịch `BEGIN IMMEDIATE`:

```python
def save_new_order(order_code, total_amount, items_json, paid_amount=None):
    result = {"inserted": False, "reason": "", "low_stock": [], "insufficient_stock": []}
    prepared = _normalize_order_input(order_code, total_amount, items_json, result)
    if prepared is None:
        return result
    order_code, total_amount, normalized_items = prepared
    ...
    conn = None
    try:
        conn = get_connection()
        conn.execute("BEGIN IMMEDIATE")                       # (1) giữ khóa ghi ngay từ đầu

        existing = conn.execute(                              # (2) đọc trạng thái hiện tại
            "SELECT payment_status FROM orders WHERE order_id = ?", (order_code,)
        ).fetchone()
        if existing is not None and existing["payment_status"] != ORDER_PENDING:
            conn.rollback()
            result["reason"] = ("duplicate_order" if existing["payment_status"] == ORDER_PAID
                                else "order_closed")
            return result

        required = {}                                         # (3) cộng dồn nguyên liệu cần dùng
        for item in normalized_items:
            rows = conn.execute(
                """
                SELECT r.ingredient_id, r.qty_per_unit,
                       i.name, i.unit, i.quantity, i.low_stock_threshold
                FROM recipe r
                JOIN ingredient i ON i.id = r.ingredient_id
                WHERE r.product_id = ?
                """,
                (item["product_id"],)
            ).fetchall()
            for row in rows:
                ingredient_id = int(row["ingredient_id"])
                if ingredient_id not in required:
                    required[ingredient_id] = {"id": ingredient_id, "name": row["name"],
                                               "unit": row["unit"],
                                               "available": float(row["quantity"]),
                                               "required": 0.0}
                required[ingredient_id]["required"] += float(row["qty_per_unit"]) * item["qty"]

        for info in required.values():                       # (4) ghi nhận nguyên liệu thiếu
            if info["available"] < info["required"]:
                result["insufficient_stock"].append({..., "missing": info["required"] - info["available"]})

        if existing is None:                                  # (5) chốt đơn
            cursor = conn.execute("INSERT INTO orders (...) VALUES (?, ?, ?, 'PAID', 'pending', ...)", ...)
        else:
            cursor = conn.execute(
                """
                UPDATE orders
                SET payment_status = 'PAID', kitchen_status = 'pending',
                    total_amount = ?, items_json = ?, paid_amount = ?,
                    closed_at = datetime('now', 'localtime')
                WHERE order_id = ? AND payment_status = 'PENDING'
                """,
                (total_amount, normalized_json, paid_amount, order_code)
            )
        if cursor.rowcount != 1:
            raise RuntimeError("Không chốt được đơn hàng")

        for info in required.values():                       # (6) trừ kho, không xuống dưới 0
            cursor = conn.execute(
                "UPDATE ingredient SET quantity = MAX(0, quantity - ?) WHERE id = ?",
                (info["required"], info["id"])
            )
            if cursor.rowcount != 1:
                raise RuntimeError(f"Không thể trừ nguyên liệu id={info['id']}")

        rows = conn.execute(                                  # (7) danh sách sắp hết để cảnh báo
            "SELECT id, name, unit, quantity, low_stock_threshold FROM ingredient "
            "WHERE quantity <= low_stock_threshold ORDER BY quantity ASC, name ASC"
        ).fetchall()
        result["low_stock"] = [dict(row) for row in rows]

        conn.commit()                                         # (8) ghi nhận TẤT CẢ cùng lúc
        result.update(inserted=True, reason="ok")
        return result
    except Exception as error:
        if conn is not None:
            conn.rollback()                                   # lỗi bất kỳ: hủy TẤT CẢ
        result["reason"] = "database_error"
        return result
    finally:
        if conn is not None:
            conn.close()
```

*(Trích rút gọn: bản đầy đủ có thêm nhánh bắt riêng `sqlite3.IntegrityError` để phân biệt đơn trùng và lỗi ràng buộc khác.)*

Phân tích theo từng bước:

**Bước 1: vì sao cần `BEGIN IMMEDIATE`.** Hàm có cấu trúc đọc – kiểm tra – ghi: đọc trạng thái đơn rồi quyết định chốt. Trong hệ thống có **hai nguồn** cùng có thể chốt một đơn: tín hiệu MQTT từ webhook, và kiosk tự tra cứu giao dịch qua API của SePay mỗi 15 giây. Có lúc cả hai cùng phát hiện tiền về gần như đồng thời. Với `BEGIN IMMEDIATE`, nguồn đến trước giữ khóa ghi từ trước khi đọc. Nguồn đến sau phải chờ và khi được chạy sẽ đọc thấy đơn đã PAID, trả về `duplicate_order`. **Kho không bao giờ bị trừ hai lần cho cùng một đơn.**

**Bước 2: tôn trọng máy trạng thái.** Đơn đã đóng (CANCELLED, EXPIRED) thì trả `order_closed`, **không trừ kho**. Người gọi khi đó chuyển sang ghi nhận LATE_PAID.

**Bước 3: phép nối và cộng dồn.** Câu `JOIN` lấy định mức kèm tồn kho hiện tại của từng nguyên liệu. Từ điển `required` **cộng dồn** nhu cầu theo nguyên liệu. Ví dụ đơn có hai bánh mì đặc biệt và một bánh mì chả lụa thì ổ bánh mì được tính là 3, trừ một lần duy nhất.

**Bước 4 và 6: đơn đã thanh toán thì không từ chối vì thiếu kho.** Đây là một quyết định nghiệp vụ quan trọng. Khi hàm này chạy, khách **đã chuyển tiền**. Từ chối đơn lúc này vì kho trên phần mềm báo thiếu là không chấp nhận được, nhất là khi số liệu kho có thể lệch với thực tế. Vì vậy hệ thống vẫn chốt đơn, trừ kho nhưng **không cho xuống dưới 0** nhờ `MAX(0, quantity - ?)`, và trả danh sách `insufficient_stock` để cảnh báo. Hiện kiosk chưa kiểm tra tồn kho trước khi khách thanh toán; món chỉ bị ẩn khỏi thực đơn khi quản trị viên tắt món đó (cột `status`). Mục 4.5.3 bàn về hạn chế này.

**Bước 5: kiểm tra `rowcount`.** Câu `UPDATE` vẫn giữ điều kiện `payment_status = 'PENDING'` như một lớp bảo vệ thứ hai. Nếu vì lý do nào đó không đổi được đúng một dòng, hàm ném ngoại lệ để hủy toàn bộ giao dịch.

**Bước 8 và nhánh lỗi: tính nguyên tử.** Chỉ khi mọi bước thành công, `commit()` mới ghi nhận tất cả. Bất kỳ lỗi nào ở bất kỳ bước nào cũng dẫn đến `rollback()`: trạng thái đơn và tồn kho trở về như trước khi gọi hàm. Không bao giờ có chuyện "đơn đã PAID nhưng kho chưa trừ" hay ngược lại.

Kết quả trả về là một từ điển giàu thông tin: `inserted`, `reason`, `low_stock`, `insufficient_stock`. Ứng dụng kiosk dựa vào đó để quyết định:
- gửi đơn xuống bếp và phát tồn kho qua MQTT (chương MQTT);
- hoặc chuyển sang ghi nhận thanh toán muộn.

### 4.4.5. Ghi nhận thanh toán muộn

```python
def mark_order_late_paid(order_code, paid_amount):
    # Trả: "late_paid" | "already_recorded" | "not_closed" | "unknown_order" | "database_error"
    ...
    conn = get_connection()
    conn.execute("BEGIN IMMEDIATE")
    row = conn.execute("SELECT payment_status FROM orders WHERE order_id = ?",
                       (order_code,)).fetchone()
    if row is None:
        conn.rollback()
        return "unknown_order"
    status = row["payment_status"]
    if status == ORDER_LATE_PAID:
        conn.rollback()
        return "already_recorded"           # webhook và polling cùng báo: chỉ ghi một lần
    if status not in CLOSED_ORDER_STATUSES:
        conn.rollback()
        return "not_closed"                 # đơn còn PENDING hoặc đã PAID: để luồng thường xử lý
    conn.execute("UPDATE orders SET payment_status = 'LATE_PAID', paid_amount = ? WHERE order_id = ?",
                 (paid_amount, order_code))
    conn.commit()
    return "late_paid"
```

Hàm này cũng theo mẫu đọc – kiểm tra – ghi trong `BEGIN IMMEDIATE`, và trả về **một trong năm kết quả có tên**. Kết quả `already_recorded` thể hiện tính **lũy đẳng** (*idempotent*): webhook và cơ chế tra cứu định kỳ cùng báo một khoản tiền thì khoản đó chỉ được ghi một lần, lần gọi thứ hai không làm thay đổi gì. Cột `paid_amount` lưu số tiền thực nhận, để quản trị viên biết chính xác cần hoàn bao nhiêu.

### 4.4.6. Bếp báo hoàn thành đơn

```python
def admin_mark_order_done(order_code):
    ...
    conn.execute("BEGIN IMMEDIATE")
    cursor = conn.execute(
        """
        UPDATE orders
        SET kitchen_status = 'done'
        WHERE order_id = ?
          AND payment_status = 'PAID'
          AND kitchen_status = 'pending'
        """,
        (order_code,)
    )
    if cursor.rowcount != 1:
        conn.rollback()
        return False
    conn.commit()
    return True
```

Đây là ví dụ thuần túy của **cập nhật có điều kiện**: một câu `UPDATE` với ba điều kiện. Đơn phải tồn tại, đã thanh toán và chưa hoàn thành. Nhân viên chạm nút trên màn hình bếp và trợ lý giọng nói có thể cùng báo hoàn thành một đơn; chỉ lần đầu tiên đổi được dòng. Lần thứ hai nhận `False`, và Smart Kiosk API chuyển thành mã HTTP **409 Conflict** (chương Flask, mục 5.4.5). Điều kiện `payment_status = 'PAID'` còn ngăn một đơn chưa thanh toán bị đánh dấu hoàn thành.

Truy vấn danh sách đơn cho bếp chỉ lấy đơn đã thanh toán và chưa hoàn thành, xếp theo thời gian tạo để đơn cũ được làm trước:

```python
SELECT order_id, total_amount, items_json, payment_status, kitchen_status, created_at
FROM orders
WHERE payment_status = 'PAID' AND kitchen_status = 'pending'
ORDER BY created_at ASC
```

### 4.4.7. Truy vấn thống kê và báo cáo

Báo cáo doanh thu tận dụng các hàm thời gian và hàm gộp của SQL:

```python
# Doanh thu và số đơn tháng hiện tại
SELECT SUM(total_amount) FROM orders
WHERE strftime('%Y-%m', created_at) = strftime('%Y-%m', 'now', 'localtime')
  AND payment_status = 'PAID'

# Doanh thu từng ngày trong 7 ngày gần nhất (biểu đồ)
SELECT strftime('%d/%m', created_at) AS day, SUM(total_amount) AS total
FROM orders
WHERE DATE(created_at) >= DATE('now', '-6 days', 'localtime')
  AND payment_status = 'PAID'
GROUP BY day
ORDER BY created_at ASC
```

`strftime` trích các thành phần năm, tháng, ngày từ chuỗi thời gian. `SUM` và `COUNT` là **hàm gộp**, còn `GROUP BY` chia dữ liệu thành từng nhóm theo ngày để tính tổng riêng. Mọi truy vấn doanh thu đều có điều kiện `payment_status = 'PAID'`, nên đơn chờ, đơn hủy và đơn thanh toán muộn (sẽ hoàn tiền) **không bao giờ** bị tính vào doanh thu. Bộ kiểm thử có một ca riêng cho quy tắc này.

Thống kê **món bán chạy** lại được tính bằng Python, vì chi tiết món nằm trong cột JSON:

```python
cursor.execute(f'''
    SELECT items_json FROM orders
    WHERE payment_status = 'PAID' AND {date_filter}
''')
item_stats = {}
for (items_json,) in cursor.fetchall():
    for item in json.loads(items_json):
        stats = item_stats.setdefault(item.get('name', 'Unknown'), {"qty": 0, "revenue": 0})
        stats["qty"] += item.get('qty', 1)
        stats["revenue"] += item.get('qty', 1) * item.get('price', 0)
top_items = sorted(item_stats.items(), key=lambda x: x[1]["qty"], reverse=True)[:5]
```

*(Trích rút gọn: bản đầy đủ có thêm khối `try` quanh việc phân tích JSON và tra nhóm món ăn hay đồ uống theo mã sản phẩm.)*

Đây là cái giá của quyết định lưu JSON (mục 4.4.2). Thống kê không viết được bằng một câu `GROUP BY` mà phải tải dữ liệu về rồi tính. Với quy mô một cửa hàng, vài nghìn đơn mỗi tháng, chi phí này không đáng kể.

Câu lệnh ở trên dùng chuỗi f-string để chèn `date_filter` vào SQL. Điều này **an toàn** vì `date_filter` chỉ nhận một trong bốn chuỗi cố định do mã nguồn định nghĩa sẵn (`day`, `week`, `month`, `year`), không bao giờ chứa dữ liệu người dùng. Hàm lọc danh sách đơn cũng theo nguyên tắc tương tự: phần điều kiện trạng thái lấy từ một từ điển cố định, còn từ khóa tìm kiếm do người dùng nhập luôn đi qua tham số:

```python
status_clauses = {
    "paid": "payment_status = 'PAID'",
    "cancelled": "payment_status IN ('CANCELLED', 'EXPIRED')",
    "late_paid": "payment_status = 'LATE_PAID'",
    ...
}
query += " AND " + status_clauses.get(status_filter, "payment_status != 'PENDING'")
if keyword:
    query += " AND order_id LIKE ?"
    params.append(f"%{keyword}%")                 # dữ liệu người dùng: luôn qua tham số
...
query += " ORDER BY created_at DESC LIMIT ? OFFSET ?"
```

Quy tắc cần nhớ: **cấu trúc câu lệnh** có thể ghép từ các mảnh cố định trong mã nguồn, còn **giá trị** đến từ bên ngoài phải luôn đi qua tham số `?`. Cặp `LIMIT ? OFFSET ?` thực hiện **phân trang**, để phần mềm quản trị không phải tải toàn bộ lịch sử đơn mỗi lần mở.

### 4.4.8. Mã OTP khôi phục mật khẩu

Chức năng quên mật khẩu lưu mã OTP trong bảng `settings` và kiểm tra trong một giao dịch:

```python
OTP_VALID_MINUTES = 10
OTP_RESEND_COOLDOWN_SECONDS = 60
OTP_MAX_ATTEMPTS = 5

def verify_and_reset_password(otp_code, new_password):
    if not new_password or len(new_password) < 6:
        return False, "weak_password"
    conn = get_connection()
    try:
        conn.execute("BEGIN IMMEDIATE")
        row = conn.execute('SELECT reset_otp_code, reset_otp_expires_at, reset_otp_attempts '
                           'FROM settings WHERE id = 1').fetchone()
        if not row or not row['reset_otp_code']:
            conn.rollback()
            return False, "no_otp_requested"
        if datetime.datetime.now() > expires_at:              # hết hạn: xóa mã
            _clear_otp(conn); conn.commit()
            return False, "expired_otp"
        attempts = int(row['reset_otp_attempts'] or 0)
        if not hmac.compare_digest(otp_code.encode("utf-8"),
                                   str(row['reset_otp_code']).encode("utf-8")):
            attempts += 1
            if attempts >= OTP_MAX_ATTEMPTS:                  # sai 5 lần: hủy mã
                _clear_otp(conn); conn.commit()
                return False, "too_many_attempts"
            conn.execute('UPDATE settings SET reset_otp_attempts = ? WHERE id = 1', (attempts,))
            conn.commit()
            return False, "invalid_otp"
        new_hash = generate_password_hash(new_password)
        conn.execute('''UPDATE settings SET admin_password_hash = ?, reset_otp_code = '',
                        reset_otp_expires_at = NULL, reset_otp_attempts = 0 WHERE id = 1''',
                     (new_hash,))
        conn.commit()                                         # đổi mật khẩu và xóa mã cùng lúc
        return True, "ok"
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()
```

*(Trích rút gọn phần đổi chuỗi thời gian hết hạn.)*

Đoạn mã kết hợp giao dịch với các biện pháp bảo mật:
- **Mã OTP sinh bằng `secrets.randbelow`**, là nguồn ngẫu nhiên an toàn về mật mã. Mô-đun `random` thông thường có thể bị đoán trước, nên không dùng.
- **Giới hạn thời gian và số lần thử.** Mã hết hạn sau 10 phút. Mã 6 chữ số có một triệu khả năng; giới hạn 5 lần nhập sai làm việc dò mã gần như không thể (xác suất đoán trúng 5 phần triệu). Bộ đếm sai được tăng **trong cùng giao dịch** `BEGIN IMMEDIATE` với việc đọc. Vì vậy dù kẻ tấn công gửi nhiều yêu cầu song song, các lần thử vẫn được đếm tuần tự, không lần nào "lọt" qua bộ đếm.
- **Chống gửi lại liên tục.** Hàm sinh mã từ chối tạo mã mới nếu mã cũ chưa được 60 giây; Smart Kiosk API trả mã HTTP 429.
- **So sánh mã trong thời gian hằng** bằng `hmac.compare_digest`.
- **Đổi mật khẩu và xóa mã trong cùng một câu lệnh.** Mã OTP không thể dùng lại lần thứ hai.

### 4.4.9. Kiểm thử tầng dữ liệu

Tệp `tests/test_order_lifecycle.py` có 17 ca kiểm thử cho vòng đời đơn hàng. Mỗi ca chạy trên **một tệp cơ sở dữ liệu tạm riêng**:

```python
@pytest.fixture
def db(tmp_path, monkeypatch):
    monkeypatch.setattr(database, "DB_PATH", str(tmp_path / "lifecycle.db"))
    database.init_database()
    return database


def test_pending_order_becomes_paid_and_deducts_stock(db):
    before = _stock_snapshot()
    assert db.create_pending_order(ORDER, 70000, ITEMS_JSON)["inserted"]
    assert _row(ORDER)["payment_status"] == "PENDING"
    assert _stock_snapshot() == before, "PENDING không được trừ kho"

    result = db.save_new_order(ORDER, 70000, ITEMS_JSON, paid_amount=70000)

    assert result["inserted"] and result["reason"] == "ok"
    row = _row(ORDER)
    assert row["payment_status"] == "PAID"
    assert row["kitchen_status"] == "pending"
    assert row["paid_amount"] == 70000


def test_cancelled_order_cannot_be_paid_and_keeps_stock(db):
    db.create_pending_order(ORDER, 70000, ITEMS_JSON)
    assert db.close_pending_order(ORDER, db.ORDER_CANCELLED)
    before = _stock_snapshot()

    result = db.save_new_order(ORDER, 70000, ITEMS_JSON)

    assert not result["inserted"]
    assert result["reason"] == "order_closed"
    assert _stock_snapshot() == before


def test_late_payment_is_recorded_once(db):
    db.create_pending_order(ORDER, 70000, ITEMS_JSON)
    db.close_pending_order(ORDER, db.ORDER_EXPIRED)

    assert db.mark_order_late_paid(ORDER, 70000) == "late_paid"
    assert db.mark_order_late_paid(ORDER, 70000) == "already_recorded"
```

Fixture `db` dùng thư mục tạm `tmp_path` của pytest, và `monkeypatch` thay đường dẫn cơ sở dữ liệu trong suốt ca kiểm thử. Nhờ SQLite chỉ là một tệp, mỗi ca có một cơ sở dữ liệu **sạch, độc lập**: không ca nào ảnh hưởng ca khác, và không bao giờ chạm vào `kiosk.db` thật. Với MySQL hay PostgreSQL, việc này phức tạp hơn nhiều vì phải dựng máy chủ kiểm thử riêng.

Các ca kiểm thử bao phủ toàn bộ máy trạng thái:
- đơn chờ thành đơn đã thanh toán và trừ kho;
- đơn đã hủy không thể thanh toán, kho giữ nguyên;
- thanh toán muộn chỉ ghi một lần;
- đóng đơn chỉ đổi đơn đang chờ;
- đơn đã thanh toán từ chối chốt lần hai;
- đơn treo hết hạn khi khởi động;
- đơn chưa thanh toán không tính vào doanh thu và không lên bếp.

Một nhóm ca khác kiểm thử các tình huống **chạy đua** qua lớp `bridge` của kiosk, ví dụ khách bấm hủy đúng lúc tiền về. Tệp `test_database_path_config.py` có 6 ca kiểm thử các quy tắc của biến `KIOSK_DB_PATH`.

---

## 4.5. Kỹ thuật và đánh giá

### 4.5.1. Các kỹ thuật thiết kế đã áp dụng

**Về tính đúng đắn khi có nhiều bên cùng truy cập:**
- Chế độ **WAL** cho phép đọc song song với ghi.
- **Busy timeout** giúp hai tiến trình cùng ghi mà không báo lỗi khóa.
- Mọi thao tác dạng đọc – kiểm tra – ghi đều nằm trong **`BEGIN IMMEDIATE`**, nên trở thành một khối nguyên tử.
- Các chuyển trạng thái dùng **cập nhật có điều kiện** kèm kiểm tra `rowcount`, giúp hai nguồn cùng báo một sự kiện không ghi đè lên nhau. Bên đến sau nhận một kết quả có tên rõ ràng: `duplicate_order`, `already_recorded`, hoặc mã HTTP 409.

**Về toàn vẹn dữ liệu:**
- Chốt đơn và trừ kho nằm trong **một giao dịch**, hoặc cùng thành công hoặc cùng bị hủy.
- **Khóa chính** chặn đơn trùng mã.
- **Khóa ngoại** chặn định mức trỏ tới nguyên liệu không tồn tại.
- Ràng buộc **`CHECK (id = 1)`** giữ bảng cấu hình chỉ một dòng.
- **Tiền lưu bằng số nguyên** tránh sai số dấu phẩy động.
- Đầu vào được **kiểm tra và chuẩn hóa** trước khi ghi, bù cho kiểu dữ liệu động của SQLite.

**Về mô hình nghiệp vụ:**
- Trạng thái đơn được thiết kế thành một **máy trạng thái** tường minh, có trạng thái LATE_PAID cho tiền về muộn.
- Đơn được ghi từ lúc tạo mã QR, để mọi khoản tiền đều đối chiếu được.
- Chi tiết đơn lưu dạng **ảnh chụp JSON**, giữ nguyên giá và tên lúc bán.
- Đơn đã thanh toán **không bị từ chối vì thiếu kho**; kho được chặn ở 0 và có cảnh báo.

**Về bảo mật:**
- Mọi giá trị từ bên ngoài đi qua **truy vấn có tham số**, chống chèn SQL.
- Mật khẩu được **băm có muối**.
- Mã OTP được sinh bằng nguồn ngẫu nhiên mật mã, giới hạn thời gian và số lần thử, so sánh trong thời gian hằng.
- Tệp cơ sở dữ liệu chỉ được mở bởi các tiến trình trên Raspberry Pi; mọi máy khác đi qua API có xác thực.

### 4.5.2. Các lỗi đã gặp và cách khắc phục

| Lỗi | Hậu quả | Cách khắc phục |
|---|---|---|
| Tiền về sau khi khách đã hủy hoặc mã QR hết hạn không được ghi nhận ở đâu | Cửa hàng nhận tiền mà không biết của ai, không hoàn được | Ghi đơn PENDING ngay khi tạo QR; thêm trạng thái EXPIRED và LATE_PAID cùng cột `paid_amount`, `closed_at` |
| Webhook và cơ chế tra cứu định kỳ cùng chốt một đơn | Nguy cơ trừ kho hai lần | Chốt đơn trong `BEGIN IMMEDIATE`; bên đến sau nhận `duplicate_order` |
| Kiosk tắt đột ngột để lại đơn PENDING vĩnh viễn | Đơn treo, tiền về sau không được xử lý đúng | Đóng đơn chờ quá hạn mỗi lần khởi động |
| Hủy đơn và hết giờ xảy ra cùng lúc | Trạng thái bị ghi đè lẫn nhau | Cập nhật có điều kiện `WHERE payment_status = 'PENDING'` |
| Ứng dụng kiosk và API mở hai tệp cơ sở dữ liệu khác nhau khi chạy từ hai thư mục khác nhau | Dữ liệu bị "chia đôi" âm thầm | Đường dẫn mặc định tính theo vị trí mã nguồn; `KIOSK_DB_PATH` bắt buộc là đường dẫn tuyệt đối |
| Mã OTP có thể bị dò bằng cách thử liên tục | Chiếm quyền quản trị | Giới hạn 5 lần sai trong giao dịch, thời gian chờ 60 giây giữa hai lần gửi |

### 4.5.3. Hạn chế còn tồn tại và hướng cải tiến

**Không xóa được món đã có công thức.** Bảng `recipe` khai báo khóa ngoại tới `product(id)` nhưng không kèm `ON DELETE CASCADE`. Hàm `admin_delete_product` chỉ xóa dòng trong `product`. Khi đã bật `PRAGMA foreign_keys=ON`, SQLite từ chối xóa với lỗi `FOREIGN KEY constraint failed`. Lỗi này đã được kiểm chứng trên một cơ sở dữ liệu thử. Ngoại lệ không được bắt trong tầng dữ liệu, nên yêu cầu xóa món qua API bị lỗi. Điều thú vị là hàm xóa nguyên liệu đã xử lý đúng bằng cách xóa định mức trước. Có hai cách sửa:
- trong `admin_delete_product`, xóa các dòng `recipe` của món trong cùng giao dịch trước khi xóa món;
- hoặc khai báo `ON DELETE CASCADE` cho khóa ngoại. Cách này cần di trú bảng, vì SQLite không sửa được khóa ngoại bằng `ALTER TABLE`.

Trong thực tế vận hành, nên ưu tiên **ẩn món** bằng cột `status` thay vì xóa hẳn. Chức năng ẩn món đã có sẵn.

**Chưa chặn món hết nguyên liệu trước khi thanh toán.** Kiosk vẫn cho khách đặt và thanh toán một món dù kho trên phần mềm không đủ nguyên liệu. Vì tiền đã về, hệ thống chốt đơn rồi mới cảnh báo thiếu kho. Hướng cải tiến:
- khi tải thực đơn, tính số phần tối đa làm được của từng món từ bảng `recipe` và `ingredient` (một câu `JOIN` với `MIN(quantity / qty_per_unit)`);
- ẩn hoặc làm mờ món không đủ nguyên liệu;
- kiểm tra lại cả giỏ hàng ngay trước khi tạo mã QR.

**Một số hàm quản trị chưa bảo đảm đóng kết nối và chưa nhóm thao tác vào giao dịch.**
- Các hàm như `admin_add_product` hay `admin_save_recipe` không dùng `try/finally`. Gặp ngoại lệ, kết nối không được đóng tường minh và lỗi được ném thẳng lên API.
- Riêng `admin_save_recipe` xóa công thức cũ rồi chèn công thức mới. Nhờ cơ chế giao dịch ngầm định của Python, hai bước này nằm trong cùng một giao dịch. Nhưng khi có lỗi giữa chừng, mã không gọi `rollback()` tường minh.

Nên thống nhất một mẫu chung: `try` / `except: rollback()` / `finally: close()`, hoặc viết một hàm hỗ trợ dạng *context manager* để mọi hàm dùng chung.

**Di trú lược đồ còn thủ công.** Cách "thử `ALTER TABLE`, lỗi thì bỏ qua" chỉ làm được việc thêm cột, và không biết cơ sở dữ liệu đang ở phiên bản nào. Cách làm bài bản hơn là lưu số phiên bản lược đồ bằng `PRAGMA user_version`, rồi chạy tuần tự các bước di trú còn thiếu.

**Chưa có chỉ mục phụ.** Các truy vấn theo `payment_status`, `kitchen_status` và `created_at` hiện quét toàn bảng. Với vài nghìn đơn, việc này vẫn tính bằng mili giây. Khi dữ liệu tích lũy nhiều năm, nên thêm `CREATE INDEX idx_orders_status_time ON orders(payment_status, created_at)`.

**Thống kê món phụ thuộc phân tích JSON bằng Python.** Có thể dùng các hàm JSON có sẵn của SQLite (`json_each`) để thống kê ngay trong SQL, hoặc tách bảng `order_item` nếu cần báo cáo phức tạp hơn.

**Sao lưu chưa tự động.** Chưa có lịch sao lưu định kỳ. Nên thêm một tác vụ hằng ngày dùng lệnh `.backup` của SQLite, an toàn kể cả khi ứng dụng đang chạy ở chế độ WAL, và chép bản sao ra một thiết bị khác.

### 4.5.4. Kiến thức trọng tâm

Phần này tóm tắt những ý cốt lõi của chương, dùng để ôn tập và chuẩn bị trả lời trước hội đồng.

**Bản chất của SQLite.** SQLite là hệ quản trị cơ sở dữ liệu quan hệ nhúng: không có máy chủ, chạy như một thư viện trong chính tiến trình ứng dụng, lưu toàn bộ dữ liệu trong một tệp. Ưu điểm là không cần cài đặt hay vận hành, nhẹ, giao dịch ACID đầy đủ, dễ sao lưu và di chuyển. Hạn chế là một bên ghi tại một thời điểm, không truy cập trực tiếp qua mạng, không có phân quyền người dùng, và kiểu dữ liệu động nên ứng dụng phải tự kiểm tra đầu vào.

**Các cơ chế cần nắm chắc:**
- **Giao dịch ACID:** nguyên tử, nhất quán, cô lập, bền vững.
- **Ba loại `BEGIN`.** `DEFERRED` là mặc định. `IMMEDIATE` lấy khóa ghi ngay, dùng cho mẫu đọc – kiểm tra – ghi để tránh tranh chấp. `EXCLUSIVE` lấy khóa độc quyền.
- **Chế độ WAL:** ghi nối vào tệp nhật ký riêng, người đọc không bị chặn bởi người ghi, checkpoint định kỳ chép về tệp chính.
- **Busy timeout:** chờ khóa thay vì báo lỗi ngay.
- **Khóa ngoại** phải bật bằng `PRAGMA foreign_keys=ON` ở mỗi kết nối.
- **Python tự chèn `BEGIN`** trước câu lệnh ghi và cần `commit()` để ghi nhận.

**Lý do lựa chọn.** Quy mô một cửa hàng, không muốn thêm gánh nặng vận hành, tiết kiệm tài nguyên Raspberry Pi, nhưng vẫn cần giao dịch và SQL đầy đủ. Giới hạn truy cập cục bộ được giải quyết bằng kiến trúc: chỉ các tiến trình trên Pi mở tệp, mọi máy khác đi qua API.

**Thiết kế trong dự án:**
- **Năm bảng:** thực đơn, nguyên liệu, công thức (bảng trung gian nhiều – nhiều), đơn hàng, cấu hình một dòng.
- **Tiền** lưu số nguyên.
- **Chi tiết đơn** là ảnh chụp JSON tại thời điểm bán.
- **Đơn hàng** là máy trạng thái PENDING → PAID / CANCELLED / EXPIRED → LATE_PAID.
- **Chốt đơn và trừ kho** trong một giao dịch `BEGIN IMMEDIATE`, nên hai nguồn báo tiền về cùng lúc cũng không trừ kho hai lần.
- **Cập nhật có điều kiện** kèm `rowcount` cho mọi chuyển trạng thái.
- **Truy vấn có tham số** chống chèn SQL.
- **OTP** có giới hạn số lần thử, tính trong giao dịch.

**Hạn chế còn tồn tại** (cần thẳng thắn nhận diện):
- Xóa món đã có công thức bị lỗi khóa ngoại.
- Chưa chặn món hết nguyên liệu trước khi thanh toán.
- Một số hàm quản trị chưa dùng mẫu đóng kết nối và hủy giao dịch thống nhất.
- Di trú lược đồ còn thủ công.
- Chưa có chỉ mục phụ.
- Chưa sao lưu tự động.

Mỗi hạn chế đều đã có hướng khắc phục cụ thể.

---

*Tài liệu tham khảo: SQLite Documentation (sqlite.org) – "Appropriate Uses For SQLite", "Datatypes In SQLite", "Transaction", "File Locking And Concurrency", "Write-Ahead Logging", "SQLite Foreign Key Support", "Date And Time Functions"; Python Standard Library – `sqlite3` (DB-API 2.0, PEP 249); E. F. Codd, "A Relational Model of Data for Large Shared Data Banks", Communications of the ACM, 1970; OWASP – SQL Injection Prevention Cheat Sheet; Werkzeug – `werkzeug.security`.*

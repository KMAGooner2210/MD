# BUỔI 2 — SQLITE VÀ VÒNG ĐỜI ĐƠN HÀNG

> **Chuỗi học tập công nghệ trong Đồ án tốt nghiệp**
> *Hệ thống tự động hóa quy trình phục vụ và điều phối thông minh trong nhà hàng*
>
> - **Mã nguồn chính:** `Smart_Kiosk_Backup/backend/database.py` (1.162 dòng)
> - **Mã nguồn liên quan:** `kiosk_app/bridge.py`, `kiosk_app/qml/PaymentPopup.qml`, `kiosk_app/qml/CartSidebar.qml`, `servers/api_server.py`, `admin_app/qml/OrdersView.qml`
> - **Mức độ:** hiểu tới tận mã nguồn; số dòng (`Lxxx`) tham chiếu theo phiên bản hiện tại của mã nguồn
> - **Số liệu thực nghiệm:** chạy trên cơ sở dữ liệu tạm (biến `KIOSK_DB_PATH`), gọi đúng các hàm thật trong `database.py`

---

## Mục lục

1. [Tổng quan SQLite](#mục-1-tổng-quan-sqlite)
2. [Kết nối và cấu hình (`get_connection`)](#mục-2-kết-nối-và-cấu-hình-get_connection)
3. [Lược đồ dữ liệu](#mục-3-lược-đồ-dữ-liệu)
4. [Vòng đời đơn hàng](#mục-4-vòng-đời-đơn-hàng)
5. [Xử lý đồng thời và chống xử lý trùng ★](#mục-5-xử-lý-đồng-thời-và-chống-xử-lý-trùng-)
6. [Quy tắc nghiệp vụ và thống kê](#mục-6-quy-tắc-nghiệp-vụ-và-thống-kê)
7. [Đánh giá và câu hỏi ôn tập](#mục-7-đánh-giá-và-câu-hỏi-ôn-tập)
- [Phụ lục: Mã nguồn các thí nghiệm](#phụ-lục-mã-nguồn-các-thí-nghiệm)

### Khuôn trình bày mỗi tiểu mục

| Phần | Nội dung |
|---|---|
| **① Khái niệm** | Định nghĩa ngắn, từ khóa *in nghiêng* |
| **② Trong đồ án** | File và số dòng, đoạn mã cốt lõi, số liệu thật |
| **③ Vì sao / Nếu không** | Lý do thiết kế; bỏ đi thì hỏng ở đâu (phần hội đồng hay hỏi) |
| **④ Ghi nhớ** | 1–3 ý để ôn nhanh |

### Ba hình nên vẽ tay

1. Sơ đồ quan hệ 5 bảng (mục 3.1)
2. Sơ đồ chuyển trạng thái đơn hàng (mục 4.2)
3. Lưu đồ hàm `save_new_order` (mục 5.5)

---

# MỤC 1. TỔNG QUAN SQLITE

## 1.1. Cơ sở dữ liệu nhúng là gì, khác MySQL ở điểm nào

**① Khái niệm**

- **SQLite** là hệ quản trị cơ sở dữ liệu quan hệ dạng *nhúng* (embedded): **không có tiến trình máy chủ cơ sở dữ liệu riêng**.
- Toàn bộ cơ sở dữ liệu (bảng, chỉ mục, dữ liệu) nằm trong *một file duy nhất* trên đĩa.
- Chương trình đọc/ghi bằng cách *gọi thư viện trực tiếp* (Python dùng module `sqlite3` có sẵn), không gửi yêu cầu qua mạng.

| Tiêu chí | **SQLite** (nhúng) | **MySQL / PostgreSQL** (khách – chủ) |
|---|---|---|
| Kiến trúc | Thư viện nằm **trong** chương trình | **Tiến trình máy chủ riêng**, chương trình kết nối qua mạng (TCP) |
| Lưu trữ | **1 file** (`kiosk.db`) | Thư mục dữ liệu do máy chủ quản lý |
| Cài đặt, vận hành | Không cần cài, không cần tài khoản | Cài máy chủ, tạo người dùng, phân quyền, sao lưu |
| Ghi đồng thời | **Mỗi thời điểm chỉ một bên được ghi** | Nhiều bên ghi song song (khóa theo dòng) |
| Phù hợp | Ứng dụng nhỏ, thiết bị nhúng, một máy | Hệ thống nhiều người dùng, nhiều máy, tải lớn |

**② Trong đồ án** — `backend/database.py`

```python
import sqlite3                                    # L1: thư viện có sẵn của Python
DB_PATH = resolve_database_path()                 # L55: .../backend/kiosk.db
conn = sqlite3.connect(DB_PATH, timeout=10)       # L58: "kết nối" = mở file
```

File thật: `backend/kiosk.db`, khoảng 45 KB.

**③ Vì sao / Nếu không**

- "Kết nối" tới SQLite thực chất là *mở file*: không có địa chỉ IP, cổng hay mật khẩu như MySQL.
- Hệ quả: chỉ chương trình **chạy trên cùng máy** (Raspberry Pi) mới truy cập trực tiếp được. Máy khác (laptop, màn hình bếp) phải đi **qua Smart Kiosk API**.

**④ Ghi nhớ**

- SQLite = cơ sở dữ liệu trong **1 file**, truy cập bằng **thư viện**, không có máy chủ.
- Giới hạn lớn nhất: **mỗi lúc chỉ một bên được ghi**.

---

## 1.2. Vì sao đồ án chọn SQLite

**① Khái niệm**

Lựa chọn cơ sở dữ liệu dựa trên *quy mô dữ liệu*, *số bên ghi cùng lúc*, *tài nguyên phần cứng* và *công sức vận hành*.

**② Trong đồ án**

| Tiêu chí | Thực tế trong đồ án |
|---|---|
| Quy mô | Một cửa hàng, vài chục món, vài trăm đơn mỗi ngày; file cơ sở dữ liệu khoảng 45 KB |
| Phần cứng | Raspberry Pi 4, không cần tốn RAM và CPU cho máy chủ cơ sở dữ liệu chạy thường trực |
| Triển khai | `deploy/pi/install_pi.sh` L34 tạo cơ sở dữ liệu một lần, giữ nguyên nếu đã có `kiosk.db` |
| Lượng ghi | Tạo đơn, chốt đơn, trừ kho, báo xong đơn: ngắn và thưa, một bên ghi tại một thời điểm là đủ |

**③ Vì sao / Nếu không**

- Dùng MySQL phải cài, cấu hình máy chủ, quản lý tài khoản, bảo đảm máy chủ tự khởi động cùng Pi, tức thêm một thành phần có thể hỏng mà quy mô không đòi hỏi.
- Sao lưu SQLite đơn giản: **chép file** (lưu ý chế độ WAL ở mục 2.2).
- **Đánh đổi:** nếu mở rộng thành chuỗi nhiều cửa hàng, nhiều kiosk ghi cùng lúc qua mạng thì nên chuyển sang PostgreSQL hoặc MySQL. Đây là ý tốt cho phần *hướng phát triển*.

**④ Ghi nhớ**

- Lý do chọn: **nhỏ gọn, không máy chủ, hợp Raspberry Pi, đủ cho một cửa hàng**.
- Hướng phát triển: **chuỗi cửa hàng → PostgreSQL**.

---

## 1.3. Thách thức: nhiều tiến trình dùng chung một file

**① Khái niệm**

- *Tiến trình* (process) là một chương trình đang chạy, có bộ nhớ riêng. Hai tiến trình **không thấy biến của nhau**, chỉ chia sẻ được qua file hoặc qua mạng.
- Khi nhiều tiến trình cùng **ghi** một file cơ sở dữ liệu, có hai rủi ro:
  - *Tranh chấp khóa*: bên này ghi thì bên kia bị chặn, có thể báo lỗi `database is locked`.
  - *Xử lý trùng hoặc lệch* (race condition): hai bên cùng đọc "đơn còn chờ" rồi cùng xử lý, dẫn đến trừ kho hai lần.

**② Trong đồ án**

Trên Raspberry Pi có **2 tiến trình cùng mở `kiosk.db`**:

```
┌──────────────────── Raspberry Pi ────────────────────┐
│   [Ứng dụng kiosk]              [Smart Kiosk API]     │
│   kiosk_app/bridge.py           servers/api_server.py │
│          │  ghi                       │ đọc + ghi     │
│          └──────────► kiosk.db ◄──────┘               │
└──────────────────────────────────────▲───────────────┘
                                       │ HTTP (qua mạng)
            Phần mềm quản trị · Màn hình bếp · Kitchen MCP
```

| Tiến trình | Ghi gì vào cơ sở dữ liệu | Mã nguồn |
|---|---|---|
| **Ứng dụng kiosk** | Tạo đơn PENDING, chốt PAID và trừ kho, hủy/hết hạn, ghi LATE_PAID | `bridge.py` L1366, L694, L1049, L788 |
| **Smart Kiosk API** | Báo xong đơn (bếp, trợ lý giọng nói), sửa thực đơn, nhập kho (phần mềm quản trị) | `api_server.py` L23 `import backend.database as db` |

- Flask có thể xử lý nhiều yêu cầu song song trên nhiều luồng (màn hình bếp và trợ lý giọng nói cùng gọi API), nên ngay trong tiến trình API cũng có **nhiều kết nối ghi đồng thời**.
- Máy chủ webhook (`servers/webhook_server.py`) **không ghi cơ sở dữ liệu**: nó chỉ nhận thông báo từ SePay rồi phát tín hiệu qua MQTT; việc chốt đơn vẫn do ứng dụng kiosk làm.

**③ Vì sao / Nếu không**

- Hai tình huống trùng lặp có thật:
  - **webhook và tra cứu SePay** cùng báo một giao dịch;
  - **màn hình bếp và trợ lý giọng nói** cùng báo xong một đơn.
- Không xử lý sẽ dẫn tới lỗi `database is locked`, đơn bị trừ kho hai lần, bếp nhận một đơn hai lần.
- Phần lớn mã trong `database.py` (WAL, `busy_timeout`, `BEGIN IMMEDIATE`, cập nhật có điều kiện) là để giải quyết vấn đề này (mục 2 và mục 5).

**④ Ghi nhớ**

- 2 tiến trình ghi chung 1 file: **ứng dụng kiosk** và **Smart Kiosk API**; webhook **không** ghi cơ sở dữ liệu.
- Rủi ro: **khóa tranh chấp** và **xử lý trùng** → giải ở **mục 2** và **mục 5**.

---

# MỤC 2. KẾT NỐI VÀ CẤU HÌNH (`get_connection`)

**Mã nguồn chung** — `backend/database.py` L57–63. **Mọi hàm** trong file đều mở kết nối qua hàm này, nên cấu hình ở đây áp dụng cho toàn hệ thống.

```python
def get_connection():
    conn = sqlite3.connect(DB_PATH, timeout=10)        # L58
    conn.row_factory = sqlite3.Row                      # L59  → 2.1
    conn.execute("PRAGMA journal_mode=WAL")             # L60  → 2.2
    conn.execute("PRAGMA busy_timeout=10000")           # L61  → 2.3
    conn.execute("PRAGMA foreign_keys=ON")              # L62  → 2.4
    return conn
```

> `PRAGMA` là lệnh riêng của SQLite để **đọc hoặc đổi cấu hình** cơ sở dữ liệu hay kết nối.

## 2.1. `row_factory`: đọc cột theo tên

**① Khái niệm**

- Mặc định mỗi dòng kết quả là một **tuple**, chỉ truy cập theo vị trí: `row[1]`.
- Đặt `row_factory = sqlite3.Row` thì truy cập được *theo tên cột* (`row["payment_status"]`) và đổi được sang `dict`.

**② Trong đồ án**

Kết quả chạy thật:

```
Không row_factory : ('DH1', 'PENDING')
Có row_factory    : PENDING | dict: {'order_id': 'DH1', 'payment_status': 'PENDING'}
```

- L431: `status = row["payment_status"]`
- L236: `return [dict(row) for row in rows]` — trả danh sách món dạng `dict` để API chuyển thành JSON.

**③ Vì sao / Nếu không**

- Truy cập theo vị trí (`row[4]`) dễ sai khi bảng được **thêm cột** sau này; đồ án có thêm cột bằng `ALTER TABLE`, nên thứ tự cột không cố định.
- `dict(row)` giúp API trả JSON ngay, không phải tự ghép tên cột.

**④ Ghi nhớ**

- `sqlite3.Row` → đọc cột **theo tên**, đổi được sang `dict` để trả JSON.

---

## 2.2. Chế độ WAL: đọc và ghi không chặn nhau

**① Khái niệm**

Để ghi an toàn khi mất điện giữa chừng, SQLite cần *file nhật ký*. Có hai cách:

| | **Rollback journal** (mặc định, `DELETE`) | **WAL** (*Write-Ahead Logging*) |
|---|---|---|
| Cách ghi | Chép **bản cũ** của trang dữ liệu ra `*.db-journal`, rồi sửa thẳng file chính | Ghi **thay đổi mới** nối tiếp vào `*.db-wal`, file chính chưa bị đụng |
| Khi commit | Cần **khóa độc quyền**: không ai được đang đọc | Chỉ cần ghi dấu commit vào cuối file WAL |
| Đọc trong lúc ghi | Bên đọc đang giữ giao dịch sẽ **chặn** bên ghi commit | Bên đọc thấy **ảnh chụp cũ**, bên ghi commit bình thường |
| File trên đĩa | `kiosk.db`, `kiosk.db-journal` | `kiosk.db`, `kiosk.db-wal`, `kiosk.db-shm` |

Định kỳ, SQLite gộp nội dung file WAL vào file chính; bước này gọi là *checkpoint*.

**② Trong đồ án** — L60: `PRAGMA journal_mode=WAL`

Thí nghiệm thật: Smart Kiosk API **đang đọc dở** danh sách đơn thì ứng dụng kiosk chốt đơn sang PAID.

```
[DELETE] Kiosk ghi trong lúc API đang đọc dở -> chờ 1.0s rồi LỖI: database is locked
[WAL   ] Kiosk ghi trong lúc API đang đọc dở -> commit THÀNH CÔNG ngay (0.00s) | API đang đọc thấy: PENDING
```

- Ở chế độ WAL, API đọc trọn vẹn bản trước khi chốt, nên dữ liệu không lẫn nửa cũ nửa mới. Lần đọc kế tiếp sẽ thấy PAID.
- File thật trên đĩa: chế độ DELETE có `demo.db`, `demo.db-journal`; chế độ WAL có `demo.db`, `demo.db-shm`, `demo.db-wal`.

**③ Vì sao / Nếu không**

- Màn hình bếp, phần mềm quản trị, Kitchen MCP gọi API để **đọc** liên tục. Không có WAL, đúng lúc khách thanh toán mà API đang đọc thì việc chốt đơn có thể báo `database is locked`.
- Chế độ WAL **lưu ngay trong file cơ sở dữ liệu** (đặt một lần là các lần mở sau vẫn giữ); đặt lại ở mọi kết nối là cách làm an toàn, không tốn gì.
- **Khi sao lưu:** phải chép cả `kiosk.db-wal`, hoặc dừng dịch vụ trước rồi mới chép `kiosk.db`; nếu không, các thay đổi chưa được checkpoint sẽ bị thiếu.

**④ Ghi nhớ**

- WAL: **đọc không chặn ghi, ghi không chặn đọc**; bên đọc thấy ảnh chụp cũ.
- **Hai bên ghi vẫn phải lần lượt** → giải ở mục 2.3 và mục 5.

---

## 2.3. `busy_timeout`: chờ thay vì báo lỗi ngay

**① Khái niệm**

- Kể cả có WAL, **mỗi thời điểm chỉ một bên được ghi**. Bên ghi thứ hai gặp tình trạng *SQLITE_BUSY*.
- `busy_timeout` là số **mili giây** kết nối tự chờ và thử lại trước khi báo lỗi `database is locked`.

**② Trong đồ án**

- L61: `PRAGMA busy_timeout=10000` → 10 giây.
- L58: `sqlite3.connect(..., timeout=10)` cũng là 10 giây (đơn vị **giây**). Hai dòng cùng tác dụng; PRAGMA đặt sau nên là giá trị có hiệu lực.

Thí nghiệm thật với thời gian chờ 0,5 giây:

```
Ghi thứ hai cùng lúc -> chờ 0.5s rồi LỖI: database is locked
```

Với 10 giây trong đồ án, bên thứ hai chỉ chờ bên thứ nhất ghi xong (thường vài mili giây) rồi chạy tiếp.

**③ Vì sao / Nếu không**

- Mặc định là 0: gặp khóa là lỗi ngay, và hai yêu cầu báo xong đơn tới cùng lúc sẽ có một bên thất bại vô cớ.
- Mọi thao tác ghi trong đồ án đều rất ngắn nên 10 giây dư sức. Quá 10 giây vẫn chưa ghi được là có sự cố thật, khi đó báo lỗi là hợp lý.

**④ Ghi nhớ**

- `busy_timeout = 10000 ms` → **xếp hàng chờ ghi** thay vì lỗi ngay.
- `timeout=10` trong `connect` có cùng tác dụng nhưng tính bằng **giây**.

---

## 2.4. `foreign_keys`: bật kiểm tra khóa ngoại

**① Khái niệm**

- *Khóa ngoại* (foreign key) là cột trỏ tới khóa chính của bảng khác, ví dụ `recipe.product_id → product.id`.
- Bật kiểm tra thì cơ sở dữ liệu **từ chối** giá trị trỏ tới dòng không tồn tại (*toàn vẹn tham chiếu*).
- SQLite **mặc định TẮT** kiểm tra này và phải bật **trên từng kết nối**.

**② Trong đồ án**

- L62: `PRAGMA foreign_keys=ON`
- Khóa ngoại khai báo ở L121–122: `recipe.product_id → product(id)`, `recipe.ingredient_id → ingredient(id)`.

Thí nghiệm thật:

```
foreign_keys mặc định = 0
Tắt: chèn công thức trỏ tới món 99 (không tồn tại) -> CHÈN ĐƯỢC (dữ liệu rác)
Bật ngay sau khi mở kết nối -> chèn món 98 không tồn tại: FOREIGN KEY constraint failed
Đặt PRAGMA khi đang giữa giao dịch -> foreign_keys = 0 (KHÔNG bật được)
```

**③ Vì sao / Nếu không**

- Không bật thì `recipe` có thể chứa công thức của món hay nguyên liệu đã bị xóa, khiến phép tính trừ kho (mục 5) sai.
- **Bẫy:** PRAGMA này **không có tác dụng khi đang giữa giao dịch**. Đồ án đặt ngay sau khi mở kết nối (L62), trước mọi câu ghi, nên đúng.
- **Mặt trái:** đây là nguyên nhân của lỗi xóa món đang có công thức (mục 7.2, điểm yếu số 1), vì khóa ngoại không khai báo `ON DELETE CASCADE`.

**④ Ghi nhớ**

- Khóa ngoại SQLite **mặc định tắt**, phải bật **trên từng kết nối**, **trước** mọi giao dịch.
- Bật thì dữ liệu sạch; đổi lại, xóa món có công thức sẽ bị chặn.

---

## 2.5. Đổi đường dẫn cơ sở dữ liệu bằng biến môi trường `KIOSK_DB_PATH`

**① Khái niệm**

- *Biến môi trường* là cặp tên–giá trị hệ điều hành truyền cho chương trình lúc chạy.
- Dùng để **đổi cấu hình mà không sửa mã**.

**② Trong đồ án** — `resolve_database_path()` L10–52

```python
configured_path = os.environ.get("KIOSK_DB_PATH")            # L11
if configured_path is None:
    return os.path.join(CURRENT_DIR, "kiosk.db")             # L15–19: mặc định backend/kiosk.db
if not normalized_path:            raise RuntimeError(...)    # L25: chuỗi rỗng
if not os.path.isabs(...):         raise RuntimeError(...)    # L30: phải là đường dẫn tuyệt đối
if not os.path.isdir(parent):      raise RuntimeError(...)    # L42: thư mục cha phải tồn tại
return os.path.realpath(normalized_path)                      # L50
```

- Chạy **một lần** khi nạp module: `DB_PATH = resolve_database_path()` (L55).
- Bài kiểm thử riêng: `tests/test_database_path_config.py`.

**③ Vì sao / Nếu không**

- **Kiểm thử:** các bài test trỏ biến này tới file cơ sở dữ liệu tạm, không làm bẩn dữ liệu thật (các thí nghiệm trong tài liệu này cũng dùng cách đó).
- **Báo lỗi sớm:** nếu cho phép đường dẫn tương đối, chạy chương trình từ thư mục khác thì SQLite **âm thầm tạo file cơ sở dữ liệu rỗng mới**, và kiosk chạy với dữ liệu trống mà không ai hay. Ba lần kiểm tra ép báo lỗi ngay từ đầu.
- `realpath` bỏ các liên kết tượng trưng, để mọi tiến trình chắc chắn trỏ tới **cùng một file thật**.

**④ Ghi nhớ**

- Mặc định `backend/kiosk.db`; đổi được qua `KIOSK_DB_PATH`.
- Bắt buộc **đường dẫn tuyệt đối** và **thư mục cha tồn tại**, để tránh âm thầm tạo cơ sở dữ liệu rỗng.

---

# MỤC 3. LƯỢC ĐỒ DỮ LIỆU

**Mã nguồn** — `backend/database.py` L70–152; số liệu đọc từ `backend/kiosk.db` (chế độ chỉ đọc).

## 3.1. Năm bảng và chức năng từng bảng

**① Khái niệm**

- *Lược đồ* (schema): bản thiết kế cơ sở dữ liệu, gồm những bảng nào, cột nào, kiểu gì, liên hệ ra sao.
- Mỗi *bảng* lưu một loại đối tượng; mỗi *dòng* là một đối tượng cụ thể.

**② Trong đồ án**

| Bảng | Lưu gì | Số dòng thật | Ai ghi |
|---|---|---|---|
| `product` | Món ăn, đồ uống | 34 | Phần mềm quản trị (qua API) |
| `ingredient` | Nguyên liệu trong kho | 62 | Phần mềm quản trị; tự giảm khi chốt đơn |
| `recipe` | Định lượng nguyên liệu cho 1 phần món | 133 | Phần mềm quản trị |
| `orders` | Đơn hàng và trạng thái | 8 | Ứng dụng kiosk; API (báo xong đơn) |
| `settings` | Cấu hình cửa hàng, mật khẩu, OTP | **luôn 1** | Phần mềm quản trị |

**Hình 1 (vẽ tay) — Sơ đồ quan hệ**

```
 ┌─────────────┐ 1        n ┌───────────────────────┐ n        1 ┌──────────────────────┐
 │ product     │────────────│ recipe                │────────────│ ingredient           │
 │ PK id       │            │ PK,FK product_id      │            │ PK id                │
 │ name        │            │ PK,FK ingredient_id   │            │ name, unit           │
 │ price       │            │ qty_per_unit          │            │ quantity             │
 │ image       │            └───────────────────────┘            │ low_stock_threshold  │
 │ category    │                                                 └──────────────────────┘
 │ status      │
 └─────┬───────┘
       ┆ (sao chép tên, giá vào items_json — KHÔNG có khóa ngoại)
 ┌─────┴───────────────┐            ┌──────────────────────────┐
 │ orders              │            │ settings (id = 1)        │
 │ PK order_id         │            │ thông tin cửa hàng       │
 │ total_amount        │            │ admin_password_hash      │
 │ paid_amount         │            │ recovery_email           │
 │ items_json          │            │ reset_otp_code / ...     │
 │ payment_status      │            └──────────────────────────┘
 │ kitchen_status      │              (đứng riêng, không liên kết)
 │ created_at,closed_at│
 └─────────────────────┘
```

**③ Vì sao / Nếu không**

- Tách **thực đơn** (`product`), **kho** (`ingredient`) và **công thức** (`recipe`) giúp một nguyên liệu (ví dụ sữa đặc) dùng chung cho nhiều món; nhập kho chỉ cần sửa **một dòng**.
- `settings` đứng riêng vì là cấu hình chung của cửa hàng.

**④ Ghi nhớ**

- 5 bảng: **product – recipe – ingredient** (thực đơn và kho), **orders** (đơn), **settings** (cấu hình).

---

## 3.2. Khóa chính, khóa ngoại, khóa ghép và kiểu dữ liệu

**① Khái niệm**

- *Khóa chính* (PRIMARY KEY): xác định **duy nhất** một dòng; không trùng, không rỗng.
- *Khóa ghép* (composite key): khóa chính gồm **nhiều cột**.
- *Khóa ngoại* (FOREIGN KEY): cột trỏ tới khóa chính của bảng khác.
- *AUTOINCREMENT*: tự cấp số tăng dần, **không dùng lại** số của dòng đã xóa.
- *CHECK*: điều kiện mọi dòng phải thỏa.

**② Trong đồ án**

| Bảng | Khóa chính | Ghi chú | Dòng |
|---|---|---|---|
| `product` | `id INTEGER PRIMARY KEY AUTOINCREMENT` | Bộ đếm hiện tại: 34 | L72 |
| `ingredient` | `id ... AUTOINCREMENT` | Bộ đếm hiện tại: 62 | L106 |
| `recipe` | **`(product_id, ingredient_id)`** — khóa ghép | 2 khóa ngoại, `on_delete = NO ACTION` | L120–122 |
| `orders` | `order_id TEXT` | Mã dạng `DH89314220`, do kiosk sinh | L83 |
| `settings` | `id INTEGER PRIMARY KEY CHECK (id = 1)` | Chỉ đúng 1 dòng | L94 |

Kiểu dữ liệu lưu thật (đọc bằng `typeof()`):

- `status BOOLEAN` → lưu là số nguyên **1/0**;
- `created_at DATETIME` → lưu là **chuỗi** `'2026-11-05 22:39:02'`.

**③ Vì sao / Nếu không**

- **Mã đơn là chuỗi**, không phải số tự tăng, vì mã **in trong nội dung chuyển khoản** (`DH` + 8 chữ số); webhook phải tìm lại được đúng chuỗi đó. Mã cũng khó đoán hơn số thứ tự 1, 2, 3.
- **Khóa ghép ở `recipe`**: không thể khai trùng cặp (món A, nguyên liệu X); nếu không, một nguyên liệu có thể bị trừ **hai lần**.
- **`CHECK (id = 1)`**: chặn việc vô tình có dòng cấu hình thứ hai (hai mật khẩu quản trị).
- **SQLite không có kiểu BOOLEAN/DATETIME thật**: chỉ có 5 lớp lưu trữ (NULL, INTEGER, REAL, TEXT, BLOB) và quy kiểu theo *type affinity*. Ngày giờ lưu dạng `YYYY-MM-DD HH:MM:SS` vẫn **so sánh và sắp xếp đúng** vì thứ tự chữ trùng thứ tự thời gian; L392 dựa vào điều này: `created_at <= datetime('now', ..., '-10 minutes')`.

**④ Ghi nhớ**

- Khóa ghép `(product_id, ingredient_id)` → không khai trùng nguyên liệu trong một công thức.
- `order_id` là chuỗi `DH…` vì nó **đi vào nội dung chuyển khoản**.
- BOOLEAN lưu thành số, DATETIME lưu thành chuỗi; chuỗi ISO vẫn so sánh đúng.

---

## 3.3. Quan hệ nhiều–nhiều qua bảng `recipe`

**① Khái niệm**

- *Quan hệ nhiều–nhiều*: một A gắn nhiều B, một B gắn nhiều A.
- Biểu diễn bằng *bảng trung gian* (bảng nối) chứa hai khóa ngoại, tách thành hai quan hệ một–nhiều. Bảng nối mang thêm thông tin riêng của từng cặp (ở đây là định lượng).

**② Trong đồ án**

- Một món dùng nhiều nguyên liệu; một nguyên liệu dùng cho nhiều món.
- `recipe.qty_per_unit` = lượng nguyên liệu cho **1 phần** món. Dữ liệu thật: 133 dòng công thức nối 34 món với 62 nguyên liệu.

Đọc công thức một món — `get_recipe_for_product` L921–925:

```sql
SELECT r.ingredient_id, r.qty_per_unit, i.name, i.unit
FROM recipe r JOIN ingredient i ON r.ingredient_id = i.id
WHERE r.product_id = ?
```

Ghi công thức — `admin_save_recipe` L929–943: **xóa toàn bộ công thức cũ của món rồi chèn lại** danh sách mới.

**③ Vì sao / Nếu không**

- Nhét danh sách nguyên liệu thành chuỗi trong `product` ("pate:20g, chả:30g") thì **không truy vấn được**: không biết món nào dùng pate, không cộng dồn được lượng trừ kho.
- Phép `JOIN` này là nền tảng của **trừ kho tự động** (mục 5.5, 6.1).
- Cách "xóa hết rồi chèn lại" đơn giản; điểm yếu nhỏ là hàm không bọc `try/rollback` (mục 7.2).

**④ Ghi nhớ**

- `recipe` = **bảng nối** product ↔ ingredient, kèm `qty_per_unit`.
- Nhờ `JOIN recipe–ingredient` mới **tính được lượng trừ kho**.

---

## 3.4. Phi chuẩn hóa có chủ đích: `items_json`

**① Khái niệm**

- *Chuẩn hóa* (normalization): mỗi thông tin chỉ lưu **một chỗ**, chỗ khác tham chiếu tới; không trùng, không lệch.
- *Phi chuẩn hóa* (denormalization): cố ý **sao chép** dữ liệu để đổi lấy lợi ích cụ thể — ở đây là *bản chụp lịch sử* (snapshot): đơn ghi đúng những gì khách mua **tại thời điểm mua**.

**② Trong đồ án**

Theo chuẩn hóa lẽ ra cần bảng `order_items(order_id, product_id, qty)`; đồ án thay bằng cột `orders.items_json`.

Đơn mẫu thật `DH89314220`:

```json
[{ "id": "8", "product_id": 8,
   "base_name": "Que pate bơ tỏi", "name": "Que pate bơ tỏi",
   "price": 17000, "qty": 1, "size": "", "ice": "", "note": "",
   "image": "file:///.../banh_mi_que_pate.jpg" }]
```

Nơi tạo bản chụp — `kiosk_app/bridge.py`:

- L1236–1247: mỗi dòng giỏ hàng gồm `name` hiển thị (ví dụ *"Cà phê sữa đá (Size L, Ít đá)"*), `size`, `ice` và **giá đã cộng phụ thu size**;
- L1219–1223: Size M +5.000 đ, Size L +10.000 đ.

Trước khi lưu, `_normalize_order_input` (`database.py` L286–304) kiểm tra từng dòng: có `product_id`, số lượng > 0, giá không âm, có tên.

**③ Vì sao / Nếu không**

- **Giá thay đổi theo thời gian**: tăng giá cà phê từ 22.000 lên 25.000 đ thì đơn tháng trước vẫn phải ghi 22.000 đ. Nếu đơn chỉ lưu `product_id` rồi tra giá hiện tại, doanh thu cũ **bị tính lại sai**.
- **Giá trong đơn vốn khác giá trong `product`** (Size L +10.000 đ); size, đá, ghi chú cũng không có cột nào trong `product`.
- **Món bị xóa hoặc đổi tên** thì hóa đơn cũ vẫn đúng tên.
- **Đọc nhanh**: màn hình bếp, trợ lý giọng nói, báo cáo chỉ đọc một cột JSON là có đủ món, size, ghi chú.
- **Đánh đổi:**
  - khó thống kê theo món bằng một câu SQL, phải đọc JSON từng đơn bằng Python (mục 6.4);
  - lưu dư đường dẫn ảnh `image`;
  - `product_id` trong JSON không có khóa ngoại.

**④ Ghi nhớ**

- `items_json` = **bản chụp đơn lúc mua**: tên, giá (đã gồm size), size, đá, ghi chú.
- Được: đúng lịch sử, đọc nhanh. Mất: khó thống kê bằng SQL, không có khóa ngoại.

---

## 3.5. Nâng cấp lược đồ bằng `ALTER TABLE` mà không mất dữ liệu

**① Khái niệm**

- *Di trú lược đồ* (schema migration): thay đổi cấu trúc bảng của cơ sở dữ liệu **đang có dữ liệu thật** mà không xóa tạo lại.
- `ALTER TABLE ... ADD COLUMN`: thêm cột vào bảng đã có; dòng cũ nhận giá trị mặc định (`DEFAULT`) hoặc `NULL`.

**② Trong đồ án**

`init_database()` chạy **mỗi lần khởi động**: `CREATE TABLE IF NOT EXISTS` (L70–124) rồi thêm các cột ra đời muộn (L128–152):

```python
for alter_sql in [
    "ALTER TABLE settings ADD COLUMN recovery_email TEXT DEFAULT ''",
    "ALTER TABLE settings ADD COLUMN reset_otp_code TEXT DEFAULT ''",
    "ALTER TABLE settings ADD COLUMN reset_otp_expires_at DATETIME",
    "ALTER TABLE settings ADD COLUMN reset_otp_attempts INTEGER DEFAULT 0",
    "ALTER TABLE orders ADD COLUMN paid_amount INTEGER",
    "ALTER TABLE orders ADD COLUMN closed_at DATETIME",
]:
    try:
        cursor.execute(alter_sql); conn.commit()
    except sqlite3.OperationalError:
        pass            # cột đã có từ lần chạy trước
```

- Bảng `orders` thật có đủ 8 cột, trong đó `paid_amount`, `closed_at` được thêm sau.
- Sau đó, nếu bảng trống: tạo 34 món mẫu (L154–209) và dòng `settings` với **mật khẩu quản trị mặc định `123456`** đã băm bằng Werkzeug (L215–225).

**③ Vì sao / Nếu không**

- Cơ sở dữ liệu trên Raspberry Pi đã có dữ liệu thật; muốn thêm cột bằng cách xóa file rồi tạo lại thì **mất hết dữ liệu**.
- SQLite **không có** cú pháp `ADD COLUMN IF NOT EXISTS`, nên đồ án chạy lệnh và bắt lỗi "cột đã tồn tại": chạy lại bao nhiêu lần cũng an toàn (*idempotent*).
- **Hạn chế:**
  - không ghi lại cơ sở dữ liệu đang ở "phiên bản" nào;
  - `except OperationalError: pass` có thể **nuốt** cả lỗi thật (gõ sai SQL);
  - cách chuyên nghiệp hơn: `PRAGMA user_version` để đánh số phiên bản lược đồ, hoặc công cụ như Alembic.
- Mật khẩu mặc định `123456` cần đổi khi triển khai thật.

**④ Ghi nhớ**

- `CREATE TABLE IF NOT EXISTS` + `ALTER TABLE ADD COLUMN` bọc `try/except` → **nâng cấp không mất dữ liệu**, chạy lại nhiều lần vẫn an toàn.
- Hạn chế: không quản lý phiên bản lược đồ, dễ nuốt lỗi thật.

---

# MỤC 4. VÒNG ĐỜI ĐƠN HÀNG

**Mã nguồn** — `backend/database.py` L241–257 và các hàm chuyển trạng thái; nơi gọi: `kiosk_app/bridge.py`, `kiosk_app/qml/PaymentPopup.qml`, `servers/api_server.py`.

## 4.1. Hai trục trạng thái: thanh toán và bếp

**① Khái niệm**

- *Máy trạng thái* (state machine): tại mỗi thời điểm đối tượng chỉ ở **một trạng thái**, và chỉ chuyển theo **bước chuyển hợp lệ** định trước.
- Đơn hàng có *hai trục trạng thái độc lập*, lưu ở hai cột khác nhau.

**② Trong đồ án** — hằng số L252–257

```python
ORDER_PENDING   = "PENDING"     # đã hiện QR, chờ tiền
ORDER_PAID      = "PAID"        # đã nhận đủ tiền
ORDER_CANCELLED = "CANCELLED"   # khách bấm hủy
ORDER_EXPIRED   = "EXPIRED"     # hết giờ chờ
ORDER_LATE_PAID = "LATE_PAID"   # tiền về sau khi đơn đã đóng
CLOSED_ORDER_STATUSES = (ORDER_CANCELLED, ORDER_EXPIRED)
```

| Trục | Cột | Giá trị | Ai quan tâm |
|---|---|---|---|
| **Thanh toán** | `payment_status` | PENDING / PAID / CANCELLED / EXPIRED / LATE_PAID | Kiosk, phần mềm quản trị, doanh thu |
| **Bếp** | `kitchen_status` | `pending` / `done` | Màn hình bếp, trợ lý giọng nói |

Cột thời gian đi kèm: `created_at` (lúc tạo đơn), `closed_at` (lúc đơn **rời PENDING**, L145).

**③ Vì sao / Nếu không**

- Thanh toán và chế biến là **hai quy trình khác nhau**: đơn đã trả tiền nhưng bếp chưa làm xong.
- Gộp chung một cột (PENDING → PAID → COOKING → DONE) thì truy vấn doanh thu phải liệt kê mọi trạng thái "đã trả tiền", rất dễ sót.
- Với hai trục, mỗi câu hỏi nghiệp vụ chỉ cần một điều kiện:
  - **bếp cần làm** = `payment_status='PAID' AND kitchen_status='pending'` (L955–956);
  - **doanh thu** = `payment_status='PAID'` (L640).
- `payment_status` dùng chữ HOA, `kitchen_status` dùng chữ thường: không sai nhưng thiếu nhất quán, cần nhớ khi viết truy vấn.

**④ Ghi nhớ**

- **2 trục**: `payment_status` (5 giá trị) và `kitchen_status` (`pending`/`done`).
- Bếp chỉ thấy đơn **PAID + pending**; doanh thu chỉ tính **PAID**.

---

## 4.2. Sơ đồ chuyển trạng thái

**① Khái niệm**

- *Sơ đồ chuyển trạng thái*: mỗi trạng thái là một ô, mỗi bước chuyển là một mũi tên ghi **sự kiện** gây ra nó.
- Bước chuyển **không có mũi tên** thì **không được phép**.
- *Trạng thái kết thúc*: không còn mũi tên đi ra.

**② Trong đồ án**

**Hình 2 (vẽ tay) — Sơ đồ trạng thái đơn hàng** (tương ứng Hình 3.7 trong báo cáo)

```
                  khách bấm "Thanh toán"
                          │
                          ▼
                    ┌───────────┐  nhận đủ tiền (webhook/tra cứu)  ┌────────┐  bếp bấm xong  ┌──────────────┐
                    │  PENDING  │─────────────────────────────────►│  PAID  │───────────────►│ kitchen:done │
                    └─────┬─────┘  [trừ kho + gửi bếp]             └────────┘                └──────────────┘
         khách bấm hủy    │    hết 300 giây / kiosk khởi động lại
              ┌───────────┴───────────┐
              ▼                       ▼
       ┌────────────┐          ┌───────────┐
       │ CANCELLED  │          │  EXPIRED  │
       └─────┬──────┘          └─────┬─────┘
             └───── tiền về muộn ────┘
                          ▼
                   ┌─────────────┐
                   │  LATE_PAID  │  [KHÔNG trừ kho, KHÔNG gửi bếp → quản trị viên hoàn tiền]
                   └─────────────┘
```

Bước chuyển **bị cấm** và mã chặn tương ứng:

| Bước chuyển bị cấm | Mã chặn |
|---|---|
| PAID → CANCELLED | `close_pending_order` chỉ đổi khi còn PENDING (L367) |
| PAID → PAID lần nữa | `save_new_order` trả `duplicate_order` (L508–516) |
| CANCELLED → PAID | trả `order_closed`, kiosk chuyển sang ghi LATE_PAID |
| LATE_PAID → LATE_PAID lần nữa | trả `already_recorded` (L432–434) |
| done → done lần nữa | `WHERE kitchen_status='pending'` (L984) |

**③ Vì sao / Nếu không**

- **Không cho PAID → CANCELLED:** khách bấm hủy đúng lúc tiền vừa về và đơn đã PAID thì đơn **giữ PAID**, bếp vẫn làm món vì khách đã trả tiền.
- **Không cho CANCELLED → PAID:** đơn đã đóng thì kiosk đã về màn hình thực đơn, khách có thể đã rời đi. Chốt PAID và gửi bếp thì bếp làm món **không ai nhận**, nên đơn thành LATE_PAID để hoàn tiền.
- **Trạng thái kết thúc** (PAID về mặt thanh toán, LATE_PAID) **không thể bị thay đổi nhầm**.

**④ Ghi nhớ**

- PENDING → **PAID** | **CANCELLED** | **EXPIRED**; CANCELLED/EXPIRED → **LATE_PAID**.
- Chỉ **PAID** mới trừ kho và gửi bếp; **LATE_PAID** chỉ ghi nhận tiền để hoàn.

---

## 4.3. Bảng hàm ↔ bước chuyển trạng thái

**① Khái niệm**

- Mỗi bước chuyển do **đúng một hàm** trong `database.py` thực hiện, là **nơi duy nhất** được sửa trạng thái.
- Các tiến trình khác chỉ **gọi** hàm khi có sự kiện, không tự viết SQL sửa trạng thái.

**② Trong đồ án**

| Sự kiện | Ai kích hoạt (nơi gọi) | Hàm trong `database.py` | Bước chuyển |
|---|---|---|---|
| Khách bấm **Thanh toán** | `CartSidebar.qml` L416 → `bridge.checkout()` L1304 → L1366 | `create_pending_order` (L313) | (chưa có) → **PENDING** |
| Khách bấm **Hủy** | `PaymentPopup.qml` L142 → `cancelPayment()` L1062 → L1049 | `close_pending_order(…, CANCELLED)` (L351) | PENDING → **CANCELLED** |
| **Hết 300 giây** | Bộ đếm `PaymentPopup.qml` L25–39 → `expirePayment()` | `close_pending_order(…, EXPIRED)` | PENDING → **EXPIRED** |
| **Kiosk khởi động lại** | `KioskBridge.__init__` L120 | `expire_stale_pending_orders(10)` (L381) | PENDING > 10 phút → **EXPIRED** |
| **Nhận đủ tiền** | `_finalize_payment` L590 → L694 (webhook/tra cứu) | `save_new_order` (L473) | PENDING → **PAID**, trừ kho |
| **Tiền về khi đơn đã đóng** | `save_new_order` trả `order_closed` → `_record_late_payment` L788 | `mark_order_late_paid` (L406) | CANCELLED/EXPIRED → **LATE_PAID** |
| **Bếp bấm HOÀN THÀNH** hoặc nói "xong đơn" | `POST /api/orders/done` (`api_server.py` L520–545) | `admin_mark_order_done` (L968) | kitchen `pending` → **done** |
| Bếp **lấy danh sách** cần làm | `api_server.py` L461 | `admin_get_kitchen_pending_orders` (L945) | (chỉ đọc) |

Tham số thời gian — `bridge.py` L88–92:

| Hằng số | Giá trị | Ý nghĩa |
|---|---|---|
| `POLLING_INTERVAL_SECONDS` | 15 | Cứ 15 giây hỏi SePay một lần |
| `POLLING_TIMEOUT_SECONDS` | 300 | Chờ thanh toán tối đa 5 phút |
| `PENDING_ORDER_STALE_MINUTES` | 10 | Lúc khởi động, đơn PENDING cũ hơn 10 phút bị coi là bỏ dở |

API trả mã theo kết quả (`api_server.py` L533–545): `admin_mark_order_done` trả `False` → **409 Conflict** (`order_not_found_or_already_done`); thiếu mã đơn → **400**. Nhờ đó màn hình bếp biết đơn đã được báo xong ở nơi khác và vẫn gỡ thẻ đơn.

**③ Vì sao / Nếu không**

- **Tập trung một chỗ:** muốn biết đơn đổi trạng thái theo những cách nào chỉ cần đọc `database.py`. Nếu mỗi tiến trình tự viết `UPDATE orders SET ...`, rất dễ có nơi quên điều kiện `AND payment_status='PENDING'`, phá vỡ quy tắc ở mục 4.2.
- **Vì sao cần `expire_stale_pending_orders` lúc khởi động?** Kiosk mất điện khi đang chờ thanh toán thì bộ đếm 300 giây trong QML **chết theo**, đơn kẹt mãi ở PENDING. Lần khởi động sau, hàm đóng các đơn đó thành EXPIRED; tiền về sau đó được ghi đúng là LATE_PAID.
- **Vì sao 10 phút**, không phải 5 phút? Chừa khoảng dư cho đơn vừa tạo vài phút trước khi khởi động lại, khách có thể vẫn đang chuyển tiền.

**④ Ghi nhớ**

- Mỗi bước chuyển = **1 hàm** trong `database.py`; kiosk và API chỉ gọi hàm.
- Bộ đếm **300 giây nằm trong QML**; mất điện thì mất theo → lúc khởi động gọi `expire_stale_pending_orders` (ngưỡng 10 phút).
- Báo xong đơn lần hai → API trả **409**.

---

## 4.4. Vì sao tạo đơn PENDING ngay lúc hiện mã QR

**① Khái niệm**

- *Đối soát* (reconciliation): khớp mỗi khoản tiền nhận được với **đúng một đơn** trong hệ thống.
- Muốn đối soát, mỗi khoản tiền phải mang **mã nhận diện** (mã đơn trong nội dung chuyển khoản), và mã đó phải **có sẵn trong cơ sở dữ liệu** trước khi tiền về.

**② Trong đồ án** — trình tự `checkout()`, `bridge.py` L1304–1400

1. **Sinh mã đơn** (khoảng L1338–1340):
   ```python
   order_code = f"DH{int(time.time()) % 1000000:06d}{random.randint(10, 99)}"
   # "DH" + 6 chữ số cuối của thời gian Unix + 2 chữ số ngẫu nhiên = 8 chữ số
   ```
2. **Chụp giỏ hàng và tổng tiền** tại thời điểm này: `order_items_snapshot`, `order_amount_snapshot`.
3. **Ghi đơn PENDING** (L1366): `database.create_pending_order(...)`. Ghi **thất bại** (ví dụ trùng mã) thì **dừng, không hiện QR** (L1372–1383).
4. **Chỉ khi ghi thành công** mới tạo link QR (L1385–1392), có số tiền và nội dung là mã đơn:
   ```
   https://qr.sepay.vn/img?acc=...&bank=...&amount=<tổng tiền>&des=<mã đơn>
   ```
5. Gửi link lên giao diện: `self.qrGenerated.emit(qr_url)`.

`create_pending_order` (`database.py` L313–348) chỉ lưu đơn, **chưa trừ kho, chưa gửi bếp**; mã trùng (`IntegrityError`) → trả `duplicate_order`.

**③ Vì sao / Nếu không**

So sánh với cách "chỉ ghi khi đã có tiền":

| Tình huống | Có đơn PENDING từ trước (thiết kế hiện tại) | Chỉ ghi khi có tiền |
|---|---|---|
| Khách bấm hủy, rồi vẫn chuyển tiền | Tìm thấy đơn CANCELLED → ghi **LATE_PAID** → hoàn tiền | Không có đơn khớp → tiền **"vô chủ"** |
| Kiosk mất điện khi đang chờ | Đơn còn trong cơ sở dữ liệu → khởi động lại thành EXPIRED → tiền về vẫn đối soát được | Mất toàn bộ thông tin giỏ hàng |
| Thống kê | Thấy cả đơn bị hủy, hết hạn → đo được tỷ lệ khách bỏ ngang | Chỉ thấy đơn thành công |

- Thứ tự **"ghi cơ sở dữ liệu trước, hiện QR sau"** bảo đảm không bao giờ có mã QR trỏ tới đơn **không tồn tại**.
- **Chụp giỏ lúc hiện QR:** giỏ trên màn hình có bị sửa sau đó thì đơn vẫn giữ đúng món và số tiền **đã in trong mã QR**.
- **Mã đơn ngắn, chỉ gồm chữ và số:** ngân hàng có thể cắt bớt hoặc lọc ký tự đặc biệt trong nội dung chuyển khoản; dạng `DH` + 8 chữ số dễ tìm lại bằng biểu thức chính quy ở máy chủ webhook.

**④ Ghi nhớ**

- Trình tự: **sinh mã → chụp giỏ → ghi PENDING → (thành công) → hiện QR**.
- Mục đích: **mọi khoản tiền về đều đối soát được**, kể cả khi khách hủy hoặc kiosk mất điện.
- Mã đơn = `DH` + 6 số cuối thời gian Unix + 2 số ngẫu nhiên.

---

# MỤC 5. XỬ LÝ ĐỒNG THỜI VÀ CHỐNG XỬ LÝ TRÙNG ★

**Mã nguồn** — `backend/database.py` (`save_new_order`, `close_pending_order`, `mark_order_late_paid`, `admin_mark_order_done`); `kiosk_app/bridge.py` L590–680 (lớp bảo vệ ở tầng ứng dụng).

## 5.1. Vấn đề: hai kênh cùng báo thanh toán, hai nơi cùng báo xong đơn

**① Khái niệm**

- *Xử lý đồng thời* (concurrency): nhiều luồng/tiến trình cùng làm việc trên **cùng dữ liệu** trong cùng khoảng thời gian.
- *Điều kiện tranh chấp* (race condition): kết quả **phụ thuộc thứ tự may rủi** giữa các bên.
- Mẫu lỗi kinh điển *kiểm tra rồi hành động* (check-then-act):
  1. A đọc: đơn còn PENDING.
  2. B đọc: đơn còn PENDING.
  3. A chốt đơn, trừ kho.
  4. B **cũng** chốt đơn, trừ kho → **kho bị trừ hai lần**.

**② Trong đồ án**

| Tình huống | Bên A | Bên B | Hậu quả nếu không xử lý |
|---|---|---|---|
| Hai kênh xác nhận thanh toán | Webhook SePay → MQTT `payment/success` | Tra cứu API SePay 15 giây/lần | Trừ kho 2 lần, bếp nhận đơn 2 lần |
| Hủy đúng lúc tiền về | Khách bấm **Hủy** | Tín hiệu thanh toán tới | Đơn CANCELLED mà vẫn trừ kho, hoặc khách mất tiền mà không có món và không được hoàn |
| Hết giờ đúng lúc bấm hủy | Bộ đếm 300 giây | Khách bấm Hủy | Ghi đè trạng thái lẫn nhau |
| Hai nơi báo xong đơn | Màn hình bếp chạm HOÀN THÀNH | Trợ lý giọng nói "xong đơn …" | Đếm trùng, phản hồi sai |

Trong ứng dụng kiosk, tín hiệu MQTT và tra cứu định kỳ chạy trên **hai luồng khác nhau** nên có thể gọi `_finalize_payment` gần như cùng lúc.

**③ Vì sao / Nếu không**

- Đây là **lỗi về tiền và kho**, nghiêm trọng nhất với hệ thống bán hàng.
- Lỗi **hiếm và khó tái hiện**: thử vài lần có thể không gặp, chạy thật hàng nghìn đơn thì sẽ gặp.

**④ Ghi nhớ**

- Lỗi kinh điển: **kiểm tra rồi hành động** bị bên khác chen vào giữa.
- 2 cặp tranh chấp chính: **webhook ↔ tra cứu**, **hủy ↔ tiền về**.

---

## 5.2. Giao dịch (transaction): commit và rollback

**① Khái niệm**

- *Giao dịch*: nhóm thao tác được coi là **một khối không thể tách rời**: `COMMIT` ghi tất cả, `ROLLBACK` hủy tất cả.
- Bốn tính chất *ACID*:

| Tính chất | Nghĩa | Trong đồ án |
|---|---|---|
| **A**tomicity — nguyên tử | Làm hết hoặc không làm gì | Chốt PAID và trừ kho cùng thành công hoặc cùng hủy |
| **C**onsistency — nhất quán | Dữ liệu luôn thỏa ràng buộc | Khóa chính, khóa ngoại, `CHECK (id=1)` |
| **I**solation — cô lập | Không thấy kết quả dở dang của nhau | Bên khác không thấy đơn "đã PAID nhưng chưa trừ kho" |
| **D**urability — bền vững | Đã commit thì mất điện không mất | Nhờ nhật ký WAL |

**② Trong đồ án**

Khuôn mẫu trong `save_new_order` (L500–631) và `mark_order_late_paid` (L421–456):

```python
conn = get_connection()
try:
    conn.execute("BEGIN IMMEDIATE")      # mở giao dịch
    ...  đọc, kiểm tra, cập nhật nhiều bảng ...
    if <bất thường>:
        conn.rollback(); return ...      # thoát sớm: hủy mọi thứ
    conn.commit()                        # tất cả cùng có hiệu lực
except Exception:
    conn.rollback()                      # lỗi giữa chừng: hủy hết
finally:
    conn.close()
```

Mẹo ở L579–580 và L591–594: gặp bất thường thì **chủ động** `raise RuntimeError(...)` để rơi vào `except` → `rollback()`. Không có trường hợp "đã đổi PAID nhưng trừ kho được một nửa".

**③ Vì sao / Nếu không**

Chốt đơn gồm **nhiều câu lệnh trên hai bảng** (một `UPDATE orders`, nhiều `UPDATE ingredient`). Lỗi sau câu đầu mà không có giao dịch thì cơ sở dữ liệu ở trạng thái **nửa vời**: đơn PAID mà kho chưa trừ, hoặc ngược lại.

**④ Ghi nhớ**

- Giao dịch = **tất cả hoặc không gì cả**: `BEGIN` → … → `COMMIT`/`ROLLBACK`.
- Gặp bất thường thì `raise` để rơi vào `rollback`.

---

## 5.3. Kỹ thuật 1: cập nhật có điều kiện và kiểm tra `rowcount`

**① Khái niệm**

- Gộp "kiểm tra" và "hành động" vào **một câu SQL**; cơ sở dữ liệu thực hiện nguyên khối, không bên nào chen vào được. Kỹ thuật này gọi là *so sánh rồi đổi* (compare-and-set), hay *khóa lạc quan*.
- `cursor.rowcount` = **số dòng thật sự bị thay đổi**: **1** là mình thắng; **0** là điều kiện không còn đúng, đã có bên khác làm trước.

**② Trong đồ án**

| Hàm | Câu lệnh (rút gọn) | Ý nghĩa |
|---|---|---|
| `close_pending_order` L364–372 | `UPDATE … SET payment_status=? WHERE order_id=? AND payment_status='PENDING'` → `return rowcount == 1` | Chỉ đóng đơn **còn chờ** |
| `save_new_order` L569–580 | `UPDATE … SET 'PAID' … WHERE order_id=? AND payment_status='PENDING'` → `if rowcount != 1: raise` | Chỉ chốt đơn **còn chờ** |
| `admin_mark_order_done` L978–990 | `UPDATE … SET kitchen_status='done' WHERE … AND payment_status='PAID' AND kitchen_status='pending'` | Chỉ báo xong đơn **đã trả tiền và chưa xong** |
| `expire_stale_pending_orders` L387–397 | `UPDATE … SET 'EXPIRED' WHERE payment_status='PENDING' AND created_at <= …` | Chỉ đóng đơn **còn chờ và đã cũ** |

Thí nghiệm (c): 30 lần cho "khách hủy" và "tiền về" **thật sự cùng lúc**:

```
trạng thái cuối: {'PAID': 25, 'LATE_PAID': 5}
```

- **Tiền tới trước** → PAID; lệnh hủy gặp `rowcount = 0` nên bị bỏ qua, khách vẫn nhận món.
- **Hủy tới trước** → CANCELLED; phía tiền nhận `order_closed` → ghi **LATE_PAID** để hoàn tiền.
- **Không lần nào** ra trạng thái sai (vừa CANCELLED vừa trừ kho).

**③ Vì sao / Nếu không**

- Nếu viết hai bước riêng (`SELECT` thấy PENDING, rồi `UPDATE … WHERE order_id=?` **không kèm điều kiện trạng thái**), lệnh hủy có thể ghi đè lên đơn vừa PAID — đúng mẫu lỗi ở mục 5.1.
- Kỹ thuật này **không cần khóa thêm**, nhanh, đủ an toàn cho **thao tác một câu lệnh** (hủy, hết hạn, báo xong).

**④ Ghi nhớ**

- **Điều kiện trạng thái nằm trong `WHERE`** của chính câu `UPDATE`, rồi kiểm tra `rowcount`.
- `rowcount == 1` → mình thắng; `0` → bên khác đã xử lý trước.

---

## 5.4. Kỹ thuật 2: `BEGIN IMMEDIATE` so với `DEFERRED`

**① Khái niệm**

Với thao tác **nhiều bước** (đọc trạng thái → tính kho → ghi nhiều bảng), một câu `UPDATE` có điều kiện không đủ; cần **giữ quyền ghi suốt quá trình**.

| Kiểu | Khi nào lấy quyền ghi | Khi hai bên cùng "đọc rồi ghi" |
|---|---|---|
| `BEGIN` (**DEFERRED**, mặc định) | **Lúc câu ghi đầu tiên chạy** | Cả hai cùng đọc; bên sau nâng quyền ghi khi dữ liệu đã cũ → SQLite **báo lỗi ngay, không chờ** (`SQLITE_BUSY_SNAPSHOT`) |
| `BEGIN IMMEDIATE` | **Ngay lúc mở giao dịch** | Bên thứ hai **xếp hàng chờ** từ đầu (nhờ `busy_timeout`); đến lượt thì đọc thấy dữ liệu mới nhất |
| `BEGIN EXCLUSIVE` | Ngay từ đầu, chặn cả người đọc | Không cần khi đã dùng WAL |

**② Trong đồ án**

`BEGIN IMMEDIATE` ở `save_new_order` (L502), `mark_order_late_paid` (L423), `admin_mark_order_done` (L977).

Thí nghiệm (a): hai luồng cùng "đọc trạng thái → nếu PENDING thì ghi PAID":

```
BEGIN            -> webhook: LỖI: database is locked | tra cứu: đã chốt PAID
BEGIN IMMEDIATE  -> webhook: thấy PAID, bỏ qua      | tra cứu: đã chốt PAID
```

- **DEFERRED:** bên thua **lỗi ngay** (khoảng 0,2 giây) dù `timeout=5`, vì SQLite không cho chờ trong trường hợp này. Đơn chỉ chốt một lần, nhưng bên thua nhận **lỗi bất ngờ** và phải tự xử lý thử lại.
- **IMMEDIATE:** bên thứ hai **chờ** đến khi bên thứ nhất commit, rồi đọc thấy PAID và **tự bỏ qua êm**. (Thời gian 2,0 giây trong lần chạy thí nghiệm là do cách dựng thí nghiệm; thực tế bên sau chỉ chờ bằng thời gian bên trước giữ giao dịch.)

**③ Vì sao / Nếu không**

- Với DEFERRED, `save_new_order` thỉnh thoảng trả `database_error` khi hai kênh trùng nhau, và kiosk báo lỗi lưu đơn **dù tiền đã về**.
- Với IMMEDIATE, bước đọc đầu hàm (L504–507) **chắc chắn thấy dữ liệu mới nhất**, vì không bên nào ghi chen được cho tới khi mình commit.
- Cái giá: bên **ghi** khác phải chờ, nhưng chốt đơn chỉ mất vài mili giây; nhờ WAL, **bên đọc không bị ảnh hưởng**.

**④ Ghi nhớ**

- **DEFERRED:** lấy quyền ghi muộn → hai bên cùng "đọc rồi ghi" thì **một bên lỗi ngay**.
- **IMMEDIATE:** lấy quyền ghi từ đầu → bên kia **xếp hàng**, đến lượt đọc thấy trạng thái mới.
- Dùng IMMEDIATE cho thao tác **nhiều bước**; thao tác một câu lệnh dùng kỹ thuật 5.3 là đủ.

---

## 5.5. Phân tích từng bước hàm `save_new_order`

**① Khái niệm**

Hàm kết hợp **cả hai kỹ thuật**: giao dịch IMMEDIATE bao ngoài, cập nhật có điều kiện bên trong làm **lớp phòng thủ thứ hai**.

**② Trong đồ án** — L473–631

**Hình 3 (vẽ tay) — Lưu đồ `save_new_order`**

```
 [Bắt đầu] save_new_order(mã đơn, tổng tiền, items_json, số tiền nhận)
     │
 (0) Chuẩn hóa dữ liệu đầu vào ── sai ──► trả invalid_* ─► [Kết thúc]          L488–491
     │
 (1) BEGIN IMMEDIATE                                                          L502
     │   Đọc payment_status của đơn                                           L504–507
     ├── đã PAID ─────────────► ROLLBACK, trả "duplicate_order"               L508–516
     ├── CANCELLED/EXPIRED ───► ROLLBACK, trả "order_closed"  (→ ghi LATE_PAID)
     │   (PENDING hoặc chưa có đơn)
     ▼
 (2) Tính nguyên liệu cần: JOIN recipe–ingredient,                            L518–542
     cộng dồn qty_per_unit × số lượng; ghi nhận nguyên liệu thiếu             L544–553
     │
 (3) Chốt đơn: UPDATE … SET 'PAID' … WHERE … AND payment_status='PENDING'     L569–578
     │   (hoặc INSERT nếu chưa có đơn — tương thích luồng cũ)                 L555–567
     ├── rowcount ≠ 1 ──► raise ──► ROLLBACK ──► "database_error"             L579–580
     ▼
 (4) Trừ kho từng nguyên liệu: quantity = MAX(0, quantity - cần)              L582–594
     ├── rowcount ≠ 1 ──► raise ──► ROLLBACK
     ▼
 (5) Lấy danh sách nguyên liệu sắp hết → COMMIT → trả "ok"                    L596–610
```

Ở **tầng ứng dụng**, trước khi gọi tới đây, `bridge.py` L601–680 có thêm một lớp chặn: khóa Python `self._payment_lock` và cờ `self._payment_confirmed`. Luồng thứ hai trong **cùng tiến trình** kiosk thấy cờ đã bật thì dừng ngay ("đã được xử lý trước đó"), không chạm cơ sở dữ liệu.

**③ Vì sao / Nếu không** — vì sao cần **nhiều lớp**?

| Lớp | Chặn được | Không chặn được |
|---|---|---|
| Khóa Python `_payment_lock` + cờ `_payment_confirmed` | Hai luồng **trong cùng tiến trình** kiosk | Kiosk vừa khởi động lại (cờ mất); tiến trình khác |
| Đọc trạng thái trong `BEGIN IMMEDIATE` | Mọi bên, kể cả khác tiến trình, kể cả sau khi khởi động lại | — (lớp chính) |
| `UPDATE … WHERE payment_status='PENDING'` + `rowcount` | Trường hợp lớp trên bị sửa sai trong tương lai | — (lớp dự phòng) |

- *Phòng thủ nhiều lớp* (defense in depth): một lớp có lỗi thì vẫn còn lớp khác.
- **Trạng thái trong cơ sở dữ liệu** là nguồn sự thật duy nhất: biến trong bộ nhớ mất khi khởi động lại, cơ sở dữ liệu thì không.

**④ Ghi nhớ**

- 5 bước: **kiểm tra trạng thái → tính kho → chốt có điều kiện → trừ kho → commit**.
- 3 lớp chặn trùng: **khóa Python** (cùng tiến trình) → **đọc trong IMMEDIATE** (mọi bên) → **UPDATE có điều kiện** (dự phòng).

---

## 5.6. Kết quả chạy thử: báo thanh toán nhiều lần, kho chỉ trừ một lần

**① Khái niệm**

Kiểm thử đồng thời: cho nhiều luồng gọi **thật sự cùng lúc** (dùng `threading.Barrier` để xuất phát cùng khoảnh khắc), lặp lại nhiều lần để lộ lỗi phụ thuộc thời điểm.

**② Trong đồ án** — chạy trên cơ sở dữ liệu tạm, gọi **đúng các hàm thật** của `database.py`

| Thí nghiệm | Kết quả |
|---|---|
| Gọi `save_new_order` 2 lần **lần lượt** | `ok`, rồi `duplicate_order`; kho 10 → **8** |
| **(b)** 20 đơn, mỗi đơn 2 luồng chốt **cùng lúc** | **20/20** lần `duplicate_order + ok`; đúng 20 đơn PAID |
| **(b)** 3 luồng cùng chốt 1 đơn 2 ổ | Kho 1000 → **998** (trừ 1 lần); kết quả `[duplicate_order, duplicate_order, ok]` |
| **(a)** DEFERRED so với IMMEDIATE | DEFERRED: một bên lỗi `database is locked`; IMMEDIATE: bên sau tự bỏ qua êm |
| **(c)** 30 lần hủy ↔ tiền về | 25 PAID, 5 LATE_PAID, **0 trạng thái sai** |

Đồ án có bài kiểm thử tự động cho vòng đời đơn: `tests/test_order_lifecycle.py`, cố tình chèn một lần hủy **ngay trước** khi chốt đơn (L228–238) và kiểm tra kết quả là LATE_PAID.

**③ Vì sao / Nếu không**

- Chạy tay vài lần **gần như không bao giờ** trúng khoảnh khắc tranh chấp; phải ép các luồng chạy cùng lúc và lặp lại nhiều lần mới kiểm chứng được.
- Các con số này là **bằng chứng thực nghiệm** tốt khi hội đồng hỏi "làm sao chắc không trừ kho hai lần?".

**④ Ghi nhớ**

- Báo thanh toán **2–3 lần cùng lúc** → chỉ **1 lần `ok`**, kho trừ **1 lần**.
- Hủy ↔ tiền về → luôn **PAID** hoặc **LATE_PAID**, không bao giờ sai.

---

# MỤC 6. QUY TẮC NGHIỆP VỤ VÀ THỐNG KÊ

**Mã nguồn** — `backend/database.py` (`save_new_order`, `mark_order_late_paid`, `get_stats_data` L633, `admin_get_orders` L751, `admin_get_report_data` L787); `kiosk_app/bridge.py` L745–900; `admin_app/qml/OrdersView.qml` L21–25.

## 6.1. Trừ kho theo công thức và cảnh báo sắp hết

**① Khái niệm**

- *Trừ kho theo định mức* (backflush): hệ thống **tự tính** lượng nguyên liệu đã dùng dựa trên công thức × số lượng bán, không bắt nhân viên khai báo.
- *Ngưỡng cảnh báo*: mỗi nguyên liệu có mức tối thiểu; bằng hoặc dưới mức đó là "sắp hết".

**② Trong đồ án**

Tính lượng cần trừ — `save_new_order` L518–542:

```python
for item in normalized_items:                       # từng món trong đơn
    rows = SELECT r.ingredient_id, r.qty_per_unit, i.quantity, ...
           FROM recipe r JOIN ingredient i ON i.id = r.ingredient_id
           WHERE r.product_id = ?
    for row in rows:
        required[ingredient_id]["required"] += qty_per_unit * item["qty"]   # cộng dồn
```

Ví dụ: đơn có 2 cà phê sữa và 1 bạc xỉu, cả hai dùng sữa đặc → lượng sữa đặc = 2 × (định lượng cà phê sữa) + 1 × (định lượng bạc xỉu), **cộng dồn vào một nguyên liệu** và trừ **một lần** (L582–594).

Gửi tồn kho xuống màn hình bếp — `bridge.py`:

- Sau khi chốt đơn thành công, `_publish_inventory_snapshot()` (gọi ở L749, định nghĩa ở L816) gửi **toàn bộ bảng nguyên liệu** qua MQTT chủ đề `kitchen/inventory`, mỗi nguyên liệu kèm cờ `"low": quantity <= low_stock_threshold` (L838–846).
- Thông điệp được **ký HMAC**; chưa cấu hình khóa ký thì **không gửi**, màn hình bếp vẫn lấy tồn kho qua API khi đồng bộ định kỳ.
- Thứ tự (L745–760): **tồn kho trước, đơn mới sau**.
- Nơi khác kiểm tra "sắp hết": `get_low_stock_ingredients()` (L913) cho phần mềm quản trị; danh sách `low_stock` trả về từ `save_new_order` (L596–604).

**③ Vì sao / Nếu không**

- Bắt nhân viên tự nhập lượng đã dùng thì không ai làm và số kho sai ngay.
- **Cộng dồn rồi trừ một lần:** mỗi nguyên liệu chỉ một câu UPDATE; kiểm tra đủ/thiếu trên **tổng nhu cầu**.
- **Gửi cả bảng** thay vì phần thay đổi: thông điệp MQTT có thể bị mất; gửi trọn trạng thái thì lần sau màn hình bếp tự đúng lại. Thông điệp còn được broker **giữ lại** (*retained*), nên màn hình bếp vừa bật là có ngay bảng mới nhất.
- **Điểm thừa:** danh sách `low_stock` do `save_new_order` tính sẵn nhưng kiosk không dùng; kiosk đọc lại toàn bộ kho và tự tính cờ `low`.

**④ Ghi nhớ**

- Trừ kho = **Σ (qty_per_unit × số lượng)**, cộng dồn theo nguyên liệu, trong **cùng giao dịch** với bước chốt đơn.
- Sau khi chốt: gửi **toàn bộ tồn kho** (kèm cờ `low`) qua MQTT `kitchen/inventory`, **trước** khi gửi đơn.

---

## 6.2. Kho không đủ mà khách đã trả tiền: vẫn chốt đơn

**① Khái niệm**

Khi số liệu phần mềm mâu thuẫn thực tế, cần chọn **ưu tiên**: *tồn kho phần mềm* chỉ là **ước tính** (định lượng không bao giờ khớp tuyệt đối), còn *giao dịch tiền* là **sự thật đã xảy ra**.

**② Trong đồ án**

- Phát hiện thiếu (L544–553): nguyên liệu nào có lượng còn < lượng cần thì ghi vào `insufficient_stock` (tên, lượng còn, lượng cần, **lượng thiếu**).
- Vẫn chốt đơn và trừ kho (L586):
  ```sql
  UPDATE ingredient SET quantity = MAX(0, quantity - ?) WHERE id = ?
  ```
  `MAX(0, …)` giữ tồn kho **không bao giờ âm**.
- **Thực tế mã hiện tại:** `insufficient_stock` chỉ được **trả về** mà **không nơi nào sử dụng** (trong toàn dự án chỉ xuất hiện ở `database.py`). Người dùng thấy tình trạng thiếu **gián tiếp**: nguyên liệu về 0 nên có cờ `low = True`, màn hình bếp và phần mềm quản trị hiện là **sắp hết/hết**.

**③ Vì sao / Nếu không**

- **Không từ chối đơn** vì khách **đã chuyển tiền**: từ chối nghĩa là phải hoàn tiền và khách không có món, trải nghiệm rất tệ. Thực tế bếp thường vẫn còn hàng vì số trong máy lệch so với kho thật.
- **Kiểm tra kho từ lúc chọn món** có thể làm (hướng phát triển: ẩn món hết nguyên liệu), nhưng lúc chốt đơn vẫn phải xử lý trường hợp kho đổi giữa lúc chọn và lúc trả tiền.
- **Không để tồn kho âm** vì số âm khó hiểu với người quản lý; đổi lại `MAX(0, …)` **mất thông tin** đã thiếu bao nhiêu, mà thông tin này (`insufficient_stock`) hiện chưa được lưu hay hiển thị.
- **Hướng cải tiến:** ghi `insufficient_stock` vào nhật ký hoặc gửi cảnh báo "cần kiểm kê" cho quản trị viên.

**④ Ghi nhớ**

- Khách đã trả tiền → **luôn chốt đơn**; kho dừng ở **0**, không âm.
- `insufficient_stock` được tính nhưng **chưa được sử dụng** → hướng cải tiến.

---

## 6.3. LATE_PAID và hoàn tiền

**① Khái niệm**

*Thanh toán muộn*: tiền về **sau khi** đơn đã đóng (khách hủy hoặc hết 5 phút). Kiosk đã về màn hình thực đơn, khách có thể đã rời đi: không thể làm món, nhưng tiền **đã vào tài khoản** nên phải **hoàn tiền**.

**② Trong đồ án**

`mark_order_late_paid` (L406–456): giao dịch `BEGIN IMMEDIATE`; lưu **số tiền thực nhận** vào `paid_amount`; **không trừ kho, không gửi bếp**. Kết quả trả về:

| Kết quả | Khi nào |
|---|---|
| `late_paid` | Vừa ghi nhận thành công |
| `already_recorded` | Đã ghi trước đó (webhook và tra cứu cùng báo) — **không ghi trùng** |
| `not_closed` | Đơn còn PENDING hoặc đã PAID, để luồng thường xử lý |
| `unknown_order` | Mã đơn không có trong cơ sở dữ liệu |
| `database_error` | Lỗi cơ sở dữ liệu |

Hai đường dẫn tới LATE_PAID trong `bridge.py`:

- mã đơn **không phải đơn đang chờ** (L606–611);
- `save_new_order` trả `order_closed` — khách hủy đúng lúc tiền về (L706–711).

Hiển thị cho quản trị viên:

- `OrdersView.qml` L21–25: đơn LATE_PAID có nhãn **"Cần hoàn tiền"** (biểu tượng "!", màu cam);
- lọc riêng bằng bộ lọc "Thanh toán muộn" trong `admin_get_orders` (L763–764).

**③ Vì sao / Nếu không**

- Không có trạng thái này thì tiền về muộn hoặc **bị bỏ qua** (khách mất tiền), hoặc **bị chốt nhầm PAID và gửi bếp** (làm món không ai nhận).
- **Ghi `paid_amount` riêng** với `total_amount`: số tiền phải hoàn là số **thực nhận**.
- **Hạn chế:** chưa có thao tác "đã hoàn tiền" (chẳng hạn trạng thái REFUNDED). Quản trị viên hoàn tiền thủ công qua ngân hàng xong, đơn vẫn hiện "Cần hoàn tiền" → hướng phát triển: nút đánh dấu đã hoàn, kèm thời điểm hoàn.

**④ Ghi nhớ**

- LATE_PAID = tiền về sau khi đơn đóng → **không làm món, phải hoàn tiền**; `paid_amount` = số tiền thực nhận.
- Chống ghi trùng bằng `already_recorded`.
- Hạn chế: **chưa có trạng thái "đã hoàn tiền"**.

---

## 6.4. Doanh thu chỉ tính đơn PAID

**① Khái niệm**

*Doanh thu* chỉ tính giao dịch bán hàng **đã hoàn tất**; tiền nhận nhầm hoặc sẽ phải hoàn **không phải doanh thu**.

**② Trong đồ án**

Tổng quan trang chủ phần mềm quản trị — `get_stats_data` L633–662, ba truy vấn đều có `payment_status = 'PAID'`:

- doanh thu **tháng này** (`SUM(total_amount)`);
- **số đơn** tháng này (`COUNT(*)`);
- **biểu đồ 7 ngày** (`GROUP BY strftime('%d/%m', created_at)`).

Trang báo cáo — `admin_get_report_data` L787–863:

- lọc theo kỳ `day` / `week` / `month` / `year` (L790–797);
- tổng doanh thu, số đơn, giá trị trung bình mỗi đơn (L799–806);
- **Top 5 món bán chạy**: đọc `items_json` từng đơn PAID bằng Python rồi đếm (L809–848), đúng như phân tích ở mục 3.4.

Danh sách đơn — `admin_get_orders` L757–770: mặc định **ẩn đơn PENDING** (chỉ là QR đang chờ, chưa phải đơn thật).

**③ Vì sao / Nếu không**

- Tính cả CANCELLED/EXPIRED thì doanh thu **thổi phồng** bởi đơn bỏ ngang.
- Tính cả LATE_PAID cũng sai vì khoản đó sẽ **hoàn lại**.
- Điểm yếu thấy khi đọc mã:

| Điểm | Mô tả | Hướng sửa |
|---|---|---|
| Top món tách theo biến thể | L824 gom theo `item["name"]`, tên có kèm size/đá → *"Cà phê sữa đá (Size M)"* và *"(Size L)"* bị đếm là **hai món** | Gom theo `product_id` hoặc `base_name` |
| Thống kê theo **ngày tạo** | Dùng `created_at` thay vì `closed_at`; đơn tạo 23:59, trả tiền 00:01 tính vào ngày hôm trước | Đổi sang `closed_at` |
| Tỷ lệ phương thức thanh toán | L857–860 ghi cứng "Tiền mặt: 0" | Đúng với hệ thống chỉ nhận QR; nên ghi chú để tránh hiểu nhầm |

**④ Ghi nhớ**

- Doanh thu = **Σ total_amount của đơn PAID** trong kỳ; không tính CANCELLED, EXPIRED, LATE_PAID.
- Top món: đọc `items_json` bằng Python; hiện **gom theo tên kèm size**, nên biến thể bị tách riêng.

---

# MỤC 7. ĐÁNH GIÁ VÀ CÂU HỎI ÔN TẬP

## 7.1. Điểm mạnh của thiết kế

| # | Điểm mạnh | Thể hiện ở đâu | Mục |
|---|---|---|---|
| 1 | **Vòng đời đơn rõ ràng, tập trung một chỗ**: mỗi bước chuyển một hàm | `database.py` L241–257 | 4.2, 4.3 |
| 2 | **Hai trục trạng thái độc lập**, truy vấn nghiệp vụ đơn giản | `payment_status`, `kitchen_status` | 4.1 |
| 3 | **Mọi khoản tiền đều đối soát được**: ghi PENDING trước, hiện QR sau | `bridge.py` L1366 → L1385 | 4.4 |
| 4 | **Chống xử lý trùng nhiều lớp**: khóa Python → `BEGIN IMMEDIATE` → `UPDATE` có điều kiện | L502, L569–580 | 5.3–5.5 |
| 5 | **Có bằng chứng thực nghiệm**: báo thanh toán 2–3 lần cùng lúc → kho trừ 1 lần; hủy ↔ tiền về không sai | thí nghiệm (a)(b)(c) | 5.6 |
| 6 | **Phù hợp nhiều tiến trình dùng chung file**: WAL, `busy_timeout`, khóa ngoại | `get_connection` L57–63 | 2.2–2.4 |
| 7 | **Giữ đúng lịch sử bán hàng**: bản chụp `items_json` (giá đã gồm size) | `bridge.py` L1236–1247 | 3.4 |
| 8 | **Thanh toán muộn có lối riêng** (LATE_PAID): không làm món thừa, không mất tiền khách | L406–456 | 6.3 |
| 9 | **Vận hành bền**: nâng cấp lược đồ không mất dữ liệu; đóng đơn treo khi khởi động; tách cơ sở dữ liệu kiểm thử | L128–152, L381, L10–52 | 2.5, 3.5, 4.3 |

## 7.2. Điểm yếu phát hiện được

| # | Điểm yếu | Vị trí | Mức | Hướng sửa |
|---|---|---|---|---|
| 1 | **Xóa món đang có công thức bị lỗi** `FOREIGN KEY constraint failed`, hàm không bắt lỗi (đã chạy thử xác nhận) | `admin_delete_product` L690; khóa ngoại L121 | **Trung bình** | Xóa `recipe` trước; hoặc `ON DELETE CASCADE`; hoặc chỉ **ẩn món** (`status=0`) để giữ lịch sử |
| 2 | **Mất thông tin thiếu kho**: `insufficient_stock` tính nhưng không dùng, kho kẹp về 0 | L544–553, L586 | **Trung bình** | Ghi nhật ký hoặc gửi cảnh báo "cần kiểm kê" |
| 3 | **Chưa có trạng thái "đã hoàn tiền"**: LATE_PAID hiện "Cần hoàn tiền" mãi | `OrdersView.qml` L25 | **Trung bình** | Thêm `REFUNDED` và nút đánh dấu, lưu thời điểm hoàn |
| 4 | Top món bán chạy **tách theo size/đá** | `admin_get_report_data` L824 | Thấp | Gom theo `product_id` hoặc `base_name` |
| 5 | Thống kê theo `created_at` thay vì `closed_at` | L639, L790–797 | Thấp | Đổi sang `closed_at` |
| 6 | Nhiều hàm quản trị **không dùng `try/finally`**, lỗi thì kết nối không được đóng | L664–749, L870–943 | Thấp | Dùng `with`/`finally` như các hàm xử lý đơn |
| 7 | `admin_save_recipe` xóa rồi chèn **không bọc `try/rollback`** | L929–943 | Thấp | Bọc giao dịch, báo lỗi rõ |
| 8 | Nâng cấp lược đồ **nuốt mọi `OperationalError`**, không đánh số phiên bản | L148–152 | Thấp | `PRAGMA user_version` hoặc Alembic |
| 9 | `items_json` lưu dư đường dẫn ảnh; `product_id` trong JSON không có khóa ngoại | `bridge.py` L1242 | Rất thấp | Bỏ `image` khỏi bản chụp |
| 10 | Chữ hoa/thường không nhất quán (`PAID` so với `pending`) | L86–87 | Rất thấp | Thống nhất một kiểu |

> Ngoài ra: **mật khẩu quản trị mặc định `123456`** (L219) là điểm đã biết, sẽ đổi khi triển khai thật.

## 7.3. Câu hỏi kiểu hội đồng

*Tự trả lời trước (chừa 4–5 dòng mỗi câu), sau đó mới đối chiếu gợi ý.*

**Nhóm A — SQLite và cấu hình**

1. Vì sao đồ án chọn SQLite mà không dùng MySQL? Khi nào nên chuyển sang cơ sở dữ liệu khác?
2. Chế độ WAL giải quyết vấn đề gì? Vấn đề nào WAL **không** giải quyết?
3. `busy_timeout` dùng để làm gì? Để bằng 0 thì chuyện gì xảy ra?
4. Vì sao phải bật `foreign_keys` trên **từng kết nối** và ngay sau khi mở?

**Nhóm B — Thiết kế dữ liệu**

5. Vì sao bảng `recipe` dùng khóa chính ghép?
6. Vì sao đơn hàng lưu `items_json` mà không tham chiếu `product`? Đánh đổi là gì?
7. Vì sao mã đơn là chuỗi `DH` + 8 chữ số?

**Nhóm C — Vòng đời và đồng thời ★**

8. Vẽ và giải thích sơ đồ trạng thái đơn hàng.
9. Vì sao tạo đơn PENDING **trước** khi hiện mã QR?
10. Webhook và tra cứu SePay cùng báo một giao dịch: chứng minh kho chỉ bị trừ một lần.
11. `BEGIN IMMEDIATE` khác `BEGIN` ở đâu? Vì sao `save_new_order` cần nó còn `close_pending_order` thì không?
12. Khách bấm hủy đúng lúc tiền về: hệ thống xử lý thế nào?

**Nhóm D — Nghiệp vụ**

13. Kho không đủ mà khách đã trả tiền: xử lý ra sao, vì sao?
14. LATE_PAID là gì? Có tính vào doanh thu không?
15. Nêu ba hạn chế của phần cơ sở dữ liệu và hướng khắc phục.

### Gợi ý trả lời (đọc sau khi tự trả lời)

| Câu | Ý chính cần có |
|---|---|
| 1 | Nhúng, 1 file, không máy chủ, hợp Pi, đủ cho 1 cửa hàng · Chuỗi nhiều cửa hàng, nhiều bên ghi qua mạng → PostgreSQL |
| 2 | Đọc không chặn ghi, ghi không chặn đọc; bên đọc thấy ảnh chụp cũ · **Không** giải quyết việc **hai bên cùng ghi** |
| 3 | Xếp hàng chờ ghi tối đa 10 giây thay vì lỗi ngay · Bằng 0 → gặp khóa là lỗi `database is locked` |
| 4 | SQLite mặc định tắt khóa ngoại, thiết lập theo kết nối · Đặt giữa giao dịch thì bị bỏ qua (đã thí nghiệm) |
| 5 | Không trùng cặp món–nguyên liệu → không trừ kho hai lần cho cùng nguyên liệu |
| 6 | Bản chụp lúc mua: đổi giá, xóa món thì hóa đơn cũ vẫn đúng; giá đã gồm size · Đổi lại: khó thống kê bằng SQL, không có khóa ngoại |
| 7 | Mã đi vào **nội dung chuyển khoản**, webhook tìm lại được; ngắn, chỉ chữ và số; khó đoán hơn số tăng dần |
| 8 | PENDING → PAID / CANCELLED / EXPIRED; CANCELLED, EXPIRED → LATE_PAID; trục bếp pending → done; các bước bị cấm |
| 9 | Mọi khoản tiền về đều có đơn để đối soát, kể cả sau khi hủy hoặc mất điện; không có QR trỏ tới đơn không tồn tại |
| 10 | 3 lớp: khóa Python → đọc trạng thái trong `BEGIN IMMEDIATE` → `UPDATE … WHERE payment_status='PENDING'` + `rowcount` · Thực nghiệm 20/20 lần `ok + duplicate`, kho trừ 1 lần |
| 11 | IMMEDIATE lấy quyền ghi từ đầu, bên kia xếp hàng; DEFERRED thì bên thua lỗi ngay · `save_new_order` nhiều bước nên cần; `close_pending_order` một câu UPDATE có điều kiện là đủ |
| 12 | Bên tới trước thắng: tiền trước → PAID (hủy gặp `rowcount=0`); hủy trước → `order_closed` → LATE_PAID · 30 lần: 25 PAID, 5 LATE_PAID, 0 sai |
| 13 | Vẫn chốt PAID vì khách đã trả tiền; kho kẹp `MAX(0, …)`; số trong máy chỉ là ước tính · Hạn chế: `insufficient_stock` chưa được dùng |
| 14 | Tiền về sau khi đơn đã đóng; không làm món, không trừ kho; lưu `paid_amount` để hoàn · Không tính doanh thu |
| 15 | Chọn 3 trong bảng 7.2 (xóa món có công thức, thiếu trạng thái hoàn tiền, mất thông tin thiếu kho…) kèm hướng sửa |

## Tóm tắt buổi 2 trong 5 dòng

1. **SQLite**: cơ sở dữ liệu nhúng 1 file; 2 tiến trình (kiosk, API) cùng ghi.
2. **Cấu hình**: WAL (đọc/ghi không chặn nhau), `busy_timeout` (xếp hàng ghi), `foreign_keys` (dữ liệu sạch).
3. **Dữ liệu**: product – recipe – ingredient; `orders.items_json` là bản chụp lúc mua; `settings` một dòng.
4. **Vòng đời**: PENDING → PAID / CANCELLED / EXPIRED → LATE_PAID; ghi PENDING trước, hiện QR sau.
5. **Đồng thời**: giao dịch + `BEGIN IMMEDIATE` + `UPDATE` có điều kiện → kho **chỉ trừ một lần** (đã kiểm chứng).

---

# PHỤ LỤC: MÃ NGUỒN CÁC THÍ NGHIỆM

Tất cả thí nghiệm chạy trên **cơ sở dữ liệu tạm**, không đụng `backend/kiosk.db`. Chạy từ thư mục `Smart_Kiosk_Backup`, thay `/duong/dan/tam` bằng một thư mục tạm có thật.

## P.1. WAL và rollback journal: ghi trong lúc có người đang đọc (mục 2.2)

```python
import sqlite3, time, os

P = "/duong/dan/tam/demo2.db"

def tao():
    for f in os.listdir(os.path.dirname(P)):
        if f.startswith("demo2.db"):
            os.remove(os.path.join(os.path.dirname(P), f))
    c = sqlite3.connect(P)
    c.execute("CREATE TABLE orders(order_id TEXT, payment_status TEXT)")
    c.execute("INSERT INTO orders VALUES('DH1','PENDING')")
    c.commit(); c.close()

for mode in ("DELETE", "WAL"):
    tao()
    rd = sqlite3.connect(P, isolation_level=None, timeout=0.2)
    rd.execute(f"PRAGMA journal_mode={mode}")
    rd.execute("BEGIN"); rd.execute("SELECT * FROM orders").fetchall()   # API đang đọc dở
    w = sqlite3.connect(P, timeout=1.0); t = time.time()
    try:
        w.execute("UPDATE orders SET payment_status='PAID'"); w.commit()
        kq = f"commit THÀNH CÔNG ngay ({time.time()-t:.2f}s)"
    except sqlite3.OperationalError as e:
        kq = f"chờ {time.time()-t:.1f}s rồi LỖI: {e}"
    rd.execute("COMMIT"); w.rollback()
    print(f"[{mode}] {kq}")
```

## P.2. Khóa ngoại và bẫy đặt PRAGMA giữa giao dịch (mục 2.4)

```python
import sqlite3
c = sqlite3.connect(":memory:")
c.execute("CREATE TABLE p(id INTEGER PRIMARY KEY)")
c.execute("CREATE TABLE r(pid INTEGER REFERENCES p(id))"); c.commit()
c.execute("PRAGMA foreign_keys=ON")
try:
    c.execute("INSERT INTO r VALUES(98)")
except sqlite3.IntegrityError as e:
    print("Bật đúng chỗ:", e)                       # FOREIGN KEY constraint failed

c2 = sqlite3.connect(":memory:")
c2.execute("CREATE TABLE p(id INTEGER PRIMARY KEY)")
c2.execute("CREATE TABLE r(pid INTEGER REFERENCES p(id))"); c2.commit()
c2.execute("INSERT INTO r VALUES(1)")              # mở giao dịch ngầm
c2.execute("PRAGMA foreign_keys=ON")                # bị bỏ qua
print(c2.execute("PRAGMA foreign_keys").fetchone()[0])   # 0
```

## P.3. Chốt đơn đồng thời và hủy ↔ tiền về (mục 5.6)

```python
import os, sys, json, threading
os.environ["KIOSK_DB_PATH"] = "/duong/dan/tam/b.db"      # đường dẫn tuyệt đối, thư mục phải có sẵn
sys.path.insert(0, "backend")
import database as db

db.init_database()
db.admin_add_ingredient("Bánh mì", "ổ", 1000, 2)
db.admin_save_recipe(1, [{"ingredient_id": 1, "qty_per_unit": 1}])
items = json.dumps([{"product_id": 1, "qty": 2, "name": "Bánh mì đặc biệt", "price": 35000}])

# (b) 3 luồng cùng chốt một đơn
code = "DH99999999"; db.create_pending_order(code, 70000, items)
before = db.get_all_ingredients()[0]["quantity"]
bar, res = threading.Barrier(3), []
def go():
    bar.wait(); res.append(db.save_new_order(code, 70000, items)["reason"])
ts = [threading.Thread(target=go) for _ in range(3)]
[t.start() for t in ts]; [t.join() for t in ts]
print(before, "->", db.get_all_ingredients()[0]["quantity"], sorted(res))
# 1000.0 -> 998.0 ['duplicate_order', 'duplicate_order', 'ok']

# (c) khách hủy và tiền về cùng lúc, lặp 30 lần
dem = {}
for i in range(30):
    code = f"DC{i:08d}"; db.create_pending_order(code, 70000, items)
    bar = threading.Barrier(2)
    def huy():
        bar.wait(); db.close_pending_order(code, "CANCELLED")
    def tien():
        bar.wait()
        if db.save_new_order(code, 70000, items)["reason"] == "order_closed":
            db.mark_order_late_paid(code, 70000)
    ts = [threading.Thread(target=huy), threading.Thread(target=tien)]
    [t.start() for t in ts]; [t.join() for t in ts]
    st = db.get_order_payment_status(code); dem[st] = dem.get(st, 0) + 1
print(dem)          # chỉ có PAID hoặc LATE_PAID, ví dụ {'PAID': 25, 'LATE_PAID': 5}
```

## P.4. Xóa món đang có công thức (mục 7.2, điểm yếu 1)

```python
# tiếp nối P.3 (đã có công thức cho món 1)
try:
    db.admin_delete_product(1)
except Exception as e:
    print(type(e).__name__, e)      # IntegrityError FOREIGN KEY constraint failed
```

# Giải thích chi tiết `backend/database.py`

> File: `Smart_Kiosk_Backup/backend/database.py` — 1248 dòng, bản trên GitHub `main` (commit `85f41af`).
> Mỗi phần: **mã gốc** → **cú pháp Python/SQL** → **logic (vì sao viết vậy)** → **nếu viết khác thì sao**.
> Các kết quả ghi "chạy thật" là đã chạy trên máy (Python 3.10.12, SQLite 3.37.2, Werkzeug 3.1.8).

---

## 0. Bản đồ file

`database.py` là **nơi duy nhất trong hệ thống được đọc/ghi `kiosk.db`**. Các chương trình khác chỉ gọi hàm của nó:

- **Ứng dụng kiosk** (`kiosk_app/bridge.py`, `from backend import database`) gọi 7 hàm vòng đời đơn: `init_database`, `expire_stale_pending_orders`, `create_pending_order`, `close_pending_order`, `save_new_order`, `mark_order_late_paid`, `get_all_products`.
- **Smart Kiosk API** (`servers/api_server.py`, `import ... as db`) gọi các hàm `admin_*`, thống kê, cửa hàng, mật khẩu, OTP.

| Dòng | Phần | Hàm chính |
|---|---|---|
| 1–56 | Import, chọn đường dẫn CSDL | `resolve_database_path`, `DB_PATH` |
| 58–77 | Kết nối và nâng cấp bảng | `get_connection`, `_add_column_if_missing` |
| 80–236 | Tạo bảng, dữ liệu mẫu | `init_database` |
| 238–245 | Thực đơn cho kiosk | `get_all_products` |
| 247–316 | Hằng số trạng thái, kiểm tra đầu vào đơn | `ORDER_*`, `_normalize_order_input` |
| 319–476 | Vòng đời đơn (phần 1) | `create_pending_order`, `close_pending_order`, `expire_stale_pending_orders`, `mark_order_late_paid`, `get_order_payment_status` |
| 479–646 | Chốt đơn + trừ kho | `save_new_order` |
| 648–663 | Trình quản lý kết nối | `_open_db` |
| 666–714 | Thống kê trang chủ | `_PAID_TIME`, `_PERIOD_FILTERS`, `get_stats_data` |
| 717–769 | Quản trị thực đơn | `admin_add/update/delete_product`, `admin_toggle_product_status` |
| 772–819 | Cửa hàng, mật khẩu | `get/update_store_info`, `verify/update_admin_password` |
| 822–925 | Danh sách đơn, báo cáo | `admin_get_orders`, `admin_get_report_data` |
| 927–1029 | Kho và công thức | `*_ingredient`, `get_recipe_for_product`, `admin_save_recipe` |
| 1032–1088 | Màn hình bếp | `admin_get_kitchen_pending_orders`, `admin_mark_order_done` |
| 1090–1237 | Email khôi phục, OTP quên mật khẩu | `generate_password_reset_otp`, `verify_and_reset_password` |
| 1239–1249 | Chạy thử khi gọi trực tiếp | `if __name__ == '__main__'` |

---

## 1. Import và chọn đường dẫn CSDL (L1–56)

```python
import sqlite3
import os
import sys
import json
import datetime
from contextlib import contextmanager

CURRENT_DIR = os.path.dirname(os.path.abspath(__file__))
```

### Cú pháp
| Dòng | Ý nghĩa |
|---|---|
| `import sqlite3` | Module **có sẵn** trong Python để dùng SQLite — không phải `pip install`. |
| `import os` | Làm việc với hệ điều hành: đường dẫn, biến môi trường. |
| `import sys` | ⚠ **Không dùng ở đâu trong file** (import thừa, vô hại). |
| `import json` | Đổi qua lại giữa chuỗi JSON và `list`/`dict` Python (`json.loads`, `json.dumps`). |
| `import datetime` | Tính thời gian hết hạn OTP. |
| `from contextlib import contextmanager` | "Decorator" để viết trình quản lý `with` bằng hàm có `yield` (dùng ở `_open_db`, phần 9). |
| `__file__` | Biến Python tự có: đường dẫn của chính file đang chạy (`.../backend/database.py`). |
| `os.path.abspath(...)` | Đổi thành đường dẫn tuyệt đối. |
| `os.path.dirname(...)` | Bỏ tên file, lấy thư mục → `.../backend`. |

→ `CURRENT_DIR` = thư mục `backend/`, **dù chương trình được chạy từ thư mục nào**. Nếu chỉ viết `"kiosk.db"` (đường dẫn tương đối), chạy kiosk từ `~` sẽ tạo ra một `~/kiosk.db` rỗng mới.

```python
def resolve_database_path():
    configured_path = os.environ.get("KIOSK_DB_PATH")

    if configured_path is None:
        return os.path.join(CURRENT_DIR, "kiosk.db")

    normalized_path = configured_path.strip()
    if not normalized_path:
        raise RuntimeError("KIOSK_DB_PATH must not be empty")
    if not os.path.isabs(normalized_path):
        raise RuntimeError("KIOSK_DB_PATH must be an absolute path")

    parent_directory = os.path.dirname(normalized_path)
    if not os.path.isdir(parent_directory):
        raise RuntimeError("KIOSK_DB_PATH parent directory does not exist")

    return os.path.realpath(normalized_path)


DB_PATH = resolve_database_path()
```
*(Mã gốc xuống dòng nhiều hơn — chỉ là cách trình bày, nghĩa giống hệt.)*

### Cú pháp
- `os.environ.get("KIOSK_DB_PATH")`: đọc **biến môi trường**; không có thì trả `None` (khác `os.environ["..."]` sẽ ném lỗi `KeyError`).
- `is None`: so sánh **đúng là None** (khác `== None` hay `not x`: chuỗi rỗng `""` cũng "falsy" nhưng không phải `None`).
- `.strip()`: bỏ khoảng trắng hai đầu.
- `raise RuntimeError(...)`: **ném lỗi** dừng chương trình kèm thông báo.
- `os.path.join(a, b)`: ghép đường dẫn đúng dấu `/` theo hệ điều hành.
- `os.path.isabs`, `os.path.isdir`: là đường dẫn tuyệt đối không? thư mục có tồn tại không?
- `os.path.realpath`: giải các liên kết tắt (symlink), `..` → đường dẫn thật.
- `DB_PATH = resolve_database_path()` nằm **ngoài hàm** → chạy **một lần duy nhất lúc `import database`**. Mọi hàm sau dùng chung biến toàn cục này.

### Logic — 2 chế độ
| Trường hợp | Kết quả |
|---|---|
| Không đặt `KIOSK_DB_PATH` (chạy thật trên Pi) | `backend/kiosk.db` |
| Đặt `KIOSK_DB_PATH=/tmp/x/test.db` (các bài test) | Dùng CSDL tạm, không đụng dữ liệu thật |
| Đặt rỗng / tương đối / thư mục không tồn tại | **Dừng ngay** với lỗi rõ ràng |

**Vì sao phải chặn kỹ?** `sqlite3.connect()` gặp file chưa có sẽ **lặng lẽ tạo file rỗng**. Gõ sai đường dẫn → kiosk chạy trên một CSDL trống, không ai biết, đơn hàng "biến mất". "Hỏng sớm, hỏng to" (*fail fast*) tốt hơn chạy sai âm thầm.

---

## 2. `get_connection()` — mở kết nối (L58–64)

```python
def get_connection():
    conn = sqlite3.connect(DB_PATH, timeout=10)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA journal_mode=WAL")
    conn.execute("PRAGMA busy_timeout=10000")
    conn.execute("PRAGMA foreign_keys=ON")
    return conn
```

| Dòng | Cú pháp | Tác dụng |
|---|---|---|
| `sqlite3.connect(DB_PATH, timeout=10)` | Trả về đối tượng `Connection` | **Mở file** CSDL; `timeout=10` giây chờ khi gặp khóa |
| `conn.row_factory = sqlite3.Row` | Gán thuộc tính | Mỗi dòng kết quả đọc được **theo tên cột** `row["name"]` và **theo vị trí** `row[0]`; `dict(row)` → `{"tên_cột": giá_trị}` |
| `conn.execute("PRAGMA ...")` | Chạy một câu SQL | `PRAGMA` = lệnh cấu hình riêng của SQLite |
| `journal_mode=WAL` | | Đọc và ghi không chặn nhau (lưu luôn trong file) |
| `busy_timeout=10000` | mili giây | Gặp khóa ghi thì thử lại tối đa 10 s rồi mới báo `database is locked` |
| `foreign_keys=ON` | | Bật kiểm tra khóa ngoại — **mặc định TẮT, phải bật trên từng kết nối** |

**`conn.execute(...)` là gì?** Là lối tắt: tự tạo một `Cursor`, chạy câu lệnh, trả về `Cursor` đó. Trên `Cursor` có:
- `.fetchone()` — lấy 1 dòng (hoặc `None` nếu hết);
- `.fetchall()` — lấy tất cả dòng thành `list`;
- `.rowcount` — **số dòng bị INSERT/UPDATE/DELETE ảnh hưởng** (dùng rất nhiều trong file).

**Vì sao mỗi lần gọi lại mở kết nối mới?** Module `sqlite3` mặc định cấm dùng một kết nối ở luồng khác luồng đã tạo nó (`check_same_thread=True`); API chạy nhiều luồng. Mở file SQLite rất nhanh (< 1 ms) → "cần thì mở, xong thì đóng".

---

## 3. Kiến thức nền bắt buộc: Python `sqlite3` tự mở giao dịch ngầm

Đây là điểm **quan trọng nhất để hiểu đúng** mọi hàm phía sau. Chạy thật:

```
isolation_level mặc định: ''
sau CREATE  in_transaction: False
sau SELECT  in_transaction: False
sau INSERT  in_transaction: True    ← Python TỰ chèn "BEGIN" ngầm trước INSERT
```

Quy tắc của Python 3.10 (chế độ mặc định):
1. Trước câu **INSERT / UPDATE / DELETE / REPLACE**, nếu chưa có giao dịch, Python **tự gửi `BEGIN`** (loại DEFERRED).
2. **SELECT** không tự mở giao dịch.
3. Thay đổi chỉ được lưu khi gọi **`conn.commit()`**. Gọi `conn.close()` mà chưa `commit()` → **mất thay đổi** (tự rollback).
4. Gửi **`conn.execute("BEGIN IMMEDIATE")`** khi chưa có giao dịch → mình tự mở giao dịch **và giành quyền ghi ngay**; Python không chèn thêm `BEGIN` nữa.

Hệ quả:
- Hàm chỉ có **một câu UPDATE có điều kiện** (`close_pending_order`) → để Python mở ngầm là đủ.
- Hàm **đọc rồi mới quyết định ghi** (`save_new_order`, `mark_order_late_paid`, `admin_delete_ingredient`…) → phải tự `BEGIN IMMEDIATE` **trước câu SELECT**, nếu không SELECT chạy ngoài giao dịch và bên khác chen vào được giữa "đọc" và "ghi".

---

## 4. `_add_column_if_missing()` — nâng cấp bảng (L66–77)

```python
def _add_column_if_missing(conn, alter_sql):
    """Chạy ALTER TABLE ... ADD COLUMN; chỉ bỏ qua lỗi 'cột đã tồn tại'. ..."""
    try:
        conn.execute(alter_sql)
        conn.commit()
    except sqlite3.OperationalError as error:
        if "duplicate column" not in str(error).lower():
            raise
```

### Cú pháp
- Tên bắt đầu bằng `_` → **quy ước "hàm nội bộ"**, chỉ dùng trong file này (Python không cấm gọi, chỉ là ước hẹn).
- `"""..."""` ngay dưới `def` → **docstring**: tài liệu của hàm, xem được bằng `help(hàm)`.
- `try: ... except LoạiLỗi as error:` → bắt **đúng một loại lỗi**, gán vào biến `error`.
- `str(error).lower()` → đổi lỗi thành chuỗi chữ thường để tìm từ khóa.
- `raise` **không kèm gì** → ném lại **chính lỗi vừa bắt** (giữ nguyên thông tin gốc).

### Logic
SQLite **không có** `ADD COLUMN IF NOT EXISTS`. Chạy `ALTER TABLE orders ADD COLUMN closed_at` lần thứ 2 → lỗi `duplicate column name: closed_at`. Hàm này: lỗi "cột đã có" → coi như xong; **mọi lỗi khác** (gõ sai SQL, đĩa đầy, file hỏng) → ném ra cho người ta thấy.

→ Gọi bao nhiêu lần cũng cho cùng kết quả (**idempotent**) — vì `init_database()` chạy **mỗi lần khởi động**.

---

## 5. `init_database()` — tạo bảng và dữ liệu mẫu (L80–236)

### 5.1 Mở đầu
```python
def init_database():
    print(f"--> Dang kiem tra Database tai: {DB_PATH}")
    conn = get_connection()
    cursor = conn.cursor()
```
- `f"...{DB_PATH}"` → **f-string**: chèn giá trị biến vào chuỗi.
- `conn.cursor()` → tạo `Cursor` để dùng lại nhiều lần (`cursor.execute`, `cursor.fetchone`).

### 5.2 Năm bảng
```sql
CREATE TABLE IF NOT EXISTS product (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    price INTEGER NOT NULL,
    image TEXT,
    category TEXT DEFAULT 'food',
    status BOOLEAN DEFAULT 1
)
```
| Thành phần | Nghĩa |
|---|---|
| `'''...'''` (Python) | Chuỗi nhiều dòng — để câu SQL dễ đọc |
| `CREATE TABLE IF NOT EXISTS` | Chưa có bảng thì tạo; **có rồi thì bỏ qua, giữ dữ liệu** |
| `INTEGER PRIMARY KEY` | Khóa chính, **trùng với `rowid`** ẩn của SQLite → tra cứu nhanh nhất |
| `AUTOINCREMENT` | Tự cấp số tăng dần, **không tái sử dụng** số của dòng đã xóa |
| `NOT NULL` | Bắt buộc có giá trị |
| `price INTEGER` | Giá lưu **số nguyên đồng** (35000) — không dùng số thực để tránh sai số làm tròn tiền |
| `DEFAULT 'food'` | Không truyền giá trị thì lấy mặc định |
| `status BOOLEAN DEFAULT 1` | SQLite không có kiểu đúng/sai thật → lưu **1/0**; 1 = đang bán |

```sql
CREATE TABLE IF NOT EXISTS orders (
    order_id TEXT PRIMARY KEY,                    -- 'DH' + 8 số, in vào nội dung chuyển khoản
    total_amount INTEGER,
    items_json TEXT,                              -- bản chụp giỏ hàng (JSON)
    payment_status TEXT DEFAULT 'PENDING',        -- trục thanh toán
    kitchen_status TEXT DEFAULT 'pending',        -- trục bếp: pending / done
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
)
```
- **Hai trục trạng thái độc lập**: tiền (`PENDING/PAID/CANCELLED/EXPIRED/LATE_PAID`) và bếp (`pending/done`).
- ⚠ `DEFAULT CURRENT_TIMESTAMP` là giờ **UTC** (chạy thật: `CURRENT_TIMESTAMP` = 14:19 trong khi giờ VN = 21:19). Nhưng **mọi câu INSERT trong file đều tự ghi `datetime('now','localtime')`**, nên giá trị mặc định này không bao giờ được dùng. Chỉ cần biết: nếu ai đó chèn đơn mà quên `created_at` thì giờ sẽ lệch 7 tiếng.
- ⚠ `order_id TEXT PRIMARY KEY` **không có `NOT NULL`** — SQLite (ngoại lệ lịch sử) cho phép NULL ở khóa chính không phải INTEGER. An toàn vì `_normalize_order_input` luôn chặn mã đơn rỗng trước khi ghi.

```sql
CREATE TABLE IF NOT EXISTS settings (
    id INTEGER PRIMARY KEY CHECK (id = 1),
    store_name TEXT DEFAULT '', ..., admin_password_hash TEXT
)
```
- `CHECK (id = 1)` → ràng buộc: dòng nào có `id` khác 1 bị từ chối → bảng **chỉ có tối đa 1 dòng** (cấu hình duy nhất của cửa hàng).
- `admin_password_hash` → lưu **mã băm** của mật khẩu, không lưu mật khẩu gốc (phần 14).

```sql
CREATE TABLE IF NOT EXISTS ingredient (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    unit TEXT NOT NULL,                          -- g, ml, quả...
    quantity REAL NOT NULL DEFAULT 0,            -- tồn kho (số thực: 12.5 g)
    low_stock_threshold REAL NOT NULL DEFAULT 0  -- dưới mức này là "sắp hết"
)
CREATE TABLE IF NOT EXISTS recipe (
    product_id INTEGER NOT NULL,
    ingredient_id INTEGER NOT NULL,
    qty_per_unit REAL NOT NULL,                  -- lượng cho 1 phần món
    PRIMARY KEY (product_id, ingredient_id),     -- khóa ghép
    FOREIGN KEY (product_id) REFERENCES product(id),
    FOREIGN KEY (ingredient_id) REFERENCES ingredient(id)
)
```
- `REAL` = số thực (nguyên liệu tính được 0,5 quả, 12,5 g); tiền thì `INTEGER`.
- `recipe` = **bảng nối nhiều–nhiều** giữa món và nguyên liệu; khóa ghép chặn khai trùng một nguyên liệu cho cùng một món; khóa ngoại chặn trỏ tới món/nguyên liệu không tồn tại.

### 5.3 Nâng cấp bảng cho CSDL cũ (L141–159)
```python
_add_column_if_missing(conn, "ALTER TABLE orders ADD COLUMN kitchen_status TEXT DEFAULT 'pending'")

for alter_sql in [
    "ALTER TABLE settings ADD COLUMN recovery_email TEXT DEFAULT ''",
    "ALTER TABLE settings ADD COLUMN reset_otp_code TEXT DEFAULT ''",
    "ALTER TABLE settings ADD COLUMN reset_otp_expires_at DATETIME",
    "ALTER TABLE settings ADD COLUMN reset_otp_attempts INTEGER DEFAULT 0",
    "ALTER TABLE orders ADD COLUMN paid_amount INTEGER",
    "ALTER TABLE orders ADD COLUMN closed_at DATETIME"
]:
    _add_column_if_missing(conn, alter_sql)
```
- `for x in [ ... ]:` → duyệt từng phần tử của danh sách.
- **Lịch sử phát triển** hiện rõ ở đây: `CREATE TABLE` chỉ chứa các cột **ban đầu**; các cột thêm sau (OTP, `paid_amount`, `closed_at`) được bổ sung bằng `ALTER TABLE`. CSDL **mới** → `CREATE` tạo bảng cũ → `ALTER` thêm cột. CSDL **cũ trên Pi** (đã có dữ liệu) → `CREATE` bỏ qua → `ALTER` thêm cột còn thiếu. **Cả hai cùng ra một lược đồ.**
- `paid_amount`: số tiền **thực nhận** (cần cho đơn LATE_PAID phải hoàn tiền). `closed_at`: lúc đơn rời PENDING — dùng để tính doanh thu theo **thời điểm thanh toán**.

### 5.4 Dữ liệu mẫu (L161–220)
```python
try:
    cursor.execute('SELECT COUNT(*) as count FROM product')
    count = cursor.fetchone()[0]
    if count == 0:
        sample_data = [
            ('Bánh mì đặc biệt', 35000, 'banh_mi_sai_gon_dac_biet.jpg', 'food'),
            ...  # 34 món
        ]
        cursor.executemany('''
            INSERT INTO product (name, price, image, category)
            VALUES (?, ?, ?, ?)
        ''', sample_data)
        conn.commit()
    else:
        print(f"--> Database da co {count} mon an.")
except Exception as e:
    print(f"--> Loi khi khoi tao DB: {e}")
```
- `COUNT(*)` → đếm số dòng; `fetchone()[0]` → lấy cột đầu tiên của dòng kết quả.
- `sample_data` là **list các tuple** — mỗi tuple một món, thứ tự khớp với 4 dấu `?`.
- `executemany(sql, list)` → chạy **cùng một câu** cho **từng tuple** (34 lần) — nhanh hơn và gọn hơn vòng `for`.
- Chỉ chèn khi bảng **trống** → khởi động lại không bị nhân đôi thực đơn.
- `except Exception as e: print(...)` → lỗi ở đây chỉ in ra, không dừng chương trình.

### 5.5 Cấu hình mặc định (L222–234)
```python
cursor.execute('SELECT COUNT(*) FROM settings WHERE id = 1')
if cursor.fetchone()[0] == 0:
    from werkzeug.security import generate_password_hash
    default_hash = generate_password_hash("123456")
    cursor.execute('''INSERT INTO settings (id, ..., admin_password_hash)
                      VALUES (1, '', '', '', '', ?)''', (default_hash,))
    conn.commit()
```
- `from ... import ...` **bên trong hàm** → import "lười": chỉ nạp Werkzeug khi thật sự cần (lần đầu tạo CSDL).
- `(default_hash,)` → **tuple 1 phần tử bắt buộc có dấu phẩy**. Viết `(default_hash)` chỉ là dấu ngoặc bình thường → sai kiểu tham số.
- Mật khẩu mặc định `123456` (đã ghi nhận: để người dùng tự đổi sau).

### 5.6 Kết thúc: `conn.close()` (L236)
⚠ Nhận xét: `init_database` dùng `get_connection()` trực tiếp, không bọc `try/finally`. Nếu một câu `CREATE`/`ALTER` ném lỗi thì hàm dừng giữa chừng mà **chưa `close()`** — nhưng vì đây là lúc khởi động và lỗi đó thì chương trình cũng dừng luôn, nên không gây treo khóa như các hàm quản trị trước đây.

---

## 6. `get_all_products()` — thực đơn cho kiosk (L238–245)

```python
def get_all_products():
    try:
        with _open_db() as conn:
            rows = conn.execute('SELECT * FROM product WHERE status = 1').fetchall()
        return [dict(row) for row in rows]
    except Exception as e:
        print(f"--> Loi khi lay san pham: {e}")
        return []
```
- `with _open_db() as conn:` → mở kết nối, **tự đóng khi ra khỏi khối** (phần 9).
- `SELECT *` → lấy mọi cột; `WHERE status = 1` → chỉ món **đang bán** (món tạm ngưng ẩn khỏi kiosk nhưng vẫn còn trong CSDL).
- `[dict(row) for row in rows]` → **list comprehension**: cách viết gọn của
  ```python
  result = []
  for row in rows:
      result.append(dict(row))
  ```
- Lỗi → trả `[]` (kiosk hiện thực đơn trống thay vì sập).

---

## 7. Vòng đời đơn hàng — hằng số và kiểm tra đầu vào (L247–316)

### 7.1 Hằng số
```python
ORDER_PENDING = "PENDING"
ORDER_PAID = "PAID"
ORDER_CANCELLED = "CANCELLED"
ORDER_EXPIRED = "EXPIRED"
ORDER_LATE_PAID = "LATE_PAID"
CLOSED_ORDER_STATUSES = (ORDER_CANCELLED, ORDER_EXPIRED)
```
- Viết HOA = **quy ước hằng số** (không nên gán lại).
- Dùng tên hằng thay vì gõ chuỗi trực tiếp → gõ sai `ORDER_PAIDD` thì Python báo lỗi ngay; gõ sai `"PAIDD"` thì không ai báo.
- `( ... , ... )` → **tuple**: danh sách không sửa được; `x in CLOSED_ORDER_STATUSES` kiểm tra thành viên.

Sơ đồ trạng thái (chú thích L247–256):
```
checkout() ──► PENDING ──nhận đủ tiền──► PAID ──bếp xong──► kitchen_status = done
                  │
                  ├── khách hủy ─────► CANCELLED ─┐
                  └── hết 5 phút / ──► EXPIRED ───┴── tiền về muộn ──► LATE_PAID
                      kiosk khởi động lại                (không trừ kho, không gửi bếp)
```

### 7.2 `_normalize_order_input()` — "người gác cổng" của dữ liệu đơn
```python
def _normalize_order_input(order_code, total_amount, items_json, result):
    order_code = str(order_code or "").strip()
    if not order_code:
        result["reason"] = "invalid_order_code"
        return None
```
- `order_code or ""` → nếu `order_code` là `None`/rỗng thì dùng `""` (toán tử `or` trả **giá trị đầu tiên "truthy"**).
- `str(...)` → ép thành chuỗi (lỡ truyền số vẫn chạy).
- **Kỹ thuật "tham số kết quả"**: `result` là `dict` do người gọi tạo. `dict` trong Python được truyền **theo tham chiếu** → hàm ghi `result["reason"] = ...` thì người gọi **thấy ngay**. Hàm trả `None` để báo "dừng", còn lý do nằm trong `result`.

```python
    try:
        total_amount = int(float(total_amount))
    except (TypeError, ValueError):
        result["reason"] = "invalid_amount"
        return None
    if total_amount <= 0:
        result["reason"] = "invalid_amount"
        return None
```
- `int(float(x))` → nhận được cả `35000`, `"35000"`, `35000.0`, `"35000.0"`. (Chỉ `int("35000.0")` sẽ lỗi.)
- `except (A, B)` → bắt **nhiều loại lỗi** cùng lúc: `TypeError` (truyền `None`), `ValueError` (truyền `"abc"`).

```python
    try:
        items = json.loads(items_json) if items_json else []
    except (TypeError, json.JSONDecodeError):
        result["reason"] = "invalid_items"
        return None
    if not isinstance(items, list) or not items:
        result["reason"] = "empty_order"
        return None
```
- `A if điều_kiện else B` → **biểu thức điều kiện** (toán tử 3 ngôi).
- `json.loads(chuỗi)` → chuỗi JSON → đối tượng Python (`list`, `dict`...).
- `isinstance(items, list)` → kiểm tra **kiểu**: JSON hợp lệ nhưng là số/`dict` thì vẫn bị từ chối.
- `not items` → danh sách rỗng `[]` là "falsy".

```python
    normalized_items = []
    for raw_item in items:
        if not isinstance(raw_item, dict):
            continue
        try:
            product_id = int(raw_item.get("product_id", 0))
            quantity = int(raw_item.get("qty", raw_item.get("quantity", 0)))
            price = int(float(raw_item.get("price", 0)))
        except (TypeError, ValueError):
            continue
        name = str(raw_item.get("name", "")).strip()
        if product_id <= 0 or quantity <= 0 or not name or price < 0:
            continue
        item = dict(raw_item)
        item["product_id"] = product_id
        item["qty"] = quantity
        item["price"] = price
        item["name"] = name
        normalized_items.append(item)

    if not normalized_items:
        result["reason"] = "invalid_items"
        return None
    return order_code, total_amount, normalized_items
```
- `d.get("key", mặc_định)` → lấy giá trị, không có khóa thì trả mặc định (không ném `KeyError`).
- `raw_item.get("qty", raw_item.get("quantity", 0))` → chấp nhận **cả hai tên** `qty` hoặc `quantity`.
- `continue` → **bỏ qua phần tử này**, sang phần tử kế tiếp (món sai bị loại, món đúng vẫn giữ).
- `dict(raw_item)` → **sao chép** dict, để không sửa dữ liệu gốc của người gọi; các trường khác (`size`, `ice`, `note`, `base_name`, `image`) **được giữ nguyên** trong bản chụp.
- `return a, b, c` → thực chất trả **một tuple** 3 phần tử.

**Vì sao cần hàm này?** Cột `items_json` là chuỗi JSON → CSDL **không thể** kiểm tra bên trong (không khóa ngoại, không kiểu). Hàm này làm thay việc đó **trước khi ghi**, và được **dùng chung** cho `create_pending_order` và `save_new_order` (viết một lần, không lệch nhau).

---

## 8. Các hàm vòng đời đơn (L319–646)

### 8.1 Mẫu quản lý kết nối "conn = None"
Các hàm vòng đời viết theo một khuôn:
```python
conn = None
try:
    conn = get_connection()
    ...                          # làm việc
    conn.commit()
    return ...
except ...:
    ...                          # (có hàm thì rollback)
    return ...
finally:
    if conn is not None:
        conn.close()
```
- `finally` **luôn chạy** — kể cả khi `try` đã `return` hay ném lỗi → kết nối **chắc chắn được đóng**.
- Vì sao `conn = None` trước? Nếu `get_connection()` ném lỗi thì `conn` chưa từng được gán → trong `finally` viết `conn.close()` sẽ lỗi `NameError`. Gán `None` trước và kiểm tra `is not None` tránh điều đó.

### 8.2 `create_pending_order()` — tạo đơn PENDING khi hiện QR (L319–354)
```python
def create_pending_order(order_code, total_amount, items_json):
    result = {"inserted": False, "reason": ""}
    prepared = _normalize_order_input(order_code, total_amount, items_json, result)
    if prepared is None:
        return result
    order_code, total_amount, normalized_items = prepared
```
- `a, b, c = prepared` → **tách tuple** (unpacking) thành 3 biến.

```python
    conn = None
    try:
        conn = get_connection()
        conn.execute(
            """
            INSERT INTO orders (
                order_id, total_amount, items_json,
                payment_status, kitchen_status, created_at
            ) VALUES (?, ?, ?, 'PENDING', 'pending', datetime('now', 'localtime'))
            """,
            (order_code, total_amount,
             json.dumps(normalized_items, ensure_ascii=False))
        )
        conn.commit()
        result["inserted"] = True
        result["reason"] = "ok"
        return result
    except sqlite3.IntegrityError:
        result["reason"] = "duplicate_order"
        return result
    except Exception as error:
        result["reason"] = "database_error"
        print(f"--> Lỗi tạo đơn chờ {order_code}: {error}")
        return result
    finally:
        if conn is not None:
            conn.close()
```
| Điểm | Giải thích |
|---|---|
| `?` | **Tham số**: giá trị gửi riêng, không bao giờ bị hiểu thành lệnh SQL → chống SQL injection |
| `'PENDING'`, `'pending'` viết thẳng | Là **hằng** do lập trình viên viết, không phải dữ liệu từ ngoài → an toàn |
| `datetime('now', 'localtime')` | Hàm của SQLite: giờ hiện tại **theo múi giờ máy** dạng `'2026-11-25 21:19:17'` |
| `json.dumps(..., ensure_ascii=False)` | List → chuỗi JSON, **giữ nguyên chữ tiếng Việt** (không thành `ế`) |
| INSERT ngầm mở giao dịch | Phần 3 — `commit()` mới lưu |
| `except sqlite3.IntegrityError` | Trùng **khóa chính** `order_id` → `duplicate_order` → kiosk **không hiện QR** |
| Thứ tự `except` | Loại **cụ thể** đặt trước, `Exception` (bắt tất cả) đặt sau — ngược lại thì nhánh cụ thể không bao giờ chạy |

**Vì sao ghi PENDING ngay khi hiện QR (chưa có tiền)?** Để **mọi khoản tiền về sau** — kể cả khi khách đã bấm hủy hay kiosk mất điện — đều tìm được một đơn trong CSDL để đối chiếu và hoàn tiền.

### 8.3 `close_pending_order()` — khách hủy / hết giờ (L357–384)
```python
def close_pending_order(order_code, new_status):
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
- `raise ValueError` ở đầu: chỉ cho phép `CANCELLED`/`EXPIRED`. Truyền `"PAID"` vào là **lỗi lập trình** → ném lỗi ngay thay vì âm thầm làm sai (không phải lỗi người dùng nên không trả `False`).
- **UPDATE có điều kiện** `AND payment_status = 'PENDING'`: "kiểm tra" và "sửa" nằm trong **một câu lệnh** — SQLite thực hiện nguyên khối, không bên nào chen vào giữa được.
- `cursor.rowcount == 1` → biểu thức so sánh trả `True/False`:
  - `1` dòng bị sửa → mình đóng đơn thành công;
  - `0` dòng → đơn **không còn PENDING** (tiền vừa về → đã PAID, hoặc đã bị hủy trước) → **không ghi đè**.

**Nếu viết hai bước** (SELECT thấy PENDING → UPDATE không điều kiện): giữa hai bước, tiền về và đơn thành PAID → lệnh hủy ghi đè PAID thành CANCELLED → khách mất tiền, bếp không làm. Thí nghiệm: 30 lần hủy↔tiền về cùng lúc → **0 lần sai**.

### 8.4 `expire_stale_pending_orders()` — dọn đơn treo khi khởi động (L387–409)
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
        conn.commit()
        return cursor.rowcount
```
- `max_age_minutes=10` → **tham số có giá trị mặc định**.
- `datetime('now', 'localtime', '-10 minutes')` → SQLite **cộng/trừ thời gian** bằng "modifier" (chạy thật: 21:19 → 21:09). Modifier được truyền qua `?` (vẫn là tham số an toàn); `int(...)` đảm bảo chỉ là số.
- So sánh `created_at <= ...` giữa **hai chuỗi** vẫn đúng vì định dạng `YYYY-MM-DD HH:MM:SS` có thứ tự chữ = thứ tự thời gian.
- Trả `rowcount` = **số đơn** vừa đóng (kiosk in ra nhật ký).

**Tình huống:** kiosk mất điện khi khách đang quét QR → đơn kẹt PENDING mãi. Lúc khởi động, kiosk gọi hàm này (`bridge.py` L120). Tiền của đơn đó về sau sẽ thành LATE_PAID.

### 8.5 `mark_order_late_paid()` — tiền về khi đơn đã đóng (L412–462)
```python
        conn = get_connection()
        conn.execute("BEGIN IMMEDIATE")
        row = conn.execute(
            "SELECT payment_status FROM orders WHERE order_id = ?", (order_code,)
        ).fetchone()
        if row is None:
            conn.rollback();  return "unknown_order"
        status = row["payment_status"]
        if status == ORDER_LATE_PAID:
            conn.rollback();  return "already_recorded"
        if status not in CLOSED_ORDER_STATUSES:
            conn.rollback();  return "not_closed"
        conn.execute(
            "UPDATE orders SET payment_status = 'LATE_PAID', paid_amount = ? WHERE order_id = ?",
            (paid_amount, order_code))
        conn.commit()
        return "late_paid"
    except Exception as error:
        if conn is not None:
            conn.rollback()
        return "database_error"
```
*(Rút gọn `;` cho dễ nhìn — mã gốc mỗi lệnh một dòng.)*

- **`BEGIN IMMEDIATE` trước SELECT**: giành quyền ghi ngay → từ lúc đọc tới lúc ghi không ai sửa được đơn này. Webhook và tra cứu định kỳ cùng báo tiền về → bên sau **xếp hàng**, tới lượt đọc thấy `LATE_PAID` → `"already_recorded"`, không ghi 2 lần.
- `conn.rollback()` **trước mỗi `return` sớm**: nhả quyền ghi ngay. (Không rollback thì `finally: close()` cũng hủy giao dịch, nhưng viết rõ ràng thì dễ hiểu hơn.)
- Trả **chuỗi mã kết quả** (không chỉ `True/False`) → người gọi biết **chính xác chuyện gì xảy ra** để xử lý: `not_closed` → để luồng chốt đơn thường làm.
- Không đổi `closed_at` → giữ thời điểm đơn bị hủy/hết hạn ban đầu.

### 8.6 `get_order_payment_status()` (L465–476)
```python
        row = conn.execute("SELECT payment_status FROM orders WHERE order_id = ?",
                           (str(order_code or "").strip(),)).fetchone()
        return row["payment_status"] if row else None
```
- Chỉ có `try/finally` (không `except`) → lỗi CSDL **được ném lên cho người gọi** xử lý, nhưng kết nối vẫn chắc chắn đóng.

### 8.7 `save_new_order()` — chốt đơn PAID và trừ kho (L479–646)

Hàm quan trọng nhất của file. Chia thành 8 bước.

**Bước A — chuẩn bị (L487–504)**
```python
result = {"inserted": False, "reason": "", "low_stock": [], "insufficient_stock": []}
prepared = _normalize_order_input(order_code, total_amount, items_json, result)
if prepared is None:
    return result
order_code, total_amount, normalized_items = prepared

try:
    paid_amount = int(float(paid_amount)) if paid_amount is not None else total_amount
except (TypeError, ValueError):
    paid_amount = total_amount

normalized_json = json.dumps(normalized_items, ensure_ascii=False)
```
- `paid_amount=None` (tham số mặc định) → không truyền thì coi như khách trả đúng tổng tiền.
- Mọi việc **không cần CSDL** làm **trước khi mở giao dịch** → thời gian giữ khóa ghi ngắn nhất.

**Bước B — giành quyền ghi và kiểm tra trạng thái (L506–522)**
```python
conn = get_connection()
conn.execute("BEGIN IMMEDIATE")
existing = conn.execute("SELECT payment_status FROM orders WHERE order_id = ?",
                        (order_code,)).fetchone()
if existing is not None and existing["payment_status"] != ORDER_PENDING:
    conn.rollback()
    result["reason"] = ("duplicate_order"
                        if existing["payment_status"] == ORDER_PAID
                        else "order_closed")
    result["payment_status"] = existing["payment_status"]
    return result
```
| `existing` | Nghĩa | Xử lý |
|---|---|---|
| `None` | Chưa có đơn (luồng cũ, không qua PENDING) | Đi tiếp → INSERT mới |
| `PENDING` | Bình thường | Đi tiếp → UPDATE |
| `PAID` | Đã chốt rồi (webhook + tra cứu trùng) | `duplicate_order`, **không trừ kho lần 2** |
| `CANCELLED/EXPIRED/LATE_PAID` | Đơn đã đóng | `order_closed` → người gọi chuyển sang `mark_order_late_paid` |

**Bước C — tính nguyên liệu cần trừ (L524–548)**
```python
required = {}
for item in normalized_items:
    rows = conn.execute("""
        SELECT r.ingredient_id, r.qty_per_unit,
               i.name, i.unit, i.quantity, i.low_stock_threshold
        FROM recipe r
        JOIN ingredient i ON i.id = r.ingredient_id
        WHERE r.product_id = ?
    """, (item["product_id"],)).fetchall()
    for row in rows:
        ingredient_id = int(row["ingredient_id"])
        if ingredient_id not in required:
            required[ingredient_id] = {"id": ingredient_id, "name": row["name"],
                                       "unit": row["unit"],
                                       "available": float(row["quantity"]),
                                       "required": 0.0}
        required[ingredient_id]["required"] += float(row["qty_per_unit"]) * item["qty"]
```
- `recipe r`, `ingredient i` → **bí danh bảng** (alias), viết `r.cột` cho gọn.
- `JOIN ... ON i.id = r.ingredient_id` → **nối** mỗi dòng công thức với dòng nguyên liệu tương ứng (đi theo khóa ngoại). Kết quả: "món này cần nguyên liệu gì, bao nhiêu, kho còn bao nhiêu".
- `required` là `dict` **khóa = id nguyên liệu** → **cộng dồn**: đơn có cà phê sữa đá ×2 và bạc xỉu ×1 đều dùng sữa đặc → gom thành **một** con số tổng (30×2 + 40×1 = 100 ml), trừ kho **một lần**.
- `x += y` → `x = x + y`.

**Bước D — phát hiện thiếu kho (L550–568)**
```python
for info in required.values():
    if info["available"] < info["required"]:
        result["insufficient_stock"].append({..., "missing": info["required"] - info["available"]})

if result["insufficient_stock"]:
    thieu = ", ".join(f"{x['name']} thiếu {x['missing']:g} {x['unit']}"
                      for x in result["insufficient_stock"])
    print(f"--> [CẢNH BÁO KHO] Đơn {order_code}: {thieu}. Cần kiểm kê lại.")
```
- `.values()` → duyệt các giá trị của dict.
- `", ".join(...)` → nối các chuỗi bằng dấu phẩy. Bên trong là **generator expression** (giống list comprehension nhưng không có `[]`).
- `{x:g}` → định dạng số "gọn": `100.0` in thành `100`, `12.5` giữ `12.5`.
- **Quyết định nghiệp vụ:** thiếu kho **vẫn chốt đơn** — khách **đã chuyển tiền**, không thể từ chối. Kho thực tế có thể lệch với số trong máy; chỉ ghi cảnh báo để chủ quán kiểm kê.

**Bước E — chốt đơn (L570–595)**
```python
if existing is None:
    cursor = conn.execute("""INSERT INTO orders (..., payment_status, kitchen_status, created_at,
                                                  paid_amount, closed_at)
                             VALUES (?, ?, ?, 'PAID', 'pending', datetime('now','localtime'), ?,
                                     datetime('now','localtime'))""", (...))
else:
    cursor = conn.execute("""
        UPDATE orders
        SET payment_status = 'PAID', kitchen_status = 'pending',
            total_amount = ?, items_json = ?, paid_amount = ?,
            closed_at = datetime('now', 'localtime')
        WHERE order_id = ? AND payment_status = 'PENDING'
    """, (total_amount, normalized_json, paid_amount, order_code))
if cursor.rowcount != 1:
    raise RuntimeError("Không chốt được đơn hàng")
```
- Vẫn dùng **UPDATE có điều kiện** dù đã kiểm tra ở bước B → **lớp phòng thủ thứ hai** (phòng khi ai đó sau này sửa code bỏ `BEGIN IMMEDIATE`).
- `rowcount != 1` → `raise` → nhảy xuống `except Exception` → **rollback toàn bộ**.
- `kitchen_status = 'pending'` → đơn xuất hiện trên màn hình bếp.

**Bước F — trừ kho (L597–609)**
```python
for info in required.values():
    cursor = conn.execute("""
        UPDATE ingredient SET quantity = MAX(0, quantity - ?) WHERE id = ?
    """, (info["required"], info["id"]))
    if cursor.rowcount != 1:
        raise RuntimeError(f"Không thể trừ nguyên liệu id={info['id']}")
```
- `quantity = quantity - ?` → tính **ngay trong SQL** dựa trên giá trị **hiện tại** của dòng (không đọc lên Python rồi ghi xuống) → không bị "đọc cũ ghi đè".
- `MAX(0, a)` (hàm SQL với 2 tham số = lấy số lớn hơn) → tồn kho **không bao giờ âm**.

**Bước G — lấy danh sách sắp hết (L611–619)**
```sql
SELECT id, name, unit, quantity, low_stock_threshold
FROM ingredient
WHERE quantity <= low_stock_threshold
ORDER BY quantity ASC, name ASC
```
- `ORDER BY a ASC, b ASC` → sắp tăng dần theo lượng còn, trùng thì theo tên.
- Đọc **trong cùng giao dịch** → thấy số liệu **sau khi vừa trừ**. Kiosk dùng danh sách này để cảnh báo.

**Bước H — commit và xử lý lỗi (L621–646)**
```python
    conn.commit()
    result["inserted"] = True
    result["reason"] = "ok"
    return result
except sqlite3.IntegrityError as error:
    if conn is not None:
        conn.rollback()
    text = str(error).lower()
    result["reason"] = ("duplicate_order" if "unique" in text or "primary key" in text
                        else "database_error")
    return result
except Exception as error:
    if conn is not None:
        conn.rollback()
    result["reason"] = "database_error"
    return result
finally:
    if conn is not None:
        conn.close()
```
- `commit()` → **tất cả** (PAID + trừ mọi nguyên liệu) có hiệu lực **cùng một lúc** → **A (Atomicity)** trong ACID.
- Bất kỳ lỗi nào ở bước B–G → `rollback()` → **như chưa từng làm gì**: không có cảnh "đã PAID mà chưa trừ kho".

**Ba lớp chống chốt trùng** (kết hợp với `bridge.py`):
1. Khóa trong Python ở kiosk (chặn hai luồng **cùng tiến trình**).
2. `BEGIN IMMEDIATE` + đọc trạng thái (chặn mọi tiến trình).
3. UPDATE có điều kiện + `rowcount` (dự phòng).

Thí nghiệm: 20 đơn × 3 luồng chốt cùng lúc → mỗi đơn đúng **1 lần `ok`**, kho 1000 → **960** (đúng).

---

## 9. `_open_db()` — trình quản lý kết nối (L648–663)

```python
@contextmanager
def _open_db():
    conn = get_connection()
    try:
        yield conn
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()
```

### Cú pháp
- `@contextmanager` → **decorator**: "bọc" hàm bên dưới để nó dùng được với `with`.
- `yield conn` → hàm **tạm dừng tại đây**, đưa `conn` cho khối `with ... as conn:`; khi khối `with` kết thúc, hàm **chạy tiếp** từ sau `yield`.

### Luồng chạy
```python
with _open_db() as conn:          # 1. chạy tới yield: mở kết nối
    conn.execute(...)             # 2. thân khối with
    conn.commit()
                                  # 3a. thân chạy xong → nhảy vào finally → close()
                                  # 3b. thân ném lỗi   → except: rollback(), raise → finally: close()
```
Tương đương với việc tự viết:
```python
conn = get_connection()
try:
    conn.execute(...)
    conn.commit()
except Exception:
    conn.rollback()
    raise
finally:
    conn.close()
```

### Lưu ý
- `_open_db` **không tự commit** — mỗi hàm ghi phải tự `conn.commit()`. Quên commit → `close()` hủy thay đổi.
- `return` **bên trong** khối `with` vẫn kích hoạt `finally` → vẫn đóng kết nối.
- `except: rollback(); raise` → hủy giao dịch rồi **ném lỗi tiếp** cho người gọi (API sẽ trả mã lỗi phù hợp, ví dụ `IntegrityError` → 409).

### Vì sao ra đời (bài học thật)
Bản cũ các hàm quản trị viết: `conn = get_connection()` … `conn.commit()` … `conn.close()` **không có `finally`**. Xóa món đang có công thức → khóa ngoại ném lỗi → bỏ qua `close()` → **kết nối giữ khóa ghi** → mọi thao tác ghi khác (kể cả kiosk chốt đơn) chờ 10 s rồi `database is locked`. Sau khi chuyển sang `_open_db`: 0,00 s.

---

## 10. Thống kê (L666–714)

```python
_PAID_TIME = "COALESCE(closed_at, created_at)"

_PERIOD_FILTERS = {
    "day":   f"DATE({_PAID_TIME}) = DATE('now', 'localtime')",
    "week":  f"DATE({_PAID_TIME}) >= DATE('now', 'localtime', '-6 days')",
    "month": f"strftime('%Y-%m', {_PAID_TIME}) = strftime('%Y-%m', 'now', 'localtime')",
    "year":  f"strftime('%Y', {_PAID_TIME}) = strftime('%Y', 'now', 'localtime')",
}

def _period_filter(period):
    return _PERIOD_FILTERS.get(period, _PERIOD_FILTERS["month"])
```

| Thành phần | Nghĩa |
|---|---|
| `COALESCE(a, b)` (SQL) | Trả giá trị **đầu tiên không NULL**: có `closed_at` thì dùng, đơn cũ (trước khi có cột này) thì dùng `created_at` |
| `DATE(x)` | Cắt lấy phần ngày `'2026-11-25'` |
| `DATE('now','localtime','-6 days')` | Ngày hôm nay lùi 6 ngày → "7 ngày gần nhất" (tính cả hôm nay) |
| `strftime('%Y-%m', x)` | Định dạng thời gian: `%Y` năm, `%m` tháng, `%d` ngày, `%H` giờ → `'2026-11'` |
| f-string lồng `{_PAID_TIME}` | Ghép **đoạn SQL cố định** vào chuỗi lúc nạp module |
| `.get(period, mặc_định)` | `period` lạ → dùng bộ lọc tháng |

**Về an toàn SQL injection:** f-string ở đây chỉ ghép các **đoạn SQL do lập trình viên viết sẵn** trong `_PERIOD_FILTERS` (danh sách trắng — *whitelist*). Giá trị `period` từ người dùng **chỉ dùng làm khóa tra dict**, không bao giờ được chèn vào câu SQL. Gửi `period="x' OR 1=1"` → không có trong dict → rơi về `"month"`.

```python
def get_stats_data(period='month'):
    date_filter = _period_filter(period)
    if period == "day":
        label_sql, order_sql = f"strftime('%H:00', {_PAID_TIME})", f"strftime('%H', {_PAID_TIME})"
    else:
        label_sql, order_sql = f"strftime('%d/%m', {_PAID_TIME})", f"DATE({_PAID_TIME})"

    with _open_db() as conn:
        row = conn.execute(f'''
            SELECT COALESCE(SUM(total_amount), 0), COUNT(*) FROM orders
            WHERE payment_status = 'PAID' AND {date_filter}
        ''').fetchone()
        revenue, count = int(row[0] or 0), int(row[1] or 0)

        rows = conn.execute(f'''
            SELECT {label_sql} AS day, SUM(total_amount) AS total
            FROM orders
            WHERE payment_status = 'PAID' AND {date_filter}
            GROUP BY {order_sql}
            ORDER BY {order_sql} ASC
        ''').fetchall()
        chart = [{"day": r["day"], "total": int(r["total"] or 0)} for r in rows]

    return {"revenue": revenue, "orders": count, "chart": chart}
```
- `a, b = x, y` → gán **hai biến một lúc**.
- `SUM(...)` / `COUNT(*)` → **hàm tổng hợp**: cộng / đếm trên mọi dòng thỏa điều kiện, trả **một dòng**. Không có đơn nào thì `SUM` trả `NULL` → `COALESCE(..., 0)` đổi thành 0.
- `int(row[0] or 0)` → phòng thêm trường hợp `None`.
- `GROUP BY biểu_thức` → **gom nhóm**: mỗi giờ (xem theo ngày) hoặc mỗi ngày (xem theo tuần/tháng) thành một dòng; `SUM` tính riêng trong từng nhóm → mỗi điểm trên biểu đồ.
- `AS day` → **đặt tên** cột kết quả để đọc `r["day"]`.
- `GROUP BY` theo `DATE(...)` (dạng `2026-11-25`) chứ không theo nhãn `25/11` → sắp xếp đúng thứ tự thời gian và không gộp nhầm 25/11 năm nay với 25/11 năm khác.
- Chỉ tính `PAID` → đơn hủy, hết hạn, thanh toán muộn **không** là doanh thu.

**Bài học:** bản cũ phần mềm quản trị kéo danh sách đơn qua API rồi tự cộng, nhưng API chỉ trả tối đa 500 đơn → 620 đơn chỉ được 10.000.000 đ thay vì 12.400.000 đ. Tính bằng SQL tại chỗ → đúng, và chỉ gửi kết quả nhỏ qua mạng.

---

## 11. Quản trị thực đơn (L717–769)

```python
def admin_add_product(name, category, price, image='default.jpg'):
    with _open_db() as conn:
        conn.execute('''INSERT INTO product (name, category, price, status, image)
                        VALUES (?, ?, ?, 1, ?)''', (name, category, price, image))
        conn.commit()
    return True
```
- Món mới mặc định **đang bán** (`status = 1`). Lỗi → `_open_db` rollback + ném lên API.

```python
def admin_update_product(pid, name, category, price, image=None):
    with _open_db() as conn:
        if image:
            cursor = conn.execute('UPDATE product SET name=?, category=?, price=?, image=? WHERE id=?', ...)
        else:
            cursor = conn.execute('UPDATE product SET name=?, category=?, price=? WHERE id=?', ...)
        conn.commit()
        return cursor.rowcount == 1
```
- Có ảnh mới thì cập nhật ảnh, không thì **giữ ảnh cũ** (2 câu SQL khác nhau).
- `rowcount == 1` → `False` nghĩa là **không có món id đó** → API trả 404.
- Đổi giá ở đây **không ảnh hưởng đơn cũ** vì đơn lưu bản chụp `items_json`.

```python
def admin_delete_product(pid):
    with _open_db() as conn:
        conn.execute("BEGIN IMMEDIATE")
        conn.execute('DELETE FROM recipe WHERE product_id = ?', (pid,))
        cursor = conn.execute('DELETE FROM product WHERE id = ?', (pid,))
        if cursor.rowcount != 1:
            conn.rollback()
            return False
        conn.commit()
        return True
```
- **Xóa bảng con (`recipe`) trước, bảng cha (`product`) sau** → không vi phạm khóa ngoại.
- Hai câu DELETE trong **một giao dịch** → hoặc cùng xóa, hoặc không xóa gì (không có cảnh xóa công thức rồi mà món vẫn còn).
- Món không tồn tại → rollback (hủy luôn câu xóa công thức vô nghĩa) → `False`.

```python
def admin_toggle_product_status(pid, status):
    ...
        cursor = conn.execute('UPDATE product SET status = ? WHERE id = ?', (1 if status else 0, pid))
```
- `1 if status else 0` → đổi giá trị đúng/sai bất kỳ thành 1/0. **Tạm ngưng bán** (ẩn khỏi kiosk) thay vì xóa — giữ được công thức và lịch sử.

---

## 12. Cửa hàng và mật khẩu (L772–819)

```python
def get_store_info():
    with _open_db() as conn:
        row = conn.execute('SELECT store_name, store_address, store_phone, store_email '
                           'FROM settings WHERE id = 1').fetchone()
    if row is None:
        return {"store_name": "", "store_address": "", "store_phone": "", "store_email": ""}
    return dict(row)
```
- Chỉ chọn 4 cột công khai — **không bao giờ** trả `admin_password_hash`, OTP ra ngoài.

```python
def verify_admin_password(password):
    from werkzeug.security import check_password_hash
    with _open_db() as conn:
        row = conn.execute('SELECT admin_password_hash FROM settings WHERE id = 1').fetchone()
    if row is None or not row['admin_password_hash']:
        return False
    return check_password_hash(row['admin_password_hash'], str(password or ""))
```

**Băm mật khẩu (hashing)** — chạy thật với Werkzeug 3.1.8:
```
generate_password_hash('123456') → 'scrypt:32768:8:1$<muối ngẫu nhiên>$<giá trị băm>'
check_password_hash(h, '123456')  → True
check_password_hash(h, '1234567') → False
băm '123456' hai lần              → ra HAI chuỗi khác nhau (vì muối ngẫu nhiên)
```
- **Băm** là biến đổi **một chiều**: từ mật khẩu ra chuỗi băm thì dễ, từ chuỗi băm **không suy ngược** ra mật khẩu.
- **Muối (salt)**: chuỗi ngẫu nhiên trộn vào trước khi băm → hai người cùng mật khẩu vẫn có chuỗi băm khác nhau; không tra được bằng bảng băm có sẵn.
- **scrypt**: thuật toán băm **cố ý chậm và tốn bộ nhớ** → kẻ trộm file CSDL dò mật khẩu bằng máy rất tốn kém.
- Kiểm tra mật khẩu = băm lại mật khẩu nhập vào **với cùng muối** rồi so sánh.
- → Lấy được `kiosk.db` cũng **không đọc được mật khẩu**.

`update_admin_password` băm mật khẩu mới rồi `UPDATE ... WHERE id = 1`; lỗi thì in và trả `False`.

---

## 13. Danh sách đơn và báo cáo (L822–925)

### 13.1 `admin_get_orders()` — lọc, tìm kiếm, phân trang
```python
def admin_get_orders(status_filter, keyword, limit, offset):
    query = "SELECT * FROM orders WHERE 1=1"
    params = []

    status_clauses = {
        "Đã thanh toán": "payment_status = 'PAID'",
        "paid": "payment_status = 'PAID'",
        "Đã hủy": "payment_status IN ('CANCELLED', 'EXPIRED')",
        ...
    }
    query += " AND " + status_clauses.get(status_filter, "payment_status != 'PENDING'")

    if keyword:
        query += " AND order_id LIKE ?"
        params.append(f"%{keyword}%")

    with _open_db() as conn:
        total_items = conn.execute(query.replace("SELECT *", "SELECT COUNT(*)"), params).fetchone()[0]
        query += " ORDER BY created_at DESC LIMIT ? OFFSET ?"
        rows = conn.execute(query, params + [limit, offset]).fetchall()

    return total_items, [dict(row) for row in rows]
```

| Kỹ thuật | Giải thích |
|---|---|
| `WHERE 1=1` | Điều kiện luôn đúng → sau đó cứ nối `" AND ..."` mà **không cần xét** "đây có phải điều kiện đầu tiên không" |
| `status_clauses` | **Whitelist** đoạn SQL; chấp nhận cả nhãn tiếng Việt (từ giao diện) và mã tiếng Anh |
| Mặc định `!= 'PENDING'` | Đơn PENDING là QR đang chờ, **chưa phải đơn thật** → không hiện trừ khi lọc riêng |
| `IN ('A','B')` | Thuộc một trong các giá trị |
| `LIKE ?` + `%từ_khóa%` | Tìm **chứa chuỗi**: `%` = bất kỳ ký tự nào. Từ khóa vẫn đi qua `?` → an toàn |
| `params` là list | Tham số được **thêm dần** cùng với điều kiện, giữ đúng thứ tự với các dấu `?` |
| `query.replace("SELECT *", "SELECT COUNT(*)")` | Dùng **cùng điều kiện lọc** để đếm tổng số đơn (cho phân trang) |
| `ORDER BY created_at DESC` | Mới nhất lên đầu |
| `LIMIT ? OFFSET ?` | **Phân trang**: lấy `limit` dòng, bỏ qua `offset` dòng đầu. Trang 3, mỗi trang 20 → `LIMIT 20 OFFSET 40` |
| `params + [limit, offset]` | Tạo list mới (không sửa `params` đã dùng cho câu đếm) |
| `return a, [..]` | Trả tuple (tổng số, danh sách trang hiện tại) |

### 13.2 `admin_get_report_data()` — báo cáo doanh thu và món bán chạy
```python
    with _open_db() as conn:
        row = conn.execute(f'''SELECT COALESCE(SUM(total_amount), 0), COUNT(*) FROM orders
                               WHERE payment_status = 'PAID' AND {date_filter}''').fetchone()
        total_rev, total_sold = int(row[0] or 0), int(row[1] or 0)   # (mã gốc viết 2 dòng)
        orders = conn.execute(f'''SELECT items_json FROM orders
                                  WHERE payment_status = 'PAID' AND {date_filter}''').fetchall()
        product_rows = conn.execute('SELECT id, name, category FROM product').fetchall()

    avg_order = int(total_rev / total_sold) if total_sold > 0 else 0
    product_map = {r["id"]: (r["name"], r["category"]) for r in product_rows}
```
- Lấy xong dữ liệu thì **ra khỏi `with`** (đóng kết nối) rồi mới xử lý bằng Python → không giữ kết nối lâu.
- `if total_sold > 0 else 0` → tránh **chia cho 0**.
- `{khóa: giá_trị for ...}` → **dict comprehension**: tạo bảng tra `id món → (tên hiện tại, loại)`.

```python
    item_stats = {}
    for (items_json,) in orders:
        if not items_json:
            continue
        try:
            items = json.loads(items_json)
        except (TypeError, ValueError):
            continue
        for item in items if isinstance(items, list) else []:
            if not isinstance(item, dict):
                continue
            pid = item.get('product_id')
            name = str(item.get('base_name') or item.get('name') or 'Unknown')
            key = pid if pid is not None else name
            ...
            if key not in item_stats:
                current = product_map.get(pid)
                item_stats[key] = {"name": current[0] if current else name, "qty": 0,
                                   "revenue": 0, "category": current[1] if current else "food"}
            item_stats[key]["qty"] += qty
            item_stats[key]["revenue"] += qty * price

    top_items_list = sorted(item_stats.values(), key=lambda x: x["qty"], reverse=True)[:5]
```
- `for (items_json,) in orders:` → mỗi dòng có **1 cột** → tách ngay thành biến (unpacking tuple 1 phần tử).
- `for item in (items if isinstance(items, list) else [])` → JSON không phải list thì duyệt danh sách rỗng (bỏ qua).
- **Gom theo `product_id`**: "Bạc xỉu (M)" và "Bạc xỉu (L)" cùng `product_id` → **một món**. Tên hiển thị: tên **hiện tại** trong thực đơn nếu món còn; món đã xóa → dùng `base_name` trong bản chụp.
- `sorted(..., key=lambda x: x["qty"], reverse=True)` → sắp **giảm dần** theo số lượng bán. `lambda x: x["qty"]` = **hàm vô danh** ngắn, trả "tiêu chí sắp xếp" của mỗi phần tử.
- `[:5]` → **cắt lát** lấy 5 phần tử đầu → top 5.

```python
    return {
        "total_revenue": f"{total_rev:,} đ".replace(",", "."),   # 12400000 → "12.400.000 đ"
        "total_sold": f"{total_sold:,}".replace(",", "."),
        "avg_order": f"{avg_order:,} đ".replace(",", "."),
        "top_items": top_items_list,
        "payment_stats": [{"label": "Chuyển khoản / QR", "value": total_sold, "color": "#4318FF"}],
    }
```
- `{x:,}` → định dạng có dấu phẩy ngăn cách hàng nghìn (`12,400,000`) → `.replace(",", ".")` → kiểu Việt Nam `12.400.000`.
- Chỉ một phương thức thanh toán: chuyển khoản qua QR.

---

## 14. Kho và công thức (L927–1029)

| Hàm | Câu SQL chính | Điểm cần chú ý |
|---|---|---|
| `get_all_ingredients` | `SELECT * FROM ingredient ORDER BY name` | Sắp theo tên |
| `admin_add_ingredient` | `INSERT INTO ingredient (...) VALUES (?, ?, ?, ?)` | |
| `admin_update_ingredient` | `UPDATE ingredient SET name, unit, low_stock_threshold` | **Cố ý không sửa `quantity`** ở đây — số lượng chỉ thay đổi qua nhập kho (cộng) hoặc bán (trừ), để có nguồn gốc rõ ràng |
| `admin_delete_ingredient` | `BEGIN IMMEDIATE` → đếm trong `recipe` → `DELETE` | Trả `"ok"` / `"not_found"` / `"in_use"` |
| `admin_restock_ingredient` | `UPDATE ingredient SET quantity = quantity + ?` | Cộng dồn trong SQL |
| `get_low_stock_ingredients` | `WHERE quantity <= low_stock_threshold` | So sánh **hai cột của cùng một dòng** |
| `get_recipe_for_product` | `recipe JOIN ingredient` | Trả công thức kèm tên, đơn vị |
| `admin_save_recipe` | `BEGIN IMMEDIATE` → `DELETE` cũ → `INSERT` mới từng dòng → `commit` | Ghi đè **nguyên khối** |

### `admin_delete_ingredient` — vì sao không xóa luôn công thức như khi xóa món?
```python
        conn.execute("BEGIN IMMEDIATE")
        used = conn.execute('SELECT COUNT(*) FROM recipe WHERE ingredient_id = ?', (iid,)).fetchone()[0]
        if used:
            conn.rollback()
            return "in_use"
        cursor = conn.execute('DELETE FROM ingredient WHERE id = ?', (iid,))
        if cursor.rowcount != 1:
            conn.rollback()
            return "not_found"
        conn.commit()
        return "ok"
```
- Xóa **món** → công thức **của chính món đó** vô nghĩa → xóa theo là đúng.
- Xóa **nguyên liệu** → nếu xóa ngầm các dòng công thức dùng nó, **các món khác** sẽ âm thầm **ngừng trừ nguyên liệu đó** mà chủ quán không biết → chọn **từ chối** (`in_use` → API trả 409) để chủ quán sửa công thức trước.
- `if used:` → số 0 là "falsy", khác 0 là "truthy".
- Đếm và xóa trong **cùng giao dịch `BEGIN IMMEDIATE`** → giữa lúc đếm và lúc xóa không ai thêm được công thức mới dùng nguyên liệu này.

### `admin_restock_ingredient` — trả về dòng sau khi cập nhật
```python
        conn.execute("BEGIN IMMEDIATE")
        cursor = conn.execute('UPDATE ingredient SET quantity = quantity + ? WHERE id = ?', (add_quantity, iid))
        if cursor.rowcount != 1:
            conn.rollback()
            return None
        row = conn.execute('SELECT * FROM ingredient WHERE id = ?', (iid,)).fetchone()
        conn.commit()
    return dict(row) if row else None
```
- `quantity = quantity + ?` thay vì đọc lên Python cộng rồi ghi xuống → nếu đúng lúc đó kiosk vừa bán (trừ kho), hai thay đổi **không ghi đè lên nhau**.
- Đọc lại dòng **trong cùng giao dịch** → trả đúng số lượng mới cho phần mềm quản trị hiển thị.

### `admin_save_recipe` — ghi đè nguyên khối
```python
        conn.execute("BEGIN IMMEDIATE")
        conn.execute('DELETE FROM recipe WHERE product_id = ?', (product_id,))
        for item in recipe_items:
            conn.execute('INSERT INTO recipe (product_id, ingredient_id, qty_per_unit) VALUES (?, ?, ?)',
                         (product_id, item["ingredient_id"], item["qty_per_unit"]))
        conn.commit()
```
- "Xóa hết rồi thêm lại" đơn giản hơn so sánh từng dòng cũ/mới.
- Một dòng lỗi (nguyên liệu không tồn tại → khóa ngoại; trùng nguyên liệu → khóa ghép) → `_open_db` **rollback** → **công thức cũ còn nguyên**, không bao giờ ở trạng thái "xóa rồi mà chưa thêm".

---

## 15. Màn hình bếp (L1032–1088)

```python
def admin_get_kitchen_pending_orders():
    ...
        rows = conn.execute("""
            SELECT order_id, total_amount, items_json, payment_status, kitchen_status, created_at
            FROM orders
            WHERE payment_status = 'PAID' AND kitchen_status = 'pending'
            ORDER BY created_at ASC
        """).fetchall()
```
- Bếp chỉ thấy đơn **đã trả tiền** và **chưa làm xong**; `ASC` → đơn **cũ nhất lên trước** (ai đến trước làm trước).
- Liệt kê cột cụ thể thay vì `SELECT *` → không gửi thừa `paid_amount`, `closed_at` cho bếp.

```python
def admin_mark_order_done(order_code):
    ...
        conn.execute("BEGIN IMMEDIATE")
        cursor = conn.execute("""
            UPDATE orders SET kitchen_status = 'done'
            WHERE order_id = ? AND payment_status = 'PAID' AND kitchen_status = 'pending'
        """, (order_code,))
        if cursor.rowcount != 1:
            conn.rollback()
            return False
        conn.commit()
        return True
```
- **Ba điều kiện** trong `WHERE`: đúng mã, **đã trả tiền**, **chưa xong**. Không đủ → 0 dòng → `False` → API trả **409** (đơn không ở trạng thái báo xong được).
- Màn hình bếp và trợ lý giọng nói **cùng báo xong** một đơn → chỉ bên đầu tiên được `True`, bên sau `False` — không báo xong 2 lần.

---

## 16. Email khôi phục và OTP quên mật khẩu (L1090–1237)

### 16.1 Hằng số
```python
OTP_VALID_MINUTES = 10              # mã sống 10 phút
OTP_RESEND_COOLDOWN_SECONDS = 60    # 60 giây mới được xin mã mới
OTP_MAX_ATTEMPTS = 5                # sai 5 lần thì hủy mã
```
- `1_000_000` (dòng 1151) → Python cho phép `_` ngăn cách chữ số cho dễ đọc, giá trị = 1000000.

### 16.2 `generate_password_reset_otp()` — tạo mã
```python
    import secrets
    email = get_recovery_email()
    if not email:
        return None, None, "no_recovery_email"

    now = datetime.datetime.now()
    conn = get_connection()
    try:
        row = conn.execute('SELECT reset_otp_code, reset_otp_expires_at FROM settings WHERE id = 1').fetchone()
        if row and row['reset_otp_code'] and row['reset_otp_expires_at']:
            try:
                expires_at = datetime.datetime.strptime(row['reset_otp_expires_at'], "%Y-%m-%d %H:%M:%S")
                issued_at = expires_at - datetime.timedelta(minutes=OTP_VALID_MINUTES)
                if (now - issued_at).total_seconds() < OTP_RESEND_COOLDOWN_SECONDS:
                    return None, email, "otp_cooldown"
            except (TypeError, ValueError):
                pass

        otp_code = f"{secrets.randbelow(1_000_000):06d}"
        expires_at = (now + datetime.timedelta(minutes=OTP_VALID_MINUTES)).strftime("%Y-%m-%d %H:%M:%S")
        conn.execute('''UPDATE settings SET reset_otp_code = ?, reset_otp_expires_at = ?,
                        reset_otp_attempts = 0 WHERE id = 1''', (otp_code, expires_at))
        conn.commit()
        return otp_code, email, "ok"
    finally:
        conn.close()
```
| Cú pháp | Ý nghĩa |
|---|---|
| `secrets.randbelow(1_000_000)` | Số ngẫu nhiên **an toàn mật mã** từ 0 đến 999999. Không dùng `random` vì `random` **đoán trước được** nếu biết trạng thái |
| `f"{n:06d}"` | In số nguyên đủ 6 chữ số, thiếu thì thêm 0 phía trước: `42` → `"000042"` |
| `datetime.datetime.now()` | Giờ hiện tại của máy |
| `datetime.timedelta(minutes=10)` | Một **khoảng thời gian** 10 phút để cộng/trừ |
| `strptime(chuỗi, mẫu)` | **Chuỗi → thời gian** (*parse*) |
| `strftime(mẫu)` | **Thời gian → chuỗi** (*format*) |
| `(a - b).total_seconds()` | Hiệu hai mốc thời gian, đổi ra giây |
| `pass` | Câu lệnh "không làm gì" (thời gian hỏng thì bỏ qua kiểm tra chờ) |
| `return a, b, c` | Trả tuple 3 phần tử: (mã, email, lý do) |

**Logic chống spam:** CSDL không lưu "lúc tạo mã", nhưng **suy ra được**: lúc tạo = lúc hết hạn − 10 phút. Chưa đủ 60 giây từ lúc tạo → từ chối (không gửi email liên tục, không ghi đè mã chủ quán vừa nhận).

### 16.3 `verify_and_reset_password()` — kiểm tra mã và đặt mật khẩu mới
```python
    import hmac
    if not new_password or len(new_password) < 6:
        return False, "weak_password"
    otp_code = str(otp_code or "").strip()

    conn = get_connection()
    try:
        conn.execute("BEGIN IMMEDIATE")
        row = conn.execute('SELECT reset_otp_code, reset_otp_expires_at, reset_otp_attempts '
                           'FROM settings WHERE id = 1').fetchone()
        if not row or not row['reset_otp_code']:
            conn.rollback();  return False, "no_otp_requested"
        try:
            expires_at = datetime.datetime.strptime(row['reset_otp_expires_at'], "%Y-%m-%d %H:%M:%S")
        except (TypeError, ValueError):
            _clear_otp(conn); conn.commit();  return False, "invalid_otp"
        if datetime.datetime.now() > expires_at:
            _clear_otp(conn); conn.commit();  return False, "expired_otp"
        attempts = int(row['reset_otp_attempts'] or 0)
        if attempts >= OTP_MAX_ATTEMPTS:
            _clear_otp(conn); conn.commit();  return False, "too_many_attempts"

        if not hmac.compare_digest(otp_code.encode("utf-8"),
                                   str(row['reset_otp_code']).encode("utf-8")):
            attempts += 1
            if attempts >= OTP_MAX_ATTEMPTS:
                _clear_otp(conn); conn.commit();  return False, "too_many_attempts"
            conn.execute('UPDATE settings SET reset_otp_attempts = ? WHERE id = 1', (attempts,))
            conn.commit()
            return False, "invalid_otp"

        from werkzeug.security import generate_password_hash
        new_hash = generate_password_hash(new_password)
        conn.execute('''UPDATE settings SET admin_password_hash = ?, reset_otp_code = '',
                        reset_otp_expires_at = NULL, reset_otp_attempts = 0 WHERE id = 1''', (new_hash,))
        conn.commit()
        return True, "ok"
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()
```
*(Rút gọn `;` — mã gốc mỗi lệnh một dòng.)*

**Thứ tự kiểm tra** (dừng ở bước đầu tiên không đạt):
```
mật khẩu mới ≥ 6 ký tự? → có yêu cầu mã chưa? → thời hạn đọc được? → còn hạn?
→ chưa quá 5 lần sai? → mã đúng? → đặt mật khẩu mới + xóa mã
```

Các điểm then chốt:
- **`BEGIN IMMEDIATE`**: kẻ xấu gửi 100 yêu cầu đoán mã **song song** → các yêu cầu **xếp hàng**, mỗi yêu cầu đọc số lần sai **mới nhất** → không vượt được giới hạn 5 lần bằng cách gửi đồng thời.
- **Sai mã vẫn `commit()`** (ghi tăng số lần sai) → nếu rollback thì bộ đếm không bao giờ tăng.
- **`hmac.compare_digest(a, b)`**: so sánh **thời gian không đổi** (*constant-time*). So sánh `==` thường dừng ngay ở ký tự sai đầu tiên → mã đúng nhiều chữ đầu thì so lâu hơn một chút → kẻ tấn công đo thời gian có thể đoán dần từng chữ số (*timing attack*). `compare_digest` luôn so hết.
- `.encode("utf-8")` → chuỗi → `bytes` (kiểu `compare_digest` yêu cầu với dữ liệu không thuần ASCII).
- Đổi mật khẩu thành công → **xóa mã ngay** → mã chỉ dùng được **một lần**.
- `_clear_otp(conn)` → **hàm phụ dùng lại** cho 4 chỗ cần xóa mã — tránh lặp code.
- `raise` cuối → lỗi bất ngờ (CSDL hỏng) ném cho API xử lý, nhưng vẫn rollback + đóng kết nối.

**Kết hợp với giới hạn đăng nhập ở API** (5 lần sai / 5 phút theo IP) → hai lớp chống dò.

---

## 17. Chạy thử khi gọi trực tiếp (L1239–1249)

```python
if __name__ == '__main__':
    print("--- TEST ON WINDOWS ---")
    init_database()
    ...
    print(get_stats_data())
```
- `__name__` là biến Python tự đặt: bằng `'__main__'` khi **chạy trực tiếp** `python database.py`; bằng `'database'` (hoặc `'backend.database'`) khi **được import**.
- → Khối này **chỉ chạy khi gõ lệnh thử file**, không chạy khi kiosk/API import. Dòng chữ "TEST ON WINDOWS" là di tích từ lúc phát triển trên Windows.

---

## 18. Tổng hợp

### 18.1 Ba kiểu quản lý kết nối trong file
| Kiểu | Dùng ở | Đặc điểm |
|---|---|---|
| `conn = None` + `try/except/finally` | Vòng đời đơn, bếp | Bắt lỗi và **trả mã lý do** (`"database_error"`), không ném lên — kiosk không được sập giữa lúc khách thanh toán |
| `with _open_db()` | Quản trị, thống kê, kho | Lỗi → rollback + **ném lên** → API đổi thành mã HTTP (409, 500) |
| `get_connection()` + `try/finally` | `init_database` (không finally), OTP | OTP cần điều khiển commit từng nhánh |

### 18.2 Hai kỹ thuật chống chạy đua (race condition)
| Kỹ thuật | Khi nào dùng | Ví dụ |
|---|---|---|
| **UPDATE có điều kiện + `rowcount`** | Thao tác **một câu lệnh** | `close_pending_order`, `admin_mark_order_done` |
| **`BEGIN IMMEDIATE`** trước khi đọc | **Đọc → quyết định → ghi nhiều câu** | `save_new_order`, `mark_order_late_paid`, `admin_delete_ingredient`, `verify_and_reset_password` |

### 18.3 Bảng mã kết quả
| Hàm | Kết quả có thể |
|---|---|
| `create_pending_order` | `ok`, `duplicate_order`, `invalid_order_code`, `invalid_amount`, `invalid_items`, `empty_order`, `database_error` |
| `save_new_order` | như trên + `order_closed` (kèm `low_stock`, `insufficient_stock`) |
| `close_pending_order` | `True` / `False` |
| `mark_order_late_paid` | `late_paid`, `already_recorded`, `not_closed`, `unknown_order`, `database_error` |
| `admin_delete_ingredient` | `ok`, `not_found`, `in_use` |
| `generate_password_reset_otp` | `ok`, `no_recovery_email`, `otp_cooldown` |
| `verify_and_reset_password` | `ok`, `weak_password`, `no_otp_requested`, `invalid_otp`, `expired_otp`, `too_many_attempts` |

### 18.4 Các nguyên tắc xuyên suốt
1. **Mọi dữ liệu từ ngoài đi qua `?`**; f-string chỉ ghép đoạn SQL trong whitelist.
2. **Tiền là số nguyên** (`INTEGER`), nguyên liệu là số thực (`REAL`).
3. **Tính trong SQL** (`quantity - ?`, `SUM`, `COUNT`) thay vì đọc lên Python rồi ghi xuống.
4. **Luôn đóng kết nối** (`finally` hoặc `_open_db`); **lỗi thì rollback**.
5. **Kiểm tra đầu vào trước khi ghi** (`_normalize_order_input`) vì SQLite không chặn sai kiểu và không kiểm tra bên trong JSON.
6. **Ngày giờ `YYYY-MM-DD HH:MM:SS` theo giờ máy** (`localtime`) → so sánh chuỗi = so sánh thời gian.

### 18.5 Những điểm nhỏ còn có thể cải thiện (không gây lỗi hiện tại)
| Điểm | Vị trí | Ghi chú |
|---|---|---|
| `import sys` không dùng | L3 | Xóa được |
| `created_at DEFAULT CURRENT_TIMESTAMP` là giờ UTC | L103 | Không dùng tới vì INSERT luôn ghi giờ địa phương |
| `order_id TEXT PRIMARY KEY` thiếu `NOT NULL` | L98 | Code đã chặn mã rỗng |
| `init_database` không đóng kết nối nếu `CREATE/ALTER` lỗi | L80–236 | Chỉ xảy ra lúc khởi động |
| Dùng `print` thay vì `logging` | toàn file | Trên Pi, `print` vẫn vào nhật ký `journalctl` của systemd |
| Mã OTP lưu **dạng thô** trong CSDL | L1153–1157 | Sống 10 phút, tối đa 5 lần thử; có thể băm như mật khẩu nếu muốn chặt hơn |
| Chưa có chỉ mục cho thống kê | — | 200.000 đơn vẫn chỉ ~9 ms |

---

## 19. Câu hỏi tự kiểm tra

1. Vì sao `DB_PATH` phải là đường dẫn tuyệt đối? Điều gì xảy ra nếu `sqlite3.connect` nhận đường dẫn sai?
2. Python tự mở giao dịch trước loại câu lệnh nào? Vì sao `save_new_order` phải tự gửi `BEGIN IMMEDIATE` trước câu SELECT?
3. `(default_hash,)` và `(default_hash)` khác nhau thế nào?
4. Trong `close_pending_order`, `rowcount == 0` có nghĩa là gì? Vì sao đó không phải lỗi?
5. `save_new_order` gặp đơn đã `CANCELLED` thì trả gì, và kiosk làm gì tiếp?
6. Vì sao trừ kho bằng `quantity = MAX(0, quantity - ?)` chứ không đọc lên Python trừ rồi ghi xuống?
7. `_open_db` có tự `commit` không? Quên `commit` thì sao?
8. Vì sao f-string trong `get_stats_data` không bị SQL injection?
9. `WHERE 1=1` dùng để làm gì?
10. Xóa món thì xóa luôn công thức, còn xóa nguyên liệu đang dùng thì từ chối — vì sao khác nhau?
11. Muối (salt) trong băm mật khẩu có tác dụng gì?
12. Vì sao so sánh OTP dùng `hmac.compare_digest` thay vì `==`? Vì sao sai mã vẫn phải `commit`?

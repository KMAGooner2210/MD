# Kiến thức và cú pháp căn bản để đọc các file SQLite

> **Dùng tài liệu này thế nào:** đọc **trước** các tài liệu chi tiết trong `hoc_tap/SQLite/`:
> `database.md`, `desktop_env.md`, `mqtt_security.md`, `seed_inventory.md`, `update_menu.md`, `bridge.md`, `api_server.md`.
> Ở đây chỉ gom **những gì tối thiểu** cần biết để đọc hiểu các file đó. Mỗi mục có ví dụ ngắn kèm kết quả.
> **Mọi kết quả in ra trong tài liệu đều đã chạy thật** (Python 3.10, SQLite qua module `sqlite3`) trên một CSDL tạm trong thư mục nháp, không đụng `kiosk.db`.

---

## 0. Bản đồ: kiến thức nào dùng ở file nào

| Kiến thức | Mục | `database.py` | `seed_inventory` | `update_menu` | `mqtt_security` | `desktop_env` | `bridge.py` / `api_server.py` |
|---|---|:-:|:-:|:-:|:-:|:-:|:-:|
| Import, biến môi trường, `os.path` | A1–A2 | ● | ● | ● | ● | ● | ● |
| Kiểu dữ liệu, f-string, comprehension | A3–A5 | ● | ● | ● | ● | ● | ● |
| `try / except / finally / raise` | A6 | ● | ● | ● | ● | | ● |
| `with`, `@contextmanager`, `yield` | A7 | ● (`_open_db`) | | ● | | | |
| Decorator, `@wraps` | A8 | ● | | | | | ● (`api_server`) |
| `json` | A9 | ● | | | ● | | ● |
| Bảng, cột, kiểu, ràng buộc | B1–B3 | ● | | | | | |
| `INSERT/SELECT/UPDATE/DELETE` | B4 | ● | ● | ● | | | |
| `JOIN`, `GROUP BY`, hàm gộp, `COALESCE` | B5–B6 | ● | | | | | |
| Ngày giờ trong SQL | B7 | ● | | | | | |
| `sqlite3`: connect, execute, `?`, fetch | C1–C4 | ● | ● | ● | | | ● |
| `commit / rollback`, `rowcount` | C5–C6 | ● | ● | ● | | | ● |
| Giao dịch, khóa, WAL, `BEGIN IMMEDIATE` | D1–D4 | ● | | ● | | | ● |
| HMAC | E1 | | | | ● | | ● |
| Sao lưu, di chuyển file | E2 | | | ● | | | |

---

# PHẦN A — Python căn bản

## A1. Import

```python
import os                                 # nạp cả module, dùng: os.path.join(...)
import sqlite3
from contextlib import contextmanager     # lấy riêng một tên trong module
from typing import Any
from __future__ import annotations        # (phải ở đầu file) cho phép viết gợi ý kiểu kiểu mới
import paho.mqtt.client as mqtt           # đặt bí danh
```

**Module hay gặp trong các file SQLite:**
| Module | Dùng để |
|---|---|
| `sqlite3` | Làm việc với CSDL SQLite (có sẵn trong Python) |
| `os`, `sys` | Đường dẫn, biến môi trường, `sys.path` |
| `json` | Chuyển dict/list ↔ chuỗi JSON |
| `datetime`, `time` | Ngày giờ |
| `hmac`, `hashlib` | Chữ ký HMAC‑SHA256 (`mqtt_security`) |
| `shutil` | Chép / di chuyển file (`update_menu`) |
| `contextlib` | Tạo trình quản lý ngữ cảnh (`_open_db`) |

**`sys.path`** — danh sách thư mục Python tìm module. Các script trong `servers/` hay `kiosk_app/` thêm thư mục gốc dự án vào đó để `from backend import database` chạy được:
```python
current_dir = os.path.dirname(os.path.abspath(__file__))   # thư mục chứa file này
project_root = os.path.dirname(current_dir)                # lùi lên một cấp
if project_root not in sys.path:
    sys.path.append(project_root)
```

## A2. Biến môi trường và đường dẫn

```python
DB_PATH = os.environ.get("KIOSK_DB_PATH", "/duong/dan/mac/dinh/kiosk.db")
```
- `os.environ` giống một dict chứa biến môi trường của tiến trình. `.get(tên, mặc_định)` → không có thì lấy mặc định.
- Giá trị **luôn là chuỗi**: cần số thì phải `int(...)`.
- Dự án dùng `KIOSK_DB_PATH` để **trỏ sang CSDL tạm** khi thử nghiệm — đó là cách mọi thí nghiệm trong loạt bài này không đụng `kiosk.db` thật.

| Hàm | Ví dụ | Kết quả |
|---|---|---|
| `os.path.join(a, b)` | `join("assets", "images")` | `assets/images` (tự thêm `/`) |
| `os.path.dirname(p)` | `dirname("/a/b/c.py")` | `/a/b` |
| `os.path.abspath(p)` | đường dẫn tương đối → tuyệt đối | |
| `os.path.exists(p)` | có tồn tại không | `True/False` |
| `os.makedirs(p, exist_ok=True)` | tạo thư mục, có rồi thì thôi | |

## A3. Kiểu dữ liệu và thao tác thường gặp

| Kiểu | Ví dụ | Ghi nhớ |
|---|---|---|
| `int`, `float` | `30000`, `0.5` | |
| `str` | `"Trà sữa"` | `.strip()` bỏ khoảng trắng hai đầu, `.upper()`, `.split(" ")` |
| `bool` | `True`, `False` | **Là lớp con của `int`**: `True == 1` → các file luôn chặn riêng `bool` khi cần số |
| `None` | | “Không có giá trị”; `fetchone()` không thấy dòng nào thì trả `None` |
| `list` | `[1, 2, 3]` | Thay đổi được |
| `tuple` | `(1, 2)` | Không thay đổi được; **tuple 1 phần tử phải có dấu phẩy**: `(1,)` |
| `dict` | `{"id": 1, "name": "Trà sữa"}` | `d["id"]` (thiếu → lỗi `KeyError`), `d.get("x")` (thiếu → `None`), `d.get("x", 0)` |

```python
>>> type((1)), type((1,))
(<class 'int'>, <class 'tuple'>)        # (1) chỉ là số 1 trong ngoặc!
>>> d = {"a": 1}; d.get("b"), d.get("b", 0)
(None, 0)
>>> repr(" dh0001 ".strip().upper())
"'DH0001'"
```
Dấu phẩy trong `(1,)` cực kỳ quan trọng với SQLite: `conn.execute("... WHERE id = ?", (order_id,))`.

**Đúng / sai theo kiểu “truthy”:** `0`, `""`, `None`, `[]`, `{}` đều được coi là **sai** trong `if`. Vì vậy `if not MQTT_SHARED_SECRET:` bắt được cả chuỗi rỗng lẫn `None`.

## A4. Hàm và tham số

```python
def admin_get_orders(status_filter="all", keyword="", limit=10, offset=0):
    ...
admin_get_orders("paid", limit=20)          # vị trí + theo tên

def sign_message(body, secret, *, now=None):   # sau dấu * : bắt buộc gọi theo tên
    ...
sign_message(data, key, now=123)

def ham(*args, **kwargs):                   # nhận mọi tham số: args là tuple, kwargs là dict
    ...
```
- **Gợi ý kiểu** (*type hint*): `def f(x: int, s: str | None = None) -> dict[str, Any]:` — chỉ để đọc và cho công cụ kiểm tra; Python **không ép**.
- `return a, b` thực chất trả **một tuple** `(a, b)`; người gọi tách: `total, rows = f()` (gặp ở `admin_get_orders`).

## A5. f-string và comprehension

```python
>>> f"{30000:,}".replace(",", ".")      # dấu phân cách nghìn kiểu Việt Nam
'30.000'
>>> f"{0.5:.1%}", f"{'x':>5}|"
('50.0%', '    x|')
>>> {k: v * 2 for k, v in {"a": 1}.items()}      # dict comprehension
{'a': 2}
>>> [x * x for x in range(4) if x % 2]           # list comprehension có điều kiện
[1, 9]
```
Gặp nhiều trong code: `[dict(row) for row in rows]` — đổi mọi dòng kết quả SQL thành dict.

## A6. Bắt lỗi: `try / except / else / finally / raise`

```python
conn = None
try:
    conn = get_connection()
    conn.execute("BEGIN IMMEDIATE")
    ...
    conn.commit()
    return True
except Exception as error:          # bắt mọi lỗi thông thường
    if conn is not None:
        conn.rollback()             # hủy mọi thay đổi dở dang
    print(f"--> Lỗi: {error}")
    return False
finally:                            # LUÔN chạy: dù thành công, lỗi, hay đã return
    if conn is not None:
        conn.close()
```
Đây là **mẫu chuẩn** của `database.py` (vd. `admin_mark_order_done`, `create_pending_order`). Ghi nhớ:
- `except LoaiLoi as e` — chỉ bắt đúng loại đó (và lớp con của nó). `sqlite3.IntegrityError` (vi phạm ràng buộc), `sqlite3.OperationalError` (khóa, sai SQL)…
- `raise` (trơn) trong `except` — **ném lại** lỗi vừa bắt sau khi đã dọn dẹp.
- `raise LoiMoi(...) from error` — ném lỗi mới nhưng **giữ nguyên nhân gốc** (gặp trong `mqtt_security`).
- `conn = None` trước `try` để `finally` không lỗi khi chính `get_connection()` thất bại.

## A7. `with`, trình quản lý ngữ cảnh, `@contextmanager` + `yield`

`with X as y:` = “vào thì làm gì, ra thì (luôn) làm gì” — kể cả khi có lỗi.
```python
with open("a.txt") as f:      # ra khỏi khối → file tự đóng
    data = f.read()
```
`database.py` tự tạo một trình quản lý ngữ cảnh `_open_db()`:
```python
from contextlib import contextmanager

@contextmanager
def _open_db():
    conn = get_connection()
    try:
        yield conn            # ← khối 'with' chạy ở đây, nhận conn
    except Exception:
        conn.rollback()       # khối 'with' ném lỗi → rollback
        raise                 # rồi ném tiếp cho người gọi
    finally:
        conn.close()          # luôn đóng

with _open_db() as conn:      # dùng ở 22 chỗ trong database.py
    conn.execute(...)
    conn.commit()
```
Chạy thử bản mô phỏng (in ra từng bước):
```
mo ket noi → dung conn → dong ket noi                       (khối with chạy bình thường)
mo ket noi → rollback → dong ket noi → lỗi vẫn được ném ra ngoài   (khối with có 1/0)
```
- `yield` biến hàm thành **generator**: chạy tới `yield` thì **tạm dừng**, trao giá trị cho khối `with`; khối xong thì chạy tiếp phần sau `yield`.
- ⚠ `with sqlite3.connect(...) as conn:` của **chính sqlite3** chỉ commit/rollback, **không đóng** kết nối — vì thế dự án tự viết `_open_db()`.

## A8. Decorator và `@wraps`

Decorator = hàm **nhận một hàm, trả về hàm mới bọc quanh nó**. `@ten` đặt trên `def` là viết tắt của `f = ten(f)`.
```python
from functools import wraps

def log(f):
    @wraps(f)                       # giữ tên, docstring của f
    def w(*a, **k):
        print("goi", f.__name__, a, k)
        return f(*a, **k)
    return w

@log
def cong(a, b=0): return a + b

>>> cong(1, b=2)
goi cong (1,) {'b': 2}
3
>>> cong.__name__
'cong'                              # không có @wraps sẽ là 'w'
```
Gặp ở: `@contextmanager`, `@app.route(...)` và `@require_api_key` (`api_server.py`), `@Slot` (`bridge.py`), `@dataclass`, `@staticmethod`.

## A9. JSON

```python
>>> json.dumps([{"name": "Trà sữa", "qty": 2}], ensure_ascii=False)
'[{"name": "Trà sữa", "qty": 2}]'
>>> json.dumps({"a": "à"})                    # mặc định ensure_ascii=True
'{"a": "\\u00e0"}'
json.loads('{"a": 1}')  # → {'a': 1}
```
- `dumps` = Python → chuỗi; `loads` = chuỗi → Python.
- `ensure_ascii=False` giữ nguyên tiếng Việt (dễ đọc khi lưu `items_json` trong CSDL).
- `separators=(",", ":")` bỏ khoảng trắng → chuỗi gọn, ổn định (quan trọng khi **ký HMAC** trên chuỗi).

---

# PHẦN B — CSDL quan hệ và SQL

## B1. Khái niệm
- **CSDL quan hệ** = tập các **bảng**. Bảng có **cột** (thuộc tính, cố định) và **hàng** (bản ghi).
- Trong dự án: `product` (món), `ingredient` (nguyên liệu), `recipe` (món cần bao nhiêu nguyên liệu), `orders` (đơn), `settings`, …
- **SQLite** = CSDL nằm gọn trong **một file** (`kiosk.db`), không cần máy chủ riêng; chương trình mở file qua thư viện `sqlite3`.
- Câu lệnh SQL không phân biệt hoa thường; quy ước viết **từ khóa IN HOA**, tên bảng/cột thường.

## B2. Kiểu dữ liệu trong SQLite

| Lớp lưu trữ | Ví dụ | `typeof(...)` |
|---|---|---|
| `INTEGER` | `1`, `30000` | `integer` |
| `REAL` | `1.5` | `real` |
| `TEXT` | `'Trà sữa'` | `text` |
| `BLOB` | dữ liệu nhị phân | `blob` |
| `NULL` | không có giá trị | `null` |

- SQLite **không có kiểu ngày giờ** riêng: dự án lưu thời gian dạng `TEXT` `'2026-11-25 16:13:25'`.
- **Không có kiểu boolean**: dùng `INTEGER` 0/1 (vd. `status`).
- **Kiểu “mềm”** (*type affinity*): cột khai `INTEGER` vẫn nhận được chuỗi không phải số — đã thử ở `database.md`. Vì vậy phải kiểm tra dữ liệu **ở Python** trước khi ghi.

## B3. Tạo bảng và ràng buộc

```sql
CREATE TABLE IF NOT EXISTS product (
    id     INTEGER PRIMARY KEY AUTOINCREMENT,
    name   TEXT    NOT NULL,
    price  INTEGER NOT NULL CHECK (price >= 0),
    status INTEGER DEFAULT 1
);
CREATE TABLE recipe (
    product_id INTEGER REFERENCES product(id),
    ingredient TEXT,
    qty        REAL,
    PRIMARY KEY (product_id, ingredient)        -- khóa ghép
);
```
| Ràng buộc | Ý nghĩa |
|---|---|
| `PRIMARY KEY` | Khóa chính: **mỗi hàng một giá trị duy nhất**, dùng để tìm đúng một hàng |
| `INTEGER PRIMARY KEY AUTOINCREMENT` | Tự đánh số 1, 2, 3… không bao giờ dùng lại số đã xóa (nhớ trong bảng `sqlite_sequence`) |
| `PRIMARY KEY (a, b)` | **Khóa ghép**: cặp (a, b) là duy nhất |
| `NOT NULL` | Bắt buộc có giá trị |
| `DEFAULT v` | Không ghi thì lấy `v` |
| `CHECK (điều kiện)` | Ghi vi phạm → lỗi |
| `REFERENCES bảng(cột)` | **Khóa ngoại**: giá trị phải tồn tại ở bảng cha. **Chỉ có hiệu lực khi bật `PRAGMA foreign_keys=ON`** trên từng kết nối |
| `IF NOT EXISTS` | Có bảng rồi thì bỏ qua → `init_database()` chạy lại nhiều lần vẫn an toàn |

Chạy thử:
```
INSERT price = -1                 → CHECK constraint failed: price>=0
INSERT recipe product_id = 99     → FOREIGN KEY constraint failed   (không có món 99)
DELETE món 1 khi recipe còn trỏ tới → FOREIGN KEY constraint failed   (không xóa được bảng cha)
```

**Sửa bảng đã có:** `ALTER TABLE orders ADD COLUMN paid_amount INTEGER` — SQLite **không có** `ADD COLUMN IF NOT EXISTS`, nên `database.py` thử chạy và chỉ bỏ qua lỗi “duplicate column” (`_add_column_if_missing`). Đây là cách **nâng cấp CSDL cũ** (*migration*) mà không mất dữ liệu.

**Chỉ mục** (*index*): `CREATE INDEX idx ON orders(created_at)` — như mục lục sách, tìm nhanh hơn (đã đo ở `slide_sqlite_17.md`: quét 8,8 ms → tra chỉ mục 0,11 ms trên 200 nghìn đơn).

## B4. Bốn câu lệnh chính (CRUD)

```sql
INSERT INTO product (name, price) VALUES ('Trà sữa', 30000);          -- Thêm
SELECT id, name FROM product WHERE price > 20000 ORDER BY price DESC LIMIT 2 OFFSET 0;  -- Đọc
UPDATE product SET price = 32000 WHERE id = 1;                         -- Sửa
DELETE FROM product WHERE id = 1;                                      -- Xóa
```
| Mệnh đề | Ý nghĩa |
|---|---|
| `SELECT *` | Mọi cột |
| `WHERE` | Lọc hàng. ⚠ `UPDATE/DELETE` **thiếu `WHERE` là sửa/xóa cả bảng** |
| `ORDER BY cột ASC/DESC` | Sắp xếp tăng/giảm |
| `LIMIT n OFFSET k` | Lấy n hàng, bỏ k hàng đầu → **phân trang** (`api_server` `/api/orders`: `offset = (page-1)*limit`) |
| `LIKE '%cà%'` | Chứa chuỗi; `%` = chuỗi bất kỳ. Không phân biệt hoa/thường **chỉ với chữ ASCII**: `'Cafe' LIKE 'cafe'` → 1, nhưng `'CÀ' LIKE 'cà'` → 0 |
| `IN (1, 3)` | Thuộc danh sách |
| `IS NULL` | So với NULL phải dùng `IS`, không dùng `=` |

**Mẫu “cập nhật có điều kiện”** — cốt lõi của vòng đời đơn:
```sql
UPDATE orders SET kitchen_status = 'done'
WHERE order_id = ? AND payment_status = 'PAID' AND kitchen_status = 'pending';
```
Điều kiện trạng thái nằm **ngay trong `WHERE`**: nếu đơn đã chuyển trạng thái rồi thì câu lệnh sửa **0 hàng** → Python biết qua `rowcount` (C6). Không cần “đọc rồi kiểm tra rồi ghi” (dễ tranh chấp).

## B5. Hàm gộp, `GROUP BY`, `COALESCE`

```sql
SELECT product_id, COUNT(*), SUM(qty) FROM recipe GROUP BY product_id;
-- [(1, 2, 150.0), (2, 1, 20.0)]
```
| Hàm | Ý nghĩa |
|---|---|
| `COUNT(*)` | Đếm hàng |
| `SUM(x)`, `AVG(x)`, `MIN`, `MAX` | Tổng, trung bình… |
| `GROUP BY cột` | Gộp các hàng cùng giá trị rồi tính hàm gộp cho từng nhóm |
| `COALESCE(a, b, …)` | Giá trị **đầu tiên khác NULL** |

Bẫy hay gặp: **`SUM` trên tập rỗng trả `NULL`, không phải 0**:
```
SELECT SUM(price) FROM product WHERE id > 100              → None
SELECT COALESCE(SUM(price), 0) FROM product WHERE id > 100 → 0
```
`database.py` dùng `COALESCE` cho doanh thu và cho `COALESCE(closed_at, created_at)` (đơn cũ chưa có `closed_at` thì lấy `created_at`).

## B6. `JOIN` — ghép hai bảng

```sql
SELECT p.name, r.ingredient, r.qty
FROM recipe r JOIN product p ON p.id = r.product_id;
-- [('Trà sữa','sua',100.0), ('Trà sữa','tran chau',50.0), ('Cà phê','ca phe',20.0)]
```
- `recipe r` — đặt **bí danh** `r` cho gọn.
- `JOIN … ON điều_kiện` (= `INNER JOIN`) — chỉ giữ cặp hàng khớp. `LEFT JOIN` giữ cả hàng bên trái không khớp (cột bên phải thành NULL).
- Dùng ở `get_recipe` (tên nguyên liệu + định lượng) và khi trừ kho theo công thức.

## B7. Ngày giờ trong SQLite

```sql
SELECT datetime('now'), datetime('now','localtime'), date('now','-7 days');
-- ('2026-11-25 16:37:15', '2026-11-25 23:37:15', '2026-11-18')
```
- `datetime('now')` là giờ **UTC** (Việt Nam = UTC+7). `'localtime'` đổi sang giờ máy.
- Chuỗi dạng `YYYY-MM-DD HH:MM:SS` **so sánh/sắp xếp được như chuỗi** → `WHERE created_at >= '2026-11-18'` chạy đúng.
- `strftime('%H', created_at)` lấy giờ → dùng để vẽ biểu đồ theo giờ.
- Lưu ý đã gặp: `created_at` trong CSDL có chỗ là UTC (thí nghiệm thấy `16:13` khi giờ máy là `23:13`).

---

# PHẦN C — Module `sqlite3` của Python

## C1. Mở kết nối — `get_connection()` của dự án

```python
def get_connection():
    conn = sqlite3.connect(DB_PATH, timeout=10)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA journal_mode=WAL")
    conn.execute("PRAGMA busy_timeout=10000")
    conn.execute("PRAGMA foreign_keys=ON")
    return conn
```
| Dòng | Ý nghĩa |
|---|---|
| `sqlite3.connect(path, timeout=10)` | Mở (hoặc tạo) file CSDL; gặp khóa thì chờ tối đa 10 s |
| `row_factory = sqlite3.Row` | Mỗi hàng đọc được theo **tên cột** (C3) |
| `PRAGMA journal_mode=WAL` | Chế độ ghi nhật ký WAL (D3) — trả về `wal` |
| `PRAGMA busy_timeout=10000` | Chờ khóa 10 000 ms trước khi báo “database is locked” |
| `PRAGMA foreign_keys=ON` | Bật kiểm tra khóa ngoại — **phải bật trên mỗi kết nối** |

`PRAGMA` = lệnh cấu hình riêng của SQLite.

## C2. `execute` và tham số `?` — chống SQL injection

```python
conn.execute("SELECT * FROM product WHERE id = ?", (1,))
conn.execute("INSERT INTO product(name, price) VALUES (?, ?)", ("Trà sữa", 30000))
```
- `?` là **chỗ trống**; giá trị đưa vào bằng **tuple** (nhớ dấu phẩy khi 1 phần tử).
- **Không bao giờ** ghép chuỗi người dùng vào SQL. Thử với đầu vào độc `x' OR '1'='1`:
```
f"... WHERE name = '{bad}'"     → COUNT = 3   (điều kiện thành luôn đúng → lộ mọi hàng)
"... WHERE name = ?", (bad,)    → COUNT = 0   (được coi là một chuỗi bình thường)
```
- Tên bảng/cột **không** thay bằng `?` được; chỉ giá trị.

## C3. Đọc kết quả: `fetchone`, `fetchall`, `sqlite3.Row`

```python
r = conn.execute("SELECT * FROM product WHERE id = ?", (1,)).fetchone()
r["name"], r[2], dict(r), r.keys()
# ('Trà sữa', 30000, {'id': 1, 'name': 'Trà sữa', 'price': 30000, 'status': 1}, ['id','name','price','status'])

conn.execute("SELECT * FROM product WHERE id = 99").fetchone()     # → None
conn.execute("SELECT id, name FROM product ORDER BY price DESC LIMIT 2").fetchall()
# [(3, 'Nước cam'), (1, 'Trà sữa')]
```
- `fetchone()` → một hàng hoặc **`None`** → luôn kiểm `if row is None` / `if row and ...`.
- `fetchall()` → **list** các hàng (rỗng `[]` nếu không có).
- `sqlite3.Row` đọc được cả `r["tên"]` và `r[chỉ_số]`; muốn trả qua JSON thì `dict(r)`.
- Lặp trực tiếp cũng được: `for row in conn.execute(...):`.

## C4. Nhiều câu lệnh một lúc

```python
conn.executemany("INSERT INTO product(name, price) VALUES (?, ?)",
                 [("Cà phê", 25000), ("Nước cam", 35000)])   # cùng câu lệnh, nhiều bộ giá trị
conn.executescript("CREATE TABLE ...; CREATE TABLE ...;")    # nhiều câu lệnh khác nhau (không có tham số)
```

## C5. `commit`, `rollback`, `close` — và giao dịch ngầm

```python
cur = conn.execute("INSERT INTO product(name, price) VALUES (?, ?)", ("Trà sữa", 30000))
conn.in_transaction     # True   ← sqlite3 TỰ mở giao dịch trước INSERT/UPDATE/DELETE
conn.commit()
conn.in_transaction     # False
```
- **Giao dịch** = nhóm thay đổi “được ăn cả, ngã về không”.
- Module `sqlite3` **tự `BEGIN`** trước câu lệnh ghi (không tự BEGIN trước `SELECT`). Thay đổi **chưa ai thấy** cho tới `commit()`.
- `rollback()` hủy mọi thay đổi từ lúc `BEGIN`.
- **Quên `commit()` rồi `close()` → mất thay đổi.** Quên `close()` → giữ khóa, chặn tiến trình khác.

## C6. Thông tin sau khi chạy: `rowcount`, `lastrowid`

```
INSERT ... → lastrowid = 1, rowcount = 1
UPDATE product SET price=? WHERE id=99 → rowcount = 0     (không có hàng nào khớp)
SELECT ... → rowcount = -1                                (không dùng cho SELECT)
```
- `cursor.lastrowid` — `id` vừa được tự đánh số.
- `cursor.rowcount` — số hàng **bị sửa/xóa/thêm**. Dự án dùng **11 lần** trong `database.py`, ví dụ `if cursor.rowcount != 1: conn.rollback(); return False` → phát hiện “đơn không tồn tại / đã đổi trạng thái” (đi cùng mẫu B4).

---

# PHẦN D — Giao dịch và nhiều tiến trình cùng dùng CSDL

## D1. Vì sao cần quan tâm
Cùng lúc có **nhiều tiến trình** mở `kiosk.db`: ứng dụng kiosk (`bridge.py`), API (`api_server.py`, nhiều luồng), công cụ `update_menu`… SQLite cho **nhiều người đọc** nhưng **một người ghi** tại một thời điểm.

## D2. `BEGIN IMMEDIATE` — giành quyền ghi ngay từ đầu

```python
conn.execute("BEGIN IMMEDIATE")      # xin khóa ghi NGAY, trước khi đọc
row = conn.execute("SELECT quantity FROM ingredient WHERE id=?", (1,)).fetchone()
... tính toán ...
conn.execute("UPDATE ingredient SET quantity=? WHERE id=?", (moi, 1))
conn.commit()
```
- `BEGIN` thường chỉ khóa khi **bắt đầu ghi** → hai tiến trình có thể cùng **đọc** số cũ rồi cùng ghi → **mất cập nhật** (trừ kho 2 lần thành 1).
- `BEGIN IMMEDIATE` giữ quyền ghi **từ trước khi đọc** → “đọc‑tính‑ghi” trở thành một khối không ai chen vào. `database.py` dùng 8 lần (chốt đơn, trừ kho, hoàn thành đơn…).

Thử: kết nối 1 đang giữ `BEGIN IMMEDIATE`, kết nối 2 (`timeout=0.2`):
```
kết nối 2 ghi  → database is locked
kết nối 2 đọc  → vẫn được (3 hàng)
```

## D3. WAL — đọc và ghi không chặn nhau
- Mặc định SQLite ghi thẳng vào file chính → người đọc phải chờ người ghi.
- **WAL** (*Write‑Ahead Log*): thay đổi ghi vào file phụ `kiosk.db-wal`, định kỳ mới gộp vào file chính → **người đọc vẫn đọc bản cũ ổn định trong khi có người ghi** (đúng như thí nghiệm D2: đọc vẫn được).
- Hệ quả: cạnh `kiosk.db` có thêm `kiosk.db-wal` và `kiosk.db-shm`. **Không chép riêng `kiosk.db`** để sao lưu khi đang chạy → dùng `conn.backup()` (E2).

## D4. `busy_timeout` / `timeout`
Gặp khóa thì **chờ** tối đa N ms rồi mới báo lỗi `database is locked`. Dự án đặt 10 s → các ghi ngắn nối tiếp nhau tự xếp hàng thay vì lỗi ngay.

**Tóm tắt mẫu ghi an toàn của dự án:**
```
get_connection()  → BEGIN IMMEDIATE → UPDATE … WHERE <điều kiện trạng thái>
→ kiểm rowcount → commit()   |  lỗi → rollback()  |  finally → close()
```

---

# PHẦN E — Kiến thức phụ trợ

## E1. HMAC (cho `mqtt_security.py`)
- **Hàm băm** SHA‑256: biến dữ liệu bất kỳ thành 64 ký tự hex; sửa 1 bit → kết quả khác hẳn; không đảo ngược được.
- **HMAC** = băm **kèm khóa bí mật**: `hmac.new(khoa_bytes, du_lieu_bytes, hashlib.sha256).hexdigest()`. Ai không có khóa thì không tạo được chữ ký đúng → chống **giả mạo** và **sửa** tin.
- So chữ ký bằng `hmac.compare_digest(a, b)` (thời gian so sánh không phụ thuộc vị trí khác nhau → chống đoán dần).
- `str.encode("utf-8")` → `bytes`; `bytes.decode("utf-8")` → `str`. HMAC làm việc trên **bytes**.

## E2. Sao lưu và di chuyển file (cho `update_menu.py`)
```python
src = sqlite3.connect(DB_PATH); dst = sqlite3.connect(backup_path)
src.backup(dst)                         # sao lưu nhất quán, an toàn với WAL
shutil.move(nguon, dich)                # di chuyển file (ảnh cũ sang thư mục sao lưu)
```
(`update_menu.py` L160: `conn.backup(dst)`; L168: `shutil.move(...)`.)

## E3. Một chút Flask và Qt (đủ để đọc `api_server.md`, `bridge.md`)
- Flask: `@app.route('/api/menu', methods=['GET'])` gắn URL vào hàm; hàm trả `jsonify(...)` (+ mã trạng thái). Chi tiết ở chủ đề HTTP/Flask.
- Qt: `Signal` / `@Slot` — cách Python và giao diện QML gọi nhau. Chi tiết ở chủ đề Qt/QML.
- `threading.Lock()` + `with lock:` — chỉ một luồng vào đoạn mã tại một thời điểm.

## E4. `if __name__ == "__main__":`
Khối này chỉ chạy khi file được **chạy trực tiếp** (`python3 database.py`), **không** chạy khi file bị `import`. `database.py` dùng nó để khởi tạo CSDL khi chạy tay.

---

# PHẦN F — Bảng tra nhanh và tự kiểm tra

## F1. Bảng tra “thấy cái này trong code → nghĩa là gì”

| Thấy | Nghĩa |
|---|---|
| `(x,)` | Tuple một phần tử (tham số cho `?`) |
| `with _open_db() as conn:` | Mở kết nối; lỗi thì rollback; luôn đóng |
| `conn.execute("BEGIN IMMEDIATE")` | Giành quyền ghi trước khi đọc |
| `cursor.rowcount != 1` | Không sửa đúng 1 hàng → không tồn tại / sai trạng thái |
| `fetchone()` rồi `if row is None` | Không tìm thấy |
| `dict(row)` | Hàng `sqlite3.Row` → dict để trả JSON |
| `COALESCE(SUM(x), 0)` | Tổng, tập rỗng thì 0 |
| `json.dumps(..., ensure_ascii=False)` | Lưu JSON giữ tiếng Việt |
| `PRAGMA foreign_keys=ON` | Bật khóa ngoại cho kết nối này |
| `except sqlite3.IntegrityError` | Vi phạm ràng buộc (trùng khóa, khóa ngoại, CHECK) |
| `database is locked` | Có người đang giữ khóa ghi quá thời gian chờ |
| `os.environ.get("KIOSK_DB_PATH", ...)` | Cho phép trỏ sang CSDL khác (thử nghiệm) |

## F2. Thứ tự đọc đề xuất
1. Tài liệu này →
2. `desktop_env.md` (ngắn, chỉ Python) →
3. `database.md` (trọng tâm) →
4. `seed_inventory.md`, `update_menu.md` (dùng lại `database.py`) →
5. `mqtt_security.md` →
6. `bridge.md`, `api_server.md` (nơi gọi `database.py`).

## F3. Câu hỏi tự kiểm tra
1. `conn.execute("... WHERE id = ?", (5))` sai ở đâu? Sửa thế nào?
2. Vì sao phải dùng `?` thay vì f-string để đưa giá trị vào SQL? Cho ví dụ đầu vào phá được câu lệnh.
3. `fetchone()` trả gì khi không có hàng nào? Viết đoạn kiểm tra an toàn.
4. Vừa `INSERT` xong nhưng tiến trình khác không thấy dữ liệu — quên gì?
5. `SELECT SUM(total_amount) FROM orders WHERE ...` trả `None` khi nào? Viết lại để luôn ra số.
6. Khóa ngoại khai trong `CREATE TABLE` nhưng vẫn chèn được giá trị không tồn tại — vì sao?
7. Giải thích mẫu `UPDATE … WHERE order_id=? AND kitchen_status='pending'` + `rowcount`. Nó thay thế cho cách làm nào, và tránh được lỗi gì?
8. `BEGIN` thường và `BEGIN IMMEDIATE` khác nhau ở thời điểm nào giành khóa? Kịch bản trừ kho nào cần `IMMEDIATE`?
9. Với WAL, khi một tiến trình đang ghi thì tiến trình khác **đọc** có bị chặn không? **Ghi** thì sao?
10. Trong `_open_db()`, nếu khối `with` ném lỗi thì thứ tự các bước xảy ra thế nào? Lỗi có bị “nuốt” không?
11. `True` có qua được `isinstance(x, int)` không? Vì sao các hàm kiểm tra số phải chặn `bool` riêng?
12. `datetime('now')` trả giờ gì? Muốn giờ Việt Nam thì viết thế nào?

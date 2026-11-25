# Giải thích chi tiết `backend/update_menu.py`

> File: `Smart_Kiosk_Backup/backend/update_menu.py` — 217 dòng, bản trên GitHub `main`.
> Thí nghiệm chạy thật trên **bản sao** CSDL thật + **thư mục ảnh giả** + **thư mục HOME giả** trong vùng nháp — `kiosk.db` thật và ảnh thật **không bị đụng**.

---

## 0. File này để làm gì? (đọc trước)

Script **chạy tay** để **đặt lại toàn bộ thực đơn** về danh sách `latest_menu` (34 món). Dùng khi thay thực đơn hàng loạt (đổi mô hình kinh doanh, dọn dữ liệu thử nghiệm trước khi đưa vào dùng thật).

⚠ **Đây là thao tác PHÁ HỦY** — nó **xóa**:
- toàn bộ món (`product`),
- toàn bộ **đơn hàng** (`orders`) — tức **cả lịch sử doanh thu**,
- toàn bộ nguyên liệu (`ingredient`) và công thức (`recipe`).

Chỉ **giữ** bảng `settings` (mật khẩu quản trị, thông tin cửa hàng).

Vì nguy hiểm như vậy, script được thiết kế với **4 lớp an toàn**:
| Lớp | Cơ chế |
|---|---|
| 1. Kiểm tra dữ liệu trước | `_check_menu()`: tên/ảnh trùng, nhóm sai, giá ≤ 0 → dừng |
| 2. **Xem trước** mặc định | Không có `--apply` → chỉ in sẽ xóa gì, **không thay đổi gì** |
| 3. **Sao lưu** trước khi xóa | Chép CSDL + dời ảnh cũ sang `~/kiosk_backup/<thời_điểm>/` |
| 4. **Giao dịch** | Xóa + nạp trong một `BEGIN … COMMIT`; lỗi → `rollback`, CSDL như cũ |

Quy trình đầy đủ:
```bash
cd backend
python3 update_menu.py            # 1. XEM TRƯỚC
python3 update_menu.py --apply    # 2. thực hiện
python3 seed_inventory.py         # 3. nạp lại 62 nguyên liệu + công thức
```
→ Cặp `update_menu --apply` + `seed_inventory` luôn ra **cùng một CSDL sạch** (đây cũng là cách đúng để chạy lại `seed_inventory` mà không bị nhân đôi nguyên liệu).

Kết quả chạy thật trên bản sao (CSDL thật lúc đó: 34 món, 8 đơn, 62 nguyên liệu, 133 công thức):
```
Se XOA      : 34 mon | 8 don hang | 62 nguyen lieu | 133 dong cong thuc
Anh cu doi  : 1 file ['mon_cu_da_xoa.jpg']
Se NAP      : 34 mon moi
Giu nguyen  : bang settings (mat khau admin, thong tin cua hang)
Day la XEM TRUOC, chua thay doi gi.
--- chạy với --apply ---
[1/4] Da sao luu DB    -> ~/kiosk_backup/20261125_213836/kiosk.db
[2/4] Da doi 1 anh cu -> ~/kiosk_backup/20261125_213836/images
[3/4] Da xoa du lieu cu va nap 34 mon moi
[4/4] Trong DB: 34 mon (food=22, drink=12) | don hang=0 | nguyen lieu=0
Sau: id món 1→34 (đánh số lại từ 1), settings vẫn còn 1 dòng
```

**Cấu trúc file:**
| Dòng | Phần |
|---|---|
| 1–29 | Import, đường dẫn, chọn CSDL, ảnh được giữ |
| 31–76 | `latest_menu` — dữ liệu 34 món |
| 79–111 | 3 hàm phụ: `_count`, `_check_menu`, `_old_images` |
| 114–214 | `update_data()` — xem trước → sao lưu → xóa/nạp → báo cáo |
| 217–218 | `if __name__ == "__main__"` |

---

## 1. Import và đường dẫn (L1–29)

```python
import sqlite3
import os
import sys
import shutil
import datetime

CURRENT_DIR = os.path.dirname(os.path.abspath(__file__))   # .../backend
PROJECT_ROOT = os.path.dirname(CURRENT_DIR)                 # .../Smart_Kiosk_Backup
IMAGES_DIR = os.path.join(PROJECT_ROOT, 'assets', 'images') # .../assets/images
```
| Module | Dùng để |
|---|---|
| `sys` | `sys.argv` (đọc tham số `--apply`), `sys.exit(1)` (thoát báo lỗi) |
| `shutil` | `shutil.move` — **dời** file ảnh (os không có hàm dời file giữa thư mục khác ổ đĩa tiện như vậy) |
| `datetime` | Tạo tên thư mục sao lưu theo thời điểm |

- `os.path.dirname` hai lần → đi lên **hai cấp** (file → `backend/` → thư mục gốc dự án).
- `os.path.join(PROJECT_ROOT, 'assets', 'images')` → ghép nhiều đoạn một lúc.

```python
def _resolve_db_path():
    """Dùng đúng đường dẫn DB mà database.py đang dùng (kể cả khi có cấu hình ghi đè)."""
    try:
        import database
        path = getattr(database, "DB_PATH", None)
        if path:
            return path
    except Exception:
        pass
    return os.path.join(CURRENT_DIR, 'kiosk.db')

DB_PATH = _resolve_db_path()
```
- **`import` bên trong hàm + `try`** → nếu import được `database.py` thì dùng **đúng** đường dẫn nó chọn (tôn trọng `KIOSK_DB_PATH`); lỗi (ví dụ thiếu thư viện) → dự phòng `backend/kiosk.db`.
- `getattr(đối_tượng, "tên", mặc_định)` → lấy thuộc tính theo **tên dạng chuỗi**; không có thì trả mặc định (không ném `AttributeError`).
- `except Exception: pass` → nuốt lỗi **có chủ đích** vì đã có phương án dự phòng.
- **Vì sao quan trọng?** Hai script cùng thao tác CSDL mà mỗi cái tự chọn đường dẫn riêng → có thể **xóa nhầm** CSDL này rồi nạp CSDL kia. Dùng chung `database.DB_PATH` → **một nguồn sự thật**.

```python
KEEP_IMAGES = {"default_food.png", "default.jpg"}
```
- `{ ... }` có phần tử, không có `:` → **set** (tập hợp): kiểm tra `x in set` rất nhanh, không chứa trùng.
- Hai ảnh **mặc định** (món chưa có ảnh) không bao giờ bị dời đi.

---

## 2. `latest_menu` — thực đơn chuẩn (L31–76)

```python
# (tên món, giá, tên file ảnh trong assets/images, nhóm)
# Nhóm: 'food' hoặc 'drink' (chỉ 'drink' mới mở popup chọn size/đá trên Kiosk)
# Tên món đã rút gọn cho vừa màn hình bếp (font 12, khung 100 px).
latest_menu = [
    ("Bánh mì đặc biệt", 35000, "banh_mi_sai_gon_dac_biet.jpg", "food"),
    ...
    ("Nước cam vắt", 30000, "nuoc_cam_vat.jpg", "drink"),
]
```
- List các tuple `(tên, giá, ảnh, nhóm)` — 22 món ăn + 12 đồ uống.
- **Chú thích chứa quyết định thiết kế:**
  - `category` **không chỉ để phân loại** — kiosk dựa vào `'drink'` để **mở hộp chọn size/đá**; món ăn thì thêm thẳng vào giỏ.
  - Tên món **rút gọn** ("Bánh mì gà" thay vì "Bánh mì gà teriyaki") vì màn hình bếp ESP32 hiển thị chữ cỡ 12 trong khung 100 px. Tên file ảnh vẫn giữ tên đầy đủ (`banh_mi_ga_teriyaki.jpg`).
- Giá là **số nguyên đồng**.
- ⚠ Danh sách này **giống hệt** `sample_data` trong `database.py` (kiểm tra thật: 34/34 dòng trùng khớp) → **dữ liệu viết ở hai nơi**. Sửa thực đơn phải sửa **cả hai**, nếu không CSDL mới (do `init_database` tạo) sẽ khác CSDL được đặt lại (do script này). Hướng gọn hơn: để một file import danh sách từ file kia.

---

## 3. Ba hàm phụ (L79–111)

### 3.1 `_count()` — đếm dòng an toàn
```python
def _count(cursor, table):
    try:
        return cursor.execute(f"SELECT COUNT(*) FROM {table}").fetchone()[0]
    except sqlite3.OperationalError:
        return 0
```
- Bảng chưa tồn tại (`no such table`) → trả 0 thay vì sập.
- **Tên bảng ghép bằng f-string** — có an toàn không? **Có**, vì `table` luôn là **hằng do lập trình viên viết** (`"product"`, `"orders"`...), không bao giờ từ người dùng. Và **không thể** dùng `?` cho tên bảng: tham số `?` chỉ thay được **giá trị**, không thay được **tên bảng/tên cột** (*identifier*).

### 3.2 `_check_menu()` — kiểm tra dữ liệu trước khi làm gì
```python
def _check_menu():
    errors = []
    names = [m[0] for m in latest_menu]
    images = [m[2] for m in latest_menu]
    for label, values in (("Ten mon", names), ("Anh", images)):
        dup = sorted({v for v in values if values.count(v) > 1})
        if dup:
            errors.append(f"{label} bi trung: {dup}")
    for name, price, image, category in latest_menu:
        if category not in ("food", "drink"):
            errors.append(f"Mon '{name}' co nhom sai: {category!r}")
        if price <= 0:
            errors.append(f"Mon '{name}' co gia khong hop le: {price}")
    return errors
```
| Cú pháp | Ý nghĩa |
|---|---|
| `[m[0] for m in latest_menu]` | List comprehension: lấy cột tên |
| `for label, values in (("Ten mon", names), ("Anh", images))` | Lặp qua **tuple các cặp** → dùng **một đoạn code** kiểm tra trùng cho cả tên lẫn ảnh |
| `{v for v in values if values.count(v) > 1}` | **Set comprehension**: các giá trị xuất hiện > 1 lần; set tự loại lặp (tên trùng 3 lần chỉ báo 1 lần) |
| `values.count(v)` | Đếm số lần `v` xuất hiện (duyệt cả list → tổng O(n²), với 34 món thì không đáng kể) |
| `sorted(...)` | Set không có thứ tự → sắp lại cho thông báo ổn định, dễ đọc |
| `{category!r}` | `!r` → in bằng `repr()`: **có dấu nháy** `'Food'` → thấy rõ khoảng trắng thừa, chữ hoa |

**Vì sao kiểm tra tên trùng?** Hai món cùng tên → `seed_inventory.py` tra tên → id sẽ chỉ gán công thức cho **một** món. **Ảnh trùng** → hai món hiện cùng một hình. **Nhóm sai** (`"Drink"`, `"do uong"`) → đồ uống không mở hộp chọn size.
Nguyên tắc: **kiểm tra hết trước, phá hủy sau** — phát hiện lỗi khi đã xóa dữ liệu thì quá muộn.

### 3.3 `_old_images()` — tìm ảnh không còn dùng
```python
def _old_images():
    """Ảnh trong assets/images không thuộc menu mới và không phải ảnh mặc định."""
    if not os.path.isdir(IMAGES_DIR):
        return []
    new_names = {m[2] for m in latest_menu}
    return sorted(
        f for f in os.listdir(IMAGES_DIR)
        if os.path.isfile(os.path.join(IMAGES_DIR, f))
        and f not in new_names and f not in KEEP_IMAGES
    )
```
- `os.listdir(thư_mục)` → danh sách **tên** (file lẫn thư mục con).
- `os.path.isfile(...)` → chỉ lấy **file**, bỏ thư mục con.
- `sorted(f for f in ... if ...)` → **generator expression** đưa thẳng vào `sorted` (không cần ngoặc vuông).
- Điều kiện nhiều dòng nối bằng `and` — viết được nhiều dòng vì đang nằm **trong ngoặc tròn**.
- Kết quả: ảnh **của món cũ** (không thuộc menu mới, không phải ảnh mặc định). Thử thật: thư mục có `ca_phe_sua_da.jpg` (thuộc menu), `default.jpg` (mặc định), `mon_cu_da_xoa.jpg` → chỉ `mon_cu_da_xoa.jpg` bị chọn.

---

## 4. `update_data()` — luồng chính (L114–214)

### 4.1 Đọc tham số và kiểm tra (L115–126)
```python
def update_data():
    apply = "--apply" in sys.argv

    errors = _check_menu()
    if errors:
        print("latest_menu co loi, dung lai:")
        for e in errors:
            print("  -", e)
        sys.exit(1)

    if not os.path.exists(DB_PATH):
        print(f"Khong thay file DB: {DB_PATH}")
        sys.exit(1)
```
- `sys.argv` → **list các tham số dòng lệnh**: `python3 update_menu.py --apply` → `['update_menu.py', '--apply']`. `"--apply" in sys.argv` → `True/False`.
- `sys.exit(1)` → thoát chương trình với **mã lỗi 1** (quy ước: 0 = thành công, khác 0 = lỗi; script khác hoặc systemd đọc được mã này).
- `print("  -", e)` → `print` nhiều đối số tự chèn **dấu cách** giữa chúng.
- **Không thấy file CSDL → dừng**: không gọi `sqlite3.connect` khi file không có, vì `connect` sẽ **tạo file rỗng** — "đặt lại thực đơn" trên một CSDL không có bảng là vô nghĩa.

### 4.2 Kết nối và thống kê (L128–135)
```python
    conn = sqlite3.connect(DB_PATH, timeout=10)
    cursor = conn.cursor()

    n_prod = _count(cursor, "product")
    n_order = _count(cursor, "orders")
    n_ing = _count(cursor, "ingredient")
    n_rec = _count(cursor, "recipe")
    old_imgs = _old_images()
```
- Dùng `sqlite3.connect` trực tiếp, **không qua** `database.get_connection()` → **không bật** `foreign_keys`, không đặt WAL/`busy_timeout` (chỉ có `timeout=10` giây — tác dụng tương đương `busy_timeout`). Không bật khóa ngoại không gây hỏng vì thứ tự xóa đã đúng (phần 4.5).

### 4.3 In bản xem trước (L137–151)
```python
    print("=" * 60)
    print(f"DB          : {DB_PATH}")
    ...
    print(f"Se XOA      : {n_prod} mon | {n_order} don hang | {n_ing} nguyen lieu | {n_rec} dong cong thuc")
    print(f"Anh cu doi  : {len(old_imgs)} file {old_imgs if old_imgs else ''}")
    print(f"Se NAP      : {len(latest_menu)} mon moi")
    print("Giu nguyen  : bang settings (mat khau admin, thong tin cua hang)")
    print("=" * 60)

    if not apply:
        print("Day la XEM TRUOC, chua thay doi gi.")
        print("Muon thuc hien: python update_menu.py --apply")
        conn.close()
        return
```
- `"=" * 60` → nhân chuỗi: 60 dấu `=` (đường kẻ).
- Biểu thức điều kiện **bên trong f-string** `{old_imgs if old_imgs else ''}` → có ảnh thì in danh sách, không thì in rỗng.
- In **đường dẫn CSDL** đầu tiên → người chạy thấy ngay đang thao tác trên CSDL nào (tránh nhầm CSDL thật/CSDL thử).
- Kỹ thuật ***dry-run*** (chạy thử): **mặc định an toàn**, muốn phá hủy phải **chủ động** gõ thêm `--apply`. Gõ nhầm lệnh cũng không mất dữ liệu.

### 4.4 Sao lưu (L153–171)
```python
    stamp = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
    backup_dir = os.path.join(os.path.expanduser("~"), "kiosk_backup", stamp)
    os.makedirs(backup_dir, exist_ok=True)

    backup_db = os.path.join(backup_dir, "kiosk.db")
    dst = sqlite3.connect(backup_db)
    conn.backup(dst)  # an toàn với chế độ WAL
    dst.close()
    print(f"[1/4] Da sao luu DB    -> {backup_db}")

    if old_imgs:
        img_backup = os.path.join(backup_dir, "images")
        os.makedirs(img_backup, exist_ok=True)
        for f in old_imgs:
            shutil.move(os.path.join(IMAGES_DIR, f), os.path.join(img_backup, f))
        print(f"[2/4] Da doi {len(old_imgs)} anh cu -> {img_backup}")
    else:
        print("[2/4] Khong co anh cu can doi")
```
| Cú pháp | Ý nghĩa |
|---|---|
| `strftime("%Y%m%d_%H%M%S")` | Thời điểm → `20261125_213836` — sắp xếp theo tên = theo thời gian; mỗi lần chạy một thư mục riêng, **không ghi đè** bản sao lưu cũ |
| `os.path.expanduser("~")` | `~` → thư mục nhà (`/home/<user>`) |
| `os.makedirs(..., exist_ok=True)` | Tạo thư mục **kèm các thư mục cha**; có rồi thì không lỗi |
| `conn.backup(dst)` | **API sao lưu trực tuyến** của SQLite: chép **toàn bộ** CSDL nguồn sang kết nối đích |
| `shutil.move(nguồn, đích)` | **Dời** file (không phải sao chép) |

**Vì sao sao lưu ra `~/kiosk_backup` (ngoài thư mục dự án)?** Không lẫn vào git, không bị xóa khi `git clean`/chép lại thư mục dự án.

**Vì sao dùng `conn.backup()` chứ không `shutil.copy("kiosk.db")`?** Ở chế độ WAL, các thay đổi mới nhất có thể còn nằm trong `kiosk.db-wal`. Chép riêng file `.db` → **bản sao thiếu dữ liệu**. `conn.backup()` đọc qua SQLite nên lấy **đầy đủ** cả phần trong `-wal`, ra **một file** hoàn chỉnh (như lệnh `.backup` đã nói ở slide 5).

**Vì sao dời ảnh cũ chứ không xóa?** Lỡ cần lại (khôi phục món cũ) thì vẫn còn, chỉ không làm rối thư mục ảnh đang dùng.

### 4.5 Xóa và nạp trong một giao dịch (L173–199)
```python
    print("--> Bat dau dat lai Menu...")
    try:
        cursor.execute("BEGIN")
        cursor.execute("DELETE FROM recipe")
        cursor.execute("DELETE FROM ingredient")
        cursor.execute("DELETE FROM product")
        cursor.execute("DELETE FROM orders")
        try:
            cursor.execute("DELETE FROM sqlite_sequence WHERE name IN ('product', 'ingredient')")
        except sqlite3.OperationalError:
            pass

        for name, price, image, category in latest_menu:
            print(f" + New Item: {name}")
            cursor.execute('''
                INSERT INTO product (name, price, image, category)
                VALUES (?, ?, ?, ?)
            ''', (name, price, image, category))

        conn.commit()
    except Exception as e:
        conn.rollback()
        print(f"LOI, da hoan tac (DB khong doi): {e}")
        print(f"Anh cu (neu da doi) nam o: {backup_dir}")
        conn.close()
        sys.exit(1)
```

**Từng dòng:**
- `BEGIN` → mở giao dịch thủ công (Python chưa mở giao dịch ngầm nào vì trước đó chỉ có SELECT) → mọi lệnh sau là **một khối**.
- `DELETE FROM bảng` **không có `WHERE`** → xóa **mọi dòng** (bảng vẫn còn, chỉ trống; khác `DROP TABLE` là xóa cả bảng).
- **Thứ tự xóa: con trước, cha sau** — `recipe` (tham chiếu `product` và `ingredient`) xóa **trước** → đúng ràng buộc khóa ngoại, dù kết nối này không bật kiểm tra.
- `sqlite_sequence` → **bảng hệ thống** SQLite tự tạo khi có cột `AUTOINCREMENT`, lưu **số id lớn nhất đã cấp** cho từng bảng (thử thật trước khi đặt lại: `('product', 34), ('ingredient', 62)`). Xóa dòng của `product`/`ingredient` → id **bắt đầu lại từ 1** (thử thật: sau khi chạy, id món là 1→34). Bọc `try` vì CSDL chưa từng có `AUTOINCREMENT` thì chưa có bảng này.
- `INSERT` với tham số `?` cho từng món; không truyền `status` → lấy mặc định `1` (đang bán).
- **Lỗi bất kỳ** → `rollback()` → **mọi lệnh DELETE cũng bị hủy** → CSDL nguyên vẹn như trước khi chạy. Không bao giờ có cảnh "đã xóa hết mà chưa nạp món mới".

**Một lưu ý về thứ tự các bước:** ảnh cũ được **dời ở bước 2, trước** giao dịch. Nếu giao dịch lỗi, CSDL quay lại như cũ nhưng **ảnh đã bị dời** → thông báo lỗi chỉ rõ ảnh nằm ở `backup_dir` để chép trả lại bằng tay.

**Vì sao xóa cả đơn hàng?** Đơn cũ không trỏ tới món bằng khóa ngoại (lưu bản chụp `items_json`) nên **không bắt buộc** phải xóa. Script xóa vì mục đích của nó là **đưa hệ thống về trạng thái sạch** (bỏ đơn thử nghiệm, doanh thu thử). Đơn thật vẫn còn trong bản sao lưu.

### 4.6 Báo cáo và kiểm tra ảnh (L201–214)
```python
    print(f"[3/4] Da xoa du lieu cu va nap {len(latest_menu)} mon moi")

    food = cursor.execute("SELECT COUNT(*) FROM product WHERE category='food'").fetchone()[0]
    drink = cursor.execute("SELECT COUNT(*) FROM product WHERE category='drink'").fetchone()[0]
    print(f"[4/4] Trong DB: {_count(cursor, 'product')} mon (food={food}, drink={drink}) | "
          f"don hang={_count(cursor, 'orders')} | nguyen lieu={_count(cursor, 'ingredient')}")
    conn.close()

    missing = [m[2] for m in latest_menu if not os.path.exists(os.path.join(IMAGES_DIR, m[2]))]
    if missing:
        print(f"\nChua co anh cho {len(missing)}/{len(latest_menu)} mon (Kiosk hien bieu tuong thay the).")
        print(f"Chep anh vao: {IMAGES_DIR}")

    print("\n✅ DAT LAI MENU THANH CONG! Buoc tiep theo: python seed_inventory.py")
```
- **Đọc lại từ CSDL** sau khi commit để **xác nhận** (22 food, 12 drink, 0 đơn, 0 nguyên liệu) — không chỉ tin rằng lệnh đã chạy đúng.
- Hai f-string **đứng cạnh nhau** trong ngoặc → Python **tự nối** thành một chuỗi (cách viết chuỗi dài qua nhiều dòng).
- Kiểm tra **ảnh còn thiếu** → nhắc chép ảnh; kiosk vẫn chạy (hiện biểu tượng thay thế).
- Nhắc **bước tiếp theo**: `seed_inventory.py` (vì nguyên liệu, công thức vừa bị xóa hết — chưa chạy thì bán không trừ kho).

---

## 5. Lưu ý khi dùng trên hệ thống thật

1. **Dừng kiosk và Smart Kiosk API trước** (`sudo systemctl stop ...`, đóng kiosk). Lý do: (a) kiosk đang giữ đơn PENDING mà đơn bị xóa → tiền về sau sẽ thành `unknown_order`; (b) API/kiosk đang ghi thì script phải chờ khóa (tối đa 10 s) rồi có thể lỗi `database is locked`.
2. **Luôn chạy xem trước** và đọc kỹ dòng `DB :` và `Se XOA :`.
3. Sau `--apply` → chạy `seed_inventory.py` → mở lại dịch vụ.
4. Bản sao lưu nằm ở `~/kiosk_backup/<thời_điểm>/` — khôi phục bằng cách dừng dịch vụ rồi chép `kiosk.db` đó về `backend/` (xóa `kiosk.db-wal`, `kiosk.db-shm` cũ nếu có).

---

## 6. Tổng kết

| Câu hỏi | Trả lời ngắn |
|---|---|
| File làm gì? | Xóa món, đơn, nguyên liệu, công thức; nạp lại 34 món chuẩn; giữ `settings` |
| An toàn thế nào? | Kiểm tra dữ liệu → xem trước mặc định → sao lưu (`conn.backup`, dời ảnh) → một giao dịch, lỗi thì rollback |
| Vì sao `conn.backup()`? | Chép đủ cả phần còn trong `-wal`, ra một file hoàn chỉnh |
| Vì sao xóa `sqlite_sequence`? | Để id món, nguyên liệu đánh lại từ 1 |
| Vì sao tên bảng ghép f-string mà vẫn an toàn? | Tên bảng là hằng trong code; `?` không dùng được cho tên bảng |
| Sau khi chạy phải làm gì? | `python3 seed_inventory.py` |
| Điểm có thể cải thiện | `latest_menu` trùng `sample_data` trong `database.py` (hai nơi phải sửa cùng lúc); ảnh dời trước giao dịch nên lỗi giao dịch thì ảnh không tự trả về |

---

## 7. Câu hỏi tự kiểm tra
1. Chạy `python3 update_menu.py` (không có `--apply`) thì CSDL thay đổi gì?
2. Vì sao kiểm tra `os.path.exists(DB_PATH)` trước khi `sqlite3.connect`?
3. Vì sao không sao lưu bằng cách chép file `kiosk.db`?
4. Vì sao xóa `recipe` trước `product`?
5. Nếu câu `INSERT` món thứ 20 bị lỗi thì 4 bảng ở trạng thái nào? Ảnh cũ ở đâu?
6. `sqlite_sequence` là gì? Không xóa nó thì món mới đầu tiên nhận id bao nhiêu?
7. `{category!r}` và `{category}` in ra khác nhau thế nào?
8. Vì sao phải dừng kiosk trước khi chạy `--apply`?

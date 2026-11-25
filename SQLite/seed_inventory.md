# Giải thích chi tiết `backend/seed_inventory.py`

> File: `Smart_Kiosk_Backup/backend/seed_inventory.py` — 145 dòng, bản trên GitHub `main`.
> Thí nghiệm chạy thật trên **CSDL tạm** (`KIOSK_DB_PATH` trỏ vào thư mục nháp) — **không đụng `kiosk.db` thật**.

---

## 0. File này để làm gì? (đọc trước)

***Seed*** (gieo hạt) = **nạp dữ liệu ban đầu** vào CSDL. File này là **script chạy tay một lần** để:
1. Thêm **62 nguyên liệu** vào bảng `ingredient` (tên, đơn vị, tồn kho ban đầu, ngưỡng sắp hết).
2. Gán **công thức** (định lượng nguyên liệu cho 1 phần) cho **34 món** → **133 dòng** trong bảng `recipe`.

Không có dữ liệu này thì hệ thống bán được nhưng **không trừ kho** (món không có công thức thì `save_new_order` không biết trừ gì).

**Nó không được kiosk hay API gọi** — chỉ chạy bằng tay khi dựng CSDL mới:
```bash
cd ~/Documents/DATN_Final_Version/Smart_Kiosk_Backup/backend
python3 update_menu.py       # (tùy chọn) đặt lại thực đơn — cuối script này in: "Buoc tiep theo: python seed_inventory.py"
python3 seed_inventory.py    # nạp nguyên liệu + công thức
```

Kết quả chạy thật trên CSDL tạm mới tinh:
```
--> Database trong. Dang tao du lieu mau...            ← init_database() tạo 34 món
--> Da khoi tao settings mac dinh (mat khau admin mac dinh: 123456).
--> Dang them nguyen lieu...                            (62 dòng "+ ...")
--> Dang gan cong thuc cho tung mon...                  (34 dòng "+ ...")
✅ HOAN TAT SEED DU LIEU NGUYEN LIEU + CONG THUC!
→ ingredient: 62 dòng | recipe: 133 dòng | 34 món có công thức
```
Khớp đúng số liệu CSDL thật trên Pi (62 nguyên liệu, 133 dòng công thức, không có tên trùng).

**Cấu trúc file** — 3 phần:
| Dòng | Phần |
|---|---|
| 1 | `import database` |
| 4–67 | `INGREDIENTS` — dữ liệu nguyên liệu |
| 69–106 | `RECIPES` — dữ liệu công thức |
| 108–146 | `main()` — logic nạp |

Kiểu viết này gọi là ***tách dữ liệu khỏi logic***: muốn thêm nguyên liệu/sửa định lượng thì chỉ sửa **danh sách**, không đụng vòng lặp.

---

## 1. `import database` (L1)

```python
import database
```
- Import **trực tiếp** `database.py` (không phải `from backend import database` như kiosk).
- Chạy được vì khi gõ `python3 đường/dẫn/seed_inventory.py`, Python tự đặt **thư mục chứa script** (`backend/`) vào đầu `sys.path` → tìm thấy `database.py` nằm cạnh. Chạy thật từ thư mục `/tmp` (`python3 ~/.../backend/seed_inventory.py`) vẫn import được.
- Dòng import **chạy luôn code cấp module** của `database.py` → `DB_PATH = resolve_database_path()` → dùng `backend/kiosk.db`, hoặc CSDL khác nếu đặt biến `KIOSK_DB_PATH`.
- ⚠ Chạy bằng `python3 -m backend.seed_inventory` từ thư mục gốc sẽ **lỗi** `ModuleNotFoundError: database` (lúc đó `sys.path[0]` là thư mục gốc, không phải `backend/`).

---

## 2. `INGREDIENTS` — danh sách nguyên liệu (L4–67)

```python
INGREDIENTS = [
    ("Ổ bánh mì", "cái", 200, 30),
    ("Pate", "g", 2000, 200),
    ...
    ("Trứng gà", "quả", 300, 30),
    ...
    ("Quả bơ", "g", 5000, 500),
]
```

### Cú pháp
- `[ ... ]` → **list**; mỗi phần tử là **tuple 4 trường** `(tên, đơn_vị, tồn_kho, ngưỡng)`.
- Viết HOA → **hằng số** (dữ liệu cố định của script).
- Dấu phẩy cuối mỗi dòng (kể cả dòng cuối) là hợp lệ.

### Ý nghĩa 4 trường → 4 cột bảng `ingredient`
| Vị trí | Ví dụ | Cột | Ý nghĩa |
|---|---|---|---|
| 0 | `"Pate"` | `name` | Tên — dùng làm **khóa nối** với `RECIPES` (phần 4) |
| 1 | `"g"` | `unit` | Đơn vị: `g`, `ml`, `cái`, `quả` |
| 2 | `2000` | `quantity` | Tồn kho ban đầu: 2000 g = 2 kg |
| 3 | `200` | `low_stock_threshold` | Còn ≤ 200 g → báo **sắp hết** |

### Quy ước dữ liệu
- **Ngưỡng ≈ 10% tồn kho ban đầu** gần như mọi dòng (2000→200, 300→30, 10000→1000) → quy tắc đơn giản, dễ giải thích: còn 10% thì cảnh báo nhập hàng.
- **Đơn vị khớp cách đong thực tế**: thịt, rau tính `g`; nước sốt, sữa tính `ml`; bánh tính `cái`; trứng, chanh, tắc tính `quả`.
- **"Quả bơ" tính `g`** còn **"Cam tươi" tính `quả`**: bơ to nhỏ khác nhau, sinh tố dùng 150 g; cam vắt thì đếm quả.
- Nhóm theo loại (bánh → thịt → rau → sốt → gia vị → xôi → ăn vặt → đồ uống) cho dễ đọc, không ảnh hưởng chương trình.

---

## 3. `RECIPES` — công thức theo tên món (L69–106)

```python
# (product_name, [(ingredient_name, qty_per_unit), ...])
RECIPES = [
    ("Bánh mì đặc biệt", [("Ổ bánh mì", 1), ("Pate", 20), ("Bơ lạt", 10), ("Thịt nguội", 30),
                          ("Chả lụa", 30), ("Đồ chua", 20), ("Dưa leo", 15), ("Rau mùi", 3), ("Ớt", 3)]),
    ...
    ("Cà phê sữa đá", [("Cà phê bột", 20), ("Sữa đặc", 30)]),
    ("Trà đào cam sả", [("Trà đen", 5), ("Đào ngâm", 60), ("Cam tươi", 0.5), ("Sả", 5), ("Đường", 15)]),
    ("Nước mía ép", [("Mía", 500)]),
    ...
]
```

### Cú pháp — cấu trúc lồng 3 tầng
```
list                                         ← cả danh sách công thức
 └─ tuple (tên_món, list_nguyên_liệu)        ← 1 món
             └─ list
                 └─ tuple (tên_nguyên_liệu, lượng)   ← 1 dòng công thức
```

### Ý nghĩa
- Đọc: *"1 phần Bánh mì đặc biệt cần 1 ổ bánh mì, 20 g pate, 10 g bơ lạt…"* → đúng nghĩa cột `qty_per_unit` (lượng **cho 1 phần**). Khách mua 2 phần → `save_new_order` nhân 2.
- **`("Cam tươi", 0.5)`** — nửa quả cam → đây là lý do cột `qty_per_unit` và `quantity` có kiểu **`REAL`** (số thực), không phải `INTEGER`.
- Bánh mì đặc biệt có **9 nguyên liệu**, nước mía chỉ **1** → tổng 133 dòng cho 34 món.

### Vì sao viết công thức theo **tên** mà không theo **id**?
Lúc viết script **chưa biết id**: id do `AUTOINCREMENT` cấp khi chèn, phụ thuộc thứ tự và lịch sử CSDL. Tên thì **ổn định và người đọc hiểu ngay** (`("Pate", 20)` dễ hiểu hơn `(3, 20)`). Script sẽ **tra tên → id** lúc chạy (phần 4).
Cái giá: tên trong `RECIPES` phải **khớp từng ký tự** với tên trong `INGREDIENTS` và với `sample_data` của `database.py` (chú thích L69 ghi *"khớp đúng tên trong sample_data"*). Gõ lệch một dấu (`"Bánh mì ga"`) → không tìm thấy → bị bỏ qua kèm cảnh báo.

---

## 4. `main()` — logic nạp (L108–142)

### 4.1 Bảo đảm bảng đã có (L109)
```python
def main():
    database.init_database()
```
- Gọi `init_database()` (idempotent) → bảo đảm 5 bảng tồn tại; CSDL **trống** thì tạo luôn 34 món mẫu + cấu hình mặc định. Không có dòng này, chạy trên CSDL mới sẽ lỗi `no such table: ingredient`.

### 4.2 Thêm nguyên liệu (L111–114)
```python
    print("--> Dang them nguyen lieu...")
    for name, unit, qty, threshold in INGREDIENTS:
        database.admin_add_ingredient(name, unit, qty, threshold)
        print(f"    + {name} ({unit}) - ton kho: {qty}")
```
- `for name, unit, qty, threshold in INGREDIENTS:` → **tách tuple ngay trong vòng `for`** (unpacking): mỗi tuple 4 phần tử gán vào 4 biến. Số biến phải **bằng** số phần tử, sai là lỗi `ValueError`.
- Gọi hàm **có sẵn** `admin_add_ingredient` (câu `INSERT` có tham số `?`) thay vì tự viết SQL → **dùng lại** đúng một đường ghi CSDL với API.
- Mỗi nguyên liệu một lời gọi = một kết nối + một `commit` → chậm hơn `executemany`, nhưng 62 dòng chạy một lần thì không đáng kể.

### 4.3 Tạo bảng tra tên → id (L116–122)
```python
    all_ingredients = database.get_all_ingredients()
    name_to_id = {ing["name"]: ing["id"] for ing in all_ingredients}

    all_products = database.get_all_products()
    product_name_to_id = {p["name"]: p["id"] for p in all_products}
```
- **Dict comprehension** `{khóa: giá_trị for x in danh_sách}` → ví dụ `{"Pate": 3, "Bơ lạt": 4, ...}`.
- Đọc lại từ CSDL **sau khi chèn** → lấy đúng id mà `AUTOINCREMENT` vừa cấp.
- Tra dict là **O(1)** (gần như tức thì) → nhanh hơn duyệt list tìm tên mỗi lần.
- ⚠ `get_all_products()` chỉ trả món **đang bán** (`WHERE status = 1`) → món đang **tạm ngưng** sẽ không có trong bảng tra → bị bỏ qua.
- ⚠ Nếu có **hai dòng cùng tên**, dict giữ **dòng sau cùng** (khóa trùng thì giá trị sau ghi đè giá trị trước) — xem phần 5.

### 4.4 Gán công thức cho từng món (L124–140)
```python
    print("\n--> Dang gan cong thuc cho tung mon...")
    for product_name, recipe_items in RECIPES:
        product_id = product_name_to_id.get(product_name)
        if not product_id:
            print(f"    [BO QUA] Khong tim thay san pham '{product_name}' trong database.")
            continue

        items_payload = []
        for ing_name, qty_per_unit in recipe_items:
            ing_id = name_to_id.get(ing_name)
            if not ing_id:
                print(f"    [CANH BAO] Khong tim thay nguyen lieu '{ing_name}' cho mon '{product_name}'.")
                continue
            items_payload.append({"ingredient_id": ing_id, "qty_per_unit": qty_per_unit})

        database.admin_save_recipe(product_id, items_payload)
        print(f"    + {product_name}: {len(items_payload)} nguyen lieu")
```

| Cú pháp | Ý nghĩa |
|---|---|
| `for product_name, recipe_items in RECIPES` | Tách tuple 2 phần tử: tên món, list nguyên liệu |
| `dict.get(khóa)` | Không có → `None` (không ném `KeyError`) |
| `if not product_id` | `None` → bỏ qua. (Cũng đúng vì id luôn ≥ 1; id 0 không bao giờ có với `AUTOINCREMENT`) |
| `continue` | Sang phần tử tiếp theo của **vòng lặp gần nhất** |
| `"\n"` | Xuống dòng — in thêm một dòng trống |
| `len(items_payload)` | Số nguyên liệu đã gán |

**Logic hai vòng lặp lồng nhau:**
1. Vòng ngoài: mỗi **món**. Không tìm thấy món → `[BO QUA]` cả món.
2. Vòng trong: mỗi **nguyên liệu** của món → đổi tên thành id, đóng gói thành dict `{"ingredient_id", "qty_per_unit"}` — **đúng định dạng** `admin_save_recipe` nhận (cũng là định dạng API nhận từ phần mềm quản trị).
3. Không tìm thấy nguyên liệu → `[CANH BAO]`, **bỏ riêng nguyên liệu đó**, vẫn lưu phần còn lại.
4. `admin_save_recipe` → **xóa công thức cũ của món rồi chèn mới trong một giao dịch `BEGIN IMMEDIATE`** (xem `giai_thich_database_py.md` §14).

**Chiến lược xử lý lỗi: "cảnh báo và chạy tiếp"** (thay vì dừng ngay): một tên gõ sai không làm hỏng 33 món còn lại; đọc dòng `[BO QUA]`/`[CANH BAO]` là biết sửa chỗ nào. Hợp với script nạp dữ liệu chạy tay.

### 4.5 Kết thúc (L142–146)
```python
    print("\n✅ HOAN TAT SEED DU LIEU NGUYEN LIEU + CONG THUC!")


if __name__ == "__main__":
    main()
```
- `if __name__ == "__main__":` → chỉ chạy `main()` khi **gọi trực tiếp** file; nếu file khác `import seed_inventory` (ví dụ để dùng lại `INGREDIENTS`) thì **không tự nạp dữ liệu**.
- Gói logic vào hàm `main()` thay vì viết thẳng ở cấp module → dễ đọc, dễ gọi lại, và biến bên trong là **biến cục bộ**.
- Thông báo không dấu (`Dang them nguyen lieu`) — thói quen tránh lỗi hiển thị tiếng Việt trên terminal Windows lúc đầu dự án.

---

## 5. ⚠ Điểm cần biết: **chạy 2 lần sẽ nhân đôi nguyên liệu**

Thí nghiệm thật — chạy script **lần thứ 2** trên cùng CSDL tạm:
```
nguyên liệu: 124 | công thức: 133
Pate: [(3, 2000.0), (65, 2000.0)]           ← 2 dòng "Pate"
Công thức trỏ tới id Pate: [(65,)]          ← công thức chuyển sang dòng MỚI
```

**Vì sao?**
| Bước | Có idempotent không? | Lý do |
|---|---|---|
| `init_database()` | ✔ | `CREATE TABLE IF NOT EXISTS`, chỉ chèn món mẫu khi bảng trống |
| Thêm nguyên liệu | **✘** | `INSERT` thẳng; bảng `ingredient` **không có ràng buộc `UNIQUE(name)`** → chèn thêm 62 dòng trùng tên |
| Bảng tra `name_to_id` | — | Tên trùng → dict giữ id **sau cùng** (dòng mới) |
| Gán công thức | ✔ | `admin_save_recipe` xóa cũ ghi mới → vẫn 133 dòng, nhưng trỏ sang **nguyên liệu mới** |

**Hậu quả nếu lỡ chạy lại trên CSDL thật:** 62 nguyên liệu cũ (với **số tồn kho thật** đã nhập/trừ) trở thành **"mồ côi"** — vẫn hiện trong trang kho nhưng không món nào dùng; bán hàng trừ vào 62 dòng **mới** (tồn kho reset về số ban đầu). Trang kho hiện **hai dòng "Pate"** gây nhầm lẫn.

**Hiện trạng:** CSDL thật **không bị** (kiểm tra chỉ đọc: 62 nguyên liệu, 0 tên trùng) → script mới chỉ được chạy đúng một lần.

**Nguyên tắc sử dụng:** chỉ chạy **một lần trên CSDL mới**. Nếu muốn script an toàn khi chạy lại, có các hướng (ghi nhận, chưa sửa):
- Bỏ qua nguyên liệu **đã có tên** trước khi chèn (kiểm tra trong `name_to_id` trước vòng thêm);
- Hoặc thêm ràng buộc `UNIQUE(name)` cho bảng `ingredient` và dùng `INSERT OR IGNORE`.

---

## 6. Liên hệ với các file khác

```
database.py  ── init_database(): tạo bảng + 34 món mẫu (sample_data)
     ▲
     │ import
seed_inventory.py ── INGREDIENTS (62) ──► admin_add_ingredient() ──► bảng ingredient
                  └─ RECIPES (34 món) ──► admin_save_recipe()   ──► bảng recipe
     ▲
update_menu.py ── đặt lại thực đơn, cuối cùng nhắc: "python seed_inventory.py"
```
Sau khi seed, khi khách mua **1 Cà phê sữa đá**, `save_new_order` JOIN `recipe` với `ingredient` → trừ **20 g Cà phê bột** và **30 ml Sữa đặc**.

---

## 7. Tổng kết

| Câu hỏi | Trả lời ngắn |
|---|---|
| File làm gì? | Nạp 62 nguyên liệu và 133 dòng công thức cho 34 món |
| Ai chạy? | Người quản trị, **bằng tay, một lần** khi dựng CSDL mới |
| Vì sao công thức viết theo tên? | Id do CSDL cấp lúc chạy; tên ổn định, dễ đọc → tra tên → id bằng dict |
| Vì sao gọi `admin_*` thay vì tự viết SQL? | Dùng lại một đường ghi đã có kiểm tra, tham số `?`, giao dịch |
| Gặp tên sai thì sao? | In cảnh báo, bỏ qua mục đó, chạy tiếp |
| Chạy 2 lần thì sao? | **Nhân đôi nguyên liệu** (không có `UNIQUE(name)`); công thức trỏ sang bản mới |

---

## 8. Câu hỏi tự kiểm tra
1. Vì sao `import database` chạy được dù không có `from backend`?
2. `for name, unit, qty, threshold in INGREDIENTS` sẽ lỗi gì nếu một tuple chỉ có 3 phần tử?
3. Vì sao phải đọc lại `get_all_ingredients()` sau khi thêm, không tự đánh số 1, 2, 3...?
4. `("Cam tươi", 0.5)` cho biết điều gì về kiểu cột trong CSDL?
5. Một món đang **tạm ngưng bán** có được gán công thức không? Vì sao?
6. Giải thích vì sao chạy lần 2 thì công thức trỏ sang id nguyên liệu **mới**.
7. `continue` ở vòng trong và vòng ngoài khác nhau thế nào về phạm vi bỏ qua?

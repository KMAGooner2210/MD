# Giải thích chi tiết `backend/desktop_env.py`

> File: `Smart_Kiosk_Backup/backend/desktop_env.py` — 53 dòng, bản trên GitHub `main`.
> Thêm vào ở commit `b86ba6c` *"Sửa crash GLib khi chạy Admin/Kiosk từ terminal VS Code bản snap"*.
> Được gọi bởi: `kiosk_app/main.py` (dòng 6–9), `admin_app/main_admin.py` (dòng 6–9). Kiểm thử: `tests/test_desktop_env.py` (2/2 đạt).

---

## 0. File này giải quyết vấn đề gì? (đọc trước)

**Triệu chứng:** mở terminal **bên trong VS Code**, gõ `python3 kiosk_app/main.py` (hoặc `admin_app/main_admin.py`) → chương trình **sập ngay** khi tạo cửa sổ, với lỗi:
```
GLib-GIO-ERROR: Settings schema 'org.gnome.settings-daemon.plugins.xsettings'
does not contain a key named 'antialiasing'
```
Chạy **cùng lệnh** ở terminal thường của Ubuntu (Ctrl+Alt+T) → **chạy bình thường**.

**Nguyên nhân, theo từng bước:**

1. Trên laptop, VS Code được cài bằng **snap** (chạy thật: `snap list` → `code ... classic`).
2. ***Snap*** là cách đóng gói ứng dụng Linux kèm **toàn bộ thư viện riêng** của nó (trong thư mục `/snap/code/...`). Để VS Code dùng thư viện giao diện (GTK) của riêng nó, snap **đặt thêm các biến môi trường** trỏ vào thư mục snap: `GSETTINGS_SCHEMA_DIR`, `GTK_PATH`, `XDG_DATA_DIRS`...
3. ***Biến môi trường*** được **tiến trình con thừa hưởng** từ tiến trình cha. Terminal mở trong VS Code là con của VS Code → mọi chương trình chạy từ terminal đó (kiosk, phần mềm quản trị) **cũng mang các biến trỏ vào snap**.
4. Qt trên Ubuntu nạp plugin để giao diện hòa hợp với GNOME/GTK. Plugin này đọc cấu hình hệ thống (***GSettings schema*** — bản mô tả các khóa cấu hình hợp lệ) theo `GSETTINGS_SCHEMA_DIR` → đọc nhầm **bộ schema cũ đóng gói trong snap**.
5. Bộ schema cũ **không có khóa `antialiasing`** mà hệ thống mới dùng → thư viện GLib coi là lỗi nghiêm trọng → **dừng chương trình**.

**Cách sửa:** trước khi nạp Qt, **gỡ các biến trỏ vào snap** (hoặc trả chúng về giá trị gốc) → Qt đọc đúng cấu hình của hệ thống.

Hình dung:
```
VS Code (snap) ── đặt biến trỏ vào /snap/... ──► terminal trong VS Code
                                                    └─► python3 kiosk_app/main.py
                                                          │ clean_snap_terminal_env()  ← gỡ biến snap
                                                          └─► nạp Qt → đọc schema của HỆ THỐNG → chạy bình thường
```

**Trên Raspberry Pi** hay khi mở bằng **biểu tượng trên màn hình** / systemd: không có các biến này → hàm **không làm gì**.

---

## 1. Docstring đầu file (L1–12)

```python
"""Dọn biến môi trường do terminal của VS Code bản snap chèn vào.
...
Phải gọi hàm trước khi tạo QGuiApplication. Trên Raspberry Pi hoặc terminal
thường không có các biến này nên hàm không làm gì.
"""
```
- Chuỗi `"""..."""` ở **đầu file** = ***docstring của module***: tài liệu cho cả file, xem bằng `help(desktop_env)`.
- Ghi rõ 3 điều người đọc cần biết: **vì sao có file**, **lỗi cụ thể** (chép nguyên văn để tìm kiếm được), **điều kiện dùng** (gọi trước khi tạo Qt).

---

## 2. Import (L14–15)

```python
import os
import sys
```
- `os.environ` → bảng biến môi trường của tiến trình hiện tại.
- `sys.platform` → tên hệ điều hành: `'linux'`, `'win32'`, `'darwin'` (macOS).

---

## 3. Hai nhóm biến (L17–25)

```python
_SNAP_GTK_VARS = (
    "GSETTINGS_SCHEMA_DIR",
    "GIO_MODULE_DIR",
    "GTK_PATH",
    "GTK_EXE_PREFIX",
    "GTK_IM_MODULE_FILE",
)

_SNAP_RESTORABLE_VARS = ("XDG_DATA_DIRS", "XDG_CONFIG_DIRS")
```

### Cú pháp
- Tên bắt đầu `_` → quy ước **dùng nội bộ** trong file; viết HOA → **hằng số**.
- `( ..., )` → **tuple**: không sửa được, phù hợp danh sách cố định. Dấu phẩy sau phần tử cuối là hợp lệ (dễ thêm dòng mới).

### Vì sao chia 2 nhóm?
| Nhóm | Biến | Bình thường (terminal Ubuntu) có không? | Xử lý |
|---|---|---|---|
| `_SNAP_GTK_VARS` | `GSETTINGS_SCHEMA_DIR` (nơi tìm schema cấu hình), `GIO_MODULE_DIR` (plugin của thư viện GIO), `GTK_PATH`, `GTK_EXE_PREFIX` (nơi tìm thư viện GTK), `GTK_IM_MODULE_FILE` (bộ gõ) | **Không có** — chỉ snap đặt | **Xóa hẳn** → thư viện dùng đường dẫn mặc định của hệ thống |
| `_SNAP_RESTORABLE_VARS` | `XDG_DATA_DIRS` (các thư mục dữ liệu dùng chung: biểu tượng, phông, schema), `XDG_CONFIG_DIRS` (các thư mục cấu hình) | **Có** — hệ thống luôn đặt | **Trả về giá trị gốc** — xóa hẳn sẽ làm mất các đường dẫn hợp lệ |

---

## 4. Hàm `clean_snap_terminal_env()` (L28–53)

### 4.1 Khai báo và tham số (L28–33)
```python
def clean_snap_terminal_env(environ=None):
    """Trả về danh sách biến đã gỡ/khôi phục."""
    if environ is None:
        environ = os.environ
    if not sys.platform.startswith("linux"):
        return []
```
- `environ=None` → tham số **có giá trị mặc định**. Chạy thật thì gọi `clean_snap_terminal_env()` (không truyền) → dùng `os.environ` thật. Khi **kiểm thử** thì truyền một `dict` giả → hàm sửa dict đó, **không đụng môi trường thật** của máy chạy test.
  → Kỹ thuật này gọi là ***tiêm phụ thuộc*** (*dependency injection*): thứ hàm thao tác được đưa từ ngoài vào, nhờ đó dễ kiểm thử.
- Vì sao không viết `def f(environ=os.environ)`? Giá trị mặc định trong Python được tính **một lần lúc định nghĩa hàm**; dùng `None` rồi gán bên trong là **mẫu chuẩn**, rõ ràng và an toàn hơn.
- `.startswith("linux")` → chuỗi bắt đầu bằng `"linux"`? Windows/macOS không có snap → trả `[]` ngay.

### 4.2 Nhóm 1 — xóa biến chỉ snap mới đặt (L35–40)
```python
    cleaned = []

    for name in _SNAP_GTK_VARS:
        if "/snap/" in environ.get(name, ""):
            del environ[name]
            cleaned.append(name)
```
| Cú pháp | Ý nghĩa |
|---|---|
| `environ.get(name, "")` | Lấy giá trị; biến không tồn tại → `""` (không ném lỗi `KeyError`) |
| `"/snap/" in chuỗi` | Kiểm tra **chuỗi con** |
| `del environ[name]` | **Xóa** biến khỏi môi trường. Với `os.environ`, Python gọi luôn hàm hệ điều hành `unsetenv` → các thư viện C (GLib, Qt) **cũng không thấy biến nữa** |
| `cleaned.append(name)` | Ghi lại để trả về (biết hàm đã làm gì) |

**Chỉ xóa khi giá trị chứa `/snap/`** → nếu chính bạn cố ý đặt `GTK_PATH` trỏ chỗ khác, hàm **không đụng vào**.

**Vì sao tìm `"/snap/"` (có cả hai dấu `/`)?** Terminal thường của Ubuntu có `XDG_DATA_DIRS` chứa `/var/lib/snapd/desktop` (thư mục **hợp lệ** của hệ thống để hiện biểu tượng ứng dụng snap). Chạy thật:
```
"/snap/" in "/var/lib/snapd/desktop"        → False   ← "snapd" không khớp → không đụng
"/snap/" in "/home/u/snap/code/274/x"       → True    ← đường dẫn VS Code snap → xử lý
```
Nếu chỉ tìm `"snap"` → xóa nhầm giá trị bình thường.

### 4.3 Nhóm 2 — trả về giá trị gốc (L42–51)
```python
    # VS Code snap lưu giá trị gốc vào <TÊN>_VSCODE_SNAP_ORIG.
    for name in _SNAP_RESTORABLE_VARS:
        original = environ.get(f"{name}_VSCODE_SNAP_ORIG")
        if original is None or "/snap/" not in environ.get(name, ""):
            continue
        if original:
            environ[name] = original
        else:
            environ.pop(name, None)
        cleaned.append(name)

    return cleaned
```

**Mấu chốt:** trước khi ghi đè `XDG_DATA_DIRS`, VS Code snap **cất giá trị gốc** vào biến tên `XDG_DATA_DIRS_VSCODE_SNAP_ORIG`. Hàm chỉ việc **lấy lại**.

| Cú pháp | Ý nghĩa |
|---|---|
| `f"{name}_VSCODE_SNAP_ORIG"` | Ghép tên: `"XDG_DATA_DIRS_VSCODE_SNAP_ORIG"` |
| `environ.get(...)` **không** có mặc định | Không tồn tại → `None`. Phải phân biệt **`None`** (không có bản gốc) với **`""`** (bản gốc là rỗng) |
| `A or B` trong `if` | Một trong hai đúng là `continue` (bỏ qua biến này) |
| `"/snap/" not in ...` | Giá trị hiện tại **không** trỏ vào snap → không cần sửa |
| `environ.pop(name, None)` | Xóa biến; **không có thì thôi**, không ném lỗi (khác `del`) |

**Bảng quyết định** cho mỗi biến:
| Bản gốc `_VSCODE_SNAP_ORIG` | Giá trị hiện tại có `/snap/`? | Hành động |
|---|---|---|
| Không tồn tại (`None`) | bất kỳ | Bỏ qua — không biết giá trị gốc thì không đoán |
| tồn tại | Không | Bỏ qua — đã sạch rồi |
| Chuỗi **khác rỗng** | Có | **Gán lại** giá trị gốc |
| Chuỗi **rỗng** `""` | Có | **Xóa** biến (trước khi VS Code chạy, biến này vốn không có) |

**Không xóa biến `_VSCODE_SNAP_ORIG`** → vô hại, các chương trình khác không đọc nó.

### 4.4 Giá trị trả về
Danh sách tên biến đã xử lý, ví dụ `['GSETTINGS_SCHEMA_DIR', 'GTK_PATH', 'XDG_DATA_DIRS']`. Chạy từ terminal thường → `[]`. Hiện nơi gọi không dùng giá trị này, nhưng nó giúp **kiểm thử** và **gỡ lỗi** (in ra để biết hàm đã làm gì).

---

## 5. Cách được gọi — vì sao phải gọi "trước khi import Qt"

`kiosk_app/main.py` (tương tự `admin_app/main_admin.py`):
```python
import sys
import os

sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))  # ①

from backend.desktop_env import clean_snap_terminal_env

# Phải chạy trước khi nạp Qt: tránh crash GLib khi mở từ terminal VS Code (snap).
clean_snap_terminal_env()                                                         # ②

from PyQt6.QtGui import QGuiApplication                                           # ③
from PyQt6.QtQml import QQmlApplicationEngine
...
```
- ① `sys.path` = danh sách thư mục Python tìm module khi `import`. Chèn **thư mục gốc dự án** vào đầu → `from backend...` tìm được dù chạy từ đâu. (`dirname` hai lần: `kiosk_app/main.py` → `kiosk_app/` → thư mục gốc.)
- ② Gọi hàm **ngay sau import, trước mọi import Qt** — khác với thói quen thường gom tất cả `import` lên đầu file. Lý do: thư viện C của Qt/GLib **đọc biến môi trường khi khởi tạo**; dọn sau khi Qt đã đọc là **muộn**.
- ③ Kiosk dùng **PyQt6**, phần mềm quản trị dùng **PySide6** — hai thư viện Python khác nhau cho cùng bộ Qt 6; cả hai đều gặp lỗi này nên đều gọi hàm.

**Chỉ ảnh hưởng tiến trình hiện tại:** sửa `os.environ` chỉ thay đổi môi trường của **chính chương trình đó** (và các tiến trình con nó tạo sau này). Terminal VS Code và VS Code **không bị ảnh hưởng**.

---

## 6. Kiểm thử — `tests/test_desktop_env.py`

```python
pytestmark = pytest.mark.skipif(not sys.platform.startswith("linux"), reason="chỉ Linux")
```
- `pytestmark` ở cấp module → áp dụng cho **mọi** test trong file: không phải Linux thì **bỏ qua** (không tính là lỗi).

**Test 1 — môi trường VS Code snap:** đưa vào dict giả lập đủ các trường hợp:
```python
env = {
    "GSETTINGS_SCHEMA_DIR": "/home/u/snap/code/274/.local/share/glib-2.0/schemas",   # nhóm 1 → xóa
    "GIO_MODULE_DIR": "/home/u/snap/code/common/.cache/gio-modules",                 # nhóm 1 → xóa
    "GTK_PATH": "/snap/code/274/usr/lib/x86_64-linux-gnu/gtk-3.0",                   # nhóm 1 → xóa
    "XDG_DATA_DIRS": "/home/u/snap/code/274/.local/share:/usr/share",                # nhóm 2, gốc khác rỗng → khôi phục
    "XDG_DATA_DIRS_VSCODE_SNAP_ORIG": "/usr/share/ubuntu:/usr/share",
    "XDG_CONFIG_DIRS": "/snap/code/274/etc/xdg:/etc/xdg",                            # nhóm 2, gốc rỗng → xóa
    "XDG_CONFIG_DIRS_VSCODE_SNAP_ORIG": "",
    "HOME": "/home/u",                                                               # không liên quan → giữ
}
```
Kiểm tra: danh sách trả về đúng 5 tên (so sánh bằng `set` vì **không quan tâm thứ tự**), và dict sau khi xử lý **khớp chính xác** kết quả mong đợi.

**Test 2 — terminal thường:** `XDG_DATA_DIRS` có `/var/lib/snapd/desktop`, có `GTK_MODULES` → hàm trả `[]` và dict **không đổi một ký tự** (`before = dict(env)` chụp lại trước, so sánh sau). Test này **chốt** lý do dùng `"/snap/"` thay vì `"snap"`.

Chạy thật: `2 passed in 0.01s`.

---

## 7. Tổng kết

| Câu hỏi | Trả lời ngắn |
|---|---|
| File làm gì? | Gỡ/khôi phục các biến môi trường mà VS Code bản snap chèn vào, trước khi nạp Qt |
| Vì sao cần? | Qt/GLib đọc nhầm bộ schema cũ của snap → thiếu khóa `antialiasing` → sập |
| Khi nào có tác dụng? | Chỉ khi chạy từ **terminal trong VS Code snap** trên Linux |
| Trên Pi / biểu tượng / systemd? | Không có biến snap → không làm gì |
| Vì sao gọi trước `import PyQt6`? | Thư viện C đọc biến môi trường lúc khởi tạo |
| Vì sao có tham số `environ`? | Để test truyền dict giả, không đụng môi trường thật |
| Vì sao tìm `"/snap/"`? | Tránh nhầm `/var/lib/snapd/desktop` hợp lệ |

**Liên hệ với đồ án:** đây là file **hỗ trợ môi trường phát triển**, không liên quan nghiệp vụ (đơn hàng, thanh toán). Nếu bị hỏi, trả lời: *"Khi phát triển em chạy ứng dụng từ terminal của VS Code bản snap thì Qt bị sập do snap chèn biến môi trường trỏ vào thư viện riêng của nó. Em viết hàm dọn các biến đó trước khi khởi tạo Qt, chỉ tác động khi giá trị trỏ vào `/snap/`, và có test cho cả trường hợp có snap lẫn terminal bình thường. Trên Pi thì hàm không làm gì."*

---

## 8. Câu hỏi tự kiểm tra
1. Vì sao chạy từ terminal thường thì không lỗi mà từ terminal VS Code thì lỗi?
2. Hai nhóm biến được xử lý khác nhau thế nào? Vì sao không xóa luôn `XDG_DATA_DIRS`?
3. `environ.get(x)` trả `None` và trả `""` khác nhau thế nào trong nhóm 2?
4. `del environ[name]` và `environ.pop(name, None)` khác nhau ở đâu?
5. Vì sao lời gọi hàm phải nằm **giữa** các dòng `import`?
6. Tham số `environ=None` giúp gì cho việc kiểm thử?

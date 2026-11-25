# Giải thích chi tiết `admin_app/admin_bridge.py`

> File: `Smart_Kiosk_Backup/admin_app/admin_bridge.py` — 360 dòng, bản trên GitHub `main`.
> Thí nghiệm chạy thật: dựng **Smart Kiosk API thật** trên cổng cục bộ 5987 với **CSDL tạm**, **khóa thử**, thư mục ảnh tải lên là **thư mục nháp**; rồi gọi bằng **đúng lớp `AdminBridge`** → không đụng dữ liệu, ảnh hay `.env` thật (kiểm tra sau khi chạy: `assets/images` thật không có file mới).

---

## 0. Bức tranh tổng thể (đọc trước)

### 0.1 File này là gì?
Phần mềm quản trị (chạy trên **laptop**) gồm giao diện **QML** + lớp Python **`AdminBridge`**. Lớp này là **máy khách HTTP** (*HTTP client*): mỗi nút bấm trên giao diện → một **yêu cầu HTTP** tới Smart Kiosk API trên Pi.

```
┌──── Laptop: phần mềm quản trị ─────┐                ┌──── Raspberry Pi ────┐
│ QML (Dashboard, Menu, Kho, ...)     │                │                      │
│   adminBackend.deleteProduct(5)     │                │                      │
│        ▼                            │  HTTP + JSON   │                      │
│ AdminBridge._request("POST",        │ ─────────────► │ api_server.py :5000  │──► kiosk.db
│   "/api/menu/delete", json={"id":5})│  X-API-Key     │                      │
│        ◄── (ok, data) ──────────────│ ◄───────────── │ 200 / 4xx + reason   │
│   lỗi → _last_error (tiếng Việt)    │                │                      │
└─────────────────────────────────────┘                └──────────────────────┘
```

**So sánh với `kiosk_app/bridge.py`:**
| | `kiosk_app/bridge.py` | `admin_app/admin_bridge.py` |
|---|---|---|
| Chạy ở đâu | Pi | Laptop |
| Truy cập CSDL | **Trực tiếp** (`from backend import database`) | **Qua HTTP** (không import `database`) |
| Thư viện Qt | PyQt6 | **PySide6** |
| Độ dài | 1442 dòng (nhiều logic) | 360 dòng (chủ yếu "gọi API") |

Phần mềm quản trị **không thể** mở `kiosk.db` trực tiếp vì CSDL là **file trên Pi** → mọi thứ đi qua API. Đây là lý do toàn bộ file là **máy khách HTTP**.

### 0.2 Kết quả chạy thử
```
getMenu()                         -> 34 món; ảnh = http://127.0.0.1:5987/images/banh_mi_sai_gon_dac_biet.jpg
fetchStats()                      -> {"chart": [], "orders": 0, "revenue": 0}
addProduct có ảnh                 -> True; máy chủ lưu thành b4ea64d4_mon_thu.png
deleteProduct(9999)               -> False | "Không tìm thấy món (có thể đã bị xóa)."
addIngredient số lượng âm         -> False | "Dữ liệu nhập không hợp lệ."
saveRecipe JSON hỏng              -> False | "Dữ liệu nhập không hợp lệ."  (không gửi đi)
login sai lần 6                   -> False | "Sai quá nhiều lần, vui lòng thử lại sau 300 giây."
getStoreInfo bằng khóa BẾP        -> '{}'  | "Khóa API không đủ quyền quản trị."
getInventory không khóa           -> []    | "Sai khóa API (ADMIN_API_KEY) – kiểm tra file .env."
fetchStats khi máy chủ TẮT        -> '{}'  | "Không kết nối được máy chủ trên Pi."   (0,00 s)
fetchStats tới IP không phản hồi  -> '{}'  | "Không kết nối được máy chủ trên Pi."   (3,01 s = timeout)
requestStats gọi chồng            -> lần 2 bị bỏ qua khi lần 1 chưa xong
```

### 0.3 Bản đồ file
| Dòng | Phần |
|---|---|
| 1–22 | Import, địa chỉ API, khóa |
| 25–63 | Bảng thông báo lỗi tiếng Việt, hàm phụ |
| 66–107 | Lớp `AdminBridge`, **hàm gọi API dùng chung `_request`** |
| 109–158 | Thư mục ảnh, thông báo lỗi, **thống kê (có luồng nền)** |
| 160–204 | Thực đơn (có tải ảnh lên) |
| 206–252 | Báo cáo, danh sách đơn, xuất CSV |
| 254–307 | Cửa hàng, đăng nhập, mật khẩu, OTP |
| 309–360 | Kho, công thức |

---

## 1. Import và cấu hình (L1–22)

```python
import sys, os, csv, json, threading
import requests
from datetime import datetime
from dotenv import load_dotenv

from PySide6.QtCore import QObject, Signal, Slot, QUrl

_current_dir = os.path.dirname(os.path.abspath(__file__))
_project_root = os.path.dirname(_current_dir)
load_dotenv(os.path.join(_project_root, ".env"))

# PI_URL = "http://192.168.131.7:5000"
PI_URL = os.environ.get("API_SERVER_URL", "http://127.0.0.1:5000")

ADMIN_API_KEY = os.environ.get("ADMIN_API_KEY", "")
_AUTH_HEADERS = {"X-API-Key": ADMIN_API_KEY} if ADMIN_API_KEY else {}
```

| Thành phần | Ý nghĩa |
|---|---|
| `requests` | Thư viện HTTP phổ biến nhất của Python (bản 2.25.1 trên máy) — gửi yêu cầu, nhận phản hồi, tự xử lý JSON/form/file |
| `csv` | Ghi file báo cáo `.csv` (mở bằng Excel) |
| `threading` | Luồng nền tải thống kê + khóa chống gọi chồng |
| `PySide6.QtCore` | `QObject`, `Signal`, `Slot` cho QML; `QUrl` đổi đường dẫn file |
| `PI_URL` | **Địa chỉ gốc** của API, đọc từ `.env` (`API_SERVER_URL`). Dòng chú thích phía trên là **di tích** lúc viết cứng IP Pi — nay đưa ra `.env` vì IP Pi đổi theo mạng Wi-Fi |
| Mặc định `127.0.0.1:5000` | Chạy API ngay trên máy đang mở phần mềm (khi thử nghiệm) |
| `_AUTH_HEADERS` | **Một dict header dùng cho mọi yêu cầu**. Không có khóa → `{}` (máy chủ sẽ trả 401/503, có thông báo rõ) |

**Biểu thức điều kiện** `A if điều_kiện else B` → tạo dict **một lần** lúc nạp module.

---

## 2. Thông báo lỗi tiếng Việt (L25–63)

### 2.1 Bảng `_REASON_MESSAGES`
```python
_REASON_MESSAGES = {
    "network_error": "Không kết nối được máy chủ trên Pi.",
    "unauthorized": "Sai khóa API (ADMIN_API_KEY) – kiểm tra file .env.",
    "forbidden": "Khóa API không đủ quyền quản trị.",
    "product_not_found": "Không tìm thấy món (có thể đã bị xóa).",
    "ingredient_in_use": "Nguyên liệu đang có trong công thức món – hãy sửa công thức trước.",
    "constraint_violation": "Dữ liệu tham chiếu không tồn tại.",
    "too_many_attempts": "Sai quá nhiều lần, vui lòng thử lại sau {retry_after} giây.",
    ...
}
```
**Hợp đồng lỗi giữa máy chủ và máy khách:** API trả **mã lý do ngắn, tiếng Anh, ổn định** (`"reason": "ingredient_in_use"`) — dành cho **máy đọc**. Máy khách **dịch** thành câu tiếng Việt cho **người đọc**. Tách hai việc này:
- Máy chủ không phụ thuộc ngôn ngữ giao diện; ESP32, Kitchen MCP cũng dùng được cùng mã.
- Đổi câu chữ trên giao diện không cần sửa máy chủ.

`"{retry_after}"` → **chỗ giữ chỗ** sẽ điền số giây từ phản hồi.

### 2.2 `_reason_message()`
```python
def _reason_message(reason, data=None):
    if not reason:
        return "Thao tác thất bại, vui lòng thử lại."
    if reason.startswith("invalid_") and reason not in _REASON_MESSAGES:
        return "Dữ liệu nhập không hợp lệ."
    template = _REASON_MESSAGES.get(reason, f"Thao tác thất bại ({reason}).")
    try:
        return template.format(**(data or {}))
    except (KeyError, IndexError):
        return template
```
| Cú pháp | Ý nghĩa |
|---|---|
| `reason.startswith("invalid_")` | Gom mọi mã `invalid_quantity`, `invalid_price`, `invalid_id`... thành **một** câu chung — không phải viết từng câu |
| `.get(reason, f"...({reason}).")` | Mã lạ (máy chủ mới thêm) → vẫn hiện được, kèm mã để tra |
| `template.format(**data)` | `**dict` → **bung dict thành tham số tên**: `"{retry_after}".format(retry_after=300)` → `"300"`. Phản hồi 429 có `{"retry_after": 300, ...}` nên điền thẳng được |
| `data or {}` | `data` là `None` → dùng dict rỗng |
| `except (KeyError, IndexError)` | Câu có `{...}` nhưng phản hồi thiếu trường → trả câu gốc thay vì lỗi |

Thử thật: `_reason_message('invalid_xyz')` → *"Dữ liệu nhập không hợp lệ."*; `_reason_message('abc')` → *"Thao tác thất bại (abc)."*

### 2.3 `_strip_file_url()`
```python
def _strip_file_url(path):
    if path.startswith("file://"):
        return QUrl(path).toLocalFile()
    return path
```
Hộp chọn file của QML trả **URL** `file:///home/.../banh_mi.jpg` (có thể mã hóa ký tự `%20`...). `open()` của Python cần **đường dẫn thường** `/home/.../banh_mi.jpg` → `QUrl(...).toLocalFile()` đổi đúng, kể cả giải mã ký tự đặc biệt.

---

## 3. Hàm gọi API dùng chung `_request()` — trái tim của file (L85–107)

```python
    def _request(self, method, path, timeout=5, **kwargs):
        """Trả (ok, data). ok=True khi HTTP 2xx; data là JSON đã giải mã (hoặc None)."""
        try:
            res = requests.request(method, f"{PI_URL}{path}", headers=_AUTH_HEADERS,
                                   timeout=timeout, **kwargs)
        except requests.RequestException as e:
            print(f"Lỗi kết nối {method} {path}:", e)
            self._last_error = _reason_message("network_error")
            return False, None
        try:
            data = res.json()
        except ValueError:
            data = None
        ok = 200 <= res.status_code < 300
        if ok and isinstance(data, dict) and data.get("success") is False:
            ok = False
        if not ok:
            reason = data.get("reason", "") if isinstance(data, dict) else ""
            self._last_error = _reason_message(reason, data if isinstance(data, dict) else None)
            print(f"API {method} {path} -> {res.status_code} {reason}")
        else:
            self._last_error = ""
        return ok, data
```

**Vì sao gom vào một hàm?** File có **23 chỗ** gọi API (`self._request(`). Nếu mỗi chỗ tự viết `requests.post(...)`, rất dễ có chỗ **quên gửi khóa**, **quên timeout**, **quên bắt lỗi mạng**. Một hàm chung bảo đảm **mọi** yêu cầu đều: ① gửi `X-API-Key`, ② có timeout, ③ bắt lỗi mạng, ④ đọc lý do lỗi, ⑤ ghi thông báo tiếng Việt.

### Từng phần
| Phần | Ý nghĩa |
|---|---|
| `requests.request(method, url, ...)` | Hàm tổng quát; `requests.get(...)`, `requests.post(...)` thực chất gọi hàm này |
| `f"{PI_URL}{path}"` | Ghép địa chỉ gốc + đường dẫn: `http://<pi>:5000` + `/api/menu/delete` |
| `headers=_AUTH_HEADERS` | Mọi yêu cầu (kể cả GET) đều mang khóa |
| `timeout=timeout` | **Bắt buộc với máy khách mạng**: `requests` **mặc định chờ vô hạn**. Không có timeout, Pi treo → phần mềm quản trị treo theo |
| `**kwargs` | Nhận và chuyển tiếp mọi tham số khác cho `requests`: `json=`, `data=`, `files=`, `params=` |
| `except requests.RequestException` | Lớp cha của **mọi lỗi mạng** của `requests`: không kết nối được, hết thời gian chờ, lỗi DNS... |
| `res.json()` | Giải mã thân phản hồi JSON → dict/list; không phải JSON (ví dụ trang HTML lỗi) → `ValueError` → `None` |
| `200 <= res.status_code < 300` | **So sánh chuỗi** của Python (tương đương `200 <= x and x < 300`) → mọi mã **2xx** là thành công |
| `data.get("success") is False` | Phòng trường hợp máy chủ trả **200** nhưng thân báo `"success": false` (ví dụ chuyển khoản không phải đơn kiosk ở webhook). Dùng `is False` (đúng là `False`) chứ không `not ...` (vì thiếu trường → `None` không nên coi là thất bại) |
| `self._last_error` | Lưu thông báo của **lần gọi gần nhất**; thành công thì xóa |
| `return ok, data` | Trả **tuple 2 phần tử**; nơi gọi viết `ok, data = self._request(...)` hoặc `ok, _ = ...` (`_` = "không dùng tới") |

### Hai kiểu lỗi mạng — thử thật
| Tình huống | Hiện tượng | Thời gian |
|---|---|---|
| Pi bật nhưng **API không chạy** (cổng đóng) | Hệ điều hành trả "Connection refused" ngay | **0,00 s** |
| Pi **tắt / sai IP / mất Wi-Fi** | Gói tin đi mà không ai trả lời → chờ tới hết `timeout` | **3,01 s** (timeout=3) |

Cả hai đều thành `network_error` → *"Không kết nối được máy chủ trên Pi."*

### Các mức timeout trong file
| Mức | Dùng cho | Lý do |
|---|---|---|
| **3 s** | Đọc dữ liệu (GET): thống kê, thực đơn, đơn, kho | Đọc nhanh; giao diện không nên chờ lâu |
| **5 s** (mặc định) | Ghi (POST): thêm/sửa/xóa, đăng nhập | Ghi CSDL + tải ảnh lên cần thêm thời gian |
| **10 s** | `forgotPassword` | Máy chủ phải **gửi email qua Gmail** trước khi trả lời |

### Cách QML hiện thông báo lỗi
Các slot ghi chỉ trả `True/False`. Khi `False`, QML gọi thêm `adminBackend.lastErrorMessage()` (được dùng ở **11 chỗ** trong QML):
```python
    @Slot(result=str)
    def lastErrorMessage(self):
        return self._last_error or "Thao tác thất bại, vui lòng thử lại."
```
→ Giữ chữ ký slot đơn giản (`bool`) mà vẫn có thông báo chi tiết.

---

## 4. Lớp `AdminBridge` và khởi tạo (L66–80)

```python
class AdminBridge(QObject):
    statsReady = Signal(str)

    def __init__(self):
        super().__init__()
        self.current_time_filter = "month"
        self.current_data = {}
        self.report_period = "month"
        self._last_error = ""
        self._stats_busy = threading.Lock()
        if not ADMIN_API_KEY:
            print("--> [CẢNH BÁO BẢO MẬT] ADMIN_API_KEY chưa cấu hình trong .env — ...")
```
- `statsReady = Signal(str)` → tín hiệu mang **chuỗi JSON** số liệu trang chủ (phần 5).
- `main_admin.py` đăng ký đối tượng cho QML với tên **`adminBackend`**: `engine.rootContext().setContextProperty("adminBackend", backend)`.
- `current_data` → lưu số liệu gần nhất để **xuất CSV**.
- `_stats_busy` → khóa chống tải thống kê chồng nhau.

---

## 5. Thống kê trang chủ — gọi đồng bộ và gọi ở luồng nền (L122–158)

### 5.1 Tải số liệu
```python
    def _load_stats(self):
        ok, data = self._request("GET", "/api/stats",
                                 params={"period": self.current_time_filter}, timeout=3)
        if not ok or not isinstance(data, dict):
            return "{}"
        self.current_data = data
        return json.dumps(data)

    @Slot(result=str)
    def fetchStats(self):
        return self._load_stats()
```
- `params={"period": "month"}` → `requests` tự tạo **query string** `?period=month` (và tự mã hóa ký tự đặc biệt).
- Trả **chuỗi JSON** cho QML (QML dùng `JSON.parse(...)`). Lỗi → `"{}"` → giao diện hiện số 0 thay vì vỡ.
- **Bài học đã sửa** (chú thích L123–124): trước đây phần mềm tự lấy danh sách đơn (tối đa 500) rồi **tự cộng** → doanh thu thiếu. Nay máy chủ **tính bằng SQL**, máy khách chỉ hiển thị.

### 5.2 `requestStats()` — tải ở luồng nền
```python
    @Slot()
    def requestStats(self):
        if not self._stats_busy.acquire(blocking=False):
            return   # lần tải trước chưa xong

        def worker():
            try:
                self.statsReady.emit(self._load_stats())
            finally:
                self._stats_busy.release()

        threading.Thread(target=worker, daemon=True).start()
```

**Vấn đề:** trang chủ tự làm mới **mỗi 5 s** (`Timer { interval: 5000 }` trong `DashboardContent.qml`). Nếu gọi `fetchStats()` **trên luồng giao diện** và Pi mất mạng → mỗi lần **đứng giao diện 3 s** (chờ timeout) → phần mềm gần như không dùng được.

**Giải pháp:**
```
QML Timer (5 s) ─► requestStats()  ── trả về NGAY, giao diện không chờ
                       └─► luồng nền: _load_stats() (có thể chờ tới 3 s)
                                └─► statsReady.emit(json) ──► QML onStatsReady(jsonStr) → vẽ lại
```
| Cú pháp | Ý nghĩa |
|---|---|
| `acquire(blocking=False)` | **Thử** lấy khóa, **không chờ**: lấy được → `True`; đang có người giữ → `False` ngay |
| `return` khi `False` | Lần tải trước **chưa xong** (Pi chậm) → bỏ qua lần này, không chồng thêm luồng. Thử thật: lần 1 `True`, lần 2 `False` |
| Hàm lồng `worker` | ***Closure***: dùng được `self` của hàm ngoài |
| `try/finally: release()` | Luôn **trả khóa** kể cả khi lỗi → không bị kẹt vĩnh viễn |
| `daemon=True` | Đóng phần mềm thì luồng tự dừng |
| `statsReady.emit(...)` từ luồng nền | Qt tự **chuyển** về luồng giao diện (*queued connection*) → hàm QML chạy an toàn |

Bên QML (`DashboardContent.qml` L39–42):
```qml
Connections {
    target: (typeof adminBackend !== "undefined") ? adminBackend : null
    function onStatsReady(jsonStr) { updateDashboard(jsonStr) }
}
```

**Vì sao không làm vậy cho mọi thao tác?** Thao tác **do người bấm** (lưu món, xóa nguyên liệu...) cần **biết kết quả ngay** để hiện thông báo; người dùng cũng chờ được vài giây. Chỉ việc **tự động lặp lại** (trang chủ 5 s/lần) mới bắt buộc chạy nền.

### 5.3 Đổi khoảng thời gian
```python
    @Slot(int, result=str)
    def updateTimeFilter(self, index):
        if index == 0: self.current_time_filter = "week"
        elif index == 1: self.current_time_filter = "day"
        elif index == 2: self.current_time_filter = "month"
        return self.fetchStats()
```
- QML gửi **vị trí** mục được chọn trong hộp chọn (0, 1, 2) → đổi thành tên mà API hiểu. Thứ tự này phải **khớp** thứ tự các mục trên giao diện.
- `if ...: lệnh` viết trên **một dòng** — hợp lệ khi thân chỉ có một lệnh.

---

## 6. Thực đơn và tải ảnh lên (L160–204)

### 6.1 `getMenu()`
```python
        ok, items = self._request("GET", "/api/menu", timeout=3)
        if not ok or not isinstance(items, list):
            return []
        for item in items:
            image_value = item.get('image', '')
            is_placeholder = (not image_value
                               or image_value == 'default_food.png'
                               or image_value.startswith('#'))
            if not is_placeholder:
                item['image_display'] = f"{PI_URL}/images/{image_value}"
            else:
                item['image_display'] = ""
        return items
```
- Ảnh món **nằm trên Pi** → hiển thị bằng **URL HTTP** `http://<pi>:5000/images/<tên>` (route công khai `serve_image` của API). Thử thật: `http://127.0.0.1:5987/images/banh_mi_sai_gon_dac_biet.jpg`.
- So sánh với kiosk: kiosk chạy **trên Pi** nên dùng `file:///` (đọc thẳng ổ đĩa); laptop phải qua HTTP.
- Ảnh mặc định hoặc mã màu `#FFD700` (API trả khi món không có ảnh) → `""` → QML vẽ ô màu.
- Điều kiện nhiều dòng trong `( ... )` nối bằng `or`.

### 6.2 Thêm / sửa món — gửi **form + file**
```python
    def _post_product(self, path, data, image_path):
        image_path = _strip_file_url(image_path)
        if image_path and os.path.exists(image_path):
            with open(image_path, 'rb') as f:
                ok, _ = self._request("POST", path, data=data, files={'image_file': f})
        else:
            ok, _ = self._request("POST", path, data=data)
        return ok

    @Slot(str, str, float, str, result=bool)
    def addProduct(self, name, category, price, image_path):
        data = {"name": name, "category": category, "price": int(round(price))}
        return self._post_product("/api/menu/add", data, image_path)
```

**Ba cách gửi dữ liệu với `requests`** (cả ba đều có trong file):
| Tham số | Thân yêu cầu | `Content-Type` | Máy chủ đọc bằng | Dùng ở |
|---|---|---|---|---|
| `params=` | (không có — nằm trên URL) | — | `request.args` | Mọi GET |
| `json=` | `{"id": 5}` | `application/json` | `request.get_json()` | Đa số POST |
| `data=` + `files=` | Nhiều phần (*multipart*) | `multipart/form-data` | `request.form` + `request.files` | Thêm/sửa món (có ảnh) |

- **Vì sao thêm món không dùng JSON?** JSON là **văn bản**, không chứa được **file nhị phân** (ảnh) gọn gàng. `multipart/form-data` chia thân yêu cầu thành **nhiều phần**: mỗi trường văn bản một phần, file ảnh một phần riêng.
- `open(image_path, 'rb')` → mở file chế độ **đọc nhị phân** (`b`); `with` → tự đóng file sau khi gửi.
- `files={'image_file': f}` → tên phần `image_file` **phải khớp** `request.files.get('image_file')` phía máy chủ.
- Không chọn ảnh / file không tồn tại → chỉ gửi `data` → khi **sửa** thì máy chủ giữ ảnh cũ.
- `int(round(price))` → QML gửi số thực `36000.0`; gửi `"36000"` cho gọn và đúng kiểu giá nguyên (chú thích L187).

Thử thật: thêm món kèm `mon_thu.png` → `True`; máy chủ lưu thành `b4ea64d4_mon_thu.png` (tiền tố ngẫu nhiên — xem `api_server.md` §5.5).

### 6.3 Xóa, bật/tắt món
```python
    def deleteProduct(self, pid):
        ok, _ = self._request("POST", "/api/menu/delete", json={"id": pid})
        return ok

    def toggleProductStatus(self, pid, status):
        ok, _ = self._request("POST", "/api/menu/toggle", json={"id": pid, "status": status})
        return ok
```
- `json={...}` → `requests` tự `json.dumps` và đặt header `Content-Type: application/json`.
- `status` kiểu `bool` Python → JSON `true/false` → máy chủ `_to_bool` nhận đúng.
- Thử thật: xóa id 9999 → `False`, `lastErrorMessage()` = *"Không tìm thấy món (có thể đã bị xóa)."*

---

## 7. Báo cáo, đơn hàng, xuất CSV (L206–252)

### 7.1 Báo cáo và danh sách đơn
```python
    def updateReportPeriod(self, index):
        periods = ["day", "week", "month", "year"]
        if 0 <= index < len(periods):
            self.report_period = periods[index]
        return self.getReportData()

    def fetchOrders(self, status_filter, keyword, page, limit):
        ok, data = self._request("GET", "/api/orders", params={
            "status_filter": status_filter, "keyword": keyword, "page": page, "limit": limit
        }, timeout=3)
        if ok and isinstance(data, dict):
            return json.dumps(data)
        return json.dumps({"data": [], "totalPages": 1, "currentPage": page, "totalItems": 0})
```
- `updateReportPeriod`: tra **danh sách theo vị trí**, kiểm tra **chỉ số hợp lệ** trước (tránh `IndexError`).
- `fetchOrders`: tham số **phân trang** đi trên URL (`?page=2&limit=20...`); máy chủ tính `offset`.
- Lỗi → trả **dữ liệu rỗng đúng cấu trúc** (`data`, `totalPages`...) → giao diện vẫn vẽ được bảng trống thay vì lỗi JavaScript.

### 7.2 `exportReport()` — xuất CSV
```python
        report_dir = os.path.join(_project_root, "data", "reports")
        os.makedirs(report_dir, exist_ok=True)
        filename = f"BaoCao_Kiosk_{datetime.now().strftime('%Y%m%d_%H%M%S')}.csv"
        with open(full_path, mode='w', newline='', encoding='utf-8-sig') as file:
            writer = csv.writer(file)
            writer.writerow(["THỜI GIAN XUẤT", datetime.now().strftime("%d/%m/%Y %H:%M")])
            writer.writerow(["TỔNG DOANH THU", self.current_data.get('revenue', 0)])
            ...
            for point in self.current_data.get('chart', []):
                writer.writerow([point['day'], point['total']])
```
| Điểm | Ý nghĩa |
|---|---|
| Ghi ở **laptop** (`data/reports/`) | Dùng số liệu **đã tải** (`current_data`), không cần API mới |
| Tên file theo thời điểm | Không ghi đè báo cáo cũ |
| `newline=''` | Bắt buộc với module `csv` — tránh dòng trống xen giữa trên Windows |
| `encoding='utf-8-sig'` | UTF-8 **kèm dấu BOM** ở đầu file → **Excel** nhận ra UTF-8 và hiện đúng tiếng Việt (UTF-8 thường thì Excel hay hiện sai dấu) |
| `csv.writer(...).writerow([...])` | Mỗi list = một dòng; tự thêm dấu phẩy, tự bọc ngoặc kép khi giá trị có dấu phẩy |
| Trả đường dẫn / `""` | QML hiện "Đã xuất tại ..." hoặc báo lỗi |

---

## 8. Cửa hàng, đăng nhập, mật khẩu (L254–307)

```python
    @Slot(str, result=bool)
    def login(self, password):
        ok, _ = self._request("POST", "/api/auth/login", json={"password": password})
        return ok
```
- Mật khẩu gửi trong **thân JSON** (không bao giờ trên URL — URL hay bị ghi vào nhật ký máy chủ, lịch sử).
- Thử thật: sai lần 6 → `False`; thông báo *"Sai quá nhiều lần, vui lòng thử lại sau 300 giây."* — số 300 lấy từ trường `retry_after` của phản hồi 429, điền qua `template.format(**data)`.
- ⚠ Đồ án **chưa dùng HTTPS** (đã quyết định để hướng phát triển) → mật khẩu đi **dạng thường** trong mạng Wi-Fi cửa hàng. Chấp nhận được trong mạng nội bộ tin cậy; triển khai rộng thì cần HTTPS.

```python
    @Slot(result=str)
    def forgotPassword(self):
        ok, data = self._request("POST", "/api/auth/forgot_password", timeout=10)
        if isinstance(data, dict):
            return json.dumps(data)
        return json.dumps({"success": False, "reason": "network_error"})
```
- `timeout=10` vì máy chủ phải gửi email trước.
- Trả **nguyên phản hồi** (kể cả khi thất bại) dạng JSON → QML tự đọc `reason` (`otp_cooldown`, `no_recovery_email`...) và `masked_email`.
- Không có phản hồi (lỗi mạng) → **tự tạo** phản hồi cùng định dạng với `reason: network_error` → QML chỉ cần **một** cách xử lý.

`resetPasswordWithOtp`, `getRecoveryEmailStatus`, `updateRecoveryEmail`, `changePassword` cùng mẫu. `changePassword` trả **`int`** (1/0) thay vì `bool` — chỉ là cách QML phía đó được viết.

---

## 9. Kho và công thức (L309–360)

| Slot | API | Trả QML |
|---|---|---|
| `getInventory()` | `GET /api/inventory` | `list` (lỗi → `[]`) |
| `addIngredient(name, unit, qty, threshold)` | `POST /api/inventory/add` | `bool` |
| `updateIngredient(iid, name, unit, threshold)` | `POST /api/inventory/update` | `bool` |
| `deleteIngredient(iid)` | `POST /api/inventory/delete` | `bool` — `in_use` → *"Nguyên liệu đang có trong công thức món – hãy sửa công thức trước."* |
| `restockIngredient(iid, add_quantity)` | `POST /api/inventory/restock` | chuỗi JSON (có nguyên liệu sau cập nhật) |
| `getRecipe(product_id)` | `GET /api/recipe?product_id=` | `list` |
| `saveRecipe(product_id, json_str)` | `POST /api/recipe/save` | `bool` |

```python
    @Slot(int, str, result=bool)
    def saveRecipe(self, product_id, recipe_items_json):
        try:
            items = json.loads(recipe_items_json)
        except ValueError:
            self._last_error = _reason_message("invalid_items")
            return False
        ok, _ = self._request("POST", "/api/recipe/save", json={"product_id": product_id, "items": items})
        return ok
```
- QML gửi danh sách nguyên liệu dạng **chuỗi JSON** (truyền mảng đối tượng phức tạp từ QML sang Python qua chuỗi là cách đơn giản, rõ kiểu).
- **Kiểm tra tại máy khách trước** → JSON hỏng thì **không gửi** yêu cầu vô ích. Thử thật: `saveRecipe(1, "khong-phai-json")` → `False`, *"Dữ liệu nhập không hợp lệ."*, máy chủ **không nhận** yêu cầu nào.
- Phần kiểm tra **thật sự** (trùng nguyên liệu, số dương, khóa ngoại) vẫn do **máy chủ** làm — máy khách có thể bị sửa hoặc thay thế, máy chủ thì không.

**Kiểu tham số slot** `@Slot(str, str, float, float, result=bool)` → QML truyền số nguyên `5` vẫn được Qt đổi sang `float`.

---

## 10. Tổng kết

### 10.1 Máy khách HTTP tốt cần gì — đối chiếu với file
| Yêu cầu | Trong file |
|---|---|
| Một hàm gọi chung | `_request()` |
| **Luôn có timeout** | 3 s đọc, 5 s ghi, 10 s gửi email |
| Bắt lỗi mạng | `except requests.RequestException` |
| Gửi xác thực mọi yêu cầu | `_AUTH_HEADERS` |
| Đọc mã trạng thái **và** thân phản hồi | `200 <= code < 300` + `success is False` + `reason` |
| Dịch mã lỗi cho người dùng | `_REASON_MESSAGES` |
| Không làm đứng giao diện với việc lặp lại | `requestStats` + luồng nền + `statsReady` |
| Không gọi chồng | `acquire(blocking=False)` |
| Lỗi thì trả dữ liệu rỗng **đúng cấu trúc** | `"{}"`, `[]`, `{"data": [], ...}` |
| Cấu hình không viết cứng | `API_SERVER_URL`, `ADMIN_API_KEY` trong `.env` |

### 10.2 Ghi chú (không gây lỗi hiện tại)
| Điểm | Ghi chú |
|---|---|
| Thao tác do người bấm gọi **đồng bộ** trên luồng giao diện | Pi mất mạng thì giao diện đứng tối đa 3–5 s (10 s với quên mật khẩu) rồi hiện thông báo — có chủ đích để biết kết quả ngay |
| `_last_error` dùng chung giữa luồng nền (thống kê) và luồng giao diện | Hiếm khi: thống kê lỗi đúng lúc người dùng vừa thao tác có thể ghi đè câu thông báo |
| Chưa có HTTPS | Khóa API và mật khẩu đi dạng thường trong mạng nội bộ |
| Kiosk dùng PyQt6, quản trị dùng PySide6 | Hai thư viện cùng bộ Qt 6, cú pháp signal/slot gần như giống nhau |

---

## 11. Câu hỏi tự kiểm tra
1. Vì sao phần mềm quản trị không import `backend.database` như kiosk?
2. Kể 5 việc `_request()` bảo đảm cho mọi lần gọi API.
3. `requests` mặc định có timeout không? Bỏ `timeout=` thì điều gì xảy ra khi Pi bị treo?
4. Vì sao "máy chủ tắt" báo lỗi ngay (0 s) còn "sai IP" phải chờ 3 s?
5. Vì sao kiểm tra thêm `data.get("success") is False` khi mã đã là 200? Vì sao dùng `is False` thay vì `not data.get("success")`?
6. `params=`, `json=`, `data=`+`files=` khác nhau thế nào? Vì sao thêm món không dùng `json=`?
7. Giải thích `template.format(**data)` với phản hồi `{"reason": "too_many_attempts", "retry_after": 300}`.
8. Vẽ luồng `requestStats` → `statsReady`. Nếu lần tải trước chưa xong thì sao?
9. Vì sao chỉ thống kê trang chủ chạy nền, còn "Lưu món" thì không?
10. Vì sao ghi CSV bằng `utf-8-sig`?
11. `saveRecipe` đã kiểm tra JSON ở máy khách, sao máy chủ vẫn phải kiểm tra lại?
12. Ảnh món trên phần mềm quản trị lấy bằng URL gì? Vì sao kiosk lại dùng `file:///`?

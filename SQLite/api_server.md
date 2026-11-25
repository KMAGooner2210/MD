# Giải thích chi tiết `servers/api_server.py`

> File: `Smart_Kiosk_Backup/servers/api_server.py` — 848 dòng, bản trên GitHub `main`.
> Thí nghiệm chạy thật bằng **Flask test client** (gọi API ngay trong Python, không mở cổng mạng) với **CSDL tạm**, **khóa thử** `khoa-admin-thu` / `khoa-bep-thu`, MQTT trỏ vào địa chỉ không tồn tại, SMTP để trống → không đụng dữ liệu thật, không gửi email, không phát tin.
> Kiểm thử liên quan: `tests/test_api_hardening.py` (18 test).

---

## 0. Bức tranh tổng thể (đọc trước)

### 0.1 File này là gì?
**Smart Kiosk API** — máy chủ **HTTP** viết bằng **Flask**, chạy trên Raspberry Pi ở **cổng 5000**. Đây là **cửa duy nhất** để các máy khác đọc/ghi CSDL `kiosk.db` (vì SQLite là file, máy khác không mở trực tiếp được).

```
 Phần mềm quản trị (laptop) ──┐  X-API-Key = ADMIN_API_KEY   (toàn quyền)
 Màn hình bếp ESP32 ──────────┤  X-API-Key = KITCHEN_API_KEY (chỉ việc bếp)
 Kitchen MCP (trợ lý giọng nói)┘  X-API-Key = KITCHEN_API_KEY
                │ HTTP (JSON)
                ▼
 ┌──────── api_server.py (Flask + Waitress, 8 luồng) ────────┐
 │ kiểm tra khóa → kiểm tra đầu vào → gọi backend/database.py │──► kiosk.db
 │ lỗi gì cũng trả JSON {"success": false, "reason": ...}     │
 │ đổi kho / xong đơn → MQTT có chữ ký → màn hình bếp          │──► broker MQTT
 └────────────────────────────────────────────────────────────┘
```

### 0.2 Kiến thức HTTP cần nhớ
| Khái niệm | Ý nghĩa |
|---|---|
| **Phương thức** `GET` | **Đọc** dữ liệu, không thay đổi gì; tham số trên URL `?period=day` |
| **Phương thức** `POST` | **Gửi** dữ liệu để thay đổi; dữ liệu nằm trong **thân** (body), thường là JSON |
| **Header** | Thông tin kèm theo yêu cầu, ví dụ `X-API-Key: ...`, `Content-Type: application/json` |
| **Mã trạng thái** | 2xx thành công · 4xx **lỗi phía gọi** · 5xx **lỗi phía máy chủ** |

Các mã dùng trong file:
| Mã | Nghĩa | Ví dụ trong đồ án |
|---|---|---|
| 200 | OK | Thành công |
| 400 | Bad Request — dữ liệu gửi lên sai | `id="abc"`, số lượng âm |
| 401 | Unauthorized — **chưa xác thực** (không có/sai khóa) | Gọi `/api/stats` không có khóa |
| 403 | Forbidden — **đã xác thực nhưng không đủ quyền** | Khóa bếp gọi `/api/stats` |
| 404 | Not Found — không có | Xóa món id 9999 |
| 405 | Method Not Allowed — sai phương thức | `DELETE /api/menu` |
| 409 | Conflict — **xung đột trạng thái** | Báo xong đơn đã xong; xóa nguyên liệu đang dùng |
| 413 | Payload Too Large | Ảnh > 8 MB |
| 429 | Too Many Requests | Đăng nhập sai quá 5 lần |
| 500 | Internal Server Error | Lỗi bất ngờ trong mã |
| 502 | Bad Gateway — dịch vụ phía sau lỗi | Gửi email OTP thất bại |
| 503 | Service Unavailable | Chưa cấu hình khóa API |

### 0.3 Kết quả chạy thử (tóm tắt)
```
GET /api/stats không khóa                        -> 401 unauthorized
GET /api/stats khóa BẾP                          -> 403 forbidden
GET /api/stats?period=day khóa ADMIN             -> 200 {"chart":[],"orders":0,"revenue":0}
GET /api/orders/pending khóa BẾP                 -> 200
POST /api/menu/delete body không phải JSON       -> 400 invalid_id
POST /api/menu/delete id=9999                    -> 404 product_not_found
POST /api/inventory/add quantity='nan'           -> 400 invalid_quantity
POST /api/recipe/save trùng nguyên liệu          -> 400 duplicate_ingredient
POST /api/recipe/save nguyên liệu 999            -> 409 constraint_violation   (khóa ngoại CSDL)
POST /api/inventory/delete nguyên liệu đang dùng -> 409 ingredient_in_use
GET /api/khong-co                                -> 404 not_found   (JSON, không phải trang HTML)
DELETE /api/menu                                 -> 405 method_not_allowed
GET /images/..%2F.env (dò đường dẫn)             -> 404 not_found
POST /api/orders/done lần 1 / lần 2              -> 200 / 409
POST /api/menu/add file virus.exe                -> 400 invalid_image
POST /api/auth/login sai lần 1…5 / lần 6         -> 401 / 429 too_many_attempts, retry_after=300
```

### 0.4 Bản đồ file
| Dòng | Phần |
|---|---|
| 1–44 | Import, tạo `app`, giới hạn tải lên, khóa API |
| 46–79 | Gửi email OTP (SMTP) |
| 82–150 | MQTT: phát tồn kho, phát "đơn đã xong" |
| 153–198 | **Xử lý lỗi tập trung** → luôn trả JSON |
| 201–265 | **Hàm kiểm tra đầu vào** + lưu ảnh tải lên |
| 268–369 | **Xác thực, phân quyền**, giới hạn đăng nhập |
| 380–825 | **Các endpoint** (26 route) |
| 828–848 | `main()`: chạy bằng Waitress |

---

## 1. Import và khởi tạo (L1–44)

```python
import sys, os, hmac, json, math, smtplib, sqlite3, threading, time, traceback, uuid
from email.mime.text import MIMEText
from functools import wraps
from werkzeug.exceptions import HTTPException
from werkzeug.utils import secure_filename
from flask import Flask, request, jsonify, send_from_directory
import paho.mqtt.client as mqtt
from dotenv import load_dotenv
```
| Thứ | Dùng để |
|---|---|
| `hmac` | `compare_digest` so sánh khóa API thời gian không đổi |
| `math` | `isfinite` (chặn `nan`, `inf`), `ceil` |
| `smtplib`, `MIMEText` | Gửi email OTP |
| `sqlite3` | **Chỉ để bắt** lỗi `sqlite3.IntegrityError` (file không tự viết SQL) |
| `threading` | Khóa cho bộ đếm đăng nhập sai |
| `traceback` | In chi tiết lỗi vào nhật ký |
| `uuid` | Tiền tố ngẫu nhiên cho tên ảnh tải lên |
| `functools.wraps` | Viết decorator kiểm tra khóa |
| `HTTPException` | Lớp cha của các lỗi HTTP do Flask sinh (404, 405, 413...) |
| `secure_filename` | Làm sạch tên file tải lên |
| `Flask, request, jsonify, send_from_directory` | Tạo ứng dụng, đọc yêu cầu, trả JSON, trả file ảnh |

```python
current_dir = os.path.dirname(os.path.abspath(__file__))
project_root = os.path.dirname(current_dir)
if project_root not in sys.path:
    sys.path.append(project_root)

import backend.database as db
from backend.mqtt_security import configure_client, sign_message

load_dotenv(os.path.join(project_root, ".env"))

app = Flask(__name__)
app.config["MAX_CONTENT_LENGTH"] = 8 * 1024 * 1024
```
- `import backend.database as db` → mọi thao tác CSDL đi qua `db.*` (28 lời gọi) — API **không tự viết SQL**.
- `load_dotenv(...)` **không có** `override=True` (khác `bridge.py`) → biến môi trường **đặt sẵn** (ví dụ bởi systemd hay khi chạy test) được **ưu tiên** hơn `.env`. Nhờ vậy thí nghiệm ở trên đặt khóa thử mà không bị `.env` ghi đè.
- `Flask(__name__)` → tạo **ứng dụng**; `__name__` giúp Flask biết vị trí module.
- `MAX_CONTENT_LENGTH = 8 MB` → yêu cầu lớn hơn bị Flask **từ chối 413** trước khi đọc hết → chặn gửi file khổng lồ làm đầy thẻ nhớ Pi.

```python
UPLOAD_FOLDER = os.path.join(project_root, 'assets', 'images')
os.makedirs(UPLOAD_FOLDER, exist_ok=True)
ALLOWED_IMAGE_EXTENSIONS = {".jpg", ".jpeg", ".png", ".webp", ".gif"}

ADMIN_API_KEY = os.environ.get("ADMIN_API_KEY", "")
KITCHEN_API_KEY = os.environ.get("KITCHEN_API_KEY", "")
```
- **Hai khóa, hai vai trò** (nguyên tắc ***quyền tối thiểu*** — *least privilege*):
  - `ADMIN_API_KEY` — phần mềm quản trị: **mọi** thao tác.
  - `KITCHEN_API_KEY` — màn hình bếp, Kitchen MCP: chỉ **đọc** đơn/kho/công thức và **báo xong đơn**.
  → ESP32 bị lấy mất (khóa nằm trong firmware) thì kẻ gian **không** sửa được thực đơn, giá, mật khẩu.

---

## 2. Gửi email OTP (L46–79)

```python
def send_otp_email(to_email, otp_code):
    if not SMTP_USER or not SMTP_PASSWORD:
        return False
    msg = MIMEText(body, "plain", "utf-8")
    msg["Subject"] = subject
    msg["From"] = SMTP_USER
    msg["To"] = to_email
    try:
        with smtplib.SMTP(SMTP_HOST, SMTP_PORT, timeout=10) as server:
            server.starttls()
            server.login(SMTP_USER, SMTP_PASSWORD)
            server.sendmail(SMTP_USER, [to_email], msg.as_string())
        return True
    except Exception as e:
        return False
```
| Bước | Ý nghĩa |
|---|---|
| `MIMEText(body, "plain", "utf-8")` | Tạo email **văn bản thường**, mã hóa UTF-8 (giữ tiếng Việt) |
| `msg["Subject"] = ...` | Gán tiêu đề, người gửi, người nhận như dict |
| `smtplib.SMTP(host, 587, timeout=10)` | Kết nối máy chủ thư Gmail cổng **587** |
| `with ... as server:` | Tự đóng kết nối khi xong |
| `starttls()` | **Nâng cấp kết nối thành mã hóa TLS** trước khi gửi mật khẩu |
| `login(...)` | Đăng nhập bằng **App Password** 16 ký tự (chú thích L47–49: Gmail không cho dùng mật khẩu thường) |
| `sendmail(from, [to], text)` | Gửi |
| Trả `True/False` | Endpoint đổi thành mã HTTP (502 khi thất bại) |

---

## 3. MQTT phía API (L82–150)

```python
try:
    _mqtt_client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
except AttributeError:
    _mqtt_client = mqtt.Client()

configure_client(_mqtt_client, role="API")

try:
    _mqtt_client.connect(MQTT_BROKER, MQTT_PORT, 60)
    _mqtt_client.loop_start()
except Exception as e:
    print(f"--> [LOI MQTT] api_server khong the ket noi Broker: {e}")
```
- Chạy **ngay lúc import module** (cấp module, ngoài hàm).
- Dùng `connect` (**chặn**, chờ kết nối xong) chứ không phải `connect_async` như kiosk. ⚠ Nếu lúc API khởi động broker không tới được → ném lỗi → **`loop_start()` không được gọi** → các tin MQTT sau đó không được gửi cho tới khi khởi động lại API. Không ảnh hưởng dữ liệu (màn hình bếp vẫn tự đồng bộ qua API mỗi 30 s); chỉ mất phần "cập nhật tức thì".
- API **chỉ gửi**, không đăng ký nhận chủ đề nào.

```python
def publish_inventory_snapshot():
    ingredients = db.get_all_ingredients()
    if not MQTT_SHARED_SECRET: return
    payload = {"version": 1, "event": "inventory.snapshot", "items": [...]}
    _mqtt_client.publish(MQTT_TOPIC_INVENTORY, sign_message(payload, MQTT_SHARED_SECRET), retain=True)

def publish_order_done(order_code):
    if not MQTT_SHARED_SECRET: return
    _mqtt_client.publish(MQTT_TOPIC_ORDER_DONE,
                         sign_message({"version": 1, "event": "order.done", "order_id": order_code}, MQTT_SHARED_SECRET),
                         retain=False)
```
- **Tồn kho** gửi sau mỗi lần thêm/sửa/xóa/nhập kho từ phần mềm quản trị — cùng định dạng với kiosk (`retain=True`: màn hình bếp vừa bật lên đã có số mới nhất).
- **`order.done`** gửi khi đơn được báo xong **ở nơi khác** (ví dụ trợ lý giọng nói) → màn hình bếp **gỡ thẻ đơn ngay**, không phải chờ lần đồng bộ 30 s. `retain=False` vì chỉ có ý nghĩa tức thời.
- Luôn **ký HMAC**; thiếu khóa → không gửi.
- Lỗi bị bắt hết → thao tác chính (đã ghi CSDL thành công) **không bị hỏng** vì MQTT lỗi.

---

## 4. Xử lý lỗi tập trung — luôn trả JSON (L153–198)

### 4.1 Lớp lỗi nghiệp vụ `ApiError`
```python
class ApiError(Exception):
    def __init__(self, reason, status=400):
        super().__init__(reason)
        self.reason = reason
        self.status = status

@app.errorhandler(ApiError)
def _handle_api_error(error):
    return jsonify({"success": False, "reason": error.reason}), error.status
```
- **Lớp lỗi riêng** mang 2 thông tin: **lý do** (mã ngắn) và **mã HTTP** (mặc định 400).
- `@app.errorhandler(ApiError)` → khi **bất kỳ** endpoint nào `raise ApiError(...)`, Flask gọi hàm này để tạo phản hồi.
- **Lợi ích:** endpoint viết **thẳng một dòng** `raise ApiError("product_not_found", 404)` — kể cả từ **sâu bên trong** hàm phụ (`_to_int` ném lỗi thì endpoint dừng ngay) — không phải `if … return jsonify(...), 404` lặp đi lặp lại.
- Hàm Flask trả `(dữ_liệu, mã)` → **tuple** → Flask hiểu phần thứ hai là mã trạng thái.

### 4.2 Lỗi HTTP do Flask sinh ra
```python
_HTTP_REASONS = {400: "bad_request", 404: "not_found", 405: "method_not_allowed",
                 413: "payload_too_large", 415: "unsupported_media_type"}

@app.errorhandler(HTTPException)
def _handle_http_exception(error):
    reason = _HTTP_REASONS.get(error.code, "http_error")
    return jsonify({"success": False, "reason": reason}), error.code
```
- Mặc định Flask trả **trang HTML** cho 404, 405... → ESP32, Kitchen MCP, phần mềm quản trị đọc JSON sẽ **lỗi phân tích**. Bắt `HTTPException` (lớp cha của mọi lỗi HTTP) → **luôn JSON**.
- Thử thật: `GET /api/khong-co` → `404 {"reason":"not_found"}`; `DELETE /api/menu` → `405 {"reason":"method_not_allowed"}`.

### 4.3 Lỗi ràng buộc CSDL → 409
```python
@app.errorhandler(sqlite3.IntegrityError)
def _handle_integrity_error(error):
    print(f"--> [CSDL] Vi phạm ràng buộc: {error}")
    return jsonify({"success": False, "reason": "constraint_violation"}), 409
```
- `database.py` dùng `_open_db()`: lỗi → rollback rồi **ném lên**. Lỗi khóa ngoại/khóa chính (`IntegrityError`) bay lên tới đây → **409 Conflict**.
- Thử thật: lưu công thức với nguyên liệu id 999 → khóa ngoại từ chối → `409 constraint_violation`.
- **Đây là chỗ nối CSDL với HTTP**: ràng buộc trong lược đồ (slide 7) trở thành mã lỗi có nghĩa cho phần mềm quản trị.

### 4.4 Lỗi bất ngờ → 500
```python
@app.errorhandler(Exception)
def _handle_unexpected_error(error):
    if isinstance(error, HTTPException):
        return _handle_http_exception(error)
    traceback.print_exc()
    return jsonify({"success": False, "reason": "internal_error"}), 500
```
- Lưới an toàn cuối cùng: lỗi lập trình bất kỳ.
- **Chi tiết lỗi chỉ in vào nhật ký** (`traceback.print_exc()` → `journalctl`), client chỉ nhận `internal_error` → **không lộ** cấu trúc mã, đường dẫn file, câu SQL cho người ngoài.
- Kiểm tra `HTTPException` trước vì handler `Exception` cũng bắt được chúng.

**Flask chọn handler nào?** Handler **cụ thể nhất** theo cây kế thừa: `ApiError` → `_handle_api_error`; `IntegrityError` → handler 409; các `Exception` khác → handler 500.

---

## 5. Kiểm tra đầu vào (L201–265)

Nguyên tắc: **không tin bất cứ thứ gì client gửi lên**; sai thì trả **400 kèm lý do**, không để lỗi Python lọt ra thành 500.

### 5.1 `_json_body()`
```python
def _json_body():
    data = request.get_json(silent=True)
    return data if isinstance(data, dict) else {}
```
- `silent=True` → body không phải JSON / thiếu `Content-Type` → trả `None` thay vì ném lỗi 400/415 kiểu HTML.
- JSON hợp lệ nhưng là **mảng/số** → cũng đổi thành `{}` → mọi `.get(...)` phía sau an toàn.
- Thử thật: gửi body `"abc"` → không vỡ, đi tiếp tới kiểm tra `id` → `400 invalid_id`.

### 5.2 `_to_int()`
```python
def _to_int(value, reason, minimum=None):
    if value is None or isinstance(value, bool):
        raise ApiError(reason)
    try:
        number = int(float(value)) if isinstance(value, (str, float)) else int(value)
    except (TypeError, ValueError, OverflowError):
        raise ApiError(reason)
    if minimum is not None and number < minimum:
        raise ApiError(reason)
    return number
```
- **Từ chối `bool`**: trong Python, `True` là **số nguyên 1** (`bool` kế thừa `int`) → không chặn thì `{"id": true}` thành `id=1`.
- Chuỗi/số thực → `int(float(...))` (nhận `"5"`, `"5.0"`, `5.0`); kiểu khác → `int(...)`.
- `OverflowError` → `int(float("inf"))`.
- `minimum` → ví dụ id ≥ 1, giá ≥ 0.
- Tham số `reason` → mỗi chỗ gọi tự chọn mã lỗi phù hợp (`invalid_id`, `invalid_price`...).

### 5.3 `_to_float()`
```python
    number = float(value)
    if not math.isfinite(number):
        raise ApiError(reason)
```
- `float("nan")`, `float("inf")` **hợp lệ trong Python** nhưng vô nghĩa với số lượng kho; `nan` còn làm mọi phép so sánh sai (`nan <= 5` là `False`, `nan >= 5` cũng `False`). `math.isfinite` chặn cả hai. Thử thật: `quantity='nan'` → `400 invalid_quantity`.
- `strictly_positive=True` → phải **> 0** (nhập kho, định lượng công thức).

### 5.4 `_to_text()` và `_to_bool()`
```python
def _to_text(value, reason, max_length=100, required=True):
    text = str(value if value is not None else "").strip()
    if (required and not text) or len(text) > max_length:
        raise ApiError(reason)
    return text

def _to_bool(value, reason):
    if isinstance(value, bool): return value
    if value in (0, 1, "0", "1"): return str(value) == "1"
    if isinstance(value, str) and value.lower() in ("true", "false"): return value.lower() == "true"
    raise ApiError(reason)
```
- Giới hạn **độ dài** chuỗi → chặn gửi tên món dài hàng MB.
- `_to_bool` nhận `true/false`, `1/0`, `"true"/"1"`; giá trị khác (`"yes"`, `2`) → lỗi, **không đoán**.

### 5.5 `_save_uploaded_image()` — tải ảnh món lên an toàn
```python
def _save_uploaded_image():
    file = request.files.get('image_file')
    if file is None or not file.filename:
        return None
    safe_name = secure_filename(file.filename)
    extension = os.path.splitext(safe_name)[1].lower()
    if not safe_name or extension not in ALLOWED_IMAGE_EXTENSIONS:
        raise ApiError("invalid_image")
    filename = f"{uuid.uuid4().hex[:8]}_{safe_name}"
    file.save(os.path.join(UPLOAD_FOLDER, filename))
    return filename
```
| Lớp bảo vệ | Chặn gì |
|---|---|
| `secure_filename` | **Tên file độc hại**: thử thật `'../../etc/passwd'` → `'etc_passwd'` (bỏ `../` → không ghi được ra ngoài thư mục ảnh). Bỏ dấu tiếng Việt: `'Bánh mì.jpg'` → `'Banh_mi.jpg'` |
| Danh sách đuôi cho phép | Chỉ `.jpg .jpeg .png .webp .gif`; thử thật `virus.exe` → `400 invalid_image` |
| Tiền tố `uuid4().hex[:8]` | Hai ảnh cùng tên `bac_xiu.jpg` **không ghi đè** nhau → `3f9a1c2e_bac_xiu.jpg` |
| `MAX_CONTENT_LENGTH` | Ảnh > 8 MB → 413 |

- `request.files` → các file trong yêu cầu dạng **form nhiều phần** (`multipart/form-data`); endpoint thêm/sửa món dùng `request.form` (không phải JSON) vì phải gửi kèm file.
- `os.path.splitext("a.JPG")` → `("a", ".JPG")` → `.lower()` → `.jpg`.
- Lưu ý: kiểm tra theo **đuôi file**, không kiểm tra **nội dung** có thật là ảnh. Rủi ro thấp vì file chỉ được trả lại như ảnh tĩnh, không bao giờ được chạy.

---

## 6. Xác thực và phân quyền (L268–369)

### 6.1 So sánh khóa
```python
def _key_matches(provided, expected):
    if not expected:
        return False
    return hmac.compare_digest(str(provided).encode("utf-8"), str(expected).encode("utf-8"))
```
- `if not expected: return False` → khóa **chưa cấu hình** thì **không khóa nào khớp** — tránh trường hợp khóa rỗng `""` khớp với header rỗng `""` (sẽ là **mở cửa cho tất cả**).
- `compare_digest` → so sánh **thời gian không đổi**, chống dò khóa bằng đo thời gian phản hồi.
- `.encode("utf-8")` → `compare_digest` với **chuỗi** chỉ nhận ký tự ASCII; gửi header có ký tự tiếng Việt sẽ làm nó ném `TypeError` → đổi sang `bytes` để luôn so sánh được.

### 6.2 Xác định vai trò
```python
def _caller_role():
    provided = request.headers.get("X-API-Key", "")
    if _key_matches(provided, ADMIN_API_KEY):   return "admin"
    if _key_matches(provided, KITCHEN_API_KEY): return "kitchen"
    return None
```
- Khóa gửi trong **header** `X-API-Key` (tiền tố `X-` = header tự đặt, không thuộc chuẩn HTTP).

### 6.3 Decorator `require_api_key` — chỉ quản trị
```python
def require_api_key(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        if not ADMIN_API_KEY:
            return jsonify({"success": False, "reason": "api_key_not_configured"}), 503
        role = _caller_role()
        if role is None:
            return jsonify({"success": False, "reason": "unauthorized"}), 401
        if role != "admin":
            return jsonify({"success": False, "reason": "forbidden"}), 403
        return f(*args, **kwargs)
    return decorated
```

**Decorator là gì?** Hàm **nhận một hàm, trả về hàm mới bọc quanh nó**. Viết:
```python
@app.route('/api/stats', methods=['GET'])
@require_api_key
def get_stats(): ...
```
tương đương:
```python
get_stats = app.route('/api/stats', methods=['GET'])( require_api_key(get_stats) )
```
→ Mỗi yêu cầu tới `/api/stats` chạy `decorated` **trước**; qua hết kiểm tra mới gọi `get_stats` thật. Decorator gần hàm nhất (`@require_api_key`) được áp dụng **trước**, nên `@app.route` phải ở **trên cùng** (đăng ký hàm **đã được bọc**).

| Cú pháp | Ý nghĩa |
|---|---|
| Hàm lồng `decorated` | ***Closure***: hàm bên trong nhớ biến `f` của hàm bên ngoài |
| `*args, **kwargs` | Nhận **mọi** tham số vị trí và tham số tên → bọc được hàm bất kỳ chữ ký (ví dụ `serve_image(filename)`) |
| `@wraps(f)` | Chép tên, docstring của `f` sang `decorated`. **Bắt buộc với Flask**: Flask dùng tên hàm làm định danh endpoint; không có `wraps`, mọi hàm được bọc đều tên `decorated` → Flask báo lỗi trùng endpoint |

**Ba tầng kiểm tra:**
| Tình huống | Mã | Thử thật |
|---|---|---|
| Máy chủ chưa đặt `ADMIN_API_KEY` | **503** — từ chối tất cả (*fail-closed*) | — |
| Không có khóa / khóa sai | **401** | `GET /api/stats` → 401 |
| Khóa bếp hợp lệ nhưng gọi chức năng quản trị | **403** | Khóa bếp → 403 |

### 6.4 `require_kitchen_or_admin_key` — việc của bếp
```python
        if not ADMIN_API_KEY and not KITCHEN_API_KEY:
            return ..., 503
        if _caller_role() is None:
            return ..., 401
        return f(*args, **kwargs)
```
Chấp nhận **cả hai** khóa. Dùng cho: `GET /api/orders`, `/api/orders/pending`, `/api/inventory`, `/api/recipe`, `POST /api/orders/done`.

### 6.5 Giới hạn đăng nhập sai
```python
LOGIN_MAX_FAILURES = 5
LOGIN_LOCK_SECONDS = 300
_login_failures = {}
_login_guard = threading.Lock()

def _login_retry_after(ip):
    with _login_guard:
        record = _login_failures.get(ip)
        if not record: return 0
        remaining = record["locked_until"] - time.monotonic()
        return int(math.ceil(remaining)) if remaining > 0 else 0

def _record_login_result(ip, success):
    with _login_guard:
        if success:
            _login_failures.pop(ip, None)
            return
        record = _login_failures.setdefault(ip, {"count": 0, "locked_until": 0.0})
        record["count"] += 1
        if record["count"] >= LOGIN_MAX_FAILURES:
            record["count"] = 0
            record["locked_until"] = time.monotonic() + LOGIN_LOCK_SECONDS
```
| Điểm | Giải thích |
|---|---|
| Dict `{ip: {"count", "locked_until"}}` trong **bộ nhớ** | Đơn giản, không cần CSDL; khởi động lại API thì bộ đếm về 0 |
| `threading.Lock()` | Waitress chạy **8 luồng** → 8 yêu cầu đăng nhập đồng thời cùng sửa dict; không khóa thì `count += 1` có thể bị mất lượt đếm |
| `time.monotonic()` | Đồng hồ **chỉ tăng**, không bị ảnh hưởng khi chỉnh giờ hệ thống / đồng bộ NTP (khác `time.time()`) → kẻ gian không "tua giờ" để thoát khóa |
| `setdefault(ip, {...})` | Có rồi thì lấy, chưa có thì tạo và lấy — một dòng |
| `math.ceil` | Làm tròn **lên**: còn 0,3 s thì báo 1 s chứ không báo 0 |
| Đăng nhập đúng → `pop` | Xóa bộ đếm |

Thử thật: sai lần 1–5 → `401 wrong_password`; lần 6 → `429 too_many_attempts, retry_after=300` kèm header `Retry-After: 300`.
→ Với mật khẩu 6 ký tự, dò thử từng mật khẩu chỉ được **5 lần / 5 phút / IP** thay vì hàng nghìn lần mỗi giây.

---

## 7. Các endpoint

### 7.1 Bảng tổng hợp
| Phương thức | Đường dẫn | Quyền | Hàm CSDL | Ghi chú |
|---|---|---|---|---|
| GET | `/api/stats?period=` | admin | `get_stats_data` | Trang chủ quản trị |
| GET | `/api/report?period=` | admin | `admin_get_report_data` | Báo cáo, top món |
| GET | `/api/menu` | **công khai** | `get_all_products` | Thực đơn đang bán |
| GET | `/images/<filename>` | **công khai** | — | Trả file ảnh |
| POST | `/api/menu/add` | admin | `admin_add_product` | **form + file** |
| POST | `/api/menu/update` | admin | `admin_update_product` | form + file; 404 nếu không có |
| POST | `/api/menu/delete` | admin | `admin_delete_product` | 404 nếu không có |
| POST | `/api/menu/toggle` | admin | `admin_toggle_product_status` | Tạm ngưng / mở bán |
| GET | `/api/store/info` | admin | `get_store_info` | |
| POST | `/api/store/update` | admin | `update_store_info` | |
| POST | `/api/auth/login` | công khai | `verify_admin_password` | 5 lần sai → 429 |
| POST | `/api/auth/forgot_password` | công khai | `generate_password_reset_otp` | Gửi OTP qua email |
| POST | `/api/auth/reset_password` | công khai | `verify_and_reset_password` | Tối đa 5 lần nhập sai OTP |
| GET / POST | `/api/auth/recovery_email` | admin | `get/update_recovery_email` | Trả email **đã che** |
| POST | `/api/auth/change_password` | admin | `verify_admin_password`, `update_admin_password` | |
| GET | `/api/orders` | bếp/admin | `admin_get_orders` | Phân trang, lọc |
| GET | `/api/orders/pending` | bếp/admin | `admin_get_kitchen_pending_orders` | Màn hình bếp đồng bộ |
| POST | `/api/orders/done` | bếp/admin | `admin_mark_order_done` | 409 nếu đã xong |
| GET | `/api/inventory` | bếp/admin | `get_all_ingredients` | |
| POST | `/api/inventory/add` · `update` · `delete` · `restock` | admin | `admin_*_ingredient` | Sau đó phát MQTT tồn kho |
| GET | `/api/recipe?product_id=` | bếp/admin | `get_recipe_for_product` | |
| POST | `/api/recipe/save` | admin | `admin_save_recipe` | |

### 7.2 Mẫu chung của một endpoint
```python
@app.route('/api/menu/delete', methods=['POST'])     # ① đường dẫn + phương thức
@require_api_key                                      # ② kiểm tra quyền
def delete_product():
    pid = _to_int(_json_body().get('id'), "invalid_id", minimum=1)   # ③ kiểm tra đầu vào
    if not db.admin_delete_product(pid):                              # ④ gọi CSDL
        raise ApiError("product_not_found", 404)                      # ⑤ lỗi nghiệp vụ
    return jsonify({"success": True})                                 # ⑥ trả JSON
```
- `@app.route(đường_dẫn, methods=[...])` → **định tuyến** (*routing*): URL nào gọi hàm nào. Phương thức không có trong danh sách → 405.
- `jsonify(...)` → đổi dict/list thành **phản hồi JSON** (đặt `Content-Type: application/json`).
- Endpoint **mỏng**: chỉ kiểm tra + gọi `db` + đổi kết quả thành HTTP. **Logic dữ liệu nằm trong `database.py`** → kiosk và API dùng chung, không viết hai lần.

### 7.3 Thống kê (L382–397)
```python
@app.route('/api/stats', methods=['GET'])
@require_api_key
def get_stats():
    period = request.args.get('period', 'month')
    if period not in ('day', 'week', 'month'):
        period = 'month'
    return jsonify(db.get_stats_data(period))
```
- `request.args` → tham số trên URL (`?period=day`).
- **Danh sách trắng**: giá trị lạ → `month`. (Thêm một lớp nữa trước `_PERIOD_FILTERS` trong `database.py`.)
- `/api/report` cho thêm `year`.
- Số liệu **tính bằng SQL** trong CSDL → chỉ gửi kết quả nhỏ (bài học 620 đơn: 10 triệu → 12,4 triệu).

### 7.4 Thực đơn công khai và ảnh (L402–412)
```python
@app.route('/api/menu', methods=['GET'])
def get_menu():
    result = [{"id": p["id"], "name": p["name"], "category": p["category"], "price": p["price"],
               "image": p["image"] if p["image"] else "#FFD700",
               "status": bool(p.get("status", 1))} for p in db.get_all_products()]
    return jsonify(result)

@app.route('/images/<filename>')
def serve_image(filename):
    return send_from_directory(UPLOAD_FOLDER, filename)
```
- **Không cần khóa**: thực đơn và ảnh món là thông tin **công khai** (dán ngoài cửa hàng cũng được). Giảm chỗ phải giữ khóa.
- Chỉ trả 6 trường cần thiết (**không** trả nguyên dòng CSDL).
- `"#FFD700"` (mã màu vàng) khi món không có ảnh → giao diện quản trị vẽ ô màu thay ảnh.
- `<filename>` → **biến trong đường dẫn**: `/images/bac_xiu.jpg` → `filename="bac_xiu.jpg"`.
- `send_from_directory(thư_mục, tên)` → **chặn thoát khỏi thư mục** (path traversal). Thử thật: `/images/..%2F.env` (cố đọc file bí mật `.env`) → `404`. **Không bao giờ** tự viết `open(os.path.join(UPLOAD_FOLDER, filename))`.

### 7.5 Thêm / sửa món — dữ liệu dạng form (L415–439)
```python
def _product_form_fields():
    name = _to_text(request.form.get('name'), "invalid_name")
    category = _to_text(request.form.get('category'), "invalid_category", max_length=20)
    price = _to_int(request.form.get('price'), "invalid_price", minimum=0)
    return name, category, price

def add_product():
    name, category, price = _product_form_fields()
    image_filename = _save_uploaded_image() or 'default_food.png'
    ...
def update_product():
    pid = _to_int(request.form.get('id'), "invalid_id", minimum=1)
    name, category, price = _product_form_fields()
    image_filename = _save_uploaded_image() or ''
    if not db.admin_update_product(pid, name, category, price, image_filename):
        raise ApiError("product_not_found", 404)
```
- `request.form` (không phải JSON) vì yêu cầu mang **file ảnh**.
- Hàm phụ `_product_form_fields` **dùng chung** cho thêm và sửa → kiểm tra giống nhau.
- `x or 'default_food.png'` → không tải ảnh (`None`) → dùng ảnh mặc định. Khi **sửa**, không tải ảnh → `''` → `admin_update_product` **giữ ảnh cũ**.

### 7.6 Đăng nhập, quên/đặt lại mật khẩu (L483–567)
```python
def login():
    ip = request.remote_addr or "unknown"
    retry_after = _login_retry_after(ip)
    if retry_after:
        response = jsonify({'success': False, 'reason': 'too_many_attempts', 'retry_after': retry_after})
        response.headers['Retry-After'] = str(retry_after)
        return response, 429
    password = str(_json_body().get('password', ''))
    success = db.verify_admin_password(password)
    _record_login_result(ip, success)
    if not success:
        return jsonify({'success': False, 'reason': 'wrong_password'}), 401
    return jsonify({'success': True})
```
- **Kiểm tra khóa trước** khi so mật khẩu → đang bị khóa thì **không** tốn công băm scrypt (cũng không cho dò).
- `request.remote_addr` → IP người gọi.
- Header chuẩn `Retry-After` → client biết bao lâu thử lại.
- **Đăng nhập đúng chỉ trả `success: true`** — không cấp token/phiên. Phần mềm quản trị **giữ `ADMIN_API_KEY` trong `.env` của laptop** để gọi các API; màn hình đăng nhập là lớp bảo vệ **giao diện** (người ngồi trước laptop phải biết mật khẩu).

```python
def forgot_password():
    otp_code, email, reason = db.generate_password_reset_otp()
    if not email:                 return ..., 400   # chưa cấu hình email khôi phục
    if reason == "otp_cooldown":  return ..., 429   # chưa đủ 60 s
    sent = send_otp_email(email, otp_code)
    if not sent:                  return ..., 502   # máy chủ email lỗi
    masked = email[0] + "***" + email[email.find("@"):] if "@" in email else "***"
    return jsonify({'success': True, 'masked_email': masked})
```
- **Không cần khóa** — người quên mật khẩu đương nhiên không đăng nhập được. An toàn vì: mã chỉ gửi tới **email khôi phục** đã đặt trước (kẻ gian không đọc được), chờ 60 s giữa hai lần, mã sống 10 phút, sai 5 lần thì hủy.
- **Che email**: `nguyen@gmail.com` → `n***@gmail.com` — người dùng biết mã gửi đi đâu, người lạ không biết email đầy đủ.
- `email.find("@")` → vị trí ký tự `@`; `email[vị_trí:]` → **cắt chuỗi** từ đó tới hết.
- Mã OTP **không bao giờ** nằm trong phản hồi HTTP — chỉ trong email.

```python
def change_password():
    if not db.verify_admin_password(old_password): raise ApiError("wrong_old_password")
    if len(new_password) < 6:                       raise ApiError("weak_password")
    success = db.update_admin_password(new_password)
```
- Dù đã có khóa admin, **vẫn bắt nhập mật khẩu cũ** → người mượn máy đang mở sẵn phần mềm không đổi được mật khẩu.

### 7.7 Danh sách đơn có phân trang (L572–632)
```python
    try:
        limit = int(request.args.get('limit', 10))
        page = int(request.args.get('page', 1))
    except (TypeError, ValueError):
        return jsonify({"success": False, "reason": "invalid_pagination"}), 400

    limit = max(1, min(limit, 500))
    page = max(1, page)
    offset = (page - 1) * limit
    total_items, rows = db.admin_get_orders(request.args.get('status_filter', 'all'),
                                            request.args.get('keyword', ''), limit, offset)
```
- `max(1, min(limit, 500))` → **kẹp** giá trị vào khoảng [1, 500] → không ai xin `limit=10000000` làm Pi quá tải.
- `offset = (page - 1) * limit` → trang 3, 20 đơn/trang → bỏ qua 40.

```python
    for row in rows:
        order = dict(row)
        created_at = order.get('created_at') or ''
        try:
            items = json.loads(order.get('items_json') or '[]')
        except (TypeError, json.JSONDecodeError):
            items = []
        payment_status = str(order.get('payment_status') or '').upper()
        is_paid = payment_status == 'PAID'
        orders.append({
            "id": order.get('order_id', ''),
            "date": created_at.split(" ")[0] if created_at else "",
            "time": created_at.split(" ")[1] if " " in created_at else "",
            "total": f"{int(order.get('total_amount') or 0):,}".replace(",", "."),
            "status": "paid" if is_paid else "cancelled",
            "payment_status": payment_status.lower(),
            "paid_amount": int(order.get('paid_amount') or 0),
            "kitchen_status": (order.get('kitchen_status') or 'pending' if is_paid else "none"),
            "items": items
        })

    return jsonify({"data": orders,
                    "totalPages": max(1, (total_items + limit - 1) // limit),
                    "currentPage": page, "totalItems": total_items})
```
- Đổi dòng CSDL thành **định dạng giao diện cần**: tách ngày/giờ (`"2026-11-25 21:19:17".split(" ")`), định dạng tiền `22.000`, giải mã `items_json` thành mảng.
- `items_json` hỏng → mảng rỗng, **không làm hỏng cả danh sách**.
- `"status"` giữ **hợp đồng cũ** (`paid`/`cancelled`) cho Kitchen MCP; `"payment_status"` là trạng thái chi tiết mới (`late_paid`, `expired`...). Thêm trường mới thay vì đổi trường cũ → **client cũ không hỏng** (*tương thích ngược*).
- `A or 'pending' if is_paid else "none"` → Python hiểu là `(A or 'pending') if is_paid else "none"`: đơn chưa trả tiền thì không có trạng thái bếp.
- **Chia làm tròn lên**: `(total + limit - 1) // limit` → 41 đơn, 20/trang → `(41+19)//20 = 3` trang. `//` là **chia lấy phần nguyên**.

### 7.8 Đơn chờ cho bếp (L634–695)
- Đọc `admin_get_kitchen_pending_orders()` (PAID + bếp chưa xong, cũ nhất trước).
- Đổi từng món về **đúng định dạng của tin MQTT `order.created`** mà kiosk gửi (`name` = tên gốc, `options.size/ice`, `note`) → firmware **dùng chung một hàm** hiển thị cho đơn đến qua MQTT và đơn lấy lại qua API.
- `"event": "order.snapshot"` / `"orders.pending.snapshot"` + `"version": 1` → ghi rõ loại dữ liệu.
- Mỗi trường đều có giá trị dự phòng (`product_id` lỗi → 0, `quantity` lỗi → 1, `max(1, quantity)`) → một món dữ liệu xấu không làm firmware lỗi.
- Đây là **đường phục hồi**: MQTT mất tin → màn hình bếp gọi endpoint này 30 s/lần để không sót đơn.

### 7.9 Báo xong đơn (L697–722)
```python
def mark_order_done():
    data = request.get_json(silent=True) or {}
    order_code = str(data.get('order_code', '')).strip()
    if not order_code:
        return ..., 400
    success = db.admin_mark_order_done(order_code)
    if not success:
        return jsonify({"success": False, "reason": "order_not_found_or_already_done",
                        "order_code": order_code}), 409
    publish_order_done(order_code)
    return jsonify({"success": True, "order_code": order_code, "kitchen_status": "done"}), 200
```
- CSDL dùng **UPDATE có điều kiện** (`PAID` và `pending`) → màn hình bếp và trợ lý giọng nói cùng báo xong một đơn: bên đầu **200**, bên sau **409**. Thử thật: lần 1 → 200, lần 2 → 409.
- **409 chứ không 404**: đơn có thể **tồn tại** nhưng **đã xong** → là xung đột trạng thái.
- Thành công mới phát `order.done` → các màn hình khác gỡ thẻ.

### 7.10 Kho (L727–790)
```python
def get_inventory():
    ingredients = db.get_all_ingredients()
    for i in ingredients:
        i["low"] = i["quantity"] <= i["low_stock_threshold"]
    return jsonify(ingredients)
```
- Tính sẵn cờ `low` → client không phải tự so sánh.

| Endpoint | Kiểm tra đặc biệt | Lỗi |
|---|---|---|
| `add` | `quantity`, `threshold` ≥ 0, hữu hạn | 400 |
| `update` | **Không** nhận `quantity` (số lượng chỉ đổi qua nhập kho / bán) | 404 nếu không có |
| `delete` | — | 404 không có · **409 `ingredient_in_use`** nếu còn món dùng |
| `restock` | `add_quantity` **> 0** | 404 không có |

- Sau mỗi thay đổi → `publish_inventory_snapshot()` → màn hình bếp cập nhật ngay.

### 7.11 Công thức (L793–825)
```python
def get_recipe():
    product_id = request.args.get('product_id', type=int)
    if not product_id:
        return ..., 400
```
- `request.args.get(..., type=int)` → Flask **tự đổi sang số**; đổi không được → `None` (không ném lỗi).

```python
def save_recipe():
    data = _json_body()
    product_id = _to_int(data.get('product_id'), "missing_product_id", minimum=1)
    raw_items = data.get('items', [])
    if not isinstance(raw_items, list):
        raise ApiError("invalid_items")
    recipe_items = []
    seen = set()
    for raw in raw_items:
        if not isinstance(raw, dict):
            raise ApiError("invalid_items")
        ingredient_id = _to_int(raw.get('ingredient_id'), "invalid_items", minimum=1)
        qty_per_unit = _to_float(raw.get('qty_per_unit'), "invalid_items", strictly_positive=True)
        if ingredient_id in seen:
            raise ApiError("duplicate_ingredient")
        seen.add(ingredient_id)
        recipe_items.append({"ingredient_id": ingredient_id, "qty_per_unit": qty_per_unit})
    success = db.admin_save_recipe(product_id, recipe_items)
    return jsonify({"success": success})
```
**Hai lớp bảo vệ — liên hệ trực tiếp slide 7:**
| Lớp | Chặn | Thử thật |
|---|---|---|
| API: `set` `seen` | Trùng nguyên liệu → lỗi **thân thiện, rõ ràng** | `400 duplicate_ingredient` |
| CSDL: khóa ghép + khóa ngoại | Chốt cuối nếu API có sai sót; nguyên liệu/món không tồn tại | `409 constraint_violation` |

- `set()` → kiểm tra "đã gặp chưa" **O(1)**.
- Danh sách rỗng `[]` hợp lệ → **xóa hết** công thức của món.

---

## 8. `main()` — chạy máy chủ (L828–848)

```python
def main():
    port = int(os.environ.get('KIOSK_API_PORT', '5000'))
    db.init_database()

    try:
        from waitress import serve
    except ImportError:
        serve = None

    if serve is not None:
        serve(app, host='0.0.0.0', port=port, threads=8)
    else:
        app.run(host='0.0.0.0', port=port, threaded=True)

if __name__ == '__main__':
    main()
```
| Điểm | Giải thích |
|---|---|
| Bảng cổng (chú thích) | Kiosk API **5000**, webhook **5001**, Kitchen MCP **3001**, Voice Gateway **8000** |
| `db.init_database()` | Chạy trên CSDL mới vẫn có bảng (không lỗi `no such table`) |
| **Waitress** | Máy chủ WSGI dùng cho **chạy thật**: ổn định, xử lý **8 yêu cầu song song** (8 luồng) |
| `app.run(...)` | Máy chủ **phát triển** của Flask — chỉ dùng khi chưa cài Waitress (in nhắc cài) |
| `try: from waitress import serve / except ImportError` | **Import tùy chọn**: có thì dùng, không có vẫn chạy |
| `host='0.0.0.0'` | Nghe trên **mọi** card mạng → laptop, ESP32 trong cùng Wi-Fi gọi được. (`127.0.0.1` thì chỉ chính Pi gọi được) |

Trên Pi, file này được systemd chạy tự động (`kiosk-api.service`).

**Vì sao 8 luồng quan trọng với SQLite?** 8 yêu cầu có thể cùng ghi → mỗi luồng mở kết nối riêng (`get_connection`) → SQLite xếp hàng nhờ `busy_timeout` + `BEGIN IMMEDIATE`; bộ đếm đăng nhập cần `threading.Lock`.

---

## 9. Tổng kết

### 9.1 Đường đi của một yêu cầu
```
yêu cầu HTTP
 → Flask định tuyến (@app.route)              — sai đường dẫn 404, sai phương thức 405
 → decorator kiểm tra khóa                     — 503 / 401 / 403
 → hàm kiểm tra đầu vào (_to_int, _to_float…)  — 400 kèm lý do
 → database.py (giao dịch, khóa, ràng buộc)    — IntegrityError → 409
 → kết quả nghiệp vụ                           — không có 404, xung đột 409
 → (nếu đổi kho / xong đơn) phát MQTT có chữ ký
 → jsonify → 200
 lỗi bất ngờ ở bất kỳ đâu                      — 500 internal_error (chi tiết chỉ trong nhật ký)
```

### 9.2 Các nguyên tắc bảo mật trong file
1. **Fail-closed**: chưa cấu hình khóa → 503, không mở cửa.
2. **Quyền tối thiểu**: khóa bếp chỉ làm việc bếp (403 nếu vượt quyền).
3. **So sánh hằng thời gian** cho khóa API.
4. **Kiểm tra mọi đầu vào**, kể cả `bool` giả số, `nan`, độ dài chuỗi, kích thước tải lên.
5. **Tên file tải lên**: `secure_filename` + danh sách đuôi + tiền tố ngẫu nhiên; trả file bằng `send_from_directory`.
6. **Giới hạn đăng nhập sai** theo IP với đồng hồ đơn điệu.
7. **Không lộ chi tiết lỗi** cho client.
8. **Không trả bí mật**: email bị che, OTP chỉ đi qua email, không trả hash mật khẩu.

### 9.3 Ghi chú (không gây lỗi hiện tại)
| Điểm | Ghi chú |
|---|---|
| MQTT `connect` lúc import, lỗi thì không `loop_start` | Broker không tới được lúc khởi động → mất cập nhật MQTT tức thì tới khi khởi động lại; màn hình bếp vẫn đồng bộ qua API |
| Bộ đếm đăng nhập theo IP, lưu trong bộ nhớ | Khởi động lại API thì về 0. Nếu sau này đặt API sau một proxy/đường hầm, mọi người dùng chung một IP → nên đọc IP thật từ header của proxy |
| Ảnh kiểm tra theo đuôi, không theo nội dung | Rủi ro thấp: file chỉ được trả như ảnh tĩnh |
| Đăng nhập không cấp token phiên | API dựa vào khóa trong `.env` của laptop; màn đăng nhập bảo vệ giao diện |
| `load_dotenv` không `override` (khác `bridge.py`) | Có chủ đích ở đây (systemd/test đặt biến được), nhưng hai file hành xử khác nhau |

---

## 10. Câu hỏi tự kiểm tra
1. 401 và 403 khác nhau thế nào? Cho ví dụ trong đồ án.
2. Vì sao `_key_matches` trả `False` khi `expected` rỗng? Nếu bỏ dòng đó thì sao?
3. Viết lại `@require_api_key` trên `get_stats` thành lời gọi hàm tương đương. Vì sao cần `@wraps`?
4. Vì sao `_to_int` phải từ chối `True`?
5. `float("nan")` có gì nguy hiểm khi lưu làm số lượng kho?
6. Ba lớp bảo vệ khi tải ảnh lên là gì? `secure_filename("../../etc/passwd")` ra gì?
7. Lưu công thức có nguyên liệu id 999 → đi qua những hàm nào để thành mã 409?
8. Vì sao báo xong đơn lần 2 trả 409 chứ không phải 404 hay 200?
9. Vì sao giới hạn đăng nhập dùng `time.monotonic()` thay vì `time.time()`?
10. `(total_items + limit - 1) // limit` tính gì? Thử với 40 và 41 đơn, 20 đơn/trang.
11. Vì sao `/api/menu` và `/images/...` không cần khóa?
12. Vì sao dùng Waitress thay vì `app.run()`? `host='0.0.0.0'` nghĩa là gì?

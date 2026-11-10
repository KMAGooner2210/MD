# TỔNG HỢP CÔNG NGHỆ, LÝ THUYẾT VÀ KỸ THUẬT TRONG DỰ ÁN

**Hệ thống tự động hóa quy trình phục vụ và điều phối thông minh trong nhà hàng**
(Smart Kiosk – Kitchen Display System – Trợ lý giọng nói Edge AI)

> Tài liệu học tập và tra cứu. Mỗi mục gồm: **lý thuyết** → **cách dùng trong dự án** → **code thật** (trích từ mã nguồn, có ghi đường dẫn) → **kỹ thuật/lưu ý**. Không bao gồm phần thiết kế mạch in.
>
> Đường dẫn gốc:
> - `Smart_Kiosk_Backup/` – kiosk, phần mềm quản trị, máy chủ API/webhook, CSDL
> - `kds_firmware.ino/kds_firmware/` – firmware màn hình bếp (KDS)
> - `Voice_Test/voice_assistant_edge_ai/` – firmware thiết bị giọng nói (`src/`), Voice Gateway (`gateway/`), Kitchen MCP (`services/kitchen_mcp/`)

---

## MỤC LỤC

- [0. Bức tranh tổng thể](#0-bức-tranh-tổng-thể)
- [Phần A. Nền tảng chung](#phần-a-nền-tảng-chung)
- [Phần B. Smart Kiosk và phần mềm quản trị](#phần-b-smart-kiosk-và-phần-mềm-quản-trị)
- [Phần C. Firmware màn hình bếp (KDS)](#phần-c-firmware-màn-hình-bếp-kds)
- [Phần D. Firmware thiết bị trợ lý giọng nói](#phần-d-firmware-thiết-bị-trợ-lý-giọng-nói)
- [Phần E. Voice Gateway](#phần-e-voice-gateway)
- [Phần F. Kitchen MCP](#phần-f-kitchen-mcp)
- [Phần G. Kỹ thuật thiết kế xuyên suốt](#phần-g-kỹ-thuật-thiết-kế-xuyên-suốt)
- [Phụ lục. Bảng tham số và thuật ngữ](#phụ-lục-bảng-tham-số-và-thuật-ngữ)

---

## 0. Bức tranh tổng thể

```
                 SePay (Internet) ──webhook HTTPS──┐            MQTT broker
                                                    ▼            ▲   │
┌────────── Raspberry Pi 4 ──────────┐   ┌─ webhook_server (Flask)  │   │ kitchen/new_order
│ kiosk_app (PyQt6 + QML)            │◄──┘   payment/success ───────┘   │ kitchen/inventory
│ api_server (Flask, :5000)          │                                  ▼
│ SQLite kiosk.db (WAL)              │◄──── HTTP REST ──── KDS: ESP32-S3 + LVGL + FreeRTOS
└──────────────▲─────────────────────┘
               │ HTTP REST (X-API-Key)
┌──────────────┴────────── Laptop ─────────────────────────────────────────┐
│ admin_app (PySide6 + QML)                                                │
│ Kitchen MCP (MCP Python SDK, Streamable HTTP :3001) ──HTTP──► api_server │
│ Voice Gateway (FastAPI :8000): STT Faster-Whisper · NLU · Router ·       │
│                                Ollama qwen2.5:3b · Tool guard · Piper TTS│
└──────────────▲───────────────────────────────────────────────────────────┘
               │ WebSocket (Bearer token) + HTTP tải WAV
        Thiết bị giọng nói: ESP32-S3 · INMP441 (I2S) · MAX98357A (I2S) · ILI9225 (SPI)
```

| Lĩnh vực | Công nghệ chính | Thành phần |
|---|---|---|
| Giao diện desktop | Qt 6, QML, PyQt6, PySide6 | Kiosk, Admin |
| Máy chủ web | Flask (WSGI), FastAPI (ASGI), Uvicorn | API, webhook, Gateway |
| Cơ sở dữ liệu | SQLite, WAL, giao dịch ACID | `backend/database.py` |
| Truyền thông | HTTP/REST, MQTT, WebSocket, JSON-RPC/MCP | Toàn hệ thống |
| Bảo mật | HMAC-SHA256, API key, Bearer token, băm mật khẩu, OTP | API, MQTT, Gateway |
| Thanh toán | VietQR (EMVCo), SePay webhook + API | Kiosk |
| Nhúng | ESP32-S3, Arduino-ESP32, FreeRTOS, I2S, SPI/QSPI, I2C, ADC, NVS | KDS, thiết bị giọng nói |
| Đồ họa nhúng | LVGL 8.3, Arduino_GFX, font khử răng cưa | KDS, màn ILI9225 |
| Xử lý tín hiệu số | Lọc DC, lọc thông cao, AGC, VAD, giới hạn mềm, lấy mẫu lại | Firmware, TTS |
| Trí tuệ nhân tạo | Whisper/Faster-Whisper, LLM (Qwen2.5 qua Ollama), tool calling, Piper (VITS) | Gateway |
| Xử lý ngôn ngữ | Chuẩn hóa Unicode, so khớp mờ, regex, so khớp theo âm | Gateway, MCP |
| Kiểm thử | pytest, kiểm thử hợp đồng, kiểm thử trên phần cứng | Mọi thành phần |

---

## Phần A. Nền tảng chung

### A.1. Python hiện đại trong dự án

Dự án dùng Python 3.10 với các kỹ thuật:

| Kỹ thuật | Ý nghĩa | Ví dụ trong dự án |
|---|---|---|
| Chú thích kiểu (`type hints`), `from __future__ import annotations` | Đọc code dễ, công cụ kiểm tra lỗi kiểu sớm | Toàn bộ Gateway |
| `Final[...]` | Hằng số không được gán lại | `PROTOCOL_NAME: Final[str] = "smart-kiosk-voice"` |
| `@dataclass(frozen=True)` | Đối tượng dữ liệu bất biến, an toàn khi chia sẻ giữa luồng | `PhoneticResult` (MCP), các model kết quả STT |
| `Enum` kế thừa `str` | Giá trị liệt kê vừa so sánh được vừa tuần tự hóa JSON được | `ToolPolicyAction(str, Enum)` |
| `async`/`await` | Lập trình bất đồng bộ, một luồng phục vụ nhiều kết nối | Gateway, MCP client |
| `MappingProxyType`, `frozenset` | Bảng tra chỉ đọc | Router, danh mục lệnh |
| Môi trường ảo (`venv`) | Mỗi dự án một bộ thư viện riêng | `.venv-gateway/` |
| Cấu hình qua biến môi trường + `.env` | Không chép cứng khóa bí mật vào code | `python-dotenv`, `config/settings.py` |

**Kiểm tra đầu vào chặt chẽ.** Gateway kiểm tra kiểu và giá trị ngay ở hàm khởi tạo, lưu ý `bool` là lớp con của `int` trong Python nên phải loại riêng:

```python
# gateway/src/voice_gateway/tts/piper_service.py
@staticmethod
def _require_positive_integer(value: int, field_name: str) -> int:
    """Số nguyên dương; bool bị từ chối dù là subclass của int."""
    if isinstance(value, bool) or not isinstance(value, int):
        raise TypeError(f"{field_name} must be an integer")
    if value <= 0:
        raise ValueError(f"{field_name} must be positive")
    return value
```

### A.2. C++ cho vi điều khiển

| Kỹ thuật | Ý nghĩa |
|---|---|
| `constexpr` | Hằng số tính lúc biên dịch, không tốn RAM |
| `static_assert` | Kiểm tra cấu hình lúc biên dịch (ví dụ hai thiết bị không dùng trùng chân GPIO) |
| `namespace` | Tách nhóm hằng số/lớp: `BoardConfig`, `AudioConfig`, `Audio`, `Device` |
| Mảng ký tự kích thước cố định | Tránh cấp phát động, chép được qua hàng đợi FreeRTOS, không phân mảnh bộ nhớ khi chạy dài ngày |
| Header C++ thuần không phụ thuộc Arduino | Kiểm thử logic (pin, font, bố cục màn hình) ngay trên máy tính |

```cpp
// src/config/board_config.h – chống trùng chân ngay khi biên dịch
static_assert(
    BATTERY_SENSE_PIN != MIC_BCLK_PIN && BATTERY_SENSE_PIN != MIC_WS_PIN &&
    BATTERY_SENSE_PIN != MIC_DATA_PIN && BATTERY_SENSE_PIN != SPEAKER_BCLK_PIN &&
    BATTERY_SENSE_PIN != SPEAKER_WS_PIN && BATTERY_SENSE_PIN != SPEAKER_DATA_PIN &&
    BATTERY_SENSE_PIN != SPEAKER_SD_PIN && BATTERY_SENSE_PIN != PTT_BUTTON_PIN &&
    BATTERY_SENSE_PIN != STATUS_LED_PIN,
    "BATTERY_SENSE_PIN conflicts with audio/button/LED pins"
);
```

### A.3. Kiểm thử tự động

| Thành phần | Công cụ | Số ca kiểm thử (11/2026) |
|---|---|---|
| Smart Kiosk (API, xác thực, vòng đời đơn, MQTT ký, webhook) | pytest + `app.test_client()` của Flask | 75 |
| Voice Gateway (unit, contract, integration, live) | pytest, pytest-asyncio | 1.039 |
| Kitchen MCP | pytest | 399 |
| Firmware giọng nói | Sketch kiểm thử riêng trong `tests/hardware/*` chạy trên ESP32 | ~30 nhóm |

Các loại kiểm thử:
- **Unit test**: một hàm/lớp, phụ thuộc bên ngoài được thay bằng đối tượng giả.
- **Contract test**: kiểm tra định dạng thông điệp hai bên thống nhất (ví dụ giao thức WebSocket thiết bị ↔ Gateway).
- **Integration test**: nhiều thành phần chạy cùng nhau (Gateway + MCP giả).
- **Live test**: chạy với dịch vụ thật (Ollama, Whisper), bật bằng biến môi trường.
- **Hardware test**: sketch Arduino riêng cho từng module (micro, loa, Wi-Fi, WebSocket, máy trạng thái).

---

## Phần B. Smart Kiosk và phần mềm quản trị

### B.1. Qt, QML và cầu nối Python

**Lý thuyết.**
- **Qt** là bộ khung ứng dụng đa nền tảng viết bằng C++. **Qt Quick** là công nghệ giao diện hiện đại của Qt, mô tả giao diện bằng **QML**, ngôn ngữ khai báo giống JSON có nhúng biểu thức JavaScript.
- **Ràng buộc thuộc tính (property binding):** một thuộc tính được gắn với biểu thức; khi giá trị trong biểu thức đổi, Qt tự tính lại. Không cần viết lệnh cập nhật giao diện thủ công.
- **Tín hiệu – khe (signal – slot):** cơ chế sự kiện của Qt. Đối tượng phát *tín hiệu*; các *khe* (hàm) đã kết nối sẽ được gọi. Qt tự chuyển lời gọi về đúng luồng giao diện nếu tín hiệu phát từ luồng khác.
- **PyQt6** (Riverbank, GPL/thương mại) và **PySide6** (The Qt Company, LGPL) là hai bộ liên kết Qt 6 cho Python, API gần như giống nhau. Kiosk dùng PyQt6, Admin dùng PySide6.

**Áp dụng.** Lớp `KioskBridge` (Python) được đăng ký vào QML; QML chỉ hiển thị và bắt thao tác, mọi nghiệp vụ nằm ở Python.

```python
# Smart_Kiosk_Backup/kiosk_app/bridge.py
from PyQt6.QtCore import QObject, pyqtSlot as Slot, pyqtSignal as Signal

class KioskBridge(QObject):
    cartUpdated = Signal()        # báo QML vẽ lại giỏ
    qrGenerated = Signal(str)     # gửi đường dẫn ảnh VietQR cho popup
    paymentSuccess = Signal()     # thanh toán xong -> đóng popup

    # các slot QML gọi: getMenuData(), addToCart(...), removeFromCart(id),
    # checkout(), cancelPayment(), expirePayment()
```

| Loại | Tên | Vai trò |
|---|---|---|
| Slot | `getMenuData()` | Trả danh sách món đang bán |
| Slot | `addToCart(id, name, price, image, size, ice)` | Thêm món hoặc tăng số lượng |
| Slot | `checkout()` | Tạo đơn PENDING, tạo mã QR, bắt đầu chờ thanh toán |
| Slot | `cancelPayment()` / `expirePayment()` | Hủy / hết hạn đơn đang chờ |
| Signal | `cartUpdated`, `qrGenerated(url)`, `paymentSuccess` | Python → QML |

**Kỹ thuật giỏ hàng.**
- **Khóa ghép** `{product_id}_{size}_{ice}`: cùng món, cùng tùy chọn thì gộp dòng (tăng `qty`); khác tùy chọn thì tách dòng.
- **Khóa giỏ khi đang thanh toán:** sau khi tạo mã QR, mọi thao tác thêm/bớt bị từ chối, bảo đảm số tiền trên mã QR khớp đơn.

**Lỗi thực tế đã gặp – `Popup` tự đóng.** `Popup` của QML mặc định đóng khi chạm ra ngoài hoặc nhấn Esc (`closePolicy` mặc định). Popup thanh toán bị khách chạm ra ngoài làm đóng mà đơn không bị hủy, kiosk kẹt. Cách sửa: `closePolicy: Popup.NoAutoClose`, chỉ đóng qua nút Hủy hoặc khi hết hạn.

### B.2. Biến môi trường và lỗi khởi động từ terminal VS Code bản snap

**Lý thuyết.** Ứng dụng Linux đọc nhiều biến môi trường (`GSETTINGS_SCHEMA_DIR`, `GTK_PATH`, `XDG_DATA_DIRS`…) để tìm thư viện, plugin, schema. VS Code cài qua **snap** chèn các biến trỏ vào thư mục riêng của snap. Ứng dụng Qt chạy từ terminal đó nạp plugin GTK, đọc nhầm schema cũ và GLib dừng chương trình.

**Áp dụng.** Gọi hàm dọn biến **trước khi** tạo `QGuiApplication`:

```python
# Smart_Kiosk_Backup/backend/desktop_env.py
_SNAP_GTK_VARS = ("GSETTINGS_SCHEMA_DIR", "GIO_MODULE_DIR", "GTK_PATH",
                  "GTK_EXE_PREFIX", "GTK_IM_MODULE_FILE")

def clean_snap_terminal_env(environ=None):
    if environ is None:
        environ = os.environ
    if not sys.platform.startswith("linux"):
        return []
    cleaned = []
    for name in _SNAP_GTK_VARS:
        if "/snap/" in environ.get(name, ""):
            del environ[name]
            cleaned.append(name)
    # ... khôi phục XDG_DATA_DIRS / XDG_CONFIG_DIRS từ bản *_VSCODE_SNAP_ORIG
```

### B.3. SQLite: mô hình quan hệ, giao dịch và WAL

**Lý thuyết.**
- **SQLite** là CSDL quan hệ **nhúng**: thư viện nằm trong chương trình, dữ liệu trong **một tệp**, không có tiến trình máy chủ.
- **Mô hình quan hệ:** bảng gồm hàng và cột; **khóa chính** xác định duy nhất một hàng; **khóa ngoại** liên kết bảng; quan hệ nhiều–nhiều dùng bảng trung gian.
- **Giao dịch ACID:**
  - *Atomicity* (nguyên tử): cả khối thành công hoặc không có gì thay đổi;
  - *Consistency* (nhất quán): luôn thỏa ràng buộc;
  - *Isolation* (cô lập): giao dịch song song không thấy trạng thái dở dang của nhau;
  - *Durability* (bền vững): đã commit thì không mất khi mất điện.
- **Các mức khóa của SQLite:** `BEGIN` (DEFERRED) chỉ lấy khóa khi cần; `BEGIN IMMEDIATE` lấy khóa **ghi** (RESERVED) ngay từ đầu, nên hai tiến trình không thể cùng đọc số liệu cũ rồi cùng ghi đè.
- **WAL (Write-Ahead Logging):** thay đổi được **ghi nối vào tệp `-wal`** trước; người đọc đọc tệp chính kết hợp WAL nên **đọc không bị chặn bởi người ghi**. Định kỳ SQLite *checkpoint* chép WAL vào tệp chính. Giới hạn: tại một thời điểm vẫn chỉ **một** người ghi.
- `busy_timeout`: thời gian chờ khi CSDL đang bị khóa thay vì báo lỗi ngay.

**Lược đồ dữ liệu.**

| Bảng | Khóa chính | Trường chính | Ghi chú |
|---|---|---|---|
| `product` | `id` tự tăng | `name`, `price`, `image`, `category`, `status` | Món ăn, đồ uống |
| `ingredient` | `id` tự tăng | `name`, `unit`, `quantity`, `low_stock_threshold` | Nguyên liệu |
| `recipe` | (`product_id`, `ingredient_id`) | `qty_per_unit` | Bảng nối nhiều–nhiều món ↔ nguyên liệu |
| `orders` | `order_id` (`DH` + 8 số) | `total_amount`, `items_json`, `payment_status`, `kitchen_status`, `paid_amount`, `created_at`, `closed_at` | Món lưu dạng JSON để giữ nguyên tên, giá lúc bán |
| `settings` | `id` (`CHECK (id = 1)`) | thông tin cửa hàng, băm mật khẩu, email khôi phục, OTP | Chỉ có đúng một hàng |

**Cấu hình kết nối:** mỗi kết nối bật `PRAGMA journal_mode=WAL`, `busy_timeout=10000` ms và `foreign_keys=ON`. Đường dẫn CSDL đổi được qua `KIOSK_DB_PATH` (bắt buộc đường dẫn tuyệt đối) để kiểm thử trên CSDL tạm.

**Vòng đời đơn hàng (máy trạng thái thanh toán).**

| `payment_status` | Khi nào | Bước tiếp theo |
|---|---|---|
| `PENDING` | Bấm Thanh toán, mã QR được tạo | `PAID` khi nhận tiền; `CANCELLED` khi khách hủy; `EXPIRED` khi hết 300 s |
| `PAID` | Giao dịch đủ tiền được xác nhận | `kitchen_status`: `pending` → `done` |
| `CANCELLED`, `EXPIRED` | Đơn đóng, không trừ kho | `LATE_PAID` nếu tiền đến muộn |
| `LATE_PAID` | Khách chuyển khoản sau khi đơn đã đóng | Quản trị hoàn tiền thủ công |

**Code: lưu đơn và trừ kho trong một giao dịch.**

```python
# Smart_Kiosk_Backup/backend/database.py (rút gọn)
def save_new_order(order_code, total_amount, items_json, paid_amount=None):
    ...
    conn = get_connection()
    conn.execute("BEGIN IMMEDIATE")                       # giành khóa ghi ngay

    existing = conn.execute(
        "SELECT payment_status FROM orders WHERE order_id = ?", (order_code,)
    ).fetchone()
    if existing is not None and existing["payment_status"] != ORDER_PENDING:
        conn.rollback()                                    # đã PAID -> trùng; đã đóng -> LATE_PAID
        result["reason"] = ("duplicate_order"
                            if existing["payment_status"] == ORDER_PAID else "order_closed")
        return result

    required = {}                                          # tổng nhu cầu nguyên liệu
    for item in normalized_items:
        rows = conn.execute("""
            SELECT r.ingredient_id, r.qty_per_unit, i.name, i.unit, i.quantity, i.low_stock_threshold
            FROM recipe r JOIN ingredient i ON i.id = r.ingredient_id
            WHERE r.product_id = ?""", (item["product_id"],)).fetchall()
        for row in rows:
            info = required.setdefault(int(row["ingredient_id"]), {...})
            info["required"] += float(row["qty_per_unit"]) * item["qty"]

    conn.execute("""UPDATE orders SET payment_status = 'PAID', kitchen_status = 'pending', ...
                    WHERE order_id = ? AND payment_status = 'PENDING'""", (...))
    for info in required.values():
        conn.execute("UPDATE ingredient SET quantity = MAX(0, quantity - ?) WHERE id = ?",
                     (info["required"], info["id"]))
    ...
    conn.commit()                                          # lỗi ở bất kỳ bước nào -> rollback
```

**Kỹ thuật rút ra:**
- **Cập nhật có điều kiện** (`WHERE ... AND payment_status = 'PENDING'`) kèm kiểm tra `rowcount == 1` là một dạng **khóa lạc quan** ở mức câu lệnh: chỉ chuyển trạng thái nếu đang ở đúng trạng thái trước.
- `MAX(0, quantity - ?)`: đơn đã trả tiền **luôn được lưu**; kho dừng ở 0 và trả danh sách `insufficient_stock` để cảnh báo.
- **Hoàn thành đơn ở bếp** cũng dùng cập nhật có điều kiện `WHERE payment_status='PAID' AND kitchen_status='pending'`. KDS và trợ lý giọng nói cùng báo hoàn thành một đơn thì chỉ một bên thành công, bên kia nhận HTTP 409.

### B.4. Flask và REST API

**Lý thuyết.**
- **HTTP** theo mô hình yêu cầu – phản hồi, không lưu trạng thái. Yêu cầu gồm *phương thức*, *đường dẫn*, *tiêu đề*, *thân*; phản hồi gồm *mã trạng thái*, *tiêu đề*, *thân*.
- **REST**: dữ liệu là *tài nguyên* có URL; dùng GET để đọc, POST để tạo/thực hiện hành động; kết quả biểu diễn bằng mã trạng thái chuẩn.
- **Flask**: khung web "vi mô" theo chuẩn **WSGI**, gắn URL với hàm bằng `@app.route`, mỗi yêu cầu xử lý đồng bộ.
- **Decorator (bộ trang trí)**: hàm bọc hàm khác để thêm hành vi (xác thực, ghi log) mà không sửa thân hàm; `functools.wraps` giữ tên và docstring của hàm gốc.

| Mã | Ý nghĩa | Dùng khi |
|---|---|---|
| 200 | Thành công | Đọc/ghi thành công |
| 400 | Sai định dạng | Thiếu `order_code`, JSON hỏng |
| 401 | Chưa xác thực | Sai/thiếu `X-API-Key` |
| 409 | Xung đột trạng thái | Đơn không tồn tại hoặc đã hoàn thành |
| 503 | Chưa sẵn sàng | Máy chủ chưa cấu hình khóa (fail-closed) |

**Code: decorator xác thực hai cấp khóa.**

```python
# Smart_Kiosk_Backup/servers/api_server.py
def _key_matches(provided, expected):
    """So sánh hằng thời gian trên bytes (str chứa ký tự không phải ASCII sẽ làm compare_digest báo lỗi)."""
    if not expected:
        return False
    return hmac.compare_digest(str(provided).encode("utf-8"), str(expected).encode("utf-8"))

def require_kitchen_or_admin_key(f):
    """ADMIN_API_KEY hoặc KITCHEN_API_KEY: chỉ dùng cho thao tác của bếp."""
    @wraps(f)
    def decorated(*args, **kwargs):
        if not ADMIN_API_KEY and not KITCHEN_API_KEY:
            return jsonify({"success": False, "reason": "api_key_not_configured"}), 503
        provided = request.headers.get("X-API-Key", "")
        if not (_key_matches(provided, ADMIN_API_KEY) or _key_matches(provided, KITCHEN_API_KEY)):
            return jsonify({"success": False, "reason": "unauthorized"}), 401
        return f(*args, **kwargs)
    return decorated

@app.route('/api/orders/done', methods=['POST'])
@require_kitchen_or_admin_key
def mark_order_done():
    order_code = str((request.get_json(silent=True) or {}).get('order_code', '')).strip()
    if not order_code:
        return jsonify({"success": False, "reason": "missing_order_code"}), 400
    if not db.admin_mark_order_done(order_code):
        return jsonify({"success": False, "reason": "order_not_found_or_already_done",
                        "order_code": order_code}), 409
    return jsonify({"success": True, "order_code": order_code, "kitchen_status": "done"}), 200
```

**Kỹ thuật:**
- **`hmac.compare_digest`**: so sánh trong thời gian hằng, tránh **tấn công đo thời gian** (timing attack), vì phép `==` dừng ở ký tự sai đầu tiên.
- **Phân quyền tối thiểu (least privilege)**: khóa **quản trị** toàn quyền, chỉ Admin dùng; khóa **bếp** chỉ được báo hoàn thành đơn, dùng cho KDS và Kitchen MCP. Lộ khóa bếp không sửa được thực đơn.
- **Fail-closed**: chưa cấu hình khóa thì trả 503 cho mọi thao tác ghi, thay vì vô tình mở cửa.
- **Cùng định dạng dữ liệu cho hai kênh:** `/api/orders/pending` trả đơn cùng cấu trúc với sự kiện MQTT, nên KDS dùng chung một hàm phân tích (`parseOrderObject`).

### B.5. Mật khẩu và OTP

| Kỹ thuật | Chi tiết |
|---|---|
| Băm mật khẩu có muối | `werkzeug.security.generate_password_hash` / `check_password_hash`. *Muối (salt)* ngẫu nhiên khiến hai mật khẩu giống nhau vẫn ra hai chuỗi băm khác, chống bảng tra sẵn (rainbow table) |
| OTP khôi phục | 6 chữ số ngẫu nhiên, hết hạn sau 10 phút, xóa ngay khi dùng xong |
| Gửi email | SMTP + STARTTLS (cổng 587), dùng *App Password* của Gmail |
| Che thông tin | Email chỉ hiển thị dạng `n***@gmail.com` |

### B.6. Thanh toán: VietQR, SePay, webhook + polling

**Lý thuyết VietQR.** Chuẩn mã QR thanh toán của NAPAS, dựa trên đặc tả **EMVCo Merchant-Presented QR**. Nội dung là chuỗi các trường **TLV** (*ID 2 chữ số – Độ dài 2 chữ số – Giá trị*):

| ID | Ý nghĩa | Ví dụ |
|---|---|---|
| 00 | Phiên bản định dạng | `01` |
| 01 | Loại mã: `11` tĩnh, `12` động | `12` |
| 38 | Thông tin người nhận: định danh NAPAS, mã ngân hàng (BIN), số tài khoản | – |
| 53 | Tiền tệ | `704` (VND) |
| 54 | Số tiền | `540570000` = ID 54, dài 05, giá trị 70000 |
| 58 | Quốc gia | `VN` |
| 62 | Thông tin thêm, gồm nội dung chuyển khoản | mã đơn |
| 63 | CRC16-CCITT của toàn chuỗi | 4 ký tự hex |

**Mã động** nhúng sẵn số tiền và nội dung là mã đơn: khách không phải gõ gì, hệ thống chỉ cần tìm mã đơn trong nội dung giao dịch. Kiosk tạo ảnh QR qua dịch vụ của SePay:

```
https://qr.sepay.vn/img?acc=<số TK>&bank=<ngân hàng>&amount=<số tiền>&des=<mã đơn>
```

**Lý thuyết webhook và polling.**
- **Webhook**: dịch vụ ngoài **chủ động** gửi HTTP POST đến URL đăng ký khi có sự kiện. Nhanh, nhưng cần URL công khai và có thể thất lạc.
- **Polling**: hệ thống **định kỳ hỏi** API. Chắc chắn hơn, không cần URL công khai, nhưng chậm theo chu kỳ.
- Dự án dùng **cả hai**, tín hiệu nào đến trước sẽ kích hoạt lưu đơn.

| Đường | Cơ chế | Thông số |
|---|---|---|
| Webhook → MQTT | `webhook_server.py` nhận `POST /sepay_webhook`, kiểm tra `Authorization: Apikey <khóa>`, chỉ nhận tiền **vào**, tìm mã `DH\d{8}`, phát `payment/success` | Gần như tức thời |
| Polling | Luồng nền gọi API giao dịch SePay với Bearer token, tìm giao dịch chứa mã đơn | 15 s/lần, tối đa 300 s |

```python
# Smart_Kiosk_Backup/servers/webhook_server.py (rút gọn)
ORDER_CODE_PATTERN = re.compile(r"DH\d{8}")

def _is_request_authorized(req):
    """Fail-closed: chưa cấu hình token thì từ chối mọi request."""
    if not SEPAY_WEBHOOK_AUTH_TOKEN:
        return False
    auth_header = req.headers.get("Authorization", "").strip()
    if auth_header.lower().startswith("apikey "):          # SePay gửi "Apikey <token>"
        auth_header = auth_header[len("apikey "):].strip()
    return hmac.compare_digest(auth_header.encode(), SEPAY_WEBHOOK_AUTH_TOKEN.encode())
```

> **Lưu ý đối chiếu tài liệu SePay:** dữ liệu webhook của SePay có các trường `id, gateway, transactionDate, accountNumber, code, content, transferType, transferAmount, referenceCode…`; nội dung chuyển khoản nằm ở **`content`**. Mã hiện đọc trường `transferContent`, nên cần đọc `content` (giữ `transferContent` làm dự phòng) để đường webhook hoạt động; đường polling vẫn xác nhận được thanh toán.

**Tính lũy đẳng (idempotency) và đồng bộ luồng.** Webhook (qua luồng MQTT) và polling (luồng nền) có thể cùng báo một đơn. Cả hai gọi `_finalize_payment`, được bảo vệ bằng khóa `threading.Lock`:

```python
# Smart_Kiosk_Backup/kiosk_app/bridge.py (rút gọn)
def _finalize_payment(self, order_code, amount, source):
    with self._payment_lock:
        order_mismatch = order_code != self._current_order_code
    if order_mismatch:
        # Không phải đơn đang chờ (đã hủy/hết hạn) -> ghi LATE_PAID để Admin hoàn tiền
        self._record_late_payment(order_code, amount, source)
        return

    with self._payment_lock:
        if self._payment_confirmed:            # lời gọi thứ hai -> bỏ qua
            return
        paid_amount = int(float(amount))
        expected_amount = int(float(self._pending_order_amount))
        if paid_amount < expected_amount:      # thiếu tiền -> chưa chốt
            return
        order_items = [dict(item) for item in self._pending_order_items]
        self._payment_confirmed = True         # đánh dấu TRONG khóa
        self.is_polling = False

    save_result = database.save_new_order(order_code, expected_amount,
                                          json.dumps(order_items, ensure_ascii=False),
                                          paid_amount=paid_amount)
    # thành công -> phát tồn kho, phát đơn xuống bếp (QoS 1), phát paymentSuccess, xóa giỏ
```

Ba lớp chống trùng:
1. Cờ `_payment_confirmed` đặt trong vùng khóa.
2. So mã đơn với đơn đang chờ.
3. Ở CSDL: `order_id` là khóa chính và cập nhật có điều kiện `PENDING → PAID`.

### B.7. MQTT và chữ ký HMAC cho thông điệp

**Lý thuyết MQTT.**
- Giao thức **xuất bản – đăng ký** trên TCP (cổng 1883, 8883 khi dùng TLS), chuẩn OASIS 3.1.1/5.0.
- **Broker** trung chuyển; bên gửi và bên nhận không cần biết nhau.
- **Chủ đề** phân cấp bằng `/`; ký tự đại diện `+` (một cấp), `#` (mọi cấp phía sau).
- **QoS 0/1/2**: tối đa một lần / ít nhất một lần (PUBACK) / đúng một lần (PUBREC–PUBREL–PUBCOMP).
- **Retained**: broker giữ thông điệp cuối của chủ đề, giao ngay cho bên đăng ký mới.
- **Keep-alive** (PINGREQ/PINGRESP) và **Last Will** (thông điệp di chúc khi mất kết nối đột ngột).

| Chủ đề | Bên gửi → bên nhận | Cấu hình | Lý do |
|---|---|---|---|
| `payment/success` | webhook → kiosk | QoS 1 | Không được mất tín hiệu tiền về |
| `kitchen/new_order` | kiosk → KDS | QoS 1 | Đơn không được mất; KDS tự khử trùng |
| `kitchen/inventory` | kiosk/API → KDS | Retained | Là "trạng thái hiện tại": KDS vừa bật nhận ngay |

**Lý thuyết HMAC.** **HMAC** (*Hash-based Message Authentication Code*) = hàm băm có khóa bí mật: `HMAC(K, m) = H((K ⊕ opad) ‖ H((K ⊕ ipad) ‖ m))`. Chỉ ai có khóa mới tạo được chữ ký đúng. Bên nhận tính lại và so sánh để biết thông điệp **đúng nguồn** và **không bị sửa**. Khi dùng broker công cộng, ai cũng có thể publish vào chủ đề, nên ký thông điệp là bắt buộc.

**Định dạng phong bì phiên bản 2.**

```json
{"v": 2, "sig": "<HMAC-SHA256 hex>", "payload": "<chuỗi JSON nội dung, có trường ts>"}
```

- Chữ ký tính trên **đúng chuỗi `payload` nhận được**, không dựng lại JSON. Nhờ vậy ESP32 kiểm tra được mà không cần sắp xếp khóa JSON.
- `ts` (thời điểm ký) cùng giới hạn tuổi 120 giây giúp **chống phát lại** (replay).
- Khóa bí mật **không bao giờ** nằm trong thông điệp.

```python
# Smart_Kiosk_Backup/backend/mqtt_security.py (rút gọn)
def sign_message(body, secret, *, now=None) -> str:
    content = {k: v for k, v in body.items() if k != "secret"}
    content["ts"] = int(time.time() if now is None else now)
    payload_text = json.dumps(content, ensure_ascii=False, separators=(",", ":"))
    signature = hmac.new(secret.encode(), payload_text.encode(), hashlib.sha256).hexdigest()
    return json.dumps({"v": 2, "sig": signature, "payload": payload_text},
                      ensure_ascii=False, separators=(",", ":"))

def verify_message(raw, secret, *, max_age_seconds=120, now=None) -> dict:
    envelope = json.loads(raw)
    payload_text, signature = envelope.get("payload"), envelope.get("sig")
    expected = hmac.new(secret.encode(), payload_text.encode(), hashlib.sha256).hexdigest()
    if not hmac.compare_digest(expected, signature.strip().lower()):
        raise MqttMessageRejected("bad_signature")
    content = json.loads(payload_text)
    # kiểm tra content["ts"] không quá max_age_seconds
    return content
```

Phía ESP32 dùng thư viện **mbedTLS** có sẵn trong ESP-IDF (xem C.4).

---

## Phần C. Firmware màn hình bếp (KDS)

### C.1. ESP32-S3 và lập trình Arduino-ESP32

- **ESP32-S3**: SoC hai nhân Xtensa LX7 240 MHz, 512 KB SRAM, PSRAM ngoài, Wi-Fi 2,4 GHz, BLE 5.
- **Arduino-ESP32** (gói của Espressif) xây trên **ESP-IDF**: viết theo kiểu `setup()`/`loop()` nhưng vẫn dùng được FreeRTOS, driver I2S, mbedTLS, NVS.
- `setup()` và `loop()` thực chất chạy trong tác vụ FreeRTOS `loopTask`.
- **Quy trình build:** tiền xử lý `.ino` → biên dịch `xtensa-esp32s3-elf-g++` → liên kết với lõi ESP-IDF → tệp `.bin` → `esptool` nạp qua USB.

### C.2. FreeRTOS: tác vụ, hàng đợi, mutex

**Lý thuyết.**
- **Tác vụ (task)**: hàm chạy vòng lặp vô hạn, có ngăn xếp và mức ưu tiên riêng.
  - Bốn trạng thái: *Running, Ready, Blocked, Suspended*.
- **Bộ lập lịch ưu tiên có chiếm quyền (preemptive)**: tác vụ sẵn sàng ưu tiên cao nhất được chạy; cùng mức thì luân phiên mỗi *tick* (1 ms trên ESP32).
  - ESP32-S3 chạy FreeRTOS **SMP**, cho phép **gắn tác vụ vào nhân** (`xTaskCreatePinnedToCore`).
- **Hàng đợi (queue)**: FIFO, dữ liệu được **sao chép**. Bên nhận có thể chờ (bị chặn) mà không tốn CPU.
- **Mutex**: khóa loại trừ tương hỗ có **kế thừa ưu tiên**, tránh *đảo ưu tiên* (tác vụ ưu tiên thấp giữ khóa làm tác vụ ưu tiên cao phải chờ lâu).
- **Thời gian chờ có giới hạn**: mọi thao tác chờ nên có hạn để không bị *deadlock*.

**Áp dụng – hai tác vụ trên hai nhân.**

| Đối tượng | Cấu hình | Vai trò |
|---|---|---|
| `TaskUI` | Nhân 1, ưu tiên 2 | Chạy `lv_timer_handler()` mỗi ~5 ms, vẽ lại bảng kho khi có cờ |
| `TaskNetwork` | Nhân 0, ưu tiên 3 | Wi-Fi, MQTT, đồng bộ HTTP, xử lý hàng đợi |
| `lvglMutex` | Mutex | Bảo vệ mọi lời gọi LVGL (LVGL không an toàn đa luồng) |
| `dataMutex` | Mutex | Bảo vệ mảng tồn kho dùng chung |
| `orderEventQueue` | Hàng đợi `KdsOrderEvent` | Đơn đã phân tích từ MQTT/HTTP → bước tạo thẻ |
| `doneOrderQueue` | Hàng đợi `DoneOrderMsg` | Yêu cầu "Hoàn thành" từ nút chạm → tác vụ mạng |

```cpp
// kds_firmware.ino
lvglMutex = xSemaphoreCreateMutex();
dataMutex = xSemaphoreCreateMutex();
orderEventQueue = xQueueCreate(ORDER_EVENT_QUEUE_LEN, sizeof(KdsOrderEvent));
doneOrderQueue  = xQueueCreate(DONE_QUEUE_LEN, sizeof(DoneOrderMsg));

xTaskCreatePinnedToCore(taskUI, "TaskUI", UI_TASK_STACK_SIZE, nullptr, 2, &taskUIHandle, 1);
xTaskCreatePinnedToCore(taskNetwork, "TaskNetwork", NETWORK_TASK_STACK_SIZE,
                        nullptr, 3, &taskNetworkHandle, 0);

// Trong TaskUI:
if (xSemaphoreTake(lvglMutex, pdMS_TO_TICKS(10)) == pdTRUE) {
    lv_timer_handler();
    xSemaphoreGive(lvglMutex);
}
```

**Ba nguyên tắc chống lỗi đồng thời:**
1. **Callback giao diện không gọi mạng.** Chạm "Hoàn thành" chỉ đổi trạng thái thẻ và đẩy mã đơn vào `doneOrderQueue`; tác vụ mạng mới gọi HTTP.
2. **Callback MQTT không chạm LVGL.** Chỉ phân tích JSON vào cấu trúc cố định rồi đẩy hàng đợi, hoặc bật cờ `inventoryUiDirty`.
3. **Mọi chờ đều có hạn** (khóa LVGL 10–300 ms).

```cpp
// Callback nút chạm (chạy trong lv_timer_handler, ĐÃ giữ lvglMutex)
if (orderCards[slot].state == ORDER_COMPLETING) return;
DoneOrderMsg msg{};
copyText(msg.orderCode, sizeof(msg.orderCode), orderCards[slot].orderCode);
if (xQueueSend(doneOrderQueue, &msg, 0) != pdTRUE) {
    setOrderButtonLocked(slot, "GỬI LẠI", true);
    return;
}
orderCards[slot].state = ORDER_COMPLETING;
setOrderButtonLocked(slot, "ĐANG LƯU...", false);
```

### C.3. LVGL: thư viện đồ họa nhúng

**Lý thuyết.**
- LVGL vẽ vào **bộ đệm trong RAM**, rồi gọi hàm **flush** do người lập trình viết để chép vùng đã vẽ ra màn hình.
- Chỉ vẽ lại **vùng bị thay đổi** (*invalidated area*); bộ đệm có thể nhỏ hơn màn hình (vẽ theo dải).
- Hàm **đọc thiết bị nhập** (*read callback*) trả tọa độ chạm; `lv_tick_inc()` báo thời gian, `lv_timer_handler()` xử lý sự kiện, hiệu ứng, vẽ.
- Mọi phần tử là **đối tượng** trong cây cha–con; hiển thị theo **style**; tương tác qua **event**.

```cpp
// kds_firmware.ino – hàm flush: đẩy vùng ảnh ra màn hình NV3041A qua QSPI
static void displayFlush(lv_disp_drv_t *disp, const lv_area_t *area, lv_color_t *colorP) {
  uint32_t width = area->x2 - area->x1 + 1;
  uint32_t height = area->y2 - area->y1 + 1;
  gfx->draw16bitRGBBitmap(area->x1, area->y1,
      reinterpret_cast<uint16_t *>(&colorP->full), width, height);
  lv_disp_flush_ready(disp);          // báo LVGL đã chép xong
}

// Bộ đệm vẽ ưu tiên vùng nhớ hỗ trợ DMA, thiếu thì lấy RAM trong
lvglBuf1 = (lv_color_t *)heap_caps_malloc(bytes, MALLOC_CAP_DMA | MALLOC_CAP_8BIT);
if (!lvglBuf1) lvglBuf1 = (lv_color_t *)heap_caps_malloc(bytes, MALLOC_CAP_INTERNAL | MALLOC_CAP_8BIT);
lv_disp_draw_buf_init(&drawBuf, lvglBuf1, nullptr, pixels);
```

**Tạo thẻ đơn (đối tượng, style, sự kiện):**

```cpp
lv_obj_t *card = lv_obj_create(ordersView);
lv_obj_add_style(card, &stCard, LV_PART_MAIN);
lv_obj_set_size(card, ORDER_CARD_WIDTH, ORDER_CARD_HEIGHT);

lv_obj_t *done = lv_btn_create(card);
lv_obj_add_event_cb(done, doneButtonEvent, LV_EVENT_CLICKED,
    reinterpret_cast<void *>(static_cast<intptr_t>(slot)));   // truyền chỉ số thẻ qua user_data
lv_obj_t *doneLabel = lv_label_create(done);
lv_label_set_text(doneLabel, "HOÀN THÀNH");
```

**Kỹ thuật giao diện:**
- **Khử trùng đơn:** `addOrderCardUI` tìm mã đơn trong các thẻ đang hiện trước khi tạo, vì một đơn có thể đến từ cả MQTT lẫn HTTP.
- **Màu theo thời gian chờ:** dưới 5 phút xanh, 5–10 phút vàng, từ 10 phút đỏ. Nhãn chỉ cập nhật khi số giây đổi để giảm vẽ lại.
- **Font tiếng Việt cho LVGL:** sinh font riêng cỡ 10/12/14 px chỉ gồm tập ký tự cần dùng (`vietnamese_chars.txt`) để firmware nhỏ.

**Cảm ứng GT911 (I2C).** Firmware reset GT911 bằng trình tự chân RST/INT, dò địa chỉ `0x5D` hoặc `0x14`, đọc tọa độ khi có chạm, lọc rung (ngưỡng 2 điểm ảnh).

**Máy trạng thái của thẻ đơn:**

```
ORDER_IDLE ──đơn mới──► ORDER_PENDING ──chạm "Hoàn thành"──► ORDER_COMPLETING
                              ▲                                  │ thành công: gỡ thẻ
                              │                                  │ thất bại 3 lần
                              └────── chạm "GỬI LẠI" ◄── ORDER_COMPLETE_FAILED
```

### C.4. Đồng bộ "đẩy + kéo" và kiểm tra chữ ký trên ESP32

**Lý thuyết.** **Đẩy** (MQTT) cho độ trễ thấp nhưng có thể lỡ thông điệp khi mất kết nối. **Kéo** (HTTP định kỳ) chậm hơn nhưng khôi phục được trạng thái. Kết hợp hai cách, và coi **CSDL là nguồn sự thật duy nhất**.

| Thông số | Giá trị |
|---|---|
| Thử kết nối lại Wi-Fi / MQTT | 5 s / 3 s |
| Đồng bộ ban đầu (thử lại) / đồng bộ định kỳ | 5 s / 30 s |
| HTTP báo hoàn thành: timeout kết nối / phản hồi | 2,5 s / 4 s |
| Bộ đệm MQTT | 8.192 byte (đủ cho bảng tồn kho) |
| Mã máy khách MQTT | `ESP32_Kitchen_<MAC>` |

```cpp
// kds_firmware.ino – kiểm tra phong bì HMAC v2 bằng mbedTLS
static bool constantTimeEquals(const char *a, const char *b, size_t length) {
  uint8_t diff = 0;
  for (size_t i = 0; i < length; ++i) diff |= static_cast<uint8_t>(a[i] ^ b[i]);
  return diff == 0;                       // không dừng sớm -> thời gian hằng
}

static bool verifySignedMessage(byte *payload, unsigned int length, JsonDocument &doc) {
  if (!MQTT_SECRET || !MQTT_SECRET[0]) return false;            // fail-closed
  JsonDocument envelope;
  if (deserializeJson(envelope, payload, length)) return false;
  if (envelope["v"].as<int>() != 2) return false;
  const char *inner = envelope["payload"].as<const char *>();
  const char *signature = envelope["sig"].as<const char *>();
  char expected[65];
  if (strlen(signature) != 64 || !computeHmacHex(inner, strlen(inner), expected) ||
      !constantTimeEquals(expected, signature, 64)) return false;
  return !deserializeJson(doc, inner);
}

static void mqttCallback(char *topic, byte *payload, unsigned int length) {
  JsonDocument doc;
  if (!verifySignedMessage(payload, length, doc)) return;
  if (strcmp(topic, MQTT_TOPIC_ORDER) == 0) {
    KdsOrderEvent event{};
    if (parseOrderObject(doc.as<JsonObjectConst>(), event))
      xQueueSend(orderEventQueue, &event, 0);          // KHÔNG đụng LVGL ở đây
  } else if (strcmp(topic, MQTT_TOPIC_INV) == 0) {
    if (parseInventory(doc["items"].as<JsonArrayConst>())) inventoryUiDirty = true;
  }
}
```

**Thử lại khi báo hoàn thành thất bại:** yêu cầu được đưa lại cuối hàng đợi với bộ đếm tăng dần; sau 3 lần, thẻ chuyển `ORDER_COMPLETE_FAILED` để bếp chủ động gửi lại.

---

## Phần D. Firmware thiết bị trợ lý giọng nói

### D.1. Kiến trúc phân lớp

| Thư mục | Module | Trách nhiệm |
|---|---|---|
| `src/config` | `board_config`, `audio_config`, `network_config`, `secrets` | Hằng số, kiểm tra tĩnh |
| `src/drivers` | `mic_driver`, `speaker_driver` | I2S + DMA |
| `src/audio` | `audio_filters`, `noise_estimator`, `voice_activity_detector`, `audio_ring_buffer`, `audio_recorder`, `audio_player`, `wav_format` | Xử lý tín hiệu, ghi, phát |
| `src/device` | `push_button`, `status_led`, `amplifier_control`, `battery_monitor` | Ngoại vi |
| `src/display` | `kitchen_screen`, `kitchen_display`, `display_text`, `vn_fonts` | Màn ILI9225 |
| `src/network` | `wifi_manager`, `websocket_audio_client`, `gateway_client`, `audio_response_client` | Kết nối, giao thức |
| `src/voice` | `voice_manager`, `local_command_handler` | Máy trạng thái, lệnh cục bộ |
| `src/edge_ai` | `mfcc_extractor`, `kws_feature_pipeline`, `kws_inference_engine` | Trích MFCC + suy luận từ khóa (TFLite Micro). Có module và kiểm thử nhưng **bản chạy hiện tại dùng nút nhấn để nói**, chưa bật nhận từ khóa |

Mỗi lớp chỉ phụ thuộc lớp dưới. Logic thuần (pin, font, bố cục) viết bằng C++ không phụ thuộc Arduino để chạy thử trên máy tính.

### D.2. I2S và định dạng âm thanh số

**Lý thuyết.**
- **I2S** (*Inter-IC Sound*) truyền âm thanh số nối tiếp bằng 3 dây: **BCLK** (xung bit), **WS/LRCLK** (chọn kênh trái/phải), **DATA**.
- Một **khung** gồm hai nửa (trái, phải); với khe 32 bit, `BCLK = 64 × fs`. Ở `fs = 16 kHz` thì BCLK = 1,024 MHz.
- **PCM** (*Pulse Code Modulation*): mỗi mẫu là một số nguyên biểu diễn biên độ; 16 bit có dải −32768…32767.
- **DMA**: phần cứng tự chép dữ liệu I2S vào bộ nhớ theo từng khối, CPU không phải đọc từng mẫu.

**Áp dụng.**

| Thông số | Giá trị |
|---|---|
| Tần số lấy mẫu | 16.000 Hz (chuẩn cho nhận dạng tiếng nói) |
| Định dạng gửi Gateway | PCM 16 bit, đơn kênh, little-endian |
| Khung đọc | 256 mẫu = 16 ms |
| DMA | 8 khối |
| Micro INMP441 | Xuất mẫu 24 bit trong khe 32 bit → dịch bit về PCM24 → PCM16 |
| Ampli MAX98357A | I2S riêng (I2S0), chân SD bật/tắt ampli để tránh rè |

Micro và loa dùng **hai bộ I2S khác nhau** (I2S1 và I2S0) để cấu hình độc lập.

### D.3. Chuỗi xử lý tín hiệu đầu vào (DSP)

```
INMP441 → [Chặn DC] → [Lọc thông cao 80 Hz] → [Ước lượng nhiễu nền] → [AGC] → [Giới hạn mềm] → [VAD]
```

**1. Chặn thành phần một chiều (DC blocker).** Micro có điện áp lệch làm tín hiệu không quanh 0. Bộ lọc IIR bậc 1:

$$y[n] = x[n] - x[n-1] + R \cdot y[n-1], \quad R = 0{,}995$$

```cpp
// src/audio/audio_filters.cpp
int32_t DcBlocker::processSample(int32_t inputSample) {
    const long double out = (long double)inputSample - previousInputSample_
                          + coefficient_ * previousOutputSample_;
    previousInputSample_ = inputSample;
    const int32_t clamped = clampToInt32(out);
    previousOutputSample_ = (float)clamped;
    return clamped;
}
```

**2. Lọc thông cao RC số (80 Hz).** Loại tiếng ù tần số thấp của quạt, máy bếp:

$$\alpha = \frac{RC}{RC + \Delta t},\quad RC = \frac{1}{2\pi f_c},\quad y[n] = \alpha\,(y[n-1] + x[n] - x[n-1])$$

```cpp
const float rcSeconds = 1.0f / (TWO_PI_VALUE * cutoffFrequencyHz_);
alpha_ = rcSeconds / (rcSeconds + sampleIntervalSeconds);
// processSample: out = alpha_ * (previousOutput + input - previousInput)
```

**3. Ước lượng nhiễu nền.** Trung bình trượt mũ (EMA) của RMS các khung im lặng, `α = 0,95`, hiệu chuẩn 50 khung đầu:

$$N_k = \alpha N_{k-1} + (1-\alpha)\,\text{RMS}_k$$

Trong đó $\text{RMS} = \sqrt{\frac{1}{N}\sum x_i^2}$ là năng lượng hiệu dụng của khung.

**4. Tự động điều chỉnh độ lợi (AGC).** Đưa giọng nói về mức ổn định dù người nói gần hay xa:

$$G_{\text{đích}} = \text{clip}\!\left(\frac{\text{RMS}_{\text{đích}}}{\text{RMS}_{\text{khung}}},\ 0{,}5,\ 4{,}0\right),\qquad G \leftarrow G + (G_{\text{đích}} - G)\cdot r$$

- `RMS_đích = 6000`.
- Tốc độ `r = 0,25` khi **giảm** độ lợi (*attack*, phản ứng nhanh với tiếng to đột ngột), `r = 0,02` khi **tăng** (*release*, tăng từ từ).
- **Cổng nhiễu:** khung có RMS ≤ 250 không được khuếch đại, tránh khuếch đại tiếng ồn.

```cpp
float AutomaticGainControl::calculateTargetGain(float frameRms) const {
    if (frameRms <= AudioConfig::AGC_NOISE_GATE_RMS) return 1.0f;   // cổng nhiễu
    float targetGain = targetRms_ / frameRms;
    return clamp(targetGain, minimumGain_, maximumGain_);
}
float AutomaticGainControl::updateGain(float targetGain) {
    const float rate = targetGain < currentGain_ ? AudioConfig::AGC_ATTACK_RATE
                                                 : AudioConfig::AGC_RELEASE_RATE;
    currentGain_ += (targetGain - currentGain_) * rate;
    ...
}
```

**5. Giới hạn mềm (soft limiter).** Ngưỡng 28.000: nén nhẹ phần vượt ngưỡng thay vì cắt cứng, tránh méo do xén (*clipping*).

**6. Phát hiện giọng nói (VAD).** Dựa trên năng lượng, có trễ để chống chập chờn:

| Tham số | Giá trị | Ý nghĩa |
|---|---|---|
| Ngưỡng nói | `max(3 × nhiễu nền, 700)` | Thích nghi với độ ồn của bếp |
| Tiếng nói mạnh | RMS 1.800 | Nhận ngay |
| Khung xác nhận bắt đầu / kết thúc | 3 / 8 khung | Chống bật tắt liên tục (hysteresis) |
| Im lặng kết thúc câu | 700 ms | Tự dừng ghi |
| Tiếng nói tối thiểu / tối đa | 250 ms / 8 s | Loại tiếng gõ; giới hạn bộ nhớ |
| Chờ người nói | 5 s | Nhấn nút mà không nói thì hủy |

### D.4. Bộ đệm vòng và pre-roll

**Lý thuyết.** **Bộ đệm vòng** (*ring buffer*) là mảng cố định với chỉ số ghi quay vòng; luôn giữ N mẫu gần nhất mà không cần dịch dữ liệu.

**Áp dụng.**
- Bộ đệm vòng **1 giây** liên tục nhận âm thanh.
- Khi nhấn nút, **400 ms trước đó** (*pre-roll*) được ghép vào đầu bản ghi, nên không mất âm đầu câu (người dùng hay nói ngay lúc vừa nhấn).
- Bản ghi tối đa 8 s × 16 kHz × 2 byte ≈ **256 KB**, đặt trong **PSRAM**.
- Gói thành **WAV**: header RIFF 44 byte (`RIFF`, `WAVE`, chunk `fmt` với PCM = 1, số kênh, tần số, byte/giây; chunk `data`).

### D.5. Máy trạng thái VoiceManager và hàng đợi sự kiện

**Lý thuyết.** **Máy trạng thái hữu hạn** (FSM): hệ thống ở đúng một trạng thái tại mỗi thời điểm; sự kiện gây chuyển trạng thái. Kết hợp **hàng đợi sự kiện** để xử lý tuần tự, tránh tranh chấp giữa các nguồn (nút, mạng, lỗi).

```
UNINITIALIZED → INITIALIZING → READY ──nhấn nút──► LISTENING/RECORDING ──nhả nút / VAD──►
FINALIZING_RECORDING → CONNECTING → UPLOADING → PROCESSING ──► COMMAND_AVAILABLE → EXECUTING_COMMAND
                                                         └──► PLAYING_RESPONSE → COMPLETED → READY
Bất kỳ lúc nào: hủy → CANCELLING ; lỗi mạng → RECOVERING → READY ; lỗi không phục hồi → ERROR_STATE
```

**Mã kết quả chi tiết** (`RECORDING_TOO_SHORT`, `RECORDING_EMPTY`, `NETWORK_DISCONNECTED`, `UPLOAD_TIMEOUT`, `GATEWAY_TIMEOUT`, `EMPTY_TRANSCRIPT`, `UNKNOWN_COMMAND`, `PLAYBACK_ERROR`…) giúp mỗi lỗi có cách xử lý riêng: câu quá ngắn chỉ về `READY`; mất mạng thì vào `RECOVERING`. Khi Gateway không trả lời, thiết bị phát **câu thông báo lưu sẵn** (*fallback to local prompt*).

**Chống dội nút (debounce):** nút cơ khi nhấn tạo nhiều xung nhiễu trong vài ms. Firmware chỉ chấp nhận trạng thái mới sau khi ổn định **30 ms**, bỏ lần nhấn ngắn hơn 50 ms, giới hạn giữ tối đa 15 s.

### D.6. Kết nối mạng: Wi-Fi, WebSocket, HTTP

**Wi-Fi với giãn cách lũy thừa (exponential backoff).** Khi kết nối lại thất bại liên tiếp, thời gian chờ tăng dần để không làm nghẽn mạng:

| Thông số | Giá trị |
|---|---|
| Timeout kết nối | 20 s |
| Chu kỳ thử lại | 5 s, 5 lần nhanh rồi giãn dần |
| Giãn cách tối đa | 60 s |
| Cảnh báo tín hiệu yếu | dưới −75 dBm |
| Chế độ ngủ Wi-Fi | Tắt (giảm độ trễ) |
| Mạng không mật khẩu | Từ chối |

**WebSocket (RFC 6455).**
- **Bắt tay:** yêu cầu HTTP `Upgrade: websocket` kèm `Sec-WebSocket-Key`; máy chủ trả `101 Switching Protocols` kèm `Sec-WebSocket-Accept = base64(SHA1(key + GUID))`.
- **Khung:** opcode văn bản (0x1), nhị phân (0x2), đóng (0x8), ping (0x9), pong (0xA).
- Khung từ máy khách được **che** (*mask*) bằng khóa 4 byte ngẫu nhiên.
- Thiết bị dùng thư viện `WebSocketsClient`: tự kết nối lại (3 s, giãn tối đa 30 s), **heartbeat** ping/pong, gửi `Authorization: Bearer <token>` trong tiêu đề lúc bắt tay, và *subprotocol* `smart-kiosk-voice-v1`.

```cpp
// src/network/websocket_audio_client.cpp (rút gọn)
socket_.setReconnectInterval(...);
socket_.enableHeartbeat(...);
if (authorizationHeader_.length() > 0) socket_.setExtraHeaders(authorizationHeader_.c_str());
socket_.begin(config_.host, config_.port, config_.path, config_.subprotocol);
// ... gửi khung điều khiển: socket_.sendTXT(json)   gửi âm thanh: socket_.sendBIN(chunk, size)
```

**Tải âm thanh trả lời** qua HTTP GET (cùng Bearer token), giới hạn 2 MB và 30 s.

### D.7. Âm lượng bằng số học điểm cố định (Q8) và lưu cấu hình NVS

**Lý thuyết.**
- Tai người cảm nhận âm lượng theo **thang logarit**, nên chỉnh theo **dB**: hệ số biên độ $g = 10^{\,\text{dB}/20}$.
- **Số học điểm cố định Q8**: biểu diễn số thực bằng số nguyên nhân 256 (8 bit phần lẻ). Nhân mẫu với hệ số rồi dịch phải 8 bit, nhanh hơn số thực trên vi điều khiển: `y = (x × gainQ8) >> 8`.

```cpp
// src/audio/audio_player.cpp
int32_t AudioPlayer::gainQ8ForVolume(int32_t volumePercent) {
    if (volumePercent <= 0) return 0;                 // 0% = tắt tiếng
    if (volumePercent > 100) volumePercent = 100;
    // Trên 60%: mỗi 10% thêm 1,5 dB (tối đa +6 dB, soft limiter chống rè).
    // Dưới 60%: mỗi 10% bớt 3 dB để nghe rõ sự khác biệt.
    const float decibel = volumePercent >= 60 ? (volumePercent - 60) * 0.15f
                                              : (volumePercent - 60) * 0.30f;
    return (int32_t)lroundf(256.0f * powf(10.0f, decibel / 20.0f));   // 60% -> 256 = x1,0
}
```

**NVS (Non-Volatile Storage)** là vùng Flash lưu cặp khóa–giá trị, giữ khi mất điện. Thư viện `Preferences` lưu mức âm lượng (không gian tên `"voice"`, khóa `"volume"`, mặc định 60 %, tối thiểu 10 % khi khởi động để không bị câm hẳn).

### D.8. Đo pin: ADC, cầu chia và bảng tra đường xả

**Lý thuyết.**
- **ADC** đổi điện áp thành số. ESP32-S3 có ADC1 và ADC2; **ADC2 không dùng được khi Wi-Fi bật**, nên dùng ADC1.
- **Cầu chia điện trở** đưa điện áp pin (tới 4,2 V) về dải đo: $V_{ADC} = V_{BAT} \cdot \frac{R_2}{R_1 + R_2} = V_{BAT}/2$.
- Điện áp Li-ion **không giảm tuyến tính** theo dung lượng (phần lớn dung lượng nằm ở 3,6–4,0 V), nên dùng **bảng tra + nội suy tuyến tính**.

```cpp
// src/device/battery_level.h
constexpr BatteryCurvePoint LI_ION_CURVE[] = {
    {4.20f, 100}, {4.10f, 90}, {4.00f, 80}, {3.92f, 70}, {3.85f, 60},
    {3.79f, 50},  {3.75f, 40}, {3.71f, 30}, {3.67f, 20}, {3.61f, 10},
    {3.50f, 5},   {3.30f, 0},
};
inline int batteryPercentFromVolts(float volts) {
    if (volts < BATTERY_PRESENT_MIN_VOLTS) return -1;          // không có pin (chạy USB)
    if (volts >= LI_ION_CURVE[0].volts) return 100;
    for (size_t i = 1; i < LI_ION_CURVE_SIZE; ++i) {
        const auto &upper = LI_ION_CURVE[i - 1], &lower = LI_ION_CURVE[i];
        if (volts >= lower.volts) {
            const float ratio = (volts - lower.volts) / (upper.volts - lower.volts);
            return (int)(lower.percent + ratio * (upper.percent - lower.percent) + 0.5f);
        }
    }
    return 0;
}
```

### D.9. Màn hình ILI9225: tiếng Việt khử răng cưa trên vi điều khiển

**Lý thuyết.**
- **UTF-8** mã hóa ký tự Unicode bằng 1–4 byte; byte đầu cho biết độ dài (`0xxxxxxx` = 1 byte, `110xxxxx` = 2, `1110xxxx` = 3, `11110xxx` = 4). Chữ tiếng Việt có dấu thường 2–3 byte.
- **Khử răng cưa (anti-aliasing):** mỗi điểm ảnh của nét chữ có **độ phủ** 0…1. Thay vì chỉ bật/tắt, ta **trộn màu** chữ với màu nền theo độ phủ.
- **RGB565**: màu 16 bit, đỏ 5 bit, lục 6 bit, lam 5 bit.

**Áp dụng – sinh font.** Công cụ `tools/gen_display_fonts.py` dùng Pillow dựng các ký tự cần thiết (ASCII + chữ tiếng Việt) từ font Noto Sans, lưu độ phủ **4 bit/điểm ảnh** (16 mức), nén hai điểm ảnh vào một byte. Font chỉ vài KB thay vì 2,2 MB của Unifont. Hệ số gamma 0,8 làm nét đậm hơn trên màn nhỏ.

```cpp
// src/display/display_text.h – giải mã UTF-8
inline uint32_t decodeUtf8(const char *text, size_t *index) {
    const uint8_t first = (uint8_t)text[*index];
    if (first < 0x80) { *index += 1; return first; }
    size_t length = 0; uint32_t codePoint = 0;
    if ((first & 0xE0) == 0xC0)      { length = 2; codePoint = first & 0x1F; }
    else if ((first & 0xF0) == 0xE0) { length = 3; codePoint = first & 0x0F; }
    else if ((first & 0xF8) == 0xF0) { length = 4; codePoint = first & 0x07; }
    else { *index += 1; return '?'; }
    for (size_t off = 1; off < length; ++off)
        codePoint = (codePoint << 6) | (text[*index + off] & 0x3F);   // 6 bit dữ liệu mỗi byte sau
    *index += length;
    return codePoint;
}

// src/display/kitchen_screen.cpp – trộn màu RGB565 theo độ phủ 0..15
uint16_t blend(uint16_t fg, uint16_t bg, uint8_t alpha15) {
    if (alpha15 == 0) return bg;
    if (alpha15 >= 15) return fg;
    const int32_t r = ((bg >> 11) & 0x1F) + ((((fg >> 11) & 0x1F) - ((bg >> 11) & 0x1F)) * alpha15) / 15;
    const int32_t g = ((bg >> 5) & 0x3F) + ((((fg >> 5) & 0x3F) - ((bg >> 5) & 0x3F)) * alpha15) / 15;
    const int32_t b = (bg & 0x1F) + (((fg & 0x1F) - (bg & 0x1F)) * alpha15) / 15;
    return (uint16_t)((r << 11) | (g << 5) | b);
}
// Lấy độ phủ của điểm ảnh i từ byte nén: alpha = (i % 2 == 0) ? (packed >> 4) : (packed & 0x0F)
```

**Kỹ thuật khác:**
- **Xuống dòng theo từ** (`wrapText`): không cắt giữa ký tự UTF-8.
- **Nhận dạng câu dạng danh sách:** câu trả lời "X gồm: a, b, c" được hiển thị thành danh sách.
- **Tách giao diện khỏi phần cứng:** lớp `KitchenScreen` vẽ qua giao diện `Canvas` trừu tượng, có bản xem trước chạy trên máy tính.
- **SPI** 27 MHz qua FSPI, chân gán bằng ma trận GPIO.

---

## Phần E. Voice Gateway

### E.1. Kiến trúc và FastAPI

**Lý thuyết.**
- **ASGI** là chuẩn máy chủ web bất đồng bộ của Python, kế thừa WSGI.
- **FastAPI** xây trên **Starlette** (định tuyến, WebSocket) và **Pydantic** (kiểm tra dữ liệu), chạy bằng **Uvicorn**.
- **Vòng lặp sự kiện (event loop):** khi gặp `await`, hàm tạm nhường quyền để phục vụ yêu cầu khác. Một tiến trình giữ được rất nhiều kết nối mở lâu dài.
- Việc **nặng về CPU** (Whisper, Piper) phải đẩy sang **luồng riêng** (`asyncio.to_thread`/executor), nếu không sẽ chặn cả vòng lặp.

**Cấu trúc gói theo trách nhiệm (kiến trúc phân lớp):**

| Gói | Trách nhiệm |
|---|---|
| `transport` | Nhận/gửi WebSocket, quản lý phiên |
| `protocol` | Hợp đồng thông điệp, lỗi, luồng âm thanh |
| `stt` | Faster-Whisper |
| `nlu` | Chuẩn hóa văn bản, danh mục lệnh, so khớp lệnh |
| `orchestration` | Router tất định, Ollama, định dạng câu trả lời |
| `mcp` | Máy khách MCP |
| `tool_policy` | Chính sách, xác nhận, nhật ký |
| `tts` | Piper, chuẩn hóa phát âm, đọc số |
| `audio_store` | Kho WAV trong bộ nhớ |
| `application`, `api`, `config` | Ghép nối, FastAPI, cấu hình |

Các gói xử lý **không phụ thuộc FastAPI hay WebSocket**, nên kiểm thử độc lập được (*dependency injection*: phụ thuộc truyền vào qua hàm khởi tạo).

**Khởi động trước (warm-up) bằng `lifespan`.** Mô hình nạp lần đầu rất chậm: lượt hỏi đầu tiên từng mất khoảng 30 s. Gateway nạp và **chạy thử** STT, TTS và LLM ngay lúc khởi động, trong luồng nền:

```python
# gateway/src/voice_gateway/orchestration/orchestrator.py
async def warm_up(self) -> None:
    """Chạy thử một lượt chat với đúng system prompt và danh sách tool.
    Chỉ nạp model thì lượt LLM đầu tiên vẫn mất ~30 s vì Ollama phải xử lý prompt dài
    lần đầu; các lượt sau ~3-4 s nhờ Ollama dùng lại phần prompt đã xử lý."""
    await self._load_tools()
    await self._chat([ToolChatMessage(role="system", content=self._system_prompt),
                      ToolChatMessage(role="user", content=WARM_UP_USER_TEXT)], 0.0)
```

Kỹ thuật này dựa trên **bộ đệm KV của tiền tố prompt** (*prompt prefix caching*): Ollama giữ kết quả đã tính cho phần đầu prompt giống nhau. Lần sau chỉ cần tính phần câu hỏi mới, nên lượt hỏi đầu tiên giảm từ khoảng 30 s xuống khoảng 3,5 s.

### E.2. Giao thức WebSocket `smart-kiosk-voice` v1.0.0

| Thuộc tính | Giá trị |
|---|---|
| Endpoint | `/ws/voice` |
| Subprotocol | `smart-kiosk-voice-v1` |
| Xác thực | `Authorization: Bearer <token>` khi bắt tay |
| Âm thanh | PCM 16 bit, 16 kHz, đơn kênh, little-endian |

Trình tự một lượt:

```
Thiết bị                                   Gateway
   │── session_start (JSON) ─────────────────►│
   │◄──────────────────────── session_ready ──│
   │── audio_stream_begin {request_id,        │
   │     session_id, request_sequence, fmt} ──►│
   │── khung nhị phân PCM … ─────────────────►│  (kiểm tra tổng dung lượng, thứ tự)
   │── audio_stream_end {total_bytes} ───────►│
   │                                          │  STT → NLU → Router/LLM → MCP → TTS
   │◄── final_transcript / command_result / error
   │── GET /audio/<id> (Bearer) ─────────────►│  tải WAV trả lời
```

**Kỹ thuật giao thức:**
- `request_id` gắn phản hồi với đúng yêu cầu; `session_id` gom các lượt của một phiên; `request_sequence` tăng dần để phát hiện thông điệp lệch thứ tự.
- Có `cancel_request`, `ping`/`pong`, `session_stop`/`session_end`.
- **Giữ phiên** cho nhiều lượt hỏi liên tiếp, không phải bắt tay lại; ngữ cảnh phiên giữ cả phiếu xác nhận đang chờ.
- **Phục hồi luồng:** sau mỗi lỗi, handler đưa luồng âm thanh về trạng thái `IDLE` để lượt sau không bị kẹt.

### E.3. Nhận dạng tiếng nói: Whisper và Faster-Whisper

**Lý thuyết Whisper.**
- Mô hình của OpenAI (2022), huấn luyện trên khoảng **680.000 giờ** âm thanh có phụ đề, gần 100 ngôn ngữ.
- **Tiền xử lý:** âm thanh 16 kHz, chia đoạn 30 s, đổi thành **phổ Mel logarit** gồm 80 dải; khung 25 ms, bước 10 ms. Thang Mel mô phỏng tai người: phân giải mịn ở tần thấp, thô ở tần cao.
- **Transformer mã hóa – giải mã:**
  - bộ mã hóa đọc phổ Mel;
  - bộ giải mã sinh từng **token**, dựa trên **cơ chế chú ý** (*attention*);
  - token đặc biệt chọn ngôn ngữ (`<|vi|>`) và nhiệm vụ (`<|transcribe|>`).
- **Tìm kiếm chùm (beam search):** giữ B chuỗi ứng viên tốt nhất ở mỗi bước, thay vì chỉ chọn token xác suất cao nhất (*greedy*).
- **Độ đo WER:** $\text{WER} = \frac{S + D + I}{N}$ (thay thế + xóa + chèn, chia số từ câu đúng). Ví dụ "công thức xôi lạc" bị nghe thành "công thước xô lạc": 2/4 = 50 %.

**Faster-Whisper:** chạy Whisper trên **CTranslate2**, nhanh hơn khoảng 4 lần và ít bộ nhớ hơn.
- **Lượng tử hóa int8:** trọng số 32/16 bit đổi thành số nguyên 8 bit, nhỏ hơn 2–4 lần, tính nhanh trên CPU.
- **VAD Silero:** bỏ đoạn im lặng, giảm thời gian và giảm hiện tượng "bịa chữ" (*hallucination*).
- **`hotwords` / `initial_prompt`:** đưa tên món vào ngữ cảnh bộ giải mã, ưu tiên nhận đúng tên riêng.

| Tham số | Giá trị |
|---|---|
| `model` | `small` (244 triệu tham số) |
| `device` / `compute_type` | `cpu` / `int8` |
| `language` | `vi` (bỏ bước tự nhận ngôn ngữ) |
| `beam_size` | 5 |
| `vad_filter`, `min_silence_duration_ms` | bật, 500 |
| `condition_on_previous_text` | tắt (tránh lặp chữ giữa các đoạn) |

```python
# gateway/src/voice_gateway/stt/faster_whisper_service.py (rút gọn)
pcm_samples = np.frombuffer(audio_bytes, dtype="<i2")          # PCM16 little-endian
waveform = pcm_samples.astype(np.float32)
waveform /= PCM16_NORMALIZATION_DIVISOR                        # 32768 -> dải [-1, 1)
segments, information = model.transcribe(
    waveform, language="vi", beam_size=5, vad_filter=True,
    vad_parameters={"min_silence_duration_ms": config.whisper_vad_min_silence_ms},
    condition_on_previous_text=False, **hint_options)           # hotwords / initial_prompt

# Độ tin cậy từ xác suất log trung bình của mỗi đoạn
@staticmethod
def _log_probability_to_confidence(log_probability: float) -> float:
    if not math.isfinite(log_probability):
        return 0.0
    return max(0.0, min(1.0, math.exp(log_probability)))
```

### E.4. Hiểu lệnh tất định (Deterministic NLU)

**Lý thuyết.**
- **Chuẩn hóa Unicode NFC:** chữ tiếng Việt có thể biểu diễn kiểu *tổ hợp sẵn* (NFC) hoặc *ký tự gốc + dấu rời* (NFD). Phải đưa về cùng một dạng trước khi so sánh, nếu không "ế" ≠ "ế".
- **So khớp mờ bằng `difflib.SequenceMatcher`:** tỷ lệ giống $\frac{2M}{|a| + |b|}$, trong đó M là số ký tự khớp theo các khối con chung dài nhất.
- **Từ dừng, từ đệm:** "ơi", "nhé", "giúp tôi"… được bỏ ở đầu/cuối câu.
- **Phủ định:** "đừng tắt loa" không được hiểu là "tắt loa". Có dấu hiệu phủ định thì trả UNKNOWN (thận trọng).

**Ba tầng so khớp lệnh thiết bị:**

```python
# gateway/src/voice_gateway/nlu/command_matcher.py (rút gọn)
def normalize_text(text: str) -> str:
    """NFC, chữ thường, dấu câu -> khoảng trắng, rút gọn khoảng trắng. Giữ dấu tiếng Việt."""
    normalized = unicodedata.normalize("NFC", text).strip().lower()
    normalized = _PUNCTUATION_PATTERN.sub(" ", normalized)
    return _SPACE_PATTERN.sub(" ", normalized).strip()

def match(self, text: str) -> CommandMatch:
    normalized = normalize_text(text)
    core = strip_particles(collapse_repeated_utterance(normalized))   # "X X" -> "X"
    exact = self._exact.get(core)
    if exact is not None:                         # 1. khớp nguyên văn
        return self._result(exact, 1.0, MATCH_STRATEGY_EXACT)
    if contains_negation(core):                   # phủ định -> không đoán
        return _UNKNOWN_MATCH
    contained = self._contained_match(core)       # 2. câu mẫu nằm trọn trong câu nói
    if contained is not None:                     #    (phải chiếm > 40% số từ nội dung)
        return contained
    return self._fuzzy_match(core)                # 3. tương đồng ký tự + thưởng từ khóa

def _fuzzy_match(self, text):
    for item in self._partial:
        similarity = SequenceMatcher(None, text, item.normalized, autojunk=False).ratio()
        hits = sum(1 for kw in self._keywords[item.definition.command] if kw in f" {text} ")
        confidence = min(1.0, similarity + min(0.08, hits * 0.03))
        ...
    # dưới ngưỡng tin cậy của lệnh -> UNKNOWN
```

| Ý định | Ví dụ |
|---|---|
| `volume_up` / `volume_down` | "tăng âm lượng", "cho loa to lên" / "nhỏ tiếng xuống" |
| `speaker_off` / `speaker_on` | "tắt loa", "im lặng" / "bật loa" |
| `confirm` / `cancel` | "xác nhận", "đồng ý" / "hủy", "bỏ qua" |
| `repeat`, `help` | "nhắc lại", "trợ giúp" |

Gateway chỉ trả **mã lệnh**; thiết bị tự thực hiện (thay đổi Q8 gain, lưu NVS).

### E.5. Router nghiệp vụ tất định bằng biểu thức chính quy

**Lý thuyết.**
- **Biểu thức chính quy (regex):** mô tả mẫu chuỗi; *nhóm có tên* `(?P<name>...)` trích phần cần lấy.
- *Lượng từ lười* `.+?` lấy ngắn nhất có thể.
- `(?:...)` là nhóm không ghi nhớ.

**Áp dụng.** Câu hỏi rõ nghĩa đi **thẳng** tới công cụ MCP, không qua LLM, nên nhanh và không phụ thuộc mô hình:

```python
# gateway/src/voice_gateway/orchestration/kitchen_query_router.py (rút gọn)
OPERATIONAL_SUMMARY_TOOL = "get_operational_kitchen_summary"
RECIPE_BY_NAME_TOOL = "get_product_recipe_by_name"

_PHRASES_BY_MODE = {
    MODE_QUEUE_COUNT: ("có bao nhiêu đơn đang chờ", "bếp đang có mấy đơn chờ", ...),
    MODE_LOW_STOCK:   ("nguyên liệu nào sắp hết", ...),
    ...
}

# Whisper hay nghe "công thức" thành "công đức", "câu thức", "công thước".
_RECIPE_WORD = r"(?:công|cong|côn|câu|không)\s+(?:thức|thuc|thước|đức|thứ|thực)"
_RECIPE_LEAD = r"(?:(?:của|để\s+làm|làm|cho|món)\s+)*"
_RECIPE_TAIL = r"(?:\s+(?:là\s+gì|như\s+thế\s+nào|thế\s+nào|ra\s+sao|gồm\s+những\s+gì|gồm\s+gì))?"
patterns = (
    r"{RECIPE}\s+món\s+(?P<product_name>.+?)\s+là\s+gì",
    r"(?P<product_name>.+?)\s+gồm\s+những\s+nguyên\s+liệu\s+gì",
    r"cho\s+tôi\s+{RECIPE}\s+{LEAD}(?P<product_name>.+?){TAIL}",
    r"{RECIPE}\s+{LEAD}(?P<product_name>.+?){TAIL}",
)
```

**Kỹ thuật:**
- **Câu mẫu đi qua cùng `normalize_text` với câu người dùng**, nên hai phía luôn so khớp trên cùng một dạng.
- **Kiểm tra trùng lúc nạp:** một câu mẫu thuộc hai chế độ thì báo lỗi ngay.
- **Mẫu được bổ sung từ log thật:** các cách nghe nhầm "công thức" được thêm sau khi phân tích log chạy thử.

**Định dạng câu trả lời bằng khuôn mẫu** (`formatters.py`): số liệu đọc đúng như dữ liệu, độ dài ổn định khi đọc to, ví dụ "Hiện có 3 đơn đang chờ, đơn chờ lâu nhất là … đã chờ 12 phút."

### E.6. Mô hình ngôn ngữ lớn: Ollama, Qwen2.5 và gọi công cụ

**Lý thuyết LLM.**
- Mạng **Transformer** hàng tỷ tham số, huấn luyện để **dự đoán token tiếp theo**.
- **Token:** mảnh từ; tiếng Việt tốn nhiều token hơn tiếng Anh.
- **Cửa sổ ngữ cảnh:** số token tối đa xét cùng lúc.
- **Nhiệt độ:** chia logit trước softmax. Nhiệt độ thấp thì gần như chọn token xác suất cao nhất, câu trả lời ổn định; nhiệt độ cao thì đa dạng hơn nhưng dễ sai.
- **System prompt:** chỉ dẫn vai trò và quy tắc.
- **Lượng tử hóa GGUF (Q4_K_M):** khoảng 4 bit/tham số; mô hình 3B từ khoảng 6 GB còn khoảng 1,9 GB, chạy được trên CPU.
- **Ảo giác (hallucination):** mô hình "bịa" thông tin. Giảm bằng cách buộc dùng dữ liệu từ công cụ.

**Gọi công cụ (tool calling) – vòng lặp tác tử (agent loop):**
1. Gửi câu hỏi + danh sách công cụ (tên, mô tả, JSON Schema tham số).
2. Mô hình trả **yêu cầu gọi công cụ** thay vì câu trả lời.
3. Chương trình thực thi, gửi kết quả lại với vai trò `tool`.
4. Mô hình viết câu trả lời cuối.

```python
# gateway/src/voice_gateway/orchestration/orchestrator.py (rút gọn)
messages = [ToolChatMessage(role="system", content=self._system_prompt),
            ToolChatMessage(role="user", content=user_text)]

route = self._query_router.route(user_text) if enforce_policy else None
if route is not None:                                   # câu rõ nghĩa: bỏ qua LLM
    response = ToolChatResponse(content="", model="deterministic-kitchen-router",
        tool_calls=(RequestedToolCall(name=route.tool_name, arguments=route.arguments),))
else:
    response = await self._chat(messages, temperature)  # Ollama chọn công cụ

for _ in range(self._max_tool_rounds):
    if not response.tool_calls:
        return _completed(response)
    messages.append(ToolChatMessage(role="assistant", content=response.content,
                                    tool_calls=response.tool_calls))
    for requested_call in response.tool_calls:
        tool = await self._require_tool(requested_call.name)
        mcp_call = McpToolCall(name=requested_call.name, arguments=dict(requested_call.arguments))
        guard_result = self._execution_guard.preflight(tool=tool, tool_call=mcp_call, ...)
        if not guard_result.is_allowed:                 # bị từ chối / cần xác nhận
            return _policy_outcome(requested_call.name, guard_result)
        result = await self._execute(mcp_call, execution_context, audit=enforce_policy)
        formatted = _format_deterministic(requested_call.name, result, route)
        if formatted is not None:                       # có khuôn mẫu -> trả lời ngay
            return OrchestrationOutcome(response=formatted, ...)
        messages.append(ToolChatMessage(role="tool", tool_name=requested_call.name,
                                        content=_serialize_tool_result(result)))
    response = await self._chat(messages, temperature)
```

**Cấu hình Ollama và lỗi thực tế `keep_alive`.**
- `keep_alive` giữ mô hình trong RAM. Ollama nhận **số nguyên giây** (âm là giữ mãi) hoặc **chuỗi có đơn vị** (`"30m"`).
- Chuỗi số trần `"-1"` (đọc từ biến môi trường) bị trả **HTTP 400**, nên phải chuyển sang int:

```python
# gateway/src/voice_gateway/orchestration/ollama_tool_client.py
def _normalize_keep_alive(value: object) -> int | str | None:
    if value is None:
        return None
    if isinstance(value, bool):
        raise TypeError("keep_alive must be a string or an integer")
    if isinstance(value, int):
        return value
    text = str(value).strip()
    try:
        return int(text)       # "-1" -> -1
    except ValueError:
        return text            # "30m" giữ nguyên
```

| Tham số | Giá trị |
|---|---|
| Mô hình | `qwen2.5:3b` (Alibaba, đa ngôn ngữ, hỗ trợ tool calling) |
| Nhiệt độ | 0,2 |
| Timeout | 120 s |
| Bộ đệm danh sách công cụ MCP | 60 s (không gọi `tools/list` mỗi câu) |
| `keep_alive` | −1 (giữ mãi) |

**Vai trò của LLM: chỉ là dự phòng.** Câu khớp lệnh thiết bị hoặc router thì không gọi LLM. Cách này nhanh hơn, không tốn tài nguyên, và tránh mô hình nhỏ đọc sai số liệu.

### E.7. Kiểm soát thao tác ghi (Tool policy)

**Lý thuyết.**
- **Phòng thủ nhiều lớp (defense in depth):** nhiều cơ chế độc lập, một lớp thủng vẫn còn lớp khác.
- **Mặc định từ chối (fail-closed).**
- **Phiếu xác nhận dùng một lần, có hạn:** chống phát lại.
- **Nhật ký kiểm toán (audit log):** che các trường nhạy cảm.

```python
# gateway/src/voice_gateway/tool_policy/policy.py (rút gọn)
class ToolPolicyAction(str, Enum):
    ALLOW = "allow"
    DENY = "deny"
    REQUIRE_CONFIRMATION = "require_confirmation"

def evaluate(self, tool, ...):
    if tool.read_only_hint:                                   # 1. công cụ chỉ đọc
        return ToolPolicyDecision(ToolPolicyAction.ALLOW, "read_only_tool", tool.name)
    if authorization_enabled and not authorized:              # 2. thiết bị không có quyền
        return ToolPolicyDecision(ToolPolicyAction.DENY, "tool_not_authorized", tool.name)
    if confirmation_enabled and not confirmed:                # 3. chưa xác nhận
        return ToolPolicyDecision(ToolPolicyAction.REQUIRE_CONFIRMATION, "confirmation_required", tool.name)
    return ToolPolicyDecision(ToolPolicyAction.ALLOW, "write_tool_approved", tool.name)
```

| Lớp | Cơ chế |
|---|---|
| Xác thực thiết bị | Bearer token trên WebSocket và endpoint âm thanh |
| Phân quyền | Chỉ thiết bị trong `TOOL_AUTHORIZED_DEVICE_IDS` được gọi công cụ ghi |
| Xác nhận | Phiếu `confirmation_id` sống **60 s**, gắn với thiết bị + phiên + tham số, dùng **một lần** |
| Nhật ký | Mọi quyết định ALLOW/DENY/REQUIRE_CONFIRMATION |
| Cấu hình an toàn | Từ chối cấu hình bật ghi tự động khi tắt phân quyền/xác nhận/nhật ký |

Ví dụ hội thoại:
1. "Hoàn thành đơn DH12345678."
2. Gateway: "Bạn muốn hoàn thành đơn DH12345678? Hãy nói xác nhận hoặc hủy."
3. "Xác nhận." (trong 60 s) → tiêu thụ phiếu, gọi `mark_order_done`, ghi nhật ký.

### E.8. Tổng hợp tiếng nói: Piper và xử lý văn bản tiếng Việt

**Lý thuyết Piper.**
1. Văn bản → **âm vị** bằng espeak-ng.
2. Âm vị → **dạng sóng** bằng mạng **VITS**: mô hình đầu–cuối, gồm bộ mã hóa biến phân có điều kiện, *normalizing flow* và huấn luyện đối kháng.
3. Chạy bằng **ONNX Runtime** trên CPU.

Giọng `vi_VN-vais1000-medium`: tệp `.onnx` + `.json`, xuất 22,05 kHz.

**Chuẩn hóa đọc số.** espeak-ng đọc sai một số trường hợp đã đo trực tiếp:

| Đầu vào | espeak-ng đọc | Đúng |
|---|---|---|
| `0.5` | "không chấm năm" | "không phẩy năm" |
| `15` | "mười năm" | "mười lăm" |
| `1000` | "một ngàn không trăm" | "một nghìn" |

Module `vietnamese_numbers.py` chỉ đổi cụm `<số> <đơn vị>` với đơn vị gam, mi-li-lít, quả, cái, nên mã đơn, giờ, ngày không bị đụng tới:

```python
# gateway/src/voice_gateway/tts/vietnamese_numbers.py (rút gọn)
_DIGIT_WORDS = ("không", "một", "hai", "ba", "bốn", "năm", "sáu", "bảy", "tám", "chín")
_GROUP_NAMES = ("", "nghìn", "triệu", "tỷ")
_QUANTITY_PATTERN = re.compile(
    r"(?<![\w.,])(\d{1,12})(?:\.(\d{1,4}))?[ \t]+(?=(?:gam|mi-li-lít|quả|cái)(?!\w))",
    flags=re.IGNORECASE)

def _below_hundred(value: int) -> str:
    if value < 10:
        return _DIGIT_WORDS[value]
    tens, unit = divmod(value, 10)
    if tens == 1:
        return "mười" if unit == 0 else f"mười {'lăm' if unit == 5 else _DIGIT_WORDS[unit]}"
    ...   # mốt (21), lăm (25), tư (24) theo quy tắc tiếng Việt
```

`vietnamese_pronunciation.py` phiên âm tên nước ngoài (ví dụ "Tiramisu" → "Ti-ra-mi-su") và đơn vị ("ml" → "mi-li-lít").

**Lấy mẫu lại 22,05 kHz → 16 kHz (resampling).**
- **Định lý Nyquist:** tín hiệu lấy mẫu ở `fs` chỉ biểu diễn đúng tần số dưới `fs/2`.
- Khi **hạ** tần số lấy mẫu, thành phần trên Nyquist mới (8 kHz) sẽ **gập** (*aliasing*) thành tạp âm. Vì vậy phải **lọc thông thấp trước**.
- Bộ lọc dùng là **FIR cửa sổ Kaiser**: nhân hàm `sinc` (bộ lọc lý tưởng) với cửa sổ Kaiser để cắt ngắn mà ít gợn. FIR đối xứng nên **pha tuyến tính**, không méo dạng sóng.

```python
# gateway/src/voice_gateway/tts/piper_service.py (rút gọn)
def _lowpass_before_downsampling(signal, *, source_sample_rate_hz, target_sample_rate_hz):
    taps = RESAMPLE_FILTER_TAPS
    cutoff = RESAMPLE_FILTER_CUTOFF_RATIO * target_sample_rate_hz / source_sample_rate_hz
    offsets = np.arange(taps, dtype=np.float64) - (taps - 1) / 2.0
    kernel = 2.0 * cutoff * np.sinc(2.0 * cutoff * offsets) * np.kaiser(taps, RESAMPLE_FILTER_KAISER_BETA)
    kernel /= kernel.sum()                               # độ lợi DC = 1
    padded = np.pad(signal, taps // 2, mode="edge")      # đệm biên, không tạo xung đầu/cuối
    return np.convolve(padded, kernel, mode="valid")

def _resample_pcm16(pcm, *, source_sample_rate_hz, target_sample_rate_hz):
    target_count = max(1, int(round(pcm.size * target_sample_rate_hz / source_sample_rate_hz)))
    source_positions = np.arange(pcm.size, dtype=np.float64)
    target_positions = np.linspace(0.0, float(pcm.size - 1), num=target_count)
    signal = pcm.astype(np.float64)
    if target_sample_rate_hz < source_sample_rate_hz:
        signal = _lowpass_before_downsampling(signal, ...)
    resampled = np.interp(target_positions, source_positions, signal)   # nội suy tuyến tính
    return np.clip(np.rint(resampled), -32768, 32767).astype(np.int16)
```

**Kho âm thanh trong bộ nhớ** (`audio_store/in_memory.py`):
- tối đa **64** tệp, tổng 32 MB, mỗi tệp sống **300 s**, tự dọn tệp cũ;
- thiết bị tải qua HTTP với Bearer token;
- Gateway không ghi nội dung câu trả lời ra log.

---

## Phần F. Kitchen MCP

### F.1. Model Context Protocol

**Lý thuyết.**
- **MCP** (Anthropic, 11/2024) là chuẩn mở kết nối ứng dụng AI với công cụ và dữ liệu.
- **Máy chủ MCP** công bố ba loại khả năng: **tools** (hàm gọi được), **resources** (dữ liệu đọc được), **prompts** (mẫu lời nhắc).
- **Máy khách MCP** nằm trong ứng dụng AI.
- Thông điệp **JSON-RPC 2.0**: `{"jsonrpc":"2.0","id":7,"method":"tools/call","params":{...}}`; phản hồi cùng `id`.
- Vòng đời: `initialize` → `tools/list` → `tools/call`.
- Kênh truyền: **stdio** (tiến trình con) hoặc **Streamable HTTP** (dịch vụ mạng).
- **Chú thích công cụ:** `readOnlyHint`, `destructiveHint`, `openWorldHint` giúp máy khách biết công cụ có đổi trạng thái hay không.

```python
# services/kitchen_mcp/src/kitchen_mcp/server.py (rút gọn)
from mcp.server import MCPServer                 # MCP Python SDK 2.x
from mcp.types import ToolAnnotations

mcp = MCPServer(
    name="Kitchen MCP",
    description="Read-only MCP server cung cấp dữ liệu trạng thái bếp và đơn hàng.",
    instructions="Chỉ sử dụng dữ liệu do các tool cung cấp. Không thay đổi trạng thái đơn hàng.",
)

@mcp.tool(
    title="Get product recipe by name",
    annotations=ToolAnnotations(readOnlyHint=True, destructiveHint=False, openWorldHint=True),
)
def get_product_recipe_by_name(product_name: str):
    """Tìm món theo tên và lấy recipe."""
    return _data_source.get_product_recipe_by_name(product_name)

# Công cụ ghi chỉ đăng ký khi KITCHEN_WRITE_ENABLED và có khóa API hợp lệ:
#   annotations=ToolAnnotations(readOnlyHint=False, destructiveHint=True, ...)

mcp.run(transport="streamable-http")   # cổng 3001, đường dẫn /mcp
```

SDK tự sinh **JSON Schema** tham số từ chú thích kiểu Python (`product_name: str`) và lấy mô tả công cụ từ docstring. Trường `instructions` của máy chủ được gửi cho máy khách khi `initialize`, nhắc mô hình chỉ dùng dữ liệu từ công cụ.

### F.2. Danh mục công cụ và nguồn dữ liệu

| Nhóm | Công cụ (19) |
|---|---|
| Trạng thái | `get_data_mode`, `get_kitchen_status`, `get_operational_kitchen_summary` |
| Đơn hàng | `get_open_orders`, `get_queue_summary`, `get_oldest_pending_order`, `get_order_status` |
| Đơn gần đây | `get_recent_orders`, `summarize_recent_orders`, `summarize_recent_order_items` |
| Kho | `get_inventory`, `get_low_stock_ingredients`, `calculate_pending_order_demand`, `check_pending_orders_inventory` |
| Thực đơn, công thức | `get_menu`, `find_product`, `get_product_recipe`, `get_product_recipe_by_name`, `calculate_recipe_requirements(_by_name)` |
| Ghi (có điều kiện) | `mark_order_done` |

**Kỹ thuật:**
- **Hai chế độ dữ liệu:** `fixture` (dữ liệu mẫu cho kiểm thử) và `live` (gọi Smart Kiosk API, timeout 5 s). Khai báo bằng `KITCHEN_DATA_MODE=live`.
- **Công cụ tổng hợp sẵn:** `get_operational_kitchen_summary` trả trong **một lần gọi** số đơn chờ, đơn cũ nhất, đơn trễ (chờ trên 15 phút), nguyên liệu sắp hết, kiểm tra kho cho đơn chờ. Ít lượt gọi, ít độ trễ, LLM không phải tự tính.
- **Tra cứu trạng thái đơn dự phòng:** nếu đơn không còn trong danh sách đang chờ, tìm trong lịch sử đơn.
- **Món bán chạy:** chỉ tính **đơn đã thanh toán**, xét tối đa 100 đơn gần nhất.

### F.3. Nhận diện tên món chịu lỗi nhận dạng giọng nói

**Bài toán.** Whisper nghe "trà chanh" thành "chà chanh", "xôi lạc" thành "xô lạc", "nước mía" thành "nước miếng", "bánh flan" thành "bánh phở len". Hệ thống cần **đoán đúng món** nhưng **không được đoán bừa**: đọc nhầm công thức tệ hơn trả lời "không tìm thấy".

**Hai tầng (tự tính lại từ thực đơn mỗi khi thực đơn đổi, không cần danh sách viết tay):**

**Tầng 1 – Từ khóa (ưu tiên độ chính xác):**
- mỗi từ trong câu hỏi phải là từ có trong tên một món;
- đúng **một** món chứa đủ các từ đó thì mới nhận;
- bỏ dấu thanh khi so sánh nhưng **giữ dấu đổi nghĩa** (ơ, ô, ê, ă, â, ư), nên "bơ" không lẫn "bò".

**Tầng 2 – Theo âm (cho lỗi nhận dạng):**
- **Chuẩn hóa âm tiết:** bỏ dấu, gộp phụ âm đầu đồng âm theo phương ngữ: `tr=ch`, `gi=d=r`, `x=s`, `ph=f`, `c=k=q`, `ngh=ng`, `gh=g`.
- **Khoảng cách sửa chữ có trọng số** (Levenshtein biến thể, quy hoạch động): chèn/xóa 1,0; đổi nguyên âm sang nguyên âm khác **0,6** (người nghe hay nhầm nguyên âm); đổi khác 1,0.
- **Độ tương đồng:** $1 - \frac{d(a,b)}{\max(|a|,|b|)}$.
- **Nhận khi:** đủ giống (≥ ngưỡng), **đủ cách biệt** với món thứ hai (*margin*), và không chứa từ thuộc món khác.
- Tên ngắn ("Xôi gà") có ngưỡng riêng: ≥ 0,80 và cách biệt ≥ 0,15.
- **Sửa lỗi theo log thật:** `STT_CONFUSIONS = {"nước miếng": "nước mía"}`, thay trước mọi bước.

```python
# services/kitchen_mcp/src/kitchen_mcp/menu_keywords.py (rút gọn)
_ONSET_RULES = (("ngh", "ng"), ("gh", "g"), ("tr", "ch"), ("gi", "d"),
                ("ph", "f"), ("qu", "kw"), ("x", "s"), ("r", "d"))

def _normalize_syllable(token: str) -> str:
    syllable = _fold(token)                      # bỏ dấu (NFD, lọc ký tự dấu)
    for source, target in _ONSET_RULES:
        if syllable.startswith(source):
            return target + syllable[len(source):]
    if syllable[:1] in ("c", "q") and not syllable.startswith("ch"):
        return "k" + syllable[1:]
    return syllable

def _distance(first: str, second: str) -> float:
    """Quy hoạch động: chèn/xóa 1,0; đổi nguyên âm<->nguyên âm 0,6; đổi khác 1,0."""
    previous = [float(j) for j in range(len(second) + 1)]
    for i, a in enumerate(first, 1):
        current = [float(i)]
        for j, b in enumerate(second, 1):
            sub = 0.0 if a == b else (0.6 if a in _VOWELS and b in _VOWELS else 1.0)
            current.append(min(previous[j] + 1.0, current[j - 1] + 1.0, previous[j - 1] + sub))
        previous = current
    return previous[-1]

def explain_phonetic(query, menu_names, *, threshold, margin, ...) -> PhoneticResult:
    key = "".join(_strip_edges([_normalize_syllable(t) for t in _clean_query(query).split()]))
    ranked = sorted(((_similarity(key, _phonetic_key(n)), n) for n in names), reverse=True)
    best_score, best_name = ranked[0]
    runner_score, runner_name = ranked[1] if len(ranked) > 1 else (0.0, None)
    if best_score < threshold:                return reject("điểm thấp")
    if best_score - runner_score < margin:    return reject("quá giống món thứ hai")
    if _foreign_syllables(...):               return reject("có từ của món khác")
    return accept(best_name)
```

**Minh bạch quyết định:**
- Hàm `explain_phonetic` trả cả lý do từ chối; công cụ dòng lệnh `python -m kitchen_mcp.menu_keywords --explain "nước miếng ép"` để chỉnh ngưỡng.
- Câu trả lời đọc to **tên món đã chọn**, nên người nghe biết hệ thống hiểu món nào.

---

## Phần G. Kỹ thuật thiết kế xuyên suốt

| Nguyên tắc | Ý nghĩa | Ở đâu trong dự án |
|---|---|---|
| **Một nguồn sự thật** | CSDL là trạng thái chính thức; thiết bị nghi ngờ thì kéo lại qua API | KDS đồng bộ 30 s; MCP đọc qua API, không đọc file CSDL |
| **Fail-closed** | Thiếu cấu hình bảo mật thì từ chối | API (503), webhook, MQTT ký, Gateway token, MCP ghi |
| **Lũy đẳng** | Xử lý lặp lại một sự kiện không gây tác dụng phụ | Thanh toán hai đường; QoS 1 + khử trùng ở KDS; hoàn thành đơn 409 |
| **Phòng thủ nhiều lớp** | Nhiều cơ chế độc lập | Khóa API hai cấp, HMAC, Bearer, phân quyền, xác nhận, nhật ký |
| **Đặc quyền tối thiểu** | Mỗi thành phần chỉ có quyền cần thiết | Khóa bếp chỉ báo hoàn thành; công cụ MCP đọc gắn `readOnlyHint` |
| **Tách xử lý chậm khỏi giao diện** | Không để mạng/AI làm đứng UI | FreeRTOS 2 nhân + hàng đợi; Qt signal; `asyncio.to_thread` |
| **Thử lại có giãn cách** | Không dồn dập khi lỗi | Wi-Fi, WebSocket, MQTT, HTTP hoàn thành đơn |
| **Tất định trước, AI sau** | Quy tắc cho câu phổ biến, LLM chỉ dự phòng | Command matcher → Router → Ollama |
| **Từ chối thay vì đoán bừa** | Sai còn tệ hơn "không biết" | Phủ định → UNKNOWN; so khớp tên món có ngưỡng + cách biệt |
| **Cấu hình qua môi trường** | Không chép cứng khóa, dễ đổi máy | `.env`, `secrets.h` (không đưa lên git) |
| **Cải tiến từ log thật** | Dữ liệu chạy thử dẫn dắt sửa lỗi | Biến thể "công thức", `STT_CONFUSIONS`, warm-up, `keep_alive` |
| **Kiểm tra lúc biên dịch / lúc nạp** | Phát hiện lỗi sớm nhất | `static_assert` chân GPIO; câu mẫu trùng chế độ báo lỗi khi nạp |
| **Bộ nhớ cố định trên vi điều khiển** | Không phân mảnh khi chạy dài ngày | Mảng ký tự cố định, PSRAM cho bản ghi |
| **Không ghi dữ liệu nhạy cảm ra log** | Riêng tư, an toàn | Gateway không log câu trả lời/đường dẫn âm thanh; audit che trường |

---

## Phụ lục. Bảng tham số và thuật ngữ

### Tham số quan trọng

| Nhóm | Tham số | Giá trị |
|---|---|---|
| Thanh toán | Thời gian chờ thanh toán / chu kỳ polling | 300 s / 15 s |
| MQTT | Tuổi tối đa thông điệp ký | 120 s |
| KDS | Đồng bộ định kỳ / bộ đệm MQTT | 30 s / 8.192 byte |
| Âm thanh | Tần số lấy mẫu / khung | 16 kHz / 256 mẫu (16 ms) |
| Âm thanh | Pre-roll / bộ đệm vòng / bản ghi tối đa | 400 ms / 1 s / 8 s |
| DSP | DC blocker / thông cao / AGC đích / giới hạn | 0,995 / 80 Hz / RMS 6000 / 28.000 |
| VAD | Ngưỡng / im lặng kết thúc / tối thiểu | max(3 × nhiễu, 700) / 700 ms / 250 ms |
| Âm lượng | Mặc định / gain tại 60 % | 60 % / Q8 = 256 (×1,0) |
| Gateway | Kho WAV | 64 tệp, 300 s |
| Gateway | Phiếu xác nhận | 60 s, dùng một lần |
| STT | Mô hình / beam / kiểu số | small / 5 / int8 |
| LLM | Mô hình / nhiệt độ / timeout / cache công cụ | qwen2.5:3b / 0,2 / 120 s / 60 s |
| TTS | Giọng / tần số ra | vi_VN-vais1000-medium / 16 kHz |
| MCP | Đơn trễ / timeout API | > 15 phút / 5 s |

### Thuật ngữ

| Thuật ngữ | Giải thích ngắn |
|---|---|
| ACID | Bốn tính chất của giao dịch CSDL |
| AGC | Tự động điều chỉnh độ lợi |
| ASGI / WSGI | Chuẩn máy chủ web bất đồng bộ / đồng bộ của Python |
| Beam search | Tìm kiếm giữ nhiều ứng viên song song khi sinh chuỗi |
| DMA | Truy cập bộ nhớ trực tiếp, không qua CPU |
| EMA | Trung bình trượt mũ |
| FIR / IIR | Bộ lọc đáp ứng xung hữu hạn / vô hạn |
| FSM | Máy trạng thái hữu hạn |
| GGUF | Định dạng mô hình lượng tử hóa của llama.cpp/Ollama |
| HMAC | Mã xác thực thông điệp dựa trên hàm băm có khóa |
| I2S | Giao tiếp âm thanh số nối tiếp |
| Idempotency | Tính lũy đẳng: lặp lại không đổi kết quả |
| JSON-RPC | Gọi hàm từ xa bằng thông điệp JSON |
| MCP | Giao thức ngữ cảnh mô hình (kết nối AI với công cụ) |
| NFC / NFD | Dạng chuẩn hóa Unicode tổ hợp sẵn / tách rời |
| NVS | Vùng Flash lưu khóa–giá trị không mất khi tắt nguồn |
| PCM | Điều chế xung mã, dạng âm thanh số chưa nén |
| Q8 | Số học điểm cố định với 8 bit phần lẻ |
| QoS | Mức chất lượng dịch vụ của MQTT |
| RMS | Căn trung bình bình phương, đo năng lượng tín hiệu |
| TLV | Cấu trúc Mã – Độ dài – Giá trị (VietQR) |
| VAD | Phát hiện có tiếng nói |
| VITS | Kiến trúc TTS đầu–cuối dùng trong Piper |
| WAL | Ghi nhật ký trước (chế độ SQLite) |
| WER | Tỷ lệ lỗi từ trong nhận dạng tiếng nói |

# Giải thích chi tiết `kiosk_app/bridge.py`

> File: `Smart_Kiosk_Backup/kiosk_app/bridge.py` — 1442 dòng, bản trên GitHub `main`.
> Kiểm thử liên quan: `tests/test_bridge_payment_security.py` + `tests/test_order_lifecycle.py` — **24/24 đạt**.
> Không chạy thử ứng dụng thật ở đây (sẽ kết nối broker công cộng và SePay); các phân tích dựa trên mã + test.

---

## 0. Bức tranh tổng thể (đọc trước)

### 0.1 `bridge.py` là gì?
Ứng dụng kiosk gồm 2 nửa:
- **Giao diện QML** (`kiosk_app/qml/*.qml`) — vẽ thực đơn, giỏ hàng, mã QR. QML **không** biết CSDL, MQTT, SePay.
- **Logic Python** — chính là lớp `KioskBridge` trong file này.

`bridge.py` là **"cây cầu"** nối hai nửa: QML gọi hàm Python khi khách bấm nút; Python báo lại QML khi có thay đổi (giỏ đổi, có QR, đã trả tiền).

```
┌──────────── Giao diện QML ────────────┐        ┌──────────────── KioskBridge (Python) ─────────────────┐
│ ProductGrid / OptionPopup ─addToCart──┼───────►│ giỏ hàng (_cart_items)                                 │
│ CartSidebar ── removeFromCart/checkout┼───────►│ checkout(): tạo đơn PENDING (CSDL) + link QR           │
│ PaymentPopup ─ cancel/expirePayment ──┼───────►│ đóng đơn CANCELLED/EXPIRED                             │
│ main.qml ◄─── cartUpdated ────────────┼────────│                                                        │
│          ◄─── qrGenerated(link) ──────┼────────│ ◄── MQTT payment/success (webhook, có chữ ký)  ──┐     │
│          ◄─── paymentSuccess ─────────┼────────│ ◄── luồng tra cứu SePay mỗi 15 s (dự phòng)  ────┤     │
└───────────────────────────────────────┘        │      └──► _finalize_payment(): chốt PAID, trừ kho │     │
                                                 │            → MQTT tồn kho + đơn mới cho bếp ◄─────┘     │
                                                 └───────────────────────────────────────────────────────┘
```

### 0.2 Vòng đời một lần mua hàng
1. Khách thêm món → `addToCart` → giỏ cập nhật.
2. Bấm **Thanh toán** → `checkout()`: tạo mã `DH…`, **chụp giỏ hàng**, ghi đơn **PENDING** vào CSDL, phát `qrGenerated(link)` → QML hiện QR; khởi động **luồng tra cứu SePay**.
3. Khách chuyển khoản → tiền về qua **2 đường song song**:
   - **Webhook → MQTT** (nhanh, vài giây) → `_on_mqtt_message` → `_finalize_payment`.
   - **Tra cứu SePay** mỗi 15 s (dự phòng khi MQTT lỗi) → `_poll_payment_status` → `_finalize_payment`.
4. `_finalize_payment`: kiểm tra đúng đơn, đủ tiền → `database.save_new_order` (PAID + trừ kho) → gửi tồn kho + đơn mới cho bếp qua MQTT → `paymentSuccess` → QML đóng QR, xóa giỏ.
5. Nếu khách bấm **Hủy** / hết **300 s** → đơn thành CANCELLED / EXPIRED. Tiền về sau đó → **LATE_PAID** (ghi để hoàn tiền).

### 0.3 Thách thức chính: **3 luồng chạy cùng lúc**
| Luồng | Chạy gì |
|---|---|
| **Luồng chính (giao diện)** | Mọi hàm QML gọi: `addToCart`, `checkout`, `cancelPayment`... |
| **Luồng mạng MQTT** (do paho tạo bằng `loop_start`) | `_on_mqtt_message` → `_finalize_payment` |
| **Luồng tra cứu SePay** (mỗi đơn một luồng) | `_poll_payment_status` → `_finalize_payment` |

Cả 3 cùng đọc/ghi **trạng thái thanh toán** (`_current_order_code`, `_payment_confirmed`...) → cần **khóa** `_payment_lock` (phần 3). Đây là lớp chống trùng **thứ nhất**; CSDL (`BEGIN IMMEDIATE` + UPDATE có điều kiện) là lớp thứ hai và thứ ba.

### 0.4 Bản đồ file
| Dòng | Phần |
|---|---|
| 1–37 | Import, đường dẫn dự án |
| 40–92 | Đọc cấu hình `.env`, hằng số |
| 99–158 | Lớp `KioskBridge`: signal, `__init__` |
| 164–191 | `_validate_configuration` |
| 197–379 | MQTT: kết nối, nhận tín hiệu thanh toán |
| 385–584 | `_poll_payment_status` — tra cứu SePay dự phòng |
| 590–774 | `_finalize_payment` — chốt đơn |
| 780–810 | `_record_late_payment` — tiền về muộn |
| 816–1016 | Gửi tồn kho, gửi đơn cho bếp qua MQTT |
| 1022–1081 | Hủy / hết hạn thanh toán |
| 1087–1129 | `getMenuData` — thực đơn cho QML |
| 1135–1297 | Giỏ hàng: thêm, bớt, xóa |
| 1303–1412 | `checkout` — tạo đơn chờ + QR |
| 1418–1442 | Tổng tiền, dữ liệu giỏ cho QML |

---

## 1. Import và đường dẫn (L1–37)

```python
import sys, os, json, time, random, threading
from datetime import datetime

import requests
import paho.mqtt.client as mqtt
from dotenv import load_dotenv

from PyQt6.QtCore import (
    QObject,
    pyqtSlot as Slot,
    pyqtSignal as Signal,
    QUrl
)
```
| Thư viện | Dùng để |
|---|---|
| `random` | 2 chữ số ngẫu nhiên trong mã đơn |
| `threading` | Luồng tra cứu SePay, khóa `Lock` |
| `requests` | Gọi API SePay qua HTTPS |
| `paho.mqtt.client` | Kết nối broker MQTT |
| `dotenv.load_dotenv` | Đọc file `.env` vào biến môi trường |
| `QObject` | Lớp gốc của Qt — muốn có signal/slot phải kế thừa |
| `pyqtSlot as Slot`, `pyqtSignal as Signal` | **Đổi tên khi import** (`as`) → mã giống cách viết của PySide6 (`Slot`, `Signal`), dễ chuyển thư viện |
| `QUrl` | Đổi đường dẫn file ảnh thành URL `file:///…` cho QML |

```python
current_dir = os.path.dirname(os.path.abspath(__file__))   # .../kiosk_app
project_root = os.path.dirname(current_dir)                 # .../Smart_Kiosk_Backup

if project_root not in sys.path:
    sys.path.append(project_root)

from backend import database
from backend.mqtt_security import (MqttMessageRejected, configure_client, sign_message, verify_message)
```
- Thêm thư mục gốc dự án vào `sys.path` (nếu chưa có) → import được gói `backend`.
- `from backend import database` → **dùng chung** module CSDL với API (`giai_thich_database_py.md`).
- Import 4 thứ từ `mqtt_security` (`giai_thich_mqtt_security_py.md`): lỗi từ chối, cấu hình đăng nhập broker, ký, kiểm tra chữ ký.

---

## 2. Cấu hình và hằng số (L40–92)

```python
load_dotenv(os.path.join(project_root, ".env"), override=True)

SEPAY_API_TOKEN = os.environ.get("SEPAY_API_TOKEN", "")
MY_BANK_NAME = os.environ.get("MY_BANK_NAME", "")
MY_BANK_ACC = os.environ.get("MY_BANK_ACC", "")
MQTT_BROKER = os.environ.get("MQTT_BROKER", "broker.emqx.io")
MQTT_PORT = int(os.environ.get("MQTT_PORT", 1883))
MQTT_KITCHEN_TOPIC = os.environ.get("MQTT_KITCHEN_TOPIC", "kitchen/new_order")
MQTT_PAYMENT_TOPIC = os.environ.get("MQTT_PAYMENT_TOPIC", "payment/success")
MQTT_INVENTORY_TOPIC = os.environ.get("MQTT_INVENTORY_TOPIC", "kitchen/inventory")
MQTT_SHARED_SECRET = os.environ.get("MQTT_SHARED_SECRET", "")

POLLING_INTERVAL_SECONDS = 15
POLLING_TIMEOUT_SECONDS = 300
PENDING_ORDER_STALE_MINUTES = 10
```
- **`.env`** = file văn bản `TÊN=giá_trị` chứa **bí mật** (token SePay, khóa MQTT, số tài khoản) — **không đưa lên git**. `load_dotenv` nạp chúng vào `os.environ`.
- `override=True` → giá trị trong `.env` **ghi đè** biến môi trường đã có sẵn → `.env` là nguồn cấu hình chính.
- `os.environ.get(tên, mặc_định)` → thiếu thì dùng mặc định (broker công cộng `broker.emqx.io`, cổng `1883`).
- `int(...)` → biến môi trường **luôn là chuỗi**, cổng phải đổi sang số.
- Bí mật mặc định là `""` → các chỗ dùng sẽ kiểm tra rỗng và **từ chối** (fail-closed).
- 3 hằng số thời gian: tra cứu mỗi **15 s**, chờ tối đa **300 s** (= 5 phút, khớp `timeoutSeconds: 300` trong `PaymentPopup.qml`), đơn PENDING cũ hơn **10 phút** lúc khởi động bị coi là bỏ dở.

---

## 3. Lớp `KioskBridge` — signal và khởi tạo (L99–158)

### 3.1 Signal — Python báo cho QML
```python
class KioskBridge(QObject):
    cartUpdated = Signal()
    qrGenerated = Signal(str)
    paymentSuccess = Signal()
```
- `class KioskBridge(QObject):` → **kế thừa** `QObject` để dùng cơ chế signal/slot của Qt.
- **Signal** = "chuông báo". Khai báo ở **cấp lớp** (không phải trong `__init__`) — đây là quy tắc bắt buộc của Qt.
- `Signal(str)` → chuông kèm **một tham số chuỗi** (link QR).
- Phát chuông: `self.cartUpdated.emit()`. Bên QML **nghe** bằng `Connections` (`main.qml` L308–321):
  ```qml
  Connections {
      target: backend
      function onCartUpdated() { updateCartInfo() }
      function onQrGenerated(link) { paymentPopup.qrSource = link; paymentPopup.open() }
      function onPaymentSuccess() { paymentPopup.close(); window.reloadMenu() }
  }
  ```
  Quy ước tên: signal `cartUpdated` → hàm xử lý `onCartUpdated` (thêm `on` + viết hoa chữ đầu).
- `backend` là tên đối tượng do `main.py` đăng ký: `engine.rootContext().setContextProperty("backend", bridge)`.

### 3.2 `__init__`
```python
    def __init__(self):
        super().__init__()

        database.init_database()

        expired_count = database.expire_stale_pending_orders(
            max_age_minutes=PENDING_ORDER_STALE_MINUTES)
        if expired_count:
            print(f"---> [THANH TOÁN] Đã đóng {expired_count} đơn chờ quá hạn từ lần chạy trước.")

        self._cart_items = []
        self._total_price = 0

        self._payment_lock = threading.Lock()

        self._current_order_code = None
        self._payment_confirmed = False
        self._payment_in_progress = False

        self._pending_order_items = []
        self._pending_order_amount = 0
        self._underpaid_amount = 0

        self.is_polling = False
        self.mqtt_client = None

        self._validate_configuration()
        self.setup_mqtt()
```
| Dòng | Ý nghĩa |
|---|---|
| `super().__init__()` | Gọi hàm khởi tạo của lớp cha `QObject` — **bắt buộc**, không có thì signal không hoạt động |
| `init_database()` | Bảo đảm CSDL có đủ bảng (idempotent) |
| `expire_stale_pending_orders(10)` | **Dọn đơn treo**: kiosk mất điện khi khách đang quét QR → đơn kẹt PENDING; khởi động lại thì đóng thành EXPIRED → tiền về sau sẽ thành LATE_PAID chứ không bị bỏ qua |
| `threading.Lock()` | Khóa bảo vệ trạng thái thanh toán giữa 3 luồng |

**Các biến trạng thái:**
| Biến | Ý nghĩa |
|---|---|
| `_cart_items` | Giỏ hàng **đang hiển thị** (list dict) |
| `_total_price` | Tổng tiền giỏ |
| `_current_order_code` | Mã đơn **đang chờ thanh toán**; `None` = không có |
| `_payment_in_progress` | Đang hiện QR → **khóa giỏ** (không thêm/bớt món) |
| `_payment_confirmed` | Đơn hiện tại **đã chốt** → báo tiền lần 2 bị bỏ qua |
| `_pending_order_items` / `_pending_order_amount` | **Bản chụp** giỏ và số tiền **lúc tạo QR** |
| `_underpaid_amount` | Số tiền lớn nhất đã nhận nhưng chưa đủ (để ghi khi đơn đóng) |
| `is_polling` | Cờ cho luồng tra cứu biết còn chạy hay dừng |

Dấu `_` đầu tên → quy ước **dùng nội bộ**. (`is_polling`, `mqtt_client` không có `_` — chỉ là cách đặt tên, không khác về kỹ thuật.)

**Vì sao cần "bản chụp" giỏ hàng?** Giữa lúc hiện QR và lúc tiền về có thể vài phút. Nếu chốt đơn theo giỏ **đang hiển thị**, mọi thay đổi trong lúc đó (dù giỏ đã bị khóa, vẫn nên phòng) sẽ làm đơn **lệch** với số tiền trên QR. Chụp lại → đơn chốt **đúng y** những gì khách đã thấy và trả tiền.

---

## 4. `_validate_configuration()` (L164–191)

Chỉ **in cảnh báo** khi thiếu `MQTT_SHARED_SECRET`, `SEPAY_API_TOKEN`, `MY_BANK_NAME`, `MY_BANK_ACC` — **không dừng** chương trình. Kiosk vẫn mở được để xem thực đơn; các chức năng thiếu cấu hình tự từ chối ở chỗ dùng (ví dụ `checkout` không tạo QR khi thiếu ngân hàng). Cảnh báo rõ ngay lúc khởi động → dễ chẩn đoán qua nhật ký.

---

## 5. MQTT — kết nối và nhận tín hiệu thanh toán (L197–379)

### 5.1 `setup_mqtt()`
```python
    def setup_mqtt(self):
        try:
            self.mqtt_client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
        except AttributeError:
            self.mqtt_client = mqtt.Client()

        self.mqtt_client.on_connect = self._on_mqtt_connect
        self.mqtt_client.on_disconnect = self._on_mqtt_disconnect
        self.mqtt_client.on_message = self._on_mqtt_message
        configure_client(self.mqtt_client, role="KIOSK")

        try:
            self.mqtt_client.connect_async(MQTT_BROKER, MQTT_PORT, 60)
            self.mqtt_client.loop_start()
        except Exception as error:
            print(f"---> [LỖI MQTT] Không thể khởi tạo kết nối broker: {error}")
```
| Điểm | Giải thích |
|---|---|
| `try … except AttributeError` | paho-mqtt **2.x** cần chỉ định phiên bản callback; bản **1.x** không có `CallbackAPIVersion` → ném `AttributeError` → dùng cách cũ. Chạy được với cả hai |
| `on_connect = self._on_mqtt_connect` | Gán **hàm gọi lại** (*callback*): paho tự gọi khi sự kiện xảy ra. Gán **phương thức** `self._…` → bên trong vẫn dùng được `self` |
| `configure_client(..., role="KIOSK")` | Tài khoản broker / TLS nếu có cấu hình |
| `connect_async` | Kết nối **không chặn**: mất mạng lúc khởi động thì kiosk vẫn mở giao diện, paho tự thử lại |
| `60` | *keepalive*: 60 s gửi tín hiệu "còn sống" cho broker |
| `loop_start()` | paho tạo **một luồng nền riêng** lo gửi/nhận và tự kết nối lại → các callback chạy **trên luồng này**, không phải luồng giao diện |

### 5.2 `_on_mqtt_connect()` — đăng ký nhận tin khi kết nối xong
```python
    def _on_mqtt_connect(self, client, userdata, flags, reason_code, properties=None):
        try:
            connect_code = int(reason_code)
        except (TypeError, ValueError):
            connect_code = reason_code

        if connect_code == 0:
            subscribe_result = client.subscribe(MQTT_PAYMENT_TOPIC, qos=1)
            ...
```
- Danh sách tham số do **paho quy định** (phiên bản 2 có thêm `properties`; mặc định `None` để chạy cả bản cũ).
- `reason_code` ở paho 2 là đối tượng `ReasonCode`, đổi được sang số; `0` = thành công.
- **Đăng ký trong `on_connect`** (không phải ngay sau `connect`): mỗi lần **mất mạng rồi kết nối lại**, `on_connect` chạy lại → **tự đăng ký lại**. Nếu chỉ đăng ký một lần, sau khi rớt mạng kiosk sẽ **không còn nhận** tin thanh toán.
- `qos=1` → *at least once*: broker bảo đảm giao **ít nhất một lần** (có thể trùng → đã có lớp chống trùng).
- Kiosk **chỉ đăng ký** `payment/success`; các chủ đề bếp/tồn kho kiosk chỉ **gửi**, không nhận.

### 5.3 `_on_mqtt_message()` — xử lý tin thanh toán
```python
    def _on_mqtt_message(self, client, userdata, msg):
        if msg.topic != MQTT_PAYMENT_TOPIC:
            return
        try:
            try:
                payload = verify_message(msg.payload, MQTT_SHARED_SECRET, max_age_seconds=120)
            except MqttMessageRejected as rejected:
                print(f"---> [BẢO MẬT] Bỏ qua tín hiệu thanh toán MQTT: {rejected}")
                return

            order_code = str(payload.get("order_code", "")).strip()
            status = str(payload.get("status", "")).strip().upper()
            amount = payload.get("amount", None)

            if status != "PAID":      ... return
            if not order_code:        ... return
            if amount is None:        ... return

            self._finalize_payment(order_code, amount, source="webhook")

        except UnicodeDecodeError as error: ...
        except json.JSONDecodeError as error: ...
        except Exception as error: ...
```
**Các lớp lọc:**
1. Sai chủ đề → bỏ.
2. **Chữ ký HMAC sai / tin cũ > 120 s / thiếu khóa** → bỏ, in `[BẢO MẬT]`.
3. Trạng thái không phải `PAID`, thiếu mã đơn, thiếu số tiền → bỏ.
4. Qua hết → `_finalize_payment(..., source="webhook")`.

- `.strip().upper()` → chấp nhận `"paid"`, `" PAID "`.
- **`try` lồng `try`**: `try` trong chỉ bắt `MqttMessageRejected` (từ chối có chủ đích → cảnh báo bảo mật); `try` ngoài bắt mọi lỗi còn lại.
- **Vì sao bắt `Exception` ở cuối?** Hàm chạy trong **luồng của paho**; lỗi không bắt sẽ in traceback và có thể làm hỏng vòng nhận tin → kiosk ngừng nghe thanh toán. Bắt hết → một tin lỗi không làm hỏng các tin sau.
- (Hai nhánh `UnicodeDecodeError`/`JSONDecodeError` thực tế ít khi chạy tới vì `verify_message` đã đổi các lỗi này thành `MqttMessageRejected`; giữ lại không hại.)

---

## 6. `_poll_payment_status()` — tra cứu SePay dự phòng (L385–584)

**Vì sao cần?** MQTT có thể lỗi (broker công cộng chập chờn, dịch vụ webhook hoặc đường hầm Internet tới Pi bị dừng). Kiosk **tự hỏi** SePay mỗi 15 s: "có giao dịch nào chứa mã đơn này không?". Hai đường độc lập → một đường hỏng vẫn chốt được đơn.

Hàm chạy trên **một luồng riêng** (tạo trong `checkout`, phần 11).

### 6.1 Bắt đầu
```python
        start_time = time.time()
        with self._payment_lock:
            if order_code != self._current_order_code:
                return
            self.is_polling = True
```
- `with self._payment_lock:` → **giữ khóa** trong khối, ra khỏi khối tự nhả (kể cả khi lỗi/`return`). Đọc/ghi trạng thái dùng chung **luôn** nằm trong khối này.
- Đơn đã bị đóng trước khi luồng kịp chạy → thoát luôn.

### 6.2 Vòng lặp chính
```python
        while True:
            with self._payment_lock:
                if not self.is_polling:            break   # bị hủy
                if self._payment_confirmed:        self.is_polling = False; break   # đã chốt (qua webhook)
                if order_code != self._current_order_code: self.is_polling = False; break  # đơn khác

            elapsed_time = time.time() - start_time
            if elapsed_time >= POLLING_TIMEOUT_SECONDS:
                break
            try:
                ... gọi SePay ...
            except requests.RequestException as error: ...
            except ValueError as error: ...
            except Exception as error: ...
            ... chờ 15 s ...
```
- **Kiểm tra trạng thái đầu mỗi vòng** trong khóa → dừng ngay khi webhook đã chốt, khách đã hủy.
- **Chỉ giữ khóa khi đọc/ghi biến**, **không** giữ trong lúc gọi mạng (có thể mất 10 s) → các luồng khác không bị chặn lâu.

### 6.3 Gọi API SePay
```python
                url = "https://my.sepay.vn/userapi/transactions/list"
                headers = {"Authorization": f"Bearer {SEPAY_API_TOKEN}", "Content-Type": "application/json"}
                params = {"limit": 10, "keyword": order_code}
                response = requests.get(url, headers=headers, params=params, timeout=10)
```
- `"https://my.sepay.vn/" "userapi/..."` → hai chuỗi **đứng cạnh nhau** được Python **tự nối**.
- `Authorization: Bearer <token>` → cách xác thực phổ biến của API.
- `params=` → `requests` tự ghép thành `?limit=10&keyword=DH…` (và tự mã hóa ký tự đặc biệt).
- `timeout=10` → không có thì mạng treo sẽ làm luồng **treo mãi**.

### 6.4 Xử lý kết quả
```python
                if response.status_code == 200:
                    data = response.json()
                    transactions = data.get("transactions", [])
                    if not isinstance(transactions, list):
                        transactions = []
                    transaction_found = False
                    for transaction in transactions:
                        if not isinstance(transaction, dict):
                            continue
                        content = str(transaction.get("transaction_content", ""))
                        if order_code not in content:
                            continue
                        transaction_amount = (
                            transaction.get("amount_in")
                            if transaction.get("amount_in") is not None
                            else transaction.get("amount")
                        )
                        if transaction_amount is None:
                            continue
                        self._finalize_payment(order_code, transaction_amount, source="polling")
                        transaction_found = True
                        break
```
- **Không tin dữ liệu ngoài**: kiểm tra kiểu (`list`, `dict`) trước khi dùng → phản hồi lạ không làm sập luồng.
- **Kiểm tra lại `order_code in content`** dù đã lọc bằng `keyword` ở server → không phụ thuộc cách SePay tìm kiếm.
- Số tiền lấy `amount_in` (hoặc `amount`). **Không bao giờ** lấy số tiền **của đơn** làm mặc định — nếu thế, giao dịch thiếu số tiền sẽ "tự động đủ" (chú thích L480–482).

```python
                    if transaction_found:
                        with self._payment_lock:
                            confirmed = self._payment_confirmed
                            if confirmed:
                                self.is_polling = False
                        if confirmed:
                            break
```
- Tìm thấy giao dịch **chưa chắc đã chốt** (ví dụ chưa đủ tiền) → chỉ dừng khi `_payment_confirmed` thật sự `True`; nếu chưa thì tiếp tục chờ tới hết giờ.
- Đọc biến **trong khóa**, quyết định `break` **ngoài khóa**.

### 6.5 Chờ 15 giây mà vẫn dừng được ngay
```python
            wait_start = time.time()
            while time.time() - wait_start < POLLING_INTERVAL_SECONDS:
                with self._payment_lock:
                    if not self.is_polling:
                        return
                time.sleep(0.2)
```
- Nếu viết `time.sleep(15)`, khách bấm Hủy thì luồng vẫn ngủ hết 15 s. Chia thành nhiều lần ngủ **0,2 s**, mỗi lần kiểm tra cờ → dừng trong vòng **0,2 s** sau khi hủy.

### 6.6 Hết giờ
```python
        with self._payment_lock:
            timed_out = (not self._payment_confirmed and order_code == self._current_order_code)
            self.is_polling = False

        if timed_out:
            self._close_current_payment(database.ORDER_EXPIRED, expected_order_code=order_code)
```
- Hết 300 s mà chưa chốt và vẫn là đơn này → đóng **EXPIRED**. Giao diện cũng có bộ đếm 300 s gọi `expirePayment()`; ai chạy trước thì đóng, bên sau bị chặn nhờ `expected_order_code` (phần 10) → **kiosk luôn được giải phóng** để nhận khách mới, kể cả khi giao diện lỗi.

---

## 7. `_finalize_payment()` — chốt đơn (L590–774)

Được gọi từ **cả hai đường** (webhook, tra cứu), có thể **cùng lúc**.

### 7.1 Bước 1 — kiểm tra đơn và số tiền (trong khóa)
```python
        with self._payment_lock:
            order_mismatch = (order_code != self._current_order_code)
        if order_mismatch:
            self._record_late_payment(order_code, amount, source)
            return
```
- Tiền về cho mã **không phải đơn đang chờ** (đơn đã hủy/hết hạn, hoặc đơn đã chốt rồi mà webhook báo lại) → chuyển cho `_record_late_payment` → CSDL tự phân loại (`late_paid` / `already_recorded` / `not_closed` / `unknown_order`).

```python
        with self._payment_lock:
            if self._payment_confirmed:
                return                                     # đường kia đã chốt rồi
            if order_code != self._current_order_code:
                self._record_late_payment(order_code, amount, source)
                return                                     # khách hủy đúng giữa 2 lần kiểm tra
            try:
                paid_amount = int(float(amount))
                expected_amount = int(float(self._pending_order_amount))
            except (TypeError, ValueError):
                return
            if expected_amount <= 0:
                return
            if paid_amount < expected_amount:
                self._underpaid_amount = max(self._underpaid_amount, paid_amount)
                return
            order_items = [dict(item) for item in self._pending_order_items]
            if not order_items:
                return
            self._payment_confirmed = True
            self.is_polling = False
```
**Kỹ thuật "kiểm tra lại trong khóa"** (*double-checked*): lần kiểm tra đầu nằm ngoài khối chính; trạng thái có thể đổi trước khi vào khối thứ hai → **kiểm tra lại** bên trong.

**Điểm then chốt — `self._payment_confirmed = True` ngay trong khóa:** webhook và tra cứu cùng gọi hàm này → chỉ **luồng đầu tiên** vào khóa thấy `False`, đặt `True`; luồng sau vào khóa thấy `True` → `return`. Đây là **lớp chống trùng thứ nhất** (trong cùng tiến trình). Hai lớp tiếp theo nằm ở CSDL.

- `dict(item) for item in ...` → **sao chép** từng món (để bước sau không bị ảnh hưởng nếu danh sách gốc bị xóa).
- Số tiền **ít hơn** giá trị đơn → không chốt; lưu `max(...)` số tiền lớn nhất đã nhận → khi đơn đóng sẽ được ghi để hoàn tiền (phần 10).
- **Thao tác CSDL nằm ngoài khóa** (bước 2) → không giữ khóa trong lúc ghi CSDL; trạng thái đã được "đánh dấu" bằng `_payment_confirmed` nên an toàn.

### 7.2 Bước 2 — lưu CSDL
```python
        try:
            items_json = json.dumps(order_items, ensure_ascii=False)
            save_result = database.save_new_order(order_code, expected_amount, items_json,
                                                  paid_amount=paid_amount)
        except Exception as error:
            with self._payment_lock:
                self._payment_confirmed = False
            return

        if save_result.get("reason") == "order_closed":
            with self._payment_lock:
                self._payment_confirmed = False
            self._record_late_payment(order_code, paid_amount, source)
            return

        if not save_result.get("inserted"):
            ...  # in LỖI NGHIÊM TRỌNG, không gửi bếp, không xóa giỏ
            with self._payment_lock:
                self._payment_confirmed = False
            return
```
- `save_new_order` = **PAID + trừ kho trong một giao dịch `BEGIN IMMEDIATE`** (xem `giai_thich_database_py.md` §8.7).
- **Lỗi thì trả cờ `_payment_confirmed` về `False`** → lần báo tiền tiếp theo (tra cứu 15 s sau) còn cơ hội chốt lại.
- `order_closed` → khách bấm hủy **đúng lúc** tiền về (CSDL đã CANCELLED) → ghi LATE_PAID.
- Thất bại khác → **không gửi bếp, không xóa giỏ**: thà báo lỗi rõ ràng còn hơn bếp làm món cho một đơn không có trong CSDL.

### 7.3 Bước 3–5 — báo bếp và cập nhật giao diện
```python
        self._publish_inventory_snapshot()
        self._publish_kitchen_order(order_code=order_code, order_items=order_items, source=source)

        self.paymentSuccess.emit()
        self.clearCart()

        with self._payment_lock:
            self._current_order_code = None
            self._pending_order_items = []
            self._pending_order_amount = 0
            self._underpaid_amount = 0
            self._payment_in_progress = False
            self.is_polling = False
```
- **Thứ tự có chủ đích:** CSDL **trước** → rồi mới MQTT. MQTT lỗi thì đơn vẫn đúng trong CSDL, màn hình bếp tự lấy lại qua API `/api/orders/pending`. Ngược lại (gửi bếp trước, lưu CSDL sau) thì có thể bếp làm món cho đơn không được lưu.
- `paymentSuccess.emit()` → QML đóng popup QR, tải lại thực đơn.
- Cuối cùng **đặt lại toàn bộ trạng thái** → sẵn sàng cho khách tiếp theo.

**Về signal phát từ luồng khác:** `_finalize_payment` chạy trên luồng MQTT hoặc luồng tra cứu, nhưng QML ở luồng giao diện. Qt xử lý điều này tự động: khi `emit` từ luồng khác, Qt **xếp sự kiện vào hàng đợi** của luồng giao diện (*queued connection*) → hàm `onPaymentSuccess` vẫn chạy **trên luồng giao diện**, an toàn.

---

## 8. `_record_late_payment()` — tiền về muộn (L780–810)

```python
        outcome = database.mark_order_late_paid(order_code, amount)
        if outcome == "late_paid":           print("... Đã ghi LATE_PAID, cần hoàn tiền.")
        elif outcome == "unknown_order":     print("... Nhận tiền cho mã đơn lạ ..., bỏ qua.")
        elif outcome not in ("already_recorded", "not_closed"):
            print("... Không ghi nhận được tiền về ...")
        return outcome
```
| Kết quả | Tình huống | In ra? |
|---|---|---|
| `late_paid` | Đơn đã hủy/hết hạn → vừa ghi LATE_PAID | Có — cần hoàn tiền |
| `already_recorded` | Webhook và tra cứu cùng báo một khoản muộn | Không (bình thường) |
| `not_closed` | Đơn đã PAID (báo tiền lần 2 sau khi chốt) | Không (bình thường) |
| `unknown_order` | Mã đơn không có trong CSDL | Có — cảnh báo |
| `database_error` | Lỗi CSDL | Có |

`elif outcome not in (...)` → chỉ in khi là **trường hợp bất thường**, tránh làm đầy nhật ký bằng các tình huống trùng lặp vốn được thiết kế để xảy ra.

---

## 9. Gửi tồn kho và đơn cho bếp (L816–1016)

### 9.1 `_publish_inventory_snapshot()`
```python
            all_ingredients = database.get_all_ingredients()
            inventory_payload = {
                "version": 1,
                "event": "inventory.snapshot",
                "created_at": datetime.now().astimezone().isoformat(timespec="seconds"),
                "items": [
                    {"id": ingredient["id"], "name": ingredient["name"], "unit": ingredient["unit"],
                     "quantity": ingredient["quantity"],
                     "low": (ingredient["quantity"] <= ingredient["low_stock_threshold"])}
                    for ingredient in all_ingredients
                ]
            }
            if not MQTT_SHARED_SECRET:
                return False
            payload_text = sign_message(inventory_payload, MQTT_SHARED_SECRET)
            publish_info = self.mqtt_client.publish(MQTT_INVENTORY_TOPIC, payload_text, qos=1, retain=True)
            if publish_info.rc != mqtt.MQTT_ERR_SUCCESS:
                return False
            return True
```
| Điểm | Giải thích |
|---|---|
| `"version": 1, "event": ...` | Đánh **phiên bản + loại sự kiện** → bên nhận (ESP32) biết cách đọc; sau này đổi định dạng không làm hỏng thiết bị cũ |
| `datetime.now().astimezone().isoformat(timespec="seconds")` | Thời gian kèm **múi giờ**, chuẩn ISO 8601: `2026-11-25T21:38:36+07:00` |
| List comprehension tạo `items` | Mỗi nguyên liệu một dict; `"low"` tính sẵn `True/False` → ESP32 không phải tự so sánh |
| `retain=True` | Broker **giữ lại tin cuối** và giao ngay cho thiết bị mới kết nối → màn hình bếp vừa bật lên đã có tồn kho mới nhất |
| `publish_info.rc` | `publish` chỉ **xếp vào hàng đợi** gửi; `rc` cho biết xếp thành công không (in "Đã đưa ... vào hàng đợi gửi" — không khẳng định đã tới nơi) |
| Thiếu khóa → không gửi | Không bao giờ gửi tin **chưa ký**; màn hình bếp vẫn đồng bộ qua API |
| `return True/False` | Báo kết quả; lỗi được bắt hết → không ảnh hưởng việc chốt đơn đã xong |

### 9.2 `_publish_kitchen_order()`
```python
            kitchen_payload = {
                "version": 1,
                "event": "order.created",
                "order_id": order_code,
                "created_at": ...,
                "items": [
                    {
                        "product_id": item.get("product_id", 0),
                        "name": item.get("base_name", item.get("name", "").split(" (", 1)[0].strip()),
                        "quantity": item.get("qty", 1),
                        "options": {"size": item.get("size", ""), "ice": item.get("ice", "")},
                        "note": item.get("note", "")
                    }
                    for item in order_items
                ]
            }
            ...
            publish_info = self.mqtt_client.publish(MQTT_KITCHEN_TOPIC, payload_text, qos=1, retain=False)
```
- **Đổi dữ liệu giỏ thành định dạng bếp cần**: tên gốc (`base_name`) + tùy chọn tách riêng (`options.size`, `options.ice`) → màn hình bếp tự trình bày.
- `.split(" (", 1)[0]` → dự phòng khi thiếu `base_name`: `"Bạc xỉu (Size L, Ít đá)"` → tách tại `" ("` **tối đa 1 lần** → lấy phần đầu `"Bạc xỉu"`.
- `retain=False` → đơn mới là **sự kiện tức thời**; nếu giữ lại, mỗi lần bếp kết nối lại sẽ nhận **lại đơn cũ**. Đơn bỏ lỡ được bếp lấy bù qua API pending.

---

## 10. Hủy / hết hạn thanh toán (L1022–1081)

```python
    def _close_current_payment(self, new_status, expected_order_code=None):
        with self._payment_lock:
            order_code = self._current_order_code
            if expected_order_code is not None and order_code != expected_order_code:
                return None
            underpaid_amount = self._underpaid_amount
            self.is_polling = False
            self._current_order_code = None
            self._payment_confirmed = False
            self._payment_in_progress = False
            self._pending_order_items = []
            self._pending_order_amount = 0
            self._underpaid_amount = 0

        if order_code:
            database.close_pending_order(order_code, new_status)
            if underpaid_amount > 0:
                self._record_late_payment(order_code, underpaid_amount, "underpaid")
        return order_code

    @Slot()
    def cancelPayment(self):
        order_code = self._close_current_payment(database.ORDER_CANCELLED)
        ...

    @Slot()
    def expirePayment(self):
        order_code = self._close_current_payment(database.ORDER_EXPIRED)
        ...
```
- **Một hàm chung**, hai lối vào: khách bấm Hủy (`cancelPayment`) và hết 300 s (`expirePayment` từ QML, hoặc luồng tra cứu).
- **Trong khóa:** lấy mã đơn rồi **xóa sạch trạng thái** → `is_polling = False` làm luồng tra cứu tự dừng; giỏ mở khóa.
- `expected_order_code` → tham số **tùy chọn**: luồng tra cứu truyền mã đơn của nó; nếu đơn hiện tại đã khác (đã chốt, đã hủy, hoặc đã sang đơn mới) → **không đụng** đơn của người khác.
- **Ngoài khóa:** ghi CSDL. `close_pending_order` là **UPDATE có điều kiện** `WHERE payment_status='PENDING'` → nếu tiền **vừa về** và đơn đã PAID, lệnh hủy **không ghi đè**.
- Có khoản tiền chưa đủ → ghi bằng `mark_order_late_paid` để quản trị viên hoàn lại.

**`@Slot()`** → đánh dấu hàm là **slot**: QML gọi được (`backend.cancelPayment()` trong `PaymentPopup.qml`). Hàm không có `@Slot` (như `_close_current_payment`, `clearCart`) thì QML **không gọi được**.

---

## 11. `getMenuData()` — thực đơn cho QML (L1087–1129)

```python
    @Slot(result=list)
    def getMenuData(self):
        raw_data = database.get_all_products()
        assets_path = os.path.join(project_root, "assets", "images")
        final_data = []
        for item in raw_data:
            image_name = item.get("image", "")
            full_path = os.path.join(assets_path, image_name) if image_name else ""   # (mã gốc viết if/else nhiều dòng)
            if full_path and os.path.exists(full_path):
                item["image_display"] = QUrl.fromLocalFile(full_path).toString()
            else:
                item["image_display"] = ""
            final_data.append(item)
        return final_data
```
- `@Slot(result=list)` → slot **trả giá trị** về QML; list các dict Python được Qt **tự đổi** thành mảng JavaScript các đối tượng.
- `get_all_products()` chỉ trả món **đang bán**.
- CSDL chỉ lưu **tên file** (`bac_xiu.jpg`) → ghép thành đường dẫn đầy đủ, **kiểm tra file tồn tại**, đổi thành URL `file:///home/.../bac_xiu.jpg` — dạng QML `Image` hiểu được. Không có ảnh → `""` → QML hiện biểu tượng thay thế.
- Thêm trường mới `image_display`, giữ `image` gốc.
- QML gọi lại hàm này **sau mỗi đơn** (`window.reloadMenu()`) → đổi giá/thêm món trên phần mềm quản trị thì khách tiếp theo thấy ngay.

---

## 12. Giỏ hàng (L1135–1297)

### 12.1 `addToCart()` — thêm món
```python
    @Slot(str, str, float, str)
    @Slot(str, str, float, str, str, str)
    def addToCart(self, p_id_str, name, price, image, size="", ice=""):
```
**Hai `@Slot` chồng lên nhau** → đăng ký **hai chữ ký** (*overload*) cho QML: gọi với 4 hoặc 6 tham số đều được. Kiểu tham số phải khớp: `str, str, float, str`.

QML gọi ở 3 nơi:
| Nơi | Lời gọi | Mục đích |
|---|---|---|
| `ProductGrid.qml` L201 | `addToCart(model.id, name, price, image_display, "", "")` | Món ăn: thêm thẳng |
| `OptionPopup.qml` L105 | `addToCart(currentPid, currentName, basePrice, image, selectedSize, selectedIce)` | Đồ uống: sau khi chọn size/đá |
| `CartSidebar.qml` L342 | `addToCart(modelData.id, modelData.name, modelData.price, modelData.image)` | Nút **+** trong giỏ (id dạng `8_Size L_Ít đá`) |

**Khóa giỏ khi đang thanh toán:**
```python
        with self._payment_lock:
            if self._payment_in_progress:
                return
```

**Mã định danh ghép (composite id)** — cùng một món nhưng khác size/đá là **dòng khác nhau** trong giỏ:
```
"3"                      → Bạc xỉu, không tùy chọn
"3_Size L_Ít đá"         → Bạc xỉu size L ít đá
"3_Size M_"              → Bạc xỉu size M, đá mặc định
```
```python
        p_id_str = str(p_id_str)
        composite_id = p_id_str
        try:
            if "_" in p_id_str:                       # bấm + trong giỏ: id đã là dạng ghép
                parts = p_id_str.split("_")
                p_id = int(parts[0])
                if len(parts) > 1: size = parts[1]
                if len(parts) > 2: ice = parts[2]
                base_name = name.split(" (", 1)[0].strip()
            else:                                     # thêm mới từ thực đơn
                p_id = int(p_id_str)
                base_name = str(name).strip()
        except (TypeError, ValueError) as error:
            return
```
- `.split("_")` → `"3_Size L_Ít đá"` → `["3", "Size L", "Ít đá"]`.
- `name.split(" (", 1)[0]` → bỏ phần `" (Size L, Ít đá)"` khỏi tên hiển thị để lấy tên gốc.

```python
        display_name = base_name
        final_price = float(price)
        if size or ice:
            option_parts = []
            if size: option_parts.append(size)
            if ice:  option_parts.append(ice)
            display_name = f"{base_name} ({', '.join(option_parts)})"    # "Bạc xỉu (Size L, Ít đá)"
            composite_id = f"{p_id}_{size}_{ice}"
            if "_" not in p_id_str:                   # chỉ cộng phụ thu khi THÊM MỚI
                if size == "Size M":   final_price += 5000
                elif size == "Size L": final_price += 10000
```
- **Phụ thu size: M +5.000 đ, L +10.000 đ.**
- **Vì sao chỉ cộng khi `"_" not in p_id_str`?** Khi bấm **+** trong giỏ, giá gửi lên là giá **đã cộng phụ thu** của dòng đó → cộng lần nữa thì size L thành +20.000.

```python
        found = False
        for item in self._cart_items:
            if item["id"] == composite_id:
                item["qty"] += 1
                found = True
                break
        if not found:
            self._cart_items.append({
                "id": composite_id, "product_id": p_id, "base_name": base_name, "name": display_name,
                "price": final_price, "image": image, "qty": 1, "size": size, "ice": ice, "note": ""
            })
        self._calculate_total()
        self.cartUpdated.emit()
```
- Đã có dòng cùng id → **tăng số lượng**; chưa có → thêm dòng mới.
- Biến cờ `found` + `break` → mẫu "tìm trong list". (Python còn có cách viết `for … else`.)
- Dict này chính là **một phần tử của `items_json`** lưu trong CSDL sau này (có `product_id`, `base_name`, `price`, `qty`, `size`, `ice`, `note`).
- Cuối cùng: tính lại tổng, **phát `cartUpdated`** → QML gọi `getCartItems()`/`getTotalPrice()` để vẽ lại.

### 12.2 `removeFromCart()` — bớt món
```python
        for item in list(self._cart_items):
            if item["id"] != composite_id:
                continue
            if item["qty"] > 1:
                item["qty"] -= 1
            else:
                self._cart_items.remove(item)
            break
```
- `list(self._cart_items)` → duyệt trên **bản sao** của danh sách. **Xóa phần tử khỏi list đang duyệt** dễ làm bỏ sót phần tử kế tiếp; duyệt bản sao, xóa trên bản gốc thì an toàn. (Ở đây có `break` ngay sau khi xóa nên không lỗi, nhưng đây là thói quen tốt.)
- Số lượng > 1 → giảm 1; bằng 1 → xóa dòng.

### 12.3 `clearCart()`
Xóa giỏ, tổng = 0, phát `cartUpdated`. Không có `@Slot` → chỉ Python gọi (sau khi thanh toán xong).

---

## 13. `checkout()` — tạo đơn chờ và mã QR (L1303–1412)

### 13.1 Kiểm tra trước
```python
        if self._total_price <= 0:        return     # giỏ trống
        if not self._cart_items:          return
        if not MY_BANK_NAME or not MY_BANK_ACC: return   # chưa cấu hình ngân hàng
```

### 13.2 Tạo mã đơn và chụp giỏ (trong khóa)
```python
        with self._payment_lock:
            if self._payment_in_progress:
                return                                   # đang có QR khác
            order_code = (
                f"DH"
                f"{int(time.time()) % 1000000:06d}"
                f"{random.randint(10, 99)}"
            )
            order_items_snapshot = [dict(item) for item in self._cart_items]
            order_amount_snapshot = float(self._total_price)

            self._current_order_code = order_code
            self._payment_confirmed = False
            self._payment_in_progress = True
            self._underpaid_amount = 0
            self._pending_order_items = order_items_snapshot
            self._pending_order_amount = order_amount_snapshot
```
**Mã đơn** `DH` + 6 số + 2 số, ví dụ `DH79318842`:
| Phần | Cách tạo | Ý nghĩa |
|---|---|---|
| `DH` | cố định | Dễ tìm trong nội dung chuyển khoản (ngân hàng có thể thêm chữ khác) |
| 6 số | `int(time.time()) % 1000000` → 6 chữ số cuối của giây Unix; `:06d` đệm 0 | Thay đổi mỗi giây; lặp lại sau ~11,6 ngày |
| 2 số | `random.randint(10, 99)` | Hai lần bấm trong **cùng một giây** vẫn khác mã (90 khả năng) |

Lỡ trùng mã → `create_pending_order` gặp lỗi **khóa chính** → `duplicate_order` → không hiện QR (13.3). Mã **không cần bí mật** — an toàn thanh toán dựa vào chữ ký HMAC và token SePay, không dựa vào việc khó đoán mã.

`_payment_in_progress = True` ngay trong khóa → **khóa giỏ** và chặn bấm Thanh toán lần 2.

### 13.3 Ghi đơn PENDING vào CSDL
```python
        pending_result = database.create_pending_order(
            order_code, order_amount_snapshot,
            json.dumps(order_items_snapshot, ensure_ascii=False))

        if not pending_result.get("inserted"):
            with self._payment_lock:
                if self._current_order_code == order_code:
                    self._current_order_code = None
                    self._payment_in_progress = False
                    self._pending_order_items = []
                    self._pending_order_amount = 0
            return
```
- **Ghi CSDL trước khi hiện QR** → mọi khoản tiền về sau đều có đơn để đối chiếu (kể cả khi kiosk mất điện ngay sau đó).
- Ghi thất bại → **trả lại trạng thái** (mở khóa giỏ) và **không hiện QR** → khách không chuyển tiền cho một đơn không tồn tại.
- `if self._current_order_code == order_code` → chỉ dọn nếu trạng thái vẫn là của **đơn này**.

### 13.4 Tạo link QR
```python
        qr_url = (
            f"https://qr.sepay.vn/img"
            f"?acc={MY_BANK_ACC}"
            f"&bank={MY_BANK_NAME}"
            f"&amount={int(order_amount_snapshot)}"
            f"&des={order_code}"
            f"&template=compact"
        )
        self.qrGenerated.emit(qr_url)
```
- Dịch vụ `qr.sepay.vn` **tạo sẵn ảnh QR VietQR** từ tham số: số tài khoản, ngân hàng, **số tiền**, **nội dung = mã đơn**. Kiosk không tự vẽ QR — chỉ gửi link, QML `Image` tải ảnh về.
- Khách quét QR → ứng dụng ngân hàng **điền sẵn** số tiền và nội dung → không phải gõ tay.
- `qrGenerated.emit(qr_url)` → QML mở `PaymentPopup` với ảnh QR và bộ đếm 300 s.

### 13.5 Khởi động luồng tra cứu
```python
        polling_thread = threading.Thread(
            target=self._poll_payment_status,
            args=(order_code, order_amount_snapshot),
            daemon=True,
            name=f"PaymentPolling-{order_code}"
        )
        polling_thread.start()
```
| Tham số | Ý nghĩa |
|---|---|
| `target=` | Hàm luồng sẽ chạy (truyền **tên hàm**, không có `()`) |
| `args=(...)` | Tuple tham số truyền vào hàm |
| `daemon=True` | Luồng **phụ**: đóng kiosk thì luồng tự kết thúc theo, không giữ chương trình chạy ngầm |
| `name=` | Tên luồng — dễ nhận ra khi gỡ lỗi |
| `.start()` | Bắt đầu chạy **song song**; `checkout()` trả về ngay → giao diện không bị đứng |

**Vì sao không gọi SePay ngay trong `checkout()`?** Hàm này chạy trên **luồng giao diện**. Gọi mạng (tối đa 10 s) rồi ngủ 15 s ở đây sẽ làm giao diện **đơ** suốt 5 phút.

---

## 14. Tổng tiền và dữ liệu giỏ cho QML (L1418–1442)

```python
    def _calculate_total(self):
        self._total_price = sum(
            float(item.get("price", 0)) * int(item.get("qty", 0))
            for item in self._cart_items
        )

    @Slot(result=list)
    def getCartItems(self):
        return [dict(item) for item in self._cart_items]

    @Slot(result=float)
    def getTotalPrice(self):
        return float(self._total_price)
```
- `sum(biểu_thức for ...)` → **generator expression** đưa thẳng vào `sum`: tổng (giá × số lượng).
- `getCartItems` trả **bản sao** → QML có sửa cũng không làm hỏng giỏ thật trong Python.
- `@Slot(result=float)` → kiểu trả về khai báo cho Qt (QML nhận số).

---

## 15. Tổng hợp

### 15.1 Ba lớp chống chốt đơn trùng
| Lớp | Ở đâu | Chặn gì |
|---|---|---|
| 1. `_payment_lock` + `_payment_confirmed` | `bridge.py` §7.1 | Webhook và tra cứu **cùng tiến trình** báo cùng lúc |
| 2. `BEGIN IMMEDIATE` + đọc trạng thái | `database.save_new_order` | Mọi tiến trình |
| 3. UPDATE có điều kiện + `rowcount` | `database.save_new_order` | Dự phòng |

### 15.2 Các nguyên tắc thiết kế
1. **Hai đường xác nhận độc lập** (MQTT nhanh + tra cứu SePay dự phòng).
2. **Ghi CSDL trước, thông báo sau** (PENDING trước QR; PAID trước khi gửi bếp).
3. **Chụp lại giỏ hàng** lúc tạo QR; khóa giỏ khi đang thanh toán.
4. **Khóa ngắn**: chỉ giữ khóa khi đọc/ghi biến; gọi mạng và CSDL ở ngoài khóa.
5. **Không tin dữ liệu ngoài**: kiểm tra chữ ký MQTT, kiểu dữ liệu phản hồi SePay, số tiền thật.
6. **Lỗi không làm sập**: callback MQTT và luồng tra cứu bắt mọi lỗi; thiếu cấu hình chỉ cảnh báo.
7. **Luôn giải phóng kiosk**: hết 300 s thì đóng đơn từ cả giao diện lẫn luồng tra cứu.

### 15.3 Ghi chú nhỏ (không gây lỗi hiện tại)
| Điểm | Ghi chú |
|---|---|
| Giá phụ thu size (5.000 / 10.000 đ) viết cứng trong Python | Muốn đổi phải sửa mã; có thể đưa vào CSDL hoặc cấu hình |
| Hai nhánh `except UnicodeDecodeError` / `except json.JSONDecodeError` trong `_on_mqtt_message` | Gần như không chạy tới vì `verify_message` đã đổi các lỗi này thành `MqttMessageRejected` |
| `clearCart()` được gọi từ luồng MQTT/tra cứu | An toàn vì giỏ đang bị khóa (`_payment_in_progress`) nên luồng giao diện không sửa giỏ cùng lúc; signal được Qt chuyển về luồng giao diện |
| Tên biến `is_polling`, `mqtt_client` không có `_` | Chỉ là quy ước đặt tên, không ảnh hưởng |

---

## 16. Câu hỏi tự kiểm tra
1. Signal và Slot khác nhau thế nào? Cho mỗi loại một ví dụ trong file.
2. Vì sao phải đăng ký chủ đề MQTT trong `_on_mqtt_connect` chứ không ngay sau `connect_async`?
3. Webhook và tra cứu SePay báo tiền **cùng một lúc** — dòng nào bảo đảm chỉ một bên chốt đơn?
4. Vì sao `_finalize_payment` không giữ khóa trong lúc gọi `save_new_order`?
5. Vì sao chờ 15 s bằng nhiều lần `sleep(0.2)` thay vì `sleep(15)`?
6. Vì sao phải ghi đơn PENDING **trước** khi phát `qrGenerated`?
7. Vì sao gửi bếp **sau** khi lưu CSDL? Nếu MQTT lỗi thì bếp lấy đơn bằng cách nào?
8. Tin tồn kho dùng `retain=True`, tin đơn mới dùng `retain=False` — vì sao?
9. Bấm **+** trên dòng "Bạc xỉu (Size L)" trong giỏ: vì sao giá không bị cộng thêm 10.000 lần nữa?
10. `daemon=True` của luồng tra cứu có tác dụng gì?
11. Khách bấm Hủy đúng lúc tiền về: lần lượt các lớp nào xử lý để tiền không bị mất?
12. Vì sao `getCartItems()` trả về bản sao chứ không trả `self._cart_items`?

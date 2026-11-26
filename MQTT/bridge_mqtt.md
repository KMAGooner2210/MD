# Giải thích chi tiết phần MQTT trong `kiosk_app/bridge.py`

> File: `Smart_Kiosk_Backup/kiosk_app/bridge.py` — 1442 dòng, lớp `KioskBridge` (cầu nối Python ↔ giao diện QML của kiosk). Phần CSDL của file đã học ở `hoc_tap/SQLite/bridge.md`; tài liệu này chỉ đi vào **MQTT** và những gì dính trực tiếp tới nó (luồng, khóa, polling dự phòng).
> Học trước: `00_kien_thuc_can_ban.md`, `webhook_server_mqtt.md` (bên **phát** `payment/success` — file này là bên **nhận**).
>
> **Thí nghiệm** (không đụng Pi, `.env` thật, `kiosk.db` thật, SePay thật):
> - Broker Mosquitto **riêng** ở cổng 18850 trong thư mục nháp (bật/tắt theo kịch bản); Mosquitto hệ thống của laptop không bị động tới.
> - Chạy **đúng lớp `KioskBridge`** trong một `QCoreApplication`, CSDL là **bản sao** `kiosk_ban_sao.db`, khóa HMAC thử, thông tin ngân hàng giả.
> - Hàm hỏi SePay (`requests.get`) được thay bằng hàm giả luôn trả “chưa có giao dịch” → **không gọi ra Internet**.
> - Một client “gián điệp” đăng ký `kitchen/#` để xem kiosk phát gì xuống bếp; một client khác đóng vai webhook phát `payment/success`.

---

## 0. Bức tranh tổng thể (đọc trước)

### 0.1 Kiosk làm gì với MQTT
Kiosk là client MQTT **vừa nhận vừa phát**:

```
                 payment/success (QoS 1)                kitchen/new_order (QoS 1)
 webhook ─────────▶ broker ─────────▶ KIOSK ─────────▶ broker ─────────▶ màn hình bếp, ESP32 giọng nói
                                       │   kitchen/inventory (QoS 1, RETAIN)
                                       └──────────────▶ broker ─────────▶ màn hình bếp
```

| Hướng | Topic | Khi nào | Hàm |
|---|---|---|---|
| **Nhận** | `payment/success` | Webhook báo tiền về | `_on_mqtt_message` (L285) |
| **Phát** | `kitchen/inventory` (retain) | Sau khi chốt đơn và trừ kho | `_publish_inventory_snapshot` (L816) |
| **Phát** | `kitchen/new_order` | Sau khi chốt đơn | `_publish_kitchen_order` (L905) |

### 0.2 Bốn luồng chạy cùng lúc trong kiosk
| Luồng | Tạo bởi | Làm gì |
|---|---|---|
| **Luồng chính (Qt/QML)** | `app.exec()` | Vẽ giao diện, nhận bấm nút (`checkout`, `cancelPayment`…) |
| **Luồng mạng paho** (`paho-mqtt-client-…`) | `loop_start()` | Nhận/gửi gói MQTT, **gọi các callback** `on_connect`, `on_message`… |
| **Luồng polling** (`PaymentPolling-DH…`) | `checkout()` | Hỏi SePay mỗi 15 s |
| (luồng phụ của Qt) | | |

Cả luồng paho lẫn luồng polling đều có thể gọi `_finalize_payment()` cho **cùng một đơn** → phải có **khóa** (`self._payment_lock`) và các cờ trạng thái. Đây là phần khó nhất của file.

### 0.3 Kết quả chạy thử

**Kịch bản 1 — luồng thanh toán đầy đủ:**

| # | Thao tác | Kiosk phản ứng | Kiểm chứng |
|---|---|---|---|
| 1 | Khởi tạo `KioskBridge` | `connected = False` ngay sau khởi tạo → 1 s sau `True`, đã đăng ký `payment/success` | Không chặn giao diện chờ broker |
| 2 | Thêm 2 ly Cà phê đen đá, `checkout()` | Tạo đơn `DH71128480`, QR, bắt đầu polling | DB: `PENDING` |
| 3 | Tin ký bằng **khóa sai** | `Bỏ qua tín hiệu thanh toán MQTT: bad_signature` | DB không đổi |
| 4 | Tin đúng khóa nhưng **cũ 200 s** | `stale_message` | DB không đổi |
| 5 | Tin **thật** | Lưu đơn, phát kho + đơn bếp, `paymentSuccess` | DB `PAID / pending / 36000`; kho Đường **4990 → 4970**, Cà phê bột **4980 → 4940** |
| 6 | Tin thật đến **lần 2** | Chỉ in “Nhận xác nhận…”, không làm gì thêm | Kho **giữ nguyên** 4970 / 4940 |
| 7 | Tin cho đơn **đã hết hạn** `DH89314220` | `THANH TOÁN MUỘN … Đã ghi LATE_PAID` | DB `LATE_PAID / 17000` |
| 8 | Bếp (gián điệp) nhận được | `kitchen/inventory` **5542 byte**, 62 nguyên liệu; `kitchen/new_order` **361 byte**, `Cà phê đen đá ×2` | Cả hai kiểm chữ ký **đúng** |
| 9 | Một “màn hình bếp” mới vào **sau**, đăng ký `kitchen/inventory` | Nhận ngay bản kho, `retain=True` | Tác dụng của retain |

**Luồng chạy của từng đoạn code** (in tên luồng ở mỗi bước):
```
on_connect / on_message / on_disconnect     → [paho-mqtt-client-]   (luồng mạng paho)
SLOT paymentSuccess, cartUpdated (phía nhận) → [MainThread]          (luồng giao diện)
```

**Kịch bản 2 — broker có sự cố:**

| # | Tình huống | Kết quả |
|---|---|---|
| A | Broker **tắt** lúc kiosk khởi động | Kiosk vẫn khởi động xong (`connected=False`), không lỗi |
| B | Bật broker sau 3 s | paho tự nối lại sau **3,7 s**, `on_connect` chạy, **đăng ký lại** `payment/success` |
| C | Tin cho mã đơn lạ | `on_message` chạy trên luồng paho, mất **1 ms**: `Nhận tiền cho mã đơn lạ DH00000000, bỏ qua` |
| D | Broker khởi động lại khi kiosk đang chạy | `on_disconnect` (`Unspecified error`) → tự nối lại, **đăng ký lại** (mid tăng 1 → 2) |
| E | Phát đơn bếp khi broker **đang tắt** | Hàm in “Gửi đơn … **thất bại**. rc=4” và trả `False` — **nhưng** sau khi broker bật lại, bếp **vẫn nhận được** `kitchen/new_order` (xem §6.3) |

---

## 1. Cấu hình MQTT (L10, L32–37, L56–89)

```python
import paho.mqtt.client as mqtt
from backend.mqtt_security import MqttMessageRejected, configure_client, sign_message, verify_message

MQTT_BROKER = os.environ.get("MQTT_BROKER", "broker.emqx.io")
MQTT_PORT = int(os.environ.get("MQTT_PORT", 1883))
MQTT_KITCHEN_TOPIC   = os.environ.get("MQTT_KITCHEN_TOPIC", "kitchen/new_order")
MQTT_PAYMENT_TOPIC   = os.environ.get("MQTT_PAYMENT_TOPIC", "payment/success")
MQTT_INVENTORY_TOPIC = os.environ.get("MQTT_INVENTORY_TOPIC", "kitchen/inventory")
MQTT_SHARED_SECRET   = os.environ.get("MQTT_SHARED_SECRET", "")

POLLING_INTERVAL_SECONDS = 15
POLLING_TIMEOUT_SECONDS = 300
```
- Khác webhook: **tên topic cũng đọc được từ `.env`** (mặc định trùng các bên khác). Tiện cho thử nghiệm (vd. đổi sang `test/payment/success` để không lẫn với hệ thống thật trên broker công cộng), nhưng phải nhớ đổi **đồng bộ** ở mọi bên — webhook và firmware viết cứng tên topic.
- Bốn hằng `MQTT_*` được đọc **một lần lúc import** (sau `load_dotenv(override=True)` ở L45–51).
- `POLLING_*`: 15 s/lần, tối đa 300 s (= thời gian QR còn hiệu lực).

Kiểm tra cấu hình lúc khởi động (`_validate_configuration`, L164): thiếu `MQTT_SHARED_SECRET` → in cảnh báo **“Mọi tín hiệu thanh toán MQTT sẽ bị TỪ CHỐI; Kiosk chỉ còn xác nhận thanh toán qua polling SePay”** — tức là hệ thống **vẫn chạy**, chỉ mất đường nhanh.

---

## 2. Khởi tạo kết nối — `setup_mqtt()` (L197–231)

```python
def setup_mqtt(self):
    try:
        self.mqtt_client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
    except AttributeError:
        self.mqtt_client = mqtt.Client()              # paho 1.x

    self.mqtt_client.on_connect = self._on_mqtt_connect
    self.mqtt_client.on_disconnect = self._on_mqtt_disconnect
    self.mqtt_client.on_message = self._on_mqtt_message
    configure_client(self.mqtt_client, role="KIOSK")

    try:
        self.mqtt_client.connect_async(MQTT_BROKER, MQTT_PORT, 60)
        self.mqtt_client.loop_start()
        print(f"---> [MQTT] Đang kết nối broker {MQTT_BROKER}:{MQTT_PORT}")
    except Exception as error:
        print(f"---> [LỖI MQTT] Không thể khởi tạo kết nối broker: {error}")
```

**Cú pháp**
- `self.mqtt_client.on_connect = self._on_mqtt_connect` — gán **phương thức gắn với đối tượng** (*bound method*) làm callback. Khi paho gọi `on_connect(client, userdata, …)`, Python tự thêm `self` ở đầu → trong callback dùng được `self._payment_lock`, `self._finalize_payment`…
- `configure_client(..., role="KIOSK")` → tài khoản `MQTT_KIOSK_USERNAME/_PASSWORD` (ACL cho phép: đọc `payment/success`, ghi `kitchen/new_order`, `kitchen/inventory`).

**`connect_async` + `loop_start` — khác webhook ở đâu?**
| | Webhook (`connect`) | Kiosk (`connect_async`) |
|---|---|---|
| Lúc gọi | **Chặn**, nối TCP ngay | **Không chặn**, chỉ ghi nhớ địa chỉ |
| Broker chưa sẵn sàng | Ném lỗi ngay → 502 | Không lỗi; luồng nền **tự thử lại** |
| Thời gian sống | Vài giây cho mỗi giao dịch | **Suốt đời ứng dụng** (kết nối lâu dài) |

Thí nghiệm A–B: broker tắt lúc khởi động → kiosk vẫn lên bình thường; bật broker 3 s sau → tự nối lại sau 3,7 s. Lý do khoảng chờ: paho nối lại với thời gian chờ **tăng dần** (mặc định 1 s, 2 s, 4 s… tối đa 120 s), nên có thể lỡ nhịp vài giây sau khi broker sống lại.
→ **Vì sao quan trọng:** Pi khởi động, kiosk có thể chạy **trước** khi mạng/broker sẵn sàng. Nếu dùng `connect()` chặn, kiosk sẽ lỗi ngay lúc bật máy.

`try/except` quanh `connect_async` gần như không bao giờ rơi vào nhánh lỗi (hàm không mở mạng), chỉ phòng lỗi cấu hình kiểu sai địa chỉ không phân giải được.

---

## 3. `_on_mqtt_connect` — đăng ký **mỗi lần** nối (L233–269)

```python
def _on_mqtt_connect(self, client, userdata, flags, reason_code, properties=None):
    try:
        connect_code = int(reason_code)
    except (TypeError, ValueError):
        connect_code = reason_code

    if connect_code == 0:
        subscribe_result = client.subscribe(MQTT_PAYMENT_TOPIC, qos=1)
        print(f"---> [MQTT] Đã kết nối broker: {MQTT_BROKER}")
        print(f"---> [MQTT] Đang nghe topic thanh toán: {MQTT_PAYMENT_TOPIC}, subscribe_result={subscribe_result}")
    else:
        print(f"---> [MQTT] Kết nối broker thất bại. reason_code={reason_code}")
```

**Cú pháp**
- Chữ ký 5 tham số là của **API callback VERSION2**; `properties=None` có giá trị mặc định để chạy được cả với paho 1.x (gọi với 4 tham số).
- `int(reason_code)`: ở paho 2.x, `reason_code` là đối tượng `ReasonCode` (in ra `Success`), ép `int` ra 0; ở 1.x nó đã là số. `try/except` để chạy được cả hai.
- `subscribe_result` là tuple `(mã_lỗi, message_id)` — thí nghiệm thấy `(MQTT_ERR_SUCCESS, 1)`, rồi `…, 2)`, `…, 4)` ở các lần nối lại.

**Vì sao đăng ký trong `on_connect` mà không đăng ký ngay sau `connect_async`?**
1. Lúc gọi `connect_async`, **chưa có kết nối** → không đăng ký được.
2. Kiosk dùng **phiên sạch**: mỗi lần mất kết nối, broker **quên** mọi đăng ký. Đặt `subscribe` trong `on_connect` thì **mỗi lần nối lại đều tự đăng ký lại**. Thí nghiệm D: broker khởi động lại → `on_connect` được gọi lại → dòng “Đang nghe topic thanh toán” in lại.
→ Đây là **mẫu chuẩn** cho mọi client MQTT dùng phiên sạch.

`qos=1` khi đăng ký: chặng broker → kiosk cũng là QoS 1 (khớp QoS webhook phát).

## 4. `_on_mqtt_disconnect` (L271–283)
Chỉ in thông báo. Không cần tự nối lại vì **luồng `loop_start` tự làm** (thấy ở thí nghiệm D, E). `reason_code`: `Unspecified error` khi broker chết đột ngột, `Normal disconnection` khi chủ động `disconnect()`.

---

## 5. `_on_mqtt_message` — nhận tín hiệu thanh toán (L285–380)

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

        if status != "PAID":        ... return
        if not order_code:          ... return
        if amount is None:          ... return

        print(f"---> [MQTT] Nhận xác nhận thanh toán đơn {order_code}")
        self._finalize_payment(order_code, amount, source="webhook")

    except UnicodeDecodeError as error: ...
    except json.JSONDecodeError as error: ...
    except Exception as error:
        print(f"---> [LỖI MQTT] Không thể xử lý tín hiệu thanh toán: {error}")
```
(Rút gọn phần in thông báo.)

### 5.1 Từng lớp kiểm tra
| Lớp | Chặn gì | Thí nghiệm |
|---|---|---|
| `msg.topic != MQTT_PAYMENT_TOPIC` | Tin của topic khác (phòng sau này đăng ký thêm) | |
| `verify_message(...)` chữ ký | Tin giả, tin bị sửa | Khóa sai → `bad_signature` |
| `verify_message(...)` `max_age_seconds=120` | **Phát lại** tin cũ (ai đó ghi lại tin thật rồi gửi lại sau) | Cũ 200 s → `stale_message` |
| `status != "PAID"` | Tin trạng thái khác | |
| Thiếu `order_code` / `amount` | Tin thiếu dữ liệu | |
| `except Exception` ngoài cùng | **Mọi lỗi bất ngờ** | |

**Vì sao phải bọc `except Exception` trong callback?** Callback chạy trên **luồng mạng paho**, và paho mặc định **không nuốt lỗi** trong callback (`suppress_exceptions = False`). Đã thử với paho 2.1.0: một `on_message` ném `ValueError` khi nhận tin thứ hai, rồi phát tiếp 2 tin nữa:
```
Exception in thread paho-mqtt-client-: ... ValueError: loi trong callback
nhận được: [b'1', b'loi']   | luồng paho còn sống: False | connected: True
```
→ Lỗi **giết luồng mạng**: hai tin sau **không bao giờ tới**, mà `is_connected()` vẫn báo `True` — client “điếc” mà không ai biết. Với kiosk, điều đó nghĩa là mọi tín hiệu thanh toán sau đó chỉ còn dựa vào polling. Vì vậy `_on_mqtt_message` bọc toàn bộ thân trong `try … except Exception` và chỉ in lỗi.

`msg.payload` là **bytes**; `verify_message` tự `decode("utf-8")` và `json.loads`. Hai nhánh `except UnicodeDecodeError` / `json.JSONDecodeError` thực tế hiếm chạy vì `verify_message` đã đổi các lỗi này thành `MqttMessageRejected("invalid_envelope")`.

### 5.2 ⚠ Callback chạy trên luồng nào?
Thí nghiệm in tên luồng: `on_message` chạy trên **`paho-mqtt-client-…`**, không phải luồng giao diện. Nghĩa là `_finalize_payment()` — gồm ghi CSDL, phát MQTT, báo giao diện — **cũng chạy trên luồng paho**. Hai hệ quả:
1. **Không được đụng trực tiếp vào giao diện** từ đây (Qt chỉ cho sửa giao diện trên luồng chính). Code chỉ **phát signal** — xem §7.
2. Trong lúc `_finalize_payment` chạy (ghi CSDL, có thể chờ khóa tới 10 s nếu CSDL bận), luồng paho **không xử lý gói nào khác**. Bình thường chỉ vài chục mili giây (thí nghiệm: tin tới 3,13 s → giao diện nhận `paymentSuccess` lúc 3,15 s).

---

## 6. Phát tin xuống bếp

### 6.1 `_publish_inventory_snapshot()` (L816–903)
```python
inventory_payload = {
    "version": 1,
    "event": "inventory.snapshot",
    "created_at": datetime.now().astimezone().isoformat(timespec="seconds"),
    "items": [
        {"id": i["id"], "name": i["name"], "unit": i["unit"], "quantity": i["quantity"],
         "low": i["quantity"] <= i["low_stock_threshold"]}
        for i in database.get_all_ingredients()
    ],
}
if not MQTT_SHARED_SECRET: ... return False           # không ký được thì không phát
payload_text = sign_message(inventory_payload, MQTT_SHARED_SECRET)
publish_info = self.mqtt_client.publish(MQTT_INVENTORY_TOPIC, payload_text, qos=1, retain=True)
if publish_info.rc != mqtt.MQTT_ERR_SUCCESS: ... return False
print("---> [MQTT] Đã đưa bản cập nhật tồn kho vào hàng đợi gửi.")
```
- **Ảnh chụp toàn bộ kho** (*snapshot*), không phải “phần thay đổi”: bên nhận chỉ việc thay cả bảng, không cần biết trạng thái trước → đơn giản, không lệch.
- `"low"` **tính sẵn** ở kiosk → firmware không cần biết ngưỡng.
- `datetime.now().astimezone().isoformat(timespec="seconds")` → chuỗi kiểu `2026-11-26T23:40:12+07:00` (có múi giờ).
- `"version": 1`, `"event"`: **phiên bản hợp đồng** — giống `/api/orders/pending`.
- **`retain=True`**: thí nghiệm 9 — màn hình bếp **mới bật/khởi động lại** nhận ngay bảng kho (`retain=True`), không phải chờ lần bán tiếp theo.
- Câu in cẩn thận: **“đưa vào hàng đợi gửi”**, không nói “đã gửi” — vì `publish()` không chặn (học ở `00_kien_thuc_can_ban.md` D4).
- Kích thước thật: **5542 byte** với 62 nguyên liệu (~89 byte/nguyên liệu sau khi ký). Firmware bếp đặt bộ đệm PubSubClient **8192 byte** → còn chỗ cho khoảng 90 nguyên liệu; quá mức đó PubSubClient **bỏ tin âm thầm** (màn hình bếp vẫn có kho nhờ đồng bộ HTTP 30 s). Firmware cũng chỉ giữ tối đa `MAX_INGREDIENTS = 80`.

### 6.2 `_publish_kitchen_order()` (L905–1020)
```python
kitchen_payload = {
    "version": 1, "event": "order.created", "order_id": order_code,
    "created_at": datetime.now().astimezone().isoformat(timespec="seconds"),
    "items": [
        {"product_id": item.get("product_id", 0),
         "name": item.get("base_name", item.get("name", "").split(" (", 1)[0].strip()),
         "quantity": item.get("qty", 1),
         "options": {"size": item.get("size", ""), "ice": item.get("ice", "")},
         "note": item.get("note", "")}
        for item in order_items
    ],
}
... sign_message ...
publish_info = self.mqtt_client.publish(MQTT_KITCHEN_TOPIC, payload_text, qos=1, retain=False)
```
- **Cùng hình dạng** với từng phần tử của `GET /api/orders/pending` (đã học ở `kds_firmware_http.md`) → firmware dùng **một** hàm `parseOrderObject()` cho cả MQTT lẫn HTTP.
- `name`: ưu tiên `base_name` (“Cà phê đen đá”); món cũ không có `base_name` thì cắt phần trong ngoặc của `name` (“Cà phê đen đá (Size S, 100% Đá)” → “Cà phê đen đá”). `str.split(" (", 1)[0]` = lấy phần trước `" ("` đầu tiên.
- **`retain=False`**: đơn mới là **sự kiện**. Nếu retain, mỗi lần màn hình bếp khởi động lại sẽ nhận lại đơn cuối cùng (kể cả đã làm xong). Đơn đang chờ khi bếp khởi động lại được lấy qua **HTTP** `/api/orders/pending`.
- Thí nghiệm 8: tin 361 byte, kiểm chữ ký đúng, nội dung `{'product_id': 24, 'name': 'Cà phê đen đá', 'quantity': 2, 'options': {'size': '', 'ice': ''}, 'note': ''}`.

### 6.3 Phát khi đang mất kết nối — `rc=4` nhưng tin vẫn đi (thí nghiệm E)
```
---> [MQTT] Gửi đơn DH12121212 thất bại. rc=4. KDS sẽ phục hồi đơn qua API pending.
_publish_kitchen_order trả về: False
... broker bật lại, kiosk tự nối lại ...
bếp nhận được sau khi broker bật lại: ['kitchen/new_order']
```
- `rc=4` = `MQTT_ERR_NO_CONN` (“chưa có kết nối”). Nhưng với **QoS ≥ 1**, paho 2.x **vẫn giữ tin trong hàng đợi** và gửi khi nối lại được.
- Hệ quả: thông báo “thất bại” là **báo động giả** trong trường hợp mất kết nối ngắn. Không gây hại: màn hình bếp **tự khử trùng** (`addOrderCardUI` bỏ qua đơn đã có thẻ), và nếu kiosk khởi động lại trước khi nối được thì tin mất nhưng bếp vẫn lấy đơn qua HTTP 30 s.
- (Ghi chú, chưa sửa.) Nếu muốn nhật ký chính xác hơn có thể phân biệt `MQTT_ERR_NO_CONN` (“đã xếp hàng, sẽ gửi khi nối lại”) với các mã lỗi khác.

### 6.4 Không ký được thì không phát
Cả hai hàm đều `return False` nếu thiếu `MQTT_SHARED_SECRET` — **không bao giờ phát tin chưa ký** (bếp sẽ từ chối tin chưa ký, và broker công cộng thì ai cũng đọc/ghi được). Bếp vẫn có dữ liệu nhờ đồng bộ HTTP.

---

## 7. Từ luồng paho sang giao diện — signal của Qt

```python
class KioskBridge(QObject):
    cartUpdated = Signal()
    qrGenerated = Signal(str)
    paymentSuccess = Signal()
...
# cuối _finalize_payment (đang chạy trên luồng paho):
self.paymentSuccess.emit()
self.clearCart()          # bên trong: self.cartUpdated.emit()
```
QML (`kiosk_app/qml/main.qml` L308–321):
```qml
Connections {
    target: backend
    function onCartUpdated() { updateCartInfo() }
    function onQrGenerated(link) { paymentPopup.qrSource = link; paymentPopup.open() }
    function onPaymentSuccess() { paymentPopup.close(); window.reloadMenu() }
}
```
- `Signal()` (= `pyqtSignal`) khai ở **cấp lớp**; `main.py` đưa đối tượng vào QML với tên `backend` (`setContextProperty("backend", bridge)`).
- `emit()` từ luồng paho, nhưng **phía nhận chạy trên luồng chính** — thí nghiệm: `SLOT paymentSuccess chay o luong nay → [MainThread]`.
- **Cơ chế:** khi signal được phát từ một luồng **khác** luồng của đối tượng nhận, Qt dùng **kết nối xếp hàng** (*queued connection*): đóng gói lời gọi thành một sự kiện, đặt vào **hàng đợi sự kiện của luồng chính**; vòng lặp sự kiện của luồng chính lấy ra và gọi slot. Đây là cách **an toàn duy nhất** để luồng nền “báo” cho giao diện.
- Vì vậy `_finalize_payment` chỉ `emit`, **không** gọi thẳng hàm giao diện nào.

---

## 8. Hai đường xác nhận và khóa `_payment_lock`

### 8.1 Đường nhanh và đường chắc
| | MQTT (đường nhanh) | Polling SePay (đường chắc) |
|---|---|---|
| Ai chủ động | Webhook đẩy qua broker | Kiosk tự hỏi `my.sepay.vn` |
| Độ trễ | Gần như tức thời | Tới 15 s |
| Mất khi | Kiosk/broker rớt đúng lúc; webhook lỗi | SePay API lỗi |
| Chạy trên | Luồng paho | Luồng `PaymentPolling-DH…` |
| Gọi | `_finalize_payment(…, source="webhook")` | `_finalize_payment(…, source="polling")` |

Polling **chỉ chạy trong lúc có đơn đang chờ** (bắt đầu ở `checkout()`, dừng khi đã chốt/hủy/hết 300 s) — không hỏi SePay liên tục cả ngày.

### 8.2 Vì sao cần khóa
Hai luồng có thể cùng gọi `_finalize_payment` cho **cùng một đơn** gần như cùng lúc, trong khi luồng chính có thể đang `cancelPayment()`. Trạng thái dùng chung:
```python
self._payment_lock = threading.Lock()
self._current_order_code      # đơn đang chờ
self._payment_confirmed       # đã chốt chưa
self._payment_in_progress     # đang có đơn chờ không
self._pending_order_items / _pending_order_amount   # ảnh chụp giỏ hàng lúc tạo QR
self.is_polling
```
Trong `_finalize_payment` (L590):
```python
with self._payment_lock:
    if self._payment_confirmed: return          # luồng kia đã chốt → thôi
    if order_code != self._current_order_code: ...  # không phải đơn đang chờ → ghi nhận muộn
    ...
    self._payment_confirmed = True              # ĐÁNH DẤU trong khóa, rồi mới ghi CSDL
```
- **Đặt cờ trong khóa, làm việc chậm ngoài khóa**: kiểm tra + đánh dấu “tôi nhận việc này” xảy ra nguyên khối; ghi CSDL (chậm) làm **sau khi nhả khóa** để không chặn luồng khác lâu.
- **Lớp bảo vệ thứ hai ở CSDL**: `save_new_order` dùng `BEGIN IMMEDIATE` và trả `duplicate_order` nếu đơn đã `PAID` (demo ở `hoc_tap/SQLite/demo`: hai luồng cùng chốt → một `ok`, một `duplicate_order`, kho trừ đúng 1 lần).
- Lưu CSDL thất bại → **gỡ cờ** `_payment_confirmed = False` để đường còn lại (hoặc tin lặp lại) thử tiếp.

### 8.3 Tin đến lần 2 và tin cho đơn đã đóng (thí nghiệm 6, 7)
Sau khi chốt, `_current_order_code = None` → tin thứ hai rơi vào nhánh “không phải đơn đang chờ” → `_record_late_payment()` → `database.mark_order_late_paid()`:
| Trạng thái đơn trong CSDL | Kết quả | In ra |
|---|---|---|
| `PAID` (tin lặp) | `not_closed` | (im lặng — đúng như thí nghiệm 6) |
| `CANCELLED/EXPIRED` | `late_paid` → `LATE_PAID` | “THANH TOÁN MUỘN … cần hoàn tiền” (thí nghiệm 7) |
| `LATE_PAID` (ghi rồi) | `already_recorded` | (im lặng) |
| Không có mã | `unknown_order` | “Nhận tiền cho mã đơn lạ … bỏ qua” (thí nghiệm C) |

→ **QoS 1 cho phép trùng**, và nhờ chuỗi kiểm tra này, tin trùng **không bao giờ** trừ kho hay gửi bếp hai lần.

---

## 9. Thứ tự việc sau khi tiền về (cuối `_finalize_payment`)
```
1. Kiểm tra đơn + số tiền (trong khóa), đặt _payment_confirmed = True
2. database.save_new_order(...)          → PAID + trừ kho (một giao dịch)
   └ lỗi → gỡ cờ, KHÔNG gửi bếp, KHÔNG xóa giỏ
3. _publish_inventory_snapshot()         → kitchen/inventory (retain)
4. _publish_kitchen_order(...)           → kitchen/new_order
5. paymentSuccess.emit(); clearCart()    → giao diện đóng QR, tải lại thực đơn
6. Xóa trạng thái đơn đang chờ (trong khóa)
```
**Vì sao lưu CSDL trước, phát MQTT sau?** CSDL là **nguồn sự thật**. Nếu phát đơn xuống bếp trước rồi lưu lỗi, bếp sẽ làm một đơn **không tồn tại** trong hệ thống. Làm ngược lại: lưu được rồi mới báo; báo lỗi thì bếp vẫn tự lấy được đơn qua HTTP.

---

## 10. Tổng kết

### 10.1 So với webhook
| | Webhook | Kiosk |
|---|---|---|
| Vai trò | Chỉ phát | Nhận + phát |
| Kết nối | Mở – gửi – đóng mỗi giao dịch | **Lâu dài** suốt đời ứng dụng |
| Nối | `connect()` chặn | `connect_async()` không chặn, tự nối lại |
| Đăng ký | — | Trong `on_connect` (đăng ký lại mỗi lần nối) |
| Callback | Không có | `on_connect`, `on_disconnect`, `on_message` (chạy trên luồng paho) |
| Báo giao diện | — | Signal Qt → kết nối xếp hàng sang luồng chính |
| Mất tin | SePay không gửi lại (đã trả 200) | Có polling SePay dự phòng |

### 10.2 Nguyên tắc thiết kế thấy được
- **Đăng ký trong `on_connect`** để sống sót qua mất kết nối.
- **Callback ngắn, bắt mọi lỗi**, không đụng giao diện — chỉ phát signal.
- **Kiểm tra – đánh dấu trong khóa, việc chậm ngoài khóa**; CSDL là lớp chống trùng thứ hai.
- **Lưu trước, báo sau.**
- **Retain cho trạng thái, không retain cho sự kiện.**
- **Không ký được thì không phát**; mọi tin nhận đều kiểm chữ ký + độ mới.
- Mỗi đường MQTT đều có **đường HTTP dự phòng** (polling SePay, `/api/orders/pending`).

### 10.3 Ghi chú (không gây lỗi hiện tại, **chưa sửa**)
1. **Thông báo “Gửi … thất bại. rc=4” có thể là báo động giả** (§6.3): tin QoS 1 vẫn được paho xếp hàng và gửi sau khi nối lại.
2. **`_finalize_payment` chạy trên luồng paho** (§5.2): nếu CSDL bị khóa lâu (vd. ai đó mở DB Browser và để thay đổi chưa ghi), luồng paho bị chặn tới `busy_timeout` 10 s, trong thời gian đó không nhận/gửi được gói MQTT nào; keepalive 60 s nên chưa tới mức rớt kết nối.
3. **Tên topic đọc từ `.env` ở kiosk nhưng viết cứng ở webhook/firmware**: đổi ở một nơi sẽ làm lệch.
4. **Kích thước tin kho tăng theo số nguyên liệu**: 62 nguyên liệu = 5542 byte; khoảng trên 90 nguyên liệu sẽ vượt bộ đệm 8192 byte của firmware bếp (tin bị bỏ, bếp vẫn đồng bộ được qua HTTP).

---

## 11. Câu hỏi tự kiểm tra

1. Kiosk dùng `connect_async` còn webhook dùng `connect`. Giải thích vì sao mỗi bên chọn như vậy; chuyện gì xảy ra nếu kiosk dùng `connect` khi Pi khởi động mà broker chưa sẵn sàng?
2. Vì sao `subscribe` nằm trong `_on_mqtt_connect`? Thí nghiệm nào chứng minh nó được gọi lại?
3. `_on_mqtt_message` chạy trên luồng nào? Vì sao trong đó chỉ được `emit` signal mà không gọi thẳng hàm giao diện?
4. Giải thích “kết nối xếp hàng” (*queued connection*) của Qt bằng lời của bạn.
5. Liệt kê các lớp kiểm tra một tin `payment/success` phải vượt qua trước khi được chốt đơn.
6. Tin thanh toán đến hai lần. Đi theo code, chỉ ra vì sao kho chỉ bị trừ một lần (có **hai** lớp bảo vệ).
7. Vì sao `kitchen/inventory` có retain còn `kitchen/new_order` thì không? Màn hình bếp khởi động lại lấy đơn đang chờ bằng cách nào?
8. Vì sao lưu CSDL **trước** rồi mới phát `kitchen/new_order`?
9. `publish()` trả `rc=4` khi mất kết nối. Tin đó có bị mất không? Thí nghiệm nào trả lời?
10. Trong `_finalize_payment`, vì sao đặt `_payment_confirmed = True` **trong** khóa nhưng gọi `save_new_order` **ngoài** khóa?
11. Nếu thiếu `MQTT_SHARED_SECRET`, kiosk còn xác nhận được thanh toán không? Bằng đường nào, chậm hơn bao nhiêu?
12. Tin `payment/success` cho một mã đơn đã `EXPIRED` dẫn tới trạng thái gì trong CSDL? Vì sao không trừ kho?

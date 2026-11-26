# Giải thích chi tiết phần MQTT trong `servers/webhook_server.py`

> File: `Smart_Kiosk_Backup/servers/webhook_server.py` — 139 dòng. Phần MQTT: **L6, L15, L27–30, L37–67** và chỗ gọi ở route `/sepay_webhook`.
> Phần HTTP/Flask của file đã học ở `hoc_tap/HTTP_Flask/webhook_server.md`; phần ký HMAC đã học ở `hoc_tap/SQLite/mqtt_security.md`. Tài liệu này đi sâu vào **MQTT**: broker, gói tin, QoS, vòng lặp mạng của paho, và chuyện gì xảy ra khi broker có vấn đề.
>
> **Thí nghiệm** (không đụng Pi, `.env` thật, hay Mosquitto hệ thống đang chạy ở cổng 1883 trên laptop):
> - Dựng **2 broker Mosquitto riêng** trong thư mục nháp: cổng **18830** (không cần đăng nhập) và **18831** (bắt đăng nhập, tài khoản `webhook`).
> - Gọi **đúng hàm** `send_mqtt_signal()` của file (đã chặn `load_dotenv`, xóa mọi biến `MQTT_*` có sẵn, dùng khóa thử `khoa-thu-nghiem`).
> - Theo dõi gói tin bằng `mosquitto -v` (nhật ký broker) và `mosquitto_sub -d` (nhật ký phía nhận).
> - Cuối cùng tắt cả 2 broker thử; broker hệ thống không bị động tới.

---

## 0. Bức tranh tổng thể (đọc trước)

### 0.1 MQTT là gì — trong 1 phút
**MQTT** là giao thức **xuất bản / đăng ký** (*publish / subscribe*) rất nhẹ, chạy trên TCP (cổng 1883; 8883 nếu TLS).

```
  Bên phát (publisher)            BROKER (bưu điện)              Bên nhận (subscriber)
  webhook_server.py  ──PUBLISH──▶  Mosquitto / EMQX  ──PUBLISH──▶  kiosk_app/bridge.py
   topic "payment/success"          tra ai đăng ký                  đã SUBSCRIBE
                                    "payment/success"               "payment/success"
```

| Khái niệm | Nghĩa | Trong dự án |
|---|---|---|
| **Broker** | Máy chủ trung gian, nhận tin rồi chuyển cho ai đăng ký | `broker.emqx.io` (công cộng, mặc định) hoặc Mosquitto nội bộ (đang hoãn) |
| **Client** | Bất kỳ ai nối tới broker — vừa phát vừa nhận được | webhook, kiosk, API, 2 ESP32 |
| **Topic** | “Địa chỉ” dạng chuỗi có `/` | `payment/success`, `kitchen/new_order`… |
| **Payload** | Nội dung, là **bytes tùy ý** (broker không đọc) | JSON đã ký HMAC |
| **QoS** | Mức bảo đảm giao tin: 0, 1, 2 | webhook dùng **1** |
| **Retain** | Broker **giữ lại** tin cuối của topic cho người đăng ký sau | webhook **không** retain |

**So với HTTP (đã học):**

| | HTTP | MQTT |
|---|---|---|
| Mô hình | Hỏi – đáp, 1‑1 | Phát – đăng ký, 1‑nhiều, qua trung gian |
| Bên gửi có cần biết địa chỉ bên nhận? | Có (URL) | **Không** — chỉ cần biết broker và topic |
| Bên nhận phải mở cổng? | Có (máy chủ lắng nghe) | **Không** — bên nhận cũng chủ động nối ra broker |
| Biết bên nhận đã nhận chưa? | Có (mã trạng thái) | **Không** — chỉ biết **broker** đã nhận (QoS 1) |

Dòng cuối là chìa khóa để hiểu các phát hiện ở §6.

### 0.2 Vì sao webhook phải “chuyển tiếp” sang MQTT?
SePay gọi webhook qua Internet (qua đường hầm tới Pi). Webhook **không** tự xác nhận đơn trong CSDL (đã học: webhook không đụng DB). Nó chỉ cần **báo cho kiosk**: “đơn DH… đã được trả”. Kiosk là ứng dụng giao diện, **không mở cổng HTTP** để nhận; nhưng kiosk **có sẵn kết nối MQTT** để nghe. Vậy webhook phát lên topic `payment/success`, kiosk đang đăng ký topic đó sẽ nhận được.

```
SePay ──HTTPS──▶ webhook_server (Flask, Pi) ──MQTT PUBLISH──▶ broker ──▶ kiosk bridge.py
                       │                                                  ├ kiểm HMAC + ts
                       └── trả HTTP 200 / 502 cho SePay                    └ chốt đơn PAID trong CSDL
```

### 0.3 Kết quả chạy thử

| # | Tình huống | `send_mqtt_signal` trả | Thời gian | Tin tới bên nhận? |
|---|---|---|---|---|
| 1 | Broker mở, có bên nhận đang nghe | `True` | **1,00 s** | **Có** (209 byte) |
| 2 | Thiếu `MQTT_SHARED_SECRET` | `False` | 0,00 s | Không phát |
| 3 | Broker tắt (cổng đóng) | `False` (`[Errno 111] Connection refused`) | 0,00 s | — |
| 4 | IP broker không tồn tại | `False` | **5,00 s** | — |
| 5 | Broker bắt đăng nhập, **đúng** mật khẩu | `True` | 1,00 s | Có |
| 6 | Broker bắt đăng nhập, **SAI** mật khẩu | ⚠ **`True`** | **5,01 s** | **KHÔNG** (broker trả CONNACK mã 5 “not authorised” 3 lần) |
| 7 | Máy nhận TCP nhưng **không nói MQTT** (broker treo) | ⚠ **`True`** | **60,07 s** | **KHÔNG** |
| 8 | Bên nhận **offline** lúc phát, vào sau (phiên sạch) | `True` | 1,00 s | **Mất** — vào sau không nhận được gì |
| 9 | Bên nhận dùng **phiên bền vững** (`-c -i kiosk-thu2`), offline lúc phát | `True` | 1,00 s | **Có** — broker giữ hộ, giao khi nối lại |

Kiểm tra tin nhận được bằng `verify_message()`:
```
đúng khóa   → {'event': 'payment.success', 'order_code': 'DH12345678', 'amount': 45000, 'status': 'PAID', 'ts': 1795624393}
sai khóa    → bad_signature
sửa 45000 → 4500 trong tin → bad_signature
phát lại sau 121 s → stale_message
```

Đo từng bước (paho 2.1.0, broker cục bộ):
```
connect 0.001s | publish + chờ PUBACK 0.001s | loop_stop 1.001s | disconnect 0.000s
```
→ Gửi thật chỉ mất ~2 ms; **1 giây** là do `loop_stop()` (giải thích §4.6).

---

## 1. Import và cấu hình MQTT (L6, L15, L27–30)

```python
import paho.mqtt.client as mqtt                                     # L6
from backend.mqtt_security import configure_client, sign_message    # L15

MQTT_BROKER = os.environ.get("MQTT_BROKER", "broker.emqx.io")       # L27
MQTT_PORT = int(os.environ.get("MQTT_PORT", 1883))                  # L28
MQTT_TOPIC = "payment/success"                                      # L29
MQTT_SHARED_SECRET = os.environ.get("MQTT_SHARED_SECRET", "")       # L30
```

**Cú pháp**
- `import paho.mqtt.client as mqtt` — **paho‑mqtt** là thư viện MQTT của Eclipse; `as mqtt` đặt bí danh ngắn. `requirements.txt` ghim `paho-mqtt>=2.0,<3.0`.
- `int(os.environ.get("MQTT_PORT", 1883))` — biến môi trường luôn là **chuỗi**, phải ép `int`; mặc định là số `1883` (ép `int(1883)` vẫn đúng).
- Hằng ở **cấp module** → đọc **một lần** lúc import. Vì file gọi `load_dotenv(override=True)` trước đó (L17–23), giá trị trong `.env` **ghi đè** biến môi trường của hệ thống.

**Logic & vì sao**
- Mặc định `broker.emqx.io` — broker **công cộng**, ai cũng nối được, không cần tài khoản. Tiện khi phát triển, nhưng **ai cũng có thể phát** lên `payment/success` → đó là lý do **mọi tin phải ký HMAC** (học ở `mqtt_security.md`; thử giả mạo sẽ học ở `thu_gia_mao_mqtt.sh`).
- Topic `payment/success` **viết cứng**, phải trùng `MQTT_PAYMENT_TOPIC` của kiosk (`bridge.py` L73 — bên đó đọc được từ biến môi trường, mặc định cùng giá trị).
- Khóa HMAC tách riêng khỏi mật khẩu broker: mật khẩu chặn **ai được nối**, chữ ký chặn **tin giả** — hai lớp độc lập.

---

## 2. Toàn bộ `send_mqtt_signal()` (L37–67)

```python
def send_mqtt_signal(order_code, amount):
    """Phát tín hiệu thanh toán đã KÝ HMAC; thiếu MQTT_SHARED_SECRET thì không phát."""
    if not MQTT_SHARED_SECRET:                                          # ①
        print(" ---[BẢO MẬT] Chưa cấu hình MQTT_SHARED_SECRET: không phát tín hiệu thanh toán.")
        return False
    try:
        try:
            client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)      # ②
        except AttributeError:
            client = mqtt.Client()
        configure_client(client, role="WEBHOOK")                        # ③
        client.connect(MQTT_BROKER, MQTT_PORT, 60)                      # ④
        client.loop_start()                                             # ⑤
        message = sign_message(                                         # ⑥
            {"event": "payment.success", "order_code": order_code,
             "amount": amount, "status": "PAID"},
            MQTT_SHARED_SECRET,
        )
        info = client.publish(MQTT_TOPIC, message, qos=1)               # ⑦
        info.wait_for_publish(timeout=5)                                # ⑧
        client.loop_stop()                                              # ⑨
        client.disconnect()                                             # ⑩
        print(f" ---[WEBHOOK] Đã gửi tín hiệu MQTT cho đơn: {order_code}")
        return True
    except Exception as e:
        print(f" ---Lỗi kết nối MQTT: {e}")
        return False
```

Vòng đời một lần gọi, đối chiếu **nhật ký thật** của broker (thí nghiệm 1):

```
 webhook (paho)                                  broker 18830
 ④ connect() ── TCP SYN/ACK ──────────────────▶  New connection from 127.0.0.1:40631
             ── CONNECT (p2, c1, k60) ────────▶  New client connected ... as auto-1BFE072D-...
             ◀── CONNACK (0, 0) ─────────────── Sending CONNACK
 ⑦ publish() ── PUBLISH (d0,q1,r0,m1, 209 B) ─▶  Received PUBLISH ... 'payment/success'
                                                 Sending PUBLISH to <kiosk> ...   (chuyển tiếp)
             ◀── PUBACK (m1) ────────────────── Sending PUBACK
 ⑧ wait_for_publish() trả về
 ⑨ loop_stop()   (~1 s)
 ⑩ disconnect() ── DISCONNECT ───────────────▶  Received DISCONNECT ... disconnected.
```

Giải mã ký hiệu trong nhật ký Mosquitto:
| Ký hiệu | Nghĩa |
|---|---|
| `p2` | Phiên bản giao thức nội bộ của Mosquitto = **MQTT 3.1.1** (paho mặc định) |
| `c1` | **Clean session** = 1: phiên sạch, ngắt là broker quên client này |
| `k60` | **Keepalive** 60 s (tham số thứ 3 của `connect`) |
| `auto-1BFE…` | paho gửi **client id rỗng** → broker **tự đặt** tên ngẫu nhiên |
| `d0` | Không phải gửi lại (*dup* = 0) |
| `q1` | QoS 1 |
| `r0` | Không retain |
| `m1` | Mã gói (*message id*) = 1, dùng để ghép PUBLISH với PUBACK |
| `CONNACK (0, 0)` | (cờ phiên cũ = 0, **mã kết quả = 0: chấp nhận**). Mã 5 = **không có quyền** (thí nghiệm 6) |

---

## 3. Từng bước — cú pháp, logic, vì sao

### ① Không có khóa thì không phát
`if not MQTT_SHARED_SECRET: return False` — **fail‑closed**: thà không báo còn hơn phát tin **không ký** (kiosk cũng sẽ từ chối tin không ký). Thí nghiệm 2: `False` trong 0,00 s, không mở kết nối nào.

### ② Tạo client, tương thích paho 1.x và 2.x
```python
try:
    client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
except AttributeError:
    client = mqtt.Client()
```
- paho **2.0** bắt buộc chọn **phiên bản API callback** (chữ ký hàm `on_connect`, `on_message`… thay đổi giữa 1.x và 2.x). Không truyền thì 2.x báo lỗi.
- paho **1.x** không có `CallbackAPIVersion` → truy cập thuộc tính không tồn tại → `AttributeError` → dùng cách tạo cũ.
- Trên laptop có sẵn cả hai: Python của PlatformIO có paho **2.1.0**, Python hệ thống có paho **1.6.1** — đúng là tình huống đoạn code này phòng sẵn.
- Webhook **không đặt callback nào** (chỉ phát), nên khác biệt API callback không ảnh hưởng logic.

### ③ `configure_client(client, role="WEBHOOK")`
(Học chi tiết ở `mqtt_security.md`.) Đọc `MQTT_WEBHOOK_USERNAME`/`_PASSWORD`, nếu không có thì `MQTT_USERNAME`/`_PASSWORD`; có thì `client.username_pw_set(...)`. `MQTT_TLS=true` thì `client.tls_set(...)`.
→ Mỗi dịch vụ **một tài khoản riêng** (`webhook`, `kiosk`, `api`) để broker phân quyền theo topic (file `deploy/mosquitto/acl`, học sau). Thí nghiệm 5: broker 18831 ghi `u'webhook'` trong nhật ký.

### ④ `client.connect(host, port, keepalive=60)`
- **Chặn** cho tới khi **mở xong TCP và gửi gói CONNECT**. **Không chờ CONNACK** — câu trả lời của broker được xử lý **sau**, trong vòng lặp mạng (⑤).
- Lỗi tầng TCP ném **ngay** ra ngoại lệ:
  - Cổng đóng → `ConnectionRefusedError` → `False` trong 0,00 s (thí nghiệm 3).
  - IP không trả lời → hết `connect_timeout` mặc định **5 s** của paho 2.x → `False` (thí nghiệm 4: 5,00 s).
- **Keepalive 60 s**: nếu 60 s không có gói nào, client phải gửi `PINGREQ`; broker không nghe gì quá 1,5 × 60 s thì coi client đã chết. Với kết nối sống < 2 s như ở đây, con số này gần như không dùng tới — **trừ** thí nghiệm 7 (xem §6.2).

### ⑤ `client.loop_start()` — luồng mạng chạy nền
paho không tự đọc/ghi socket. Phải có **vòng lặp mạng** (*network loop*) chạy liên tục để: đọc CONNACK, gửi PUBLISH trong hàng đợi, nhận PUBACK, gửi PINGREQ, **tự nối lại** khi rớt.

| Cách | Chạy ở đâu | Dùng khi |
|---|---|---|
| `loop_forever()` | **Chặn** luồng hiện tại | Chương trình chỉ làm MQTT |
| `loop_start()` / `loop_stop()` | **Tạo luồng nền** riêng | Chương trình còn việc khác (Flask, Qt…) |
| `loop()` | Gọi tay từng nhịp | Tự quản vòng lặp (giống `mqttClient.loop()` trong firmware) |

Webhook dùng `loop_start()` vì đang ở giữa một request Flask, cần luồng nền lo gói tin trong khi luồng chính chờ ở ⑧. Kiosk (`bridge.py`) cũng dùng `loop_start()` nhưng chạy **suốt đời ứng dụng**.

### ⑥ `sign_message({...}, MQTT_SHARED_SECRET)`
Nội dung có 4 trường + `ts` tự thêm. Tin thật nhận được ở thí nghiệm 1 (209 byte):
```json
{"v":2,"sig":"2dfa32fc…cddf","payload":"{\"event\":\"payment.success\",\"order_code\":\"DH12345678\",\"amount\":45000,\"status\":\"PAID\",\"ts\":1795624177}"}
```
- `payload` là **chuỗi JSON lồng trong JSON** (thấy rõ dấu `\"`): bên nhận tính HMAC trên **đúng chuỗi đó**, không phải JSON dựng lại → ESP32 kiểm được mà không cần sắp xếp khóa.
- `"event": "payment.success"` — có tên sự kiện trong nội dung, phòng trường hợp sau này một topic chở nhiều loại tin.
- `amount` = số tiền SePay báo; kiosk đọc trường này (`bridge.py` L325) để ghi `paid_amount` cho đơn.

### ⑦ `client.publish(topic, payload, qos=1)`
- **Không chặn**: chỉ đặt gói vào **hàng đợi gửi** rồi trả về ngay một đối tượng `MQTTMessageInfo` (`info`). Luồng nền ⑤ mới thật sự gửi.
- `retain` mặc định `False`: tin thanh toán chỉ có nghĩa **tức thời**. Nếu retain, mỗi lần kiosk khởi động lại sẽ nhận lại tin thanh toán **cũ** (dù HMAC + `ts` 120 s và điều kiện CSDL vẫn chặn được, nhưng không nên tạo rác).

**Ba mức QoS:**
| QoS | Tên | Gói tin | Bảo đảm | Rủi ro |
|---|---|---|---|---|
| 0 | *At most once* | PUBLISH | Gửi một lần, không xác nhận | **Mất** |
| **1** | *At least once* | PUBLISH → **PUBACK** | Gửi lại tới khi có PUBACK | **Trùng** |
| 2 | *Exactly once* | PUBLISH → PUBREC → PUBREL → PUBCOMP | Đúng một lần | Chậm, nặng |

Chọn **QoS 1** vì: mất tin thanh toán thì tệ (khách đã trả tiền mà kiosk không biết), còn **trùng** thì vô hại — kiosk chốt đơn bằng `UPDATE … WHERE payment_status='PENDING'` + `rowcount` (đã học ở SQLite), tin thứ hai không đổi được gì. Đây là mẫu chuẩn: **QoS 1 + xử lý lũy đẳng ở bên nhận** thay cho QoS 2.

> Lưu ý: QoS là **từng chặng**. Chặng webhook → broker là QoS 1 (PUBACK từ broker). Chặng broker → kiosk dùng **QoS nhỏ hơn** trong hai giá trị: QoS của tin và QoS kiosk đăng ký (kiosk đăng ký `qos=1`, `bridge.py` L249–252). Nhật ký thí nghiệm 1 thấy đủ hai chặng: `Sending PUBACK to <webhook>` và `Received PUBACK from <bên nhận>`.

### ⑧ `info.wait_for_publish(timeout=5)`
Chặn luồng chính **tối đa 5 s**, tới khi luồng nền nhận được PUBACK. Không có dòng này thì ⑨⑩ có thể đóng kết nối **trước** khi tin kịp đi.
**Quan trọng:** khi hết 5 s, hàm **chỉ trả về** (không ném ngoại lệ). Muốn biết tin đã đi chưa phải hỏi thêm `info.is_published()` — **file không hỏi**. Xem §6.1.

### ⑨ `client.loop_stop()`
Đặt cờ “dừng” cho luồng nền rồi **chờ luồng đó kết thúc** (`join`).

### ⑩ `client.disconnect()`
Gửi gói `DISCONNECT` lịch sự. Broker thấy DISCONNECT “sạch” → **không** phát *last will* (webhook cũng không đặt will: nhật ký “No will message specified”).

### Mỗi lần gọi một kết nối mới
Mở → gửi → đóng trong ~1 s. Đơn giản, không phải giữ hay theo dõi kết nối sống, không lo kết nối “chết ngầm” giữa hai giao dịch cách nhau hàng giờ. Đây là **lựa chọn đã chốt** cho webhook.

---

## 4. Chỗ gọi trong route (nhắc lại từ tài liệu HTTP)

```python
sent = send_mqtt_signal(order_code, amount)
if sent:
    return jsonify({"success": True, ...}), 200
return jsonify({"success": False, ...}), 502
```
- `True` → **200** cho SePay: “đã nhận, xong việc” → SePay **không gửi lại**.
- `False` → **502 Bad Gateway**: “tôi là trạm trung gian, chặng sau hỏng” → SePay có thể **gửi lại webhook**.
→ Giá trị `True/False` của `send_mqtt_signal` **quyết định SePay có thử lại hay không**. Vì vậy độ chính xác của nó quan trọng — xem §6.

---

## 5. Bên nhận offline — vì sao vẫn không mất tiền của khách

Thí nghiệm 8 và 9 cho thấy một tính chất cốt lõi của MQTT:

| Bên nhận | Lúc phát đang offline | Kết quả |
|---|---|---|
| **Phiên sạch** (`clean_session=1`, id ngẫu nhiên) — như kiosk hiện nay (`bridge.py` L200–220) | Broker không biết ai đang chờ | **Tin mất** |
| **Phiên bền vững** (`clean_session=0`, id cố định) + đăng ký QoS 1 | Broker **giữ hộ** tin QoS ≥ 1 | Nối lại là nhận |

Webhook nhận PUBACK → trả 200 → SePay dừng. Nhưng PUBACK chỉ chứng minh **broker** đã nhận, **không** chứng minh kiosk đã nhận. Nếu kiosk đang rớt mạng đúng lúc đó, tin `payment/success` mất.

**Lưới an toàn có sẵn:** kiosk **tự hỏi SePay** mỗi **15 s** (`POLLING_INTERVAL_SECONDS = 15`, “POLLING SEPAY DỰ PHÒNG” ở `bridge.py` L382+) trong lúc chờ thanh toán; `_validate_configuration` (L165) còn ghi rõ: thiếu khóa MQTT thì “Kiosk chỉ còn xác nhận thanh toán qua polling SePay”. Nghĩa là MQTT là **đường nhanh**, polling là **đường chắc** — mất tin MQTT thì khách chờ lâu hơn tối đa khoảng 15 s chứ không mất đơn. (Phần polling sẽ học kỹ ở `bridge.py`.)

---

## 6. Hai phát hiện khi broker “nửa sống nửa chết” (ghi lại, **chưa sửa**)

### 6.1 Sai mật khẩu broker → vẫn trả `True` (thí nghiệm 6)
Diễn biến (nhật ký broker 18831):
```
New connection from 127.0.0.1:36481   → Sending CONNACK (0, 5) → disconnected, not authorised.
New connection from 127.0.0.1:34921   → Sending CONNACK (0, 5) → disconnected, not authorised.
New connection from 127.0.0.1:60029   → Sending CONNACK (0, 5) → disconnected, not authorised.
```
1. `connect()` (④) **thành công** vì TCP mở được và gói CONNECT đã gửi — nó **không chờ** CONNACK.
2. Broker trả CONNACK mã **5** và ngắt. Luồng nền paho **tự nối lại** (1 s, 2 s…) — bị từ chối tiếp.
3. `publish()` chỉ xếp hàng; tin **không bao giờ** được gửi.
4. `wait_for_publish(5)` hết giờ, **lặng lẽ trả về**.
5. Code đi tiếp, in “Đã gửi tín hiệu MQTT”, **`return True`** → SePay nhận **200** → không gửi lại.

Hậu quả thực tế: chỉ xảy ra khi **cấu hình sai** (đổi mật khẩu broker mà quên sửa `.env`). Khi đó **mọi** giao dịch đều không qua MQTT, nhưng **nhật ký webhook lại báo thành công** → khó phát hiện; kiosk vẫn chốt được nhờ polling 15 s (§5), chỉ chậm hơn.
Hướng sửa (để tham khảo, chưa làm): sau ⑧ kiểm `if not info.is_published(): return False`, hoặc dùng `on_connect` để biết mã CONNACK.

### 6.2 Broker treo (nhận TCP nhưng không trả lời) → chặn **60 s** rồi vẫn `True` (thí nghiệm 7)
Dùng `faulthandler` in ngăn xếp lúc đang kẹt → kẹt ở **`client.loop_stop()`** (`webhook_server.py` L61). Đọc mã nguồn paho 2.1.0 (`loop_forever`):
```python
if (self._thread_terminate is True
    and len(self._out_packet) == 0
        and len(self._out_messages) == 0):
    rc = MQTT_ERR_NOMEM; run = False      # chỉ thoát khi KHÔNG còn tin chờ gửi
```
Luồng nền chỉ chịu dừng khi **hàng đợi gửi trống**. Tin QoS 1 chưa được PUBACK nên **còn nằm trong hàng đợi** → luồng chạy tiếp cho tới khi **keepalive 60 s** hết hạn, paho coi kết nối chết, thoát → `loop_stop()` mới trả về → `return True`.
Trong 60 s đó, request HTTP của SePay **bị treo** (và chiếm một luồng của máy chủ). Tình huống này hiếm (broker công cộng quá tải, hoặc mạng “nửa chết”).

### 6.3 Vì sao mỗi lần gửi tốn đúng 1 giây (thí nghiệm đo từng bước)
`loop_stop()` mất 1,001 s dù tin đã đi xong: luồng nền đang **ngủ trong `select()` với thời hạn 1 s** chờ dữ liệu socket; cờ dừng chỉ được xét sau khi `select` hết hạn. Không phải lỗi — chỉ làm mỗi webhook chậm 1 s. (Gọi `disconnect()` **trước** `loop_stop()` sẽ đánh thức vòng lặp sớm hơn; ghi lại để biết.)

### 6.4 Bắt `Exception` chung
Mọi lỗi **ném ra** (TCP, DNS, TLS) cùng một hậu quả → `False` → 502 — hợp lý. Nhưng như 6.1–6.2 cho thấy, nhiều lỗi MQTT **không ném ra ngoại lệ** mà nằm trong trạng thái của client; bắt ngoại lệ không đủ để biết tin đã đi hay chưa.

---

## 7. Tổng kết

### 7.1 Những gì cần nhớ về paho‑mqtt từ file này
| Hàm | Chặn? | Ném lỗi khi | Ghi nhớ |
|---|---|---|---|
| `connect()` | Có (tới khi gửi xong CONNECT) | TCP/DNS/TLS lỗi | **Không** chờ CONNACK → sai mật khẩu không ném |
| `loop_start()` | Không | — | Bắt buộc phải có để gói tin thật sự đi |
| `publish()` | Không | Gần như không | Chỉ xếp hàng; trả `MQTTMessageInfo` |
| `wait_for_publish(t)` | Tới `t` giây | Không (khi hết giờ) | Phải hỏi thêm `is_published()` |
| `loop_stop()` | Tới khi luồng thoát | — | Chờ hàng đợi gửi trống → có thể kẹt tới keepalive |
| `disconnect()` | Không | — | Ngắt sạch, không phát will |

### 7.2 Nguyên tắc thiết kế thấy được
- **Tách vai**: webhook chỉ **chuyển tiếp**, không chạm CSDL; kiosk mới quyết định.
- **QoS 1 + bên nhận lũy đẳng** thay cho QoS 2.
- **Ký mọi tin** vì broker (công cộng) không đáng tin; mật khẩu broker là lớp bổ sung.
- **Không retain** tin sự kiện tức thời.
- **Hai đường xác nhận**: MQTT (nhanh) + polling SePay (chắc).
- **PUBACK ≠ người nhận đã nhận**: QoS 1 chỉ bảo đảm tới broker.

---

## 8. Câu hỏi tự kiểm tra

1. Kể 3 khác biệt giữa HTTP và MQTT. Vì sao kiosk không cần mở cổng nào mà vẫn nhận được tin từ webhook?
2. Trong nhật ký Mosquitto, `(p2, c1, k60)` và `(d0, q1, r0, m1)` nghĩa là gì?
3. `client.connect()` thành công có nghĩa là broker đã **chấp nhận** đăng nhập chưa? Thí nghiệm nào chứng minh?
4. Bỏ dòng `client.loop_start()` thì chuyện gì xảy ra với tin QoS 1?
5. Bỏ dòng `info.wait_for_publish(timeout=5)` thì chuyện gì có thể xảy ra?
6. Vì sao chọn QoS 1 mà không chọn QoS 0 hay QoS 2? Cơ chế nào ở kiosk khiến tin trùng trở nên vô hại?
7. Webhook nhận được PUBACK. Điều đó có bảo đảm kiosk đã chốt đơn không? Nếu kiosk đang mất mạng thì khách có bị mất đơn không — vì sao?
8. Vì sao tin `payment/success` không dùng `retain=True`?
9. Giải thích tại sao sai mật khẩu broker mà `send_mqtt_signal` vẫn trả `True`. SePay nhận mã HTTP nào, và hệ quả?
10. Vì sao broker “treo” làm `loop_stop()` chặn đúng ~60 s? Con số 60 đến từ đâu?
11. Mỗi webhook tốn ~1 s dù gửi thật chỉ 2 ms — 1 s đó nằm ở đâu?
12. Phân biệt vai trò của **mật khẩu broker** và **khóa HMAC**. Nếu chỉ có một trong hai thì kẻ tấn công làm được gì?
13. `try: mqtt.Client(mqtt.CallbackAPIVersion.VERSION2) except AttributeError: mqtt.Client()` — trên laptop này, mỗi nhánh chạy với Python nào?

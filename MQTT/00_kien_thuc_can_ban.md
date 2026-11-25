# Kiến thức và cú pháp căn bản để đọc các file MQTT

> **Dùng tài liệu này thế nào:** đọc **trước** các tài liệu trong `hoc_tap/MQTT/` (bắt đầu là `webhook_server_mqtt.md`; tiếp theo sẽ có phần MQTT của `bridge.py`, `api_server.py`, firmware bếp, ESP32 giọng nói, cấu hình Mosquitto).
> Cần biết trước: Python căn bản (`hoc_tap/SQLite/00_kien_thuc_can_ban.md`), IP/cổng/TCP (`hoc_tap/HTTP_Flask/00_kien_thuc_can_ban.md` phần A), HMAC (`hoc_tap/SQLite/mqtt_security.md`).
> **Mọi kết quả in ra đều đã chạy thật** trên một broker Mosquitto 2.0.11 **riêng** dựng trong thư mục nháp (cổng 18840), với paho‑mqtt 2.1.0. Không đụng Pi, broker công cộng hay Mosquitto hệ thống của laptop; broker thử đã tắt sau khi chạy.

---

## 0. Bản đồ: kiến thức nào dùng ở file nào

| Kiến thức | Mục | `webhook_server` | `bridge.py` | `api_server` | firmware bếp | ESP32 giọng nói | `mosquitto/` |
|---|---|:-:|:-:|:-:|:-:|:-:|:-:|
| Broker, client, topic, payload | A1–A3 | ● | ● | ● | ● | ● | ● |
| Wildcard `+` `#` | A4 | | | | | | ● |
| QoS 0/1/2 | B1 | ● | ● | ● | ● | ● | |
| Retain | B2 | ● | ● | ● | ● | | ● |
| Phiên sạch / bền vững, client id | B3 | ● | ● | ● | ● | ● | |
| Keepalive, Last Will | B4–B5 | ● | ● | ● | ● | ● | |
| Gói tin CONNECT…DISCONNECT | C | ● | ● | ● | ● | ● | |
| paho: client, callback, vòng lặp | D | ● | ● | ● | | | |
| PubSubClient (C++) | E | | | | ● | ● | |
| Đăng nhập, ACL, TLS, ký HMAC | F | ● | ● | ● | ● | ● | ● |

---

# PHẦN A — Mô hình phát – đăng ký

## A1. Ba vai
```
   PUBLISHER                    BROKER                      SUBSCRIBER
 (bên phát tin)   ──PUBLISH──▶  (bưu điện)  ──PUBLISH──▶   (bên đăng ký nhận)
                  topic + payload           tra danh sách ai đăng ký topic này
```
- **Broker**: máy chủ trung gian. Dự án dùng `broker.emqx.io` (công cộng) hoặc Mosquitto nội bộ (`deploy/mosquitto/`, đang hoãn).
- **Client**: mọi chương trình nối tới broker. **Một client vừa phát vừa nhận được** (kiosk nhận `payment/success`, phát `kitchen/new_order`).
- Bên phát **không biết** ai nhận, có bao nhiêu người nhận, có ai đang online không. Bên nhận cũng **không biết** ai phát. Hai bên **tách rời** (*decoupled*) qua broker.
- **Cả bên phát lẫn bên nhận đều chủ động nối ra** broker → không thiết bị nào phải mở cổng. Đây là lý do ESP32 và kiosk nhận được tin mà không cần làm máy chủ.

## A2. Topic
- Chuỗi UTF‑8, các tầng ngăn bởi `/`: `kitchen/new_order`, `payment/success`.
- **Phân biệt hoa thường**: thử phát `Kitchen/new_order` → người đăng ký `kitchen/+` **không nhận**.
- Không cần “tạo” topic trước — phát là có.
- Quy ước: không bắt đầu bằng `/`, không dấu cách. Topic bắt đầu bằng `$` (vd. `$SYS/...`) dành cho broker.

**Topic của dự án:**
| Topic | Phát | Nhận | Retain |
|---|---|---|---|
| `payment/success` | webhook | kiosk | không |
| `kitchen/new_order` | kiosk | màn hình bếp, ESP32 giọng nói | không |
| `kitchen/inventory` | kiosk, API | màn hình bếp | **có** |
| `kitchen/order_done` | API | màn hình bếp | không |

## A3. Payload
- **Bytes tùy ý**, broker **không đọc, không kiểm tra**. Dự án dùng JSON đã ký HMAC:
```json
{"v":2,"sig":"2dfa32fc…","payload":"{\"event\":\"payment.success\",\"order_code\":\"DH12345678\",\"amount\":45000,\"status\":\"PAID\",\"ts\":1795624177}"}
```
- Giới hạn của giao thức là 256 MB, nhưng thư viện trên thiết bị nhỏ giới hạn nhiều hơn (PubSubClient mặc định **256 byte**, xem E).

## A4. Ký tự đại diện (*wildcard*) — chỉ dùng khi **đăng ký**
| Ký tự | Khớp | Ví dụ |
|---|---|---|
| `+` | **đúng một** tầng | `kitchen/+` khớp `kitchen/new_order`, **không** khớp `kitchen/a/b` |
| `#` | **mọi** tầng còn lại (kể cả 0 tầng), phải đứng cuối | `payment/#` khớp `payment/success` **và** `payment` |

Thử thật — đăng ký `kitchen/+` và `payment/#`, rồi phát 5 topic:
```
kitchen/new_order   → nhận
kitchen/a/b         → KHÔNG (+ chỉ một tầng)
payment/success     → nhận
payment             → nhận  (# khớp cả "không còn tầng nào")
Kitchen/new_order   → KHÔNG (khác hoa thường)
```
Không được dùng wildcard khi **phát**.

---

# PHẦN B — Các tính năng giao nhận

## B1. QoS — mức bảo đảm giao tin
| QoS | Tên | Gói tin trao đổi | Bảo đảm | Rủi ro |
|---|---|---|---|---|
| 0 | *At most once* | PUBLISH | Gửi một lần, không ai xác nhận | **Có thể mất** |
| **1** | *At least once* | PUBLISH → PUBACK | Gửi lại tới khi có PUBACK | **Có thể trùng** |
| 2 | *Exactly once* | PUBLISH → PUBREC → PUBREL → PUBCOMP | Đúng một lần | Chậm, 4 gói |

**QoS tính theo từng chặng:**
- Chặng bên phát → broker: QoS do **bên phát** chọn.
- Chặng broker → bên nhận: lấy **QoS nhỏ hơn** giữa QoS của tin và QoS bên nhận đã đăng ký.

Thử: phát `qos=2`, bên nhận đăng ký `qos=0` → nhận được với `qos=0`.

**Dự án dùng QoS 1 + bên nhận xử lý lũy đẳng** (tin trùng không gây hại, vì CSDL chỉ đổi trạng thái khi điều kiện `WHERE` còn đúng) — mẫu phổ biến thay cho QoS 2.

⚠ PUBACK chỉ chứng minh **broker** đã nhận, **không** chứng minh bên nhận cuối đã nhận.

## B2. Retain — broker giữ lại tin cuối
- Phát với `retain=True` → broker **lưu tin cuối cùng** của topic đó; ai **đăng ký sau** sẽ nhận ngay tin này.
- Mỗi topic chỉ giữ **một** tin (tin mới ghi đè).
- **Xóa** tin đã giữ: phát **payload rỗng** với `retain=True`.

Thử thật:
```
phát "kho-v1" (retain) rồi "kho-v2" (retain)
B vào sau, đăng ký     → B nhận "kho-v2" với retain=True        (chỉ tin cuối)
phát "" (retain)       → xóa
B2 vào sau, đăng ký    → không nhận gì
```
Chú ý cờ `retain` phía nhận: **chỉ `True` khi tin được lấy từ kho lưu lúc vừa đăng ký**; người đang online nhận tin “trực tiếp” thì thấy `retain=False` (thử: A đang online nhận `kho-v1`, `kho-v2` với `retain=False`).

**Dùng khi nào?** Cho **trạng thái** (“kho hiện tại là…”) — màn hình bếp khởi động lại là có ngay bảng kho, không phải chờ. **Không** dùng cho **sự kiện** (“vừa thanh toán”, “đơn mới”) — nếu retain, mỗi lần khởi động lại sẽ nhận lại sự kiện cũ. Đó là lý do chỉ `kitchen/inventory` có retain.

## B3. Client id và phiên (*session*)
- Mỗi client có một **client id**. Để trống thì broker tự đặt (Mosquitto đặt `auto-1BFE072D-…`).
- **Hai client trùng id → broker đá client cũ.** Thử: X1 và X2 cùng id `kiosk-01` → X2 nối vào, X1 nhận `on_disconnect` ngay. (Nếu cả hai cùng tự nối lại thì sẽ đá nhau liên tục — lỗi kinh điển khi chép một firmware ra nhiều thiết bị với id cố định. Firmware bếp ghép id từ địa chỉ MAC chip: `"ESP32_Kitchen_" + ESP.getEfuseMac()` để tránh chuyện này.)

| | **Phiên sạch** (`clean_session=True`) | **Phiên bền vững** (`clean_session=False`) |
|---|---|---|
| Khi ngắt | Broker **quên** client, đăng ký, tin chờ | Broker **nhớ** đăng ký và **giữ hộ** tin QoS ≥ 1 |
| Khi nối lại | Phải đăng ký lại | `session_present=True`, nhận tin đã lỡ |
| Cần | | **client id cố định** |
| Dự án | Tất cả client đều dùng phiên sạch | — |

Hệ quả: client đang mất mạng lúc tin được phát thì **mất tin** (đã thử ở `webhook_server_mqtt.md`). Dự án bù bằng **đồng bộ lại qua HTTP / polling** (bếp hỏi API 30 s/lần, kiosk hỏi SePay 15 s/lần).
→ **Đăng ký nên đặt trong `on_connect`**: mỗi lần nối lại (phiên sạch) đều phải đăng ký lại.

## B4. Keepalive
- Khai lúc CONNECT (paho: tham số thứ 3 `connect(host, port, 60)`; PubSubClient mặc định **15 s**, ESP32 giọng nói đặt 30 s).
- Nếu trong khoảng đó không có gói nào, client tự gửi **PINGREQ**, broker đáp **PINGRESP**.
- Broker không nghe gì quá **1,5 × keepalive** → coi client đã chết, đóng kết nối (và phát Last Will).
- Client không nhận được PINGRESP → coi kết nối đã chết, nối lại.

## B5. Last Will — “di chúc”
Khai lúc CONNECT: “nếu tôi **chết đột ngột** (không gửi DISCONNECT), hãy phát tin này giúp tôi”.
```python
c.will_set("device/kds/status", "offline", qos=1, retain=True)
c.connect(H, P, keepalive=2)
```
Thử: đóng socket không gửi DISCONNECT (giả lập mất điện) → người đăng ký `device/#` nhận `device/kds/status = offline`.
- Ngắt **lịch sự** (`disconnect()`) thì **không** phát will.
- Dự án **chưa dùng** will (nhật ký: “No will message specified”). Đây là cách chuẩn để làm đèn “thiết bị online/offline” nếu sau này cần.

---

# PHẦN C — Gói tin MQTT (đọc nhật ký broker)

| Gói | Hướng | Ý nghĩa |
|---|---|---|
| **CONNECT** | client → broker | Xin nối: client id, keepalive, clean session, tài khoản, will |
| **CONNACK** | broker → client | Trả lời: mã 0 = chấp nhận; **5 = không có quyền** (sai tài khoản) |
| **PUBLISH** | hai chiều | Một tin: topic, payload, QoS, retain, dup, message id |
| **PUBACK** | | Xác nhận PUBLISH QoS 1 |
| PUBREC / PUBREL / PUBCOMP | | Bắt tay 4 bước của QoS 2 |
| **SUBSCRIBE** / **SUBACK** | | Đăng ký topic / broker xác nhận (kèm QoS được cấp) |
| UNSUBSCRIBE / UNSUBACK | | Hủy đăng ký |
| **PINGREQ** / **PINGRESP** | | Giữ kết nối sống |
| **DISCONNECT** | client → broker | Ngắt lịch sự |

Ký hiệu trong nhật ký Mosquitto (`log_type all`):
```
New client connected from 127.0.0.1:40631 as auto-1BFE…  (p2, c1, k60, u'webhook')
                                                          │   │   │    └ tên đăng nhập
                                                          │   │   └ keepalive 60 s
                                                          │   └ clean session = 1
                                                          └ giao thức MQTT 3.1.1
Received PUBLISH from … (d0, q1, r0, m1, 'payment/success', ... (209 bytes))
                         │   │   │   └ message id
                         │   │   └ retain = 0
                         │   └ QoS 1
                         └ dup = 0 (không phải gửi lại)
```

---

# PHẦN D — paho‑mqtt (Python): kiosk, API, webhook

## D1. Tạo client và cấu hình
```python
import paho.mqtt.client as mqtt

client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)   # paho 2.x bắt buộc chọn API callback
# client = mqtt.Client(client_id="kiosk-01", clean_session=False)   # nếu muốn phiên bền vững
client.username_pw_set("kiosk", "mat-khau")              # (qua configure_client của dự án)
client.tls_set(ca_certs="/etc/mosquitto/ca.crt")         # nếu MQTT_TLS=true
client.will_set("topic", "offline", qos=1, retain=True)  # nếu cần (dự án không dùng)
```
Dự án viết `try: mqtt.Client(mqtt.CallbackAPIVersion.VERSION2) except AttributeError: mqtt.Client()` để chạy được cả paho 1.x.

## D2. Callback — hàm được paho **gọi lại** khi có sự kiện
```python
def on_connect(client, userdata, flags, reason_code, properties):
    if reason_code == 0:                       # nối thành công
        client.subscribe("payment/success", qos=1)    # đăng ký TRONG on_connect

def on_message(client, userdata, msg):
    msg.topic        # "payment/success"
    msg.payload      # bytes → msg.payload.decode("utf-8")
    msg.qos, msg.retain

def on_disconnect(client, userdata, flags, reason_code, properties):
    ...

client.on_connect = on_connect
client.on_message = on_message
client.on_disconnect = on_disconnect
```
- Chữ ký trên là của **API callback VERSION2** (paho 2.x); VERSION1 ít tham số hơn.
- ⚠ Callback chạy trên **luồng mạng của paho**, **không phải** luồng giao diện. Trong `bridge.py` (Qt), không được sửa giao diện trực tiếp trong `on_message` — phải chuyển sang luồng Qt bằng **signal** (học ở `bridge.py`).
- Đã thử: `on_connect rc=Success session_present=False`; bị đá vì trùng id → `on_disconnect rc=Unspecified error`.

## D3. Nối và vòng lặp mạng
```python
client.connect(host, port, 60)          # CHẶN tới khi gửi xong CONNECT (không chờ CONNACK)
client.connect_async(host, port, 60)    # KHÔNG chặn; nối trong vòng lặp (kiosk dùng cách này)
client.loop_start()                     # tạo luồng nền xử lý mạng + tự nối lại
...
client.loop_stop()                      # dừng luồng nền (chờ nó thoát)
client.disconnect()
```
| Vòng lặp | Chặn? | Dùng khi |
|---|---|---|
| `loop_forever()` | Có | Chương trình chỉ làm MQTT |
| `loop_start()` / `loop_stop()` | Không (luồng nền) | Có việc khác: Flask, Qt — **dự án dùng cách này** |
| `loop(timeout)` | Gọi tay từng nhịp | Tự quản vòng lặp |

**Không có vòng lặp thì không gói nào thật sự được gửi/nhận.**
- `connect()` ném lỗi ngay nếu TCP hỏng (cổng đóng: 0 s; IP không tồn tại: 5 s), nhưng **sai mật khẩu thì không ném** — chỉ biết qua `on_connect` (mã khác 0).
- `connect_async()` + `loop_start()`: ứng dụng khởi động được **cả khi broker chưa sẵn sàng**; paho tự thử nối lại.

## D4. Phát tin
```python
info = client.publish("kitchen/new_order", payload_text, qos=1, retain=False)
info.mid                      # message id
info.is_published()           # đã nhận PUBACK chưa?
info.wait_for_publish(5)      # chờ tối đa 5 s (hết giờ: lặng lẽ trả về, không ném lỗi)
```
Thử thật:
```
publish trả về sau 0.29 ms, mid=10, is_published ngay = False
sau wait_for_publish: True
```
→ `publish()` **không chặn**: chỉ xếp tin vào hàng đợi; luồng nền gửi sau. Muốn chắc đã tới broker thì `wait_for_publish` **rồi kiểm** `is_published()`.

## D5. Đăng ký
```python
client.subscribe("payment/success", qos=1)
client.subscribe([("kitchen/+", 1), ("payment/#", 1)])     # nhiều topic một lần
client.unsubscribe("payment/success")
```

---

# PHẦN E — PubSubClient (C++, ESP32): firmware bếp và ESP32 giọng nói

```cpp
#include <WiFi.h>
#include <PubSubClient.h>

WiFiClient espClient;                    // kênh TCP
PubSubClient mqttClient(espClient);      // MQTT chạy trên kênh đó

mqttClient.setServer(MQTT_BROKER_HOST, MQTT_BROKER_PORT);
mqttClient.setCallback(mqttCallback);    // hàm nhận tin
mqttClient.setBufferSize(8192);          // mặc định chỉ 256 byte!
// mqttClient.setKeepAlive(30);          // mặc định 15 s

String clientId = "ESP32_Kitchen_" + String((uint32_t)ESP.getEfuseMac(), HEX);   // id duy nhất theo chip
if (mqttClient.connect(clientId.c_str(), MQTT_USERNAME, MQTT_PASSWORD)) {
  mqttClient.subscribe("kitchen/new_order");          // QoS 0 mặc định; subscribe(topic, 1) cho QoS 1
}

void loop() {
  if (!mqttClient.connected()) { /* thử nối lại sau 3 s */ }
  mqttClient.loop();                     // PHẢI gọi thường xuyên: nhận tin, gửi PINGREQ
}

void mqttCallback(char *topic, byte *payload, unsigned int length) {
  // payload KHÔNG có ký tự kết thúc '\0' → luôn dùng 'length'
}
```
| Đặc điểm PubSubClient 2.8 | Ghi nhớ |
|---|---|
| Phiên bản giao thức | MQTT 3.1.1 |
| `MQTT_MAX_PACKET_SIZE` mặc định | **256 byte** → tin lớn hơn bị **bỏ âm thầm** → phải `setBufferSize` (bếp: 8192, vì tin đơn/kho đã ký khá dài) |
| Keepalive mặc định | 15 s (`MQTT_KEEPALIVE`) |
| Phát | Chỉ **QoS 0** |
| Đăng ký | QoS 0 hoặc 1 |
| `connect()` | **Chặn** và chờ CONNACK → trả `true/false`; `state()` cho biết lý do (vd. 5 = không có quyền) |
| Không tự nối lại | Code phải tự kiểm `connected()` và gọi lại `connect()` |
| Không có luồng nền | Phải gọi `loop()` trong vòng lặp; không gọi thì mất kết nối sau keepalive |

---

# PHẦN F — Bảo mật MQTT trong dự án

| Lớp | Chống gì | Ở đâu |
|---|---|---|
| **Ký HMAC mọi tin** (`{"v":2,"sig","payload"}` + `ts`) | Tin giả, tin bị sửa, phát lại tin cũ (> 120 s) | `mqtt_security.py`, `verifySignedMessage` trong firmware |
| **Đăng nhập broker**, mỗi dịch vụ một tài khoản | Người lạ nối vào broker | `configure_client(role=...)`, `mosquitto.conf` (`allow_anonymous false`) |
| **ACL** (quyền theo topic) | Dịch vụ bị chiếm quyền phát lung tung | `deploy/mosquitto/acl` |
| **TLS** (cổng 8883, `MQTT_TLS=true`) | Nghe lén, sửa trên đường truyền | tùy chọn, chưa bật |
| Không mở cổng 1883 ra Internet | Tấn công từ ngoài | `mosquitto.conf` |

**Vì sao vẫn phải ký khi đã có mật khẩu?** Broker công cộng `broker.emqx.io` **không có mật khẩu**: ai biết tên topic cũng phát được `payment/success` giả. Chữ ký HMAC là lớp **không phụ thuộc broker** — đổi broker nào cũng vẫn an toàn.

**Đọc file ACL** (`deploy/mosquitto/acl`):
```
user webhook
topic write payment/success          # webhook chỉ được PHÁT tín hiệu thanh toán
user kiosk
topic read payment/success
topic write kitchen/new_order
topic write kitchen/inventory
user api
topic write kitchen/inventory
user kds
topic read kitchen/new_order
topic read kitchen/inventory
```
Nguyên tắc **quyền tối thiểu**: tài khoản không có trong file thì không đọc/ghi được topic nào.
> 📌 Ghi chú (chưa sửa, vì Mosquitto nội bộ đang hoãn): file ACL **chưa có** `kitchen/order_done` — thiếu `topic write kitchen/order_done` cho `api` và `topic read kitchen/order_done` cho `kds`. Khi nào triển khai Mosquitto theo file này thì tin “đơn đã xong ở nơi khác” sẽ bị chặn; màn hình bếp vẫn gỡ thẻ nhờ đồng bộ HTTP 30 s. ESP32 giọng nói cũng chưa có tài khoản trong ACL (hiện cấu hình cứng `broker.emqx.io`).

---

# PHẦN G — MQTT hay HTTP?

| Câu hỏi | HTTP | MQTT |
|---|---|---|
| Cần câu trả lời ngay cho **chính yêu cầu này**? | ✔ | |
| Một nguồn → **nhiều** nơi nhận? | | ✔ |
| Bên nhận **không mở cổng** được (thiết bị sau NAT, ứng dụng giao diện)? | | ✔ |
| Bên nhận có thể **offline**, cần lấy lại toàn bộ trạng thái? | ✔ (GET ảnh chụp) | (retain chỉ giữ tin cuối) |
| Đẩy tức thời, tiết kiệm (thiết bị nhỏ)? | | ✔ |

Dự án kết hợp: **MQTT để đẩy nhanh** sự kiện, **HTTP để hỏi lại trạng thái đầy đủ và để ghi có xác nhận** (báo đơn xong).

---

# PHẦN H — Bảng tra nhanh và tự kiểm tra

## H1. Thấy cái này trong code → nghĩa là gì
| Thấy | Nghĩa |
|---|---|
| `mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)` | Client paho 2.x, callback kiểu mới |
| `configure_client(client, role="KIOSK")` | Gắn tài khoản `MQTT_KIOSK_*` + TLS nếu bật |
| `connect_async(...)` + `loop_start()` | Nối không chặn, luồng nền tự nối lại |
| `client.subscribe(...)` trong `on_connect` | Đăng ký lại mỗi lần nối (phiên sạch) |
| `publish(..., qos=1, retain=True)` | Tin trạng thái, người vào sau vẫn nhận |
| `publish(..., qos=1)` (không retain) | Tin sự kiện tức thời |
| `info.wait_for_publish(5)` | Chờ PUBACK tối đa 5 s (không ném lỗi khi hết giờ) |
| `sign_message(...)` / `verify_message(...)` | Ký / kiểm chữ ký + độ mới |
| `mqttClient.setBufferSize(8192)` | Nới giới hạn 256 byte của PubSubClient |
| `mqttClient.loop()` | Nhịp xử lý mạng — phải gọi đều |
| `"ESP32_Kitchen_" + getEfuseMac()` | Client id duy nhất theo chip |
| CONNACK `(0, 5)` trong nhật ký | Sai tài khoản / không có quyền |

## H2. Thứ tự đọc đề xuất
Tài liệu này → `hoc_tap/SQLite/mqtt_security.md` (ôn) → `webhook_server_mqtt.md` → phần MQTT của `bridge.py` → `api_server.py` → firmware bếp → ESP32 giọng nói → cấu hình Mosquitto + `thu_gia_mao_mqtt.sh`.

## H3. Câu hỏi tự kiểm tra
1. Vì sao trong MQTT không thiết bị nào phải mở cổng mà vẫn nhận được tin?
2. `kitchen/+` có khớp `kitchen/a/b` không? `payment/#` có khớp `payment` không?
3. Phát QoS 2, bên nhận đăng ký QoS 0 — bên nhận nhận với QoS mấy?
4. Vì sao dự án chọn QoS 1 thay vì QoS 2? Điều gì ở bên nhận khiến tin trùng vô hại?
5. Vì sao `kitchen/inventory` dùng retain còn `payment/success` thì không? Xóa tin retained thế nào?
6. Người nhận đang online thấy cờ `retain` của tin retained là gì? Người vào sau thì sao?
7. Hai thiết bị cùng client id thì chuyện gì xảy ra? Firmware bếp tránh bằng cách nào?
8. Phiên sạch và phiên bền vững khác nhau thế nào? Vì sao phải đăng ký trong `on_connect`?
9. Keepalive 60 s: broker sau bao lâu mới coi client đã chết?
10. Last Will được phát khi nào và **không** được phát khi nào?
11. `client.connect()` thành công có nghĩa là đăng nhập đúng chưa? Làm sao biết chắc?
12. `publish()` trả về ngay — tin đã tới broker chưa? Kiểm tra thế nào?
13. Vì sao không được cập nhật giao diện Qt trực tiếp trong `on_message`?
14. Tin đơn mới dài 1 KB gửi tới ESP32 dùng PubSubClient mặc định thì chuyện gì xảy ra?
15. Đã có mật khẩu broker và ACL, vì sao vẫn cần ký HMAC?

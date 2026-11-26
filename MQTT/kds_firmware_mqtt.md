# Giải thích chi tiết phần MQTT trong `kds_firmware.ino` (màn hình bếp)

> File: `kds_firmware.ino/kds_firmware/kds_firmware.ino` — 1854 dòng, ESP32‑S3 (bo JC4827W543C). Phần MQTT khoảng 150 dòng: **L3, L11, L29–32, L77, L84, L145–146, L310, L1474–1599, L1631–1634, L1678–1691**.
> Phần HTTP của file đã học ở `hoc_tap/HTTP_Flask/kds_firmware_http.md`. Học trước: `00_kien_thuc_can_ban.md` (phần E — PubSubClient), `bridge_mqtt.md`, `api_server_mqtt.md` (bên **phát** các tin mà màn hình bếp nhận).
>
> **Thí nghiệm — chạy mã thật mà không cần mạch:** biên dịch **chính mã nguồn PubSubClient 2.8** (trong `~/Arduino/libraries`) + **ArduinoJson 7.4.3** + các hàm `computeHmacHex`, `constantTimeEquals`, `verifySignedMessage` và phần phân loại topic của `mqttCallback` **chép từ firmware** thành một chương trình chạy trên laptop. Chỉ thay 2 chỗ phụ thuộc phần cứng:
> - `WiFiClient` của ESP32 → một lớp `PosixClient` dùng socket TCP của Linux (cùng giao diện `Client` của Arduino);
> - `mbedtls_md_hmac` → `HMAC()` của OpenSSL (cùng thuật toán HMAC‑SHA256).
>
> Broker Mosquitto **riêng** (cổng 18860 không đăng nhập, 18861 bắt đăng nhập user `kds`) trong thư mục nháp; tin được phát bằng **đúng** `sign_message` của dự án, bảng kho lấy từ **bản sao** CSDL (62 nguyên liệu). Không đụng Pi, mạch, `secrets.h` hay Mosquitto hệ thống; broker thử đã tắt sau khi chạy. Ngoài ra firmware đã được biên dịch thật cho ESP32‑S3 ở bài HTTP (exit 0).

---

## 0. Bức tranh tổng thể (đọc trước)

### 0.1 Màn hình bếp chỉ **nhận** MQTT
| Topic | Bên phát | Nhận để làm gì |
|---|---|---|
| `kitchen/new_order` | Kiosk (QoS 1) | Hiện **thẻ đơn mới** ngay |
| `kitchen/inventory` | Kiosk, API (retain) | Cập nhật **bảng kho** |
| `kitchen/order_done` | API | **Gỡ thẻ** đơn đã xong ở nơi khác (giọng nói, quản trị) |

Chiều ngược lại (bếp báo “đơn xong”) đi bằng **HTTP POST** vì cần biết chắc kết quả (đã học ở bài HTTP).
Mọi thứ MQTT nhận được chỉ là **đường nhanh**; đường chắc là **HTTP GET mỗi 30 s** (`/api/orders/pending`, `/api/inventory`).

### 0.2 Kết quả chạy thử

**A. Bộ đệm MẶC ĐỊNH 256 byte (nếu quên `setBufferSize`)** — broker gửi đủ 6 tin, nhưng callback chỉ được gọi cho tin **≤ 256 byte**:
| Tin | Kích thước | Kết quả |
|---|---|---|
| Kho (retained) | 5497 B | **Không gọi callback** — bỏ âm thầm |
| Đơn mới (ký đúng) | 361 B | **Không gọi callback** — bỏ âm thầm |
| Đơn xong | 179 B | Nhận, kiểm chữ ký đúng → gỡ thẻ |
| Đơn mới **ký sai khóa** | 361 B | Bỏ âm thầm (chưa tới bước kiểm chữ ký) |
| Đơn mới **không ký** (JSON trần) | 233 B | Nhận → `Unsigned or unsupported message` |

**B. `setBufferSize(8192)` như firmware:**
```
[   45 ms] kitchen/inventory  length=5497 → KHO: 62 nguyên liệu          (tin retained, nhận ngay khi đăng ký)
[ 1243 ms] kitchen/new_order  length=361  → ĐƠN MỚI DH71128480, Cà phê đen đá x2
[ 1645 ms] kitchen/order_done length=179  → ĐƠN XONG ở nơi khác: DH71128480 (gỡ thẻ)
[ 2047 ms] kitchen/new_order  length=361  → [MQTT] Bad signature        (ký bằng khóa sai)
[ 2448 ms] kitchen/new_order  length=233  → [MQTT] Unsigned or unsupported message
[ 2849 ms] kitchen/new_order  length=361  → ĐƠN MỚI DH00000001 ...      (ký đúng nhưng ts cũ 24 GIỜ — vẫn nhận!)
[ 5410 ms] kitchen/inventory  length=0    → [MQTT] Invalid envelope     (lệnh xóa tin retained, vô hại)
```

**C. Nhật ký broker** — PubSubClient đăng ký **QoS 0**:
```
Received SUBSCRIBE from ESP32_Kitchen_thu   kitchen/new_order (QoS 0)
Sending PUBLISH to ESP32_Kitchen_thu (d0, q0, r1, m0, 'kitchen/inventory', ... (5497 bytes))
Sending PUBLISH to ESP32_Kitchen_thu (d0, q0, r0, m0, 'kitchen/new_order', ... (361 bytes))
```
→ Tin kiosk phát QoS 1 nhưng chặng broker → bếp chỉ là **QoS 0** (lấy mức nhỏ hơn).

**D. Đăng nhập và mất kết nối:**
| Tình huống | `connect()` | `state()` |
|---|---|---|
| Đúng tài khoản `kds` | `true` | 0 `MQTT_CONNECTED` |
| **Sai** mật khẩu | `false` | **5** `MQTT_CONNECT_UNAUTHORIZED` |
| Không đăng nhập (broker bắt buộc) | `false` | **5** |
| Broker tắt | `false` | **−2** `MQTT_CONNECT_FAILED` |
| Broker chết khi đang chạy | `connected()` → `false` | **−3** `MQTT_CONNECTION_LOST` |

---

## 1. Khai báo (L3, L11, L29–32, L77, L84, L145–146)

```cpp
#include <PubSubClient.h>          // L3
#include "mbedtls/md.h"            // L11  — HMAC-SHA256 có sẵn trong ESP-IDF
const char *MQTT_TOPIC_ORDER      = "kitchen/new_order";     // L29
const char *MQTT_TOPIC_INV        = "kitchen/inventory";     // L30
const char *MQTT_TOPIC_ORDER_DONE = "kitchen/order_done";    // L32
constexpr int ORDER_EVENT_QUEUE_LEN = 3;                     // L77
constexpr uint32_t MQTT_RECONNECT_MS = 3000;                 // L84
WiFiClient espClient;                                        // L145
PubSubClient mqttClient(espClient);                          // L146
```
- **PubSubClient chỉ lo giao thức MQTT**; nó gửi/nhận byte qua một đối tượng `Client` được truyền vào. `WiFiClient` là kênh TCP qua Wi‑Fi. Chính nhờ thiết kế “tách lớp” này mà thí nghiệm thay được `WiFiClient` bằng `PosixClient` mà **không sửa một dòng** PubSubClient.
- Cả hai là **biến toàn cục** (sống suốt chương trình) — trên vi điều khiển thường tránh cấp phát động lúc chạy.
- `secrets.h` (không lên git) cung cấp `MQTT_BROKER_HOST`, `MQTT_BROKER_PORT`, `MQTT_USERNAME`, `MQTT_PASSWORD`, **`MQTT_SECRET`** — khóa HMAC phải **trùng** `MQTT_SHARED_SECRET` trong `.env` của Smart Kiosk (theo `secrets.example.h`).

## 2. Cấu hình lúc khởi động task mạng (L1678–1680)
```cpp
mqttClient.setServer(MQTT_BROKER_HOST, MQTT_BROKER_PORT);
mqttClient.setCallback(mqttCallback);
mqttClient.setBufferSize(8192);
```
| Dòng | Ý nghĩa |
|---|---|
| `setServer` | Chỉ ghi nhớ địa chỉ; chưa nối |
| `setCallback(mqttCallback)` | Hàm được gọi khi có tin. Chữ ký bắt buộc: `void f(char *topic, byte *payload, unsigned int length)` |
| **`setBufferSize(8192)`** | Bộ đệm **cả gói tin** (header + topic + payload). Mặc định chỉ **256 byte** (`MQTT_MAX_PACKET_SIZE`) |

### 2.1 Vì sao `setBufferSize(8192)` là dòng sống còn
Thí nghiệm A: không có dòng này thì **tin đơn mới (361 B) và tin kho (5497 B) đều bị bỏ âm thầm** — không lỗi, không log, callback không được gọi. Màn hình bếp sẽ trông như “MQTT không hoạt động”, chỉ cập nhật theo nhịp HTTP 30 s.
Đọc mã PubSubClient (`readPacket`): gói dài hơn bộ đệm vẫn được **đọc hết khỏi socket** (để không lệch luồng byte) nhưng **trả độ dài 0** → `loop()` coi như không có tin.
Chú ý thêm: tin **giả mạo** cỡ 361 B cũng bị bỏ ở bước này, **trước** khi tới bước kiểm chữ ký — nên ở thí nghiệm A ta không thấy dòng `Bad signature`.

8192 B đủ cho: đơn mới (~361 B/đơn 1 món), kho (~89 B/nguyên liệu → khoảng 90 nguyên liệu). Bộ đệm cấp trên heap; ESP32‑S3 có PSRAM nên 8 KB không đáng kể.

---

## 3. Giữ kết nối — `maintainMqtt()` (L1582–1599)

```cpp
static void maintainMqtt() {
  static uint32_t lastAttempt = 0;
  if (WiFi.status() != WL_CONNECTED || mqttClient.connected()) return;
  if (millis() - lastAttempt < MQTT_RECONNECT_MS) return;
  lastAttempt = millis();
  String clientId = "ESP32_Kitchen_" + String((uint32_t)ESP.getEfuseMac(), HEX);
  bool connected = (MQTT_USERNAME && MQTT_USERNAME[0])
    ? mqttClient.connect(clientId.c_str(), MQTT_USERNAME, MQTT_PASSWORD)
    : mqttClient.connect(clientId.c_str());
  if (connected) {
    bool orderSub = mqttClient.subscribe(MQTT_TOPIC_ORDER);
    bool invSub   = mqttClient.subscribe(MQTT_TOPIC_INV);
    bool doneSub  = mqttClient.subscribe(MQTT_TOPIC_ORDER_DONE);
    Serial.printf("[MQTT] Subscribe order=%d inventory=%d done=%d\n", orderSub, invSub, doneSub);
  } else {
    Serial.printf("[MQTT] Connect failed state=%d\n", mqttClient.state());
  }
}
```

### 3.1 Cú pháp
- `static uint32_t lastAttempt = 0;` — biến `static` **trong hàm**: khởi tạo **một lần**, giữ giá trị giữa các lần gọi (giống biến toàn cục nhưng chỉ hàm này thấy).
- `millis() - lastAttempt < MQTT_RECONNECT_MS` — trừ rồi so → đúng cả khi `millis()` tràn sau ~49,7 ngày (đã học ở bài HTTP).
- `String((uint32_t)ESP.getEfuseMac(), HEX)` — `getEfuseMac()` trả địa chỉ MAC 48 bit **khắc cứng trong chip**; lấy 32 bit thấp, in hệ 16 → mỗi chip một **client id khác nhau** (vd. `ESP32_Kitchen_a1b2c3d4`).
- `cond ? A : B` — toán tử ba ngôi: có tên đăng nhập thì `connect(id, user, pass)`, không thì `connect(id)`.

### 3.2 Logic
1. **Không có Wi‑Fi** hoặc **đang nối rồi** → thôi.
2. Lần thử trước **chưa quá 3 s** → thôi (giới hạn nhịp thử lại, khỏi dội broker).
3. `connect()` — **chặn**: nối TCP (Wi‑Fi client của ESP32 chờ tối đa **3000 ms** theo mặc định trong `NetworkClient.cpp`), gửi CONNECT, **chờ CONNACK** tối đa `MQTT_SOCKET_TIMEOUT` = **15 s**. Khác paho: PubSubClient **biết ngay** đăng nhập đúng hay sai (thí nghiệm D: sai mật khẩu → `false`, `state = 5`).
4. Nối được → **đăng ký lại cả 3 topic** — cùng lý do kiosk đăng ký trong `on_connect`: PubSubClient dùng **phiên sạch** (`cleanSession = true` mặc định) nên mỗi lần nối lại broker quên đăng ký cũ.
5. `subscribe(topic)` không ghi QoS → **QoS 0** (thí nghiệm C). PubSubClient chỉ hỗ trợ đăng ký QoS 0 hoặc 1.

### 3.3 Được gọi ở đâu (task mạng, L1689–1691)
```cpp
for (;;) {
  maintainWiFi();
  maintainMqtt();
  if (mqttClient.connected()) mqttClient.loop();
  ...                                  // đồng bộ HTTP, hàng đợi báo xong, cập nhật tiêu đề
  vTaskDelay(pdMS_TO_TICKS(5));
}
```
- **PubSubClient không có luồng nền** (khác paho `loop_start`): phải gọi `loop()` **đều đặn** — mỗi lần gọi, nó đọc gói đến (gọi callback), và gửi **PINGREQ** khi tới hạn keepalive.
- Keepalive mặc định **15 s** (`MQTT_KEEPALIVE`; firmware không đổi). Nếu task mạng bị **chặn** lâu hơn ~1,5 × 15 s (vd. HTTP chờ timeout nhiều lần liên tiếp), broker sẽ coi màn hình bếp đã chết và ngắt → vòng sau `maintainMqtt` tự nối lại.
- Toàn bộ việc mạng chạy ở **lõi 0**; giao diện LVGL ở **lõi 1** → `connect()` chặn tới 3 + 15 s cũng không làm đứng màn hình.

### 3.4 Đèn trạng thái trên màn hình (L307–312)
```cpp
bool mqttOk = mqttClient.connected();
lv_label_set_text(lblConnection, mqttOk ? "ONLINE" : (wifiOk ? "MQTT..." : "OFFLINE"));
```
`ONLINE` (xanh) = đã nối broker; `MQTT...` (cam) = có Wi‑Fi nhưng chưa nối được broker (sai tài khoản, broker sập…); `OFFLINE` (đỏ) = mất Wi‑Fi. Mất broker (thí nghiệm D, `state = −3`) → đèn chuyển `MQTT...`.

---

## 4. Kiểm chữ ký trên vi điều khiển (L1474–1531)

### 4.1 `computeHmacHex()`
```cpp
static bool computeHmacHex(const char *message, size_t length, char out[65]) {
  uint8_t digest[32];
  const mbedtls_md_info_t *info = mbedtls_md_info_from_type(MBEDTLS_MD_SHA256);
  if (!info) return false;
  if (mbedtls_md_hmac(info, (const unsigned char *)MQTT_SECRET, strlen(MQTT_SECRET),
                      (const unsigned char *)message, length, digest) != 0) return false;
  static const char hex[] = "0123456789abcdef";
  for (int i = 0; i < 32; ++i) {
    out[i * 2]     = hex[digest[i] >> 4];     // nửa cao của byte
    out[i * 2 + 1] = hex[digest[i] & 0x0F];   // nửa thấp
  }
  out[64] = '\0';
  return true;
}
```
- **mbedTLS** là thư viện mật mã có sẵn trong ESP‑IDF (cũng dùng cho TLS) — không cần thêm thư viện.
- HMAC‑SHA256 cho **32 byte**; đổi sang **64 ký tự hex thường** để so với `sig` do Python tạo (`hexdigest()` cũng là hex thường).
- `digest[i] >> 4` / `digest[i] & 0x0F` — tách một byte thành hai chữ số hex (vd. `0xA7` → `'a'`, `'7'`).
- `reinterpret_cast<const unsigned char *>(...)` (trong mã gốc) — ép kiểu con trỏ `char*` sang `unsigned char*` mà mbedTLS yêu cầu.
- Mảng `char out[65]` = 64 ký tự + `'\0'` kết thúc chuỗi C.

### 4.2 `constantTimeEquals()`
```cpp
static bool constantTimeEquals(const char *a, const char *b, size_t length) {
  uint8_t diff = 0;
  for (size_t i = 0; i < length; ++i) diff |= (uint8_t)(a[i] ^ b[i]);
  return diff == 0;
}
```
`a[i] ^ b[i]` = 0 khi hai ký tự giống nhau; `|=` gom mọi khác biệt. Luôn chạy **đủ 64 vòng** dù sai ngay ký tự đầu → thời gian so sánh không tiết lộ “đúng được mấy ký tự đầu”. Tương đương `hmac.compare_digest` của Python. (`strcmp` dừng ở chỗ khác đầu tiên → không dùng.)

### 4.3 `verifySignedMessage()` — 5 cửa kiểm tra
```cpp
static bool verifySignedMessage(byte *payload, unsigned int length, JsonDocument &doc) {
  if (!MQTT_SECRET || !MQTT_SECRET[0])                             { … "MQTT_SECRET empty: reject"; return false; }   // ①
  JsonDocument envelope;
  if (deserializeJson(envelope, payload, length))                  { … "Invalid envelope"; return false; }            // ②
  if (envelope["v"].as<int>() != 2 || !envelope["payload"].is<const char *>() ||
      !envelope["sig"].is<const char *>())                          { … "Unsigned or unsupported message"; return false; } // ③
  const char *inner = envelope["payload"].as<const char *>();
  const char *signature = envelope["sig"].as<const char *>();
  char expected[65];
  if (strlen(signature) != 64 || !computeHmacHex(inner, strlen(inner), expected) ||
      !constantTimeEquals(expected, signature, 64))                { … "Bad signature"; return false; }               // ④
  if (deserializeJson(doc, inner))                                 { … "Invalid payload JSON"; return false; }        // ⑤
  return true;
}
```
| Cửa | Chặn | Thí nghiệm B |
|---|---|---|
| ① Chưa cấu hình khóa | Fail‑closed: không có khóa thì **không tin gì cả** | |
| ② Không phải JSON | Rác, payload rỗng | Lệnh xóa retained (0 B) → `Invalid envelope` |
| ③ Không đúng vỏ v2 | **Tin không ký** | JSON trần 233 B → `Unsigned or unsupported message` |
| ④ Chữ ký sai | **Tin giả / bị sửa** | Ký bằng khóa sai → `Bad signature` |
| ⑤ Nội dung không phải JSON | | |

Chi tiết đáng chú ý:
- `deserializeJson(envelope, payload, length)` — truyền **độ dài**, vì `payload` của PubSubClient **không có `'\0'` kết thúc**.
- HMAC tính trên **đúng chuỗi `inner`** nhận được (không phải JSON dựng lại) → không cần sắp xếp khóa JSON, khớp từng byte với Python (`separators=(",", ":")`, `ensure_ascii=False`). Đây chính là lý do thiết kế vỏ `{"v":2,"sig","payload":"<chuỗi>"}` trong `mqtt_security.py`.
- `strlen(signature) != 64` kiểm **trước** để `constantTimeEquals` không đọc quá mảng.
- Kiểm chữ ký **trước** khi phân tích nội dung → tin giả không bao giờ chạm tới logic tạo/gỡ thẻ.

### 4.4 ⚠ Không kiểm `ts` (độ mới)
Python (`verify_message`) từ chối tin cũ hơn 120 s; **firmware thì không**. Thí nghiệm B: tin đơn mới ký đúng nhưng `ts` lùi **24 giờ** vẫn được nhận → tạo thẻ `DH00000001`.
- **Vì sao firmware bỏ qua `ts`?** Tin **kho là retained**: màn hình bếp khởi động lại có thể nhận một bản kho phát từ hàng giờ trước — vẫn hợp lệ. Nếu kiểm 120 s thì bản kho này bị loại. Hơn nữa ESP32 không có đồng hồ thời gian thực; muốn so `ts` phải đồng bộ NTP.
- **Rủi ro:** kẻ nghe lén (broker công cộng, không TLS) có thể **phát lại** một tin `new_order` hay `order_done` cũ đã ký → hiện thẻ “ma” hoặc gỡ nhầm thẻ.
- **Giới hạn thiệt hại:** ≤ 30 s — lần đồng bộ HTTP tiếp theo sẽ gỡ thẻ ma (đơn không có trong `/api/orders/pending`) hoặc vẽ lại thẻ bị gỡ nhầm. Không ảnh hưởng tiền hay CSDL (bếp không ghi gì dựa trên MQTT).
- (Ghi chú, chưa sửa.) Nếu muốn chặt hơn: chỉ áp kiểm `ts` cho `new_order`/`order_done` (không áp cho kho), sau khi ESP32 đồng bộ NTP.

---

## 5. `mqttCallback()` — phân loại và xử lý (L1533–1563)

```cpp
static void mqttCallback(char *topic, byte *payload, unsigned int length) {
  JsonDocument doc;
  if (!verifySignedMessage(payload, length, doc)) return;

  if (strcmp(topic, MQTT_TOPIC_ORDER) == 0) {
    if (!doc.is<JsonObjectConst>()) return;
    KdsOrderEvent event{};
    if (!parseOrderObject(doc.as<JsonObjectConst>(), event)) return;
    if (xQueueSend(orderEventQueue, &event, 0) == pdTRUE)
      Serial.printf("[MQTT] Order queued: %s\n", event.orderCode);
  } else if (strcmp(topic, MQTT_TOPIC_ORDER_DONE) == 0) {
    const char *code = doc["order_id"] | "";
    if (!code[0] || !lvglMutex) return;
    if (xSemaphoreTake(lvglMutex, pdMS_TO_TICKS(300)) == pdTRUE) {
      int slot = findOrderSlotLocked(code);
      if (slot >= 0) { removeOrderCardLocked(slot); Serial.printf("… card removed: %s\n", code); }
      xSemaphoreGive(lvglMutex);
    }
  } else if (strcmp(topic, MQTT_TOPIC_INV) == 0) {
    if (!doc["items"].is<JsonArrayConst>()) return;
    if (parseInventory(doc["items"].as<JsonArrayConst>())) inventoryUiDirty = true;
  }
}
```

**Callback chạy ở đâu?** Bên trong `mqttClient.loop()` → trên **task mạng (lõi 0)** — giống paho gọi callback trên luồng mạng. Vì vậy không vẽ giao diện tùy tiện ở đây.

| Topic | Cách xử lý | Vì sao |
|---|---|---|
| `new_order` | `parseOrderObject` → **đẩy vào hàng đợi** `orderEventQueue` (3 ô, không chờ) → cuối vòng lặp `processOrderEventQueue()` (L1631) tạo thẻ | Tách việc nhận tin khỏi việc tạo thẻ; `addOrderCardUI` tự lấy khóa LVGL và **tự khử trùng** (đơn đã có thẻ thì bỏ qua) |
| `order_done` | Lấy khóa LVGL (chờ tối đa 300 ms) → tìm thẻ → gỡ | Việc nhỏ, làm ngay. Thẻ không có (đã gỡ) → `slot = −1`, không làm gì |
| `inventory` | `parseInventory` (giữ `dataMutex`) → đặt cờ `inventoryUiDirty` | **Không vẽ ở đây**: task UI (lõi 1) thấy cờ thì tự vẽ lại bảng kho |

- `strcmp(topic, …) == 0` — so sánh chuỗi C.
- `doc["order_id"] | ""` — toán tử `|` của ArduinoJson: thiếu trường thì lấy `""`.
- `parseOrderObject` dùng **chung** với HTTP (chấp nhận cả `order_id`/`id`, `name`/`n`, `quantity`/`q`).
- **Hàng đợi 3 ô, gửi không chờ**: nếu một lúc có hơn 3 đơn mới dồn tới trước khi vòng lặp kịp xử lý, đơn thứ 4 bị bỏ khỏi đường MQTT — và được **đồng bộ HTTP 30 s** vẽ bù. (Trước đây đồng bộ HTTP cũng đẩy qua hàng đợi này nên chỉ khôi phục được 3 đơn/lần; bản hiện tại tạo thẻ trực tiếp — đã ghi ở bài HTTP.)

---

## 6. So sánh PubSubClient (ESP32) và paho (Python)

| | paho (kiosk, API, webhook) | PubSubClient (bếp) |
|---|---|---|
| Luồng mạng | `loop_start()` tạo luồng nền | Không có — tự gọi `loop()` trong vòng lặp |
| Nối | `connect()` không chờ CONNACK / `connect_async()` | `connect()` **chờ CONNACK** → biết ngay sai tài khoản (`state 5`) |
| Tự nối lại | Có (luồng nền) | Không — code tự làm (`maintainMqtt`, mỗi 3 s) |
| Đăng ký lại sau khi nối | Trong `on_connect` | Ngay sau `connect()` thành công |
| QoS đăng ký | 0/1/2 | 0 hoặc 1 (firmware dùng 0) |
| Phát | QoS 0/1/2 | Chỉ QoS 0 (firmware không phát) |
| Kích thước tin | Gần như không giới hạn | Bộ đệm mặc định **256 B** — phải `setBufferSize` |
| Payload | `bytes` | `byte*` + `length`, **không có `'\0'`** |
| Kiểm `ts` khi nhận | Có (120 s, kiosk) | **Không** |

---

## 7. Tổng kết

### 7.1 Những gì cần nhớ
- **`setBufferSize`** quyết định màn hình bếp có nghe được gì hay không (thí nghiệm A).
- **Đăng ký lại sau mỗi lần nối** (phiên sạch) và **tự thử lại có giới hạn nhịp** (3 s).
- **Kiểm chữ ký trước mọi thứ**, so sánh hằng thời gian, HMAC trên đúng chuỗi nhận được.
- **Callback ngắn**: đẩy vào hàng đợi / đặt cờ, việc vẽ để task UI.
- MQTT chỉ là đường nhanh; **HTTP 30 s là nguồn sự thật** sửa mọi sai lệch (tin mất, tin quá cỡ, tin phát lại, hàng đợi đầy).

### 7.2 Ghi chú (không gây lỗi hiện tại, **chưa sửa**)
1. **Không kiểm `ts`** → tin cũ đã ký có thể bị phát lại (§4.4); thiệt hại giới hạn ≤ 30 s và chỉ trên màn hình.
2. **Đăng ký QoS 0**: chặng broker → bếp không có xác nhận; mất tin thì chờ HTTP 30 s.
3. **Kích thước tin kho tăng theo số nguyên liệu**: ~90 nguyên liệu sẽ vượt 8192 B → bị bỏ âm thầm như thí nghiệm A (firmware cũng chỉ giữ tối đa `MAX_INGREDIENTS = 80`).
4. **`connect()` chặn tới ~3 s (TCP) + 15 s (CONNACK)** khi broker “nửa sống”; trong lúc đó task mạng không làm việc khác (đồng bộ HTTP, gửi báo xong). Giao diện không bị ảnh hưởng (lõi khác).
5. **Hàng đợi đơn 3 ô**: đơn thứ 4 dồn tới cùng lúc chỉ hiện sau lần đồng bộ HTTP.
6. **ACL Mosquitto chưa có `kitchen/order_done` cho user `kds`** (chỉ ảnh hưởng khi triển khai Mosquitto nội bộ — đang hoãn).

---

## 8. Câu hỏi tự kiểm tra

1. Vì sao thay được `WiFiClient` bằng `PosixClient` mà không sửa PubSubClient? Điều đó nói gì về thiết kế thư viện?
2. Bỏ dòng `setBufferSize(8192)` thì màn hình bếp nhận được những tin nào? Vì sao không có thông báo lỗi nào?
3. Ở thí nghiệm A, vì sao tin giả mạo 361 B không in `Bad signature`?
4. `connect()` của PubSubClient khác `connect()` của paho ở điểm nào? Thí nghiệm nào cho thấy?
5. Giải thích `state = 5`, `−2`, `−3`.
6. Vì sao phải đăng ký lại cả 3 topic sau mỗi lần nối?
7. Không gọi `mqttClient.loop()` trong 30 s thì chuyện gì xảy ra?
8. Chữ ký `sig` là chuỗi gì? Vì sao firmware tính HMAC trên chuỗi `payload` nhận được chứ không phải trên JSON tự dựng lại?
9. Vì sao không dùng `strcmp` để so chữ ký?
10. Firmware không kiểm `ts`. Nêu lý do hợp lý, rủi ro, và cơ chế nào giới hạn thiệt hại.
11. Ba topic được xử lý khác nhau thế nào trong `mqttCallback`? Vì sao tin kho không vẽ giao diện ngay trong callback?
12. Đèn trạng thái hiện `MQTT...` nghĩa là gì? Kể hai nguyên nhân có thể.
13. Kiosk phát `kitchen/new_order` với QoS 1. Màn hình bếp nhận với QoS mấy? Vì sao?

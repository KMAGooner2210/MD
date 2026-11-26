# Giải thích chi tiết `order_notifier.cpp` — ESP32 giọng nói báo “ting ting” khi có đơn mới

> Thư mục: `Voice_Test/voice_assistant_edge_ai/src/`
> - `network/order_notifier.h` (74 dòng), `network/order_notifier.cpp` (181 dòng) — lớp `VoiceNetwork::OrderNotifier`.
> - `config/order_notify_config.h` (39 dòng) — địa chỉ broker, topic, bộ đệm.
> - Chỗ dùng trong `voice_assistant_edge_ai.ino`: khai báo L63–70, `serviceNewOrderChime()` L411–476, `setup()` L1798–1803, `loop()` L1842–1855.
>
> Học trước: `kds_firmware_mqtt.md` — file này dùng **cùng** PubSubClient và **cùng** cách kiểm chữ ký, nên tài liệu tập trung vào chỗ **khác**.
>
> **Thí nghiệm — chạy đúng file thật trên laptop:** chép `order_notifier.cpp/.h` (đã `diff`: **giống hệt** bản thật) và biên dịch cùng **PubSubClient 2.8** + **ArduinoJson 7.4.3** thật. Các phần phụ thuộc phần cứng được thay bằng “vỏ” tối giản: `WiFiClient` → socket Linux, `WiFi.status()` luôn “đã kết nối”, `ESP.getEfuseMac()` trả số cố định, `mbedtls_md_hmac` → `HMAC()` của OpenSSL. Riêng `order_notify_config.h` dùng **bản chép không include `secrets.h`** (không đọc khóa thật). Broker Mosquitto riêng (cổng 18860 mở, 18861 bắt đăng nhập) trong thư mục nháp; tin phát bằng đúng `sign_message` của dự án. Chương trình thử giả lập “trợ lý đang bận hỏi đáp” trong khoảng 3–5 s. Broker thử đã tắt sau khi chạy.

---

## 0. Bức tranh tổng thể (đọc trước)

### 0.1 Thiết bị này làm gì với MQTT
ESP32 giọng nói (thiết bị đặt ở bếp, nói chuyện với trợ lý qua gateway trên laptop) **nghe đúng một topic** `kitchen/new_order` — **cùng tin** mà màn hình bếp nghe — và phát tiếng **“ting ting”** + hiện “Có đơn mới DH…” trên màn hình nhỏ của nó.
```
Kiosk ──kitchen/new_order (ký HMAC)──▶ broker ──┬──▶ Màn hình bếp: hiện thẻ đơn
                                                └──▶ ESP32 giọng nói: "ting ting"
```
Đây là ví dụ rõ nhất về **ưu điểm phát–đăng ký**: thêm một thiết bị nghe mới **không phải sửa kiosk** — kiosk không biết (và không cần biết) có bao nhiêu bên nghe.

### 0.2 Kết quả chạy thử
| t | Tin phát lên `kitchen/new_order` | Thiết bị phản ứng |
|---|---|---|
| 0,0 s | Đơn `DH00000011`, ký đúng | `new_order=DH00000011` → **TING TING** |
| 0,4 s | Cùng đơn `DH00000011` lần 2 | **Im lặng** (bỏ qua đơn trùng) |
| 0,8 s | Ký **khóa sai** | `reject: bad signature` |
| 1,2 s | **Không ký** (JSON trần) | `reject: unsigned message` |
| 1,6 s | Ký đúng, dùng trường `order_code` thay `order_id` | `new_order=DH00000012` → **TING TING** |
| 2,2 s & 2,6 s | Hai đơn `DH00000013`, `DH00000014` **trong lúc trợ lý đang bận** | Nhận cả hai, nhưng khi rảnh (5,0 s) chỉ **một** TING TING — cho `DH00000014` |
| 4,6 s | Ký đúng nhưng `ts` cũ **1 ngày** | Vẫn **TING TING** (không kiểm `ts`) |

| Tình huống khác | Kết quả |
|---|---|
| Nhật ký broker | `connected … as VoiceKitchen_a1b2c3d4 (p2, c1, k30)` — keepalive **30 s**; `SUBSCRIBE kitchen/new_order (QoS 1)` |
| Khóa ký rỗng | `begin → 0`, `[ORDER_NOTIFY] disabled: chua khai bao ORDER_NOTIFY_MQTT_SECRET trong secrets.h` |
| Broker **bắt đăng nhập** | `connect_failed state=5` — thiết bị **không gửi tài khoản** |

---

## 1. Cấu hình — `order_notify_config.h`

```cpp
#include "secrets.h"
#ifndef ORDER_NOTIFY_MQTT_SECRET
#define ORDER_NOTIFY_MQTT_SECRET ""
#endif

namespace OrderNotifyConfig {
constexpr char BROKER_HOST[] = "broker.emqx.io";
constexpr uint16_t BROKER_PORT = 1883;
constexpr char TOPIC_NEW_ORDER[] = "kitchen/new_order";
constexpr uint32_t RECONNECT_INTERVAL_MS = 5000;
constexpr uint16_t MQTT_BUFFER_BYTES = 8192;
constexpr const char *MQTT_SECRET = ORDER_NOTIFY_MQTT_SECRET;
}
```
**Cú pháp**
- `#ifndef X` / `#define X ""` / `#endif` — **giá trị mặc định lúc biên dịch**: nếu `secrets.h` không định nghĩa `ORDER_NOTIFY_MQTT_SECRET` thì dùng chuỗi rỗng → chương trình vẫn biên dịch được, và tính năng **tự tắt** (đúng thí nghiệm “khóa rỗng”).
- `namespace OrderNotifyConfig { … }` — gom hằng số vào một “họ tên”, dùng `OrderNotifyConfig::BROKER_HOST`; tránh trùng tên với chỗ khác trong dự án lớn.
- `constexpr char BROKER_HOST[] = "…";` — mảng ký tự hằng, xác định lúc biên dịch, nằm trong flash.

**Logic & vì sao**
- **Broker viết cứng `broker.emqx.io`**, cổng 1883 — khác màn hình bếp (đọc `MQTT_BROKER_HOST` từ `secrets.h`). Xem ghi chú §6.
- Thử nối lại mỗi **5 s** (bếp: 3 s). Bộ đệm **8192 B** (đã thấy ở bài bếp: mặc định 256 B sẽ bỏ âm thầm tin 361 B).
- Khóa ký phải **trùng** `MQTT_SHARED_SECRET` (kiosk) và `MQTT_SECRET` (màn hình bếp).

---

## 2. Lớp `OrderNotifier` — khai báo (`order_notifier.h`)

```cpp
namespace VoiceNetwork {
class OrderNotifier {
public:
    static constexpr size_t ORDER_ID_LEN = 24;
    bool begin(const char *host, uint16_t port, const char *topic, const char *secret);
    bool isEnabled() const { return enabled_; }
    bool isConnected();
    void update(uint32_t nowMs);                         // gọi mỗi vòng loop()
    bool takeNewOrder(char *orderId, size_t capacity);   // lấy đơn mới rồi xóa cờ
private:
    static void onMessageStatic(char *topic, uint8_t *payload, unsigned int length);
    void onMessage(char *topic, uint8_t *payload, unsigned int length);
    bool verifyAndExtractOrderId(const uint8_t *payload, unsigned int length, char *orderId);
    bool computeHmacHex(const char *message, size_t length, char out[65]) const;

    static OrderNotifier *instance_;
    WiFiClient network_;
    PubSubClient mqtt_{network_};
    ...
    bool pending_ = false;
    char pendingOrderId_[ORDER_ID_LEN] = {0};
    char lastOrderId_[ORDER_ID_LEN] = {0};
};
}
```
**Khác màn hình bếp ở đâu?** Bếp viết MQTT bằng **hàm và biến toàn cục**; ở đây gói vào **một lớp** có giao diện nhỏ: `begin` – `update` – `takeNewOrder`. Phần còn lại của firmware (âm thanh, nhận lệnh giọng nói…) không cần biết gì về MQTT.

**Cú pháp đáng chú ý**
- `public:` / `private:` — phần ngoài được gọi / phần chỉ dùng nội bộ.
- `bool isEnabled() const` — `const` sau hàm: hàm **không sửa** đối tượng.
- `PubSubClient mqtt_{network_};` — khởi tạo thành viên bằng `{}` ngay trong khai báo, truyền **tham chiếu** tới `network_` (khai báo **trước** nó — thứ tự khai báo quyết định thứ tự khởi tạo).
- Hậu tố `_` (`mqtt_`, `pending_`) — quy ước đặt tên biến thành viên.

### 2.1 Vì sao cần `instance_` và `onMessageStatic`?
`setCallback` của PubSubClient nhận một **hàm thường** (con trỏ hàm `void(*)(char*, uint8_t*, unsigned int)`). Một **phương thức** như `onMessage` cần biết “đối tượng nào” (`this`) nên không đưa thẳng vào được. Mẹo kinh điển:
```cpp
OrderNotifier *OrderNotifier::instance_ = nullptr;          // biến tĩnh: một cho cả lớp
...
instance_ = this;                                            // trong begin()
mqtt_.setCallback(onMessageStatic);                          // hàm static = hàm thường
...
void OrderNotifier::onMessageStatic(char *topic, uint8_t *payload, unsigned int length) {
    if (instance_ != nullptr) instance_->onMessage(topic, payload, length);   // chuyển cho đối tượng
}
```
Giới hạn: chỉ dùng được **một** `OrderNotifier` trong chương trình — đủ cho thiết bị này. (Trên ESP32, PubSubClient còn cho truyền `std::function`/lambda; cách static ở đây chạy được trên mọi nền tảng.)

---

## 3. `begin()` (L14–38) — bật/tắt theo khóa

```cpp
enabled_ = secret_ != nullptr && secret_[0] != '\0';
if (!enabled_) {
    Serial.println("[ORDER_NOTIFY] disabled: chua khai bao ORDER_NOTIFY_MQTT_SECRET trong secrets.h");
    return false;
}
instance_ = this;
mqtt_.setServer(host_, port_);
mqtt_.setCallback(onMessageStatic);
mqtt_.setBufferSize(OrderNotifyConfig::MQTT_BUFFER_BYTES);
mqtt_.setKeepAlive(30);
```
- **Không có khóa → tắt hẳn**, không nối broker: fail‑closed, và trợ lý giọng nói vẫn chạy bình thường (tính năng phụ không được làm hỏng tính năng chính).
- `setKeepAlive(30)` — 30 s thay vì mặc định 15 s (nhật ký broker: `k30`) → ít gói PING hơn. Đổi lại broker phát hiện thiết bị chết chậm hơn (~45 s); với tính năng “ting ting” không quan trọng.

---

## 4. `update()` (L44–67) — giữ kết nối trong `loop()`

```cpp
void OrderNotifier::update(uint32_t nowMs) {
    if (!enabled_ || WiFi.status() != WL_CONNECTED) return;
    if (mqtt_.connected()) { mqtt_.loop(); return; }

    if (attemptedOnce_ && nowMs - lastAttemptMs_ < OrderNotifyConfig::RECONNECT_INTERVAL_MS) return;
    attemptedOnce_ = true;
    lastAttemptMs_ = nowMs;

    String clientId = "VoiceKitchen_" + String(static_cast<uint32_t>(ESP.getEfuseMac()), HEX);
    if (mqtt_.connect(clientId.c_str())) {
        const bool subscribed = mqtt_.subscribe(topic_, 1);
        Serial.printf("[ORDER_NOTIFY] connected subscribe=%d\n", subscribed ? 1 : 0);
    } else {
        Serial.printf("[ORDER_NOTIFY] connect_failed state=%d\n", mqtt_.state());
    }
}
```
Cùng khuôn với `maintainMqtt()` của màn hình bếp, khác ở:
| | Màn hình bếp | ESP32 giọng nói |
|---|---|---|
| Client id | `ESP32_Kitchen_<MAC>` | `VoiceKitchen_<MAC>` (thí nghiệm: `VoiceKitchen_a1b2c3d4`) — **khác tiền tố** để hai thiết bị không đá nhau |
| Lần thử đầu | Chờ 3 s kể từ khởi động | `attemptedOnce_`: **thử ngay** lần đầu, sau đó mới giãn 5 s |
| Đăng nhập broker | Có (`MQTT_USERNAME/PASSWORD`) | **Không** — `connect(clientId)` |
| QoS đăng ký | 0 | **1** (`subscribe(topic_, 1)`) |
| Gọi ở | Task mạng riêng (lõi 0) | `loop()` chính của sketch |

- `attemptedOnce_` giải một chi tiết nhỏ: lúc mới bật, `nowMs` nhỏ hơn 5000 nên phép so `nowMs - lastAttemptMs_ < 5000` sẽ bắt chờ vô ích; cờ này cho lần đầu đi qua ngay.
- `connect()` của PubSubClient **chặn** (TCP + chờ CONNACK, tối đa ~3 s + 15 s nếu broker “nửa sống”). Ở đây nó nằm trong **`loop()` chính** → trong lúc chờ, các việc khác của `loop()` cũng phải chờ. Bình thường nối nhanh (thí nghiệm: tức thì); chỉ đáng kể khi mạng/broker có vấn đề và chỉ xảy ra mỗi 5 s khi đang mất kết nối.

---

## 5. Nhận tin — `onMessage()` + `verifyAndExtractOrderId()` (L84–179)

### 5.1 Lọc và chống trùng
```cpp
void OrderNotifier::onMessage(char *topic, uint8_t *payload, unsigned int length) {
    if (topic == nullptr || strcmp(topic, topic_) != 0) return;
    char orderId[ORDER_ID_LEN] = {0};
    if (!verifyAndExtractOrderId(payload, length, orderId)) return;

    // Kiosk và đồng bộ có thể gửi lại cùng một đơn: chỉ báo một lần.
    if (strncmp(orderId, lastOrderId_, ORDER_ID_LEN) == 0) return;

    strlcpy(lastOrderId_, orderId, sizeof(lastOrderId_));
    strlcpy(pendingOrderId_, orderId, sizeof(pendingOrderId_));
    pending_ = true;
    Serial.printf("[ORDER_NOTIFY] new_order=%s\n", orderId);
}
```
- **Chống trùng** bằng cách nhớ **mã đơn vừa báo** (`lastOrderId_`). Cần thiết vì đăng ký **QoS 1** → broker có thể giao lại cùng tin (thí nghiệm: `DH00000011` lần 2 → im lặng). Chỉ nhớ **một** mã: đủ cho trùng liên tiếp; nếu tin trùng đến xen sau một đơn khác thì sẽ kêu lại (hiếm).
- `strlcpy(dst, src, size)` — chép chuỗi **không tràn** mảng, luôn có `'\0'` ở cuối (an toàn hơn `strcpy`).
- `strncmp(a, b, n)` — so tối đa `n` ký tự.
- Callback **không phát âm thanh**: chỉ đặt cờ `pending_` + lưu mã. Việc phát nằm ở `loop()` (§5.3) — callback ngắn, giống bếp.

### 5.2 Kiểm chữ ký — giống màn hình bếp, khác 2 điểm
```cpp
JsonDocument envelope;
if (deserializeJson(envelope, payload, length)) { … "reject: invalid envelope"; return false; }
if (envelope["v"].as<int>() != 2 || !envelope["payload"].is<const char *>() ||
    !envelope["sig"].is<const char *>())          { … "reject: unsigned message"; return false; }
const char *inner = envelope["payload"].as<const char *>();
const char *signature = envelope["sig"].as<const char *>();
char expected[65];
if (strlen(signature) != 64 || !computeHmacHex(inner, strlen(inner), expected)) { … "reject: bad signature"; }
uint8_t diff = 0;
for (size_t i = 0; i < 64; ++i) diff |= static_cast<uint8_t>(expected[i] ^ signature[i]);   // so hằng thời gian
if (diff != 0) { … "reject: bad signature"; return false; }

JsonDocument doc;
if (deserializeJson(doc, inner)) { … "reject: invalid payload"; return false; }
const char *id = doc["order_id"] | "";
if (id[0] == '\0') id = doc["order_code"] | "";
if (id[0] == '\0') { … "reject: missing order_id"; return false; }
strlcpy(orderId, id, ORDER_ID_LEN);
```
1. **So sánh hằng thời gian viết trực tiếp** trong hàm (bếp tách thành `constantTimeEquals`). Cùng nguyên lý: luôn duyệt đủ 64 ký tự.
2. **Chỉ lấy mã đơn**, chấp nhận cả `order_id` (tin đơn mới của kiosk) lẫn `order_code` (dạng tin thanh toán) — thí nghiệm: tin chỉ có `order_code` vẫn kêu. Thiết bị không cần danh sách món, chỉ cần biết “có đơn mới, mã gì”.
3. Khóa rỗng không cần kiểm lại ở đây vì `begin()` đã tắt tính năng.
4. **Không kiểm `ts`** — giống bếp; thí nghiệm: tin cũ 1 ngày vẫn kêu. Hậu quả tệ nhất: một tiếng “ting ting” thừa (không có HTTP nào để “sửa”, nhưng cũng không có gì để sai lệch).

### 5.3 Từ cờ tới tiếng chuông — trong sketch chính
```cpp
// loop() — L1842–1855
if (wifiManager.isConnected()) orderNotifier.update(currentTimeMs);
if (orderNotifier.takeNewOrder(newOrderChimeOrderId, sizeof(newOrderChimeOrderId))) {
    newOrderChimePending = true;
}
```
```cpp
// serviceNewOrderChime() — L411–476 (rút gọn)
if (newOrderChimePlaying) {
    if (!voiceManager.isIdle()) { audioPlayer.cancel(); … "chime_cancelled reason=USER_SPEAKING"; return; }
    … phát tiếp tới khi xong …
}
if (!newOrderChimePending || newOrderChimeSamples == 0 || !voiceManager.isIdle()) return;   // đang bận thì CHỜ
newOrderChimePending = false;
audioPlayer.playPcm16(newOrderChime, newOrderChimeSamples);
kitchenDisplay.setExchange("Có đơn mới", newOrderChimeOrderId);
```
Mẫu **“cờ + lấy ra”** (`takeNewOrder` đọc rồi xóa cờ) tách **nơi nhận tin** khỏi **nơi quyết định khi nào phát**:
- Trợ lý **rảnh** → phát ngay, màn hình nhỏ hiện “Có đơn mới DH…”.
- Trợ lý **đang hỏi đáp** → **chờ** xong phiên mới báo (không chen ngang câu trả lời).
- Đang kêu mà người dùng **bấm nói** → **dừng chuông ngay** (`USER_SPEAKING`).
- Nhiều đơn đến trong lúc bận → mã đơn bị **ghi đè**, chỉ kêu **một lần** với mã **mới nhất** (thí nghiệm: `DH00000013` và `DH00000014` đến lúc bận → một tiếng cho `DH00000014`). Với mục đích “nhắc có đơn” thì đủ; danh sách đầy đủ nằm trên màn hình bếp.

Tiếng chuông được **tổng hợp sẵn** lúc khởi động (`Audio::buildNewOrderChime`, L1792) vào mảng PCM → phát không cần mạng, không cần tệp âm thanh.

---

## 6. Ghi chú (không gây lỗi hiện tại, **chưa sửa**)
1. **Broker viết cứng `broker.emqx.io`** trong khi kiosk/API/bếp đọc địa chỉ broker từ cấu hình. Nếu sau này chuyển cả hệ thống sang Mosquitto nội bộ (đang hoãn) mà quên sửa file này, thiết bị giọng nói sẽ nghe **nhầm broker** và không bao giờ kêu.
2. **Không gửi tài khoản broker** → với broker bắt đăng nhập (`allow_anonymous false` như `deploy/mosquitto/mosquitto.conf`), thiết bị bị từ chối (thí nghiệm: `connect_failed state=5`). File ACL cũng chưa có tài khoản cho thiết bị này. Chỉ ảnh hưởng khi triển khai Mosquitto nội bộ.
3. **Không kiểm `ts`** → tin cũ đã ký phát lại sẽ gây một tiếng chuông thừa.
4. **`connect()` chặn trong `loop()` chính** khi broker có vấn đề (mỗi 5 s một lần thử).
5. **Chống trùng chỉ nhớ một mã**; **nhiều đơn lúc bận → một tiếng chuông** (chủ ý, nhưng nên biết).

---

## 7. So sánh hai thiết bị nghe `kitchen/new_order`

| | Màn hình bếp | ESP32 giọng nói |
|---|---|---|
| Cần gì từ tin | Toàn bộ đơn (món, số lượng, tùy chọn) | Chỉ **mã đơn** |
| Topic | 3 topic | 1 topic |
| QoS đăng ký | 0 | 1 |
| Chống trùng | `addOrderCardUI` bỏ qua thẻ đã có | Nhớ mã đơn vừa báo |
| Mất tin thì | HTTP 30 s vẽ bù | Không kêu (không có đường bù — chấp nhận được) |
| Broker | Từ `secrets.h`, có đăng nhập | Viết cứng `broker.emqx.io`, không đăng nhập |
| Cấu trúc code | Hàm + biến toàn cục | Lớp `OrderNotifier` đóng gói |
| Kiểm chữ ký | Có (mbedTLS, hằng thời gian) | Có (giống hệt về nguyên lý) |
| Kiểm `ts` | Không | Không |

---

## 8. Câu hỏi tự kiểm tra

1. Vì sao thêm thiết bị “ting ting” mà không phải sửa một dòng nào của kiosk? Đó là ưu điểm nào của MQTT?
2. `#ifndef ORDER_NOTIFY_MQTT_SECRET … #define … ""` dùng để làm gì? Chuyện gì xảy ra khi `secrets.h` không có khóa?
3. Vì sao phải có `onMessageStatic` và `instance_`? Giới hạn của cách làm này là gì?
4. `attemptedOnce_` giải quyết vấn đề gì lúc thiết bị vừa khởi động?
5. Vì sao client id của thiết bị này phải khác tiền tố với màn hình bếp?
6. Thiết bị đăng ký QoS 1. Điều đó dẫn tới khả năng gì, và code xử lý bằng cách nào?
7. Callback `onMessage` có phát âm thanh trực tiếp không? Vì sao?
8. Ba đơn mới đến trong lúc người dùng đang hỏi trợ lý. Sau phiên hỏi đáp, thiết bị kêu mấy lần, hiện mã đơn nào?
9. Đang kêu “ting ting” thì người dùng bấm nói — chuyện gì xảy ra?
10. Nếu hệ thống chuyển sang Mosquitto nội bộ có bắt đăng nhập, thiết bị này cần sửa những gì để còn hoạt động? (gợi ý: §6 điểm 1, 2 và file ACL)
11. So sánh cách kiểm chữ ký ở đây với `verifySignedMessage` của màn hình bếp: giống và khác ở đâu?

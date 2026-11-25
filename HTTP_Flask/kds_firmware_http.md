# Giải thích chi tiết phần HTTP trong `kds_firmware.ino`

> File: `kds_firmware.ino/kds_firmware/kds_firmware.ino` — 1854 dòng, firmware màn hình bếp (KDS = *Kitchen Display System*) trên **ESP32‑S3** (bo JC4827W543C, màn 480×272).
> Tài liệu này **chỉ** đi vào phần **HTTP** (khoảng 250 dòng). Phần MQTT/HMAC, LVGL, FreeRTOS sẽ học ở các chủ đề sau; ở đây chỉ nhắc khi cần để hiểu HTTP.
>
> **Thí nghiệm** (không đụng mạch, không đụng Pi):
> 1. Dựng **Smart Kiosk API thật** trên laptop (cổng 5988) với **CSDL tạm** và **khóa thử** (`kitchen-test`, `admin-test`), rồi dùng `curl` gửi **đúng các yêu cầu** firmware gửi (cùng đường dẫn, header, body).
> 2. Chép logic “mã trạng thái → quyết định” và phép tính thời gian nghỉ ra một chương trình C++ nhỏ, chạy trên laptop.
> 3. `arduino-cli compile` toàn bộ firmware (chỉ biên dịch, **không nạp**).

---

## 0. Bức tranh tổng thể (đọc trước)

### 0.1 Màn hình bếp nói chuyện với ai, bằng gì?

```
                       ┌──────────── Raspberry Pi ────────────┐
                       │  Smart Kiosk API (Flask, cổng 5000)  │
 ┌── ESP32 bếp ──┐     │   GET  /api/orders/pending           │
 │               │──①──│   GET  /api/inventory                │
 │  HTTP client  │──②──│   POST /api/orders/done              │
 │               │     └──────────────────────────────────────┘
 │  MQTT client  │◀─③── broker: kitchen/new_order, kitchen/inventory,
 └───────────────┘          kitchen/order_done  (tin ký HMAC)
```

| Kênh | Hướng | Dùng để | Đặc điểm |
|---|---|---|---|
| ① HTTP **GET** | bếp **hỏi** Pi | Lấy **toàn bộ** đơn chờ + toàn bộ kho (ảnh chụp – *snapshot*) | Chủ động, chắc chắn có câu trả lời (hoặc biết là lỗi) |
| ② HTTP **POST** | bếp **báo** Pi | “Đơn DHxxxxxxxx đã làm xong” | Cần biết **chắc** đã lưu → phải có mã trạng thái trả về |
| ③ MQTT | Pi **đẩy** xuống bếp | Đơn mới, kho thay đổi, đơn xong ở nơi khác | Tức thời, nhưng **mất tin nếu lúc đó bếp đang mất mạng** |

**Vì sao cần cả HTTP lẫn MQTT?** MQTT nhanh nhưng “bắn rồi quên”: nếu ESP32 khởi động lại hoặc rớt Wi‑Fi đúng lúc có đơn, tin đó mất. HTTP GET là **lưới an toàn**: mỗi 30 s bếp hỏi lại “CSDL nói còn những đơn nào?” và sửa màn hình cho khớp. Còn **báo xong** phải dùng HTTP POST vì bếp cần **biết kết quả** (đã lưu / đã có người làm / sai khóa) để quyết định gỡ thẻ hay hiện nút “GỬI LẠI”.

> So với `admin_bridge.py` (Python, `requests`): cùng là **máy khách HTTP**, cùng gửi `X-API-Key`, cùng đặt timeout — nhưng ở đây viết bằng **C++** trên vi điều khiển, **không có ngoại lệ (exception)**, RAM ít, và phải tự lo **thử lại**.

### 0.2 Kết quả chạy thử

| Yêu cầu (giống hệt firmware) | Máy chủ trả | Firmware hiểu thành |
|---|---|---|
| `GET /api/orders/pending` + khóa bếp | **200**, JSON `{"data":[...]}` 2 đơn PAID (đơn chưa trả tiền không có) | Vẽ/giữ thẻ |
| `GET /api/orders/pending` không khóa / sai khóa | **401** `unauthorized` | `httpGetJson` trả `false`, 5 s hoặc 30 s sau hỏi lại |
| `GET /api/inventory` + khóa bếp | **200** `[]` (CSDL tạm chưa có nguyên liệu) | `parseInventory` trả `false` → xem §9 ghi chú 1 |
| `GET /api/report` + khóa bếp | **403** `forbidden` | (firmware không gọi — chứng minh khóa bếp bị giới hạn quyền) |
| `POST /api/orders/done` DH00000001 | **200** | `DONE_OK` → gỡ thẻ |
| Gửi lại lần 2 cùng mã | **409** | `DONE_OK` (đã xong rồi → vẫn gỡ thẻ) |
| Đơn chưa trả tiền / mã không tồn tại | **409** | `DONE_OK` |
| Body thiếu `order_code` | **400** | `DONE_FATAL` → “GỬI LẠI” ngay |
| Sai khóa | **401** | `DONE_FATAL` → “GỬI LẠI” ngay |
| `GET pending` sau khi xong DH00000001 | **200**, chỉ còn DH00000002 | Lần đồng bộ sau gỡ thẻ DH00000001 nếu còn |
| Máy chủ tắt (cổng đóng) | lỗi kết nối sau **0,0001 s** | `status < 0` → `DONE_RETRY` |
| IP không tồn tại, `connect-timeout 2.5` | lỗi sau **2,50 s** | `status < 0` → `DONE_RETRY` |
| `arduino-cli compile` | **exit 0**; chương trình dùng 1.435.227 B (45 % flash), biến toàn cục 131.580 B (40 % RAM) | — |

Chương trình C++ mô phỏng (§6.4, §6.5) cho:
```
status  200 -> DONE_OK        status  404 -> DONE_RETRY
status  409 -> DONE_OK        status  500 -> DONE_RETRY
status  400 -> DONE_FATAL     status  503 -> DONE_RETRY
status  401 -> DONE_FATAL     status   -1 -> DONE_RETRY  (connection refused)
status  403 -> DONE_FATAL     status  -11 -> DONE_RETRY  (read timeout)
status  200 nhưng success=false -> DONE_RETRY
retryCount=1 -> nghỉ 1000 ms; =2 -> 2000 ms; =3 -> 4000 ms
```

### 0.3 Bản đồ phần HTTP trong file

| Dòng | Thành phần | Vai trò |
|---|---|---|
| L5 | `#include <HTTPClient.h>` | Thư viện HTTP của lõi Arduino‑ESP32 |
| L27 + `secrets.h` | `API_SERVER_HOST`, `API_SERVER_PORT`, `KDS_API_KEY` | Địa chỉ Pi và khóa bếp |
| L78, L85–86 | `DONE_QUEUE_LEN=4`, `INITIAL_SYNC_RETRY_MS=5000`, `PERIODIC_SYNC_MS=30000` | Hằng số nhịp đồng bộ |
| L136–143 | `struct DoneOrderMsg`, `enum DoneResult` | Tin nhắn “báo xong” và 3 kết quả |
| L189 | `apiUrl()` | Ghép URL |
| L382 | `doneButtonEvent()` | Bấm nút “HOÀN THÀNH” → bỏ vào hàng đợi |
| L1332, L1351 | `parseInventory()`, `parseOrderObject()` | Đọc JSON thành struct C |
| L1395 | `httpGetJson()` | **Hàm GET dùng chung** |
| L1415–1473 | `syncInventoryFromApi()`, `syncOrdersFromApi()`, `syncInitialData()` | Đồng bộ với CSDL |
| L1602 | `postDoneOrder()` | **POST báo xong**, phân loại kết quả |
| L1636–1674 | `markDoneFailedLocked()`, `processDoneQueue()` | Thử lại có nghỉ tăng dần |
| L1676 | `taskNetwork()` | Vòng lặp mạng: lúc nào gọi HTTP |

---

## 1. Cấu hình và ghép URL

```cpp
#include <HTTPClient.h>          // L5
#include "secrets.h"             // L27
...
static String apiUrl(const char *path) {                                     // L189
  return "http://" + String(API_SERVER_HOST) + ":" + String(API_SERVER_PORT) + path;
}
```

**Cú pháp**
- `#include "secrets.h"` (ngoặc kép) = tìm file **trong thư mục sketch** trước; `<HTTPClient.h>` (ngoặc nhọn) = tìm trong thư viện.
- `secrets.h` không lên git; `secrets.example.h` là mẫu, có các tên:
  ```cpp
  static const char *API_SERVER_HOST = "192.168.1.10";        // mẫu, không phải giá trị thật
  static constexpr uint16_t API_SERVER_PORT = 5000;
  static const char *KDS_API_KEY = "kitchen-api-key";
  ```
  - `static` ở phạm vi file = biến chỉ thấy trong file này (*internal linkage*).
  - `constexpr` = hằng số tính lúc biên dịch.
  - `uint16_t` = số nguyên không dấu 16 bit, vừa đủ cho cổng 0–65535.
- `static String apiUrl(...)`: `String` là lớp chuỗi của Arduino, cấp phát động trên heap. `"http://" + String(...)`: phải bọc `String(...)` vì trong C++, `const char* + const char*` là **cộng con trỏ**, không phải nối chuỗi; có ít nhất một vế là `String` thì toán tử `+` của `String` mới được dùng.

**Logic.** Ví dụ `apiUrl("/api/orders/pending")` → `http://<IP Pi>:5000/api/orders/pending`.

**Vì sao như vậy.**
- Địa chỉ Pi **cố định lúc biên dịch**: đổi IP Pi thì phải nạp lại firmware. Đây là đánh đổi để đơn giản (không cần mDNS hay màn hình cài đặt).
- Khóa bếp tách khỏi khóa admin: nếu ai đó đọc được flash ESP32 thì cũng chỉ có quyền xem đơn/kho và báo xong (đúng như thí nghiệm `GET /api/report` → 403).

---

## 2. `httpGetJson()` — hàm GET dùng chung (L1395–1413)

```cpp
static bool httpGetJson(const String &url, JsonDocument &doc) {
  HTTPClient http;
  if (!http.begin(url)) return false;
  http.setConnectTimeout(2500);
  http.setTimeout(4000);
  // Smart Kiosk yêu cầu khóa bếp cho đơn chờ và tồn kho.
  http.addHeader("X-API-Key", KDS_API_KEY);
  int status = http.GET();
  if (status != HTTP_CODE_OK) {
    Serial.printf("[HTTP] GET failed %d: %s\n", status, url.c_str());
    http.end();
    return false;
  }
  String body = http.getString();
  http.end();
  DeserializationError error = deserializeJson(doc, body);
  if (error) Serial.printf("[JSON] %s\n", error.c_str());
  return !error;
}
```

### 2.1 Cú pháp
| Đoạn | Nghĩa |
|---|---|
| `const String &url` | Truyền **tham chiếu hằng**: không chép chuỗi, không cho sửa |
| `JsonDocument &doc` | Truyền **tham chiếu** để hàm **ghi kết quả vào** biến của người gọi (C++ không trả được 2 giá trị một cách tự nhiên → trả `bool` thành công, dữ liệu đi qua tham số ra) |
| `HTTPClient http;` | Đối tượng **cục bộ trên stack**: mỗi lần gọi tạo mới, ra khỏi hàm tự hủy |
| `http.begin(url)` | Phân tích URL (giao thức, host, cổng, đường dẫn). **Chưa** kết nối mạng. URL sai dạng → `false` |
| `setConnectTimeout(2500)` | Tối đa 2,5 s để **mở kết nối TCP** (mặc định của thư viện là 5000 ms) |
| `setTimeout(4000)` | Tối đa 4 s **chờ dữ liệu** sau khi đã nối |
| `addHeader("X-API-Key", ...)` | Thêm header xác thực, giống `_AUTH_HEADERS` bên `admin_bridge.py` |
| `int status = http.GET();` | **Lúc này mới** nối TCP, gửi yêu cầu, đọc dòng trạng thái + header. Trả **mã HTTP** (200, 401…) hoặc **số âm** nếu lỗi tầng dưới |
| `HTTP_CODE_OK` | Hằng `200` do thư viện định nghĩa |
| `http.getString()` | Đọc **toàn bộ thân** phản hồi vào một `String` trong RAM |
| `http.end()` | Đóng kết nối, giải phóng tài nguyên. **Phải gọi ở mọi nhánh** (cả nhánh lỗi) |
| `deserializeJson(doc, body)` | ArduinoJson 7: phân tích chuỗi JSON vào `doc`; trả `DeserializationError` (ép sang `bool` = có lỗi hay không) |
| `error.c_str()` | Tên lỗi dạng chữ, vd. `IncompleteInput`, `NoMemory` |

**Mã âm của `HTTPClient`** (đọc trong `HTTPClient.h` của lõi ESP32 3.3.10):

| Mã | Tên | Khi nào |
|---|---|---|
| −1 | `HTTPC_ERROR_CONNECTION_REFUSED` | Không nối được TCP (cổng đóng, IP không trả lời quá 2,5 s) |
| −4 | `HTTPC_ERROR_NOT_CONNECTED` | Chưa có kết nối |
| −5 | `HTTPC_ERROR_CONNECTION_LOST` | Đứt giữa chừng |
| −11 | `HTTPC_ERROR_READ_TIMEOUT` | Đã nối nhưng quá 4 s không có dữ liệu |

→ Khác `requests` của Python (ném **ngoại lệ** `ConnectionError`/`Timeout`), ở đây lỗi mạng và lỗi HTTP đi chung **một con số**: âm = mạng, dương = máy chủ đã trả lời.

### 2.2 Yêu cầu thật sự đi trên dây
Đọc `HTTPClient.cpp` (hàm dựng header, L1137–1170), một lần `GET /api/orders/pending` trông như sau (mỗi dòng kết thúc `\r\n`):
```
GET /api/orders/pending HTTP/1.1
Host: 192.168.x.x:5000
User-Agent: ESP32HTTPClient
Connection: keep-alive
Accept-Encoding: identity;q=1,chunked;q=0.1,*;q=0
X-API-Key: <khóa bếp>

```
- `Host` có kèm `:5000` vì cổng khác 80/443.
- `User-Agent`, `Connection`, `Accept-Encoding` do **thư viện tự thêm**; firmware chỉ thêm `X-API-Key` (và `Content-Type` khi POST).
- `Connection: keep-alive` nhưng vì `http` là biến cục bộ và gọi `end()` ngay, nên **mỗi yêu cầu vẫn mở một kết nối TCP mới** — đơn giản, chấp nhận được với nhịp 30 s.

### 2.3 Logic & vì sao
1. Không phải 200 → in log, **đóng kết nối**, trả `false`. Người gọi không cần biết lý do cụ thể: GET thất bại thì lần đồng bộ sau thử lại.
2. Thành công → đọc thân **rồi mới** `end()` (đóng trước thì mất dữ liệu).
3. JSON hỏng (vd. bị cắt giữa chừng) → `false`, không dùng dữ liệu nửa vời.

**Unicode:** máy chủ Flask trả tiếng Việt dạng thoát `à` (thấy rõ trong thí nghiệm: `"Trà sữa"`). ArduinoJson mặc định **giải mã** `\uXXXX` thành UTF‑8, nên chuỗi trong `doc` là “Trà sữa” thật, font tiếng Việt của LVGL vẽ được.

---

## 3. Đồng bộ kho — `syncInventoryFromApi()` (L1415–1423)

```cpp
static bool syncInventoryFromApi() {
  JsonDocument doc;
  if (!httpGetJson(apiUrl("/api/inventory"), doc)) return false;
  if (!doc.is<JsonArrayConst>()) return false;
  if (!parseInventory(doc.as<JsonArrayConst>())) return false;
  inventoryUiDirty = true;
  Serial.println("[SYNC] Inventory parsed; UI refresh deferred");
  return true;
}
```

- `JsonDocument doc;` — ArduinoJson 7 tự cấp phát bộ nhớ theo nhu cầu (bản 6 phải khai kích thước trước).
- `doc.is<JsonArrayConst>()` — **kiểm tra kiểu trước khi dùng**: `/api/inventory` trả **mảng** `[...]`; nếu máy chủ đổi định dạng hoặc trả `{"success":false}` thì dừng an toàn. `is<T>()` / `as<T>()` là **template**: `<...>` chọn kiểu lúc biên dịch.
- `JsonArrayConst` — bản **chỉ đọc** của mảng JSON.
- `inventoryUiDirty = true;` — **không vẽ giao diện ở đây**. Hàm này chạy trong **task mạng** (lõi 0); LVGL chỉ được vẽ ở **task UI** (lõi 1). Cờ `volatile bool` báo “dữ liệu đã đổi”, task UI thấy cờ thì tự vẽ lại. Đây là mẫu “**đánh dấu bẩn, vẽ sau**” (*dirty flag*).

`parseInventory()` (L1332) đọc từng phần tử: bỏ qua phần tử thiếu `id`/`name`/`unit` hoặc sai kiểu; `item["quantity"] | 0.0F` — toán tử `|` của ArduinoJson nghĩa là “**lấy giá trị, nếu thiếu/sai kiểu thì dùng mặc định**”; trường `low` do máy chủ tính sẵn (`quantity <= low_stock_threshold`, `api_server.py` L731–732) → bếp không cần biết ngưỡng. Giới hạn `MAX_INGREDIENTS = 80` vì mảng tĩnh. Hàm lấy `dataMutex` vì task UI cũng đọc `inventoryList`.

---

## 4. Đồng bộ đơn — `syncOrdersFromApi()` (L1425–1462) — “CSDL là nguồn sự thật”

```cpp
static bool syncOrdersFromApi() {
  JsonDocument doc;
  if (!httpGetJson(apiUrl("/api/orders/pending"), doc)) return false;
  if (!doc["data"].is<JsonArrayConst>()) return false;

  static char pendingCodes[MAX_ORDER_CARDS * 2][ORDER_CODE_LEN];
  int pendingCount = 0;
  int added = 0;

  for (JsonObjectConst source : doc["data"].as<JsonArrayConst>()) {
    KdsOrderEvent event{};
    if (!parseOrderObject(source, event)) continue;
    if (pendingCount < (int)(sizeof(pendingCodes) / sizeof(pendingCodes[0])))
      copyText(pendingCodes[pendingCount++], ORDER_CODE_LEN, event.orderCode);
    if (addOrderCardUI(event)) ++added;
  }

  int removed = 0;
  if (lvglMutex && xSemaphoreTake(lvglMutex, pdMS_TO_TICKS(300)) == pdTRUE) {
    for (int slot = 0; slot < MAX_ORDER_CARDS; ++slot) {
      if (!orderCards[slot].active || orderCards[slot].state == ORDER_COMPLETING) continue;
      bool stillPending = false;
      for (int i = 0; i < pendingCount; ++i) {
        if (strcmp(orderCards[slot].orderCode, pendingCodes[i]) == 0) { stillPending = true; break; }
      }
      if (!stillPending) { removeOrderCardLocked(slot); ++removed; }
    }
    xSemaphoreGive(lvglMutex);
  }
  ...
}
```

### 4.1 Cú pháp đáng chú ý
- `static char pendingCodes[24][24];` — `static` **trong hàm** = biến sống suốt chương trình, nằm ở vùng nhớ tĩnh chứ **không phải stack**. Lý do: 24×24 = 576 byte; để trên stack của task mạng (12 KB) cũng được, nhưng firmware chủ ý giảm áp lực stack.
- `sizeof(pendingCodes) / sizeof(pendingCodes[0])` — mẹo C cổ điển tính **số phần tử** mảng (576 / 24 = 24).
- `for (JsonObjectConst source : ...)` — vòng lặp **range‑for** của C++11, giống `for source in data:` của Python.
- `KdsOrderEvent event{};` — `{}` = khởi tạo mọi trường về 0/rỗng.
- `strcmp(a, b) == 0` — so sánh chuỗi C (bằng nhau thì trả 0). Không dùng `==` vì `==` trên `char*` so sánh **địa chỉ**.
- `xSemaphoreTake(lvglMutex, pdMS_TO_TICKS(300))` — khóa FreeRTOS, chờ tối đa 300 ms; không lấy được thì **bỏ qua bước gỡ thẻ lần này** (lần sau làm), không treo task.

### 4.2 Logic — hai pha
1. **Thêm:** mọi đơn có trong phản hồi → `addOrderCardUI()` (tự khử trùng: thẻ đã có thì thôi). Đây là chỗ **cứu đơn bị mất tin MQTT**.
2. **Gỡ:** thẻ đang hiện mà **không còn trong danh sách CSDL** → gỡ. Trường hợp: đơn được báo xong **ở nơi khác** (giọng nói qua Kitchen MCP, phần mềm quản trị) và tin MQTT `kitchen/order_done` bị lỡ. Thí nghiệm: sau khi POST done DH00000001, `GET pending` chỉ còn DH00000002 → lần đồng bộ sau thẻ DH00000001 sẽ bị gỡ.
3. **Trừ ra thẻ `ORDER_COMPLETING`** (đang gửi POST): tránh tranh chấp — nếu đồng bộ gỡ thẻ trong khi hàng đợi “báo xong” vẫn đang xử lý nó thì hai luồng logic cùng thao tác một thẻ.

### 4.3 Phía máy chủ trả gì (`api_server.py` L634–695)
Máy chủ **chuẩn hóa** dữ liệu cho bếp: lấy `items_json` (ảnh chụp đơn lúc đặt), tách `base_name` (“Trà sữa” thay vì “Trà sữa (L)”), gom `size`/`ice` vào `options`. `parseOrderObject()` (L1351) đọc lại đúng các khóa đó, nhưng **chịu cả hai định dạng**: `order_id` hoặc `id`, `name` hoặc `n`, `quantity` hoặc `q` — định dạng rút gọn là của tin MQTT (cho tin ngắn). Một hàm phân tích dùng chung cho HTTP và MQTT.

Chỉ đơn `payment_status='PAID' AND kitchen_status='pending'` mới về bếp — thí nghiệm: đơn `PENDING` (chưa trả tiền) không xuất hiện.

---

## 5. Khi nào gọi GET — `taskNetwork()` (L1676–1727)

```cpp
if (!initialSyncDone && WiFi.status() == WL_CONNECTED &&
    now - lastSyncAttempt >= INITIAL_SYNC_RETRY_MS) {          // 5 s
  lastSyncAttempt = now;
  initialSyncDone = syncInitialData();                          // kho + đơn
  if (initialSyncDone) lastPeriodicSync = now;
}
if (initialSyncDone && WiFi.status() == WL_CONNECTED &&
    now - lastPeriodicSync >= PERIODIC_SYNC_MS) {               // 30 s
  lastPeriodicSync = now;
  syncInventoryFromApi();
  syncOrdersFromApi();
}
```

- **Khởi động:** cứ 5 s thử đồng bộ đầy đủ đến khi **cả kho và đơn** đều thành công (`inventoryOk && ordersOk`).
- **Sau đó:** 30 s/lần, gọi cả hai, **bỏ qua kết quả** (thất bại thì 30 s sau thử lại).
- `now - lastSyncAttempt >= ...` — kiểu so sánh **trừ rồi so**, đúng cả khi `millis()` tràn số (xem §6.5).
- Không có Wi‑Fi → không gọi HTTP (tránh mất 2,5 s chờ vô ích).

**Hệ quả của việc gọi đồng bộ (blocking):** `http.GET()` **chặn task mạng** tới 2,5 + 4 = 6,5 s mỗi yêu cầu khi Pi mất mạng. Trong lúc đó `mqttClient.loop()` không chạy. Nhưng **màn hình vẫn mượt** vì LVGL chạy ở task UI trên **lõi kia** (lõi 1). Đây là lý do tách hai task trên hai lõi — tương tự `requestStats()` chạy luồng nền ở `admin_bridge.py`.

---

## 6. Báo đơn xong — từ nút bấm tới POST (phần quan trọng nhất)

### 6.1 Toàn cảnh
```
 Task UI (lõi 1)                 hàng đợi doneOrderQueue (4 ô)        Task mạng (lõi 0)
 doneButtonEvent ──xQueueSend──▶ [DH..01][      ][      ][      ] ──▶ processDoneQueue
 thẻ: "ĐANG LƯU..."                                                    └ postDoneOrder → HTTP POST
                                                                          ├ DONE_OK    → gỡ thẻ
                                                                          ├ DONE_RETRY → nghỉ 1/2/4 s, xếp lại cuối hàng
                                                                          └ DONE_FATAL → thẻ "GỬI LẠI"
```
**Vì sao không gọi HTTP ngay trong hàm xử lý nút bấm?** Hàm đó chạy trong task UI; gọi POST ở đó thì Pi chậm 6,5 s là **màn hình đứng 6,5 s**. Nên UI chỉ **bỏ tin vào hàng đợi** (mất vài micro giây), việc mạng để task mạng làm. Hàng đợi FreeRTOS **an toàn giữa các task** (giống `queue.Queue` của Python).

### 6.2 Phía nút bấm — `doneButtonEvent()` (L382–396)
```cpp
if (orderCards[slot].state == ORDER_COMPLETING) return;     // đang gửi thì bấm nữa cũng bỏ qua
DoneOrderMsg msg{};
copyText(msg.orderCode, sizeof(msg.orderCode), orderCards[slot].orderCode);
if (xQueueSend(doneOrderQueue, &msg, 0) != pdTRUE) {         // 0 = không chờ
  setOrderButtonLocked(slot, "GỬI LẠI", true);               // hàng đầy → cho bấm lại
  return;
}
orderCards[slot].state = ORDER_COMPLETING;
setOrderButtonLocked(slot, "ĐANG LƯU...", false);            // khóa nút
```
- `(int)(intptr_t)lv_event_get_user_data(event)` (dòng trên đoạn trích) — số thứ tự thẻ được “nhét” vào con trỏ `void*` lúc tạo nút; `intptr_t` là kiểu số nguyên đủ rộng để chứa con trỏ, ép hai bước để trình biên dịch không cảnh báo.
- `xQueueSend` **chép** cả struct vào hàng đợi (không chép con trỏ) → không lo struct cục bộ bị hủy.
- Chống bấm hai lần: trạng thái `ORDER_COMPLETING` + nút bị vô hiệu.

### 6.3 `DoneOrderMsg` và `DoneResult` (L136–143)
```cpp
struct DoneOrderMsg {
  char orderCode[ORDER_CODE_LEN] = {0};   // mảng cố định 24 byte, không dùng String
  uint8_t retryCount = 0;
  uint32_t notBeforeMs = 0;               // chưa tới thời điểm này thì chưa gửi lại
};
enum DoneResult : uint8_t { DONE_OK, DONE_RETRY, DONE_FATAL };
```
- Dùng `char[24]` chứ không dùng `String`: hàng đợi FreeRTOS chép **từng byte**; `String` chứa con trỏ tới heap, chép byte sẽ thành hai đối tượng cùng trỏ một vùng nhớ → hỏng. Struct trong hàng đợi phải là **dữ liệu thuần** (*POD*).
- `enum ... : uint8_t` — chỉ định kiểu nền 1 byte. Ba kết quả thay vì `bool` vì có **hai kiểu thất bại khác bản chất** (xem 6.4).

### 6.4 `postDoneOrder()` (L1602–1629) — phân loại mã trạng thái
```cpp
static DoneResult postDoneOrder(const char *orderCode) {
  if (WiFi.status() != WL_CONNECTED) return DONE_RETRY;
  HTTPClient http;
  if (!http.begin(apiUrl("/api/orders/done"))) return DONE_RETRY;
  http.setConnectTimeout(2500);
  http.setTimeout(4000);
  http.addHeader("Content-Type", "application/json");
  http.addHeader("X-API-Key", KDS_API_KEY);
  JsonDocument req;
  req["order_code"] = orderCode;
  String body;
  serializeJson(req, body);                 // → {"order_code":"DH00000001"}
  int status = http.POST(body);
  String response = http.getString();
  http.end();
  if (status == 409) return DONE_OK;
  if (status == 400 || status == 401 || status == 403) {
    Serial.printf("[HTTP] Done %s rejected %d\n", orderCode, status);
    return DONE_FATAL;
  }
  if (status < 200 || status >= 300) return DONE_RETRY;
  JsonDocument res;
  if (deserializeJson(res, response)) return DONE_RETRY;
  return (res["success"] | false) ? DONE_OK : DONE_RETRY;
}
```

**Cú pháp**
- `req["order_code"] = orderCode;` + `serializeJson(req, body)` — dựng JSON bằng thư viện thay vì tự ghép chuỗi `"{\"order_code\":\"" + ...` → không lo thoát ký tự `"`.
- `Content-Type: application/json` — **bắt buộc**: Flask `request.get_json()` chỉ đọc body khi header này đúng; thiếu thì máy chủ coi như body rỗng → 400 `missing_order_code`.
- `http.POST(body)` — tự thêm `Content-Length`.
- `res["success"] | false` — thiếu trường thì coi là `false`.

**Logic — bảng quyết định (đã chạy thử, §0.2):**

| Mã | Nghĩa từ máy chủ | Quyết định | Lý do |
|---|---|---|---|
| 200 + `success:true` | Đã lưu | `DONE_OK` | — |
| **409** | Đơn không còn ở trạng thái chờ (đã xong ở nơi khác, hoặc không tồn tại) | **`DONE_OK`** | Mục tiêu của bếp là “đơn này không còn phải làm” → đã đạt. Đây là **tính lũy đẳng** (*idempotent*): gửi 2 lần cũng ra cùng kết quả cuối |
| 400 / 401 / 403 | Dữ liệu sai / sai khóa / không đủ quyền | `DONE_FATAL` | Lỗi **không tự hết**: thử lại 3 lần nữa cũng y vậy, chỉ phí 7 s và che giấu lỗi cấu hình |
| 5xx, mã âm (mạng), 404, JSON hỏng, `success:false` | Lỗi tạm thời hoặc không rõ | `DONE_RETRY` | Pi đang khởi động lại, Wi‑Fi chập chờn… có thể tự hết |

**Vì sao 409 = OK lại quan trọng?** Kịch bản: POST lần 1 tới Pi, Pi **đã lưu**, nhưng phản hồi bị mất trên đường về (timeout) → firmware nhận `-11` → thử lại → lần 2 Pi trả **409** (vì đã `done`). Nếu coi 409 là lỗi thì thẻ kẹt “GỬI LẠI” dù đơn đã xong. Phía máy chủ, `admin_mark_order_done()` dùng `UPDATE ... WHERE kitchen_status='pending'` + kiểm `rowcount == 1` (đã học ở chủ đề SQLite) — chính điều kiện này sinh ra 409 an toàn.

### 6.5 `processDoneQueue()` (L1642–1674) — thử lại với **nghỉ tăng dần** (*exponential backoff*)
```cpp
DoneOrderMsg msg{};
if (xQueueReceive(doneOrderQueue, &msg, 0) != pdTRUE) return;      // hàng rỗng

bool waiting = msg.notBeforeMs && (int32_t)(millis() - msg.notBeforeMs) < 0;
if (waiting && xQueueSendToBack(doneOrderQueue, &msg, 0) == pdTRUE) return;

DoneResult result = waiting ? DONE_FATAL : postDoneOrder(msg.orderCode);
...
} else if (result == DONE_RETRY && msg.retryCount < 3) {
  ++msg.retryCount;
  msg.notBeforeMs = millis() + (1000UL << (msg.retryCount - 1));   // 1 s, 2 s, 4 s
  if (msg.notBeforeMs == 0) msg.notBeforeMs = 1;
  if (xQueueSendToBack(doneOrderQueue, &msg, 0) != pdTRUE) markDoneFailedLocked(slot);
} else {
  markDoneFailedLocked(slot);                                     // "GỬI LẠI"
}
```

**Cú pháp**
- `1000UL << (n - 1)` — dịch trái bit = nhân 2ⁿ⁻¹; `UL` = `unsigned long` để không tràn kiểu `int`. Ra 1000, 2000, 4000 ms.
- `(int32_t)(millis() - msg.notBeforeMs) < 0` — “**chưa tới giờ**”. Viết kiểu này chứ không viết `millis() < notBeforeMs` vì `millis()` là `uint32_t`, sau **~49,7 ngày** (2³² ms) quay về 0. Mô phỏng trên laptop với `notBeforeMs` đã tràn thành 704:

  | millis | `millis < notBefore` (ngây thơ) | cách firmware |
  |---|---|---|
  | 4294967000 | 0 → **gửi ngay, sai** | 1 → chờ, đúng |
  | 4294967295 | 0 → **sai** | 1 → chờ |
  | 500 | 1 | 1 |
  | 704 | 0 | 0 → gửi |

  Phép trừ số không dấu vẫn ra đúng “khoảng cách”, ép sang có dấu thì biết trước/sau (miễn hai thời điểm cách nhau < 24,8 ngày).
- `if (msg.notBeforeMs == 0) msg.notBeforeMs = 1;` — vì `0` được dùng làm nghĩa “không phải chờ”; nếu phép cộng vô tình ra đúng 0 thì đổi thành 1.

**Logic**
1. Lấy một tin. **Chưa tới giờ** → bỏ lại **cuối hàng** rồi thoát (task mạng quay lại sau 5 ms). Nhờ vậy một đơn đang chờ nghỉ **không chặn** đơn khác phía sau.
2. Tới giờ → POST. Rồi lấy khóa giao diện để cập nhật thẻ; **không lấy được khóa** → trả tin về hàng đợi (lần sau gửi lại thì máy chủ trả 409 = OK, vẫn đúng).
3. Tổng cộng tối đa **4 lần gửi** (1 lần đầu + 3 lần thử lại). Tệ nhất khi Pi không phản hồi: 4 × 6,5 s + 1 + 2 + 4 = **~33 s** rồi thẻ chuyển “GỬI LẠI” — đầu bếp bấm lại khi mạng ổn.

**Vì sao nghỉ tăng dần?** Thử lại liên tục khi Pi đang quá tải chỉ làm nó quá tải thêm; chờ lâu dần cho mạng/máy chủ thời gian hồi phục. Đây là kỹ thuật chuẩn (AWS SDK, gRPC, `urllib3.Retry` đều dùng).

---

## 7. Đối chiếu với phía máy chủ — bảng quyền

| Endpoint | Decorator (`api_server.py`) | Khóa bếp | Khóa admin | Không khóa |
|---|---|---|---|---|
| `GET /api/orders/pending` | `require_kitchen_or_admin_key` | 200 | 200 | 401 |
| `GET /api/inventory` | `require_kitchen_or_admin_key` | 200 | 200 | 401 |
| `POST /api/orders/done` | `require_kitchen_or_admin_key` | 200/409 | 200/409 | 401 |
| `GET /api/report` (bếp không dùng) | `require_api_key` | **403** | 200 | 401 |

- 401 = “anh là ai?” (thiếu/sai khóa); 403 = “biết anh là ai, nhưng không được làm việc này”. Firmware xử lý cả hai như `DONE_FATAL`.
- Sau POST thành công, máy chủ phát MQTT `kitchen/order_done` (`publish_order_done`, `api_server.py` L133) — các màn hình khác (nếu có) gỡ thẻ ngay; chính màn này đã gỡ rồi nên `findOrderSlotLocked` trả −1, không làm gì.

---

## 8. Tổng kết

### 8.1 Máy khách HTTP tốt trên vi điều khiển cần gì — đối chiếu
| Yêu cầu | Trong firmware |
|---|---|
| Luôn có timeout | `setConnectTimeout(2500)`, `setTimeout(4000)` ở **mọi** yêu cầu |
| Xác thực | `X-API-Key` khóa bếp (quyền tối thiểu) |
| Không chặn giao diện | HTTP chỉ ở task mạng (lõi 0), UI ở lõi 1, giao tiếp qua hàng đợi + cờ |
| Phân biệt lỗi tạm thời / vĩnh viễn | `DONE_RETRY` vs `DONE_FATAL` |
| Thử lại có kiểm soát | Tối đa 3 lần, nghỉ 1/2/4 s, không chặn đơn khác |
| Lũy đẳng | 409 = OK |
| Tự sửa trạng thái | GET snapshot 30 s/lần: thêm đơn sót, gỡ đơn đã xong |
| Kiểm tra dữ liệu vào | `is<T>()` trước `as<T>()`, bỏ qua phần tử sai |
| Giải phóng tài nguyên | `http.end()` ở mọi nhánh |
| Dữ liệu qua hàng đợi là POD | `char[24]` thay `String` |

### 8.2 Ghi chú (không gây lỗi hiện tại, **chưa sửa**)
1. **Kho rỗng ⇒ “đồng bộ ban đầu” không bao giờ xong.** `parseInventory()` trả `false` khi 0 nguyên liệu → `syncInitialData()` = `false` → `initialSyncDone` mãi `false`. Hậu quả nhẹ: vòng “thử lại 5 s” vẫn gọi **cả kho lẫn đơn**, nên đơn vẫn hiện; chỉ là gọi dày hơn (5 s thay 30 s). CSDL thật có 62 nguyên liệu nên không gặp. Thí nghiệm CSDL tạm `[]` đúng là rơi vào nhánh này.
2. **`pendingCodes` chứa tối đa 24 mã.** Nếu một lúc có > 24 đơn chờ, thẻ của đơn thứ 25 trở đi (vd. đơn mới đến qua MQTT) có thể bị gỡ nhầm ở bước 2 rồi không vẽ lại được vì 12 ô thẻ đã đầy. Quầy nhỏ khó chạm mức này.
3. **404 bị coi là `DONE_RETRY`.** Máy chủ hiện không bao giờ trả 404 ở `/api/orders/done` (không tìm thấy → 409), nên không ảnh hưởng; nếu sau này đổi máy chủ trả 404 thì nên thêm 404 vào nhóm `DONE_OK` hoặc `DONE_FATAL`.
4. **GET không phân biệt 401.** Sai khóa thì `httpGetJson` chỉ in log và thử lại mãi; màn hình không báo “sai khóa”. Muốn biết phải xem Serial.
5. **Đồng hồ trên thẻ tính từ lúc thẻ hiện, không theo `created_at`.** `createdAt` được đọc (L1359) nhưng bộ đếm dùng `startMillis` (L531) → ESP32 khởi động lại thì mọi đơn đếm lại từ 0. (Lưu ý thêm: `created_at` trong CSDL là giờ **UTC** — thí nghiệm thấy `16:13` khi giờ máy là 23:13.)
6. **HTTP không mã hóa.** Khóa bếp đi dạng chữ thường trong mạng Wi‑Fi (giống `admin_bridge.py`; HTTPS đã để vào hướng phát triển).
7. Địa chỉ Pi cố định trong `secrets.h` — đổi IP phải nạp lại.

---

## 9. Câu hỏi tự kiểm tra

1. `http.begin(url)` có mở kết nối mạng không? Dòng nào mới thật sự gửi dữ liệu đi?
2. `http.GET()` trả `-1` và trả `401` khác nhau thế nào về **nơi** xảy ra lỗi?
3. Vì sao `apiUrl()` phải viết `"http://" + String(API_SERVER_HOST)` chứ không viết `"http://" + API_SERVER_HOST`?
4. Nếu bỏ dòng `Content-Type: application/json` trong `postDoneOrder`, máy chủ trả mã gì, và firmware hiển thị gì trên thẻ?
5. Vì sao 409 được coi là thành công? Mô tả kịch bản mà nếu coi 409 là lỗi thì thẻ kẹt.
6. Vì sao 401/403 không thử lại, còn 503 thì có?
7. Tính lại: Pi tắt hẳn (IP không trả lời), bấm “HOÀN THÀNH” — sau tối đa bao lâu thẻ chuyển “GỬI LẠI”?
8. `DoneOrderMsg` dùng `char orderCode[24]` chứ không dùng `String`. Chuyện gì xảy ra nếu dùng `String` trong struct gửi qua `xQueueSend`?
9. Viết lại điều kiện “chưa tới giờ” theo cách ngây thơ và chỉ ra giá trị `millis()` làm nó sai.
10. Đơn được báo xong bằng giọng nói nhưng tin MQTT `kitchen/order_done` bị mất. Sau tối đa bao lâu thẻ trên bếp biến mất, nhờ hàm nào, ở pha nào?
11. Vì sao `syncOrdersFromApi()` **không** gỡ thẻ đang ở trạng thái `ORDER_COMPLETING`?
12. Gọi `http.GET()` chặn tới 6,5 s — vì sao màn hình cảm ứng vẫn không bị đứng?
13. So sánh với `admin_bridge.py`: chỗ nào tương ứng với `_request()`, với `timeout=3`, với `except requests.RequestException`?

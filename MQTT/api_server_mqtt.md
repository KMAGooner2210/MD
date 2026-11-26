# Giải thích chi tiết phần MQTT trong `servers/api_server.py`

> File: `Smart_Kiosk_Backup/servers/api_server.py` — 848 dòng. Phần MQTT: **L82–149** (cấu hình, kết nối, 2 hàm phát) và **5 chỗ gọi** trong các route (L717, L746, L761, L775, L789).
> Phần HTTP/Flask của file đã học ở `hoc_tap/SQLite/api_server.md`. Học trước: `00_kien_thuc_can_ban.md`, `webhook_server_mqtt.md`, `bridge_mqtt.md`.
>
> **Thí nghiệm** (không đụng Pi, `.env`, `kiosk.db` thật hay Mosquitto hệ thống):
> - Broker Mosquitto **riêng** cổng 18850 trong thư mục nháp, **bật/tắt theo kịch bản** (cấu hình tối giản, *không* bật `persistence`).
> - Import **đúng module** `api_server` với CSDL **bản sao**, khóa thử `admin-test` / `kitchen-test`, khóa HMAC thử; gọi route qua `app.test_client()`.
> - Một client “màn hình bếp giả” đăng ký `kitchen/#` và **kiểm chữ ký** mọi tin nhận được.
> - Chạy 2 kịch bản trong 2 tiến trình riêng: broker **đang chạy** lúc API khởi động, và broker **đang tắt** lúc API khởi động.

---

## 0. Bức tranh tổng thể (đọc trước)

### 0.1 API làm gì với MQTT
API **chỉ phát**, không đăng ký topic nào. Hai loại tin:

| Topic | Khi nào | Ai cần | Hàm |
|---|---|---|---|
| `kitchen/inventory` (retain) | Quản trị viên **thêm / sửa / xóa / nhập** nguyên liệu | Màn hình bếp (bảng kho) | `publish_inventory_snapshot()` L104 |
| `kitchen/order_done` | Một đơn được **báo xong** qua API (từ màn hình bếp, Kitchen MCP/giọng nói, phần mềm quản trị) | Màn hình bếp gỡ thẻ ngay | `publish_order_done()` L133 |

```
Phần mềm quản trị ──HTTP──▶ API ──┬─▶ database.py (ghi kho)
                                   └─▶ MQTT kitchen/inventory ──▶ broker ──▶ màn hình bếp
Trợ lý giọng nói ──HTTP──▶ API ──┬─▶ database.py (đơn → done)
                                  └─▶ MQTT kitchen/order_done ──▶ broker ──▶ màn hình bếp (gỡ thẻ)
```
**Vì sao API phải phát?** Kiosk chỉ phát kho **sau mỗi lần bán**. Khi quản trị viên nhập kho trên laptop, thay đổi đi qua **API**, kiosk không biết → nếu API không phát thì màn hình bếp phải chờ tới lần đồng bộ HTTP 30 s mới thấy. Tương tự với đơn được báo xong bằng giọng nói.

### 0.2 Kiểu kết nối: client **toàn cục**, nối **một lần lúc import**
| | Webhook | Kiosk | **API** |
|---|---|---|---|
| Thời gian sống | Mỗi giao dịch | Suốt ứng dụng | **Suốt tiến trình** |
| Cách nối | `connect()` | `connect_async()` | **`connect()` (chặn) lúc import** |
| Callback | Không | `on_connect/on_message/on_disconnect` | **Không** |
| QoS khi phát | 1 | 1 | **0 (mặc định)** |
| Kiểm kết quả phát | `wait_for_publish` | `publish_info.rc` | **Không** |

### 0.3 Kết quả chạy thử

**Kịch bản 1 — broker đang chạy lúc API khởi động:**
| # | Thao tác | Kết quả |
|---|---|---|
| 1 | Import `api_server` | `--> [MQTT] api_server da ket noi Broker: 127.0.0.1`, `connected = True` |
| 2 | `POST /api/inventory/restock` (khóa admin) | HTTP **200** → bếp nhận `kitchen/inventory`, **qos=0**, 5497 byte, 62 nguyên liệu, chữ ký đúng |
| 3 | `POST /api/orders/done` `DH88720928` (khóa bếp) | HTTP **200** → bếp nhận `kitchen/order_done`, qos=0, **179 byte**, `order_id = DH88720928` |
| 4 | Tắt broker, nhập kho | HTTP **200** (API vẫn trả thành công), `connected = False` |
| 5 | Bật lại broker | API **tự nối lại** sau ~0,9 s (nhờ `loop_start` đang chạy) |
| 6 | Bếp đăng ký lại `kitchen/#` | **Không nhận gì**: bản kho phát ở bước 4 (QoS 0, lúc mất kết nối) đã **mất**; bản retained cũ cũng mất vì broker thử không bật `persistence` |
| 7 | Nhập kho lần nữa | Bếp nhận `kitchen/inventory` bình thường |

**Kịch bản 2 — broker đang TẮT lúc API khởi động:**
| # | Thao tác | Kết quả |
|---|---|---|
| 1 | Import `api_server` | `--> [LOI MQTT] api_server khong the ket noi Broker: [Errno 111] Connection refused`; `connected = False`; **luồng paho: `None`** |
| 2 | Bật broker, chờ 8 s | `connected = False` — **không bao giờ tự nối** |
| 3 | `POST /api/inventory/restock` | HTTP **200**, `success: true` — không có dòng lỗi nào được in |
| 4 | Thử `publish` tay | `rc = MQTT_ERR_NO_CONN` |
| 5 | Bếp | **Không nhận gì** |

---

## 1. Cấu hình và kết nối (L82–101)

```python
MQTT_BROKER = os.environ.get("MQTT_BROKER", "broker.emqx.io")
MQTT_PORT = int(os.environ.get("MQTT_PORT", 1883))
MQTT_TOPIC_INVENTORY = "kitchen/inventory"
MQTT_TOPIC_ORDER_DONE = "kitchen/order_done"
MQTT_SHARED_SECRET = os.environ.get("MQTT_SHARED_SECRET", "")

try:
    _mqtt_client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
except AttributeError:
    _mqtt_client = mqtt.Client()

configure_client(_mqtt_client, role="API")

try:
    _mqtt_client.connect(MQTT_BROKER, MQTT_PORT, 60)
    _mqtt_client.loop_start()
    print(f"--> [MQTT] api_server da ket noi Broker: {MQTT_BROKER}")
except Exception as e:
    print(f"--> [LOI MQTT] api_server khong the ket noi Broker: {e}")
```

### 1.1 Cú pháp
- Đoạn này nằm ở **cấp module**, không trong hàm nào → chạy **đúng một lần khi module được import** (lúc `python servers/api_server.py` khởi động, hoặc khi test `import api_server`).
- `_mqtt_client` — dấu `_` đầu tên = quy ước “nội bộ của module”. Là **biến toàn cục**, mọi route dùng chung **một** client.
- `role="API"` → tài khoản `MQTT_API_USERNAME/_PASSWORD`. Theo `deploy/mosquitto/acl`, user `api` chỉ được **ghi `kitchen/inventory`** (thiếu `kitchen/order_done` — đã ghi chú ở `00_kien_thuc_can_ban.md` phần F).
- Topic **viết cứng** (khác kiosk đọc từ `.env`).

### 1.2 Logic
1. `connect()` **chặn**: mở TCP + gửi CONNECT ngay. Lỗi TCP → ném ngoại lệ.
2. Thành công → `loop_start()` tạo luồng nền: gửi tin, giữ keepalive, **tự nối lại** khi rớt (kịch bản 1, bước 5).
3. Thất bại → chỉ in lỗi. **`loop_start()` nằm trong cùng khối `try` nên không được gọi** → không có luồng nền → không ai thử nối lại (kịch bản 2: `luồng paho: None`, 8 s sau vẫn `connected = False`).

### 1.3 Vì sao vẫn chạy được trong thực tế?
- Dịch vụ `deploy/pi/kiosk-api.service` có `After=network-online.target` + `Wants=network-online.target` → systemd cố đợi mạng sẵn sàng rồi mới chạy API, giảm khả năng nối hỏng lúc khởi động.
- Tin MQTT của API chỉ là **đường nhanh**: màn hình bếp vẫn tự đồng bộ kho và đơn qua HTTP mỗi 30 s.
- Nhưng `Restart=on-failure` **không giúp** ở đây: tiến trình không chết, Flask vẫn phục vụ bình thường, chỉ có MQTT “im lặng” vĩnh viễn.

---

## 2. `publish_inventory_snapshot()` (L104–130)

```python
def publish_inventory_snapshot():
    try:
        ingredients = db.get_all_ingredients()
        if not MQTT_SHARED_SECRET:
            print("--> [BẢO MẬT] Chưa cấu hình MQTT_SHARED_SECRET: không phát tồn kho qua MQTT.")
            return
        payload = {
            "version": 1,
            "event": "inventory.snapshot",
            "items": [
                {"id": i["id"], "name": i["name"], "unit": i["unit"],
                 "quantity": i["quantity"], "low": i["quantity"] <= i["low_stock_threshold"]}
                for i in ingredients
            ]
        }
        _mqtt_client.publish(MQTT_TOPIC_INVENTORY, sign_message(payload, MQTT_SHARED_SECRET), retain=True)
    except Exception as e:
        print(f"--> Loi publish inventory MQTT: {e}")
```

### 2.1 Cú pháp và logic
- Đọc **toàn bộ** kho từ CSDL (đã ghi xong) → đóng gói **ảnh chụp** → ký → phát **retain**.
- **Cùng hợp đồng** với tin kho của kiosk (`version`, `event`, `items[id,name,unit,quantity,low]`), chỉ thiếu `created_at` → firmware xử lý hai nguồn như nhau. Thí nghiệm: 5497 byte (kiosk: 5542 byte — phần chênh chủ yếu là trường `created_at`, và lượng tồn kho lúc đo cũng khác).
- **Hai bên cùng phát retain lên một topic** (kiosk sau mỗi lần bán, API sau mỗi lần sửa kho): broker chỉ giữ **tin cuối**. Không mâu thuẫn vì cả hai đều đọc **từ CSDL** tại thời điểm phát → tin cuối luôn phản ánh kho mới nhất.
- `publish(...)` không ghi `qos` → **QoS 0**. Bếp nhận với `qos=0` (thí nghiệm). `retain=True`.
- Không kiểm tra giá trị trả về của `publish`. `try/except` chỉ bắt lỗi **ném ra** (vd. lỗi đọc CSDL) — mà `publish` khi mất kết nối **không ném**, chỉ trả `rc` (xem §5).

### 2.2 Được gọi ở đâu (sau khi ghi CSDL xong)
| Dòng | Route | Thao tác |
|---|---|---|
| L746 | `POST /api/inventory/add` | Thêm nguyên liệu |
| L761 | `POST /api/inventory/update` | Sửa tên/đơn vị/ngưỡng |
| L775 | `POST /api/inventory/delete` | Xóa nguyên liệu |
| L789 | `POST /api/inventory/restock` | Nhập thêm kho |

Thứ tự **ghi CSDL → phát MQTT** giống kiosk: CSDL là nguồn sự thật; phát lỗi thì bếp vẫn đồng bộ lại được qua HTTP.
Việc phát nằm **trong** request HTTP: nếu broker chậm, request của quản trị viên chậm theo một chút (với QoS 0 thì `publish` chỉ xếp hàng, gần như tức thời).

---

## 3. `publish_order_done()` (L133–149)

```python
def publish_order_done(order_code):
    """Báo cho KDS gỡ thẻ ngay khi đơn được báo xong ở nơi khác (vd. giọng nói).

    Không retain: chỉ có ý nghĩa tức thời; KDS vẫn tự đồng bộ 30 s/lần.
    """
    if not MQTT_SHARED_SECRET:
        return
    try:
        _mqtt_client.publish(
            MQTT_TOPIC_ORDER_DONE,
            sign_message({"version": 1, "event": "order.done", "order_id": order_code}, MQTT_SHARED_SECRET),
            retain=False,
        )
    except Exception as e:
        print(f"--> Loi publish order_done MQTT: {e}")
```
Gọi ở `POST /api/orders/done` (L717) **chỉ khi** `admin_mark_order_done()` thành công (đơn thật sự chuyển `pending → done`). Đơn đã xong rồi (409) thì **không** phát — tránh tin thừa.

**Đường đi khi đầu bếp nói “đơn DH… xong rồi”:**
```
Trợ lý giọng nói → Kitchen MCP → POST /api/orders/done (khóa bếp)
  → database: UPDATE … SET kitchen_status='done' WHERE … AND kitchen_status='pending'
  → publish_order_done("DH…")  → kitchen/order_done (179 byte, ký HMAC)
  → màn hình bếp (mqttCallback): verifySignedMessage → tìm thẻ → removeOrderCardLocked
```
Thẻ biến mất **ngay**, thay vì chờ tới 30 s.

**Vì sao không retain?** Tin “đơn X đã xong” là **sự kiện**. Retain thì màn hình bếp mỗi lần khởi động lại sẽ nhận một tin cũ vô nghĩa. Trạng thái đúng khi khởi động lấy qua HTTP `/api/orders/pending`.

**Khi chính màn hình bếp bấm “Hoàn thành”:** firmware POST `/api/orders/done` → API cũng phát `kitchen/order_done` → chính màn hình đó nhận lại; thẻ đã gỡ rồi nên `findOrderSlotLocked` trả −1, không làm gì. Vô hại, và có ích nếu sau này có **nhiều** màn hình bếp.

---

## 4. Vì sao QoS 0 ở đây chấp nhận được?

| Tin | Mất thì sao | Có đường bù |
|---|---|---|
| `kitchen/inventory` từ API | Bếp hiển thị kho cũ | Bếp `GET /api/inventory` mỗi 30 s; lần bán tiếp theo kiosk phát lại (QoS 1) |
| `kitchen/order_done` | Thẻ đã xong vẫn còn trên màn | Bếp `GET /api/orders/pending` mỗi 30 s → gỡ thẻ |

So sánh: `payment/success` của webhook dùng **QoS 1 + `wait_for_publish`** vì mất tin thì **khách chờ** (dù có polling bù). Tin của API chỉ là **làm mới hiển thị**, chậm 30 s không ảnh hưởng nghiệp vụ.
Chọn QoS theo **hậu quả khi mất tin**, không phải “cứ cao là tốt”.

---

## 5. Khi broker có sự cố — đọc kết quả thí nghiệm

### 5.1 Broker rớt khi API đang chạy (kịch bản 1, bước 4–7)
- Luồng `loop_start` **tự nối lại** (~0,9 s sau khi broker sống lại).
- Tin phát **trong lúc mất kết nối** với **QoS 0** bị **bỏ** (paho chỉ giữ hộ tin QoS ≥ 1 — so sánh: tin QoS 1 của kiosk ở `bridge_mqtt.md` §6.3 thì vẫn được gửi sau khi nối lại).
- Tin **retained** trên broker: broker thử của thí nghiệm không bật `persistence` nên khởi động lại là mất. `deploy/mosquitto/mosquitto.conf` của dự án **có** `persistence true` → retained được giữ qua lần khởi động lại broker. Với broker công cộng `broker.emqx.io` thì không kiểm soát được.
- API **vẫn trả 200** cho quản trị viên — đúng, vì việc chính (ghi CSDL) đã thành công.

### 5.2 Broker không có lúc API khởi động (kịch bản 2)
Phân tích ở §1.2: không có `loop_start` → client **chết vĩnh viễn** trong tiến trình đó. Mọi `publish` trả `MQTT_ERR_NO_CONN`, không ném lỗi, không in gì → **im lặng hoàn toàn**. Cách duy nhất để hồi phục là khởi động lại dịch vụ API.

So sánh cách kiosk làm (`connect_async` + `loop_start` luôn chạy) → kiosk tự hồi phục trong cùng tình huống (thí nghiệm A–B của `bridge_mqtt.md`).

---

## 6. Dùng chung một client trong máy chủ nhiều luồng
`api_server` chạy bằng **Waitress 8 luồng**: hai yêu cầu nhập kho cùng lúc có thể cùng gọi `_mqtt_client.publish(...)` từ hai luồng khác nhau. paho bảo vệ hàng đợi gửi bằng khóa nội bộ nên `publish()` gọi từ nhiều luồng là an toàn. Không cần (và không nên) tạo client mới cho mỗi request — mỗi client là một kết nối TCP + một luồng nền.

---

## 7. Tác dụng phụ lúc import (lưu ý khi viết test)
Vì kết nối nằm ở cấp module, **chỉ cần `import api_server`** là đã nối tới broker — mặc định là **`broker.emqx.io` công cộng** nếu không đặt `MQTT_BROKER`. Các thí nghiệm trong loạt bài này luôn đặt `MQTT_BROKER=127.0.0.1` và cổng riêng (hoặc cổng 1 để nối thất bại ngay) trước khi import, để test không gửi gì ra Internet. Thiết kế dễ test hơn là gom phần nối vào một hàm gọi trong `if __name__ == "__main__"` hoặc lúc khởi động máy chủ.

---

## 8. Tổng kết

### 8.1 Ba client Python của dự án, đặt cạnh nhau
| | Webhook | Kiosk | API |
|---|---|---|---|
| Vai trò | Phát `payment/success` | Nhận thanh toán; phát đơn + kho | Phát kho + đơn đã xong |
| Kết nối | Mỗi giao dịch | Lâu dài | Lâu dài |
| Nối lúc đầu | `connect()` | `connect_async()` | `connect()` lúc import |
| Tự nối lại | Không cần | Có | Có — **nếu lần đầu thành công** |
| QoS | 1 | 1 | 0 |
| Retain | Không | Kho: có; đơn: không | Kho: có; đơn xong: không |
| Kiểm kết quả phát | `wait_for_publish` (chưa kiểm `is_published`) | `rc` | Không |
| Không có khóa HMAC | Không phát | Không phát | Không phát |

### 8.2 Nguyên tắc thấy được
- **Ghi CSDL trước, phát sau**; phát thất bại không làm hỏng thao tác chính.
- **Chọn QoS theo hậu quả mất tin**; mọi tin của API đều có đường HTTP bù.
- **Retain cho trạng thái, không retain cho sự kiện.**
- Không phát tin chưa ký.

### 8.3 Ghi chú (không gây lỗi hiện tại, **chưa sửa**)
1. **Broker không có đúng lúc API khởi động → API im lặng MQTT tới khi khởi động lại** (§5.2; đã ghi nhận từ buổi học API, nay chứng minh bằng thí nghiệm). Bếp vẫn đúng nhờ HTTP 30 s.
2. **Không kiểm kết quả `publish`** → không có dòng log nào khi phát thất bại.
3. **Kết nối broker xảy ra lúc import** → test phải đặt biến môi trường trước khi import (§7).
4. **ACL Mosquitto chưa có `kitchen/order_done`** cho user `api` (ghi) và `kds` (đọc) — chỉ ảnh hưởng khi triển khai Mosquitto nội bộ (đang hoãn).

---

## 9. Câu hỏi tự kiểm tra

1. Vì sao API phải tự phát `kitchen/inventory` khi kiosk đã phát sau mỗi lần bán?
2. Đoạn kết nối MQTT chạy khi nào? Vì sao chỉ một lần?
3. Nếu broker không có lúc API khởi động, chuyện gì xảy ra với MQTT của API trong suốt tiến trình đó? Chỉ ra dòng code gây ra điều này.
4. Vì sao `Restart=on-failure` của systemd không cứu được tình huống ở câu 3?
5. Kiosk và API cùng phát retain lên `kitchen/inventory`. Có mâu thuẫn không? Màn hình bếp mới bật nhận được bản của ai?
6. API phát với QoS mấy? Vì sao chấp nhận được với tin kho và tin đơn xong, nhưng webhook thì phải dùng QoS 1?
7. Tin QoS 0 phát lúc mất kết nối và tin QoS 1 phát lúc mất kết nối — paho xử lý khác nhau thế nào? (đối chiếu thí nghiệm của `bridge_mqtt.md` §6.3)
8. Khi nào `publish_order_done` **không** được gọi dù có yêu cầu `POST /api/orders/done`?
9. Màn hình bếp tự bấm “Hoàn thành” rồi nhận lại chính tin `kitchen/order_done` của đơn đó. Có hại không? Có ích khi nào?
10. Vì sao `try/except` quanh `publish` không phát hiện được việc tin không đi?
11. Hai luồng Waitress cùng gọi `_mqtt_client.publish` có an toàn không?
12. Đề xuất một cách sửa để API tự hồi phục MQTT khi broker có mặt muộn (gợi ý: so sánh với `bridge.py`).

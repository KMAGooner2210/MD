# Giải thích chi tiết `servers/webhook_server.py`

> File: `Smart_Kiosk_Backup/servers/webhook_server.py` — 139 dòng, bản trên GitHub `main`.
> Chạy trên Pi bằng `deploy/pi/kiosk-webhook.service` (cổng **5001**).
> Thí nghiệm chạy thật bằng **Flask test client** với **token thử** `token-thu`, **không nạp `.env` thật**, MQTT trỏ vào địa chỉ không tồn tại (hoặc thay bằng hàm giả) → không phát tín hiệu thanh toán thật nào.

---

## 0. Bức tranh tổng thể (đọc trước)

### 0.1 Webhook là gì?
Có hai cách để biết "tiền đã về chưa":

| Cách | Ai chủ động | Ví dụ đời thường | Trong đồ án |
|---|---|---|---|
| ***Polling*** (hỏi định kỳ) | **Mình** hỏi bên kia | Cứ 15 phút gọi điện hỏi bưu cục "hàng tới chưa?" | Kiosk hỏi API SePay mỗi 15 s (`bridge.py` §6) |
| ***Webhook*** (gọi ngược) | **Bên kia** báo cho mình | Để lại số điện thoại, hàng tới bưu cục **tự gọi** | **File này**: SePay gọi vào khi có giao dịch |

**Webhook** = một **địa chỉ HTTP** mình đăng ký với dịch vụ bên ngoài (SePay); khi có sự kiện (tiền vào tài khoản), **dịch vụ đó gửi yêu cầu `POST`** chứa thông tin giao dịch tới địa chỉ này. Ưu điểm: **nhanh** (vài giây), không phải hỏi liên tục.

### 0.2 Vị trí trong hệ thống
```
Khách chuyển khoản ──► Ngân hàng ──► SePay (theo dõi tài khoản)
                                        │ POST https://<địa chỉ công khai>/sepay_webhook
                                        │ Authorization: Apikey <token>
                                        ▼  (Internet → đường hầm → Pi:5001)
                         ┌───────── webhook_server.py (Pi) ─────────┐
                         │ ① kiểm tra token  ② đọc JSON             │
                         │ ③ chỉ tiền VÀO    ④ tìm mã DH + 8 số     │
                         │ ⑤ ký HMAC → publish MQTT payment/success │
                         └──────────────────────────────────────────┘
                                        │ MQTT (broker)
                                        ▼
                         kiosk bridge.py: kiểm chữ ký → _finalize_payment → chốt đơn, trừ kho
```

**Điểm thiết kế quan trọng:** webhook **không đụng CSDL** — không import `database`, không chốt đơn, không trừ kho. Nó chỉ **"chuyển lời"**: nhận HTTP từ SePay → phát tin MQTT có chữ ký. Lý do: đây là máy chủ **mở ra Internet** (SePay ở ngoài phải gọi vào được) → càng ít quyền càng an toàn. Mọi quyết định về đơn hàng nằm ở **một chỗ duy nhất**: kiosk.

**Làm sao SePay (trên Internet) gọi được vào Pi (trong Wi-Fi cửa hàng)?** Pi không có địa chỉ công khai, nên cần một ***đường hầm*** (tunnel) từ Internet về cổng 5001 của Pi. Trên Pi dùng dịch vụ đường hầm ngrok (`kiosk-ngrok.service`, hiện **chưa có trong repo** — xem bảng ở `buoi_03_http_flask_api.md`).

### 0.3 Kết quả chạy thử
```
GET  /                                     -> 200 Webhook Server is running.
GET  /sepay_webhook (sai phương thức)      -> 405 (trang HTML)
POST không có Authorization                -> 401 Unauthorized
POST token sai                             -> 401 Unauthorized
POST body không phải JSON                  -> 400 Invalid JSON
POST đúng token, MQTT không tới được       -> 502 Không chuyển tiếp được tín hiệu
POST "Apikey <token>" (MQTT giả lập)       -> 200 Đã xử lý, order_code=DH12345678
POST token trần không tiền tố              -> 200 Đã xử lý
POST transferType=out                      -> 200 Bỏ qua giao dịch tiền ra
POST nội dung không có mã đơn              -> 200 Không phải đơn hàng Kiosk
```

---

## 1. Import và cấu hình (L1–34)

```python
import re
import os
import sys
import hmac
from flask import Flask, request, jsonify
import paho.mqtt.client as mqtt
from dotenv import load_dotenv

current_dir = os.path.dirname(os.path.abspath(__file__))
project_root = os.path.dirname(current_dir)
if project_root not in sys.path:
    sys.path.append(project_root)

from backend.mqtt_security import configure_client, sign_message

load_dotenv(os.path.join(project_root, ".env"), override=True)

app = Flask(__name__)

MQTT_BROKER = os.environ.get("MQTT_BROKER", "broker.emqx.io")
MQTT_PORT = int(os.environ.get("MQTT_PORT", 1883))
MQTT_TOPIC = "payment/success"
MQTT_SHARED_SECRET = os.environ.get("MQTT_SHARED_SECRET", "")

SEPAY_WEBHOOK_AUTH_TOKEN = os.environ.get("SEPAY_WEBHOOK_AUTH_TOKEN", "")

ORDER_CODE_PATTERN = re.compile(r"DH\d{8}")
```

| Thành phần | Ý nghĩa |
|---|---|
| `re` | **Biểu thức chính quy** (regex) — tìm mã đơn trong nội dung chuyển khoản |
| `hmac` | Dùng `compare_digest` để so token (thời gian không đổi) |
| Chỉ import `configure_client`, `sign_message` | Webhook chỉ **ký** tin, không bao giờ **kiểm** tin (nó không nhận MQTT) |
| **Không** import `backend.database` | Webhook **không có quyền** đọc/ghi CSDL |
| `override=True` | Giá trị trong `.env` ghi đè biến môi trường có sẵn (giống `bridge.py`, khác `api_server.py`) |
| `SEPAY_WEBHOOK_AUTH_TOKEN` | **Bí mật chung** giữa SePay và webhook — đặt trong cấu hình webhook trên trang SePay và trong `.env` |

### Biểu thức chính quy `r"DH\d{8}"`
| Phần | Nghĩa |
|---|---|
| `r"..."` | **Chuỗi thô** (*raw string*): `\` giữ nguyên, không bị Python hiểu là ký tự thoát — viết regex luôn nên dùng |
| `DH` | Hai chữ **D, H** in hoa, đúng như vậy |
| `\d` | Một **chữ số** 0–9 |
| `{8}` | Lặp **đúng 8 lần** |
| `re.compile(...)` | Biên dịch regex **một lần** lúc nạp module, dùng lại nhiều lần → nhanh hơn |

Thử thật với `.search()` (tìm **ở bất kỳ đâu** trong chuỗi):
```
'DH12345678'          -> DH12345678
'abcDH12345678xyz'    -> DH12345678   ← nằm giữa chữ khác vẫn tìm được (ngân hàng hay chèn thêm mã giao dịch)
'DH123456789'         -> DH12345678   ← 9 số: lấy 8 số đầu (regex không có ranh giới từ)
'dh12345678'          -> None         ← phân biệt hoa/thường
'DH 12345678'         -> None         ← có dấu cách ở giữa
'DH1234567'           -> None         ← thiếu số
```
Mã đơn kiosk sinh ra luôn đúng `DH` + **8 số** (`bridge.py` §13.2) → khớp chính xác. Trường hợp 9 số không xảy ra với mã thật; kể cả nếu có, kiosk vẫn **đối chiếu với đơn đang chờ** nên không chốt nhầm.

---

## 2. `send_mqtt_signal()` — chuyển tiếp sang MQTT (L37–67)

```python
def send_mqtt_signal(order_code, amount):
    """Phát tín hiệu thanh toán đã KÝ HMAC; thiếu MQTT_SHARED_SECRET thì không phát."""
    if not MQTT_SHARED_SECRET:
        print(" ---[BẢO MẬT] Chưa cấu hình MQTT_SHARED_SECRET: không phát tín hiệu thanh toán.")
        return False
    try:
        try:
            client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
        except AttributeError:
            client = mqtt.Client()
        configure_client(client, role="WEBHOOK")
        client.connect(MQTT_BROKER, MQTT_PORT, 60)
        client.loop_start()
        message = sign_message(
            {"event": "payment.success", "order_code": order_code, "amount": amount, "status": "PAID"},
            MQTT_SHARED_SECRET,
        )
        info = client.publish(MQTT_TOPIC, message, qos=1)
        info.wait_for_publish(timeout=5)
        client.loop_stop()
        client.disconnect()
        return True
    except Exception as e:
        print(f" ---Lỗi kết nối MQTT: {e}")
        return False
```

### Từng bước
| Bước | Ý nghĩa |
|---|---|
| Thiếu khóa → `False` | **Không bao giờ phát tin chưa ký** (kiosk cũng sẽ từ chối) |
| `try … except AttributeError` | Chạy được với paho-mqtt 1.x và 2.x |
| `configure_client(role="WEBHOOK")` | Tài khoản `MQTT_WEBHOOK_USERNAME`/`MQTT_USERNAME`, TLS nếu bật |
| `connect(...)` | Kết nối **chặn** (chờ xong); broker không tới được → ném lỗi → `False` |
| `loop_start()` | Luồng nền của paho lo gửi/nhận — **cần có** để tin `qos=1` thật sự được gửi đi |
| `sign_message({...})` | Nội dung: `event`, `order_code`, `amount`, `status`; tự thêm `ts` và chữ ký |
| `publish(..., qos=1)` | **At least once**: broker xác nhận đã nhận (gói PUBACK) |
| `wait_for_publish(timeout=5)` | **Chờ tối đa 5 s** tới khi tin gửi xong → bảo đảm tin đã đi trước khi ngắt kết nối |
| `loop_stop()`, `disconnect()` | Dọn dẹp |
| Trả `True/False` | Endpoint đổi thành HTTP 200 / 502 |

### Mỗi giao dịch một kết nối MQTT mới
Hàm **mở kết nối → gửi → đóng** cho **mỗi** lần SePay gọi. Đây là lựa chọn **đơn giản**: không phải giữ và theo dõi kết nối lâu dài, không lo kết nối "chết ngầm". Cái giá: mỗi giao dịch tốn thêm thời gian bắt tay với broker (thường dưới 1 giây) — chấp nhận được vì một cửa hàng chỉ có vài trăm giao dịch/ngày.
(So sánh: kiosk và API giữ **một kết nối lâu dài** vì gửi/nhận liên tục.)

### Vì sao bắt `Exception` chung?
Ở đây mọi lỗi (mạng, broker từ chối, hết thời gian chờ) đều có **cùng hậu quả**: không chuyển tiếp được → trả `False` → HTTP 502. Không cần phân biệt từng loại.

---

## 3. `_is_request_authorized()` — xác thực SePay (L70–81)

```python
def _is_request_authorized(req):
    """Fail-closed: chưa cấu hình token thì từ chối mọi request."""
    if not SEPAY_WEBHOOK_AUTH_TOKEN:
        return False
    auth_header = req.headers.get("Authorization", "").strip()
    # SePay gửi dạng "Apikey <token>"; vẫn chấp nhận token trần cho tương thích.
    if auth_header.lower().startswith("apikey "):
        auth_header = auth_header[len("apikey "):].strip()
    return hmac.compare_digest(
        auth_header.encode("utf-8"),
        SEPAY_WEBHOOK_AUTH_TOKEN.encode("utf-8"),
    )
```

**Vì sao cần xác thực?** Địa chỉ webhook nằm trên Internet. **Ai biết địa chỉ** cũng gửi được `POST` giả: *"DH12345678 đã nhận 70.000 đ"*. Webhook ký HMAC rồi phát MQTT → kiosk thấy chữ ký **đúng** (vì chính webhook ký) → **chốt đơn không có tiền**. → Cửa vào này phải được khóa: chỉ yêu cầu mang **đúng token bí mật** mới được xử lý.

| Dòng | Cú pháp / ý nghĩa |
|---|---|
| `if not SEPAY_WEBHOOK_AUTH_TOKEN: return False` | **Fail-closed**: chưa cấu hình → không ai qua được (tránh `""` khớp với header rỗng) |
| `req.headers.get("Authorization", "")` | Header chuẩn HTTP dùng cho xác thực; thiếu → `""` |
| `.lower().startswith("apikey ")` | SePay gửi `Authorization: Apikey <token>`; so **không phân biệt hoa thường** |
| `auth_header[len("apikey "):]` | **Cắt chuỗi** bỏ 7 ký tự đầu → còn token. Dùng `len(...)` thay vì gõ số 7 → tự đúng nếu đổi tiền tố |
| Vẫn nhận token trần | Tương thích cách cấu hình cũ / công cụ thử (`test_fake_pay.py`) |
| `hmac.compare_digest(bytes, bytes)` | So sánh **thời gian không đổi** → không dò token bằng đo thời gian; đổi sang `bytes` để không lỗi khi header có ký tự lạ |

**Tham số `req`** thay vì dùng thẳng biến toàn cục `request` → hàm nhận đối tượng yêu cầu từ ngoài vào, dễ đọc và dễ thử.

**Vì sao token xác thực ở webhook là đủ, khi kiosk còn kiểm thêm chữ ký MQTT?** Hai lớp bảo vệ **hai đoạn đường khác nhau**:
| Đoạn đường | Kẻ gian có thể | Lớp chặn |
|---|---|---|
| Internet → webhook (HTTP) | Gửi `POST` giả vào webhook | **Token SePay** (file này) |
| Webhook → broker công cộng → kiosk (MQTT) | Publish tin giả thẳng lên broker, bỏ qua webhook | **Chữ ký HMAC** (`mqtt_security.py`) |

Thiếu một lớp là hở một đường.

---

## 4. Các route (L84–128)

### 4.1 `GET /` — kiểm tra sống
```python
@app.route('/', methods=['GET'])
def home():
    return "Webhook Server is running."
```
- Trả **chuỗi thuần** (Flask tự đóng gói thành phản hồi `200`, `Content-Type: text/html`).
- Dùng để kiểm tra nhanh máy chủ và đường hầm còn chạy (mở địa chỉ công khai trên trình duyệt). Không lộ thông tin gì.

### 4.2 `POST /sepay_webhook` — nhận giao dịch
```python
@app.route('/sepay_webhook', methods=['POST'])
def sepay_webhook():
    if not SEPAY_WEBHOOK_AUTH_TOKEN:
        return jsonify({"success": False, "message": "Webhook not configured"}), 503

    if not _is_request_authorized(request):
        return jsonify({"success": False, "message": "Unauthorized"}), 401

    data = request.get_json(silent=True)
    if not isinstance(data, dict):
        return jsonify({"success": False, "message": "Invalid JSON"}), 400
```
**Thứ tự kiểm tra:** cấu hình → **xác thực** → định dạng dữ liệu. Xác thực **trước khi đọc body** → kẻ gian không có token thì không chạm được tới phần xử lý dữ liệu.

| Tình huống | Mã | Vì sao mã này |
|---|---|---|
| Máy chủ chưa đặt token | **503** | Lỗi **phía máy chủ** (thiếu cấu hình), không phải lỗi người gọi |
| Token sai / thiếu | **401** | Chưa xác thực |
| Body không phải JSON object | **400** | Dữ liệu gửi lên sai |

```python
    try:
        content = " ".join(
            str(data.get(field) or "")
            for field in ("content", "description", "transferContent")
        )
        amount = data.get("transferAmount", 0)
        transfer_type = str(data.get("transferType", "in") or "in").lower()
        if transfer_type != "in":
            return jsonify({"success": True, "message": "Bỏ qua giao dịch tiền ra"}), 200
```
**Dữ liệu SePay gửi** (các trường file này dùng):
| Trường | Ý nghĩa |
|---|---|
| `content` | Nội dung chuyển khoản (chứa mã đơn) |
| `description` | Nội dung đầy đủ từ ngân hàng (dự phòng) |
| `transferContent` | Dự phòng thêm |
| `transferAmount` | Số tiền |
| `transferType` | `"in"` (tiền vào) / `"out"` (tiền ra) |

- `" ".join(... for field in (...))` → **gộp 3 trường** thành một chuỗi rồi tìm mã trong đó → ngân hàng nào đặt mã ở trường nào cũng tìm được.
- `str(data.get(field) or "")` → trường thiếu hoặc `null` → `""`; trường là số → đổi thành chuỗi.
- `data.get("transferType", "in") or "in"` → thiếu **hoặc** `null`/rỗng đều coi là `"in"`.
- **Bỏ qua tiền ra** nhưng vẫn trả **200 `success: true`**: SePay theo dõi **mọi** giao dịch của tài khoản (kể cả chủ quán chuyển tiền đi) → đó không phải lỗi, chỉ là "không liên quan" → báo đã nhận để SePay không coi là gửi thất bại.

```python
        match = ORDER_CODE_PATTERN.search(content)
        if match:
            order_code = match.group(0)
            sent = send_mqtt_signal(order_code, amount)
            if not sent:
                return jsonify({"success": False, "message": "Không chuyển tiếp được tín hiệu"}), 502
            return jsonify({"success": True, "message": "Đã xử lý", "order_code": order_code}), 200
        print("--- Nội dung không khớp định dạng mã đơn hàng. Bỏ qua.")
        return jsonify({"success": False, "message": "Không phải đơn hàng Kiosk"}), 200
    except Exception as e:
        print(f"--- Lỗi xử lý webhook: {e}")
        return jsonify({"success": False}), 500
```
| Cú pháp | Ý nghĩa |
|---|---|
| `pattern.search(chuỗi)` | Tìm **lần khớp đầu tiên** ở bất kỳ vị trí nào; không có → `None` |
| `match.group(0)` | Toàn bộ đoạn khớp (`"DH12345678"`) |

**Các kết quả và ý nghĩa mã HTTP:**
| Tình huống | Mã | Lý do |
|---|---|---|
| Có mã đơn, đã phát MQTT | **200** | Thành công |
| Có mã đơn, **không phát được** MQTT | **502 Bad Gateway** | Webhook nhận đúng nhưng **dịch vụ phía sau** (broker) lỗi → báo SePay là **chưa xử lý xong** |
| Không có mã đơn (chuyển khoản thường) | **200** `success: false` | Đã nhận, chỉ là không phải đơn kiosk → không phải lỗi, không cần gửi lại |
| Tiền ra | **200** | Như trên |
| Lỗi bất ngờ | **500** | Lỗi máy chủ |

**Vì sao phân biệt 502 với 200?** Bên gửi webhook nhìn **mã trạng thái** để biết lần gửi có thành công không. Trả 502 khi MQTT lỗi → SePay ghi nhận lần gọi **thất bại** (xem được trong nhật ký webhook trên trang SePay, có thể gửi lại). Nếu trả 200 thì tín hiệu bị **mất im lặng**. Dù vậy, kiosk vẫn còn **đường dự phòng**: tự tra cứu SePay mỗi 15 s → đơn vẫn được chốt.

**Webhook không kiểm tra số tiền** có đủ không — chỉ chuyển nguyên `amount`. Việc so với giá trị đơn do **kiosk** làm (`_finalize_payment`), vì chỉ kiosk biết đơn đang chờ bao nhiêu tiền. Một nơi quyết định → không có hai bộ luật lệch nhau.

### 4.3 Ghi chú cuối file
```python
# Đã bỏ endpoint /api/stats khỏi webhook: máy chủ webhook được mở ra Internet
# cho SePay gọi, không được lộ doanh thu. Thống kê lấy qua Smart Kiosk API.
```
Trước đây webhook có route `/api/stats` **không cần khóa** → ai biết địa chỉ công khai cũng xem được doanh thu. Đã gỡ: máy chủ mở ra Internet chỉ giữ **đúng những gì SePay cần** (nguyên tắc ***thu nhỏ bề mặt tấn công***).

---

## 5. Chạy máy chủ (L135–139) và dịch vụ systemd

```python
if __name__ == '__main__':
    print("--- WEBHOOK SERVER ĐANG CHẠY TRÊN CỔNG 5001 ---")
    if not SEPAY_WEBHOOK_AUTH_TOKEN:
        print("--- [CẢNH BÁO KHỞI ĐỘNG] Chưa cấu hình SEPAY_WEBHOOK_AUTH_TOKEN trong .env ---")
    app.run(host='0.0.0.0', port=int(os.environ.get('WEBHOOK_PORT', '5001')))
```
- Cổng mặc định **5001**, đổi được bằng `WEBHOOK_PORT`.
- `host='0.0.0.0'` → nghe mọi card mạng (đường hầm, công cụ thử từ laptop gọi vào được).
- Dùng **máy chủ phát triển** của Flask (`app.run`) — khác API dùng Waitress. Đủ cho lưu lượng rất thấp (vài trăm yêu cầu/ngày); Flask hiện đại mặc định xử lý mỗi yêu cầu trên một luồng riêng.

`deploy/pi/kiosk-webhook.service`:
```ini
[Unit]
Description=Smart Kiosk webhook SePay (Flask, cong 5001)
After=network-online.target
Wants=network-online.target

[Service]
User=__APP_USER__
WorkingDirectory=__APP_DIR__
ExecStart=__APP_DIR__/.venv/bin/python servers/webhook_server.py
Restart=on-failure
RestartSec=3
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```
| Dòng | Ý nghĩa |
|---|---|
| `After=` / `Wants=network-online.target` | Chờ **có mạng** rồi mới chạy |
| `User=`, `WorkingDirectory=` | Chạy bằng tài khoản thường (không phải root), trong thư mục dự án. `__APP_USER__`, `__APP_DIR__` được `install_pi.sh` thay bằng giá trị thật (lệnh `sed`) |
| `ExecStart=.venv/bin/python ...` | Dùng Python của **môi trường ảo** dự án (đủ thư viện) |
| `Restart=on-failure`, `RestartSec=3` | Chương trình chết vì lỗi → **tự chạy lại sau 3 s** |
| `PYTHONUNBUFFERED=1` | `print` ra ngay, không bị giữ trong bộ đệm → `journalctl -u kiosk-webhook` thấy nhật ký tức thì |
| `WantedBy=multi-user.target` | Tự chạy khi Pi khởi động |

Xem nhật ký: `journalctl -u kiosk-webhook -f`.

---

## 6. Thử thanh toán giả: `tests/test_fake_pay.py`

Script chạy tay **mô phỏng SePay**: hỏi mã đơn đang hiện trên kiosk và số tiền → gửi `POST` (kèm token đọc từ `.env`) tới `http://127.0.0.1:5001/sepay_webhook` (hoặc `SEPAY_WEBHOOK_TEST_URL`). Dùng để thử toàn bộ chuỗi **webhook → MQTT → kiosk chốt đơn** mà không cần chuyển tiền thật. (Tên bắt đầu `test_` nhưng **không phải** bài pytest — nó hỏi nhập từ bàn phím.)

---

## 7. Tổng kết

### 7.1 Đường đi của một giao dịch
```
SePay POST /sepay_webhook
 → chưa cấu hình token?            503
 → Authorization sai?              401
 → body không phải JSON object?    400
 → tiền ra?                        200 (bỏ qua)
 → không có DH + 8 số?             200 success:false (bỏ qua)
 → ký HMAC + publish MQTT
     lỗi                           502
     thành công                    200 success:true
 → (kiosk) kiểm chữ ký, tuổi tin, đúng đơn, đủ tiền → chốt PAID + trừ kho
```

### 7.2 Nguyên tắc thiết kế
1. **Quyền tối thiểu**: máy chủ mở ra Internet **không chạm CSDL**, chỉ chuyển lời.
2. **Fail-closed**: thiếu token → 503; thiếu khóa MQTT → không phát.
3. **Hai lớp cho hai đoạn đường**: token SePay (HTTP) + chữ ký HMAC (MQTT).
4. **Mã HTTP có nghĩa**: 200 cho "đã nhận" (kể cả bỏ qua), 502 cho "không chuyển tiếp được".
5. **Một nơi quyết định**: số tiền, trạng thái đơn do kiosk xét.
6. **Thu nhỏ bề mặt tấn công**: đã gỡ `/api/stats`.

### 7.3 Ghi chú (không gây lỗi hiện tại)
| Điểm | Ghi chú |
|---|---|
| Lỗi 404/405 trả **trang HTML** | Webhook không có `errorhandler` như API; SePay chỉ gọi đúng route nên không ảnh hưởng |
| Regex phân biệt hoa/thường, không có ranh giới từ | Mã kiosk luôn `DH` + 8 số; kiosk đối chiếu lại với đơn đang chờ |
| Chạy bằng máy chủ phát triển Flask | Đủ cho lưu lượng thấp |
| `kiosk-ngrok.service` chưa có trong repo | Đường hầm cấu hình trực tiếp trên Pi |

---

## 8. Câu hỏi tự kiểm tra
1. Webhook khác polling thế nào? Đồ án dùng cả hai để làm gì?
2. Vì sao webhook không import `backend.database`?
3. Nếu bỏ kiểm tra `Authorization`, kẻ gian làm gì được? Chữ ký MQTT có cứu được không? Vì sao?
4. Vì sao kiểm tra token **trước** khi đọc body JSON?
5. `r"DH\d{8}"` khớp những chuỗi nào trong: `"CK DH12345678"`, `"dh12345678"`, `"DH 12345678"`?
6. Vì sao chuyển khoản không có mã đơn lại trả **200** chứ không phải 400?
7. Vì sao MQTT lỗi thì trả **502** chứ không phải 200 hay 500? Khi đó đơn có bị mất không?
8. `wait_for_publish(timeout=5)` để làm gì? Bỏ đi thì chuyện gì có thể xảy ra?
9. Webhook có kiểm tra số tiền đủ không? Ai làm việc đó, vì sao?
10. `Restart=on-failure` và `PYTHONUNBUFFERED=1` trong file dịch vụ có tác dụng gì?

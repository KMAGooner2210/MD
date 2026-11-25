# Giải thích chi tiết `backend/mqtt_security.py`

> File: `Smart_Kiosk_Backup/backend/mqtt_security.py` — 135 dòng, bản trên GitHub `main`.
> Thí nghiệm dưới đây chạy thật với **khóa thử** `"khoa-thu-nghiem"` (không phải khóa thật trong `.env`).
> Kiểm thử: `tests/test_mqtt_security.py` + `tests/test_bridge_payment_security.py` — **18/18 đạt**.

---

## 0. File này giải quyết vấn đề gì? (đọc trước)

### 0.1 Bối cảnh: MQTT trong đồ án
***MQTT*** là giao thức nhắn tin kiểu **"bảng tin"**: bên gửi **đăng (publish)** thông điệp lên một ***chủ đề (topic)***, ai **đăng ký (subscribe)** chủ đề đó thì nhận được. Ở giữa là ***broker*** — máy chủ chuyển tin.

| Chủ đề | Ai gửi | Ai nhận | Nội dung |
|---|---|---|---|
| `payment/success` | Webhook (nhận báo từ SePay) | **Kiosk** | "Đơn DH… đã trả 70.000 đ" → kiosk chốt đơn, trừ kho |
| kitchen (đơn mới) | Kiosk | Màn hình bếp | Đơn cần làm |
| inventory (tồn kho) | Kiosk, API | Màn hình bếp | Số lượng nguyên liệu |
| `kitchen/order_done` | API | Màn hình bếp | Gỡ thẻ đơn đã xong |

### 0.2 Lỗ hổng nếu không ký
Đồ án dùng **broker công cộng** (`broker.emqx.io` — mặc định trong `webhook_server.py` L27): **ai trên Internet cũng kết nối và publish được** vào bất kỳ chủ đề nào. Kẻ gian chỉ cần biết tên chủ đề và gửi:
```json
{"order_code": "DH12345678", "status": "PAID"}
```
→ kiosk tưởng khách đã trả tiền → **chốt đơn, trừ kho, bếp làm món — không ai trả tiền**.

### 0.3 Giải pháp: ký số bằng HMAC
Hình dung **con dấu của cửa hàng**:
- Chỉ webhook, kiosk, API và màn hình bếp có **khóa bí mật chung** (`MQTT_SHARED_SECRET`) — như con dấu cất trong két.
- Bên gửi **đóng dấu** (tính chữ ký từ nội dung + khóa) lên mỗi thông điệp.
- Bên nhận **tự đóng dấu lại** trên nội dung nhận được và **so với dấu đi kèm**. Khớp → thật; lệch → giả hoặc đã bị sửa.
- Kẻ gian thấy được thông điệp (broker công cộng) nhưng **không có khóa** → không làm ra được chữ ký đúng.

Kết quả chạy thật:
```
Sửa amount 70000 -> 1000, giữ chữ ký cũ    -> TỪ CHỐI: bad_signature
Ký bằng khóa khác                          -> TỪ CHỐI: bad_signature
Gửi lại sau 121 giây (phát lại)            -> TỪ CHỐI: stale_message
Dấu thời gian ở tương lai +300 s           -> TỪ CHỐI: stale_message
Thông điệp cũ không ký {"status":"PAID"}   -> TỪ CHỐI: unsupported_envelope
```

### 0.4 Ba mối đe dọa và cách chặn
| Mối đe dọa | Ví dụ | Chặn bằng |
|---|---|---|
| **Giả mạo** (*spoofing*) | Tự tạo tin "DH… PAID" | Chữ ký HMAC — không có khóa thì không ký được |
| **Sửa nội dung** (*tampering*) | Đổi `amount` 70000 → 1000 | Chữ ký tính trên toàn bộ nội dung — sửa 1 ký tự là chữ ký lệch |
| **Phát lại** (*replay*) | Chép một tin thật cũ, gửi lại hôm sau | Dấu thời gian `ts` nằm **trong** phần được ký + giới hạn tuổi 120 s |

---

## 1. Docstring đầu file — "đặc tả" định dạng (L1–17)

```
Định dạng thông điệp (phiên bản 2):
    {"v": 2, "sig": "<HMAC-SHA256 hex>", "payload": "<chuỗi JSON nội dung>"}
```
Mỗi thông điệp là một **phong bì** (*envelope*) gồm 3 trường:

| Trường | Ý nghĩa |
|---|---|
| `v` | **Phiên bản** định dạng = 2. Bên nhận thấy số khác → từ chối (chặn luôn tin cũ không ký "phiên bản 1") |
| `sig` | Chữ ký: 64 ký tự hex |
| `payload` | Nội dung — **một CHUỖI** JSON (không phải đối tượng JSON lồng) |

Thông điệp thật chạy ra:
```json
{"v":2,"sig":"b4644b1877efb6a974bf9144959954c027a4343efeeeb2a8e672aade4fd4599c",
 "payload":"{\"order_code\":\"DH12345678\",\"amount\":70000,\"status\":\"PAID\",\"ts\":1764000000}"}
```

**Vì sao `payload` là chuỗi chứ không phải object?** (ý quan trọng nhất của thiết kế)

Chữ ký tính trên **dãy byte cụ thể**. Cùng một nội dung JSON có thể viết thành nhiều chuỗi khác nhau:
```
{"a": 1, "b": 2}     {"a":1,"b":2}     {"b":2,"a":1}      ← cùng nghĩa, KHÁC byte → khác chữ ký
```
Nếu `payload` là object, bên nhận phải **dựng lại chuỗi JSON** rồi tính chữ ký → Python và ESP32 (thư viện ArduinoJson) có thể dựng **khác nhau** (khoảng trắng, thứ tự khóa, cách viết dấu tiếng Việt) → chữ ký lệch dù tin thật.
Giữ `payload` nguyên dạng **chuỗi** → bên nhận tính HMAC trên **đúng chuỗi nhận được**, không dựng lại gì → **Python và firmware C++ luôn khớp**. Docstring ghi: *"firmware ESP32 kiểm tra được mà không cần sắp xếp JSON"*.

Các dòng còn lại của docstring:
- *"Khóa bí mật KHÔNG bao giờ nằm trong thông điệp"* — chữ ký chứng minh **có** khóa mà không để lộ khóa.
- *"Thiếu khóa thì không ký, không chấp nhận (fail-closed)"* — quên cấu hình thì **hệ thống từ chối hết**, không phải **cho qua hết**.
- Phần đăng nhập broker và TLS (phần 7).

---

## 2. Import và hằng số (L19–33)

```python
from __future__ import annotations

import hashlib
import hmac
import json
import os
import time
from typing import Any

ENVELOPE_VERSION = 2
DEFAULT_MAX_AGE_SECONDS = 120


class MqttMessageRejected(ValueError):
    """Thông điệp MQTT không hợp lệ hoặc không xác thực được."""
```

| Dòng | Ý nghĩa |
|---|---|
| `from __future__ import annotations` | Cho phép viết **chú thích kiểu** kiểu mới (`str \| None`, `dict[str, Any]`) trên Python cũ hơn mà không lỗi; chú thích không bị tính lúc chạy. Phải đứng **đầu file** (sau docstring) |
| `hashlib` | Các hàm băm: `hashlib.sha256` |
| `hmac` | Tính HMAC và so sánh an toàn `compare_digest` |
| `time` | `time.time()` → số giây Unix hiện tại |
| `typing.Any` | Kiểu "bất kỳ" trong chú thích |
| `class MqttMessageRejected(ValueError)` | **Lớp lỗi riêng**, kế thừa `ValueError`. Thân lớp chỉ có docstring (hợp lệ — không cần `pass`) |

**Vì sao tạo lớp lỗi riêng?** Kiosk viết `except MqttMessageRejected as rejected:` → bắt **đúng** lỗi "thông điệp bị từ chối" (in cảnh báo bảo mật rồi bỏ qua), còn lỗi lập trình khác (`KeyError`, `AttributeError`) vẫn **lộ ra** chứ không bị nuốt nhầm. Nội dung lỗi là **mã lý do ngắn** (`"bad_signature"`, `"stale_message"`) → dễ đọc nhật ký, dễ kiểm thử.

---

## 3. HMAC-SHA256 là gì?

### 3.1 Hàm băm SHA-256
Biến **dữ liệu bất kỳ** thành **32 byte** (64 ký tự hex):
- **Một chiều**: từ kết quả không suy ngược ra dữ liệu.
- **Thay đổi nhỏ → kết quả khác hoàn toàn** (chạy thật, chỉ đổi `DH12345678` → `DH12345679`):
  ```
  b4644b1877efb6a974bf9144959954c0...
  f555e0e77cd8440d0119d65b375f99d8...
  ```

### 3.2 Vì sao không chỉ băm nội dung?
Băm SHA-256 thì **ai cũng tính được** → kẻ gian sửa nội dung rồi tự băm lại là xong. Cần thứ mà **chỉ người có khóa** tính được.

### 3.3 HMAC = băm có khóa
```
HMAC-SHA256(khóa, nội_dung) = SHA256( (khóa ⊕ opad) ‖ SHA256( (khóa ⊕ ipad) ‖ nội_dung ) )
```
Không cần nhớ công thức — chỉ cần nhớ: **kết quả phụ thuộc cả khóa lẫn nội dung**; không có khóa thì không tính được, và cũng không suy được khóa từ chữ ký. (Không dùng kiểu tự chế `SHA256(khóa + nội_dung)` vì có lỗ hổng đã biết — *length extension*; HMAC là chuẩn RFC 2104.)

### 3.4 HMAC khác "chữ ký số" (RSA/ECDSA) thế nào?
| | HMAC (đồ án dùng) | Chữ ký số khóa công khai |
|---|---|---|
| Khóa | **Một khóa chung** cho cả bên ký và bên kiểm | Khóa riêng để ký, khóa công khai để kiểm |
| Tốc độ | Rất nhanh, nhẹ cho ESP32 | Chậm hơn |
| Hạn chế | Ai kiểm được thì **cũng ký được** → mọi thiết bị giữ khóa phải đáng tin | Bên kiểm không ký giả được |

Đồ án chọn HMAC vì các bên (Pi, laptop, ESP32 bếp) đều là thiết bị **của cửa hàng**, và ESP32 cần tính nhanh.

---

## 4. Hai hàm nhỏ (L36–47)

```python
def _secret_bytes(secret: str | None) -> bytes:
    if not secret or not str(secret).strip():
        raise MqttMessageRejected("mqtt_secret_not_configured")
    return str(secret).encode("utf-8")
```
- `secret: str | None` → **chú thích kiểu**: tham số là chuỗi hoặc `None`. `-> bytes` → hàm trả về `bytes`. Python **không ép** kiểu lúc chạy; chú thích giúp người đọc và công cụ kiểm tra.
- Khóa `None`, `""`, hay toàn khoảng trắng `"   "` → **ném lỗi** → không bao giờ ký/kiểm bằng khóa rỗng. Đây chính là **fail-closed**: nếu cho phép khóa rỗng, ai cũng tính được HMAC với khóa rỗng → mất hết tác dụng.
- `.encode("utf-8")` → chuỗi → `bytes` (HMAC làm việc trên byte).

```python
def compute_signature(secret: str, payload_text: str) -> str:
    return hmac.new(
        _secret_bytes(secret),
        payload_text.encode("utf-8"),
        hashlib.sha256,
    ).hexdigest()
```
- `hmac.new(khóa, dữ_liệu, thuật_toán)` → tạo đối tượng HMAC.
- `.hexdigest()` → kết quả dạng **chuỗi hex chữ thường** 64 ký tự (`.digest()` sẽ ra 32 byte thô).
- Firmware tính **y hệt** bằng `mbedtls_md_hmac(...)` rồi đổi sang hex chữ thường (`kds_firmware.ino` L1477–1492).

---

## 5. `sign_message()` — ký thông điệp (L50–67)

```python
def sign_message(
    body: dict[str, Any],
    secret: str,
    *,
    now: float | None = None,
) -> str:
    """Trả về chuỗi JSON đã ký để publish. Không bao giờ chèn khóa vào nội dung."""
    if not isinstance(body, dict):
        raise TypeError("body must be a dict")
    content = {key: value for key, value in body.items() if key != "secret"}
    content["ts"] = int(time.time() if now is None else now)
    payload_text = json.dumps(content, ensure_ascii=False, separators=(",", ":"))
    signature = compute_signature(secret, payload_text)
    return json.dumps(
        {"v": ENVELOPE_VERSION, "sig": signature, "payload": payload_text},
        ensure_ascii=False,
        separators=(",", ":"),
    )
```

### Cú pháp
| Thành phần | Ý nghĩa |
|---|---|
| `dict[str, Any]` | Chú thích: dict có khóa là chuỗi, giá trị bất kỳ |
| `*` đứng một mình | Mọi tham số **sau nó bắt buộc gọi theo tên**: phải viết `sign_message(b, k, now=123)`, **không** được `sign_message(b, k, 123)` → tránh truyền nhầm vị trí |
| `now: float \| None = None` | Thời điểm ký; mặc định `None` = "bây giờ". Test truyền số cố định để kết quả **lặp lại được** |
| `{k: v for k, v in body.items() if k != "secret"}` | **Dict comprehension có điều kiện**: sao chép dict, bỏ khóa `"secret"` |
| `A if điều_kiện else B` | Biểu thức điều kiện |
| `int(...)` | Giây Unix dạng số nguyên (bỏ phần lẻ) |
| `separators=(",", ":")` | Bỏ khoảng trắng sau `,` và `:` → chuỗi **gọn nhất** |
| `ensure_ascii=False` | Giữ chữ tiếng Việt nguyên dạng |

### Logic từng bước
1. **Kiểm tra kiểu** — `body` phải là dict.
2. **Lọc trường `"secret"`** — phòng lập trình viên lỡ tay đưa khóa vào nội dung (thông điệp đi qua broker công cộng!). Chạy thật: đưa `"secret": "lo-tay"` vào → thông điệp ra **không có** trường này.
3. **Thêm dấu thời gian `ts`** — nằm **trong** phần được ký → kẻ gian **không sửa được** `ts` để làm "mới" một tin cũ.
4. **Đổi nội dung thành chuỗi** `payload_text` (gọn, giữ tiếng Việt).
5. **Ký** chuỗi đó.
6. **Đóng phong bì** `{"v", "sig", "payload"}` và trả về chuỗi JSON để publish.

Chạy thật:
```
json.dumps({"a":1,"b":2})                          → {"a": 1, "b": 2}
json.dumps({"a":1,"b":2}, separators=(",", ":"))   → {"a":1,"b":2}
```
(Bên nhận không dựng lại chuỗi nên `separators` không bắt buộc cho đúng chữ ký — chỉ để **thông điệp ngắn hơn**, có lợi cho ESP32 bộ nhớ nhỏ.)

---

## 6. `verify_message()` — kiểm tra thông điệp (L70–115)

Thứ tự kiểm tra như **cửa an ninh nhiều lớp**, hỏng lớp nào dừng ngay ở đó:
```
có khóa? → giải mã được phong bì? → đúng phiên bản 2? → có đủ payload, sig dạng chuỗi?
→ CHỮ KÝ ĐÚNG? → payload là JSON object? → có ts? → ts còn mới? → trả nội dung
```

### 6.1 Khai báo
```python
def verify_message(
    raw: bytes | str,
    secret: str,
    *,
    max_age_seconds: float | None = DEFAULT_MAX_AGE_SECONDS,
    now: float | None = None,
) -> dict[str, Any]:
    secret_bytes = _secret_bytes(secret)
```
- `raw: bytes | str` → nhận cả **byte** (thư viện paho-mqtt đưa `msg.payload` dạng byte) lẫn chuỗi.
- `max_age_seconds=120` mặc định; truyền `None` → **bỏ kiểm tra tuổi** (dùng trong test).
- Kiểm tra khóa **đầu tiên** → thiếu khóa thì từ chối mọi thứ (fail-closed).

### 6.2 Giải mã phong bì
```python
    try:
        text = raw.decode("utf-8") if isinstance(raw, (bytes, bytearray)) else str(raw)
        envelope = json.loads(text)
    except (UnicodeDecodeError, json.JSONDecodeError) as error:
        raise MqttMessageRejected("invalid_envelope") from error
```
- `isinstance(raw, (bytes, bytearray))` → kiểm tra **một trong nhiều kiểu** (truyền tuple).
- `.decode("utf-8")` → byte → chuỗi; byte rác → `UnicodeDecodeError`.
- `raise LỗiMới(...) from error` → **chuỗi nguyên nhân** (*exception chaining*): lỗi mới giữ thông tin lỗi gốc (`__cause__`) → dễ gỡ lỗi; người gọi chỉ cần bắt một loại `MqttMessageRejected`.

### 6.3 Kiểm tra cấu trúc
```python
    if not isinstance(envelope, dict) or envelope.get("v") != ENVELOPE_VERSION:
        raise MqttMessageRejected("unsupported_envelope")

    payload_text = envelope.get("payload")
    signature = envelope.get("sig")
    if not isinstance(payload_text, str) or not isinstance(signature, str):
        raise MqttMessageRejected("invalid_envelope")
```
- JSON hợp lệ nhưng là **mảng/số** → không phải dict → từ chối. (Nếu không kiểm tra, `.get` trên list sẽ ném `AttributeError` — lỗi lập trình, không phải `MqttMessageRejected`.)
- `v != 2` → từ chối **tin không ký kiểu cũ** `{"order_code":..., "status":"PAID"}` (không có `v` → `.get("v")` = `None`).
- Kiểm tra **kiểu** `payload`, `sig` là chuỗi → chặn `{"v":2, "payload": {...}, "sig": 123}`.

### 6.4 Kiểm tra chữ ký — lõi bảo mật
```python
    expected = hmac.new(secret_bytes, payload_text.encode("utf-8"), hashlib.sha256).hexdigest()
    if not hmac.compare_digest(expected, signature.strip().lower()):
        raise MqttMessageRejected("bad_signature")
```
- Tính lại HMAC trên **đúng chuỗi `payload` nhận được** → so với `sig`.
- `.strip().lower()` → chấp nhận chữ ký có khoảng trắng thừa hoặc hex chữ HOA (khoan dung về **định dạng**, không khoan dung về **giá trị**).
- **`hmac.compare_digest`** → so sánh **thời gian không đổi**: `==` thường dừng ngay ở ký tự sai đầu tiên → kẻ gian đo thời gian phản hồi có thể dò dần từng ký tự chữ ký. `compare_digest` luôn so hết. Firmware cũng tự viết `constantTimeEquals` (XOR từng byte, gộp bằng OR) cùng mục đích.
- **Kiểm chữ ký TRƯỚC khi đọc nội dung** → nội dung chưa xác thực không bao giờ được dùng vào việc gì.

### 6.5 Giải mã nội dung
```python
    try:
        content = json.loads(payload_text)
    except json.JSONDecodeError as error:
        raise MqttMessageRejected("invalid_payload") from error
    if not isinstance(content, dict):
        raise MqttMessageRejected("invalid_payload")
```
- Chữ ký đúng nhưng nội dung không phải JSON object (chỉ xảy ra nếu bên gửi hợp lệ có lỗi) → vẫn từ chối gọn gàng.

### 6.6 Kiểm tra độ mới — chống phát lại
```python
    if max_age_seconds is not None:
        try:
            timestamp = float(content["ts"])
        except (KeyError, TypeError, ValueError) as error:
            raise MqttMessageRejected("missing_timestamp") from error
        current = time.time() if now is None else now
        # Cho phép lệch đồng hồ 30 giây về phía tương lai.
        if timestamp > current + 30 or current - timestamp > max_age_seconds:
            raise MqttMessageRejected("stale_message")

    return content
```
| Lỗi bắt | Khi nào |
|---|---|
| `KeyError` | Không có trường `ts` |
| `TypeError` | `ts` là `null`, list... |
| `ValueError` | `ts` là chuỗi không phải số `"abc"` |

Điều kiện từ chối:
| Điều kiện | Nghĩa |
|---|---|
| `current - timestamp > 120` | Tin **cũ hơn 120 giây** → nghi phát lại |
| `timestamp > current + 30` | Tin đến **từ tương lai** quá 30 giây → bất thường |

**Vì sao cho lệch 30 giây về tương lai?** Bên ký và bên kiểm có thể nằm trên **hai máy khác nhau** (trên Pi thật thì webhook và kiosk cùng chạy trên Pi nên chung đồng hồ; nhưng khi phát triển có thể chạy webhook trên laptop), và đồng hồ hai máy không bao giờ khớp tuyệt đối. Tin ký lúc 10:00:05 theo đồng hồ máy gửi có thể đến kiosk khi đồng hồ kiosk mới 10:00:02 → "đến từ tương lai 3 giây" là **bình thường**. Nhưng `ts` cách hiện tại cả **5 phút về tương lai** thì có thể là kẻ gian cố tạo tin "sống lâu" → từ chối (thử thật: +300 s → `stale_message`).

**Vì sao 120 giây?** Đủ rộng cho trễ mạng + broker công cộng + lệch đồng hồ, nhưng đủ hẹp để một tin chép được **không dùng lại được** về sau.

**Trong vòng 120 giây, gửi lại đúng tin đó thì sao?** Vẫn qua lớp này — nhưng gặp lớp sau: `save_new_order` thấy đơn đã `PAID` → `duplicate_order` → **không trừ kho lần 2**. Bảo mật theo **nhiều lớp** (*defense in depth*): mỗi lớp chặn một phần, cùng nhau chặn hết.

---

## 7. Đăng nhập broker và TLS (L118–135)

```python
def _env(name: str, role: str | None) -> str:
    if role:
        value = os.environ.get(f"MQTT_{role.upper()}_{name}", "")
        if value:
            return value
    return os.environ.get(f"MQTT_{name}", "")
```
**Tra biến môi trường theo vai trò, có dự phòng.** Ví dụ kiosk gọi `_env("USERNAME", "kiosk")`:
1. Tìm `MQTT_KIOSK_USERNAME` → có thì dùng (mỗi dịch vụ một tài khoản riêng).
2. Không có → dùng `MQTT_USERNAME` chung.
3. Cũng không có → `""`.

`role.upper()` → `"kiosk"` → `"KIOSK"`.

```python
def configure_client(client, role: str | None = None) -> None:
    """Gắn tài khoản và TLS (nếu có cấu hình) vào một paho.mqtt.client.Client."""
    username = _env("USERNAME", role)
    password = _env("PASSWORD", role)
    if username:
        client.username_pw_set(username, password or None)

    if os.environ.get("MQTT_TLS", "").strip().lower() in ("1", "true", "yes", "on"):
        ca_cert = os.environ.get("MQTT_CA_CERT", "").strip() or None
        client.tls_set(ca_certs=ca_cert)
```
| Dòng | Ý nghĩa |
|---|---|
| `client` không có chú thích kiểu | Nhận đối tượng `paho.mqtt.client.Client` — file **không import paho** nên test không cần cài paho |
| `-> None` | Hàm không trả gì, chỉ **cấu hình** đối tượng được truyền vào |
| `username_pw_set(u, p or None)` | Gắn tài khoản; mật khẩu rỗng → `None` (đăng nhập không mật khẩu) |
| `... in ("1", "true", "yes", "on")` | Chấp nhận nhiều cách viết "bật" |
| `tls_set(ca_certs=...)` | Bật **TLS**: mã hóa đường truyền tới broker. `None` → dùng chứng chỉ gốc có sẵn của hệ thống |

Được gọi ở: kiosk (`role="KIOSK"`, `bridge.py` L211), webhook (`role="WEBHOOK"`, `webhook_server.py` L47), API (`role="API"`, `api_server.py` L94).

### Phân biệt 3 lớp bảo vệ MQTT
| Lớp | Bảo vệ gì | Trong đồ án |
|---|---|---|
| **Tài khoản broker** | Ai được **kết nối** broker | Hỗ trợ — có tác dụng khi dùng broker riêng (broker công cộng thì ai cũng vào) |
| **TLS** | Không ai **nghe lén / sửa trên đường truyền** | Hỗ trợ, bật bằng `MQTT_TLS=true` |
| **Chữ ký HMAC** | **Nội dung** thật, chưa bị sửa, còn mới — **kể cả khi broker không đáng tin** | **Luôn bật**, thiếu khóa thì từ chối |

→ Lớp HMAC là lớp **quan trọng nhất** với broker công cộng: nó không tin vào broker, chỉ tin vào khóa.

**Lưu ý:** HMAC **không mã hóa** — ai cũng **đọc được** nội dung tin (mã đơn, số tiền, tên món). HMAC chỉ bảo đảm **không giả, không sửa được**. Muốn giấu nội dung thì cần TLS + broker riêng.

---

## 8. Ai dùng các hàm này?

| Nơi | Hàm | Việc |
|---|---|---|
| `servers/webhook_server.py` L36–66 | `configure_client`, `sign_message` | SePay báo tiền về → ký `{"event":"payment.success","order_code","amount","status":"PAID"}` → publish `payment/success` (qos=1). Thiếu khóa → **không phát** |
| `kiosk_app/bridge.py` L285–305 | `verify_message(..., max_age_seconds=120)` | Nhận `payment/success` → kiểm tra → sai thì in `[BẢO MẬT] Bỏ qua tín hiệu...` và **bỏ qua** |
| `kiosk_app/bridge.py` L862, L976 | `sign_message` | Ký tin tồn kho (retain) và đơn mới gửi bếp |
| `servers/api_server.py` L126, L143 | `sign_message` | Ký tin tồn kho và `kitchen/order_done` |
| `kds_firmware.ino` L1476–1532 | (C++) `verifySignedMessage` | Màn hình bếp kiểm tra cùng định dạng phiên bản 2 |

**Khác biệt đáng chú ý của firmware:** màn hình bếp kiểm tra chữ ký nhưng **không kiểm tra tuổi `ts`**. Hợp lý vì: tin tồn kho được publish với **`retain=True`** — broker giữ tin cuối cùng và giao lại cho thiết bị mới kết nối, có thể đã cũ hàng giờ; kiểm tra tuổi sẽ làm màn hình bếp vừa khởi động **không nhận được tồn kho**. Mặt khác, tin cho bếp chỉ dùng để **hiển thị** (không làm thay đổi tiền hay kho), và bếp còn tự đồng bộ qua API định kỳ. Tin **thanh toán** thì liên quan tiền → kiosk **bắt buộc** kiểm tra tuổi.

---

## 9. Kiểm thử — 18 bài đạt

`tests/test_mqtt_security.py` kiểm tra: ký rồi kiểm tra đúng nội dung; lọc trường `secret`; giữ tiếng Việt và dấu `"`; sửa `payload` → `bad_signature`; sai khóa → `bad_signature`; tin không ký kiểu cũ → từ chối; quá 120 s → `stale_message`; ts tương lai +300 s → từ chối; khóa rỗng/`None`/khoảng trắng → `mqtt_secret_not_configured` (cả khi ký lẫn khi kiểm).

`tests/test_bridge_payment_security.py` kiểm tra **ở mức kiosk**: tin thanh toán ký đúng → chốt đơn; ký bằng `"khoa-cua-ke-gia-mao"` → không chốt; tin cũ 600 s → không chốt.

---

## 10. Tổng kết

| Câu hỏi | Trả lời ngắn |
|---|---|
| File làm gì? | Ký và kiểm tra thông điệp MQTT bằng HMAC-SHA256; gắn tài khoản/TLS cho kết nối broker |
| Vì sao cần? | Broker công cộng — ai cũng publish được tin "PAID" giả |
| Chặn những gì? | Giả mạo, sửa nội dung (chữ ký), phát lại (ts trong phần ký + 120 s) |
| Vì sao `payload` là chuỗi? | Bên nhận băm đúng chuỗi nhận được, không dựng lại JSON → Python và ESP32 luôn khớp |
| Thiếu khóa thì sao? | **Fail-closed**: không ký, không chấp nhận |
| Vì sao `compare_digest`? | So sánh thời gian không đổi, chống dò chữ ký bằng đo thời gian |
| HMAC có giấu nội dung không? | **Không** — chỉ chống giả/sửa; giấu nội dung cần TLS |

**Câu trả lời khi bị hỏi:** *"Em dùng broker MQTT công cộng nên ai cũng có thể gửi tin 'đã thanh toán' giả. Vì vậy mọi tin đều được ký HMAC-SHA256 bằng khóa bí mật chung, gồm cả dấu thời gian. Kiosk tính lại chữ ký trên đúng chuỗi nội dung nhận được, so sánh bằng hàm thời gian không đổi, và từ chối tin cũ hơn 120 giây để chống phát lại. Thiếu khóa thì hệ thống từ chối tất cả. Nếu tin thật bị gửi lại trong 120 giây thì lớp CSDL vẫn trả `duplicate_order`, không trừ kho hai lần."*

---

## 11. Câu hỏi tự kiểm tra
1. Vì sao băm SHA-256 thông thường không đủ, phải dùng HMAC?
2. Vì sao `ts` phải nằm **bên trong** `payload` chứ không đặt cạnh `sig` trong phong bì?
3. Vì sao bên nhận không tự `json.dumps` lại nội dung rồi mới tính chữ ký?
4. Dấu `*` trong danh sách tham số của `sign_message` có tác dụng gì?
5. `raise ... from error` khác `raise ...` thế nào?
6. Vì sao cho phép lệch 30 giây về tương lai nhưng không cho 300 giây?
7. Một tin thanh toán thật bị kẻ gian gửi lại sau 60 giây — lớp nào chặn việc trừ kho lần 2?
8. Vì sao màn hình bếp không kiểm tra tuổi tin tồn kho?
9. Nếu quên đặt `MQTT_SHARED_SECRET` trên kiosk thì điều gì xảy ra? Vì sao thiết kế như vậy?
10. HMAC có làm kẻ gian **không đọc được** số tiền trong tin không?

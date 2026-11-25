# Kiến thức và cú pháp căn bản để đọc các file HTTP/Flask

> **Dùng tài liệu này thế nào:** đọc **trước** các tài liệu chi tiết trong `hoc_tap/HTTP_Flask/`:
> `webhook_server.md`, `admin_bridge.md`, `kds_firmware_http.md`, `kitchen_mcp_client.md` (và `hoc_tap/SQLite/api_server.md`).
> Python căn bản (import, `try/except`, decorator, `json`…) đã có ở `hoc_tap/SQLite/00_kien_thuc_can_ban.md` — ở đây không nhắc lại.
> **Mọi kết quả in ra đều đã chạy thật**: một ứng dụng Flask nhỏ viết riêng cho tài liệu (có route, khóa API, `errorhandler` giống `api_server.py`), chạy cục bộ ở cổng 5991, gọi bằng `requests`, `urllib` và socket thô. Không đụng Pi, API thật hay `.env`.

---

## 0. Bản đồ: kiến thức nào dùng ở file nào

| Kiến thức | Mục | `api_server` | `webhook_server` | `admin_bridge` | firmware bếp | Kitchen MCP |
|---|---|:-:|:-:|:-:|:-:|:-:|
| IP, cổng, TCP, URL | A | ● | ● | ● | ● | ● |
| Cấu trúc yêu cầu / phản hồi, header | B1–B3 | ● | ● | ● | ● | ● |
| Phương thức, mã trạng thái | B4–B5 | ● | ● | ● | ● | ● |
| Dạng thân: JSON, form, multipart | B6 | ● | ● | ● | ● | ● |
| Flask: route, `request`, `jsonify` | C1–C4 | ● | ● | | | |
| Flask: `errorhandler`, decorator xác thực | C5–C6 | ● | | | | |
| Flask: file, ảnh, chạy máy chủ | C7–C8 | ● | ● | | | |
| `requests` | D1 | | | ● | | |
| `urllib` | D2 | | | | | ● |
| `HTTPClient` (ESP32) | D3 | | | | ● | |
| Timeout, lỗi mạng | D4 | | | ● | ● | ● |
| Khóa API, bảo mật cơ bản | E | ● | ● | ● | ● | ● |
| Webhook vs polling | F | | ● | | ● | |

---

# PHẦN A — Mạng căn bản

## A1. IP, cổng, TCP
- **Địa chỉ IP** = địa chỉ của **máy** trong mạng: `192.168.1.10` (Pi trong mạng LAN).
- **Cổng** (*port*) = “số phòng” của **chương trình** trên máy đó: API dùng **5000**, MQTT **1883**. Một máy chạy nhiều dịch vụ, mỗi dịch vụ một cổng.
- **TCP** = kênh truyền tin cậy (đúng thứ tự, không mất) giữa hai chương trình. HTTP và MQTT đều chạy **trên** TCP. Trước khi gửi gì phải **mở kết nối TCP** (bắt tay 3 bước).

| Địa chỉ đặc biệt | Nghĩa |
|---|---|
| `127.0.0.1` / `localhost` | Chính máy này (chỉ chương trình **cùng máy** nối được) |
| `0.0.0.0` (khi **lắng nghe**) | Mọi card mạng của máy → máy khác trong LAN nối được. `api_server` chạy `host='0.0.0.0'` để laptop và ESP32 gọi được |

## A2. URL
```
http://192.168.1.10:5000/api/orders?limit=10&page=2
└┬─┘   └────┬─────┘ └┬─┘└────┬───┘ └──────┬───────┘
giao thức  host     cổng  đường dẫn   chuỗi truy vấn (query string)
```
- Thiếu cổng thì mặc định: `http` → 80, `https` → 443.
- Chuỗi truy vấn: `?khóa=giá_trị&khóa2=giá_trị2`. Ký tự đặc biệt phải **mã hóa phần trăm**:
```
{"q": "trà sữa & đá"}  →  q=tr%C3%A0+s%E1%BB%AFa+%26+%C4%91%C3%A1
```
(`&` thành `%26`; nếu không mã hóa, `&` sẽ cắt tham số làm đôi.)

## A3. Hai kiểu lỗi kết nối (gặp ở mọi máy khách)
| Tình huống | Điều xảy ra | Thời gian (đo thật với `requests`, timeout 2 s) |
|---|---|---|
| Máy có thật, **cổng không ai nghe** | Máy đích trả lời “từ chối” ngay | `ConnectionError` sau **0,00 s** |
| **IP không tồn tại / mất mạng** | Không ai trả lời, phải chờ hết hạn | `ConnectTimeout` sau **2,00 s** |

Vì vậy **mọi yêu cầu phải có timeout**, nếu không chương trình có thể chờ vô hạn.

---

# PHẦN B — Giao thức HTTP

## B1. Mô hình hỏi – đáp
Máy khách (*client*) gửi **một yêu cầu** (*request*), máy chủ (*server*) trả **một phản hồi** (*response*). Mỗi cặp **độc lập** — HTTP **không nhớ** lần trước (*stateless*), nên **mỗi yêu cầu phải tự mang khóa API**.

## B2. Hình dạng thật trên dây
Yêu cầu (gửi tay bằng socket):
```
GET /api/menu/1 HTTP/1.1          ← dòng yêu cầu: PHƯƠNG THỨC  ĐƯỜNG DẪN  PHIÊN BẢN
Host: 127.0.0.1:5991              ← các header "Tên: giá trị"
Connection: close
                                  ← một dòng trống = hết header
```
Phản hồi thật của Flask:
```
HTTP/1.1 200 OK                   ← dòng trạng thái: PHIÊN BẢN  MÃ  LÝ DO
Server: Werkzeug/3.1.9 Python/3.10.12
Date: Wed, 25 Nov 2026 16:44:19 GMT
Content-Type: application/json    ← thân là JSON
Content-Length: 50                ← thân dài 50 byte
Connection: close

{"id":1,"name":"Trà sữa","price":30000}     ← thân (body)
```
Mỗi dòng kết thúc bằng `\r\n`. HTTP/1.1 là **văn bản thuần** — đọc được bằng mắt. (Flask mặc định viết tiếng Việt dạng `à`; mọi thư viện JSON đều giải mã lại được.)

## B3. Header hay gặp trong dự án
| Header | Hướng | Ý nghĩa |
|---|---|---|
| `Host` | yêu cầu | Máy chủ đích (bắt buộc trong HTTP/1.1) |
| `Content-Type` | cả hai | Kiểu của thân: `application/json`, `multipart/form-data`, `text/html`… |
| `Content-Length` | cả hai | Độ dài thân (byte) |
| `Accept` | yêu cầu | Kiểu phản hồi mong muốn |
| `X-API-Key` | yêu cầu | **Khóa API tự đặt** của dự án (tiền tố `X-` = header không chuẩn) |
| `Authorization` | yêu cầu | Header xác thực chuẩn — SePay gửi `Authorization: Apikey <token>` |
| `User-Agent` | yêu cầu | Chương trình gửi (`python-requests/…`, `ESP32HTTPClient`) |

## B4. Phương thức (*method*)
| Phương thức | Dùng để | Có thân? | Trong dự án |
|---|---|---|---|
| **GET** | Đọc | Không (tham số đặt trên URL) | `/api/menu`, `/api/orders/pending`, `/api/inventory` |
| **POST** | Tạo / thực hiện hành động | Có | `/api/orders/done`, `/api/menu/add`, `/sepay_webhook` |
| PUT, PATCH, DELETE | Thay / sửa / xóa | | Dự án **không dùng**: sửa, xóa cũng đi qua POST (`/api/menu/delete`) cho đơn giản |

Gọi sai phương thức → **405 Method Not Allowed** (thử: `DELETE /api/menu` → 405).
**GET phải không làm đổi dữ liệu** — trình duyệt, proxy có thể tự gọi lại GET.

## B5. Mã trạng thái
| Nhóm | Mã | Nghĩa | Gặp ở dự án |
|---|---|---|---|
| 2xx thành công | **200** OK | | hầu hết |
| | **201** Created | Đã tạo mới | (ví dụ trong tài liệu này) |
| 4xx **lỗi phía khách** | **400** Bad Request | Dữ liệu gửi lên sai | thiếu `order_code`, giá âm |
| | **401** Unauthorized | Không có / sai khóa | thiếu `X-API-Key` |
| | **403** Forbidden | Biết anh là ai nhưng **không đủ quyền** | khóa bếp gọi `/api/report` |
| | **404** Not Found | Không có tài nguyên | món không tồn tại |
| | **405** Method Not Allowed | Sai phương thức | |
| | **409** Conflict | Xung đột trạng thái | đơn đã xong rồi |
| | **413** Payload Too Large | Thân quá lớn | ảnh vượt `MAX_CONTENT_LENGTH` |
| | **429** Too Many Requests | Gọi quá nhiều | đăng nhập sai 5 lần |
| 5xx **lỗi phía máy chủ** | **500** Internal Server Error | Lỗi chưa lường trước | |
| | **502** Bad Gateway | Máy trung gian: chặng sau hỏng | webhook không gửi được MQTT |
| | **503** Service Unavailable | Chưa sẵn sàng / chưa cấu hình | thiếu khóa API trong `.env` |

Quy tắc nhớ: **4xx = khách sửa yêu cầu** (thử lại y nguyên vô ích), **5xx = máy chủ có vấn đề** (thử lại sau có thể được). Firmware bếp dựa đúng vào quy tắc này để quyết định thử lại.

## B6. Ba dạng thân yêu cầu
| Dạng | `Content-Type` | Trông như | Flask đọc bằng | Dùng khi |
|---|---|---|---|---|
| **JSON** | `application/json` | `{"order_code":"DH00000001"}` | `request.get_json()` | Dữ liệu có cấu trúc |
| **Form** | `application/x-www-form-urlencoded` | `name=Cam&price=35000` | `request.form["name"]` | Biểu mẫu HTML |
| **Multipart** | `multipart/form-data; boundary=…` | nhiều “phần” ngăn bởi chuỗi boundary | `request.form` + `request.files` | **Gửi kèm file** (ảnh món) |

Kiểm tra bằng `requests`: `json=` → `application/json`; `data=` + `files=` → `multipart/form-data; boundary=a5ed07d3…`.
⚠ Gửi JSON mà **quên `Content-Type: application/json`** → Flask coi như không có JSON → thường ra 400.

## B7. JSON API và “hợp đồng lỗi” của dự án
Mọi phản hồi lỗi có cùng hình dạng:
```json
{"success": false, "reason": "invalid_name"}
```
- `reason` là **mã ngắn cho máy đọc**; máy khách tự dịch sang câu tiếng Việt cho người (`_REASON_MESSAGES` trong `admin_bridge.py`).
- Thành công: `{"success": true, ...}` hoặc trả thẳng dữ liệu (danh sách món, kho).

## B8. HTTPS
HTTPS = HTTP chạy trong đường hầm **TLS** mã hóa. Dự án chạy **HTTP thường trong LAN** (khóa API đi dạng chữ rõ) — HTTPS để vào hướng phát triển. Riêng chiều SePay → webhook đi **HTTPS qua đường hầm** Internet tới Pi.

---

# PHẦN C — Flask (máy chủ)

## C1. Ứng dụng tối thiểu
```python
from flask import Flask, request, jsonify

app = Flask(__name__)                       # tạo ứng dụng

@app.route("/api/menu", methods=["GET"])    # gắn URL + phương thức vào hàm
def menu():
    return jsonify([{"id": 1, "name": "Trà sữa", "price": 30000}])

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```
- `@app.route(...)` là **decorator**: đăng ký hàm `menu` làm **hàm xử lý** (*view*) cho `GET /api/menu`.
- Không ghi `methods` thì mặc định chỉ `GET`.
- Mỗi yêu cầu tới, Flask gọi đúng hàm, lấy giá trị trả về làm phản hồi.

## C2. Tham số trong đường dẫn
```python
@app.route("/api/menu/<int:pid>")
def mot_mon(pid): ...
```
| URL | Kết quả |
|---|---|
| `/api/menu/1` | 200, `pid = 1` (đã là `int`) |
| `/api/menu/9` | 404 JSON do code tự báo |
| `/api/menu/abc` | **404 HTML** — không khớp `<int:...>` nên Flask không tìm thấy route |

`api_server.py` dùng `'/images/<filename>'` (kiểu chuỗi mặc định) để phát ảnh.

## C3. Đối tượng `request` — đọc yêu cầu
`request` là biến “toàn cục theo từng yêu cầu”: trong hàm xử lý, nó luôn trỏ tới yêu cầu **đang** được phục vụ.

| Thuộc tính | Lấy gì | Ví dụ |
|---|---|---|
| `request.args.get("q", "")` | Chuỗi truy vấn | `/api/menu?q=TRÀ` → `"TRÀ"` |
| `request.args.get("page", type=int)` | ép kiểu luôn (sai → `None`) | |
| `request.get_json(silent=True)` | Thân JSON → dict/list; **`silent=True`**: không phải JSON thì trả `None` thay vì ném lỗi | |
| `request.form["name"]` | Trường form / multipart | |
| `request.files["image"]` | File tải lên | |
| `request.headers.get("X-API-Key", "")` | Header | |
| `request.remote_addr` | IP người gọi | giới hạn đăng nhập theo IP |
| `request.method` | `"GET"`, `"POST"` | |

Mẫu đọc JSON an toàn của dự án:
```python
data = request.get_json(silent=True) or {}       # None → {}
order_code = str(data.get("order_code", "")).strip()
if not order_code:
    return jsonify({"success": False, "reason": "missing_order_code"}), 400
```
Thử: thân `name=Cam` (không phải JSON) → **400** `invalid_name`, không phải 500.

## C4. Trả phản hồi
```python
return jsonify({"success": True, "id": 2}), 201        # (thân, mã trạng thái)
return jsonify(ds_mon)                                 # mã mặc định 200
```
- `jsonify(...)` → thân JSON + `Content-Type: application/json`.
- Trả **tuple** `(thân, mã)` để đặt mã trạng thái; `(thân, mã, headers)` nếu cần thêm header.

## C5. Xử lý lỗi tập trung: `@app.errorhandler`
```python
class ApiError(Exception):
    def __init__(self, status, reason):
        self.status, self.reason = status, reason

@app.errorhandler(ApiError)
def h_api(e):
    return jsonify({"success": False, "reason": e.reason}), e.status

@app.route("/api/menu/<int:pid>")
def mot_mon(pid):
    if pid not in MON:
        raise ApiError(404, "product_not_found")      # ném ở đâu cũng được
    return jsonify(MON[pid])
```
→ `GET /api/menu/9` = **404** `{"reason":"product_not_found","success":false}`.
- Hàm xử lý chỉ cần **`raise`**, không phải tự `return jsonify(...)` ở mọi nhánh.
- `api_server.py` đăng ký 4 bộ: `ApiError` (lỗi tự đặt), `HTTPException` (404/405/413 của Flask → đổi sang JSON), `sqlite3.IntegrityError` (→ 409), `Exception` (mọi lỗi khác → 500 JSON).
- Không có bộ `HTTPException` thì 404/405 trả **HTML** (thử: `/api/menu/abc` → `text/html`, `<title>404 Not Found</title>`) — máy khách đọc JSON sẽ hỏng. Đây là lý do `api_server` có bộ đó; `webhook_server` thì không (ghi ở `webhook_server.md`).

## C6. Decorator xác thực
```python
from functools import wraps
import hmac

def can_khoa(f):
    @wraps(f)
    def w(*a, **k):
        if not hmac.compare_digest(request.headers.get("X-API-Key", ""), KEY):
            return jsonify({"success": False, "reason": "unauthorized"}), 401
        return f(*a, **k)          # khóa đúng → chạy hàm thật
    return w

@app.route("/api/menu/add", methods=["POST"])
@can_khoa                          # ← nằm DƯỚI @app.route
def them(): ...
```
| Thử | Kết quả |
|---|---|
| Không khóa | **401** `unauthorized` |
| Có khóa, JSON đúng | **201** `{"id":2,"success":true}` |
| Có khóa, JSON thiếu tên | **400** `invalid_name` |

- **Thứ tự decorator**: áp từ **dưới lên**. `@can_khoa` bọc hàm trước, rồi `@app.route` đăng ký **hàm đã bọc** → mọi yêu cầu đều qua kiểm tra. Đảo ngược thì route đăng ký hàm **chưa bọc** → không có bảo vệ.
- `@wraps(f)` quan trọng với Flask: Flask dùng **tên hàm** làm định danh route; không có `@wraps` thì mọi hàm bọc đều tên `w` → lỗi trùng.
- `api_server.py` có 2 decorator: `require_api_key` (chỉ admin; khóa bếp → 403) và `require_kitchen_or_admin_key`.

## C7. File: tải lên và phát ra
```python
from werkzeug.utils import secure_filename
f = request.files["image"]
ten = uuid.uuid4().hex[:8] + "_" + secure_filename(f.filename)    # "../../etc/x" → "etc_x"; "Trà sữa.png" → "Tra_sua.png"
f.save(os.path.join(UPLOAD_FOLDER, ten))

from flask import send_from_directory
@app.route("/images/<filename>")
def anh(filename):
    return send_from_directory(UPLOAD_FOLDER, filename)   # chặn "../" thoát khỏi thư mục
app.config["MAX_CONTENT_LENGTH"] = 8 * 1024 * 1024       # api_server.py L33; vượt → 413
```

## C8. Chạy máy chủ
| Cách | Dùng khi | Ghi nhớ |
|---|---|---|
| `app.run(host, port)` | Phát triển | Máy chủ thử của Werkzeug, in cảnh báo “development server” |
| `waitress.serve(app, host, port, threads=8)` | Chạy thật (`api_server`) | Máy chủ WSGI dùng được lâu dài, **8 luồng** phục vụ song song |
| `app.test_client()` | Viết test | Gọi route **không cần mạng** — toàn bộ bảng thử ở C2–C6 dùng cách này |

**WSGI** = chuẩn giao tiếp giữa máy chủ web và ứng dụng Python; Flask là ứng dụng WSGI, Waitress là máy chủ WSGI.
Vì có nhiều luồng, dữ liệu dùng chung giữa các yêu cầu (vd. bộ đếm đăng nhập sai) phải có `threading.Lock()`.

---

# PHẦN D — Máy khách HTTP

## D1. `requests` (dùng ở `admin_bridge.py`)
```python
import requests
r = requests.get(url, params={"q": "trà sữa & đá"}, headers={"X-API-Key": k}, timeout=3)
r = requests.post(url, json={"order_code": "DH00000001"}, timeout=5)
r = requests.post(url, data={"name": "Cam"}, files={"image": open("a.png", "rb")}, timeout=5)

r.status_code      # 200
r.ok               # True nếu < 400
r.json()           # thân → dict/list (thân không phải JSON → ValueError)
r.url              # URL thật đã gửi (đã mã hóa params)
r.raise_for_status()   # 4xx/5xx → ném requests.HTTPError
```
Đã chạy thật:
```
r.url → http://127.0.0.1:5991/api/menu?q=tr%C3%A0+s%E1%BB%AFa+%26+%C4%91%C3%A1
POST không khóa → KHÔNG ném lỗi, status 401, {'reason': 'unauthorized', 'success': False}
raise_for_status() → HTTPError: 401 Client Error: UNAUTHORIZED for url: …
```
| Ngoại lệ | Khi |
|---|---|
| `requests.ConnectionError` | Không nối được (cổng đóng) |
| `requests.ConnectTimeout` / `ReadTimeout` | Hết giờ nối / hết giờ đọc |
| `requests.RequestException` | **Lớp cha** của tất cả → `admin_bridge` bắt cái này |

⚠ **`requests` mặc định chờ vô hạn** — phải luôn truyền `timeout=`.
⚠ `requests` **không** ném lỗi với 4xx/5xx; phải tự xem `status_code` (hoặc `raise_for_status()`).

## D2. `urllib` (thư viện chuẩn, dùng ở Kitchen MCP)
```python
from urllib.request import Request, urlopen
from urllib.error import HTTPError, URLError
from urllib.parse import urlencode

req = Request(url + "?" + urlencode({"limit": 10}),
              headers={"X-API-Key": k, "Accept": "application/json"}, method="GET")
try:
    with urlopen(req, timeout=5) as resp:
        data = json.loads(resp.read().decode("utf-8"))
except HTTPError as e:          # 4xx/5xx — urllib TỰ NÉM
    print(e.code, e.read().decode())
except URLError:                # không nối được
    ...
```
Thử: `GET /api/menu/9` → `HTTPError` mã 404, thân `{"reason":"product_not_found","success":false}`.
Khác `requests`: `urllib` **ném** lỗi với 4xx/5xx; thân là **bytes** (phải `.decode`); POST phải tự `json.dumps(...).encode()`. Vì `HTTPError` là lớp con của `URLError`, phải `except HTTPError` **trước**.

## D3. `HTTPClient` trên ESP32 (firmware bếp)
```cpp
HTTPClient http;
http.begin(url);                       // chưa nối mạng
http.setConnectTimeout(2500); http.setTimeout(4000);
http.addHeader("X-API-Key", KDS_API_KEY);
int status = http.GET();               // > 0: mã HTTP;  < 0: lỗi mạng (-1 refused, -11 read timeout)
String body = http.getString();
http.end();                            // luôn đóng
```
Không có ngoại lệ: **một con số** cho cả hai loại lỗi.

## D4. So sánh nhanh
| | `requests` | `urllib` | `HTTPClient` |
|---|---|---|---|
| 4xx/5xx | trả về bình thường | **ném** `HTTPError` | số dương |
| Lỗi mạng | ném `RequestException` | ném `URLError`/`TimeoutError` | số âm |
| Timeout mặc định | **vô hạn** | vô hạn (theo socket) | 5 s |
| Gửi JSON | `json=` | `json.dumps().encode()` + header | `serializeJson` + header |
| Tham số URL | `params=` | `urlencode` | tự ghép |
| Đóng kết nối | tự động | `with` | `http.end()` |

---

# PHẦN E — Bảo mật cơ bản trong API của dự án

| Kỹ thuật | Chống gì | Ở đâu |
|---|---|---|
| **Khóa API** trong header (`X-API-Key`) | Người lạ trong mạng gọi API | `api_server` |
| **Hai mức khóa**: admin / bếp | Lộ khóa bếp (nằm trong flash ESP32) không sửa được thực đơn | 401 vs 403 |
| `hmac.compare_digest(a, b)` | Đoán khóa dần theo thời gian so sánh (*timing attack*) | xác thực khóa, token SePay |
| **Fail‑closed**: chưa cấu hình khóa → từ chối hết (503) | Quên cấu hình thành “mở toang” | cả 2 máy chủ |
| Kiểm tra đầu vào → 400 | Dữ liệu rác vào CSDL, lỗi 500 | `_to_int`, `_to_float` |
| Giới hạn số lần (429) | Dò mật khẩu | đăng nhập: 5 lần / 300 s / IP |
| `secure_filename` + `send_from_directory` | Ghi/đọc file ngoài thư mục (`../`) | ảnh món |
| `MAX_CONTENT_LENGTH` (413) | Gửi file khổng lồ làm đầy đĩa | tải ảnh |
| Không in bí mật ra log | Lộ khóa qua nhật ký | mọi nơi |

---

# PHẦN F — Webhook và polling

| | **Polling** (hỏi dồn) | **Webhook** (gọi ngược) |
|---|---|---|
| Ai chủ động | Mình hỏi “có gì mới không?” định kỳ | Dịch vụ ngoài **gọi vào** máy chủ của mình khi có sự kiện |
| Độ trễ | Tới một chu kỳ hỏi | Gần như tức thời |
| Cần mở cổng ra Internet | Không | **Có** (dự án dùng đường hầm Internet tới Pi) |
| Trong dự án | Kiosk hỏi SePay mỗi 15 s (dự phòng); firmware bếp hỏi API mỗi 30 s | SePay → `POST /sepay_webhook` |

Dự án dùng **cả hai**: webhook là đường nhanh, polling là đường chắc.

---

# PHẦN G — Bảng tra nhanh và tự kiểm tra

## G1. Thấy cái này trong code → nghĩa là gì
| Thấy | Nghĩa |
|---|---|
| `@app.route("/x", methods=["POST"])` | Hàm bên dưới xử lý `POST /x` |
| `return jsonify(d), 400` | Trả JSON với mã 400 |
| `request.get_json(silent=True) or {}` | Đọc JSON, không phải JSON thì coi như `{}` |
| `request.args.get("page", type=int)` | Tham số URL, ép sang số |
| `raise ApiError(404, "...")` | Trả lỗi JSON qua `errorhandler` |
| `@require_api_key` dưới `@app.route` | Route cần khóa admin |
| `host='0.0.0.0'` | Cho máy khác trong mạng truy cập |
| `requests.get(..., timeout=3)` | Gọi HTTP, chờ tối đa 3 s |
| `except requests.RequestException` | Mọi lỗi mạng của `requests` |
| `except HTTPError` trước `except URLError` | Phân biệt “máy chủ trả lỗi” với “không nối được” |
| `files={"image": ...}` | Gửi multipart kèm file |
| `"Content-Type": "application/json"` | Báo thân là JSON (thiếu → máy chủ không đọc) |

## G2. Thứ tự đọc đề xuất
Tài liệu này → `webhook_server.md` (máy chủ nhỏ nhất) → `hoc_tap/SQLite/api_server.md` (máy chủ chính) → `admin_bridge.md` → `kds_firmware_http.md` → `kitchen_mcp_client.md`.

## G3. Câu hỏi tự kiểm tra
1. Phân biệt IP và cổng. Vì sao `api_server` lắng nghe `0.0.0.0` chứ không phải `127.0.0.1`?
2. Viết tay một yêu cầu HTTP `POST /api/orders/done` có khóa và thân JSON (đủ dòng yêu cầu, header, dòng trống, thân).
3. 401 và 403 khác nhau thế nào? Cho ví dụ mỗi loại trong dự án.
4. Vì sao 4xx không nên thử lại còn 5xx thì có thể?
5. Gửi JSON nhưng quên `Content-Type: application/json` thì Flask `get_json(silent=True)` trả gì?
6. `/api/menu/abc` với route `<int:pid>` trả gì, dạng gì? Làm sao để nó trả JSON?
7. Đảo thứ tự `@app.route` và `@require_api_key` thì chuyện gì xảy ra?
8. Vì sao decorator bọc route phải có `@wraps`?
9. `requests.get(url)` không có `timeout` — rủi ro là gì?
10. Cùng một phản hồi 404: `requests` và `urllib` cư xử khác nhau ra sao?
11. Khi nào dùng `json=`, khi nào dùng `data=` + `files=`?
12. Vì sao so khóa bằng `hmac.compare_digest` mà không dùng `==`?
13. Webhook và polling khác nhau ở đâu? Vì sao dự án dùng cả hai cho thanh toán?

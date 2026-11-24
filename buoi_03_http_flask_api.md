# BUỔI 3 — HTTP/REST, FLASK VÀ SMART KIOSK API

> **Chuỗi học tập công nghệ trong Đồ án tốt nghiệp**
> *Hệ thống tự động hóa quy trình phục vụ và điều phối thông minh trong nhà hàng*
>
> - **Mã nguồn chính:** `Smart_Kiosk_Backup/servers/api_server.py` (848 dòng, 26 endpoint)
> - **Mã nguồn liên quan:** `backend/database.py`, `admin_app/admin_bridge.py`, `admin_app/qml/DashboardContent.qml`, `kds_firmware.ino/kds_firmware/kds_firmware.ino`, `Voice_Test/voice_assistant_edge_ai/services/kitchen_mcp/src/kitchen_mcp/` (`client.py`, `write_client.py`, `config.py`), `gateway/src/voice_gateway/tool_policy/policy.py`, `deploy/pi/`, `deploy/laptop/`, `tests/test_api_hardening.py`, `tests/test_orders_done_isolated_http.py`
> - **Phiên bản mã:** Smart_Kiosk_Backup `85f41af` (đã gồm bản sửa lỗi `b8d97a9`), kds_firmware `7b02d68`, Voice_Test `d381809`; Flask 3.1.3, Werkzeug 3.1.9, Waitress (trên Pi)
> - **Mức độ:** hiểu tới tận mã nguồn; số dòng (`Lxxx`) theo phiên bản trên
> - **Số liệu thực nghiệm:** chạy trên CSDL tạm với khóa giả, hoặc gửi yêu cầu thật từ laptop tới API trên Pi (khóa đọc từ `.env`, không in ra)

> **Ghi chú về lịch sử sửa lỗi.** Buổi học này ban đầu phát hiện nhiều lỗi bằng thí nghiệm (doanh thu tính thiếu, khóa CSDL bị treo, endpoint đọc không cần khóa…). Tất cả đã được sửa ở commit `b8d97a9`. Trong tài liệu, mỗi lỗi được trình bày theo dạng **Trước khi sửa → Sau khi sửa** — đây là bằng chứng tốt khi bảo vệ: *phát hiện bằng thực nghiệm, sửa có kiểm thử*.

---

## Mục lục

1. [Nền tảng HTTP](#mục-1-nền-tảng-http)
2. [REST và thiết kế API của đồ án](#mục-2-rest-và-thiết-kế-api-của-đồ-án)
3. [Flask từ bên trong](#mục-3-flask-từ-bên-trong)
4. [Xác thực và phân quyền ★](#mục-4-xác-thực-và-phân-quyền-)
5. [Các bên gọi API (phía client)](#mục-5-các-bên-gọi-api-phía-client)
6. [Truy vết đầu cuối hai yêu cầu](#mục-6-truy-vết-đầu-cuối-hai-yêu-cầu)
7. [Đánh giá và câu hỏi ôn tập](#mục-7-đánh-giá-và-câu-hỏi-ôn-tập)
- [Phụ lục A: Triển khai và các file tự chạy](#phụ-lục-a-triển-khai-và-các-file-tự-chạy)
- [Phụ lục B: Cách chạy lại các thí nghiệm](#phụ-lục-b-cách-chạy-lại-các-thí-nghiệm)

### Khuôn trình bày mỗi tiểu mục

| Phần | Nội dung |
|---|---|
| **① Khái niệm** | Định nghĩa ngắn, từ khóa *in nghiêng* |
| **② Trong đồ án** | File và số dòng, đoạn mã cốt lõi, kết quả chạy thật |
| **③ Vì sao / Nếu không** | Lý do thiết kế; bỏ đi thì hỏng ở đâu (phần hội đồng hay hỏi) |
| **④ Ghi nhớ** | 1–3 ý để ôn nhanh |

### Bốn hình nên vẽ tay

1. Ai gọi endpoint nào (mục 2.1)
2. Sơ đồ tuần tự yêu cầu báo xong đơn (mục 6.1)
3. Luồng decorator kiểm tra khóa (mục 4.3)
4. Hai task và hàng đợi của màn hình bếp (mục 5.2)

---

# MỤC 1. NỀN TẢNG HTTP

## 1.1. Mô hình yêu cầu – phản hồi; HTTP không lưu trạng thái

**① Khái niệm**

- *HTTP* (HyperText Transfer Protocol) là giao thức tầng ứng dụng chạy trên TCP. **Client** gửi một *yêu cầu* (request), **server** trả đúng một *phản hồi* (response). Server **không bao giờ tự gửi trước**.
- *Không lưu trạng thái* (stateless): mỗi yêu cầu **độc lập**, server không nhớ yêu cầu trước; mọi thông tin cần thiết (khóa, dữ liệu) phải nằm **trong chính yêu cầu**.

**② Trong đồ án**

- **Server:** Smart Kiosk API trên Raspberry Pi, cổng 5000, chạy bằng Waitress qua `main()` (`api_server.py` L828–844).
- **Client:** phần mềm quản trị (laptop), màn hình bếp (ESP32-S3), Kitchen MCP (trợ lý giọng nói, laptop).
- **Mỗi yêu cầu tự mang khóa:** hàm `_caller_role()` (L280) đọc `request.headers.get("X-API-Key")` ở **mọi lần gọi**; không có cookie hay phiên đăng nhập.

**③ Vì sao / Nếu không**

- **Stateless giúp server đơn giản và bền:** Pi khởi động lại thì client chỉ cần gửi lại yêu cầu; màn hình bếp chỉ cần nhớ một khóa trong firmware.
- **Server không tự đẩy dữ liệu xuống:** với HTTP, màn hình bếp phải **hỏi định kỳ** (polling, 30 s/lần). Vì vậy đồ án dùng thêm **MQTT** để đẩy tức thời: đơn mới (`kitchen/new_order`), tồn kho (`kitchen/inventory`) và — sau khi sửa — đơn đã xong (`kitchen/order_done`). HTTP dùng để **đồng bộ lại** và **gửi lệnh**.

**④ Ghi nhớ**

- HTTP = **client hỏi, server trả lời**; server không tự gửi.
- Không lưu trạng thái → **mỗi yêu cầu tự mang `X-API-Key`**.
- Đẩy tức thời cần giao thức khác: **MQTT**.

---

## 1.2. Cấu trúc một yêu cầu và một phản hồi

**① Khái niệm**

| Phần | Ví dụ | Ý nghĩa |
|---|---|---|
| **Dòng yêu cầu** | `POST /api/orders/done HTTP/1.1` | **Phương thức** + **đường dẫn** + phiên bản |
| **Header** | `X-API-Key: …`, `Content-Type: application/json` | Thông tin kèm theo dạng `Tên: giá trị` |
| **Dòng trống** | | Kết thúc phần header |
| **Body** | `{"order_code":"DH12345678"}` | Dữ liệu gửi đi (GET thường không có) |

- Phản hồi tương tự: **dòng trạng thái** (`HTTP/1.1 200 OK`), header, dòng trống, body.
- **GET** lấy dữ liệu, không thay đổi gì; **POST** gửi dữ liệu để xử lý hoặc thay đổi.
- *Query string*: phần sau `?`, gồm các cặp `khóa=giá trị` nối bằng `&`, dùng để lọc, phân trang.

**② Trong đồ án**

Yêu cầu thật của màn hình bếp khi bấm HOÀN THÀNH (`>` gửi đi, `<` nhận về; khóa giả):

```
> POST /api/orders/done HTTP/1.1        ← dòng yêu cầu
> Host: 127.0.0.1:5099
> X-API-Key: khoa_bep_demo              ← khóa xác thực
> Content-Type: application/json        ← body là JSON
> Content-Length: 27
>                                       ← dòng trống
{"order_code":"DH12345678"}             ← body

< HTTP/1.1 200 OK                       ← dòng trạng thái
< Content-Type: application/json
<
{"kitchen_status":"done","order_code":"DH12345678","success":true}
```

Phía firmware (`kds_firmware.ino` L1602–1612):

```cpp
http.begin(apiUrl("/api/orders/done"));
http.addHeader("Content-Type", "application/json");
http.addHeader("X-API-Key", KDS_API_KEY);
int status = http.POST(body);
```

Phía Flask (`api_server.py` L700–701):

```python
data = request.get_json(silent=True) or {}
order_code = str(data.get('order_code', '')).strip()
```

Query string: `GET /api/orders?status_filter=paid&page=1&limit=10&keyword=DH` (L572–632). Tham số đúng tên là **`status_filter`**.

> **Lưu ý:** Flask **bỏ qua im lặng** tham số lạ — gõ nhầm `?status=paid` vẫn ra 200, chỉ là bộ lọc không có tác dụng.

Ba chi tiết thấy khi đọc phản hồi thật:

- Header `Date` luôn theo **giờ GMT** (HTTP quy định).
- Tiếng Việt trong JSON hiện thành `Bánh mì`: `jsonify` mặc định thoát ký tự Unicode; vẫn là JSON hợp lệ.
- Gửi body **không phải JSON**: `get_json(silent=True)` trả `None` thay vì ném lỗi → `data = {}` → **400 `missing_order_code`** rõ ràng.

**③ Vì sao / Nếu không**

- **Thiếu `Content-Type: application/json`** thì Flask không coi body là JSON — lỗi hay gặp nhất khi viết firmware.
- **Khóa đặt trong header, không trong URL**: đường dẫn bị ghi vào nhật ký truy cập; nhật ký thật chỉ có `"POST /api/orders/done HTTP/1.1" 401 -`, không lộ khóa.

**④ Ghi nhớ**

- Yêu cầu = **dòng yêu cầu + header + dòng trống + body**.
- Gửi JSON phải có `Content-Type: application/json`.
- Khóa nằm trong **header**, không nằm trong URL.

---

## 1.3. Mã trạng thái mà đồ án thực sự dùng

**① Khái niệm**

| Nhóm | Nghĩa |
|---|---|
| **2xx** | Thành công |
| **4xx** | **Lỗi do client** (gửi sai, thiếu quyền, xung đột); gửi lại y nguyên vẫn lỗi |
| **5xx** | **Lỗi do server**; thử lại sau có thể được |

**② Trong đồ án** — kiểm tra **thật trên Pi** từ laptop (sau khi triển khai bản sửa lỗi):

| Yêu cầu | Mã | `reason` / body |
|---|---|---|
| `GET /api/menu`, không khóa | **200** | danh sách 34 món (thực đơn công khai) |
| `GET /api/orders/pending`, **không khóa** | **401** Unauthorized | `unauthorized` |
| `GET /api/orders/pending`, khóa bếp | **200** | đơn chờ |
| `GET /api/inventory`, khóa bếp | **200** | 62 nguyên liệu |
| `GET /api/report`, **khóa bếp** | **403** Forbidden | `forbidden` |
| `GET /api/report`, khóa quản trị | **200** | báo cáo |
| `GET /api/khong_co` | **404** Not Found | `not_found` (**JSON**) |
| `GET /api/tts` (đã xóa) | **404** | `not_found` |
| `POST /api/orders/done` đơn không tồn tại, khóa bếp | **409** Conflict | `order_not_found_or_already_done` |
| `POST /api/menu/delete` thiếu id, khóa quản trị | **400** Bad Request | `invalid_id` |

Các mã khác trong mã nguồn (đã kiểm tra bằng test tự động):

| Mã | Khi nào | Vị trí |
|---|---|---|
| **405** Method Not Allowed | GET vào endpoint chỉ nhận POST | Flask tự sinh, `errorhandler` đổi sang JSON (L178) |
| **413** Payload Too Large | ảnh tải lên > 8 MB | `MAX_CONTENT_LENGTH` (L33) |
| **415** Unsupported Media Type | (còn trong bảng, ít gặp vì đã dùng `silent=True`) | L178 |
| **429** Too Many Requests | xin OTP quá dày; **đăng nhập sai 5 lần** | L504–521, L483–502 |
| **500** | lỗi không lường trước | `errorhandler(Exception)` (L192) |
| **502** Bad Gateway | không gửi được email OTP | L504–521 |
| **503** Service Unavailable | server **chưa cấu hình khóa** | decorator (L302, L330) |

**③ Vì sao / Nếu không**

| Mã | Client nên làm gì | Ví dụ màn hình bếp |
|---|---|---|
| 200 | Thành công | Gỡ thẻ đơn |
| 409 | Không phải lỗi thật: việc đã xong ở nơi khác | **Vẫn gỡ thẻ** |
| 400 / 401 / 403 | Thử lại cũng vô ích | **Không thử lại**, hiện nút GỬI LẠI |
| 5xx / lỗi mạng | Có thể tự hết | Thử lại có nghỉ 1–2–4 s |

- **Chưa có khóa → 503, không phải 401**: lỗi nằm ở **server** (chưa cấu hình), không phải client.
- **401 khác 403:** 401 = *không biết bạn là ai* (thiếu/sai khóa); 403 = *biết bạn là ai nhưng không cho phép* (khóa bếp gọi endpoint quản trị).

> **Trước khi sửa:** khóa bếp bị từ chối cũng nhận 401; đổi mật khẩu sai lại trả **200** kèm `success:false`; lỗi do Flask sinh (404/405/415/500) là **trang HTML**. **Sau khi sửa:** 403 đúng nghĩa, đổi mật khẩu sai trả 400, mọi lỗi đều là JSON.

**④ Ghi nhớ**

- **2xx** thành công · **4xx** client sai · **5xx** server có vấn đề.
- **401** thiếu/sai khóa · **403** đủ khóa nhưng không đủ quyền · **409** đã có bên khác làm rồi.
- **503** = server chưa cấu hình khóa (khóa chặt).

---

## 1.4. Tự gửi yêu cầu bằng `curl` và đọc kết quả

**① Khái niệm**

*curl* là công cụ dòng lệnh gửi yêu cầu HTTP — cách nhanh nhất để **thử API mà không cần viết client**.

| Tùy chọn | Ý nghĩa |
|---|---|
| `-X POST` | Chọn phương thức |
| `-H "Tên: giá trị"` | Thêm header |
| `-d '…'` | Body (tự chuyển sang POST) |
| `-v` | In **toàn bộ** yêu cầu và phản hồi |
| `-s` | Ẩn thanh tiến trình |
| `-w "%{http_code}"` | In mã trạng thái |

**② Trong đồ án** — dùng khi demo (thay `<KHÓA_BẾP>` bằng giá trị trong `.env`; **không chép khóa vào báo cáo**):

```bash
curl -s http://192.168.3.7:5000/api/menu                                # công khai → 200
curl -s -w " -> %{http_code}\n" http://192.168.3.7:5000/api/orders/pending   # không khóa → 401
curl -s -H "X-API-Key: <KHÓA_BẾP>" http://192.168.3.7:5000/api/orders/pending
curl -v -X POST http://192.168.3.7:5000/api/orders/done \
     -H "X-API-Key: <KHÓA_BẾP>" -H "Content-Type: application/json" \
     -d '{"order_code":"DH12345678"}'
```

Đọc `-v`: dòng `>` là client **gửi**, `<` là server **trả**, `*` là ghi chú của curl.

**③ Vì sao / Nếu không**

- **Khoanh vùng lỗi:** curl chạy được → lỗi ở **client** (firmware, phần mềm quản trị); curl cũng lỗi → lỗi ở **server hoặc mạng**.
- **Khi bảo vệ:** chạy curl trực tiếp cho thấy 401 → 200 → 403 → 409 là bằng chứng thuyết phục cho phần phân quyền và chống báo xong hai lần.

**④ Ghi nhớ**

- `curl -v` = xem **toàn bộ** cuộc hội thoại HTTP.
- **curl chạy được thì lỗi ở client; curl lỗi thì lỗi ở server hoặc mạng.**

---

# MỤC 2. REST VÀ THIẾT KẾ API CỦA ĐỒ ÁN

## 2.1. Tài nguyên và endpoint — bảng 26 endpoint

**① Khái niệm**

- *REST* (REpresentational State Transfer): phong cách thiết kế API xoay quanh *tài nguyên*; mỗi tài nguyên có một **địa chỉ**, thao tác thể hiện bằng **phương thức HTTP**.
- *Endpoint* = **phương thức + đường dẫn**. Cùng đường dẫn khác phương thức là **hai endpoint khác nhau** (`GET` và `POST /api/auth/recovery_email`, L533, L543).

**② Trong đồ án** — trích từ `api_server.py`. 🔓 không cần khóa · 🔑A khóa quản trị · 🔑K khóa bếp hoặc quản trị.

| Nhóm | Endpoint | Quyền | Hàm CSDL / việc làm | Ai gọi |
|---|---|---|---|---|
| **Thống kê** | `GET /api/stats?period=` (L382) | 🔑A | `get_stats_data(period)` | Trang chủ quản trị |
| | `GET /api/report?period=` (L391) | 🔑A | `admin_get_report_data` | Trang báo cáo |
| **Thực đơn** | `GET /api/menu` (L402) | 🔓 | `get_all_products` | Quản trị, MCP |
| | `GET /images/<filename>` (L410) | 🔓 | Gửi file ảnh món | Quản trị |
| | `POST /api/menu/add` (L422) | 🔑A | `admin_add_product` + lưu ảnh | Quản trị |
| | `POST /api/menu/update` (L431) | 🔑A | `admin_update_product` | Quản trị |
| | `POST /api/menu/delete` (L442) | 🔑A | `admin_delete_product` (xóa cả công thức) | Quản trị |
| | `POST /api/menu/toggle` (L451) | 🔑A | `admin_toggle_product_status` | Quản trị |
| **Cửa hàng** | `GET /api/store/info` (L464) | 🔑A | `get_store_info` | Quản trị |
| | `POST /api/store/update` (L470) | 🔑A | `update_store_info` | Quản trị |
| **Tài khoản** | `POST /api/auth/login` (L483) | 🔓 (giới hạn 5 lần sai) | `verify_admin_password` | Quản trị |
| | `POST /api/auth/forgot_password` (L504) | 🔓 | Tạo OTP, gửi email | Quản trị |
| | `POST /api/auth/reset_password` (L523) | 🔓 | Kiểm tra OTP, đặt mật khẩu mới | Quản trị |
| | `GET / POST /api/auth/recovery_email` (L533, L543) | 🔑A | Đọc / sửa email khôi phục | Quản trị |
| | `POST /api/auth/change_password` (L553) | 🔑A | Đổi mật khẩu | Quản trị |
| **Đơn hàng** | `GET /api/orders` (L572) | 🔑K | `admin_get_orders` (phân trang) | Quản trị, MCP |
| | `GET /api/orders/pending` (L634) | 🔑K | `admin_get_kitchen_pending_orders` | **Bếp**, MCP |
| | `POST /api/orders/done` (L697) | 🔑K | `admin_mark_order_done` + MQTT `order_done` | **Bếp**, MCP |
| **Kho** | `GET /api/inventory` (L727) | 🔑K | `get_all_ingredients` | Quản trị, **Bếp**, MCP |
| | `POST /api/inventory/add · update · delete · restock` (L736–791) | 🔑A | `admin_*_ingredient` + **gửi tồn kho qua MQTT** | Quản trị |
| **Công thức** | `GET /api/recipe` (L793) | 🔑K | `get_recipe_for_product` | Quản trị, MCP |
| | `POST /api/recipe/save` (L802) | 🔑A | `admin_save_recipe` | Quản trị |

- **Phần mềm quản trị** gọi 23 endpoint; **màn hình bếp** chỉ gọi **3** (`orders/pending`, `orders/done`, `inventory`); **Kitchen MCP** gọi 6.
- **Ứng dụng kiosk không gọi API nào**: chạy cùng Pi, gọi thẳng `database.py`.

> **Trước khi sửa:** có 27 endpoint, trong đó `/api/tts` không ai dùng và đang lỗi 403 từ dịch vụ Microsoft; `/api/stats` không ai dùng; **10 endpoint đọc không cần khóa**. **Sau khi sửa:** xóa `/api/tts`, trang chủ quản trị dùng `/api/stats`, chỉ thực đơn và ảnh còn công khai.

**Hình 1 (vẽ tay) — Ai gọi endpoint nào**

```
 [Phần mềm quản trị, laptop] ──23 endpoint, khóa A──────────────┐
 [Màn hình bếp ESP32] ──pending, inventory, done (khóa K)───────┤
 [Kitchen MCP, laptop] ──menu, orders, pending, recipe,          │
                         inventory, done (khóa K)────────────────┤
                                                                 ▼
                                    [Smart Kiosk API :5000, Pi] ──► database.py ──► kiosk.db
                                                                 ▲
 [Ứng dụng kiosk, Pi] ─────────── gọi thẳng database.py, KHÔNG qua API (cùng máy)
```

**③ Vì sao / Nếu không**

- **Gom mọi truy cập từ máy khác vào một API:** SQLite chỉ mở được trên cùng máy; API vừa là **cửa vào qua mạng**, vừa là **nơi kiểm tra quyền**.
- **Mỗi client chỉ dùng phần nó cần:** khóa bếp chỉ đọc được đơn/kho/công thức và **một** thao tác ghi — *nguyên tắc đặc quyền tối thiểu* (mục 4.2).
- **Endpoint không ai dùng là bề mặt tấn công thừa** → đã xóa `/api/tts`.

**④ Ghi nhớ**

- 26 endpoint, **7 nhóm**; chỉ **thực đơn, ảnh, đăng nhập/OTP** là không cần khóa.
- Đọc đơn/kho/công thức: **khóa K**; báo cáo, thống kê, cửa hàng và mọi thao tác ghi quản trị: **khóa A**.

---

## 2.2. Quy ước phản hồi `{"success", "reason"}`

**① Khái niệm**

*Hợp đồng API* (API contract): thỏa thuận về **định dạng** yêu cầu và phản hồi. Định dạng thống nhất thì client chỉ cần một hàm đọc kết quả cho mọi endpoint.

**② Trong đồ án**

| Dạng | Ví dụ thật | Endpoint |
|---|---|---|
| **Danh sách trần** | `[{...}, ...]` (34 món) | `GET /api/menu`, `/api/inventory`, `/api/recipe` |
| **Đối tượng phân trang** | `{"currentPage":1,"data":[…],"totalItems":620,"totalPages":2}` | `GET /api/orders` |
| **Đối tượng số liệu** | `{"revenue":12400000,"orders":620,"chart":[…]}` | `GET /api/stats` |
| **Kết quả thao tác** | `{"success":true,"order_code":…,"kitchen_status":"done"}` | các POST |
| **Lỗi** (mọi nguồn) | `{"success":false,"reason":"invalid_id"}` | mọi endpoint |

**Cơ chế lỗi thống nhất** (`api_server.py` L155–200):

```python
class ApiError(Exception):                       # L155: lỗi nghiệp vụ/đầu vào
    def __init__(self, reason, status=400): ...

@app.errorhandler(ApiError)                      # L164 → {"success":false,"reason":...}, mã tương ứng
@app.errorhandler(HTTPException)                 # L178 → 400/404/405/413/415 do Flask sinh: đổi sang JSON
@app.errorhandler(sqlite3.IntegrityError)        # L186 → 409 "constraint_violation"
@app.errorhandler(Exception)                     # L192 → 500 "internal_error"; chi tiết chỉ ghi log
```

Trường `reason` là **mã lỗi máy đọc được** (`snake_case` tiếng Anh). Phần mềm quản trị đổi sang câu tiếng Việt qua bảng `_REASON_MESSAGES` (`admin_bridge.py` L26).

Kiểm tra đầu vào `GET /api/orders` (L572–590): `limit` không phải số → **400 `invalid_pagination`**; quá lớn → **kẹp về 500**; `page` < 1 → kẹp về 1.

**③ Vì sao / Nếu không**

- **Có `reason`** → client phân biệt "sai khóa" với "món đang được dùng" và hiện đúng thông báo.
- **Mọi lỗi đều là JSON** → firmware ESP32 gọi `deserializeJson` không bao giờ gặp trang HTML.
- **Lỗi 500 không lộ chi tiết:** `traceback.print_exc()` chỉ ghi vào nhật ký dịch vụ (`journalctl -u kiosk-api`); client chỉ nhận `internal_error`.

> **Trước khi sửa:** lỗi do Flask sinh là **trang HTML**; `/api/tts` trả cả `str(e)` (URL dịch vụ, thông điệp lỗi thư viện) cho bất kỳ ai gọi.

**④ Ghi nhớ**

- Quy ước: `{"success": bool, "reason": "<mã_lỗi>"}` + mã HTTP phù hợp.
- Bốn `errorhandler`: **ApiError · HTTPException · IntegrityError · Exception** → luôn JSON.

---

## 2.3. So với REST chuẩn: hành động nằm trong đường dẫn

**① Khái niệm**

| Phương thức | Ý nghĩa | *Idempotent*? |
|---|---|---|
| `GET /items/5` | Đọc | ✔ |
| `POST /items` | Tạo mới | ✘ mỗi lần tạo thêm |
| `PUT` / `PATCH /items/5` | Sửa | PUT ✔ |
| `DELETE /items/5` | Xóa | ✔ |

*Idempotent*: gọi một hay nhiều lần **cho cùng kết quả** — quyết định client có được **tự động thử lại** hay không.

**② Trong đồ án** — kiểu "**RPC qua HTTP**": chỉ GET/POST, **động từ trong đường dẫn**, ID trong body.

| Việc | Đồ án | REST chuẩn |
|---|---|---|
| Xóa món | `POST /api/menu/delete` `{"id":5}` | `DELETE /api/menu/5` |
| Sửa món | `POST /api/menu/update` | `PUT /api/menu/5` |
| Bật/tắt món | `POST /api/menu/toggle` | `PATCH /api/menu/5` |
| Báo xong đơn | `POST /api/orders/done` | `PATCH /api/orders/DH…` |

**③ Vì sao / Nếu không**

- **Ưu:** đơn giản cho client nhúng (ESP32 `HTTPClient` có sẵn GET/POST); nhìn đường dẫn là biết hành động.
- **Nhược:** không tận dụng ngữ nghĩa HTTP; công cụ/proxy coi mọi POST là "không an toàn để thử lại".
- **Bù lại:** idempotent **ở tầng nghiệp vụ** — `orders/done` gọi 2 lần thì lần 2 trả 409 và không làm gì thêm (buổi 2, mục 5.3) → màn hình bếp **thử lại vẫn an toàn**.
- Trả lời hội đồng: *"API theo phong cách REST ở mức đơn giản: tài nguyên rõ ràng, JSON, mã trạng thái đúng nghĩa; chỉ dùng GET/POST và để động từ trong đường dẫn để đơn giản cho thiết bị nhúng; an toàn khi gọi lặp được bảo đảm bằng CSDL."*

**④ Ghi nhớ**

- **GET để đọc, POST cho mọi thao tác ghi**, động từ trong đường dẫn.
- `orders/done` idempotent nhờ **409 + UPDATE có điều kiện**.

---

## 2.4. Ánh xạ endpoint xuống hàm CSDL — và lỗi dây chuyền đã sửa

**① Khái niệm**

Hàm xử lý endpoint nên là *lớp mỏng* 4 bước: **đọc** yêu cầu → **kiểm tra** dữ liệu → **gọi** đúng một hàm nghiệp vụ → **đổi** kết quả thành mã HTTP + JSON.

**② Trong đồ án**

Mẫu chuẩn — `POST /api/orders/done` (L697–719):

```python
@require_kitchen_or_admin_key
def mark_order_done():
    data = request.get_json(silent=True) or {}            # 1. đọc (không ném lỗi)
    order_code = str(data.get('order_code', '')).strip()
    if not order_code: return jsonify({...}), 400         # 2. kiểm tra
    success = db.admin_mark_order_done(order_code)        # 3. gọi hàm nghiệp vụ
    if not success:   return jsonify({...}), 409          # 4. kết quả → mã HTTP
    publish_order_done(order_code)                        #    báo màn hình bếp qua MQTT
    return jsonify({...}), 200
```

Sau khi sửa, **mọi endpoint ghi** theo cùng mẫu nhờ các hàm kiểm tra dùng chung (L203–266):

| Hàm | Kiểm tra |
|---|---|
| `_json_body()` | `get_json(silent=True)`, không phải dict → `{}` |
| `_to_int(value, reason, minimum)` | từ chối `None`, **`bool`**, chữ, vô cực; ngưỡng tối thiểu |
| `_to_float(..., strictly_positive)` | từ chối `NaN`, vô cực; số dương |
| `_to_text(value, reason, max_length)` | bỏ khoảng trắng, bắt buộc có, giới hạn độ dài |
| `_to_bool(value, reason)` | chỉ nhận `true/false/0/1` |
| `_save_uploaded_image()` | đuôi ảnh hợp lệ, tên an toàn, **không trùng** |

Ví dụ `POST /api/menu/delete` sau khi sửa (L442–448):

```python
pid = _to_int(_json_body().get('id'), "invalid_id", minimum=1)   # sai → 400
if not db.admin_delete_product(pid):
    raise ApiError("product_not_found", 404)                      # không có → 404
return jsonify({"success": True})
```

**⚠ Lỗi dây chuyền phát hiện ở buổi học — trước và sau khi sửa:**

```
TRƯỚC KHI SỬA                                              SAU KHI SỬA
1) xóa món 2 (không công thức)  -> 200   (0.0 s)            1) xóa món 2                    -> 200 (0.0 s)
2) xóa món 1 (có công thức)     -> IntegrityError            2) xóa món 1 (có công thức)     -> 200, xóa cả công thức
3) xóa món 3 ngay sau đó        -> database is locked (10 s) 3) xóa món kế tiếp              -> 200 (0.0 s)
```

Nguyên nhân cũ: `admin_delete_product` mở giao dịch ghi bằng câu `DELETE`, ném lỗi khóa ngoại, **không có `try/finally`** → kết nối không rollback, không đóng → **giữ khóa ghi** tới khi Python thu gom → mọi thao tác ghi khác (theo cơ chế SQLite, kể cả kiosk chốt đơn) chờ `busy_timeout` 10 s rồi lỗi.

Cách sửa (`database.py`):

- `_open_db()` (L649): context manager **luôn rollback khi lỗi và luôn đóng kết nối**; mọi hàm quản trị dùng nó.
- `admin_delete_product` (L745): `BEGIN IMMEDIATE` → xóa `recipe` của món → xóa món → kiểm tra `rowcount` (không có món → `False` → API trả 404).
- `admin_delete_ingredient` (L957): nguyên liệu còn trong công thức món nào thì trả `in_use` → API **409 `ingredient_in_use`** (không xóa ngầm công thức của món khác).

**③ Vì sao / Nếu không**

- **Lớp mỏng đủ 4 bước** → lỗi đầu vào luôn ra **400 JSON**, lỗi nghiệp vụ ra **404/409**, client biết vì sao.
- **Một hàm thiếu `try/finally` có thể làm tê liệt thao tác ghi của cả hệ thống** — vì SQLite khóa ở mức **file**.
- **Đơn cũ không bị ảnh hưởng khi xóa món** vì đơn lưu bản chụp `items_json` (buổi 2, mục 3.4).

**④ Ghi nhớ**

- Hàm xử lý = **đọc → kiểm tra → gọi một hàm CSDL → đổi thành mã HTTP**.
- Kiểm tra đầu vào dùng chung: `_to_int`, `_to_float`, `_to_text`, `_to_bool`, `_json_body`.
- `_open_db()`: **luôn rollback + đóng** → không còn khóa CSDL bị treo (đã kiểm chứng).

---

# MỤC 3. FLASK TỪ BÊN TRONG

## 3.1. WSGI, đối tượng `app` và `@app.route`

**① Khái niệm**

- *WSGI* (Web Server Gateway Interface): **chuẩn giao tiếp** giữa **máy chủ web** (Werkzeug, Waitress, Gunicorn… — nhận byte từ mạng, phân tích HTTP) và **ứng dụng web** (Flask — quyết định trả lời gì).
- Ứng dụng chỉ là **một hàm** `app(environ, start_response)`: `environ` là dict chứa thông tin yêu cầu; `start_response(status, headers)` báo mã và header; giá trị trả về là body.
- *Flask*: micro-framework trên WSGI, lo **định tuyến**, đối tượng `request`, tạo phản hồi.
- Decorator `@app.route(...)` **đăng ký** hàm vào **bảng định tuyến** `app.url_map`.

**② Trong đồ án**

```python
app = Flask(__name__)                                  # L30
@app.route('/api/orders/done', methods=['POST'])       # L697: đăng ký đường dẫn + phương thức
@require_kitchen_or_admin_key                          # L698: bọc thêm kiểm tra khóa
def mark_order_done(): ...
```

Thí nghiệm — bảng định tuyến thật (bản trước khi xóa `/api/tts`):

```
Số quy tắc URL: 28            ← 27 endpoint lúc đó + 1 quy tắc /static Flask tự thêm
/api/auth/recovery_email ['GET', 'HEAD', 'OPTIONS'] -> get_recovery_email
/api/auth/recovery_email ['OPTIONS', 'POST']        -> update_recovery_email
/api/orders/done         ['OPTIONS', 'POST']        -> mark_order_done
```

- Cùng đường dẫn, khác phương thức → **hai hàm khác nhau**; Flask tự thêm `HEAD` (cho GET) và `OPTIONS`.
- Có đường dẫn nhưng sai phương thức → **405**; không có đường dẫn → **404** (nay đều trả JSON).

Thí nghiệm — gọi ứng dụng như một hàm, không cần máy chủ:

```python
body = b''.join(app(environ, start_response))      # environ: GET /api/store/info
# → 200 OK | {"store_address":"","store_email":"","store_name":"","store_phone":""}
```

`app` **chỉ là một hàm Python** — nhờ vậy `app.test_client()` (dùng trong 103 bài test) gọi thẳng được.

Thí nghiệm — decorator giữ tên hàm nhờ `functools.wraps`:

```
tên hàm sau decorator: mark_order_done | có __wrapped__: True
```

**③ Vì sao / Nếu không**

- **Thiếu `@wraps`:** mọi endpoint có khóa đều tên `decorated` → Flask báo lỗi lúc khởi động `View function mapping is overwriting an existing endpoint function`.
- **Thứ tự decorator:** `@app.route` phải **trên cùng**; đảo lại thì Flask đăng ký hàm **chưa được bọc** → endpoint **không kiểm tra khóa** (lỗ hổng âm thầm). Mọi endpoint trong đồ án đúng thứ tự (test `test_read_endpoints_require_key` kiểm chứng).
- **Tách máy chủ khỏi ứng dụng:** đổi từ máy chủ phát triển sang **Waitress** không phải sửa dòng mã Flask nào (mục 3.3).

**④ Ghi nhớ**

- WSGI: ứng dụng = **hàm `app(environ, start_response)`**.
- `@app.route` ghi vào **`app.url_map`**; sai đường dẫn 404, sai phương thức 405.
- **`@app.route` trên cùng**, decorator khóa nằm dưới, dùng **`@wraps`**.

---

## 3.2. `request`, `jsonify` và cặp (body, mã trạng thái)

**① Khái niệm**

- `request` là đối tượng **toàn cục** chứa yêu cầu **đang xử lý**, gắn với *ngữ cảnh* của **từng yêu cầu** → nhiều yêu cầu song song trên nhiều luồng **không lẫn dữ liệu**.

| Thuộc tính | Đọc gì | Đồ án dùng ở |
|---|---|---|
| `request.args` | Query string | `/api/orders`, `/api/stats`, `/api/report`, `/api/recipe` |
| `request.get_json(silent=True)` | Body JSON; **`None` nếu sai định dạng** | `_json_body()` L203, `orders/done` L700 |
| `request.form`, `request.files` | Form nhiều phần (tải ảnh) | `_product_form_fields()` L415, `_save_uploaded_image()` L253 |
| `request.headers` | Header | `_caller_role()` L280 |
| `request.remote_addr` | IP người gọi | giới hạn đăng nhập sai (L483) |

- `jsonify(obj)` chuyển sang JSON và đặt `Content-Type: application/json`.
- Hàm xử lý trả `(phản hồi, mã)`; không ghi mã → **200**. Có thể `raise ApiError(...)` để `errorhandler` trả lỗi.

**② Trong đồ án**

Lưu ảnh tải lên an toàn (`_save_uploaded_image`, L253–266):

```python
safe_name = secure_filename(file.filename)                 # chặn ../, ký tự lạ
extension = os.path.splitext(safe_name)[1].lower()
if not safe_name or extension not in ALLOWED_IMAGE_EXTENSIONS:   # .jpg .jpeg .png .webp .gif
    raise ApiError("invalid_image")
filename = f"{uuid.uuid4().hex[:8]}_{safe_name}"           # tiền tố ngẫu nhiên: không ghi đè
file.save(os.path.join(UPLOAD_FOLDER, filename))
```

Kết quả thật của `secure_filename`:

```
'../../.env'       → 'env'              ← chặn ghi ra ngoài thư mục ảnh (và nay bị từ chối vì không có đuôi ảnh)
'/etc/passwd'      → 'etc_passwd'
'ảnh bánh mì.jpg'  → 'anh_banh_mi.jpg'
```

Test tự động: tải 2 ảnh cùng tên `anh.jpg` → 2 tên file khác nhau đều kết thúc bằng `_anh.jpg`; tải file tên `../../.env` → **400 `invalid_image`**.

**③ Vì sao / Nếu không**

- **`silent=True`** = khác biệt giữa *400 có lý do* và *415/500*.
- **Không có `secure_filename`:** tên `../../.env` có thể ghi đè **file chứa khóa bí mật** — lỗ hổng *path traversal*.
- **Không có tiền tố ngẫu nhiên:** hai ảnh trùng tên thì ảnh sau ghi đè ảnh trước, kể cả của món khác (lỗi cũ, đã sửa).
- **Giới hạn 8 MB** (`MAX_CONTENT_LENGTH`): chặn tải file khổng lồ làm đầy thẻ nhớ Pi.

**④ Ghi nhớ**

- Đọc đầu vào: `args` · `get_json(silent=True)` · `form`/`files` · `headers`.
- Ảnh: **`secure_filename` + kiểm tra đuôi + tiền tố ngẫu nhiên + giới hạn 8 MB**.

---

## 3.3. Máy chủ: Werkzeug (phát triển) và Waitress (chạy thật); đa luồng

**① Khái niệm**

- `app.run()` chạy **máy chủ phát triển Werkzeug**; từ Flask 1.0 mặc định `threaded=True` (mỗi yêu cầu một luồng). Werkzeug tự cảnh báo *"Do not use it in a production deployment."*
- *Waitress*: máy chủ WSGI thuần Python cho môi trường thật, có **hàng đợi và số luồng cố định**, không cần biên dịch — hợp Raspberry Pi.
- `host='0.0.0.0'`: nhận yêu cầu trên **mọi card mạng** (máy khác gọi được); `127.0.0.1`: chỉ chính máy đó.

**② Trong đồ án** — `main()` (L828–844):

```python
def main():
    port = int(os.environ.get('KIOSK_API_PORT', '5000'))
    db.init_database()                      # tạo bảng nếu CSDL mới; chạy lại nhiều lần vẫn an toàn
    try:
        from waitress import serve
    except ImportError:
        serve = None
    if serve is not None:
        serve(app, host='0.0.0.0', port=port, threads=8)        # Pi: dùng Waitress
    else:
        app.run(host='0.0.0.0', port=port, threaded=True)       # chưa cài thì dùng Werkzeug
```

Log thật trên Pi sau khi triển khai: `--- API SERVER (ADMIN) chạy bằng Waitress trên cổng 5000 ---`.

Thí nghiệm — đa luồng so với đơn luồng (máy chủ Werkzeug, 4 yêu cầu cùng lúc, mỗi yêu cầu giả lập 0,5 s):

```
threaded=1: 4 yêu cầu song song mất 0.52 s; luồng: Thread-1 Thread-2 Thread-3 Thread-4
threaded=0: 4 yêu cầu song song mất 2.02 s; luồng: MainThread
```

Waitress cũng xử lý song song (`threads=8`).

**③ Vì sao / Nếu không**

- **Đa luồng là cần thiết:** phần mềm quản trị tải báo cáo trong khi màn hình bếp báo xong đơn — một luồng thì bếp phải **xếp hàng**.
- **Nối buổi 2:** đa luồng → **nhiều yêu cầu cùng ghi CSDL** ngay trong một tiến trình → cần WAL, `busy_timeout`, `BEGIN IMMEDIATE`, UPDATE có điều kiện. **Mỗi lần gọi hàm mở một kết nối SQLite riêng** (`get_connection()`), không dùng chung giữa các luồng (`sqlite3` mặc định cấm).
- **Fallback sang `app.run`:** Pi chưa cài Waitress vẫn chạy được — triển khai không bị gãy.
- **`init_database()` trong `main()`, không ở lúc import:** tránh việc chạy test (import module) vô tình ghi vào CSDL thật.
- **`0.0.0.0`** cần để laptop và màn hình bếp gọi được; đổi lại API mở ra mọi mạng Pi tham gia — vì vậy **đọc dữ liệu nhạy cảm cũng phải có khóa** (mục 4.7).

> **Trước khi sửa:** chạy bằng `app.run` (máy chủ phát triển); API không tự tạo bảng (CSDL mới báo `no such table`).

**④ Ghi nhớ**

- Pi chạy **Waitress (8 luồng)**; thiếu thì tự dùng **Werkzeug**.
- Đa luồng → **mỗi yêu cầu một kết nối SQLite riêng**.
- Đổi máy chủ **không sửa mã Flask** (nhờ WSGI).

---

## 3.4. Phục vụ file ảnh và đọc cấu hình từ `.env`

**① Khái niệm**

- `send_from_directory(thư_mục, tên)` gửi file **chỉ trong thư mục cho trước**, tự chặn `../`.
- *Biến môi trường / `.env`*: tách **cấu hình và bí mật** khỏi mã nguồn; `python-dotenv` nạp `.env` vào `os.environ`.

**② Trong đồ án**

```python
@app.route('/images/<filename>')                            # L410–412
def serve_image(filename):
    return send_from_directory(UPLOAD_FOLDER, filename)        # assets/images/
```

Thí nghiệm — thử đọc `.env` bằng đường dẫn vượt thư mục:

```
GET /images/../.env         -> 404
GET /images/..%2f.env       -> 404
GET /images/%2e%2e%2f.env   -> 404
```

Đọc cấu hình (L28–53): `load_dotenv(project_root/.env)`, rồi `ADMIN_API_KEY`, `KITCHEN_API_KEY`, `SMTP_*`, `MQTT_*`. `.env` trên Pi có đủ 13 biến, không biến nào rỗng (đã kiểm tra tên biến, không đọc giá trị).

**③ Vì sao / Nếu không**

- **Bí mật trong mã nguồn** → đẩy lên GitHub là **lộ khóa**; với `.env`, mỗi máy có khóa riêng.
- **`load_dotenv` không ghi đè** biến đã có sẵn → test đặt khóa giả từ dòng lệnh không bị `.env` thật lấn át.
- **Cấu hình đọc một lần lúc khởi động** → sửa `.env` phải `sudo systemctl restart kiosk-api`.

**④ Ghi nhớ**

- `send_from_directory` + `<filename>` → **không đọc được file ngoài thư mục ảnh**.
- Bí mật nằm trong **`.env`**; sửa xong phải khởi động lại dịch vụ.

---

# MỤC 4. XÁC THỰC VÀ PHÂN QUYỀN ★

## 4.1. Khóa API trong header `X-API-Key`

**① Khái niệm**

- *Xác thực* (authentication): **"bạn là ai?"**. *Phân quyền* (authorization): **"bạn được làm gì?"**.
- *Khóa API*: chuỗi bí mật dùng chung giữa server và client, gửi kèm **mỗi yêu cầu**.

**② Trong đồ án**

| Client | Khóa | Gửi ở đâu |
|---|---|---|
| Phần mềm quản trị (laptop) | `ADMIN_API_KEY` trong `.env` laptop | `admin_bridge.py` L22 `_AUTH_HEADERS`, gửi ở **mọi** yêu cầu qua `_request()` (L85) |
| Màn hình bếp ESP32 | khóa bếp trong `secrets.h` | `kds_firmware.ino` L1401 (GET) và L1609 (POST) |
| Kitchen MCP (laptop) | khóa bếp trong `~/.config/tro-ly-giong-noi/kitchen-mcp.env` (quyền 600) | `client.py` L356 (đọc), `write_client.py` L136 (ghi) |

**③ Vì sao / Nếu không**

- **Hợp thiết bị nhúng:** ESP32 chỉ thêm một header, không phải xử lý đăng nhập, cookie, làm mới token.
- **Điểm yếu cố hữu:** HTTP **không mã hóa** → ai bắt gói tin trong cùng Wi-Fi đọc được khóa. Khắc phục: **HTTPS (TLS)** — đưa vào hướng phát triển vì ESP32 phải quản lý chứng chỉ; trước mắt nên dùng Wi-Fi riêng cho thiết bị cửa hàng.

**④ Ghi nhớ**

- **Xác thực** = bạn là ai · **phân quyền** = bạn được làm gì.
- Ba client đều gửi khóa ở **mọi** yêu cầu (sau khi sửa); chưa có HTTPS.

---

## 4.2. Hai cấp quyền: khóa quản trị và khóa bếp

**① Khái niệm**

*Nguyên tắc đặc quyền tối thiểu* (least privilege): mỗi bên chỉ được **đúng quyền nó cần**; lộ một khóa thì thiệt hại bị giới hạn.

**② Trong đồ án**

| Decorator | Nhận khóa | Dùng cho |
|---|---|---|
| `require_api_key` (L290) | **Chỉ khóa quản trị** | thống kê, báo cáo, cửa hàng, tài khoản, và mọi thao tác ghi thực đơn/kho/công thức |
| `require_kitchen_or_admin_key` (L322) | Khóa quản trị **hoặc** khóa bếp | đọc `orders`, `orders/pending`, `inventory`, `recipe`; ghi **duy nhất** `orders/done` |

Hàm `_caller_role()` (L280) trả `"admin"`, `"kitchen"` hoặc `None`; `require_api_key` dùng nó để phân biệt **401** (không phải ai cả) và **403** (là bếp nhưng không đủ quyền).

Bảng quyền thật (test `test_read_endpoints_require_key` và kiểm tra trên Pi):

```
                           không khóa    khóa bếp   khóa quản trị
GET  /api/orders/pending          401         200            200
GET  /api/inventory               401         200            200
GET  /api/report                  401         403            200
GET  /api/stats                   401         403            200
POST /api/menu/delete             401         403            200/404
POST /api/orders/done             401     200/409        200/409
GET  /api/menu                    200         200            200
```

**③ Vì sao / Nếu không**

- Khóa bếp nằm **trong firmware ESP32**; ai cầm bo mạch có thể đọc flash. Nếu dùng khóa quản trị, kẻ đó có thể **sửa giá, xóa món, đổi email khôi phục rồi chiếm tài khoản**. Với khóa bếp, tệ nhất là **đọc đơn/kho** và **báo xong đơn giả**.
- Khóa bếp **không đọc được doanh thu** (`/api/report`, `/api/stats` → 403).

**④ Ghi nhớ**

- **2 cấp:** quản trị (mọi thứ) · bếp (đọc đơn/kho/công thức + báo xong đơn).
- **401** = không biết bạn là ai · **403** = biết nhưng không cho phép.

---

## 4.3. Decorator kiểm tra khóa — đọc tới mã nguồn

**① Khái niệm**

- *Decorator*: hàm nhận hàm `f`, trả về **hàm mới bọc quanh `f`**, chạy **kiểm tra trước**; không đạt thì trả lỗi, **`f` không bao giờ chạy**.
- `@ten_decorator` trên `def f` = `f = ten_decorator(f)`.
- *Closure*: hàm bên trong (`decorated`) **nhớ** biến `f` của hàm bên ngoài.

**② Trong đồ án** — `require_api_key`, L290–319:

```python
def require_api_key(f):
    @wraps(f)                                   # giữ tên, docstring của f
    def decorated(*args, **kwargs):
        if not ADMIN_API_KEY:
            return jsonify({...}), 503          # (a) server chưa cấu hình → khóa chặt
        role = _caller_role()                   # đọc X-API-Key, so sánh hằng thời gian
        if role is None:
            return jsonify({...}), 401          # (b) thiếu/sai khóa
        if role != "admin":
            return jsonify({...}), 403          # (c) khóa bếp: không đủ quyền
        return f(*args, **kwargs)               # (d) đạt → gọi hàm gốc
    return decorated
```

**Hình 3 (vẽ tay) — Luồng kiểm tra khóa**

```
 Yêu cầu ──► [Flask tìm hàm trong url_map] ──► decorated()
                                                 │
                         server có khóa?  ── không ──► 503 api_key_not_configured
                                                 │ có
                         X-API-Key khớp?  ── không ──► 401 unauthorized
                                                 │ khớp
                         đủ quyền?        ── không ──► 403 forbidden   (chỉ require_api_key)
                                                 │ đủ
                                                 ▼
                                    hàm xử lý ──► database.py ──► 200 / 400 / 404 / 409
```

**③ Vì sao / Nếu không**

- **Viết một lần, dùng cho mọi endpoint cần khóa**; chép tay vào từng hàm thì quên một chỗ là có lỗ hổng.
- **Kiểm tra trước khi đọc body:** yêu cầu không khóa bị loại ngay, không tốn công phân tích JSON, truy vấn CSDL.

**④ Ghi nhớ**

- Decorator = **bọc hàm**: kiểm tra → (đạt) gọi `f` / (không đạt) trả lỗi.
- Thứ tự: **503 → 401 → 403 → gọi hàm**.

---

## 4.4. `hmac.compare_digest` — chống dò khóa bằng thời gian

**① Khái niệm**

- `==` **dừng ở ký tự sai đầu tiên** → khóa đoán đúng càng nhiều ký tự đầu thì so sánh **càng lâu**.
- *Tấn công kênh phụ thời gian* (timing attack): đo thời gian phản hồi để đoán **từng ký tự**.
- `hmac.compare_digest`: so sánh **thời gian không đổi**, luôn duyệt hết chuỗi.

**② Trong đồ án** — `_key_matches`, L270–277:

```python
def _key_matches(provided, expected):
    if not expected:                          # khóa trên server rỗng → luôn từ chối
        return False
    return hmac.compare_digest(
        str(provided).encode("utf-8"),        # đổi sang bytes: str ngoài ASCII sẽ làm compare_digest báo lỗi
        str(expected).encode("utf-8"),
    )
```

OTP cũng so sánh bằng `compare_digest` (`database.py`, hàm `verify_and_reset_password`); firmware so chữ ký HMAC MQTT bằng `constantTimeEquals`.

Đo thật (khóa giả dài 2 triệu ký tự để phóng đại):

```
==              sai ký tự đầu:     0.1 µs | sai ký tự cuối:     51.2 µs   ← lệch ~500 lần
compare_digest  sai ký tự đầu:  2005.4 µs | sai ký tự cuối:   2008.6 µs   ← như nhau
```

**③ Vì sao / Nếu không**

- Với khóa ~40 ký tự, chênh lệch chỉ vài nano giây và bị nhiễu mạng che; tấn công qua mạng **khó nhưng đã được chứng minh** với đo thống kê rất nhiều lần.
- **`if not expected: return False`:** thiếu dòng này, server chưa cấu hình khóa (chuỗi rỗng) và client gửi header rỗng thì `"" == ""` → **ai cũng vào được**.

**④ Ghi nhớ**

- `==` dừng sớm → lộ thông tin qua thời gian; **`compare_digest`** → thời gian không đổi.
- Khóa rỗng **luôn bị từ chối**.

---

## 4.5. Khóa chặt khi thiếu cấu hình — 503

**① Khái niệm**

*Fail closed*: cấu hình bảo mật thiếu/hỏng thì **từ chối tất cả**. Ngược lại *fail open*: **cho qua tất cả**.

**② Trong đồ án**

- `require_api_key` (L292–302) và `require_kitchen_or_admin_key` (L325–330): chưa có khóa → **503 `api_key_not_configured`**, in `[TỪ CHỐI] ADMIN_API_KEY chưa cấu hình`.
- Test `test_missing_server_api_key_fails_closed`: server không khóa, client gửi bất kỳ → 503, **hàm CSDL không được gọi**.
- Cùng nguyên tắc ở: máy chủ webhook SePay (thiếu token → từ chối mọi request); Gateway giọng nói (công cụ ghi **luôn cần xác nhận** khi chưa bật chế độ tự động, `policy.py`); Kitchen MCP (**ghi tắt mặc định**).

**③ Vì sao / Nếu không**

- Kiểu *"chưa cấu hình khóa thì bỏ qua kiểm tra"* → **quên tạo `.env` khi cài Pi = API mở toang**, không ai biết.
- Khóa chặt → quên cấu hình **lộ ra ngay**, buộc người cài sửa.

> **Lỗi test phát hiện khi sửa:** test 503 từng **hỏng** vì `KITCHEN_API_KEY` trong `.env` thật lọt vào test (có khóa bếp → không còn 503). Đã sửa test để xóa khóa bếp trước khi chạy — bài học: **test phải độc lập với cấu hình máy**.

**④ Ghi nhớ**

- **Fail closed:** thiếu khóa → **503**; quên cấu hình lộ ra ngay thay vì âm thầm mở cửa.

---

## 4.6. Đăng nhập, quên mật khẩu bằng OTP, đổi mật khẩu

**① Khái niệm**

- *Băm mật khẩu*: CSDL lưu **kết quả băm một chiều** bằng hàm **cố ý chậm** (scrypt) kèm *salt* ngẫu nhiên.
- *OTP*: mã **dùng một lần**, **có hạn**, gửi qua kênh khác (email).
- *Giới hạn số lần thử* (rate limiting): sai quá số lần thì tạm khóa, chống dò mật khẩu.

**② Trong đồ án**

Băm bằng Werkzeug; dạng lưu: `scrypt:32768:8:1$<salt>$<kết quả băm>`.

**Đăng nhập** (`/api/auth/login`, L483–502) và bộ đếm (L344–375):

```python
LOGIN_MAX_FAILURES = 5          # sai 5 lần
LOGIN_LOCK_SECONDS = 300        # khóa 5 phút theo địa chỉ IP
_login_guard = threading.Lock() # bảo vệ bộ đếm khi nhiều luồng cùng ghi
```

- Đang bị khóa → **429 `too_many_attempts`** kèm header `Retry-After` (số giây còn lại).
- Sai mật khẩu → **401 `wrong_password`**; đúng → 200 và **xóa bộ đếm**.
- Đồng hồ dùng `time.monotonic()` (không bị ảnh hưởng khi chỉnh giờ hệ thống).

Thí nghiệm (test `test_login_locks_after_five_failures`, và qua phần mềm quản trị):

```
Sai lần 1..5                      -> 401 "Sai mật khẩu, vui lòng thử lại."
Lần 6 (dù đúng mật khẩu 123456)    -> 429 "Sai quá nhiều lần, vui lòng thử lại sau 300 giây."
```

**OTP** (`database.py`): 6 chữ số từ `secrets`, hạn **10 phút**, chờ **60 s** giữa hai lần xin (→ 429), sai **5 lần thì hủy mã**; đặt lại mật khẩu trong `BEGIN IMMEDIATE`. Kết quả thật: sai 1–4 → `invalid_otp`, lần 5 → `too_many_attempts`, lần 6 → `no_otp_requested`.

**Đổi mật khẩu** (L553–569): cần khóa quản trị + mật khẩu cũ; sai mật khẩu cũ hoặc mật khẩu mới < 6 ký tự → **400**.

**③ Vì sao / Nếu không**

- **Trước khi sửa:** sai 30 lần liên tiếp đều 200, ~15 lần/giây, **không bị khóa**. **Sau khi sửa:** tối đa 5 lần mỗi 5 phút cho mỗi IP.
- **Bộ đếm trong bộ nhớ:** đơn giản, đủ cho một cửa hàng; khởi động lại dịch vụ thì bộ đếm về 0 (chấp nhận được).
- **Đăng nhập chỉ là cổng giao diện:** không cấp token; quyền ghi thật nằm ở `ADMIN_API_KEY` trong `.env` laptop. Ai lấy được `.env` gọi được API quản trị mà không cần mật khẩu → **bảo vệ file `.env`** là quan trọng.
- Mật khẩu mặc định `123456` cần đổi khi triển khai thật.

**④ Ghi nhớ**

- Mật khẩu: **scrypt + salt**. Đăng nhập: **5 lần sai/5 phút → 429**.
- OTP: **6 số · 10 phút · chờ 60 s · sai 5 lần hủy**.
- Quyền ghi thật nằm ở **khóa API**, không ở mật khẩu đăng nhập.

---

## 4.7. Bề mặt tấn công: trước và sau khi siết

**① Khái niệm**

*Bề mặt tấn công* (attack surface): tổng các điểm người ngoài có thể tương tác; mỗi endpoint **không cần khóa** là một phần.

**② Trong đồ án**

| Endpoint | Trước khi sửa | Sau khi sửa | Lý do |
|---|---|---|---|
| `/api/orders`, `/api/report`, `/api/stats` (doanh thu, từng đơn) | 🔓 | 🔑K / 🔑A / 🔑A | dữ liệu kinh doanh |
| `/api/inventory`, `/api/recipe` | 🔓 | 🔑K | công thức là bí quyết |
| `/api/orders/pending` | 🔓 | 🔑K | |
| `/api/store/info` | 🔓 | 🔑A | |
| `/api/tts` | 🔓, lỗi 403, lộ `str(e)` | **đã xóa** | không ai dùng |
| `/api/auth/login` | 🔓, không giới hạn | 🔓, **5 lần/5 phút** | |
| `/api/menu`, `/images/…` | 🔓 | 🔓 | thông tin công khai |

Kiểm tra thật từ laptop tới Pi: không khóa đọc đơn chờ → **401**; khóa bếp đọc báo cáo → **403**.

**③ Vì sao / Nếu không**

- Do `host='0.0.0.0'`, mọi thiết bị **cùng Wi-Fi với Pi** gọi được API. Trước khi sửa, khách dùng chung Wi-Fi có thể đọc **toàn bộ doanh thu và đơn hàng**.
- **Siết phía server phải đi cùng sửa phía client** (phần mềm quản trị, firmware, MCP gửi khóa ở GET) và **triển khai theo thứ tự**: cập nhật client trước (bản mới vẫn chạy với server cũ), server sau cùng (Phụ lục A).
- **Còn lại cho hướng phát triển:** HTTPS; Wi-Fi riêng cho thiết bị cửa hàng.

**④ Ghi nhớ**

- Sau khi sửa: chỉ **thực đơn, ảnh, đăng nhập/OTP** là công khai.
- Siết server ⇒ phải **sửa client** và **triển khai client trước, server sau**.

---

# MỤC 5. CÁC BÊN GỌI API (PHÍA CLIENT)

## 5.1. Phần mềm quản trị — `requests`, hàm gọi chung, tải nền

**① Khái niệm**

- `requests`: thư viện HTTP phổ biến nhất của Python; `params=` ghép query string; `json=` chuyển JSON và đặt `Content-Type`.
- *Slot* (PySide6/Qt): hàm Python đánh dấu `@Slot` để QML gọi.
- *Signal*: Python phát tín hiệu, QML nhận qua `Connections`; tín hiệu phát từ luồng khác được Qt **chuyển về luồng giao diện** (queued connection).

**② Trong đồ án** — `admin_app/admin_bridge.py` (360 dòng)

**Hàm gọi API dùng chung** `_request()` (L85–107):

```python
def _request(self, method, path, timeout=5, **kwargs):
    try:
        res = requests.request(method, f"{PI_URL}{path}", headers=_AUTH_HEADERS,   # luôn gửi khóa
                               timeout=timeout, **kwargs)                          # luôn có timeout
    except requests.RequestException:
        self._last_error = _reason_message("network_error"); return False, None
    data = res.json() (nếu được)
    ok = 200 <= res.status_code < 300 and data.get("success") is not False
    if not ok: self._last_error = _reason_message(data.get("reason"), data)          # câu tiếng Việt
    return ok, data
```

- QML hiện lỗi bằng `adminBackend.lastErrorMessage()` (L118), ví dụ: *"Nguyên liệu đang có trong công thức món – hãy sửa công thức trước."*, *"Sai quá nhiều lần, vui lòng thử lại sau 300 giây."*
- `defaultImageFolder()` (L110): hộp chọn ảnh món luôn mở sẵn ở `<dự án>/assets/images` (tính theo vị trí phần mềm, không ghi cứng đường dẫn).

**Trang chủ tải nền** (`requestStats`, L137; QML `DashboardContent.qml`):

```python
statsReady = Signal(str)                              # L68
def requestStats(self):
    if not self._stats_busy.acquire(blocking=False): return   # lần trước chưa xong thì bỏ qua
    threading.Thread(target=lambda: (self.statsReady.emit(self._load_stats()),
                                     self._stats_busy.release()), daemon=True).start()
```

```qml
Connections { target: adminBackend; function onStatsReady(jsonStr) { updateDashboard(jsonStr) } }
Timer { interval: 5000; onTriggered: adminBackend.requestStats() }
```

`_load_stats()` (L122) gọi `GET /api/stats?period=day|week|month` — **SQL ở server tính sẵn**.

**Trước và sau khi sửa** (thí nghiệm thật):

| | Trước | Sau |
|---|---|---|
| Doanh thu trang chủ, 620 đơn × 20.000 đ | **10.000.000 đ** (thiếu 2,4 triệu: client tự cộng `limit=9999` nhưng API kẹp 500) | **12.400.000 đ** (đúng) |
| Pi mất mạng, trang chủ cập nhật 5 s/lần | giao diện **đứng 3 s** mỗi lần | `requestStats` trả về sau **0,002 s**, tín hiệu về luồng giao diện |
| Thông báo lỗi | "Lỗi: không xóa được!" chung chung | đúng lý do (`reason`) |
| Gửi khóa khi đọc | ✘ | ✔ |

**③ Vì sao / Nếu không**

- **Luôn đặt `timeout`:** mặc định `requests` **chờ vô hạn**.
- **Không tính trùng ở hai nơi:** lỗi doanh thu cũ sinh ra vì trang chủ tự cộng ở client còn trang báo cáo tính bằng SQL. Nay **một nguồn sự thật**: SQL ở server.
- **Tải nền chỉ cho việc lặp lại tự động** (trang chủ 5 s/lần); thao tác người bấm vẫn đồng bộ, có timeout 3–5 s — đơn giản hơn và người dùng chờ kết quả là tự nhiên.

**④ Ghi nhớ**

- `_request()`: **luôn khóa + luôn timeout + ghi lý do lỗi**.
- Trang chủ: **`/api/stats` (SQL) + luồng nền + signal**.

---

## 5.2. Màn hình bếp ESP32 — `HTTPClient`, hai task, hàng đợi, thử lại có nghỉ

**① Khái niệm**

- `HTTPClient` (Arduino-ESP32): `begin(url)` → `addHeader` → `GET()`/`POST(body)` → `getString()` → `end()`.
- *FreeRTOS task*: luồng trên vi điều khiển; ESP32-S3 có **2 nhân**.
- *Hàng đợi* (queue): kênh an toàn chuyển dữ liệu giữa task.
- *Exponential backoff*: thử lại với khoảng nghỉ **tăng dần**.

**② Trong đồ án** — `kds_firmware.ino`

Hai task trên hai nhân (L1838–1842): `TaskUI` (nhân 1: LVGL, chạm) và `TaskNetwork` (nhân 0: Wi-Fi, MQTT, **mọi HTTP**).

Đọc dữ liệu (`httpGetJson`, L1395–1412): timeout kết nối 2,5 s + phản hồi 4 s, **gửi khóa bếp** (L1401). Khởi động: thử 5 s/lần (`INITIAL_SYNC_RETRY_MS`, L85); sau đó **30 s/lần** (`PERIODIC_SYNC_MS`, L86) — lưới an toàn khi lỡ tin MQTT.

Báo xong đơn (`postDoneOrder`, L1602–1630) trả **3 kết quả** (`enum DoneResult`, L143):

```cpp
if (status == 409) return DONE_OK;                               // đã xong ở nơi khác → gỡ thẻ
if (status == 400 || status == 401 || status == 403) return DONE_FATAL;   // thử lại vô ích
if (status < 200 || status >= 300) return DONE_RETRY;            // mạng, 5xx → thử lại
```

Xử lý hàng đợi (`processDoneQueue`, L1642–1678):

```
lấy tin khỏi hàng đợi
 ├─ chưa hết giờ nghỉ (notBeforeMs) → trả lại cuối hàng, chưa gửi
 ├─ gửi → DONE_OK            → gỡ thẻ
 ├─ DONE_RETRY, < 3 lần      → nghỉ 1 s, 2 s, 4 s rồi thử lại
 ├─ DONE_FATAL / hết lượt    → thẻ chuyển "GỬI LẠI"
 └─ không lấy được khóa giao diện → trả tin về hàng đợi (tránh thẻ kẹt "ĐANG LƯU...")
```

Nhận MQTT `kitchen/order_done` (L32, L1543–1553): kiểm chữ ký HMAC rồi **gỡ thẻ ngay** khi đơn được báo xong ở nơi khác.

Log serial thật sau khi nạp firmware mới (với server mới trên Pi):

```
[MQTT] Subscribe order=1 inventory=1 done=1
[SYNC] Inventory result=1
[SYNC] Orders pending=0 shown=0 removed=0
[NET] WiFi=3 MQTT=1
```

**Hình 4 (vẽ tay) — Hai task và hàng đợi**

```
[TaskUI, nhân 1]  chạm HOÀN THÀNH ─► doneOrderQueue {mã đơn, retryCount, notBeforeMs} ─► giao diện vẫn mượt
[TaskNetwork, nhân 0]  processDoneQueue ─► postDoneOrder ─► OK: gỡ thẻ │ RETRY: nghỉ 1-2-4 s │ FATAL: GỬI LẠI
                       mqttCallback("kitchen/order_done") ─► gỡ thẻ ngay
                       mỗi 30 s: GET /api/orders/pending ─► gỡ thẻ đơn không còn chờ
```

**③ Vì sao / Nếu không**

- **Tách mạng khỏi giao diện:** một yêu cầu chậm có thể mất 6,5 s; chạy trên task giao diện thì màn hình **đứng hình**.
- **409 = thành công:** nửa còn lại của tính idempotent (server không báo xong hai lần, client hiểu đúng 409).
- **Nghỉ tăng dần:** trước khi sửa, 3 lần thử lại cách nhau ~5 ms — mất mạng 1 giây là hết lượt. **Không thử lại khi 4xx:** sai khóa thì thử lại vô ích.
- **Lỗi sẵn có đã vá:** trước đây, không lấy được khóa giao diện thì tin nhắn bị bỏ → thẻ **kẹt mãi ở "ĐANG LƯU..."** (đồng bộ 30 s cũng bỏ qua thẻ đang lưu).

**④ Ghi nhớ**

- **HTTP ở nhân 0, giao diện ở nhân 1, nối bằng hàng đợi.**
- `DoneResult`: **OK (200/409) · RETRY (mạng/5xx, nghỉ 1-2-4 s) · FATAL (400/401/403)**.
- Gỡ thẻ đơn xong ở nơi khác: **MQTT `order_done` ngay**, đồng bộ 30 s làm dự phòng.

---

## 5.3. Kitchen MCP — `urllib` và ngoại lệ có kiểu

**① Khái niệm**

- `urllib.request`: thư viện HTTP **có sẵn trong Python**; mã 4xx/5xx ném `HTTPError`.
- *Ngoại lệ có kiểu*: mỗi loại lỗi một lớp riêng.
- *Kiểm tra trước khi gửi*: loại dữ liệu sai ngay ở client.

**② Trong đồ án** — `services/kitchen_mcp/src/kitchen_mcp/`

- Hai client tách biệt: `KioskReadClient` (đọc) và `KioskWriteClient` (chỉ `mark_order_done`).
- **Đọc gửi khóa bếp** (`client.py` `_get_json`, L339–365): có `kiosk_api_key` thì thêm `X-API-Key`. Test mới: `test_read_requests_send_kitchen_key_when_configured`.
- **Ghi tắt mặc định** (`write_client.py`): phải bật `KITCHEN_WRITE_ENABLED=true` và có `KIOSK_API_KEY`.
- **Kiểm tra mã đơn trước khi gửi:** chuẩn hóa chữ HOA, khớp `ORDER_ID_PATTERN` = **`DH` + 8 chữ số** (`client.py` L22).
- **Đổi mã HTTP thành ngoại lệ** (`_raise_http_error`, `write_client.py` L230): 401 → `KioskUnauthorizedError`, **409 → `KioskWriteConflictError`**, 503 → `KioskWriteDisabledError`, mất mạng → `KioskConnectionError`.
- **Kiểm tra phản hồi chặt:** mã 200, JSON hợp lệ, `success is True`, **`order_code` trả về trùng mã đã gửi**.

**Cấu hình khi chạy thật** (`~/.config/tro-ly-giong-noi/kitchen-mcp.env`, quyền 600, do `cai_dat_laptop.sh` tạo):

```
KITCHEN_DATA_MODE=live
KIOSK_API_BASE_URL=http://192.168.3.7:5000
KIOSK_API_KEY=<khóa bếp>
KITCHEN_WRITE_ENABLED=true
```

Kiểm tra thật: `chế độ: live | ghi: True | có khóa: True | nguyên liệu: 62 | món: 34`.

**③ Vì sao / Nếu không**

- **MCP không đọc file `.env`, chỉ đọc biến môi trường** → chạy `kitchen-mcp` trần sẽ ở chế độ **`fixture`** (dữ liệu mẫu). Vì vậy dịch vụ systemd nạp file cấu hình riêng.
- **Ghi tắt mặc định:** trợ lý dựa trên mô hình ngôn ngữ có thể "hiểu nhầm" và gọi công cụ ghi — *an toàn theo mặc định*.
- **409 với MCP là ngoại lệ, không phải thành công:** người nói "xong đơn DH…" cần được báo lại nếu đọc sai mã.

**④ Ghi nhớ**

- `urllib` có sẵn; **mỗi mã lỗi → một ngoại lệ có kiểu**.
- Đọc và ghi đều gửi **khóa bếp**; ghi **tắt mặc định**; mã đơn kiểm tra **`DH` + 8 số**.

---

## 5.4. So sánh ba client

| Tiêu chí | Phần mềm quản trị | Màn hình bếp ESP32 | Kitchen MCP |
|---|---|---|---|
| Thư viện | `requests` | `HTTPClient` | `urllib` |
| Timeout | đọc 3 s, ghi 5 s, OTP 10 s | kết nối 2,5 s + phản hồi 4 s | 5 s (cấu hình được) |
| Gửi khóa khi ĐỌC | ✔ (sau khi sửa) | ✔ (sau khi sửa) | ✔ (sau khi sửa) |
| Thử lại | ✘ (người dùng bấm lại) | ✔ 3 lần, nghỉ 1-2-4 s, không thử khi 4xx | ✘ (trợ lý báo lại người nói) |
| Xử lý 409 | hiện lý do | **= thành công**, gỡ thẻ | ngoại lệ riêng |
| Phân biệt loại lỗi | ✔ `reason` → câu tiếng Việt | ✔ OK / RETRY / FATAL | ✔ 5 loại ngoại lệ |
| Giao diện đứng khi mạng chậm | trang chủ: không (luồng nền); nút bấm: tối đa 3–5 s | không (task riêng) | không có giao diện |

**Ghi nhớ:** client tốt = **timeout + thử lại có kiểm soát + phân biệt loại lỗi + không chặn giao diện**. Ba client xử lý 409 khác nhau và **đều đúng** với ngữ cảnh của mình.

---

# MỤC 6. TRUY VẾT ĐẦU CUỐI HAI YÊU CẦU

## 6.1. Yêu cầu 1: bếp chạm HOÀN THÀNH — từ ngón tay tới CSDL

**① Khái niệm**

- *Truy vết đầu cuối*: theo một yêu cầu qua **mọi tầng** (thiết bị → mạng → máy chủ web → phân quyền → nghiệp vụ → CSDL) rồi theo phản hồi ngược lại.
- *Sơ đồ tuần tự*: mỗi thành phần một cột dọc, thông điệp là mũi tên ngang, **thời gian chạy từ trên xuống**.

**② Trong đồ án**

**Hình 2 (vẽ tay) — Sơ đồ tuần tự báo xong đơn**

```
 TaskUI     Hàng đợi   TaskNetwork     Waitress/Flask   decorator      mark_order_done   database.py     kiosk.db     MQTT broker
 (nhân 1)              (nhân 0)        (Pi :5000)       (L322)         (L697)            (L1055)
   │chạm      │            │               │               │                │                │              │             │
   │─push────►│            │               │               │                │                │              │             │
   │ĐANG LƯU  │─receive───►│               │               │                │                │              │             │
   │          │            │─POST /api/orders/done (X-API-Key, JSON)───────►│                │              │             │
   │          │            │               │─url_map──────►│ có khóa? khớp? │                │              │             │
   │          │            │               │               │──đạt──────────►│ get_json(silent)               │             │
   │          │            │               │               │                │───────────────►│BEGIN IMMEDIATE│             │
   │          │            │               │               │                │                │─UPDATE…WHERE─►│             │
   │          │            │               │               │                │                │◄─rowcount=1──│             │
   │          │            │               │               │                │◄──True─────────│ COMMIT       │             │
   │          │            │               │               │                │─publish_order_done (ký HMAC)────────────────►│
   │          │            │◄─200 {"success":true,"kitchen_status":"done"}───│                │              │             │
   │◄── gỡ thẻ (giữ lvglMutex) ─┤          │               │                │                │              │             │
```

| # | Tầng | Việc | Vị trí | Nếu lỗi |
|---|---|---|---|---|
| 1 | Giao diện ESP32 | Chạm → đẩy `{mã đơn, retryCount=0}` vào hàng đợi | TaskUI | — |
| 2 | Mạng ESP32 | Lấy khỏi hàng đợi, tạo JSON, 2 header, `POST` | `postDoneOrder` L1602 | mạng → RETRY (nghỉ 1-2-4 s) |
| 3 | Máy chủ | Waitress nhận, gọi `app(environ, …)` | `main()` L828 | — |
| 4 | Flask | Tra `url_map` | | 404 / 405 (JSON) |
| 5 | Decorator | Có khóa? Khớp (`compare_digest`)? | L322–338 | 503 / 401 |
| 6 | Hàm xử lý | `get_json(silent=True)`, lấy `order_code` | L700–706 | thiếu → 400 |
| 7 | Nghiệp vụ | `BEGIN IMMEDIATE` + `UPDATE … WHERE payment_status='PAID' AND kitchen_status='pending'` | `database.py` L1055 | `rowcount=0` → `False` |
| 8 | Hàm xử lý | `False` → **409**; đúng → phát MQTT `order_done` → 200 | L707–719 | |
| 9 | Mạng ESP32 | 200 / **409** → DONE_OK | L1622 | 4xx → FATAL; khác → RETRY |
| 10 | Giao diện | Giữ `lvglMutex`, gỡ thẻ | `processDoneQueue` L1642 | hết lượt → **GỬI LẠI** |

Bước 5–8 kiểm chứng bằng test HTTP thật (mục 6.3): **401 → 401 → 200 → 409**.

**Biến thể — báo xong bằng giọng nói:**

```
 Đầu bếp nói "xong đơn DH…" ─► thiết bị giọng nói ESP32 ─► Voice Gateway (laptop :8000)
   ─► Whisper (nhận dạng) ─► NLU / mô hình ngôn ngữ chọn công cụ mark_order_done
   ─► ToolExecutionPolicy: công cụ GHI → REQUIRE_CONFIRMATION     (policy.py, evaluate L300)
   ─► thiết bị hỏi lại, đầu bếp xác nhận ─► resume_confirmed_tool  (orchestrator.py L318)
   ─► Kitchen MCP (:3001) ─► KioskWriteClient: kiểm tra DH+8 số, POST kèm khóa bếp
   ─► [bước 3 → 8 như trên] ─► MQTT order_done ─► màn hình bếp GỠ THẺ NGAY
   ─► Gateway đọc "Đã hoàn tất đơn DH…" (orchestrator.py L565)
```

**③ Vì sao / Nếu không**

- **Mỗi tầng chặn một loại lỗi** → khoanh vùng theo mã: 404/405 đường dẫn · 503/401/403 khóa · 400 dữ liệu · 409 trạng thái đơn.
- **Hai lớp chống báo xong trùng ở hai tầng:** CSDL (UPDATE có điều kiện) và HTTP (409 + client hiểu 409).
- **Bước xác nhận giọng nói** chặn nhận dạng nhầm số đơn hoặc mô hình tự gọi công cụ ghi.

> **Trước khi sửa:** API không phát tin khi báo xong; đơn báo xong bằng giọng nói **còn trên màn hình bếp tới 30 s** (chờ đồng bộ định kỳ). **Sau khi sửa:** MQTT `kitchen/order_done` → gỡ thẻ ngay.

**④ Ghi nhớ**

- **Chạm → hàng đợi → task mạng → POST → Waitress → url_map → decorator → hàm xử lý → `database.py` → MQTT + mã HTTP → gỡ thẻ.**
- Giọng nói: **Gateway → xác nhận → MCP → cùng endpoint**, màn hình bếp gỡ thẻ ngay nhờ MQTT.

---

## 6.2. Yêu cầu 2: trang chủ phần mềm quản trị — luồng nền, signal, SQL tổng hợp

**① Khái niệm**

- *Luồng nền + signal*: việc chậm (gọi mạng) chạy ở luồng phụ; xong thì **phát tín hiệu**, Qt đưa kết quả về **luồng giao diện** để cập nhật an toàn.
- *Tổng hợp bằng SQL*: `SUM`, `COUNT`, `GROUP BY` chạy trong CSDL, chỉ trả kết quả nhỏ về client.

**② Trong đồ án**

```
 [QML Timer 5 s] ─► adminBackend.requestStats()          (luồng giao diện, trả về ngay)
        │
        └► Thread: _load_stats() ─► GET /api/stats?period=day (khóa quản trị, timeout 3 s)
                                        │
                     [Pi] require_api_key ─► get_stats_data("day")   (database.py L687)
                                        │      SELECT COALESCE(SUM(total_amount),0), COUNT(*) FROM orders
                                        │      WHERE payment_status='PAID' AND DATE(COALESCE(closed_at, created_at)) = DATE('now','localtime')
                                        │      … GROUP BY giờ → biểu đồ
                                        ◄── {"revenue":12400000,"orders":620,"chart":[{"day":"16:00","total":…}]}
        ◄── statsReady.emit(json)  ─► [Qt chuyển về luồng giao diện] ─► updateDashboard(json) ─► vẽ lại biểu đồ
```

Kết quả thật:

```
requestStats trả về sau 0.000 s (không chờ mạng)
Nhận statsReady: [('MainThread', 12400000)]        ← tín hiệu về đúng luồng giao diện
Pi mất mạng: requestStats trả về sau 0.002 s        ← giao diện không đứng
(so sánh) fetchStats đồng bộ mất 3.0 s
```

Chi tiết đáng chú ý ở `get_stats_data`:

- **`COALESCE(closed_at, created_at)`** (`_PAID_TIME`, L673): doanh thu tính theo **lúc thanh toán**; đơn cũ chưa có `closed_at` vẫn đúng.
- **`period`** chỉ nhận `day/week/month` (API kiểm tra, L384–389); `day` vẽ theo **giờ**, `week/month` vẽ theo **ngày**, sắp xếp theo thời gian thật (không theo chuỗi `dd/mm`).

**③ Vì sao / Nếu không**

- **Tính ở server bằng SQL:** chỉ một con số và vài điểm biểu đồ đi qua mạng, thay vì kéo hàng trăm đơn về client. Trước khi sửa, kéo về bị **kẹp 500 đơn** nên doanh thu tính thiếu.
- **Khóa `_stats_busy`:** mạng chậm hơn 5 s thì lần hẹn giờ sau **bỏ qua**, không chồng nhiều luồng.
- **Không cập nhật QML từ luồng phụ:** chỉ phát tín hiệu; Qt tự chuyển về luồng giao diện.

**④ Ghi nhớ**

- **Timer → requestStats (luồng nền) → /api/stats (SQL) → statsReady → updateDashboard.**
- Doanh thu theo **thời điểm thanh toán**, tính **bằng SQL ở server**.

---

## 6.3. Kiểm thử: test HTTP độc lập và bộ test chống lỗi

**① Khái niệm**

- *Kiểm thử tích hợp qua HTTP thật*: chạy **máy chủ thật** trên cổng thật, gửi **yêu cầu HTTP thật**, kiểm tra **cả phản hồi lẫn CSDL**.
- *Test hồi quy* (regression test): khóa lại hành vi đã sửa để lỗi cũ không quay lại.
- *Cô lập*: CSDL tạm, cổng ngẫu nhiên, **giả lập** (mock) kết nối ngoài.

**② Trong đồ án**

**Test HTTP độc lập** — `tests/test_orders_done_isolated_http.py` (script chạy tay; `conftest.py` chủ đích loại khỏi pytest):

```python
with patch("paho.mqtt.client.Client.connect", return_value=0), \
     patch("paho.mqtt.client.Client.loop_start", return_value=None):
    from servers import api_server              # nạp API nhưng KHÔNG kết nối broker thật
http_server = make_server("127.0.0.1", 0, api_server.app, threaded=True)   # cổng 0 → hệ điều hành tự chọn
```

Kết quả chạy lại sau khi sửa:

```
MISSING_KEY_STATUS=401      (CSDL vẫn PAID / pending)
WRONG_KEY_STATUS=401        (CSDL vẫn PAID / pending)
VALID_KEY_STATUS=200        (CSDL → PAID / done)
REPLAY_STATUS=409           (CSDL vẫn done)
ISOLATED_HTTP_WRITE_GATE=PASS
```

**Bộ test hồi quy mới** — `tests/test_api_hardening.py` (27 test), mỗi lỗi đã sửa có ít nhất một test:

| Nhóm | Ví dụ test |
|---|---|
| Khóa CSDL | `test_delete_product_with_recipe_removes_recipe_and_does_not_lock_db` (thao tác kế tiếp < 2 s) |
| Đầu vào | `test_delete_product_bad_input_is_400_json` (`{}`, chuỗi form, `true`, `"abc"`) |
| Lỗi JSON | `test_flask_errors_are_json` (404, 405) |
| Phân quyền đọc | `test_read_endpoints_require_key` (7 endpoint × 3 loại khóa) |
| Đăng nhập | `test_login_locks_after_five_failures`, `test_login_success_resets_counter` |
| Thống kê | `test_stats_counts_all_orders_not_only_500` (620 đơn) |
| Top món | `test_top_items_group_variants_by_product` (Size M + Size L = 1 món) |
| Ảnh | `test_uploaded_images_do_not_overwrite_each_other`, `test_upload_rejects_non_image` |
| MQTT | `test_order_done_publishes_mqtt` |

**Tổng:** Smart Kiosk **103/103** test đạt (trước khi sửa 75/76, test hỏng do cấu hình máy); Kitchen MCP **401/401**; firmware **build thành công** (45% bộ nhớ chương trình). Chốt đơn đồng thời (buổi 2) vẫn đúng: 20 đơn × 3 luồng → mỗi đơn đúng 1 `ok`, kho 1000 → 960.

**③ Vì sao / Nếu không**

- **Kiểm tra CSDL sau yêu cầu bị từ chối** chứng minh 401 **thật sự không ghi gì**.
- **Test hồi quy** bảo đảm sửa chỗ này không làm hỏng chỗ khác, và lỗi cũ không quay lại.
- **Test phải độc lập với máy:** test 503 từng hỏng vì `.env` thật lọt vào — đã sửa bằng cách đặt lại khóa trong fixture.

**④ Ghi nhớ**

- Test tích hợp = **máy chủ thật + HTTP thật + kiểm tra CSDL**: 401 → 401 → 200 → 409.
- **27 test hồi quy** cho các lỗi đã sửa; tổng **103/103** + **401/401**.

---

# MỤC 7. ĐÁNH GIÁ VÀ CÂU HỎI ÔN TẬP

## 7.1. Điểm mạnh

| # | Điểm mạnh | Thể hiện ở đâu | Mục |
|---|---|---|---|
| 1 | **Một cửa vào duy nhất qua mạng**; kiosk cùng máy gọi thẳng CSDL | 26 endpoint | 2.1 |
| 2 | **Phân quyền hai cấp, 401/403 đúng nghĩa, đặc quyền tối thiểu** | L280–338 | 4.2 |
| 3 | **Kiểm tra khóa đúng kỹ thuật**: decorator + `@wraps`, `compare_digest`, từ chối khóa rỗng | L270–338 | 4.3, 4.4 |
| 4 | **Fail closed** ở mọi nơi: API (503), webhook, MCP (ghi tắt), Gateway (bắt xác nhận) | | 4.5 |
| 5 | **Đăng nhập và OTP có giới hạn thử** | L344–375, `database.py` | 4.6 |
| 6 | **Mọi lỗi là JSON có `reason`**; kiểm tra đầu vào dùng chung | L155–266 | 2.2, 2.4 |
| 7 | **Báo xong đơn idempotent đầu cuối** + MQTT gỡ thẻ ngay | L697–719, firmware | 2.3, 6.1 |
| 8 | **Màn hình bếp tách mạng khỏi giao diện**, thử lại có nghỉ, phân biệt lỗi tạm/lỗi cố định | firmware L1602–1678 | 5.2 |
| 9 | **Kitchen MCP phòng thủ chặt** (ghi tắt mặc định, kiểm tra mẫu mã đơn, ngoại lệ có kiểu) | `kitchen_mcp/` | 5.3 |
| 10 | **Số liệu tính bằng SQL ở server; trang chủ tải nền** | `/api/stats`, `requestStats` | 5.1, 6.2 |
| 11 | **An toàn file**: `secure_filename`, đuôi ảnh, tên không trùng, giới hạn 8 MB, `send_from_directory` | L253–266, L410 | 3.2, 3.4 |
| 12 | **Chạy bằng Waitress, tự chạy bằng systemd**, có fallback | `main()`, `deploy/` | 3.3, Phụ lục A |
| 13 | **Có test hồi quy cho từng lỗi đã sửa**; 103/103 + 401/401 | `tests/` | 6.3 |

## 7.2. Các lỗi đã phát hiện ở buổi học và tình trạng sửa

| # | Lỗi phát hiện (kiểm chứng bằng thí nghiệm) | Tình trạng |
|---|---|---|
| 1 | Doanh thu trang chủ tính thiếu khi > 500 đơn | ✅ Dùng `/api/stats` (SQL) |
| 2 | Khóa CSDL bị treo 10 s sau khi xóa món lỗi khóa ngoại | ✅ `_open_db()` + xóa công thức trong giao dịch |
| 3 | 10 endpoint đọc không cần khóa | ✅ Đã khóa; sửa cả 3 client |
| 4 | Đăng nhập không giới hạn số lần thử | ✅ 5 lần / 5 phút → 429 |
| 5 | HTTP không mã hóa | ⛔ Để hướng phát triển (HTTPS) |
| 6 | Thiếu kiểm tra đầu vào (500/415 HTML) | ✅ Hàm kiểm tra dùng chung |
| 7 | Lỗi Flask trả HTML | ✅ 4 `errorhandler` → JSON |
| 8 | Phần mềm quản trị: đứng khi mạng chậm, bỏ phí `reason` | 🟡 Trang chủ tải nền, lỗi đúng lý do; nút bấm còn đồng bộ (có timeout) |
| 9 | `/api/tts` chết, lộ `str(e)` | ✅ Đã xóa |
| 10 | Thẻ đơn báo xong bằng giọng nói còn 30 s | ✅ MQTT `order_done` |
| 11 | Màn hình bếp thử lại không nghỉ, thử cả khi 401 | ✅ Nghỉ 1-2-4 s, không thử khi 4xx |
| 12 | Ảnh trùng tên ghi đè nhau | ✅ Tiền tố ngẫu nhiên |
| 13 | 401 thay 403; đổi mật khẩu sai trả 200 | ✅ |
| 14 | Chạy máy chủ phát triển | ✅ Waitress trên Pi |
| 15 | Test HTTP là script | ⬜ Giữ nguyên: chủ đích (`conftest.py`) |
| 16 | `/api/stats` không ai dùng | ✅ Trang chủ dùng |
| + | Thẻ kẹt "ĐANG LƯU..." khi không lấy được khóa giao diện (firmware) | ✅ Trả tin về hàng đợi |
| + | Test 503 hỏng do `.env` thật lọt vào | ✅ Fixture đặt lại khóa |

**Còn lại cho hướng phát triển:** HTTPS; Wi-Fi riêng cho thiết bị cửa hàng; gọi API ở luồng nền cho cả các nút bấm của phần mềm quản trị; đăng nhập cấp token có thời hạn thay cho khóa cố định trong `.env`; mật khẩu quản trị mặc định cần đổi khi triển khai thật.

## 7.3. Câu hỏi kiểu hội đồng

*Tự trả lời trước (chừa 4–5 dòng mỗi câu), sau đó mới đối chiếu gợi ý.*

**Nhóm A — HTTP và REST**

1. HTTP "không lưu trạng thái" nghĩa là gì? Ảnh hưởng thế nào tới cách các thiết bị gửi khóa?
2. Vì sao đã có HTTP API mà màn hình bếp vẫn cần MQTT? Kể 3 chủ đề MQTT bếp nghe.
3. API của đồ án có đúng chuẩn REST không?
4. Idempotent là gì? Vì sao màn hình bếp thử gửi lại `orders/done` vẫn an toàn?
5. Phân biệt 400, 401, 403, 409, 503 trong đồ án; mỗi mã client nên làm gì?

**Nhóm B — Flask**

6. WSGI là gì? Vì sao đổi sang Waitress không phải sửa mã Flask?
7. Máy chủ có đa luồng không? Liên quan gì tới cấu hình SQLite ở buổi 2?
8. Điều gì chặn kẻ xấu đọc `../.env` qua `/images/` hoặc ghi đè nó bằng ảnh tải lên?
9. Vì sao mọi lỗi phải trả JSON? Đồ án làm điều đó thế nào?

**Nhóm C — Bảo mật ★**

10. Trình bày phân quyền hai cấp. Lộ khóa bếp thì thiệt hại tối đa là gì?
11. Tấn công dò theo thời gian là gì? `compare_digest` và `if not expected: return False` chống gì?
12. "Fail closed" thể hiện ở những chỗ nào trong hệ thống?
13. Đăng nhập phần mềm quản trị có thật sự bảo vệ API không? Đã chống dò mật khẩu thế nào?
14. Trước khi sửa, ai cùng Wi-Fi đọc được gì? Bây giờ thì sao? Vì sao phải triển khai client trước server?

**Nhóm D — Đầu cuối và chất lượng**

15. Vẽ sơ đồ tuần tự khi bếp bấm HOÀN THÀNH. Báo xong bằng giọng nói khác ở đâu?
16. Vì sao doanh thu trang chủ từng thấp hơn trang báo cáo? Đã sửa thế nào?
17. Vì sao một lần xóa món thất bại từng làm các thao tác ghi khác chờ 10 giây?
18. Màn hình bếp xử lý mỗi loại kết quả HTTP thế nào khi báo xong đơn?
19. Làm sao chứng minh các lỗi đã sửa không quay lại?

### Gợi ý trả lời (đọc sau khi tự trả lời)

| Câu | Ý chính cần có |
|---|---|
| 1 | Mỗi yêu cầu độc lập, server không nhớ phiên → mỗi yêu cầu tự mang `X-API-Key`; Pi khởi động lại client không phải đăng nhập lại |
| 2 | HTTP chỉ client hỏi – server trả; MQTT đẩy tức thời: `kitchen/new_order`, `kitchen/inventory`, `kitchen/order_done`; HTTP để đồng bộ lại (30 s) và gửi lệnh |
| 3 | REST mức đơn giản: tài nguyên rõ, JSON, mã trạng thái đúng nghĩa; GET/POST + động từ trong đường dẫn để đơn giản cho ESP32; an toàn gọi lặp bảo đảm bằng CSDL |
| 4 | Gọi nhiều lần cùng kết quả; lần 2 `UPDATE … WHERE kitchen_status='pending'` có `rowcount=0` → 409; bếp coi 409 là thành công |
| 5 | 400 dữ liệu sai · 401 thiếu/sai khóa · 403 đủ khóa thiếu quyền · 409 đã xong ở nơi khác · 503 server chưa cấu hình khóa; 4xx không thử lại, 5xx/mạng thử lại có nghỉ |
| 6 | Chuẩn giao tiếp máy chủ web ↔ ứng dụng; app chỉ là hàm `app(environ, start_response)`; `main()` gọi `waitress.serve(app)` thay cho `app.run` |
| 7 | Có (Werkzeug `threaded=True`, Waitress 8 luồng; thí nghiệm 0,52 s so với 2,02 s) → nhiều yêu cầu cùng ghi → WAL, `busy_timeout`, `BEGIN IMMEDIATE`; mỗi yêu cầu một kết nối SQLite |
| 8 | `<filename>` không khớp `/` + `send_from_directory` (4 kiểu thử đều 404); tải lên: `secure_filename`, chỉ nhận đuôi ảnh, tiền tố ngẫu nhiên, ≤ 8 MB |
| 9 | Client (ESP32, MCP) phân tích JSON; trang HTML làm hỏng phân tích; 4 `errorhandler` (ApiError, HTTPException, IntegrityError, Exception); lỗi 500 không lộ chi tiết |
| 10 | Quản trị: mọi thứ; bếp: đọc đơn/kho/công thức + báo xong đơn; khóa bếp nằm trong firmware → tệ nhất đọc đơn/kho và báo xong đơn giả; không đọc được doanh thu (403) |
| 11 | `==` dừng ở ký tự sai đầu → lộ thông tin qua thời gian; `compare_digest` không đổi (đo: 0,1 so với 51 µs vs 2005 so với 2008 µs); chặn `"" == ""` khi chưa cấu hình |
| 12 | API thiếu khóa → 503; webhook thiếu token → từ chối; MCP ghi tắt mặc định; Gateway công cụ ghi luôn cần xác nhận |
| 13 | Không cấp token, quyền ghi nằm ở `ADMIN_API_KEY` trong `.env`; chống dò: 5 lần sai/5 phút → 429 kèm `Retry-After`; OTP sai 5 lần hủy mã |
| 14 | Trước: đọc được đơn, doanh thu, kho, công thức; nay chỉ thực đơn và ảnh công khai; client mới chạy được với server cũ, còn server mới làm client cũ nhận 401 → cập nhật client trước |
| 15 | Chạm → hàng đợi → task mạng → POST → Waitress → url_map → decorator → hàm xử lý → `database.py` → MQTT + mã → gỡ thẻ; giọng nói qua Gateway → xác nhận → MCP → cùng endpoint → MQTT gỡ thẻ ngay |
| 16 | Trang chủ tự cộng ở client từ `limit=9999` nhưng API kẹp 500 → chỉ cộng 500 đơn mới nhất (620 đơn thiếu 2,4 triệu); nay dùng `/api/stats` tính bằng SQL |
| 17 | `DELETE` mở giao dịch ghi rồi ném lỗi khóa ngoại; không `try/finally` → kết nối giữ khóa ghi (khóa ở mức file) → thao tác ghi khác chờ 10 s rồi `database is locked`; sửa bằng `_open_db()` và xóa công thức trong giao dịch |
| 18 | 200/409 → gỡ thẻ; 400/401/403 → không thử, hiện GỬI LẠI; mạng/5xx → thử 3 lần nghỉ 1-2-4 s; MQTT `order_done` gỡ thẻ ngay |
| 19 | 27 test hồi quy trong `test_api_hardening.py`, mỗi lỗi một test; tổng 103/103 + 401/401; chạy lại thí nghiệm trên Pi thật |

## Tóm tắt buổi 3 trong 5 dòng

1. **HTTP**: client hỏi – server trả, không lưu trạng thái → mỗi yêu cầu mang `X-API-Key`; 400/401/403/409/503 có ý nghĩa riêng.
2. **API**: 26 endpoint, GET đọc – POST ghi; mọi lỗi là JSON `{"success":false,"reason":…}`; báo xong đơn idempotent nhờ 409.
3. **Flask**: ứng dụng WSGI, `url_map` + decorator; chạy bằng **Waitress** (Werkzeug dự phòng), đa luồng → mỗi yêu cầu một kết nối SQLite.
4. **Bảo mật**: khóa hai cấp, `compare_digest`, fail closed, giới hạn đăng nhập/OTP, **đọc dữ liệu nhạy cảm cũng cần khóa**; chưa có HTTPS.
5. **Client**: quản trị (`requests`, tải nền, SQL ở server), bếp (2 task, thử lại 1-2-4 s, MQTT `order_done`), MCP (ngoại lệ có kiểu, ghi tắt mặc định).

---

# PHỤ LỤC A: TRIỂN KHAI VÀ CÁC FILE TỰ CHẠY

## A.1. Trên Raspberry Pi

| Thành phần | File đã cài | Bản gốc trong repo | Chạy khi |
|---|---|---|---|
| Smart Kiosk API (5000) | `/etc/systemd/system/kiosk-api.service` | `deploy/pi/kiosk-api.service` | Pi khởi động (systemd, `Restart=on-failure`) |
| Webhook SePay (5001) | `/etc/systemd/system/kiosk-webhook.service` | `deploy/pi/kiosk-webhook.service` | Pi khởi động |
| Đường hầm ngrok cho webhook | `/etc/systemd/system/kiosk-ngrok.service` | *(chưa có trong repo)* | Pi khởi động |
| Giao diện kiosk | `~/.config/autostart/smart-kiosk.desktop` | `deploy/pi/smart-kiosk.desktop` | đăng nhập desktop |
| Biểu tượng mở lại kiosk | `~/Desktop/smart-kiosk.desktop` | `deploy/pi/smart-kiosk.desktop`, `smart-kiosk.svg` | bấm đúp |
| Script mở kiosk | `deploy/pi/run_kiosk.sh` | | gọi từ 2 file trên |

`run_kiosk.sh`: chờ API sẵn sàng (≤ 20 s); **không mở trùng cửa sổ** (chỉ khớp đúng tiến trình `python … kiosk_app/main.py`); nếu kiosk vừa thoát còn đang tắt thì **chờ tối đa 5 s** rồi mở; ghi log không bộ đệm vào `/tmp/smart-kiosk.log`.

Lệnh minh họa khi được hỏi:

```bash
ls /etc/systemd/system/kiosk-*.service
systemctl status kiosk-api kiosk-webhook kiosk-ngrok
cat ~/.config/autostart/smart-kiosk.desktop
journalctl -u kiosk-api -f
```

## A.2. Trên laptop

| Thành phần | File | Chạy khi |
|---|---|---|
| Phần mềm quản trị | `~/Desktop/quan-tri-smart-kiosk.desktop` → `Smart_Kiosk_Backup/deploy/laptop/run_admin.sh` | bấm biểu tượng |
| Kitchen MCP (3001) | `~/.config/systemd/user/kitchen-mcp.service` (+ `~/.config/tro-ly-giong-noi/kitchen-mcp.env`, quyền 600) | mở phần mềm quản trị **hoặc** bấm biểu tượng Trợ lý |
| Voice Gateway (8000) | `~/.config/systemd/user/voice-gateway.service` | như trên |
| Biểu tượng Trợ lý giọng nói | `~/Desktop/tro-ly-giong-noi.desktop` → `Voice_Test/.../deploy/laptop/tro_ly_giong_noi.sh` | bấm: bật/khởi động lại; chuột phải: trạng thái, log, tắt |
| Ollama (11434) | dịch vụ hệ thống `ollama.service` | máy khởi động |

Hai server trợ lý **không tự chạy khi đăng nhập**; `run_admin.sh` bật chúng bằng `systemctl --user --no-block start` trước khi mở phần mềm quản trị. Cài lại: `Smart_Kiosk_Backup/deploy/laptop/cai_dat_laptop.sh` và `Voice_Test/voice_assistant_edge_ai/deploy/laptop/cai_dat_laptop.sh`.

## A.3. Thứ tự triển khai khi siết phân quyền

1. Cập nhật **client** trước (bản mới vẫn chạy với server cũ): nạp firmware màn hình bếp; Kitchen MCP có `KIOSK_API_KEY`; phần mềm quản trị bản mới.
2. Cập nhật **server** trên Pi sau cùng: sao lưu `.bak`, chép file, `pip install waitress`, `sudo systemctl restart kiosk-api`.
3. Kiểm tra bằng curl: không khóa → 401; khóa bếp đọc đơn chờ → 200; khóa bếp đọc báo cáo → 403.

---

# PHỤ LỤC B: CÁCH CHẠY LẠI CÁC THÍ NGHIỆM

Mọi thí nghiệm dùng **CSDL tạm, khóa giả**, không đụng `backend/kiosk.db`. Thay `/duong/dan/tam` bằng thư mục tạm có thật (đường dẫn tuyệt đối).

```bash
export KIOSK_DB_PATH=/duong/dan/tam/thu.db
export ADMIN_API_KEY=khoa_admin_demo KITCHEN_API_KEY=khoa_bep_demo
export MQTT_BROKER=127.0.0.1 MQTT_PORT=1 SMTP_HOST=127.0.0.1 SMTP_PORT=1   # không gửi gì ra ngoài
```

## B.1. Bộ test đầy đủ

```bash
cd ~/Documents/DATN_Final_Version/Smart_Kiosk_Backup
python3 -m pytest -q                       # 103 passed
python3 -m pytest -q tests/test_api_hardening.py -v

cd ~/Documents/DATN_Final_Version/Voice_Test/voice_assistant_edge_ai/services/kitchen_mcp
PYTHONPATH=src ../../.venv-gateway/bin/python -m pytest -q   # 401 passed
```

## B.2. Chạy API cục bộ và gửi curl

```python
# chay_api.py — đặt ngoài thư mục dự án
import sys, os
sys.path.insert(0, os.path.expanduser("~/Documents/DATN_Final_Version/Smart_Kiosk_Backup"))
from servers import api_server
api_server.db.init_database()
api_server.app.run(host="127.0.0.1", port=5099, threaded=True)
```

```bash
python3 chay_api.py &
curl -s -w " -> %{http_code}\n" http://127.0.0.1:5099/api/orders/pending                        # 401
curl -s -w " -> %{http_code}\n" -H "X-API-Key: khoa_bep_demo" http://127.0.0.1:5099/api/report   # 403
curl -s -w " -> %{http_code}\n" http://127.0.0.1:5099/api/khong_co                               # 404 JSON
```

## B.3. Test HTTP độc lập

```bash
cd ~/Documents/DATN_Final_Version/Smart_Kiosk_Backup
python3 -c "import sys,json;sys.path.insert(0,'backend');import database as db;db.init_database();\
it=json.dumps([{'product_id':1,'qty':1,'name':'Bánh mì','price':35000}]);\
db.create_pending_order('DH20261124',35000,it);print(db.save_new_order('DH20261124',35000,it)['reason'])"
KITCHEN_API_KEY= TEST_ORDER_ID=DH20261124 PYTHONPATH=. python3 tests/test_orders_done_isolated_http.py
```

## B.4. `==` so với `compare_digest`

```python
import hmac, timeit
bi = "k"*2_000_000; sai_dau = "X"+"k"*1_999_999; sai_cuoi = "k"*1_999_999+"X"
for ten, f in [("==", lambda x: x == bi), ("compare_digest", lambda x: hmac.compare_digest(x.encode(), bi.encode()))]:
    t1 = min(timeit.repeat(lambda: f(sai_dau), number=20, repeat=5))/20*1e6
    t2 = min(timeit.repeat(lambda: f(sai_cuoi), number=20, repeat=5))/20*1e6
    print(f"{ten:15} sai đầu {t1:9.1f} µs | sai cuối {t2:9.1f} µs")
```

## B.5. Đa luồng so với đơn luồng (máy chủ Werkzeug)

```python
# chay_luong.py <cổng> <1|0>
import sys, os, threading, time
sys.path.insert(0, os.path.expanduser("~/Documents/DATN_Final_Version/Smart_Kiosk_Backup"))
from servers import api_server
api_server.db.init_database()
goc = api_server.app.wsgi_app
def do_luong(environ, start_response):          # lớp WSGI trung gian: in tên luồng, giả lập 0,5 s
    print("LUONG", threading.current_thread().name, flush=True); time.sleep(0.5)
    return goc(environ, start_response)
api_server.app.wsgi_app = do_luong
api_server.app.run(host="127.0.0.1", port=int(sys.argv[1]), threaded=(sys.argv[2] == "1"))
```

Gửi 4 yêu cầu song song: `for i in 1 2 3 4; do curl -s -o /dev/null http://127.0.0.1:5097/api/menu & done; wait`.

## B.6. Trang chủ tải nền (cần API ở B.2 đang chạy)

```python
import sys, os, json, time, threading
sys.path.insert(0, os.path.expanduser("~/Documents/DATN_Final_Version/Smart_Kiosk_Backup/admin_app"))
os.environ["API_SERVER_URL"] = "http://127.0.0.1:5099"; os.environ["ADMIN_API_KEY"] = "khoa_admin_demo"
from PySide6.QtCore import QCoreApplication, QTimer
app = QCoreApplication(sys.argv)
from admin_bridge import AdminBridge
b = AdminBridge(); b.current_time_filter = "day"; got = []
b.statsReady.connect(lambda s: got.append((threading.current_thread().name, json.loads(s).get("revenue"))))
t = time.time(); b.requestStats(); print(f"requestStats trả về sau {time.time()-t:.3f}s")
QTimer.singleShot(1500, app.quit); app.exec(); print("Nhận statsReady:", got)
```

# CHƯƠNG 5. FLASK – KHUNG PHÁT TRIỂN WEB CHO MÁY CHỦ NGHIỆP VỤ

## Mục tiêu của chương

Sau khi học xong chương này, người đọc cần nắm được:

- Flask là gì, ra đời trong bối cảnh nào và có những đặc điểm gì nổi bật so với các khung phát triển web khác.
- Cơ chế hoạt động bên trong của một ứng dụng Flask: chuẩn WSGI, vòng đời một yêu cầu HTTP, cơ chế context, định tuyến, decorator và cách tạo phản hồi.
- Lý do dự án chọn Flask cho Smart Kiosk API và máy chủ webhook, trong khi Voice Gateway lại dùng FastAPI.
- Cách Flask được áp dụng cụ thể trong dự án: tổ chức endpoint, xác thực bằng khóa API, xử lý dữ liệu, tải tệp, gửi email, nhận webhook và kiểm thử.
- Các kỹ thuật thiết kế đã áp dụng, các lỗi đã gặp và những hạn chế còn tồn tại, đủ để trình bày và trả lời câu hỏi trước hội đồng.

**Vị trí trong dự án.** Flask chạy trên Raspberry Pi 4 của khối kiosk, gồm hai máy chủ:

| Máy chủ | Tệp mã nguồn | Cổng | Vai trò |
|---|---|---|---|
| Smart Kiosk API | `Smart_Kiosk_Backup/servers/api_server.py` | 5000 | Nguồn dữ liệu trung tâm: thực đơn, kho, công thức, đơn hàng, báo cáo, tài khoản quản trị |
| Webhook SePay | `Smart_Kiosk_Backup/servers/webhook_server.py` | 5001 | Nhận thông báo giao dịch ngân hàng từ Internet và chuyển tiếp sang kiosk |

Phiên bản sử dụng: Flask 3.1 và Werkzeug 3 (khai báo trong `requirements.txt` là `Flask>=3.0,<4.0`).

---

## 5.1. Khái niệm và đặc điểm

### 5.1.1. Flask là gì

Flask là một **khung phát triển web vi mô** (*micro web framework*) viết bằng ngôn ngữ Python. "Khung phát triển" ở đây là một bộ thư viện và quy ước giúp lập trình viên xây dựng ứng dụng web mà không phải tự viết lại những phần lặp đi lặp lại như đọc và phân tích yêu cầu HTTP, gắn địa chỉ URL với đoạn mã xử lý, hay đóng gói phản hồi gửi về trình duyệt.

Chữ "vi mô" không có nghĩa là Flask chỉ dùng được cho ứng dụng nhỏ. Nó phản ánh triết lý thiết kế: Flask chỉ giữ lại **phần lõi tối thiểu** của một ứng dụng web, gồm định tuyến, xử lý yêu cầu và phản hồi, sinh khuôn mẫu HTML và công cụ kiểm thử. Những thành phần khác như truy cập cơ sở dữ liệu, đăng nhập người dùng hay kiểm tra dữ liệu đầu vào không được đóng gói sẵn. Người lập trình được tự do chọn thư viện phù hợp, hoặc tự viết như dự án này đã làm. Nhờ vậy, Flask gọn nhẹ, dễ hiểu và không áp đặt cấu trúc dự án.

### 5.1.2. Nguồn gốc và hệ sinh thái

Flask do Armin Ronacher phát triển và công bố năm 2010, khi đó thuộc nhóm lập trình viên Pocoo, nay được duy trì bởi tổ chức mã nguồn mở Pallets. Một chi tiết thú vị là Flask khởi đầu như một trò đùa ngày Cá tháng Tư: tác giả gộp vài thư viện sẵn có thành một tệp duy nhất để chế giễu các khung phát triển "một tệp" đang thịnh hành. Phản hồi tích cực của cộng đồng khiến dự án được phát triển nghiêm túc, và đến nay Flask là một trong hai khung web Python phổ biến nhất, cùng với Django.

Flask không tự làm mọi việc mà dựa trên một số thư viện cùng tổ chức Pallets:

| Thư viện | Vai trò |
|---|---|
| **Werkzeug** | Bộ công cụ WSGI. Phân tích yêu cầu HTTP, định tuyến, cung cấp các đối tượng `Request` và `Response`, máy chủ phát triển, cùng nhiều tiện ích như làm sạch tên tệp (`secure_filename`) và băm mật khẩu (`generate_password_hash`) |
| **Jinja2** | Bộ sinh khuôn mẫu HTML. Dự án không dùng vì hai máy chủ chỉ trả dữ liệu JSON |
| **Click** | Xây dựng câu lệnh dòng lệnh `flask` |
| **ItsDangerous** | Ký dữ liệu, dùng cho cookie phiên làm việc |
| **MarkupSafe**, **Blinker** | Thoát ký tự HTML an toàn; tín hiệu nội bộ |

Có thể hình dung Werkzeug là "động cơ" xử lý giao thức, còn Flask là "khung xe" sắp xếp các bộ phận thành một ứng dụng hoàn chỉnh, dễ dùng.

### 5.1.3. Đặc điểm nổi bật

**Gọn nhẹ và dễ học.** Một ứng dụng Flask hoàn chỉnh có thể chỉ gồm năm, sáu dòng mã. Thư viện ít phụ thuộc, khởi động nhanh và tốn ít bộ nhớ. Đây là lợi thế khi chạy trên Raspberry Pi, nơi tài nguyên phải chia cho cả ứng dụng giao diện kiosk.

**Định tuyến bằng decorator.** Địa chỉ URL được gắn với hàm xử lý bằng cú pháp `@app.route("/đường-dẫn")` đặt ngay trên hàm. Người đọc mã nhìn vào là thấy ngay URL nào do hàm nào xử lý.

**Linh hoạt, không áp đặt.** Flask không bắt buộc cấu trúc thư mục, không bắt buộc dùng ORM (công cụ ánh xạ đối tượng – quan hệ). Dự án tận dụng điều này để quản lý SQLite bằng thư viện `sqlite3` thuần, kiểm soát chính xác giao dịch `BEGIN IMMEDIATE`.

**Xử lý đồng bộ theo chuẩn WSGI.** Mỗi yêu cầu được xử lý tuần tự trên một luồng từ đầu đến cuối. Mô hình này đơn giản, dễ gỡ lỗi, và phù hợp với các yêu cầu ngắn, độc lập.

**Hỗ trợ kiểm thử tốt.** Flask có sẵn máy khách kiểm thử, cho phép gửi yêu cầu giả vào ứng dụng mà không cần mở cổng mạng. Đây là nền tảng của bộ kiểm thử bảo mật API trong dự án.

**Mở rộng bằng tiện ích.** Cộng đồng có hàng trăm tiện ích mở rộng như Flask-Login, Flask-SQLAlchemy hay Flask-Limiter. Dự án chưa dùng tiện ích nào để giữ hệ thống đơn giản.

Bên cạnh ưu điểm, Flask cũng có những **hạn chế** cần biết:
- Không có cơ chế kiểm tra dữ liệu đầu vào tự động, nên lập trình viên phải tự kiểm tra.
- Không hỗ trợ lập trình bất đồng bộ một cách tự nhiên, không có WebSocket sẵn.
- Máy chủ đi kèm chỉ dành cho giai đoạn phát triển, không dùng cho môi trường sản phẩm.

Chính các hạn chế này giải thích vì sao Voice Gateway, nơi cần WebSocket và xử lý bất đồng bộ, được xây bằng FastAPI thay vì Flask.

| Đặc điểm | Ưu điểm | Lưu ý |
|---|---|---|
| Lõi tối thiểu | Nhẹ, dễ học, tự do lựa chọn | Phải tự lo kiểm tra dữ liệu, bảo mật |
| Đồng bộ (WSGI) | Đơn giản, dễ gỡ lỗi | Không hợp với kết nối mở lâu dài |
| Decorator định tuyến | Mã dễ đọc | Phải chú ý thứ tự decorator |
| Máy khách kiểm thử | Kiểm thử nhanh, không cần mạng | – |
| Máy chủ phát triển | Tiện khi lập trình | Không dùng cho sản phẩm |

---

## 5.2. Nguyên lý hoạt động

Để hiểu Flask làm gì, trước hết cần hiểu giao thức mà Flask xử lý, rồi đến chuẩn WSGI kết nối Flask với máy chủ web, sau cùng là những gì diễn ra bên trong Flask khi một yêu cầu đến.

### 5.2.1. Nền tảng: giao thức HTTP và kiến trúc REST

**HTTP** (*HyperText Transfer Protocol*) là giao thức tầng ứng dụng hoạt động theo mô hình **yêu cầu – phản hồi**: máy khách gửi một yêu cầu, máy chủ xử lý rồi trả về một phản hồi. HTTP là giao thức **không lưu trạng thái** (*stateless*): máy chủ không nhớ các yêu cầu trước, nên mỗi yêu cầu phải tự mang đủ thông tin cần thiết, ví dụ khóa xác thực phải gửi kèm trong mọi yêu cầu.

Một yêu cầu HTTP gồm bốn phần: **phương thức** (GET, POST…), **đường dẫn**, các **tiêu đề** (*header*) và phần **thân** (*body*). Ví dụ yêu cầu màn hình bếp gửi khi nhân viên bấm "Hoàn thành":

```http
POST /api/orders/done HTTP/1.1
Host: 192.168.1.20:5000
X-API-Key: <khóa của bếp>
Content-Type: application/json

{"order_code": "DH12345678"}
```

Phản hồi gồm **mã trạng thái**, tiêu đề và thân:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"success": true, "order_code": "DH12345678", "kitchen_status": "done"}
```

**REST** (*Representational State Transfer*) là phong cách thiết kế giao diện lập trình do Roy Fielding đề xuất năm 2000. Trong REST, dữ liệu được coi là các **tài nguyên**, mỗi tài nguyên có một địa chỉ URL riêng (ví dụ `/api/menu`, `/api/inventory`). Máy khách thao tác trên tài nguyên bằng phương thức HTTP: GET để đọc, POST để tạo mới hoặc thực hiện hành động. Kết quả được biểu diễn bằng **mã trạng thái chuẩn**. Nhờ dùng chung một "ngôn ngữ" như vậy, máy khách đọc mã trạng thái là biết cách phản ứng mà không cần đọc nội dung.

| Mã | Ý nghĩa | Tình huống trong dự án |
|---|---|---|
| 200 OK | Thành công | Lấy thực đơn, nhập kho thành công |
| 400 Bad Request | Yêu cầu sai định dạng | Thiếu mã đơn, số lượng nhập kho âm |
| 401 Unauthorized | Chưa xác thực | Thiếu hoặc sai khóa API |
| 409 Conflict | Xung đột với trạng thái hiện tại | Đơn đã được hoàn thành trước đó |
| 429 Too Many Requests | Gọi quá dày | Yêu cầu mã OTP liên tục |
| 502 Bad Gateway | Lỗi ở dịch vụ phía sau | Webhook không chuyển tiếp được sang MQTT |
| 503 Service Unavailable | Dịch vụ chưa sẵn sàng | Máy chủ chưa được cấu hình khóa bảo mật |

### 5.2.2. Chuẩn WSGI – cầu nối giữa máy chủ web và ứng dụng Python

Một ứng dụng web không tự lắng nghe kết nối mạng. Việc đó do **máy chủ web** đảm nhận: nhận kết nối TCP, đọc các byte của yêu cầu HTTP, rồi chuyển cho ứng dụng xử lý. Để mọi máy chủ web có thể làm việc với mọi khung phát triển Python, cộng đồng đặt ra chuẩn **WSGI** (*Web Server Gateway Interface*, đặc tả trong PEP 3333).

Theo WSGI, một ứng dụng chỉ đơn giản là **một đối tượng gọi được** nhận hai tham số: `environ` là từ điển chứa toàn bộ thông tin yêu cầu, và `start_response` là hàm để gửi mã trạng thái cùng tiêu đề phản hồi. Ứng dụng trả về phần thân dưới dạng danh sách byte:

```python
def application(environ, start_response):
    # environ["REQUEST_METHOD"] == "GET", environ["PATH_INFO"] == "/hello", ...
    start_response("200 OK", [("Content-Type", "text/plain; charset=utf-8")])
    return ["Xin chào".encode("utf-8")]
```

Viết ứng dụng trực tiếp ở mức này rất vất vả. Flask chính là một ứng dụng WSGI được xây sẵn: đối tượng `app = Flask(__name__)` là đối tượng gọi được theo đúng chuẩn trên. Bên trong, Flask chuyển `environ` thành đối tượng `request` thân thiện, và chuyển giá trị hàm xử lý trả về thành phản hồi WSGI. Vì tuân theo chuẩn chung, cùng một ứng dụng Flask có thể chạy trên máy chủ phát triển của Werkzeug khi lập trình, và trên Gunicorn hay Waitress khi triển khai, mà không phải sửa mã.

### 5.2.3. Vòng đời của một yêu cầu trong Flask

Khi màn hình bếp gửi yêu cầu báo hoàn thành đơn, các bước sau diễn ra:

```
Máy khách (KDS)
   │  POST /api/orders/done   X-API-Key: ...   {"order_code": "DH12345678"}
   ▼
Máy chủ WSGI (Werkzeug, đa luồng)
   │ 1. Nhận kết nối TCP, phân tích HTTP thành environ, giao cho một luồng
   │ 2. Gọi app(environ, start_response)
   ▼
Ứng dụng Flask
   │ 3. Tạo và kích hoạt application context và request context
   │ 4. Định tuyến: tìm route khớp đường dẫn + phương thức → mark_order_done
   │ 5. Chạy các decorator bọc hàm: require_kitchen_or_admin_key kiểm tra khóa
   │ 6. Chạy hàm xử lý, trả về (jsonify(...), 409)
   │ 7. Chuyển giá trị trả về thành đối tượng Response
   │ 8. Gỡ bỏ context, trả Response cho máy chủ WSGI
   ▼
Máy khách nhận: HTTP/1.1 409 CONFLICT  {"success": false, "reason": "..."}
```

Điều đáng chú ý là bước 5: kiểm tra khóa API **không nằm trong** hàm xử lý nghiệp vụ mà nằm ở một decorator bọc bên ngoài. Hàm nghiệp vụ chỉ được chạy khi khóa hợp lệ. Mục 5.2.6 sẽ giải thích cơ chế này.

### 5.2.4. Cơ chế context – vì sao `request` dùng như biến toàn cục mà vẫn an toàn

Người mới học Flask thường thắc mắc: hàm xử lý không nhận tham số nào, vậy làm sao đọc được dữ liệu yêu cầu? Câu trả lời là biến `request` được import từ Flask và dùng trực tiếp:

```python
from flask import request

@app.route("/api/orders/done", methods=["POST"])
def mark_order_done():
    data = request.get_json(silent=True)   # request ở đây là yêu cầu NÀO?
```

Máy chủ phát triển của Flask xử lý nhiều yêu cầu song song trên nhiều luồng. Nếu `request` là biến toàn cục thông thường, hai yêu cầu đến cùng lúc sẽ ghi đè dữ liệu của nhau. Flask giải quyết bằng **proxy cục bộ theo ngữ cảnh** (*context-local proxy*): `request` thực chất là một đối tượng trung gian. Mỗi khi được truy cập, nó tra xem luồng hoặc ngữ cảnh hiện tại đang xử lý yêu cầu nào và trả về đúng đối tượng của yêu cầu đó. Từ Flask 2.3, cơ chế này dựa trên thư viện chuẩn `contextvars`.

Flask có hai loại ngữ cảnh:

| Ngữ cảnh | Đối tượng truy cập | Tồn tại khi |
|---|---|---|
| Ngữ cảnh ứng dụng (*application context*) | `current_app`, `g` | Đang xử lý yêu cầu, hoặc trong khối `with app.app_context()` |
| Ngữ cảnh yêu cầu (*request context*) | `request`, `session` | Đang xử lý một yêu cầu HTTP |

Hệ quả thực tế: chỉ dùng được `request` bên trong hàm xử lý hoặc các hàm được gọi từ đó. Dùng ở mức mô-đun (ngoài mọi yêu cầu) sẽ gây lỗi "Working outside of request context".

### 5.2.5. Định tuyến

**Định tuyến** (*routing*) là việc chọn hàm xử lý phù hợp với một yêu cầu. Khi khởi động, Flask (thông qua Werkzeug) biên dịch mỗi route thành một mẫu so khớp. Khi yêu cầu đến, Flask so đường dẫn và phương thức với bảng route:
- khớp đường dẫn nhưng sai phương thức (gửi GET vào route chỉ nhận POST) thì trả **405 Method Not Allowed**;
- không khớp đường dẫn nào thì trả **404 Not Found**.

Route có thể chứa **biến đường dẫn**. Phần nằm trong `<...>` được trích ra và truyền vào hàm làm tham số; có thể kèm bộ chuyển đổi kiểu để Flask tự kiểm tra và đổi kiểu:

```python
@app.route("/images/<filename>")              # mặc định là chuỗi
def serve_image(filename): ...

@app.route("/api/order/<int:order_id>")        # chỉ khớp khi là số nguyên, tự đổi sang int
def get_order(order_id): ...
```

### 5.2.6. Đọc dữ liệu yêu cầu và tạo phản hồi

Dữ liệu từ máy khách có thể đến từ nhiều vị trí trong yêu cầu HTTP. Flask cung cấp thuộc tính tương ứng cho từng vị trí:

| Thuộc tính | Lấy dữ liệu từ | Ví dụ trong dự án |
|---|---|---|
| `request.args` | Chuỗi truy vấn trên URL `?page=2&limit=10` | Phân trang danh sách đơn |
| `request.form` | Biểu mẫu (`application/x-www-form-urlencoded`, `multipart/form-data`) | Tên, giá món khi thêm món |
| `request.files` | Tệp tải lên (`multipart/form-data`) | Ảnh món ăn |
| `request.json` | Thân JSON, yêu cầu `Content-Type: application/json` | Dữ liệu nhập kho |
| `request.get_json(silent=True)` | Như trên, nhưng trả `None` thay vì báo lỗi khi thân không hợp lệ | Báo hoàn thành đơn |
| `request.headers` | Tiêu đề HTTP | Khóa `X-API-Key` |

Cần phân biệt `request.json` và `request.get_json(silent=True)`. Thuộc tính `request.json` sẽ báo lỗi 415 (*Unsupported Media Type*) nếu tiêu đề `Content-Type` không phải JSON, và báo lỗi 400 nếu nội dung không phân tích được. Trong khi đó `get_json(silent=True)` lặng lẽ trả về `None`, để lập trình viên tự xử lý và trả về thông báo lỗi có ý nghĩa hơn. Cách viết thường dùng là `request.get_json(silent=True) or {}`.

Ở chiều ngược lại, hàm xử lý có thể trả về nhiều dạng giá trị, và Flask tự chuyển thành phản hồi HTTP:

```python
return jsonify({"success": True})                          # JSON, mã 200 mặc định
return jsonify({"success": False, "reason": "x"}), 400     # bộ (thân, mã trạng thái)
return send_from_directory(UPLOAD_FOLDER, filename)        # trả tệp trong thư mục
return send_file(io.BytesIO(data), mimetype="audio/mpeg")  # trả tệp tạo trong bộ nhớ
```

Hàm `jsonify` chuyển từ điển hoặc danh sách Python thành chuỗi JSON, đồng thời đặt tiêu đề `Content-Type: application/json`, nên máy khách biết cách đọc.

### 5.2.7. Decorator – bọc hàm để thêm hành vi

**Decorator** là một hàm nhận vào một hàm khác và trả về một hàm mới, có thêm hành vi trước hoặc sau khi gọi hàm gốc. Cú pháp `@ten_decorator` đặt trên định nghĩa hàm chỉ là cách viết gọn của phép gán `ham = ten_decorator(ham)`. Trong ứng dụng web, decorator rất hợp để tách các mối quan tâm chung như xác thực, ghi nhật ký hay giới hạn tần suất ra khỏi mã nghiệp vụ.

```python
from functools import wraps
from flask import request, jsonify

def require_token(f):
    @wraps(f)                                     # giữ nguyên tên và docstring của f
    def decorated(*args, **kwargs):
        if request.headers.get("X-Token") != "bi-mat":
            return jsonify({"reason": "unauthorized"}), 401   # chặn, không gọi f
        return f(*args, **kwargs)                 # hợp lệ: chạy hàm gốc
    return decorated

@app.route("/secret", methods=["POST"])
@require_token
def secret():
    return jsonify({"ok": True})
```

Có hai điểm bắt buộc phải nắm khi kết hợp decorator với Flask.

*Thứ nhất, `functools.wraps`.* Flask dùng **tên hàm** làm tên định danh của endpoint. Nếu không có `wraps`, mọi hàm được bọc đều mang tên `decorated`. Khi hai route cùng dùng một decorator, Flask phát hiện hai endpoint trùng tên và báo lỗi ngay khi khởi động. `wraps` sao chép tên và tài liệu của hàm gốc sang hàm bọc, khắc phục vấn đề này.

*Thứ hai, thứ tự decorator.* Decorator được áp dụng **từ dưới lên**. `@app.route` phải nằm **trên cùng** để Flask đăng ký hàm **đã được bọc** kiểm tra khóa. Nếu đặt `@app.route` ở dưới, Flask sẽ đăng ký hàm gốc trước khi nó được bọc, và mọi yêu cầu sẽ đi thẳng vào hàm nghiệp vụ, bỏ qua bước xác thực. Đây là lỗi bảo mật khó phát hiện vì chương trình vẫn chạy bình thường.

### 5.2.8. Máy chủ phát triển và máy chủ sản phẩm

Lệnh `app.run()` khởi động **máy chủ phát triển của Werkzeug**. Từ Flask 1.0, máy chủ này mặc định chạy **đa luồng** (`threaded=True`): mỗi yêu cầu được giao cho một luồng riêng. Máy chủ phát triển tiện lợi vì khởi động nhanh, có chế độ gỡ lỗi và tự nạp lại mã. Tuy nhiên nó không được thiết kế để chịu tải, không tối ưu bảo mật và không có cơ chế quản lý tiến trình. Khi triển khai thật, cùng ứng dụng đó nên chạy trên một **máy chủ WSGI chuyên dụng**:

```bash
# Gunicorn (Linux): 2 tiến trình, mỗi tiến trình 4 luồng
gunicorn -w 2 --threads 4 -b 0.0.0.0:5000 servers.api_server:app

# Waitress (thuần Python, chạy được cả trên Windows)
waitress-serve --listen=0.0.0.0:5000 servers.api_server:app
```

### 5.2.9. Kiểm thử ứng dụng Flask

Flask cung cấp **máy khách kiểm thử** qua `app.test_client()`. Máy khách này gửi yêu cầu trực tiếp vào ứng dụng WSGI trong bộ nhớ, không mở cổng mạng, nên kiểm thử chạy rất nhanh và không phụ thuộc môi trường:

```python
def test_hello():
    client = app.test_client()
    response = client.get("/hello")
    assert response.status_code == 200
    assert response.get_json() == {"message": "Xin chào"}
```

Kết hợp với thư viện `pytest` và kỹ thuật **giả lập** (*mock*), tức thay một hàm phụ thuộc bên ngoài như gửi MQTT hay gửi email bằng hàm giả, ta kiểm thử được logic của từng endpoint một cách độc lập. Dự án dùng đúng kỹ thuật này để kiểm thử bảo mật (mục 5.4.9).

---

## 5.3. Lý do lựa chọn

Khi xây dựng phần máy chủ nghiệp vụ, có ba khung phát triển Python phổ biến để cân nhắc: Flask, FastAPI và Django. Bảng dưới so sánh ba lựa chọn theo các tiêu chí quan trọng với dự án.

| Tiêu chí | Flask | FastAPI | Django |
|---|---|---|---|
| Kiểu khung | Vi mô, đồng bộ (WSGI) | Hiện đại, bất đồng bộ (ASGI) | Đầy đủ "kèm pin" |
| Kiểm tra dữ liệu đầu vào | Tự viết | Tự động bằng Pydantic | Form, serializer |
| WebSocket | Không có sẵn | Có sẵn | Qua Django Channels |
| Tài liệu API tự sinh | Không | Có (OpenAPI/Swagger) | Qua thư viện ngoài |
| ORM, trang quản trị | Không | Không | Có sẵn |
| Tài nguyên trên Raspberry Pi | Rất nhẹ | Nhẹ | Nặng hơn |
| Độ khó học | Thấp | Trung bình (cần hiểu async) | Cao hơn |

Dự án chọn **Flask cho Smart Kiosk API và webhook SePay** vì bốn lý do.

*Thứ nhất, bản chất các yêu cầu.* Mọi yêu cầu tới hai máy chủ này đều **ngắn và độc lập**: đọc hoặc ghi vài bảng SQLite trong vài mili giây rồi trả kết quả. Không có kết nối nào cần giữ mở lâu dài. Với loại tải này, mô hình đồng bộ của Flask là đủ, lại đơn giản và dễ gỡ lỗi hơn mô hình bất đồng bộ.

*Thứ hai, tài nguyên phần cứng.* Hai máy chủ chạy chung Raspberry Pi với ứng dụng giao diện kiosk viết bằng Qt. Flask nhẹ, khởi động nhanh và tốn ít bộ nhớ, không tranh tài nguyên với giao diện.

*Thứ ba, quyền kiểm soát cơ sở dữ liệu.* Nghiệp vụ lưu đơn và trừ kho cần giao dịch `BEGIN IMMEDIATE` và câu lệnh cập nhật có điều kiện viết tay. Flask không áp đặt ORM nên dự án dùng `sqlite3` thuần một cách tự nhiên. Với Django, ta sẽ phải "đi vòng" qua ORM của nó.

*Thứ tư, khả năng kiểm thử.* Máy khách kiểm thử có sẵn cho phép viết bộ kiểm thử bảo mật API bằng pytest một cách gọn gàng.

Ngược lại, **Voice Gateway dùng FastAPI** vì yêu cầu hoàn toàn khác: Gateway phải giữ **kết nối WebSocket mở lâu dài** với thiết bị giọng nói và điều phối các tác vụ AI kéo dài vài giây. Mô hình bất đồng bộ ASGI của FastAPI cho phép một tiến trình phục vụ nhiều kết nối như vậy mà không bị chặn, điều Flask không làm tốt. Việc dùng hai khung khác nhau cho hai loại tải khác nhau là một quyết định có chủ đích: **chọn công cụ theo bản chất bài toán**, không dùng một công cụ cho mọi việc.

---

## 5.4. Ứng dụng và code trong dự án

### 5.4.1. Tổng quan hai máy chủ Flask trong hệ thống

Smart Kiosk API là **nguồn dữ liệu trung tâm** của toàn hệ thống. Mọi thành phần muốn đọc hoặc thay đổi dữ liệu kinh doanh đều đi qua API này: phần mềm quản trị trên laptop, màn hình bếp ESP32 và dịch vụ Kitchen MCP của trợ lý giọng nói. Riêng ứng dụng kiosk ghi đơn trực tiếp vào SQLite vì chạy cùng máy với cơ sở dữ liệu. Máy chủ webhook thì đóng vai trò "cửa ngõ" nhận thông báo từ dịch vụ SePay trên Internet.

```
                        Internet
   SePay ──POST /sepay_webhook (Apikey)──► webhook_server :5001 ──MQTT payment/success──► Kiosk
                                                                                            │ ghi đơn
   Admin (laptop) ──HTTP + X-API-Key ─────┐                                                 ▼
   KDS (ESP32) ──GET pending / POST done ─┼──► api_server :5000 ◄──────────► SQLite kiosk.db
   Kitchen MCP ──GET + POST done ─────────┘          │
                                                     └──MQTT kitchen/inventory──► KDS
```

Smart Kiosk API có 27 endpoint, chia thành các nhóm sau:

| Nhóm | Endpoint chính | Xác thực ghi | Bên gọi |
|---|---|---|---|
| Thực đơn | `GET /api/menu`; `POST /api/menu/add`, `/update`, `/delete`, `/toggle`; `GET /images/<tệp>` | Khóa quản trị | Admin, MCP |
| Đơn hàng | `GET /api/orders`, `GET /api/orders/pending`, `POST /api/orders/done` | Khóa bếp hoặc quản trị (riêng `done`) | Admin, KDS, MCP |
| Kho | `GET /api/inventory`; `POST /api/inventory/add`, `/update`, `/delete`, `/restock` | Khóa quản trị | KDS, MCP, Admin |
| Công thức | `GET /api/recipe`; `POST /api/recipe/save` | Khóa quản trị | Admin, MCP |
| Báo cáo, cửa hàng | `GET /api/stats`, `/api/report`, `/api/store/info`; `POST /api/store/update` | Khóa quản trị | Admin |
| Tài khoản | `/api/auth/login`, `/forgot_password`, `/reset_password`, `/change_password`, `/recovery_email` | Khóa quản trị (đổi mật khẩu, email) | Admin |
| Giọng đọc | `GET /api/tts` | – | Kiosk |

Các phần tiếp theo trình bày những đoạn mã tiêu biểu, đại diện cho từng kỹ thuật được sử dụng.

### 5.4.2. Khởi tạo ứng dụng và cấu hình qua biến môi trường

Mọi giá trị nhạy cảm như khóa API, mật khẩu email hay khóa ký MQTT không được viết cứng trong mã mà đọc từ biến môi trường. Thư viện `python-dotenv` nạp các biến này từ tệp `.env` ở thư mục gốc dự án. Tệp `.env` không được đưa lên kho mã (git); chỉ có tệp mẫu `.env.example`.

```python
# Smart_Kiosk_Backup/servers/api_server.py
from functools import wraps
from werkzeug.utils import secure_filename
from flask import Flask, request, jsonify, send_from_directory, send_file
from dotenv import load_dotenv

current_dir = os.path.dirname(os.path.abspath(__file__))
project_root = os.path.dirname(current_dir)
if project_root not in sys.path:
    sys.path.append(project_root)          # cho phép import gói backend/

import backend.database as db
from backend.mqtt_security import configure_client, sign_message

load_dotenv(os.path.join(project_root, ".env"))   # phải chạy TRƯỚC khi đọc biến môi trường

app = Flask(__name__)

UPLOAD_FOLDER = os.path.join(project_root, 'assets', 'images')
os.makedirs(UPLOAD_FOLDER, exist_ok=True)

ADMIN_API_KEY = os.environ.get("ADMIN_API_KEY", "")
# Khóa riêng cho KDS và Kitchen MCP: CHỈ được báo hoàn thành đơn.
KITCHEN_API_KEY = os.environ.get("KITCHEN_API_KEY", "")
```

Ở cuối tệp, máy chủ được khởi động với cổng đọc từ biến môi trường:

```python
if __name__ == '__main__':
    # Bảng cổng: Kiosk API 5000, webhook 5001, Kitchen MCP 3001, Voice Gateway 8000.
    app.run(host='0.0.0.0', port=int(os.environ.get('KIOSK_API_PORT', '5000')))
```

Địa chỉ `0.0.0.0` nghĩa là máy chủ nhận kết nối từ mọi giao diện mạng. Điều này cần thiết vì màn hình bếp và laptop ở các máy khác trong cùng mạng Wi-Fi nội bộ.

### 5.4.3. Xác thực bằng decorator với hai cấp khóa

Đây là đoạn mã quan trọng nhất về bảo mật của máy chủ. Thay vì kiểm tra khóa trong từng hàm, dự án viết hai decorator dùng chung và đặt lên các route cần bảo vệ.

```python
def _key_matches(provided, expected):
    """So sánh hằng thời gian trên bytes."""
    if not expected:
        return False
    return hmac.compare_digest(
        str(provided).encode("utf-8"),
        str(expected).encode("utf-8"),
    )


def require_api_key(f):
    """Chỉ ADMIN_API_KEY: dùng cho mọi endpoint quản trị."""
    @wraps(f)
    def decorated(*args, **kwargs):
        if not ADMIN_API_KEY:
            return jsonify({"success": False, "reason": "api_key_not_configured"}), 503
        provided = request.headers.get("X-API-Key", "")
        if not _key_matches(provided, ADMIN_API_KEY):
            return jsonify({"success": False, "reason": "unauthorized"}), 401
        return f(*args, **kwargs)
    return decorated


def require_kitchen_or_admin_key(f):
    """ADMIN_API_KEY hoặc KITCHEN_API_KEY: chỉ dùng cho thao tác của bếp."""
    @wraps(f)
    def decorated(*args, **kwargs):
        if not ADMIN_API_KEY and not KITCHEN_API_KEY:
            return jsonify({"success": False, "reason": "api_key_not_configured"}), 503
        provided = request.headers.get("X-API-Key", "")
        if not (_key_matches(provided, ADMIN_API_KEY) or _key_matches(provided, KITCHEN_API_KEY)):
            return jsonify({"success": False, "reason": "unauthorized"}), 401
        return f(*args, **kwargs)
    return decorated
```

Đoạn mã này thể hiện bốn nguyên tắc bảo mật, mỗi nguyên tắc đều có thể được hội đồng hỏi tới.

**So sánh trong thời gian hằng.** Hàm `_key_matches` không dùng phép `==` mà dùng `hmac.compare_digest`. Phép so sánh chuỗi thông thường dừng ngay ở ký tự sai đầu tiên, nên chuỗi sai ở ký tự thứ nhất được so nhanh hơn chuỗi sai ở ký tự thứ mười. Kẻ tấn công có thể gửi hàng loạt yêu cầu, đo thời gian phản hồi và đoán dần từng ký tự của khóa. Kiểu tấn công này gọi là **tấn công đo thời gian** (*timing attack*). Hàm `compare_digest` luôn so sánh hết chuỗi trong cùng một khoảng thời gian, loại bỏ kênh rò rỉ này. Chuỗi được đổi sang byte UTF-8 trước khi so, vì `compare_digest` không chấp nhận chuỗi `str` chứa ký tự ngoài bảng mã ASCII.

**Mặc định từ chối (fail-closed).** Khi khóa quản trị chưa được cấu hình, decorator trả mã 503 cho mọi yêu cầu ghi thay vì cho qua. Một thiết kế ngược lại ("chưa cấu hình khóa thì không kiểm tra") nghe có vẻ tiện khi phát triển, nhưng chỉ cần quên cấu hình khi triển khai là toàn bộ hệ thống mở cửa cho bất kỳ ai. Fail-closed bảo đảm sai sót cấu hình luôn dẫn tới trạng thái **an toàn**.

**Đặc quyền tối thiểu (least privilege).** Hệ thống có hai loại khóa. Khóa quản trị có toàn quyền ghi và chỉ phần mềm quản trị nắm giữ. Khóa bếp chỉ mở được đúng một endpoint là `/api/orders/done`, và được nạp vào firmware màn hình bếp cùng dịch vụ Kitchen MCP. Thiết bị nhúng dễ bị trích xuất firmware hơn máy tính; nếu khóa bếp bị lộ, kẻ xấu cũng chỉ đánh dấu hoàn thành được đơn hàng, không sửa được thực đơn, giá bán hay mật khẩu quản trị.

**Không cho khóa rỗng khớp nhau.** Dòng `if not expected: return False` có vẻ thừa nhưng rất quan trọng. Khi khóa bếp không được cấu hình (chuỗi rỗng), một yêu cầu không gửi tiêu đề `X-API-Key` cũng mang giá trị rỗng. Nếu không có dòng này, hai chuỗi rỗng sẽ "khớp" nhau và yêu cầu được chấp nhận.

Các route cần bảo vệ chỉ việc đặt decorator ngay dưới `@app.route`:

```python
@app.route('/api/inventory/restock', methods=['POST'])
@require_api_key                       # chỉ khóa quản trị
def restock_ingredient(): ...

@app.route('/api/orders/done', methods=['POST'])
@require_kitchen_or_admin_key          # khóa bếp hoặc khóa quản trị
def mark_order_done(): ...
```

### 5.4.4. Endpoint đọc dữ liệu và thống nhất định dạng giữa hai kênh

Màn hình bếp nhận đơn qua hai kênh: kênh đẩy MQTT khi có đơn mới, và kênh kéo HTTP khi khởi động hoặc đồng bộ định kỳ để bù những đơn bị lỡ. Nếu hai kênh trả dữ liệu theo hai định dạng khác nhau, firmware ESP32 phải viết hai hàm phân tích, tốn bộ nhớ và dễ sai lệch. Vì vậy endpoint `/api/orders/pending` được thiết kế để trả mỗi đơn **đúng cùng cấu trúc** với thông điệp MQTT `kitchen/new_order`:

```python
@app.route('/api/orders/pending', methods=['GET'])
def get_pending_orders():
    rows = db.admin_get_kitchen_pending_orders()
    result = []
    for raw_row in rows:
        row = dict(raw_row)
        try:
            raw_items = json.loads(row.get('items_json') or '[]')
        except (TypeError, json.JSONDecodeError):
            raw_items = []                              # dữ liệu hỏng không làm sập cả phản hồi

        items = []
        for raw_item in raw_items:
            if not isinstance(raw_item, dict):
                continue
            display_name = str(raw_item.get('name', '')).strip()
            base_name = str(raw_item.get('base_name')
                            or display_name.split(' (', 1)[0].strip())   # "Trà đào (Size L)" -> "Trà đào"
            try:
                quantity = int(raw_item.get('qty', raw_item.get('quantity', 1)))
            except (TypeError, ValueError):
                quantity = 1
            items.append({
                "product_id": int(raw_item.get('product_id', 0) or 0),
                "name": base_name,
                "quantity": max(1, quantity),
                "options": {"size": str(raw_item.get('size', '')),
                            "ice": str(raw_item.get('ice', ''))},
                "note": str(raw_item.get('note', '')),
            })

        result.append({
            "version": 1,
            "event": "order.snapshot",
            "order_id": row.get('order_id', ''),
            "created_at": row.get('created_at', ''),
            "total_amount": int(row.get('total_amount') or 0),
            "items": items,
        })

    return jsonify({"version": 1, "event": "orders.pending.snapshot", "data": result})
```

Ba kỹ thuật đáng chú ý trong đoạn mã này. Thứ nhất, các trường `version` và `event` biến định dạng dữ liệu thành một **hợp đồng có phiên bản** (*versioned contract*): sau này nếu cần thay đổi cấu trúc, máy chủ tăng số phiên bản và thiết bị cũ vẫn nhận biết được. Thứ hai, mã được viết theo lối **phòng thủ**: chấp nhận cả `qty` lẫn `quantity` (tên trường cũ và mới), bỏ qua phần tử không phải từ điển, đặt mọi phép đổi kiểu trong khối `try`, để một món hỏng dữ liệu không làm hỏng cả danh sách. Thứ ba, máy chủ làm sẵn các xử lý như tách tên món gốc khỏi phần tùy chọn, để vi điều khiển với tài nguyên hạn chế không phải làm.

Tương tự, endpoint tồn kho tính sẵn cờ "sắp hết" để màn hình bếp chỉ việc tô màu:

```python
@app.route('/api/inventory', methods=['GET'])
def get_inventory():
    ingredients = db.get_all_ingredients()
    for i in ingredients:
        i["low"] = i["quantity"] <= i["low_stock_threshold"]
    return jsonify(ingredients)
```

### 5.4.5. Endpoint ghi có điều kiện: bếp báo hoàn thành đơn

Endpoint `/api/orders/done` minh họa cách dùng mã trạng thái HTTP để truyền đạt kết quả nghiệp vụ một cách chính xác.

```python
@app.route('/api/orders/done', methods=['POST'])
@require_kitchen_or_admin_key
def mark_order_done():
    data = request.get_json(silent=True) or {}
    order_code = str(data.get('order_code', '')).strip()

    if not order_code:
        return jsonify({"success": False, "reason": "missing_order_code"}), 400

    success = db.admin_mark_order_done(order_code)
    if not success:
        return jsonify({"success": False,
                        "reason": "order_not_found_or_already_done",
                        "order_code": order_code}), 409

    return jsonify({"success": True, "order_code": order_code, "kitchen_status": "done"}), 200
```

Trong hệ thống có hai nguồn có thể báo hoàn thành cùng một đơn: nhân viên chạm nút trên màn hình bếp, hoặc nói lệnh với trợ lý giọng nói. Hàm `admin_mark_order_done` ở tầng cơ sở dữ liệu dùng **câu lệnh cập nhật có điều kiện**, chỉ chuyển trạng thái khi đơn đang ở `pending` và đã thanh toán. Vì vậy nếu hai yêu cầu đến gần như đồng thời, chỉ yêu cầu đầu tiên thành công, yêu cầu thứ hai nhận mã **409 Conflict**.

Việc chọn mã 409 thay vì 404 hay 500 là có chủ đích:
- **500** báo lỗi máy chủ, máy khách nên thử lại.
- **404** báo không tìm thấy tài nguyên.
- **409** báo yêu cầu hợp lệ nhưng xung đột với trạng thái hiện tại.

Nhận 409, màn hình bếp hiểu rằng đơn đã được xử lý xong và không cần thử lại. Cách viết `get_json(silent=True) or {}` bảo đảm thân yêu cầu rỗng hay sai định dạng cũng đi vào nhánh 400 với lý do rõ ràng, thay vì sinh lỗi 415 khó hiểu.

Ở phía máy khách, firmware màn hình bếp gửi yêu cầu này bằng thư viện `HTTPClient` của ESP32, đặt thời gian chờ ngắn để không treo tác vụ mạng:

```cpp
// kds_firmware.ino – postDoneOrder (rút gọn)
HTTPClient http;
http.begin(apiUrl("/api/orders/done"));
http.setConnectTimeout(2500);                 // 2,5 giây kết nối
http.setTimeout(4000);                        // 4 giây chờ phản hồi
http.addHeader("Content-Type", "application/json");
http.addHeader("X-API-Key", KDS_API_KEY);     // khóa bếp, không phải khóa quản trị
int status = http.POST(body);                 // body = {"order_code": "..."}
```

### 5.4.6. Endpoint nhận tệp tải lên: thêm món kèm ảnh

Khi quản trị viên thêm món mới, phần mềm quản trị gửi một yêu cầu dạng `multipart/form-data` gồm các trường chữ (tên, loại, giá) và một tệp ảnh. Flask tách sẵn hai phần này vào `request.form` và `request.files`:

```python
@app.route('/api/menu/add', methods=['POST'])
@require_api_key
def add_product():
    name = request.form.get('name')
    category = request.form.get('category')
    price = int(float(request.form.get('price')))

    image_filename = 'default_food.png'
    if 'image_file' in request.files:
        file = request.files['image_file']
        if file.filename != '':
            filename = secure_filename(file.filename)       # làm sạch tên tệp
            file.save(os.path.join(UPLOAD_FOLDER, filename))
            image_filename = filename

    success = db.admin_add_product(name, category, price, image_filename)
    return jsonify({'success': success})
```

Điểm quan trọng nhất là hàm `secure_filename` của Werkzeug. Tên tệp do máy khách gửi lên là dữ liệu **không đáng tin**. Một kẻ tấn công có thể đặt tên tệp là `../../.env` hòng ghi đè tệp cấu hình chứa khóa bí mật bên ngoài thư mục ảnh; kiểu tấn công này gọi là **duyệt thư mục** (*path traversal*). `secure_filename` loại bỏ các ký tự và thành phần nguy hiểm, biến tên trên thành `env`, bảo đảm tệp chỉ được ghi vào thư mục ảnh. Tương tự, khi phục vụ ảnh, hàm `send_from_directory` chỉ trả về tệp nằm trong thư mục được chỉ định.

Phía phần mềm quản trị gửi tệp bằng thư viện `requests`, kèm khóa quản trị trong tiêu đề:

```python
# Smart_Kiosk_Backup/admin_app/admin_bridge.py (rút gọn)
_AUTH_HEADERS = {"X-API-Key": ADMIN_API_KEY} if ADMIN_API_KEY else {}
res = requests.post(url, data=data, files=files, headers=_AUTH_HEADERS, timeout=5)
```

Tham số `timeout=5` là thói quen tốt: không có thời gian chờ, một máy chủ không phản hồi sẽ làm giao diện quản trị treo vô thời hạn.

### 5.4.7. Endpoint có tác dụng phụ: nhập kho và phát sự kiện MQTT

Khi kho thay đổi, màn hình bếp cần được cập nhật ngay. Sau khi ghi cơ sở dữ liệu, endpoint nhập kho gọi hàm phát toàn bộ bảng tồn kho lên chủ đề MQTT `kitchen/inventory`:

```python
@app.route('/api/inventory/restock', methods=['POST'])
@require_api_key
def restock_ingredient():
    data = request.json or {}
    iid = data.get('id')
    add_quantity = float(data.get('add_quantity', 0))
    if add_quantity <= 0:
        return jsonify({"success": False, "reason": "invalid_quantity"}), 400

    updated = db.admin_restock_ingredient(iid, add_quantity)
    publish_inventory_snapshot()                    # tác dụng phụ: báo cho màn hình bếp
    return jsonify({"success": updated is not None, "ingredient": updated})


def publish_inventory_snapshot():
    try:
        ingredients = db.get_all_ingredients()
        if not MQTT_SHARED_SECRET:
            return                                  # fail-closed: không có khóa ký thì không phát
        payload = {
            "version": 1,
            "event": "inventory.snapshot",
            "items": [{"id": i["id"], "name": i["name"], "unit": i["unit"],
                       "quantity": i["quantity"],
                       "low": i["quantity"] <= i["low_stock_threshold"]} for i in ingredients],
        }
        _mqtt_client.publish(MQTT_TOPIC_INVENTORY,
                             sign_message(payload, MQTT_SHARED_SECRET),   # ký HMAC
                             retain=True)                                  # giữ bản cuối trên broker
    except Exception as e:
        print(f"--> Loi publish inventory MQTT: {e}")
```

Thiết kế này dựa trên hai quyết định. Thứ nhất, máy chủ phát **toàn bộ bảng tồn kho** (*snapshot*) thay vì chỉ phần thay đổi. Màn hình bếp không phải ghép các thay đổi theo đúng thứ tự, và nếu một thông điệp bị lỡ thì thông điệp sau vẫn mang trạng thái đầy đủ. Thứ hai, mọi lỗi MQTT đều bị bắt lại và chỉ ghi nhật ký. Một sự cố ở kênh thông báo phụ **không được làm hỏng** yêu cầu HTTP chính: dữ liệu đã nằm an toàn trong cơ sở dữ liệu, và màn hình bếp sẽ tự lấy lại bảng kho ở lần đồng bộ HTTP kế tiếp. Máy khách MQTT được tạo một lần khi khởi động máy chủ và dùng chung cho mọi yêu cầu.

### 5.4.8. Endpoint tài khoản: khôi phục mật khẩu bằng mã OTP qua email

Chức năng quên mật khẩu kết hợp Flask với thư viện gửi thư `smtplib` của Python:

```python
def send_otp_email(to_email, otp_code):
    if not SMTP_USER or not SMTP_PASSWORD:
        return False
    msg = MIMEText(f"Mã OTP khôi phục mật khẩu của bạn là: {otp_code}\n\n"
                   f"Mã có hiệu lực trong 10 phút.", "plain", "utf-8")
    msg["Subject"] = "Mã khôi phục mật khẩu - Smart Kiosk Admin"
    msg["From"] = SMTP_USER
    msg["To"] = to_email
    try:
        with smtplib.SMTP(SMTP_HOST, SMTP_PORT, timeout=10) as server:
            server.starttls()                       # nâng kết nối lên TLS trước khi đăng nhập
            server.login(SMTP_USER, SMTP_PASSWORD)
            server.sendmail(SMTP_USER, [to_email], msg.as_string())
        return True
    except Exception as e:
        print(f"--> Loi gui email OTP: {e}")
        return False


@app.route('/api/auth/forgot_password', methods=['POST'])
def forgot_password():
    otp_code, email, reason = db.generate_password_reset_otp()
    if not email:
        return jsonify({'success': False, 'reason': 'no_recovery_email'})
    if reason == "otp_cooldown":
        return jsonify({'success': False, 'reason': 'otp_cooldown'}), 429
    if not send_otp_email(email, otp_code):
        return jsonify({'success': False, 'reason': 'email_send_failed'})
    masked = email[0] + "***" + email[email.find("@"):] if "@" in email else "***"
    return jsonify({'success': True, 'masked_email': masked})
```

Mỗi chi tiết trong đoạn mã đều có lý do bảo mật:
- **`starttls()`**: với cổng 587, kết nối ban đầu chưa mã hóa. Lệnh STARTTLS nâng nó lên TLS **trước khi** gửi mật khẩu, nên mật khẩu SMTP không đi qua mạng ở dạng rõ. Gmail yêu cầu dùng **mật khẩu ứng dụng** (*App Password*) thay cho mật khẩu tài khoản thường.
- **Mã 429:** khi người dùng yêu cầu mã OTP quá dày, máy chủ trả *Too Many Requests*, tránh bị lợi dụng để gửi thư rác.
- **Che email:** phản hồi chỉ hiện dạng `n***@gmail.com`, nên người không có quyền không đọc được địa chỉ khôi phục đầy đủ.
- Phần lưu trữ ở tầng cơ sở dữ liệu:
  - mật khẩu quản trị được **băm có muối** bằng `werkzeug.security`, không lưu dạng rõ;
  - mã OTP có hiệu lực 10 phút và bị xóa ngay sau khi dùng.

### 5.4.9. Máy chủ webhook SePay

SePay là dịch vụ trung gian theo dõi tài khoản ngân hàng. Khi có giao dịch, SePay chủ động gửi yêu cầu POST đến địa chỉ đã đăng ký; cơ chế này gọi là **webhook**. Máy chủ webhook là một ứng dụng Flask riêng, nhỏ gọn:

```python
# Smart_Kiosk_Backup/servers/webhook_server.py
ORDER_CODE_PATTERN = re.compile(r"DH\d{8}")

def _is_request_authorized(req):
    """Fail-closed: chưa cấu hình token thì từ chối mọi request."""
    if not SEPAY_WEBHOOK_AUTH_TOKEN:
        return False
    auth_header = req.headers.get("Authorization", "").strip()
    if auth_header.lower().startswith("apikey "):         # SePay gửi dạng "Apikey <token>"
        auth_header = auth_header[len("apikey "):].strip()
    return hmac.compare_digest(auth_header.encode("utf-8"),
                               SEPAY_WEBHOOK_AUTH_TOKEN.encode("utf-8"))


@app.route('/sepay_webhook', methods=['POST'])
def sepay_webhook():
    if not SEPAY_WEBHOOK_AUTH_TOKEN:
        return jsonify({"success": False, "message": "Webhook not configured"}), 503
    if not _is_request_authorized(request):
        return jsonify({"success": False, "message": "Unauthorized"}), 401

    data = request.get_json(silent=True)
    if not isinstance(data, dict):
        return jsonify({"success": False, "message": "Invalid JSON"}), 400
    try:
        content = str(data.get("transferContent", "") or "")
        amount = data.get("transferAmount", 0)
        if str(data.get("transferType", "in") or "in").lower() != "in":     # chỉ nhận tiền VÀO
            return jsonify({"success": True, "message": "Bỏ qua giao dịch tiền ra"}), 200

        match = ORDER_CODE_PATTERN.search(content)
        if match:
            order_code = match.group(0)
            if not send_mqtt_signal(order_code, amount):                    # phát tín hiệu đã ký
                return jsonify({"success": False, "message": "Không chuyển tiếp được tín hiệu"}), 502
            return jsonify({"success": True, "order_code": order_code}), 200
        return jsonify({"success": False, "message": "Không phải đơn hàng Kiosk"}), 200
    except Exception as e:
        print(f"--- Lỗi xử lý webhook: {e}")
        return jsonify({"success": False}), 500
```

Kiến trúc của webhook phản ánh nguyên tắc **giảm bề mặt tấn công**. Máy chủ này buộc phải mở ra Internet để SePay gọi được, nên nó là thành phần dễ bị tấn công nhất. Vì vậy:
- webhook được tách thành một tiến trình riêng, **không đọc, không ghi** cơ sở dữ liệu;
- webhook chỉ làm đúng một việc: kiểm tra, trích mã đơn rồi chuyển tiếp tín hiệu đã ký sang kiosk qua MQTT;
- phiên bản đầu từng có endpoint thống kê doanh thu trên máy chủ này, và đã bị gỡ bỏ vì làm lộ dữ liệu kinh doanh ra Internet.

Kẻ tấn công có chiếm được webhook cũng không chạm tới được dữ liệu.

Các mã trạng thái cũng được chọn có tính toán:
- **200** cho giao dịch không thuộc kiosk, để SePay hiểu là đã nhận và không gửi lại liên tục;
- **502** khi không chuyển tiếp được sang MQTT, báo lỗi nằm ở dịch vụ phía sau;
- **401** và **503** cho lỗi xác thực và cấu hình.

Biểu thức chính quy `DH\d{8}` dùng `search` thay vì so khớp toàn chuỗi, vì ngân hàng hoặc người chuyển có thể thêm chữ khác vào nội dung chuyển khoản.

### 5.4.10. Kiểm thử máy chủ Flask

Dự án kiểm thử các máy chủ Flask bằng pytest và máy khách kiểm thử của Flask. Ví dụ dưới đây kiểm thử phần xác thực của webhook:

```python
# Smart_Kiosk_Backup/tests/test_webhook_security.py
TOKEN = "webhook-test-token"
PAYLOAD = {"transferType": "in", "transferContent": "CK DH12345678 thanh toan", "transferAmount": 70000}

@pytest.fixture
def webhook():
    module = importlib.import_module("servers.webhook_server")
    original = module.SEPAY_WEBHOOK_AUTH_TOKEN
    module.SEPAY_WEBHOOK_AUTH_TOKEN = TOKEN
    module.app.config.update(TESTING=True)
    yield module
    module.SEPAY_WEBHOOK_AUTH_TOKEN = original          # trả lại trạng thái sau kiểm thử

def _post(module, headers=None, json=PAYLOAD):
    return module.app.test_client().post("/sepay_webhook", json=json, headers=headers or {})

def test_rejects_everything_when_token_not_configured(webhook):
    webhook.SEPAY_WEBHOOK_AUTH_TOKEN = ""
    with patch.object(webhook, "send_mqtt_signal", return_value=True) as signal:
        response = _post(webhook, {"Authorization": "Apikey anything"})
    assert response.status_code == 503
    signal.assert_not_called()                         # tuyệt đối không phát tín hiệu

@pytest.mark.parametrize("header", [{}, {"Authorization": "Apikey wrong"}, {"Authorization": "Apikey "}])
def test_rejects_missing_or_wrong_token(webhook, header):
    with patch.object(webhook, "send_mqtt_signal", return_value=True) as signal:
        response = _post(webhook, header)
    assert response.status_code == 401
    signal.assert_not_called()
```

Bộ kiểm thử này tận dụng bốn công cụ:
- **`fixture`**: chuẩn bị môi trường trước mỗi ca và dọn dẹp sau đó.
- **`app.test_client()`**: gửi yêu cầu vào ứng dụng mà không cần mạng.
- **`patch.object`**: thay hàm phát MQTT thật bằng hàm giả, vừa không cần broker vừa kiểm tra được hàm đó **có bị gọi hay không**. Đây là điều cốt lõi khi kiểm thử bảo mật: yêu cầu sai khóa không những phải bị trả 401, mà còn **không được** gây ra tác dụng phụ nào.
- **`parametrize`**: chạy cùng một ca kiểm thử với nhiều bộ dữ liệu (thiếu khóa, sai khóa, khóa rỗng).

Ngoài tệp này, dự án còn có các bộ kiểm thử cho khóa quản trị (`test_api_auth.py`), khóa bếp (`test_kitchen_api_key.py`), báo hoàn thành đơn (`test_orders_done_auth.py`) và OTP (`test_otp_security.py`). Toàn bộ khối Smart Kiosk có 75 ca kiểm thử tự động, chạy bằng lệnh `python3 -m pytest -q` trong thư mục `Smart_Kiosk_Backup`.

---

## 5.5. Kỹ thuật và đánh giá

### 5.5.1. Các kỹ thuật thiết kế đã áp dụng

Nhìn lại toàn bộ phần mã Flask, có thể rút ra một số kỹ thuật thiết kế xuyên suốt. Phần bảo mật được xây theo **nhiều lớp độc lập**:
- decorator gom logic xác thực về một chỗ, nên các hàm nghiệp vụ chỉ còn nghiệp vụ;
- so sánh khóa trong thời gian hằng chống tấn công đo thời gian;
- nguyên tắc mặc định từ chối biến mọi sai sót cấu hình thành trạng thái an toàn;
- hai cấp khóa thực hiện nguyên tắc đặc quyền tối thiểu;
- máy chủ webhook mở ra Internet được tách riêng và tước bỏ quyền truy cập dữ liệu.

Về giao tiếp giữa các thành phần, API dùng **mã trạng thái HTTP có ý nghĩa nghiệp vụ** (400, 401, 409, 429, 502, 503), giúp máy khách biết nên thử lại, báo lỗi hay bỏ qua. Dữ liệu trả về mang **số phiên bản và tên sự kiện**, cho phép mở rộng định dạng mà không phá vỡ thiết bị cũ. Đặc biệt, endpoint đơn chờ trả **cùng định dạng** với thông điệp MQTT, nên màn hình bếp chỉ cần một hàm phân tích cho cả hai kênh.

Về độ bền vững, mã được viết theo lối **phòng thủ**: đọc JSON bằng `get_json(silent=True)`, đổi kiểu trong khối `try`, giới hạn tham số phân trang (`limit` tối đa 500) để chặn truy vấn quá nặng. Các **tác dụng phụ** như phát MQTT được bọc riêng, để sự cố ở đó không làm hỏng yêu cầu chính. Mọi giá trị nhạy cảm được đọc từ **biến môi trường**, tách cấu hình khỏi mã nguồn.

### 5.5.2. Các lỗi đã gặp và cách khắc phục

Trong quá trình phát triển và rà soát, phần máy chủ Flask đã được sửa một số lỗi đáng chú ý:

| Lỗi | Hậu quả | Cách khắc phục |
|---|---|---|
| Phiên bản đầu bỏ qua kiểm tra khi chưa cấu hình khóa | Quên cấu hình là mọi endpoint ghi mở cho bất kỳ ai | Chuyển sang mặc định từ chối, trả 503 |
| Màn hình bếp dùng chung khóa quản trị | Trích được firmware là có toàn quyền hệ thống | Tách khóa bếp chỉ dùng cho `/api/orders/done` |
| Webhook có endpoint thống kê doanh thu | Máy chủ mở ra Internet làm lộ dữ liệu kinh doanh | Gỡ endpoint khỏi webhook |
| Hai nguồn cùng báo hoàn thành một đơn | Có nguy cơ ghi đè trạng thái | Cập nhật có điều kiện ở tầng cơ sở dữ liệu, bên đến sau nhận 409 |
| `compare_digest` báo lỗi với ký tự tiếng Việt | Máy chủ lỗi khi khóa chứa ký tự ngoài ASCII | Đổi chuỗi sang byte UTF-8 trước khi so sánh |

### 5.5.3. Hạn chế còn tồn tại và hướng cải tiến

Một đánh giá trung thực cần chỉ ra cả những điểm chưa hoàn thiện.

**Webhook đọc sai tên trường nội dung chuyển khoản.** Theo tài liệu chính thức của SePay, nội dung chuyển khoản nằm trong trường `content`, nhưng mã hiện đọc trường `transferContent`. Vì vậy trên thực tế webhook không tìm được mã đơn, và việc xác nhận thanh toán đang dựa vào cơ chế dự phòng: ứng dụng kiosk tự tra cứu giao dịch qua API của SePay mỗi 15 giây. Hệ thống vẫn hoạt động đúng nhưng chậm hơn thiết kế. Cách sửa là đọc `data.get("content") or data.get("transferContent")`, đồng thời cập nhật dữ liệu kiểm thử theo mẫu thật của SePay.

**Đang dùng máy chủ phát triển.** Hai máy chủ đang chạy bằng `app.run()`. Với lưu lượng của một cửa hàng thì đủ dùng, nhưng khi triển khai thật nên chuyển sang Waitress hoặc Gunicorn như đã trình bày ở mục 5.2.8.

**Một số endpoint chưa kiểm tra đầu vào đầy đủ.** Ví dụ `add_product` đổi giá bằng `int(float(request.form.get('price')))`: nếu thiếu trường giá, máy chủ trả lỗi 500 thay vì 400 có lý do rõ ràng. Một số endpoint dùng `request.json[...]` trực tiếp. Hướng cải tiến là kiểm tra trường bắt buộc trước khi xử lý, hoặc dùng thư viện kiểm tra dữ liệu như Pydantic hay Marshmallow.

**Đăng nhập chưa giới hạn số lần thử.** Endpoint `/api/auth/login` chưa có cơ chế chống dò mật khẩu. Có thể bổ sung Flask-Limiter hoặc khóa tạm thời sau một số lần nhập sai.

**Các endpoint đọc chưa yêu cầu xác thực.** Danh sách đơn hàng và báo cáo doanh thu đọc được từ mọi máy trong mạng nội bộ. Đây là đánh đổi có chủ đích, vì hệ thống chạy trong mạng riêng của cửa hàng. Nếu mở rộng ra nhiều chi nhánh hoặc mạng dùng chung, cần thêm khóa cho cả thao tác đọc.

**Tệp ảnh có thể bị ghi đè và chưa giới hạn loại tệp.** `secure_filename` giữ nguyên tên gốc đã làm sạch, nên hai ảnh trùng tên sẽ ghi đè nhau. Nên thêm tiền tố ngẫu nhiên vào tên tệp, chỉ chấp nhận các định dạng ảnh và đặt giới hạn dung lượng bằng cấu hình `MAX_CONTENT_LENGTH`.

### 5.5.4. Kiến thức trọng tâm

Phần này tóm tắt những ý cốt lõi của chương, dùng để ôn tập và chuẩn bị trả lời trước hội đồng.

Flask là khung phát triển web vi mô theo chuẩn WSGI, xây trên thư viện Werkzeug. Đặc trưng của nó là lõi tối thiểu, nhẹ, dễ học và linh hoạt; đổi lại, người lập trình phải tự lo phần kiểm tra dữ liệu và bảo mật. Một yêu cầu đi qua máy chủ WSGI, được Flask tạo ngữ cảnh, định tuyến đến hàm xử lý, chạy qua các decorator rồi trả về đối tượng phản hồi. Biến `request` dùng như biến toàn cục nhưng an toàn đa luồng nhờ cơ chế proxy cục bộ theo ngữ cảnh.

Dự án chọn Flask cho Smart Kiosk API và webhook vì các yêu cầu ngắn, độc lập, chạy trên Raspberry Pi tài nguyên hạn chế, và cần toàn quyền kiểm soát giao dịch SQLite. Voice Gateway dùng FastAPI vì cần WebSocket và xử lý bất đồng bộ. Đây là ví dụ của nguyên tắc chọn công cụ theo bản chất bài toán.

Về bảo mật, cần nhớ bốn kỹ thuật:
- **decorator xác thực** gom kiểm tra quyền về một chỗ, với lưu ý về `functools.wraps` và thứ tự decorator;
- **so sánh thời gian hằng** chống tấn công đo thời gian;
- **mặc định từ chối** khi thiếu cấu hình;
- **hai cấp khóa** theo nguyên tắc đặc quyền tối thiểu.

Máy chủ webhook mở ra Internet được tách riêng và không truy cập dữ liệu để giảm bề mặt tấn công.

Về thiết kế giao tiếp, mã trạng thái HTTP mang ý nghĩa nghiệp vụ. Tiêu biểu nhất là mã **409** khi hai nguồn cùng báo hoàn thành một đơn. Dữ liệu có phiên bản, và endpoint đơn chờ dùng chung định dạng với thông điệp MQTT để màn hình bếp chỉ cần một hàm phân tích.

Cuối cùng, cần thẳng thắn nhận diện các hạn chế: webhook đang đọc sai tên trường nội dung chuyển khoản (hệ thống vẫn đúng nhờ cơ chế tra cứu dự phòng), máy chủ đang chạy ở chế độ phát triển, và một số endpoint còn thiếu kiểm tra đầu vào cũng như giới hạn số lần đăng nhập. Mỗi hạn chế đều đã có hướng khắc phục cụ thể.

---

*Tài liệu tham khảo: Flask Documentation 3.x (flask.palletsprojects.com); Werkzeug Documentation; PEP 3333 – Python Web Server Gateway Interface v1.0.1; R. T. Fielding, "Architectural Styles and the Design of Network-based Software Architectures", 2000; RFC 9110 – HTTP Semantics; Tài liệu tích hợp webhook SePay (docs.sepay.vn); Python Standard Library: `hmac`, `smtplib`, `email.mime`.*

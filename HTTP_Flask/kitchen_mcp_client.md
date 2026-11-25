# Giải thích chi tiết `kitchen_mcp/client.py` và `write_client.py`

> Thư mục: `Voice_Test/voice_assistant_edge_ai/services/kitchen_mcp/src/kitchen_mcp/`
> - `client.py` — 1059 dòng (viết “dọc”, mỗi tham số một dòng nên dài; logic thực khoảng 350 dòng) — **máy khách HTTP chỉ đọc**.
> - `write_client.py` — 254 dòng — **máy khách HTTP ghi**, chỉ làm đúng một việc: báo đơn xong.
> - Có nhắc `config.py` (538 dòng) ở chỗ cần thiết.
>
> **Thí nghiệm** (không đụng Pi, CSDL thật, `.env` thật):
> 1. Dựng **Smart Kiosk API thật** trên laptop (cổng 5988) với **CSDL tạm** (2 đơn PAID, 1 đơn chưa trả tiền, 2 nguyên liệu, công thức món 1) và **khóa thử**; gọi bằng **đúng hai lớp** `KioskReadClient` / `KioskWriteClient`.
> 2. Dựng **máy chủ giả** (`http.server`) trả dữ liệu **sai hợp đồng** hoặc **trả chậm** để xem máy khách phản ứng thế nào.
> 3. Chạy test có sẵn: `test_client.py`, `test_write_client.py`, `test_config.py`, `test_write_config.py` → **74 passed**.

---

## 0. Bức tranh tổng thể (đọc trước)

### 0.1 Hai file này là gì?
**Kitchen MCP** là một “máy chủ công cụ” (*MCP server*) chạy trên **laptop** để **trợ lý giọng nói** (mô hình ngôn ngữ) hỏi được: “còn mấy đơn chờ?”, “sữa tươi còn bao nhiêu?”, “trà sữa cần nguyên liệu gì?”, và làm được: “đơn DH… xong rồi”. Bản thân MCP **không có dữ liệu** — nó hỏi Smart Kiosk API trên Pi qua HTTP. Hai file này chính là **phần nói chuyện HTTP** đó.

```
 Người nói ─▶ gateway giọng nói ─▶ LLM ─▶ gọi công cụ MCP (vd. get_inventory)
                                              │
                         server.py / data_source.py (Kitchen MCP, laptop)
                                              │
                    client.py (đọc)  ─────────┼──── write_client.py (ghi)
                     urllib GET                │      urllib POST
                                              ▼
                              Smart Kiosk API (Flask) trên Pi
```

So với hai máy khách đã học:

| | `admin_bridge.py` | `kds_firmware.ino` | **Kitchen MCP** |
|---|---|---|---|
| Ngôn ngữ / thư viện | Python + `requests` | C++ + `HTTPClient` | Python + **`urllib`** (thư viện chuẩn) |
| Khóa | admin | bếp | **bếp** |
| Lỗi báo bằng | `(ok, data)` + `lastErrorMessage` | số âm / mã HTTP | **ngoại lệ có phân loại** |
| Kiểm tra dữ liệu trả về | nhẹ | nhẹ (`is<T>()`, bỏ qua phần tử sai) | **rất chặt** (sai 1 trường → từ chối cả phản hồi) |
| Thử lại | không | có (1/2/4 s) | **không** |

**Vì sao kiểm tra chặt như vậy?** Người tiêu thụ dữ liệu ở đây là **mô hình ngôn ngữ**. Đưa cho nó dữ liệu nửa đúng nửa sai thì nó sẽ **nói sai một cách tự tin** (“sữa còn đủ”). Thà báo “không đọc được kho” còn hơn. Đây là nguyên tắc **fail‑closed**: nghi ngờ thì từ chối.

**Vì sao dùng `urllib` mà không dùng `requests`?** MCP là một dịch vụ nhỏ độc lập; dùng thư viện chuẩn thì **không thêm phụ thuộc**, ít thứ phải cài/kiểm duyệt bản quyền. Đổi lại, `urllib` dài dòng hơn (phải tự `encode`, tự `json.loads`, tự bắt nhiều loại lỗi).

### 0.2 Kết quả chạy thử

**Gọi API thật (CSDL tạm):**

| Gọi | Kết quả |
|---|---|
| `get_pending_orders()` khóa bếp | `['DH00000001', 'DH00000002']` (đơn chưa trả tiền không có) |
| `.items[0].to_dict()` | `{'product_id': 1, 'name': 'Trà sữa', 'quantity': 2, 'options': {'size': 'L', 'ice': 'Ít đá'}, 'note': 'ít ngọt'}` |
| `get_inventory()` | `[('Sữa tươi', 300.0, True), ('Trân châu', 2000.0, False)]` — `low` đúng theo ngưỡng 500 |
| `get_product_recipe(1)` | 2 nguyên liệu: sữa 100 ml, trân châu 50 g |
| `get_product_recipe(0)` | `ValueError` — **chưa gửi HTTP** đã chặn |
| `get_recent_orders(limit=2)` | `total_pages=1, current_page=1, total_items=2` |
| `get_recent_orders(limit=True)` | `ValueError` (chặn `bool` giả làm số) |
| `get_menu()` khóa bếp / **không khóa** | 34 / 34 món (menu công khai) |
| `get_pending_orders()` **không khóa** | `KioskResponseError: HTTP 401` ← `HTTPError` |
| API tắt | `KioskConnectionError` ← `URLError`, **0,00 s** |
| IP không tồn tại, timeout 1,5 s | `KioskConnectionError` ← `URLError`, **1,50 s** |
| `mark_order_done(" dh00000002 ")` | `{'success': True, 'order_id': 'DH00000002', 'kitchen_status': 'done'}` (đã chuẩn hóa) |
| Lần 2 cùng mã | `KioskWriteConflictError` ← HTTP 409 |
| `mark_order_done("DH123")` | `ValueError` — không gửi HTTP |
| Sai khóa | `KioskUnauthorizedError` ← HTTP 401 |

**Máy chủ giả trả sai hợp đồng / trả chậm:**

| Máy chủ giả trả | Máy khách |
|---|---|
| Kho: `quantity=10, threshold=50` nhưng `low=false` | `KioskResponseError: [0].low is inconsistent…` |
| Menu là HTML | `KioskResponseError: … not valid JSON` ← `JSONDecodeError` |
| Đơn chờ `"version": 2` | `KioskResponseError: Unsupported pending-orders version` |
| POST done trả `order_code` **khác** mã đã gửi | `KioskResponseError: … different order ID` |
| Im lặng 3 s **trước** khi gửi header (timeout 1 s) | `KioskConnectionError` ← `TimeoutError`, 1,00 s |
| Gửi header rồi **im lặng giữa thân** | `KioskConnectionError` ← `TimeoutError`, 1,00 s |

**Kiểm tra cấu hình (`KitchenSettings`):**

| Cấu hình | Kết quả |
|---|---|
| `repr(settings)` | **Không có** `kiosk_api_key` trong chuỗi in ra |
| Tạo `KioskWriteClient` khi `write_enabled=False` | `KioskWriteDisabledError` |
| `write_enabled=True` + chế độ `fixture` | `KitchenConfigError: … requires KITCHEN_DATA_MODE=live` |
| `write_enabled=True` + không khóa | `KitchenConfigError: KIOSK_API_KEY is required…` |
| URL `ftp://pi` | `KitchenConfigError: … must use http or https` |
| `timeout=0` | `KitchenConfigError: … greater than zero` |

### 0.3 Bản đồ

| `client.py` | Nội dung |
|---|---|
| L1–19 | Import: `json`, `re`, `dataclass`, `urllib.error`, `urllib.parse.urlencode`, `urllib.request` |
| L22 | `ORDER_ID_PATTERN = r"^DH[0-9]{8}$"` |
| L27–40 | 3 lớp ngoại lệ |
| L43–191 | 5 dataclass: `KioskOrderItem`, `KioskMenuProduct`, `KioskRecipeIngredient`, `KioskInventoryItem`, `KioskOrder` |
| L194–337 | `KioskReadClient` + 5 hàm công khai `get_*` |
| L339–419 | **`_get_json()` — trái tim** |
| L422–963 | `_parse_*`: menu, recipe, inventory, recent orders, pending orders, order, item |
| L966–1059 | `_require_dict/string/number/boolean/integer` |

| `write_client.py` | Nội dung |
|---|---|
| L28–49 | 4 ngoại lệ ghi: `KioskWriteError`, `…Unauthorized…`, `…Disabled…`, `…Conflict…` |
| L52–69 | `MarkOrderDoneResult` |
| L72–100 | `KioskWriteClient.__init__` — **cổng kiểm tra quyền ghi** |
| L102–227 | `mark_order_done()` |
| L229–254 | `_raise_http_error()` — mã HTTP → ngoại lệ |

---

## 1. Ngoại lệ có phân cấp (`client.py` L27–40, `write_client.py` L28–49)

```python
class KioskClientError(RuntimeError): ...           # gốc
class KioskConnectionError(KioskClientError): ...   # không nói chuyện được
class KioskResponseError(KioskClientError): ...     # nói được nhưng câu trả lời không dùng được

class KioskWriteError(KioskClientError): ...                # gốc của lỗi ghi
class KioskUnauthorizedError(KioskWriteError): ...          # 401
class KioskWriteDisabledError(KioskWriteError): ...         # tắt ghi / 503
class KioskWriteConflictError(KioskWriteError): ...         # 409
```
```
RuntimeError
└── KioskClientError
    ├── KioskConnectionError
    ├── KioskResponseError
    └── KioskWriteError
        ├── KioskUnauthorizedError
        ├── KioskWriteDisabledError
        └── KioskWriteConflictError
```

**Cú pháp.** `class B(A):` = B **kế thừa** A. Thân lớp chỉ có **docstring** (chuỗi đầu lớp) là đủ để thành một lớp hợp lệ.

**Vì sao làm cây ngoại lệ?** Người gọi chọn được **mức chi tiết** muốn bắt:
- `except KioskClientError` — mọi lỗi liên quan Smart Kiosk.
- `except KioskWriteConflictError` — chỉ trường hợp “đơn đã xong” (để trợ lý nói “đơn này đã hoàn thành rồi” thay vì “lỗi hệ thống”).

Đây là cách Python tương đương với `DoneResult` của firmware hay `reason` của `admin_bridge.py`: **phân loại lỗi** để phản ứng đúng.

---

## 2. Dataclass — “hợp đồng dữ liệu” (`client.py` L43–191)

```python
@dataclass(frozen=True, slots=True)
class KioskOrderItem:
    product_id: int
    name: str
    quantity: int
    size: str
    ice: str
    note: str

    def to_dict(self) -> dict[str, object]:
        return {"product_id": self.product_id, "name": self.name, "quantity": self.quantity,
                "options": {"size": self.size, "ice": self.ice}, "note": self.note}
```

**Cú pháp**
- `@dataclass` — **decorator** tự sinh `__init__`, `__repr__`, `__eq__` từ các dòng `tên: kiểu`.
- `frozen=True` — **bất biến**: gán `item.quantity = 5` → `FrozenInstanceError`. Dữ liệu đã kiểm tra thì không ai sửa lén được.
- `slots=True` (Python ≥ 3.10) — không tạo `__dict__` cho mỗi đối tượng → nhẹ hơn, và gõ sai tên thuộc tính thì báo lỗi ngay.
- `items: tuple[KioskOrderItem, ...]` — tuple **độ dài bất kỳ** toàn phần tử `KioskOrderItem`. Dùng tuple (bất biến) chứ không dùng list, cho khớp `frozen`.
- `-> dict[str, object]` — **gợi ý kiểu** trả về; Python không ép, nhưng công cụ kiểm tra kiểu và người đọc hiểu.
- `from __future__ import annotations` (L3) — gợi ý kiểu được lưu dạng chuỗi, không tính lúc chạy.

**Logic.** Máy khách chuyển JSON thô → **đối tượng có kiểu rõ ràng** → khi cần gửi cho MCP thì `to_dict()` lại. Bước giữa là chỗ **kiểm tra**. `to_dict()` gọi là “MCP‑safe dict” vì chỉ chứa kiểu JSON cơ bản (`str`, `int`, `float`, `bool`, `dict`, `list`).

Ví dụ thật từ thí nghiệm: `size`/`ice` lưu phẳng trong dataclass, nhưng `to_dict()` gom lại thành `options: {size, ice}` — đúng hình dạng API trả, để LLM thấy cùng một cấu trúc.

---

## 3. `KioskReadClient` — 5 hàm đọc công khai (L194–337)

```python
class KioskReadClient:
    _PENDING_ORDERS_PATH = "/api/orders/pending"
    _RECENT_ORDERS_PATH  = "/api/orders"
    _INVENTORY_PATH      = "/api/inventory"
    _RECIPE_PATH         = "/api/recipe"
    _MENU_PATH           = "/api/menu"

    def __init__(self, settings: KitchenSettings) -> None:
        self._settings = settings
```

- Đường dẫn là **thuộc tính lớp** (dùng chung mọi đối tượng), có `_` đầu = quy ước “nội bộ”.
- **Không có hàm nào ghi** → đúng tên “ReadClient”: dù LLM bị lừa (prompt injection) thì lớp này cũng **không thể** sửa dữ liệu.

| Hàm | Endpoint | Kiểm tra **trước** khi gửi |
|---|---|---|
| `get_pending_orders()` | `GET /api/orders/pending` | — |
| `get_inventory()` | `GET /api/inventory` | — |
| `get_menu()` | `GET /api/menu` | — |
| `get_product_recipe(product_id)` | `GET /api/recipe?product_id=…` | số nguyên ≥ 1, không phải `bool` |
| `get_recent_orders(*, limit, page, status_filter, keyword)` | `GET /api/orders?limit=…&page=…&…` | `limit` 1–500, `page` ≥ 1, chuỗi là chuỗi |

### 3.1 Kiểm tra đầu vào
```python
if (isinstance(limit, bool) or not isinstance(limit, int) or limit < 1 or limit > 500):
    raise ValueError("limit must be an integer from 1 to 500")
```
- **Vì sao chặn `bool` riêng?** Trong Python `bool` là **lớp con** của `int`: `isinstance(True, int)` là `True`, và `True == 1`. LLM có thể sinh `"limit": true` → nếu không chặn thì lặng lẽ thành `limit=1`. Thí nghiệm `limit=True` → `ValueError`. (Cùng ý với `_to_int` từ chối `bool` ở `api_server.py`.)
- Giới hạn 500 khớp với máy chủ (`max(1, min(limit, 500))`, `api_server.py` L584). Kiểm tra ở **cả hai đầu**: máy khách báo lỗi rõ ràng sớm, máy chủ vẫn tự bảo vệ.
- `*` trong `def get_recent_orders(self, *, limit=10, ...)` — mọi tham số sau `*` **bắt buộc gọi bằng tên**: `get_recent_orders(limit=2)` được, `get_recent_orders(2)` lỗi. Tránh nhầm thứ tự `limit`/`page`.

### 3.2 Chuỗi truy vấn bằng `urlencode`
```python
query = urlencode({"limit": limit, "page": page, "status_filter": status_filter, "keyword": keyword})
payload = self._get_json(self._RECENT_ORDERS_PATH + "?" + query)
```
`urlencode` **mã hóa phần trăm** ký tự đặc biệt: `keyword="trà sữa & đá"` → `keyword=tr%C3%A0+s%E1%BB%AFa+%26+%C4%91%C3%A1`. Nếu tự ghép chuỗi thì dấu `&` trong từ khóa sẽ **cắt** tham số làm đôi. (Bên `requests` là `params=`; ở firmware không cần vì không có tham số.)

---

## 4. `_get_json()` — trái tim của máy khách đọc (L339–419)

```python
def _get_json(self, path: str) -> object:
    url = self._settings.kiosk_api_base_url + path
    headers = {"Accept": "application/json"}
    if self._settings.kiosk_api_key:
        headers["X-API-Key"] = self._settings.kiosk_api_key

    request = Request(url=url, headers=headers, method="GET")

    try:
        with urlopen(request, timeout=self._settings.kiosk_api_timeout_seconds) as response:
            status = getattr(response, "status", 200)
            if status != 200:
                raise KioskResponseError(f"Smart Kiosk returned HTTP {status}")
            raw_body = response.read()
    except HTTPError as error:
        raise KioskResponseError(f"Smart Kiosk returned HTTP {error.code}") from error
    except (URLError, TimeoutError, OSError) as error:
        raise KioskConnectionError("Smart Kiosk is unavailable") from error

    try:
        decoded = raw_body.decode("utf-8")
    except UnicodeDecodeError as error:
        raise KioskResponseError("Smart Kiosk response is not valid UTF-8") from error

    try:
        return json.loads(decoded)
    except json.JSONDecodeError as error:
        raise KioskResponseError("Smart Kiosk response is not valid JSON") from error
```
(Rút gọn định dạng; logic giữ nguyên.)

### 4.1 Cú pháp từng phần
| Đoạn | Nghĩa |
|---|---|
| `Request(url=..., headers=..., method="GET")` | Đối tượng **mô tả** yêu cầu; chưa gửi |
| `Accept: application/json` | Báo máy chủ “tôi muốn JSON” (Flask vẫn trả JSON dù thiếu, nhưng đây là phép lịch sự chuẩn HTTP) |
| `urlopen(request, timeout=5.0)` | **Gửi** và nhận dòng trạng thái + header. `timeout` áp cho **từng thao tác socket** (nối, mỗi lần đọc) chứ không phải tổng thời gian |
| `with ... as response:` | **Trình quản lý ngữ cảnh**: ra khỏi khối là tự đóng kết nối, kể cả khi có ngoại lệ — tương đương `http.end()` ở mọi nhánh của firmware, nhưng **ngôn ngữ tự lo** |
| `getattr(response, "status", 200)` | Lấy thuộc tính `status`; nếu đối tượng không có (vd. đối tượng giả trong test) thì dùng 200 |
| `response.read()` | Đọc **toàn bộ thân** thành `bytes` |
| `except HTTPError` | `urlopen` **tự ném** `HTTPError` khi mã 4xx/5xx — khác `requests` (không ném, phải tự xem `status_code`) |
| `except (URLError, TimeoutError, OSError)` | Bắt **nhiều loại** trong một `except` bằng tuple |
| `raise X(...) from error` | **Nối chuỗi ngoại lệ**: lỗi mới giữ lỗi gốc trong `__cause__` → log vẫn thấy nguyên nhân thật (cột “←” trong bảng thí nghiệm chính là `__cause__`) |
| `raw_body.decode("utf-8")` | `bytes` → `str`. `urllib` trả `bytes`, không tự giải mã như `requests` (`.json()`) |

### 4.2 Thứ tự `except` — vì sao `HTTPError` phải đứng **trước**
```
OSError
└── URLError
    └── HTTPError
```
`HTTPError` là **lớp con** của `URLError`, mà `URLError` là lớp con của `OSError`. Python thử `except` **từ trên xuống**, gặp cái khớp đầu tiên là dừng. Nếu đặt `except (URLError, …)` lên trước thì **401 cũng bị coi là “mất kết nối”** — sai bản chất. Thí nghiệm: không khóa → `KioskResponseError: HTTP 401 ← HTTPError` (đúng nhánh).

### 4.3 Hai kiểu thất bại, giống mọi máy khách khác
| Tình huống | Ngoại lệ gốc | Ném ra | Thời gian (thí nghiệm) |
|---|---|---|---|
| Cổng đóng | `URLError(ConnectionRefused)` | `KioskConnectionError` | 0,00 s |
| IP không trả lời | `URLError(timeout)` | `KioskConnectionError` | = timeout (1,50 s) |
| Đã nối, chậm trước header | `TimeoutError` | `KioskConnectionError` | = timeout (1,00 s) |
| Đã có header, chậm giữa thân | `TimeoutError` | `KioskConnectionError` | = timeout (1,00 s) |
| 4xx/5xx | `HTTPError` | `KioskResponseError` | ngay |
| Thân không phải UTF‑8 / JSON | `UnicodeDecodeError` / `JSONDecodeError` | `KioskResponseError` | ngay |

`TimeoutError` được bắt riêng vì khi hết giờ **trong lúc đọc** (sau khi `urlopen` đã trả về), Python ném thẳng `TimeoutError` chứ không bọc trong `URLError` — đúng như hai dòng “chậm” ở trên.

### 4.4 Khóa có điều kiện
```python
if self._settings.kiosk_api_key:
    headers["X-API-Key"] = self._settings.kiosk_api_key
```
Không có khóa vẫn gửi được — vì `/api/menu` công khai (thí nghiệm: không khóa vẫn đọc được 34 món, nhưng đơn chờ → 401). Khi cài trên laptop, `deploy/laptop/cai_dat_laptop.sh` tạo file `kitchen-mcp.env` với `KITCHEN_DATA_MODE=live`, `KIOSK_API_BASE_URL`, `KIOSK_API_KEY` lấy từ **`KITCHEN_API_KEY`** (khóa bếp) và `KITCHEN_WRITE_ENABLED=true`.

---

## 5. Kiểm tra phản hồi — `_parse_*` và `_require_*` (L422–1059)

### 5.1 Bộ “người gác cổng” `_require_*`
```python
@staticmethod
def _require_integer(value: object, field: str, *, minimum: int) -> int:
    if isinstance(value, bool) or not isinstance(value, int):
        raise KioskResponseError(f"{field} must be an integer")
    if value < minimum:
        raise KioskResponseError(f"{field} must be at least {minimum}")
    return value
```
| Hàm | Chặn |
|---|---|
| `_require_dict` | không phải `dict` |
| `_require_string(…, allow_empty=False)` | không phải `str`; chuỗi rỗng sau `strip()` (trừ khi cho phép) |
| `_require_number(…, minimum)` | `bool`, không phải `int`/`float`, nhỏ hơn mức tối thiểu; trả về `float` |
| `_require_boolean` | không phải `bool` thật (chuỗi `"true"` hay số `1` đều bị từ chối) |
| `_require_integer(…, minimum)` | `bool`, không phải `int`, nhỏ hơn mức tối thiểu |

- `@staticmethod` — hàm nằm trong lớp nhưng **không cần** `self`/`cls`.
- `@classmethod` (ở các `_parse_*`) — nhận `cls`, gọi được `cls._require_*`.
- Tham số `field` (vd. `"data[0].items[1].quantity"`) để **thông báo lỗi chỉ đúng chỗ hỏng** — như đường dẫn JSON.

### 5.2 Từng bộ phân tích
- **`_parse_inventory`** (L547): ngoài kiểu, còn kiểm tra **nhất quán**: `low` phải bằng `quantity <= low_stock_threshold`. Thí nghiệm: máy chủ giả trả `10 ≤ 50` mà `low=false` → từ chối. Đây là kiểm tra **chéo** giữa máy chủ và máy khách: nếu một bên đổi quy tắc (vd. `<` thay `<=`) thì phát hiện ngay chứ không để LLM báo sai “còn đủ”.
- **`_parse_recipe`** (L488): `qty_per_unit` phải **> 0** (công thức cần 0 ml sữa là vô nghĩa).
- **`_parse_pending_orders`** (L770) + **`_parse_order`** (L819) + **`_parse_item`** (L898): kiểm tra **phiên bản hợp đồng** `version == 1` và `event == "orders.pending.snapshot"` / `"order.snapshot"`; mã đơn phải khớp `^DH[0-9]{8}$` (sau `.upper()`). Thí nghiệm `version: 2` → từ chối. **Vì sao có trường version?** Nếu sau này máy chủ đổi định dạng, máy khách cũ **biết mình không hiểu** thay vì đọc sai.
- **`_parse_recent_orders`** (L634): đọc phong bì `{data, totalPages, currentPage, totalItems}` và đổi sang tên kiểu Python (`total_pages`…); `status` chỉ nhận `paid`/`cancelled`, `kitchen_status` chỉ `pending`/`done`/`none` — khớp chú thích trong `api_server.py` L613–622.
- **`_parse_menu`** (L422): `price ≥ 0`, `status` là `bool`, `image` được rỗng.

**Đặc điểm chung: “một hỏng, hỏng cả”.** Firmware bỏ qua phần tử sai và dùng phần còn lại; ở đây chỉ cần **một** trường sai là cả phản hồi bị từ chối. Hai lựa chọn hợp với hai người dùng: màn hình bếp thì “hiện được bao nhiêu hay bấy nhiêu”, còn LLM thì “nói thiếu một món” có thể nguy hiểm hơn “nói không đọc được”.

---

## 6. `KioskWriteClient` — ghi có **nhiều lớp chốt chặn**

### 6.1 Cổng vào: `__init__` (L77–100)
```python
if not isinstance(settings, KitchenSettings):
    raise TypeError("settings must be KitchenSettings")
if not settings.write_enabled:
    raise KioskWriteDisabledError("Kitchen write capability is disabled")
if not settings.kiosk_api_key:
    raise KitchenConfigError("KIOSK_API_KEY is required")
```
**Không tạo được đối tượng** nếu chưa bật ghi hoặc thiếu khóa. Cộng với `config.py`:
- `write_enabled=True` **bắt buộc** `data_mode="live"` (không ghi khi đang chạy dữ liệu mẫu);
- và bắt buộc có khóa.

Và `server.py`: chỉ khi `write_enabled` mới **đăng ký** công cụ `mark_order_done` cho LLM (`_enable_write_tool`), nếu không thì LLM **không hề thấy** công cụ ghi. Công cụ được gắn nhãn `destructiveHint=True`, mô tả “requires authorization and confirmation” để gateway hỏi xác nhận trước khi gọi.

→ **Phòng thủ nhiều lớp** (*defense in depth*): cấu hình → hàm tạo → đăng ký công cụ → xác nhận của người dùng → khóa API phía máy chủ → điều kiện `UPDATE ... WHERE kitchen_status='pending'` trong CSDL.

### 6.2 `mark_order_done()` (L102–227)
```python
normalized_order_id = str(order_id).strip().upper()
if not ORDER_ID_PATTERN.fullmatch(normalized_order_id):
    raise ValueError("order_id must match DH followed by 8 digits")

body = json.dumps({"order_code": normalized_order_id}).encode("utf-8")
request = Request(url=..., data=body,
                  headers={"Accept": "application/json",
                           "Content-Type": "application/json",
                           "X-API-Key": self._settings.kiosk_api_key},
                  method="POST")
```
**Cú pháp**
- `str(order_id).strip().upper()` — chuẩn hóa: LLM có thể sinh `" dh00000002 "` (thí nghiệm: vẫn thành công với mã `DH00000002`).
- `fullmatch` — khớp **cả chuỗi** (dù mẫu đã có `^…$`; dùng `fullmatch` là chắc chắn hai lần). So với webhook dùng `search` vì ở đó mã nằm **lẫn trong** nội dung chuyển khoản.
- `json.dumps(...).encode("utf-8")` — `urllib` cần **`bytes`** cho `data=`; có `data` thì mặc định là POST, nhưng vẫn ghi rõ `method="POST"`.
- `Content-Type: application/json` — như firmware: thiếu thì Flask `get_json()` không đọc → 400.

**Kiểm tra phản hồi thành công — ba điều kiện:**
```python
if payload.get("success") is not True: raise ...
if returned_order_id != normalized_order_id: raise KioskResponseError("… different order ID")
if payload.get("kitchen_status") != "done": raise ...
```
- `is not True` — chỉ chấp nhận đúng giá trị `True` (không chấp nhận `1`, `"true"`).
- **So mã đơn trả về với mã đã gửi**: thí nghiệm máy chủ giả trả `DH00000099` khi gửi `DH00000001` → từ chối. Không để trợ lý nói “đã xong đơn 01” khi thực ra máy chủ xác nhận một đơn khác.

### 6.3 `_raise_http_error()` (L229–254) — mã HTTP → ngoại lệ
| Mã | Ngoại lệ | Trợ lý có thể nói |
|---|---|---|
| 401 | `KioskUnauthorizedError` | “Khóa API bị từ chối” |
| **409** | `KioskWriteConflictError` | “Đơn không còn chờ / đã xong rồi” |
| 503 | `KioskWriteDisabledError` | “Máy chủ chưa cấu hình khóa” |
| khác (400, 403, 500…) | `KioskWriteError` | lỗi chung |

**So sánh cách xử lý 409:**
- Firmware: 409 = **thành công** (mục tiêu là gỡ thẻ; đơn đã xong thì thẻ phải biến mất).
- Kitchen MCP: 409 = **ngoại lệ riêng**. Vì ở đây có **người hỏi**: nếu đầu bếp nói “xong đơn 02” mà đơn 02 đã xong từ trước (hoặc đọc nhầm mã), trợ lý nên **nói rõ** thay vì xác nhận “đã hoàn thành” — có thể người nói nhầm sang đơn khác.

**Không thử lại** — chủ ý: thao tác ghi do người ra lệnh; lỗi thì báo lại cho người quyết định. (Và nhờ điều kiện CSDL, gửi lại cũng an toàn, chỉ ra 409.)

---

## 7. `KitchenSettings` — cấu hình được kiểm tra một lần (`config.py`)

```python
@dataclass(frozen=True, slots=True)
class KitchenSettings:
    data_mode: str = "fixture"
    kiosk_api_base_url: str = "http://127.0.0.1:5000"
    kiosk_api_timeout_seconds: float = 5.0
    write_enabled: bool = False
    delayed_order_minutes: int = 15
    kiosk_api_key: str = field(default="", repr=False)

    def __post_init__(self) -> None:
        ...                                       # chuẩn hóa + kiểm tra từng trường
        object.__setattr__(self, "kiosk_api_base_url", normalized_base_url)
```
- **`field(repr=False)`** — khóa **không xuất hiện** khi in đối tượng. Thí nghiệm: `print(settings)` chỉ hiện `data_mode`, URL, timeout, `write_enabled`, `delayed_order_minutes`. Log hay thông báo lỗi có in cấu hình cũng không lộ khóa.
- **`__post_init__`** — dataclass gọi hàm này **sau** `__init__` tự sinh; chỗ để kiểm tra.
- **`object.__setattr__(self, …)`** — dataclass `frozen` chặn `self.x = …`; muốn ghi giá trị đã chuẩn hóa thì phải “đi cửa sau” qua `object.__setattr__`. Chỉ dùng **trong** `__post_init__`.
- `_normalize_base_url`: bỏ `/` cuối (thí nghiệm truyền `http://127.0.0.1:5988/` → lưu `http://127.0.0.1:5988`, để `base + "/api/..."` không thành `//api`), chỉ cho `http`/`https`, phải có host.
- Timeout phải hữu hạn và > 0 (`math.isfinite`) — vì `urlopen(timeout=None)` nghĩa là **chờ vô hạn**, giống cạm bẫy của `requests`.
- Giá trị mặc định đều **an toàn**: `fixture` (dữ liệu mẫu, không gọi mạng), ghi tắt.

**Mặc định của `KitchenSettings` từ biến môi trường** (`from_environment`): `KITCHEN_DATA_MODE`, `KIOSK_API_BASE_URL`, `KIOSK_API_TIMEOUT_SECONDS`, `KITCHEN_WRITE_ENABLED` (nhận `1/true/yes/on` và `0/false/no/off`), `KITCHEN_DELAYED_ORDER_MINUTES`, `KIOSK_API_KEY`.

---

## 8. Test được viết thế nào (đọc thêm)
`tests/test_client.py` **không cần máy chủ**: dùng `monkeypatch.setattr("kitchen_mcp.client.urlopen", fake_urlopen)` thay hàm `urlopen` bằng hàm giả trả `FakeResponse` (có `status`, `read()`, `__enter__`/`__exit__`). Đây là lý do code dùng `getattr(response, "status", 200)`: đối tượng giả đơn giản vẫn chạy được. Lưu ý: phải `from urllib.request import urlopen` **vào module** `client` thì mới thay được bằng đường dẫn `kitchen_mcp.client.urlopen`.

Chạy: `PYTHONPATH=src python3 -m pytest tests/test_client.py tests/test_write_client.py tests/test_config.py tests/test_write_config.py` → **74 passed**.

---

## 9. Tổng kết

### 9.1 Ba máy khách HTTP của cùng một API
| Tiêu chí | `admin_bridge.py` | `kds_firmware.ino` | Kitchen MCP |
|---|---|---|---|
| Timeout | 3/5/10 s | 2,5 s nối + 4 s đọc | 5 s (cấu hình được) |
| Đóng kết nối | `requests` tự lo | `http.end()` thủ công | `with urlopen(...)` |
| Lỗi mạng | `RequestException` | mã âm | `URLError`/`TimeoutError` → `KioskConnectionError` |
| 4xx/5xx | đọc `status_code` + `reason` | đọc `status` | `HTTPError` (tự ném) |
| 409 khi báo xong | — | **thành công** | **ngoại lệ riêng** |
| Thử lại | không | 1/2/4 s | không |
| Dữ liệu sai | dùng tạm | bỏ phần tử sai | **từ chối cả phản hồi** |
| Gửi JSON | `json=` | `serializeJson` | `json.dumps().encode()` |
| Tham số URL | `params=` | (không dùng) | `urlencode` |

### 9.2 Ghi chú (không gây lỗi hiện tại, **chưa sửa**)
1. **403 không có ngoại lệ riêng** trong `write_client`: rơi vào `KioskWriteError("HTTP 403")`. Hiện khóa bếp có quyền báo xong nên không gặp; nếu sau này cấu hình nhầm sang khóa không đủ quyền thì thông báo kém rõ hơn 401.
2. **`status != 200` gần như không bao giờ đúng**: `urlopen` đã ném `HTTPError` cho 4xx/5xx, còn 3xx thì tự chuyển hướng. Chỉ các mã 2xx khác 200 (201, 204) mới rơi vào nhánh này — máy chủ không trả các mã đó. Đây là kiểm tra “phòng xa”.
3. **Timeout là theo từng thao tác socket, không phải tổng thời gian.** Máy chủ nhỏ giọt từng byte, mỗi lần nhanh hơn 5 s, thì về lý thuyết một yêu cầu có thể kéo dài hơn 5 s. Với API cục bộ thì không thực tế.
4. **“cancelled” ở `/api/orders` nghĩa là “không PAID”**, gồm cả đơn đang chờ thanh toán/hết hạn (máy chủ gộp, `api_server.py` L606–615 — trường chi tiết nằm ở `payment_status`, Kitchen MCP không đọc). Khi trợ lý trả lời “đơn bị hủy” cần hiểu theo nghĩa rộng này.
5. **Kiểm tra `low` chặt**: nếu sau này đổi quy tắc ngưỡng ở máy chủ mà quên sửa máy khách, công cụ đọc kho sẽ **lỗi toàn bộ** (đúng tinh thần fail‑closed, nhưng cần nhớ sửa hai nơi).
6. **HTTP không mã hóa** (cấu hình đã chấp nhận `https`, nhưng API trên Pi chạy HTTP).

---

## 10. Câu hỏi tự kiểm tra

1. Vì sao Kitchen MCP dùng `urllib` mà không dùng `requests`? Đổi lại phải tự làm thêm những gì?
2. Đổi chỗ hai khối `except HTTPError` và `except (URLError, TimeoutError, OSError)` thì gọi đơn chờ không khóa sẽ ra ngoại lệ gì? Vì sao?
3. `raise KioskConnectionError(...) from error` — `from error` giúp gì khi đọc log?
4. Vì sao `_require_integer` phải kiểm tra `isinstance(value, bool)` **trước** `isinstance(value, int)`?
5. `keyword="sữa & đá"` — nếu ghép chuỗi tay `"?keyword=" + keyword` thì máy chủ nhận được gì?
6. Firmware coi 409 là thành công, Kitchen MCP coi 409 là lỗi riêng. Giải thích vì sao **cả hai đều đúng** trong ngữ cảnh của mình.
7. Liệt kê các lớp chốt chặn phải vượt qua để LLM thực sự đổi được trạng thái một đơn.
8. `frozen=True` làm `self.x = ...` bị cấm, vậy `__post_init__` ghi giá trị đã chuẩn hóa bằng cách nào?
9. Vì sao `kiosk_api_key` dùng `field(repr=False)`? Viết một tình huống cụ thể mà thiếu nó sẽ lộ khóa.
10. Máy chủ trả kho có một nguyên liệu `low` sai. Firmware và Kitchen MCP xử lý khác nhau thế nào? Bạn chọn cách nào cho từng thiết bị, vì sao?
11. Mã đơn trả về khác mã đã gửi — chuyện này có thể xảy ra khi nào trong thực tế, và nếu không kiểm tra thì hậu quả là gì?
12. `timeout=5` của `urlopen` và `setConnectTimeout(2500)` + `setTimeout(4000)` của firmware khác nhau ở điểm nào?

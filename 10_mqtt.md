# CHƯƠNG 10. MQTT – GIAO THỨC TRUYỀN SỰ KIỆN THỜI GIAN THỰC

## Mục tiêu của chương

Sau khi học xong chương này, người đọc cần nắm được:

- MQTT là gì, ra đời trong bối cảnh nào và vì sao trở thành giao thức phổ biến nhất của Internet vạn vật (IoT).
- Cơ chế hoạt động của MQTT: mô hình xuất bản – đăng ký qua broker, vòng đời kết nối, cấu trúc gói tin, chủ đề, ba mức chất lượng dịch vụ (QoS), thông điệp lưu giữ, phiên làm việc, keep-alive và di chúc.
- Các vấn đề bảo mật của MQTT, và cách chữ ký HMAC bảo vệ nội dung thông điệp ngay cả khi broker không đáng tin cậy.
- Lý do dự án chọn MQTT làm kênh đẩy sự kiện, đồng thời vẫn giữ HTTP song song.
- Cách MQTT được áp dụng cụ thể trong dự án ở cả ba phía: máy chủ Python, ứng dụng kiosk và màn hình bếp ESP32.
- Các kỹ thuật thiết kế đã áp dụng, các lỗi đã gặp và những hạn chế còn tồn tại, đủ để trình bày và trả lời câu hỏi trước hội đồng.

**Vị trí trong dự án.** MQTT là "hệ thần kinh" truyền sự kiện giữa các khối của hệ thống:

| Thành phần | Tệp mã nguồn | Vai trò MQTT |
|---|---|---|
| Mô-đun bảo mật | `Smart_Kiosk_Backup/backend/mqtt_security.py` | Ký, xác minh thông điệp; cấu hình tài khoản, TLS |
| Webhook SePay | `Smart_Kiosk_Backup/servers/webhook_server.py` | Phát tín hiệu "đã thanh toán" |
| Ứng dụng kiosk | `Smart_Kiosk_Backup/kiosk_app/bridge.py` | Nhận tín hiệu thanh toán; phát đơn mới và tồn kho xuống bếp |
| Smart Kiosk API | `Smart_Kiosk_Backup/servers/api_server.py` | Phát tồn kho sau khi quản trị viên sửa kho |
| Màn hình bếp (KDS) | `kds_firmware.ino/kds_firmware/kds_firmware.ino` | Nhận đơn mới và tồn kho, kiểm tra chữ ký |
| Broker nội bộ | `Smart_Kiosk_Backup/deploy/mosquitto/` | Cấu hình Mosquitto có tài khoản và phân quyền |

Thư viện sử dụng: paho-mqtt 2.x (Python), PubSubClient (ESP32/Arduino), mbedTLS (tính HMAC trên ESP32), Mosquitto (broker).

---

## 10.1. Khái niệm và đặc điểm

### 10.1.1. MQTT là gì

**MQTT** (*Message Queuing Telemetry Transport*) là một **giao thức truyền thông điệp** ở tầng ứng dụng, hoạt động theo mô hình **xuất bản – đăng ký** (*publish – subscribe*) và chạy trên nền TCP/IP. Nói một cách hình ảnh, MQTT giống một **tòa soạn báo**: người viết bài gửi bài về tòa soạn theo từng chuyên mục, người đọc đăng ký chuyên mục mình quan tâm, và tòa soạn chịu trách nhiệm chuyển bài đến đúng người đọc. Người viết và người đọc không cần biết nhau.

Trong MQTT có ba vai trò:

| Vai trò | Nhiệm vụ | Trong dự án |
|---|---|---|
| **Bên xuất bản** (*publisher*) | Gửi thông điệp lên một **chủ đề** (*topic*) | Webhook, kiosk, Smart Kiosk API |
| **Bên đăng ký** (*subscriber*) | Báo với broker mình quan tâm chủ đề nào, rồi nhận thông điệp của chủ đề đó | Kiosk, màn hình bếp |
| **Broker** (máy chủ trung gian) | Nhận mọi thông điệp và chuyển đến đúng các bên đã đăng ký | `broker.emqx.io` khi thử nghiệm; Mosquitto nội bộ khi triển khai |

Một thiết bị có thể vừa là bên xuất bản vừa là bên đăng ký. Ví dụ ứng dụng kiosk **nhận** tín hiệu thanh toán nhưng lại **phát** đơn hàng xuống bếp. Tất cả các bên, gọi chung là **máy khách** (*client*), đều chỉ kết nối tới broker chứ không kết nối trực tiếp với nhau.

### 10.1.2. Nguồn gốc và chuẩn hóa

MQTT được Andy Stanford-Clark (IBM) và Arlen Nipper (Arcom) thiết kế năm 1999 để giám sát đường ống dẫn dầu qua **kết nối vệ tinh**: băng thông thấp, chập chờn và tính tiền theo từng byte. Yêu cầu khắt khe đó định hình toàn bộ triết lý của giao thức: **gói tin phải nhỏ nhất có thể**, **thiết bị đầu cuối phải đơn giản**, và **phải chịu được mạng không ổn định**. Tiêu đề nhỏ nhất của một gói tin MQTT chỉ có 2 byte, trong khi một yêu cầu HTTP thông thường mang theo hàng trăm byte tiêu đề.

Năm 2013, IBM chuyển giao MQTT cho tổ chức tiêu chuẩn mở OASIS. Các mốc quan trọng:
- **MQTT 3.1.1 (2014):** chuẩn OASIS, sau đó được công nhận thành tiêu chuẩn quốc tế ISO/IEC 20922:2016. Đây là phiên bản được hỗ trợ rộng rãi nhất.
- **MQTT 5.0 (2019):** bổ sung mã lý do chi tiết, thuộc tính người dùng, thời hạn thông điệp, đăng ký dùng chung.

Ngày nay MQTT là giao thức phổ biến nhất của IoT. Nó được dùng trong nhà thông minh, nhà máy, xe điện, và cả các nền tảng đám mây lớn như AWS IoT Core hay Azure IoT Hub. Dự án dùng MQTT 3.1.1, vì đây là phiên bản mà thư viện PubSubClient trên ESP32 hỗ trợ.

### 10.1.3. Đặc điểm nổi bật

**Tách rời bên gửi và bên nhận.** Đây là đặc điểm quan trọng nhất. Bên gửi chỉ cần biết địa chỉ broker và tên chủ đề, không cần biết có bao nhiêu bên nhận, chúng ở đâu, đang bật hay tắt. Sự tách rời thể hiện ở ba chiều:
- **Không gian:** hai bên không cần biết địa chỉ IP của nhau.
- **Thời gian:** nhờ thông điệp lưu giữ và phiên bền vững, hai bên không nhất thiết trực tuyến cùng lúc.
- **Đồng bộ:** bên gửi phát xong là đi tiếp, không bị chặn để chờ bên nhận xử lý.

Trong dự án, kiosk không cần biết địa chỉ màn hình bếp. Muốn thêm màn bếp thứ hai, chỉ cần cho nó đăng ký cùng chủ đề, không phải sửa một dòng mã nào ở kiosk.

**Gọn nhẹ.** Gói tin nhỏ, thư viện máy khách nhỏ, phù hợp với vi điều khiển chỉ có vài trăm kilobyte RAM như ESP32.

**Máy chủ chủ động đẩy dữ liệu.** Kết nối TCP được giữ mở liên tục, nên khi có thông điệp, broker đẩy ngay xuống bên nhận. Bên nhận không phải hỏi định kỳ "có gì mới không?". Nhờ vậy đơn hàng xuất hiện trên màn hình bếp gần như tức thì.

**Ba mức chất lượng dịch vụ.** Lập trình viên chọn được mức đánh đổi giữa độ tin cậy và chi phí cho từng loại thông điệp.

**Hỗ trợ sẵn cho mạng chập chờn:**
- **Thông điệp lưu giữ** giúp thiết bị vừa kết nối nhận ngay trạng thái mới nhất.
- **Keep-alive** giúp phát hiện kết nối đã chết.
- **Di chúc** cho phép broker báo hộ khi một thiết bị mất kết nối bất thường.

Bên cạnh đó, MQTT có những **hạn chế** cần biết:
- **Cần broker trung gian.** Broker hỏng thì toàn bộ kênh sự kiện ngừng hoạt động.
- **Không tự bảo vệ nội dung.** Giao thức không mã hóa, không ký thông điệp; bảo mật phụ thuộc vào cấu hình broker, TLS hoặc cơ chế do ứng dụng tự xây.
- **Không có khái niệm "phản hồi".** Bên gửi không biết bên nhận đã xử lý thành công hay chưa.
- **Không quy định định dạng nội dung.** Các bên phải tự thống nhất cấu trúc dữ liệu.

Dự án khắc phục hạn chế thứ hai bằng chữ ký HMAC. Với hạn chế thứ ba, dự án dùng HTTP cho những thao tác cần biết kết quả.

| Đặc điểm | Ý nghĩa với dự án |
|---|---|
| Xuất bản – đăng ký | Kiosk không cần biết IP của màn hình bếp; thêm màn bếp không phải sửa kiosk |
| Gọn nhẹ | Chạy tốt trên ESP32-S3 |
| Đẩy tức thì | Đơn hiện trên bếp ngay khi khách thanh toán xong |
| QoS 0/1/2 | Đơn hàng và tín hiệu tiền dùng QoS 1 |
| Thông điệp lưu giữ | Màn bếp vừa bật nhận ngay bảng tồn kho |
| Không tự bảo mật nội dung | Dự án tự ký HMAC-SHA256 cho mọi thông điệp |
| Không có phản hồi | Báo hoàn thành đơn (cần biết kết quả) dùng HTTP |

---

## 10.2. Nguyên lý hoạt động

### 10.2.1. Mô hình xuất bản – đăng ký qua broker

Trung tâm của MQTT là broker. Broker giữ một **bảng đăng ký** ghi nhận máy khách nào quan tâm chủ đề nào. Khi nhận một thông điệp, broker tra bảng và chuyển thông điệp đến mọi máy khách phù hợp. Hình dưới minh họa luồng thông điệp trong dự án:

```
                         ┌──────────────────── BROKER ────────────────────┐
 Webhook ─PUBLISH payment/success─────►│ payment/success   → [kiosk]                   │──► Kiosk
 Kiosk ───PUBLISH kitchen/new_order───►│ kitchen/new_order → [KDS]                     │──► KDS
 Kiosk, API ─PUBLISH kitchen/inventory►│ kitchen/inventory → [KDS]  (giữ bản cuối cùng)│──► KDS
                         └────────────────────────────────────────────────┘
```

So sánh với HTTP sẽ thấy rõ sự khác biệt. Trong HTTP, máy khách phải biết địa chỉ máy chủ và gọi trực tiếp; một yêu cầu chỉ đến được một máy chủ. Trong MQTT, mọi bên chỉ nói chuyện với broker; một thông điệp có thể đến nhiều bên nhận, và việc thêm hay bớt bên nhận hoàn toàn trong suốt với bên gửi.

### 10.2.2. Vòng đời kết nối và các gói điều khiển

MQTT chạy trên một **kết nối TCP lâu dài**: cổng 1883 khi không mã hóa, cổng 8883 khi dùng TLS. Mọi trao đổi giữa máy khách và broker đều là các **gói điều khiển** (*control packet*). Vòng đời một kết nối diễn ra như sau:

```
Máy khách                                     Broker
   │── Mở kết nối TCP tới cổng 1883 ─────────►│
   │── CONNECT (client_id, tài khoản,         │
   │           keep_alive=60, clean session) ►│
   │◄──────────────────── CONNACK (mã 0) ─────│  0 = chấp nhận kết nối
   │── SUBSCRIBE kitchen/new_order, QoS 1 ────►│
   │◄──────────────────── SUBACK (QoS cấp) ───│
   │            ... thời gian rảnh ...         │
   │── PINGREQ ───────────────────────────────►│  khi không có gói nào trong keep_alive giây
   │◄──────────────────────────── PINGRESP ───│
   │◄───────────── PUBLISH kitchen/new_order ─│  khi có bên phát thông điệp
   │── PUBACK ────────────────────────────────►│  xác nhận (với QoS 1)
   │── DISCONNECT ────────────────────────────►│  đóng phiên chủ động
```

| Gói | Hướng | Ý nghĩa |
|---|---|---|
| CONNECT / CONNACK | Máy khách → broker / broker → máy khách | Mở phiên, xác thực, báo kết quả |
| PUBLISH | Hai chiều | Mang thông điệp: chủ đề, nội dung, QoS, cờ lưu giữ |
| PUBACK, PUBREC, PUBREL, PUBCOMP | Hai chiều | Xác nhận cho QoS 1 và QoS 2 |
| SUBSCRIBE / SUBACK | Máy khách → broker / ngược lại | Đăng ký chủ đề; broker trả về mức QoS được cấp |
| UNSUBSCRIBE / UNSUBACK | Máy khách → broker / ngược lại | Hủy đăng ký |
| PINGREQ / PINGRESP | Máy khách → broker / ngược lại | Duy trì kết nối, phát hiện kết nối chết |
| DISCONNECT | Máy khách → broker | Đóng phiên chủ động |

Mã trong CONNACK cho biết lý do khi kết nối bị từ chối. Ví dụ, mã 4 nghĩa là sai tên đăng nhập hoặc mật khẩu, mã 5 nghĩa là không được phép kết nối. Thư viện PubSubClient trên ESP32 trả về giá trị này qua hàm `state()`, và firmware màn hình bếp in ra để gỡ lỗi.

### 10.2.3. Cấu trúc gói tin

Mọi gói MQTT gồm ba phần:

```
┌──── Tiêu đề cố định (2–5 byte) ────┐┌── Tiêu đề biến đổi ──┐┌──── Nội dung ────┐
│ Byte 1: loại gói (4 bit)           ││ Tên chủ đề,           ││ Dữ liệu thô,     │
│         + cờ DUP, QoS (2 bit),     ││ mã gói (khi QoS > 0)  ││ định dạng tùy ý  │
│         RETAIN                     ││                       ││                  │
│ Byte 2–5: độ dài phần còn lại      ││                       ││                  │
└────────────────────────────────────┘└───────────────────────┘└──────────────────┘
```

Trường **độ dài phần còn lại** được mã hóa theo kiểu **độ dài biến đổi**: mỗi byte dùng 7 bit chứa dữ liệu, bit cao nhất báo "còn byte tiếp theo". Một byte biểu diễn được độ dài tới 127. Bốn byte biểu diễn được tới khoảng 256 MB. Nhờ vậy gói tin ngắn chỉ tốn đúng một byte cho trường độ dài.

Phần **nội dung** là chuỗi byte thô; MQTT không quan tâm bên trong là gì. Dự án dùng chuỗi JSON mã hóa UTF-8, bọc trong một "phong bì" có chữ ký (mục 10.2.9).

### 10.2.4. Chủ đề và ký tự đại diện

**Chủ đề** là chuỗi UTF-8 có phân cấp, các cấp ngăn cách bằng dấu `/`, phân biệt chữ hoa và chữ thường, ví dụ `kitchen/new_order`. Chủ đề không cần khai báo trước: broker tự tạo khi có bên phát hoặc đăng ký.

Khi **đăng ký**, máy khách có thể dùng hai ký tự đại diện:

| Ký tự | Ý nghĩa | Ví dụ | Khớp với |
|---|---|---|---|
| `+` | Đúng **một** cấp | `kitchen/+` | `kitchen/new_order`, `kitchen/inventory`; không khớp `kitchen/a/b` |
| `#` | **Mọi** cấp phía sau; chỉ đặt ở cuối | `kitchen/#` | Mọi chủ đề bắt đầu bằng `kitchen/` |

Ký tự đại diện chỉ dùng khi đăng ký, không dùng khi phát. Quy ước đặt tên tốt là đi từ tổng quát đến cụ thể (`<khu vực>/<loại sự kiện>`), không bắt đầu bằng `/`, không chứa khoảng trắng. Dự án theo đúng quy ước này: nhóm `kitchen/` cho mọi thứ liên quan đến bếp, nhóm `payment/` cho thanh toán.

### 10.2.5. Ba mức chất lượng dịch vụ (QoS)

QoS (*Quality of Service*) quy định mức bảo đảm khi chuyển một thông điệp:

```
QoS 0 – tối đa một lần ("gửi rồi quên")
  Bên gửi ──PUBLISH──► Bên nhận                 Có thể mất, không bao giờ trùng

QoS 1 – ít nhất một lần
  Bên gửi ──PUBLISH (mã 7)──► Bên nhận
  Bên gửi ◄──── PUBACK (7) ── Bên nhận           Chưa nhận PUBACK thì gửi lại (cờ DUP)
                                                 Không mất, nhưng CÓ THỂ TRÙNG

QoS 2 – đúng một lần (bắt tay bốn bước)
  Bên gửi ──PUBLISH (mã 7)──► Bên nhận   (lưu mã 7, chưa giao cho ứng dụng)
  Bên gửi ◄──── PUBREC (7) ── Bên nhận
  Bên gửi ──── PUBREL (7) ──► Bên nhận   (giao cho ứng dụng, xóa mã 7)
  Bên gửi ◄─── PUBCOMP (7) ── Bên nhận           Không mất, không trùng, nhưng chậm nhất
```

QoS 1 có thể gây **trùng lặp** vì lý do sau: bên nhận đã nhận và xử lý thông điệp, nhưng gói PUBACK bị mất trên đường về. Bên gửi tưởng thông điệp chưa đến nên gửi lại, và bên nhận nhận cùng một thông điệp hai lần. QoS 2 loại bỏ trùng lặp bằng cách bắt tay thêm hai bước, đổi lại tốn gấp đôi số gói tin. Trong thực tế, cách phổ biến là dùng **QoS 1 kết hợp khử trùng ở bên nhận**. Dự án áp dụng đúng cách này, vì thư viện PubSubClient không hỗ trợ QoS 2.

Một điểm rất quan trọng và hay bị hiểu nhầm: **QoS áp dụng riêng cho từng chặng**, chặng bên phát → broker và chặng broker → bên nhận. Mức QoS mà bên nhận thực sự nhận được là:

$$\text{QoS}_{\text{giao}} = \min(\text{QoS}_{\text{phát}},\ \text{QoS}_{\text{đăng ký}})$$

Như vậy bên phát dùng QoS 1 nhưng bên nhận đăng ký QoS 0 thì chặng thứ hai chỉ là QoS 0. Mục 10.5.3 sẽ chỉ ra đây chính là một hạn chế còn tồn tại trong dự án.

### 10.2.6. Thông điệp lưu giữ

Khi phát với cờ **retain** bật, broker **giữ lại thông điệp cuối cùng** của chủ đề đó. Mọi máy khách đăng ký chủ đề này **về sau** sẽ nhận ngay bản được giữ, không phải chờ lần phát kế tiếp. Muốn xóa bản đang giữ, phát một thông điệp rỗng có cờ retain.

Cần phân biệt hai loại dữ liệu:
- **Trạng thái hiện tại** (*state*), như bảng tồn kho, nhiệt độ hay trạng thái bật/tắt, **nên** dùng lưu giữ: thiết bị mới kết nối cần biết ngay giá trị hiện tại.
- **Sự kiện** (*event*), như một đơn hàng mới, **không nên** dùng lưu giữ: nếu broker giữ đơn cuối cùng, mỗi lần màn hình bếp khởi động lại sẽ nhận lại một đơn cũ, có thể đã hoàn thành từ lâu.

Dự án áp dụng đúng nguyên tắc này: chủ đề `kitchen/inventory` có lưu giữ, chủ đề `kitchen/new_order` thì không.

### 10.2.7. Phiên làm việc, keep-alive và di chúc

**Phiên sạch và phiên bền vững.** Khi kết nối, máy khách chọn một trong hai loại phiên:

| | Phiên sạch (*clean session*) | Phiên bền vững (*persistent session*) |
|---|---|---|
| Khi ngắt kết nối | Broker xóa mọi đăng ký và thông điệp chờ | Broker giữ đăng ký và **lưu thông điệp QoS 1, 2** phát trong lúc vắng mặt |
| Khi kết nối lại | Máy khách phải đăng ký lại | Máy khách nhận bù các thông điệp đã lỡ |
| Yêu cầu | – | `client_id` phải cố định giữa các lần kết nối |

Dự án dùng phiên sạch, là mặc định của cả paho-mqtt và PubSubClient. Sau mỗi lần kết nối lại, các máy khách tự đăng ký lại. Những đơn bị lỡ trong lúc mất kết nối được bù bằng đồng bộ HTTP từ cơ sở dữ liệu, không dựa vào khả năng lưu trữ của broker.

**Keep-alive.** Khi kết nối, máy khách khai báo khoảng thời gian keep-alive, ví dụ 60 giây. Nếu trong khoảng đó không có gói nào được gửi, máy khách gửi PINGREQ và broker trả PINGRESP. Nếu broker không nhận được gì trong 1,5 lần khoảng keep-alive, nó coi máy khách đã chết và đóng kết nối. Ngược lại, máy khách không nhận được PINGRESP thì biết kết nối có vấn đề và chủ động kết nối lại. Cơ chế này cần thiết vì TCP có thể "chết lặng lẽ" khi mất Wi-Fi: không bên nào nhận được tín hiệu đóng kết nối.

**Di chúc** (*Last Will and Testament*). Khi kết nối, máy khách có thể gửi kèm một thông điệp di chúc, ví dụ chủ đề `kitchen/status` với nội dung `offline`. Nếu máy khách **mất kết nối bất thường** (mất điện, mất Wi-Fi, hết thời gian keep-alive), broker sẽ phát thông điệp này thay cho nó. Nếu máy khách ngắt kết nối chủ động bằng gói DISCONNECT, di chúc không được phát. Dự án hiện chưa dùng di chúc (xem mục 10.5.3).

### 10.2.8. Bảo mật trong MQTT

Bản thân giao thức MQTT không mã hóa và không xác thực nội dung. Bảo mật được xây từ nhiều lớp độc lập:

| Lớp | Cơ chế | Chống được |
|---|---|---|
| Vận chuyển | TLS (cổng 8883) | Nghe lén, sửa gói tin trên đường truyền |
| Xác thực máy khách | Tên đăng nhập, mật khẩu trong gói CONNECT; hoặc chứng chỉ máy khách | Thiết bị lạ kết nối vào broker |
| Phân quyền | Danh sách kiểm soát truy cập (ACL) trên broker: ai được đọc, ai được ghi chủ đề nào | Thiết bị bị chiếm quyền phát thông điệp bừa bãi |
| Nội dung | Chữ ký HMAC trên từng thông điệp (tầng ứng dụng) | Giả mạo, sửa nội dung, kể cả khi broker không đáng tin |
| Chống phát lại | Dấu thời gian kèm tuổi tối đa | Kẻ xấu ghi lại một thông điệp hợp lệ rồi phát lại sau |

Rủi ro lớn nhất xuất hiện khi dùng **broker công cộng** như `broker.emqx.io`: bất kỳ ai trên Internet cũng kết nối được, đọc được và phát được vào mọi chủ đề. Nếu thông điệp không được ký, ai biết tên chủ đề `payment/success` đều có thể phát một tín hiệu giả "đơn đã thanh toán" và lấy hàng miễn phí. Đây là lý do dự án ký mọi thông điệp.

### 10.2.9. Chữ ký HMAC

**HMAC** (*Hash-based Message Authentication Code*, RFC 2104) là cơ chế tạo "chữ ký" cho thông điệp từ một **hàm băm** và một **khóa bí mật** mà chỉ các bên hợp lệ biết:

$$\text{HMAC}(K, m) = H\big((K' \oplus \text{opad}) \,\|\, H((K' \oplus \text{ipad}) \,\|\, m)\big)$$

Trong đó $H$ là hàm băm SHA-256, $K'$ là khóa đã chuẩn hóa về độ dài khối 64 byte, `ipad` là các byte `0x36` lặp lại, `opad` là các byte `0x5c` lặp lại, và $\|$ là phép nối chuỗi.

Cách dùng như sau. Bên gửi tính HMAC của nội dung rồi gửi kèm. Bên nhận **tính lại** HMAC trên nội dung nhận được bằng cùng khóa và so với chữ ký đi kèm. Hai giá trị khớp nhau chứng minh hai điều:
- **Xác thực nguồn gốc:** chỉ ai có khóa mới tạo được chữ ký đúng.
- **Toàn vẹn:** chỉ cần thay đổi một bit trong nội dung, chữ ký sẽ khác hoàn toàn.

Có ba điểm quan trọng cần nhớ:
- **Khóa không bao giờ đi trên mạng.** Thông điệp chỉ mang chữ ký, nên kẻ nghe lén không lấy được khóa.
- **Không thay được bằng `SHA256(khóa ‖ nội dung)`.** Cách ghép đơn giản này mắc lỗi **tấn công mở rộng độ dài** (*length extension*): kẻ tấn công có thể nối thêm dữ liệu vào thông điệp và tự tính được chữ ký hợp lệ mà không cần biết khóa. Cấu trúc băm hai lần lồng nhau của HMAC loại bỏ lỗ hổng này.
- **Phải so sánh chữ ký trong thời gian hằng**, giống như so sánh khóa API ở chương Flask, để chống tấn công đo thời gian.

HMAC không mã hóa nội dung: người nghe lén vẫn đọc được thông điệp, chỉ không giả mạo hay sửa được. Muốn giữ bí mật nội dung thì cần thêm TLS.

```python
import hmac, hashlib
chu_ky = hmac.new(b"khoa-bi-mat", b'{"order_code":"DH12345678"}', hashlib.sha256).hexdigest()
# Kết quả: chuỗi 64 ký tự hex (32 byte)
```

### 10.2.10. Lập trình MQTT với paho-mqtt (Python)

Thư viện paho-mqtt của dự án Eclipse là thư viện MQTT phổ biến nhất cho Python. Ví dụ tối giản dưới đây thể hiện mẫu sử dụng chuẩn:

```python
import paho.mqtt.client as mqtt

def on_connect(client, userdata, flags, reason_code, properties):
    if reason_code == 0:
        client.subscribe("demo/#", qos=1)          # đăng ký TRONG on_connect

def on_message(client, userdata, msg):
    print(msg.topic, msg.payload.decode("utf-8"))  # chạy trên LUỒNG MẠNG của paho

client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
client.on_connect = on_connect
client.on_message = on_message
client.connect_async("localhost", 1883, keepalive=60)
client.loop_start()                                # tạo luồng nền xử lý mạng
client.publish("demo/hello", "xin chào", qos=1, retain=False)
```

Đoạn mã này có bốn điểm cần hiểu rõ.

*Thứ nhất, `CallbackAPIVersion.VERSION2`.* Từ paho-mqtt 2.0, khi tạo máy khách phải chỉ rõ phiên bản giao diện callback. Phiên bản 2 truyền `reason_code` và `properties` theo kiểu MQTT 5, thống nhất cho mọi phiên bản giao thức.

*Thứ hai, `loop_start()`.* Hàm này tạo một **luồng nền** đảm nhận mọi việc mạng: đọc và ghi socket, gửi PINGREQ, gọi callback, và **tự kết nối lại** khi mất kết nối. Hệ quả là các callback như `on_message` **không chạy trên luồng chính**. Với ứng dụng có giao diện, điều này đòi hỏi phải chuyển dữ liệu về luồng giao diện một cách an toàn.

*Thứ ba, đăng ký trong `on_connect`.* Vì dùng phiên sạch, mỗi lần kết nối lại thì mọi đăng ký cũ đều mất. Đặt lệnh `subscribe` trong `on_connect` bảo đảm đăng ký được thực hiện lại tự động sau mỗi lần kết nối.

*Thứ tư, `publish()` không chờ gửi xong.* Hàm chỉ đưa thông điệp vào hàng đợi rồi trả về ngay một đối tượng `MQTTMessageInfo`:
- trường `rc` cho biết việc **đưa vào hàng đợi** có thành công không;
- muốn chờ broker xác nhận thật sự thì gọi `wait_for_publish()`, sau đó kiểm tra `is_published()`.

### 10.2.11. Lập trình MQTT với PubSubClient (ESP32)

Trên ESP32, dự án dùng thư viện PubSubClient của Nick O'Leary. Mẫu sử dụng khác với paho ở chỗ không có luồng nền: lập trình viên phải tự gọi hàm xử lý mạng thường xuyên.

```cpp
#include <WiFi.h>
#include <PubSubClient.h>

WiFiClient net;
PubSubClient mqtt(net);

void onMessage(char *topic, byte *payload, unsigned int length) {
  // payload KHÔNG kết thúc bằng '\0' -> luôn dùng length khi đọc
}

void setup() {
  WiFi.begin("ten-wifi", "mat-khau");
  mqtt.setServer("192.168.1.10", 1883);
  mqtt.setCallback(onMessage);
  mqtt.setBufferSize(4096);                  // mặc định chỉ 256 byte
}

void loop() {
  if (!mqtt.connected() && mqtt.connect("esp32-demo")) {
    mqtt.subscribe("demo/#", 1);             // tham số thứ hai: QoS đăng ký (0 hoặc 1)
  }
  mqtt.loop();                               // PHẢI gọi liên tục
}
```

Những điểm cần lưu ý với PubSubClient:
- **Hàm `loop()` phải được gọi liên tục.** Đây là nơi thư viện đọc socket, gọi callback và gửi PINGREQ. Nếu không gọi đủ thường xuyên, broker sẽ coi thiết bị đã chết.
- **Bộ đệm mặc định chỉ 256 byte.** Thông điệp lớn hơn bị bỏ đi **âm thầm**, không báo lỗi, nên phải tăng bằng `setBufferSize()`.
- **Nội dung không kết thúc bằng ký tự `'\0'`.** Phải luôn đọc theo tham số `length`, không dùng các hàm chuỗi C như `strlen`.
- **Về QoS:** thư viện chỉ phát được QoS 0, đăng ký được QoS 0 hoặc 1, và không hỗ trợ QoS 2. Hàm `subscribe(topic)` không có tham số thứ hai sẽ đăng ký với **QoS 0**.

---

## 10.3. Lý do lựa chọn

Để truyền sự kiện giữa các khối của hệ thống, có thể cân nhắc nhiều giải pháp. Bảng dưới so sánh MQTT với bốn giải pháp tiêu biểu.

| Tiêu chí | MQTT | HTTP hỏi định kỳ | WebSocket | AMQP (RabbitMQ) | CoAP |
|---|---|---|---|---|---|
| Mô hình | Xuất bản – đăng ký qua broker | Yêu cầu – phản hồi | Kênh hai chiều điểm – điểm | Hàng đợi, định tuyến phức tạp | Yêu cầu – phản hồi trên UDP |
| Độ trễ khi có sự kiện | Gần như tức thì | Phụ thuộc chu kỳ hỏi | Tức thì | Tức thì | Tức thì (chế độ observe) |
| Một thông điệp đến nhiều bên | Có sẵn | Không | Phải tự lập trình | Có | Hạn chế |
| Chi phí trên vi điều khiển | Thấp | Trung bình (mở kết nối mỗi lần) | Trung bình | Cao | Thấp |
| Cần máy chủ trung gian | Broker nhẹ | Không | Không | Broker nặng | Không |
| Phản hồi kết quả | Không có sẵn | Có (mã trạng thái) | Tự định nghĩa | Có | Có |

Dự án chọn **MQTT làm kênh đẩy sự kiện** vì năm lý do.

*Thứ nhất, yêu cầu thời gian thực.* Khi khách thanh toán xong, đơn phải xuất hiện trên màn hình bếp ngay lập tức. Nếu màn bếp hỏi API định kỳ, độ trễ phụ thuộc chu kỳ hỏi: hỏi 30 giây một lần thì có thể trễ tới 30 giây, còn hỏi mỗi giây thì tốn tài nguyên cho hàng nghìn lần hỏi không có gì mới. MQTT đẩy đơn ngay khi có, với chi phí gần như bằng không lúc rảnh.

*Thứ hai, tách rời các thành phần.* Kiosk không cần biết địa chỉ màn bếp, webhook không cần biết địa chỉ kiosk. Hệ thống dễ mở rộng: thêm màn bếp, thêm màn hình gọi số hay thêm dịch vụ thống kê chỉ cần đăng ký chủ đề tương ứng.

*Thứ ba, phù hợp với ESP32.* Giao thức gọn nhẹ, có thư viện PubSubClient ổn định và phổ biến.

*Thứ tư, thông điệp lưu giữ giải quyết gọn bài toán đồng bộ trạng thái.* Màn bếp vừa bật cần ngay bảng tồn kho hiện tại; broker trao bản lưu giữ ngay khi màn bếp đăng ký.

*Thứ năm, kết nối hai tiến trình độc lập.* Webhook và ứng dụng kiosk là hai chương trình riêng biệt. Ứng dụng kiosk là ứng dụng giao diện, không mở cổng HTTP để nhận lệnh. Broker là cách đơn giản và an toàn nhất để webhook "báo tin" cho kiosk mà không phải mở thêm cổng mạng nào trên kiosk.

**Vì sao vẫn giữ HTTP song song.** MQTT không cho bên gửi biết kết quả xử lý, và thông điệp có thể bị lỡ khi thiết bị mất kết nối. Vì vậy:
- Thao tác cần biết thành công hay thất bại, như bếp báo hoàn thành đơn và cần nhận mã 409 khi đơn đã được người khác hoàn thành, dùng HTTP.
- Việc khôi phục trạng thái sau khi mất kết nối cũng dùng HTTP: màn bếp tải lại đơn chờ và tồn kho từ API khi khởi động và mỗi 30 giây.

Thiết kế cuối cùng là **đẩy bằng MQTT kết hợp kéo bằng HTTP**, trong đó cơ sở dữ liệu là **nguồn sự thật duy nhất**. MQTT bảo đảm tốc độ, HTTP bảo đảm độ tin cậy. Đây là một mẫu thiết kế phổ biến trong các hệ thống IoT thực tế.

---

## 10.4. Ứng dụng và code trong dự án

### 10.4.1. Tổng quan luồng thông điệp

Sơ đồ dưới mô tả toàn bộ hành trình của các thông điệp MQTT trong hệ thống, từ lúc tiền về tài khoản đến lúc đơn hiện trên màn hình bếp:

```
 SePay ─HTTPS─► webhook_server ─PUBLISH payment/success (QoS 1, ký HMAC)───┐
                                                                            ▼
                                                                     ┌─ BROKER ─┐
 kiosk_app (bridge.py) ◄──────── SUBSCRIBE payment/success (QoS 1) ──┤          │
   │ xác minh chữ ký và độ mới → chốt thanh toán → ghi cơ sở dữ liệu  │          │
   ├─PUBLISH kitchen/inventory (QoS 1, lưu giữ, ký) ─────────────────►│          │
   └─PUBLISH kitchen/new_order (QoS 1, không lưu giữ, ký) ───────────►│          │
 api_server ─PUBLISH kitchen/inventory (lưu giữ, ký) khi sửa kho ────►│          │
                                                                     └────┬─────┘
 KDS (ESP32) ◄──── SUBSCRIBE kitchen/new_order, kitchen/inventory ────────┘
   │ xác minh chữ ký → hàng đợi FreeRTOS → vẽ thẻ đơn, bảng kho
   └── HTTP mỗi 30 giây: GET /api/orders/pending, /api/inventory (bù thông điệp lỡ)
```

Hệ thống dùng ba chủ đề:

| Chủ đề | Bên phát | Bên nhận | QoS phát | Lưu giữ | Sự kiện | Nội dung chính |
|---|---|---|---|---|---|---|
| `payment/success` | Webhook | Kiosk | 1 | Không | `payment.success` | `order_code`, `amount`, `status: "PAID"` |
| `kitchen/new_order` | Kiosk | KDS | 1 | Không | `order.created` | `order_id`, `created_at`, danh sách món kèm tùy chọn |
| `kitchen/inventory` | Kiosk, API | KDS | 1 (kiosk), 0 (API) | Có | `inventory.snapshot` | Danh sách nguyên liệu: `id`, `name`, `unit`, `quantity`, `low` |

Cấu hình MQTT đọc từ biến môi trường trong tệp `.env` (Python) và tệp `secrets.h` (ESP32):

| Biến | Ý nghĩa |
|---|---|
| `MQTT_BROKER`, `MQTT_PORT` | Địa chỉ broker; mặc định `broker.emqx.io`, cổng 1883 |
| `MQTT_SHARED_SECRET` | Khóa HMAC dùng chung; trên ESP32 là `MQTT_SECRET`, hai giá trị phải trùng nhau |
| `MQTT_<VAI_TRÒ>_USERNAME`, `_PASSWORD` | Tài khoản riêng cho từng dịch vụ: `WEBHOOK`, `KIOSK`, `API` |
| `MQTT_TLS`, `MQTT_CA_CERT` | Bật TLS và chỉ định chứng chỉ gốc |

### 10.4.2. Phong bì thông điệp có chữ ký

Mọi thông điệp MQTT trong dự án đều được bọc trong một **phong bì** (*envelope*) phiên bản 2 có cấu trúc như sau:

```json
{"v":2,"sig":"3f9a…(64 ký tự hex)","payload":"{\"event\":\"payment.success\",\"order_code\":\"DH12345678\",\"amount\":70000,\"status\":\"PAID\",\"ts\":1794321000}"}
```

Phong bì có ba trường:
- `v`: số phiên bản định dạng.
- `payload`: nội dung thật, **dưới dạng một chuỗi JSON** chứ không phải đối tượng JSON lồng nhau. Nội dung có thêm trường `ts`, là thời điểm ký tính bằng giây Unix.
- `sig`: HMAC-SHA256 của chính chuỗi `payload` đó.

Mô-đun `backend/mqtt_security.py` cài đặt việc ký và xác minh:

```python
# Smart_Kiosk_Backup/backend/mqtt_security.py
ENVELOPE_VERSION = 2
DEFAULT_MAX_AGE_SECONDS = 120


class MqttMessageRejected(ValueError):
    """Thông điệp MQTT không hợp lệ hoặc không xác thực được."""


def _secret_bytes(secret):
    if not secret or not str(secret).strip():
        raise MqttMessageRejected("mqtt_secret_not_configured")    # fail-closed
    return str(secret).encode("utf-8")


def compute_signature(secret, payload_text):
    return hmac.new(_secret_bytes(secret), payload_text.encode("utf-8"), hashlib.sha256).hexdigest()


def sign_message(body, secret, *, now=None):
    """Trả về chuỗi JSON đã ký để publish. Không bao giờ chèn khóa vào nội dung."""
    content = {key: value for key, value in body.items() if key != "secret"}
    content["ts"] = int(time.time() if now is None else now)
    payload_text = json.dumps(content, ensure_ascii=False, separators=(",", ":"))
    signature = compute_signature(secret, payload_text)
    return json.dumps({"v": ENVELOPE_VERSION, "sig": signature, "payload": payload_text},
                      ensure_ascii=False, separators=(",", ":"))


def verify_message(raw, secret, *, max_age_seconds=DEFAULT_MAX_AGE_SECONDS, now=None):
    """Xác minh chữ ký và độ mới; trả về nội dung hoặc ném MqttMessageRejected."""
    secret_bytes = _secret_bytes(secret)
    try:
        text = raw.decode("utf-8") if isinstance(raw, (bytes, bytearray)) else str(raw)
        envelope = json.loads(text)
    except (UnicodeDecodeError, json.JSONDecodeError) as error:
        raise MqttMessageRejected("invalid_envelope") from error

    if not isinstance(envelope, dict) or envelope.get("v") != ENVELOPE_VERSION:
        raise MqttMessageRejected("unsupported_envelope")
    payload_text = envelope.get("payload")
    signature = envelope.get("sig")
    if not isinstance(payload_text, str) or not isinstance(signature, str):
        raise MqttMessageRejected("invalid_envelope")

    expected = hmac.new(secret_bytes, payload_text.encode("utf-8"), hashlib.sha256).hexdigest()
    if not hmac.compare_digest(expected, signature.strip().lower()):
        raise MqttMessageRejected("bad_signature")

    content = json.loads(payload_text)                 # chỉ phân tích SAU khi chữ ký hợp lệ
    if not isinstance(content, dict):
        raise MqttMessageRejected("invalid_payload")

    if max_age_seconds is not None:
        timestamp = float(content["ts"])
        current = time.time() if now is None else now
        # Cho phép đồng hồ lệch 30 giây về phía tương lai.
        if timestamp > current + 30 or current - timestamp > max_age_seconds:
            raise MqttMessageRejected("stale_message")
    return content
```

*(Trích rút gọn: bản đầy đủ có thêm khối `try` quanh `json.loads(payload_text)` và quanh việc đọc `ts`, trả về các mã lỗi `invalid_payload` và `missing_timestamp`.)*

Thiết kế này có năm quyết định quan trọng.

**Ký trên chuỗi chứ không ký trên đối tượng.** Cùng một dữ liệu có thể được chuyển thành nhiều chuỗi JSON khác nhau, tùy thứ tự khóa, khoảng trắng hay cách viết ký tự Unicode. Python có thể sinh `{"a":1,"b":2}`, còn thư viện ArduinoJson trên ESP32 có thể sinh thứ tự khác. Nếu bên nhận phân tích JSON rồi tự dựng lại chuỗi để băm, chữ ký sẽ lệch dù dữ liệu không hề thay đổi. Giải pháp là gửi nội dung **dưới dạng chuỗi** trong trường `payload`, và bên nhận băm **đúng chuỗi nhận được**. Nhờ vậy ESP32 không cần sắp xếp JSON, chỉ cần băm một chuỗi byte.

**Chống phát lại bằng dấu thời gian.** Chữ ký chứng minh thông điệp là thật, nhưng không ngăn được việc ai đó ghi lại một thông điệp thật rồi phát lại sau. Ví dụ, kẻ xấu ghi lại tín hiệu "đơn DH12345678 đã thanh toán", rồi hôm sau đặt một đơn có cùng mã và phát lại tín hiệu cũ. Trường `ts` nằm **bên trong** phần được ký nên không sửa được. Bên nhận từ chối thông điệp cũ hơn 120 giây. Khoảng dung sai 30 giây về phía tương lai dành cho trường hợp đồng hồ giữa các máy lệch nhau một chút.

**Mặc định từ chối.** Hàm `_secret_bytes` ném ngoại lệ khi khóa rỗng hoặc chỉ có khoảng trắng. Thiếu khóa thì không ký được và cũng không xác minh được, nên quên cấu hình không bao giờ dẫn tới việc chấp nhận thông điệp không ký.

**Xác minh trước, phân tích sau.** Nội dung bên trong chỉ được phân tích JSON **sau khi** chữ ký đã hợp lệ. Dữ liệu chưa xác thực không bao giờ được đưa vào xử lý.

**Mã lỗi có tên.** Các lý do từ chối như `bad_signature`, `stale_message` hay `unsupported_envelope` được ghi rõ, nên khi đọc nhật ký biết ngay vì sao một thông điệp bị loại.

Ngoài ra, `separators=(",", ":")` cho JSON gọn nhất, không có khoảng trắng thừa. `ensure_ascii=False` giữ nguyên chữ tiếng Việt thay vì đổi thành dạng `\uXXXX`, nên thông điệp ngắn hơn và dễ đọc khi gỡ lỗi. Dòng lọc bỏ khóa `secret` là lớp phòng vệ cuối, đề phòng đoạn mã cũ nào đó còn chèn khóa vào nội dung như định dạng phiên bản 1 trước đây (mục 10.5.2).

Cũng trong mô-đun này, hàm `configure_client` gắn tài khoản và TLS cho máy khách paho theo vai trò của từng dịch vụ:

```python
def _env(name, role):
    if role:
        value = os.environ.get(f"MQTT_{role.upper()}_{name}", "")
        if value:
            return value
    return os.environ.get(f"MQTT_{name}", "")


def configure_client(client, role=None):
    """Gắn tài khoản và TLS (nếu có cấu hình) vào một paho.mqtt.client.Client."""
    username = _env("USERNAME", role)
    password = _env("PASSWORD", role)
    if username:
        client.username_pw_set(username, password or None)
    if os.environ.get("MQTT_TLS", "").strip().lower() in ("1", "true", "yes", "on"):
        client.tls_set(ca_certs=os.environ.get("MQTT_CA_CERT", "").strip() or None)
```

Mỗi dịch vụ gọi hàm với vai trò của mình (`WEBHOOK`, `KIOSK`, `API`) và nhận tài khoản riêng, khớp với bảng phân quyền trên broker (mục 10.4.7).

### 10.4.3. Webhook phát tín hiệu thanh toán

Khi SePay báo có tiền vào và nội dung chuyển khoản chứa mã đơn, webhook phát tín hiệu lên chủ đề `payment/success`:

```python
# Smart_Kiosk_Backup/servers/webhook_server.py
def send_mqtt_signal(order_code, amount):
    """Phát tín hiệu thanh toán đã KÝ HMAC; thiếu MQTT_SHARED_SECRET thì không phát."""
    if not MQTT_SHARED_SECRET:
        return False
    try:
        client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
        configure_client(client, role="WEBHOOK")
        client.connect(MQTT_BROKER, MQTT_PORT, 60)
        client.loop_start()
        message = sign_message(
            {"event": "payment.success", "order_code": order_code,
             "amount": amount, "status": "PAID"},
            MQTT_SHARED_SECRET,
        )
        info = client.publish(MQTT_TOPIC, message, qos=1)
        info.wait_for_publish(timeout=5)          # chờ broker xác nhận PUBACK
        client.loop_stop()
        client.disconnect()
        return True
    except Exception as e:
        print(f" ---Lỗi kết nối MQTT: {e}")
        return False
```

Webhook dùng mô hình **"kết nối – phát – ngắt"** cho mỗi giao dịch. Mô hình này đơn giản và không phải duy trì kết nối lâu dài, nhưng tốn thêm vài trăm mili giây cho mỗi lần tiền về. Với lưu lượng một cửa hàng, vài giao dịch mỗi phút, đánh đổi này chấp nhận được. Phương án giữ kết nối thường trực đã được cân nhắc và quyết định chưa áp dụng.

Hàm `wait_for_publish` chờ tối đa 5 giây để broker gửi lại PUBACK, vì thông điệp dùng QoS 1. Nếu `send_mqtt_signal` trả về `False`, webhook trả mã HTTP 502 cho SePay để báo lỗi nằm ở dịch vụ phía sau. Mục 10.5.3 chỉ ra một điểm còn thiếu của đoạn mã này.

### 10.4.4. Kiosk nhận tín hiệu thanh toán

Ứng dụng kiosk khởi tạo máy khách MQTT khi khởi động và đăng ký chủ đề thanh toán:

```python
# Smart_Kiosk_Backup/kiosk_app/bridge.py (định dạng lại cho gọn)
def setup_mqtt(self):
    try:
        self.mqtt_client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
    except AttributeError:
        self.mqtt_client = mqtt.Client()                 # tương thích paho-mqtt 1.x
    self.mqtt_client.on_connect = self._on_mqtt_connect
    self.mqtt_client.on_disconnect = self._on_mqtt_disconnect
    self.mqtt_client.on_message = self._on_mqtt_message
    configure_client(self.mqtt_client, role="KIOSK")
    try:
        self.mqtt_client.connect_async(MQTT_BROKER, MQTT_PORT, 60)
        self.mqtt_client.loop_start()
    except Exception as error:
        print(f"---> [LỖI MQTT] Không thể khởi tạo kết nối broker: {error}")

def _on_mqtt_connect(self, client, userdata, flags, reason_code, properties=None):
    if int(reason_code) == 0:
        client.subscribe(MQTT_PAYMENT_TOPIC, qos=1)      # đăng ký lại sau MỖI lần kết nối
    else:
        print(f"---> [MQTT] Kết nối broker thất bại. reason_code={reason_code}")
```

Kiosk dùng `connect_async` thay cho `connect`. Với `connect`, chương trình bị chặn cho đến khi kết nối xong. Nếu lúc khởi động broker chậm hoặc chưa có mạng, giao diện kiosk sẽ đứng hình. `connect_async` đưa việc kết nối vào luồng nền do `loop_start` tạo ra, nên giao diện hiện lên ngay. Luồng nền cũng tự kết nối lại khi mất mạng, và mỗi lần kết nối lại thì `_on_mqtt_connect` đăng ký lại chủ đề.

Khi có thông điệp, callback `_on_mqtt_message` kiểm tra nhiều lớp trước khi chốt đơn:

```python
def _on_mqtt_message(self, client, userdata, msg):
    if msg.topic != MQTT_PAYMENT_TOPIC:
        return
    try:
        try:
            payload = verify_message(msg.payload, MQTT_SHARED_SECRET, max_age_seconds=120)
        except MqttMessageRejected as rejected:
            print(f"---> [BẢO MẬT] Bỏ qua tín hiệu thanh toán MQTT: {rejected}")
            return

        order_code = str(payload.get("order_code", "")).strip()
        status = str(payload.get("status", "")).strip().upper()
        amount = payload.get("amount", None)
        if status != "PAID" or not order_code or amount is None:
            return                                         # thiếu dữ liệu: bỏ qua

        self._finalize_payment(order_code, amount, source="webhook")
    except Exception as error:
        print(f"---> [LỖI MQTT] Không thể xử lý tín hiệu thanh toán: {error}")
```

Tín hiệu phải qua bốn lớp kiểm tra mới được chấp nhận:
1. Chữ ký hợp lệ.
2. Thông điệp không quá 120 giây.
3. Trạng thái là `PAID`, có đủ mã đơn và số tiền.
4. Trong `_finalize_payment`, mã đơn và số tiền còn được đối chiếu với đơn đang chờ thanh toán trên kiosk.

Callback này chạy trên **luồng mạng của paho**, không phải luồng giao diện Qt, nên phải chú ý về đồng thời:
- Hàm `_finalize_payment` dùng khóa `threading.Lock`, phòng trường hợp tín hiệu từ webhook và kết quả tra cứu định kỳ của kiosk đến cùng lúc.
- Mọi cập nhật giao diện được gửi qua **signal của Qt**. Signal tự chuyển lời gọi về luồng giao diện, nên không bao giờ thao tác giao diện từ luồng lạ.

### 10.4.5. Kiosk phát đơn và tồn kho xuống bếp

Sau khi chốt thanh toán và lưu đơn vào cơ sở dữ liệu thành công, kiosk phát hai thông điệp theo thứ tự: bảng tồn kho mới, rồi đơn mới.

```python
# Smart_Kiosk_Backup/kiosk_app/bridge.py – trong hàm chốt đơn
print(f"---> [DATABASE] Đã lưu đơn {order_code} thành công.")
self._publish_inventory_snapshot()                                   # 3. tồn kho mới
self._publish_kitchen_order(order_code=order_code,                   # 4. đơn mới
                            order_items=order_items, source=source)
```

Hàm phát đơn mới (rút gọn phần dựng danh sách món):

```python
def _publish_kitchen_order(self, order_code, order_items, source):
    if self.mqtt_client is None:
        return False
    try:
        kitchen_payload = {
            "version": 1,
            "event": "order.created",
            "order_id": order_code,
            "created_at": datetime.now().astimezone().isoformat(timespec="seconds"),
            "items": [{
                "product_id": item.get("product_id", 0),
                "name": item.get("base_name",
                                 item.get("name", "").split(" (", 1)[0].strip()),
                "quantity": item.get("qty", 1),
                "options": {"size": item.get("size", ""), "ice": item.get("ice", "")},
                "note": item.get("note", ""),
            } for item in order_items],
        }
        if not MQTT_SHARED_SECRET:
            print(f"---> [BẢO MẬT] Chưa cấu hình MQTT_SHARED_SECRET: không phát đơn {order_code}. "
                  f"KDS sẽ lấy đơn qua API pending.")
            return False
        publish_info = self.mqtt_client.publish(
            MQTT_KITCHEN_TOPIC, sign_message(kitchen_payload, MQTT_SHARED_SECRET),
            qos=1, retain=False)                           # sự kiện: KHÔNG lưu giữ
        if publish_info.rc != mqtt.MQTT_ERR_SUCCESS:
            print(f"---> [MQTT] Gửi đơn {order_code} thất bại. rc={publish_info.rc}. "
                  f"KDS sẽ phục hồi đơn qua API pending.")
            return False
        return True
    except Exception as error:
        print(f"---> [CẢNH BÁO] Lỗi gửi đơn {order_code} xuống KDS: {error}")
        return False
```

Hàm `_publish_inventory_snapshot` có cấu trúc tương tự. Điểm khác: nội dung là toàn bộ bảng nguyên liệu kèm cờ `low` (sắp hết), sự kiện là `inventory.snapshot`, và được phát với `qos=1, retain=True`.

Đoạn mã này thể hiện bốn nguyên tắc thiết kế.

**Phát sau khi ghi.** Đơn chỉ được phát **sau khi** cơ sở dữ liệu đã lưu thành công. Màn bếp vì vậy không bao giờ hiện một "đơn ma" không có trong cơ sở dữ liệu. Nếu làm ngược lại (phát trước, ghi sau) mà việc ghi thất bại, bếp sẽ làm một đơn không tồn tại.

**Lưu giữ đúng chỗ.** Tồn kho là trạng thái nên được lưu giữ; đơn là sự kiện nên không lưu giữ (mục 10.2.6).

**Sự cố MQTT không làm mất đơn.** Khi phát thất bại, đơn vẫn nằm an toàn trong cơ sở dữ liệu, và màn bếp sẽ lấy lại qua `GET /api/orders/pending` ở lần đồng bộ kế tiếp. Nhật ký ghi rõ hướng phục hồi, nên người vận hành biết hệ thống vẫn ổn.

**Hiểu đúng giá trị trả về.** `publish_info.rc` chỉ cho biết thông điệp đã vào hàng đợi gửi của paho, chưa chắc đã đến broker. Mã lỗi thường gặp là `MQTT_ERR_NO_CONN` khi đang mất kết nối.

Ngoài ra, tên món được rút gọn thành tên gốc. Ví dụ "Trà đào (Size L)" thành "Trà đào", còn kích cỡ và lượng đá đi trong trường `options` riêng, để màn bếp hiển thị gọn trên màn hình nhỏ.

### 10.4.6. Màn hình bếp ESP32 nhận và xác minh thông điệp

Firmware màn hình bếp chạy trên FreeRTOS với hai tác vụ chính:
- **tác vụ mạng** (`taskNetwork`) chạy trên nhân 0, lo Wi-Fi, MQTT và HTTP;
- **tác vụ giao diện** chạy trên nhân 1, lo vẽ màn hình bằng thư viện LVGL.

**Duy trì kết nối.** Tác vụ mạng cấu hình máy khách MQTT một lần rồi chạy vòng lặp vô hạn:

```cpp
// kds_firmware.ino (trích)
const char *MQTT_TOPIC_ORDER = "kitchen/new_order";
const char *MQTT_TOPIC_INV = "kitchen/inventory";
constexpr uint32_t MQTT_RECONNECT_MS = 3000;

static void taskNetwork(void *) {
  mqttClient.setServer(MQTT_BROKER_HOST, MQTT_BROKER_PORT);
  mqttClient.setCallback(mqttCallback);
  mqttClient.setBufferSize(8192);              // mặc định 256 byte: không đủ cho bảng tồn kho
  for (;;) {
    maintainWiFi();
    maintainMqtt();
    if (mqttClient.connected()) mqttClient.loop();
    // ... đồng bộ HTTP lúc khởi động và mỗi 30 giây (PERIODIC_SYNC_MS) ...
    processOrderEventQueue();
    processDoneQueue();
    vTaskDelay(pdMS_TO_TICKS(5));
  }
}

static void maintainMqtt() {
  static uint32_t lastAttempt = 0;
  if (WiFi.status() != WL_CONNECTED || mqttClient.connected()) return;
  if (millis() - lastAttempt < MQTT_RECONNECT_MS) return;     // thử lại mỗi 3 giây
  lastAttempt = millis();
  String clientId = "ESP32_Kitchen_" + String((uint32_t)ESP.getEfuseMac(), HEX);
  bool connected = (MQTT_USERNAME && MQTT_USERNAME[0])
    ? mqttClient.connect(clientId.c_str(), MQTT_USERNAME, MQTT_PASSWORD)
    : mqttClient.connect(clientId.c_str());
  if (connected) {
    mqttClient.subscribe(MQTT_TOPIC_ORDER);
    mqttClient.subscribe(MQTT_TOPIC_INV);
  } else {
    Serial.printf("[MQTT] Connect failed state=%d\n", mqttClient.state());
  }
}
```

Những chi tiết đáng chú ý:
- **`client_id` duy nhất.** Mã định danh được ghép từ địa chỉ MAC của chip. Broker không cho hai máy khách trùng `client_id`: máy kết nối sau sẽ đá máy trước ra. Nếu hai màn bếp dùng chung một mã cố định, chúng sẽ liên tục đá nhau.
- **Thử lại có giãn cách.** Khi mất kết nối, firmware thử lại 3 giây một lần chứ không thử liên tục, tránh làm nghẽn mạng và broker.
- **Đăng ký lại sau mỗi lần kết nối,** vì PubSubClient dùng phiên sạch. Ngay khi đăng ký `kitchen/inventory`, broker gửi bản tồn kho được lưu giữ, nên màn bếp có dữ liệu kho mà không phải chờ.
- **`setBufferSize(8192)`.** Bảng tồn kho vài chục nguyên liệu bọc trong phong bì có chữ ký dài hơn rất nhiều so với 256 byte mặc định.

**Xác minh chữ ký trên ESP32.** Firmware dùng thư viện mbedTLS có sẵn trong ESP-IDF để tính HMAC:

```cpp
static bool computeHmacHex(const char *message, size_t length, char out[65]) {
  uint8_t digest[32];
  const mbedtls_md_info_t *info = mbedtls_md_info_from_type(MBEDTLS_MD_SHA256);
  if (!info) return false;
  if (mbedtls_md_hmac(info, reinterpret_cast<const unsigned char *>(MQTT_SECRET), strlen(MQTT_SECRET),
                      reinterpret_cast<const unsigned char *>(message), length, digest) != 0) {
    return false;
  }
  static const char hex[] = "0123456789abcdef";
  for (int i = 0; i < 32; ++i) {
    out[i * 2] = hex[digest[i] >> 4];        // 4 bit cao
    out[i * 2 + 1] = hex[digest[i] & 0x0F];  // 4 bit thấp
  }
  out[64] = '\0';
  return true;
}

// So sánh trong thời gian hằng: không dừng ở ký tự sai đầu tiên.
static bool constantTimeEquals(const char *a, const char *b, size_t length) {
  uint8_t diff = 0;
  for (size_t i = 0; i < length; ++i) diff |= static_cast<uint8_t>(a[i] ^ b[i]);
  return diff == 0;
}

static bool verifySignedMessage(byte *payload, unsigned int length, JsonDocument &doc) {
  if (!MQTT_SECRET || !MQTT_SECRET[0]) return false;                 // fail-closed
  JsonDocument envelope;
  if (deserializeJson(envelope, payload, length)) return false;      // đọc theo length
  if (envelope["v"].as<int>() != 2 || !envelope["payload"].is<const char *>() ||
      !envelope["sig"].is<const char *>()) return false;
  const char *inner = envelope["payload"].as<const char *>();
  const char *signature = envelope["sig"].as<const char *>();
  char expected[65];
  if (strlen(signature) != 64 || !computeHmacHex(inner, strlen(inner), expected) ||
      !constantTimeEquals(expected, signature, 64)) {
    Serial.println("[MQTT] Bad signature");
    return false;
  }
  return !deserializeJson(doc, inner);                               // phân tích SAU khi xác minh
}
```

Hàm `computeHmacHex` đổi 32 byte kết quả thành 64 ký tự hex chữ thường, khớp chính xác với `hexdigest()` của Python. Đây là điều kiện để chữ ký do Python tạo được ESP32 chấp nhận.

Hàm `constantTimeEquals` là phiên bản viết tay của `hmac.compare_digest`. Nó gom kết quả XOR của mọi cặp ký tự bằng phép OR và không dừng sớm, nên thời gian chạy không phụ thuộc vị trí ký tự sai đầu tiên.

Giống phía Python, firmware chỉ phân tích nội dung bên trong **sau khi** chữ ký đã hợp lệ.

**Callback không được vẽ giao diện.** Callback nhận thông điệp chạy bên trong `mqttClient.loop()`, tức là trên tác vụ mạng. Thư viện LVGL lại không an toàn đa luồng, nên callback tuyệt đối không gọi hàm vẽ:

```cpp
static void mqttCallback(char *topic, byte *payload, unsigned int length) {
  JsonDocument doc;
  if (!verifySignedMessage(payload, length, doc)) return;

  if (strcmp(topic, MQTT_TOPIC_ORDER) == 0) {
    if (!doc.is<JsonObjectConst>()) return;
    KdsOrderEvent event{};
    if (!parseOrderObject(doc.as<JsonObjectConst>(), event)) return;
    xQueueSend(orderEventQueue, &event, 0);           // chỉ đẩy vào hàng đợi, không chờ
  } else if (strcmp(topic, MQTT_TOPIC_INV) == 0) {
    if (!doc["items"].is<JsonArrayConst>()) return;
    if (parseInventory(doc["items"].as<JsonArrayConst>()))
      inventoryUiDirty = true;                        // bật cờ; tác vụ giao diện tự vẽ lại
  }
}
```

Callback chỉ làm ba việc: xác minh, phân tích dữ liệu vào cấu trúc kích thước cố định `KdsOrderEvent`, rồi đẩy vào **hàng đợi FreeRTOS**. Hàng đợi sao chép nguyên cấu trúc, nên không có con trỏ trỏ vào vùng nhớ đã giải phóng. Tác vụ giao diện lấy sự kiện ra và vẽ thẻ đơn. Với tồn kho, callback chỉ bật cờ `inventoryUiDirty`, tác vụ giao diện thấy cờ thì vẽ lại bảng kho.

Tham số thời gian chờ bằng 0 trong `xQueueSend` nghĩa là nếu hàng đợi đầy thì bỏ qua ngay, không làm tác vụ mạng bị chặn. Đơn bị bỏ ở đây vẫn được bù qua đồng bộ HTTP.

Hàm `parseOrderObject` dùng chung cho cả thông điệp MQTT `order.created` lẫn dữ liệu HTTP `order.snapshot`, vì API được thiết kế trả cùng định dạng (xem chương Flask, mục 5.4.4). Khi tạo thẻ đơn, giao diện bỏ qua đơn đã hiển thị. Nhờ bước **khử trùng** này, một đơn đến hai lần (do QoS 1 gửi lại, hoặc do đến từ cả MQTT lẫn HTTP) cũng chỉ hiện một thẻ.

### 10.4.7. Broker nội bộ Mosquitto có xác thực và phân quyền

Trong giai đoạn thử nghiệm, hệ thống dùng broker công cộng `broker.emqx.io`. Nhờ chữ ký HMAC, không ai giả mạo được thông điệp. Để triển khai thực tế, dự án đã chuẩn bị sẵn cấu hình cho broker **Mosquitto** chạy trong mạng nội bộ:

```conf
# Smart_Kiosk_Backup/deploy/mosquitto/mosquitto.conf
listener 1883 0.0.0.0                 # chỉ mở trong mạng LAN, KHÔNG chuyển tiếp ra Internet
allow_anonymous false                 # bắt buộc đăng nhập
password_file /etc/mosquitto/conf.d/smart_kiosk/passwd
acl_file /etc/mosquitto/conf.d/smart_kiosk/acl
persistence true                      # giữ thông điệp lưu giữ qua các lần khởi động lại
```

```conf
# Smart_Kiosk_Backup/deploy/mosquitto/acl – quyền tối thiểu cho từng dịch vụ
user webhook
topic write payment/success

user kiosk
topic read payment/success
topic write kitchen/new_order
topic write kitchen/inventory

user api
topic write kitchen/inventory

user kds
topic read kitchen/new_order
topic read kitchen/inventory
```

Bảng phân quyền thể hiện nguyên tắc **đặc quyền tối thiểu**:
- **Webhook** chỉ được phát đúng một chủ đề.
- **Màn bếp** chỉ được đọc, không được phát. Nếu tài khoản của màn bếp bị trích xuất từ firmware, kẻ xấu cũng không thể phát đơn giả hay tín hiệu thanh toán giả.

Kết hợp với chữ ký HMAC, hệ thống có **hai lớp bảo vệ độc lập**: thủng lớp này vẫn còn lớp kia. Dòng `persistence true` cho broker ghi thông điệp lưu giữ xuống đĩa, nên khởi động lại broker vẫn còn bảng tồn kho.

Các lệnh tạo tài khoản và kiểm tra:

```bash
sudo apt install mosquitto mosquitto-clients
sudo mosquitto_passwd -c /etc/mosquitto/conf.d/smart_kiosk/passwd webhook   # -c: tạo tệp mới
sudo mosquitto_passwd    /etc/mosquitto/conf.d/smart_kiosk/passwd kiosk
sudo systemctl restart mosquitto

mosquitto_sub -h <ip-broker> -t 'kitchen/#'                         # không đăng nhập: bị từ chối
mosquitto_sub -h <ip-broker> -u kds -P '<mật khẩu>' -t 'kitchen/#' -v   # màn bếp: đọc được
```

Công cụ `mosquitto_sub` và `mosquitto_pub` cũng rất hữu ích khi gỡ lỗi:
- `mosquitto_sub -t 'kitchen/#' -v` xem mọi thông điệp bếp kèm tên chủ đề.
- `mosquitto_pub -t 'kitchen/inventory' -r -n` xóa bản tồn kho đang được lưu giữ.

### 10.4.8. Kiểm thử chữ ký thông điệp

Tệp `Smart_Kiosk_Backup/tests/test_mqtt_security.py` kiểm thử mô-đun ký và xác minh bằng pytest. Một số ca tiêu biểu:

```python
SECRET = "khóa-thử-nghiệm"
NOW = 1_800_000_000

def _body():
    return {"event": "payment.success", "order_code": "DH12345678", "amount": 70000, "status": "PAID"}

def test_tampered_payload_is_rejected():
    envelope = json.loads(sign_message(_body(), SECRET, now=NOW))
    envelope["payload"] = envelope["payload"].replace("70000", "1")     # sửa số tiền
    with pytest.raises(MqttMessageRejected, match="bad_signature"):
        verify_message(json.dumps(envelope), SECRET, now=NOW)

def test_secret_is_never_put_in_message():
    body = _body()
    body["secret"] = SECRET
    raw = sign_message(body, SECRET, now=NOW)
    assert SECRET not in raw

def test_stale_message_is_rejected():
    raw = sign_message(_body(), SECRET, now=NOW)
    with pytest.raises(MqttMessageRejected, match="stale_message"):
        verify_message(raw, SECRET, max_age_seconds=120, now=NOW + 121)

@pytest.mark.parametrize("secret", ["", "   ", None])
def test_missing_secret_fails_closed(secret):
    with pytest.raises(MqttMessageRejected, match="mqtt_secret_not_configured"):
        sign_message(_body(), secret, now=NOW)
```

Toàn bộ bộ kiểm thử bao phủ các tình huống sau:

| Tình huống | Kết quả mong đợi |
|---|---|
| Ký rồi xác minh | Ra đúng nội dung, có trường `ts` |
| Mã gọi lỡ truyền khóa vào nội dung | Khóa không xuất hiện trong thông điệp |
| Nội dung tiếng Việt, có dấu ngoặc kép | Ký và xác minh không làm hỏng dữ liệu |
| Sửa số tiền từ 70000 thành 1 | Bị từ chối: `bad_signature` |
| Sai khóa | Bị từ chối: `bad_signature` |
| Định dạng cũ không ký | Bị từ chối: `unsupported_envelope` |
| Thông điệp cũ hơn 120 giây, hoặc đến từ tương lai | Bị từ chối: `stale_message` |
| Khóa rỗng, chỉ có khoảng trắng, hoặc `None` | Từ chối ở cả bước ký lẫn xác minh |

Tham số `now` cho phép **cố định thời gian** khi kiểm thử. Đây là kỹ thuật **tiêm phụ thuộc** (*dependency injection*) áp dụng cho đồng hồ: thay vì đọc giờ hệ thống bên trong hàm, ta truyền thời điểm vào từ bên ngoài. Nhờ vậy kiểm thử "thông điệp quá 120 giây" chạy tức thì, không phải chờ 2 phút thật.

---

## 10.5. Kỹ thuật và đánh giá

### 10.5.1. Các kỹ thuật thiết kế đã áp dụng

Phần bảo mật MQTT được xây theo **nhiều lớp độc lập**:
- **chữ ký HMAC-SHA256** trên mọi thông điệp bảo vệ nội dung, kể cả khi broker là công cộng;
- **dấu thời gian nằm trong phần được ký** chống phát lại;
- **so sánh trong thời gian hằng** ở cả Python lẫn C chống tấn công đo thời gian;
- **mặc định từ chối** khi thiếu khóa;
- **tài khoản riêng kèm phân quyền** theo đặc quyền tối thiểu trên broker nội bộ.

Định dạng phong bì **ký trên chuỗi** giải quyết bài toán tương thích giữa Python và ESP32 một cách gọn gàng.

Về độ tin cậy, dự án áp dụng mẫu **đẩy kết hợp kéo**: MQTT cho tốc độ, HTTP cho phục hồi, cơ sở dữ liệu là nguồn sự thật duy nhất. Bốn kỹ thuật bổ trợ:
- thông điệp chỉ được phát **sau khi** cơ sở dữ liệu đã lưu;
- bên nhận **khử trùng** nên chịu được việc QoS 1 gửi lặp;
- **lưu giữ** cho trạng thái, **không lưu giữ** cho sự kiện;
- máy khách tự **đăng ký lại** sau mỗi lần kết nối.

Về kiến trúc phần mềm, callback MQTT **không bao giờ chạm vào giao diện**. Kiosk chuyển về luồng giao diện qua signal Qt; màn bếp chuyển qua hàng đợi FreeRTOS hoặc cờ. Mạng chậm vì thế không làm đứng màn hình, và không xảy ra tranh chấp giữa các luồng. Dữ liệu mang **số phiên bản và tên sự kiện**, sẵn sàng mở rộng mà không làm hỏng thiết bị cũ.

### 10.5.2. Các lỗi đã gặp và cách khắc phục

| Lỗi | Hậu quả | Cách khắc phục |
|---|---|---|
| Định dạng phiên bản 1 chèn khóa bí mật vào nội dung thông điệp để màn bếp so sánh | Trên broker công cộng, ai nghe được một thông điệp là lấy được khóa, rồi giả tín hiệu "đã thanh toán" | Chuyển sang phong bì phiên bản 2 ký HMAC; khóa không bao giờ rời thiết bị; từ chối định dạng cũ |
| Bảng tồn kho không bao giờ đến màn bếp | PubSubClient bỏ âm thầm thông điệp lớn hơn bộ đệm 256 byte | Tăng bộ đệm lên 8192 byte |
| Vẽ giao diện LVGL ngay trong callback MQTT | Màn hình treo hoặc lỗi ngẫu nhiên do hai tác vụ cùng vẽ | Callback chỉ đẩy hàng đợi hoặc bật cờ; tác vụ giao diện vẽ |
| Mất đơn khi màn bếp khởi động lại hoặc mất Wi-Fi | Đơn phát lúc màn bếp ngoại tuyến không bao giờ đến | Thêm đồng bộ HTTP lúc khởi động và mỗi 30 giây |

### 10.5.3. Hạn chế còn tồn tại và hướng cải tiến

**Chặng broker → màn bếp chỉ đạt QoS 0.** Kiosk phát với QoS 1, nhưng màn bếp gọi `subscribe(topic)` không có tham số QoS, nên mặc định là QoS 0. Theo quy tắc lấy giá trị nhỏ nhất, broker giao xuống màn bếp với QoS 0. Trên thực tế hệ thống không mất đơn nhờ đồng bộ HTTP, nhưng một đơn bị lỡ có thể đến trễ tới 30 giây. Cách sửa chỉ là một dòng: `subscribe(MQTT_TOPIC_ORDER, 1)` và `subscribe(MQTT_TOPIC_INV, 1)`.

**Màn bếp chưa kiểm tra tuổi thông điệp.** Phía Python từ chối thông điệp cũ hơn 120 giây, còn màn bếp chỉ kiểm tra chữ ký. Ai ghi lại được một thông điệp đơn hợp lệ có thể phát lại. Ảnh hưởng nhỏ, vì đơn đang hiển thị sẽ bị khử trùng, nhưng một đơn đã hoàn thành có thể hiện lại. Hướng cải tiến:
- đồng bộ giờ cho ESP32 qua NTP bằng `configTime()` rồi kiểm tra trường `ts`; hoặc
- hỏi API xem đơn còn ở trạng thái chờ hay không trước khi tạo thẻ.

**Webhook chưa kiểm tra kết quả chờ xác nhận.** `wait_for_publish(timeout=5)` hết thời gian chờ thì không báo lỗi. Nếu broker chưa xác nhận trong 5 giây, hàm vẫn trả `True` và SePay nhận phản hồi thành công. Nên kiểm tra thêm `info.is_published()` và trả `False` khi chưa được xác nhận. Hậu quả hiện tại được giảm nhẹ nhờ kiosk tự tra cứu giao dịch qua API của SePay mỗi 15 giây.

**Smart Kiosk API không tự phục hồi nếu broker chưa sẵn sàng lúc khởi động.** API dùng `connect()` đồng bộ. Nếu lần kết nối đầu tiên thất bại, `loop_start()` không được gọi, và máy khách sẽ không bao giờ thử lại. API cũng phát tồn kho với QoS 0 và không kiểm tra `rc`. Nên đổi sang `connect_async()` kèm `loop_start()` như phía kiosk, phát với `qos=1` và ghi nhật ký khi lỗi. Màn bếp vẫn tự bù tồn kho qua HTTP nên dữ liệu không sai lâu dài.

**Đang thử nghiệm trên broker công cộng.** Chữ ký HMAC ngăn giả mạo, nhưng **nội dung đơn hàng vẫn đọc được công khai** vì không mã hóa, và hệ thống phụ thuộc một dịch vụ bên ngoài. Khi triển khai thật, nên chuyển sang Mosquitto nội bộ đã chuẩn bị sẵn. Nếu bắt buộc đi qua Internet thì bật TLS.

**Chưa dùng di chúc.** Hệ thống chưa biết khi nào màn bếp mất kết nối. Có thể đặt di chúc trên chủ đề `kitchen/kds/<MAC>/status` với nội dung `offline` (lưu giữ), và phát `online` mỗi khi kết nối thành công, để phần mềm quản trị hiển thị trạng thái màn bếp.

### 10.5.4. Kiến thức trọng tâm

Phần này tóm tắt những ý cốt lõi của chương, dùng để ôn tập và chuẩn bị trả lời trước hội đồng.

**Bản chất của MQTT.** MQTT là giao thức xuất bản – đăng ký qua broker, chạy trên một kết nối TCP lâu dài, gói tin nhỏ nhất chỉ 2 byte, ra đời cho mạng băng thông thấp và chập chờn. Đặc điểm quan trọng nhất là tách rời bên gửi và bên nhận theo không gian, thời gian và đồng bộ. Hạn chế chính là cần broker, không tự bảo vệ nội dung, và không có phản hồi kết quả.

**Các khái niệm cần nắm chắc:**
- **Chủ đề phân cấp,** với hai ký tự đại diện `+` (một cấp) và `#` (mọi cấp phía sau).
- **Ba mức QoS:** QoS 0 có thể mất; QoS 1 không mất nhưng có thể trùng; QoS 2 đúng một lần nhưng chậm. QoS áp dụng theo từng chặng, mức thực nhận là giá trị nhỏ nhất của mức phát và mức đăng ký.
- **Thông điệp lưu giữ** dành cho trạng thái, không dành cho sự kiện.
- **Phiên sạch** buộc máy khách đăng ký lại sau mỗi lần kết nối.
- **Keep-alive** phát hiện kết nối chết; **di chúc** báo hộ khi máy khách mất kết nối bất thường.

**Lý do lựa chọn.** Dự án chọn MQTT vì cần đẩy đơn xuống bếp tức thì, cần tách rời các thành phần, cần chạy nhẹ trên ESP32, và cần thông điệp lưu giữ cho bảng tồn kho. HTTP được giữ song song cho thao tác cần kết quả và cho việc khôi phục trạng thái. Đây là mẫu đẩy kết hợp kéo, với cơ sở dữ liệu là nguồn sự thật duy nhất.

**Bảo mật.** Mọi thông điệp được bọc trong phong bì ký HMAC-SHA256. Chữ ký tính trên chuỗi nội dung để Python và ESP32 cho cùng kết quả, kèm dấu thời gian chống phát lại, so sánh trong thời gian hằng và mặc định từ chối khi thiếu khóa. Broker nội bộ Mosquitto với tài khoản riêng và phân quyền tối thiểu là lớp bảo vệ thứ hai.

**Kỹ thuật lập trình:**
- Callback MQTT chạy trên luồng hoặc tác vụ mạng nên không được chạm giao diện. Kiosk dùng signal Qt, màn bếp dùng hàng đợi FreeRTOS.
- Thông điệp chỉ phát sau khi cơ sở dữ liệu đã lưu.
- Bên nhận khử trùng để chịu được QoS 1 gửi lặp.
- `client_id` duy nhất theo địa chỉ MAC.
- Bộ đệm PubSubClient phải được tăng.

**Hạn chế còn tồn tại** (cần thẳng thắn nhận diện):
- Chặng xuống màn bếp chỉ đạt QoS 0.
- Màn bếp chưa kiểm tra tuổi thông điệp.
- Webhook chưa kiểm tra kết quả chờ xác nhận.
- API chưa tự kết nối lại khi broker chưa sẵn sàng lúc khởi động.
- Hệ thống vẫn chạy trên broker công cộng.

Mỗi hạn chế đều không làm mất đơn, nhờ cơ chế đồng bộ HTTP, và đều đã có hướng khắc phục cụ thể.

---

*Tài liệu tham khảo: OASIS, MQTT Version 3.1.1 (2014) và MQTT Version 5.0 (2019); ISO/IEC 20922:2016; RFC 2104 – HMAC: Keyed-Hashing for Message Authentication; Eclipse Paho MQTT Python Client Documentation (paho-mqtt 2.x); PubSubClient API Documentation (N. O'Leary); Eclipse Mosquitto Documentation – mosquitto.conf, ACL; Mbed TLS – Message Digest API; ArduinoJson 7 Documentation.*

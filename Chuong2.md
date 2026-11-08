# BÀI GIẢNG CHUYÊN SÂU: CHƯƠNG 2 - TIẾN TRÌNH VÀ TRAO ĐỔI THÔNG TIN

*Môn học: Các hệ thống phân tán và ứng dụng (Distributed Systems & Applications)*

*Khoa Công nghệ Thông tin & Truyền thông - ĐH Bách Khoa Hà Nội (HUST)*

## 1. TIẾN TRÌNH VÀ LUỒNG (PROCESSES AND THREADS)

### 1.1. Bản chất: Tiến trình (Process) vs. Luồng (Thread)

Trong môi trường phân tán, hiệu năng tính toán và khả năng phản hồi phụ thuộc rất lớn vào cách một nút mạng phân bổ tài nguyên xử lý: ỏ+------------------------------------------------------------------------+ |ừ Usái thanh ghi, đổi bả tng PROCESSớ

```
  | Hàng đợi công việc (Queue)|
             +------------+------------+
                          |
         +----------------+----------------+
         v                v                v
   [ Worker 1 ]      [ Worker 2 ]     [ Worker N ]

```

#### B. Mô hình Máy trạng thái hữu hạn (FSM / Event-driven Architecture):

* Đại diện tiêu biểu: **Node.js, Nginx, Redis**.

* Vận hành bằng một luồng đơn chạy vòng lặp sự kiện (**Event Loop**) kết hợp với các cơ chế Non-blocking I/O multiplexing của hệ điều hành (`epoll` trên Linux, `kqueue` trên BSD/macOS).

* Không tốn bộ nhớ cấp stack cho hàng nghìn thread, không tốn chi phí context-switching.

### 1.4. Quản lý vòng đời Server: Inetd và Daemon

Để không phải chạy nền hàng chục server dịch vụ hiếm khi dùng (gây lãng phí RAM), hệ điều hành Unix/Linux sử dụng mô hình **Superserver (`inetd` / `xinetd`)**:

* `inetd` đóng vai trò là một Daemon tổng quản, mở và lắng nghe trên tất cả các cổng của các dịch vụ đã đăng ký trong `/etc/services` và `/etc/inetd.conf`.

* Khi có gói tin đến một cổng nhất định (ví dụ port 9999), `inetd` thực hiện `fork()` và `exec()` để khởi động tiến trình server thực tế phục vụ dịch vụ đó, truyền luồng socket qua `stdin/stdout`. Xử lý xong, server con tự hủy giải phóng tài nguyên.

## 2. NỀN TẢNG TRAO ĐỔI THÔNG TIN: SOCKET & TCP/UDP

### 2.1. Phân tầng và Đóng gói (Encapsulation)

Trong mạng Internet, việc truyền dữ liệu tuân thủ cấu trúc đóng gói nhiều lớp:

```
+-----------------------------------------------------------------------+
| Ethernet Header | IP Header | TCP Header | Dữ liệu ứng dụng | FCS CRC |
|    (14 bytes)   | (20 bytes)| (20 bytes) |  (Payload / MSS) | (4 bytes|
+-----------------------------------------------------------------------+
|<--------------------------- IP MTU (1500 bytes) --------------------->|
                  |<---------- TCP MSS (1460 bytes) ---------->|

```

* **MTU (Maximum Transmission Unit):** Kích thước khung dữ liệu lớn nhất mà tầng Data Link truyền tải được mà không bị phân mảnh (Ethernet mặc định $1500\text{ bytes}$).

* **MSS (Maximum Segment Size):** Kích thước dữ liệu payload tối đa của một segment TCP: 

  $$
  \text{MSS} = \text{MTU} - (\text{IP Header} + \text{TCP Header}) = 1500 - 20 - 20 = 1460\text{ bytes}
  $$

### 2.2. So sánh Giao thức Giao vận (Transport Protocols)

| Tiêu chí | TCP (Transmission Control Protocol) | UDP (User Datagram Protocol) | 
| ----- | ----- | ----- | 
| **Loại hình** | Hướng kết nối (Connection-oriented) | Không hướng kết nối (Connectionless) | 
| **Độ tin cậy** | Đảm bảo 100% dữ liệu không mất, đúng thứ tự (ACK, Retransmit) | Không tin cậy (Best-effort), có thể mất hoặc đảo trật tự | 
| **Kiểm soát luồng & nghẽn** | Có (Sliding Window, Congestion Window) | Không có | 
| **Đơn vị truyền** | Dòng byte liên tục (Byte-stream, không giữ ranh giới tin) | Gói tin rời rạc (Datagram, giữ nguyên ranh giới tin) | 
| **Ứng dụng** | Web (HTTP), Email (SMTP), File transfer (FTP), RPC | DNS, Streaming media, VoIP, Game thời gian thực | 

### 2.3. Lập trình Berkeley Sockets & Vòng đời Kết nối TCP

#### A. Luồng System Calls thiết lập kết nối:

```
      SERVER                                              CLIENT
  [ socket() ]                                         [ socket() ]
       |                                                    |
   [ bind() ]   (Gán IP + Port)                             |
       |                                                    |
  [ listen() ]  (Chuyển sang passive state)                 |
       |                                                    |
  [ accept() ]  (Chặn/Chờ kết nối)                          |
       |                                                    |
       |<------------- Bắt tay 3 bước (3-way handshake) --- [ connect() ]
       |               1. SYN J                              |
       |               2. SYN K, ACK J+1                    |
       |               3. ACK K+1                           |
  (Trả về connfd)                                     (connect() trả về)
       |                                                    |
  [ read() ]   <=============== Dữ liệu ===============>  [ write() ]
  [ write() ]  <=======================================>  [ read() ]
       |                                                    |
  [ close() ]  <--------- Bắt tay 4 bước đóng kết nối -- [ close() ]
                           (FIN - ACK - FIN - ACK)

```

#### B. Cơ chế Hàng đợi của hàm `listen(sockfd, backlog)`

Nhân Linux duy trì **2 hàng đợi** cho mỗi listening socket:

1. **Incomplete connection queue (SYN_RCVD state):** Chứa các kết nối client mới gửi `SYN`, đang chờ hoàn tất bắt tay 3 bước.

2. **Completed connection queue (ESTABLISHED state):** Chứa các kết nối đã hoàn thành bắt tay 3 bước, đang đợi tiến trình ứng dụng gọi lệnh `accept()` bốc ra xử lý.

* Tham số `backlog`: Quy định ngưỡng trần tối đa cho tổng dung lượng hàng đợi kết nối.

#### C. Kỹ thuật Socket Đa tiến trình (Concurrent TCP Server với `fork()`):

```
// Mã mẫu C minh họa cơ chế fork server
listenfd = socket(...);
bind(listenfd, ...);
listen(listenfd, BACKLOG);

for (;;) {
    connfd = accept(listenfd, ...); // Block chờ kết nối mới
    
    if ((pid = fork()) == 0) {
        // TIẾN TRÌNH CON (Child Process)
        close(listenfd);  // Con không cần lắng nghe, đóng listening socket
        do_work(connfd);  // Xử lý đọc/ghi qua socket kết nối
        close(connfd);   // Xong việc, đóng socket kết nối
        exit(0);         // Con kết thúc
    }
    // TIẾN TRÌNH CHA (Parent Process)
    close(connfd); // BẮT BUỘC: Cha phải đóng connfd vì reference count bị nhân đôi sau fork()!
}

```

> **Bẫy kỹ thuật (Pitfall):** Nếu tiến trình cha không gọi `close(connfd)`, số đếm tham chiếu (`reference count`) của socket sẽ không bao giờ về $0$ khi tiến trình con kết thúc. Kết quả là cờ `FIN` của TCP không thể gửi đi, liên kết mạng bị rò rỉ (socket leak) và treo vĩnh viễn ở trạng thái kết nối.

## 3. LỜI GỌI THỦ TỤC TỪ XA (RPC - REMOTE PROCEDURE CALL)

### 3.1. Bản chất và Mục tiêu của RPC

* **Vấn đề của Socket:** Socket chỉ truyền tải chuỗi byte phi cấu trúc (`byte[]`). Lập trình viên phải tự parse, tự serialize/deserialize, tự xử lý byte order $\rightarrow$ thủ công và dễ sinh lỗi.

* **Mục tiêu của RPC:** Cho phép một chương trình trên máy $A$ gọi một hàm/thủ tục trên máy tính $B$ từ xa với cú pháp tự nhiên như một lời gọi hàm cục bộ trong cùng một không gian nhớ: 

$$
\text{Result} = \text{ExecuteRemoteFunction}(x, y)
$$

* Cung cấp **Tính trong suốt về truy cập (Access Transparency)**.

### 3.2. Cơ chế Hoạt động Bên dưới (Under the Hood) của RPC

Để bắc cầu qua ranh giới mạng và sự khác biệt về phần cứng, RPC đưa vào 2 thành phần đại diện trung gian:

* **Client Stub:** Thư viện giả lập hàm ở phía client.

* **Server Stub (Skeleton):** Bộ phân phối và tiếp nhận hàm ở phía server.

```
+---------------+                                   +---------------+
| CLIENT MACHINE|                                   | SERVER MACHINE|
|               |                                   |               |
| Client Process|                                   | Server Process|
|  k=add(i, j)  |                                   |   add(i, j)   |
|       | (1)   |                                   |       ^ (6)   |
|       v       |                                   |       |       |
|  Client Stub  |                                   |  Server Stub  |
| [Marshalling] |                                   |[Unmarshalling]|
|       | (2)   |                                   |       ^ (5)   |
|       v       |                                   |       |       |
|   Client OS   |=== (3) Gửi gói tin qua mạng ====> |   Server OS   |
|   (Socket)    |             (Request)             |    (Socket)   |
+---------------+                                   +---------------+

```

#### Trình tự 6 bước thực thi một lời gọi RPC:

1. **Client Call:** Tiến trình client thực hiện lời gọi hàm cục bộ đến `Client Stub`, truyền các tham số thông thường qua stack.

2. **Marshalling (Serialization):** `Client Stub` đóng gói tên thủ tục và các tham số thành một mảng byte có định dạng chuẩn tắc, độc lập với kiến trúc phần cứng.

3. **Network Transit:** `Client OS` gửi bản tin request qua giao thức mạng (UDP hoặc TCP) tới server; tiến trình client bị tạm dừng (block) chờ kết quả.

4. **Server OS Delivery:** `Server OS` nhận các gói tin từ socket và chuyển tiếp cho `Server Stub`.

5. **Unmarshalling (Deserialization):** `Server Stub` bóc tách dữ liệu mảng byte, khôi phục lại các biến kiểu dữ liệu nguyên bản, đặt lên stack cục bộ của server.

6. **Local Execution:** `Server Stub` gọi hàm `add()` thực tế trên máy chủ, nhận kết quả trả về, thực hiện quy trình marshalling ngược lại để gửi kết quả về cho client.

### 3.3. Các Thách thức Kỹ thuật Lớn trong RPC

#### A. Sai lệch Biểu diễn Dữ liệu (Big-Endian vs. Little-Endian)

* Máy Intel x86 biểu diễn số nguyên dạng **Little-Endian** (byte thấp lưu ở địa chỉ nhỏ).

* Máy SPARC/ARM (mặc định mạng) biểu diễn dạng **Big-Endian** (byte cao lưu ở địa chỉ nhỏ).

* *Ví dụ:* Số nguyên $5$ ($32\text{-bit}$):

  * Big-Endian: `00 00 00 05`

  * Little-Endian: `05 00 00 00`

* $\rightarrow$ Nếu gửi mảng byte trực tiếp không qua quy chuẩn, số $5$ từ Intel gửi sang SPARC sẽ bị hiểu nhầm thành $5 \times 2^{24} = 83.886.080$!

* **Giải pháp:** Sử dụng chuẩn mã hóa trung gian như **XDR (External Data Representation)** của SUN RPC, hoặc **NDR** của DCE RPC, hay **Protocol Buffers** ngày nay.

#### B. Vấn đề Truyền Tham chiếu (Pointers / References)

* Bộ nhớ trên 2 máy độc lập là hoàn toàn tách biệt. Một con trỏ `char *buf = 0x7ffd12` trên Client không có ý nghĩa gì trên Server.

* **Giải pháp:**

  1. *Copy-Restore:* Copy toàn bộ vùng nhớ mà con trỏ đang trỏ tới, gửi sang server, server xử lý trên vùng copy, sau đó gửi ngược lại toàn bộ cấu trúc để client ghi đè lên bộ nhớ gốc. Nhược điểm: Rất tốn băng thông nếu cấu trúc dữ liệu dạng đồ thị/cây phức tạp.

  2. *Cấm hoàn toàn việc truyền con trỏ raw.* Bắt buộc sử dụng các định danh toàn cục (Global Unique IDs).

### 3.4. Các Biến thể của Mô hình RPC

```
1. Synchronous RPC (Đồng bộ truyền thống)
Client:   ---[Call RPC]================(Treo chờ)=================>[Nhận Reply & Chạy tiếp]---
Server:                        [Nhận Req]----(Xử lý)----[Trả lời]

2. Asynchronous / Nowait RPC (Bất đồng bộ)
Client:   ---[Call RPC]--->[Tiếp tục chạy ngay mà không đợi kết quả]---
Server:              [Nhận Req]----(Xử lý ngầm)

3. Deferred-Synchronous RPC (Đồng bộ trì hoãn)
Client:   ---[Call RPC]--->[Làm việc khác]--->[Quay lại Poll hoặc lấy kết quả sau]---

4. Callback RPC (Hàm gọi ngược)
Client:   ---[Call RPC + Gửi kèm Callback Reference]--->[Chạy tiếp]
Server:                                                     |
                                                (Xong việc) v
                                          [Gọi Callback ngược về Client]

5. One-Way / Batch RPC (Gộp lô)
Client:   Gộp nhiều lệnh RPC [sub1, sub2, sub3] vào một segment mạng gửi 1 lần duy nhất.

```

### 3.5. RMI (Remote Method Invocation) - Hướng Đối Tượng Phân Tán

RMI là bước tiến hóa của RPC trong thế giới Lập trình Hướng đối tượng (OOP), tiêu biểu trong hệ sinh thái Java:

* **Khác biệt cốt lõi:** RPC chỉ hỗ trợ gọi các hàm thủ tục thuần túy. RMI cho phép thao tác trên các **Đối tượng từ xa (Remote Objects)**.

* **Truyền tham chiếu đối tượng (Remote Object Reference):**

  * Trong RPC, mọi dữ liệu truyền đi đều phải là tham trị (Pass-by-value).

  * Trong RMI, nếu một đối tượng kế thừa giao diện `Remote`, khi truyền qua mạng, hệ thống chỉ gửi một chuỗi định danh tham chiếu toàn cục. Phía bên nhận sẽ nhận được một đối tượng **Proxy (Stub)**; mọi lời gọi hàm trên Proxy sẽ được định tuyến ngược về đối tượng gốc trên máy chủ.

* **Kiến trúc 3 tầng của Java RMI:**

  1. *Stub/Skeleton Layer:* Đóng/mở gói tham số và xử lý ngoại lệ mạng.

  2. *Remote Reference Layer (RRL):* Quản lý ngữ nghĩa liên kết (điểm-điểm hoặc multicast).

  3. *Transport Layer:* Quản lý socket TCP kết nối vật lý.

## 4. TRAO ĐỔI THÔNG TIN HƯỚNG THÔNG ĐIỆP (MESSAGE-ORIENTED COMMUNICATION)

### 4.1. Thông điệp Tạm thời vs. Thông điệp Bền vững

* **Truyền thông điệp Tạm thời (Transient Messaging):** Thông điệp chỉ tồn tại khi cả ứng dụng gửi và ứng dụng nhận đang cùng chạy. Nếu bên nhận tắt hoặc mạng đứt tại thời điểm gửi, thông điệp bị hủy bỏ (Ví dụ: Berkeley Socket, MPI).

* **Truyền thông điệp Bền vững (Persistent Messaging):** Thông điệp được lưu trữ an toàn trên đĩa cứng của hệ thống trung gian (**Message Queue**) cho đến khi bên nhận sẵn sàng lấy về. Cho phép bên gửi và bên nhận hoàn toàn lệch pha về mặt thời gian (Time Uncoupling).

```
[ Sender Chạy ] ---> Gửi Msg vào [ Hàng đợi bền vững (Disk) ]
      |
(Sender Tắt)                                    [ Receiver Bật lên ]
                                                          |
                                           Lấy Msg từ hàng đợi ra xử lý

```

### 4.2. Middleware Hướng Thông Điệp (MOM - Message-Oriented Middleware)

MOM (như **RabbitMQ, Apache ActiveMQ, IBM MQ**) giải quyết bài toán tích hợp các hệ thống doanh nghiệp không đồng nhất.

#### A. Kiến trúc Định tuyến Hàng đợi (Message Queue Routers):

Khi mạng mở rộng, các Message Broker liên kết với nhau thành mạng lưới Router hàng đợi:

* Client A gửi bản tin vào `Send Queue` cục bộ.

* Mạng lưới router ($R1 \rightarrow R2$) định tuyến gói tin dựa trên bảng chuyển tiếp mà không cần sự can thiệp của ứng dụng.

* Bản tin rơi vào `Receive Queue` của Client B.

#### B. Message Broker & Chuyển đổi định dạng (Conversion Rules):

Đóng vai trò là "thông dịch viên" trung tâm giữa các ứng dụng viết bằng các công nghệ khác nhau:

* Tiếp nhận thông điệp từ Client gửi.

* Tra cứu luật biến đổi (Mapping Rules) trong kho tri thức (Repository).

* Chuyển đổi từ cấu trúc XML sang JSON hoặc Protobuf trước khi đẩy sang đích.

#### C. Các mẫu phân phối tin nhắn kinh điển (Enterprise Patterns):

1. **Work Queues (Competing Consumers):** Một hàng đợi, nhiều worker cùng lắng nghe; mỗi task chỉ được xử lý bởi **một worker duy nhất** (Round-robin hoặc Fair dispatch).

2. **Publish/Subscribe (Fanout Exchange):** Thông điệp được nhân bản và phát đồng loạt tới tất cả các subscriber đã đăng ký.

3. **Routing (Direct Exchange):** Định tuyến thông điệp tới hàng đợi cụ thể dựa trên khóa định tuyến chính xác (**Routing Key**).

4. **Topics (Topic Exchange):** Định tuyến mềm dẻo dựa trên mẫu so khớp chuỗi (wildcard pattern: `*.logs`, `audit.#`).

## 5. TRAO ĐỔI THÔNG TIN HƯỚNG DÒNG (STREAM-ORIENTED COMMUNICATION)

### 5.1. Dữ liệu Liên tục & Các chế độ Truyền dòng

Khác với dữ liệu truyền thống (văn bản, số liệu) có thể truyền rời rạc từng gói, **Phương tiện truyền thông liên tục (Continuous Media - Audio/Video)** phụ thuộc nghiêm ngặt vào yếu tố **thời gian**:

* **Truyền không đồng bộ (Asynchronous Streaming):** Các gói tin truyền đi không có ràng buộc về thời điểm đến, chỉ cần đến đúng thứ tự (ví dụ: tải file mp4 về máy trước rồi mới xem).

* **Truyền đồng bộ (Synchronous Streaming):** Có quy định trễ tối đa cho mỗi gói tin.

* **Truyền đẳng thời (Isochronous Streaming):** Yêu cầu cực kỳ khắt khe: độ trễ giữa các gói tin phải cố định; không được nhanh hơn và không được chậm hơn (Jitter bằng $0$).

### 5.2. Chất lượng Dịch vụ (Quality of Service - QoS)

Các thông số then chốt quyết định trải nghiệm người dùng đối với dòng dữ liệu:

* **Throughput (Băng thông):** Tốc độ truyền tải bit/giây (bps).

* **End-to-End Delay:** Tổng thời gian từ lúc camera thu hình đến khi phát ra màn hình client.

  * *Hội* thoại tương *tác thời gian thực (VoIP/Zoom):* $\text{Delay} < 150\text{ ms}$ là lý tưởng, $> 400\text{ ms}$ là giật cục không thể nói chuyện được.

* **Jitter (Độ biến thiên trễ):** Sự chênh lệch thời gian đến giữa các gói tin liên tiếp.

#### Kỹ thuật Playback Buffer để triệt tiêu Jitter:

Mạng IP có bản chất là Best-effort $\rightarrow$ gói tin đến máy khách với thời gian xộc xệch, không đều đặn.

* **Giải pháp:** Client không phát video ngay khi vừa nhận được gói tin, mà giữ lại trong một vùng nhớ đệm (**Playout Buffer**) trong một khoảng thời gian $\Delta t$ (ví dụ trễ $2\text{ - }5\text{ giây}$).

* Các gói tin được bốc ra khỏi buffer với tốc độ đều đặn theo chu kỳ cố định $\rightarrow$ Video mượt mà, loại bỏ hiện tượng giật hình do biến thiên mạng.

```
Nguồn phát:     [1] [2] [3] [4] [5] [6] (Khoảng cách đều đặn)
                     \   \   \   \   \
Mạng truyền:          \   \   \   \   \ (Gặp Jitter - dạt gói, lệch trễ)
                       v   v   v   v   v
Client Buffer:  |..chờ..| [1]  [2]  [3]  [4] [5] | (Hấp thụ độ trễ)
                              |    |    |    |   |
Màn hình Play:                v    v    v    v   v (Phát ra đều đặn tuyệt đối)

```

#### Forward Error Correction (FEC) & Interleaving:

Trong truyền dòng thời gian thực, cơ chế truyền lại gói mất (ARQ/ACK) của TCP bị loại bỏ vì quá trễ. Thay vào đó dùng UDP kết hợp:

* **Interleaved Transmission (Truyền đan xen):** Thay vì gửi tuần tự cụm frame $[1, 2, 3, 4][5, 6, 7, 8]$, bộ mã hóa sẽ xáo trộn các mẩu dữ liệu nhỏ ra nhiều gói khác nhau $[1, 5, 9, 13][2, 6, 10, 14]$.

* *Hệ quả:* Khi mạng làm rớt một gói tin lớn, lỗi mất mát sẽ bị phân tán đều thành các vết xước siêu nhỏ trên từng khung hình mà tai/mắt con người không nhận ra, thay vì mất đứt một khoảng video dài $1$ giây.

### 5.3. Đồng bộ hóa Dòng (Stream Synchronization)

* **Lip Synchronization (Đồng bộ khẩu hình):** Đồng bộ giữa dòng hình ảnh (Video stream) và dòng âm thanh (Audio stream) phát ra từ 2 socket riêng biệt.

* Ngưỡng dung sai của thính/thị giác con người:

  * mốc lệch trễ (Skew): Âm thanh phát trước video tối đa $20\text{ ms}$ hoặc âm thanh phát sau video tối đa $80\text{ ms}$ thì não bộ vẫn cảm nhận là khớp khẩu hình. Lệch quá mức này người xem sẽ nhận ra ngay hiện tượng "hình một đằng, tiếng một nẻo".

## 6. CÂU HỎI TỰ VẤN & BÀI TOÁN THIẾT KẾ THỰC CHIẾN

### 6.1. Câu hỏi bẫy tư duy (Active Recall)

**Câu 1:** *Khi lập trình RPC qua mạng diện rộng (WAN), tại sao lập trình viên không nên cố gắng giấu hoàn toàn tính chất phân tán (100% Access Transparency)?*

* **Giải thích:** Một lời gọi hàm cục bộ hoàn thành trong vòng vài nano-giây và luôn thành công nếu mã nguồn đúng. Lời gọi qua WAN mất hàng trăm mili-giây và có thể thất bại bất cứ lúc nào do đứt cáp, sập router hoặc timeout. Nếu cố tạo cảm giác nó như hàm cục bộ mà không bộc lộ giao diện xử lý ngoại lệ (Exception/Network Error Handling), lập trình viên ứng dụng sẽ không lường trước các trường hợp nghẽn và treo ứng dụng (thread starvation).

**Câu 2:** *Tại sao các hệ thống giao dịch tài chính (Core Banking) bắt buộc sử dụng Kiến trúc Hướng thông điệp bền vững (MOM) thay vì gọi trực tiếp RPC/gRPC giữa các dịch vụ?*

* **Giải thích:** Tính sẵn sàng và khả năng chịu lỗi. Với RPC (đồng bộ), nếu Server nhận tiền bị ngắt mạng tạm thời lúc Server chuyển tiền đang gọi hàm, giao dịch sẽ fail hoặc rơi vào trạng thái phân rã không nhất quán. Khi dùng MOM, giao dịch được ghi bền vững vào Queue trên đĩa cứng; kể cả khi toàn bộ hệ thống đích sập, thông điệp vẫn nằm an toàn trong hàng đợi và sẽ được xử lý tuần tự ngay khi hệ thống phục hồi, đảm bảo nguyên tắc xử lý ít nhất một lần (At-least-once delivery) và bảo toàn tính toàn vẹn tài chính.

### 6.2. Bài toán Thiết kế Thực tế (System Design Scenario)

**Đề bài:** Hãy thiết kế kiến trúc truyền thông cho hệ thống ứng dụng họp trực tuyến (như Zoom / Google Meet) cho 100 người trong cùng một phòng họp.

**Giải pháp kiến trúc áp dụng kiến thức Chương 2:**

1. **Tín hiệu điều khiển (Signaling / Room Management):**

   * Sử dụng **RPC/WebSocket trên nền TCP**: Dùng để quản lý trạng thái phòng họp, danh sách người tham gia, bật/tắt micro. Dữ liệu này có kích thước nhỏ, cần độ tin cậy tuyệt đối (Reliable).

2. 

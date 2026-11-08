# BÀI GIẢNG CHUYÊN SÂU: CHƯƠNG 2 - TIẾN TRÌNH VÀ TRAO ĐỔI THÔNG TIN

*Môn học: Các hệ thống phân tán và ứng dụng (Distributed Systems & Applications)*
*Khoa Công nghệ Thông tin & Truyền thông - ĐH Bách Khoa Hà Nội (HUST)*

> **Ghi chú biên tập:** Bản gốc bị lỗi ở §1.1 (ký tự hỏng), thiếu §1.2 - §1.3A và bị cụt ở §6.2. Các phần này đã được khôi phục/viết lại theo đúng mạch nội dung còn sót lại; giảng viên nên đối chiếu với bản gốc của mình.

---

## 1. TIẾN TRÌNH VÀ LUỒNG (PROCESSES AND THREADS)

### 1.1. Bản chất: Tiến trình (Process) vs. Luồng (Thread)

Trong môi trường phân tán, hiệu năng tính toán và khả năng phản hồi phụ thuộc rất lớn vào cách một nút mạng phân bổ tài nguyên xử lý.

| Tiêu chí | Tiến trình (Process) | Luồng (Thread) |
| :--- | :--- | :--- |
| **Bản chất** | Chương trình đang thực thi, có không gian địa chỉ riêng | Đơn vị thực thi nằm trong một tiến trình |
| **Tài nguyên** | Sở hữu bộ nhớ, file descriptor, bảng trang riêng | Chia sẻ code, heap, file descriptor với các thread cùng tiến trình; chỉ có stack và thanh ghi riêng |
| **Chi phí tạo / chuyển ngữ cảnh** | Cao (lưu thanh ghi, đổi bảng trang, làm mất hiệu lực TLB) | Thấp hơn (không đổi không gian địa chỉ) |
| **Cô lập lỗi** | Tốt (một tiến trình lỗi ít ảnh hưởng tiến trình khác) | Kém (một thread lỗi bộ nhớ có thể làm sập cả tiến trình) |
| **Giao tiếp** | Cần IPC (pipe, socket, shared memory) | Dùng chung bộ nhớ, nhưng phải đồng bộ (lock, mutex) |

### 1.2. Luồng ở mức người dùng và mức nhân (User-level vs Kernel-level Threads)

* **User-level thread:** thư viện ở không gian người dùng quản lý, chuyển ngữ cảnh rất rẻ, nhưng một lời gọi hệ thống bị chặn (blocking syscall) có thể làm treo toàn bộ tiến trình.
* **Kernel-level thread:** do hệ điều hành lập lịch, chạy song song thật trên nhiều lõi, không bị treo lan, nhưng chuyển ngữ cảnh tốn hơn.
* Hệ điều hành hiện đại chủ yếu dùng kernel thread (mô hình 1:1), một số runtime (Go, Erlang) bổ sung lớp luồng nhẹ (goroutine / lightweight process) lập lịch ở mức người dùng.

### 1.3. Các mô hình tổ chức Server

#### A. Mô hình Dispatcher/Worker (Thread pool):

* Một luồng **Dispatcher** nhận yêu cầu, đặt vào hàng đợi; một tập **Worker** lấy việc từ hàng đợi để xử lý (có thể chặn khi đợi I/O mà không làm ảnh hưởng luồng khác).
* Dùng **thread pool** (số worker cố định) để tránh chi phí tạo thread cho từng yêu cầu và chặn trần mức tiêu thụ tài nguyên.

```
              Yêu cầu từ Client
                     |
                     v
              [ Dispatcher ]
                     |
                     v
       | Hàng đợi công việc (Queue) |
                     |
        +------------+------------+
        v            v            v
  [ Worker 1 ]  [ Worker 2 ]  [ Worker N ]
```

#### B. Mô hình Máy trạng thái hữu hạn (FSM / Event-driven Architecture):

* Đại diện tiêu biểu: **Node.js, Nginx, Redis**.
* Vận hành dựa trên vòng lặp sự kiện (**Event Loop**) kết hợp Non-blocking I/O multiplexing của hệ điều hành (`epoll` trên Linux, `kqueue` trên BSD/macOS). Mỗi luồng/tiến trình xử lý nhiều kết nối bằng cách lưu trạng thái của từng kết nối.
* Không tốn bộ nhớ stack cho hàng nghìn thread và giảm đáng kể chi phí context-switching.
* *Lưu ý thực tế:* Node.js chạy mã JavaScript trên một luồng chính; Redis thực thi lệnh trên một luồng (từ Redis 6 có thêm luồng I/O); **Nginx** dùng một tiến trình master và *nhiều* tiến trình worker, mỗi worker có event loop riêng. Nhược điểm chung: một tác vụ tính toán nặng hoặc lời gọi chặn sẽ làm nghẽn cả vòng lặp.

### 1.4. Quản lý vòng đời Server: Inetd và Daemon

Để không phải chạy nền hàng chục server dịch vụ hiếm khi dùng (gây lãng phí RAM), hệ điều hành Unix/Linux có thể dùng mô hình **Superserver (`inetd` / `xinetd`)**:

* `inetd` đóng vai trò là một Daemon tổng quản, mở và lắng nghe trên tất cả các cổng của các dịch vụ đã khai báo trong `/etc/inetd.conf` (ánh xạ tên dịch vụ - số cổng tra từ `/etc/services`).
* Khi có kết nối đến một cổng nhất định (ví dụ port 9999), `inetd` thực hiện `fork()` và `exec()` để khởi động tiến trình server thực tế, gắn socket vào `stdin/stdout/stderr` của tiến trình đó. Xử lý xong, server con kết thúc và giải phóng tài nguyên.
* *Ghi chú:* trên các bản phân phối Linux hiện đại, vai trò này thường được thay bằng **systemd socket activation**.

---

## 2. NỀN TẢNG TRAO ĐỔI THÔNG TIN: SOCKET & TCP/UDP

### 2.1. Phân tầng và Đóng gói (Encapsulation)

Trong mạng Internet, việc truyền dữ liệu tuân thủ cấu trúc đóng gói nhiều lớp:

```
+--------------------------------------------------------------------------+
| Ethernet Header | IP Header | TCP Header | Dữ liệu ứng dụng | FCS (CRC)  |
|   (14 bytes)    | (20 bytes)| (20 bytes) | (Payload <= MSS) | (4 bytes)  |
+--------------------------------------------------------------------------+
                  |<------------------ MTU = 1500 bytes ------------------>|
                  |  (gói IP = payload của khung Ethernet)                 |
                                          |<------ MSS = 1460 bytes ----->|
```

* **MTU (Maximum Transmission Unit):** Kích thước tối đa của gói dữ liệu (gói IP) mà một khung ở tầng Data Link mang được mà không bị phân mảnh (Ethernet mặc định $1500\text{ bytes}$, **không** tính Ethernet header và FCS).
* **MSS (Maximum Segment Size):** Kích thước dữ liệu payload tối đa của một segment TCP:

  $$
  \text{MSS} = \text{MTU} - (\text{IP Header} + \text{TCP Header}) = 1500 - 20 - 20 = 1460\text{ bytes}
  $$

  *(Giả sử header không có tùy chọn; thực tế khi bật TCP Timestamps, MSS thường là 1448 bytes.)*

### 2.2. So sánh Giao thức Giao vận (Transport Protocols)

| Tiêu chí | TCP (Transmission Control Protocol) | UDP (User Datagram Protocol) |
| :--- | :--- | :--- |
| **Loại hình** | Hướng kết nối (Connection-oriented) | Không hướng kết nối (Connectionless) |
| **Độ tin cậy** | Giao nhận tin cậy, đúng thứ tự (ACK, Retransmit); báo lỗi khi kết nối đứt | Không tin cậy (Best-effort), có thể mất, trùng hoặc đảo thứ tự |
| **Kiểm soát luồng & nghẽn** | Có (Sliding Window, Congestion Window) | Không có (ứng dụng tự lo) |
| **Đơn vị truyền** | Dòng byte liên tục (Byte-stream, không giữ ranh giới tin) | Gói tin rời rạc (Datagram, giữ nguyên ranh giới tin) |
| **Ứng dụng** | Web (HTTP/1.1, HTTP/2), Email (SMTP), File transfer (FTP), RPC/gRPC | DNS, VoIP, Game thời gian thực, truyền thông thời gian thực (RTP), QUIC/HTTP/3 |

### 2.3. Lập trình Berkeley Sockets & Vòng đời Kết nối TCP

#### A. Luồng System Calls thiết lập kết nối:

```
      SERVER                                              CLIENT
  [ socket() ]                                         [ socket() ]
       |                                                    |
   [ bind() ]   (Gán IP + Port)                             |
       |                                                    |
  [ listen() ]  (Chuyển sang trạng thái LISTEN)             |
       |                                                    |
  [ accept() ]  (Chặn/Chờ kết nối)                    [ connect() ]
       |                                                    |
       |<------------ 1. SYN (seq = J) -------------------- |
       | ------------ 2. SYN (seq = K), ACK (J+1) --------> |
       |<------------ 3. ACK (K+1) ------------------------ |
       |            (Bắt tay 3 bước - 3-way handshake)      |
  (accept() trả về connfd)                       (connect() trả về)
       |                                                    |
  [ read() ]   <=============== Dữ liệu ===============>  [ write() ]
  [ write() ]  <=======================================>  [ read() ]
       |                                                    |
  [ close() ]  <---- Đóng kết nối 4 bước (FIN-ACK-FIN-ACK) ---> [ close() ]
```

* `connect()` trả về sau khi client nhận SYN-ACK (và gửi ACK); `accept()` trả về khi kết nối đã hoàn tất bắt tay.
* Việc đóng kết nối có thể bắt đầu từ bất kỳ phía nào (phía gọi `close()` trước gửi FIN trước).

#### B. Cơ chế Hàng đợi của hàm `listen(sockfd, backlog)`

Nhân Linux duy trì **2 hàng đợi** cho mỗi listening socket:

1. **Incomplete connection queue / SYN queue (SYN_RCVD state):** Chứa các kết nối client mới gửi `SYN`, đang chờ hoàn tất bắt tay 3 bước.
2. **Completed connection queue / Accept queue (ESTABLISHED state):** Chứa các kết nối đã hoàn thành bắt tay 3 bước, đang đợi tiến trình ứng dụng gọi `accept()` bốc ra xử lý.

* **Tham số `backlog`:** Trên Linux (từ nhân 2.2), `backlog` quy định độ dài tối đa của **hàng đợi accept** (bị chặn trên bởi `net.core.somaxconn`); kích thước hàng đợi SYN do tham số riêng (`tcp_max_syn_backlog`) điều khiển. (Một số tài liệu kinh điển, như Stevens, mô tả `backlog` là tổng của hai hàng đợi trên các hệ cũ hơn.)

#### C. Kỹ thuật Socket Đa tiến trình (Concurrent TCP Server với `fork()`):

```c
// Mã mẫu C minh họa cơ chế fork server
listenfd = socket(...);
bind(listenfd, ...);
listen(listenfd, BACKLOG);

signal(SIGCHLD, sigchld_handler);  // handler gọi waitpid() để thu dọn tiến trình zombie

for (;;) {
    connfd = accept(listenfd, ...); // Block chờ kết nối mới

    if ((pid = fork()) == 0) {
        // TIẾN TRÌNH CON (Child Process)
        close(listenfd);  // Con không cần lắng nghe, đóng listening socket
        do_work(connfd);  // Xử lý đọc/ghi qua socket kết nối
        close(connfd);    // Xong việc, đóng socket kết nối
        exit(0);          // Con kết thúc
    }
    // TIẾN TRÌNH CHA (Parent Process)
    close(connfd); // BẮT BUỘC: Cha phải đóng connfd vì reference count bị nhân đôi sau fork()!
}
```

> **Bẫy kỹ thuật 1:** Nếu tiến trình cha không gọi `close(connfd)`, số đếm tham chiếu của socket không bao giờ về $0$ khi tiến trình con đóng nó. Kết quả là TCP không gửi `FIN`, kết nối không bao giờ được đóng (socket/file descriptor leak) và cha sớm cạn file descriptor.
>
> **Bẫy kỹ thuật 2:** Tiến trình con kết thúc mà cha không gọi `wait()`/`waitpid()` sẽ trở thành tiến trình **zombie**; cần bắt tín hiệu `SIGCHLD` để thu dọn.

---

## 3. LỜI GỌI THỦ TỤC TỪ XA (RPC - REMOTE PROCEDURE CALL)

### 3.1. Bản chất và Mục tiêu của RPC

* **Vấn đề của Socket:** Socket chỉ truyền tải chuỗi byte phi cấu trúc (`byte[]`). Lập trình viên phải tự parse, tự serialize/deserialize, tự xử lý byte order $\rightarrow$ thủ công và dễ sinh lỗi.
* **Mục tiêu của RPC:** Cho phép một chương trình trên máy $A$ gọi một hàm/thủ tục trên máy $B$ từ xa với cú pháp giống một lời gọi hàm cục bộ:

$$
\text{Result} = \text{ExecuteRemoteFunction}(x, y)
$$

* Cung cấp **Tính trong suốt về truy cập (Access Transparency)**.

### 3.2. Cơ chế Hoạt động Bên dưới (Under the Hood) của RPC

Để bắc cầu qua ranh giới mạng và sự khác biệt về phần cứng, RPC đưa vào 2 thành phần đại diện trung gian:

* **Client Stub:** Hàm đại diện (proxy) ở phía client, có cùng chữ ký với hàm thật.
* **Server Stub (Skeleton):** Thành phần phía server nhận thông điệp, giải mã và gọi hàm thật.

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
|   Client OS   |=== (3) Gửi request qua mạng ====> |   Server OS   |
|   (Socket)    |                                   |    (Socket)   |
+---------------+                                   +---------------+
                                                       (4) OS chuyển
                                                       thông điệp lên Stub
```

#### Trình tự thực thi một lời gọi RPC (chiều đi: bước 1-6, chiều về: bước 7-9):

1. **Client Call:** Tiến trình client gọi hàm cục bộ tới `Client Stub`, truyền tham số theo cách thông thường (qua stack).
2. **Marshalling (Serialization):** `Client Stub` đóng gói tên thủ tục và các tham số thành thông điệp (mảng byte) theo định dạng chuẩn, độc lập kiến trúc phần cứng.
3. **Network Transit:** `Client OS` gửi thông điệp request qua mạng (UDP hoặc TCP) tới server; tiến trình client bị chặn (block) chờ kết quả (với RPC đồng bộ).
4. **Server OS Delivery:** `Server OS` nhận gói tin từ socket và chuyển cho `Server Stub`.
5. **Unmarshalling (Deserialization):** `Server Stub` bóc tách dữ liệu, khôi phục các tham số về đúng kiểu, đặt lên stack cục bộ của server.
6. **Local Execution:** `Server Stub` gọi hàm `add()` thực tế trên máy chủ và nhận kết quả trả về.
7. **Marshalling kết quả:** `Server Stub` đóng gói kết quả thành thông điệp reply và nhờ `Server OS` gửi về client.
8. **Client nhận reply:** `Client OS` nhận thông điệp và chuyển cho `Client Stub`.
9. **Unmarshalling & Return:** `Client Stub` giải mã kết quả, trả về cho tiến trình client như kết quả của một hàm cục bộ; client được tiếp tục chạy.

### 3.3. Các Thách thức Kỹ thuật Lớn trong RPC

#### A. Sai lệch Biểu diễn Dữ liệu (Big-Endian vs. Little-Endian)

* Máy Intel/AMD x86 biểu diễn số nguyên dạng **Little-Endian** (byte thấp lưu ở địa chỉ nhỏ).
* Các kiến trúc như SPARC, PowerPC (cấu hình cổ điển), Motorola 68k dùng **Big-Endian** (byte cao lưu ở địa chỉ nhỏ). Big-Endian cũng là **thứ tự byte của mạng** (network byte order). *(ARM có thể chạy cả hai chế độ nhưng trong thực tế hầu hết chạy Little-Endian.)*
* *Ví dụ:* Số nguyên $5$ ($32\text{-bit}$):
  * Big-Endian: `00 00 00 05`
  * Little-Endian: `05 00 00 00`
* $\rightarrow$ Nếu gửi nguyên mảng byte của máy Little-Endian cho máy Big-Endian mà không chuẩn hóa, số $5$ sẽ bị hiểu nhầm thành $5 \times 2^{24} = 83.886.080$!
* **Giải pháp:** Dùng chuẩn mã hóa trung gian như **XDR (External Data Representation)** của Sun RPC (ONC RPC), **NDR** của DCE RPC, hoặc các định dạng serialization hiện đại như **Protocol Buffers**.

#### B. Vấn đề Truyền Tham chiếu (Pointers / References)

* Không gian địa chỉ của 2 máy là hoàn toàn tách biệt. Một con trỏ `char *buf = 0x7ffd12` trên Client không có ý nghĩa gì trên Server.
* **Giải pháp:**
  1. *Copy-Restore:* Sao chép vùng nhớ mà con trỏ trỏ tới và gửi sang server, server xử lý trên bản sao, rồi gửi bản đã sửa về để client ghi đè lên dữ liệu gốc. Nhược điểm: tốn băng thông nếu cấu trúc lớn hoặc phức tạp (đồ thị, cây).
  2. *Không truyền con trỏ thô:* thay vào đó truyền dữ liệu theo giá trị (đã marshalling), hoặc dùng **tham chiếu từ xa / định danh toàn cục (Global Unique IDs)** để trỏ tới đối tượng ở phía chủ.

### 3.4. Các Biến thể của Mô hình RPC

```
1. Synchronous RPC (Đồng bộ truyền thống)
Client:   ---[Call RPC]================(Treo chờ)=================>[Nhận Reply & Chạy tiếp]---
Server:                        [Nhận Req]----(Xử lý)----[Trả lời]

2. Asynchronous RPC (Bất đồng bộ)
Client:   ---[Call RPC]--->[Nhận Ack "đã nhận yêu cầu"]--->[Chạy tiếp, không đợi kết quả]---
Server:              [Nhận Req]--[Ack ngay]--(Xử lý ngầm)

3. Deferred-Synchronous RPC (Đồng bộ trì hoãn)
Client:   ---[Call RPC]--->[Làm việc khác]--->[Poll hoặc đợi để lấy kết quả sau]---
          (thực chất là 2 RPC bất đồng bộ: một để yêu cầu, một để nhận kết quả)

4. Callback RPC (Hàm gọi ngược)
Client:   ---[Call RPC + Gửi kèm Callback Reference]--->[Chạy tiếp]
Server:                                                     |
                                                (Xong việc) v
                                          [Gọi Callback ngược về Client]

5. One-Way RPC
Client:   ---[Call RPC]--->[Chạy tiếp ngay, không chờ cả kết quả lẫn xác nhận]---
          (không bảo đảm server đã nhận hay thực thi)

6. Batch RPC (Gộp lô, kỹ thuật tối ưu)
Client:   Gộp nhiều lệnh [sub1, sub2, sub3] vào một thông điệp, gửi 1 lần để giảm số lần round-trip.
```

### 3.5. RMI (Remote Method Invocation) - Hướng Đối Tượng Phân Tán

RMI là bước tiến hóa của RPC trong lập trình hướng đối tượng, tiêu biểu trong hệ sinh thái Java:

* **Khác biệt cốt lõi:** RPC gọi các thủ tục/hàm; RMI cho phép gọi phương thức trên các **Đối tượng từ xa (Remote Objects)**.
* **Cách truyền tham số trong Java RMI:**
  * Đối tượng cài đặt giao diện `Remote` được truyền **theo tham chiếu**: hệ thống gửi một tham chiếu từ xa (về bản chất là stub đã tuần tự hóa). Phía nhận có một đối tượng **Proxy (Stub)**; mọi lời gọi phương thức trên Proxy được chuyển ngược về đối tượng gốc trên máy chủ.
  * Đối tượng thông thường phải `Serializable` và được truyền **theo giá trị** (sao chép sang máy nhận).
* **Kiến trúc 3 tầng của Java RMI:**
  1. *Stub/Skeleton Layer:* Đóng/mở gói tham số và xử lý ngoại lệ mạng (từ Java 1.2, skeleton phía server không còn bắt buộc, được thay bằng reflection).
  2. *Remote Reference Layer (RRL):* Quản lý ngữ nghĩa lời gọi/tham chiếu (ví dụ unicast).
  3. *Transport Layer:* Quản lý kết nối TCP.

---

## 4. TRAO ĐỔI THÔNG TIN HƯỚNG THÔNG ĐIỆP (MESSAGE-ORIENTED COMMUNICATION)

### 4.1. Thông điệp Tạm thời vs. Thông điệp Bền vững

* **Truyền thông điệp Tạm thời (Transient Messaging):** Thông điệp chỉ tồn tại khi cả ứng dụng gửi và ứng dụng nhận đang cùng chạy. Nếu bên nhận tắt hoặc mạng đứt tại thời điểm gửi, thông điệp bị hủy bỏ (Ví dụ: Berkeley Socket, MPI).
* **Truyền thông điệp Bền vững (Persistent Messaging):** Thông điệp được lưu trữ bền vững trên hệ thống trung gian (**Message Queue**, thường trên đĩa) cho đến khi bên nhận sẵn sàng lấy về. Cho phép bên gửi và bên nhận lệch pha về thời gian (Time Uncoupling).

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

Khi mạng mở rộng, các Message Broker liên kết với nhau thành mạng lưới router hàng đợi:

* Client A gửi bản tin vào `Send Queue` cục bộ.
* Mạng lưới router ($R1 \rightarrow R2$) chuyển tiếp bản tin dựa trên bảng định tuyến (ánh xạ tên hàng đợi đích sang router kế tiếp) mà ứng dụng không cần can thiệp.
* Bản tin rơi vào `Receive Queue` của Client B.

#### B. Message Broker & Chuyển đổi định dạng (Conversion Rules):

Đóng vai trò là "thông dịch viên" trung tâm giữa các ứng dụng viết bằng các công nghệ khác nhau:

* Tiếp nhận thông điệp từ Client gửi.
* Tra cứu luật biến đổi (Mapping Rules) trong kho tri thức (Repository).
* Chuyển đổi định dạng (ví dụ XML sang JSON hoặc Protobuf) trước khi đẩy sang đích.

#### C. Các mẫu phân phối tin nhắn kinh điển (Enterprise Patterns, theo thuật ngữ RabbitMQ):

1. **Work Queues (Competing Consumers):** Một hàng đợi, nhiều worker cùng lắng nghe; mỗi task chỉ được xử lý bởi **một worker duy nhất** (Round-robin hoặc Fair dispatch).
2. **Publish/Subscribe (Fanout Exchange):** Thông điệp được nhân bản và phát đồng loạt tới tất cả các hàng đợi (subscriber) đã gắn với exchange.
3. **Routing (Direct Exchange):** Định tuyến thông điệp tới hàng đợi dựa trên khớp chính xác **Routing Key**.
4. **Topics (Topic Exchange):** Định tuyến dựa trên mẫu so khớp với routing key dạng các từ cách nhau bởi dấu chấm; `*` khớp đúng một từ, `#` khớp không hoặc nhiều từ (ví dụ `*.logs`, `audit.#`).

---

## 5. TRAO ĐỔI THÔNG TIN HƯỚNG DÒNG (STREAM-ORIENTED COMMUNICATION)

### 5.1. Dữ liệu Liên tục & Các chế độ Truyền dòng

Khác với dữ liệu rời rạc (văn bản, số liệu), **Phương tiện truyền thông liên tục (Continuous Media - Audio/Video)** phụ thuộc nghiêm ngặt vào yếu tố **thời gian**:

* **Truyền không đồng bộ (Asynchronous Streaming):** Không có ràng buộc về thời điểm đến của các phần tử dữ liệu (ví dụ: tải file mp4 về máy trước rồi mới xem).
* **Truyền đồng bộ (Synchronous Streaming):** Quy định **độ trễ đầu-cuối tối đa** cho mỗi phần tử dữ liệu.
* **Truyền đẳng thời (Isochronous Streaming):** Quy định cả độ trễ **tối đa và tối thiểu** (độ trễ đầu-cuối bị chặn trong một khoảng), tức là jitter bị giới hạn. (Không đòi hỏi jitter bằng $0$.)

### 5.2. Chất lượng Dịch vụ (Quality of Service - QoS)

Các thông số then chốt quyết định trải nghiệm người dùng đối với dòng dữ liệu:

* **Throughput (Băng thông):** Tốc độ truyền tải bit/giây (bps).
* **End-to-End Delay:** Tổng thời gian từ lúc thu (camera/micro) đến khi phát ra phía client.
  * *Hội thoại tương tác thời gian thực (VoIP/Zoom):* theo khuyến nghị ITU-T G.114, độ trễ một chiều $< 150\text{ ms}$ là lý tưởng, $150\text{-}400\text{ ms}$ chấp nhận được với một số ứng dụng, $> 400\text{ ms}$ là không chấp nhận được cho hội thoại.
* **Jitter (Độ biến thiên trễ):** Sự chênh lệch thời gian đến giữa các gói tin liên tiếp.

#### Kỹ thuật Playout Buffer để hấp thụ Jitter:

Mạng IP có bản chất Best-effort $\rightarrow$ gói tin đến máy khách với thời gian không đều.

* **Giải pháp:** Client không phát ngay khi nhận được gói, mà giữ trong một vùng nhớ đệm (**Playout/Jitter Buffer**) trong khoảng $\Delta t$, rồi phát ra theo lịch đều đặn.
* **Chọn $\Delta t$ là sự đánh đổi giữa độ mượt và độ trễ:**
  * Streaming một chiều (video on demand, livestream): có thể đệm vài giây.
  * Hội thoại tương tác (VoIP, họp trực tuyến): chỉ đệm khoảng vài chục đến vài trăm ms (thường dùng adaptive jitter buffer), nếu không sẽ vượt ngân sách độ trễ $150\text{ ms}$.
* Gói đến sau thời điểm phát dự kiến vẫn bị coi như mất.

```
Nguồn phát:        [1]    [2]    [3]    [4]    [5]      (cách đều)
                     \      \      \      \      \
Mạng IP:              \      \      \      \      \    (mỗi gói trễ khác nhau)
                       v      v      v      v      v
Đến client:           [1]  [2]   [3]        [4] [5]     (không đều)
                        \   |    /           |   /
Playout buffer:        giữ lại thêm Δt để "san phẳng"
                        |    |    |    |    |
Phát ra màn hình:      [1]  [2]  [3]  [4]  [5]          (đều đặn, trễ thêm Δt)
```

#### Che giấu mất gói: Forward Error Correction (FEC) & Interleaving

Trong truyền dòng thời gian thực, cơ chế truyền lại (ARQ/ACK) của TCP thường không phù hợp vì quá trễ. Thay vào đó dùng UDP (thường qua RTP) kết hợp:

* **FEC (Forward Error Correction):** Gửi kèm dữ liệu dư thừa (ví dụ gói parity XOR, mã Reed-Solomon) để bên nhận tự khôi phục gói bị mất mà không cần xin gửi lại. Cái giá: tốn thêm băng thông.
* **Interleaved Transmission (Truyền đan xen):** Thay vì gửi tuần tự cụm frame $[1, 2, 3, 4][5, 6, 7, 8]$, bộ mã hóa xáo trộn thứ tự các đơn vị dữ liệu sang các gói khác nhau, ví dụ $[1, 5, 9, 13][2, 6, 10, 14]$.
  * *Hệ quả:* Khi mất một gói, phần dữ liệu mất không liên tục mà rải rác thành nhiều khoảng trống ngắn, dễ che giấu bằng nội suy hơn là mất một đoạn video liên tiếp.
  * *Cái giá:* Tăng độ trễ (phải đợi đủ dữ liệu để xáo trộn/giải xáo trộn).

### 5.3. Đồng bộ hóa Dòng (Stream Synchronization)

* **Lip Synchronization (Đồng bộ khẩu hình):** Đồng bộ giữa dòng hình ảnh (Video stream) và dòng âm thanh (Audio stream) thường được truyền qua các luồng/socket riêng. Thực tế dùng **timestamp** (RTP timestamp kết hợp RTCP Sender Report) để căn chỉnh hai luồng phía nhận.
* Ngưỡng dung sai của con người (khuyến nghị ITU-R BT.1359): người xem bắt đầu nhận ra lệch khi âm thanh **sớm hơn** hình khoảng $45\text{ ms}$ hoặc **muộn hơn** hình khoảng $125\text{ ms}$; ngưỡng còn chấp nhận được khoảng $+90\text{ ms}$ / $-185\text{ ms}$. Quy tắc ngón tay cái hay dùng trong giáo trình: độ lệch (skew) trong khoảng $\pm 80\text{ ms}$.
* Con người nhạy hơn khi âm thanh đến *trước* hình so với khi âm thanh đến *sau* hình (vì ngoài đời ánh sáng luôn đến trước âm thanh).

---

## 6. CÂU HỎI TỰ VẤN & BÀI TOÁN THIẾT KẾ THỰC CHIẾN

### 6.1. Câu hỏi bẫy tư duy (Active Recall)

**Câu 1:** *Khi lập trình RPC qua mạng diện rộng (WAN), tại sao lập trình viên không nên cố gắng giấu hoàn toàn tính chất phân tán (100% Access Transparency)?*

* **Giải thích:** Một lời gọi hàm cục bộ hoàn thành trong cỡ nano-giây đến micro-giây và (nếu mã đúng) không gặp lỗi mạng. Lời gọi qua WAN mất hàng chục đến hàng trăm mili-giây và có thể thất bại bất cứ lúc nào (đứt cáp, sập router, timeout, server chết giữa chừng), thậm chí client không biết server đã thực thi hay chưa. Nếu cố tạo cảm giác như hàm cục bộ mà không bộc lộ cơ chế xử lý lỗi (timeout, exception, retry với thao tác idempotent), ứng dụng sẽ không lường trước các trường hợp lỗi, dẫn tới treo luồng và cạn tài nguyên (thread starvation). Đây chính là hai "ảo tưởng" mạng tin cậy và độ trễ bằng không ở Chương 1 (xem thêm bài "A Note on Distributed Computing", Waldo và cộng sự, 1994).

**Câu 2:** *Tại sao nhiều hệ thống giao dịch tài chính (Core Banking) ưu tiên dùng Kiến trúc Hướng thông điệp bền vững (MOM) cho các luồng tích hợp thay vì chỉ gọi trực tiếp RPC/gRPC giữa các dịch vụ?*

* **Giải thích:** Vì tính sẵn sàng và khả năng chịu lỗi. Với RPC đồng bộ, nếu dịch vụ nhận tiền bị gián đoạn lúc dịch vụ chuyển tiền đang gọi, giao dịch có thể thất bại hoặc rơi vào trạng thái không nhất quán (đã trừ tiền nhưng chưa cộng tiền). Với MOM, giao dịch được ghi bền vững vào hàng đợi; kể cả khi hệ thống đích tạm sập, thông điệp vẫn nằm an toàn và được xử lý khi hệ thống phục hồi (Time Uncoupling).
* **Lưu ý:** Hầu hết MOM chỉ đảm bảo **ít nhất một lần (At-least-once)**, nghĩa là có thể giao trùng. Do đó phía nhận phải **idempotent** (hoặc khử trùng lặp bằng message ID) để đạt hiệu ứng "xử lý đúng một lần". Thứ tự xử lý chỉ được bảo đảm trong phạm vi một hàng đợi/phân vùng với một consumer. Thực tế các ngân hàng kết hợp cả RPC (cho tác vụ cần phản hồi tức thì) và MOM (cho tích hợp bất đồng bộ), kèm các mẫu như Saga hoặc giao dịch phân tán.

### 6.2. Bài toán Thiết kế Thực tế (System Design Scenario)

**Đề bài:** Hãy thiết kế kiến trúc truyền thông cho hệ thống ứng dụng họp trực tuyến (như Zoom / Google Meet) cho 100 người trong cùng một phòng họp.

**Giải pháp kiến trúc áp dụng kiến thức Chương 2:**

1. **Tín hiệu điều khiển (Signaling / Room Management):**
   * Sử dụng **RPC (ví dụ gRPC/HTTP) hoặc WebSocket trên nền TCP**: quản lý trạng thái phòng họp, danh sách người tham gia, bật/tắt micro, trao đổi thông tin phiên (SDP, ICE candidates). Dữ liệu nhỏ, cần độ tin cậy và đúng thứ tự (Reliable).

2. **Luồng truyền thông thời gian thực (Media Plane):**
   * Dùng **UDP** (thường qua RTP/SRTP trong WebRTC), không dùng TCP, vì truyền lại gói cũ tạo độ trễ không chấp nhận được (head-of-line blocking).
   * Mục tiêu độ trễ một chiều $< 150\text{ ms}$ cho âm thanh (đẳng thời/đồng bộ ở mức mềm).

3. **Topology cho 100 người:**
   * **Mesh P2P** không khả thi: mỗi người phải gửi và nhận khoảng $99$ luồng, tổng số kết nối $\approx n(n-1)/2$ (khoảng $4950$), băng thông upload của client không đủ.
   * **MCU** (trộn tất cả luồng thành một luồng): tiết kiệm băng thông client nhưng server phải giải mã/mã hóa nên rất tốn CPU.
   * **SFU (Selective Forwarding Unit):** mỗi client gửi 1 luồng lên server, server chuyển tiếp có chọn lọc các luồng cần thiết tới người nhận (ví dụ chỉ chuyển video của vài người đang nói/được ghim). Đây là lựa chọn phổ biến. Có thể kết hợp **simulcast/SVC** để cung cấp nhiều chất lượng cho từng người nhận, và nhiều SFU phân tán theo vùng địa lý (ý tưởng edge server của Chương 1).

4. **Chất lượng dịch vụ và chống lỗi mạng:**
   * **Adaptive jitter buffer** ở client (vài chục đến vài trăm ms), không phải bộ đệm vài giây.
   * Âm thanh dùng **FEC** (ví dụ in-band FEC của Opus) để che giấu mất gói; video dùng **NACK/retransmit chọn lọc** khi RTT thấp, hoặc yêu cầu keyframe.
   * **Điều khiển nghẽn** thích ứng (ví dụ transport-wide congestion control) để giảm bitrate khi mạng xấu.
   * Vượt NAT/firewall bằng **ICE/STUN/TURN**.

5. **Đồng bộ dòng:**
   * Căn chỉnh âm thanh - hình ảnh của từng người nói bằng **RTP timestamp + RTCP Sender Report**, giữ độ lệch trong ngưỡng khoảng $\pm 80\text{ ms}$.

6. **Dịch vụ phụ trợ (chat, thông báo, ghi âm/ghi hình, lưu biên bản):**
   * Dùng **Message Queue bền vững / Pub-Sub** (RabbitMQ, Kafka...) để tách rời các dịch vụ: sự kiện "cuộc họp kết thúc" được phát tới dịch vụ ghi hình, dịch vụ gửi email biên bản, dịch vụ thống kê. Nhờ Time Uncoupling, dịch vụ nào tạm ngưng vẫn không mất sự kiện.

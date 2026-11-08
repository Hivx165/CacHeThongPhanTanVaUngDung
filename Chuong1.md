# BÀI GIẢNG CHUYÊN SÂU: CHƯƠNG 1 - TỔNG QUAN VÀ KIẾN TRÚC HỆ PHÂN TÁN
*Môn học: Các hệ thống phân tán và ứng dụng (Distributed Systems & Applications)*  
*Khoa Công nghệ Thông tin & Truyền thông - ĐH Bách Khoa Hà Nội (HUST)*

---

## 1. BẢN ĐỒ KHÁI NIỆM & FIRST PRINCIPLES (NGUYÊN LÝ GỐC)

### 1.1. Tại sao hệ thống đơn lẻ (Monolithic / Single-node) thất bại?
Trước khi hệ phân tán ra đời, sự phát triển của hệ thống máy tính dựa vào việc nâng cấp một máy chủ vật lý duy nhất (**Scale-up / Vertical Scaling**). Tuy nhiên, phương pháp này chạm trần bởi các rào cản vật lý:

1. **Giới hạn bán dẫn & Tản nhiệt:** Định luật Moore chậm lại; không thể tăng xung nhịp CPU mãi mãi do rào cản tiêu tán nhiệt năng và chi phí sản xuất chip nguyên khối cực kỳ đắt đỏ.
2. **Điểm nghẽn lỗi đơn (Single Point of Failure - SPOF):** Bất kể máy chủ đắt tiền đến đâu, một sự cố mất nguồn, cháy bo mạch hay hỏng RAM cục bộ đều khiến toàn bộ dịch vụ ngưng trệ.
3. **Độ trễ vật lý (Speed of Light):** Một máy chủ đặt tại Hà Nội không thể phản hồi người dùng tại New York với độ trễ dưới $10\text{ ms}$ do giới hạn vận tốc ánh sáng truyền qua cáp quang ($\approx 200.000\text{ km/s}$).

**Giải pháp tất yếu:** Chuyển sang mô hình **Scale-out (Horizontal Scaling)**: Kết hợp hàng nghìn cụm máy tính thông thường (commodity hardware), liên kết qua mạng truyền thông để cùng gánh vác tải tính toán và lưu trữ.

---

### 1.2. Định nghĩa Hệ Thống Phân Tán (Distributed System)

> **Định nghĩa của Andrew S. Tanenbaum:**  
> *"Hệ thống phân tán là một tập hợp các máy tính độc lập liên kết với nhau qua mạng, nhưng thể hiện trước mắt người dùng như một hệ thống đơn nhất và nhất quán (single coherent system)."*

> **Định nghĩa kinh điển của Leslie Lamport:**  
> *"Hệ phân tán là hệ thống mà lỗi của một chiếc máy tính bạn còn chẳng biết có tồn tại lại có thể làm chiếc máy tính của chính bạn sập."*

#### Ẩn dụ trực quan: Quán ăn 1 đầu bếp vs. Dây chuyền bếp công nghiệp
* **Hệ thống đơn (Single Machine):** Một siêu đầu bếp làm tất cả: nhận order, thái thịt, nấu ăn, tính tiền. Khi khách đông gấp 10 lần $\rightarrow$ đầu bếp kiệt sức (quá tải CPU/RAM). Khi đầu bếp ốm $\rightarrow$ quán đóng cửa (SPOF).
* **Hệ thống phân tán (Distributed System):** 50 đầu bếp bình thường, mỗi người phụ trách một công đoạn, giao tiếp với nhau bằng cách ném các mẩu order giấy (truyền thông báo - *Message Passing*). Người mua hàng bên ngoài chỉ nhìn thấy quầy nhận món thống nhất, không cần biết bên trong có bao nhiêu đầu bếp và ai vừa đổi ca.

---

## 2. PHÂN LOẠI HỆ THỐNG MÁY TÍNH

Trải nghiệm của người dùng và lập trình viên đối với sự hiện diện của phần cứng:

```
+---------------------------+-----------------------------------------------+
| Loại hệ thống             | Biểu hiện trước mắt người dùng               |
+---------------------------+-----------------------------------------------+
| Networked System          | Nhận biết rõ ràng nhiều máy tính riêng rẽ     |
| (Mạng máy tính thông thường)| (Ví dụ: dùng SSH kết nối vào IP 192.168.1.10)  |
+---------------------------+-----------------------------------------------+
| Distributed System        | Xuất hiện như một cỗ máy duy nhất             |
| (Hệ thống phân tán)       | (Ví dụ: Tìm kiếm trên Google, cụm Cassandra)  |
+---------------------------+-----------------------------------------------+
| Ubiquitous System         | Máy tính vô hình, hòa tan vào môi trường vật lý|
| (Điện toán phổ biến / IoT)| (Ví dụ: Hệ thống nhà thông minh tự chỉnh nhiệt)|
+---------------------------+-----------------------------------------------+
```

---

## 3. CÁC ĐẶC TRƯNG CỐT LÕI CỦA HỆ PHÂN TÁN

### 3.1. Chia sẻ tài nguyên (Resource Sharing)
* Cho phép nhiều tiến trình dùng chung phần cứng (CPU, Storage, Network) và tài nguyên logic (Cơ sở dữ liệu, dịch vụ file).
* **Hệ quả tiêu cực:** Làm nở rộng bề mặt tấn công bảo mật (Attack Surface) và phức tạp hóa bài toán kiểm soát truy cập phân tán.

---

### 3.2. Tính trong suốt (Distribution Transparency)
Mục tiêu là che giấu sự phân tán về mặt vật lý trước người dùng và ứng dụng:

| Phân loại trong suốt | Bản chất kỹ thuật | Ví dụ minh họa |
| :--- | :--- | :--- |
| **Truy cập (Access)** | Che giấu sự khác biệt về biểu diễn dữ liệu (Endianness, Padding) và cách gọi hàm (Local Call vs Remote Call). | Gọi hàm RPC `calc.add(a, b)` như gọi hàm nội tại trong cùng tiến trình. |
| **Địa điểm (Location)** | Che giấu vị trí vật lý / địa chỉ IP của tài nguyên. | Truy cập `https://mysite.com/report.pdf` thay vì chỉ định rõ máy chủ `10.0.0.5:/var/data/`. |
| **Di trú (Migration)** | Che giấu việc tài nguyên chuyển sang máy khác khi đang rảnh rỗi. | Chuyển file tĩnh từ server A sang server B trong nội bộ cụm lưu trữ NFS. |
| **Chuyển địa điểm (Relocation)** | Che giấu việc tài nguyên di chuyển ngay trong lúc đang được sử dụng. | Thiết bị di động chuyển vùng trạm BTS/Wi-Fi mà cuộc gọi VoIP không bị ngắt. |
| **Sao lưu (Replication)** | Che giấu việc dữ liệu tồn tại ở nhiều bản sao dự phòng. | Ghi một bản ghi vào cụm Raft/Kafka, hệ thống ngầm đồng bộ sang 3 node mà client không cần gửi 3 lần. |
| **Tương tranh (Concurrency)** | Che giấu việc nhiều client đồng thời truy xuất tài nguyên chung. | Hai giao dịch ngân hàng rút tiền đồng thời được quản lý bằng Distributed Lock, không gây sai lệch số dư. |
| **Thứ lỗi (Failure)** | Che giấu sự cố sập node mạng và quá trình tự phục hồi. | Một node Worker chết, Kubernetes tự điều phối Pod sang node khác mà người dùng chỉ nhận trễ nhẹ. |
| **Bền vững (Persistence)** | Che giấu việc tài nguyên nằm trên RAM hay đã ghi bền vững xuống ổ cứng. | Hệ thống Cache ghi trễ (Write-back / Write-behind) xuống Database. |

---

### 3.3. Tính mở (Openness)
* Cung cấp các giao diện dịch vụ được mô tả bằng ngôn ngữ chuẩn hóa (Interface Definition Language - IDL) như Protobuf, OpenAPI, Thrift.
* **Tính khả chuyển (Portability):** Ứng dụng chạy được trên nhiều hệ điều hành và phần cứng khác nhau mà không cần sửa code.
* **Khả năng tương tác (Interoperability):** Các thành phần viết bằng ngôn ngữ khác nhau (Go, Java, Python) vẫn giao tiếp liền mạch qua giao thức chuẩn.
* **Tách biệt cơ chế và chính sách (Separation of Mechanism and Policy):**
  * *Cơ chế (Mechanism):* Cung cấp khả năng lưu trữ bộ đệm (Caching).
  * *Chính sách (Policy):* Cho phép người dùng quyết định dữ liệu được lưu đệm trong bao lâu ($TTL$) và giải thuật loại bỏ (LRU hay LFU).

---

### 3.4. Tính co giãn (Scalability)
Hệ thống phải duy trì hiệu năng khi đối mặt với sự mở rộng theo 3 chiều:

```
                      [Không gian Địa lý]
                     (LAN -> WAN, Độ trễ cao,
                      Giao thức Asynchronous)
                                ^
                                |
                                |
[Quy mô Node/User] <------------+------------> [Miền Quản trị]
(Triệt tiêu SPOF, Bottlenecks)                  (Multi-domain, Cross-trust,
                                                 Chính sách bảo mật xung đột)
```

#### Các điểm nghẽn của mô hình tập trung khi mở rộng quy mô:
* **Dịch vụ tập trung (Centralized Services):** Một máy chủ xác thực gánh toàn bộ lượt đăng nhập.
* **Dữ liệu tập trung (Centralized Data):** Toàn bộ hệ thống cùng đọc/ghi vào một database duy nhất.
* **Thuật toán tập trung (Centralized Algorithms):** Thuật toán yêu cầu một node phải thu thập trạng thái của toàn bộ hệ thống mới đưa ra quyết định.

---

## 4. PHÂN LOẠI KIẾN TRÚC PHẦN MỀM (SOFTWARE ARCHITECTURES)

### 4.1. Kiến trúc phân tầng (Layered Architecture)
* **Nguyên lý:** Chia nhỏ bài toán thành các lớp trừu tượng; tầng trên gọi xuống tầng dưới kề cận qua giao diện rõ ràng. Luồng yêu cầu (Request) đi từ trên xuống dưới, luồng kết quả (Response) đi ngược từ dưới lên.
* **Ví dụ mô hình 3 tầng (3-Tier):**
  1. *User Interface Layer (Presentation):* Web browser, Mobile UI.
  2. *Processing Layer (Application/Logic):* Xử lý nghiệp vụ, kiểm tra ràng buộc logic, thuật toán xếp hạng.
  3. *Data Layer:* Hệ quản trị cơ sở dữ liệu (RDBMS, NoSQL, File system).

---

### 4.2. Kiến trúc hướng đối tượng (Object-Based Architecture)
* Các thành phần phần mềm được đóng gói thành các Object kết nối lỏng với nhau qua cơ chế gọi phương thức từ xa (Remote Method Invocation - RMI / RPC).
* Sử dụng bộ chuyển đổi:
  * **Stub (phía Client):** Đóng gói tham số thành chuỗi byte qua mạng (*Marshalling*).
  * **Skeleton (phía Server):** Giải mã tham số (*Unmarshalling*) và gọi hàm đối tượng tương ứng.
* Điển hình: Tiêu chuẩn CORBA (Common Object Request Broker Architecture), Java RMI.

---

### 4.3. Kiến trúc hướng sự kiện (Event-Based Architecture)
* Mô hình **Publish/Subscribe**: Các thành phần giao tiếp gián tiếp thông qua trục phân phối sự kiện (Event Bus / Message Broker).
* **Đặc tính tách rời (Decoupling):**
  * *Tách rời không gian (Space uncoupling):* Publisher không cần biết Subscriber là ai, IP nào.
  * *Tách rời thời gian (Time uncoupling):* Publisher có thể phát ra tin nhắn ngay cả khi Subscriber đang offline; tin nhắn được lưu trữ trong hàng đợi để xử lý sau.

---

### 4.4. Kiến trúc hướng dữ liệu (Data-Centered / Shared Data Space)
* Các tiến trình độc lập không gọi trực tiếp lẫn nhau mà giao tiếp thông qua việc đọc/ghi dữ liệu vào một không gian lưu trữ chung bền vững (Tuple Space / Blackboard).
* Tách rời cả về định danh lẫn thời gian.

---

### 4.5. Kiến trúc Microservices & Container Orchestration
* Chuyển đổi từ Monolithic (nguyên khối) sang tập hợp các dịch vụ nhỏ gọn, mỗi dịch vụ chạy trên tiến trình riêng và triển khai độc lập.
* **Ưu điểm:** Cách ly lỗi (Fault isolation), linh hoạt công nghệ, mở rộng độc lập từng thành phần.
* **Thách thức:** Quản lý hàng trăm container $\rightarrow$ Cần công cụ điều phối (**Container Orchestration**) như Kubernetes (K8s), Docker Swarm.

#### Sơ đồ cấu trúc điều phối Pod cơ bản trong Kubernetes:

```
+-------------------------------------------------------------------+
|                           MASTER NODE                             |
|  [ API Server ] <---> [ K8s Cluster Services / Scheduler / etcd ] |
+-------------------------------------------------------------------+
             |                                  |
             v                                  v
+--------------------------+       +--------------------------+
|       WORKER NODE 1      |       |       WORKER NODE 2      |
|  [ Kubelet / K-Proxy ]   |       |  [ Kubelet / K-Proxy ]   |
|  +--------------------+  |       |  +--------------------+  |
|  | Pod 1 (Replication)|  |       |  | Pod 1 (Replication)|  |
|  +--------------------+  |       |  +--------------------+  |
|  | Pod 2              |  |       |  | Pod 2 (Replication)|  |
|  +--------------------+  |       |  +--------------------+  |
+--------------------------+       +--------------------------+
```

---

## 5. KIẾN TRÚC HỆ THỐNG (SYSTEM ARCHITECTURES & TOPOLOGIES)

### 5.1. Kiến trúc Tập trung: Mô hình Client-Server
Phân chia vai trò rõ ràng giữa 2 phía:

```
        +--------------+
        |    Server    | (Lắng nghe cổng, cung cấp dịch vụ)
        +--------------+
          ^    ^     ^
          |    |     |   Giao thức: Hướng nối (TCP) hoặc Không nối (UDP)
    +-----+    |     +-----+
    |          |           |
+--------+ +--------+ +--------+
|Client 1| |Client 2| |Client N| (Gửi request, chờ đợi, hiển thị UI)
+--------+ +--------+ +--------+
```

#### Phân loại Client 2-Tier:
* **Thin Client (Client mỏng):** Client chỉ chạy giao diện người dùng. Mọi tính toán và lưu trữ đều thực hiện trên Server (Ví dụ: Terminal dumb, Google Stadia streaming game).
* **Fat/Rich Client (Client béo):** Client tải phần lớn logic ứng dụng về máy cục bộ, server chủ yếu làm nhiệm vụ lưu trữ cơ sở dữ liệu.

---

### 5.2. Mạng Ngang Hàng (Peer-to-Peer - P2P)

Mọi node tham gia mạng đều có quyền và nghĩa vụ ngang hàng (**Servent = Server + Client**). Truyền thông dựa trên **Mạng lớp phủ (Overlay Network)**: một mạng logic trong đó các liên kết logic chạy xuyên qua các bộ định tuyến IP vật lý.

```
       MẠNG LỚP PHỦ (Overlay Network - Logic)
             [A] ----------- [B]
              | \             |
              |   \           |
              |     \         |
             [C] ----\------ [D]
             ===================
       MẠNG HẠ TẦNG VẬT LÝ (Physical IP Routers)
         (r1) --- (r2) --- (r3) --- (r4)
```

#### A. P2P Không có cấu trúc (Unstructured P2P)
* Topology hình thành ngẫu nhiên (Random Graph).
* Khi cần tìm file: Dùng giải thuật **Flooding** (quảng bá đến toàn bộ láng giềng trong bán kính $TTL$) hoặc dùng **Superpeer** (chọn node mạng có băng thông lớn và uptime cao làm trạm chỉ mục trung gian).

#### B. P2P Có cấu trúc (Structured P2P / Distributed Hash Table - DHT)
Topology được định hướng theo công thức toán học chính xác, tiêu biểu là giải thuật **Chord DHT**.

---

### 5.3. Giải thuật Chi Tiết: Vòng Định Tuyến Chord DHT

#### 1. Nguyên lý phân bổ:
* Sử dụng cùng một hàm băm $m$-bit (ví dụ $m=4$, không gian định danh là vòng tròn $0$ đến $2^m - 1 = 15$).
* **Gán ID Node:** $\text{Node ID} = \text{Hash}(\text{IP:Port}) \pmod{2^m}$
* **Gán ID Khóa (Key):** $\text{Key ID} = \text{Hash}(\text{Data Key}) \pmod{2^m}$
* **Quy tắc gán vị trí:** Khóa $k$ được lưu trữ tại node đầu tiên có ID $\ge k$ theo chiều kim đồng hồ. Node đó gọi là kế nhiệm của $k$: $\text{Successor}(k)$.

#### 2. Minh họa không gian vòng tròn Chord ($m=4$, giá trị từ $0 \rightarrow 15$):

```
                       (Node 0)
                  15              1 (Node 1)
             14                        2
          13                              3
      (Node 12)                        (Node 4)
          11                              5
             10                        6
                   9              7 (Node 7)
                       (Node 8)
```

Giả sử hệ thống có 5 node thực tế đang hoạt động: $N = \{1, 4, 7, 8, 12\}$.
* Phân phối các khóa dữ liệu $k$:
  * $k = 1 \rightarrow \text{Successor}(1) = 1 \Rightarrow \text{Lưu tại Node 1}$
  * $k = 2, 3, 4 \rightarrow \text{Successor}(k) = 4 \Rightarrow \text{Lưu tại Node 4}$
  * $k = 5, 6, 7 \rightarrow \text{Successor}(k) = 7 \Rightarrow \text{Lưu tại Node 7}$
  * $k = 8 \rightarrow \text{Successor}(8) = 8 \Rightarrow \text{Lưu tại Node 8}$
  * $k = 9, 10, 11, 12 \rightarrow \text{Successor}(k) = 12 \Rightarrow \text{Lưu tại Node 12}$
  * $k = 13, 14, 15, 0 \rightarrow \text{Successor}(k) = 1 \Rightarrow \text{Lưu tại Node 1}$ (vòng quay đầu)

#### 3. Bảng Finger Table (Tăng tốc tìm kiếm từ $O(N)$ về $O(\log N)$):
Mỗi node $p$ duy trì một bảng chuyển tiếp gồm $m$ hàng. Mục thứ $i$ ($1 \le i \le m$) trỏ tới:

$$\text{Finger}[i] = \text{Successor}((p + 2^{i-1}) \pmod{2^m})$$

##### Ví dụ: Thiết lập Finger Table cho Node $p = 4$ với $m = 4$:
* $i = 1$: $\text{Successor}((4 + 2^0) \pmod{16}) = \text{Successor}(5) = 7$
* $i = 2$: $\text{Successor}((4 + 2^1) \pmod{16}) = \text{Successor}(6) = 7$
* $i = 3$: $\text{Successor}((4 + 2^2) \pmod{16}) = \text{Successor}(8) = 8$
* $i = 4$: $\text{Successor}((4 + 2^3) \pmod{16}) = \text{Successor}(12) = 12$

$$\text{Bảng Finger Table của Node 4: } [7, 7, 8, 12]$$

Khi Node 4 nhận được yêu cầu tìm khóa $k = 11$:
* Node 4 kiểm tra bảng: $11$ nằm giữa mục thứ 3 (Node 8) và mục thứ 4 (Node 12).
* Node 4 chuyển tiếp gói tin trực tiếp sang Node 8 (node lớn nhất nhưng nhỏ hơn 11). Bước nhảy thu hẹp $1/2$ không gian tìm kiếm tương tự như Binary Search.

---

### 5.4. Kiến trúc Hỗn hợp (Hybrid Architectures)
* **Hệ thống Máy chủ Biên (Edge-Server Systems / CDN):**
  * Máy chủ gốc (Origin Server) lưu trữ toàn bộ dữ liệu nguồn.
  * Các Edge Server (như Akamai, Cloudflare) đặt phân tán tại các ISP biên để cache nội dung tĩnh sát với người dùng cuối, giảm thiểu tải cho mạng lõi Internet và giảm triệt để độ trễ.
* **Hệ phân tán hợp tác (Ví dụ: BitTorrent):**
  * Dùng máy chủ trung tâm (**Tracker**) hoặc bảng phân tán (**DHT**) để quản lý danh sách các node trong mạng (*Swarm*).
  * Việc tải/chia sẻ các mảnh file nhỏ (*chunks*) diễn ra hoàn toàn bằng mạng ngang hàng P2P giữa các máy *Seeder* và *Leecher*.

---

## 6. MIDDLEWARE TRONG HỆ PHÂN TÁN

### 6.1. Định vị tầng Middleware
Middleware là lớp phần mềm trung gian nằm giữa ứng dụng phân tán và hệ điều hành mạng cục bộ:

```
+--------------------------------------------------------------+
|            ỨNG DỤNG PHÂN TÁN (Distributed Applications)      |
+--------------------------------------------------------------+
|                    TẦNG MIDDLEWARE                           |
|  - Cung cấp tính trong suốt phân tán (Transparency)          |
|  - Giao tiếp bậc cao (RPC / RMI / Message Queue)             |
|  - Dịch vụ định danh (Naming: DNS, LDAP, Consul)             |
|  - Dịch vụ giao dịch phân tán & Lưu trữ bền vững             |
+--------------------------------------------------------------+
|       HỆ ĐIỀU HÀNH MẠNG CỤC BỘ (NOS - Linux / Windows / BSD) |
+--------------------------------------------------------------+
|                       HẠ TẦNG MẠNG (TCP/IP)                  |
+--------------------------------------------------------------+
```

---

### 6.2. Bảng so sánh DOS, NOS và Middleware

| Tiêu chí | Hệ điều hành Phân tán (DOS) | Hệ điều hành Mạng (NOS) | Hệ thống trên Middleware |
| :--- | :--- | :--- | :--- |
| **Mức độ trong suốt** | Rất cao (Xem toàn bộ cụm máy như 1 máy đơn) | Thấp (Người dùng phải biết rõ danh tính máy đích) | Cao (Đạt được ở tầng ứng dụng) |
| **Hệ điều hành trên các nút** | Bắt buộc phải đồng nhất (Plan9, Amoeba) | Có thể khác nhau | Độc lập hoàn toàn với HĐH bên dưới |
| **Số lượng bản sao OS** | Một hệ điều hành dùng chung toàn cụm | Mỗi nút một bản HĐH riêng lẻ | Mỗi nút một HĐH riêng lẻ |
| **Cơ chế trao đổi thông tin** | Bộ nhớ chia sẻ (Shared Memory) / Msg Passing | Trao đổi qua File hoặc Raw Socket | Lời gọi hàm từ xa (RPC), Hàng đợi tin nhắn |
| **Quản trị tài nguyên** | Toàn cục tập trung hoặc phân tán | Cục bộ theo từng nút | Phân tán ở mức dịch vụ ứng dụng |
| **Khả năng co giãn** | Kém (Tối đa vài trăm CPU) | Khá tốt | Rất lớn (Hàng triệu nút) |
| **Tính chất hệ thống** | Đóng (Tightly-coupled) | Mở (Loosely-coupled) | Mở & Mềm dẻo (Open & Extensible) |

---

### 6.3. Cơ chế Chặn (Interceptors Pattern)
Để đảm bảo tính mềm dẻo mà không làm phá vỡ cấu trúc code ứng dụng, Middleware sử dụng mẫu thiết kế **Interceptor**:
* Khi Client gọi hàm `B.do_something(value)`, lời gọi hàm bị **Request-level Interceptor** chặn lại.
* Interceptor có thể inject thêm mã xử lý: Ghi log, nén dữ liệu, kiểm tra quyền bảo mật, hoặc chèn mã Trace ID.
* Ở tầng dưới, **Message-level Interceptor** chặn gói tin trước khi gửi xuống socket để tiến hành mã hóa TLS hoặc định tuyến lại nếu đường truyền lỗi.

---

## 7. 8 ẢO TƯỞNG CỦA LẬP TRÌNH VIÊN (FALLACIES OF DISTRIBUTED COMPUTING)

Bất kỳ hệ thống phân tán nào gặp sự cố nghiêm trọng đều xuất phát từ việc lập trình viên ngầm mặc định 8 giả định sai lầm sau:

1. **Mạng luôn tin cậy (The network is reliable):** Thực tế: Gói tin luôn có thể bị rớt, trễ vô hạn hoặc trùng lặp.
2. **Độ trễ bằng không (Latency is zero):** Thực tế: Gọi qua mạng luôn chậm hơn gọi bộ nhớ cục bộ từ $10^3$ đến $10^6$ lần.
3. **Băng thông là vô hạn (Bandwidth is infinite):** Thực tế: Nghẽn băng thông xảy ra thường xuyên khi truyền tải dữ liệu lớn.
4. **Mạng luôn an toàn (The network is secure):** Thực tế: Dữ liệu truyền qua mạng công cộng luôn có nguy cơ bị nghe lén, giả mạo.
5. **Cấu trúc mạng không đổi (Topology doesn't change):** Thực tế: Router chết, server khởi động lại, IP thay đổi diễn ra liên tục.
6. **Chỉ có một người quản trị duy nhất (There is one administrator):** Thực tế: Hệ thống phân tán lớn luôn vận hành xuyên biên giới với nhiều đội ngũ có chính sách khác nhau.
7. **Chi phí vận chuyển bằng không (Transport cost is zero):** Thực tế: Đóng gói và giải mã dữ liệu (Serialization/Deserialization) tốn năng lực CPU đáng kể.
8. **Mạng là đồng nhất (The network is homogeneous):** Thực tế: Hệ thống gồm vô số thiết bị từ máy chủ đa nhân đến thiết bị nhúng chạy các OS khác nhau.

---

## 8. BÀI TẬP TỰ ĐÁNH GIÁ (ACTIVE RECALL)

### Câu hỏi bản chất
**Câu 1:** *Tại sao việc đạt được 100% độ trong suốt về lỗi (Failure Transparency) trong hệ thống phân tán là bất khả thi?*  
* **Giải đáp:** Để che giấu hoàn toàn việc một máy chủ bị sập, hệ thống buộc phải liên tục gửi lại request (infinite retry) và chờ đợi. Trong mạng không đồng bộ (Asynchronous Network), không thể phân biệt giữa một máy chủ chạy cực kỳ chậm với một máy chủ đã chết hẳn. Chờ đợi vô hạn để giấu lỗi sẽ khiến tiến trình client bị treo vĩnh viễn, đánh đổi lại bằng việc phá hủy hoàn toàn hiệu năng và tính sẵn sàng của hệ thống.

**Câu 2:** *Tại sao các hệ thống lưu trữ phân tán hiện đại như Cassandra hay DynamoDB lại sử dụng kỹ thuật Consistent Hashing (biến thể của Chord DHT) thay vì hàm băm modulo thông thường $\text{hash}(key) \pmod N$?*  
* **Giải đáp:** Với hàm băm $\text{hash}(key) \pmod N$, khi số lượng node $N$ thay đổi (thêm node mới hoặc 1 node bị chết), hầu hết các khóa sẽ bị đổi vị trí vì mẫu số thay đổi, gây ra hiện tượng dịch chuyển dữ liệu hàng loạt trên toàn bộ cụm (*Cache Stampede*). Ngược lại, trên vòng Chord/Consistent Hashing, việc thêm hoặc bớt một node chỉ ảnh hưởng và tái định tuyến một phần nhỏ khóa kề cận ($O(K/N)$ khóa), giữ cho phần lớn hệ thống không bị xáo trộn.
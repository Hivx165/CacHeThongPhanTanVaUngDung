# BÀI GIẢNG CHUYÊN SÂU: CHƯƠNG 3 - ĐỊNH DANH TRONG HỆ PHÂN TÁN

*Môn học: Các hệ thống phân tán và ứng dụng (Distributed Systems & Applications)*
*Khoa Công nghệ Thông tin & Truyền thông - ĐH Bách Khoa Hà Nội (HUST)*

---

## 1. NGUYÊN LÝ GỐC: TÊN (NAME), ĐỊNH DANH (IDENTIFIER) VÀ ĐỊA CHỈ (ADDRESS)

Trong một hệ thống phân tán, các thực thể (entities - tiến trình, tệp tin, máy chủ, dịch vụ, trang web) nằm rải rác trên nhiều máy tính khác nhau. Để tương tác với một thực thể, hệ thống phải giải quyết bài toán: **Làm sao tìm được thực thể đó mà không bị ràng buộc cứng vào vị trí vật lý của nó?**

```
+------------------+         Định vị         +-----------------------+
| Tên / Định danh  | ======================> | Điểm truy cập / Địa chỉ|
| (Entity Name/ID) |   (Name Resolution)     |   (Access Point / IP) |
+------------------+                         +-----------------------+
         |                                               |
         +------------------- [ THỰC THỂ ] --------------+
                                (Entity)
```

### 1.1. Phân biệt các khái niệm cốt lõi

1. **Thực thể (Entity):** Đối tượng cần thao tác trong hệ thống (tiến trình, luồng, đối tượng phần mềm, tệp tin, thiết bị phần cứng).
2. **Điểm truy cập (Access Point) & Địa chỉ (Address):**
   * Điểm truy cập là nơi một thực thể có thể được liên lạc hoặc truy cập.
   * **Địa chỉ** chính là tên của điểm truy cập đó (ví dụ: cặp IP + Port `192.168.1.15:8080`).
   * *Đặc tính then chốt:* Một thực thể có thể có **nhiều địa chỉ** cùng lúc (máy chủ có 2 card mạng), và địa chỉ có thể thay đổi theo thời gian (thực thể di dời). Nếu gắn tên trực tiếp với địa chỉ, mỗi khi thực thể di dời (migration/relocation), mọi tham chiếu tới nó sẽ bị hỏng.
3. **Định danh (Identifier):**
   * Một định danh là một tên (chuỗi bit/ký tự) đại diện cho thực thể và phải thỏa mãn **3 điều kiện**:
     1. Một định danh chỉ trỏ tới **nhiều nhất một** thực thể.
     2. Mỗi thực thể được tham chiếu bởi **nhiều nhất một** định danh.
     3. Một định danh **luôn trỏ tới cùng một thực thể**, không bao giờ được tái sử dụng (kể cả khi thực thể đã không còn tồn tại).
4. **Tên đọc được với con người (Human-friendly Name):**
   * Chuỗi ký tự có nghĩa đối với con người (ví dụ: `ftp.cs.vu.nl`, `/home/student/ds.txt`). Không đòi hỏi tính duy nhất bất biến như Identifier.

### 1.2. Phân biệt URI, URL và URN

Chuẩn định danh tài nguyên trên mạng Internet:

```
foo://example.com:8042/over/there?name=ferret#nose
\_/   \______________/ \________/ \_________/ \__/
 |           |             |           |        |
scheme   authority        path       query   fragment
```

* **URI (Uniform Resource Identifier):** Thuật ngữ tổng quát chỉ xâu ký tự định danh tài nguyên.
* **URL (Uniform Resource Locator):** URI chỉ ra **vị trí và cách truy cập** (địa chỉ + giao thức: `https://hust.edu.vn/index.html`). Nếu tài nguyên chuyển sang server khác, URL bị vỡ (lỗi 404).
* **URN (Uniform Resource Name):** URI đóng vai trò **định danh thuần túy**, độc lập với vị trí (ví dụ mã số sách tiêu chuẩn quốc tế: `urn:isbn:0-486-27557-4`).
* *Ghi chú:* Cách phân loại URL/URN mang tính khái niệm; chuẩn RFC 3986 hiện nay chủ yếu dùng thuật ngữ URI.

---

## 2. KHÔNG GIAN TÊN PHẲNG (FLAT NAMING)

### 2.1. Bản chất kỹ thuật

* Không gian tên phẳng là tập hợp các định danh dưới dạng chuỗi bit không có cấu trúc (ví dụ địa chỉ MAC, UUID, giá trị băm), không chứa cấu trúc phân cấp và **không chứa thông tin về vị trí** hay cấu trúc mạng của thực thể.
* *Ưu điểm:* Linh hoạt, gọn nhẹ, không bị gắn chặt vào domain quản trị.
* *Thách thức:* Cho một chuỗi bit định danh $ID$, làm thế nào để tìm ra địa chỉ hiện tại của thực thể đó?

---

### 2.2. Các giải pháp phân giải tên phẳng

#### A. Quảng bá và Truyền thông nhóm (Broadcasting / Multicasting)
* **Cơ chế:** Nút cần phân giải gửi gói tin Broadcast/Multicast lên mạng cục bộ: *"Ai giữ định danh ID này, hãy cho tôi biết địa chỉ"*. Nút sở hữu thực thể nhận được tin nhắn sẽ gửi phản hồi Unicast chứa địa chỉ của mình.
* **Đại diện kinh điển - Giao thức ARP (Address Resolution Protocol):**
  * Ánh xạ từ địa chỉ logic (IP) sang địa chỉ vật lý (MAC) trên cùng một mạng cục bộ (LAN).
  * Máy A gửi: `Who has IP 192.168.1.5? Tell 192.168.1.2` bằng khung Broadcast Ethernet (`FF:FF:FF:FF:FF:FF`).
  * Máy B (giữ IP `192.168.1.5`) gửi trả Unicast: `192.168.1.5 is at MAC B`.
* **Hạn chế:** Kém co giãn (mọi nút trong mạng đều phải xử lý gói quảng bá), tốn băng thông; bị chặn bởi Router nên không thể chạy trên mạng diện rộng (WAN).

```
   [ Client A ] ----------------- Broadcast request: Who has ID=X? ----------------> (Mạng LAN)
         ^                                                                               |
         | <--------------- Unicast reply: ID=X is at 192.168.1.50 ------------------ [ Node B ]
```

#### B. Chuyển tiếp con trỏ (Forwarding Pointers)
* **Cơ chế:** Khi thực thể di chuyển từ vị trí $P_1$ sang $P_2$, tại vị trí cũ $P_1$ nó để lại một con trỏ tham chiếu (stub) trỏ tới $P_2$. Khi client gọi tới $P_1$, lời gọi được chuyển tiếp dọc theo chuỗi con trỏ tới vị trí hiện tại.
* **Vấn đề chuỗi con trỏ dài:** Sau nhiều lần di trú $P_1 \rightarrow P_2 \rightarrow \dots \rightarrow P_n$, độ trễ truy cập tăng tuyến tính theo độ dài chuỗi $O(n)$ và nếu một nút trung gian bị sập, chuỗi bị đứt.
* **Tối ưu hóa bằng Shortcut (Rút ngắn đường dẫn):**
  * Khi Client gửi request qua chuỗi $P_1 \rightarrow P_2 \rightarrow P_3$, nút đích $P_3$ khi trả kết quả sẽ gửi kèm địa chỉ hiện tại của mình về cho Client.
  * Client cập nhật tham chiếu trỏ thẳng tới $P_3$, bỏ qua các mắt xích $P_1, P_2$. Các stub cũ không còn được tham chiếu có thể được thu dọn (garbage collection, xem §2.5).

```
Ban đầu:     [ Client ] -------> [ P1 ] -------> [ P2 ] -------> [ P3 (Object) ]

Tối ưu hóa:  [ Client ] =======================================> [ P3 (Object) ]
             (Shortcut được thiết lập; các stub cũ ở P1, P2 có thể được thu hồi)
```

#### C. Giải pháp Dựa trên Trạm gốc (Home-based Approaches)
* **Cơ chế:** Áp dụng cho các thực thể di động (như Mobile IP). Thực thể có một trạm gốc cố định (**Home Location / Home Agent**) luôn biết vị trí hiện tại của nó.
* Khi thực thể di chuyển sang một mạng ngoại vi (**Foreign Network**), nó nhận một địa chỉ tạm thời (**Care-of Address - CoA**) và đăng ký CoA này với Home Agent.
* **Quy trình định tuyến qua trạm gốc (Triangle Routing):**
  1. Client gửi gói tin tới địa chỉ nhà (Home Agent) của thực thể.
  2. Home Agent tra cứu CoA hiện tại, đóng gói (tunneling) và chuyển tiếp tới CoA.
  3. Thực thể (hoặc Home Agent) có thể thông báo CoA cho Client để các gói sau gửi trực tiếp (route optimization, như trong Mobile IPv6).

```
                    +------------------------+
                    |  Home Location (Host)  |
                    +------------------------+
                     ^                      |
    1. Gửi tới Home  |                      | 2. Tunneling tới
                     |                      |    vị trí hiện tại (CoA)
            +----------------+              v
            | Client Location| ======> +---------------------------+
            +----------------+  3. Gửi | Present Location (CoA)    |
                               trực tiếp (sau tối ưu hóa)          |
                                       +---------------------------+
```

* **Điểm yếu:**
  * *Co giãn địa lý:* Nếu Client và thực thể đều ở Tokyo nhưng Home Agent đặt tại New York, gói tin phải đi vòng qua nửa vòng trái đất, gây độ trễ vô lý.
  * Nếu Home Agent bị sập thì không thể liên lạc với thực thể (trạm gốc là điểm lỗi đơn). Home Agent cũng phải tồn tại lâu dài, ngay cả khi thực thể đã vĩnh viễn rời đi.

---

### 2.3. Bảng băm phân tán: Giải thuật Chord DHT (Trọng tâm toán học)

Chord là giải pháp phân giải không gian tên phẳng có cấu trúc topo chặt chẽ, tìm khóa với độ phức tạp $O(\log N)$ bước nhảy.

#### 1. Định nghĩa toán học của vòng Chord $m$-bit:
* Không gian định danh gồm $2^m$ phần tử, sắp xếp theo thứ tự vòng tròn từ $0$ đến $2^m - 1$.
* Cả **ID của nút** (băm từ địa chỉ IP) và **khóa dữ liệu (Key)** đều được băm (Chord gốc dùng SHA-1) vào cùng một không gian modulo $2^m$:
  $$\text{Node ID} = \text{Hash}(\text{IP}) \pmod{2^m}$$
  $$\text{Key ID} = \text{Hash}(\text{Key}) \pmod{2^m}$$
* **Hàm Kế nhiệm Successor:** $\text{succ}(k)$ là nút mạng thực tế đầu tiên có $\text{ID} \ge k$ theo chiều kim đồng hồ. Dữ liệu có khóa $k$ được lưu trữ tại $\text{succ}(k)$.

#### 2. Cấu trúc Finger Table của Nút $p$:
Mỗi nút $p$ duy trì một bảng chuyển tiếp gồm đúng $m$ phần tử. Mục thứ $i$ ($1 \le i \le m$) trỏ tới:
$$FT_p[i] = \text{succ}((p + 2^{i-1}) \pmod{2^m})$$

#### 3. Bài toán mẫu tính Finger Table ($m=5 \Rightarrow 2^5 = 32$ vị trí: $0 \rightarrow 31$):
Giả sử tập hợp các nút thực tế đang online gồm:
$$\mathcal{N} = \{1, 4, 9, 11, 14, 18, 20, 21, 28\}$$

##### Tính toán chi tiết cho Nút $p = 1$:
* $i = 1$: $\text{succ}((1 + 2^0) \pmod{32}) = \text{succ}(2) = 4$
* $i = 2$: $\text{succ}((1 + 2^1) \pmod{32}) = \text{succ}(3) = 4$
* $i = 3$: $\text{succ}((1 + 2^2) \pmod{32}) = \text{succ}(5) = 9$
* $i = 4$: $\text{succ}((1 + 2^3) \pmod{32}) = \text{succ}(9) = 9$
* $i = 5$: $\text{succ}((1 + 2^4) \pmod{32}) = \text{succ}(17) = 18$
$$\Rightarrow \text{Finger Table}(Node\ 1) = [4, 4, 9, 9, 18]$$

##### Tính toán chi tiết cho Nút $p = 28$:
* $i = 1$: $\text{succ}((28 + 1) \pmod{32}) = \text{succ}(29) = 1$ (quay đầu qua 0)
* $i = 2$: $\text{succ}((28 + 2) \pmod{32}) = \text{succ}(30) = 1$
* $i = 3$: $\text{succ}((28 + 4) \pmod{32}) = \text{succ}(0) = 1$
* $i = 4$: $\text{succ}((28 + 8) \pmod{32}) = \text{succ}(4) = 4$
* $i = 5$: $\text{succ}((28 + 16) \pmod{32}) = \text{succ}(12) = 14$
$$\Rightarrow \text{Finger Table}(Node\ 28) = [1, 1, 1, 4, 14]$$

#### 4. Quy tắc định tuyến và kịch bản phân giải khóa:
**Quy tắc:** Nút $p$ nhận truy vấn khóa $k$:
* Nếu $k \in (p, \text{succ}(p)]$ (theo chiều vòng tròn), kết luận khóa nằm tại $\text{succ}(p) = FT_p[1]$.
* Ngược lại, chuyển tiếp tới nút trong Finger Table **gần $k$ nhất mà vẫn đứng trước $k$** (theo chiều kim đồng hồ kể từ $p$).

**Yêu cầu:** Nút $28$ cần tìm nơi lưu trữ của khóa $k = 12$.
1. **Tại Node 28:** Bảng $[1, 1, 1, 4, 14]$. Theo chiều kim đồng hồ từ 28, các nút $1, 4$ đứng trước $12$, còn $14$ đã vượt qua $12$. Nút gần $12$ nhất mà vẫn đứng trước nó là $4$ $\rightarrow$ chuyển tới **Node 4**.
2. **Tại Node 4:** Bảng Finger Table:
   * $FT_4[1] = \text{succ}(5) = 9$
   * $FT_4[2] = \text{succ}(6) = 9$
   * $FT_4[3] = \text{succ}(8) = 9$
   * $FT_4[4] = \text{succ}(12) = 14$
   * $FT_4[5] = \text{succ}(20) = 20$
   * Nút $14$ đã vượt qua $12$; nút gần $12$ nhất mà đứng trước là $9$ $\rightarrow$ chuyển tới **Node 9**.
3. **Tại Node 9:** $\text{succ}(9) = 11$ và $12 \notin (9, 11]$. Finger Table của Node 9 là $[11, 11, 14, 18, 28]$; nút gần $12$ nhất mà đứng trước là $11$ $\rightarrow$ chuyển tới **Node 11**.
4. **Tại Node 11:** $\text{succ}(11) = 14$ và $12 \in (11, 14]$ $\rightarrow$ kết luận khóa $k = 12$ nằm tại **Node 14**.

Đường đi: $28 \rightarrow 4 \rightarrow 9 \rightarrow 11 \rightarrow 14$ (4 bước nhảy).

---

### 2.4. Giải pháp Phân cấp (Hierarchical Approaches)

* Mạng diện rộng được chia thành cây các miền phân cấp (Hierarchy of Domains).
* Miền cấp thấp nhất (lá - leaf domain) tương ứng với mạng cục bộ LAN hoặc ô di động (cell).
* Mỗi miền $D$ có một nút thư mục đại diện $\text{dir}(D)$. Nút gốc $\text{dir}(\text{Root})$ bao phủ toàn cầu.
* **Nguyên tắc bản ghi vị trí (Location Record):**
  * Nút lá lưu trực tiếp địa chỉ thực tế của thực thể.
  * Nút cha không lưu địa chỉ thực tế, chỉ lưu **con trỏ trỏ xuống nút con** đang quản lý thực thể đó.
* **Cơ chế tìm kiếm:** Đi từ nút lá hiện tại lên dần các nút cha cho đến khi gặp một nút có bản ghi về thực thể $E$, sau đó lần theo con trỏ đi xuống nút lá đích.
* **Hạn chế:** Nút gốc phải lưu bản ghi cho *mọi* thực thể nên dễ thành điểm nghẽn; giải pháp là phân vùng (partition) thư mục của các nút cấp cao trên nhiều máy chủ.

```
                  [ dir(Root) ]
                  /           \
           [ dir(Domain A) ]   [ dir(Domain B) ]
             /          \             |
      [ Leaf A1 ]   [ Leaf A2 ]   [ Leaf B1 ]
          |
    (Địa chỉ E)
```

---

### 2.5. Thu gom rác và Đếm tham chiếu (Garbage Collection)

Khi một đối tượng phân tán không còn được tham chiếu bởi ai, nó cần được thu hồi, nếu không sẽ gây rò rỉ bộ nhớ. (Lưu ý: đây là thu hồi *đối tượng/tài nguyên*, không mâu thuẫn với quy tắc *định danh không bao giờ được tái sử dụng* ở §1.1.)

#### Cơ chế Đếm tham chiếu (Reference Counting):
* Mỗi đối tượng duy trì một biến đếm số lượng tham chiếu đang trỏ tới nó. Khi biến đếm $= 0$, đối tượng được giải phóng.
* **Lỗi Đếm hai lần (Counted Twice):** Tiến trình $P$ gửi bản tin tăng đếm $+1$ tới đối tượng $O$. $O$ tăng lên và gửi ACK. ACK bị mất $\rightarrow$ $P$ timeout và gửi lại $+1$ $\rightarrow$ biến đếm bị tính thừa $1$, đối tượng không bao giờ được giải phóng (Memory Leak). *Cách khắc phục:* thao tác tăng/giảm phải idempotent (ví dụ đánh số thứ tự bản tin) hoặc dùng *reference listing* (lưu danh sách các tiến trình tham chiếu thay vì số đếm).
* **Lỗi Race Condition khi chuyển tham chiếu:** Tiến trình $P_1$ chuyển tham chiếu cho $P_2$, rồi gửi bản tin $-1$ tới $O$. Do độ trễ mạng, bản tin $-1$ từ $P_1$ đến trước bản tin $+1$ từ $P_2$ $\rightarrow$ biến đếm về $0$ $\rightarrow$ $O$ bị giải phóng trong khi $P_2$ vẫn đang cần dùng. *Cách khắc phục:* **weighted reference counting** (trọng số tham chiếu chia đôi khi sao chép, không cần gửi thông điệp tăng đếm).
* **Chu trình cô lập (Unreachable Cycles):** Hai đối tượng trỏ chéo nhau ($A \rightarrow B$, $B \rightarrow A$) nhưng không còn tiến trình sống nào trỏ tới $\rightarrow$ biến đếm luôn $\ge 1 \rightarrow$ đếm tham chiếu không thu hồi được. Cần **thu gom dựa trên truy vết (tracing)** từ tập gốc (root set).

---

## 3. KHÔNG GIAN TÊN CÓ CẤU TRÚC (STRUCTURED NAMING)

### 3.1. Đồ thị Không gian tên (Naming Graph)

* Không gian tên được tổ chức thành đồ thị có hướng với 2 loại nút:
  * **Nút lá (Leaf Node):** Chứa thông tin trạng thái hoặc địa chỉ của thực thể (không có cạnh ra).
  * **Nút thư mục (Directory Node):** Chứa bảng liên kết ánh xạ giữa nhãn cạnh (Edge Label) và các nút con kế tiếp.
* **Đường dẫn tuyệt đối (Absolute Path):** Bắt đầu từ nút gốc (Root Node): `/home/steen/keys`.
* **Đường dẫn tương đối (Relative Path):** Bắt đầu từ một nút ngữ cảnh hiện thời.
* **Cơ chế bao đóng (Closure Mechanism):** Quá trình phân giải tên luôn cần biết trước cách tìm điểm bắt đầu (nút gốc hoặc nút ngữ cảnh). Ví dụ: hệ thống tệp biết inode gốc từ superblock; DNS biết địa chỉ các root server.

---

### 3.2. Không gian tên Hệ thống tệp UNIX

Mỗi tệp tin trong UNIX được định danh bên trong hệ thống tệp bởi một chỉ mục số nguyên gọi là **Inode**:

```
+-------------------------------------------------------------+
| Boot Block | Super Block |  Danh sách Inodes  | Khối dữ liệu|
|            |             | [Inode 1][Inode 2] | (Data Blocks|
+-------------------------------------------------------------+
```

* **Cấu trúc Inode:** Chứa quyền hạn (permissions), chủ sở hữu, kích thước, thời gian truy cập/sửa đổi, **biến đếm liên kết (link count)** và các con trỏ trỏ trực tiếp/gián tiếp tới các Data Blocks. (*Tên tệp không nằm trong Inode.*)
* **Thư mục trong UNIX:** Là một tệp đặc biệt, chứa bảng ánh xạ giữa `Tên file (String)` và `Số Inode`.

#### So sánh Hard Link và Soft Link:

| Tiêu chí | Liên kết cứng (Hard Link) | Liên kết mềm (Soft/Symbolic Link) |
| :--- | :--- | :--- |
| **Cú pháp lệnh** | `ln file1 link1` | `ln -s file1 link2` |
| **Bản chất Inode** | Thêm một mục thư mục trỏ chung vào **cùng một số Inode** gốc. | Tạo ra một **Inode mới** (kiểu symlink). |
| **Nội dung lưu trữ** | Dùng chung các khối dữ liệu với file gốc. | Lưu **đường dẫn bằng chữ** tới file đích (đường dẫn ngắn có thể nằm ngay trong Inode). |
| **Xuyên File System** | Không (chỉ trong cùng một phân vùng/hệ thống tệp); thường không cho phép với thư mục. | Có (đường dẫn có thể trỏ sang phân vùng khác, kể cả hệ thống tệp mạng đã mount). |
| **Khi xóa file gốc** | Dữ liệu vẫn còn nếu link count $> 0$. | Liên kết bị vỡ (Dangling/Broken link). |

---

### 3.3. Phân tán Không gian tên (Name Space Distribution)

Để phục vụ quy mô toàn cầu, không gian tên được phân rã thành 3 tầng:

```
                          [ TẦNG TOÀN CỤC (Global Layer) ]
                         (Nút gốc '.', Top-level: com, edu, vn)
                                        |
                       [ TẦNG QUẢN TRỊ (Administrational Layer) ]
                         (Miền doanh nghiệp/trường học: hust.edu.vn)
                                        |
                          [ TẦNG QUẢN LÝ (Managerial Layer) ]
                         (Phòng ban, máy trạm cục bộ, tệp cá nhân)
```

#### Bảng so sánh 3 tầng phân tán:

| Đặc trưng | Tầng Toàn cục (Global) | Tầng Quản trị (Administrational) | Tầng Quản lý (Managerial) |
| :--- | :--- | :--- | :--- |
| **Phạm vi địa lý** | Toàn cầu (Worldwide) | Một tổ chức | Một phòng ban / mạng LAN |
| **Số lượng nút/máy chủ** | Ít | Nhiều | Rất nhiều |
| **Tần suất thay đổi** | Rất ít (ổn định) | Thấp | Cao |
| **Lan truyền cập nhật** | Chậm, trì hoãn (lazy) | Tức thời | Tức thời |
| **Thời gian đáp ứng chấp nhận được** | Cỡ giây | Cỡ mili-giây | Tức thì |
| **Số lượng bản sao** | Nhiều (nhân bản mạnh) | Không hoặc ít | Không |
| **Caching phía Client** | Có | Có | Đôi khi |

---

### 3.4. Cơ chế Phân giải Tên: Đệ quy vs. Không đệ quy

Ví dụ: phân giải tên `www.hust.edu.vn`.

#### A. Phân giải Không đệ quy (Iterative Name Resolution)
* **Quy trình:** Bên phân giải (resolver) tự mình lần lượt hỏi từng cấp server, mỗi server chỉ trả về *địa chỉ của server cấp kế tiếp*:
  1. Resolver hỏi Root Server $\rightarrow$ Root trả về: *"Hãy hỏi server `.vn` tại IP $X$"*.
  2. Resolver hỏi server `.vn` $\rightarrow$ trả về: *"Hãy hỏi server `edu.vn` tại IP $Y$"*.
  3. Resolver hỏi server `edu.vn` $\rightarrow$ trả về: *"Hãy hỏi server của `hust.edu.vn` tại IP $Z$"*.
  4. Resolver hỏi server có thẩm quyền của `hust.edu.vn` $\rightarrow$ nhận địa chỉ IP cuối cùng.
* **Đặc tính:** Các server cấp trên không phải giữ trạng thái cho từng truy vấn, tải nhẹ hơn; nhưng bên phân giải phải chịu toàn bộ chi phí trao đổi qua mạng, và việc cache hiệu quả chỉ ở phía resolver.

#### B. Phân giải Đệ quy (Recursive Name Resolution)
* **Quy trình:** Mỗi server nhận truy vấn sẽ **tự chuyển tiếp** tới server cấp kế tiếp và đợi kết quả: Client $\rightarrow$ Server A $\rightarrow$ Server B $\rightarrow$ ... $\rightarrow$ Authoritative Server; kết quả được trả ngược lại dọc theo chuỗi.
* **Đặc tính:** Cho phép cache ở mọi trạm trung gian và giảm số lần trao đổi của client; nhưng mỗi server trung gian phải giữ trạng thái cho truy vấn đang chờ và chịu tải lớn hơn.

```
PHÂN GIẢI KHÔNG ĐỆ QUY (Iterative)              PHÂN GIẢI ĐỆ QUY (Recursive)

 Resolver ---(1) hỏi---> Root                    Client --(1)--> Server A
          <--(2) "hỏi .vn"--                                      |(2)
 Resolver ---(3) hỏi---> Server .vn                              v
          <--(4) "hỏi edu.vn"--                               Server B
 Resolver ---(5) hỏi---> Server edu.vn                            |(3)
          <--(6) "hỏi hust.edu.vn"--                              v
 Resolver ---(7) hỏi---> Authoritative                       Authoritative
          <--(8) địa chỉ IP--                                     |
                                                  Client <--(6)-- A <--(5)-- B <--(4)--
                                                  (kết quả trả ngược dọc theo chuỗi)
```

> **Trong thực tế DNS** thường là *hỗn hợp*: máy khách (stub resolver) gửi truy vấn **đệ quy** tới DNS Server cục bộ (Local/Recursive Resolver); Resolver này lại thực hiện phân giải **không đệ quy** tới Root, TLD và Authoritative Server rồi cache kết quả.

---

### 3.5. Hệ thống Tên miền DNS (Domain Name System)

DNS là một cơ sở dữ liệu phân tán, phân cấp dùng để ánh xạ tên miền sang địa chỉ IP (và nhiều thông tin khác). Các bản ghi được cache theo thời gian sống **TTL**.

#### Các loại Bản ghi Tài nguyên cốt lõi (Resource Records - RR):
* **SOA (Start of Authority):** Khai báo các tham số chính của zone (serial, refresh, retry, expire, minimum TTL, email quản trị).
* **A (Address):** Ánh xạ tên máy (hostname) sang địa chỉ IPv4 ($32\text{-bit}$).
* **AAAA:** Ánh xạ tên máy sang địa chỉ IPv6 ($128\text{-bit}$).
* **NS (Name Server):** Khai báo máy chủ có thẩm quyền của một zone (dùng cả cho việc *ủy quyền - delegation* zone con).
* **CNAME (Canonical Name):** Bí danh (Alias) trỏ về tên chính thức (tương tự Soft Link).
* **MX (Mail Exchange):** Khai báo máy chủ tiếp nhận thư điện tử cho domain kèm mức độ ưu tiên.
* **PTR (Pointer):** Dùng cho phân giải ngược (Reverse DNS lookup): ánh xạ từ địa chỉ IP sang tên miền (trong zone `in-addr.arpa` cho IPv4, `ip6.arpa` cho IPv6).

---

## 4. ĐỊNH DANH DỰA TRÊN THUỘC TÍNH (ATTRIBUTE-BASED NAMING)

Khi người dùng **không biết chính xác tên hoặc ID** của thực thể mà chỉ biết các thuộc tính mô tả (ví dụ: *"Tìm máy in màu hai mặt, tốc độ $> 30\text{ trang/phút}$ ở tầng 3"*), hệ thống chuyển sang mô hình **Dịch vụ thư mục (Directory Services)**.

### 4.1. Khung mô tả RDF (Resource Description Framework)
Dữ liệu được mô tả dưới dạng bộ ba (**Subject, Predicate, Object**):
* *Ví dụ:* `(Eric_Miller, personalTitle, "Dr.")`, `(Book_A, author, Tolkien)`.

### 4.2. Giao thức LDAP (Lightweight Directory Access Protocol)
* Là phiên bản gọn nhẹ của giao thức truy cập thư mục thuộc chuẩn X.500 (ITU-T).
* Thông tin tổ chức thành cây thông tin thư mục **DIT (Directory Information Tree)**:
  * **C (Country):** Quốc gia (ví dụ: `C=VN`).
  * **O (Organization):** Tổ chức/Doanh nghiệp (`O=HUST`).
  * **OU (Organizational Unit):** Phòng ban/Khoa (`OU=SoICT`).
  * **CN (Common Name):** Tên định danh thực thể (`CN=Main Server`).
* **Định danh phân biệt (Distinguished Name - DN):** Tương đương đường dẫn tuyệt đối, viết từ nút lá lên gốc:
  $$\text{DN} = \text{CN=Main Server, OU=SoICT, O=HUST, C=VN}$$
* **Định danh phân biệt tương đối (Relative Distinguished Name - RDN):** Nhãn của từng nút đơn lẻ trên cây (ví dụ `CN=Main Server`).
* *Ghi chú:* Triển khai hiện đại (Active Directory, OpenLDAP) thường dùng thành phần miền `DC` (ví dụ `dc=hust,dc=edu,dc=vn`).

---

### 4.3. Ánh xạ Thư mục vào DHT: Cây Thuộc tính-Giá trị (Attribute-Value Tree)

Làm sao hỗ trợ tìm kiếm theo thuộc tính trên một mạng P2P dùng DHT (Chord)? Ý tưởng: tổ chức thuộc tính thành cây và băm các đường dẫn của cây vào DHT.

```
                [ Root: type=book ]
                 /               \
         [ author=Tolkien ]     [ genre=fantasy ]
                 |                       |
          [ title=LOTR ]           [ title=LOTR ]
```

* **Cơ chế băm chuỗi tiền tố (Prefix Path Hashing):**
  * Các đường dẫn từ gốc tới các nút được băm thành khóa (dùng ký tự phân tách không xuất hiện trong giá trị, ví dụ `/`):
    $$h_1 = \text{Hash}(\text{"type=book"})$$
    $$h_2 = \text{Hash}(\text{"type=book/author=Tolkien"})$$
    $$h_3 = \text{Hash}(\text{"type=book/author=Tolkien/title=LOTR"})$$
  * Nút chịu trách nhiệm cho khóa $h_2$ trên vòng Chord lưu danh sách các tài nguyên khớp. Khi client tìm *"sách của Tolkien"*, chỉ cần tính $h_2$ và gửi truy vấn tới đúng nút đó.
* **Hạn chế:**
  * Cần quy ước **thứ tự thuộc tính** thống nhất; truy vấn theo thứ tự khác phải dùng nhánh khác của cây.
  * Không hỗ trợ tốt truy vấn khoảng (range query).
  * Nút giữ khóa của các tiền tố ngắn (như `type=book`) có nguy cơ quá tải (hotspot).

---

### 4.4. Mạng Lớp phủ Ngữ nghĩa (Semantic Overlay Networks - SON)

* **Tư tưởng:** Trong mạng chia sẻ file P2P, thay vì kết nối các nút ngẫu nhiên, hệ thống kết nối các nút có **sở thích hoặc danh mục tài nguyên tương đồng** thành các cụm ngữ nghĩa, giúp truy vấn nhanh tìm thấy kết quả ở các láng giềng gần.
* **Mô hình Gossiping 2 tầng (Two-layer Gossip Protocol):**
  1. *Tầng dưới (Random Overlay):* Giao thức gossip duy trì tập láng giềng ngẫu nhiên (peer sampling) để giữ kết nối toàn cục và khám phá nút mới.
  2. *Tầng trên (Semantic Overlay):* Dùng **hàm khoảng cách ngữ nghĩa (Semantic Proximity Function)** để chọn và kết nối trực tiếp với các nút có danh sách tệp tin giống mình nhất.

---

## 5. CÁC VẤN ĐỀ AN NINH TRONG ĐỊNH DANH

### 5.1. Tấn công đầu độc bộ đệm ARP (ARP Spoofing / Poisoning)
* **Lỗ hổng:** Giao thức ARP không có cơ chế xác thực. Một máy tính có thể gửi bản tin ARP Reply giả mạo dù không ai hỏi (Gratuitous ARP).
* **Kịch bản:** Kẻ tấn công gửi liên tục các bản tin ARP giả:
  * Gửi tới nạn nhân: *"IP của Default Gateway đang mang MAC của Kẻ tấn công"*.
  * Gửi tới Router: *"IP của Nạn nhân đang mang MAC của Kẻ tấn công"*.
* **Hậu quả:** Lưu lượng của nạn nhân bị chuyển hướng qua máy tấn công (Man-In-The-Middle - MITM), bị nghe lén hoặc sửa đổi.
* **Phòng chống:** Dynamic ARP Inspection / DHCP Snooping trên switch, ARP tĩnh cho các máy quan trọng, mã hóa đầu-cuối (TLS).

### 5.2. Tấn công khuếch đại DNS (DNS Amplification Attack)
* **Lỗ hổng:** DNS thường dùng **UDP** (không cần bắt tay 3 bước nên IP nguồn dễ bị giả mạo - IP Spoofing) và một truy vấn nhỏ có thể sinh ra phản hồi rất lớn.
* **Cơ chế tấn công (phản xạ + khuếch đại):**
  1. Kẻ tấn công gửi gói tin DNS Query nhỏ (khoảng $64\text{ bytes}$) tới một Open Recursive DNS Server.
  2. Trong gói tin, kẻ tấn công giả mạo IP nguồn thành IP của **Nạn nhân (Victim)**.
  3. Truy vấn yêu cầu bản ghi lớn (ví dụ `dig ANY isc.org`).
  4. DNS Server gửi phản hồi lớn (khoảng $3.223\text{ bytes}$) về IP của Nạn nhân.
* **Hệ số khuếch đại:** Theo cảnh báo US-CERT TA13-088A, khoảng **$28$ đến $54$ lần** (có thể cao hơn khi dùng EDNS0). Nhiều nguồn như vậy cùng lúc gây DDoS làm nghẽn đường truyền của nạn nhân.
* **Phòng chống:** Lọc IP giả mạo tại biên mạng (BCP 38), đóng Open Resolver, giới hạn tốc độ phản hồi (Response Rate Limiting), hạn chế truy vấn `ANY` (RFC 8482).

---

## 6. ACTIVE RECALL: BÀI TẬP VÀ CÂU HỎI BẪY TƯ DUY

### 6.1. Câu hỏi Kiểm tra Bản chất

**Câu 1:** *Tại sao bảng ánh xạ tên tập trung (Centralized Mapping Table) thất bại trên quy mô Internet, dù nó giải quyết bài toán phân giải tên đơn giản và nhất quán?*
* **Giải đáp:** Mô hình tập trung vi phạm cả 3 chiều co giãn:
  1. *Quy mô (Size):* Trở thành điểm nghẽn cổ chai (Bottleneck) chịu tải hàng tỷ request; đồng thời là điểm lỗi đơn (SPOF).
  2. *Địa lý (Geography):* Người dùng ở xa máy chủ phải chịu độ trễ đường truyền lớn.
  3. *Quản trị (Administration):* Các tổ chức, quốc gia khó chấp nhận để một thực thể duy nhất kiểm soát toàn bộ dữ liệu định danh của mình.

**Câu 2:** *Sự khác biệt bản chất giữa việc xóa file có Hard Link và xóa file là mục tiêu của Soft Link trong UNIX là gì?*
* **Giải đáp:**
  * Với **Hard Link**: lệnh `rm` (unlink) chỉ xóa một mục thư mục và giảm biến đếm liên kết trong Inode đi 1 (`link_count--`). Dữ liệu chỉ thực sự được thu hồi khi `link_count == 0` (và không còn tiến trình nào đang mở file). Các Hard Link còn lại vẫn truy cập dữ liệu bình thường.
  * Với **Soft Link**: nếu file đích bị xóa (và không còn hard link nào khác), Inode và Data Block của nó được giải phóng. Soft Link vẫn tồn tại nhưng đường dẫn bên trong trỏ vào chỗ không còn $\rightarrow$ **Dangling Link (liên kết gãy)**.

---

### 6.2. Bài tập Tính toán Chord DHT Tự luyện

**Đề bài:** Cho mạng Chord DHT với $m = 4$ ($16$ vị trí từ $0 \rightarrow 15$). Các nút mạng hiện đang online gồm:
$$\mathcal{N} = \{0, 3, 7, 10, 14\}$$

**Yêu cầu:**
1. Hãy lập bảng **Finger Table** cho Nút $p = 7$.
2. Trình bày chi tiết từng chặng phân giải để tìm vị trí lưu trữ của khóa $k = 2$ bắt đầu từ Nút $7$.

#### Đáp án:
1. **Finger Table của Nút 7 ($m=4$):**
   * $i = 1$: $\text{succ}((7 + 1) \pmod{16}) = \text{succ}(8) = 10$
   * $i = 2$: $\text{succ}((7 + 2) \pmod{16}) = \text{succ}(9) = 10$
   * $i = 3$: $\text{succ}((7 + 4) \pmod{16}) = \text{succ}(11) = 14$
   * $i = 4$: $\text{succ}((7 + 8) \pmod{16}) = \text{succ}(15) = 0$
   $$\Rightarrow \text{Finger Table}(Node\ 7) = [10, 10, 14, 0]$$

2. **Truy vết tìm khóa $k = 2$ từ Nút 7:**
   * **Tại Node 7:** $\text{succ}(7) = 10$ và $2 \notin (7, 10]$. Tính khoảng cách theo chiều kim đồng hồ từ $7$: tới khóa là $(2 - 7) \bmod 16 = 11$; tới các finger $[10, 10, 14, 0]$ lần lượt là $3, 3, 7, 9$. Finger gần khóa nhất mà chưa vượt qua nó (khoảng cách lớn nhất $\le 11$) là $9$, tức **Node 0**. Chuyển truy vấn tới Node 0.
   * **Tại Node 0:** $\text{succ}(0) = 3$ và $2 \in (0, 3]$. Kết luận khóa $k = 2$ được lưu trữ tại **Node 3**.
   * Đường đi: $7 \rightarrow 0 \rightarrow 3$ (2 bước nhảy).

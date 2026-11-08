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

### 1.1. Phân biệt bộ ba khái niệm cốt lõi

1. **Thực thể (Entity):** Đối tượng cần thao tác trong hệ thống (tiến trình, luồng, đối tượng phần mềm, tệp tin, thiết bị phần cứng).
2. **Điểm truy cập (Access Point) & Địa chỉ (Address):**
   * Điểm truy cập là nơi một thực thể có thể được liên lạc hoặc truy cập.
   * **Địa chỉ** chính là tên của điểm truy cập đó (ví dụ: bộ tứ IP + Port `192.168.1.15:8080`).
   * *Đặc tính then chốt:* Một thực thể có thể có **nhiều địa chỉ** cùng lúc (máy chủ có 2 card mạng, đối tượng di chuyển qua mạng). Nếu liên kết tên trực tiếp với địa chỉ, mỗi khi thực thể di dời (migration/relocation), toàn bộ phần mềm gọi nó sẽ bị hỏng liên kết.
3. **Định danh (Identifier):**
   * Một định danh là một chuỗi bit/ký tự đại diện cho thực thể và phải thỏa mãn **3 điều kiện tiên quyết**:
     1. Một định danh chỉ trỏ tới **nhiều nhất một** thực thể.
     2. Mỗi thực thể được định danh bởi **duy nhất một** định danh.
     3. Một định danh **không bao giờ được tái sử dụng** (mãi mãi gắn liền với thực thể đó, kể cả khi thực thể đã chết).
4. **Tên người dùng đọc được (Human-friendly Name):**
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
* **URN (Uniform Resource Name):** URI chỉ đóng vai trò **định danh thuần túy**, hoàn toàn độc lập với vị trí (ví dụ mã số sách tiêu chuẩn quốc tế: `urn:isbn:0-486-27557-4`).

---

## 2. KHÔNG GIAN TÊN PHẲNG (FLAT NAMING)

### 2.1. Bản chất kỹ thuật

* Không gian tên phẳng là tập hợp các định danh dưới dạng chuỗi bit ngẫu nhiên, không chứa cấu trúc phân cấp và **không chứa bất kỳ thông tin nào về vị trí địa lý hay cấu trúc mạng** của thực thể.
* *Ưu điểm:* Linh hoạt, gọn nhẹ, không bị gắn chặt vào domain quản trị.
* *Thách thức:* Cho một chuỗi bit định danh $ID$, làm thế nào để tìm ra địa chỉ IP hiện tại của thực thể đó?

---

### 2.2. Các giải pháp phân giải tên phẳng

#### A. Quảng bá và Truyền thông nhóm (Broadcasting / Multicasting)
* **Cơ chế:** Nút cần phân giải gửi gói tin Broadcast/Multicast lên mạng cục bộ: *"Ai giữ định danh ID này, hãy trả lời về IP của tôi"*. Nút sở hữu thực thể nhận được tin nhắn sẽ gửi phản hồi Unicast chứa địa chỉ của mình.
* **Đại diện kinh điển - Giao thức ARP (Address Resolution Protocol):**
  * Ánh xạ từ địa chỉ logic (IP) sang địa chỉ vật lý (MAC) trên cùng một mạng cục bộ (LAN).
  * Máy A gửi: `Who has IP 192.168.1.5? Tell 192.168.1.2 (MAC A)` bằng gói tin Broadcast Ethernet (`FF:FF:FF:FF:FF:FF`).
  * Máy B (giữ IP `192.168.1.5`) gửi trả Unicast: `192.168.1.5 is at MAC B`.
* **Hạn chế:** Kém co giãn ($O(N)$ gói tin), tốn băng thông; bị chặn bởi Router nên không thể chạy trên mạng diện rộng (WAN).

```
   [ Client A ] ----------------- Broadcast request: Who has ID=X? ----------------> (Mạng LAN)
         ^                                                                               |
         | <--------------- Unicast reply: ID=X is at 192.168.1.50 ------------------ [ Node B ]
```

#### B. Chuyển tiếp con trỏ (Forwarding Pointers)
* **Cơ chế:** Khi thực thể di chuyển từ vị trí $P_1$ sang $P_2$, tại vị trí cũ $P_1$ nó để lại một con trỏ tham chiếu (stub) trỏ tới $P_2$. Khi client gọi tới $P_1$, lời gọi sẽ tự động được chuyển tiếp dọc theo chuỗi con trỏ tới $P_2$.
* **Vấn đề chuỗi con trỏ dài vô hạn:** Sau nhiều lần di trú $P_1 \rightarrow P_2 \rightarrow P_3 \rightarrow \dots \rightarrow P_n$, độ trễ truy cập tăng tuyến tính $O(k)$ và nếu một nút trung gian bị sập, toàn bộ chuỗi bị đứt gãy.
* **Tối ưu hóa bằng Shortcut (Rút ngắn đường dẫn):**
  * Khi Client gửi request tới $P_1 \rightarrow P_2 \rightarrow P_3$, nút đích $P_3$ khi trả kết quả sẽ gửi kèm địa chỉ hiện tại của mình về cho Client.
  * Client cập nhật trực tiếp con trỏ của mình trỏ thẳng tới $P_3$, bỏ qua các mắt xích $P_1, P_2$.

```
Ban đầu:     [ Client ] -------> [ P1 ] -------> [ P2 ] -------> [ P3 (Object) ]
                                                            
Tối ưu hóa:  [ Client ] =======================================> [ P3 (Object) ]
             (Shortcut được thiết lập, các stub cũ ở P1, P2 được thu hồi)
```

#### C. Giải pháp Dựa trên Trạm gốc (Home-based Approaches)
* **Cơ chế:** Áp dụng cho các thực thể di động liên tục (như Mobile IP). Thực thể được cấp một trạm gốc cố định (**Home Location / Home Agent**).
* Khi thực thể di chuyển sang một mạng ngoại vi khác (**Foreign Network**), nó nhận một địa chỉ tạm thời (**Care-of Address - CoA**) và gửi thông báo cập nhật CoA về Home Agent.
* **Quy trình định tuyến 2 chặng (Triangle Routing):**
  1. Client gửi gói tin tới Home Agent của thực thể.
  2. Home Agent đón gói tin, đóng gói (tunneling) và chuyển tiếp tới địa chỉ tạm thời CoA.
  3. Client có thể cập nhật trực tiếp CoA để gửi các gói tin tiếp theo (Direct Routing).

```
                    +------------------------+
                    |  Home Location (Host)  |
                    +------------------------+
                     ^                      |
    1. Gửi tới Home  |                      | 3. Tunneling tới
                     |                      |    vị trí hiện tại
            +----------------+              v
            | Client Location| ======> +---------------------------+
            +----------------+  4. Gửi | Present Location (CoA)    |
                                thẳng  +---------------------------+
```

* **Điểm yếu:** Vấn đề co giãn địa lý. Nếu Client và Host đang ở cạnh nhau tại Tokyo, nhưng Home Agent lại đặt tại New York, gói tin phải đi vòng qua nửa vòng trái đất gây độ trễ vô lý.

---

### 2.3. Bảng băm phân tán: Giải thuật Chord DHT (Trọng tâm toán học)

Chord là giải pháp phân giải không gian tên phẳng có cấu trúc topo chặt chẽ, đảm bảo tìm kiếm khóa với độ phức tạp $O(\log N)$.

#### 1. Định nghĩa toán học của vòng Chord $m$-bit:
* Không gian định danh gồm $2^m$ phần tử, sắp xếp theo thứ tự vòng tròn từ $0$ đến $2^m - 1$.
* Cả **Địa chỉ IP của nút** và **Khóa của dữ liệu (Key)** đều được băm bằng SHA-1 vào cùng một không gian modulo $2^m$:
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

#### 4. Kịch bản phân giải khóa (Resolution Walkthrough):
**Yêu cầu:** Nút $28$ cần tìm nơi lưu trữ của khóa $k = 12$.
1. Nút $28$ kiểm tra bảng Finger Table $[1, 1, 1, 4, 14]$:
   * Giá trị $12$ nằm giữa mục 4 ($FT_{28}[4] = 4$) và mục 5 ($FT_{28}[5] = 14$).
   * Node $28$ chuyển tiếp truy vấn tới node lớn nhất nhưng nhỏ hơn 12 $\rightarrow$ gửi tới **Node 4**.
2. Tại Node $4$, bảng Finger Table là:
   * $FT_4[1] = \text{succ}(5) = 9$
   * $FT_4[2] = \text{succ}(6) = 9$
   * $FT_4[3] = \text{succ}(8) = 9$
   * $FT_4[4] = \text{succ}(12) = 14$
   * $FT_4[5] = \text{succ}(20) = 20$
   * Giá trị $12$ nằm giữa mục 3 (Node 9) và mục 4 (Node 14) $\rightarrow$ gửi tới **Node 9**.
3. Tại Node $9$: Nút kế tiếp trực tiếp $\text{succ}(9) = 11$, và $\text{succ}(11) = 14$. Vì $11 < 12 \le 14$, truy vấn tới Node 11 rồi kết luận khóa $k=12$ nằm tại **Node 14**.

---

### 2.4. Giải pháp Phân cấp (Hierarchical Approaches)

* Mạng diện rộng được chia thành cây các miền phân cấp (Hierarchy of Domains).
* Miền cấp thấp nhất (lá - leaf domain) tương ứng với mạng cục bộ LAN hoặc ô di động (cell).
* Mỗi miền $D$ có một nút thư mục đại diện $\text{dir}(D)$. Nút gốc $\text{dir}(\text{Root})$ bao phủ toàn cầu.
* **Nguyên tắc bản ghi vị trí (Location Record):**
  * Nút lá lưu trực tiếp địa chỉ thực tế của thực thể.
  * Nút cha không lưu địa chỉ thực tế, chỉ lưu **con trỏ trỏ xuống nút con** đang quản lý thực thể đó.
* **Cơ chế tìm kiếm:** Đi từ nút lá hiện tại lên dần nút cha cho đến khi gặp một nút có lưu bản ghi trỏ tới thực thể $E$, sau đó lần theo con trỏ đi xuống nút lá đích.

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

### 2.5. Bài toán Cạn kiệt & Rác trong Không gian tên (Garbage Collection)

Khi thực thể bị xóa bỏ hoặc chuyển vùng, các định danh/tham chiếu cũ không còn dùng nữa nếu không thu dọn sẽ gây cạn kiệt bộ nhớ.

#### Cơ chế Đếm tham chiếu (Reference Counting):
* Mỗi đối tượng duy trì một biến đếm số lượng tham chiếu đang trỏ tới nó. Khi biến đếm $= 0$, đối tượng tự hủy.
* **Lỗi Đếm hai lần (Counted Twice):** Tiến trình $P$ gửi bản tin tăng đếm $+1$ tới đối tượng $O$. $O$ tăng lên và gửi lời báo nhận (ACK). ACK bị mất mạng $\rightarrow$ $P$ timeout và gửi lại $+1$ lần nữa $\rightarrow$ Biến đếm bị lệch thừa $1$, đối tượng không bao giờ được giải phóng (Memory Leak).
* **Lỗi Đếm chậm (Race Condition):** Tiến trình $P_1$ chuyển quyền tham chiếu cho $P_2$. $P_1$ gửi bản tin $-1$ tới $O$. Do độ trễ mạng, bản tin $-1$ từ $P_1$ đến trước bản tin $+1$ từ $P_2 \rightarrow$ Biến đếm về $0 \rightarrow$ $O$ tự hủy trong khi $P_2$ vẫn đang chờ sử dụng.
* **Chu trình cô lập (Unreachable Cycles):** Hai thực thể trỏ chéo lẫn nhau ($A \rightarrow B$ và $B \rightarrow A$) nhưng không có tiến trình sống nào trong hệ thống trỏ tới chúng $\rightarrow$ Biến đếm luôn $\ge 1 \rightarrow$ Bộ thu gom đếm tham chiếu bất lực.

---

## 3. KHÔNG GIAN TÊN CÓ CẤU TRÚC (STRUCTURED NAMING)

### 3.1. Đồ thị Không gian tên (Naming Graph)

* Không gian tên được tổ chức thành đồ thị có hướng với 2 loại nút:
  * **Nút lá (Leaf Node):** Chứa thông tin trạng thái hoặc địa chỉ của thực thể (không có cạnh ra).
  * **Nút thư mục (Directory Node):** Chứa bảng liên kết ánh xạ giữa nhãn cạnh (Edge Label) và các nút con kế tiếp.
* **Đường dẫn tuyệt đối (Absolute Path):** Bắt đầu từ nút gốc (Root Node): `/home/steen/keys`.
* **Đường dẫn tương đối (Relative Path):** Bắt đầu từ một nút ngữ cảnh hiện thời.
* **Nguyên tắc bao đóng (Closure Principle):** Mọi quá trình phân giải tên đều bắt buộc phải biết trước địa chỉ của điểm bắt đầu (nút gốc hoặc nút ngữ cảnh).

---

### 3.2. Không gian tên Hệ thống tệp UNIX

Mọi tệp tin trong hệ điều hành UNIX được định danh vật lý bởi một chỉ mục số nguyên gọi là **Inode**:

```
+-------------------------------------------------------------+
| Boot Block | Super Block |  Danh sách Inodes  | Khối dữ liệu|
|            |             | [Inode 1][Inode 2] | (Data Blocks|
+-------------------------------------------------------------+
```

* **Cấu trúc Inode:** Chứa quyền hạn (permissions), kích thước file, thời gian sửa đổi và danh sách các con trỏ trỏ trực tiếp/gián tiếp tới các Data Blocks chứa dữ liệu thực.
* **Thư mục trong UNIX:** Bản chất là một tệp đặc biệt, chứa bảng ánh xạ giữa `Tên file (String)` và `Số Inode`.

#### So sánh Hard Link và Soft Link:

| Tiêu chí | Liên kết vật lý (Hard Link) | Liên kết biểu tượng (Soft/Symbolic Link) |
| :--- | :--- | :--- |
| **Cú pháp lệnh** | `ln file1 link1` | `ln -s file1 link2` |
| **Bản chất Inode** | Trỏ chung vào **cùng một số Inode** gốc. | Tạo ra một **Inode hoàn toàn mới**. |
| **Nội dung lưu trữ** | Dùng chung các khối dữ liệu (Data Blocks) với file gốc. | Data Block chứa **đường dẫn bằng chữ** tới file đích. |
| **Xuyên File System**| Không (chỉ chạy trong cùng 1 phân vùng đĩa). | Có (trỏ xuyên qua các ổ đĩa và mạng máy tính khác). |
| **Khi xóa file gốc** | Dữ liệu vẫn còn nguyên nếu biến đếm Hard Link $> 0$. | Liên kết bị vỡ (Dangling/Broken pointer). |

---

### 3.3. Phân tán Không gian tên (Name Space Distribution)

Để phục vụ quy mô toàn cầu, không gian tên được phân rã thành 3 tầng quản trị:

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
| **Phạm vi địa lý** | Toàn cầu (Worldwide) | Một tổ chức / quốc gia | Một phòng ban / mạng LAN |
| **Số lượng máy chủ**| Cực ít (Vài chục máy) | Hàng nghìn máy | Hàng triệu máy |
| **Tần suất cập nhật**| Cực hiếm khi đổi | Thấp (Vài tuần/lần) | Rất cao (Từng giây/phút) |
| **Thời gian đáp ứng**| Vài chục đến hàng trăm ms | Vài ms | Tức thì ($< 1\text{ ms}$) |
| **Số lượng bản sao** | Cực nhiều (Replication cao) | Trung bình | Không có hoặc rất ít |
| **Phía Client Caching**| Bắt buộc phải cache | Nên cache | Thường không cache |

---

### 3.4. Cơ chế Phân giải Tên: Đệ quy vs. Không đệ quy

#### A. Phân giải Không đệ quy (Iterative Name Resolution)
* **Quy trình:** Client tự mình lần lượt gửi request tới từng cấp server:
  1. Client hỏi Root Server $\rightarrow$ Root Server trả về: *"Tôi không biết, hãy đi hỏi server `.vn` tại IP $X$"*.
  2. Client gửi tiếp tới server `.vn` $\rightarrow$ Server `.vn` trả về: *"Hãy hỏi server `edu.vn` tại IP $Y$"*.
  3. Client hỏi tiếp server `edu.vn` $\rightarrow$ Nhận địa chỉ IP cuối cùng.
* **Đặc tính:** Giảm tải tối đa cho các Root và TLD Server; Client phải chịu toàn bộ chi phí trễ mạng tích lũy.

#### B. Phân giải Đệ quy (Recursive Name Resolution)
* **Quy trình:** Client chỉ hỏi một lần tới Local DNS Server; Local DNS Server tự động đóng vai trò trung gian hỏi Root $\rightarrow$ Root hỏi TLD $\rightarrow$ TLD hỏi Authoritative Server; kết quả được truyền ngược lại dọc theo chuỗi gọi.
* **Đặc tính:** Cho phép tận dụng bộ đệm (Caching) ở tất cả các trạm trung gian; nhưng máy chủ trung gian phải duy trì trạng thái kết nối và chịu tải lớn.

```
PHÂN GIẢI KHÔNG ĐỆ QUY (Iterative)         PHÂN GIẢI ĐỆ QUY (Recursive)

    [ Root ]      [ TLD ]                     [ Root ]
     ^   |         ^   |                        ^   |
 (1) |   | (2) (3) |   | (4)                (2) |   | (3)
     |   v         |   v                        |   v
   [ Client / Resolver ]                      [ Local Server ] <--- (1) [ Client ]
                                                ^   |          ---> (6)
                                            (4) |   | (5)
                                                |   v
                                              [ Authoritative ]
```

---

### 3.5. Hệ thống Tên miền DNS (Domain Name System)

DNS là một cơ sở dữ liệu phân tán, phân cấp dùng để ánh xạ tên miền sang địa chỉ IP.

#### Các loại Bản ghi Tài nguyên cốt lõi (Resource Records - RR):
* **SOA (Start of Authority):** Khai báo các tham số cấu hình chính của zone (thời gian làm tươi, thời gian TTL, email admin).
* **A (Address):** Ánh xạ tên máy (hostname) sang địa chỉ IPv4 ($32\text{-bit}$).
* **AAAA:** Ánh xạ tên máy sang địa chỉ IPv6 ($128\text{-bit}$).
* **NS (Name Server):** Khai báo máy chủ có thẩm quyền quản lý zone con được ủy quyền (*Delegation*).
* **CNAME (Canonical Name):** Bí danh (Alias) trỏ về tên chính thức của máy chủ (tương đương Soft Link).
* **MX (Mail Exchange):** Khai báo máy chủ tiếp nhận thư điện tử cho domain kèm theo mức độ ưu tiên.
* **PTR (Pointer):** Dùng cho phân giải ngược (Reverse DNS lookup): Ánh xạ từ địa chỉ IP sang tên miền ngược lại (trong zone `in-addr.arpa`).

---

## 4. ĐỊNH DANH DỰA TRÊN THUỘC TÍNH (ATTRIBUTE-BASED NAMING)

Khi người dùng **không biết chính xác tên hoặc ID** của thực thể mà chỉ biết các thuộc tính mô tả (Ví dụ: *"Tìm máy in màu hai mặt, tốc độ $> 30\text{ trang/phút}$ ở tầng 3"*), hệ thống phải chuyển sang mô hình **Dịch vụ thư mục (Directory Services)**.

### 4.1. Khung mô tả RDF (Resource Description Framework)
Dữ liệu được mô tả dưới dạng bộ ba (**Subject, Predicate, Object**):
* *Ví dụ:* `(Eric_Miller, personalTitle, "Dr.")`, `(Book_A, author, Tolkien)`.

### 4.2. Giao thức LDAP (Lightweight Directory Access Protocol)
* Kế thừa gọn nhẹ từ chuẩn X.500 của ITU-T.
* Thông tin tổ chức thành cây thông tin thư mục **DIT (Directory Information Tree)**:
  * **C (Country):** Quốc gia (ví dụ: `C=VN`).
  * **O (Organization):** Tổ chức/Doanh nghiệp (`O=HUST`).
  * **OU (Organizational Unit):** Phòng ban/Khoa (`OU=SoICT`).
  * **CN (Common Name):** Tên định danh thực thể (`CN=Main Server`).
* **Định danh phân biệt (Distinguished Name - DN):** Tương đương đường dẫn tuyệt đối:
  $$\text{DN} = \text{CN=Main Server, OU=SoICT, O=HUST, C=VN}$$
* **Định danh phân biệt tương đối (Relative Distinguished Name - RDN):** Nhãn của từng nút đơn lẻ trên cây.

---

### 4.3. Ánh xạ Thư mục vào DHT: AVTree (Attribute-Value Tree)

Làm sao hỗ trợ tìm kiếm theo thuộc tính trên một mạng P2P dùng giải thuật băm phân tán (DHT Chord)?

```
                [ Root: type=book ]
                 /               \
         [ author=Tolkien ]     [ genre=fantasy ]
                 |                       |
          [ title=LOTR ]           [ title=LOTR ]
```

* **Cơ chế băm chuỗi tiền tố (Prefix Path Hashing):**
  * Nút mạng băm toàn bộ các đường dẫn từ gốc tới nút lá thành các khóa:
    $$h_1 = \text{Hash}(\text{"type=book"})$$
    $$h_2 = \text{Hash}(\text{"type=book-author=Tolkien"})$$
    $$h_3 = \text{Hash}(\text{"type=book-author=Tolkien-title=LOTR"})$$
  * Khi client tìm kiếm: *"Tìm sách của Tolkien"*, hệ thống chỉ cần tính $h_2$ và gửi lệnh truy vấn tới đúng nút chịu trách nhiệm quản lý khóa $h_2$ trên vòng Chord.

---

### 4.4. Mạng Lớp phủ Ngữ nghĩa (Semantic Overlay Networks - SON)

* **Tư tưởng:** Trong mạng chia sẻ file P2P, thay vì kết nối các nút ngẫu nhiên, hệ thống kết nối các nút có **sở thích hoặc danh mục tài nguyên tương đồng nhau** thành các cụm ngữ nghĩa.
* **Mô hình Gossiping 2 tầng (Two-layer Gossip Protocol):**
  1. *Tầng dưới (Random Overlay):* Chạy giao thức lan truyền tin đồn ngẫu nhiên để duy trì kết nối mạng toàn cục và khám phá các nút mới.
  2. *Tầng trên (Semantic Overlay):* Dùng **hàm khoảng cách ngữ nghĩa (Semantic Proximity Function)** để lọc và kết nối trực tiếp với các nút có tỷ lệ trùng lặp danh sách tệp tin cao nhất.

---

## 5. AN NINH VÀ CẠNH BIÊN TRONG ĐỊNH DANH

### 5.1. Tấn công đầu độc bộ đệm ARP (ARP Spoofing / Poisoning)
* **Lỗ hổng:** Giao thức ARP hoàn toàn không có cơ chế xác thực. Một máy tính có thể gửi bản tin ARP Reply giả mạo dù không ai hỏi (Gratuitous ARP).
* **Kịch bản:** Kẻ tấn công gửi liên tục các bản tin ARP giả:
  * Gửi tới nạn nhân: *"IP của Default Gateway đang mang MAC của Kẻ tấn công"*.
  * Gửi tới Router: *"IP của Nạn nhân đang mang MAC của Kẻ tấn công"*.
* **Hậu quả:** Toàn bộ lưu lượng Internet của nạn nhân bị chuyển hướng qua máy tấn công (Man-In-The-Middle - MITM).

### 5.2. Tấn công khuếch đại DNS (DNS Amplification Attack)
* **Lỗ hổng:** DNS chạy trên nền **UDP** (không cần bắt tay 3 bước, IP nguồn dễ dàng bị giả mạo - IP Spoofing).
* **Cơ chế tấn công:**
  1. Kẻ tấn công gửi gói tin DNS Query có kích thước nhỏ ($64\text{ bytes}$) tới một Open Recursive DNS Server.
  2. Trong gói tin, kẻ tấn công giả mạo IP nguồn thành IP của **Nạn nhân (Victim)**.
  3. Lệnh truy vấn yêu cầu trả về toàn bộ bản ghi của một domain lớn (`dig ANY isc.org`).
  4. DNS Server phản hồi gói tin có kích thước khổng lồ ($3.223\text{ bytes}$) dội thẳng về IP của Nạn nhân.
* **Hệ số khuếch đại:** Lên tới **$50$ đến $70$ lần**, làm tê liệt hoàn toàn đường truyền của nạn nhân bằng lưu lượng phản hồi cực lớn (DDoS).

---

## 6. ACTIVE RECALL: BÀI TẬP VÀ CÂU HỎI BẪY TƯ DUY

### 6.1. Câu hỏi Kiểm tra Bản chất

**Câu 1:** *Tại sao bảng ánh xạ tên tập trung (Centralized Mapping Table) thất bại hoàn toàn trên quy mô Internet, dù nó giải quyết bài toán phân giải tên cực kỳ đơn giản và nhất quán?*
* **Giải đáp:** Mô hình tập trung vi phạm cả 3 chiều co giãn:
  1. *Quy mô (Size):* Trở thành điểm nghẽn cổ chai (Bottleneck) chịu tải hàng tỷ request/giây; tạo thành điểm lỗi đơn (SPOF) - máy chủ này sập là toàn bộ Internet tê liệt.
  2. *Địa lý (Geography):* Người dùng ở cách xa máy chủ nửa vòng trái đất sẽ chịu độ trễ đường truyền khổng lồ.
  3. *Quản trị (Administration):* Không một quốc gia hay tổ chức nào chấp nhận để một thực thể duy nhất độc quyền quản lý toàn bộ dữ liệu định danh của mình.

**Câu 2:** *Sự khác biệt bản chất giữa việc xóa file có Hard Link và xóa file là mục tiêu của Soft Link trong hệ điều hành UNIX là gì?*
* **Giải đáp:**
  * Khi xóa file có **Hard Link**: Lệnh `rm` chỉ đơn thuần giảm biến đếm số liên kết trong Inode đi 1 (`link_count--`). Dữ liệu trên đĩa chỉ thực sự bị thu hồi khi `link_count == 0`. Các Hard Link còn lại vẫn truy cập dữ liệu bình thường.
  * Khi xóa file là mục tiêu của **Soft Link**: File gốc bị xóa $\rightarrow$ Inode gốc và Data Block bị giải phóng. Soft Link vẫn tồn tại nhưng nội dung đường dẫn bên trong nó giờ đây trỏ vào một vùng không tồn tại $\rightarrow$ Trở thành **Dangling Pointer (Liên kết gãy)**.

---

### 6.2. Bài tập Tính toán Chord DHT Tự luyện

**Đề bài:** Cho mạng Chord DHT với $m = 4$ ($16$ vị trí từ $0 \rightarrow 15$). Các nút mạng hiện đang online gồm:
$$\mathcal{N} = \{0, 3, 7, 10, 14\}$$

**Yêu cầu:**
1. Hãy lập bảng **Finger Table** cho Nút $p = 7$.
2. Trình bày chi tiết từng chặng phân giải để tìm kiếm vị trí lưu trữ của khóa $k = 2$ bắt đầu từ Nút $7$.

#### Đáp án:
1. **Finger Table của Nút 7 ($m=4$):**
   * $i = 1$: $\text{succ}((7 + 1) \pmod{16}) = \text{succ}(8) = 10$
   * $i = 2$: $\text{succ}((7 + 2) \pmod{16}) = \text{succ}(9) = 10$
   * $i = 3$: $\text{succ}((7 + 4) \pmod{16}) = \text{succ}(11) = 14$
   * $i = 4$: $\text{succ}((7 + 8) \pmod{16}) = \text{succ}(15) = 0$  
   $$\Rightarrow \text{Finger Table}(Node\ 7) = [10, 10, 14, 0]$$

2. **Truy vết tìm khóa $k = 2$ từ Nút 7:**
   * Nút 7 cần tìm khóa $2$. Trên vòng tròn modulo 16, khoảng cách từ 7 qua 0 tới 2:
   * Node 7 kiểm tra Finger Table $[10, 10, 14, 0]$. Nút lớn nhất đứng trước $2$ theo chiều kim đồng hồ là **Node 0**.
   * Node 7 chuyển tiếp truy vấn tới **Node 0**.
   * Tại Node 0: $\text{succ}((0 + 1) \pmod{16}) = \text{succ}(1) = 3$. Vì $0 < 2 \le 3$, kết luận khóa $k=2$ được lưu trữ tại **Node 3**.
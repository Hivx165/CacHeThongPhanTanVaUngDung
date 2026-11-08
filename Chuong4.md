# BÀI GIẢNG CHUYÊN SÂU: CHƯƠNG 4 - ĐỒNG BỘ HÓA TRONG HỆ PHÂN TÁN

*Môn học: Các hệ thống phân tán và ứng dụng (Distributed Systems & Applications)*

*Khoa Công nghệ Thông tin & Truyền thông - ĐH Bách Khoa Hà Nội (HUST)*

---

## 1. NGUYÊN LÝ GỐC: TẠI SAO BÀI TOÁN ĐỒNG BỘ LẠI KHÓ TRONG HỆ PHÂN TÁN?

Trong một máy tính đơn lẻ (Single Machine), bài toán đồng bộ hóa và thứ tự rất đơn giản vì:
* Có một **đồng hồ phần cứng chung** (Centralized Clock/Oscillator).
* Có một **vùng nhớ chia sẻ chung** (Shared Memory) quản lý bởi một hệ điều hành duy nhất.

Khi bước sang môi trường phân tán:
1. **Không có đồng hồ vật lý dùng chung (No Global Physical Clock):** Mỗi máy tính sở hữu một bộ dao động thạch anh riêng biệt. Do các yếu tố vật lý (nhiệt độ, từ trường, lão hóa tinh thể), các đồng hồ này chạy với tốc độ khác nhau $\rightarrow$ hiện tượng **Lệch pha đồng hồ (Clock Drift)**.
2. **Không có bộ nhớ chia sẻ (No Shared Memory):** Các tiến trình chỉ có thể phối hợp thông qua việc gửi thông điệp mạng (Message Passing) qua đường truyền có độ trễ bất định (Unpredictable Latency).
3. **Thảm họa logic khi đồng hồ không khớp (Ví dụ lệnh `make` biên dịch C):**
   * Giả sử máy biên dịch (Compiler) có đồng hồ chạy nhanh: $2144 \rightarrow 2147$.
   * Máy soạn thảo (Editor) có đồng hồ chạy chậm: $2142 \rightarrow 2145$.
   * Lập trình viên sửa file `output.c` lúc $2143$ (theo giờ Editor) và gửi đi biên dịch. Compiler tạo ra file nhị phân `output.o` lúc $2144$ (theo giờ Compiler).
   * Lập trình viên sửa tiếp `output.c` lúc $2143.5$. Nhưng theo giờ Editor, file chỉ mang timestamp $2143.5$, trong khi file nhị phân `output.o` đã mang timestamp $2144$.
   * Trình quản lý biên dịch `make` kiểm tra: Thấy `output.o` ($2144$) mới hơn `output.c` ($2143.5$) $\rightarrow$ **Bỏ qua không biên dịch lại code mới!**

---

## 2. ĐỒNG BỘ HÓA ĐỒNG HỒ VẬT LÝ (PHYSICAL CLOCK SYNCHRONIZATION)

### 2.1. Bản chất phần cứng của Đồng hồ máy tính
* Đồng hồ máy tính gồm một tinh thể thạch anh dao động ở tần số xác định, kết hợp với:
  * **Counter (Bộ đếm):** Giảm giá trị sau mỗi dao động.
  * **Holding Register:** Nạp lại giá trị ban đầu khi Counter về $0$ và phát tín hiệu ngắt (**Clock Tick**).
* **Clock Skew (Độ chênh lệch):** Sự khác biệt tức thời về giá trị hiển thị giữa 2 đồng hồ tại một thời điểm $t$: $C_A(t) - C_B(t)$.
* **Clock Drift (Tốc độ trôi dạt):** Sự sai khác về tốc độ đếm thời gian thực theo thời gian: $\frac{dC}{dt} \ne 1$.

---

### 2.2. Giao thức Đồng bộ Thời gian Mạng: NTP (Network Time Protocol)

NTP vận hành theo mô hình Client-Server nhằm xác định độ lệch pha $\theta$ và độ trễ khứ hồi $\delta$ giữa Client $A$ và Time Server $B$.

#### Sơ đồ trao đổi gói tin NTP:
```
  Nút B (Server)                 T2 (Nhận Req)       T3 (Gửi Res)
  -------------------------------------+-----------------+--------> Thời gian B
                                      ^                 |
                                     /                   \
                                    /                     \
  ---------------------------------+-----------------------+------> Thời gian A
  Nút A (Client)             T1 (Gửi Req)               T4 (Nhận Res)
```

* $T_1$: Thời điểm $A$ gửi bản tin yêu cầu (theo giờ của $A$).
* $T_2$: Thời điểm $B$ nhận bản tin yêu cầu (theo giờ của $B$).
* $T_3$: Thời điểm $B$ gửi bản tin phản hồi (theo giờ của $B$).
* $T_4$: Thời điểm $A$ nhận bản tin phản hồi (theo giờ của $A$).

#### Công thức toán học tính toán:
1. **Tổng thời gian trễ khứ hồi (Round-Trip Delay $\delta$):**
   $$\delta = (T_4 - T_1) - (T_3 - T_2)$$
2. **Độ lệch pha đồng hồ giữa A và B (Clock Offset $\theta$):**
   Giả định thời gian truyền tin hai chiều là đối xứng ($dT_{req} \approx dT_{res} = \frac{\delta}{2}$):
   $$T_2 = T_1 + dT_{req} + \theta$$
   $$T_4 = T_3 + dT_{res} - \theta$$
   Trừ hai phương trình cho nhau, ta rút ra công thức Offset chuẩn:
   $$\theta = \frac{(T_2 - T_1) + (T_3 - T_4)}{2}$$

* **Cơ chế điều chỉnh (Slewing):** Nếu đồng hồ của $A$ chạy chậm ($\theta > 0$), $A$ tăng nhịp đếm. Nếu chạy nhanh ($\theta < 0$), **tuyệt đối không được vặn lùi đồng hồ** (gây lỗi trật tự file hệ thống), mà phải làm chậm dần tốc độ tăng clock tick cho đến khi khớp giờ chuẩn.

---

### 2.3. Giải thuật Berkeley (Tập trung, Phân tán nội bộ)

Khác với NTP lấy giờ tuyệt đối bên ngoài (UTC/GPS), Berkeley đồng bộ **thời gian nội bộ tương đối** trong một mạng đóng:
1. Một nút đóng vai trò chủ động là **Time Daemon (Master)** định kỳ gửi yêu cầu thăm dò giờ tới tất cả các nút con (**Slaves**).
2. Các slave gửi lại giá trị thời gian cục bộ của mình.
3. Time Daemon ước lượng độ lệch của từng slave dựa trên độ trễ RTT, **loại bỏ các giá trị ngoại lai (outliers)** do máy lỗi/độ trễ bất thường gây ra, sau đó tính **giá trị trung bình cộng**.
4. Time Daemon không gửi lại giờ tuyệt đối, mà gửi **lượng gia giảm cần hiệu chỉnh ($\pm \Delta t$)** cho từng máy (kể cả chính nó).

```
  [Time Daemon: 3:00] -- Gửi thăm dò --> [Slave 1: 2:50] và [Slave 2: 3:25]
  [Slave 1: trả lời -10m] | [Slave 2: trả lời +25m] | [Daemon: 0m]
  --> Trung bình hiệu chỉnh = (-10 + 25 + 0) / 3 = +5 phút.
  --> Daemon gửi lệnh:
      + Slave 1: Hiệu chỉnh +15 phút (2:50 + 15m = 3:05)
      + Slave 2: Hiệu chỉnh -20 phút (3:25 - 20m = 3:05, làm chậm tick)
      + Daemon : Tự hiệu chỉnh +5 phút (3:00 + 5m  = 3:05)
```

---

### 2.4. Đồng bộ trong Mạng Không dây: Giải thuật RBS (Reference Broadcast Synchronization)

Trong mạng cảm biến không dây (WSN), đường truyền (Critical Path) bị ảnh hưởng nặng bởi thời gian đóng gói tin (Message Preparation) và trễ truy cập kênh truyền (NIC Access Time).

* **Tư tưởng RBS:** Nút phát $A$ phát một bản tin quảng bá tham chiếu (Reference Broadcast) không chứa timestamp.
* Hai nút nhận $B$ và $C$ ghi nhận thời điểm nhận gói tin đó theo đồng hồ cục bộ của riêng mình ($T_{B,k}$ và $T_{C,k}$).
* $B$ và $C$ trao đổi thời điểm nhận để tính toán độ lệch:
  $$\text{Offset}[B, C] = \frac{1}{M}\sum_{k=1}^{M}(T_{B, k} - T_{C, k})$$
* **Ưu điểm:** Triệt tiêu hoàn toàn sự bất định về thời gian đóng gói và tranh chấp kênh truyền ở phía phát.

---

## 3. ĐỒNG BỘ HÓA ĐỒNG HỒ LOGIC (LOGICAL CLOCKS)

Leslie Lamport chứng minh rằng: Trong nhiều bài toán phân tán, **ta không cần biết chính xác sự kiện xảy ra vào mấy giờ mấy phút theo giờ vật lý**, mà chỉ cần mọi nút **thống nhất tuyệt đối về trật tự nhân quả trước - sau giữa các sự kiện**.

### 3.1. Quan hệ "Xảy ra trước" (Happens-Before Relation: $\to$)

1. Nếu $a$ và $b$ là 2 sự kiện trong cùng một tiến trình, và $a$ xuất hiện trước $b$, thì $a \to b$.
2. Nếu $a$ là sự kiện gửi thông điệp $m$, và $b$ là sự kiện nhận thông điệp $m$ đó ở tiến trình khác, thì $a \to b$ (hệ quả không thể xảy ra trước nguyên nhân).
3. **Tính chất bắc cầu (Transitivity):** Nếu $a \to b$ và $b \to c$ thì $a \to c$.
4. **Sự kiện Tương tranh (Concurrent Events - $a \parallel b$):** Nếu $a \not\to b$ và $b \not\to a$, hai sự kiện này là độc lập/tương tranh; không thể xác định quan hệ nhân quả giữa chúng.

---

### 3.2. Đồng hồ Logic Lamport (Lamport Timestamps)

Mỗi tiến trình $P_i$ duy trì một bộ đếm số nguyên cục bộ $C_i$.

#### Thuật toán cập nhật đồng hồ Lamport:
1. **Quy tắc 1 (Sự kiện nội bộ):** Trước khi một sự kiện cục bộ xảy ra tại $P_i$:
   $$C_i \leftarrow C_i + 1$$
2. **Quy tắc 2 (Gửi tin nhắn):** Khi $P_i$ gửi thông điệp $m$, nó đính kèm timestamp logic $ts(m) = C_i$.
3. **Quy tắc 3 (Nhận tin nhắn):** Khi $P_j$ nhận được thông điệp $m$ mang timestamp $ts(m)$, nó cập nhật đồng hồ:
   $$C_j \leftarrow \max(C_j, ts(m)) + 1$$

```
   P1       0 -----(6)-----(12)--- m1(ts=12) ---> [Gửi]
                             \
   P2       0 ---(8)---(16)---v---(Cập nhật max(16,12)+1=17 -> hiệu chỉnh tick)
```

#### Ứng dụng: Đảm bảo thứ tự toàn cục của quảng bá nhóm (Totally-Ordered Multicasting)
* Để đồng bộ dữ liệu trên các bản sao (Replicated Database), mọi truy vấn cập nhật phải được thực thi theo cùng một thứ tự trên mọi node.
* **Cơ chế:** Gắn timestamp Lamport vào bản tin cập nhật. Xử lý hòa (tie-breaking) khi 2 sự kiện trùng timestamp bằng ID tiến trình: $(ts, P_i) < (ts, P_j) \iff ts_i < ts_j \lor (ts_i = ts_j \land i < j)$.

---

### 3.3. Vector Clocks (Đồng hồ Vector)

* **Giới hạn của Lamport Clock:** Nếu $a \to b \Rightarrow C(a) < C(b)$. Tuy nhiên, chiều ngược lại **KHÔNG ĐÚNG**: $C(a) < C(b)$ **không thể suy ra** $a \to b$ (có thể $a$ và $b$ là tương tranh).
* **Giải pháp Vector Clock:** Cho phép phát hiện chính xác quan hệ nhân quả giữa hai sự kiện bất kỳ.

#### Cấu trúc và Quy tắc cập nhật:
Mỗi tiến trình $P_i$ trong hệ thống gồm $N$ tiến trình duy trì một vector $VC_i[1..N]$:
* $VC_i[i]$: Số sự kiện đã xảy ra tại chính $P_i$.
* $VC_i[j]$ ($j \ne i$): Số sự kiện xảy ra tại $P_j$ mà $P_i$ đã nhận biết được.

1. **Trước mỗi sự kiện tại $P_i$:** $VC_i[i] \leftarrow VC_i[i] + 1$.
2. **Khi gửi tin $m$:** Gửi kèm toàn bộ vector $ts(m) = VC_i$.
3. **Khi $P_j$ nhận tin $m$:** Cập nhật từng phần tử của vector:
   $$VC_j[k] \leftarrow \max(VC_j[k], ts(m)[k]) \quad \forall k \in [1, N]$$
   Sau đó tăng phần tử của chính mình: $VC_j[j] \leftarrow VC_j[j] + 1$.

#### So sánh hai nhãn Vector Clocks:
* $VC(a) < VC(b) \iff (\forall k: VC(a)[k] \le VC(b)[k]) \land (\exists k: VC(a)[k] < VC(b)[k]) \Rightarrow a \to b$.
* Nếu tồn tại phần tử này lớn hơn, phần tử kia nhỏ hơn $\Rightarrow$ Hai sự kiện **Tương tranh ($a \parallel b$)**.

#### Thực thi Giao tiếp Nhân quả (Causal Delivery Condition):
Một thông điệp $m$ từ $P_i$ gửi tới $P_j$ chỉ được đẩy lên tầng ứng dụng khi và chỉ khi:
1. $ts(m)[i] = VC_j[i] + 1$ (Đây là tin nhắn tiếp theo trực tiếp từ $P_i$).
2. $ts(m)[k] \le VC_j[k]$ với mọi $k \ne i$ ($P_j$ đã nhận đủ tất cả các tin nhắn tiền đề mà $P_i$ đã thấy trước khi gửi tin $m$).

---

## 4. CÁC THUẬT TOÁN LOẠI TRỪ LẪN NHAU (DISTRIBUTED MUTUAL EXCLUSION)

Đảm bảo tại một thời điểm, chỉ có **duy nhất một tiến trình** được phép truy cập vào Miền găng (Critical Section - CS) để thao tác trên tài nguyên dùng chung.

### 4.1. Giải thuật Tập trung (Centralized Algorithm)
* Một nút được chỉ định làm **Coordinator (Điều phối)** quản lý một hàng đợi (Queue).
* **Quy trình:**
  1. Tiến trình muốn vào CS gửi thông điệp `Request` cho Coordinator.
  2. Nếu CS đang rảnh, Coordinator trả lời `Grant`.
  3. Nếu CS đang bận, Coordinator không trả lời, đưa yêu cầu vào Queue.
  4. Tiến trình dùng xong CS gửi `Release`. Coordinator lấy tiến trình tiếp theo trong Queue ra và gửi `Grant`.
* **Đánh giá:** Rất đơn giản, không bị Starvation/Deadlock. Nhược điểm: Coordinator là **SPOF** và điểm nghẽn hiệu năng.

---

### 4.2. Giải thuật Phân tán Ricart-Agrawala (Dựa trên Lamport Timestamps)
Không dùng coordinator trung tâm; việc cấp quyền dựa trên sự đồng thuận tuyệt đối của tất cả các nút:

```
  [Tiến trình Pi muốn vào CS]
            |
  Quảng bá Request(ts, i) tới TẤT CẢ các tiến trình khác
            |
  Các tiến trình Pj nhận Request:
    + Nếu Pj không muốn vào CS: Gửi ngay "OK".
    + Nếu Pj đang ở trong CS: Lưu yêu cầu vào Hàng đợi cục bộ (không trả lời).
    + Nếu Pj cũng đang muốn vào CS:
         So sánh timestamp:
         - Nếu ts(Pi) < ts(Pj): Gửi "OK" cho Pi (ưu tiên yêu cầu đến trước).
         - Nếu ts(Pi) > ts(Pj): Giữ lại trong hàng đợi của Pj.
            |
  [Pi chỉ được vào CS khi nhận đủ (N - 1) phản hồi "OK"]
            |
  (Khi rời khỏi CS): Pi gửi "OK" giải phóng tất cả các tiến trình trong hàng đợi của mình.
```

* **Độ phức tạp thông điệp:** $2(N - 1)$ thông điệp cho mỗi lần vào CS ($N - 1$ Request + $N - 1$ Reply).
* **Điểm yếu chí mạng:** Có $N$ điểm chịu lỗi đơn (Single Points of Failure). Bất kỳ nút nào trong mạng bị treo/rớt mạng không thể gửi "OK", toàn bộ hệ thống bị nghẽn (Deadlock).

---

### 4.3. Giải thuật Token Ring (Vòng luân chuyển Token)
* Các tiến trình được xếp vào một vòng logic ($P_0 \to P_1 \to \dots \to P_{N-1} \to P_0$).
* Một thẻ bài duy nhất (**Token**) liên tục được truyền tuần tự quanh vòng: $P_i \to P_{(i+1) \pmod N}$.
* **Quy tắc:** Tiến trình nào có Token mới được quyền vào CS. Dùng xong (hoặc không có nhu cầu), tiến trình chuyển Token cho nút kế tiếp.
* **Đánh giá:** Không xảy ra đói tài nguyên (Starvation).
* **Sự cố:** Nếu Token bị mất (node giữ token bị crash), phải chạy giải thuật bầu chọn để tạo lại đúng một Token duy nhất; chi phí duy trì vòng ring khi có node tham gia/rời mạng.

---

### 4.4. Giải thuật Suzuki-Kasami's Broadcast
* Cải tiến giải thuật Token bằng kỹ thuật phát quảng bá yêu cầu:
  * Token mang theo một **Hàng đợi (Queue)** các tiến trình đang chờ và một mảng $LN[1..N]$ ghi nhận số lần CS đã phục vụ cho từng tiến trình.
  * Mỗi tiến trình duy trì mảng $RN[1..N]$ đếm số yêu cầu cao nhất nhận được từ các nút khác.
* Khi muốn vào CS, nút phát quảng bá bản tin `Request(i, sn)`. Nút đang giữ token sẽ nạp $i$ vào hàng đợi của Token và chuyển Token tới $i$ khi rảnh.
* Giảm thiểu số thông điệp so với Token Ring thuần túy khi hệ thống có tải thấp.

---

## 5. CÁC THUẬT TOÁN BẦU CHỌN (ELECTION ALGORITHMS)

Khi một nút điều phối (Coordinator, Master, Sequencer) bị chết, các nút còn lại phải tự động thỏa thuận bầu ra một nút mới thay thế. Mục tiêu: **Chọn ra nút đang hoạt động có ID lớn nhất**.

### 5.1. Giải thuật Kẻ bắt nạt (Bully Algorithm)

Bất kỳ tiến trình nào phát hiện Coordinator không phản hồi đều có quyền khởi xướng bầu chọn:

1. Tiến trình $P$ gửi bản tin `ELECTION` tới tất cả các tiến trình có **ID cao hơn nó** ($P_{k}$ với $k > P$).
2. Nếu không có ai trả lời sau khoảng thời gian Timeout, $P$ tự coi mình thắng cuộc và gửi bản tin `COORDINATOR` tới toàn bộ các nút trong mạng.
3. Nếu có một nút $P_k$ ($k > P$) phản hồi bằng bản tin `OK`:
   * $P$ hiểu rằng có kẻ mạnh hơn mình đang sống $\rightarrow$ $P$ nhường quyền và dừng lại.
   * $P_k$ tiếp tục lặp lại quy trình: gửi `ELECTION` lên các nút cao hơn nó.

```
Kịch bản: Nút 7 (Coordinator cũ) bị chết. Nút 4 phát hiện sự cố:
(a) Nút 4 gửi ELECTION tới 5, 6, 7.
(b) Nút 5 và 6 còn sống, gửi trả "OK" cho 4. Nút 7 không trả lời.
    Nút 4 dừng lại, chuyển quyền cho 5 và 6.
(c) Nút 5 gửi ELECTION tới 6 và 7. Nút 6 gửi ELECTION tới 7.
(d) Nút 6 trả lời "OK" cho 5. Nút 7 không trả lời. Nút 5 dừng lại.
(e) Nút 6 gửi ELECTION tới 7 (không ai trả lời) -> Nút 6 tuyên bố chiến thắng!
    Nút 6 gửi bản tin COORDINATOR tới 0, 1, 2, 3, 4, 5.
```

* **Đặc tính:** Đúng như tên gọi "Kẻ bắt nạt" - Nút có ID lớn nhất luôn chiếm quyền chỉ huy. Nếu một nút ID cao vừa phục hồi sau sự cố, nó lập tức "bắt nạt" hạ bệ Coordinator hiện tại để giành quyền.

---

### 5.2. Giải thuật Vòng tròn (Ring Algorithm)
* Các nút tổ chức thành vòng logic có hướng. Mọi thông điệp truyền theo chiều kim đồng hồ.
* **Quy trình bầu chọn:**
  1. Khi nút $P_k$ phát hiện coordinator chết, nó tạo bản tin `ELECTION` mang danh sách chứa chính nó: $[k]$, rồi gửi cho nút kế tiếp còn sống.
  2. Nút tiếp theo nhận tin, chèn ID của mình vào danh sách: $[k, k+1, \dots]$ rồi chuyển tiếp đi.
  3. Khi bản tin quay trở lại đúng nút khởi xướng ban đầu $P_k$:
     * $P_k$ quét qua toàn bộ danh sách, chọn ra **ID lớn nhất**.
     * $P_k$ phát bản tin `COORDINATOR` mang ID người thắng cuộc đi một vòng nữa để thông báo cho toàn mạng.

---

### 5.3. Bầu chọn trong Mạng Cảm biến Không dây (Wireless Ad-hoc Networks)
Trong mạng không dây, topology thay đổi liên tục, nút bị giới hạn năng lượng:
* Mỗi nút có một chỉ số năng lực (**Capacity/Resource** - ví dụ dung lượng pin còn lại, băng thông).
* Quá trình bầu chọn xây dựng một **Cây bao trùm (Spanning Tree)** thông qua cơ chế phát sóng quảng bá:
  * Nút gốc phát sóng thăm dò. Các nút lân cận nhận được bản tin đầu tiên sẽ ghi nhận nút gửi là nút cha (**Parent**).
  * Các nút lá thu thập giá trị dung lượng cao nhất từ các nhánh con, phản hồi ngược lên nút cha theo cây: `[Node_ID, Max_Capacity]`.
  * Nút gốc nhận báo cáo từ tất cả các nhánh và chọn ra nút có năng lực tối ưu nhất làm Master.

---

### 5.4. Bầu chọn Superpeer trong Mạng P2P diện rộng
Để chọn ra các nút làm **Superpeer** trong mạng quy mô lớn (hàng triệu nút), thuật toán phải thỏa mãn 4 tiêu chí:
1. Độ trễ từ các node thường đến Superpeer phải nhỏ.
2. Superpeer phải phân bố đồng đều trên không gian mạng overlay.
3. Tỷ lệ số lượng Superpeer / Regular peer phải được duy trì cố định.
4. Mỗi Superpeer không được quá tải (giới hạn số node thường kết nối).

* **Mô hình lực đẩy vật lý (Repulsion Force Model):** Các nút đang giữ token Superpeer tác động lực đẩy ảo lên nhau. Nếu hai Superpeer nằm quá gần nhau trong không gian mạng, lực đẩy sẽ khiến một nút đẩy token Superpeer dịch chuyển sang một nút thường khác ở xa hơn, đảm bảo tính phân tán đều toàn cục.

---

## 6. ACTIVE RECALL: BÀI TẬP VÀ CÂU HỎI THI KỸ THUẬT

### 6.1. Câu hỏi Bẫy Tư duy

**Câu 1:** *Tại sao trong hệ thống phân tán, khi đồng bộ thời gian bằng NTP, nếu đồng hồ của client chạy nhanh hơn server (Clock chạy trước giờ thực), ta không được phép gán lùi thời gian lại (ví dụ từ 10:05:00 set về 10:00:00)?*
* **Giải đáp:** Việc lùi thời gian sẽ phá vỡ tính đơn điệu tăng dần của thời gian ($t_{sau} > t_{truoc}$). Điều này gây ra thảm họa logic cho các hệ thống:
  * Hệ cơ sở dữ liệu và file log: Ghi đè hoặc tráo đổi trật tự giao dịch (Transaction Logs).
  * Lệnh build/make: File sinh ra lại có timestamp nằm trong quá khứ so với file mã nguồn.
  * Các bộ lập lịch (Cron jobs): Có thể kích hoạt lặp lại hai lần các tác vụ định kỳ nguy hiểm (ví dụ trừ tiền định kỳ). Do đó, hệ thống chỉ được phép dùng kỹ thuật **Slewing** (làm chậm tốc độ clock tick cho đến khi khớp giờ).

**Câu 2:** *Tại sao giải thuật phân tán Ricart-Agrawala lại ít được sử dụng trong các hệ thống sản xuất (production) thực tế so với giải thuật tập trung có dự phòng?*
* **Giải đáp:** Vì giải thuật Ricart-Agrawala đánh đổi tính khả dụng để lấy sự phân quyền thuần túy:
  1. Yêu cầu **$N$ điểm chịu lỗi**: Một nút duy nhất trong $N$ nút bị treo hoặc phân mảnh mạng (Network Partition) sẽ làm treo toàn bộ cụm.
  2. Băng thông tốn kém: Chi phí $2(N-1)$ thông điệp cho mỗi lần vào miền găng không thể mở rộng (scale) khi cụm lên đến hàng trăm máy.
  3. Cần biết trước danh sách toàn bộ $N$ tiến trình trong hệ thống (không hỗ trợ các hệ thống động có các nút thường xuyên join/leave).

---

### 6.2. Bài tập Tính toán Lamport & Vector Clocks

**Đề bài:** Cho 3 tiến trình $P_1, P_2, P_3$ với các sự kiện sau:
* $P_1$ thực hiện sự kiện $a$, sau đó gửi tin nhắn $m_1$ tới $P_2$ (sự kiện $b$).
* $P_2$ thực hiện sự kiện $c$, sau đó nhận tin nhắn $m_1$ (sự kiện $d$).
* $P_2$ gửi tin nhắn $m_2$ tới $P_3$ (sự kiện $e$).
* $P_3$ nhận tin nhắn $m_2$ (sự kiện $f$), sau đó thực hiện sự kiện nội bộ $g$.

**Yêu cầu:** Hãy tính nhãn Vector Clocks cho tất cả các sự kiện từ $a$ đến $g$.

#### Đáp án chi tiết:
Khởi tạo vector ban đầu: $VC = (0, 0, 0)$ cho cả 3 tiến trình.

1. **Tại $P_1$:**
   * Sự kiện $a$ (nội bộ): $VC(a) = (1, 0, 0)$.
   * Sự kiện $b$ (gửi $m_1$): $VC(b) = (2, 0, 0)$. Gửi tin $m_1$ với $ts(m_1) = (2, 0, 0)$.
2. **Tại $P_2$:**
   * Sự kiện $c$ (nội bộ, xảy ra trước khi nhận $m_1$): $VC(c) = (0, 1, 0)$.
   * Sự kiện $d$ (nhận $m_1$):
     $$VC(d) = (\max(0, 2), \max(1, 0) + 1, \max(0, 0)) = (2, 2, 0)$$
   * Sự kiện $e$ (gửi $m_2$): $VC(e) = (2, 3, 0)$. Gửi tin $m_2$ với $ts(m_2) = (2, 3, 0)$.
3. **Tại $P_3$:**
   * Sự kiện $f$ (nhận $m_2$):
     $$VC(f) = (\max(0, 2), \max(0, 3), \max(0, 0) + 1) = (2, 3, 1)$$
   * Sự kiện $g$ (nội bộ): $VC(g) = (2, 3, 2)$.

*Kết luận về quan hệ nhân quả:*
* So sánh $VC(a) = (1, 0, 0)$ và $VC(c) = (0, 1, 0) \Rightarrow$ **$a \parallel c$ (Hai sự kiện độc lập tương tranh)**.
* So sánh $VC(b) = (2, 0, 0)$ và $VC(g) = (2, 3, 2) \Rightarrow VC(b) < VC(g) \Rightarrow b \to g$ ($b$ là tiền đề nhân quả của $g$).
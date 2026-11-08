# BÀI GIẢNG CHUYÊN SÂU: CHƯƠNG 4 - ĐỒNG BỘ HÓA TRONG HỆ PHÂN TÁN

*Môn học: Các hệ thống phân tán và ứng dụng (Distributed Systems & Applications)*
*Khoa Công nghệ Thông tin & Truyền thông - ĐH Bách Khoa Hà Nội (HUST)*

---

## 1. NGUYÊN LÝ GỐC: TẠI SAO BÀI TOÁN ĐỒNG BỘ LẠI KHÓ TRONG HỆ PHÂN TÁN?

Trong một máy tính đơn lẻ (Single Machine), bài toán đồng bộ hóa và xác định thứ tự đơn giản hơn nhiều vì:
* Mọi tiến trình đọc cùng một **nguồn thời gian chung** (đồng hồ hệ thống của máy).
* Có một **vùng nhớ chia sẻ chung** (Shared Memory) quản lý bởi một hệ điều hành duy nhất.

Khi bước sang môi trường phân tán:
1. **Không có đồng hồ vật lý dùng chung (No Global Physical Clock):** Mỗi máy tính sở hữu một bộ dao động thạch anh riêng. Do các yếu tố vật lý (nhiệt độ, điện áp, lão hóa tinh thể), các đồng hồ chạy với tốc độ hơi khác nhau $\rightarrow$ hiện tượng **trôi đồng hồ (Clock Drift)** và dẫn tới **lệch đồng hồ (Clock Skew)**.
2. **Không có bộ nhớ chia sẻ (No Shared Memory):** Các tiến trình chỉ phối hợp qua việc gửi thông điệp (Message Passing) trên đường truyền có độ trễ bất định.
3. **Thảm họa logic khi đồng hồ không khớp (Ví dụ lệnh `make` biên dịch C):**
   * Máy chạy trình biên dịch (Compiler) có đồng hồ **chạy nhanh hơn** máy chạy trình soạn thảo (Editor). Ví dụ, cùng một khoảnh khắc thực: đồng hồ Compiler chỉ $2144$, đồng hồ Editor chỉ $2142$.
   * Lập trình viên biên dịch `output.c`, tạo ra `output.o` với timestamp $2144$ (theo giờ Compiler).
   * Ngay sau đó, lập trình viên sửa `output.c`. Do đồng hồ Editor chạy chậm, file `output.c` mới chỉ nhận timestamp $2143$ (theo giờ Editor), dù về thời gian thực nó được sửa **sau** khi `output.o` được tạo.
   * `make` so sánh: `output.o` ($2144$) có vẻ mới hơn `output.c` ($2143$) $\rightarrow$ **bỏ qua không biên dịch lại, nên file thực thi không chứa phần code vừa sửa!**

---

## 2. ĐỒNG BỘ HÓA ĐỒNG HỒ VẬT LÝ (PHYSICAL CLOCK SYNCHRONIZATION)

### 2.1. Bản chất phần cứng của Đồng hồ máy tính
* Đồng hồ máy tính gồm một tinh thể thạch anh dao động ở tần số xác định, kết hợp với:
  * **Counter (Bộ đếm):** Giảm giá trị sau mỗi dao động.
  * **Holding Register:** Nạp lại giá trị ban đầu cho Counter khi nó về $0$, đồng thời phát tín hiệu ngắt (**Clock Tick**).
* **Clock Skew (Độ lệch):** Sự khác biệt tức thời về giá trị giữa 2 đồng hồ tại một thời điểm $t$: $C_A(t) - C_B(t)$.
* **Clock Drift (Tốc độ trôi):** Sự sai khác về *tốc độ* chạy so với thời gian chuẩn: $\frac{dC}{dt} \ne 1$.
* Nhà sản xuất cho biết **tốc độ trôi tối đa $\rho$**: $1 - \rho \le \frac{dC}{dt} \le 1 + \rho$. Hai đồng hồ có thể lệch nhau tối đa $2\rho\,\Delta t$ sau khoảng $\Delta t$. Để giữ skew không vượt quá $\delta$, cần đồng bộ lại ít nhất mỗi $\frac{\delta}{2\rho}$ đơn vị thời gian.

---

### 2.2. Giao thức Đồng bộ Thời gian Mạng: NTP (Network Time Protocol)

NTP vận hành theo mô hình Client-Server (xếp theo các tầng **stratum**: stratum 0 là đồng hồ nguyên tử/GPS, stratum 1 nối trực tiếp với stratum 0, v.v.), nhằm xác định độ lệch $\theta$ và độ trễ khứ hồi $\delta$ giữa Client $A$ và Time Server $B$.

#### Sơ đồ trao đổi gói tin NTP:
```
 Client A :  T1 ---(Request)---.                      .---(Response)---> T4
                                \                    /
 Server B :                      '--> T2 ...... T3 --'
```

* $T_1$: Thời điểm $A$ gửi yêu cầu (theo giờ của $A$).
* $T_2$: Thời điểm $B$ nhận yêu cầu (theo giờ của $B$).
* $T_3$: Thời điểm $B$ gửi phản hồi (theo giờ của $B$).
* $T_4$: Thời điểm $A$ nhận phản hồi (theo giờ của $A$).

#### Công thức toán học:
1. **Độ trễ khứ hồi (Round-Trip Delay $\delta$):**
   $$\delta = (T_4 - T_1) - (T_3 - T_2)$$
2. **Độ lệch đồng hồ (Clock Offset $\theta$, tính là giờ $B$ trừ giờ $A$):**
   Giả định thời gian truyền hai chiều đối xứng ($d_{req} \approx d_{res} = \frac{\delta}{2}$):
   $$T_2 = T_1 + d_{req} + \theta$$
   $$T_4 = T_3 + d_{res} - \theta$$
   Từ đó:
   $$\theta = \frac{(T_2 - T_1) + (T_3 - T_4)}{2}$$

* **Cơ chế điều chỉnh (Slewing):** Nếu đồng hồ của $A$ chạy chậm hơn $B$ ($\theta > 0$), $A$ tăng nhịp để bắt kịp. Nếu chạy nhanh hơn ($\theta < 0$), **không nên vặn lùi đột ngột** (làm đứt tính đơn điệu của thời gian, gây lỗi thứ tự file/log); thay vào đó làm chậm dần tốc độ tăng của đồng hồ cho tới khi khớp giờ chuẩn. (Trong thực tế, các triển khai NTP chỉ "bước" - step - thời gian khi độ lệch rất lớn.)

---

### 2.3. Giải thuật Berkeley (Master-based, đồng bộ nội bộ)

Khác với NTP lấy giờ tuyệt đối bên ngoài (UTC/GPS), Berkeley đồng bộ **thời gian nội bộ** của một nhóm máy: mục tiêu là các máy thống nhất với nhau, không nhất thiết đúng với UTC.
1. Một nút đóng vai trò **Time Daemon (Master)** định kỳ gửi giờ của mình và yêu cầu các nút còn lại (**Slaves**) cho biết độ lệch so với giờ đó.
2. Các slave trả lời độ lệch của mình.
3. Time Daemon điều chỉnh độ lệch theo RTT, **loại bỏ các giá trị ngoại lai (outliers)**, rồi tính **trung bình cộng** (bao gồm cả chính nó).
4. Time Daemon không gửi lại giờ tuyệt đối mà gửi **lượng hiệu chỉnh ($\pm \Delta t$)** cho từng máy (kể cả chính nó).

```
  [Time Daemon: 3:00] -- Gửi giờ & thăm dò --> [Slave 1: 2:50] và [Slave 2: 3:25]
  [Slave 1: trả lời -10m] | [Slave 2: trả lời +25m] | [Daemon: 0m]
  --> Trung bình độ lệch = (-10 + 25 + 0) / 3 = +5 phút  => giờ chung là 3:05.
  --> Daemon gửi lệnh:
      + Slave 1: Hiệu chỉnh +15 phút (2:50 + 15m = 3:05)
      + Slave 2: Hiệu chỉnh -20 phút (3:25 - 20m = 3:05, thực hiện bằng cách làm chậm tick)
      + Daemon : Tự hiệu chỉnh +5 phút (3:00 + 5m  = 3:05)
```

---

### 2.4. Đồng bộ trong Mạng Không dây: Giải thuật RBS (Reference Broadcast Synchronization)

Trong mạng cảm biến không dây (WSN), độ bất định lớn nằm ở phía người gửi: thời gian chuẩn bị/đóng gói tin (Message Preparation) và độ trễ truy cập kênh truyền (NIC/Channel Access Time).

* **Tư tưởng RBS:** Một nút phát bản tin quảng bá tham chiếu (Reference Broadcast), không chứa timestamp.
* Các nút nhận $B$ và $C$ ghi lại thời điểm nhận bản tin theo đồng hồ cục bộ ($T_{B,k}$ và $T_{C,k}$ cho bản tin thứ $k$).
* $B$ và $C$ trao đổi các thời điểm này để tính độ lệch (trung bình trên $M$ bản tin quảng bá):
  $$\text{Offset}[B, C] = \frac{1}{M}\sum_{k=1}^{M}(T_{B, k} - T_{C, k})$$
* **Ưu điểm:** Loại bỏ độ bất định phía phát (send time, access time) vì mọi nút nhận cùng một bản tin vật lý gần như đồng thời; chỉ còn sai số nhỏ ở phía nhận. Việc lấy trung bình nhiều bản tin giúp giảm sai số, và có thể dùng hồi quy tuyến tính để ước lượng cả độ trôi.

---

## 3. ĐỒNG BỘ HÓA ĐỒNG HỒ LOGIC (LOGICAL CLOCKS)

Leslie Lamport (1978) chỉ ra rằng trong nhiều bài toán phân tán, **ta không cần biết chính xác sự kiện xảy ra lúc mấy giờ theo giờ vật lý**, mà chỉ cần các nút **thống nhất về thứ tự nhân quả trước - sau giữa các sự kiện**.

### 3.1. Quan hệ "Xảy ra trước" (Happens-Before Relation: $\to$)

1. Nếu $a$ và $b$ là 2 sự kiện trong cùng một tiến trình, và $a$ xuất hiện trước $b$, thì $a \to b$.
2. Nếu $a$ là sự kiện gửi thông điệp $m$, và $b$ là sự kiện nhận chính $m$ đó ở tiến trình khác, thì $a \to b$ (nguyên nhân phải xảy ra trước hệ quả).
3. **Tính chất bắc cầu (Transitivity):** Nếu $a \to b$ và $b \to c$ thì $a \to c$.
4. **Sự kiện Tương tranh (Concurrent Events - $a \parallel b$):** Nếu $a \not\to b$ và $b \not\to a$, hai sự kiện độc lập; không có quan hệ nhân quả giữa chúng.

Quan hệ $\to$ là một **thứ tự bán phần (partial order)**: không phải mọi cặp sự kiện đều so sánh được.

---

### 3.2. Đồng hồ Logic Lamport (Lamport Timestamps)

Mỗi tiến trình $P_i$ duy trì một bộ đếm số nguyên cục bộ $C_i$.

#### Thuật toán cập nhật đồng hồ Lamport:
1. **Quy tắc 1 (Sự kiện nội bộ):** Trước khi một sự kiện xảy ra tại $P_i$:
   $$C_i \leftarrow C_i + 1$$
2. **Quy tắc 2 (Gửi tin nhắn):** Khi $P_i$ gửi $m$ (sau khi áp dụng Quy tắc 1 cho sự kiện gửi), nó đính kèm $ts(m) = C_i$.
3. **Quy tắc 3 (Nhận tin nhắn):** Khi $P_j$ nhận $m$ mang $ts(m)$:
   $$C_j \leftarrow \max(C_j, ts(m)) + 1$$

#### Ví dụ:
```
  P1:  e1(C=1) ---- gửi m (C=2, ts(m)=2) ------------------.
                                                            \
  P2:  f1(C=1) -- f2(C=2) ------------- nhận m: max(2,2)+1 = 3
```
Nếu $P_2$ đang có $C_2 = 16$ khi nhận $m$ với $ts(m) = 12$ thì $C_2 \leftarrow \max(16, 12) + 1 = 17$. Nếu $C_2 = 5$, đồng hồ "nhảy" lên $13$.

**Tính chất:** $a \to b \Rightarrow C(a) < C(b)$ (chiều ngược lại không đúng, xem §3.3).

#### Ứng dụng: Quảng bá nhóm có thứ tự toàn cục (Totally-Ordered Multicasting)
* Để các bản sao (Replicated Database) nhất quán, mọi thao tác cập nhật phải được thực thi theo **cùng một thứ tự** trên mọi bản sao.
* **Cơ chế:** Mỗi bản tin cập nhật mang timestamp Lamport. Mỗi tiến trình đưa bản tin nhận được vào hàng đợi cục bộ sắp theo timestamp và quảng bá xác nhận (ACK) cho mọi tiến trình. Một bản tin chỉ được chuyển cho ứng dụng khi nằm ở đầu hàng đợi và **đã được mọi tiến trình xác nhận** (đảm bảo không còn bản tin nào có timestamp nhỏ hơn đang trên đường).
* **Phá hòa (tie-breaking)** khi timestamp trùng nhau bằng ID tiến trình:
  $$(ts_i, i) < (ts_j, j) \iff ts_i < ts_j \ \lor\ (ts_i = ts_j \land i < j)$$

---

### 3.3. Vector Clocks (Đồng hồ Vector)

* **Giới hạn của Lamport Clock:** $a \to b \Rightarrow C(a) < C(b)$. Tuy nhiên, chiều ngược lại **KHÔNG ĐÚNG**: $C(a) < C(b)$ **không suy ra** $a \to b$ (có thể $a \parallel b$).
* **Vector Clock** cho phép phát hiện chính xác quan hệ nhân quả giữa hai sự kiện bất kỳ.

#### Cấu trúc và Quy tắc cập nhật:
Mỗi tiến trình $P_i$ trong hệ thống $N$ tiến trình duy trì vector $VC_i[1..N]$:
* $VC_i[i]$: Số sự kiện đã xảy ra tại chính $P_i$.
* $VC_i[j]$ ($j \ne i$): Số sự kiện xảy ra tại $P_j$ mà $P_i$ biết được.

1. **Trước mỗi sự kiện tại $P_i$:** $VC_i[i] \leftarrow VC_i[i] + 1$.
2. **Khi gửi tin $m$:** Gửi kèm toàn bộ vector $ts(m) = VC_i$.
3. **Khi $P_j$ nhận tin $m$:** Cập nhật từng phần tử:
   $$VC_j[k] \leftarrow \max(VC_j[k], ts(m)[k]) \quad \forall k \in [1, N]$$
   rồi tăng phần tử của chính mình: $VC_j[j] \leftarrow VC_j[j] + 1$.

*Chi phí:* mỗi thông điệp mang $O(N)$ số nguyên.

#### So sánh hai nhãn Vector Clocks:
* $a \to b \iff VC(a) < VC(b)$, trong đó $VC(a) < VC(b)$ nghĩa là $(\forall k: VC(a)[k] \le VC(b)[k]) \land (\exists k: VC(a)[k] < VC(b)[k])$.
* Nếu có phần tử $VC(a)[k] < VC(b)[k]$ và phần tử khác $VC(a)[l] > VC(b)[l]$ thì $a \parallel b$.

#### Giao tiếp Nhân quả (Causal Delivery):
Với mục đích này, người ta dùng một biến thể: $VC_i[i]$ **chỉ tăng khi $P_i$ gửi thông điệp** (tức là $VC_i[i]$ đếm số thông điệp $P_i$ đã gửi) và khi nhận thông điệp thì không tăng. Một thông điệp $m$ từ $P_i$ tới $P_j$ chỉ được chuyển lên tầng ứng dụng khi:
1. $ts(m)[i] = VC_j[i] + 1$ (đây là thông điệp kế tiếp trực tiếp từ $P_i$).
2. $ts(m)[k] \le VC_j[k]$ với mọi $k \ne i$ ($P_j$ đã nhận đủ các thông điệp mà $P_i$ đã thấy trước khi gửi $m$).

Nếu chưa thỏa mãn, thông điệp được giữ lại trong bộ đệm cho tới khi đủ điều kiện.

---

## 4. CÁC THUẬT TOÁN LOẠI TRỪ LẪN NHAU (DISTRIBUTED MUTUAL EXCLUSION)

Đảm bảo tại một thời điểm, chỉ có **duy nhất một tiến trình** được vào Miền găng (Critical Section - CS) để thao tác trên tài nguyên dùng chung.

### 4.1. Giải thuật Tập trung (Centralized Algorithm)
* Một nút được chỉ định làm **Coordinator (Điều phối)** quản lý một hàng đợi (Queue).
* **Quy trình:**
  1. Tiến trình muốn vào CS gửi `Request` cho Coordinator.
  2. Nếu CS đang rảnh, Coordinator trả lời `Grant`.
  3. Nếu CS đang bận, Coordinator không trả lời (hoặc trả lời từ chối), đưa yêu cầu vào Queue.
  4. Tiến trình dùng xong CS gửi `Release`. Coordinator lấy tiến trình kế tiếp trong Queue ra và gửi `Grant`.
* **Đánh giá:** Đơn giản, công bằng (FIFO), không đói tài nguyên (Starvation), chỉ **3 thông điệp** cho mỗi lần vào CS. Nhược điểm: Coordinator là **SPOF** và điểm nghẽn hiệu năng; client không phân biệt được "Coordinator đã chết" với "đang bị chặn trong hàng đợi".

---

### 4.2. Giải thuật Phân tán Ricart-Agrawala (Dựa trên Lamport Timestamps)
Không dùng coordinator; quyền vào CS dựa trên sự đồng thuận của tất cả các nút:

```
  [Tiến trình Pi muốn vào CS]
            |
  Quảng bá Request(ts, i) tới TẤT CẢ các tiến trình khác
            |
  Các tiến trình Pj nhận Request:
    + Nếu Pj không muốn vào CS và không ở trong CS: Gửi ngay "OK".
    + Nếu Pj đang ở trong CS: Lưu yêu cầu vào hàng đợi cục bộ (không trả lời).
    + Nếu Pj cũng đang muốn vào CS:
         So sánh (timestamp, ID):
         - Nếu (ts_i, i) < (ts_j, j): Gửi "OK" cho Pi (yêu cầu của Pi có trước).
         - Ngược lại: Giữ trong hàng đợi của Pj.
            |
  [Pi chỉ được vào CS khi nhận đủ (N - 1) phản hồi "OK"]
            |
  (Khi rời khỏi CS): Pi gửi "OK" cho tất cả các tiến trình trong hàng đợi của mình.
```

* **Độ phức tạp thông điệp:** $2(N - 1)$ thông điệp cho mỗi lần vào CS ($N - 1$ Request + $N - 1$ Reply).
* **Điểm yếu:** Có $N$ điểm chịu lỗi đơn. Nếu một nút bị crash hoặc mất kết nối và không gửi "OK", mọi tiến trình đang chờ sẽ bị chặn vô hạn (trừ khi bổ sung cơ chế phát hiện lỗi).

---

### 4.3. Giải thuật Token Ring (Vòng luân chuyển Token)
* Các tiến trình xếp thành một vòng logic ($P_0 \to P_1 \to \dots \to P_{N-1} \to P_0$).
* Một thẻ bài duy nhất (**Token**) được truyền tuần tự quanh vòng: $P_i \to P_{(i+1) \bmod N}$.
* **Quy tắc:** Tiến trình nào giữ Token mới được vào CS. Dùng xong (hoặc không có nhu cầu), chuyển Token cho nút kế tiếp.
* **Đánh giá:** Không đói tài nguyên (thời gian chờ tối đa là $N-1$ lượt chuyển). Nhược điểm: Token luôn luân chuyển ngay cả khi không ai cần (tốn băng thông); nếu Token bị mất (nút giữ Token bị crash) rất khó phát hiện (không phân biệt được "Token bị mất" với "Token đang được dùng") và phải bầu chọn để tạo lại đúng một Token; cần xử lý khi có nút tham gia/rời vòng.

---

### 4.4. Giải thuật Suzuki-Kasami (Token + quảng bá yêu cầu)
* Token chỉ di chuyển khi có yêu cầu, nhờ phát quảng bá yêu cầu:
  * Token mang một **Hàng đợi (Queue)** các tiến trình đang chờ và mảng $LN[1..N]$, trong đó $LN[j]$ là số thứ tự của yêu cầu gần nhất của $P_j$ đã được phục vụ.
  * Mỗi tiến trình $P_i$ giữ mảng $RN_i[1..N]$, trong đó $RN_i[j]$ là số thứ tự yêu cầu lớn nhất $P_i$ từng nhận được từ $P_j$.
* Khi muốn vào CS mà chưa giữ Token, nút $i$ tăng số thứ tự và quảng bá `Request(i, sn)`. Mọi nút cập nhật $RN[i] = \max(RN[i], sn)$.
* Khi nút giữ Token dùng xong CS, nó đặt $LN[i] = RN_i[i]$, thêm vào Queue các nút $j$ chưa có trong Queue mà $RN_i[j] = LN[j] + 1$ (có yêu cầu chưa phục vụ), rồi chuyển Token cho nút ở đầu Queue.
* **Chi phí:** tối đa $N$ thông điệp mỗi lần vào CS ($N-1$ Request + $1$ Token), và $0$ thông điệp nếu nút đã giữ sẵn Token. Hiệu quả hơn Token Ring khi tải thấp (Token không chạy vô ích).

---

## 5. CÁC THUẬT TOÁN BẦU CHỌN (ELECTION ALGORITHMS)

Khi nút điều phối (Coordinator, Master, Sequencer) bị chết, các nút còn lại phải tự động thống nhất chọn nút mới. Mục tiêu: **chọn nút đang hoạt động có ID lớn nhất**.

### 5.1. Giải thuật Kẻ bắt nạt (Bully Algorithm)

Bất kỳ tiến trình nào phát hiện Coordinator không phản hồi đều có thể khởi xướng bầu chọn:

1. Tiến trình $P$ gửi bản tin `ELECTION` tới tất cả các tiến trình có **ID cao hơn nó**.
2. Nếu không ai trả lời sau Timeout, $P$ tự coi mình thắng và gửi `COORDINATOR` tới tất cả các nút khác.
3. Nếu có nút $P_k$ ($k > P$) phản hồi `OK`: $P$ nhường quyền và dừng lại; $P_k$ tiếp tục lặp lại quy trình, gửi `ELECTION` lên các nút có ID cao hơn nó.

```
Kịch bản: Nút 7 (Coordinator cũ) bị chết. Nút 4 phát hiện sự cố:
(a) Nút 4 gửi ELECTION tới 5, 6, 7.
(b) Nút 5 và 6 còn sống, gửi "OK" cho 4. Nút 7 không trả lời.
    Nút 4 dừng lại.
(c) Nút 5 gửi ELECTION tới 6 và 7. Nút 6 gửi ELECTION tới 7.
(d) Nút 6 trả lời "OK" cho 5. Nút 7 không trả lời. Nút 5 dừng lại.
(e) Nút 6 không nhận được phản hồi từ 7 -> Nút 6 tuyên bố chiến thắng
    và gửi COORDINATOR tới 0, 1, 2, 3, 4, 5.
```

* **Đặc tính:** Nút có ID lớn nhất còn sống luôn thắng. Nếu một nút ID cao khởi động lại sau sự cố, nó khởi xướng bầu chọn và chiếm lại vai trò Coordinator.
* **Chi phí:** trường hợp xấu nhất $O(N^2)$ thông điệp.
* **Giả định:** Mạng đồng bộ với việc phát hiện lỗi bằng timeout đáng tin cậy; khi mạng bị phân mảnh có thể sinh ra hai Coordinator (split brain).

---

### 5.2. Giải thuật Vòng tròn (Ring Algorithm)
* Các nút tổ chức thành vòng logic có hướng; thông điệp truyền theo một chiều.
* **Quy trình:**
  1. Khi nút $P_k$ phát hiện coordinator chết, nó tạo bản tin `ELECTION` mang danh sách chứa chính nó: $[k]$, gửi cho nút kế tiếp còn sống (bỏ qua nút chết).
  2. Mỗi nút nhận tin chèn ID của mình vào danh sách ($[k, a, b, \dots]$) rồi chuyển tiếp.
  3. Khi bản tin quay về đúng $P_k$: $P_k$ chọn **ID lớn nhất** trong danh sách, đổi bản tin thành `COORDINATOR` mang ID người thắng và cho nó đi thêm một vòng để thông báo cho toàn mạng.
* Nếu nhiều nút khởi xướng cùng lúc thì kết quả vẫn giống nhau (chỉ tốn thêm thông điệp).

---

### 5.3. Bầu chọn trong Mạng Không dây (Wireless Ad-hoc Networks)
Topology thay đổi liên tục, nút bị giới hạn năng lượng, nên tiêu chí bầu chọn thường là **năng lực/tài nguyên** (ví dụ pin còn lại, băng thông) thay vì chỉ ID:
* Nút khởi xướng đóng vai trò gốc của một **Cây bao trùm (Spanning Tree)**, phát bản tin `ELECTION` tới các láng giềng.
* Nút nhận bản tin `ELECTION` **đầu tiên** ghi nhận nút gửi là nút cha (**Parent**) rồi chuyển tiếp bản tin cho các láng giềng khác (các bản tin tới sau chỉ được xác nhận).
* Các nút lá phản hồi ngược lên cha thông tin `[Node_ID, Capacity]` của mình; mỗi nút trong chọn nút tốt nhất giữa bản thân và các nhánh con, rồi báo lên cha.
* Nút gốc nhận báo cáo từ mọi nhánh, chọn nút có năng lực tốt nhất làm Master và quảng bá kết quả.
* Khi có nhiều nút cùng khởi xướng, mỗi cuộc bầu chọn mang một định danh; cuộc bầu chọn có định danh "lớn hơn" sẽ thắng.

---

### 5.4. Bầu chọn Superpeer trong Mạng P2P diện rộng
Để chọn ra các **Superpeer** trong mạng quy mô lớn (hàng triệu nút), giải thuật cần đáp ứng 4 yêu cầu:
1. Các nút thường (regular peer) truy cập Superpeer với độ trễ thấp.
2. Superpeer phân bố đều trên không gian mạng overlay.
3. Tỷ lệ Superpeer / tổng số nút được giữ ở mức định trước.
4. Mỗi Superpeer chỉ phục vụ tối đa một số lượng nút thường cố định (không quá tải).

* **Cách 1 - Dựa trên DHT:** dành riêng một phần không gian định danh cho Superpeer (ví dụ $k$ bit đầu của ID đặt trước, tỷ lệ $2^{-k}$).
* **Cách 2 - Mô hình lực đẩy (Repulsion Force Model):** Các "token Superpeer" tác động lực đẩy ảo lên nhau. Nếu hai token ở quá gần nhau trong không gian overlay, lực đẩy khiến một token dịch chuyển sang một nút ở xa hơn, nhờ đó các Superpeer phân tán đều.

---

## 6. ACTIVE RECALL: BÀI TẬP VÀ CÂU HỎI THI KỸ THUẬT

### 6.1. Câu hỏi Bẫy Tư duy

**Câu 1:** *Khi đồng bộ thời gian bằng NTP, nếu đồng hồ của client chạy nhanh hơn server, tại sao không nên đặt lùi thời gian một cách đột ngột (ví dụ từ 10:05:00 về 10:00:00)?*
* **Giải đáp:** Việc lùi thời gian phá vỡ tính đơn điệu tăng của đồng hồ ($t_{sau} \ge t_{trước}$). Hậu quả:
  * CSDL và file log: ghi sai thứ tự giao dịch, hai sự kiện khác nhau có thể mang cùng hoặc đảo ngược timestamp.
  * `make`/hệ thống build: file sinh ra có timestamp nằm trong quá khứ so với file nguồn.
  * Bộ lập lịch (cron): có thể chạy lặp lại tác vụ định kỳ.
  * Giải pháp: **Slewing** (làm chậm dần tốc độ tick cho đến khi khớp giờ) và dùng đồng hồ đơn điệu (monotonic clock) khi cần đo khoảng thời gian.

**Câu 2:** *Tại sao giải thuật Ricart-Agrawala hiếm được dùng trong hệ thống thực tế so với giải thuật tập trung có dự phòng hoặc dịch vụ khóa dựa trên đồng thuận (ZooKeeper, etcd)?*
* **Giải đáp:**
  1. Có **$N$ điểm chịu lỗi**: một nút treo hoặc phân mảnh mạng làm mọi tiến trình khác bị chặn (nếu không có cơ chế phát hiện lỗi và cấu hình lại).
  2. Chi phí $2(N-1)$ thông điệp cho mỗi lần vào miền găng không co giãn khi $N$ lên hàng trăm.
  3. Cần biết trước toàn bộ danh sách $N$ tiến trình, không hợp với hệ thống động (nút liên tục tham gia/rời).

---

### 6.2. Bài tập Tính toán Lamport & Vector Clocks

**Đề bài:** Cho 3 tiến trình $P_1, P_2, P_3$ với các sự kiện:
* $P_1$ thực hiện sự kiện $a$, sau đó gửi tin nhắn $m_1$ tới $P_2$ (sự kiện $b$).
* $P_2$ thực hiện sự kiện $c$, sau đó nhận tin nhắn $m_1$ (sự kiện $d$).
* $P_2$ gửi tin nhắn $m_2$ tới $P_3$ (sự kiện $e$).
* $P_3$ nhận tin nhắn $m_2$ (sự kiện $f$), sau đó thực hiện sự kiện nội bộ $g$.

**Yêu cầu:** Tính nhãn Vector Clock (và Lamport Clock để so sánh) cho các sự kiện từ $a$ đến $g$.

#### Đáp án chi tiết:
Khởi tạo $VC = (0, 0, 0)$ cho cả 3 tiến trình.

1. **Tại $P_1$:**
   * $a$ (nội bộ): $VC(a) = (1, 0, 0)$.
   * $b$ (gửi $m_1$): $VC(b) = (2, 0, 0)$, $ts(m_1) = (2, 0, 0)$.
2. **Tại $P_2$:**
   * $c$ (nội bộ): $VC(c) = (0, 1, 0)$.
   * $d$ (nhận $m_1$): $VC(d) = (\max(0, 2), \max(1, 0) + 1, \max(0, 0)) = (2, 2, 0)$.
   * $e$ (gửi $m_2$): $VC(e) = (2, 3, 0)$, $ts(m_2) = (2, 3, 0)$.
3. **Tại $P_3$:**
   * $f$ (nhận $m_2$): $VC(f) = (\max(0, 2), \max(0, 3), \max(0, 0) + 1) = (2, 3, 1)$.
   * $g$ (nội bộ): $VC(g) = (2, 3, 2)$.

**Lamport Clock (để so sánh):** $C(a) = 1$, $C(b) = 2$; $C(c) = 1$, $C(d) = \max(1, 2) + 1 = 3$, $C(e) = 4$; $C(f) = \max(0, 4) + 1 = 5$, $C(g) = 6$.

*Kết luận về quan hệ nhân quả:*
* $VC(a) = (1, 0, 0)$ và $VC(c) = (0, 1, 0)$ không so sánh được $\Rightarrow$ **$a \parallel c$** (tương tranh).
* $VC(b) = (2, 0, 0) < VC(g) = (2, 3, 2) \Rightarrow b \to g$ ($b$ là tiền đề nhân quả của $g$).
* *Điểm khác biệt:* Lamport cho $C(c) = 1 < C(b) = 2$ nhưng $c \parallel b$ (vì $VC(c) = (0,1,0)$ và $VC(b) = (2,0,0)$ không so sánh được). Chỉ nhìn số của Lamport không thể kết luận có quan hệ nhân quả hay không; Vector Clock thì có thể.

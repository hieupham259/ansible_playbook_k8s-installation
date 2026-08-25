# Ưu tiên và Công bằng cho API (API Priority and Fairness)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/cluster-administration/flow-control/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), bài 16/18 · Kiểm chứng ở [Lab 9b](labs/LAB-9B-POD-SECURITY-VA-HARDENING.md).

Bài dài hơn 750 dòng, nhưng gần một nửa là **danh sách metric** — bỏ hẳn ở lần đọc này. Đây là
lớp bảo vệ **tính sẵn sàng** của API server, khác với các bài trước vốn bảo vệ tính bí mật và
toàn vẹn: nó lo chuyện một client hỏng làm ngập API server khiến kubelet không báo cáo được và
việc bầu chọn leader thất bại. Bài [122](122-multi-tenancy-vi.md) đã nhắc tên tính năng này;
đây là chỗ giải thích nó.

**Phải hiểu ở lần đọc này:**

- Vấn đề APF giải quyết: `--max-requests-inflight` và `--max-mutating-requests-inflight` chỉ
  giới hạn **tổng** khối lượng đang xử lý, **không đảm bảo request quan trọng nhất vẫn được phục
  vụ** khi lưu lượng cao. APF phân loại và cô lập request chi tiết hơn, cộng một lượng **xếp
  hàng có giới hạn** để không request nào bị từ chối trong các đợt bùng phát rất ngắn.
- Hai tài nguyên và vai trò từng cái: **FlowSchema** phân loại từng request đến và khớp nó với
  **đúng một** PriorityLevelConfiguration; **PriorityLevelConfiguration** định nghĩa một mức ưu
  tiên, phần ngân sách concurrency mà nó xử lý được, và hành vi xếp hàng.
- Cách khớp FlowSchema: mọi request được đối chiếu **bắt đầu từ `matchingPrecedence` nhỏ nhất
  rồi tăng dần**, và **FlowSchema khớp đầu tiên thắng**. Trường `distinguisherMethod.type` —
  `ByUser`, `ByNamespace`, hoặc để trống — quyết định request được tách thành các flow thế nào.
- Hai tầng cô lập: **giữa các mức ưu tiên**, mỗi mức có giới hạn concurrency riêng nên không làm
  đói nhau; **trong một mức ưu tiên**, thuật toán fair-queuing cộng shuffle sharding ngăn một
  flow làm đói flow khác. Khi vượt mức cho phép, trường `type` quyết định: **`Reject` trả HTTP
  429 ngay**, còn **`Queue`** thì xếp hàng.
- Hai loại đối tượng cấu hình: **bắt buộc** (`exempt` cho request không chịu flow control —
  trong đó có mọi request từ nhóm `system:masters`; và `catch-all`) và **được đề xuất** với sáu
  mức `node-high`, `system`, `leader-election`, `workload-high`, `workload-low`,
  `global-default`. **Xóa một đối tượng được đề xuất thì quá trình bảo trì khôi phục lại nó**;
  muốn tự kiểm soát phải đặt annotation `apf.kubernetes.io/autoupdate-spec` thành `false`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Toàn bộ mục *Khả năng quan sát → Metrics* | chưa học endpoint `/metrics` và Prometheus | giai đoạn 11 |
| *Số ghế mà một request chiếm dụng* và *Điều chỉnh thời gian thực thi cho request watch* | là chi tiết nội bộ của thuật toán | giai đoạn 9, khi tinh chỉnh APF thật |
| Bảng xác suất shuffle sharding theo `handSize` và `queues` | là bảng tham chiếu lúc tinh chỉnh | không cần |
| *Kịch bản server đệ quy* | cần hiểu admission webhook và API aggregation | bài [173](173-admission-webhooks-vi.md) và giai đoạn 14 |
| Chi tiết *Bảo trì các đối tượng cấu hình* — quy tắc `metadata.generation` | tra cứu khi sửa cấu hình mặc định | giai đoạn 9, khi tinh chỉnh APF thật |
| *Miễn trừ concurrency cho health check* và mục *Thực hành tốt* | là thao tác thêm hoặc sửa FlowSchema | giai đoạn 9, khi làm Lab 9b |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.29 [stable]`

Kiểm soát hành vi của Kubernetes API server trong tình huống quá tải (overload) là một
nhiệm vụ then chốt của người quản trị cluster. kube-apiserver có sẵn một số cơ chế kiểm
soát (cụ thể là các cờ dòng lệnh `--max-requests-inflight` và
`--max-mutating-requests-inflight`) để giới hạn khối lượng công việc đang xử lý dở mà nó
chấp nhận, nhằm ngăn một cơn lũ request đến làm quá tải và có thể làm sập API server.
Tuy nhiên, các cờ này không đủ để đảm bảo rằng những request quan trọng nhất vẫn được
phục vụ trong giai đoạn lưu lượng cao.

Tính năng API Priority and Fairness (APF) là một giải pháp thay thế, cải tiến so với các
giới hạn max-inflight nói trên. APF phân loại và cô lập các request theo cách chi tiết
hơn nhiều. Nó cũng đưa vào một lượng hàng đợi (queuing) có giới hạn, để không request nào
bị từ chối trong các đợt bùng phát (burst) rất ngắn. Các request được điều phối
(dispatch) ra khỏi hàng đợi bằng kỹ thuật xếp hàng công bằng (fair queuing), nhờ đó ví dụ
một controller hoạt động không đúng mực cũng không làm đói (starve) các thành phần khác
(kể cả khi cùng một mức ưu tiên).

Tính năng này được thiết kế để hoạt động tốt với các controller tiêu chuẩn — những
controller dùng informer và phản ứng với lỗi của API request bằng cơ chế lùi thời gian
theo cấp số nhân (exponential back-off) — cũng như với các client khác hoạt động theo
cách tương tự.

> **Thận trọng:**
> Một số request được phân loại là "long-running" — chẳng hạn thực thi lệnh từ xa hoặc
> theo dõi log (log tailing) — không chịu sự chi phối của bộ lọc API Priority and
> Fairness. Điều này cũng đúng với cờ `--max-requests-inflight` khi tính năng API
> Priority and Fairness không được bật. API Priority and Fairness _có_ áp dụng cho các
> request **watch**. Khi API Priority and Fairness bị tắt, các request **watch** không
> chịu giới hạn `--max-requests-inflight`.

## Bật/Tắt API Priority and Fairness (Enabling/Disabling API Priority and Fairness)

Tính năng API Priority and Fairness được điều khiển bằng một cờ dòng lệnh và được bật mặc
định. Xem
[Options](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/#options)
để có giải thích tổng quát về các tùy chọn dòng lệnh sẵn có của kube-apiserver và cách
bật/tắt chúng. Tên tùy chọn dòng lệnh cho APF là "--enable-priority-and-fairness". Tính
năng này cũng liên quan tới một API Group với: (a) một phiên bản `v1` ổn định (stable),
được giới thiệu ở 1.29 và bật mặc định; (b) một phiên bản `v1beta3`, bật mặc định và bị
đánh dấu deprecated ở v1.29. Bạn có thể tắt phiên bản beta `v1beta3` của API group bằng
cách thêm các cờ dòng lệnh sau vào lệnh khởi chạy `kube-apiserver`:

```shell
kube-apiserver \
--runtime-config=flowcontrol.apiserver.k8s.io/v1beta3=false \
 # …và các cờ khác như thường lệ
```

Cờ dòng lệnh `--enable-priority-and-fairness=false` sẽ tắt tính năng API Priority and
Fairness.

## Kịch bản server đệ quy (Recursive server scenarios)

API Priority and Fairness phải được dùng thận trọng trong các kịch bản server đệ quy
(recursive server). Đây là những kịch bản trong đó một server A, khi đang phục vụ một
request, lại phát ra một request phụ tới một server B nào đó. Thậm chí server B có thể
tiếp tục gọi ngược lại một request phụ nữa tới server A. Trong các tình huống mà kiểm
soát Priority and Fairness được áp dụng cho cả request gốc lẫn một số request phụ, dù ở
độ sâu nào của chuỗi đệ quy, đều có nguy cơ đảo ngược ưu tiên (priority inversion) và/hoặc
deadlock.

Một ví dụ về đệ quy là khi `kube-apiserver` phát ra một lời gọi admission webhook tới
server B, và trong khi phục vụ lời gọi đó, server B lại tạo thêm một request phụ ngược về
`kube-apiserver`. Một ví dụ khác về đệ quy là khi một đối tượng `APIService` chỉ thị
`kube-apiserver` ủy quyền (delegate) các request thuộc một API group nhất định cho một
server ngoài tùy chỉnh B (đây là một trong những thứ được gọi là "aggregation").

Khi đã biết chắc request gốc thuộc về một mức ưu tiên nhất định, và các request phụ chịu
kiểm soát được phân loại vào các mức ưu tiên cao hơn, thì đó là một giải pháp khả thi. Khi
các request gốc có thể thuộc bất kỳ mức ưu tiên nào, các request phụ chịu kiểm soát bắt
buộc phải được miễn trừ khỏi giới hạn của Priority and Fairness. Một cách để làm điều đó
là dùng các đối tượng cấu hình việc phân loại và xử lý, được bàn ở phần dưới. Một cách
khác là tắt hoàn toàn Priority and Fairness trên server B, dùng các kỹ thuật đã bàn ở
trên. Cách thứ ba, đơn giản nhất khi server B không phải là `kube-apiserver`, là xây dựng
server B với Priority and Fairness bị tắt ngay trong mã nguồn.

## Khái niệm (Concepts)

Có vài tính năng riêng biệt cấu thành nên API Priority and Fairness. Các request đến được
phân loại theo các thuộc tính của request bằng _FlowSchema_, rồi được gán vào các mức ưu
tiên (priority level). Các mức ưu tiên tạo ra một mức độ cô lập nhất định bằng cách duy
trì các giới hạn concurrency riêng biệt, nhờ đó các request được gán vào những mức ưu tiên
khác nhau không thể làm đói lẫn nhau. Trong phạm vi một mức ưu tiên, một thuật toán xếp
hàng công bằng (fair-queuing) ngăn các request thuộc những _flow_ khác nhau làm đói lẫn
nhau, đồng thời cho phép xếp hàng các request để lưu lượng bùng phát không gây ra lỗi
request khi tải trung bình vẫn ở mức chấp nhận được.

### Mức ưu tiên (Priority Levels)

Khi không bật APF, tổng mức concurrency trong API server bị giới hạn bởi các cờ
`--max-requests-inflight` và `--max-mutating-requests-inflight` của `kube-apiserver`. Khi
bật APF, các giới hạn concurrency do những cờ này định nghĩa được cộng lại, rồi tổng đó
được chia cho một tập _mức ưu tiên_ có thể cấu hình được. Mỗi request đến được gán vào
đúng một mức ưu tiên, và mỗi mức ưu tiên chỉ điều phối số request đồng thời tối đa bằng
giới hạn riêng của nó.

Ví dụ, cấu hình mặc định gồm các mức ưu tiên riêng cho request bầu chọn leader (leader
election), request từ các controller có sẵn (built-in controller), và request từ các Pod.
Điều này nghĩa là một Pod hoạt động không đúng mực làm ngập API server bằng request cũng
không thể ngăn việc bầu chọn leader hay các hành động của controller có sẵn thành công.

Các giới hạn concurrency của những mức ưu tiên được điều chỉnh định kỳ, cho phép các mức
ưu tiên đang dùng ít cho các mức đang bị dùng nhiều mượn tạm concurrency. Các giới hạn này
dựa trên giới hạn danh nghĩa (nominal limit) cùng các cận trên/cận dưới về lượng
concurrency mà một mức ưu tiên được phép cho mượn và được phép mượn, tất cả đều dẫn xuất
từ các đối tượng cấu hình được nhắc tới bên dưới.

### Số ghế mà một request chiếm dụng (Seats Occupied by a Request)

Mô tả về quản lý concurrency ở trên là câu chuyện nền tảng. Các request có thời lượng khác
nhau nhưng được tính như nhau tại bất kỳ thời điểm nào khi so với giới hạn concurrency của
một mức ưu tiên. Trong câu chuyện nền tảng đó, mỗi request chiếm một đơn vị concurrency.
Từ "ghế" (seat) được dùng để chỉ một đơn vị concurrency, lấy cảm hứng từ việc mỗi hành
khách trên tàu hoặc máy bay chiếm một trong số ghế cố định sẵn có.

Nhưng một số request chiếm nhiều hơn một ghế. Trong đó có những request **list** mà server
ước lượng sẽ trả về một số lượng lớn đối tượng. Người ta đã nhận thấy các request kiểu này
đặt một gánh nặng đặc biệt lớn lên server. Vì lý do đó, server ước lượng số đối tượng sẽ
được trả về và coi request đó chiếm số ghế tỉ lệ thuận với con số ước lượng ấy.

### Điều chỉnh thời gian thực thi cho request watch (Execution time tweaks for watch requests)

API Priority and Fairness có quản lý các request **watch**, nhưng việc này kéo theo vài
điểm lệch nữa so với hành vi nền tảng. Điểm thứ nhất liên quan tới việc một request
**watch** được coi là chiếm ghế của nó trong bao lâu. Tùy theo tham số của request, phản
hồi cho một request **watch** có thể bắt đầu hoặc không bắt đầu bằng các thông báo
**create** cho toàn bộ những đối tượng liên quan đã tồn tại từ trước. API Priority and
Fairness coi một request **watch** đã dùng xong ghế của nó ngay khi đợt bùng phát thông
báo ban đầu đó (nếu có) kết thúc.

Các thông báo thông thường được gửi thành một đợt bùng phát đồng thời tới tất cả các luồng
phản hồi **watch** liên quan mỗi khi server được thông báo về một sự kiện
create/update/delete đối tượng. Để tính đến khối lượng công việc này, API Priority and
Fairness coi mọi request ghi (write) là còn chiếm ghế thêm một khoảng thời gian nữa sau
khi việc ghi thực tế đã xong. Server ước lượng số thông báo sẽ phải gửi và điều chỉnh số
ghế cũng như thời gian chiếm ghế của request ghi để bao gồm cả phần việc phụ trội này.

### Xếp hàng (Queuing)

Ngay cả trong phạm vi một mức ưu tiên cũng có thể có rất nhiều nguồn lưu lượng khác nhau.
Trong tình huống quá tải, việc ngăn một luồng request làm đói các luồng khác là rất có giá
trị (đặc biệt trong trường hợp khá phổ biến là một client bị lỗi làm ngập kube-apiserver
bằng request — lý tưởng nhất thì client lỗi đó gần như không gây ảnh hưởng đo đếm được lên
các client khác). Điều này được xử lý bằng cách dùng thuật toán fair-queuing để xử lý các
request được gán vào cùng một mức ưu tiên. Mỗi request được gán vào một _flow_, được nhận
diện bằng tên của FlowSchema khớp cộng với một _flow distinguisher_ (bộ phân biệt flow) —
có thể là người dùng gửi request, namespace của tài nguyên đích, hoặc không có gì — và hệ
thống cố gắng dành trọng số xấp xỉ ngang nhau cho các request thuộc những flow khác nhau
trong cùng một mức ưu tiên. Để cho phép xử lý riêng biệt từng instance, các controller có
nhiều instance nên xác thực bằng những username khác nhau.

Sau khi phân loại một request vào một flow, tính năng API Priority and Fairness có thể gán
request đó vào một hàng đợi. Việc gán này dùng kỹ thuật gọi là shuffle sharding, giúp sử
dụng các hàng đợi tương đối hiệu quả để cách ly các flow cường độ thấp khỏi các flow cường
độ cao.

Các chi tiết của thuật toán xếp hàng có thể tinh chỉnh cho từng mức ưu tiên, cho phép
người quản trị đánh đổi giữa mức dùng bộ nhớ, tính công bằng (tính chất mà các flow độc
lập đều tiến triển được khi tổng lưu lượng vượt quá năng lực), khả năng dung nạp lưu lượng
bùng phát, và độ trễ tăng thêm do việc xếp hàng gây ra.

### Request được miễn trừ (Exempt requests)

Một số request được coi là đủ quan trọng để không phải chịu bất kỳ giới hạn nào do tính
năng này áp đặt. Các miễn trừ này ngăn một cấu hình flow control bị thiết lập sai làm vô
hiệu hóa hoàn toàn một API server.

## Tài nguyên (Resources)

API flow control gồm hai loại tài nguyên.
[PriorityLevelConfiguration](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#prioritylevelconfiguration-v1-flowcontrol-apiserver-k8s-io)
định nghĩa các mức ưu tiên sẵn có, phần ngân sách concurrency mà mỗi mức có thể xử lý, và
cho phép tinh chỉnh hành vi xếp hàng.
[FlowSchema](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#flowschema-v1-flowcontrol-apiserver-k8s-io)
được dùng để phân loại từng request đến, khớp mỗi request với đúng một
PriorityLevelConfiguration.

### PriorityLevelConfiguration

Một PriorityLevelConfiguration đại diện cho một mức ưu tiên. Mỗi
PriorityLevelConfiguration có giới hạn độc lập về số request đang xử lý dở, và các giới
hạn về số request được xếp hàng.

Giới hạn concurrency danh nghĩa của một PriorityLevelConfiguration không được khai báo
bằng một con số ghế tuyệt đối, mà bằng "nominal concurrency shares" (phần chia concurrency
danh nghĩa). Tổng giới hạn concurrency của API Server được phân phối cho các
PriorityLevelConfiguration hiện có theo tỉ lệ các phần chia này, để trao cho mỗi mức giới
hạn danh nghĩa của nó tính theo số ghế. Điều này cho phép người quản trị cluster tăng hoặc
giảm tổng lượng lưu lượng tới một server bằng cách khởi động lại `kube-apiserver` với một
giá trị khác cho `--max-requests-inflight` (hoặc `--max-mutating-requests-inflight`), và
tất cả các PriorityLevelConfiguration sẽ thấy mức concurrency tối đa được phép của mình
tăng (hoặc giảm) theo cùng một tỉ lệ.

> **Thận trọng:**
> Ở các phiên bản trước `v1beta3`, trường tương ứng trong PriorityLevelConfiguration có
> tên là "assured concurrency shares" thay vì "nominal concurrency shares". Ngoài ra, ở
> bản Kubernetes 1.25 trở về trước không có việc điều chỉnh định kỳ: các giới hạn
> nominal/assured luôn được áp dụng mà không điều chỉnh.

Các cận về lượng concurrency mà một mức ưu tiên được phép cho mượn và được phép mượn được
biểu diễn trong PriorityLevelConfiguration dưới dạng phần trăm của giới hạn danh nghĩa của
mức đó. Chúng được quy đổi thành số ghế tuyệt đối bằng cách nhân với giới hạn danh nghĩa /
100.0 rồi làm tròn. Giới hạn concurrency được điều chỉnh động của một mức ưu tiên bị ràng
buộc nằm giữa (a) cận dưới là giới hạn danh nghĩa của nó trừ đi số ghế có thể cho mượn và
(b) cận trên là giới hạn danh nghĩa của nó cộng với số ghế nó được phép mượn. Ở mỗi lần
điều chỉnh, các giới hạn động được suy ra bằng việc mỗi mức ưu tiên thu hồi lại những ghế
đã cho mượn mà gần đây lại có nhu cầu, rồi cùng nhau đáp ứng một cách công bằng nhu cầu về
ghế gần đây trên các mức ưu tiên, trong phạm vi các cận vừa mô tả.

> **Thận trọng:**
> Khi bật tính năng Priority and Fairness, tổng giới hạn concurrency của server được đặt
> bằng tổng của `--max-requests-inflight` và `--max-mutating-requests-inflight`. Không còn
> sự phân biệt nào giữa request thay đổi dữ liệu (mutating) và không thay đổi dữ liệu
> (non-mutating); nếu bạn muốn xử lý chúng riêng biệt cho một tài nguyên nào đó, hãy tạo
> các FlowSchema riêng khớp tương ứng với các verb mutating và non-mutating.

Khi lượng request đến được gán vào một PriorityLevelConfiguration vượt quá mức concurrency
cho phép của nó, trường `type` trong spec của nó quyết định điều gì sẽ xảy ra với các
request dư ra. Kiểu `Reject` nghĩa là lưu lượng vượt mức sẽ bị từ chối ngay lập tức với
lỗi HTTP 429 (Too Many Requests). Kiểu `Queue` nghĩa là các request vượt ngưỡng sẽ được
xếp hàng, với các kỹ thuật shuffle sharding và fair queuing được dùng để cân bằng tiến độ
giữa các flow request.

Cấu hình xếp hàng cho phép tinh chỉnh thuật toán fair queuing cho một mức ưu tiên. Chi
tiết thuật toán có thể đọc trong
[đề xuất cải tiến (enhancement proposal)](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/1040-priority-and-fairness),
nhưng tóm lại:

* Tăng `queues` làm giảm tỉ lệ va chạm (collision) giữa các flow khác nhau, đổi lại là
  tăng mức dùng bộ nhớ. Giá trị 1 ở đây thực chất tắt logic fair-queuing, nhưng vẫn cho
  phép request được xếp hàng.

* Tăng `queueLengthLimit` cho phép duy trì được các đợt bùng phát lưu lượng lớn hơn mà
  không phải bỏ request nào, đổi lại là tăng độ trễ và mức dùng bộ nhớ.

* Thay đổi `handSize` cho phép bạn điều chỉnh xác suất va chạm giữa các flow khác nhau và
  tổng concurrency khả dụng cho một flow đơn lẻ trong tình huống quá tải.

  > **Ghi chú:**
  > `handSize` lớn hơn làm giảm khả năng hai flow riêng lẻ va chạm nhau (và do đó giảm khả
  > năng một flow làm đói flow kia), nhưng lại làm tăng khả năng một số ít flow chiếm lĩnh
  > apiserver. `handSize` lớn hơn cũng có thể làm tăng độ trễ mà một flow lưu lượng cao
  > đơn lẻ có thể gây ra. Số request tối đa có thể được xếp hàng từ một flow đơn lẻ là
  > `handSize * queueLengthLimit`.

Dưới đây là một bảng trình bày một tập hợp thú vị các cấu hình shuffle sharding, với mỗi
cấu hình cho thấy xác suất một con chuột (flow cường độ thấp) bị các con voi (flow cường
độ cao) đè bẹp, ứng với một tập số lượng voi mang tính minh họa. Xem
https://play.golang.org/p/Gi0PLgVHiUg — chương trình tính ra bảng này.

*Bảng: Ví dụ các cấu hình Shuffle Sharding (Example Shuffle Sharding Configurations)*

| HandSize | Queues | 1 con voi | 4 con voi | 16 con voi |
|----------|--------|-----------|-----------|------------|
| 12 | 32 | 4.428838398950118e-09 | 0.11431348830099144 | 0.9935089607656024 |
| 10 | 32 | 1.550093439632541e-08 | 0.0626479840223545 | 0.9753101519027554 |
| 10 | 64 | 6.601827268370426e-12 | 0.00045571320990370776 | 0.49999929150089345 |
| 9 | 64 | 3.6310049976037345e-11 | 0.00045501212304112273 | 0.4282314876454858 |
| 8 | 64 | 2.25929199850899e-10 | 0.0004886697053040446 | 0.35935114681123076 |
| 8 | 128 | 6.994461389026097e-13 | 3.4055790161620863e-06 | 0.02746173137155063 |
| 7 | 128 | 1.0579122850901972e-11 | 6.960839379258192e-06 | 0.02406157386340147 |
| 7 | 256 | 7.597695465552631e-14 | 6.728547142019406e-08 | 0.0006709661542533682 |
| 6 | 256 | 2.7134626662687968e-12 | 2.9516464018476436e-07 | 0.0008895654642000348 |
| 6 | 512 | 4.116062922897309e-14 | 4.982983350480894e-09 | 2.26025764343413e-05 |
| 6 | 1024 | 6.337324016514285e-16 | 8.09060164312957e-11 | 4.517408062903668e-07 |

### FlowSchema

Một FlowSchema khớp với một số request đến và gán chúng vào một mức ưu tiên. Mọi request
đến đều được đối chiếu với các FlowSchema, bắt đầu từ những FlowSchema có
`matchingPrecedence` nhỏ nhất về mặt số học rồi tăng dần lên. FlowSchema khớp đầu tiên sẽ
thắng.

> **Thận trọng:**
> Chỉ FlowSchema khớp đầu tiên với một request là có ý nghĩa. Nếu nhiều FlowSchema cùng
> khớp với một request đến, request sẽ được gán dựa trên FlowSchema có
> `matchingPrecedence` cao nhất. Nếu nhiều FlowSchema có `matchingPrecedence` bằng nhau
> cùng khớp một request, FlowSchema có `name` nhỏ hơn theo thứ tự từ điển sẽ thắng, nhưng
> tốt hơn là đừng dựa vào điều này, mà hãy đảm bảo không có hai FlowSchema nào có cùng
> `matchingPrecedence`.

Một FlowSchema khớp với một request nếu ít nhất một trong các `rules` của nó khớp. Một rule
khớp nếu ít nhất một trong các `subjects` của nó *và* ít nhất một trong các `resourceRules`
hoặc `nonResourceRules` của nó (tùy thuộc request đến là cho một tài nguyên hay một URL
không phải tài nguyên) khớp với request.

Với trường `name` trong subjects, và các trường `verbs`, `apiGroups`, `resources`,
`namespaces`, `nonResourceURLs` của resource rule và non-resource rule, bạn có thể chỉ
định ký tự đại diện `*` để khớp mọi giá trị của trường đó, tức là loại trường đó ra khỏi
việc xem xét.

Trường `distinguisherMethod.type` của một FlowSchema quyết định cách các request khớp với
schema đó được tách thành các flow. Nó có thể là `ByUser`, khi đó một người dùng gửi
request sẽ không thể làm đói năng lực của những người dùng khác; `ByNamespace`, khi đó các
request cho tài nguyên trong một namespace sẽ không thể làm đói năng lực dành cho request
tới tài nguyên ở các namespace khác; hoặc để trống (hoặc bỏ hẳn `distinguisherMethod`),
khi đó mọi request khớp với FlowSchema này được coi là thuộc cùng một flow duy nhất. Lựa
chọn đúng cho một FlowSchema cụ thể phụ thuộc vào tài nguyên và môi trường cụ thể của bạn.

## Cấu hình mặc định (Defaults)

Mỗi kube-apiserver duy trì hai loại đối tượng cấu hình APF: bắt buộc (mandatory) và được
đề xuất (suggested).

### Các đối tượng cấu hình bắt buộc (Mandatory Configuration Objects)

Bốn đối tượng cấu hình bắt buộc phản ánh hành vi rào chắn (guardrail) cố định và có sẵn.
Đây là hành vi mà các server đã có trước khi những đối tượng đó tồn tại, và khi những đối
tượng đó tồn tại thì spec của chúng phản ánh hành vi này. Bốn đối tượng bắt buộc như sau.

* Mức ưu tiên bắt buộc `exempt` được dùng cho các request hoàn toàn không chịu flow
  control: chúng sẽ luôn được điều phối ngay lập tức. FlowSchema bắt buộc `exempt` phân
  loại mọi request từ nhóm `system:masters` vào mức ưu tiên này. Bạn có thể định nghĩa các
  FlowSchema khác để đưa những request khác vào mức ưu tiên này, nếu phù hợp.

* Mức ưu tiên bắt buộc `catch-all` được dùng kết hợp với FlowSchema bắt buộc `catch-all`
  để đảm bảo mọi request đều được phân loại theo cách nào đó. Thông thường bạn không nên
  dựa vào cấu hình catch-all này, mà nên tạo FlowSchema và PriorityLevelConfiguration
  catch-all của riêng mình (hoặc dùng mức ưu tiên được đề xuất `global-default` vốn được
  cài đặt mặc định) cho phù hợp. Vì nó không được kỳ vọng sẽ dùng đến trong hoạt động bình
  thường, mức ưu tiên bắt buộc `catch-all` có phần chia concurrency rất nhỏ và không xếp
  hàng request.

### Các đối tượng cấu hình được đề xuất (Suggested Configuration Objects)

Các FlowSchema và PriorityLevelConfiguration được đề xuất tạo thành một cấu hình mặc định
hợp lý. Bạn có thể sửa chúng và/hoặc tạo thêm các đối tượng cấu hình khác nếu muốn. Nếu
cluster của bạn nhiều khả năng phải chịu tải nặng thì bạn nên cân nhắc xem cấu hình nào sẽ
hoạt động tốt nhất.

Cấu hình được đề xuất nhóm các request vào sáu mức ưu tiên:

* Mức ưu tiên `node-high` dành cho các cập nhật tình trạng sức khỏe (health update) từ
  node.

* Mức ưu tiên `system` dành cho các request không liên quan tới sức khỏe từ nhóm
  `system:nodes`, tức là các Kubelet — vốn phải liên lạc được với API server thì workload
  mới có thể được lập lịch lên chúng.

* Mức ưu tiên `leader-election` dành cho các request bầu chọn leader từ các controller có
  sẵn (cụ thể là các request tới `endpoints`, `configmaps` hoặc `leases` đến từ người dùng
  `system:kube-controller-manager` hoặc `system:kube-scheduler` và các service account
  trong namespace `kube-system`). Việc cô lập chúng khỏi lưu lượng khác là quan trọng, vì
  lỗi trong bầu chọn leader khiến các controller tương ứng bị lỗi và khởi động lại, kéo
  theo lưu lượng tốn kém hơn khi các controller mới đồng bộ lại informer của chúng.

* Mức ưu tiên `workload-high` dành cho các request khác từ các controller có sẵn.

* Mức ưu tiên `workload-low` dành cho request từ bất kỳ service account nào khác, thường
  bao gồm tất cả request từ các controller đang chạy trong Pod.

* Mức ưu tiên `global-default` xử lý toàn bộ lưu lượng còn lại, ví dụ các lệnh `kubectl`
  tương tác do người dùng không có đặc quyền chạy.

Các FlowSchema được đề xuất có nhiệm vụ dẫn hướng request vào các mức ưu tiên nói trên, và
không được liệt kê ở đây.

### Bảo trì các đối tượng cấu hình bắt buộc và được đề xuất (Maintenance of the Mandatory and Suggested Configuration Objects)

Mỗi `kube-apiserver` duy trì độc lập các đối tượng cấu hình bắt buộc và được đề xuất, dùng
hành vi khởi tạo ban đầu và hành vi định kỳ. Do đó, trong tình huống có hỗn hợp các server
ở những phiên bản khác nhau, có thể xảy ra hiện tượng "giật qua giật lại" (thrashing)
chừng nào các server còn có quan điểm khác nhau về nội dung đúng của những đối tượng này.

Mỗi `kube-apiserver` thực hiện một lượt bảo trì ban đầu trên các đối tượng cấu hình bắt
buộc và được đề xuất, sau đó thực hiện bảo trì định kỳ (mỗi phút một lần) cho các đối
tượng đó.

Với các đối tượng cấu hình bắt buộc, việc bảo trì gồm đảm bảo đối tượng tồn tại và, nếu
tồn tại, thì có spec đúng. Server từ chối cho phép việc tạo mới hoặc cập nhật với một spec
không nhất quán với hành vi rào chắn của server.

Việc bảo trì các đối tượng cấu hình được đề xuất được thiết kế để cho phép ghi đè spec của
chúng. Ngược lại, việc xóa thì không được tôn trọng: quá trình bảo trì sẽ khôi phục lại
đối tượng. Nếu bạn không muốn một đối tượng cấu hình được đề xuất nào đó thì bạn cần giữ
nó lại nhưng đặt spec của nó sao cho ít gây hệ quả nhất. Việc bảo trì các đối tượng được
đề xuất cũng được thiết kế để hỗ trợ di trú (migration) tự động khi một phiên bản
`kube-apiserver` mới được triển khai, dù có thể xảy ra thrashing trong lúc còn tồn tại
hỗn hợp các server.

Việc bảo trì một đối tượng cấu hình được đề xuất gồm tạo nó — với spec do server đề xuất —
nếu đối tượng chưa tồn tại. Ngược lại, nếu đối tượng đã tồn tại, hành vi bảo trì phụ thuộc
vào việc các `kube-apiserver` hay người dùng đang kiểm soát đối tượng đó. Ở trường hợp
đầu, server đảm bảo spec của đối tượng đúng như server đề xuất; ở trường hợp sau, spec
được để nguyên.

Câu hỏi ai đang kiểm soát đối tượng được trả lời bằng cách trước hết tìm annotation có key
`apf.kubernetes.io/autoupdate-spec`. Nếu có annotation đó và giá trị của nó là `true` thì
các kube-apiserver kiểm soát đối tượng. Nếu có annotation đó và giá trị là `false` thì
người dùng kiểm soát đối tượng. Nếu không điều kiện nào ở trên đúng thì trường
`metadata.generation` của đối tượng được xem xét. Nếu nó bằng 1 thì các kube-apiserver
kiểm soát đối tượng. Ngược lại, người dùng kiểm soát đối tượng. Các quy tắc này được đưa
vào ở bản 1.22 và việc chúng xét tới `metadata.generation` là để phục vụ việc di trú từ
hành vi đơn giản hơn trước đó. Người dùng muốn kiểm soát một đối tượng cấu hình được đề
xuất nên đặt annotation `apf.kubernetes.io/autoupdate-spec` của nó thành `false`.

Việc bảo trì một đối tượng cấu hình bắt buộc hoặc được đề xuất cũng bao gồm việc đảm bảo
nó có annotation `apf.kubernetes.io/autoupdate-spec` phản ánh chính xác việc các
kube-apiserver có kiểm soát đối tượng đó hay không.

Việc bảo trì cũng bao gồm xóa những đối tượng vốn không phải bắt buộc cũng không phải được
đề xuất nhưng lại được gắn annotation `apf.kubernetes.io/autoupdate-spec=true`.

## Miễn trừ concurrency cho health check (Health check concurrency exemption)

Cấu hình được đề xuất không dành ưu đãi đặc biệt nào cho các request health check tới
kube-apiserver từ kubelet cục bộ của chính chúng — vốn thường dùng cổng bảo mật (secured
port) nhưng không cung cấp thông tin xác thực. Với cấu hình được đề xuất, các request này
được gán vào FlowSchema `global-default` và mức ưu tiên `global-default` tương ứng, nơi mà
lưu lượng khác có thể chèn ép chúng.

Nếu bạn thêm FlowSchema bổ sung sau đây, các request đó sẽ được miễn trừ khỏi việc giới
hạn tốc độ (rate limiting).

> **Thận trọng:**
> Thực hiện thay đổi này cũng cho phép bất kỳ bên có ý đồ xấu nào gửi các request health
> check khớp với FlowSchema này, với khối lượng tùy ý. Nếu bạn có bộ lọc lưu lượng web
> hoặc cơ chế bảo mật bên ngoài tương tự để bảo vệ API server của cluster khỏi lưu lượng
> internet nói chung, bạn có thể cấu hình các quy tắc để chặn mọi request health check
> xuất phát từ bên ngoài cluster.

```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: FlowSchema
metadata:
  name: health-for-strangers
spec:
  matchingPrecedence: 1000
  priorityLevelConfiguration:
    name: exempt
  rules:
    - nonResourceRules:
      - nonResourceURLs:
          - "/healthz"
          - "/livez"
          - "/readyz"
        verbs:
          - "*"
      subjects:
        - kind: Group
          group:
            name: "system:unauthenticated"
```

## Khả năng quan sát (Observability)

### Metrics

> **Ghi chú:**
> Ở các phiên bản Kubernetes trước v1.20, các label `flow_schema` và `priority_level` có
> tên không nhất quán, lần lượt là `flowSchema` và `priorityLevel`. Nếu bạn đang chạy
> Kubernetes phiên bản v1.19 trở về trước, bạn nên tham khảo tài liệu ứng với phiên bản
> của mình.

Khi bạn bật tính năng API Priority and Fairness, kube-apiserver sẽ xuất ra thêm các
metric. Việc theo dõi các metric này có thể giúp bạn xác định liệu cấu hình của mình có
đang bóp nghẹt (throttle) lưu lượng quan trọng một cách không phù hợp hay không, hoặc tìm
ra các workload hoạt động không đúng mực có thể đang gây hại cho sức khỏe hệ thống.

#### Mức độ trưởng thành BETA (Maturity level BETA)

* `apiserver_flowcontrol_rejected_requests_total` là một counter vector (tích lũy kể từ
  khi server khởi động) đếm các request bị từ chối, phân tách theo các label `flow_schema`
  (cho biết FlowSchema nào đã khớp với request), `priority_level` (cho biết mức ưu tiên mà
  request được gán vào), và `reason`. Label `reason` sẽ nhận một trong các giá trị sau:

  * `queue-full`, cho biết đã có quá nhiều request được xếp hàng.
  * `concurrency-limit`, cho biết PriorityLevelConfiguration được cấu hình để từ chối thay
    vì xếp hàng các request vượt mức.
  * `time-out`, cho biết request vẫn còn nằm trong hàng đợi khi giới hạn thời gian chờ của
    nó hết hạn.
  * `cancelled`, cho biết request không được purge lock và đã bị đẩy ra khỏi hàng đợi.

* `apiserver_flowcontrol_dispatched_requests_total` là một counter vector (tích lũy kể từ
  khi server khởi động) đếm các request đã bắt đầu thực thi, phân tách theo `flow_schema`
  và `priority_level`.

* `apiserver_flowcontrol_current_inqueue_requests` là một gauge vector giữ số lượng
  request đang được xếp hàng (chưa thực thi) tại thời điểm tức thời, phân tách theo
  `priority_level` và `flow_schema`.

* `apiserver_flowcontrol_current_executing_requests` là một gauge vector giữ số lượng
  request đang thực thi (không phải đang chờ trong hàng đợi) tại thời điểm tức thời, phân
  tách theo `priority_level` và `flow_schema`.

* `apiserver_flowcontrol_current_executing_seats` là một gauge vector giữ số ghế đang bị
  chiếm tại thời điểm tức thời, phân tách theo `priority_level` và `flow_schema`.

* `apiserver_flowcontrol_request_wait_duration_seconds` là một histogram vector về thời
  gian các request nằm trong hàng đợi, phân tách theo các label `flow_schema`,
  `priority_level` và `execute`. Label `execute` cho biết request đã bắt đầu thực thi hay
  chưa.

  > **Ghi chú:**
  > Vì mỗi FlowSchema luôn gán request vào đúng một PriorityLevelConfiguration, bạn có thể
  > cộng các histogram của tất cả FlowSchema thuộc một mức ưu tiên để có được histogram
  > hiệu dụng cho các request được gán vào mức ưu tiên đó.

* `apiserver_flowcontrol_nominal_limit_seats` là một gauge vector giữ giới hạn concurrency
  danh nghĩa của mỗi mức ưu tiên, được tính từ tổng giới hạn concurrency của API server và
  phần chia concurrency danh nghĩa được cấu hình cho mức ưu tiên đó.

#### Mức độ trưởng thành ALPHA (Maturity level ALPHA)

* `apiserver_current_inqueue_requests` là một gauge vector về các đỉnh gần đây (high water
  mark) của số request đang xếp hàng, nhóm theo một label tên `request_kind` có giá trị là
  `mutating` hoặc `readOnly`. Các đỉnh này mô tả con số lớn nhất quan sát được trong cửa
  sổ một giây vừa kết thúc gần nhất. Chúng bổ trợ cho gauge vector cũ hơn là
  `apiserver_current_inflight_requests`, vốn giữ đỉnh của cửa sổ trước đó về số request
  đang thực sự được phục vụ.

* `apiserver_current_inqueue_seats` là một gauge vector về tổng, trên các request đang xếp
  hàng, của số ghế lớn nhất mà mỗi request sẽ chiếm, nhóm theo các label `flow_schema` và
  `priority_level`.

* `apiserver_flowcontrol_read_vs_write_current_requests` là một histogram vector gồm các
  quan sát, được thực hiện vào cuối mỗi nanosecond, về số request phân tách theo các label
  `phase` (nhận giá trị `waiting` và `executing`) và `request_kind` (nhận giá trị
  `mutating` và `readOnly`). Mỗi giá trị quan sát được là một tỉ lệ, nằm giữa 0 và 1, giữa
  số request chia cho giới hạn tương ứng về số request (giới hạn dung lượng hàng đợi với
  `waiting` và giới hạn concurrency với `executing`).

* `apiserver_flowcontrol_request_concurrency_in_use` là một gauge vector giữ số ghế đang
  bị chiếm tại thời điểm tức thời, phân tách theo `priority_level` và `flow_schema`.

* `apiserver_flowcontrol_priority_level_request_utilization` là một histogram vector gồm
  các quan sát, được thực hiện vào cuối mỗi nanosecond, về số request phân tách theo các
  label `phase` (nhận giá trị `waiting` và `executing`) và `priority_level`. Mỗi giá trị
  quan sát được là một tỉ lệ, nằm giữa 0 và 1, giữa một số lượng request chia cho giới hạn
  tương ứng về số request (giới hạn dung lượng hàng đợi với `waiting` và giới hạn
  concurrency với `executing`).

* `apiserver_flowcontrol_priority_level_seat_utilization` là một histogram vector gồm các
  quan sát, được thực hiện vào cuối mỗi nanosecond, về mức sử dụng giới hạn concurrency
  của một mức ưu tiên, phân tách theo `priority_level`. Mức sử dụng này là phân số (số ghế
  đang bị chiếm) / (giới hạn concurrency). Metric này xét tất cả các giai đoạn thực thi
  (cả giai đoạn bình thường lẫn phần trễ thêm ở cuối một thao tác ghi để bù cho công việc
  gửi thông báo tương ứng) của tất cả request trừ các request WATCH; với những request đó,
  nó chỉ xét giai đoạn ban đầu gửi thông báo về các đối tượng đã tồn tại từ trước. Mỗi
  histogram trong vector cũng được gắn label `phase: executing` (không có giới hạn ghế cho
  giai đoạn chờ).

* `apiserver_flowcontrol_request_queue_length_after_enqueue` là một histogram vector về độ
  dài của các hàng đợi, phân tách theo `priority_level` và `flow_schema`, được lấy mẫu bởi
  chính các request được đưa vào hàng đợi. Mỗi request được xếp hàng đóng góp một mẫu vào
  histogram của nó, báo cáo độ dài hàng đợi ngay sau khi request đó được thêm vào. Lưu ý
  rằng cách này tạo ra thống kê khác với một khảo sát không thiên lệch.

  > **Ghi chú:**
  > Một giá trị ngoại lai (outlier) trong histogram ở đây có nghĩa là nhiều khả năng một
  > flow đơn lẻ (tức là các request từ một người dùng hoặc cho một namespace, tùy theo cấu
  > hình) đang làm ngập API server và đang bị bóp nghẹt. Ngược lại, nếu histogram của một
  > mức ưu tiên cho thấy mọi hàng đợi của mức ưu tiên đó đều dài hơn so với các mức ưu tiên
  > khác, thì có thể nên tăng phần chia concurrency của PriorityLevelConfiguration đó.

* `apiserver_flowcontrol_request_concurrency_limit` giống hệt
  `apiserver_flowcontrol_nominal_limit_seats`. Trước khi cơ chế mượn concurrency giữa các
  mức ưu tiên được đưa vào, metric này luôn bằng
  `apiserver_flowcontrol_current_limit_seats` (vốn khi đó chưa tồn tại như một metric
  riêng).

* `apiserver_flowcontrol_lower_limit_seats` là một gauge vector giữ cận dưới của giới hạn
  concurrency động của mỗi mức ưu tiên.

* `apiserver_flowcontrol_upper_limit_seats` là một gauge vector giữ cận trên của giới hạn
  concurrency động của mỗi mức ưu tiên.

* `apiserver_flowcontrol_demand_seats` là một histogram vector đếm các quan sát, vào cuối
  mỗi nanosecond, về tỉ lệ (nhu cầu ghế) / (giới hạn concurrency danh nghĩa) của mỗi mức
  ưu tiên. Nhu cầu ghế của một mức ưu tiên là tổng, trên cả các request đang xếp hàng lẫn
  các request đang ở giai đoạn thực thi ban đầu, của giá trị lớn nhất giữa số ghế bị chiếm
  ở giai đoạn thực thi ban đầu và giai đoạn thực thi cuối của request.

* `apiserver_flowcontrol_demand_seats_high_watermark` là một gauge vector giữ, cho mỗi mức
  ưu tiên, nhu cầu ghế lớn nhất quan sát được trong chu kỳ điều chỉnh mượn concurrency gần
  nhất.

* `apiserver_flowcontrol_demand_seats_average` là một gauge vector giữ, cho mỗi mức ưu
  tiên, nhu cầu ghế trung bình có trọng số theo thời gian quan sát được trong chu kỳ điều
  chỉnh mượn concurrency gần nhất.

* `apiserver_flowcontrol_demand_seats_stdev` là một gauge vector giữ, cho mỗi mức ưu tiên,
  độ lệch chuẩn tổng thể có trọng số theo thời gian của nhu cầu ghế quan sát được trong
  chu kỳ điều chỉnh mượn concurrency gần nhất.

* `apiserver_flowcontrol_demand_seats_smoothed` là một gauge vector giữ, cho mỗi mức ưu
  tiên, nhu cầu ghế đã được làm trơn và bao hình (smoothed enveloped) xác định tại lần
  điều chỉnh concurrency gần nhất.

* `apiserver_flowcontrol_target_seats` là một gauge vector giữ, cho mỗi mức ưu tiên, mục
  tiêu concurrency đưa vào bài toán phân bổ mượn ghế.

* `apiserver_flowcontrol_seat_fair_frac` là một gauge giữ phân số phân bổ công bằng (fair
  allocation fraction) được xác định trong lần điều chỉnh mượn gần nhất.

* `apiserver_flowcontrol_current_limit_seats` là một gauge vector giữ, cho mỗi mức ưu
  tiên, giới hạn concurrency động được suy ra ở lần điều chỉnh gần nhất.

* `apiserver_flowcontrol_request_execution_seconds` là một histogram vector về thời gian
  các request thực sự thực thi, phân tách theo `flow_schema` và `priority_level`.

* `apiserver_flowcontrol_watch_count_samples` là một histogram vector về số request WATCH
  đang hoạt động có liên quan tới một thao tác ghi cho trước, phân tách theo `flow_schema`
  và `priority_level`.

* `apiserver_flowcontrol_work_estimated_seats` là một histogram vector về số ghế ước lượng
  (giá trị lớn nhất giữa giai đoạn thực thi ban đầu và giai đoạn cuối) gắn với các request,
  phân tách theo `flow_schema` và `priority_level`.

* `apiserver_flowcontrol_request_dispatch_no_accommodation_total` là một counter vector về
  số sự kiện mà về nguyên tắc lẽ ra đã có thể dẫn tới việc một request được điều phối
  nhưng đã không xảy ra, do thiếu concurrency khả dụng, phân tách theo `flow_schema` và
  `priority_level`.

* `apiserver_flowcontrol_epoch_advance_total` là một counter vector về số lần thử đẩy
  ngược đồng hồ tiến độ (progress meter) của một mức ưu tiên về phía sau để tránh tràn số
  (numeric overflow), nhóm theo `priority_level` và `success`.

## Thực hành tốt khi dùng API Priority and Fairness (Good practices for using API Priority and Fairness)

Khi một mức ưu tiên vượt quá concurrency được phép, các request có thể chịu độ trễ tăng
lên hoặc bị loại bỏ kèm lỗi HTTP 429 (Too Many Requests). Để tránh những tác dụng phụ này
của APF, bạn có thể điều chỉnh workload của mình hoặc tinh chỉnh thiết lập APF nhằm đảm
bảo có đủ ghế để phục vụ các request.

Để phát hiện xem request có đang bị từ chối do APF hay không, hãy kiểm tra các metric sau:

- apiserver_flowcontrol_rejected_requests_total: tổng số request bị từ chối theo từng
  FlowSchema và PriorityLevelConfiguration.
- apiserver_flowcontrol_current_inqueue_requests: số request hiện đang xếp hàng theo từng
  FlowSchema và PriorityLevelConfiguration.
- apiserver_flowcontrol_request_wait_duration_seconds: độ trễ tăng thêm cho các request
  đang chờ trong hàng đợi.
- apiserver_flowcontrol_priority_level_seat_utilization: mức sử dụng ghế theo từng
  PriorityLevelConfiguration.

### Điều chỉnh workload (Workload modifications) {#good-practice-workload-modifications}

Để ngăn request bị xếp hàng làm tăng độ trễ hoặc bị loại bỏ do APF, bạn có thể tối ưu các
request của mình bằng cách:

- Giảm tốc độ thực thi request. Số request ít hơn trong một khoảng thời gian cố định sẽ
  dẫn tới việc cần ít ghế hơn tại một thời điểm.
- Tránh phát ra đồng thời một lượng lớn các request tốn kém. Request có thể được tối ưu để
  dùng ít ghế hơn hoặc có độ trễ thấp hơn, nhờ đó chúng giữ ghế trong thời gian ngắn hơn.
  Request list có thể chiếm hơn 1 ghế tùy theo số đối tượng được lấy về trong request đó.
  Hạn chế số đối tượng lấy về trong một request list, ví dụ bằng cách dùng phân trang
  (pagination), sẽ dùng tổng số ghế ít hơn trong khoảng thời gian ngắn hơn. Hơn nữa, thay
  các request list bằng request watch sẽ cần tổng phần chia concurrency thấp hơn, vì
  request watch chỉ chiếm 1 ghế trong đợt bùng phát thông báo ban đầu của nó. Nếu dùng
  streaming list ở phiên bản 1.27 trở lên, request watch sẽ chiếm số ghế bằng với một
  request list cho đợt bùng phát thông báo ban đầu, vì toàn bộ trạng thái của tập hợp phải
  được truyền theo luồng (stream). Lưu ý rằng trong cả hai trường hợp, request watch sẽ
  không giữ ghế nào sau giai đoạn ban đầu này.

Hãy nhớ rằng việc APF xếp hàng hoặc từ chối request có thể do một trong hai nguyên nhân:
số lượng request tăng lên, hoặc độ trễ của các request hiện có tăng lên. Ví dụ, nếu các
request bình thường mất 1s để thực thi bỗng bắt đầu mất 60s, thì có khả năng APF sẽ bắt
đầu từ chối request vì các request đang chiếm ghế lâu hơn bình thường do độ trễ tăng này.
Nếu APF bắt đầu từ chối request trên nhiều mức ưu tiên mà workload không thay đổi đáng kể,
thì có khả năng có một vấn đề tiềm ẩn về hiệu năng của control plane, chứ không phải do
workload hay thiết lập APF.

### Thiết lập priority and fairness (Priority and fairness settings) {#good-practice-apf-settings}

Bạn cũng có thể sửa các đối tượng FlowSchema và PriorityLevelConfiguration mặc định, hoặc
tạo các đối tượng mới thuộc những kiểu này, để phù hợp hơn với workload của bạn.

Các thiết lập APF có thể được sửa để:

- Cấp thêm ghế cho các request ưu tiên cao.
- Cô lập những request không thiết yếu hoặc tốn kém — vốn sẽ làm đói một mức concurrency
  nếu dùng chung với các flow khác.

#### Cấp thêm ghế cho các request ưu tiên cao (Give more seats to high priority requests)

1. Nếu có thể, số ghế khả dụng trên tất cả các mức ưu tiên của một `kube-apiserver` cụ thể
   có thể được tăng lên bằng cách tăng giá trị cho các cờ `max-requests-inflight` và
   `max-mutating-requests-inflight`. Ngoài ra, việc mở rộng theo chiều ngang số instance
   `kube-apiserver` sẽ làm tăng tổng concurrency cho mỗi mức ưu tiên trên toàn cluster,
   với giả định là request được cân bằng tải đầy đủ.
1. Bạn có thể tạo một FlowSchema mới tham chiếu tới một PriorityLevelConfiguration có mức
   concurrency lớn hơn. PriorityLevelConfiguration mới này có thể là một mức đã có sẵn
   hoặc một mức mới với tập phần chia concurrency danh nghĩa riêng. Ví dụ, có thể đưa vào
   một FlowSchema mới để đổi PriorityLevelConfiguration cho các request của bạn từ
   global-default sang workload-low nhằm tăng số ghế khả dụng cho người dùng của bạn. Việc
   tạo một PriorityLevelConfiguration mới sẽ làm giảm số ghế dành cho các mức hiện có. Hãy
   nhớ rằng việc chỉnh sửa một FlowSchema hoặc PriorityLevelConfiguration mặc định sẽ đòi
   hỏi đặt annotation `apf.kubernetes.io/autoupdate-spec` thành false.
1. Bạn cũng có thể tăng NominalConcurrencyShares cho PriorityLevelConfiguration đang phục
   vụ các request ưu tiên cao của bạn. Ngoài ra, với phiên bản 1.26 trở lên, bạn có thể
   tăng LendablePercent cho các mức ưu tiên cạnh tranh, để mức ưu tiên đang xét có một
   nguồn ghế lớn hơn để mượn.

#### Cô lập các request không thiết yếu để tránh làm đói các flow khác (Isolate non-essential requests from starving other flows)

Để cô lập request, bạn có thể tạo một FlowSchema có subject khớp với người dùng đang tạo
ra những request đó, hoặc tạo một FlowSchema khớp với chính nội dung của request (tương
ứng với resourceRules). Tiếp theo, bạn có thể ánh xạ FlowSchema này tới một
PriorityLevelConfiguration có phần chia ghế thấp.

Ví dụ, giả sử các request list event từ những Pod đang chạy trong namespace default dùng
10 ghế mỗi request và thực thi trong 1 phút. Để ngăn những request tốn kém này ảnh hưởng
tới request từ các Pod khác đang dùng FlowSchema service-accounts sẵn có, bạn có thể áp
dụng FlowSchema sau để cô lập các lời gọi list này khỏi các request khác.

Ví dụ đối tượng FlowSchema để cô lập các request list event:

```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: FlowSchema
metadata:
  name: list-events-default-service-account
spec:
  distinguisherMethod:
    type: ByUser
  matchingPrecedence: 8000
  priorityLevelConfiguration:
    name: catch-all
  rules:
    - resourceRules:
      - apiGroups:
          - '*'
        namespaces:
          - default
        resources:
          - events
        verbs:
          - list
      subjects:
        - kind: ServiceAccount
          serviceAccount:
            name: default
            namespace: default
```

- FlowSchema này bắt tất cả các lời gọi list event do service account default trong
  namespace default thực hiện. Độ ưu tiên khớp (matching precedence) 8000 nhỏ hơn giá trị
  9000 mà FlowSchema service-accounts sẵn có đang dùng, nên các lời gọi list event này sẽ
  khớp với list-events-default-service-account thay vì service-accounts.
- PriorityLevelConfiguration catch-all được dùng để cô lập các request này. Mức ưu tiên
  catch-all có phần chia concurrency rất nhỏ và không xếp hàng request.

## Tiếp theo (What's next)

- Bạn có thể xem [tài liệu tham khảo](https://kubernetes.io/docs/reference/debug-cluster/flow-control/)
  về flow control để tìm hiểu thêm về việc khắc phục sự cố.
- Để có thông tin nền tảng về chi tiết thiết kế của API priority and fairness, xem
  [đề xuất cải tiến (enhancement proposal)](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/1040-priority-and-fairness).
- Bạn có thể đưa ra góp ý và yêu cầu tính năng qua
  [SIG API Machinery](https://github.com/kubernetes/community/tree/main/sig-api-machinery)
  hoặc [kênh slack](https://kubernetes.slack.com/messages/api-priority-and-fairness) của
  tính năng này.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 9:

1. FlowSchema và PriorityLevelConfiguration — mỗi tài nguyên chịu trách nhiệm việc gì, và một
   request đến có thể được gán vào bao nhiêu mức ưu tiên?
2. Hai FlowSchema cùng khớp một request, một cái có `matchingPrecedence: 8000`, cái kia
   `9000`. Cái nào thắng? Trả lời kèm quy tắc duyệt mà bài mô tả.
3. Trên cluster lab, một controller lỗi chạy trong Pod bắn dồn dập request vào API server của
   `lab-k8s-master`. Theo cấu hình được đề xuất, request đó rơi vào mức ưu tiên nào, và cơ chế nào
   giữ cho kubelet của hai worker cùng việc bầu chọn leader không bị bỏ đói?
4. Khi lượng request vào một mức ưu tiên vượt quá concurrency cho phép, hai kiểu hành vi là gì,
   và client nhìn thấy lỗi nào?
5. Bạn sửa spec một PriorityLevelConfiguration mặc định, ít phút sau thấy nó trở lại như cũ.
   Vì sao, và phải làm gì để giữ được thay đổi?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **FlowSchema phân loại**: nó khớp với các request đến và gán chúng vào một mức ưu tiên.
   **PriorityLevelConfiguration định nghĩa mức ưu tiên**: giới hạn concurrency riêng của mức đó,
   phần ngân sách concurrency nó được hưởng, và hành vi xếp hàng. Mỗi request được gán vào
   **đúng một** mức ưu tiên — bài nói mỗi FlowSchema luôn gán request vào đúng một
   PriorityLevelConfiguration.
2. **Cái có `matchingPrecedence: 8000` thắng.** Quy tắc: mọi request được đối chiếu với các
   FlowSchema **bắt đầu từ những FlowSchema có `matchingPrecedence` nhỏ nhất về mặt số học rồi
   tăng dần**, và **FlowSchema khớp đầu tiên sẽ thắng** — tức là **số nhỏ hơn được xét trước**.
   Chính ví dụ ở cuối bài dùng đúng cơ chế này: FlowSchema
   `list-events-default-service-account` đặt `matchingPrecedence: 8000` để thắng FlowSchema
   `service-accounts` sẵn có vốn dùng 9000. Bài cũng khuyên **đừng để hai FlowSchema có cùng
   `matchingPrecedence`**.
3. Rơi vào **`workload-low`** — mức ưu tiên dành cho request từ bất kỳ service account nào khác,
   **thường bao gồm tất cả request từ các controller đang chạy trong Pod**. Cơ chế bảo vệ là
   **mỗi mức ưu tiên có giới hạn concurrency riêng biệt**, nên các mức khác không bị làm đói:
   request của kubelet thuộc mức **`system`** (request không liên quan tới sức khỏe từ nhóm
   `system:nodes`), cập nhật sức khỏe từ node thuộc **`node-high`**, và bầu chọn leader thuộc
   **`leader-election`** — mức mà bài nhấn mạnh phải cô lập, vì lỗi bầu chọn leader làm các
   controller lỗi và khởi động lại, kéo theo lưu lượng còn tốn kém hơn khi chúng đồng bộ lại
   informer.
4. Trường `type` trong spec của PriorityLevelConfiguration quyết định: **`Reject`** nghĩa là lưu
   lượng vượt mức **bị từ chối ngay lập tức với lỗi HTTP 429 (Too Many Requests)**; **`Queue`**
   nghĩa là request vượt ngưỡng **được xếp hàng**, dùng shuffle sharding và fair queuing để cân
   bằng tiến độ giữa các flow. Request bị xếp hàng quá lâu rồi hết thời gian chờ cũng bị từ chối.
5. Vì đó là một **đối tượng cấu hình được đề xuất**, và mỗi `kube-apiserver` **bảo trì định kỳ
   mỗi phút một lần**: nếu các kube-apiserver đang kiểm soát đối tượng, chúng **đảm bảo spec
   đúng như server đề xuất**. Để giữ thay đổi, phải chuyển quyền kiểm soát sang người dùng bằng
   cách đặt annotation **`apf.kubernetes.io/autoupdate-spec` thành `false`**. Lưu ý thêm: **xóa
   thì không được tôn trọng** — quá trình bảo trì sẽ khôi phục lại đối tượng, nên muốn vô hiệu
   một mức thì phải **giữ nó lại nhưng đặt spec sao cho ít gây hệ quả nhất**.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

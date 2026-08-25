# Eviction do áp lực node (Node-pressure Eviction)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 7 → nhóm [7a](00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction), bài 7/13 ·
Kiểm chứng ở Lab 7a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Điểm khác biệt lớn nhất so với mọi bài trước trong nhóm: **chủ thể ở đây là kubelet, không
phải kube-scheduler**. Không có API nào được gọi, không có PodDisruptionBudget nào được hỏi —
node tự cứu mình. Đây cũng là bài mà `requests` của bài
[110](110-manage-resources-containers-vi.md) và QoS class của bài [54](54-pod-qos-vi.md) trả
bài: cả thứ tự trục xuất lẫn ngưỡng đều tính từ hai thứ đó.

Bài dài vì phải mô tả ba bố cục filesystem khác nhau. Cluster lab dùng bố cục đơn giản nhất —
mọi thứ nằm trên một `nodefs` duy nhất — nên bạn có thể lướt nhanh mọi đoạn nói về `imagefs`
tách riêng và `containerfs`.

**Phải hiểu ở lần đọc này:**

- kubelet **tự** chấm dứt Pod, đặt phase của Pod thành `Failed`. Nó **không tôn trọng**
  PodDisruptionBudget hay `terminationGracePeriodSeconds` của Pod — khác hẳn eviction khởi
  phát qua API ở bài [143](143-api-eviction-vi.md).
- Cơ chế đo: **tín hiệu eviction** (`memory.available`, `nodefs.available`,
  `nodefs.inodesFree`, `imagefs.available`, `pid.available`…) được so với **ngưỡng eviction**.
  Ngưỡng **mềm** đi kèm grace period bắt buộc; ngưỡng **cứng** không có grace period, kubelet
  kill Pod ngay với grace period `0s`. Nhớ các ngưỡng cứng mặc định: `memory.available<100Mi`,
  `nodefs.available<10%`, `imagefs.available<15%`, `nodefs.inodesFree<5%`,
  `imagefs.inodesFree<5%` trên node Linux.
- **Bẫy cấu hình quan trọng nhất của bài:** nếu bạn đổi giá trị của **một** tham số ngưỡng
  cứng, các tham số còn lại **không kế thừa** mặc định mà bị đặt về **không**. Muốn giữ mặc
  định thì phải khai đủ, hoặc bật `MergeDefaultEvictionSettings`.
- kubelet **thu hồi tài nguyên cấp node trước** khi động tới Pod của người dùng: garbage
  collect pod/container đã chết, rồi xóa các image không dùng đến.
- Thứ tự trục xuất dựa trên ba tham số theo đúng thứ tự: (1) mức sử dụng có **vượt requests**
  hay không, (2) **Priority** của Pod, (3) mức vượt so với requests là bao nhiêu. Kết quả:
  nhóm `BestEffort`/`Burstable` **vượt requests** bị evict trước, nhóm `Guaranteed` và
  `Burstable` **dưới requests** bị evict sau cùng. Ghi chú của bài nhấn mạnh kubelet **không**
  dùng QoS class để quyết định thứ tự — QoS chỉ dùng để **ước lượng**.
- Ba điều kiện node `MemoryPressure`, `DiskPressure`, `PIDPressure` được kubelet báo cáo khi
  chạm ngưỡng, và control plane **ánh xạ chúng thành taint** — nối thẳng sang bài
  [139](139-taint-and-toleration-vi.md), đó là lý do node đang chịu áp lực cũng ngừng nhận Pod
  mới.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ba bố cục filesystem, toàn bộ `containerfs` và *split image filesystem* | cluster lab chỉ có một `nodefs` | không cần |
| Cách lấy `memory.available` trên node Windows, `GetPerformanceInfo()` | không có node Windows | giai đoạn 15 |
| *Các tính năng garbage collection của kubelet đã lỗi thời* | các flag đã bị thay bằng eviction | không cần |
| *Lượng thu hồi eviction tối thiểu* (`--eviction-minimum-reclaim`) | tinh chỉnh khi eviction lặp lại nhiều lần | không cần |
| *Hành vi khi node hết bộ nhớ* — bảng `oom_score_adj`, `oom_killer` | là cơ chế của kernel Linux, không phải eviction của kubelet | giai đoạn 2, bài [33](33-cgroups-vi.md) |
| *Tài nguyên có thể lập lịch và chính sách eviction*, `--system-reserved`/`--kube-reserved` | thuộc phần dự trữ tài nguyên cho daemon hệ thống | [checkpoint tasks](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster) |
| *Các vấn đề đã biết* — `active_file`, kubelet không thấy áp lực ngay | là trường hợp biên khi tinh chỉnh | không cần |

---

Eviction do áp lực node (node-pressure eviction) là quá trình trong đó kubelet chủ động
chấm dứt các pod để thu hồi tài nguyên trên các node.

kubelet giám sát các tài nguyên như bộ nhớ, dung lượng đĩa và inode của filesystem
trên các node trong cluster của bạn.
Khi một hoặc nhiều tài nguyên này đạt tới mức tiêu thụ nhất định, kubelet
có thể chủ động làm thất bại (fail) một hoặc nhiều pod trên node để thu hồi tài nguyên
và ngăn chặn tình trạng cạn kiệt tài nguyên (starvation).

Trong quá trình eviction do áp lực node, kubelet đặt [phase](./47-pod-lifecycle-vi.md#pod-phase) của
các pod được chọn thành `Failed`, và chấm dứt (terminate) Pod đó.

Eviction do áp lực node không giống với
[eviction khởi phát qua API (API-initiated eviction)](143-api-eviction-vi.md).

kubelet không tôn trọng PodDisruptionBudget mà bạn đã cấu hình
hay `terminationGracePeriodSeconds` của pod. Nếu bạn dùng [ngưỡng eviction mềm](#soft-eviction-thresholds),
kubelet tôn trọng giá trị `eviction-max-pod-grace-period` mà bạn đã cấu hình. Nếu bạn dùng
[ngưỡng eviction cứng](#hard-eviction-thresholds), kubelet dùng grace period `0s` (tắt ngay lập tức) khi chấm dứt Pod.

## Hành vi tự phục hồi (Self healing behavior)

kubelet cố gắng [thu hồi tài nguyên cấp node](#reclaim-node-resources)
trước khi chấm dứt các pod của người dùng cuối. Ví dụ, nó xóa các container image
không dùng đến khi tài nguyên đĩa bị cạn kiệt.

Nếu các pod được quản lý bởi một đối tượng quản lý workload
(chẳng hạn StatefulSet
hoặc Deployment) có khả năng
thay thế các pod bị lỗi, control plane (`kube-controller-manager`) sẽ tạo các
pod mới thay cho các pod đã bị evict.

### Tự phục hồi cho static pod (Self healing for static pods)

Nếu bạn đang chạy một [static pod](46-pods-vi.md#static-pods)
trên một node đang chịu áp lực tài nguyên, kubelet có thể evict static
Pod đó. Sau đó kubelet cố gắng tạo một bản thay thế, bởi vì static Pod luôn
thể hiện ý định chạy một Pod trên node đó.

kubelet tính đến _độ ưu tiên (priority)_ của static pod khi tạo
bản thay thế. Nếu manifest của static pod chỉ định độ ưu tiên thấp, và trong
control plane của cluster có các Pod với độ ưu tiên cao hơn được định nghĩa, và
node đang chịu áp lực tài nguyên, thì kubelet có thể không tạo được chỗ trống cho
static pod đó. kubelet vẫn tiếp tục cố gắng chạy tất cả các static pod ngay cả
khi node đang có áp lực tài nguyên.

## Tín hiệu và ngưỡng eviction (Eviction signals and thresholds)

kubelet dùng nhiều tham số khác nhau để ra quyết định eviction, như những tham số sau:

- Tín hiệu eviction (eviction signal)
- Ngưỡng eviction (eviction threshold)
- Khoảng thời gian giám sát (monitoring interval)

### Tín hiệu eviction (Eviction signals) {#eviction-signals}

Tín hiệu eviction là trạng thái hiện tại của một tài nguyên cụ thể tại một
thời điểm nhất định. kubelet dùng tín hiệu eviction để ra quyết định eviction bằng cách
so sánh tín hiệu với các ngưỡng eviction — là lượng tài nguyên tối thiểu
nên có sẵn trên node.

kubelet dùng các tín hiệu eviction sau:

| Tín hiệu eviction        | Mô tả                                                                                 | Chỉ Linux  |
|--------------------------|---------------------------------------------------------------------------------------|------------|
| `memory.available`       | `memory.available` := `node.status.capacity[memory]` - `node.stats.memory.workingSet` |            |
| `nodefs.available`       | `nodefs.available` := `node.stats.fs.available`                                       |            |
| `nodefs.inodesFree`      | `nodefs.inodesFree` := `node.stats.fs.inodesFree`                                     |      •     |
| `imagefs.available`      | `imagefs.available` := `node.stats.runtime.imagefs.available`                         |            |
| `imagefs.inodesFree`     | `imagefs.inodesFree` := `node.stats.runtime.imagefs.inodesFree`                       |      •     |
| `containerfs.available`  | `containerfs.available` := `node.stats.runtime.containerfs.available`                 |            |
| `containerfs.inodesFree` | `containerfs.inodesFree` := `node.stats.runtime.containerfs.inodesFree`               |      •     |
| `pid.available`          | `pid.available` := `node.stats.rlimit.maxpid` - `node.stats.rlimit.curproc`           |      •     |

Trong bảng này, cột **Mô tả** cho biết cách kubelet lấy giá trị của
tín hiệu. Mỗi tín hiệu hỗ trợ giá trị theo phần trăm hoặc giá trị tuyệt đối. kubelet
tính giá trị phần trăm tương đối so với tổng dung lượng gắn với
tín hiệu đó.

#### Tín hiệu bộ nhớ (Memory signals)

Trên các node Linux, giá trị của `memory.available` được lấy từ cgroupfs thay vì từ các công cụ
như `free -m`. Điều này quan trọng vì `free -m` không hoạt động bên trong
container, và nếu người dùng dùng tính năng [node allocatable](253-reserve-compute-resources-vi.md#node-allocatable),
thì các quyết định về cạn kiệt tài nguyên
được đưa ra cục bộ cho Pod của người dùng cuối trong hệ thống phân cấp cgroup, cũng như tại
node gốc. [Script này](https://kubernetes.io/examples/admin/resource/memory-available.sh) hoặc
[script cho cgroupv2](https://kubernetes.io/examples/admin/resource/memory-available-cgroupv2.sh)
tái hiện đúng chuỗi các bước mà kubelet thực hiện để tính
`memory.available`. kubelet loại trừ inactive_file (số byte của
bộ nhớ được ánh xạ từ file (file-backed memory) trong danh sách LRU không hoạt động) khỏi phép tính, vì nó giả định rằng
lượng bộ nhớ đó có thể được thu hồi khi có áp lực.

Trên các node Windows, giá trị của `memory.available` được lấy từ mức commit bộ nhớ
toàn cục của node (truy vấn qua system call [`GetPerformanceInfo()`](https://learn.microsoft.com/windows/win32/api/psapi/nf-psapi-getperformanceinfo))
bằng cách lấy [`CommitLimit`](https://learn.microsoft.com/windows/win32/api/psapi/ns-psapi-performance_information) toàn cục của node trừ đi [`CommitTotal`](https://learn.microsoft.com/windows/win32/api/psapi/ns-psapi-performance_information) toàn cục của node. Lưu ý rằng `CommitLimit` có thể thay đổi nếu kích thước page-file của node thay đổi!

#### Tín hiệu filesystem (Filesystem signals) {#filesystem-signals}

kubelet nhận biết ba định danh (identifier) filesystem cụ thể có thể dùng với
các tín hiệu eviction (`<identifier>.inodesFree` hoặc `<identifier>.available`):

1. `nodefs`: Filesystem chính của node, được dùng cho các volume đĩa cục bộ,
    các emptyDir volume không dựa trên bộ nhớ, nơi lưu log, lưu trữ tạm thời (ephemeral storage),
    và nhiều thứ khác. Ví dụ, `nodefs` chứa `/var/lib/kubelet`.

1. `imagefs`: Một filesystem tùy chọn mà các container runtime có thể dùng để lưu
   các container image (là các lớp chỉ đọc — read-only layer). Nếu không có
   `containerfs` riêng, filesystem image này cũng lưu các lớp ghi được (writable layer) của container.

1. `containerfs`: Một filesystem tùy chọn mà các container runtime có thể dùng để
   lưu các lớp ghi được của container. Khi dùng `containerfs`, filesystem `imagefs`
   có thể được tách ra để chỉ lưu image (các lớp chỉ đọc) và không lưu
   gì khác.

Các định danh này mô tả các filesystem theo cách kubelet quan sát chúng. Chúng
không phải lúc nào cũng tương ứng với ba mount point khác nhau: trong các bố cục phổ biến, hai hoặc cả
ba định danh có thể trỏ đến cùng một filesystem bên dưới.

> **Ghi chú:**
>
> **TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.31 [beta]`
>
> Tính năng _split image filesystem_ (tách filesystem image) bổ sung các tín hiệu eviction, ngưỡng và
> metric mới cho `containerfs`. Để dùng `containerfs`, bản phát hành Kubernetes
> v1.36 yêu cầu bật
> [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) `KubeletSeparateDiskGC`.
> Với Kubernetes v1.36, chỉ CRI-O (v1.29 trở
> lên) hỗ trợ filesystem `containerfs`.

kubelet hỗ trợ ba bố cục (layout) phổ biến cho các filesystem của container:

- Mọi thứ nằm trên một `nodefs` duy nhất, còn được gọi là "rootfs" hay
  đơn giản là "root". Trong bố cục này, `nodefs`, `imagefs`, và `containerfs`
  đều trỏ đến cùng một filesystem bên dưới.

- Kho lưu trữ của container runtime nằm trên một đĩa riêng, tách khỏi filesystem
  gốc. Trong bố cục này, `imagefs` và `containerfs` trỏ đến cùng một
  filesystem bên dưới, nơi lưu cả các lớp image lẫn các lớp ghi được
  của container. Bố cục này thường được gọi là filesystem "split disk"
  (hay "separate disk" — đĩa tách riêng).

- Các lớp ghi được của container nằm trên `nodefs`, còn các container image
  (các lớp chỉ đọc) được lưu trên một `imagefs` riêng. Trong bố cục này,
  `containerfs` và `nodefs` trỏ đến cùng một filesystem bên dưới. Bố cục này
  thường được gọi là filesystem "split image" (tách image).

kubelet sẽ cố gắng tự động phát hiện các filesystem này cùng cấu hình hiện tại
của chúng trực tiếp từ container runtime bên dưới và sẽ bỏ qua
các filesystem cục bộ khác của node.

kubelet không hỗ trợ các filesystem container hay cấu hình lưu trữ khác,
và hiện tại không hỗ trợ nhiều filesystem cho image và container.

### Các tính năng garbage collection của kubelet đã lỗi thời (Deprecated kubelet garbage collection features)

Một số tính năng garbage collection của kubelet đã bị đánh dấu lỗi thời (deprecated) để thay bằng eviction:

| Flag hiện có | Lý do |
| ------------- | --------- |
| `--maximum-dead-containers` | lỗi thời khi log cũ được lưu bên ngoài ngữ cảnh của container |
| `--maximum-dead-containers-per-container` | lỗi thời khi log cũ được lưu bên ngoài ngữ cảnh của container |
| `--minimum-container-ttl-duration` | lỗi thời khi log cũ được lưu bên ngoài ngữ cảnh của container |

### Ngưỡng eviction (Eviction thresholds)

Bạn có thể chỉ định các ngưỡng eviction tùy chỉnh để kubelet dùng khi
ra quyết định eviction. Bạn có thể cấu hình ngưỡng eviction [mềm](#soft-eviction-thresholds) và
[cứng](#hard-eviction-thresholds).

Ngưỡng eviction có dạng `[eviction-signal][operator][quantity]`, trong đó:

- `eviction-signal` là [tín hiệu eviction](#eviction-signals) được dùng.
- `operator` là [toán tử quan hệ](https://en.wikipedia.org/wiki/Relational_operator#Standard_relational_operators)
  bạn muốn, chẳng hạn `<` (nhỏ hơn).
- `quantity` là lượng ngưỡng eviction, chẳng hạn `1Gi`. Giá trị của `quantity`
  phải khớp với cách biểu diễn số lượng (quantity) mà Kubernetes sử dụng. Bạn có thể dùng
  giá trị tuyệt đối hoặc phần trăm (`%`).

Ví dụ, nếu một node có tổng cộng 10GiB bộ nhớ và bạn muốn kích hoạt eviction khi
bộ nhớ khả dụng giảm xuống dưới 1GiB, bạn có thể định nghĩa ngưỡng eviction là
`memory.available<10%` hoặc `memory.available<1Gi` (không thể dùng cả hai cùng lúc).

#### Ngưỡng eviction mềm (Soft eviction thresholds) {#soft-eviction-thresholds}

Ngưỡng eviction mềm là sự kết hợp giữa một ngưỡng eviction với một grace period bắt buộc
do quản trị viên chỉ định. kubelet không evict pod cho đến khi
grace period này bị vượt qua. kubelet trả về lỗi khi khởi động nếu bạn
không chỉ định grace period.

Bạn có thể chỉ định cả grace period cho ngưỡng eviction mềm lẫn grace period
tối đa cho phép khi chấm dứt pod, để kubelet dùng trong quá trình eviction. Nếu bạn
chỉ định grace period tối đa cho phép và ngưỡng eviction mềm bị chạm tới,
kubelet dùng giá trị nhỏ hơn trong hai grace period đó. Nếu bạn không chỉ định
grace period tối đa cho phép, kubelet kill các pod bị evict ngay lập tức mà không
chấm dứt một cách êm thấm (graceful termination).

Bạn có thể dùng các flag sau để cấu hình ngưỡng eviction mềm:

- `eviction-soft`: Một tập các ngưỡng eviction như `memory.available<1.5Gi`
  có thể kích hoạt eviction pod nếu bị duy trì quá grace period được chỉ định.
- `eviction-soft-grace-period`: Một tập các grace period cho eviction như `memory.available=1m30s`
  định nghĩa thời gian một ngưỡng eviction mềm phải được duy trì trước khi kích hoạt eviction một Pod.
- `eviction-max-pod-grace-period`: Grace period tối đa cho phép (tính bằng giây)
  được dùng khi chấm dứt pod để phản ứng với việc chạm ngưỡng eviction mềm.

#### Ngưỡng eviction cứng (Hard eviction thresholds) {#hard-eviction-thresholds}

Ngưỡng eviction cứng không có grace period. Khi chạm tới ngưỡng eviction cứng,
kubelet kill các pod ngay lập tức mà không chấm dứt êm thấm, để thu hồi
tài nguyên đang cạn kiệt.

Bạn có thể dùng flag `eviction-hard` để cấu hình một tập các ngưỡng eviction
cứng như `memory.available<1Gi`.

kubelet có các ngưỡng eviction cứng mặc định sau:

- `memory.available<100Mi` (node Linux)
- `memory.available<500Mi` (node Windows)
- `nodefs.available<10%`
- `imagefs.available<15%`
- `nodefs.inodesFree<5%` (node Linux)
- `imagefs.inodesFree<5%` (node Linux)

Các giá trị mặc định này của ngưỡng eviction cứng chỉ được thiết lập nếu không có
tham số nào bị thay đổi. Nếu bạn thay đổi giá trị của bất kỳ tham số nào,
thì giá trị của các tham số còn lại sẽ không được kế thừa như giá trị
mặc định mà sẽ bị đặt về không (zero). Để cung cấp các giá trị tùy chỉnh, bạn
nên cung cấp tất cả các ngưỡng tương ứng. Bạn cũng có thể đặt tùy chọn cấu hình
MergeDefaultEvictionSettings của kubelet thành true trong file cấu hình kubelet.
Nếu đặt thành true và có tham số nào đó bị thay đổi, thì các tham số còn lại sẽ
kế thừa giá trị mặc định của chúng thay vì 0.

Các ngưỡng eviction mặc định của `containerfs.available` và `containerfs.inodesFree` (node Linux)
sẽ được thiết lập như sau:

- Nếu `containerfs` và `nodefs` trỏ đến cùng một filesystem bên dưới, thì
  các ngưỡng của `containerfs` được đặt giống như của `nodefs`.

- Nếu `containerfs` và `imagefs` trỏ đến cùng một filesystem bên dưới, thì
  các ngưỡng của `containerfs` được đặt giống như của `imagefs`.

Việc ghi đè tùy chỉnh các ngưỡng liên quan đến `containerfs` không được
hỗ trợ, và một cảnh báo sẽ được đưa ra nếu bạn cố làm vậy; mọi
giá trị tùy chỉnh được cung cấp sẽ bị bỏ qua.

## Khoảng thời gian giám sát eviction (Eviction monitoring interval)

kubelet đánh giá các ngưỡng eviction dựa trên `housekeeping-interval` được cấu hình,
mặc định là `10s`.

## Điều kiện node (Node conditions) {#node-conditions}

kubelet báo cáo các [điều kiện node (node condition)](https://kubernetes.io/docs/concepts/architecture/nodes#condition)
để phản ánh rằng node đang chịu áp lực vì ngưỡng eviction cứng hoặc mềm
đã bị chạm tới, bất kể các grace period đã cấu hình.

kubelet ánh xạ tín hiệu eviction sang điều kiện node như sau:

| Điều kiện node    | Tín hiệu eviction                                                                       | Mô tả                                                                                |
|-------------------|---------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------|
| `MemoryPressure`  | `memory.available`                                                                    | Bộ nhớ khả dụng trên node đã chạm một ngưỡng eviction                           |
| `DiskPressure`    | `nodefs.available`, `nodefs.inodesFree`, `imagefs.available`, `imagefs.inodesFree`, `containerfs.available`, hoặc `containerfs.inodesFree` | Dung lượng đĩa và inode khả dụng trên filesystem gốc, filesystem image, hoặc filesystem container của node đã chạm một ngưỡng eviction              |
| `PIDPressure`     | `pid.available`                                                                       | Số định danh tiến trình (process identifier) khả dụng trên node (Linux) đã giảm xuống dưới một ngưỡng eviction |

Control plane cũng [ánh xạ](139-taint-and-toleration-vi.md#taint-nodes-by-condition)
các điều kiện node này thành các taint.

kubelet cập nhật các điều kiện node dựa trên
`--node-status-update-frequency` được cấu hình, mặc định là `10s`.

### Dao động điều kiện node (Node condition oscillation)

Trong một số trường hợp, node dao động lên xuống quanh các ngưỡng eviction mềm mà không
duy trì đủ grace period đã định. Điều này khiến điều kiện node được báo cáo
liên tục chuyển giữa `true` và `false`, dẫn đến các quyết định eviction sai lầm.

Để chống lại hiện tượng dao động, bạn có thể dùng flag `eviction-pressure-transition-period`,
flag này kiểm soát thời gian kubelet phải chờ trước khi chuyển một điều kiện
node sang trạng thái khác. Chu kỳ chuyển tiếp này có giá trị mặc định là `5m`.

### Thu hồi tài nguyên cấp node (Reclaiming node level resources) {#reclaim-node-resources}

kubelet cố gắng thu hồi tài nguyên cấp node trước khi evict các pod của người dùng cuối.

Khi điều kiện node `DiskPressure` được báo cáo, kubelet thu hồi tài nguyên
cấp node dựa trên các filesystem trên node.

#### Khi không có `imagefs` hoặc `containerfs` (Without `imagefs` or `containerfs`)

Nếu node chỉ có một filesystem `nodefs` chạm các ngưỡng eviction,
kubelet giải phóng dung lượng đĩa theo thứ tự sau:

1. Garbage collect các pod và container đã chết.
1. Xóa các image không dùng đến.

#### Khi có `imagefs` (With `imagefs`)

Nếu node có một filesystem `imagefs` chuyên dụng cho các container runtime sử dụng,
kubelet thực hiện như sau:

- Nếu filesystem `nodefs` chạm các ngưỡng eviction, kubelet garbage
  collect các pod và container đã chết.

- Nếu filesystem `imagefs` chạm các ngưỡng eviction, kubelet
  xóa tất cả các image không dùng đến.

#### Khi có `imagefs` và `containerfs` (With `imagefs` and `containerfs`)

Nếu node có một `containerfs` chuyên dụng bên cạnh filesystem `imagefs`
được cấu hình cho các container runtime sử dụng, thì kubelet sẽ cố gắng
thu hồi tài nguyên như sau:

- Nếu filesystem `containerfs` chạm các ngưỡng eviction, kubelet
  garbage collect các pod và container đã chết.

- Nếu filesystem `imagefs` chạm các ngưỡng eviction, kubelet
  xóa tất cả các image không dùng đến.

### Lựa chọn pod cho eviction của kubelet (Pod selection for kubelet eviction) {#pod-selection-for-kubelet-eviction}

Nếu các nỗ lực thu hồi tài nguyên cấp node của kubelet không đưa được tín hiệu
eviction xuống dưới ngưỡng, kubelet bắt đầu evict các pod của người dùng cuối.

kubelet dùng các tham số sau để xác định thứ tự evict pod:

1. Mức sử dụng tài nguyên của pod có vượt quá requests hay không
1. [Độ ưu tiên của Pod (Pod Priority)](141-pod-priority-preemption-vi.md)
1. Mức sử dụng tài nguyên của pod so với requests

Kết quả là kubelet xếp hạng và evict các pod theo thứ tự sau:

1. Các pod `BestEffort` hoặc `Burstable` có mức sử dụng vượt quá requests. Các pod này
   bị evict dựa trên Priority của chúng, rồi đến mức độ mà mức sử dụng
   vượt quá request.

1. Các pod `Guaranteed` và các pod `Burstable` có mức sử dụng thấp hơn requests
   bị evict sau cùng, dựa trên Priority của chúng.

> **Ghi chú:**
>
> kubelet không dùng [lớp QoS](./54-pod-qos-vi.md) của pod để xác định thứ tự eviction.
> Bạn có thể dùng lớp QoS để ước lượng thứ tự evict pod có khả năng xảy ra nhất khi
> thu hồi các tài nguyên như bộ nhớ. Việc phân loại QoS không áp dụng cho các request EphemeralStorage,
> vì vậy kịch bản trên sẽ không áp dụng nếu node, chẳng hạn, đang ở trạng thái `DiskPressure`.

Các pod `Guaranteed` chỉ được đảm bảo khi requests và limits được chỉ định cho
tất cả các container và chúng bằng nhau. Các pod này sẽ không bao giờ bị evict vì
mức tiêu thụ tài nguyên của một pod khác. Nếu một daemon hệ thống (như `kubelet`
và `journald`) đang tiêu thụ nhiều tài nguyên hơn lượng đã được dành riêng qua
các cấp phát `system-reserved` hoặc `kube-reserved`, và node chỉ còn lại các pod
`Guaranteed` hoặc `Burstable` đang dùng ít tài nguyên hơn requests,
thì kubelet buộc phải chọn evict một trong các pod này để bảo toàn sự ổn định của node
và hạn chế tác động của việc cạn kiệt tài nguyên lên các pod khác. Trong trường hợp này, nó
sẽ chọn evict các pod có Priority thấp nhất trước.

Nếu bạn đang chạy một [static pod](46-pods-vi.md#static-pods)
và muốn tránh việc nó bị evict khi có áp lực tài nguyên, hãy đặt trực tiếp trường
`priority` cho Pod đó. Static pod không hỗ trợ trường
`priorityClassName`.

Khi kubelet evict pod để phản ứng với tình trạng cạn kiệt inode hoặc process ID, nó dùng
độ ưu tiên tương đối của các Pod để xác định thứ tự eviction, bởi vì inode và PID không có
requests.

kubelet sắp xếp các pod theo cách khác nhau tùy vào việc node có filesystem
`imagefs` hoặc `containerfs` chuyên dụng hay không:

#### Khi không có `imagefs` hoặc `containerfs` (`nodefs` và `imagefs` dùng chung một filesystem) {#without-imagefs}

- Nếu `nodefs` kích hoạt eviction, kubelet sắp xếp các pod dựa trên
  tổng mức sử dụng đĩa của chúng (`local volumes + logs và writable layer của tất cả các container`).

#### Khi có `imagefs` (filesystem `nodefs` và `imagefs` tách riêng) {#with-imagefs}

- Nếu `nodefs` kích hoạt eviction, kubelet sắp xếp các pod dựa trên mức sử dụng
  `nodefs` (`local volumes + logs của tất cả các container`).

- Nếu `imagefs` kích hoạt eviction, kubelet sắp xếp các pod dựa trên
  mức sử dụng writable layer của tất cả các container.

#### Khi có `imagefs` và `containerfs` (`imagefs` và `containerfs` đã được tách riêng) {#with-containersfs}

- Nếu `containerfs` kích hoạt eviction, kubelet sắp xếp các pod dựa trên
  mức sử dụng `containerfs` (`local volumes + logs và writable layer của tất cả các container`).

- Nếu `imagefs` kích hoạt eviction, kubelet sắp xếp các pod dựa trên
  hạng `storage of images` (kho lưu trữ image), đại diện cho mức sử dụng đĩa của một image nhất định.

### Lượng thu hồi eviction tối thiểu (Minimum eviction reclaim)

> **Ghi chú:**
>
> Kể từ Kubernetes v1.36, bạn không thể đặt giá trị tùy chỉnh
> cho metric `containerfs.available`. Cấu hình cho metric cụ thể
> này sẽ được thiết lập tự động để phản ánh các giá trị đã đặt cho `nodefs`
> hoặc `imagefs`, tùy theo cấu hình.

Trong một số trường hợp, việc evict pod chỉ thu hồi được một lượng nhỏ tài nguyên đang cạn kiệt.
Điều này có thể khiến kubelet liên tục chạm các ngưỡng eviction đã cấu hình
và kích hoạt nhiều lần eviction.

Bạn có thể dùng flag `--eviction-minimum-reclaim` hoặc [file cấu hình kubelet](224-kubelet-config-file-vi.md)
để cấu hình lượng thu hồi tối thiểu cho mỗi tài nguyên. Khi kubelet nhận thấy
một tài nguyên đang cạn kiệt, nó tiếp tục thu hồi tài nguyên đó cho đến khi
thu hồi được lượng mà bạn chỉ định.

Ví dụ, cấu hình sau thiết lập các lượng thu hồi tối thiểu:

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
evictionHard:
  memory.available: "500Mi"
  nodefs.available: "1Gi"
  imagefs.available: "100Gi"
evictionMinimumReclaim:
  memory.available: "0Mi"
  nodefs.available: "500Mi"
  imagefs.available: "2Gi"
```

Trong ví dụ này, nếu tín hiệu `nodefs.available` chạm ngưỡng eviction,
kubelet thu hồi tài nguyên cho đến khi tín hiệu đạt ngưỡng 1GiB,
rồi tiếp tục thu hồi thêm lượng tối thiểu 500MiB, cho đến khi giá trị
dung lượng nodefs khả dụng đạt 1.5GiB.

Tương tự, kubelet cố gắng thu hồi tài nguyên `imagefs` cho đến khi giá trị `imagefs.available`
đạt `102Gi`, tương ứng 102 GiB dung lượng lưu trữ container image khả dụng. Nếu lượng
dung lượng mà kubelet có thể thu hồi nhỏ hơn 2GiB, kubelet không thu hồi gì cả.

Giá trị mặc định của `eviction-minimum-reclaim` là `0` cho tất cả các tài nguyên.

## Hành vi khi node hết bộ nhớ (Node out of memory behavior)

Nếu node gặp sự kiện _hết bộ nhớ_ (out of memory — OOM) trước khi kubelet
kịp thu hồi bộ nhớ, node phụ thuộc vào [oom_killer](https://lwn.net/Articles/391222/)
để phản ứng.

kubelet đặt một giá trị `oom_score_adj` cho mỗi container dựa trên QoS của pod.

| Quality of Service | `oom_score_adj`                                                                   |
|--------------------|-----------------------------------------------------------------------------------|
| `Guaranteed`       | -997                                                                              |
| `BestEffort`       | 1000                                                                              |
| `Burstable`        | _min(max(2, 1000 - (1000 × memoryRequestBytes) / machineMemoryCapacityBytes), 999)_ |

> **Ghi chú:**
>
> kubelet cũng đặt giá trị `oom_score_adj` là `-997` cho mọi container trong các Pod có
> Priority `system-node-critical`.

Nếu kubelet không thể thu hồi bộ nhớ trước khi node gặp OOM,
`oom_killer` tính một `oom_score` dựa trên tỷ lệ phần trăm bộ nhớ mà container
đang dùng trên node, rồi cộng thêm `oom_score_adj` để có `oom_score` hiệu dụng
cho mỗi container. Sau đó nó kill container có điểm cao nhất.

Điều này có nghĩa là các container trong những pod có QoS thấp và tiêu thụ lượng bộ nhớ lớn
so với requests khi lập lịch sẽ bị kill trước tiên.

Khác với eviction pod, nếu một container bị OOM kill, kubelet có thể khởi động lại nó
dựa trên `restartPolicy` của container.

## Các thực hành tốt (Good practices) {#node-pressure-eviction-good-practices}

Các phần sau mô tả thực hành tốt cho việc cấu hình eviction.

### Tài nguyên có thể lập lịch và chính sách eviction (Schedulable resources and eviction policies)

Khi bạn cấu hình kubelet với một chính sách eviction, bạn nên đảm bảo rằng
scheduler sẽ không lập lịch các pod nếu chúng sẽ kích hoạt eviction vì chúng
ngay lập tức gây ra áp lực bộ nhớ.

Hãy xem xét kịch bản sau:

- Dung lượng bộ nhớ của node: 10GiB
- Người vận hành muốn dành 10% dung lượng bộ nhớ cho các daemon hệ thống (kernel, `kubelet`, v.v.)
- Người vận hành muốn evict Pod ở mức sử dụng 95% bộ nhớ để giảm tần suất OOM hệ thống.

Để điều này hoạt động, kubelet được khởi chạy như sau:

```none
--eviction-hard=memory.available<500Mi
--system-reserved=memory=1.5Gi
```

Trong cấu hình này, flag `--system-reserved` dành 1.5GiB bộ nhớ
cho hệ thống, tức là `10% tổng bộ nhớ + lượng ngưỡng eviction`.

Node có thể chạm ngưỡng eviction nếu một pod dùng nhiều hơn request của nó,
hoặc nếu hệ thống dùng nhiều hơn 1GiB bộ nhớ, khiến tín hiệu `memory.available`
giảm xuống dưới 500MiB và kích hoạt ngưỡng.

### DaemonSet và eviction do áp lực node (DaemonSets and node-pressure eviction) {#daemonset}

Độ ưu tiên của pod là yếu tố chính trong việc ra quyết định eviction. Nếu bạn không muốn
kubelet evict các pod thuộc về một DaemonSet, hãy cho các pod đó độ ưu tiên
đủ cao bằng cách chỉ định một `priorityClassName` phù hợp trong spec của pod.
Bạn cũng có thể dùng độ ưu tiên thấp hơn, hoặc mặc định, để chỉ cho phép các pod của
DaemonSet đó chạy khi có đủ tài nguyên.

## Các vấn đề đã biết (Known issues)

Các phần sau mô tả những vấn đề đã biết liên quan đến việc xử lý cạn kiệt tài nguyên.

### kubelet có thể không quan sát thấy áp lực bộ nhớ ngay lập tức (kubelet may not observe memory pressure right away)

Theo mặc định, kubelet thăm dò (poll) cAdvisor để thu thập số liệu sử dụng bộ nhớ theo
chu kỳ đều đặn. Nếu mức sử dụng bộ nhớ tăng nhanh trong khoảng thời gian đó,
kubelet có thể không quan sát thấy `MemoryPressure` đủ nhanh, và OOM killer
vẫn sẽ được kích hoạt.

Bạn có thể dùng flag `--kernel-memcg-notification` để bật API thông báo `memcg`
trên kubelet, giúp nhận được thông báo ngay lập tức khi một ngưỡng
bị vượt qua.

Nếu bạn không cố đạt mức tận dụng tài nguyên cực hạn, mà chỉ muốn một mức
overcommit hợp lý, một giải pháp khả thi cho vấn đề này là dùng các flag `--kube-reserved`
và `--system-reserved` để cấp phát bộ nhớ cho hệ thống.

### Bộ nhớ active_file không được tính là bộ nhớ khả dụng (active_file memory is not considered as available memory)

Trên Linux, kernel theo dõi số byte của bộ nhớ được ánh xạ từ file (file-backed memory) trong danh sách
LRU (least recently used) hoạt động dưới dạng thống kê `active_file`. kubelet coi các vùng bộ nhớ `active_file`
là không thể thu hồi. Với các workload sử dụng nhiều lưu trữ cục bộ
dựa trên block (block-backed), bao gồm cả lưu trữ cục bộ tạm thời (ephemeral local storage), cache ở cấp kernel của dữ liệu file
và block đồng nghĩa với việc nhiều trang cache được truy cập gần đây có khả năng
bị tính là `active_file`. Nếu có đủ nhiều các buffer block của kernel nằm trong
danh sách LRU hoạt động, kubelet dễ coi đây là mức sử dụng tài nguyên cao và
taint node là đang gặp áp lực bộ nhớ — kích hoạt eviction pod.

Để biết thêm chi tiết, xem [https://github.com/kubernetes/kubernetes/issues/43916](https://github.com/kubernetes/kubernetes/issues/43916)

Bạn có thể khắc phục hành vi đó bằng cách đặt memory limit và memory request
bằng nhau cho các container có khả năng thực hiện hoạt động I/O cường độ cao. Bạn sẽ cần
ước lượng hoặc đo đạc giá trị memory limit tối ưu cho container đó.

## Tiếp theo (What's next)

- Tìm hiểu về [Eviction khởi phát qua API (API-initiated Eviction)](143-api-eviction-vi.md)
- Tìm hiểu về [Độ ưu tiên và Preemption của Pod (Pod Priority and Preemption)](141-pod-priority-preemption-vi.md)
- Tìm hiểu về [PodDisruptionBudget](339-configure-pdb-vi.md)
- Tìm hiểu về [Chất lượng dịch vụ (Quality of Service)](288-quality-service-pod-vi.md) (QoS)
- Xem [Eviction API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#create-eviction-pod-v1-core)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 7:

1. Kể ba tham số kubelet dùng để xếp thứ tự evict Pod, theo đúng thứ tự. Vì sao bài khẳng định
   kubelet **không** dùng QoS class để quyết định thứ tự, dù bảng thứ tự lại nhắc tên
   `BestEffort`, `Burstable`, `Guaranteed`?
2. Trên `k8s-worker2`, bạn muốn evict sớm hơn nên đặt trong file cấu hình kubelet đúng một
   dòng `evictionHard: memory.available: "500Mi"` và không đụng gì khác. Ngưỡng
   `nodefs.available` lúc này là bao nhiêu?
3. Một Pod `Guaranteed` có bao giờ bị evict do áp lực node không?
4. `k8s-worker2` sắp đầy đĩa. kubelet làm gì **trước khi** chấm dứt Pod của bạn? Điều kiện
   node nào xuất hiện, và vì sao node đó cũng ngừng nhận Pod mới?
5. So với `kubectl drain`, eviction do áp lực node đối xử với PodDisruptionBudget và
   `terminationGracePeriodSeconds` thế nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Theo đúng thứ tự: **(1) mức sử dụng tài nguyên của Pod có vượt quá `requests` hay không,
   (2) độ ưu tiên (Priority) của Pod, (3) mức sử dụng so với `requests`.** Không có tham số nào
   là "QoS class". QoS chỉ là **hệ quả** của requests/limits, nên nhóm Pod theo QoS cho ra kết
   quả *gần đúng* với thứ tự thật — bài dùng tên QoS cho dễ hình dung, rồi ghi chú rằng bạn chỉ
   nên dùng QoS để **ước lượng**. Bằng chứng rõ nhất là phân loại QoS không áp dụng cho request
   `EphemeralStorage`, nên khi node ở trạng thái `DiskPressure` thì cách suy theo QoS không còn
   đúng nữa.
2. **Bằng 0** — tức là tín hiệu `nodefs.available` gần như không bao giờ chạm ngưỡng nữa. Bài
   nói rõ: các giá trị mặc định của ngưỡng eviction cứng **chỉ được thiết lập nếu không có tham
   số nào bị thay đổi**; khi bạn đổi một tham số, các tham số còn lại **không được kế thừa như
   giá trị mặc định mà bị đặt về không**. Muốn giữ 10% cho `nodefs.available` thì phải khai đủ
   tất cả các ngưỡng, hoặc đặt `MergeDefaultEvictionSettings: true`.
3. **Có, trong đúng một trường hợp.** Pod `Guaranteed` không bao giờ bị evict **vì mức tiêu
   thụ của một Pod khác**. Nhưng nếu daemon hệ thống (như `kubelet`, `journald`) tiêu thụ nhiều
   hơn phần đã được dành riêng, mà trên node chỉ còn các Pod `Guaranteed` hoặc `Burstable` đang
   dùng **dưới** requests, thì kubelet **buộc phải chọn evict một trong số đó** để giữ node ổn
   định — và nó chọn Pod có **Priority thấp nhất** trước. Ngoài ra, khi node cạn inode hoặc
   PID, kubelet xếp thứ tự **chỉ bằng Priority**, vì inode và PID không có requests.
4. kubelet **thu hồi tài nguyên cấp node trước**: với bố cục một `nodefs` duy nhất như cluster
   lab, nó (1) garbage collect các pod và container đã chết, rồi (2) xóa các image không dùng
   đến. Chỉ khi làm vậy vẫn không đưa được tín hiệu xuống dưới ngưỡng, nó mới bắt đầu evict Pod
   của người dùng cuối. Điều kiện node xuất hiện là **`DiskPressure`**, và control plane **ánh
   xạ điều kiện node thành taint** `node.kubernetes.io/disk-pressure`; bộ lập lịch kiểm tra
   taint chứ không kiểm tra điều kiện node, nên Pod mới không còn được đặt lên node đó.
5. **Ngược nhau.** Eviction do áp lực node **không tôn trọng** PodDisruptionBudget và **không
   tôn trọng** `terminationGracePeriodSeconds` của Pod. Với ngưỡng **mềm**, kubelet chỉ dùng
   `eviction-max-pod-grace-period` mà quản trị viên cấu hình; với ngưỡng **cứng**, nó dùng
   grace period **`0s`**, tức là tắt ngay. Nói cách khác, khi node đang tự cứu mình thì mọi
   thỏa thuận ở tầng API đều không có hiệu lực.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

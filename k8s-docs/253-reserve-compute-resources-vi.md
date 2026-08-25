# Dành riêng tài nguyên tính toán cho các System Daemon (Reserve Compute Resources for System Daemons)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 20 — Cấu hình lại cluster đang chạy](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy), bài 6/6 ·
Giai đoạn 20 **không có lab riêng**: bạn tự chấm bằng **Checkpoint** ghi ở cuối mục giai đoạn đó
trong lộ trình, chạy trên cluster ba VM dựng ở [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md).

Đây là **bài cuối của giai đoạn 20** và là chỗ trả lời trực tiếp một vế của Checkpoint: đặt
`system-reserved` rồi **đọc lại `Allocatable` của node thấy giảm đúng lượng đã dành**. Bài nối tiếp
[110](110-manage-resources-containers-vi.md) — `requests` quyết định lập lịch — và
[142](142-node-pressure-eviction-vi.md) — eviction do áp lực node; ở đây hai thứ đó gặp nhau trong
một phép trừ.

Mọi thiết lập trong bài đều là trường của `KubeletConfiguration`, tức đặt qua file cấu hình kubelet
của bài [224](224-kubelet-config-file-vi.md) vừa đọc — chính bài này nói vậy ở mục *Trước khi bạn
bắt đầu*.

**Phải hiểu ở lần đọc này:**

- Định nghĩa `Allocatable`: **lượng tài nguyên tính toán khả dụng cho các Pod**, và
  **scheduler không cấp phát vượt quá `Allocatable`**. Ba tài nguyên được hỗ trợ là `cpu`, `memory`
  và `ephemeral-storage`; giá trị đọc được ở object `v1.Node` và ở `kubectl describe node`.
- Ba khoản bị trừ khỏi `Capacity` và mỗi khoản dành cho ai: **`kubeReserved`** cho system daemon
  **của Kubernetes** (kubelet, container runtime — **không** dành cho daemon chạy dạng Pod);
  **`systemReserved`** cho daemon **của hệ điều hành** (`sshd`, `udev`, bộ nhớ kernel, phiên đăng
  nhập người dùng); **`evictionHard`** là phần dành cho eviction và cũng **không khả dụng cho Pod**.
- Phép tính ở mục *Ví dụ minh họa* và hệ quả kép của nó: 16 CPU / 32Gi / 100Gi trừ ba khoản trên ra
  **14.5 CPU, 28.5Gi và 88Gi**; từ đó **scheduler chặn theo tổng `requests`**, còn **kubelet evict
  theo mức sử dụng thật** vượt cùng ngưỡng đó.
- Ranh giới giữa **khai báo** và **thực thi**: đặt `kubeReserved`/`systemReserved` chỉ làm
  `Allocatable` giảm đi. Muốn *ép* daemon nằm trong phần dành riêng thì phải thêm `kube-reserved`
  hoặc `system-reserved` vào **`enforceNodeAllocatable`** *và* chỉ định `kubeReservedCgroup` hoặc
  `systemReservedCgroup`. kubelet **không tự tạo** cgroup đó và **khởi động thất bại** nếu cgroup
  không hợp lệ; với driver `systemd` tên cgroup phải kèm hậu tố `.slice`.
- Thứ tự khuyến nghị ở mục *Hướng dẫn chung*: bắt đầu bằng thực thi `Allocatable` cho **`pods`**,
  rồi mới tới tài nguyên nén được, rồi mới tới tài nguyên không nén được — và lý do phải rất thận
  trọng với `systemReserved`: thực thi sai làm dịch vụ hệ thống **thiếu CPU, bị OOM kill, hoặc
  không fork được tiến trình**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Danh sách CPU dành riêng tường minh* — `reservedSystemCPUs` | bài nói rõ nó được thiết kế cho Telco/NFV, và việc chuyển daemon vào cpuset đó cần **cơ chế khác bên ngoài Kubernetes** | không cần cho lộ trình; ba khoản trừ phải nhớ nằm ở các mục *Kube Reserved*, *System Reserved* và *Ngưỡng Eviction* của chính bài này |
| Mục *Cấu hình cgroup driver* trong bài | đã chốt và đã đối chiếu xong | bài [218](218-configure-cgroup-driver-vi.md), bài trước trong chính giai đoạn 20; kiểm chứng ở [Lab 2](labs/LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md) phần B2.2 |
| Cơ chế eviction mà `evictionHard` kích hoạt — ngưỡng, thứ tự trục xuất theo QoS | ở đây chỉ cần biết phần này **bị trừ khỏi `Allocatable`** | bài [142](142-node-pressure-eviction-vi.md), đã đọc ở [giai đoạn 7a](00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction) |
| Thực thi thật `systemReserved` trên cluster lab | mỗi worker chỉ 2 vCPU và 6 GB; bài khuyến nghị chỉ thực thi khi đã đo đạc node toàn diện và tự tin khôi phục được nếu tiến trình hệ thống bị oom-kill | Checkpoint giai đoạn 20 chỉ yêu cầu **đặt** `system-reserved` rồi **đọc lại `Allocatable`**, không yêu cầu thực thi |

---

Các node Kubernetes có thể được lập lịch (schedule) tới mức `Capacity` (dung lượng tối đa). Theo
mặc định, các Pod có thể tiêu thụ toàn bộ dung lượng khả dụng trên một node. Đây là một vấn đề vì
các node thường chạy khá nhiều system daemon (tiến trình nền của hệ thống) phục vụ cho hệ điều
hành và cho chính Kubernetes. Trừ khi tài nguyên được dành riêng cho các system daemon này, các
Pod và các system daemon sẽ cạnh tranh tài nguyên với nhau và dẫn đến tình trạng thiếu hụt tài
nguyên (resource starvation) trên node.

`kubelet` cung cấp một tính năng có tên 'Node Allocatable' giúp dành riêng (reserve) tài nguyên
tính toán cho các system daemon. Kubernetes khuyến nghị quản trị viên cluster cấu hình
'Node Allocatable' dựa trên mật độ workload trên mỗi node.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, nhập `kubectl version`.

Bạn có thể cấu hình các [thiết lập cấu hình](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)
kubelet dưới đây bằng [file cấu hình kubelet](224-kubelet-config-file-vi.md).

## Node Allocatable

![node capacity](https://kubernetes.io/images/docs/node-capacity.svg)

'Allocatable' trên một node Kubernetes được định nghĩa là lượng tài nguyên tính toán khả dụng cho
các Pod. Scheduler không cấp phát vượt quá (over-subscribe) 'Allocatable'. Hiện tại, 'CPU',
'memory' và 'ephemeral-storage' là các tài nguyên được hỗ trợ.

Node Allocatable được thể hiện như một phần của object `v1.Node` trong API và như một phần của
output lệnh `kubectl describe node` trên CLI.

Tài nguyên có thể được dành riêng cho hai nhóm system daemon trong `kubelet`.

### Bật QoS và cgroup ở cấp Pod (Enabling QoS and Pod level cgroups)

Để thực thi (enforce) đúng đắn các ràng buộc node allocatable trên node, bạn phải bật hệ thống
phân cấp cgroup mới thông qua thiết lập `cgroupsPerQOS`. Thiết lập này được bật theo mặc định.
Khi được bật, `kubelet` sẽ đặt tất cả Pod của người dùng cuối vào bên dưới một hệ thống phân cấp
cgroup do `kubelet` quản lý.

### Cấu hình cgroup driver (Configuring a cgroup driver)

`kubelet` hỗ trợ thao tác trên hệ thống phân cấp cgroup của máy chủ (host) bằng một cgroup
driver. Driver được cấu hình thông qua thiết lập `cgroupDriver`.

Các giá trị được hỗ trợ như sau:

* `cgroupfs` là driver mặc định, thao tác trực tiếp trên filesystem cgroup của host để quản lý
  các cgroup sandbox.
* `systemd` là driver thay thế, quản lý các cgroup sandbox bằng các slice tạm thời (transient
  slice) cho những tài nguyên mà hệ thống init đó hỗ trợ.

Tùy vào cấu hình của container runtime đi kèm, người vận hành có thể phải chọn một cgroup driver
cụ thể để đảm bảo hệ thống hoạt động đúng. Ví dụ, nếu người vận hành dùng cgroup driver `systemd`
do runtime `containerd` cung cấp, thì `kubelet` phải được cấu hình để dùng cgroup driver
`systemd`.

### Kube Reserved

- **Thiết lập trong KubeletConfiguration**: `kubeReserved: {}`. Giá trị ví dụ
  `{cpu: 100m, memory: 100Mi, ephemeral-storage: 1Gi, pid=1000}`
- **Thiết lập trong KubeletConfiguration**: `kubeReservedCgroup: ""`

`kubeReserved` dùng để khai báo lượng tài nguyên dành riêng cho các system daemon của Kubernetes
như `kubelet`, `container runtime`, v.v. Nó không dùng để dành riêng tài nguyên cho các system
daemon chạy dưới dạng Pod. `kubeReserved` thường là một hàm số theo `mật độ Pod` (pod density)
trên các node.

Ngoài `cpu`, `memory` và `ephemeral-storage`, có thể chỉ định thêm `pid` để dành riêng một số
lượng process ID nhất định cho các system daemon của Kubernetes.

Để tùy chọn thực thi `kubeReserved` lên các system daemon của Kubernetes, hãy chỉ định control
group cha của các kube daemon làm giá trị cho thiết lập `kubeReservedCgroup`, và
[thêm `kube-reserved` vào `enforceNodeAllocatable`](#enforcing-node-allocatable).

Khuyến nghị đặt các system daemon của Kubernetes bên dưới một control group cấp cao nhất (ví dụ
`runtime.slice` trên các máy dùng systemd). Lý tưởng nhất, mỗi system daemon nên chạy trong một
control group con của riêng nó. Tham khảo
[đề xuất thiết kế](https://git.k8s.io/design-proposals-archive/node/node-allocatable.md#recommended-cgroups-setup)
để biết thêm chi tiết về hệ thống phân cấp control group được khuyến nghị.

Lưu ý rằng kubelet **không** tự tạo `kubeReservedCgroup` nếu nó chưa tồn tại. kubelet sẽ khởi
động thất bại nếu một cgroup không hợp lệ được chỉ định. Với cgroup driver `systemd`, bạn nên
tuân theo một khuôn mẫu cụ thể khi đặt tên cho cgroup mà bạn định nghĩa: tên đó phải là giá trị
bạn đặt cho `kubeReservedCgroup`, kèm theo hậu tố `.slice`.

### System Reserved

- **Thiết lập trong KubeletConfiguration**: `systemReserved: {}`. Giá trị ví dụ
  `{cpu: 100m, memory: 100Mi, ephemeral-storage: 1Gi, pid=1000}`
- **Thiết lập trong KubeletConfiguration**: `systemReservedCgroup: ""`

`systemReserved` dùng để khai báo lượng tài nguyên dành riêng cho các system daemon của hệ điều
hành như `sshd`, `udev`, v.v. `systemReserved` cũng nên dành riêng `memory` cho `kernel`, vì bộ
nhớ của `kernel` hiện chưa được tính vào các Pod trong Kubernetes. Việc dành riêng tài nguyên cho
các phiên đăng nhập của người dùng cũng được khuyến nghị (`user.slice` trong thế giới systemd).

Ngoài `cpu`, `memory` và `ephemeral-storage`, có thể chỉ định thêm `pid` để dành riêng một số
lượng process ID nhất định cho các system daemon của hệ điều hành.

Để tùy chọn thực thi `systemReserved` lên các system daemon, hãy chỉ định control group cha của
các system daemon của hệ điều hành làm giá trị cho thiết lập `systemReservedCgroup`, và
[thêm `system-reserved` vào `enforceNodeAllocatable`](#enforcing-node-allocatable).

Khuyến nghị đặt các system daemon của hệ điều hành bên dưới một control group cấp cao nhất (ví dụ
`system.slice` trên các máy dùng systemd).

Lưu ý rằng `kubelet` **không** tự tạo `systemReservedCgroup` nếu nó chưa tồn tại. `kubelet` sẽ
thất bại nếu một cgroup không hợp lệ được chỉ định. Với cgroup driver `systemd`, bạn nên tuân
theo một khuôn mẫu cụ thể khi đặt tên cho cgroup mà bạn định nghĩa: tên đó phải là giá trị bạn
đặt cho `systemReservedCgroup`, kèm theo hậu tố `.slice`.

### Danh sách CPU dành riêng tường minh (Explicitly Reserved CPU List) {#explicitly-reserved-cpu-list}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.17 [stable]`

**Thiết lập trong KubeletConfiguration**: `reservedSystemCPUs:`. Giá trị ví dụ `0-3`

`reservedSystemCPUs` dùng để định nghĩa một tập CPU (cpuset) tường minh cho các system daemon
của hệ điều hành và các system daemon của Kubernetes. `reservedSystemCPUs` dành cho những hệ
thống không có ý định định nghĩa các cgroup cấp cao nhất riêng biệt cho system daemon của hệ
điều hành và system daemon của Kubernetes xét theo tài nguyên cpuset.
Nếu kubelet **không** có `kubeReservedCgroup` và `systemReservedCgroup`, thì cpuset tường minh
do `reservedSystemCPUs` cung cấp sẽ được ưu tiên hơn các CPU được định nghĩa bởi hai tùy chọn
`kubeReservedCgroup` và `systemReservedCgroup`.

Tùy chọn này được thiết kế đặc biệt cho các trường hợp sử dụng Telco/NFV, nơi các ngắt
(interrupt)/bộ định thời (timer) không được kiểm soát có thể ảnh hưởng đến hiệu năng của
workload. Bạn có thể dùng tùy chọn này để định nghĩa cpuset tường minh cho các daemon của hệ
thống/Kubernetes cũng như cho các interrupt/timer, nhờ đó phần CPU còn lại trên hệ thống có thể
được dùng riêng cho các workload, ít bị ảnh hưởng bởi các interrupt/timer không được kiểm soát
hơn. Để di chuyển các system daemon, các daemon của Kubernetes và các interrupt/timer sang cpuset
tường minh được định nghĩa bởi tùy chọn này, cần dùng cơ chế khác bên ngoài Kubernetes. Ví dụ:
trên CentOS, bạn có thể làm điều này bằng bộ công cụ tuned.

### Ngưỡng Eviction (Eviction Thresholds)

**Thiết lập trong KubeletConfiguration**: `evictionHard: {memory.available: "100Mi", nodefs.available: "10%", nodefs.inodesFree: "5%", imagefs.available: "15%"}`.
Giá trị ví dụ: `{memory.available: "<500Mi"}`

Áp lực bộ nhớ (memory pressure) ở cấp node dẫn đến tình trạng OOM (out of memory) của hệ thống,
ảnh hưởng đến toàn bộ node và tất cả các Pod đang chạy trên đó. Node có thể tạm thời ngừng hoạt
động cho đến khi bộ nhớ được thu hồi. Để tránh (hoặc giảm xác suất xảy ra) OOM hệ thống, kubelet
cung cấp cơ chế quản lý
[cạn kiệt tài nguyên](142-node-pressure-eviction-vi.md)
(out of resource). Việc trục xuất (eviction) chỉ được hỗ trợ đối với `memory` và
`ephemeral-storage`. Bằng cách dành riêng một phần bộ nhớ thông qua thiết lập `evictionHard`,
`kubelet` sẽ cố gắng trục xuất các Pod mỗi khi lượng bộ nhớ khả dụng trên node giảm xuống dưới
giá trị đã dành riêng. Về mặt giả định, nếu các system daemon không tồn tại trên node, các Pod
cũng không thể dùng nhiều hơn `capacity - eviction-hard`. Vì lý do này, phần tài nguyên dành
riêng cho eviction không khả dụng cho các Pod.

### Thực thi Node Allocatable (Enforcing Node Allocatable) {#enforcing-node-allocatable}

**Thiết lập trong KubeletConfiguration**: `enforceNodeAllocatable: [pods]`. Giá trị ví dụ:
`[pods,system-reserved,kube-reserved]`

Scheduler coi 'Allocatable' là `capacity` khả dụng cho các Pod.

Theo mặc định, `kubelet` thực thi 'Allocatable' đối với toàn bộ các Pod. Việc thực thi được thực
hiện bằng cách trục xuất các Pod mỗi khi tổng mức sử dụng của tất cả các Pod vượt quá
'Allocatable'. Bạn có thể xem thêm chi tiết về chính sách eviction tại trang
[eviction do áp lực node](142-node-pressure-eviction-vi.md)
(node pressure eviction). Việc thực thi này được điều khiển bằng cách chỉ định giá trị `pods`
cho thiết lập `enforceNodeAllocatable` trong KubeletConfiguration.

Một cách tùy chọn, có thể cấu hình để `kubelet` thực thi `kubeReserved` và `systemReserved` bằng
cách chỉ định các giá trị `kube-reserved` và `system-reserved` trong cùng thiết lập đó. Ngoài
ra, có thể chỉ thực thi các tài nguyên nén được (compressible resources) bằng cách chỉ định
`kube-reserved-compressible` và `system-reserved-compressible`. Lưu ý rằng để thực thi
`kubeReserved` hoặc `systemReserved`, cần chỉ định tương ứng `kubeReservedCgroup` hoặc
`systemReservedCgroup`.

## Hướng dẫn chung (General Guidelines)

Các system daemon được kỳ vọng sẽ được đối xử tương tự như
[các Pod Guaranteed](288-quality-service-pod-vi.md#create-a-pod-that-gets-assigned-a-qos-class-of-guaranteed).
Các system daemon có thể bùng nổ (burst) mức sử dụng trong phạm vi control group bao quanh chúng,
và hành vi này cần được quản lý như một phần của việc triển khai Kubernetes. Ví dụ, `kubelet` nên
có control group của riêng nó và chia sẻ phần tài nguyên `kubeReserved` với container runtime.
Tuy nhiên, kubelet không thể bùng nổ và dùng hết toàn bộ tài nguyên khả dụng của Node nếu
`kubeReserved` được thực thi.

Hãy hết sức cẩn thận khi thực thi phần dành riêng `systemReserved`, vì nó có thể khiến các dịch
vụ hệ thống quan trọng bị thiếu CPU, bị OOM kill, hoặc không thể fork tiến trình trên node.
Khuyến nghị là chỉ thực thi `systemReserved` khi người dùng đã đo đạc (profile) các node của
mình một cách toàn diện để đưa ra các ước lượng chính xác, và tự tin vào khả năng khôi phục nếu
bất kỳ tiến trình nào trong nhóm đó bị oom-kill.

Việc chỉ thực thi các tài nguyên nén được cho `kubeReserved` và `systemReserved` ít có khả năng
gây gián đoạn hơn, trong khi vẫn đảm bảo tài nguyên được phân bổ hợp lý khi có sự tranh chấp
(contention).

* Để bắt đầu, hãy thực thi 'Allocatable' đối với `pods`.
* Khi đã có hệ thống giám sát (monitoring) và cảnh báo (alerting) đầy đủ để theo dõi các kube
  daemon và system daemon, hãy thử thực thi các tài nguyên nén được cho `kubeReserved` và
  `systemReserved`.
* Thử thực thi các tài nguyên không nén được (non-compressible) của `kubeReserved` dựa trên các
  ước lượng kinh nghiệm về mức sử dụng.
* Nếu thực sự cần thiết, hãy thực thi các tài nguyên không nén được của `systemReserved` theo
  thời gian.

Yêu cầu tài nguyên của các kube system daemon có thể tăng lên theo thời gian khi ngày càng nhiều
tính năng được bổ sung. Theo thời gian, dự án Kubernetes sẽ cố gắng giảm mức sử dụng của các
system daemon trên node, nhưng đó chưa phải là ưu tiên ở thời điểm hiện tại. Vì vậy, hãy dự trù
rằng dung lượng `Allocatable` sẽ giảm trong các bản phát hành tương lai.

## Ví dụ minh họa (Example Scenario)

Dưới đây là một ví dụ minh họa cách tính Node Allocatable:

* Node có `32Gi` `memory`, `16 CPU` và `100Gi` `Storage`
* `kubeReserved` được đặt thành `{cpu: 1000m, memory: 2Gi, ephemeral-storage: 1Gi}`
* `systemReserved` được đặt thành `{cpu: 500m, memory: 1Gi, ephemeral-storage: 1Gi}`
* `evictionHard` được đặt thành `{memory.available: "<500Mi", nodefs.available: "<10%"}`

Trong kịch bản này, 'Allocatable' sẽ là 14.5 CPU, 28.5Gi bộ nhớ và `88Gi` lưu trữ cục bộ.
Scheduler đảm bảo rằng tổng `requests` về bộ nhớ của tất cả các Pod trên node này không vượt quá
28.5Gi và lưu trữ không vượt quá 88Gi.
Kubelet trục xuất các Pod mỗi khi tổng mức sử dụng bộ nhớ của các Pod vượt quá 28.5Gi, hoặc khi
tổng mức sử dụng đĩa vượt quá 88Gi. Nếu tất cả các tiến trình trên node tiêu thụ CPU nhiều nhất
có thể, thì tổng cộng các Pod không thể tiêu thụ nhiều hơn 14.5 CPU.

Nếu `kubeReserved` và/hoặc `systemReserved` không được thực thi và các system daemon vượt quá
phần dành riêng của chúng, `kubelet` sẽ trục xuất các Pod mỗi khi tổng mức sử dụng bộ nhớ của
node cao hơn 31.5Gi hoặc `storage` lớn hơn 90Gi.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 20:

1. `Capacity` và `Allocatable` của một node khác nhau ở đâu? Ba khoản nào bị trừ ra, và mỗi khoản
   dành cho nhóm tiến trình nào?
2. Trên `lab-k8s-worker2` — 2 vCPU, 6 GB — bạn đặt `kubeReserved: {cpu: 200m, memory: 512Mi}`,
   `systemReserved: {cpu: 200m, memory: 512Mi}` và giữ `evictionHard: {memory.available: "100Mi"}`.
   `Allocatable` cho `cpu` bằng bao nhiêu, phép trừ cho `memory` gồm những số hạng nào, và bạn đọc
   kết quả ở đâu?
3. **Câu bẫy.** Bạn đặt `kubeReserved: {cpu: 500m}` và áp dụng thành công. kubelet với container
   runtime từ giờ có bị **giới hạn** trong 500m không? Nếu muốn đúng như vậy thì còn phải làm gì?
4. Bài cảnh báo hai điều về `kubeReservedCgroup` và `systemReservedCgroup` chưa tồn tại hoặc đặt
   sai. Hai điều đó là gì, và với cgroup driver `systemd` thì tên cgroup phải theo khuôn nào?
5. Vì sao bài khuyên **bắt đầu bằng thực thi `Allocatable` cho `pods`** rồi mới tính tới hai khoản
   dành riêng, và vì sao `systemReserved` là khoản nguy hiểm nhất khi thực thi?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. `Capacity` là **dung lượng tối đa của node**; `Allocatable` là **lượng tài nguyên khả dụng cho
   các Pod**, và scheduler **không cấp phát vượt quá** nó. Ba khoản bị trừ: **`kubeReserved`** cho
   các system daemon **của Kubernetes** như kubelet và container runtime; **`systemReserved`** cho
   các system daemon **của hệ điều hành** như `sshd`, `udev`, cộng bộ nhớ cho kernel và các phiên
   đăng nhập của người dùng; và **`evictionHard`** — phần dành cho eviction, cũng **không khả dụng
   cho Pod**.
2. `cpu`: **2000m − 200m − 200m = 1600m**, tức **1.6 CPU** — `evictionHard` ở đây chỉ đặt cho
   `memory` nên không trừ vào CPU. `memory`: **`Capacity` của node − 512Mi − 512Mi − 100Mi**; lấy
   `Capacity` thật từ node chứ đừng giả định 6 GB tròn. Đọc kết quả ở **`kubectl describe node
   lab-k8s-worker2`**, hoặc trong object `v1.Node` — bài nói Node Allocatable được thể hiện ở cả
   hai chỗ đó.
3. **Không.** Đặt `kubeReserved` mới chỉ là **khai báo**: nó làm `Allocatable` giảm đi để scheduler
   chừa chỗ, chứ **không** ép kubelet và container runtime nằm trong 500m đó. Muốn thực thi thì
   phải làm hai việc cùng lúc: thêm **`kube-reserved` vào `enforceNodeAllocatable`** *và* chỉ định
   **`kubeReservedCgroup`** trỏ tới control group cha của các kube daemon. Đây là chỗ dễ tưởng đã
   xong nhất — bài tách hẳn hai khái niệm khai báo và thực thi ra.
4. Thứ nhất: **kubelet không tự tạo cgroup đó nếu nó chưa tồn tại**. Thứ hai: **kubelet sẽ khởi
   động thất bại nếu cgroup được chỉ định không hợp lệ** — tức đặt sai là node không lên. Với
   cgroup driver `systemd`, tên phải là **đúng giá trị bạn đặt cho `kubeReservedCgroup` /
   `systemReservedCgroup` kèm hậu tố `.slice`**.
5. Vì thực thi `Allocatable` cho `pods` là **hành vi mặc định và ít rủi ro nhất**: kubelet chỉ evict
   Pod khi tổng mức sử dụng của các Pod vượt `Allocatable`. Hai khoản dành riêng thì bài xếp sau, và
   còn khuyên thử **tài nguyên nén được trước** vì ít khả năng gây gián đoạn hơn. `systemReserved`
   nguy hiểm nhất vì thực thi nó có thể khiến **các dịch vụ hệ thống quan trọng thiếu CPU, bị OOM
   kill, hoặc không thể fork tiến trình trên node** — bài khuyến nghị chỉ làm sau khi đã đo đạc node
   toàn diện và tự tin khôi phục được.

</details>

Hết giai đoạn 20. Câu nào chưa trả lời được thì quay lại đúng mục tương ứng, rồi làm
**Checkpoint** ở cuối mục
[Giai đoạn 20 — Cấu hình lại cluster đang chạy](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy):
đổi một tham số kubelet qua file cấu hình rồi chứng minh nó có hiệu lực, bật một feature gate và
kiểm chứng, đặt `system-reserved` rồi đọc lại `Allocatable` của node thấy giảm đúng lượng đã dành.

# Dành riêng tài nguyên tính toán cho các System Daemon (Reserve Compute Resources for System Daemons)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/>

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
kubelet dưới đây bằng [file cấu hình kubelet](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/).

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

### Danh sách CPU dành riêng tường minh (Explicitly Reserved CPU List)

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
[cạn kiệt tài nguyên](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)
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
[eviction do áp lực node](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)
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
[các Pod Guaranteed](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/#create-a-pod-that-gets-assigned-a-qos-class-of-guaranteed).
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

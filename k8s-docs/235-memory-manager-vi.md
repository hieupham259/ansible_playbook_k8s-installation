# Kiểm soát các chính sách quản lý bộ nhớ trên một Node (Control Memory Management Policies on a Node)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/memory-manager/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 25 — Quản trị tài nguyên theo namespace](00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace),
bài 5/7 · Phần II không có lab riêng: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md), tự chấm bằng **Checkpoint giai đoạn 25**.

Bài này **khác hẳn** ba bài đầu giai đoạn 25: nó không phải chính sách theo namespace mà là chính
sách **cấp node**, đặt trong `KubeletConfiguration` — mà việc đổi tham số kubelet của một node
đang chạy thuộc [giai đoạn 20](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy),
bài [224](224-kubelet-config-file-vi.md). Nền lý thuyết đã có ở bài
[74 — Các trình quản lý tài nguyên](74-resource-managers-vi.md) (nhóm 7b).

Nói thẳng về giới hạn của cluster lab: cả ba VM chỉ có **một NUMA domain**, và chính sách mặc
định là `None`. Trên cluster này bạn **đo được** hai thứ: đếm số NUMA node bằng
`ls -d /sys/devices/system/node/node[0-9]*` và đọc `memoryManagerPolicy` **hiệu lực** qua
`configz`, đúng như [Lab 14 phần B10.4](labs/LAB-14-CRD-VA-OPERATOR.md#b104-topology-manager-đang-ở-policy-nào)
đã làm. Bạn **không đo được** hiệu quả của chính sách `Static`: với một NUMA domain thì "số NUMA
node tối thiểu" luôn bằng 1, không có ranh giới nào để căn. Đọc bài để biết cấu hình đúng khi
vận hành máy chủ vật lý nhiều socket, đừng sửa kubelet của lab để "thử cho biết".

**Phải hiểu ở lần đọc này:**

- Memory Manager đảm bảo cấp phát bộ nhớ (và hugepages) cho Pod thuộc lớp QoS `Guaranteed`, và
  nó chỉ là một **hint provider**: nó sinh gợi ý NUMA affinity rồi đưa cho **Topology Manager**,
  và chính Topology Manager mới quyết định Pod bị từ chối hay được chấp nhận vào node. Muốn căn
  chỉnh bộ nhớ với tài nguyên khác thì **CPU Manager và Topology Manager phải được bật trước**
  (mục mở đầu, *Memory Manager hoạt động như thế nào?* và *Điều kiện tiên quyết để căn chỉnh tài
  nguyên*).
- Ba chính sách đặt qua `memoryManagerPolicy`: `None` (mặc định — hành xử như thể Memory Manager
  không tồn tại, trả về gợi ý topology mặc định), `Static` (chỉ Linux), `BestEffort` (chỉ
  Windows). Dưới `Static`, chỉ Pod `Guaranteed` nhận gợi ý cụ thể; Pod `BestEffort` và `Burstable`
  nhận gợi ý mặc định và không được dành riêng bộ nhớ (mục *Các chính sách*).
- Khi chọn `Static` thì **bắt buộc** phải cấu hình `reservedMemory`. Lý do rất cụ thể: ngưỡng
  `evictionHard` mặc định là `100Mi` chứ không phải 0, nên nếu bỏ qua, kubelet **không khởi động
  Memory Manager** và báo lỗi (mục *Cấu hình bộ nhớ dành riêng* và khối Ghi chú của nó).
- Ràng buộc số học phải thuộc: tổng `reservedMemory` trên tất cả NUMA node phải **bằng**
  `kubeReserved` + `systemReserved` + `evictionHard` `memory.available`. Không tuân theo thì
  Memory Manager báo lỗi ngay khi khởi động (mục *Ràng buộc về việc dành riêng bộ nhớ NUMA*).
- Điều kiện để một Pod vào lớp `Guaranteed` theo đúng hai ví dụ của bài: `requests` **bằng**
  `limits`, và **cả CPU lẫn memory đều phải được chỉ định**. CPU nguyên (`"2"`) hay CPU lẻ
  (`"300m"`) đều thuộc `Guaranteed` (mục *Đưa một Pod vào lớp QoS Guaranteed*).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Trước khi bạn bắt đầu* — minikube và ba playground trực tuyến | cluster lab đã có sẵn ba VM; lộ trình không dùng minikube hay cluster dùng chung | [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |
| Mục *Hỗ trợ Windows* và chính sách `BestEffort` | ba node lab đều là Linux; tính năng còn alpha, cần feature gate và container runtime hỗ trợ | [Giai đoạn 15 — Windows](00-ALO-TRINH-ADMIN.md#giai-đoạn-15--windows-nếu-môi-trường-có-node-windows), chỉ mở khi môi trường thật có node Windows |
| Mục *Cú pháp bộ nhớ dành riêng* (Ví dụ 1, Ví dụ 2, `hugepages-1Gi`) và *Các cấu hình cần tránh* | chỉ dùng được khi bật `Static` trên máy nhiều NUMA domain; VM lab chỉ có một | khi vận hành máy chủ vật lý nhiều socket; đối chiếu bài [74](74-resource-managers-vi.md) ở [nhóm 7b](00-ALO-TRINH-ADMIN.md#7b-chính-sách-giới-hạn-tài-nguyên) — đã đọc |
| Sơ đồ luồng admission và các bộ đếm nội bộ *Node Map and Memory Maps* | là chi tiết cài đặt bên trong kubelet, chỉ cần khi phải tra lỗi thật | bài [313 — Khắc phục sự cố Topology Management](313-debug-topology-vi.md) (dòng Thực hành [giai đoạn 14](00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng)) — đã đọc, kiểm chứng ở [Lab 14 B10.4](labs/LAB-14-CRD-VA-OPERATOR.md#b104-topology-manager-đang-ở-policy-nào) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.32 [stable]`

*Memory Manager* (trình quản lý bộ nhớ) của Kubernetes kích hoạt tính năng cấp phát bộ nhớ
(và hugepages) được đảm bảo cho các Pod thuộc lớp QoS (QoS class) `Guaranteed`.

Memory Manager sử dụng một giao thức sinh gợi ý (hint generation protocol) để đưa ra mối quan hệ
gắn kết NUMA (NUMA affinity) phù hợp nhất cho một Pod. Memory Manager cung cấp các gợi ý affinity
này cho trình quản lý trung tâm (*Topology Manager*). Dựa trên cả các gợi ý và chính sách của
Topology Manager, Pod sẽ bị từ chối hoặc được chấp nhận vào node.

Hơn nữa, Memory Manager đảm bảo rằng lượng bộ nhớ mà một Pod yêu cầu được cấp phát từ số lượng
NUMA node tối thiểu.

Để tìm hiểu kiến thức nền về tài nguyên bộ nhớ cho Pod, hãy đọc
[Gán tài nguyên bộ nhớ cho Container và Pod](264-assign-memory-resource-vi.md).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Phiên bản Kubernetes server của bạn phải bằng hoặc mới hơn v1.32. Để kiểm tra phiên bản, nhập
`kubectl version`. Nếu bạn đang chạy một phiên bản Kubernetes cũ hơn, hãy xem tài liệu tương ứng
với phiên bản Kubernetes mà bạn đang chạy.

### Điều kiện tiên quyết để căn chỉnh tài nguyên (Resource alignment prerequisites) {#resource-alignment-prerequisites}

Để căn chỉnh (align) tài nguyên bộ nhớ với các tài nguyên khác được yêu cầu trong spec của Pod:

- CPU Manager phải được bật và chính sách CPU Manager phù hợp phải được cấu hình trên Node.
  Xem [kiểm soát các chính sách quản lý CPU](200-cpu-management-policies-vi.md);
- Topology Manager phải được bật và chính sách Topology Manager phù hợp phải được cấu hình trên Node.
  Xem [kiểm soát các chính sách quản lý topology](259-topology-manager-vi.md).

### Hỗ trợ Windows (Windows support)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.32 [alpha]`

Hỗ trợ Windows có thể được bật thông qua feature gate `WindowsCPUAndMemoryAffinity`
và yêu cầu container runtime phải hỗ trợ tính năng này.
Chỉ các chính sách [None](#policy-none) và [BestEffort](#policy-best-effort) được hỗ trợ trên Windows.

## Memory Manager hoạt động như thế nào? (How does the Memory Manager operate?)

Đối với các node Linux, Memory Manager cung cấp việc cấp phát bộ nhớ (và hugepages) được đảm bảo
cho các Pod thuộc lớp QoS Guaranteed.
Để đưa Memory Manager vào hoạt động ngay lập tức, hãy làm theo hướng dẫn trong mục
[Cấu hình Memory Manager](#memory-manager-configuration), và sau đó,
chuẩn bị rồi triển khai một Pod `Guaranteed` như minh họa trong mục
[Đưa một Pod vào lớp QoS Guaranteed](#placing-a-pod-in-the-guaranteed-qos-class).

Memory Manager là một bên cung cấp gợi ý (hint provider), và nó cung cấp các gợi ý topology cho
Topology Manager; Topology Manager sau đó căn chỉnh các tài nguyên được yêu cầu theo các gợi ý
topology này. Trên Linux, nó cũng áp đặt `cgroups` (cụ thể là `cpuset.mems`) cho các Pod.
Sơ đồ luồng đầy đủ liên quan tới quá trình chấp nhận (admission) và triển khai Pod được minh họa
dưới đây:

![Memory Manager trong quá trình chấp nhận và triển khai Pod](https://kubernetes.io/images/docs/memory-manager-diagram.svg)

Trong quá trình này, Memory Manager cập nhật các bộ đếm nội bộ của nó được lưu trong
[Node Map and Memory Maps][2] để quản lý việc cấp phát bộ nhớ được đảm bảo.

Memory Manager được kích hoạt trong lúc kubelet khởi động nếu quản trị viên node cấu hình
`reservedMemory` cho kubelet (xem mục [Cấu hình bộ nhớ dành riêng](#reserved-memory-flag)).
Trong trường hợp này, kubelet cập nhật node map của nó để phản ánh phần bộ nhớ dành riêng
(reservation) này.

Khi chính sách `Static` được cấu hình, bạn **bắt buộc phải** cấu hình bộ nhớ dành riêng cho node
(ví dụ, bằng trường cấu hình `reservedMemory` trong cấu hình kubelet).

Một chủ đề quan trọng trong ngữ cảnh hoạt động của Memory Manager là việc quản lý các nhóm NUMA
(NUMA groups). Mỗi khi yêu cầu bộ nhớ của Pod vượt quá dung lượng của một NUMA node đơn lẻ,
Memory Manager sẽ cố gắng tạo một nhóm bao gồm nhiều NUMA node và có dung lượng bộ nhớ mở rộng.

## Cấu hình Memory Manager (Memory Manager configuration) {#memory-manager-configuration}

Các Manager khác phải được cấu hình từ trước (xem
[điều kiện tiên quyết để căn chỉnh tài nguyên](#resource-alignment-prerequisites)).
Đặt trường cấu hình `memoryManagerPolicy` trong
[cấu hình kubelet](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)
thành tên của [chính sách](#policies) mà bạn chọn.

Tùy chọn, một lượng bộ nhớ có thể được dành riêng cho các tiến trình hệ thống hoặc tiến trình
kubelet để tăng độ ổn định của node (xem mục [Cấu hình bộ nhớ dành riêng](#reserved-memory-flag)).

### Các chính sách (Policies) {#policies}

Memory manager của Kubernetes cung cấp ba chính sách. Bạn có thể chọn một chính sách qua trường
cấu hình `memoryManagerPolicy` trong cấu hình kubelet; các giá trị khả dụng trong Kubernetes
v1.36 là:

* [`None`](#policy-none) (mặc định)
* [`Static`](#policy-static) (chỉ trên Linux)
* [`BestEffort`](#policy-best-effort) (chỉ trên Windows)

#### Chính sách None (None policy) {#policy-none}

Đây là chính sách mặc định và không ảnh hưởng tới việc cấp phát bộ nhớ theo bất kỳ cách nào.
Nó hoạt động giống như thể Memory Manager hoàn toàn không tồn tại.

Chính sách `None` trả về gợi ý topology mặc định. Gợi ý đặc biệt này biểu thị rằng Hint Provider
(trong trường hợp này là Memory Manager) không có ưu tiên nào về NUMA affinity đối với bất kỳ
tài nguyên nào.

#### Chính sách Static (Static policy) {#policy-static}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.32 [stable]`

**Chính sách này chỉ được hỗ trợ trên Linux.**

Trong trường hợp Pod `Guaranteed`, chính sách Memory Manager `Static` trả về các gợi ý topology
liên quan tới tập hợp các NUMA node nơi bộ nhớ có thể được đảm bảo, và dành riêng bộ nhớ đó bằng
cách cập nhật đối tượng [NodeMap][2] nội bộ.

Trong trường hợp Pod `BestEffort` hoặc `Burstable`, chính sách Memory Manager `Static` gửi lại
gợi ý topology mặc định vì không có yêu cầu về bộ nhớ được đảm bảo, và không dành riêng bộ nhớ
trong đối tượng [NodeMap][2] nội bộ.

Chính sách này chỉ được hỗ trợ trên Linux.

#### Chính sách BestEffort (BestEffort policy) {#policy-best-effort}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.32 [alpha]`

**Chính sách này chỉ được hỗ trợ trên Windows.**

Trên Windows, việc gán NUMA node hoạt động khác với Linux.
Không có cơ chế nào đảm bảo rằng việc truy cập bộ nhớ chỉ đến từ một NUMA node cụ thể.
Thay vào đó, bộ lập lịch (scheduler) của hệ điều hành Windows sẽ chọn NUMA node tối ưu nhất dựa
trên các CPU đã được gán. Windows có thể sử dụng các NUMA node khác nếu bộ lập lịch của Windows
cho rằng chúng tối ưu hơn.

Chính sách này vẫn theo dõi lượng bộ nhớ khả dụng và lượng bộ nhớ được yêu cầu thông qua
_node map_ nội bộ. Memory manager sẽ cố gắng hết sức (best effort) để đảm bảo có đủ bộ nhớ khả
dụng trên một NUMA node trước khi thực hiện việc gán tài nguyên.
Điều này có nghĩa là trong hầu hết các trường hợp, việc gán bộ nhớ sẽ hoạt động đúng như chỉ định.

## Cấu hình bộ nhớ dành riêng (Reserved memory configuration) {#reserved-memory-flag}

Với vai trò quản trị viên, bạn có thể cấu hình tổng lượng bộ nhớ dành riêng (reserved memory)
cho một node. Giá trị được cấu hình trước này sau đó được dùng để tính lượng bộ nhớ
[node allocatable](253-reserve-compute-resources-vi.md#node-allocatable)
thực tế khả dụng cho các Pod.

Kubernetes scheduler sử dụng thông tin về bộ nhớ allocatable để tối ưu việc
[lập lịch](136-scheduling-eviction-vi.md) cho Pod.
Cơ chế _node allocatable_ thường được các quản trị viên node dùng để dành riêng tài nguyên hệ
thống của node K8s cho kubelet hoặc các tiến trình của hệ điều hành, nhằm giúp đảm bảo độ ổn định
của node.

Các thiết lập kubelet liên quan bao gồm `kubeReserved`, `systemReserved` và `reservedMemory`.
Thiết lập `reservedMemory` cho phép bạn chia nhỏ tổng lượng bộ nhớ dành riêng và gán nó
cho nhiều NUMA node.

Bạn chỉ định một danh sách các phần bộ nhớ dành riêng, phân tách bằng dấu phẩy, thuộc các loại
bộ nhớ khác nhau, cho từng NUMA node.
Bạn cũng có thể chỉ định các phần dành riêng trải rộng trên nhiều NUMA node, dùng dấu chấm phẩy
làm ký tự phân tách.

Memory Manager sẽ không dùng phần bộ nhớ dành riêng này để chạy các workload container.

Ví dụ, nếu bạn có một NUMA node "NUMA0" với 10GiB bộ nhớ khả dụng, và bạn cấu hình
`reservedMemory` để dành riêng `1Gi` (bộ nhớ) cho NUMA0, Memory Manager sẽ coi rằng chỉ còn
9GiB khả dụng cho các Pod.

Bạn có thể bỏ qua tham số này, tuy nhiên bạn cần lưu ý rằng tổng lượng bộ nhớ dành riêng từ tất
cả các NUMA node phải bằng lượng bộ nhớ theo cơ chế _node allocatable_.

Nếu có ít nhất một tham số node allocatable khác không, bạn sẽ cần chỉ định `reservedMemory`
cho ít nhất một NUMA node.
Trên thực tế, giá trị ngưỡng `evictionHard` mặc định bằng `100Mi`, vì vậy nếu bạn dùng chính sách
`Static`, việc chỉ định `reservedMemory` là bắt buộc.

### Cú pháp bộ nhớ dành riêng của Memory Manager (Memory manager reserved memory syntax) {#reserved-memory-syntax}

Dưới đây là một số ví dụ về cách đặt cấu hình `reservedMemory` cho kubelet.

```yaml
  # Ví dụ 1
  reservedMemory:
  - numaNode: 0 # chỉ số NUMA node
    limits:
      memory: "1Gi" # lượng byte
  - numaNode: 1
    limits:
      memory: "2Gi" # lượng byte
```

```yaml
  # Ví dụ 2
  reservedMemory:
  - numaNode: 0
    limits:
      "memory": "512Gi"
  - numaNode: 1
    limits:
      "memory": "512Gi"
      "hugepages-1Gi": "2Gi" # chỉ có ý nghĩa trên Linux
```

### Ràng buộc về việc dành riêng bộ nhớ NUMA (Constraints on NUMA memory reservation)

Khi bạn chỉ định các giá trị cho `reservedMemory`, chúng phải tương thích với các giá trị
`kubeReserved` và `systemReserved` đang có hiệu lực, cùng với bất kỳ thiết lập `memory.available`
nào mà bạn đặt trong `evictionHard`.

```math
\begin{equation*}
\sum_{ \textnormal{i} = 0}^{ \textnormal{node count}} { \textit{reservedMemory} [ \textnormal{i} ]} = \textit{kubeReserved} + \textit{systemReserved} + \textit{evictionHard} \, \boxed{\textnormal{memory.available}}
\end{equation*}\\\
\text{trong đó i là chỉ số của một NUMA node}
```

Nếu bạn không tuân theo công thức trên, Memory Manager sẽ báo lỗi khi khởi động.

Nói cách khác, ví dụ 1 (ở trên) minh họa rằng với bộ nhớ thông thường (`type=memory`),
Kubernetes dành riêng tổng cộng 3GiB; nghĩa là:

```math
\begin{equation*}
\sum_{ \textnormal{i} = 0}^{ \textnormal{node count}} \textit{reservedMemory}_{ [ \textnormal{i} ] }  =  \underbrace{\textit{reservedMemory} [ 0 ] + \textit{reservedMemory} [ 1 ] }_{\textnormal{type=memory}}
            = 1 \textnormal{GiB} + 2 \textnormal{GiB}
            = 3 \textnormal{GiB}
\end{equation*}\\\
\text{trong đó i là chỉ số của một NUMA node}
```

Một số ví dụ về các thiết lập cấu hình kubelet liên quan tới cấu hình node allocatable:

```yaml
  kubeReserved: { cpu: "500m", memory: "50Mi" } # nửa CPU, 50MiB bộ nhớ
  systemReserved: { cpu: "500m", memory: "256Mi" } # nửa CPU, 256MiB bộ nhớ
```

> **Ghi chú:**
>
> Ngưỡng hard eviction mặc định là 100MiB, chứ **không** phải là 0.
> Hãy nhớ tăng lượng bộ nhớ mà bạn dành riêng qua `reservedMemory` thêm đúng bằng ngưỡng
> hard eviction đó. Nếu không, kubelet sẽ không khởi động Memory Manager và sẽ hiển thị lỗi.
>
> Dưới đây là một ví dụ về cấu hình đúng khi sử dụng `reservedMemory`:
>
> ```yaml
>   # đoạn cấu hình này dựa trên giá trị mặc định của evictionHard
>   memoryManagerPolicy: Static
>   kubeReserved: { cpu: "4", memory: "4Gi" }
>   systemReserved: { cpu: "1", memory: "1Gi" }
>   reservedMemory:
>   - numaNode: 0
>     limits:
>       memory: "3Gi"
>   - numaNode: 1
>     limits:
>       memory: "2148Mi" # 3GiB trừ 100MiB
> ```

### Các cấu hình cần tránh (Configurations to avoid) {#reserved-memory-configurations-to-avoid}

Hãy tránh các cấu hình sau:

1. trùng lặp: cùng một NUMA node hoặc cùng loại bộ nhớ, nhưng với giá trị khác nhau;
1. đặt giới hạn (limit) bằng 0 cho bất kỳ loại bộ nhớ nào;
1. các ID NUMA node không tồn tại trong phần cứng của máy;
1. tên loại bộ nhớ khác với `memory` hoặc `hugepages-<size>`
   (hugepages với `<size>` cụ thể cũng phải tồn tại).

## Đưa một Pod vào lớp QoS Guaranteed (Placing a Pod in the Guaranteed QoS class) {#placing-a-pod-in-the-guaranteed-qos-class}

Nếu chính sách được chọn là bất kỳ chính sách nào khác `None`, Memory Manager sẽ nhận diện các
Pod thuộc lớp QoS `Guaranteed`.
Memory Manager cung cấp các gợi ý topology cụ thể cho Topology Manager đối với mỗi Pod `Guaranteed`.
Đối với các Pod thuộc lớp QoS khác `Guaranteed`, Memory Manager cung cấp các gợi ý topology mặc
định cho Topology Manager.

Các đoạn trích từ manifest Pod dưới đây sẽ đưa Pod vào lớp QoS `Guaranteed`.

Một Pod với số CPU nguyên chạy trong lớp QoS `Guaranteed` khi `requests` bằng `limits`:

```yaml
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "2"
        example.com/device: "1"
      requests:
        memory: "200Mi"
        cpu: "2"
        example.com/device: "1"
```

Tương tự, một Pod dùng chung CPU (sharing CPU) cũng chạy trong lớp QoS `Guaranteed` khi
`requests` bằng `limits`.

```yaml
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "300m"
        example.com/device: "1"
      requests:
        memory: "200Mi"
        cpu: "300m"
        example.com/device: "1"
```

Lưu ý rằng cả yêu cầu về CPU lẫn bộ nhớ đều phải được chỉ định thì Pod mới thuộc lớp QoS Guaranteed.

## Tiếp theo (What's next)

- Đọc [Xử lý sự cố quản lý Topology](313-debug-topology-vi.md)
- Đọc [KEP](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/1769-memory-manager)
  (Kubernetes enhancement proposal — đề xuất cải tiến Kubernetes) về memory manager
- Đọc về [các trình quản lý tài nguyên cấp Pod](74-resource-managers-vi.md#pod-level-resource-managers).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 25:

1. Memory Manager **không** tự quyết định một Pod được nhận hay bị từ chối. Vậy nó làm gì, ai ra
   quyết định cuối cùng, và hai manager nào phải được bật sẵn thì bộ nhớ mới căn chỉnh được với
   các tài nguyên khác trong spec của Pod?
2. **Câu bẫy.** Bạn đặt `memoryManagerPolicy: Static` nhưng để trống `reservedMemory`, với lý do
   "chưa muốn dành riêng gì cả thì cứ để 0". Kết quả là gì, và con số `100Mi` dính líu thế nào?
3. Hai đoạn manifest ở mục *Đưa một Pod vào lớp QoS Guaranteed*: một Pod xin `cpu: "2"`, một Pod
   xin `cpu: "300m"`, cả hai đều có `memory: "200Mi"`. Cả hai có cùng thuộc lớp `Guaranteed`
   không? Điều kiện thật sự để vào lớp đó là gì?
4. Trên `lab-k8s-worker2`, `ls -d /sys/devices/system/node/node[0-9]*` trả về đúng **một** dòng,
   còn `configz` không thấy khóa `memoryManagerPolicy` nào được khai. Từ hai dữ kiện đó: chính
   sách nào đang có hiệu lực, và vì sao bật `Static` trên chính máy này cũng **không** chứng minh
   được điều bài nói?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Memory Manager là một **hint provider**: nó dùng giao thức sinh gợi ý để đưa ra NUMA affinity
   phù hợp nhất cho Pod rồi **giao gợi ý đó cho Topology Manager**. Quyết định cuối cùng thuộc về
   **Topology Manager**, dựa trên cả gợi ý lẫn chính sách của chính nó — Pod bị từ chối hay được
   chấp nhận vào node là ở bước đó. Hai manager phải bật trước: **CPU Manager** và **Topology
   Manager**, mỗi cái với chính sách phù hợp trên node.
2. **kubelet sẽ không khởi động Memory Manager và báo lỗi.** Lý do: `reservedMemory` phải bằng
   `kubeReserved` + `systemReserved` + `evictionHard` `memory.available`, mà ngưỡng hard eviction
   **mặc định đã là `100Mi`, không phải 0**. Nên "để trống cho bằng 0" phá vỡ công thức ngay từ
   đầu. Với chính sách `Static`, việc chỉ định `reservedMemory` là **bắt buộc**, và khi tính bạn
   phải cộng thêm đúng phần ngưỡng hard eviction đó — như ví dụ trong bài đặt `2148Mi`, tức 3GiB
   trừ 100MiB.
3. **Có, cả hai đều là `Guaranteed`.** Điều kiện không nằm ở chỗ CPU nguyên hay lẻ, mà ở chỗ
   **`requests` bằng `limits`** và **cả CPU lẫn memory đều được chỉ định**. Bài nêu hẳn hai ví
   dụ — một Pod CPU nguyên và một Pod dùng chung CPU (`300m`) — để chặn đúng cái nhầm này. (Lưu ý
   đây là điều kiện vào lớp QoS `Guaranteed`; điều kiện được cấp **CPU độc quyền** dưới chính sách
   `static` của CPU Manager lại là chuyện của bài [200](200-cpu-management-policies-vi.md).)
4. Không khai tức là đang chạy giá trị mặc định, tức chính sách **`None`** — hành xử đúng như thể
   Memory Manager không tồn tại và chỉ trả về gợi ý topology mặc định. Bật `Static` cũng vô nghĩa
   trên máy này vì **máy chỉ có một NUMA domain**: đảm bảo cốt lõi của Memory Manager là cấp bộ
   nhớ từ **số NUMA node tối thiểu** và ghim Pod vào một tập NUMA node, mà với đúng một node thì
   mọi cách cấp phát đều thỏa mãn — không có ranh giới nào để căn, không có gì để đo. Ngoài ra
   đổi `memoryManagerPolicy` là **sửa `KubeletConfiguration` của node đang chạy**, việc thuộc
   [giai đoạn 20](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy), bài
   [224](224-kubelet-config-file-vi.md) — đừng làm chỉ để thử.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

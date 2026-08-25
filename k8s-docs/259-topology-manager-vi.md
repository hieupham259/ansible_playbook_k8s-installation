# Kiểm soát các chính sách quản lý topology trên một node (Control Topology Management Policies on a node)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/topology-manager/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 25 — Quản trị tài nguyên theo namespace](00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace),
bài 6/7 · Phần II không có lab riêng: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md), tự chấm bằng **Checkpoint giai đoạn 25**.

Giống bài [235](235-memory-manager-vi.md) ngay trước nó, đây là chính sách **cấp node** chứ không
phải chính sách theo namespace: đổi nó là sửa cấu hình kubelet của một node đang chạy, việc thuộc
[giai đoạn 20](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy), bài
[224](224-kubelet-config-file-vi.md). Bài này là mảnh cuối của bộ ba CPU Manager
([200](200-cpu-management-policies-vi.md)) — Memory Manager ([235](235-memory-manager-vi.md)) —
Topology Manager; nền lý thuyết chung ở bài [74](74-resource-managers-vi.md) (nhóm 7b).

Nói thẳng về giới hạn của cluster lab: ba VM đều chỉ có **một NUMA domain** và chính sách đang là
`none`. Trên cluster này bạn **đo được** hai thứ: đếm NUMA node bằng
`ls -d /sys/devices/system/node/node[0-9]*` và đọc chính sách **hiệu lực** qua `configz` (Lab 14
đọc nó dưới tên `topologyManagerPolicy`), đúng như
[Lab 14 phần B10.4](labs/LAB-14-CRD-VA-OPERATOR.md#b104-topology-manager-đang-ở-policy-nào) đã
làm. Bạn **không dựng được** tình huống Pod bị từ chối vì topology: với một NUMA domain thì mọi
cách sắp đặt đều thỏa mãn, kể cả `single-numa-node`. Đọc bài để biết cấu hình đúng khi vận hành
máy chủ nhiều socket, đừng sửa kubelet của lab để "thử cho biết".

**Phải hiểu ở lần đọc này:**

- Topology Manager là thành phần của kubelet đóng vai trò **nguồn thông tin chuẩn** điều phối các
  *Hint Provider* (CPU Manager, Device Manager, Memory Manager). Nó nhận gợi ý dạng **bitmask**
  các NUMA node kèm cờ `preferred`, hội tụ về một gợi ý duy nhất, rồi Pod được chấp nhận hay bị
  từ chối dựa trên gợi ý đó. Nó căn chỉnh Pod thuộc **mọi lớp QoS** (mục *Topology Manager hoạt
  động như thế nào* và *Scope và chính sách của Topology Manager*).
- Có **hai trục cấu hình độc lập**. `scope` — `container` (mặc định) hoặc `pod` — đặt bằng
  `topologyManagerScope` trong [file cấu hình kubelet](224-kubelet-config-file-vi.md); `policy`
  đặt bằng flag kubelet `--topology-manager-policy`. Scope `container` căn chỉnh **từng container
  riêng lẻ**, không có khái niệm gom nhóm; scope `pod` coi Pod là một thể thống nhất và dồn mọi
  container vào **một tập NUMA node chung** (mục *Các scope của Topology Manager*).
- Bốn chính sách khác nhau ở chỗ **làm gì khi affinity không được ưu tiên**: `none` (mặc định) —
  không căn chỉnh gì; `best-effort` — vẫn **chấp nhận** Pod vào node; `restricted` — **từ chối**;
  `single-numa-node` — từ chối nếu không dồn được vào **một** NUMA node duy nhất (bốn mục chính
  sách).
- Hậu quả vận hành khi Pod bị từ chối: Pod rơi vào trạng thái `Terminated` với lỗi pod admission
  failure, và **scheduler sẽ không lập lịch lại nó**. Phải dùng Deployment/ReplicaSet hoặc một
  control loop bên ngoài để triển khai lại (mục *Chính sách `restricted`* và *Chính sách
  `single-numa-node`*).
- Hai hạn chế đã biết: số NUMA node tối đa là **8** (vượt qua thì bùng nổ trạng thái khi liệt kê
  tổ hợp NUMA affinity), và **scheduler không nhận biết topology** — nên Pod hoàn toàn có thể
  được lập lịch lên một node rồi mới thất bại tại chính node đó (mục *Các hạn chế đã biết*).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Trước khi bạn bắt đầu* — minikube và ba playground trực tuyến | cluster lab đã có sẵn ba VM; lộ trình không dùng minikube hay cluster dùng chung | [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |
| Mục *Hỗ trợ Windows* | ba node lab đều là Linux; tính năng còn alpha, cần feature gate và container runtime hỗ trợ | [Giai đoạn 15 — Windows](00-ALO-TRINH-ADMIN.md#giai-đoạn-15--windows-nếu-môi-trường-có-node-windows), chỉ mở khi môi trường thật có node Windows |
| Mục *Các tùy chọn chính sách* — `prefer-closest-numa-nodes`, `max-allowable-numa-nodes` và ba feature gate của chúng | chỉ có ý nghĩa trên máy nhiều NUMA node; và bài còn khuyến cáo **không** dùng `max-allowable-numa-nodes` | khi vận hành máy chủ vật lý nhiều socket; cách bật/tắt feature gate ở bài [196](196-configure-feature-gates-vi.md), [giai đoạn 20](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) — đã đọc |
| Mục *Tương tác của Pod với các chính sách* — năm manifest mẫu và gợi ý mà từng Hint Provider trả về, kèm `example.com/device` | cần CPU Manager chạy `static` và một device plugin thật; cluster lab không có cả hai | bài [200](200-cpu-management-policies-vi.md) — bài 4/7 vừa đọc — cho phần CPU Manager; phần thiết bị ở bài [184 — Device Plugin](184-device-plugins-vi.md) ([giai đoạn 14](00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng)) — đã đọc |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.27 [stable]`

Ngày càng có nhiều hệ thống tận dụng sự kết hợp giữa CPU và các bộ tăng tốc phần cứng (hardware
accelerators) để hỗ trợ việc thực thi yêu cầu độ trễ thấp và tính toán song song thông lượng cao.
Chúng bao gồm các workload trong những lĩnh vực như viễn thông, tính toán khoa học, học máy
(machine learning), dịch vụ tài chính và phân tích dữ liệu. Những hệ thống lai (hybrid) như vậy
tạo nên một môi trường hiệu năng cao.

Để khai thác hiệu năng tốt nhất, cần có các tối ưu hóa liên quan đến cách ly CPU (CPU isolation),
tính cục bộ của bộ nhớ và thiết bị (memory and device locality). Tuy nhiên, trong Kubernetes,
các tối ưu hóa này lại được xử lý bởi một tập các thành phần rời rạc nhau.

_Topology Manager_ (trình quản lý topology) là một thành phần của kubelet, nhằm mục đích điều phối
tập các thành phần chịu trách nhiệm cho những tối ưu hóa này.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Phiên bản Kubernetes server của bạn phải bằng hoặc mới hơn v1.18. Để kiểm tra phiên bản, nhập
`kubectl version`.

## Topology Manager hoạt động như thế nào (How topology manager works)

Trước khi Topology Manager ra đời, CPU Manager và Device Manager trong Kubernetes đưa ra các
quyết định cấp phát tài nguyên độc lập với nhau. Điều này có thể dẫn đến những lần cấp phát
không mong muốn trên các hệ thống nhiều socket, và các ứng dụng nhạy cảm với hiệu năng/độ trễ
sẽ chịu thiệt hại do những lần cấp phát không mong muốn này. "Không mong muốn" ở đây nghĩa là,
ví dụ, CPU và thiết bị được cấp phát từ những NUMA Node khác nhau, do đó phát sinh thêm độ trễ.

Topology Manager là một thành phần của kubelet, đóng vai trò như một nguồn thông tin chuẩn
(source of truth) để các thành phần khác của kubelet có thể đưa ra các lựa chọn cấp phát tài nguyên
được căn chỉnh theo topology (topology aligned).

Topology Manager cung cấp một giao diện cho các thành phần, được gọi là *Hint Providers*
(bên cung cấp gợi ý), để gửi và nhận thông tin topology. Topology Manager có một tập các chính sách
cấp node (node level policies) được giải thích bên dưới.

Topology Manager nhận thông tin topology từ các *Hint Providers* dưới dạng một bitmask biểu thị
các NUMA Node khả dụng cùng với chỉ báo cấp phát ưu tiên (preferred allocation). Các chính sách
của Topology Manager thực hiện một tập các thao tác trên các gợi ý (hint) được cung cấp và hội tụ
về gợi ý do chính sách quyết định để cho ra kết quả tối ưu. Nếu một gợi ý không mong muốn được
lưu lại, trường preferred của gợi ý đó sẽ được đặt thành false. Trong các chính sách hiện tại,
preferred là mask ưu tiên hẹp nhất.
Gợi ý được chọn sẽ được lưu như một phần của Topology Manager. Tùy vào chính sách được cấu hình,
Pod có thể được chấp nhận hoặc bị từ chối khỏi node dựa trên gợi ý đã chọn.
Gợi ý sau đó được lưu trong Topology Manager để các *Hint Providers* sử dụng khi đưa ra
các quyết định cấp phát tài nguyên.

Luồng xử lý có thể được thấy trong sơ đồ sau.

![topology_manager_flow](https://kubernetes.io/images/docs/topology-manager-flow.png)

## Hỗ trợ Windows (Windows Support)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.32 [alpha]`

Hỗ trợ Topology Manager có thể được bật trên Windows bằng cách dùng feature gate
`WindowsCPUAndMemoryAffinity`, và nó cần sự hỗ trợ từ phía container runtime.

## Scope và chính sách của Topology Manager (Topology manager scopes and policies)

Topology Manager hiện tại:

- căn chỉnh các Pod thuộc mọi lớp QoS (QoS class).
- căn chỉnh các tài nguyên được yêu cầu mà Hint Provider có cung cấp gợi ý topology.

Nếu các điều kiện này được thỏa mãn, Topology Manager sẽ căn chỉnh các tài nguyên được yêu cầu.

Để tùy biến cách việc căn chỉnh này được thực hiện, Topology Manager cung cấp hai tùy chọn
riêng biệt: `scope` (phạm vi) và `policy` (chính sách).

`scope` xác định mức độ chi tiết mà bạn muốn việc căn chỉnh tài nguyên được thực hiện,
ví dụ ở cấp `pod` hay cấp `container`. Còn `policy` xác định chính sách thực tế được dùng để
thực hiện việc căn chỉnh, ví dụ `best-effort`, `restricted` và `single-numa-node`.
Chi tiết về các `scope` và `policy` hiện có được trình bày bên dưới.

> **Ghi chú:**
>
> Để căn chỉnh tài nguyên CPU với các tài nguyên được yêu cầu khác trong spec của Pod,
> CPU Manager cần được bật và chính sách CPU Manager phù hợp cần được cấu hình trên Node.
> Xem [Kiểm soát các chính sách quản lý CPU trên Node](200-cpu-management-policies-vi.md).

> **Ghi chú:**
>
> Để căn chỉnh tài nguyên bộ nhớ (và hugepages) với các tài nguyên được yêu cầu khác trong spec
> của Pod, Memory Manager cần được bật và chính sách Memory Manager phù hợp cần được cấu hình
> trên Node. Tham khảo tài liệu [Memory Manager](235-memory-manager-vi.md).

## Các scope của Topology Manager (Topology manager scopes)

Topology Manager có thể xử lý việc căn chỉnh tài nguyên theo một vài scope riêng biệt:

* `container` (mặc định)
* `pod`

Một trong hai tùy chọn có thể được chọn tại thời điểm khởi động kubelet, bằng cách thiết lập
`topologyManagerScope` trong
[file cấu hình kubelet](224-kubelet-config-file-vi.md).

### Scope `container`

Scope `container` được dùng theo mặc định. Bạn cũng có thể thiết lập tường minh
`topologyManagerScope` thành `container` trong
[file cấu hình kubelet](224-kubelet-config-file-vi.md).

Trong scope này, Topology Manager thực hiện một chuỗi các lần căn chỉnh tài nguyên tuần tự,
tức là với mỗi container (trong một Pod), một phép căn chỉnh riêng được tính toán. Nói cách khác,
với scope cụ thể này, không có khái niệm gom nhóm các container vào một tập NUMA node nhất định.
Trên thực tế, Topology Manager thực hiện việc căn chỉnh tùy ý từng container riêng lẻ vào
các NUMA node.

Khái niệm gom nhóm các container đã được ủng hộ và triển khai một cách có chủ đích trong
scope tiếp theo, đó là scope `pod`.

### Scope `pod`

Để chọn scope `pod`, thiết lập `topologyManagerScope` trong
[file cấu hình kubelet](224-kubelet-config-file-vi.md) thành `pod`.

Scope này cho phép gom toàn bộ các container trong một Pod vào một tập NUMA node chung. Nghĩa là,
Topology Manager coi Pod như một thể thống nhất và cố gắng cấp phát toàn bộ Pod (tất cả các
container) vào hoặc một NUMA node duy nhất, hoặc một tập NUMA node chung. Các ví dụ sau minh họa
những phép căn chỉnh mà Topology Manager thực hiện trong các trường hợp khác nhau:

* tất cả các container có thể được và đã được cấp phát vào một NUMA node duy nhất;
* tất cả các container có thể được và đã được cấp phát vào một tập NUMA node dùng chung.

Tổng lượng của một tài nguyên cụ thể mà toàn bộ Pod yêu cầu được tính theo công thức
[requests/limits hiệu dụng](50-init-containers-vi.md#resource-sharing-within-containers),
và do đó, giá trị tổng này bằng giá trị lớn nhất giữa:

* tổng các request của tất cả app container,
* request lớn nhất trong các init container,

cho một tài nguyên.

Việc dùng scope `pod` kết hợp với chính sách Topology Manager `single-numa-node` đặc biệt
có giá trị đối với các workload nhạy cảm với độ trễ hoặc các ứng dụng thông lượng cao thực hiện
IPC (giao tiếp liên tiến trình). Bằng cách kết hợp cả hai tùy chọn, bạn có thể đặt tất cả các
container trong một Pod vào một NUMA node duy nhất; nhờ đó, chi phí giao tiếp liên NUMA có thể
được loại bỏ đối với Pod đó.

Trong trường hợp chính sách `single-numa-node`, một Pod chỉ được chấp nhận nếu tồn tại một tập
NUMA node phù hợp trong số các phương án cấp phát khả dĩ. Xem xét lại ví dụ ở trên:

* một tập chỉ chứa một NUMA node duy nhất - dẫn đến Pod được chấp nhận (admitted),
* trong khi một tập chứa nhiều NUMA node hơn - dẫn đến Pod bị từ chối (vì thay vì một
  NUMA node, cần đến hai hoặc nhiều NUMA node để thỏa mãn việc cấp phát).

Tóm lại, Topology Manager trước tiên tính toán một tập NUMA node rồi kiểm tra tập đó với
chính sách của Topology Manager, việc này dẫn đến Pod bị từ chối hoặc được chấp nhận.

## Các chính sách của Topology Manager (Topology manager policies)

Topology Manager hỗ trợ bốn chính sách cấp phát. Bạn có thể thiết lập một chính sách qua flag
của kubelet, `--topology-manager-policy`. Có bốn chính sách được hỗ trợ:

* `none` (mặc định)
* `best-effort`
* `restricted`
* `single-numa-node`

> **Ghi chú:**
>
> Nếu Topology Manager được cấu hình với scope **pod**, container mà chính sách xem xét
> sẽ phản ánh yêu cầu của toàn bộ Pod, và do đó mỗi container trong Pod
> sẽ nhận được quyết định căn chỉnh topology **giống nhau**.

### Chính sách `none` {#policy-none}

Đây là chính sách mặc định và không thực hiện bất kỳ phép căn chỉnh topology nào.

### Chính sách `best-effort` {#policy-best-effort}

Với mỗi container trong một Pod, kubelet với chính sách quản lý topology `best-effort` sẽ gọi
từng Hint Provider để tìm hiểu mức khả dụng tài nguyên của chúng. Dùng thông tin này, Topology
Manager lưu lại NUMA Node affinity ưu tiên cho container đó. Nếu affinity không được ưu tiên,
Topology Manager vẫn sẽ lưu lại thông tin này và vẫn chấp nhận Pod vào node.

Các *Hint Providers* sau đó có thể dùng thông tin này khi đưa ra
quyết định cấp phát tài nguyên.

### Chính sách `restricted` {#policy-restricted}

Với mỗi container trong một Pod, kubelet với chính sách quản lý topology `restricted` sẽ gọi từng
Hint Provider để tìm hiểu mức khả dụng tài nguyên của chúng. Dùng thông tin này, Topology
Manager lưu lại NUMA Node affinity ưu tiên cho container đó. Nếu affinity không được ưu tiên,
Topology Manager sẽ từ chối Pod này khỏi node. Điều này dẫn đến Pod rơi vào trạng thái
`Terminated` với lỗi từ chối chấp nhận Pod (pod admission failure).

Một khi Pod đã ở trạng thái `Terminated`, Kubernetes scheduler sẽ **không** cố gắng
lập lịch lại Pod đó. Bạn nên dùng một ReplicaSet hoặc Deployment để kích hoạt việc triển khai lại
Pod. Cũng có thể triển khai một vòng lặp điều khiển (control loop) bên ngoài để kích hoạt việc
triển khai lại các Pod gặp lỗi `Topology Affinity`.

Nếu Pod được chấp nhận, các *Hint Providers* sau đó có thể dùng thông tin này khi đưa ra
quyết định cấp phát tài nguyên.

### Chính sách `single-numa-node` {#policy-single-numa-node}

Với mỗi container trong một Pod, kubelet với chính sách quản lý topology `single-numa-node`
sẽ gọi từng Hint Provider để tìm hiểu mức khả dụng tài nguyên của chúng. Dùng thông tin này,
Topology Manager xác định liệu affinity trên một NUMA Node duy nhất có khả thi hay không. Nếu có,
Topology Manager sẽ lưu lại điều này và các *Hint Providers* sau đó có thể dùng thông tin này
khi đưa ra quyết định cấp phát tài nguyên. Ngược lại, nếu điều này không khả thi, Topology Manager
sẽ từ chối Pod khỏi node. Điều này dẫn đến Pod ở trạng thái `Terminated` với lỗi từ chối
chấp nhận Pod.

Một khi Pod đã ở trạng thái `Terminated`, Kubernetes scheduler sẽ **không** cố gắng
lập lịch lại Pod đó. Bạn nên dùng một Deployment có replicas để kích hoạt việc triển khai lại
Pod. Cũng có thể triển khai một vòng lặp điều khiển bên ngoài để kích hoạt việc triển khai lại
các Pod gặp lỗi `Topology Affinity`.

## Các tùy chọn chính sách của Topology Manager (Topology manager policy options)

Việc hỗ trợ các tùy chọn chính sách của Topology Manager yêu cầu
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`TopologyManagerPolicyOptions` được bật (nó được bật theo mặc định).

Bạn có thể bật hoặc tắt các nhóm tùy chọn theo mức độ trưởng thành của chúng bằng các feature gate
sau:

* `TopologyManagerPolicyBetaOptions` mặc định được bật. Bật để hiện các tùy chọn ở mức beta.
* `TopologyManagerPolicyAlphaOptions` mặc định bị tắt. Bật để hiện các tùy chọn ở mức alpha.

Bạn vẫn phải bật từng tùy chọn bằng tùy chọn `TopologyManagerPolicyOptions` của kubelet.

### `prefer-closest-numa-nodes` {#policy-option-prefer-closest-numa-nodes}

Tùy chọn `prefer-closest-numa-nodes` đạt mức GA từ Kubernetes 1.32. Trong Kubernetes v1.36,
tùy chọn chính sách này được hiển thị theo mặc định với điều kiện
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`TopologyManagerPolicyOptions` được bật.

Theo mặc định, Topology Manager không nhận biết được khoảng cách giữa các NUMA node (NUMA
distances), và không tính đến chúng khi đưa ra các quyết định chấp nhận Pod. Hạn chế này xuất hiện
trên các hệ thống nhiều socket, cũng như trên các hệ thống một socket nhiều NUMA, và có thể gây
suy giảm hiệu năng đáng kể đối với việc thực thi yêu cầu độ trễ thấp và các ứng dụng thông lượng
cao nếu Topology Manager quyết định căn chỉnh tài nguyên trên các NUMA node không kề nhau.

Nếu bạn chỉ định tùy chọn chính sách `prefer-closest-numa-nodes`, các chính sách `best-effort` và
`restricted` sẽ ưu tiên các tập NUMA node có khoảng cách giữa chúng ngắn hơn khi đưa ra quyết định
chấp nhận Pod.

Bạn có thể bật tùy chọn này bằng cách thêm `prefer-closest-numa-nodes=true` vào các tùy chọn
chính sách của Topology Manager.

Theo mặc định (khi không có tùy chọn này), Topology Manager căn chỉnh tài nguyên hoặc trên một
NUMA node duy nhất, hoặc trong trường hợp cần nhiều hơn một NUMA node, dùng số lượng NUMA node
tối thiểu.

### `max-allowable-numa-nodes` {#policy-option-max-allowable-numa-nodes}

Tùy chọn `max-allowable-numa-nodes` đạt mức GA từ Kubernetes 1.35. Trong Kubernetes v1.36,
tùy chọn chính sách này được hiển thị theo mặc định với điều kiện
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`TopologyManagerPolicyOptions` được bật.

Thời gian để chấp nhận một Pod gắn liền với số lượng NUMA node trên máy vật lý.
Theo mặc định, Kubernetes không chạy kubelet với Topology Manager được bật trên bất kỳ node
(Kubernetes) nào phát hiện thấy nhiều hơn 8 NUMA node.

> **Ghi chú:**
>
> Nếu bạn chọn tùy chọn chính sách `max-allowable-numa-nodes`, các node có nhiều hơn 8 NUMA node
> có thể được phép chạy với Topology Manager được bật. Dự án Kubernetes chỉ có dữ liệu hạn chế về
> tác động của việc dùng Topology Manager trên các node (Kubernetes) có nhiều hơn 8 NUMA node.
> Do thiếu dữ liệu này, việc dùng tùy chọn chính sách này với Kubernetes v1.36 là **không**
> được khuyến nghị và rủi ro do bạn tự chịu.

Bạn có thể bật tùy chọn này bằng cách thêm `max-allowable-numa-nodes=<integer>` vào các tùy chọn
chính sách của Topology Manager, trong đó giá trị nguyên phải lớn hơn 8. Giá trị mặc định là 8,
giữ nguyên giới hạn hiện có.

Việc đặt một giá trị cho `max-allowable-numa-nodes` tự nó không ảnh hưởng đến độ trễ của việc
chấp nhận Pod, nhưng việc gán một Pod vào một node (Kubernetes) có nhiều NUMA thì có ảnh hưởng.
Những cải tiến tiềm năng trong tương lai của Kubernetes có thể cải thiện hiệu năng chấp nhận Pod
và độ trễ cao xảy ra khi số lượng NUMA node tăng lên.

## Tương tác của Pod với các chính sách của Topology Manager (Pod interactions with topology manager policies)

Xem xét các container trong manifest Pod sau:

```yaml
spec:
  containers:
  - name: nginx
    image: nginx
```

Pod này chạy trong lớp QoS `BestEffort` vì không có `requests` hay `limits` tài nguyên nào
được chỉ định.

```yaml
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      limits:
        memory: "200Mi"
      requests:
        memory: "100Mi"
```

Pod này chạy trong lớp QoS `Burstable` vì requests nhỏ hơn limits.

Nếu chính sách được chọn là bất kỳ chính sách nào khác `none`, Topology Manager sẽ xem xét các
đặc tả Pod này. Topology Manager sẽ tham vấn các Hint Providers để lấy các gợi ý topology.
Trong trường hợp chính sách CPU Manager là `static`, nó sẽ trả về gợi ý topology mặc định, vì
các Pod này không yêu cầu tài nguyên CPU một cách tường minh.

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

Pod này với request CPU là số nguyên chạy trong lớp QoS `Guaranteed` vì `requests` bằng
`limits`.

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

Pod này với request CPU chia sẻ (sharing CPU request) chạy trong lớp QoS `Guaranteed` vì
`requests` bằng `limits`.

```yaml
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      limits:
        example.com/deviceA: "1"
        example.com/deviceB: "1"
      requests:
        example.com/deviceA: "1"
        example.com/deviceB: "1"
```

Pod này chạy trong lớp QoS `BestEffort` vì không có request CPU và bộ nhớ.

Topology Manager sẽ xem xét các Pod ở trên. Topology Manager sẽ tham vấn các Hint
Providers, tức CPU Manager và Device Manager, để lấy các gợi ý topology cho các Pod.

Trong trường hợp Pod `Guaranteed` với request CPU là số nguyên, chính sách CPU Manager `static`
sẽ trả về các gợi ý topology liên quan đến CPU độc quyền (exclusive CPU), và Device Manager sẽ
gửi lại các gợi ý cho thiết bị được yêu cầu.

Trong trường hợp Pod `Guaranteed` với request CPU chia sẻ, chính sách CPU Manager `static`
sẽ trả về gợi ý topology mặc định vì không có request CPU độc quyền, và Device Manager
sẽ gửi lại các gợi ý cho thiết bị được yêu cầu.

Trong cả hai trường hợp trên của Pod `Guaranteed`, chính sách CPU Manager `none` sẽ trả về
gợi ý topology mặc định.

Trong trường hợp Pod `BestEffort`, chính sách CPU Manager `static` sẽ gửi lại gợi ý topology
mặc định vì không có request CPU, và Device Manager sẽ gửi lại các gợi ý cho từng thiết bị
được yêu cầu.

Dùng thông tin này, Topology Manager tính toán gợi ý tối ưu cho Pod và lưu lại
thông tin đó, để các Hint Providers sử dụng khi chúng thực hiện việc gán
tài nguyên.

## Các hạn chế đã biết (Known limitations)

1. Số lượng NUMA node tối đa mà Topology Manager cho phép là 8. Với nhiều hơn 8 NUMA node,
   sẽ xảy ra bùng nổ trạng thái (state explosion) khi cố gắng liệt kê các tổ hợp NUMA affinity
   khả dĩ và sinh các gợi ý cho chúng. Xem [`max-allowable-numa-nodes`](#policy-option-max-allowable-numa-nodes)
   (beta) để biết thêm các tùy chọn.

1. Scheduler không nhận biết topology, vì vậy có khả năng một Pod được lập lịch lên một node
   rồi sau đó thất bại trên node đó do Topology Manager.

## Tiếp theo (What's next)

* Đọc về [Các trình quản lý tài nguyên cấp Pod](74-resource-managers-vi.md#pod-level-resource-managers).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 25:

1. `scope` và `policy` là hai trục khác nhau. Mỗi trục đặt bằng cái gì, mặc định là giá trị nào,
   và scope `pod` khác scope `container` ở điểm cốt lõi nào?
2. Khi gợi ý affinity **không** được ưu tiên, ba chính sách `best-effort`, `restricted` và
   `single-numa-node` phản ứng khác nhau ra sao? Riêng `single-numa-node` từ chối Pod trong tình
   huống cụ thể nào?
3. **Câu bẫy.** Một Pod bị chính sách `restricted` từ chối khỏi node. Pod rơi vào trạng thái nào,
   và scheduler có tự đi tìm node khác cho nó không? Bạn phải làm gì để nó được thử lại?
4. Trên `lab-k8s-worker2`, `ls -d /sys/devices/system/node/node[0-9]*` trả về đúng **một** dòng và
   `configz` cho thấy chính sách đang là `none`. Giả sử bạn đổi sang `single-numa-node` trên đúng
   máy đó rồi chạy một Pod `Guaranteed`: Pod bị từ chối hay được chấp nhận? Vì sao kết quả đó
   **không** phải bằng chứng chính sách đang làm việc?
5. Bài nêu hai hạn chế đã biết. Vì sao Kubernetes mặc định không chạy kubelet với Topology Manager
   trên node phát hiện thấy hơn 8 NUMA node? Và hạn chế thứ hai — scheduler không nhận biết
   topology — gây ra hệ quả gì trong vận hành?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. `scope` đặt bằng trường **`topologyManagerScope` trong file cấu hình kubelet**, mặc định là
   **`container`**; `policy` đặt bằng **flag kubelet `--topology-manager-policy`**, mặc định là
   **`none`**. Khác biệt cốt lõi: với scope `container`, Topology Manager tính **một phép căn
   chỉnh riêng cho từng container** và **không có khái niệm gom nhóm** — các container của cùng
   một Pod có thể rơi vào những NUMA node khác nhau. Với scope `pod`, nó coi **Pod là một thể
   thống nhất** và dồn toàn bộ container vào một NUMA node duy nhất hoặc một tập NUMA node chung.
2. **`best-effort` vẫn lưu lại thông tin affinity và vẫn chấp nhận Pod vào node**;
   **`restricted` từ chối Pod**. `single-numa-node` không xét "ưu tiên hay không" mà xét một điều
   kiện hẹp hơn: **affinity trên một NUMA node duy nhất có khả thi không** — nếu không khả thi,
   tức phải cần từ hai NUMA node trở lên mới thỏa mãn được yêu cầu, thì Pod bị từ chối.
3. Pod rơi vào trạng thái **`Terminated`** với lỗi từ chối chấp nhận Pod (pod admission failure).
   Và đây là chỗ dễ sai: **scheduler sẽ *không* lập lịch lại Pod đó** — nó không tự đi tìm node
   khác. Trực giác "Pod hỏng thì Kubernetes tự thử node khác" sai vì việc từ chối xảy ra ở
   **kubelet**, sau khi scheduler đã xong việc. Muốn có lần thử lại thì phải để Pod nằm dưới một
   **ReplicaSet hoặc Deployment**, hoặc viết một control loop bên ngoài kích hoạt triển khai lại
   các Pod gặp lỗi `Topology Affinity`.
4. **Pod được chấp nhận** — và chính vì nó *luôn* được chấp nhận nên đây không phải bằng chứng gì
   cả. `single-numa-node` chỉ từ chối khi việc cấp phát cần **từ hai NUMA node trở lên**; máy chỉ
   có **một** NUMA domain thì trường hợp đó không bao giờ tồn tại, mọi phương án cấp phát đều nằm
   gọn trong một node. Muốn thấy chính sách thật sự chặn thì phải có máy nhiều NUMA domain. Thêm
   nữa, đổi chính sách là **sửa cấu hình kubelet của node đang chạy**, việc thuộc
   [giai đoạn 20](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy), bài
   [224](224-kubelet-config-file-vi.md) — đừng làm chỉ để thử.
5. Vì với hơn 8 NUMA node sẽ xảy ra **bùng nổ trạng thái (state explosion)** khi liệt kê mọi tổ
   hợp NUMA affinity khả dĩ và sinh gợi ý cho chúng; 8 là giới hạn mặc định, và tùy chọn
   `max-allowable-numa-nodes` mới cho vượt — bài khuyến cáo không dùng. Hạn chế thứ hai:
   **scheduler không nhận biết topology**, nên nó có thể chọn một node mà Topology Manager của
   node đó sau đó từ chối — Pod được lập lịch xong rồi mới thất bại tại node, đúng cơ chế đã mô tả
   ở câu 3.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

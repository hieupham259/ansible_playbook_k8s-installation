# Các trình quản lý tài nguyên (Resource managers)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/resource-managers/>

Để hỗ trợ các workload nhạy cảm với độ trễ (latency-critical) và có thông lượng cao
(high-throughput), Kubernetes cung cấp một bộ các Trình quản lý tài nguyên (Resource
Manager). Các trình quản lý này có mục tiêu điều phối và tối ưu việc căn chỉnh
(alignment) tài nguyên của node cho những Pod được cấu hình với yêu cầu cụ thể về
tài nguyên CPU, thiết bị (device), và bộ nhớ (hugepages).

## Trình quản lý topology (Topology manager)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.27 [stable]`

*Topology Manager* là một thành phần của kubelet có mục tiêu điều phối tập hợp các
thành phần chịu trách nhiệm cho những tối ưu hóa này. Để tìm hiểu thêm, hãy đọc
[Kiểm soát các chính sách quản lý topology trên một node](https://kubernetes.io/docs/tasks/administer-cluster/topology-manager/).

## Trình quản lý CPU (CPU manager)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.26 [stable]`

*CPU Manager* là một thành phần của kubelet cung cấp việc cấp phát tài nguyên độc quyền
(exclusive) cho tài nguyên CPU. Nó tham vấn Topology Manager để đưa ra các quyết định
gán tài nguyên. Để tìm hiểu thêm, hãy đọc
[Kiểm soát các chính sách quản lý CPU trên node](https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/).

### Các chính sách gán CPU cho Pod (Policies for assigning CPUs to Pods)

Một khi Pod đã được gán (bound) vào một Node, kubelet trên node đó có thể cần hoặc là
ghép kênh (multiplex) phần cứng hiện có (ví dụ: chia sẻ CPU giữa nhiều Pod), hoặc là cấp
phát phần cứng bằng cách dành riêng một số tài nguyên (ví dụ: gán một hoặc nhiều CPU cho
một Pod sử dụng độc quyền).

Theo mặc định, kubelet dùng [CFS quota](https://en.wikipedia.org/wiki/Completely_Fair_Scheduler)
(hạn ngạch CFS) để cưỡng chế giới hạn CPU của pod. Khi node chạy nhiều pod thiên về CPU
(CPU-bound), workload có thể bị di chuyển sang các CPU core khác nhau tùy vào việc pod
có bị hạn chế tốc độ (throttle) hay không và những CPU core nào đang khả dụng tại thời
điểm lập lịch. Nhiều workload không nhạy cảm với sự di chuyển này và do đó vẫn hoạt động
tốt mà không cần bất kỳ can thiệp nào.

Tuy nhiên, với những workload mà độ bám cache CPU (CPU cache affinity) và độ trễ lập lịch
ảnh hưởng đáng kể đến hiệu năng, kubelet cho phép dùng các chính sách quản lý CPU thay
thế để quyết định một số ưu tiên về vị trí đặt (placement) trên node.
Điều này được hiện thực bằng _CPU Manager_ và chính sách của nó.
Có hai chính sách khả dụng:

- `none`: chính sách `none` bật một cách tường minh cơ chế CPU affinity mặc định hiện
  có, không cung cấp affinity nào ngoài những gì bộ lập lịch của hệ điều hành tự động
  thực hiện. Giới hạn sử dụng CPU cho
  [các pod Guaranteed](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/) và
  [các pod Burstable](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)
  được cưỡng chế bằng CFS quota.
- `static`: chính sách `static` cho phép các container trong pod `Guaranteed` có CPU
  `requests` là số nguyên được truy cập các CPU độc quyền trên node. Tính độc quyền này
  được cưỡng chế bằng [cpuset cgroup controller](https://www.kernel.org/doc/Documentation/cgroup-v2.txt).

> **Ghi chú:**
> Các dịch vụ hệ thống như container runtime và bản thân kubelet vẫn có thể tiếp tục chạy
> trên những CPU độc quyền này. Tính độc quyền chỉ áp dụng đối với các pod khác.

CPU Manager không hỗ trợ việc đưa CPU ra khỏi hoạt động (offline) và đưa trở lại hoạt
động (online) tại thời điểm chạy (runtime).

#### Chính sách static (Static policy)

Chính sách static cho phép quản lý CPU chi tiết hơn và gán CPU độc quyền.
Chính sách này quản lý một pool CPU dùng chung (shared pool) mà ban đầu chứa toàn bộ CPU
của node. Số lượng CPU có thể cấp phát độc quyền bằng tổng số CPU trên node trừ đi phần
CPU dự trữ (reservation) được đặt trong cấu hình kubelet. Các CPU được dự trữ bởi những
tùy chọn này được lấy ra, theo số lượng nguyên, từ pool dùng chung ban đầu theo thứ tự
tăng dần của ID physical core. Pool dùng chung này là tập các CPU mà mọi container trong
các pod `BestEffort` và `Burstable` chạy trên đó. Các container trong pod `Guaranteed`
có CPU `requests` là số lẻ (fractional) cũng chạy trên các CPU trong pool dùng chung.
Chỉ những container thuộc pod `Guaranteed` và có CPU `requests` là số nguyên mới được
gán CPU độc quyền.

> **Ghi chú:**
> Kubelet yêu cầu mức dự trữ CPU lớn hơn 0 khi chính sách static được bật.
> Lý do là mức dự trữ CPU bằng 0 sẽ cho phép pool dùng chung trở nên rỗng.

Khi các pod `Guaranteed` có container thỏa mãn các yêu cầu để được gán tĩnh (statically
assigned) được lập lịch lên node, các CPU sẽ bị lấy ra khỏi pool dùng chung và được đặt
vào cpuset của container đó. CFS quota không được dùng để giới hạn mức sử dụng CPU của
những container này vì mức sử dụng của chúng đã bị giới hạn bởi chính miền lập lịch
(scheduling domain). Nói cách khác, số CPU trong cpuset của container bằng đúng giá trị
CPU `limit` nguyên được chỉ định trong spec của pod. Việc gán tĩnh này làm tăng CPU
affinity và giảm số lần chuyển ngữ cảnh (context switch) do throttle đối với workload
thiên về CPU.

Hãy xem xét các container trong những pod spec sau:

```yaml
spec:
  containers:
  - name: nginx
    image: nginx
```

Pod ở trên chạy trong lớp QoS `BestEffort` vì không có `requests` hay `limits` tài
nguyên nào được chỉ định. Nó chạy trong pool dùng chung.

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

Pod ở trên chạy trong lớp QoS `Burstable` vì `requests` tài nguyên không bằng `limits`
và giá trị `cpu` không được chỉ định. Nó chạy trong pool dùng chung.

```yaml
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "2"
      requests:
        memory: "100Mi"
        cpu: "1"
```

Pod ở trên chạy trong lớp QoS `Burstable` vì `requests` tài nguyên không bằng `limits`.
Nó chạy trong pool dùng chung.

```yaml
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "2"
      requests:
        memory: "200Mi"
        cpu: "2"
```

Pod ở trên chạy trong lớp QoS `Guaranteed` vì `requests` bằng `limits`.
Và giới hạn tài nguyên CPU của container là một số nguyên lớn hơn hoặc bằng một.
Container `nginx` được cấp 2 CPU độc quyền.

```yaml
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "1.5"
      requests:
        memory: "200Mi"
        cpu: "1.5"
```

Pod ở trên chạy trong lớp QoS `Guaranteed` vì `requests` bằng `limits`.
Nhưng giới hạn tài nguyên CPU của container là một số lẻ. Nó chạy trong pool dùng chung.

```yaml
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "2"
```

Pod ở trên chạy trong lớp QoS `Guaranteed` vì chỉ có `limits` được chỉ định, và
`requests` được đặt bằng `limits` khi không được chỉ định tường minh. Và giới hạn tài
nguyên CPU của container là một số nguyên lớn hơn hoặc bằng một. Container `nginx` được
cấp 2 CPU độc quyền.

##### Các tùy chọn của chính sách static (Static policy options) {#cpu-policy-static--options}

Dưới đây là các tùy chọn chính sách khả dụng cho chính sách quản lý CPU static,
liệt kê theo thứ tự bảng chữ cái:

`align-by-socket` (alpha, ẩn theo mặc định)
: Căn chỉnh CPU theo ranh giới physical package / socket, thay vì theo ranh giới NUMA
  logic (khả dụng từ Kubernetes v1.25)

`distribute-cpus-across-cores` (alpha, ẩn theo mặc định)
: Cấp phát các virtual core, đôi khi được gọi là hardware thread, trải trên các physical
  core khác nhau (khả dụng từ Kubernetes v1.31)

`distribute-cpus-across-numa` (beta, hiện theo mặc định)
: Trải CPU trên các miền (domain) NUMA khác nhau, hướng tới sự cân bằng đều giữa các
  miền được chọn (khả dụng từ Kubernetes v1.23)

`full-pcpus-only` (GA, hiện theo mặc định)
: Luôn cấp phát trọn vẹn các physical core (khả dụng từ Kubernetes v1.22, GA từ
  Kubernetes v1.33)

`strict-cpu-reservation` (GA, hiện theo mặc định)
: Ngăn tất cả các pod, bất kể lớp Quality of Service của chúng, chạy trên các CPU đã
  dự trữ (khả dụng từ Kubernetes v1.32, GA từ Kubernetes v1.35)

`prefer-align-cpus-by-uncorecache` (GA, hiện theo mặc định)
: Căn chỉnh CPU theo ranh giới uncore (Last-Level) cache theo phương thức nỗ lực tối đa
  (best-effort) (khả dụng từ Kubernetes v1.32)

Bạn có thể bật/tắt từng nhóm tùy chọn dựa trên mức độ trưởng thành (maturity level) của
chúng bằng các feature gate sau:

* `CPUManagerPolicyBetaOptions` (bật theo mặc định). Tắt để ẩn các tùy chọn mức beta.
* `CPUManagerPolicyAlphaOptions` (tắt theo mặc định). Bật để hiện các tùy chọn mức alpha.

Bạn vẫn phải bật từng tùy chọn thông qua trường `cpuManagerPolicyOptions` trong file cấu
hình kubelet.

Để biết thêm chi tiết về từng tùy chọn mà bạn có thể cấu hình, hãy đọc tiếp.

###### `full-pcpus-only`

Nếu tùy chọn chính sách `full-pcpus-only` được chỉ định, chính sách static sẽ luôn cấp
phát trọn vẹn các physical core.
Theo mặc định, khi không có tùy chọn này, chính sách static cấp phát CPU theo cách phân
bổ vừa khít nhất có nhận biết topology (topology-aware best-fit).
Trên các hệ thống bật SMT, chính sách này có thể cấp phát từng virtual core riêng lẻ,
tương ứng với các hardware thread.
Điều này có thể dẫn đến việc các container khác nhau dùng chung cùng một physical core;
hành vi này lại góp phần gây ra
[vấn đề hàng xóm ồn ào (noisy neighbours problem)](https://en.wikipedia.org/wiki/Cloud_computing_issues#Performance_interference_and_noisy_neighbors).
Khi tùy chọn này được bật, pod chỉ được kubelet chấp nhận (admit) nếu yêu cầu CPU của tất
cả các container của nó có thể được đáp ứng bằng cách cấp phát trọn vẹn các physical core.
Nếu pod không vượt qua bước admission, nó sẽ bị đưa vào trạng thái Failed với thông báo
`SMTAlignmentError`.

###### `distribute-cpus-across-numa`

Nếu tùy chọn chính sách `distribute-cpus-across-numa` được chỉ định, chính sách static
sẽ phân bổ CPU đều trên các NUMA node trong trường hợp cần nhiều hơn một NUMA node để
đáp ứng việc cấp phát.
Theo mặc định, `CPUManager` sẽ dồn (pack) CPU vào một NUMA node cho đến khi node đó đầy,
phần CPU còn lại đơn giản tràn (spill over) sang NUMA node kế tiếp.
Điều này có thể gây ra những điểm nghẽn không mong muốn trong code chạy song song dựa
trên các barrier (và các cơ chế đồng bộ tương tự), vì loại code này thường chỉ chạy nhanh
bằng worker chậm nhất của nó (worker đó bị chậm do có ít CPU khả dụng hơn trên ít nhất
một NUMA node).
Bằng cách phân bổ CPU đều trên các NUMA node, các nhà phát triển ứng dụng có thể dễ dàng
đảm bảo rằng không worker nào phải chịu ảnh hưởng NUMA nhiều hơn các worker khác, cải
thiện hiệu năng tổng thể của những loại ứng dụng này.

###### `align-by-socket`

Nếu tùy chọn chính sách `align-by-socket` được chỉ định, CPU sẽ được coi là căn chỉnh
tại ranh giới socket khi quyết định cách cấp phát CPU cho một container. Theo mặc định,
`CPUManager` căn chỉnh việc cấp phát CPU tại ranh giới NUMA, điều này có thể dẫn đến suy
giảm hiệu năng nếu cần lấy CPU từ nhiều hơn một NUMA node để đáp ứng việc cấp phát. Mặc
dù nó cố gắng đảm bảo tất cả CPU được cấp phát từ số lượng NUMA node _tối thiểu_, không
có gì bảo đảm những NUMA node đó nằm trên cùng một socket. Bằng cách chỉ thị cho
`CPUManager` căn chỉnh CPU một cách tường minh tại ranh giới socket thay vì ranh giới
NUMA, chúng ta có thể tránh được những vấn đề như vậy. Lưu ý, tùy chọn chính sách này
không tương thích với chính sách `single-numa-node` của `TopologyManager` và không áp
dụng cho phần cứng có số socket lớn hơn số NUMA node.

###### `distribute-cpus-across-cores`

Nếu tùy chọn chính sách `distribute-cpus-across-cores` được chỉ định, chính sách static
sẽ cố gắng cấp phát các virtual core (hardware thread) trải trên các physical core khác
nhau. Theo mặc định, `CPUManager` có xu hướng dồn CPU vào càng ít physical core càng
tốt, điều này có thể dẫn đến tranh chấp giữa các CPU trên cùng một physical core và gây
ra các điểm nghẽn hiệu năng. Bằng cách bật chính sách `distribute-cpus-across-cores`,
chính sách static đảm bảo CPU được phân bổ trên càng nhiều physical core càng tốt, giảm
tranh chấp trên cùng một physical core và nhờ đó cải thiện hiệu năng tổng thể. Tuy
nhiên, cần lưu ý rằng chiến lược này có thể kém hiệu quả hơn khi hệ thống chịu tải nặng.
Trong những điều kiện như vậy, lợi ích của việc giảm tranh chấp giảm đi. Ngược lại, hành
vi mặc định có thể giúp giảm chi phí giao tiếp giữa các core (inter-core communication),
có khả năng mang lại hiệu năng tốt hơn trong điều kiện tải cao.

###### `strict-cpu-reservation`

Tham số `reservedSystemCPUs` trong [KubeletConfiguration](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/),
hoặc tùy chọn dòng lệnh kubelet đã lỗi thời (deprecated) `--reserved-cpus`, định nghĩa
một tập CPU tường minh dành cho các daemon hệ thống của hệ điều hành và các daemon hệ
thống của Kubernetes. Chi tiết thêm về tham số này có tại trang
[Danh sách CPU dự trữ tường minh](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#explicitly-reserved-cpu-list).
Theo mặc định, sự cô lập này chỉ được hiện thực cho các pod guaranteed có CPU request là
số nguyên, chứ không áp dụng cho các pod burstable và best-effort (và cả các pod
guaranteed có CPU request là số lẻ). Bước admission chỉ so sánh CPU request với lượng CPU
có thể cấp phát (allocatable). Vì CPU limit cao hơn request, hành vi mặc định cho phép
các pod burstable và best-effort dùng hết dung lượng của `reservedSystemCPUs` và khiến
các dịch vụ của hệ điều hành host bị "đói" tài nguyên trong các triển khai thực tế.
Nếu tùy chọn chính sách `strict-cpu-reservation` được bật, chính sách static sẽ không
cho phép bất kỳ workload nào sử dụng các CPU core được chỉ định trong `reservedSystemCPUs`.

###### `prefer-align-cpus-by-uncorecache`

Nếu chính sách `prefer-align-cpus-by-uncorecache` được chỉ định, chính sách static sẽ
cấp phát tài nguyên CPU cho từng container sao cho tất cả CPU được gán cho một container
cùng chia sẻ một khối uncore cache (còn gọi là Last-Level Cache hay LLC). Theo mặc định,
`CPUManager` sẽ dồn chặt các CPU được gán, điều này có thể khiến container bị gán CPU từ
nhiều uncore cache khác nhau. Tùy chọn này cho phép `CPUManager` cấp phát CPU theo cách
tối đa hóa việc sử dụng hiệu quả uncore cache. Việc cấp phát được thực hiện theo phương
thức nỗ lực tối đa (best-effort), hướng tới việc gắn (affine) càng nhiều CPU càng tốt
trong cùng một uncore cache. Nếu nhu cầu CPU của container vượt quá dung lượng CPU của
một uncore cache đơn lẻ, `CPUManager` sẽ tối thiểu hóa số uncore cache được sử dụng nhằm
duy trì sự căn chỉnh uncore cache tối ưu. Một số workload cụ thể có thể được hưởng lợi
về hiệu năng nhờ việc giảm độ trễ giữa các cache (inter-cache latency) và giảm hàng xóm
ồn ào ở cấp độ cache. Nếu `CPUManager` không thể căn chỉnh tối ưu trong khi node vẫn còn
đủ tài nguyên, container vẫn sẽ được chấp nhận (admit) với hành vi dồn chặt mặc định.

## Trình quản lý memory (Memory manager)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.32 [stable]`

*Memory Manager* là một thành phần của kubelet cung cấp việc cấp phát tài nguyên độc
quyền cho tài nguyên memory. Nó tham vấn Topology Manager để đưa ra các quyết định gán
tài nguyên. Để tìm hiểu thêm, hãy đọc
[Kiểm soát các chính sách quản lý memory trên một node](https://kubernetes.io/docs/tasks/administer-cluster/memory-manager/).

### Các chính sách gán memory cho Pod (Policies for assigning memory to Pods) {#memory-management-policies}

*Memory Manager* của Kubernetes cấp phát tài nguyên RAM (memory, và tùy chọn cả Linux
huge pages) cho các pod thuộc lớp QoS (QoS class) `Guaranteed`.

Memory Manager sử dụng giao thức sinh gợi ý (hint generation protocol) để đưa ra NUMA
affinity phù hợp nhất cho một pod. Memory Manager cung cấp các gợi ý affinity này cho
trình quản lý trung tâm (*Topology Manager*). Dựa trên cả các gợi ý lẫn chính sách của
Topology Manager, pod sẽ bị từ chối hoặc được chấp nhận vào node.

Hơn nữa, Memory Manager đảm bảo rằng lượng memory mà pod yêu cầu được cấp phát từ số
lượng NUMA node tối thiểu.

Để tìm hiểu thêm, hãy đọc [Kiểm soát các chính sách quản lý memory trên một node](https://kubernetes.io/docs/tasks/administer-cluster/memory-manager/).

## Trình quản lý thiết bị (Device manager)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.26 [stable]`

*Device Manager* là một thành phần của kubelet thực hiện cấp phát các thiết bị phần cứng
cho pod thông qua device plugin API. Nó tham vấn Topology Manager, sử dụng thông tin
topology do các device plugin cung cấp, để đưa ra các quyết định gán tài nguyên. Để tìm
hiểu thêm, hãy đọc
[Tích hợp Device Plugin với Topology Manager](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/#device-plugin-integration-with-the-topology-manager).

## Các trình quản lý tài nguyên cấp Pod (Pod-level resource managers) {#pod-level-resource-managers}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Hỗ trợ tài nguyên cấp Pod (pod-level) cho các trình quản lý tài nguyên hiện có
(Topology, CPU, và Memory) mở rộng chúng để xử lý các đặc tả tài nguyên ở cấp pod. Khi
được bật (thông qua các feature gate `PodLevelResources` và `PodLevelResourceManagers`),
các trình quản lý tài nguyên có thể dùng trực tiếp `.spec.resources` làm cơ sở cho các
quyết định cấp phát của chúng, tiến hóa từ mô hình cấp phát thuần túy theo từng container
sang mô hình lấy pod làm trung tâm. Cơ chế phân chia này mang lại một mô hình quản lý tài
nguyên linh hoạt và mạnh mẽ hơn, đặc biệt cho các workload nhạy cảm về hiệu năng. Nó cho
phép bạn định nghĩa các mô hình cấp phát lai (hybrid), trong đó một số container trong
Pod nhận tài nguyên độc quyền được căn chỉnh theo NUMA (NUMA-aligned), trong khi các
container khác chia sẻ phần tài nguyên còn lại từ một pool dùng chung cấp pod.

Điều quan trọng là phân biệt các khả năng mà mỗi scope của Topology Manager mang lại, và
cách điều đó thay đổi hành vi của các trình quản lý tài nguyên. Scope `pod` cho phép cấp
phát dựa trên ngân sách (budget) của toàn bộ pod, tạo ra một pool dùng chung cấp pod cho
các container không phải Guaranteed, song song với các cấp phát độc quyền. Ngược lại,
scope `container` cho phép một mô hình cấp phát lai, trong đó từng container riêng lẻ có
thể nhận tài nguyên độc quyền căn chỉnh theo NUMA trong khi các container khác chạy trong
pool dùng chung của node, mà không cần căn chỉnh toàn bộ pod như một đơn vị duy nhất.

Cả init container tiêu chuẩn lẫn init container có thể khởi động lại (restartable init
container — sidecar) đều được hỗ trợ đầy đủ. Chúng có thể được cấp các lát tài nguyên độc
quyền (exclusive slice) hoặc sử dụng pool dùng chung của pod, và các quy tắc vòng đời
của chúng (ví dụ: tài nguyên có thể tái sử dụng đối với init container tiêu chuẩn so với
việc giữ chỗ lâu dài đối với sidecar) được các trình quản lý tài nguyên cấp pod tôn trọng.

### Thuật ngữ (Glossary)

Đặc tả tài nguyên cấp Pod (Pod level resources specification)
:   Ngân sách tài nguyên được định nghĩa ở cấp Pod trong `.spec.resources`, chỉ định
    tổng thể requests và limits cho toàn bộ pod.

Container Guaranteed (Guaranteed Container)
:   Trong ngữ cảnh của tính năng này, một container được coi là `Guaranteed` nếu nó chỉ
    định resource requests bằng với limits cho cả CPU (việc cấp phát CPU độc quyền yêu
    cầu một giá trị nguyên dương) lẫn Memory. Trạng thái này khiến nó đủ điều kiện được
    các trình quản lý tài nguyên cấp phát tài nguyên độc quyền.

Lát độc quyền (Exclusive slice)
:   Một phần tài nguyên dành riêng (ví dụ: các CPU cụ thể hoặc các trang bộ nhớ cụ thể)
    được cấp phát duy nhất cho một container, đảm bảo sự cô lập khỏi các container khác.

Pool dùng chung của pod (Pod shared pool)
:   Phần còn lại trong tài nguyên đã cấp phát cho pod sau khi tất cả các lát độc quyền
    đã được giữ chỗ. Những tài nguyên này được chia sẻ bởi tất cả các container trong
    pod không nhận cấp phát độc quyền. Mặc dù các container trong pool này chia sẻ tài
    nguyên với nhau, chúng được cô lập nghiêm ngặt khỏi các lát độc quyền và khỏi pool
    dùng chung toàn node.

### Cách các trình quản lý tài nguyên cấp Pod hoạt động (How pod-level resource managers work)

Các trình quản lý tài nguyên CPU và Memory hoạt động khác nhau tùy theo scope của
Topology Manager được cấu hình.

#### Scope pod của Topology Manager và tài nguyên cấp Pod (Topology manager's pod scope and pod-level resources)

Khi scope của Topology Manager được đặt là `pod`, Kubelet thực hiện một lần căn chỉnh
NUMA duy nhất cho toàn bộ pod dựa trên ngân sách tài nguyên định nghĩa trong
`.spec.resources`.

Pool tài nguyên đã căn chỉnh theo NUMA thu được sau đó được phân chia:

1.  **Các lát độc quyền (Exclusive Slices):** Các container chỉ định tài nguyên
    `Guaranteed` (requests bằng limits cho cả CPU lẫn memory, và CPU request là một số
    nguyên dương) được cấp phát các lát độc quyền từ tổng cấp phát của pod.
2.  **Pool dùng chung của pod (Pod Shared Pool):** Phần tài nguyên còn lại tạo thành một
    pool dùng chung, được chia sẻ giữa tất cả các container khác trong pod không nhận
    cấp phát độc quyền. Mặc dù các container trong pool này chia sẻ tài nguyên với nhau,
    chúng được cô lập nghiêm ngặt khỏi các lát độc quyền và khỏi pool dùng chung toàn node.

Lưu ý rằng khi các init container tiêu chuẩn chạy xong (run to completion), tài nguyên
của chúng được thêm vào một tập tài nguyên tái sử dụng của riêng pod, thay vì được trả
về pool tài nguyên của node. Vì chúng chạy tuần tự, những tài nguyên này trở nên có thể
tái sử dụng cho các app container tiếp theo (hoặc cho các lát độc quyền riêng của chúng,
hoặc cho pool dùng chung).

Điều này cho phép bạn đặt cạnh nhau (co-locate) các container cần tài nguyên độc quyền
(ví dụ: một ứng dụng chính hiệu năng cao) với những container không cần (ví dụ: các
sidecar phục vụ logging hoặc monitoring), tất cả trong một pod được căn chỉnh NUMA duy nhất.

Hãy xem xét các container trong pod spec sau, trong đó scope của Topology Manager là
`pod` và pod có tổng ngân sách 4 CPU. `main-app` yêu cầu một lát độc quyền 2 CPU, trong
khi các sidecar chia sẻ 2 CPU còn lại trong pool dùng chung của pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-scope-mixed
  annotations:
    kubernetes.io/description: "A pod demonstrating pod-level scope where one container gets exclusive resources and others share the remaining pod resources in a shared pool."
spec:
  # Ở cấp Pod, Pod có CPU request bằng limits và memory request cũng bằng
  # memory limits. Container main-app thỏa mãn các yêu cầu của lớp QoS
  # Guaranteed ở cấp container, còn các container sidecar không chỉ định
  # resource request nào. Với scope pod, điều này nghĩa là kubelet có thể
  # gán tĩnh 4 CPU cho toàn bộ Pod, trong đó 2 CPU được gán độc quyền cho
  # container main-app, và 2 CPU còn lại được các sidecar chia sẻ trong
  # pool dùng chung của pod.
  resources:
    requests:
      cpu: "4"
      memory: "4Gi"
    limits:
      cpu: "4"
      memory: "4Gi"
  initContainers:
  - name: metrics-sidecar
    image: registry.example/example-image:v1
    restartPolicy: Always
  - name: logging-sidecar
    image: registry.example/example-image:v1
    restartPolicy: Always
  containers:
  - name: main-app
    image: registry.example/example-image:v1
    resources:
      requests:
        cpu: "2"
        memory: "2Gi"
      limits:
        cpu: "2"
        memory: "2Gi"
```

**Những điểm quan trọng cần cân nhắc:**

Khi dùng tài nguyên cấp pod với scope pod của Topology Manager, có một số điểm quan
trọng cần cân nhắc:

*   **Ràng buộc pool dùng chung rỗng (Empty shared pool restriction):** Cấu hình này
    không cho phép các đặc tả pod tạo ra một pool dùng chung của pod bị rỗng nếu vẫn có
    container cần đến nó. Nếu tổng resource requests của tất cả các container
    `Guaranteed` bằng đúng tổng ngân sách tài nguyên, và có ít nhất một container khác
    cần pool dùng chung, pod sẽ bị từ chối tại bước admission.

    Ví dụ, pod sau yêu cầu ngân sách cấp pod là 4 CPU. `main-app` cần 3 CPU độc quyền
    và `metrics-sidecar` cần 1 CPU độc quyền. Vì còn 0 CPU trong pool dùng chung cho
    `logging-sidecar`, pod này bị từ chối (việc kiểm tra tương tự cũng được áp dụng cho
    memory):

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: empty-shared-pool
      annotations:
        kubernetes.io/description: "A pod demonstrating a configuration that is rejected because exclusive containers consume the entire pod resource budget, leaving no resources for the remaining container in the shared pool."
    spec:
      # Ở cấp Pod, Pod có CPU request bằng limits và memory request cũng bằng
      # memory limits. Các container main-app và metrics-sidecar thỏa mãn các
      # yêu cầu của lớp QoS Guaranteed ở cấp container, còn container
      # logging-sidecar không chỉ định resource request nào. Vì các container
      # Guaranteed tiêu thụ toàn bộ ngân sách tài nguyên của pod, để lại
      # 0 CPU cho pool dùng chung mà logging-sidecar cần, pod này sẽ bị
      # từ chối tại bước admission.
      resources:
        requests:
          cpu: "4"
          memory: "4Gi"
        limits:
          cpu: "4"
          memory: "4Gi"
      initContainers:
      - name: metrics-sidecar
        image: registry.example/example-image:v1
        restartPolicy: Always
        resources:
          requests:
            cpu: "1"
            memory: "1Gi"
          limits:
            cpu: "1"
            memory: "1Gi"
      - name: logging-sidecar
        image: registry.example/example-image:v1
        restartPolicy: Always
      containers:
      - name: main-app
        image: registry.example/example-image:v1
        resources:
          requests:
            cpu: "3"
            memory: "3Gi"
          limits:
            cpu: "3"
            memory: "3Gi"
    ```

*   **Tài nguyên bị lãng phí (Wasted resources):** Bất kỳ tài nguyên nào bị cấp phát dư
    khi dùng scope `pod` (tổng requests của các container nhỏ hơn ngân sách cấp pod và
    không có container nào dùng pool dùng chung, hoặc các container trong pool dùng
    chung không sử dụng hết phần còn lại) sẽ vẫn được gán và giữ chỗ cho pod, thực chất
    là bị lãng phí trong suốt quá trình thực thi của pod.

*   **Pool bền vững (Persistent pool):** Tổng pool tài nguyên của pod (sự căn chỉnh NUMA
    và tổng dung lượng đã giữ chỗ) là bền vững. Nếu một container thuộc pool dùng chung
    bị crash và khởi động lại, tổng mức giữ chỗ tài nguyên của pod vẫn được neo an toàn
    trên node. Tài nguyên chỉ được giải phóng trở lại pool chung của node khi toàn bộ
    pod bị chấm dứt (terminate).

#### Scope container của Topology Manager và tài nguyên cấp Pod (Topology manager's container scope and pod-level resources)

Khi scope của Topology Manager được đặt là `container`, Kubelet đánh giá từng container
riêng lẻ để cấp phát độc quyền.

Nếu tổng thể pod đạt được lớp QoS `Guaranteed` (nhờ có các giá trị phù hợp trong
`.spec.resources` cấp Pod), bạn có thể kết hợp linh hoạt các container:

*   Các container có `Guaranteed` requests của riêng chúng nhận tài nguyên độc quyền
    căn chỉnh theo NUMA.
*   Các container `non-Guaranteed` khác trong pod chạy trong pool dùng chung của node.
*   Tổng mức tiêu thụ tài nguyên của tất cả các container vẫn bị cưỡng chế bởi
    `.spec.resources` limits của pod.

Scope này hữu ích khi bạn có một sidecar hạ tầng cần được căn chỉnh vào một NUMA node cụ
thể để truy cập thiết bị, trong khi workload chính có thể chạy trong pool dùng chung
chung của node.

Hãy xem xét các container trong pod spec sau, trong đó scope của Topology Manager là
`container` và pod đại diện cho một workload gồm một sidecar hạ tầng và hai worker ứng
dụng, với tổng ngân sách 4 CPU. `infrastructure-sidecar` nhận một lát 2 CPU độc quyền,
căn chỉnh theo NUMA. Hai worker ứng dụng (`worker-1` và `worker-2`) chạy trong pool dùng
chung toàn node:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: container-scope-mixed
  annotations:
    kubernetes.io/description: "A pod demonstrating container-level scope where one container gets exclusive resources and others run in the node's shared pool."
spec:
  # Ở cấp Pod, Pod có CPU request bằng limits và memory request cũng bằng
  # memory limits. Container infrastructure-sidecar thỏa mãn các yêu cầu của
  # lớp QoS Guaranteed ở cấp container, còn các container worker không chỉ
  # định resource request nào. Với scope container, kubelet đánh giá từng
  # container riêng lẻ để cấp phát độc quyền. Điều này nghĩa là
  # infrastructure-sidecar nhận một lát 2 CPU độc quyền, trong khi các
  # container worker chạy trong pool dùng chung chung của node, tất cả đều
  # bị giới hạn bởi limits tổng thể của pod.
  resources:
    requests:
      cpu: "4"
      memory: "4Gi"
    limits:
      cpu: "4"
      memory: "4Gi"
  initContainers:
  - name: infrastructure-sidecar
    image: registry.example/example-image:v1
    restartPolicy: Always
    resources:
      requests:
        cpu: "2"
        memory: "2Gi"
      limits:
        cpu: "2"
        memory: "2Gi"
  containers:
  - name: worker-1
    image: registry.example/example-image:v1
  - name: worker-2
    image: registry.example/example-image:v1
```

#### Quota CPU (CFS) (CPU quota (CFS))

Khi chạy các workload hỗn hợp bên trong một pod, sự cô lập được cưỡng chế theo cách khác
nhau tùy vào kiểu cấp phát:

*   **Các container độc quyền (Exclusive Containers):** Các container được cấp lát CPU
    độc quyền sẽ bị tắt cơ chế cưỡng chế CPU CFS quota, cho phép chúng chạy mà không bị
    bộ lập lịch Linux hạn chế tốc độ (throttle).
*   **Các container trong pool dùng chung của pod (Pod Shared Pool Containers):** Các
    container rơi vào pool dùng chung của pod được bật CPU CFS quota, đảm bảo chúng
    không tiêu thụ nhiều hơn phần ngân sách pod còn lại và ngăn chúng can thiệp vào các
    container độc quyền.

#### Pool bền vững và việc khởi động lại (Persistent pool and restarts)

Tổng pool tài nguyên của pod (sự căn chỉnh NUMA và tổng dung lượng đã giữ chỗ) là bền
vững. Nếu một container đang sử dụng pool dùng chung của pod bị crash và khởi động lại,
tổng mức giữ chỗ tài nguyên của pod vẫn được neo an toàn trên node. Tài nguyên chỉ được
giải phóng trở lại pool chung của node khi toàn bộ pod bị chấm dứt.

#### Hạ cấp Kubelet và checkpoint trạng thái (Kubelet downgrades and state checkpoints)

> **Thận trọng:**
> Việc bật tính năng `PodLevelResourceManagers` đưa vào các phiên bản trạng thái (state
> version) mới cho các trình quản lý CPU và Memory.
>
> Nếu bạn hạ cấp (downgrade) Kubelet xuống một phiên bản không hỗ trợ tính năng này,
> hoặc nếu bạn tắt các feature gate một cách tường minh sau khi chúng đã từng hoạt động,
> Kubelet phiên bản cũ hơn sẽ không đọc được các file checkpoint mới hơn do sự không
> tương thích phiên bản này. Để khôi phục, quản trị viên phải drain node bị ảnh hưởng,
> xóa thủ công các
> [file checkpoint trạng thái nội bộ](https://kubernetes.io/docs/reference/node/kubelet-files/#resource-managers-state)
> (`cpu_manager_state` và `memory_manager_state`), rồi khởi động lại Kubelet.

### Khả năng quan sát và metrics (Observability and metrics)

Bạn có thể giám sát hành vi và tình trạng của các trình quản lý tài nguyên trên cả các
cấp phát cấp container lẫn cấp pod bằng các metric sau của Kubelet (được bật thông qua
feature gate `PodLevelResourceManagers`):

*   `resource_manager_allocations_total`: Đếm tổng số lần cấp phát tài nguyên độc quyền
    được một trình quản lý thực hiện. Label `source` ("pod" hoặc "node") phân biệt giữa
    các cấp phát lấy từ pool cấp node so với pool cấp pod đã được cấp phát trước.
*   `resource_manager_allocation_errors_total`: Đếm số lỗi gặp phải trong quá trình cấp
    phát tài nguyên độc quyền, phân biệt theo `source` cấp phát dự kiến ("pod" hoặc
    "node").
*   `resource_manager_container_assignments`: Theo dõi số lượng tích lũy các container
    sẽ được cấp một kiểu gán tài nguyên cụ thể. Label `assignment_type`
    ("node_exclusive", "pod_exclusive", "pod_shared") cho biết bao nhiêu container đang
    chạy với tài nguyên độc quyền (từ pool của node hoặc pool của pod) so với pool dùng
    chung cấp pod.

### Hạn chế và lưu ý (Limitations and caveats)

*   Chức năng này chỉ được hiện thực cho chính sách CPU Manager `static` và chính sách
    Memory Manager `Static`. Lưu ý rằng chính sách `BestEffort` không được hỗ trợ cho
    Memory Manager.
*   Tính năng này chỉ được hỗ trợ trên các node Linux. Trên các node Windows, các trình
    quản lý tài nguyên sẽ hoạt động như một no-op (không làm gì) đối với các cấp phát
    cấp pod.

## Tiếp theo (What's next)

Để biết thông tin chi tiết hơn về các trình quản lý tài nguyên cấp node, xem:

*   [Các trình quản lý tài nguyên của node (Node Resource Managers)](https://kubernetes.io/docs/concepts/policy/node-resource-managers/)

Để biết thông tin chi tiết hơn về cách cấu hình và sử dụng các trình quản lý tài nguyên
cấp pod, xem:

*   [Gán tài nguyên CPU và memory ở cấp Pod (Assign Pod-level CPU and memory resources)](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pod-level-resources/)

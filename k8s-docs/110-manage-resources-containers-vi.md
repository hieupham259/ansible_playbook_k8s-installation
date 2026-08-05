# Quản lý tài nguyên cho Pod và Container (Resource Management for Pods and Containers)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/>

Khi bạn chỉ định một Pod, bạn có thể tùy chọn chỉ định lượng tài nguyên mà mỗi
container cần. Các tài nguyên phổ biến nhất để chỉ định là CPU và bộ nhớ (RAM);
ngoài ra còn có những tài nguyên khác.

Khi bạn chỉ định _yêu cầu (request)_ tài nguyên cho các container trong một Pod,
kube-scheduler sẽ dùng thông tin này để quyết định đặt Pod lên node nào.
Khi bạn chỉ định _giới hạn (limit)_ tài nguyên cho một container, kubelet sẽ thực thi
các giới hạn đó để container đang chạy không được phép sử dụng tài nguyên đó
nhiều hơn giới hạn bạn đã đặt. Kubelet cũng dành riêng tối thiểu lượng _request_
của tài nguyên hệ thống đó cho riêng container đó sử dụng.

## Yêu cầu và giới hạn (Requests and limits) {#requests-and-limits}

Nếu node nơi Pod đang chạy còn đủ tài nguyên khả dụng, một container có thể (và
được phép) sử dụng nhiều tài nguyên hơn lượng `request` mà nó chỉ định cho tài nguyên đó.

Ví dụ: nếu bạn đặt request `memory` là 256 MiB cho một container, và container đó nằm trong
một Pod được lập lịch lên một Node có 8GiB bộ nhớ và không có Pod nào khác, thì container
có thể thử dùng nhiều RAM hơn.

Limits lại là một câu chuyện khác. Cả limit `cpu` lẫn `memory` đều được áp dụng bởi kubelet
(và container runtime), và cuối cùng được thực thi bởi kernel. Trên các node Linux,
kernel Linux thực thi các limit bằng cgroups.
Hành vi thực thi limit `cpu` và limit `memory` có đôi chút khác nhau.

Limit `cpu` được thực thi bằng cơ chế điều tiết CPU (CPU throttling). Khi một container
tiến gần đến limit `cpu` của nó, kernel sẽ hạn chế quyền truy cập CPU tương ứng với
limit của container. Do đó, limit `cpu` là một giới hạn cứng do kernel thực thi.
Các container không thể sử dụng nhiều CPU hơn mức được chỉ định trong limit `cpu` của chúng.

Limit `memory` được kernel thực thi bằng cách kill khi hết bộ nhớ (out of memory — OOM kill).
Khi một container dùng nhiều hơn limit `memory` của nó, kernel có thể chấm dứt container đó.
Tuy nhiên, việc chấm dứt chỉ xảy ra khi kernel phát hiện áp lực bộ nhớ (memory pressure).
Vì vậy, một container cấp phát bộ nhớ vượt mức có thể không bị kill ngay lập tức.
Điều này có nghĩa là limit `memory` được thực thi một cách bị động (reactive).
Một container có thể dùng nhiều bộ nhớ hơn limit `memory` của nó, nhưng nếu vậy,
nó có thể bị kill.

> **Ghi chú:** Có một tính năng alpha `MemoryQoS` bổ sung cơ chế điều tiết bộ nhớ và
> tùy chọn đặt trước bộ nhớ theo tầng trên các node Linux dùng cgroup v2. Để biết chi tiết, xem
> [Memory QoS với cgroup v2](./54-pod-qos-vi.md#memory-qos-with-cgroup-v2).

> **Ghi chú:** Nếu bạn chỉ định limit cho một tài nguyên nhưng không chỉ định request nào,
> và không có cơ chế nào ở thời điểm admission áp dụng request mặc định cho tài nguyên đó,
> thì Kubernetes sẽ sao chép limit bạn đã chỉ định và dùng nó làm giá trị request
> cho tài nguyên đó.

## Các loại tài nguyên (Resource types) {#resource-types}

Một *loại tài nguyên (resource type)* có một đơn vị cơ sở và có thể được yêu cầu (request),
giới hạn (limit), hoặc cả hai.
Kubernetes có các loại tài nguyên tích hợp sẵn sau:

| Loại tài nguyên | Mô tả | Đơn vị cơ sở |
|---|---|---|
| `cpu` | Năng lực xử lý tính toán | cpu (core) |
| `memory` | RAM | Byte |
| `ephemeral-storage` | [Lưu trữ tạm thời cục bộ](./95-ephemeral-storage-vi.md) | Byte |
| `hugepages-<size>` | [Huge pages](#huge-pages) (chỉ trên Linux) | Byte |

Cluster cũng có thể cung cấp
[tài nguyên mở rộng (extended resources)](#extended-resources)
(tài nguyên có tên tùy chỉnh, thường được cung cấp bởi các device plugin).

### Huge pages {#huge-pages}

Với các workload Linux, bạn có thể chỉ định tài nguyên _huge page_.
Huge page là một tính năng riêng của Linux, trong đó kernel của node cấp phát
các khối bộ nhớ lớn hơn nhiều so với kích thước trang (page size) mặc định.

Ví dụ: trên một hệ thống có page size mặc định là 4KiB, bạn có thể chỉ định một limit
`hugepages-2Mi: 80Mi`. Nếu container thử cấp phát hơn 40 huge page loại 2MiB
(tổng cộng 80 MiB), việc cấp phát đó sẽ thất bại.

> **Ghi chú:** Bạn không thể overcommit các tài nguyên `hugepages-*`.
> Điều này khác với các tài nguyên `memory` và `cpu`.

CPU và bộ nhớ được gọi chung là *tài nguyên tính toán (compute resources)*, hay đơn giản là
*tài nguyên (resources)*. Tài nguyên tính toán là các đại lượng đo được, có thể được yêu cầu,
cấp phát và tiêu thụ. Chúng khác với
[tài nguyên API (API resources)](./21-kubernetes-api-vi.md). Tài nguyên API, chẳng hạn Pod và
[Service](./82-service-vi.md), là các đối tượng có thể được đọc và sửa đổi
thông qua Kubernetes API server.

## Yêu cầu và giới hạn tài nguyên của Pod và container (Resource requests and limits of Pod and container) {#resource-requests-and-limits-of-pod-and-container}

Với mỗi container, bạn có thể chỉ định các limit và request tài nguyên,
bao gồm những mục sau:

* `spec.containers[].resources.limits.cpu`
* `spec.containers[].resources.limits.memory`
* `spec.containers[].resources.limits.ephemeral-storage`
* `spec.containers[].resources.limits.hugepages-<size>`
* `spec.containers[].resources.requests.cpu`
* `spec.containers[].resources.requests.memory`
* `spec.containers[].resources.requests.ephemeral-storage`
* `spec.containers[].resources.requests.hugepages-<size>`

Mặc dù bạn chỉ có thể chỉ định request và limit cho từng container riêng lẻ,
việc suy nghĩ về tổng thể các request và limit tài nguyên của cả Pod cũng rất hữu ích.
Với một tài nguyên cụ thể, *request/limit tài nguyên của Pod* là tổng các
request/limit tài nguyên loại đó của từng container trong Pod.

## Chỉ định tài nguyên ở cấp Pod (Pod-level resource specification) {#pod-level-resource-specification}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.34 [beta]`

Với điều kiện cluster của bạn đã bật
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) `PodLevelResources`,
bạn có thể chỉ định các request và limit tài nguyên ở
cấp Pod. Ở cấp Pod, Kubernetes v1.36
chỉ hỗ trợ request hoặc limit tài nguyên cho các loại tài nguyên cụ thể: `cpu` và /
hoặc `memory` và / hoặc `hugepages`. Với tính năng này, Kubernetes cho phép bạn khai báo một
ngân sách tài nguyên tổng thể cho Pod, điều này đặc biệt hữu ích khi làm việc với số lượng lớn
container mà việc ước lượng chính xác nhu cầu tài nguyên của từng container là khó khăn.
Ngoài ra, nó cho phép các container trong một Pod chia sẻ tài nguyên nhàn rỗi với nhau,
giúp cải thiện hiệu suất sử dụng tài nguyên.

Với một Pod, bạn có thể chỉ định các limit và request tài nguyên cho CPU và bộ nhớ bằng cách thêm các trường sau:
* `spec.resources.limits.cpu`
* `spec.resources.limits.memory`
* `spec.resources.limits.hugepages-<size>`
* `spec.resources.requests.cpu`
* `spec.resources.requests.memory`
* `spec.resources.requests.hugepages-<size>`

## Đơn vị tài nguyên trong Kubernetes (Resource units in Kubernetes) {#resource-units-in-kubernetes}

### Đơn vị tài nguyên CPU (CPU resource units) {#meaning-of-cpu}

Limit và request cho tài nguyên CPU được đo bằng đơn vị *cpu*.
Trong Kubernetes, 1 đơn vị CPU tương đương với **1 core CPU vật lý**,
hoặc **1 core ảo**, tùy thuộc node là một máy vật lý
hay một máy ảo chạy bên trong một máy vật lý.

Cho phép yêu cầu theo phân số. Khi bạn định nghĩa một container với
`spec.containers[].resources.requests.cpu` đặt là `0.5`, bạn đang yêu cầu một nửa
thời gian CPU so với khi yêu cầu `1.0` CPU.
Với đơn vị tài nguyên CPU, biểu thức [số lượng (quantity)](https://kubernetes.io/docs/reference/kubernetes-api/common-definitions/quantity/) `0.1` tương đương với
biểu thức `100m`, có thể đọc là "một trăm millicpu". Một số người nói
"một trăm millicore", và cách nói này được hiểu là cùng một nghĩa.

Tài nguyên CPU luôn được chỉ định dưới dạng một lượng tài nguyên tuyệt đối, không bao giờ là lượng tương đối. Ví dụ,
`500m` CPU biểu thị lượng năng lực tính toán gần như tương đương dù container đó
chạy trên máy một core, hai core hay 48 core.

> **Ghi chú:** Kubernetes không cho phép bạn chỉ định tài nguyên CPU với độ chính xác nhỏ hơn
> `1m` hoặc `0.001` CPU. Để tránh vô tình dùng một giá trị CPU không hợp lệ, nên chỉ định đơn vị CPU
> theo dạng milliCPU thay vì dạng thập phân khi dùng ít hơn 1 đơn vị CPU.
>
> Ví dụ: bạn có một Pod dùng `5m` hoặc `0.005` CPU và muốn giảm
> tài nguyên CPU của nó. Khi dùng dạng thập phân, sẽ khó nhận ra rằng `0.0005` CPU
> là một giá trị không hợp lệ, còn khi dùng dạng milliCPU, sẽ dễ nhận ra rằng
> `0.5m` là một giá trị không hợp lệ.

### Đơn vị tài nguyên bộ nhớ (Memory resource units) {#meaning-of-memory}

Limit và request cho `memory` được đo bằng byte. Bạn có thể biểu diễn bộ nhớ
dưới dạng một số nguyên thuần hoặc một số điểm cố định (fixed-point) với một trong các hậu tố
[số lượng (quantity)](https://kubernetes.io/docs/reference/kubernetes-api/common-definitions/quantity/) sau:
E, P, T, G, M, k. Bạn cũng có thể dùng các dạng lũy thừa của hai tương đương: Ei, Pi, Ti, Gi,
Mi, Ki. Kubernetes API cũng cho phép hậu tố m (cho millibyte: 1/1000 của một byte),
nhưng hậu tố này không hữu ích khi chỉ định: bạn luôn phải gán số byte nguyên, hoặc đôi khi là các khối lớn hơn như bội số của 1 gibibyte.

Dưới đây là một vài ví dụ về các giá trị bộ nhớ biểu thị xấp xỉ cùng một giá trị:

```shell
128974848, 129e6, 129M,  128974848000m, 123Mi
```

Hãy chú ý đến chữ hoa/thường của các hậu tố. "M" nghĩa là megabyte, còn "m" nghĩa là millibyte. Nếu bạn yêu cầu `400m` memory, đó là yêu cầu 0,4 byte. Người gõ giá trị đó có lẽ muốn yêu cầu 400 mebibyte (`400Mi`)
hoặc 400 megabyte (`400M`).

## Ví dụ về tài nguyên container (Container resources example) {#example-1}

Pod sau có hai container. Cả hai container đều được định nghĩa với request
0.25 CPU
và 64MiB (2<sup>26</sup> byte) bộ nhớ. Mỗi container có limit 0.5
CPU và 128MiB bộ nhớ. Có thể nói Pod này có request 0.5 CPU và 128
MiB bộ nhớ, và limit 1 CPU và 256MiB bộ nhớ.

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: app
    image: images.my-company.example/app:v4
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: log-aggregator
    image: images.my-company.example/log-aggregator:v6
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

## Ví dụ về tài nguyên Pod (Pod resources example) {#example-2}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.34 [beta]`

Tính năng này có thể được bật bằng cách đặt
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) `PodLevelResources`.
Pod sau có request tường minh là 1 CPU và 100 MiB bộ nhớ, và
limit tường minh là 1 CPU và 200 MiB bộ nhớ. Container `pod-resources-demo-ctr-1`
có request và limit tường minh được đặt. Ngược lại, container
`pod-resources-demo-ctr-2` sẽ đơn giản chia sẻ phần tài nguyên khả dụng
trong phạm vi tài nguyên của Pod, vì nó không có request và limit tường minh
được đặt.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-resources-demo
  namespace: pod-resources-example
spec:
  resources:
    limits:
      cpu: "1"
      memory: "200Mi"
    requests:
      cpu: "1"
      memory: "100Mi"
  containers:
  - name: pod-resources-demo-ctr-1
    image: nginx
    resources:
      limits:
        cpu: "0.5"
        memory: "100Mi"
      requests:
        cpu: "0.5"
        memory: "50Mi"
  - name: pod-resources-demo-ctr-2
    image: fedora
    command:
    - sleep
    - inf
```

## Cách các Pod có yêu cầu tài nguyên được lập lịch (How Pods with resource requests are scheduled) {#how-pods-with-resource-requests-are-scheduled}

Khi bạn tạo một Pod, Kubernetes scheduler sẽ chọn một node cho Pod
chạy trên đó. Mỗi node có một dung lượng (capacity) tối đa cho từng loại tài nguyên:
lượng CPU và bộ nhớ nó có thể cung cấp cho các Pod. Scheduler bảo đảm rằng,
với mỗi loại tài nguyên, tổng các request tài nguyên của các container
đã được lập lịch nhỏ hơn dung lượng của node.
Lưu ý rằng mặc dù mức sử dụng bộ nhớ hoặc CPU thực tế trên các node rất thấp,
scheduler vẫn từ chối đặt một Pod lên node nếu việc kiểm tra dung lượng
thất bại. Điều này bảo vệ khỏi tình trạng thiếu hụt tài nguyên trên node
khi mức sử dụng tài nguyên tăng lên về sau, ví dụ trong đợt
cao điểm hằng ngày về tần suất request.

## Cách Kubernetes áp dụng request và limit tài nguyên (How Kubernetes applies resource requests and limits) {#how-pods-with-resource-limits-are-run}

Khi kubelet khởi động một container thuộc một Pod, kubelet sẽ chuyển các request
và limit về bộ nhớ và CPU của container đó cho container runtime.

Trên Linux, container runtime thường cấu hình
các cgroup của kernel để áp dụng và thực thi các
limit bạn đã định nghĩa.

- Limit CPU định nghĩa một mức trần cứng về lượng thời gian CPU mà container có thể sử dụng.
  Trong mỗi khoảng lập lịch (lát thời gian — time slice), kernel Linux kiểm tra xem limit này
  có bị vượt quá hay không; nếu có, kernel sẽ chờ trước khi cho phép cgroup đó tiếp tục thực thi.
- Request CPU thường định nghĩa một trọng số (weighting). Nếu nhiều container (cgroup) khác nhau
  muốn chạy trên một hệ thống đang có tranh chấp, các workload có request CPU lớn hơn được cấp
  nhiều thời gian CPU hơn các workload có request nhỏ.
- Request memory chủ yếu được dùng trong quá trình lập lịch Pod (của Kubernetes). Trên node dùng
  cgroups v2, container runtime có thể dùng request memory làm gợi ý để đặt
  `memory.min` và `memory.low`.
- Limit memory định nghĩa một giới hạn bộ nhớ cho cgroup đó. Nếu container thử
  cấp phát nhiều bộ nhớ hơn limit này, hệ thống con out-of-memory của kernel Linux sẽ kích hoạt
  và, thông thường, can thiệp bằng cách dừng một trong các tiến trình trong container đã thử
  cấp phát bộ nhớ. Nếu tiến trình đó là PID 1 của container, và container được đánh dấu
  là có thể khởi động lại, Kubernetes sẽ khởi động lại container đó.
- Limit memory của Pod hoặc container cũng có thể áp dụng cho các trang (page) trong các volume
  dựa trên bộ nhớ (memory backed volume), chẳng hạn một `emptyDir`. Kubelet theo dõi các volume emptyDir dạng `tmpfs`
  như là mức sử dụng bộ nhớ của container, thay vì như [lưu trữ tạm thời](./95-ephemeral-storage-vi.md) cục bộ.
  Khi dùng `emptyDir` dựa trên bộ nhớ,
  hãy nhớ xem các lưu ý [bên dưới](#memory-backed-emptydir).

Nếu một container vượt quá request memory của nó và node mà nó chạy trên đó bị thiếu
bộ nhớ tổng thể, nhiều khả năng Pod chứa container đó sẽ bị
trục xuất (evicted).

Một container có thể được phép hoặc không được phép vượt limit CPU của nó trong thời gian dài.
Tuy nhiên, container runtime không chấm dứt Pod hoặc container vì sử dụng CPU quá mức.

Để xác định một container không thể được lập lịch hay đang bị kill do giới hạn tài nguyên,
xem mục [Khắc phục sự cố](#troubleshooting).

### Thay đổi kích thước tài nguyên của container (Resizing container resources) {#resizing-container-resources}

Sau khi tạo một Pod, bạn có thể cần điều chỉnh tài nguyên CPU hoặc bộ nhớ của nó dựa trên
mô hình sử dụng thực tế. Kubernetes cung cấp hai cách tiếp cận để thay đổi kích thước tài nguyên Pod:

#### Thay đổi kích thước tại chỗ (In-place resize) {#pod-resize-inplace}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [stable]`

Bạn có thể sửa đổi các `requests` và `limits` CPU và bộ nhớ của các container
trong một Pod đang chạy mà không cần tạo lại nó. Cách này được gọi là _co giãn dọc Pod tại chỗ (in-place Pod vertical scaling)_
hay _thay đổi kích thước Pod tại chỗ (in-place Pod resize)_. Để thực hiện thay đổi kích thước tại chỗ, hãy cập nhật đặc tả tài nguyên
của container thông qua subresource `/resize` của Pod. Bạn có thể kiểm soát việc container
có cần khởi động lại hay không bằng cách đặt trường `resizePolicy` trong đặc tả container.

> **Ghi chú:** Thay đổi kích thước tại chỗ hiện áp dụng cho tài nguyên cấp container. Để thay đổi kích thước tài nguyên
> cấp Pod, xem [Thay đổi kích thước tài nguyên CPU và bộ nhớ của Pod](https://kubernetes.io/docs/tasks/configure-pod-container/resize-pod-resources/).

#### Thay đổi kích thước bằng cách khởi chạy các Pod thay thế (Resizing by launching replacement Pods) {#resizing-by-launching-replacement-pods}

Cách tiếp cận cloud native để thay đổi tài nguyên của một Pod là cập nhật Pod template
trong đối tượng workload (chẳng hạn một Deployment hoặc StatefulSet) và để controller
của workload đó thay các Pod bằng những Pod mới có tài nguyên đã được cập nhật. Cách tiếp cận này
hoạt động với mọi phiên bản Kubernetes và có thể thay đổi bất kỳ đặc tả Pod nào.

Để biết thêm chi tiết về thay đổi kích thước Pod, xem [Thay đổi kích thước Pod](./47-pod-lifecycle-vi.md#pod-resize).
Để có hướng dẫn chi tiết về thay đổi kích thước tại chỗ, xem
[Thay đổi kích thước tài nguyên CPU và bộ nhớ được gán cho Container](https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/).
Bạn cũng có thể dùng [Vertical Pod Autoscaler](./73-vertical-pod-autoscale-vi.md)
để tự động quản lý các khuyến nghị tài nguyên cho Pod.

### Giám sát mức sử dụng tài nguyên tính toán và bộ nhớ (Monitoring compute & memory resource usage) {#monitoring-compute-memory-resource-usage}

Kubelet báo cáo mức sử dụng tài nguyên của một Pod như một phần của
[`status`](https://kubernetes.io/docs/concepts/overview/working-with-objects/#object-spec-and-status) của Pod.

Nếu các [công cụ giám sát](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-usage-monitoring/)
tùy chọn có sẵn trong cluster của bạn, thì mức sử dụng tài nguyên của Pod có thể được truy xuất
trực tiếp từ [Metrics API](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/#metrics-api)
hoặc từ các công cụ giám sát của bạn.

### Những điều cần cân nhắc với volume `emptyDir` dựa trên bộ nhớ (Considerations for memory backed `emptyDir` volumes) {#memory-backed-emptydir}

> **Thận trọng:** Nếu bạn không chỉ định `sizeLimit` cho một volume `emptyDir`, volume đó có thể
> tiêu thụ tới mức limit bộ nhớ của pod đó (`Pod.spec.containers[].resources.limits.memory`).
> Nếu bạn không đặt limit bộ nhớ, pod sẽ không có giới hạn trên về mức tiêu thụ bộ nhớ,
> và có thể tiêu thụ toàn bộ bộ nhớ khả dụng trên node. Kubernetes lập lịch các pod dựa trên
> request tài nguyên (`Pod.spec.containers[].resources.requests`) và sẽ không
> xét đến mức sử dụng bộ nhớ vượt quá request khi quyết định liệu một pod khác có vừa trên
> một node cho trước hay không. Điều này có thể dẫn đến từ chối dịch vụ (denial of service) và khiến
> hệ điều hành phải thực hiện xử lý hết bộ nhớ (OOM). Có thể tạo bất kỳ số lượng `emptyDir` nào
> có khả năng tiêu thụ toàn bộ bộ nhớ khả dụng trên node, khiến OOM
> càng dễ xảy ra hơn.

Từ góc độ quản lý bộ nhớ, có một số điểm tương đồng giữa
việc một tiến trình dùng bộ nhớ làm vùng làm việc và việc dùng `emptyDir`
dựa trên bộ nhớ. Nhưng khi dùng bộ nhớ như một volume, chẳng hạn `emptyDir` dựa trên bộ nhớ,
có thêm những điểm dưới đây mà bạn cần cẩn trọng:

* Các file lưu trên một volume dựa trên bộ nhớ gần như hoàn toàn do
  ứng dụng của người dùng quản lý. Khác với khi dùng làm vùng làm việc cho một tiến trình,
  bạn không thể dựa vào những thứ như thu gom rác (garbage collection) ở cấp ngôn ngữ.
* Mục đích của việc ghi file vào một volume là để lưu dữ liệu hoặc truyền dữ liệu giữa
  các ứng dụng. Cả Kubernetes lẫn hệ điều hành đều có thể không tự động xóa file
  khỏi volume, vì vậy bộ nhớ mà các file đó chiếm dụng không thể được thu hồi khi
  hệ thống hoặc pod chịu áp lực bộ nhớ.
* Một `emptyDir` dựa trên bộ nhớ hữu ích nhờ hiệu năng của nó, nhưng bộ nhớ
  thường nhỏ hơn nhiều về dung lượng và đắt hơn nhiều so với các phương tiện lưu trữ khác,
  chẳng hạn đĩa hoặc SSD. Việc dùng lượng lớn bộ nhớ cho các volume `emptyDir`
  có thể ảnh hưởng đến hoạt động bình thường của pod hoặc của cả node,
  vì vậy nên được sử dụng một cách cẩn thận.

Nếu bạn đang quản trị một cluster hoặc namespace, bạn cũng có thể đặt
[ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/) để giới hạn mức dùng bộ nhớ;
bạn cũng có thể muốn định nghĩa một [LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/)
để thực thi bổ sung.
Nếu bạn chỉ định `spec.containers[].resources.limits.memory` cho từng Pod,
thì kích thước tối đa của một volume `emptyDir` sẽ là limit bộ nhớ của pod.

Một cách khác, người quản trị cluster có thể thực thi giới hạn kích thước cho
các volume `emptyDir` trong các Pod mới bằng một cơ chế chính sách như
[ValidatingAdmissionPolicy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy).

## Lưu trữ tạm thời cục bộ (Local ephemeral storage) {#local-ephemeral-storage}

Để nắm các khái niệm chung về lưu trữ tạm thời cục bộ và các gợi ý về
cấu hình request và/hoặc limit của lưu trữ tạm thời cho một container,
vui lòng xem trang [lưu trữ tạm thời cục bộ](./95-ephemeral-storage-vi.md).

### Giám sát tài nguyên cho lưu trữ tạm thời cục bộ (Resource monitoring for local ephemeral storage) {#resource-monitoring-for-local-ephemeral-storage}

Kubelet có thể đo lượng lưu trữ tạm thời cục bộ đang được sử dụng. Nó
làm được điều này miễn là bạn đã bật tính năng cô lập dung lượng lưu trữ tạm thời cục bộ (local ephemeral storage capacity isolation).

Kubernetes theo dõi lượng lưu trữ tạm thời mà một Pod sử dụng từ các nguồn sau:
* Ghi vào lớp ghi được (rootfs) của container, các container image, hoặc cả hai.
* Ghi vào các volume `emptyDir` cục bộ.
* Log của chính Pod (thường được lưu dưới `/var/log/pods`).
* Các file hệ thống do Kubernetes quản lý được ánh xạ vào Pod, chẳng hạn `/etc/hosts`.

## Tài nguyên mở rộng (Extended resources) {#extended-resources}

Tài nguyên mở rộng là các tên tài nguyên đầy đủ (fully-qualified) nằm ngoài
miền `kubernetes.io`. Chúng cho phép người vận hành cluster quảng bá và người dùng
tiêu thụ các tài nguyên không được tích hợp sẵn trong Kubernetes.

Cần hai bước để sử dụng Tài nguyên mở rộng. Thứ nhất, người vận hành cluster
phải quảng bá một Tài nguyên mở rộng. Thứ hai, người dùng phải yêu cầu
Tài nguyên mở rộng đó trong các Pod.

### Quản lý tài nguyên mở rộng (Managing extended resources) {#managing-extended-resources}

#### Tài nguyên mở rộng cấp node (Node-level extended resources) {#node-level-extended-resources}

Tài nguyên mở rộng cấp node gắn liền với các node.

##### Tài nguyên do device plugin quản lý (Device plugin managed resources) {#device-plugin-managed-resources}
Xem [Device
Plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
để biết cách quảng bá các tài nguyên do device plugin quản lý trên từng node.

##### Các tài nguyên khác (Other resources) {#other-resources}

Để quảng bá một tài nguyên mở rộng cấp node mới, người vận hành cluster có thể
gửi một HTTP request `PATCH` đến API server để chỉ định số lượng
khả dụng trong `status.capacity` của một node trong cluster. Sau thao tác
này, `status.capacity` của node sẽ bao gồm tài nguyên mới. Trường
`status.allocatable` được kubelet tự động cập nhật với tài nguyên mới
theo cách bất đồng bộ.

Vì scheduler dùng giá trị `status.allocatable` của node khi
đánh giá độ phù hợp (fitness) của Pod, scheduler chỉ tính đến giá trị mới sau
lần cập nhật bất đồng bộ đó. Có thể có một độ trễ ngắn giữa thời điểm vá (patch)
dung lượng node với tài nguyên mới và thời điểm Pod đầu tiên yêu cầu
tài nguyên đó có thể được lập lịch lên node.

**Ví dụ:**

Đây là một ví dụ cho thấy cách dùng `curl` để tạo một HTTP request
quảng bá năm tài nguyên "example.com/foo" trên node `k8s-node-1` có master
là `k8s-master`.

```shell
curl --header "Content-Type: application/json-patch+json" \
--request PATCH \
--data '[{"op": "add", "path": "/status/capacity/example.com~1foo", "value": "5"}]' \
http://k8s-master:8080/api/v1/nodes/k8s-node-1/status
```

> **Ghi chú:** Trong request trên, `~1` là mã hóa của ký tự `/`
> trong đường dẫn patch. Giá trị đường dẫn thao tác trong JSON-Patch được diễn giải như một
> JSON-Pointer. Để biết thêm chi tiết, xem
> [IETF RFC 6901, mục 3](https://datatracker.ietf.org/doc/html/rfc6901#section-3).

#### Tài nguyên mở rộng cấp cluster (Cluster-level extended resources) {#cluster-level-extended-resources}

Tài nguyên mở rộng cấp cluster không gắn với các node. Chúng thường được quản lý
bởi các scheduler extender — thành phần xử lý việc tiêu thụ tài nguyên và hạn ngạch (quota) tài nguyên.

Bạn có thể chỉ định các tài nguyên mở rộng được xử lý bởi scheduler extender
trong [cấu hình scheduler](https://kubernetes.io/docs/reference/config-api/kube-scheduler-config.v1/)

**Ví dụ:**

Cấu hình sau đây cho một scheduler policy chỉ ra rằng
tài nguyên mở rộng cấp cluster "example.com/foo" được xử lý bởi
scheduler extender.

- Scheduler chỉ gửi một Pod đến scheduler extender nếu Pod yêu cầu
     "example.com/foo".
- Trường `ignoredByScheduler` chỉ định rằng scheduler không kiểm tra
     tài nguyên "example.com/foo" trong predicate `PodFitsResources` của nó.

```json
{
  "kind": "Policy",
  "apiVersion": "v1",
  "extenders": [
    {
      "urlPrefix":"<extender-endpoint>",
      "bindVerb": "bind",
      "managedResources": [
        {
          "name": "example.com/foo",
          "ignoredByScheduler": true
        }
      ]
    }
  ]
}
```

#### Cấp phát tài nguyên mở rộng bằng DRA (Extended resources allocation by DRA) {#extended-resources-allocation-by-dra}
Cấp phát tài nguyên mở rộng bằng DRA cho phép người quản trị cluster chỉ định một `extendedResourceName`
trong DeviceClass; khi đó các thiết bị khớp với DeviceClass này có thể được yêu cầu từ các yêu cầu
tài nguyên mở rộng của một pod. Đọc thêm về
[Cấp phát tài nguyên mở rộng bằng DRA](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#extended-resource).

### Tiêu thụ tài nguyên mở rộng (Consuming extended resources) {#consuming-extended-resources}

Người dùng có thể tiêu thụ tài nguyên mở rộng trong đặc tả Pod giống như CPU và bộ nhớ.
Scheduler đảm nhiệm việc kế toán tài nguyên (resource accounting) để không cấp phát
đồng thời cho các Pod nhiều hơn lượng khả dụng.

API server giới hạn số lượng của tài nguyên mở rộng ở các số nguyên.
Ví dụ về các số lượng _hợp lệ_ là `3`, `3000m` và `3Ki`. Ví dụ về
các số lượng _không hợp lệ_ là `0.5` và `1500m` (vì `1500m` sẽ cho kết quả `1.5`).

> **Ghi chú:** Tài nguyên mở rộng thay thế cho Opaque Integer Resources.
> Người dùng có thể dùng bất kỳ tiền tố tên miền nào khác `kubernetes.io` — miền này được dành riêng.

Để tiêu thụ một tài nguyên mở rộng trong một Pod, hãy đưa tên tài nguyên vào làm khóa
trong map `spec.containers[].resources.limits` của đặc tả container.

> **Ghi chú:** Tài nguyên mở rộng không thể được overcommit, vì vậy request và limit
> phải bằng nhau nếu cả hai cùng xuất hiện trong đặc tả container.

Một Pod chỉ được lập lịch khi tất cả các request tài nguyên được thỏa mãn, bao gồm
CPU, bộ nhớ và bất kỳ tài nguyên mở rộng nào. Pod sẽ duy trì ở trạng thái `PENDING`
chừng nào request tài nguyên chưa thể được thỏa mãn.

**Ví dụ:**

Pod bên dưới yêu cầu 2 CPU và 1 "example.com/foo" (một tài nguyên mở rộng).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: myimage
    resources:
      requests:
        cpu: 2
        example.com/foo: 1
      limits:
        example.com/foo: 1
```

## Giới hạn PID (PID limiting) {#pid-limiting}

Giới hạn Process ID (PID) cho phép cấu hình một kubelet
để giới hạn số lượng PID mà một Pod nhất định có thể tiêu thụ. Xem
[Giới hạn PID (PID Limiting)](https://kubernetes.io/docs/concepts/policy/pid-limiting/) để biết thêm thông tin.

## Khắc phục sự cố (Troubleshooting) {#troubleshooting}

### Pod của tôi ở trạng thái pending với thông báo sự kiện `FailedScheduling` (My Pods are pending with event message `FailedScheduling`) {#my-pods-are-pending-with-event-message-failedscheduling}

Nếu scheduler không tìm được node nào mà Pod có thể vừa, Pod sẽ ở trạng thái
chưa được lập lịch cho đến khi tìm được một chỗ. Một
[Event](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/event-v1/) sẽ được tạo ra
mỗi khi scheduler không tìm được chỗ cho Pod. Bạn có thể dùng `kubectl`
để xem các event của một Pod; ví dụ:

```shell
kubectl describe pod frontend | grep -A 9999999999 Events
```
```
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  23s   default-scheduler  0/42 nodes available: insufficient cpu
```

Trong ví dụ trên, Pod có tên "frontend" không được lập lịch do
thiếu tài nguyên CPU trên mọi node. Các thông báo lỗi tương tự cũng có thể gợi ý
thất bại do thiếu bộ nhớ (PodExceedsFreeMemory). Nói chung, nếu một Pod
đang pending với thông báo dạng này, có một số việc để thử:

- Thêm node vào cluster.
- Chấm dứt các Pod không cần thiết để nhường chỗ cho các Pod đang pending.
- Kiểm tra xem Pod có lớn hơn tất cả các node hay không. Ví dụ, nếu tất cả các
  node có dung lượng `cpu: 1`, thì một Pod có request `cpu: 1.1` sẽ
  không bao giờ được lập lịch.
- Kiểm tra các taint trên node. Nếu phần lớn node của bạn bị taint, và Pod mới
  không dung thứ (tolerate) taint đó, scheduler chỉ xem xét đặt Pod lên
  các node còn lại không có taint đó.

Bạn có thể kiểm tra dung lượng node và lượng đã cấp phát bằng lệnh
`kubectl describe nodes`. Ví dụ:

```shell
kubectl describe nodes e2e-test-node-pool-4lw4
```
```
Name:            e2e-test-node-pool-4lw4
[ ... lines removed for clarity ...]
Capacity:
 cpu:                               2
 memory:                            7679792Ki
 pods:                              110
Allocatable:
 cpu:                               1800m
 memory:                            7474992Ki
 pods:                              110
[ ... lines removed for clarity ...]
Non-terminated Pods:        (5 in total)
  Namespace    Name                                  CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ---------    ----                                  ------------  ----------  ---------------  -------------
  kube-system  fluentd-gcp-v1.38-28bv1               100m (5%)     0 (0%)      200Mi (2%)       200Mi (2%)
  kube-system  kube-dns-3297075139-61lj3             260m (13%)    0 (0%)      100Mi (1%)       170Mi (2%)
  kube-system  kube-proxy-e2e-test-...               100m (5%)     0 (0%)      0 (0%)           0 (0%)
  kube-system  monitoring-influxdb-grafana-v4-z1m12  200m (10%)    200m (10%)  600Mi (8%)       600Mi (8%)
  kube-system  node-problem-detector-v0.1-fj7m3      20m (1%)      200m (10%)  20Mi (0%)        100Mi (1%)
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  CPU Requests    CPU Limits    Memory Requests    Memory Limits
  ------------    ----------    ---------------    -------------
  680m (34%)      400m (20%)    920Mi (11%)        1070Mi (13%)
```

Trong output trên, bạn có thể thấy rằng nếu một Pod yêu cầu nhiều hơn 1.120 CPU
hoặc nhiều hơn 6.23Gi bộ nhớ, Pod đó sẽ không vừa trên node.

Bằng cách nhìn vào phần “Pods”, bạn có thể thấy những Pod nào đang chiếm chỗ
trên node.

Lượng tài nguyên khả dụng cho các Pod nhỏ hơn dung lượng của node vì
các daemon hệ thống sử dụng một phần tài nguyên khả dụng. Trong Kubernetes API,
mỗi Node có một trường `.status.allocatable`
(xem [NodeStatus](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/node-v1/#NodeStatus)
để biết chi tiết).

Trường `.status.allocatable` mô tả lượng tài nguyên khả dụng
cho các Pod trên node đó (ví dụ: 15 CPU ảo và 7538 MiB bộ nhớ).
Để biết thêm thông tin về tài nguyên có thể cấp phát (allocatable) của node trong Kubernetes, xem
[Dành riêng tài nguyên tính toán cho các daemon hệ thống](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/).

Bạn có thể cấu hình [hạn ngạch tài nguyên (resource quota)](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
để giới hạn tổng lượng tài nguyên mà một namespace có thể tiêu thụ.
Kubernetes thực thi quota cho các đối tượng trong một namespace cụ thể khi có một
ResourceQuota trong namespace đó.
Ví dụ, nếu bạn gán các namespace cụ thể cho các nhóm (team) khác nhau, bạn
có thể thêm ResourceQuota vào các namespace đó. Việc đặt hạn ngạch tài nguyên giúp
ngăn một nhóm sử dụng quá nhiều một tài nguyên nào đó đến mức việc lạm dụng này ảnh hưởng đến các nhóm khác.

Bạn cũng nên cân nhắc quyền truy cập mà bạn cấp cho namespace đó:
quyền ghi **đầy đủ** vào một namespace cho phép người có quyền đó xóa bất kỳ
tài nguyên nào, bao gồm cả một ResourceQuota đã được cấu hình.

### Container của tôi bị chấm dứt (My container is terminated) {#my-container-is-terminated}

Container của bạn có thể bị chấm dứt vì thiếu tài nguyên (resource-starved). Để kiểm tra
xem một container có đang bị kill vì chạm giới hạn tài nguyên hay không, hãy gọi
`kubectl describe pod` trên Pod cần quan tâm:

```shell
kubectl describe pod simmemleak-hra99
```

Output tương tự như:
```
Name:                           simmemleak-hra99
Namespace:                      default
Image(s):                       saadali/simmemleak
Node:                           kubernetes-node-tf0f/10.240.216.66
Labels:                         name=simmemleak
Status:                         Running
Reason:
Message:
IP:                             10.244.2.75
Containers:
  simmemleak:
    Image:  saadali/simmemleak:latest
    Limits:
      cpu:          100m
      memory:       50Mi
    State:          Running
      Started:      Tue, 07 Jul 2019 12:54:41 -0700
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
      Started:      Fri, 07 Jul 2019 12:54:30 -0700
      Finished:     Fri, 07 Jul 2019 12:54:33 -0700
    Ready:          False
    Restart Count:  5
Conditions:
  Type      Status
  Ready     False
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  42s   default-scheduler  Successfully assigned simmemleak-hra99 to kubernetes-node-tf0f
  Normal  Pulled     41s   kubelet            Container image "saadali/simmemleak:latest" already present on machine
  Normal  Created    41s   kubelet            Created container simmemleak
  Normal  Started    40s   kubelet            Started container simmemleak
  Normal  Killing    32s   kubelet            Killing container with id ead3fb35-5cf5-44ed-9ae1-488115be66c6: Need to kill Pod
```

Trong ví dụ trên, `Restart Count:  5` cho biết container `simmemleak`
trong Pod đã bị chấm dứt và khởi động lại năm lần (tính đến hiện tại).
Lý do `OOMKilled` cho thấy container đã thử dùng nhiều bộ nhớ hơn limit của nó.

Bước tiếp theo của bạn có thể là kiểm tra mã ứng dụng xem có rò rỉ bộ nhớ (memory leak) hay không. Nếu bạn
thấy ứng dụng hoạt động đúng như bạn mong đợi, hãy cân nhắc đặt limit
bộ nhớ cao hơn (và có thể cả request) cho container đó.

## Tiếp theo (What's next)

* Thực hành [gán tài nguyên Memory cho container và Pod](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/).
* Thực hành [gán tài nguyên CPU cho container và Pod](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/).
* Đọc cách tài liệu tham chiếu API định nghĩa một [container](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Container)
  và [các yêu cầu tài nguyên](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#resources) của nó
* Đọc thêm về [lưu trữ tạm thời cục bộ](./95-ephemeral-storage-vi.md)
* Đọc thêm về [tài liệu tham chiếu cấu hình kube-scheduler (v1)](https://kubernetes.io/docs/reference/config-api/kube-scheduler-config.v1/)
* Đọc thêm về [các lớp Quality of Service cho Pod](./54-pod-qos-vi.md)
* Đọc thêm về [Cấp phát tài nguyên mở rộng bằng DRA](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#extended-resource)

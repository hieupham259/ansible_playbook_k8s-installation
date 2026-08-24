# Quản lý bộ nhớ swap (Swap memory management)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/cluster-administration/swap-memory-management/>
>
> Cách Kubernetes sử dụng bộ nhớ swap trên node, các hành vi swap, khả năng quan sát, rủi ro và thực hành tốt.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 12](00-ALO-TRINH-ADMIN.md#giai-đoạn-12--quản-trị-cluster-nâng-cao), bài 3/8 ·
Kiểm chứng ở Lab 12 (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài này **trả lời một câu hỏi bạn đã ôm từ Lab 00**. Ở mục
[A4.1](labs/LAB-00-MOI-TRUONG-1.35.7.md#a41-cập-nhật-os-tắt-swap-và-bật-kernel-prerequisites) bạn đã chạy
`swapoff -a` theo kiểu copy-paste, không cần hiểu vì sao. Đây là chỗ hiểu: swap từng bị cấm hoàn
toàn, nay đã có hỗ trợ nhưng kèm rất nhiều điều kiện. Cluster lab vẫn **tắt swap**, nên đọc bài
này để nắm ranh giới chứ chưa phải để bật nó lên.

**Phải hiểu ở lần đọc này:**

- Mặc định trên Linux, **kubelet không khởi động trên node đang bật swap**; muốn chạy phải đặt
  `failSwapOn: false`. (Node Windows thì ngược lại: kubelet không khởi động khi swap **tắt**.)
- Cho kubelet chạy được với swap **chưa** đồng nghĩa workload dùng được swap: mặc định kubelet
  chỉ thị cho CRI cấp phát **0 byte swap** cho workload, và hành vi mặc định là **`NoSwap`**. Chỉ
  `LimitedSwap` mới cho workload đụng vào swap.
- Ngay cả với `NoSwap`, các tiến trình **ngoài** container do Kubernetes quản lý — dịch vụ
  systemd, và cả bản thân kubelet — **vẫn có thể** bị swap.
- Với `LimitedSwap`, chỉ Pod QoS **Burstable** được dùng swap; `BestEffort` và `Guaranteed` bị
  cấm. Giới hạn tính theo tỉ lệ:
  (`containerMemoryRequest` / `nodeTotalMemory`) × `totalPodsSwapAvailable`. Đặt memory request
  bằng đúng memory limit là cách **chủ động không dùng swap**.
- **Scheduler không xét swap** khi đặt Pod — Pod chỉ request `memory`, không request swap. Hệ quả
  là rủi ro "hàng xóm ồn ào", và cách phòng mà bài đề xuất là **gắn taint cho node có swap**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Khả năng quan sát việc dùng swap* — `/metrics/resource`, `machine_swap_bytes`, `kubectl top --show-swap`, `node.status.nodeInfo.swap.capacity` | cần metrics-server và pipeline metric mới đọc được | Lab 11a, CP8 giám sát và cảnh báo |
| *Khám phá swap bằng Node Feature Discovery* | là add-on riêng phải cài thêm, không thuộc core | không cần |
| *Volume dựa trên bộ nhớ* — tùy chọn tmpfs `noswap`, kernel 6.3 | chi tiết kernel, chỉ quan trọng khi thật sự bật swap | CP5 cấu hình lại cluster đang chạy |
| *Trục xuất Pod* — quan hệ giữa eviction threshold và `vm.min_free_kbytes` | là bài toán chỉnh ngưỡng trên node đã bật swap | CP5 cấu hình lại cluster đang chạy |
| *Thực hành tốt khi dùng swap* — `memory.swap.max=0` cho system slice, `io.latency`, chọn đĩa | là runbook cho ngày bạn quyết định bật swap thật | CP1 vòng đời node |

---

Kubernetes có thể được cấu hình để sử dụng bộ nhớ swap trên một node,
cho phép kernel giải phóng bộ nhớ vật lý bằng cách hoán đổi (swap out) các trang bộ nhớ ra thiết bị lưu trữ nền.
Điều này hữu ích cho nhiều tình huống sử dụng khác nhau.
Ví dụ, các node chạy workload có thể hưởng lợi từ việc dùng swap,
chẳng hạn những workload chiếm dụng bộ nhớ lớn nhưng tại mỗi thời điểm chỉ truy cập một phần bộ nhớ đó.
Nó cũng giúp ngăn Pod bị chấm dứt (terminate) trong các đợt tăng đột biến về áp lực bộ nhớ,
bảo vệ node khỏi các đợt tăng đột biến bộ nhớ ở mức hệ thống vốn có thể ảnh hưởng tới sự ổn định của node,
cho phép quản lý bộ nhớ trên node linh hoạt hơn, và nhiều lợi ích khác nữa.

Để tìm hiểu cách cấu hình swap trong cluster của bạn, hãy đọc
[Cấu hình bộ nhớ swap trên các node Kubernetes](https://kubernetes.io/docs/tutorials/cluster-management/provision-swap-memory/).

## Hỗ trợ theo hệ điều hành (Operating system support)

* Các node Linux có hỗ trợ swap; bạn cần cấu hình từng node để bật nó.
  Theo mặc định, kubelet sẽ **không** khởi động trên một node Linux đang bật swap.
* Các node Windows yêu cầu phải có không gian swap.
  Theo mặc định, kubelet **không** khởi động trên một node Windows đang tắt swap.

## Cơ chế hoạt động ra sao? (How does it work?)

Có khá nhiều cách khác nhau mà người ta có thể hình dung về việc dùng swap trên một node.
Nếu kubelet đã đang chạy trên node, bạn cần khởi động lại nó sau khi swap được cấp phát (provision) thì kubelet mới nhận diện được swap.

Khi kubelet khởi động trên một node đã được cấp phát swap và swap sẵn sàng sử dụng
(với cấu hình `failSwapOn: false`), kubelet sẽ:
- Có thể khởi động trên node đã bật swap này.
- Chỉ thị cho phần cài đặt Container Runtime Interface (CRI), thường được gọi là container runtime,
cấp phát mặc định 0 (zero) bộ nhớ swap cho các workload Kubernetes.

Cấu hình swap trên một node được phơi bày cho quản trị viên cluster thông qua
[`memorySwap` trong KubeletConfiguration](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1).
Với vai trò quản trị viên cluster, bạn có thể chỉ định hành vi của node khi
có bộ nhớ swap bằng cách đặt `memorySwap.swapBehavior`.

### Các hành vi swap (Swap behaviors)

Bạn cần chọn một [hành vi swap (swap behavior)](https://kubernetes.io/docs/reference/node/swap-behavior/) để
sử dụng. Các node khác nhau trong cluster của bạn có thể dùng các hành vi swap khác nhau.

Các hành vi swap bạn có thể chọn cho node Linux là:

`NoSwap` (mặc định)
: Các workload chạy dưới dạng Pod trên node này không dùng và không thể dùng swap.

`LimitedSwap`
: Các workload Kubernetes có thể tận dụng bộ nhớ swap.

> **Ghi chú:** Nếu bạn chọn hành vi NoSwap, và bạn cấu hình kubelet để chấp nhận (tolerate)
> không gian swap (`failSwapOn: false`), thì các workload của bạn không dùng swap chút nào.
>
> Tuy nhiên, các tiến trình nằm ngoài các container do Kubernetes quản lý, chẳng hạn các dịch vụ
> systemd (và thậm chí cả bản thân kubelet!) **vẫn có thể** tận dụng swap.

Bạn có thể đọc [cấu hình bộ nhớ swap trên các node Kubernetes](https://kubernetes.io/docs/tutorials/cluster-management/provision-swap-memory/) để tìm hiểu cách bật swap cho cluster của mình.

### Tích hợp với container runtime (Container runtime integration)

Kubelet sử dụng API của container runtime, và chỉ thị cho container runtime
áp dụng cấu hình cụ thể (ví dụ, trong trường hợp cgroup v2 là `memory.swap.max`) theo cách
kích hoạt cấu hình swap mong muốn cho một container. Với các runtime dùng control group (hay cgroup),
container runtime sau đó chịu trách nhiệm ghi các thiết lập này vào cgroup ở mức container.

## Khả năng quan sát việc dùng swap (Observability for swap use)

### Thống kê metric ở mức node và container (Node and container level metric statistics)

Kubelet giờ đây thu thập các thống kê metric ở mức node và mức container,
có thể truy cập tại các HTTP endpoint của kubelet là `/metrics/resource` (chủ yếu được dùng bởi các công cụ
giám sát như Prometheus) và `/stats/summary` (chủ yếu được dùng bởi các Autoscaler).
Điều này cho phép các client có thể yêu cầu trực tiếp kubelet
giám sát mức sử dụng swap và lượng bộ nhớ swap còn lại khi dùng `LimitedSwap`.
Ngoài ra, một metric `machine_swap_bytes` đã được thêm vào cadvisor để hiển thị
tổng dung lượng swap vật lý của máy.
Xem [trang này](https://kubernetes.io/docs/reference/instrumentation/node-metrics/) để biết thêm thông tin.

Ví dụ, các metric `/metrics/resource` sau được hỗ trợ:
- `node_swap_usage_bytes`: Mức sử dụng swap hiện tại của node, tính bằng byte.
- `container_swap_usage_bytes`: Lượng swap mà container đang sử dụng hiện tại, tính bằng byte.
- `container_swap_limit_bytes`: Giới hạn swap hiện tại của container, tính bằng byte.

### Dùng `kubectl top --show-swap` (Using `kubectl top --show-swap`)

Truy vấn metric thì hữu ích, nhưng có phần cồng kềnh, vì các metric này
được thiết kế cho phần mềm sử dụng chứ không phải cho con người đọc.
Để tiêu thụ dữ liệu này theo cách thân thiện hơn với người dùng,
lệnh `kubectl top` đã được mở rộng để hỗ trợ các metric swap, thông qua cờ `--show-swap`.

Để nhận thông tin về mức sử dụng swap trên các node, có thể dùng `kubectl top nodes --show-swap`:
```shell
kubectl top nodes --show-swap
```

Kết quả sẽ tương tự như sau:
```
NAME    CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)   SWAP(bytes)    SWAP(%)       
node1   1m           10%      2Mi             10%         1Mi            0%   
node2   5m           10%      6Mi             10%         2Mi            0%   
node3   3m           10%      4Mi             10%         <unknown>      <unknown>   
```

Để nhận thông tin về mức sử dụng swap của các pod, có thể dùng `kubectl top pods --show-swap`:
```shell
kubectl top pod -n kube-system --show-swap
```

Kết quả sẽ tương tự như sau:
```
NAME                                      CPU(cores)   MEMORY(bytes)   SWAP(bytes)
coredns-58d5bc5cdb-5nbk4                  2m           19Mi            0Mi
coredns-58d5bc5cdb-jsh26                  3m           37Mi            0Mi
etcd-node01                               51m          143Mi           5Mi
kube-apiserver-node01                     98m          824Mi           16Mi
kube-controller-manager-node01            20m          135Mi           9Mi
kube-proxy-ffgs2                          1m           24Mi            0Mi
kube-proxy-fhvwx                          1m           39Mi            0Mi
kube-scheduler-node01                     13m          69Mi            0Mi
metrics-server-8598789fdb-d2kcj           5m           26Mi            0Mi   
```

### Node báo cáo dung lượng swap trong trạng thái node (Nodes to report swap capacity as part of node status)

Một trường trạng thái mới của node đã được bổ sung, `node.status.nodeInfo.swap.capacity`, để báo cáo dung lượng swap của một node.

Ví dụ, có thể dùng lệnh sau để lấy dung lượng swap của các node trong một cluster:
```shell
kubectl get nodes -o go-template='{{range .items}}{{.metadata.name}}: {{if .status.nodeInfo.swap.capacity}}{{.status.nodeInfo.swap.capacity}}{{else}}<unknown>{{end}}{{"\n"}}{{end}}'
```

Kết quả sẽ tương tự như sau:
```
node1: 21474836480
node2: 42949664768
node3: <unknown>
```

> **Ghi chú:** Giá trị `<unknown>` cho biết trường `.status.nodeInfo.swap.capacity` không được thiết lập cho Node đó.
> Điều này rất có thể nghĩa là node không được cấp phát swap, hoặc ít khả năng hơn là
> kubelet không thể xác định được dung lượng swap của node.

### Khám phá swap bằng Node Feature Discovery (NFD) {#node-feature-discovery}

[Node Feature Discovery](https://github.com/kubernetes-sigs/node-feature-discovery)
là một addon của Kubernetes dùng để phát hiện các tính năng và cấu hình phần cứng.
Nó có thể được tận dụng để khám phá xem những node nào đã được cấp phát swap.

Ví dụ, để tìm ra những node nào đã được cấp phát swap,
hãy dùng lệnh sau:
```shell
kubectl get nodes -o jsonpath='{range .items[?(@.metadata.labels.feature\.node\.kubernetes\.io/memory-swap)]}{.metadata.name}{"\t"}{.metadata.labels.feature\.node\.kubernetes\.io/memory-swap}{"\n"}{end}'
```

Kết quả sẽ tương tự như sau:
```
k8s-worker1: true
k8s-worker2: true
k8s-worker3: false
```

Trong ví dụ này, swap được cấp phát trên các node `k8s-worker1` và `k8s-worker2`, nhưng không có trên `k8s-worker3`.

## Rủi ro và lưu ý (Risks and caveats)

> **Thận trọng:** Chúng tôi hết sức khuyến khích bạn mã hóa không gian swap.
> Xem phần [volume dựa trên bộ nhớ (memory-backed volumes)](#memory-backed-volumes) để biết thêm thông tin.

Việc có swap trên một hệ thống làm giảm tính dự đoán được.
Mặc dù swap có thể cải thiện hiệu năng nhờ làm cho nhiều RAM khả dụng hơn, việc hoán đổi dữ liệu
trở lại bộ nhớ là một thao tác nặng, đôi khi chậm hơn nhiều bậc độ lớn,
điều này có thể gây ra sự suy giảm hiệu năng ngoài dự kiến.
Hơn nữa, swap làm thay đổi hành vi của hệ thống khi chịu áp lực bộ nhớ.
Bật swap làm tăng rủi ro "hàng xóm ồn ào" (noisy neighbors),
khi các Pod thường xuyên dùng RAM của chúng có thể khiến các Pod khác bị swap.
Ngoài ra, vì swap cho phép các workload trong Kubernetes dùng nhiều bộ nhớ hơn theo cách không thể hạch toán một cách dự đoán được,
và do các cấu hình đóng gói (packing) ngoài dự kiến,
hiện tại scheduler không tính đến mức sử dụng bộ nhớ swap.
Điều này làm tăng thêm rủi ro "hàng xóm ồn ào".

Hiệu năng của một node đã bật bộ nhớ swap phụ thuộc vào thiết bị lưu trữ vật lý bên dưới.
Khi bộ nhớ swap được sử dụng, hiệu năng sẽ tệ đi đáng kể trong môi trường bị hạn chế
về số thao tác I/O mỗi giây (IOPS), chẳng hạn một máy ảo (VM) trên cloud có
giới hạn (throttling) I/O, so với các phương tiện lưu trữ nhanh hơn như ổ đĩa thể rắn (SSD)
hay NVMe.
Vì swap có thể gây áp lực I/O, khuyến nghị nên dành mức ưu tiên độ trễ I/O cao hơn
cho các daemon quan trọng của hệ thống. Xem phần liên quan trong mục
[các thực hành được khuyến nghị](#good-practice-for-using-swap-in-a-kubernetes-cluster) bên dưới.

### Volume dựa trên bộ nhớ (Memory-backed volumes) {#memory-backed-volumes}

Trên các node Linux, các volume dựa trên bộ nhớ (chẳng hạn các volume mount kiểu [`secret`](109-secret-vi.md),
hoặc [`emptyDir`](91-volumes-vi.md#emptydir) với `medium: Memory`)
được cài đặt bằng hệ thống tệp `tmpfs`.
Nội dung của những volume như vậy phải luôn nằm trong bộ nhớ, do đó không nên
bị hoán đổi (swap) xuống đĩa.
Để đảm bảo nội dung của các volume này vẫn nằm trong bộ nhớ, tùy chọn tmpfs `noswap`
được sử dụng.

Kernel Linux chính thức hỗ trợ tùy chọn `noswap` từ phiên bản 6.3 (có thể tìm thêm thông tin
tại [Yêu cầu phiên bản kernel Linux](https://kubernetes.io/docs/reference/node/kernel-version-requirements/#requirements-other)).
Tuy nhiên, các bản phân phối khác nhau thường chọn backport tùy chọn mount này về cả
các phiên bản Linux cũ hơn.

Để xác minh xem node có hỗ trợ tùy chọn `noswap` hay không, kubelet sẽ làm như sau:
* Nếu phiên bản kernel cao hơn 6.3 thì tùy chọn `noswap` được giả định là được hỗ trợ.
* Ngược lại, kubelet sẽ thử mount một tmpfs giả (dummy) với tùy chọn `noswap` lúc khởi động.
  Nếu kubelet thất bại với lỗi báo tùy chọn không xác định, `noswap` sẽ được giả định là
  không được hỗ trợ, do đó sẽ không được dùng.
  Một mục log của kubelet sẽ được phát ra để cảnh báo người dùng rằng các volume dựa trên bộ nhớ có thể bị swap xuống đĩa.
  Nếu kubelet thành công, tmpfs giả sẽ bị xóa và tùy chọn `noswap` sẽ được sử dụng.
  * Nếu tùy chọn `noswap` không được hỗ trợ, kubelet sẽ phát ra một mục log cảnh báo,
    rồi tiếp tục chạy.

Xem [phần tương ứng ở trang gốc](https://kubernetes.io/docs/concepts/cluster-administration/swap-memory-management/#setting-up-encrypted-swap) với ví dụ về thiết lập swap không mã hóa.
Tuy nhiên, việc xử lý swap đã mã hóa không nằm trong phạm vi của kubelet;
đúng hơn, đó là vấn đề cấu hình chung của hệ điều hành và cần được giải quyết ở mức đó.
Việc cấp phát swap đã mã hóa để giảm thiểu rủi ro này là trách nhiệm của quản trị viên.

### Trục xuất Pod (Evictions)

Việc cấu hình các ngưỡng trục xuất (eviction threshold) theo bộ nhớ cho các node đã bật swap có thể khá phức tạp.

Khi swap bị tắt, việc cấu hình các ngưỡng trục xuất của kubelet thấp hơn một chút
so với dung lượng bộ nhớ của node là hợp lý.
Lý do là chúng ta muốn Kubernetes bắt đầu trục xuất Pod trước khi node cạn bộ nhớ
và kích hoạt trình hủy tiến trình khi hết bộ nhớ (Out Of Memory - OOM killer), bởi OOM killer không hiểu biết về Kubernetes,
do đó không xét đến những thứ như QoS, mức ưu tiên của pod, hay các yếu tố đặc thù khác của Kubernetes.

Khi swap được bật, tình hình phức tạp hơn.
Trong Linux, tham số `vm.min_free_kbytes` định nghĩa ngưỡng bộ nhớ để kernel
bắt đầu thu hồi bộ nhớ một cách tích cực, bao gồm cả việc hoán đổi các trang ra swap.
Nếu các ngưỡng trục xuất của kubelet được đặt sao cho việc trục xuất diễn ra
trước khi kernel bắt đầu thu hồi bộ nhớ, điều đó có thể khiến các workload không bao giờ
có cơ hội swap ra khi node chịu áp lực bộ nhớ.
Tuy nhiên, đặt các ngưỡng trục xuất quá cao lại có thể dẫn đến việc node cạn bộ nhớ
và kích hoạt OOM killer, điều này cũng không lý tưởng.

Để giải quyết vấn đề này, khuyến nghị đặt các ngưỡng trục xuất của kubelet
thấp hơn một chút so với giá trị `vm.min_free_kbytes`.
Bằng cách này, node có thể bắt đầu swap trước khi kubelet bắt đầu trục xuất Pod,
cho phép các workload swap ra dữ liệu không dùng đến và ngăn việc trục xuất xảy ra.
Mặt khác, vì chỉ thấp hơn một chút, kubelet nhiều khả năng vẫn bắt đầu trục xuất Pod
trước khi node cạn bộ nhớ, nhờ đó tránh được OOM killer.

Giá trị của `vm.min_free_kbytes` có thể được xác định bằng cách chạy lệnh sau trên node:
```shell
cat /proc/sys/vm/min_free_kbytes
```

### Không gian swap không được tận dụng (Unutilized swap space)

Với hành vi `LimitedSwap`, lượng swap khả dụng cho một Pod được xác định tự động,
dựa trên tỉ lệ giữa lượng bộ nhớ được yêu cầu (request) so với tổng bộ nhớ của node
(Để biết thêm chi tiết, xem [phần bên dưới](#how-is-the-swap-limit-being-determined-with-limitedswap)).

Thiết kế này có nghĩa là thông thường sẽ có một phần swap bị hạn chế, không dùng được
cho các workload Kubernetes.
Ví dụ, vì Kubernetes v1.36 không cho phép dùng swap cho
các Pod thuộc lớp QoS Guaranteed,
nên lượng swap tỉ lệ với memory request của các pod Guaranteed sẽ
không được các workload Kubernetes sử dụng.

Hành vi này mang theo một số rủi ro trong tình huống có nhiều pod không đủ điều kiện để swap.
Mặt khác, nó thực chất giữ lại một lượng bộ nhớ swap dành riêng cho hệ thống, có thể được dùng bởi các tiến trình
nằm ngoài phạm vi Kubernetes, chẳng hạn các daemon hệ thống và thậm chí cả bản thân kubelet.

## Thực hành tốt khi dùng swap trong cluster Kubernetes (Good practice for using swap in a Kubernetes cluster) {#good-practice-for-using-swap-in-a-kubernetes-cluster}

### Tắt swap cho các daemon quan trọng của hệ thống (Disable swap for system-critical daemons)

Trong giai đoạn thử nghiệm và dựa trên phản hồi của người dùng, người ta quan sát thấy rằng hiệu năng
của các daemon và dịch vụ quan trọng của hệ thống có thể suy giảm.
Điều này ngụ ý rằng các daemon hệ thống, bao gồm cả kubelet, có thể hoạt động chậm hơn bình thường.
Nếu gặp vấn đề này, nên cấu hình cgroup của system slice
để ngăn việc swap (tức là đặt `memory.swap.max=0`).

### Bảo vệ các daemon quan trọng của hệ thống về độ trễ I/O (Protect system-critical daemons for I/O latency)

Swap có thể làm tăng tải I/O trên một node.
Khi áp lực bộ nhớ khiến kernel liên tục swap các trang ra và vào,
các daemon và dịch vụ quan trọng của hệ thống vốn phụ thuộc vào các thao tác I/O có thể
bị suy giảm hiệu năng.

Để giảm thiểu điều này, khuyến nghị người dùng systemd ưu tiên system slice về mặt độ trễ I/O.
Với người không dùng systemd,
nên thiết lập một cgroup riêng cho các daemon và tiến trình hệ thống rồi ưu tiên độ trễ I/O theo cách tương tự.
Có thể đạt được điều này bằng cách đặt `io.latency` cho system slice,
qua đó trao cho nó mức ưu tiên I/O cao hơn.
Xem [tài liệu về cgroup](https://www.kernel.org/doc/Documentation/admin-guide/cgroup-v2.rst) để biết thêm thông tin.

### Swap và các node control plane (Swap and control plane nodes)

Dự án Kubernetes khuyến nghị chạy các node control plane mà không cấu hình bất kỳ không gian swap nào.
Control plane chủ yếu chứa các Pod thuộc lớp QoS Guaranteed, nên nhìn chung có thể tắt swap.
Mối lo ngại chính là việc swap các dịch vụ quan trọng trên control plane có thể ảnh hưởng tiêu cực đến hiệu năng.

### Dùng đĩa riêng cho swap (Use of a dedicated disk for swap)

Dự án Kubernetes khuyến nghị dùng swap đã mã hóa, mỗi khi bạn chạy các node có bật swap.
Nếu swap nằm trên một phân vùng hoặc trên hệ thống tệp gốc (root filesystem), các workload có thể can nhiễu
tới các tiến trình hệ thống cần ghi xuống đĩa.
Khi chúng dùng chung một đĩa, các tiến trình có thể làm quá tải swap,
gây gián đoạn I/O của kubelet, container runtime và systemd, điều này sẽ ảnh hưởng tới các workload khác.
Vì không gian swap nằm trên đĩa, việc đảm bảo đĩa đủ nhanh cho các tình huống sử dụng dự kiến là rất quan trọng.
Ngoài ra, có thể cấu hình mức ưu tiên I/O giữa các vùng ánh xạ khác nhau của cùng một thiết bị nền.

### Lập lịch có nhận biết swap (Swap-aware scheduling)

Kubernetes v1.36 không hỗ trợ phân bổ Pod lên các node theo cách có tính đến
mức sử dụng bộ nhớ swap. Scheduler thường dùng các _request_ tài nguyên hạ tầng
để định hướng việc đặt Pod, và các Pod không request không gian swap; chúng chỉ request `memory`.
Điều này nghĩa là scheduler không xét đến bộ nhớ swap khi ra quyết định lập lịch.
Dù đây là điều chúng tôi đang tích cực triển khai, nó vẫn chưa được hiện thực hóa.

Để quản trị viên đảm bảo rằng các Pod không bị lập lịch lên các node
có bộ nhớ swap trừ khi chúng được chủ đích dùng swap,
quản trị viên có thể gắn taint cho các node có swap để phòng ngừa vấn đề này.
Taint sẽ đảm bảo rằng các workload chấp nhận (tolerate) swap sẽ không tràn sang các node không có swap khi chịu tải.

### Chọn thiết bị lưu trữ để có hiệu năng tối ưu (Selecting storage for optimal performance)

Thiết bị lưu trữ được chỉ định cho không gian swap có vai trò then chốt trong việc duy trì khả năng phản hồi của hệ thống
khi mức sử dụng bộ nhớ cao.
Ổ đĩa cứng quay (HDD) không phù hợp cho nhiệm vụ này vì bản chất cơ học của chúng gây độ trễ đáng kể,
dẫn tới suy giảm hiệu năng nghiêm trọng và hiện tượng thrashing của hệ thống.
Với nhu cầu hiệu năng hiện đại, một thiết bị như ổ đĩa thể rắn (SSD) có lẽ là lựa chọn phù hợp cho swap,
vì khả năng truy cập điện tử độ trễ thấp của nó giảm thiểu sự chậm trễ.


## Chi tiết về hành vi swap (Swap behavior details)

### Giới hạn swap được xác định thế nào với LimitedSwap? (How is the swap limit being determined with LimitedSwap?) {#how-is-the-swap-limit-being-determined-with-limitedswap}

Việc cấu hình bộ nhớ swap, bao gồm cả các giới hạn của nó, là một thách thức
đáng kể. Nó không chỉ dễ bị cấu hình sai, mà vì là một thuộc tính ở mức hệ thống, bất kỳ
cấu hình sai nào cũng có thể ảnh hưởng tới toàn bộ node chứ không chỉ một workload cụ
thể. Để giảm thiểu rủi ro này và bảo đảm sức khỏe của node, chúng tôi đã hiện thực hóa
swap với cơ chế tự động cấu hình các giới hạn.

Với `LimitedSwap`, các Pod không thuộc phân loại QoS Burstable (tức là
các Pod QoS `BestEffort`/`Guaranteed`) bị cấm sử dụng bộ nhớ swap.
Các Pod QoS `BestEffort` có mẫu tiêu thụ bộ nhớ khó dự đoán và thiếu
thông tin về mức sử dụng bộ nhớ của chúng, khiến việc xác định một mức cấp phát
swap an toàn trở nên khó khăn.
Ngược lại, các Pod QoS `Guaranteed` thường được dùng cho những ứng dụng dựa vào
việc cấp phát chính xác các tài nguyên mà workload chỉ định, với bộ nhớ phải khả dụng ngay lập tức.
Để duy trì các bảo đảm về an toàn và sức khỏe node nói trên,
những Pod này không được phép dùng bộ nhớ swap khi `LimitedSwap` có hiệu lực.
Ngoài ra, các pod có mức ưu tiên cao cũng không được phép dùng swap nhằm đảm bảo phần bộ nhớ
chúng tiêu thụ luôn nằm trong RAM, do đó luôn sẵn sàng để dùng.

Trước khi trình bày chi tiết cách tính giới hạn swap, cần định nghĩa các thuật ngữ sau:
* `nodeTotalMemory`: Tổng lượng bộ nhớ vật lý khả dụng trên node.
* `totalPodsSwapAvailable`: Tổng lượng bộ nhớ swap trên node mà các Pod có thể dùng (một phần bộ nhớ swap có thể được dành riêng cho hệ thống).
* `containerMemoryRequest`: Memory request của container.

Giới hạn swap được cấu hình như sau:  
( `containerMemoryRequest` / `nodeTotalMemory` ) × `totalPodsSwapAvailable`

Nói cách khác, lượng swap mà một container có thể dùng tỉ lệ thuận với
memory request của nó, tổng bộ nhớ vật lý của node và tổng lượng bộ nhớ swap trên
node mà các Pod có thể dùng.

Cần lưu ý rằng, với các container nằm trong các Pod QoS Burstable, có thể
chọn không dùng swap (opt-out) bằng cách chỉ định memory request bằng đúng memory limit.
Các container được cấu hình theo cách này sẽ không có quyền truy cập bộ nhớ swap.


## Tiếp theo (What's next)

- Để tìm hiểu về quản lý swap trên các node Linux, hãy đọc
  [cấu hình bộ nhớ swap trên các node Kubernetes](https://kubernetes.io/docs/tutorials/cluster-management/provision-swap-memory/).
- Bạn có thể xem [bài blog về Kubernetes và swap](https://kubernetes.io/blog/2025/03/25/swap-linux-improvements/)
- Để có thông tin nền tảng, vui lòng xem KEP gốc, [KEP-2400](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/2400-node-swap),
và [thiết kế](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/2400-node-swap/README.md) của nó.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 12:

1. Ở [Lab 00 mục A4.1](labs/LAB-00-MOI-TRUONG-1.35.7.md#a41-cập-nhật-os-tắt-swap-và-bật-kernel-prerequisites)
   bạn đã chạy `swapoff -a` trên cả ba node trước khi `kubeadm init`. Bài này giải thích vì sao
   bước đó là bắt buộc? Nếu muốn giữ swap trên `k8s-worker1` thì phải đổi gì?
2. **Câu bẫy.** Bạn đặt `failSwapOn: false` trên `k8s-worker1`, khởi động lại kubelet, node lên
   `Ready`. Workload đã dùng được swap chưa? Còn kubelet và các dịch vụ systemd trên node đó thì
   sao?
3. **Câu bẫy.** Một Pod QoS `Guaranteed` chạy trên node đã bật `LimitedSwap`. Nó được cấp bao
   nhiêu swap, và vì sao kết quả đó ngược với trực giác "Pod quan trọng nhất thì được nhiều nhất"?
4. Bạn bật swap trên `k8s-worker1` nhưng không bật trên `k8s-worker2`. kube-scheduler có tính đến
   khác biệt đó khi đặt Pod không? Bài đề xuất làm gì để Pod không vô tình rơi lên node có swap?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Vì **theo mặc định kubelet sẽ không khởi động trên một node Linux đang bật swap**. Không tắt
   swap thì kubelet chết ngay và node không bao giờ vào được cluster. Muốn giữ swap, phải đặt
   **`failSwapOn: false`** trong cấu hình kubelet rồi khởi động lại kubelet — và bài nhắc thêm:
   nếu kubelet đã chạy sẵn thì phải khởi động lại nó **sau khi** swap được cấp phát thì nó mới
   nhận diện được swap.
2. **Chưa dùng được.** Khi kubelet khởi động trên node có swap với `failSwapOn: false`, nó **chỉ
   thị cho CRI cấp phát mặc định 0 bộ nhớ swap cho workload Kubernetes**, và hành vi mặc định là
   **`NoSwap`** — workload "không dùng và không thể dùng swap". Nhưng **kubelet và các dịch vụ
   systemd thì vẫn có thể swap**, vì chúng nằm ngoài các container do Kubernetes quản lý. Chỗ dễ
   nhầm chính là gộp hai việc "kubelet chịu chạy khi có swap" và "Pod được dùng swap" làm một.
3. **Không được cấp swap nào.** Với `LimitedSwap`, các Pod **không** thuộc QoS Burstable — tức
   `BestEffort` và `Guaranteed` — **bị cấm sử dụng swap**. Lý do bài đưa ra lật ngược trực giác:
   chính vì Pod `Guaranteed` thường dành cho ứng dụng dựa vào việc cấp phát chính xác tài nguyên
   đã chỉ định, **với bộ nhớ phải khả dụng ngay lập tức**, nên cho nó swap là phá đúng thứ nó cần.
   Hệ quả kèm theo: phần swap tỉ lệ với memory request của các Pod Guaranteed sẽ **không được
   workload Kubernetes dùng tới**.
4. **Không.** Kubernetes v1.36 **không hỗ trợ phân bổ Pod theo mức sử dụng swap**: scheduler dựa
   vào resource request, mà Pod chỉ request `memory` chứ không request swap. Bài đề xuất quản trị
   viên **gắn taint cho các node có swap**, để chỉ workload chủ đích dùng swap (và có toleration
   tương ứng) mới rơi vào đó, đồng thời chúng không tràn sang node không có swap khi chịu tải.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

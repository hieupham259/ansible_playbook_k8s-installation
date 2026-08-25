# Các thực hành tốt về Dynamic Resource Allocation dành cho quản trị viên cluster (Good practices for Dynamic Resource Allocation as a Cluster Admin)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/cluster-administration/dra/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 13](00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao),
bài 2/15 · Kiểm chứng ở [Lab 13](labs/LAB-13-DRA.md).

**Giai đoạn 13 không bắt buộc với admin mới.** Phần lớn giai đoạn này là tính năng alpha/beta
hoặc dành cho nền tảng chuyên biệt (AI/HPC, GPU). Chỉ đọc khi đã vững giai đoạn 1–12 hoặc khi
công việc thực sự cần. Cluster lab 3 VM trên VMware, không GPU, **không chạy được** bài này:
không có phần cứng thì không có DRA driver để triển khai, nâng cấp hay drain. Đọc để hiểu
khái niệm và ghi nhớ các bẫy vận hành, không phải để làm theo.

Bài này tuy mang tên "quản trị cluster" nhưng lộ trình **không** xếp nó vào giai đoạn 12 cùng
các bài quản trị khác. Lý do: toàn bộ nội dung phụ thuộc vào DRA — nó nói về quyền trên các
API DRA, về DRA driver và về tải mà Pod dùng claim DRA đặt lên control plane. Đọc nó trước
[bài 149](149-dynamic-resource-allocation-vi.md) thì không có gì để móc vào.

**Phải hiểu ở lần đọc này:**

- Ranh giới phân quyền ở mục *Tách quyền truy cập các API liên quan đến DRA*: DeviceClass và
  ResourceSlice giới hạn cho admin và driver; ResourceClaim và ResourceClaimTemplate — hai API
  có phạm vi namespace — mở cho người triển khai Pod.
- DRA driver là **ứng dụng bên thứ ba** chạy trên mỗi node, thường triển khai dạng DaemonSet;
  Kubernetes không cung cấp sẵn driver nào.
- Ba hệ quả khi driver ngừng hoạt động để nâng cấp mà không có *nâng cấp liền mạch*: Pod mới
  không khởi động được trừ khi claim đã được chuẩn bị sẵn, việc dọn dẹp sau Pod cuối cùng bị
  trì hoãn nên tài nguyên không tái sử dụng được, còn Pod đang chạy vẫn chạy.
- Quy tắc ở mục *Khi drain một node, hãy drain DRA driver muộn nhất có thể*, và lý do: driver
  chịu trách nhiệm hủy chuẩn bị thiết bị đã cấp phát cho Pod.
- Cách đọc ba metric workqueue của controller ResourceClaim để phân biệt hai tình huống: "thiếu
  QPS/burst hoặc CPU/bộ nhớ" và "mức đồng thời không đủ" — vì mức đồng thời được cố định trong
  controller, admin chỉ chỉnh được bằng cách giảm QPS tạo pod.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Con số QPS 75 / Burst 150 từ bài kiểm tra 100 node | là điểm khởi đầu tham chiếu, phụ thuộc quy mô cụ thể | khi công việc thực sự cần |
| Các truy vấn PromQL cho `kube-scheduler`, `kubelet` và DRA kubeletplugin | cần thiết bị thật và stack giám sát để có dữ liệu | khi công việc thực sự cần |
| Liveness probe qua socket gRPC của driver | cấu hình phụ thuộc từng driver | khi công việc thực sự cần |
| Vì sao Pod dùng claim làm tăng tải API server | cơ chế cấp phát nằm ở bài trước | [bài 149](149-dynamic-resource-allocation-vi.md) — mục *Luồng công việc của Kubernetes* |

---

Trang này mô tả các thực hành tốt khi cấu hình một cluster Kubernetes sử dụng
Dynamic Resource Allocation (DRA — cấp phát tài nguyên động). Các hướng dẫn này dành cho
quản trị viên cluster.

## Tách quyền truy cập các API liên quan đến DRA (Separate permissions to DRA related APIs)

DRA được điều phối thông qua một số API khác nhau. Hãy dùng các công cụ ủy quyền
(authorization) (như RBAC, hoặc một giải pháp khác) để kiểm soát quyền truy cập vào đúng
các API tùy theo vai trò (persona) của người dùng.

Nhìn chung, DeviceClass và ResourceSlice nên được giới hạn cho quản trị viên và các DRA
driver. Những người vận hành cluster sẽ triển khai các Pod có claim sẽ cần quyền truy cập
vào các API ResourceClaim và ResourceClaimTemplate; cả hai API này đều có phạm vi theo
namespace (namespace scoped).

## Triển khai và bảo trì DRA driver (DRA driver deployment and maintenance)

DRA driver là các ứng dụng bên thứ ba chạy trên mỗi node của cluster để giao tiếp với phần
cứng của node đó và các thành phần DRA gốc (native) của Kubernetes. Quy trình cài đặt phụ
thuộc vào driver bạn chọn, nhưng nhiều khả năng được triển khai dưới dạng một DaemonSet tới
tất cả hoặc một nhóm node được chọn (dùng node selector hoặc cơ chế tương tự) trong cluster
của bạn.

### Dùng driver có hỗ trợ nâng cấp liền mạch nếu có (Use drivers with seamless upgrade if available)

Các DRA driver hiện thực giao diện
[gói `kubeletplugin`](https://pkg.go.dev/k8s.io/dynamic-resource-allocation/kubeletplugin).
Driver của bạn có thể hỗ trợ _nâng cấp liền mạch_ (seamless upgrade) bằng cách hiện thực
một thuộc tính của giao diện này, cho phép hai phiên bản của cùng một DRA driver cùng tồn
tại trong một thời gian ngắn. Điều này chỉ khả dụng với kubelet phiên bản 1.33 trở lên và
có thể không được driver của bạn hỗ trợ đối với các cluster không đồng nhất (heterogeneous)
có các node gắn kèm chạy những phiên bản Kubernetes cũ hơn — hãy kiểm tra tài liệu của
driver để chắc chắn.

Nếu nâng cấp liền mạch khả dụng trong tình huống của bạn, hãy cân nhắc sử dụng nó để giảm
thiểu độ trễ lập lịch (scheduling delay) khi driver của bạn cập nhật.

Nếu bạn không thể dùng nâng cấp liền mạch, trong thời gian driver ngừng hoạt động để nâng
cấp, bạn có thể quan sát thấy:
* Các Pod không thể khởi động trừ khi các claim mà chúng phụ thuộc đã được chuẩn bị
  (prepared) sẵn để sử dụng.
* Việc dọn dẹp sau pod cuối cùng sử dụng một claim bị trì hoãn cho đến khi driver khả dụng
  trở lại. Pod không được đánh dấu là đã kết thúc (terminated). Điều này ngăn việc tái sử
  dụng các tài nguyên mà pod đó đã dùng cho các pod khác.
* Các pod đang chạy sẽ tiếp tục chạy.

### Xác nhận DRA driver của bạn cung cấp liveness probe và tận dụng nó (Confirm your DRA driver exposes a liveness probe and utilize it)

DRA driver của bạn nhiều khả năng hiện thực một socket gRPC cho việc kiểm tra sức khỏe
(healthcheck) như một phần của các thực hành tốt cho DRA driver. Cách dễ nhất để tận dụng
socket gRPC này là cấu hình nó làm liveness probe cho DaemonSet triển khai DRA driver của
bạn. Tài liệu hoặc công cụ triển khai của driver có thể đã bao gồm điều này, nhưng nếu bạn
tự xây dựng cấu hình riêng hoặc không chạy DRA driver dưới dạng một pod Kubernetes, hãy đảm
bảo rằng công cụ điều phối của bạn khởi động lại DRA driver khi các healthcheck tới socket
gRPC này thất bại. Làm như vậy sẽ giảm thiểu thời gian ngừng hoạt động ngoài ý muốn của DRA
driver và cho nó nhiều cơ hội tự phục hồi hơn, giảm độ trễ lập lịch hoặc thời gian xử lý
sự cố.

### Khi drain một node, hãy drain DRA driver muộn nhất có thể (When draining a node, drain the DRA driver as late as possible)

DRA driver chịu trách nhiệm hủy chuẩn bị (unprepare) mọi thiết bị đã được cấp phát cho các
Pod, và nếu DRA driver bị drain trước khi các Pod có claim được xóa, nó sẽ không thể hoàn
tất việc dọn dẹp của mình. Nếu bạn hiện thực logic drain tùy chỉnh cho các node, hãy cân
nhắc kiểm tra rằng không còn ResourceClaim hoặc ResourceClaimTemplate nào đang được cấp
phát/đặt trước (allocated/reserved) trước khi kết thúc chính DRA driver.

## Giám sát và tinh chỉnh các thành phần cho mức tải cao hơn, đặc biệt trong môi trường quy mô lớn (Monitor and tune components for higher load, especially in high scale environments)

Thành phần control plane kube-scheduler và controller ResourceClaim nội bộ được điều phối
bởi thành phần kube-controller-manager đảm nhận phần việc nặng trong quá trình lập lịch các
Pod có claim, dựa trên metadata được lưu trong các API DRA. So với các Pod không dùng DRA,
số lượng lời gọi tới API server, mức sử dụng bộ nhớ và CPU mà các thành phần này cần đều
tăng lên đối với các Pod dùng claim DRA. Ngoài ra, các thành phần cục bộ trên node như DRA
driver và kubelet sử dụng các API DRA để cấp phát yêu cầu phần cứng tại thời điểm tạo Pod
sandbox. Đặc biệt trong các môi trường quy mô lớn, nơi cluster có nhiều node và/hoặc triển
khai nhiều workload sử dụng nhiều resource claim do DRA định nghĩa, quản trị viên cluster
nên cấu hình các thành phần liên quan để đón trước mức tải gia tăng.

Hệ quả của các thành phần được tinh chỉnh sai có thể gây ảnh hưởng trực tiếp hoặc lan
truyền theo kiểu quả cầu tuyết, tạo ra những triệu chứng khác nhau trong vòng đời Pod. Nếu
cấu hình QPS và burst của thành phần `kube-scheduler` quá thấp, scheduler có thể nhanh
chóng xác định được một node phù hợp cho Pod nhưng lại mất nhiều thời gian hơn để gắn
(bind) Pod vào node đó. Với DRA, trong quá trình lập lịch Pod, các tham số QPS và Burst
trong cấu hình client-go bên trong `kube-controller-manager` là then chốt.

Các giá trị cụ thể để tinh chỉnh cluster của bạn phụ thuộc vào nhiều yếu tố như số lượng
node/pod, tốc độ tạo pod, mức biến động (churn), ngay cả trong các môi trường không dùng
DRA; xem [README của SIG Scalability về các ngưỡng khả năng mở rộng của Kubernetes](https://github.com/kubernetes/community/blob/main/sig-scalability/configs-and-limits/thresholds.md)
để biết thêm thông tin. Trong các bài kiểm tra quy mô thực hiện trên một cluster bật DRA
với 100 node, gồm 720 pod tồn tại lâu dài (bão hòa 90%) và 80 pod biến động (churn 10%, 10
lần), với QPS tạo job là 10, QPS của `kube-controller-manager` có thể đặt thấp tới mức 75
và Burst là 150 mà vẫn đạt các mục tiêu chỉ số tương đương với các triển khai không dùng
DRA. Ở cận dưới này, người ta quan sát thấy bộ giới hạn tốc độ (rate limiter) phía client
được kích hoạt đủ để bảo vệ API server khỏi các đợt bùng nổ, nhưng vẫn đủ cao để các SLO về
khởi động pod không bị ảnh hưởng. Dù đây là một điểm khởi đầu tốt, bạn có thể hình dung rõ
hơn cách tinh chỉnh các thành phần có ảnh hưởng lớn nhất đến hiệu năng DRA cho triển khai
của mình bằng cách giám sát các metric sau đây. Để biết thêm thông tin về tất cả các metric
ổn định (stable) trong Kubernetes, xem
[Tài liệu tham khảo về Metrics của Kubernetes](https://kubernetes.io/docs/reference/instrumentation/metrics/).

### Các metric của `kube-controller-manager` (`kube-controller-manager` metrics)

Các metric sau đây xem xét kỹ controller ResourceClaim nội bộ do thành phần
`kube-controller-manager` quản lý.

* Tốc độ thêm vào workqueue (Workqueue Add Rate): Giám sát
  `sum(rate(workqueue_adds_total{name="resource_claim"}[5m]))` để đánh giá mức độ nhanh
  chóng các mục được thêm vào controller ResourceClaim.
* Độ sâu workqueue (Workqueue Depth): Theo dõi
  `sum(workqueue_depth{endpoint="kube-controller-manager", name="resource_claim"})` để
  nhận diện bất kỳ sự tồn đọng nào trong controller ResourceClaim.
* Thời lượng xử lý workqueue (Workqueue Work Duration): Quan sát
  `histogram_quantile(0.99, sum(rate(workqueue_work_duration_seconds_bucket{name="resource_claim"}[5m])) by (le))`
  để hiểu tốc độ mà controller ResourceClaim xử lý công việc.

Nếu bạn gặp tình trạng Workqueue Add Rate thấp, Workqueue Depth cao, và/hoặc Workqueue Work
Duration cao, điều đó gợi ý rằng controller đang không hoạt động tối ưu. Hãy cân nhắc tinh
chỉnh các tham số như QPS, burst, và cấu hình CPU/bộ nhớ.

Nếu bạn gặp tình trạng Workqueue Add Rate cao, Workqueue Depth cao, nhưng Workqueue Work
Duration ở mức hợp lý, điều đó cho thấy controller vẫn đang xử lý công việc, nhưng mức độ
đồng thời (concurrency) có thể chưa đủ. Mức đồng thời được cố định (hardcoded) trong
controller, vì vậy với tư cách quản trị viên cluster, bạn có thể tinh chỉnh bằng cách giảm
QPS tạo pod, sao cho tốc độ thêm vào workqueue của resource claim dễ kiểm soát hơn.

### Các metric của `kube-scheduler` (`kube-scheduler` metrics)

Các metric scheduler sau đây là những metric mức cao, tổng hợp hiệu năng trên tất cả các
Pod được lập lịch, không chỉ riêng những Pod dùng DRA. Điều quan trọng cần lưu ý là các
metric đầu-cuối (end-to-end) rốt cuộc chịu ảnh hưởng bởi hiệu năng của
`kube-controller-manager` trong việc tạo ResourceClaim từ ResourceClaimTemplate ở những
triển khai sử dụng nhiều ResourceClaimTemplate.

* Thời lượng đầu-cuối của scheduler (Scheduler End-to-End Duration): Giám sát
  `histogram_quantile(0.99, sum(increase(scheduler_pod_scheduling_sli_duration_seconds_bucket[5m])) by (le))`.
* Độ trễ thuật toán của scheduler (Scheduler Algorithm Latency): Theo dõi
  `histogram_quantile(0.99, sum(increase(scheduler_scheduling_algorithm_duration_seconds_bucket[5m])) by (le))`.

### Các metric của `kubelet` (`kubelet` metrics)

Khi một Pod đã gắn vào node cần được thỏa mãn một ResourceClaim, kubelet gọi các phương
thức `NodePrepareResources` và `NodeUnprepareResources` của DRA driver. Bạn có thể quan sát
hành vi này từ góc nhìn của kubelet với các metric sau.

* Kubelet NodePrepareResources: Giám sát
  `histogram_quantile(0.99, sum(rate(dra_operations_duration_seconds_bucket{operation_name="PrepareResources"}[5m])) by (le))`.
* Kubelet NodeUnprepareResources: Theo dõi
  `histogram_quantile(0.99, sum(rate(dra_operations_duration_seconds_bucket{operation_name="UnprepareResources"}[5m])) by (le))`.

### Các thao tác DRA kubeletplugin (DRA kubeletplugin operations)

Các DRA driver hiện thực giao diện
[gói `kubeletplugin`](https://pkg.go.dev/k8s.io/dynamic-resource-allocation/kubeletplugin),
giao diện này cung cấp metric riêng cho các thao tác gRPC bên dưới
`NodePrepareResources` và `NodeUnprepareResources`. Bạn có thể quan sát hành vi này từ góc
nhìn của kubeletplugin nội bộ với các metric sau.

* Thao tác gRPC NodePrepareResources của DRA kubeletplugin: Quan sát
  `histogram_quantile(0.99, sum(rate(dra_grpc_operations_duration_seconds_bucket{method_name=~".*NodePrepareResources"}[5m])) by (le))`.
* Thao tác gRPC NodeUnprepareResources của DRA kubeletplugin: Quan sát
  `histogram_quantile(0.99, sum(rate(dra_grpc_operations_duration_seconds_bucket{method_name=~".*NodeUnprepareResources"}[5m])) by (le))`.

## Tiếp theo (What's next)

* [Tìm hiểu thêm về DRA](149-dynamic-resource-allocation-vi.md)
* Đọc [Tài liệu tham khảo về Metrics của Kubernetes](https://kubernetes.io/docs/reference/instrumentation/metrics/)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho một giai đoạn tùy chọn:

1. Trong bốn API chính của DRA, nhóm nào nên giới hạn cho quản trị viên và driver, nhóm nào
   mở cho người triển khai Pod? Đặc điểm nào của hai API sau khiến ranh giới đó tự nhiên?
2. Bạn drain một node có DRA driver chạy dạng DaemonSet. Vì sao dừng driver sớm — cùng lúc
   hoặc trước các Pod có claim — lại là sai, dù trực giác nói rằng dọn DaemonSet trước cho gọn?
3. Bạn thấy Workqueue Add Rate **cao**, Workqueue Depth **cao**, nhưng Workqueue Work Duration
   ở mức hợp lý. Kết luận gì, và với tư cách quản trị viên bạn chỉnh được cái gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **DeviceClass và ResourceSlice** nên giới hạn cho quản trị viên và các DRA driver — đó là
   nơi khai báo danh mục thiết bị và công bố phần cứng của cluster. **ResourceClaim và
   ResourceClaimTemplate** cần mở cho những người triển khai Pod có claim. Bài chỉ ra đúng đặc
   điểm khiến ranh giới này gọn: **cả hai API sau đều có phạm vi theo namespace**, nên cấp
   quyền cho đúng namespace của từng nhóm là đủ, trong khi hai API trước ở phạm vi cluster.
2. Vì **DRA driver chịu trách nhiệm hủy chuẩn bị (unprepare) mọi thiết bị đã cấp phát cho
   Pod**. Nếu driver bị drain trước khi các Pod có claim được xóa, nó **không thể hoàn tất việc
   dọn dẹp**. Bài khuyên drain DRA driver **muộn nhất có thể**, và nếu tự viết logic drain thì
   kiểm tra không còn ResourceClaim/ResourceClaimTemplate nào đang được cấp phát hoặc đặt trước
   rồi mới kết thúc driver. Đây cũng chính là triệu chứng mô tả ở phần nâng cấp: khi driver
   vắng mặt, Pod cuối cùng dùng claim không được đánh dấu đã kết thúc và tài nguyên của nó
   không tái sử dụng được.
3. Kết luận: **controller vẫn đang xử lý được công việc, nhưng mức độ đồng thời không đủ** —
   nếu là vấn đề tài nguyên hay rate limit thì Work Duration đã cao và Add Rate đã thấp. Điều
   bạn **không** chỉnh được là mức đồng thời: nó **được cố định (hardcoded) trong controller**.
   Đòn bẩy duy nhất của quản trị viên là **giảm QPS tạo pod**, để tốc độ thêm vào workqueue của
   resource claim trở nên dễ kiểm soát. Trường hợp ngược lại — Add Rate thấp, Depth cao, Work
   Duration cao — mới là lúc tinh chỉnh QPS, burst và CPU/bộ nhớ của controller.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

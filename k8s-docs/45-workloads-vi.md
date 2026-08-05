# Workload (Workloads)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/>
>
> Tìm hiểu về Pod — đối tượng tính toán nhỏ nhất có thể triển khai trong Kubernetes —
> và các tầng trừu tượng cấp cao hơn giúp bạn chạy chúng.

Workload là một ứng dụng chạy trên Kubernetes.
Dù workload của bạn là một thành phần đơn lẻ hay nhiều thành phần phối hợp với nhau,
trên Kubernetes bạn chạy nó bên trong một tập các [_Pod_](./46-pods-vi.md).
Trong Kubernetes, một Pod đại diện cho một tập gồm một hoặc nhiều container
đang chạy trên cluster của bạn.

Các Pod trong Kubernetes có [vòng đời được định nghĩa rõ ràng](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/).
Ví dụ, khi một Pod đang chạy trong cluster của bạn thì một lỗi nghiêm trọng trên
node nơi Pod đó đang chạy có nghĩa là tất cả các Pod trên node đó đều thất bại.
Kubernetes coi mức độ thất bại này là chung cuộc: bạn sẽ cần tạo một Pod mới
để khôi phục, kể cả khi node đó sau này khỏe mạnh trở lại.

Tuy nhiên, để mọi việc dễ dàng hơn đáng kể, bạn không cần quản lý trực tiếp từng Pod.
Thay vào đó, bạn có thể dùng các _tài nguyên workload_ (workload resources) quản lý
một tập Pod thay cho bạn. Các tài nguyên này cấu hình các controller
đảm bảo đúng số lượng Pod thuộc đúng loại đang chạy, khớp với trạng thái
mà bạn đã chỉ định.

Kubernetes cung cấp sẵn một số tài nguyên workload:

* [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) và [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
  (thay thế tài nguyên cũ ReplicationController).
  Deployment phù hợp để quản lý một workload ứng dụng phi trạng thái (stateless) trên cluster của bạn,
  trong đó mọi Pod thuộc Deployment đều có thể hoán đổi cho nhau và có thể được thay thế khi cần.
* [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) cho phép bạn
  chạy một hoặc nhiều Pod liên quan có theo dõi trạng thái theo cách nào đó. Ví dụ, nếu workload
  của bạn ghi dữ liệu lâu dài, bạn có thể chạy một StatefulSet ánh xạ mỗi Pod với một
  [PersistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/). Mã của bạn, chạy trong các
  Pod của StatefulSet đó, có thể sao chép (replicate) dữ liệu sang các Pod khác trong cùng StatefulSet
  để cải thiện khả năng chống chịu tổng thể.
* [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) định nghĩa các Pod cung cấp
  những tiện ích cục bộ cho node.
  Mỗi khi bạn thêm vào cluster một node khớp với đặc tả (specification) trong một DaemonSet,
  control plane sẽ lập lịch (schedule) một Pod của DaemonSet đó lên node mới.
  Mỗi Pod trong một DaemonSet thực hiện công việc tương tự như một system daemon trên máy chủ
  Unix / POSIX cổ điển. Một DaemonSet có thể là thành phần nền tảng cho hoạt động của cluster,
  chẳng hạn một plugin để chạy [mạng của cluster (cluster networking)](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-network-model),
  nó có thể giúp bạn quản lý node,
  hoặc có thể cung cấp hành vi tùy chọn giúp nâng cao nền tảng container mà bạn đang vận hành.
* [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) và
  [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) cung cấp những cách khác nhau để
  định nghĩa các tác vụ chạy đến khi hoàn tất rồi dừng lại.
  Bạn có thể dùng [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) để
  định nghĩa một tác vụ chạy đến khi hoàn tất, chỉ một lần. Bạn có thể dùng
  [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) để chạy
  cùng một Job nhiều lần theo lịch.

Trong hệ sinh thái Kubernetes rộng hơn, bạn có thể tìm thấy các tài nguyên workload của bên thứ ba
cung cấp các hành vi bổ sung. Bằng cách dùng một
[custom resource definition](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/),
bạn có thể thêm một tài nguyên workload của bên thứ ba nếu bạn muốn một hành vi cụ thể
không thuộc phần lõi của Kubernetes. Ví dụ, nếu bạn muốn chạy một nhóm Pod cho ứng dụng của mình nhưng
dừng công việc trừ khi _tất cả_ các Pod đều khả dụng (có lẽ cho một tác vụ phân tán thông lượng cao nào đó),
thì bạn có thể triển khai hoặc cài đặt một phần mở rộng (extension) có cung cấp tính năng đó.

## Sắp đặt workload (Workload placement)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [alpha]`

Trong khi các tài nguyên workload tiêu chuẩn (như Deployment và Job) quản lý vòng đời của các Pod,
bạn có thể có những yêu cầu lập lịch phức tạp, trong đó các nhóm Pod phải được đối xử như một đơn vị duy nhất.

[Workload API](https://kubernetes.io/docs/concepts/workloads/workload-api/) cho phép bạn định nghĩa các `PodGroupTemplates` để nhóm các Pod lại và áp dụng các chính sách lập lịch nâng cao cho chúng,
chẳng hạn [gang scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/).
Các controller tạo ra các đối tượng [PodGroup](https://kubernetes.io/docs/concepts/workloads/podgroup-api/) từ các template này lúc chạy (runtime),
và các `Pod` tham chiếu tới `PodGroup` của chúng qua
trường `spec.schedulingGroup`. Điều này đặc biệt hữu ích cho các workload xử lý theo lô (batch processing)
và học máy (machine learning), nơi yêu cầu sắp đặt kiểu "tất cả hoặc không gì cả" (all-or-nothing).

## Tiếp theo (What's next)

Bên cạnh việc đọc về từng loại API (API kind) dành cho quản lý workload, bạn có thể đọc cách
thực hiện các tác vụ cụ thể:

* [Chạy một ứng dụng phi trạng thái bằng Deployment](https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/)
* Chạy một ứng dụng có trạng thái, dưới dạng [một thực thể đơn lẻ](https://kubernetes.io/docs/tasks/run-application/run-single-instance-stateful-application/)
  hoặc dưới dạng [một tập được nhân bản](https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/)
* [Chạy các tác vụ tự động với CronJob](https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/)

Để tìm hiểu về các cơ chế của Kubernetes cho việc tách mã nguồn khỏi cấu hình,
hãy xem [Configuration](https://kubernetes.io/docs/concepts/configuration/).

Có hai khái niệm hỗ trợ cung cấp bối cảnh về cách Kubernetes quản lý Pod
cho các ứng dụng:
* [Garbage collection](./36-garbage-collection-vi.md) dọn dẹp các đối tượng
  khỏi cluster của bạn sau khi _tài nguyên sở hữu_ (owning resource) của chúng đã bị gỡ bỏ.
* [Controller _time-to-live after finished_](https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/)
  gỡ bỏ các Job khi một khoảng thời gian định trước đã trôi qua kể từ lúc chúng hoàn tất.

Khi ứng dụng của bạn đã chạy, bạn có thể muốn đưa nó ra internet dưới dạng
một [Service](https://kubernetes.io/docs/concepts/services-networking/service/) hoặc, chỉ với ứng dụng web,
dùng một [Ingress](./11-ingress-vi.md).

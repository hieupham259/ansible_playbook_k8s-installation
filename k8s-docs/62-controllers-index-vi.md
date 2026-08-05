# Quản lý Workload (Workload Management)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/controllers/>

Kubernetes cung cấp một số API tích hợp sẵn để quản lý các workload
và các thành phần của những workload đó theo cách khai báo (declarative).

Xét cho cùng, ứng dụng của bạn chạy dưới dạng các container bên trong Pod;
tuy nhiên, việc quản lý từng Pod riêng lẻ sẽ tốn rất nhiều công sức. Ví dụ, nếu một Pod
thất bại, có lẽ bạn muốn chạy một Pod mới để thay thế nó. Kubernetes có thể làm điều đó cho bạn.

Bạn dùng Kubernetes API để tạo một đối tượng (object) workload đại diện cho một mức trừu tượng
cao hơn so với Pod, sau đó control plane của Kubernetes tự động quản lý các đối tượng Pod
thay cho bạn, dựa trên đặc tả (specification) của đối tượng workload mà bạn đã định nghĩa.

Các API tích hợp sẵn để quản lý workload gồm:

[Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) (và gián tiếp là [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)),
cách phổ biến nhất để chạy một ứng dụng trên cluster của bạn.
Deployment phù hợp để quản lý một workload ứng dụng phi trạng thái (stateless) trên cluster của bạn,
trong đó mọi Pod thuộc Deployment đều có thể hoán đổi cho nhau và có thể được thay thế khi cần.
(Deployment là sự thay thế cho API ReplicationController cũ).

[StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) cho phép bạn
quản lý một hoặc nhiều Pod — tất cả chạy cùng một mã ứng dụng — trong đó các Pod dựa vào việc
có một định danh (identity) riêng biệt. Điều này khác với Deployment, nơi các Pod được kỳ vọng
là có thể hoán đổi cho nhau.
Cách dùng phổ biến nhất của StatefulSet là để có thể tạo liên kết giữa các Pod của nó và
bộ lưu trữ bền vững (persistent storage) của chúng. Ví dụ, bạn có thể chạy một StatefulSet
gắn mỗi Pod với một [PersistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).
Nếu một trong các Pod của StatefulSet thất bại, Kubernetes tạo một Pod thay thế được kết nối
tới cùng PersistentVolume đó.

[DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) định nghĩa các Pod
cung cấp những tiện ích cục bộ cho một node cụ thể;
ví dụ, một driver cho phép các container trên node đó truy cập một hệ thống lưu trữ. Bạn dùng DaemonSet
khi driver đó, hoặc một dịch vụ cấp node khác, phải chạy trên node mà nó hữu ích.
Mỗi Pod trong một DaemonSet đóng vai trò tương tự một system daemon trên máy chủ Unix / POSIX
cổ điển.
Một DaemonSet có thể là thành phần nền tảng cho hoạt động của cluster của bạn,
chẳng hạn một plugin cho phép node đó truy cập
[mạng của cluster (cluster networking)](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-network-model),
nó có thể giúp bạn quản lý node,
hoặc có thể cung cấp những tiện ích ít thiết yếu hơn giúp nâng cao nền tảng container mà bạn đang vận hành.
Bạn có thể chạy DaemonSet (và các pod của chúng) trên mọi node trong cluster, hoặc chỉ trên một
tập con các node (ví dụ, chỉ cài driver tăng tốc GPU trên những node có gắn GPU).

Bạn có thể dùng [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) và / hoặc
[CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) để
định nghĩa các tác vụ chạy đến khi hoàn tất rồi dừng lại. Một Job đại diện cho một tác vụ chạy
một lần, trong khi mỗi CronJob lặp lại theo lịch.

Các chủ đề khác trong mục này:

- [Tự động dọn dẹp các Job đã hoàn tất (Automatic Cleanup for Finished Jobs)](https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/)
- [ReplicationController](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)

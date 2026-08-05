# Bộ lập lịch của Kubernetes (Kubernetes Scheduler)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/>

Trong Kubernetes, _lập lịch_ (scheduling) là việc đảm bảo các Pod được ghép cặp
với các Node sao cho kubelet có thể chạy chúng.

## Tổng quan về lập lịch (Scheduling overview) {#scheduling}

Một bộ lập lịch (scheduler) theo dõi các Pod mới được tạo mà chưa được gán
Node nào. Với mỗi Pod mà scheduler phát hiện, scheduler chịu trách nhiệm
tìm Node tốt nhất để Pod đó chạy trên. Scheduler đưa ra quyết định
sắp đặt này dựa trên các nguyên tắc lập lịch được mô tả bên dưới.

Nếu bạn muốn hiểu vì sao các Pod được đặt lên một Node cụ thể,
hoặc nếu bạn đang dự định tự triển khai một scheduler tùy chỉnh, trang này
sẽ giúp bạn tìm hiểu về lập lịch.

## kube-scheduler

[kube-scheduler](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)
là scheduler mặc định của Kubernetes và chạy như một phần của control plane.
kube-scheduler được thiết kế sao cho, nếu bạn muốn và cần, bạn có thể
viết thành phần lập lịch của riêng mình và dùng nó thay thế.

Kube-scheduler chọn một node tối ưu để chạy các pod mới được tạo hoặc chưa
được lập lịch (unscheduled). Vì các container trong pod — và bản thân các pod —
có thể có những yêu cầu khác nhau, scheduler lọc bỏ những node
không đáp ứng các nhu cầu lập lịch cụ thể của Pod. Ngoài ra, API cho phép
bạn chỉ định node cho một Pod ngay khi tạo Pod, nhưng cách này không phổ biến
và chỉ được dùng trong các trường hợp đặc biệt.

Trong một cluster, các Node đáp ứng các yêu cầu lập lịch của một Pod
được gọi là các node _khả thi_ (feasible). Nếu không có node nào phù hợp, pod
sẽ ở trạng thái chưa được lập lịch cho đến khi scheduler có thể sắp đặt được nó.

Scheduler tìm các Node khả thi cho một Pod, sau đó chạy một tập các
hàm để chấm điểm các Node khả thi này và chọn Node có điểm cao nhất
trong số đó để chạy Pod. Scheduler sau đó thông báo cho
API server về quyết định này trong một quá trình gọi là _binding_ (gắn kết).

Các yếu tố cần được xét đến khi ra quyết định lập lịch bao gồm
yêu cầu tài nguyên riêng lẻ và tổng hợp, các ràng buộc về phần cứng / phần mềm /
chính sách, các đặc tả affinity và anti-affinity, tính cục bộ của
dữ liệu (data locality), sự can nhiễu giữa các workload, v.v.

### Chọn node trong kube-scheduler (Node selection in kube-scheduler) {#kube-scheduler-implementation}

kube-scheduler chọn node cho pod qua một thao tác gồm 2 bước:

1. Lọc (Filtering)
1. Chấm điểm (Scoring)

Bước _lọc_ tìm ra tập các Node khả thi để lập lịch Pod.
Ví dụ, bộ lọc PodFitsResources kiểm tra xem một Node ứng viên
có đủ tài nguyên khả dụng để đáp ứng các yêu cầu tài nguyên (resource request)
cụ thể của Pod hay không. Sau bước này, danh sách node chứa những Node
phù hợp; thường sẽ có nhiều hơn một Node. Nếu danh sách rỗng, Pod đó
(tạm thời) chưa thể được lập lịch.

Trong bước _chấm điểm_, scheduler xếp hạng các node còn lại để chọn
nơi đặt Pod phù hợp nhất. Scheduler gán một điểm số cho mỗi Node
đã vượt qua bước lọc, dựa trên các quy tắc chấm điểm đang có hiệu lực.

Cuối cùng, kube-scheduler gán Pod cho Node có thứ hạng cao nhất.
Nếu có nhiều hơn một node với điểm số bằng nhau, kube-scheduler chọn
ngẫu nhiên một trong số đó.

Có hai cách được hỗ trợ để cấu hình hành vi lọc và chấm điểm
của scheduler:

1. [Scheduling Policies](https://kubernetes.io/docs/reference/scheduling/policies) cho phép bạn cấu hình các _Predicate_ cho bước lọc và các _Priority_ cho bước chấm điểm.
1. [Scheduling Profiles](https://kubernetes.io/docs/reference/scheduling/config/#profiles) cho phép bạn cấu hình các Plugin hiện thực các giai đoạn lập lịch khác nhau, bao gồm: `QueueSort`, `Filter`, `Score`, `Bind`, `Reserve`, `Permit`, và các giai đoạn khác. Bạn cũng có thể cấu hình kube-scheduler để chạy các profile khác nhau.

## Tiếp theo (What's next)

* Đọc về [tinh chỉnh hiệu năng bộ lập lịch (scheduler performance tuning)](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduler-perf-tuning/)
* Đọc về [ràng buộc phân bố Pod theo topology (Pod topology spread constraints)](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
* Đọc [tài liệu tham khảo](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/) của kube-scheduler
* Đọc tài liệu tham khảo [cấu hình kube-scheduler (v1)](https://kubernetes.io/docs/reference/config-api/kube-scheduler-config.v1/)
* Tìm hiểu về [cấu hình nhiều scheduler](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/)
* Tìm hiểu về [các chính sách quản lý topology (topology management policies)](https://kubernetes.io/docs/tasks/administer-cluster/topology-manager/)
* Tìm hiểu về [Pod Overhead](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-overhead/)
* Tìm hiểu về việc lập lịch cho các Pod sử dụng volume trong:
  * [Hỗ trợ Volume Topology](https://kubernetes.io/docs/concepts/storage/storage-classes/#volume-binding-mode)
  * [Theo dõi dung lượng lưu trữ (Storage Capacity Tracking)](https://kubernetes.io/docs/concepts/storage/storage-capacity/)
  * [Giới hạn Volume theo từng Node (Node-specific Volume Limits)](https://kubernetes.io/docs/concepts/storage/storage-limits/)

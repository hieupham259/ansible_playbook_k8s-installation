# Lập lịch, Preemption và Eviction (Scheduling, Preemption and Eviction)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/>

Trong Kubernetes, lập lịch (scheduling) là việc đảm bảo các Pod
được ghép cặp với các Node sao cho kubelet có thể chạy chúng. Preemption (chiếm chỗ)
là quá trình chấm dứt các Pod có Priority (độ ưu tiên) thấp hơn
để các Pod có Priority cao hơn có thể được lập lịch lên Node. Eviction (thu hồi) là quá trình
chấm dứt một hoặc nhiều Pod trên các Node.

## Lập lịch (Scheduling)

* [Bộ lập lịch của Kubernetes (Kubernetes Scheduler)](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
* [Gán Pod cho Node (Assigning Pods to Nodes)](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
* [Pod Overhead](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-overhead/)
* [Ràng buộc phân bố Pod theo topology (Pod Topology Spread Constraints)](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
* [Taint và Toleration (Taints and Tolerations)](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
* [Scheduling Framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework)
* [Cấp phát tài nguyên động (Dynamic Resource Allocation)](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation)
* [Tinh chỉnh hiệu năng bộ lập lịch (Scheduler Performance Tuning)](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduler-perf-tuning/)
* [Đóng gói tài nguyên cho các tài nguyên mở rộng (Resource Bin Packing for Extended Resources)](https://kubernetes.io/docs/concepts/scheduling-eviction/resource-bin-packing/)
* [Mức sẵn sàng lập lịch của Pod (Pod Scheduling Readiness)](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-scheduling-readiness/)
* [Lập lịch PodGroup (PodGroup Scheduling)](https://kubernetes.io/docs/concepts/scheduling-eviction/podgroup-scheduling/)
* [Gang Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/)
* [Lập lịch nhận biết topology (Topology-aware Scheduling)](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-aware-scheduling/)
* [Preemption nhận biết workload (Workload-Aware preemption)](https://kubernetes.io/docs/concepts/scheduling-eviction/workload-aware-preemption/)
* [Descheduler](https://github.com/kubernetes-sigs/descheduler#descheduler-for-kubernetes)
* [Các tính năng do Node khai báo (Node Declared Features)](https://kubernetes.io/docs/concepts/scheduling-eviction/node-declared-features/)

## Sự gián đoạn Pod (Pod Disruption)

[Sự gián đoạn Pod (Pod disruption)](./53-disruptions-vi.md) là quá trình trong đó
các Pod trên các Node bị chấm dứt một cách tự nguyện (voluntary) hoặc không tự nguyện (involuntary).

Gián đoạn tự nguyện được khởi phát một cách có chủ đích bởi chủ sở hữu ứng dụng hoặc
quản trị viên cluster. Gián đoạn không tự nguyện là ngoài ý muốn và có thể bị kích hoạt bởi
những vấn đề không thể tránh khỏi như Node cạn kiệt tài nguyên (resources),
hoặc do việc xóa nhầm.

* [Độ ưu tiên và Preemption của Pod (Pod Priority and Preemption)](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
* [Eviction do áp lực node (Node-pressure Eviction)](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)
* [Eviction khởi phát qua API (API-initiated Eviction)](https://kubernetes.io/docs/concepts/scheduling-eviction/api-eviction/)

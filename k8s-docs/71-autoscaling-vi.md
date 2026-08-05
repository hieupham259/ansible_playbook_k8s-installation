# Tự động co giãn Workload (Autoscaling Workloads)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/autoscaling/>
>
> Với tính năng tự động co giãn (autoscaling), bạn có thể tự động cập nhật workload của mình
> theo cách này hay cách khác. Điều này cho phép cluster của bạn phản ứng với các thay đổi
> về nhu cầu tài nguyên một cách linh hoạt và hiệu quả hơn.

Trong Kubernetes, bạn có thể _co giãn_ (scale) một workload tùy theo nhu cầu tài nguyên hiện tại.
Điều này cho phép cluster của bạn phản ứng với các thay đổi về nhu cầu tài nguyên
một cách linh hoạt và hiệu quả hơn.

Khi co giãn một workload, bạn có thể tăng hoặc giảm số lượng bản sao (replica) do workload
quản lý, hoặc điều chỉnh tại chỗ (in-place) lượng tài nguyên khả dụng cho các bản sao đó.

Cách tiếp cận thứ nhất được gọi là _co giãn ngang_ (horizontal scaling), còn cách thứ hai
được gọi là _co giãn dọc_ (vertical scaling).

Có cả cách thủ công lẫn tự động để co giãn workload, tùy theo trường hợp sử dụng của bạn.

## Co giãn workload thủ công (Scaling workloads manually)

Kubernetes hỗ trợ _co giãn thủ công_ (manual scaling) cho workload. Co giãn ngang có thể
thực hiện bằng CLI `kubectl`.
Với co giãn dọc, bạn cần _patch_ (vá) định nghĩa tài nguyên của workload.

Xem ví dụ cho cả hai chiến lược bên dưới.

- **Co giãn ngang**: [Chạy nhiều instance của ứng dụng](https://kubernetes.io/docs/tutorials/kubernetes-basics/scale/scale-intro/)
- **Co giãn dọc**: [Thay đổi kích thước tài nguyên CPU và bộ nhớ gán cho container](https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources)

## Tự động co giãn workload (Scaling workloads automatically)

Kubernetes cũng hỗ trợ _tự động co giãn_ (automatic scaling) cho workload — đây là trọng tâm
của trang này.

Khái niệm _Autoscaling_ trong Kubernetes chỉ khả năng tự động cập nhật một
đối tượng quản lý một tập các Pod (ví dụ một Deployment).

### Co giãn workload theo chiều ngang (Scaling workloads horizontally)

Trong Kubernetes, bạn có thể tự động co giãn một workload theo chiều ngang bằng
[HorizontalPodAutoscaler](./72-horizontal-pod-autoscale-vi.md) (HPA).

Nó được hiện thực dưới dạng một tài nguyên API Kubernetes và một controller,
định kỳ điều chỉnh số lượng replica trong một workload sao cho khớp với mức sử dụng
tài nguyên quan sát được, chẳng hạn mức sử dụng CPU hoặc bộ nhớ.

Có một [hướng dẫn thực hành từng bước](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough)
về cách cấu hình HorizontalPodAutoscaler cho một Deployment.

### Co giãn workload theo chiều dọc (Scaling workloads vertically)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.25 [stable]`

Bạn có thể tự động co giãn một workload theo chiều dọc bằng
[VerticalPodAutoscaler](https://kubernetes.io/docs/concepts/workloads/autoscaling/vertical-pod-autoscale/) (VPA).
Khác với HPA, VPA không có sẵn cùng Kubernetes theo mặc định, mà là một add-on
mà bạn hoặc quản trị viên cluster có thể cần triển khai trước khi sử dụng.

Sau khi cài đặt, nó cho phép bạn tạo các CustomResourceDefinition (CRD) cho workload,
định nghĩa _cách thức_ và _thời điểm_ co giãn tài nguyên của các replica được quản lý.

> **Ghi chú:**
> Bạn cần cài đặt [Metrics Server](https://github.com/kubernetes-sigs/metrics-server)
> vào cluster để VPA hoạt động.

#### Co giãn dọc Pod tại chỗ (In-place pod vertical scaling)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [stable]`

Tính đến Kubernetes v1.36, VPA chưa hỗ trợ thay đổi kích thước Pod tại chỗ (in-place),
nhưng phần tích hợp này đang được phát triển.
Để thay đổi kích thước Pod tại chỗ theo cách thủ công, xem
[Thay đổi kích thước tài nguyên container tại chỗ](https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/).

### Tự động co giãn dựa trên kích thước cluster (Autoscaling based on cluster size)

Với các workload cần được co giãn theo kích thước của cluster (ví dụ
`cluster-dns` hoặc các thành phần hệ thống khác), bạn có thể dùng
[_Cluster Proportional Autoscaler_](https://github.com/kubernetes-sigs/cluster-proportional-autoscaler).
Giống như VPA, nó không thuộc phần lõi của Kubernetes, mà được lưu trữ như một
dự án riêng trên GitHub.

Cluster Proportional Autoscaler theo dõi số lượng node có thể lập lịch (schedulable)
và số core, rồi co giãn số replica của workload đích một cách tương ứng.

Nếu số lượng replica cần giữ nguyên, bạn có thể co giãn workload theo chiều dọc dựa trên
kích thước cluster bằng
[_Cluster Proportional Vertical Autoscaler_](https://github.com/kubernetes-sigs/cluster-proportional-vertical-autoscaler).
Dự án này **hiện đang ở giai đoạn beta** và có thể tìm thấy trên GitHub.

Trong khi Cluster Proportional Autoscaler co giãn số lượng replica của một workload,
thì Cluster Proportional Vertical Autoscaler điều chỉnh lượng tài nguyên yêu cầu
(resource requests) của một workload (ví dụ một Deployment hoặc DaemonSet)
dựa trên số lượng node và/hoặc số core trong cluster.

### Tự động co giãn theo sự kiện (Event driven Autoscaling)

Bạn cũng có thể co giãn workload dựa trên các sự kiện (event), ví dụ bằng
[_Kubernetes Event Driven Autoscaler_ (**KEDA**)](https://keda.sh/).

KEDA là một dự án đã tốt nghiệp (graduated) của CNCF, cho phép bạn co giãn workload
dựa trên số lượng sự kiện cần xử lý, ví dụ số lượng thông điệp (message) trong một
hàng đợi (queue). Có sẵn rất nhiều adapter cho các nguồn sự kiện khác nhau để bạn lựa chọn.

### Tự động co giãn theo lịch (Autoscaling based on schedules)

Một chiến lược khác để co giãn workload là **lên lịch** cho các thao tác co giãn,
ví dụ nhằm giảm mức tiêu thụ tài nguyên trong các khung giờ thấp điểm.

Tương tự tự động co giãn theo sự kiện, hành vi này có thể đạt được bằng cách dùng KEDA
kết hợp với [`Cron` scaler](https://keda.sh/docs/latest/scalers/cron/) của nó.
`Cron` scaler cho phép bạn định nghĩa lịch (và múi giờ) để co giãn workload ra hoặc vào.

## Co giãn hạ tầng cluster (Scaling cluster infrastructure)

Nếu việc co giãn workload là chưa đủ để đáp ứng nhu cầu, bạn cũng có thể co giãn
chính hạ tầng cluster của mình.

Co giãn hạ tầng cluster thường có nghĩa là thêm hoặc bớt node.
Đọc [Tự động co giãn node (Node autoscaling)](https://kubernetes.io/docs/concepts/cluster-administration/node-autoscaling/)
để biết thêm thông tin.

## Tiếp theo (What's next)

- Tìm hiểu thêm về co giãn theo chiều ngang
  - [Co giãn một StatefulSet](https://kubernetes.io/docs/tasks/run-application/scale-stateful-set/)
  - [Hướng dẫn thực hành HorizontalPodAutoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
- [Thay đổi kích thước tài nguyên container tại chỗ](https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/)
- [Tự động co giãn Service DNS trong một cluster](https://kubernetes.io/docs/tasks/administer-cluster/dns-horizontal-autoscaling/)
- Tìm hiểu về [tự động co giãn node (Node autoscaling)](https://kubernetes.io/docs/concepts/cluster-administration/node-autoscaling/)

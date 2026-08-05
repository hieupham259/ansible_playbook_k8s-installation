# Vòng đời của PodGroup (PodGroup Lifecycle)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/podgroup-api/lifecycle/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Một [PodGroup](https://kubernetes.io/docs/concepts/workloads/podgroup-api/) được lập lịch
như một đơn vị và được bảo vệ khỏi việc bị xóa sớm trong khi các Pod của nó vẫn đang chạy.

## Quyền sở hữu và vòng đời (Ownership and lifecycle)

Các `PodGroup` được sở hữu bởi workload controller đã tạo ra chúng (ví dụ, một Job)
thông qua cơ chế `ownerReferences` tiêu chuẩn. Khi đối tượng sở hữu bị xóa, các `PodGroup`
sẽ tự động được thu gom (garbage collected).

Tên của `PodGroup` phải là duy nhất trong một namespace và phải là
[DNS subdomain](https://kubernetes.io/docs/concepts/overview/working-with-objects/names#dns-subdomain-names) hợp lệ.

## Thứ tự tạo (Creation ordering)

Các controller phải tạo các đối tượng theo thứ tự sau:

1. `Workload` — mẫu chính sách lập lịch.
2. `PodGroup` — thực thể (instance) runtime.
3. `Pods` — với `spec.schedulingGroup.podGroupName` trỏ đến `PodGroup`.

Nếu một `PodGroup` chứa `podGroupTemplateRef` trỏ đến một `Workload` không
tồn tại (hoặc đang bị xóa), API server sẽ từ chối yêu cầu tạo `PodGroup` đó.
`Workload` được tham chiếu phải tồn tại trước thì `PodGroup` mới có thể được tạo.

Nếu một `Pod` tham chiếu đến một `PodGroup` chưa tồn tại, `Pod` đó sẽ ở trạng thái pending.
Scheduler sẽ tự động đưa `Pod` vào hàng đợi lập lịch ngay khi `PodGroup` được tạo.

## Bảo vệ khi xóa (Deletion protection)

Một `PodGroup` không thể bị xóa hoàn toàn trong khi bất kỳ Pod nào của nó vẫn đang chạy.
Một finalizer chuyên dụng bảo đảm rằng việc xóa bị chặn cho đến khi tất cả các `Pod`
tham chiếu đến `PodGroup` đã đạt đến pha kết thúc (`Succeeded` hoặc `Failed`).

## PodGroup do controller quản lý và do người dùng quản lý (Controller-managed and user-managed PodGroups)

Trong hầu hết các trường hợp, các workload controller (ví dụ, Job) tự động tạo các `PodGroup`
(controller-managed — do controller quản lý). Controller xác định `podGroupName` cho mỗi Pod
tại thời điểm tạo, tương tự cách một `DaemonSet` đặt node affinity cho từng Pod.

Nếu bạn cần kiểm soát nhiều hơn việc đặt tên và vòng đời, bạn có thể tạo trực tiếp các đối tượng `PodGroup` và tự đặt
`spec.schedulingGroup.podGroupName` trong các Pod template của mình
(user-managed — do người dùng quản lý). Cách này cho bạn toàn quyền kiểm soát việc tạo và đặt tên `PodGroup`.

## Giới hạn (Limitations)

* Tất cả các Pod trong một `PodGroup` phải dùng cùng một `.spec.schedulerName`.
  Nếu phát hiện sự không khớp, scheduler sẽ từ chối tất cả các Pod trong nhóm với trạng thái không thể lập lịch (unschedulable).
* Trường `spec.schedulingPolicy.gang.minCount` trên một PodGroup là bất biến (immutable).
  Một khi đã tạo, bạn không thể thay đổi số lượng Pod tối thiểu phải có thể lập lịch được để nhóm được chấp nhận (admit).
* Trường `spec.schedulingGroup` trên một Pod là bất biến.
  Một khi đã được đặt, một Pod không thể chuyển sang một PodGroup khác.
* Số lượng `PodGroupTemplate` tối đa trong một `Workload` là 8.
* Condition `PodGroupScheduled` chỉ phản ánh kết quả của lần thử lập lịch
  ban đầu. Một khi condition này được đặt thành `True`, scheduler sẽ không cập nhật nó
  nếu sau đó các Pod bị lỗi, bị trục xuất (evict), hoặc ngừng chạy.

## Tiếp theo (What's next)

* Tìm hiểu tổng quan và cấu trúc của [PodGroup API](https://kubernetes.io/docs/concepts/workloads/podgroup-api/).
* Tìm hiểu về [Workload API](https://kubernetes.io/docs/concepts/workloads/workload-api/) — API cung cấp các `PodGroupTemplate`.
* Xem cách các Pod tham chiếu đến PodGroup của chúng thông qua trường [scheduling group](https://kubernetes.io/docs/concepts/workloads/pods/scheduling-group/).
* Hiểu về thuật toán [gang scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/).
* Đọc [các chính sách lập lịch của PodGroup](https://kubernetes.io/docs/concepts/workloads/workload-api/policies/) để biết chi tiết về `basic` và `gang`.

# Workload API

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/workload-api/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Tài nguyên API `Workload` định nghĩa các yêu cầu lập lịch (scheduling) và cấu trúc của một
ứng dụng gồm nhiều Pod. Trong khi các workload controller như [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
quản lý trạng thái lúc chạy (runtime state) của ứng dụng, `Workload` chỉ định cách các nhóm `Pod`
cần được lập lịch. Job controller là controller tích hợp sẵn duy nhất tạo ra các đối tượng
[PodGroup](https://kubernetes.io/docs/concepts/workloads/podgroup-api/) từ
`PodGroupTemplates` của `Workload` tại thời điểm chạy.

## Workload là gì? (What is a Workload?)

Tài nguyên API Workload thuộc nhóm API (API group) `scheduling.k8s.io/v1alpha2`,
và cluster của bạn phải bật API group đó, cũng như bật
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`GenericWorkload`, trước khi bạn có thể sử dụng API này.

`Workload` là một mẫu chính sách (policy template) tĩnh, tồn tại lâu dài. Nó định nghĩa
những chính sách lập lịch nào cần được áp dụng cho các nhóm Pod, nhưng bản thân nó không theo dõi
trạng thái lúc chạy. Trạng thái lập lịch lúc chạy được duy trì bởi các đối tượng
[PodGroup](https://kubernetes.io/docs/concepts/workloads/podgroup-api/),
do các controller tạo ra từ `PodGroupTemplates` của `Workload`.

## Cấu trúc API (API structure)

Một `Workload` gồm hai trường: một danh sách `PodGroupTemplates` và một tham chiếu controller
(tùy chọn). Toàn bộ spec của `Workload` là bất biến (immutable) sau khi tạo: bạn không thể
sửa đổi các template hiện có, thêm template mới, hay xóa template khỏi `podGroupTemplates`.

### PodGroupTemplates

Danh sách `spec.podGroupTemplates` định nghĩa các thành phần riêng biệt của workload.
Ví dụ, một job học máy (machine learning) có thể có một template `driver` và một template `worker`.

Mỗi mục trong `podGroupTemplates` phải có:
1. Một `name` duy nhất, được dùng để tham chiếu đến template trong `spec.podGroupTemplateRef` của `PodGroup`.
2. Một [chính sách lập lịch](./79-workload-policies-vi.md) (`basic` hoặc `gang`).

Nếu [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) [`WorkloadAwarePreemption`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#WorkloadAwarePreemption) được bật, mỗi mục trong `podGroups` cũng có thể có [độ ưu tiên và chế độ gián đoạn](./78-workload-disruption-priority-vi.md).

Số lượng PodGroupTemplate tối đa trong một Workload là 8.

```yaml
apiVersion: scheduling.k8s.io/v1alpha2
kind: Workload
metadata:
  name: training-job-workload
  namespace: some-ns
spec:
  controllerRef:
    apiGroup: batch
    kind: Job
    name: training-job
  podGroupTemplates:
  - name: workers
    schedulingPolicy:
      gang:
        # Gang chỉ có thể được lập lịch nếu 4 pod chạy được cùng lúc
        minCount: 4
    priorityClassName: high-priority # Chỉ áp dụng khi bật feature gate WorkloadAwarePreemption
    disruptionMode: PodGroup # Chỉ áp dụng khi bật feature gate WorkloadAwarePreemption
```

Khi một workload controller tạo `PodGroup` từ một trong các template này, nó sao chép
`schedulingPolicy` vào spec riêng của `PodGroup`. Các thay đổi đối với `Workload` chỉ ảnh hưởng
đến những `PodGroup` mới được tạo, không ảnh hưởng đến các `PodGroup` hiện có.

### Tham chiếu đến đối tượng điều khiển workload (Referencing a workload controlling object)

Trường `controllerRef` liên kết Workload trở lại đối tượng cấp cao cụ thể định nghĩa ứng dụng,
chẳng hạn như một [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) hoặc một CRD tùy chỉnh.
Điều này hữu ích cho khả năng quan sát (observability) và các công cụ hỗ trợ.
Dữ liệu này không được dùng để lập lịch hay quản lý Workload.

## Gang scheduling với Job (Gang scheduling with Jobs)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Khi feature gate
[`WorkloadWithJob`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
được bật, [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) controller
tự động tạo các đối tượng Workload và PodGroup cho các Job dạng indexed chạy song song có
`.spec.parallelism` bằng `.spec.completions`. `minCount` của chính sách gang
được đặt bằng parallelism của Job, do đó tất cả các Pod phải có thể được lập lịch cùng nhau
trước khi bất kỳ Pod nào trong số đó được gán (bind) vào node.

Đây là con đường tích hợp sẵn để sử dụng gang scheduling với Job.
Bạn không cần tự tạo các đối tượng Workload hay PodGroup vì Job controller
xử lý việc này một cách tự động. Các workload controller khác (chẳng hạn
JobSet) có thể tự quản lý các đối tượng Workload và PodGroup của riêng chúng một cách độc lập.

## Tiếp theo (What's next)

* Tìm hiểu về [các chính sách lập lịch PodGroup](./79-workload-policies-vi.md).
* Xem cách các PodGroup được tạo từ Workload trong phần tổng quan [PodGroup API](https://kubernetes.io/docs/concepts/workloads/podgroup-api/).
* Đọc về cách các Pod tham chiếu đến PodGroup của chúng thông qua trường [nhóm lập lịch (scheduling group)](./59-scheduling-group-vi.md).
* Tìm hiểu về [Lập lịch workload nhận biết topology](./80-workload-topology-scheduling-vi.md).
* Hiểu thuật toán [gang scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/).

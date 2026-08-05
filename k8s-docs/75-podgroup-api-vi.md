# PodGroup API

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/podgroup-api/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

PodGroup là một đối tượng runtime đại diện cho một nhóm các Pod được lập lịch cùng nhau
như một đơn vị duy nhất. Trong khi [Workload API](https://kubernetes.io/docs/concepts/workloads/workload-api/)
định nghĩa các mẫu (template) chính sách lập lịch, PodGroup là đối tượng runtime tương ứng,
mang theo cả chính sách lẫn trạng thái lập lịch cho một thực thể (instance) cụ thể của nhóm đó.

## PodGroup là gì? (What is a PodGroup?)

Tài nguyên PodGroup API là một phần của API group `scheduling.k8s.io/v1alpha2`,
và cluster của bạn phải bật API group đó, cũng như
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) `GenericWorkload`,
trước khi bạn có thể sử dụng API này.

PodGroup là một đơn vị lập lịch khép kín. Nó định nghĩa nhóm các Pod cần được lập lịch
cùng nhau, mang theo chính sách lập lịch chi phối việc sắp đặt (placement), và ghi lại
trạng thái runtime của quyết định lập lịch đó.

## Cấu trúc API (API structure)

Một PodGroup gồm phần `spec` định nghĩa hành vi lập lịch mong muốn và
phần `status` phản ánh trạng thái lập lịch hiện tại.

### Chính sách lập lịch (Scheduling policy)

Mỗi PodGroup mang một [chính sách lập lịch](https://kubernetes.io/docs/concepts/workloads/workload-api/policies/)
(`basic` hoặc `gang`) trong `spec.schedulingPolicy`. Khi một workload controller tạo
PodGroup, chính sách này được sao chép từ PodGroupTemplate của Workload tại thời điểm tạo.
Với các PodGroup độc lập (standalone), bạn đặt chính sách này trực tiếp.

```yaml
spec:
  schedulingPolicy:
    gang:
      minCount: 4
```

### Tham chiếu template (Template reference)

Trường tùy chọn `spec.podGroupTemplateRef` liên kết PodGroup ngược về PodGroupTemplate
trong Workload mà từ đó nó được tạo ra. Điều này hữu ích cho khả năng quan sát (observability) và các công cụ hỗ trợ.

```yaml
spec:
  podGroupTemplateRef:
    workload:
      workloadName: training-policy
      podGroupTemplateName: worker
```

### Yêu cầu thiết bị DRA cho một PodGroup (Requesting DRA devices for a PodGroup)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Các thiết bị (device) khả dụng thông qua cơ chế cấp phát tài nguyên động
(Dynamic Resource Allocation — DRA)
có thể được một PodGroup yêu cầu thông qua trường `spec.resourceClaims` của nó:

```yaml
apiVersion: scheduling.k8s.io/v1alpha2
kind: PodGroup
metadata:
  name: training-group
  namespace: some-ns
spec:
  ...
  resourceClaims:
  - name: pg-claim
    resourceClaimName: my-pg-claim
  - name: pg-claim-template
    resourceClaimTemplateName: my-pg-template
```

Các ResourceClaim gắn với PodGroup có thể được chia sẻ bởi tất cả các Pod thuộc
nhóm đó. Vì chỉ cần một tham chiếu đến PodGroup trong `status.reservedFor` của
ResourceClaim thay vì từng Pod riêng lẻ, nên bất kỳ số lượng Pod nào trong cùng một
PodGroup đều có thể dùng chung một ResourceClaim. Các ResourceClaim cũng có thể được
sinh ra từ các ResourceClaimTemplate cho mỗi PodGroup, cho phép các thiết bị được
cấp phát cho mỗi ResourceClaim được sinh ra đó được chia sẻ bởi các Pod trong từng PodGroup.

Để biết thêm chi tiết và một ví dụ đầy đủ hơn, xem
[tài liệu DRA](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#workload-resource-claims).

### Trạng thái (Status)

Scheduler cập nhật `status.conditions` để báo cáo liệu nhóm đã được lập lịch
thành công hay chưa. Condition chính là `PodGroupScheduled`, có giá trị `True`
khi tất cả các Pod bắt buộc đã được sắp đặt xong và `False` khi việc lập lịch thất bại.

> **Ghi chú:**
> Condition `PodGroupScheduled` chỉ phản ánh quyết định lập lịch ban đầu.
> Scheduler không cập nhật nó nếu sau đó các Pod bị lỗi hoặc bị trục xuất (evict). Xem
> [Giới hạn](https://kubernetes.io/docs/concepts/workloads/podgroup-api/lifecycle/#limitations)
> để biết chi tiết.

Xem trang [Vòng đời của PodGroup](https://kubernetes.io/docs/concepts/workloads/podgroup-api/lifecycle/#podgroup-status)
để biết danh sách đầy đủ các condition và lý do (reason).

## Tạo một PodGroup (Creating a PodGroup)

Tài nguyên PodGroup API là một phần của API group `scheduling.k8s.io/v1alpha2`
(và cluster của bạn phải bật API group đó, cũng như
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) `GenericWorkload`,
trước khi bạn có thể sử dụng API này).

Manifest sau đây tạo một PodGroup với chính sách gang scheduling, yêu cầu
ít nhất 4 Pod phải có thể lập lịch được đồng thời:

```yaml
apiVersion: scheduling.k8s.io/v1alpha2
kind: PodGroup
metadata:
  name: training-worker-0
  namespace: default
spec:
  schedulingPolicy:
    gang:
      minCount: 4
```

Bạn có thể xem các PodGroup trong cluster của mình:

```shell
kubectl get podgroups
```

Để xem status đầy đủ bao gồm các condition lập lịch:

```shell
kubectl describe podgroup training-worker-0
```

## Cách các thành phần khớp với nhau (How it fits together)

Mối quan hệ giữa controller, Workload, PodGroup và Pod tuân theo mẫu sau:

1. Workload controller tạo một Workload định nghĩa các PodGroupTemplate cùng với các chính sách lập lịch.
2. Với mỗi thực thể runtime, controller tạo một PodGroup từ một trong các PodGroupTemplate của Workload.
3. Controller tạo các Pod tham chiếu đến PodGroup
   thông qua trường `spec.schedulingGroup.podGroupName`.

Controller của [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) hiện là
workload controller tích hợp sẵn duy nhất tuân theo mẫu này.
Các controller tùy chỉnh có thể triển khai cùng luồng xử lý này cho các loại workload của riêng chúng.

```yaml
apiVersion: scheduling.k8s.io/v1alpha2
kind: Workload
metadata:
  name: training-policy
spec:
  podGroupTemplates:
  - name: worker
    schedulingPolicy:
      gang:
        minCount: 4
---
apiVersion: scheduling.k8s.io/v1alpha2
kind: PodGroup
metadata:
  name: training-worker-0
spec:
  podGroupTemplateRef:
    workload:
      workloadName: training-policy
      podGroupTemplateName: worker
  schedulingPolicy:
    gang:
      minCount: 4
---
apiVersion: v1
kind: Pod
metadata:
  name: worker-0
spec:
  schedulingGroup:
    podGroupName: training-worker-0
  containers:
  - name: ml-worker
    image: training:v1
```

Workload đóng vai trò là một định nghĩa chính sách tồn tại lâu dài, trong khi các PodGroup
xử lý trạng thái runtime tạm thời cho từng thực thể. Sự tách biệt này có nghĩa là
các cập nhật status của từng PodGroup riêng lẻ không tranh chấp trên đối tượng Workload dùng chung.

## Tiếp theo (What's next)

* Tìm hiểu chi tiết về [vòng đời của PodGroup](https://kubernetes.io/docs/concepts/workloads/podgroup-api/lifecycle/).
* Đọc về [Workload API](https://kubernetes.io/docs/concepts/workloads/workload-api/) — API cung cấp các PodGroupTemplate.
* Xem cách các Pod tham chiếu đến PodGroup của chúng thông qua trường [scheduling group](https://kubernetes.io/docs/concepts/workloads/pods/scheduling-group/).
* Hiểu về thuật toán [gang scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/).

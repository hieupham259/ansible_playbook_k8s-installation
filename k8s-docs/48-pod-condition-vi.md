# Các Condition của Pod (Pod Conditions)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/pods/pod-condition/>

Trong Kubernetes, nhiều đối tượng có các _condition_ (điều kiện/trạng thái).
Condition là các dấu mốc cho một khía cạnh nào đó của trạng thái thực tế của thứ mà đối tượng đại diện.
Pod có các condition, và các Pod condition của Kubernetes là một khía cạnh quan trọng giúp các controller
(và những người đang xử lý sự cố) hiểu được sức khỏe của một Pod.

[Phase](./47-pod-lifecycle-vi.md#pod-phase) của một Pod cung cấp một bản tóm tắt ở mức cao
về vị trí của Pod trong vòng đời của nó, nhưng một giá trị đơn lẻ không thể nắm bắt được bức tranh
toàn cảnh. Ví dụ, một Pod có thể đang ở phase `Running` nhưng chưa sẵn sàng phục vụ lưu lượng (traffic).
Các Pod condition bổ trợ cho phase bằng cách theo dõi độc lập nhiều khía cạnh của trạng thái Pod,
chẳng hạn như nó đã được lập lịch (schedule) hay chưa, các container của nó đã sẵn sàng hay chưa,
một thao tác thay đổi kích thước (resize) có đang diễn ra hay không, hay Pod có sắp bị gián đoạn
(disrupt) do một taint hay không.

## Cấu trúc của một Pod condition (Structure of a Pod condition) {#structure-of-a-pod-condition}

Trạng thái (status) của một Pod bao gồm một mảng các
[PodCondition](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#podcondition-v1-core)
cho biết Pod đã vượt qua những điểm kiểm tra (checkpoint) nhất định hay chưa.

Mỗi phần tử của mảng PodCondition có các trường sau:

| Tên trường           | Mô tả                                                                                                |
|:---------------------|:-----------------------------------------------------------------------------------------------------|
| `type`               | Tên của Pod condition này.                                                                           |
| `status`             | Cho biết condition đó có áp dụng hay không, với các giá trị khả dĩ `"True"`, `"False"`, hoặc `"Unknown"`. |
| `lastProbeTime`      | Dấu thời gian của lần probe gần nhất đối với Pod condition này.                                       |
| `lastTransitionTime` | Dấu thời gian của lần gần nhất Pod chuyển từ trạng thái này sang trạng thái khác.                     |
| `reason`             | Chuỗi văn bản dạng máy đọc được, viết theo UpperCamelCase, cho biết lý do của lần chuyển trạng thái gần nhất của condition. |
| `message`            | Thông điệp dạng người đọc được, cho biết chi tiết về lần chuyển trạng thái gần nhất.                  |
| `observedGeneration` | Giá trị `.metadata.generation` của Pod tại thời điểm condition được ghi nhận. Xem [Pod generation](https://kubernetes.io/docs/concepts/workloads/pods/#pod-generation). |

*Các trường của một PodCondition*

## Các Pod condition có sẵn (Built-in Pod conditions) {#built-in-pod-conditions}

Kubernetes quản lý các Pod condition sau:

[Các condition vòng đời](#lifecycle-pod-conditions): được đặt khi Pod tiến triển qua vòng đời của nó, đại thể theo thứ tự sau:
`PodScheduled`, `PodReadyToStartContainers`, `Initialized`, `ContainersReady`, `Ready`.

[Các condition khác](#other-pod-conditions): được đặt để phản ứng với các thao tác hoặc sự kiện cụ thể:
`DisruptionTarget`, `PodResizePending`, `PodResizeInProgress`.

Ngoài các condition có sẵn ở trên, bạn có thể định nghĩa các condition tùy chỉnh
bằng [Pod readiness gate](#enhanced-pod-readiness).

## Các Pod condition vòng đời (Lifecycle Pod conditions) {#lifecycle-pod-conditions}

Khi một Pod tiến triển qua vòng đời của nó, kubelet đặt các condition sau, đại thể theo thứ tự này:

1. `PodScheduled`: Pod đã được lập lịch lên một node.
1. `PodReadyToStartContainers`: Pod sandbox đã được tạo thành công và mạng đã được cấu hình. Sandbox và mạng được thiết lập bởi container runtime và CNI plugin.
1. `Initialized`: tất cả các [init container](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) đã hoàn tất thành công. Với một Pod không có init container, condition này được đặt là `True` trước khi tạo sandbox.
1. `ContainersReady`: tất cả các container trong Pod đã sẵn sàng. Độ sẵn sàng của một container được xác định bởi [readiness probe](https://kubernetes.io/docs/concepts/workloads/pods/probes/#readiness-probe) của nó, nếu được cấu hình.
1. `Ready`: Pod có khả năng phục vụ các yêu cầu và nên được thêm vào các nhóm cân bằng tải (load balancing pool) của tất cả các [Service](https://kubernetes.io/docs/concepts/services-networking/service/) khớp với nó. Các Pod không ở trạng thái `Ready` sẽ bị loại khỏi các endpoint của Service.

> **Ghi chú:**
>
> Condition `Ready` phụ thuộc vào nhiều thứ hơn là chỉ `ContainersReady`. Nếu Pod chỉ định `readinessGates`, tất cả các condition tùy chỉnh đó cũng phải là `True` thì Pod mới được coi là `Ready`. Xem [Độ sẵn sàng của Pod](#enhanced-pod-readiness) để biết chi tiết.

Bạn có thể xem các condition của một Pod bằng kubectl:

```shell
kubectl get pod <tên-pod> -o yaml
```

Dưới đây là hình dạng của `status.conditions` đối với một Pod đang chạy:

```yaml
status:
  conditions:
    - type: PodScheduled
      status: "True"
      lastProbeTime: null
      lastTransitionTime: "2026-03-29T08:52:21Z"
      observedGeneration: 1
    - type: PodReadyToStartContainers
      status: "True"
      lastProbeTime: null
      lastTransitionTime: "2026-04-11T06:02:16Z"
      observedGeneration: 1
    - type: Initialized
      status: "True"
      lastProbeTime: null
      lastTransitionTime: "2026-03-29T08:52:21Z"
      observedGeneration: 1
    - type: ContainersReady
      status: "True"
      lastProbeTime: null
      lastTransitionTime: "2026-04-11T06:02:45Z"
      observedGeneration: 1
    - type: Ready
      status: "True"
      lastProbeTime: null
      lastTransitionTime: "2026-04-11T06:02:45Z"
      observedGeneration: 1
```

### PodReadyToStartContainers {#pod-ready-to-start-containers}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.29 [beta]`

> **Ghi chú:**
>
> Trong giai đoạn phát triển ban đầu, condition này có tên là `PodHasNetwork`.

Sau khi một Pod được lập lịch lên một node, nó cần được kubelet chấp nhận (admit)
và cần có các volume lưu trữ cần thiết được mount. Một khi các giai đoạn này hoàn tất,
kubelet phối hợp với một container runtime
(sử dụng Container Runtime Interface (CRI))
để thiết lập một runtime sandbox và cấu hình mạng cho Pod.
Nếu feature gate `PodReadyToStartContainersCondition` được bật
(nó được bật mặc định cho Kubernetes v1.36),
condition `PodReadyToStartContainers` sẽ được thêm vào trường `status.conditions` của Pod.

Condition `PodReadyToStartContainers` được kubelet đặt là `False`
khi nó phát hiện một Pod không có runtime sandbox với mạng đã được cấu hình. Điều này xảy ra trong các kịch bản sau:

- Ở giai đoạn đầu vòng đời của Pod, khi kubelet chưa bắt đầu thiết lập sandbox cho Pod bằng container runtime.
- Ở giai đoạn sau trong vòng đời của Pod, khi Pod sandbox đã bị hủy do một trong hai nguyên nhân:
  - node khởi động lại, mà Pod không bị trục xuất (evict)
  - với các container runtime dùng máy ảo để cách ly, máy ảo sandbox của Pod khởi động lại, đòi hỏi phải tạo một sandbox mới và cấu hình mạng container mới.

Condition `PodReadyToStartContainers` được kubelet đặt là `True` sau khi runtime plugin hoàn tất thành công việc tạo sandbox và cấu hình mạng cho Pod. Kubelet có thể bắt đầu kéo (pull) các container image và tạo container sau khi condition `PodReadyToStartContainers` đã được đặt là `True`.

Với một Pod có init container, kubelet đặt condition `Initialized` là `True` sau khi các init container đã hoàn tất thành công (điều này xảy ra sau khi runtime plugin tạo sandbox và cấu hình mạng thành công). Với một Pod không có init container, kubelet đặt condition `Initialized` là `True` trước khi việc tạo sandbox và cấu hình mạng bắt đầu.

## Các Pod condition khác (Other Pod conditions) {#other-pod-conditions}

Các condition sau không thuộc tiến trình vòng đời thông thường của Pod.
Chúng được đặt để phản ứng với các thao tác hoặc sự kiện cụ thể.

### DisruptionTarget {#disruption-target}

Một condition `DisruptionTarget` dành riêng cho Pod được thêm vào để cho biết rằng
Pod sắp bị xóa do một sự gián đoạn (disruption).
Trường `reason` của condition này còn cho biết
một trong các lý do sau cho việc kết thúc Pod:

`PreemptionByScheduler`
: Pod sắp bị scheduler chiếm chỗ (preempt) để nhường chỗ cho một Pod mới có độ ưu tiên (priority) cao hơn. Để biết thêm thông tin, xem [Pod priority preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/).

`DeletionByTaintManager`
: Pod sắp bị Taint Manager (một phần của node lifecycle controller bên trong `kube-controller-manager`) xóa do một taint `NoExecute` mà Pod không dung thứ (tolerate); xem các đợt trục xuất (eviction) dựa trên taint.

`EvictionByEvictionAPI`
: Pod đã được đánh dấu để trục xuất bằng Kubernetes API.

`DeletionByPodGC`
: Pod, do gắn với một Node không còn tồn tại, sắp bị xóa bởi cơ chế [thu gom rác cho Pod](./47-pod-lifecycle-vi.md#pod-garbage-collection).

`TerminationByKubelet`
: Pod đã bị kubelet kết thúc, do một trong các nguyên nhân: trục xuất vì áp lực tài nguyên trên node (node pressure eviction),
  [tắt node nhẹ nhàng (graceful node shutdown)](https://kubernetes.io/docs/concepts/architecture/nodes/#graceful-node-shutdown),
  hoặc chiếm chỗ để nhường cho [các Pod quan trọng của hệ thống](https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/).

Trong mọi kịch bản gián đoạn khác, chẳng hạn trục xuất do vượt quá
[giới hạn tài nguyên container của Pod](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/),
các Pod không nhận condition `DisruptionTarget` vì các gián đoạn đó nhiều khả năng
do chính Pod gây ra và sẽ tái diễn khi thử lại.

> **Ghi chú:**
>
> Một sự gián đoạn Pod có thể bị gián đoạn giữa chừng. Control plane có thể thử tiếp tục
> gián đoạn chính Pod đó, nhưng điều này không được bảo đảm. Kết quả là,
> condition `DisruptionTarget` có thể được thêm vào một Pod, nhưng Pod đó sau cùng có thể không
> thực sự bị xóa. Trong tình huống như vậy, sau một khoảng thời gian,
> Pod disruption condition sẽ bị xóa bỏ.

Cùng với việc dọn dẹp các Pod, bộ thu gom rác cho Pod (PodGC) cũng đánh dấu chúng là thất bại nếu chúng đang ở một
phase chưa kết thúc (xem thêm [Thu gom rác cho Pod](./47-pod-lifecycle-vi.md#pod-garbage-collection)).

Khi dùng Job (hoặc CronJob), bạn có thể muốn dùng các Pod disruption condition này như một phần của
[chính sách xử lý lỗi Pod (Pod failure policy)](https://kubernetes.io/docs/concepts/workloads/controllers/job#pod-failure-policy) của Job.

Để biết thêm chi tiết, xem [Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/).

### PodResizePending và PodResizeInProgress {#pod-resize-conditions}

Kubelet cập nhật các condition trạng thái của Pod để cho biết trạng thái của một yêu cầu thay đổi kích thước (resize):

- `type: PodResizePending`: Kubelet không thể đáp ứng yêu cầu ngay lập tức. Trường `message` cung cấp lời giải thích lý do.
  - `reason: Infeasible`: Yêu cầu resize là bất khả thi trên node hiện tại (ví dụ, yêu cầu nhiều tài nguyên hơn mức node có).
  - `reason: Deferred`: Yêu cầu resize hiện chưa khả thi, nhưng có thể trở nên khả thi sau này (ví dụ nếu một Pod khác bị gỡ bỏ). Kubelet sẽ thử lại thao tác resize.
- `type: PodResizeInProgress`: Kubelet đã chấp nhận yêu cầu resize và đã cấp phát tài nguyên, nhưng các thay đổi vẫn đang được áp dụng. Việc này thường diễn ra ngắn nhưng có thể lâu hơn tùy loại tài nguyên và hành vi của runtime. Mọi lỗi trong quá trình thực thi được báo cáo trong trường `message` (cùng với `reason: Error`).

Nếu yêu cầu resize ở trạng thái _Deferred_ (bị hoãn), kubelet sẽ định kỳ thử lại thao tác resize, ví dụ khi một Pod khác bị gỡ bỏ hoặc thu nhỏ quy mô.

Để biết thêm chi tiết về resize Pod, xem [Thay đổi tài nguyên CPU và bộ nhớ gán cho Container](https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/).

## Độ sẵn sàng nâng cao của Pod (Enhanced Pod readiness) {#enhanced-pod-readiness}

Ứng dụng của bạn có thể chèn phản hồi hoặc tín hiệu bổ sung vào `.status` của Pod;
điều này được gọi là _enhanced Pod readiness_ (độ sẵn sàng nâng cao của Pod).
Để dùng tính năng này, hãy đặt `readinessGates` trong `spec` của Pod để chỉ định một danh sách
các condition bổ sung mà kubelet đánh giá cho độ sẵn sàng của Pod.
Sau đó bạn triển khai, hoặc cài đặt, một controller quản lý các condition tùy chỉnh này,
và kubelet dùng chúng như một đầu vào bổ sung để quyết định Pod có sẵn sàng hay không.

Readiness gate được xác định bởi trạng thái hiện tại của các trường `status.condition` của Pod.
Nếu Kubernetes không tìm thấy condition như vậy trong trường `status.conditions` của một Pod, trạng thái của condition đó được mặc định là "`False`".

```yaml
kind: Pod
...
spec:
  readinessGates:
    - conditionType: "www.example.com/feature-1"
status:
  conditions:
    - type: Ready                              # một PodCondition có sẵn (built-in)
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
    - type: "www.example.com/feature-1"        # một PodCondition bổ sung
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
  containerStatuses:
    - containerID: docker://abcd...
      ready: true
...
```

Các Pod condition bạn thêm vào phải có tên tuân theo [định dạng khóa label](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#syntax-and-character-set) của Kubernetes.

### Trạng thái cho độ sẵn sàng của Pod (Status for Pod readiness) {#status-for-pod-readiness}

Để đặt các `status.conditions` này cho Pod, ứng dụng và các
operator nên dùng hành động `PATCH` trên status subresource của Pod. Bạn có thể dùng `kubectl patch`
với `--subresource=status`, hoặc một [thư viện client Kubernetes](https://kubernetes.io/docs/reference/using-api/client-libraries/) để viết
mã đặt các Pod condition tùy chỉnh cho độ sẵn sàng của Pod.

Với một Pod dùng các condition tùy chỉnh, Pod đó **chỉ** được đánh giá là sẵn sàng khi cả hai điều sau cùng đúng:

- Tất cả các container trong Pod đều sẵn sàng.
- Tất cả các condition được chỉ định trong `readinessGates` đều là `True`.

Khi các container của một Pod đã Ready nhưng ít nhất một condition tùy chỉnh bị thiếu hoặc là `False`,
kubelet đặt condition `Ready` của Pod thành `status: "False"` với `reason: ReadinessGatesNotReady`.

## Tiếp theo (What's next)

- Tìm hiểu về [Vòng đời của Pod](./47-pod-lifecycle-vi.md).
- Tìm hiểu về [Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/).
- Tìm hiểu về [container probe](https://kubernetes.io/docs/concepts/workloads/pods/probes/) và cách chúng ảnh hưởng đến độ sẵn sàng của Pod.
- Tìm hiểu cách [thay đổi tài nguyên Pod tại chỗ](https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/).

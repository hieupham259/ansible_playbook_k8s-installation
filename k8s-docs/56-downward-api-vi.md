# Downward API

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/pods/downward-api/>
>
> Có hai cách để phơi bày (expose) các trường của Pod và container cho một container đang chạy:
> biến môi trường, và dưới dạng các file được điền bởi một loại volume đặc biệt.
> Hai cách phơi bày các trường của Pod và container này được gọi chung là downward API.

Đôi khi việc một container có thông tin về chính nó là hữu ích, mà không cần gắn kết
(coupled) quá chặt với Kubernetes. _Downward API_ cho phép các container sử dụng thông
tin về chính chúng hoặc về cluster mà không cần dùng Kubernetes client hay API server.

Một ví dụ là một ứng dụng có sẵn giả định rằng một biến môi trường quen thuộc nào đó chứa
một định danh (identifier) duy nhất. Một khả năng là bọc (wrap) ứng dụng lại, nhưng cách
đó tẻ nhạt, dễ gây lỗi, và vi phạm mục tiêu gắn kết lỏng (low coupling). Một lựa chọn tốt
hơn là dùng tên của Pod làm định danh, và tiêm (inject) tên của Pod vào biến môi trường
quen thuộc đó.

Trong Kubernetes, có hai cách để phơi bày các trường của Pod và container cho một
container đang chạy:

* dưới dạng [biến môi trường](https://kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/)
* dưới dạng [các file trong một volume `downwardAPI`](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/)

Hai cách phơi bày các trường của Pod và container này được gọi chung là _downward API_.

## Các trường khả dụng (Available fields)

Chỉ một số trường của Kubernetes API là khả dụng thông qua downward API. Mục này liệt kê
những trường bạn có thể cung cấp.

Bạn có thể truyền thông tin từ các trường khả dụng ở cấp Pod bằng `fieldRef`. Ở cấp độ
API, `spec` của một Pod luôn định nghĩa ít nhất một
[Container](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Container).
Bạn có thể truyền thông tin từ các trường khả dụng ở cấp Container bằng
`resourceFieldRef`.

### Thông tin khả dụng qua `fieldRef` (Information available via `fieldRef`) {#downwardapi-fieldRef}

Với một số trường cấp Pod, bạn có thể cung cấp chúng cho container dưới dạng biến môi
trường hoặc dùng một volume `downwardAPI`. Các trường khả dụng qua cả hai cơ chế gồm:

`metadata.name`
: tên của Pod

`metadata.namespace`
: namespace của Pod

`metadata.uid`
: ID duy nhất của Pod

`metadata.annotations['<KEY>']`
: giá trị của annotation có tên `<KEY>` của Pod (ví dụ: `metadata.annotations['myannotation']`)

`metadata.labels['<KEY>']`
: giá trị dạng chữ của label có tên `<KEY>` của Pod (ví dụ: `metadata.labels['mylabel']`)

Các thông tin sau khả dụng qua biến môi trường **nhưng không khả dụng dưới dạng fieldRef
của volume downwardAPI**:

`spec.serviceAccountName`
: tên service account của Pod

`spec.nodeName`
: tên của node nơi Pod đang thực thi

`status.hostIP`
: địa chỉ IP chính của node mà Pod được gán vào

`status.hostIPs`
: các địa chỉ IP này là phiên bản dual-stack của `status.hostIP`, địa chỉ đầu tiên luôn giống với `status.hostIP`.

`status.podIP`
: địa chỉ IP chính của Pod (thường là địa chỉ IPv4 của nó)

`status.podIPs`
: các địa chỉ IP này là phiên bản dual-stack của `status.podIP`, địa chỉ đầu tiên luôn giống với `status.podIP`

Các thông tin sau khả dụng qua `fieldRef` của volume `downwardAPI`, **nhưng không khả
dụng dưới dạng biến môi trường**:

`metadata.labels`
: tất cả các label của Pod, được định dạng `label-key="escaped-label-value"` với mỗi label trên một dòng

`metadata.annotations`
: tất cả các annotation của Pod, được định dạng `annotation-key="escaped-annotation-value"` với mỗi annotation trên một dòng

### Thông tin khả dụng qua `resourceFieldRef` (Information available via `resourceFieldRef`) {#downwardapi-resourceFieldRef}

Các trường cấp container này cho phép bạn cung cấp thông tin về
[request và limit](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#requests-and-limits)
của các tài nguyên như CPU và bộ nhớ.

> **Ghi chú:**
>
> **TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [stable]`
>
> Tài nguyên CPU và bộ nhớ của container có thể được thay đổi kích thước (resize) trong
> khi container đang chạy. Nếu điều này xảy ra, volume downward API sẽ được cập nhật,
> nhưng các biến môi trường sẽ không được cập nhật trừ khi container khởi động lại.
> Xem [Thay đổi kích thước tài nguyên CPU và bộ nhớ đã gán cho Container](https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/)
> để biết thêm chi tiết.

`resource: limits.cpu`
: Limit CPU của container

`resource: requests.cpu`
: Request CPU của container

`resource: limits.memory`
: Limit bộ nhớ của container

`resource: requests.memory`
: Request bộ nhớ của container

`resource: limits.hugepages-*`
: Limit hugepages của container

`resource: requests.hugepages-*`
: Request hugepages của container

`resource: limits.ephemeral-storage`
: Limit ephemeral-storage (lưu trữ tạm thời) của container

`resource: requests.ephemeral-storage`
: Request ephemeral-storage của container

#### Thông tin dự phòng cho limit tài nguyên (Fallback information for resource limits)

Nếu limit CPU và bộ nhớ không được chỉ định cho một container, và bạn dùng downward API
để cố phơi bày thông tin đó, thì kubelet mặc định sẽ phơi bày giá trị có thể cấp phát
(allocatable) tối đa cho CPU và bộ nhớ dựa trên phép tính
[node allocatable](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#node-allocatable).

## Tiếp theo (What's next)

Bạn có thể đọc về [volume `downwardAPI`](https://kubernetes.io/docs/concepts/storage/volumes/#downwardapi).

Bạn có thể thử dùng downward API để phơi bày thông tin cấp container hoặc cấp Pod:
* dưới dạng [biến môi trường](https://kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/)
* dưới dạng [các file trong volume `downwardAPI`](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/)

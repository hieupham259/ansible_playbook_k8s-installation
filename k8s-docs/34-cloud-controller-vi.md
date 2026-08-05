# Cloud Controller Manager

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/architecture/cloud-controller/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.11 [beta]`

Các công nghệ hạ tầng cloud cho phép bạn chạy Kubernetes trên cloud công cộng (public),
riêng (private) và lai (hybrid). Kubernetes tin vào mô hình hạ tầng tự động hóa,
điều khiển bằng API mà không có sự gắn kết chặt (tight coupling) giữa các thành phần.

cloud-controller-manager là một thành phần thuộc control plane của Kubernetes,
nhúng logic điều khiển đặc thù cho từng cloud. Cloud controller manager cho phép bạn
kết nối cluster của mình với API của nhà cung cấp cloud (cloud provider), đồng thời
tách các thành phần tương tác với nền tảng cloud đó khỏi các thành phần chỉ tương tác
với cluster của bạn.

Bằng cách tách rời logic tương tác giữa Kubernetes và hạ tầng cloud bên dưới,
thành phần cloud-controller-manager giúp các nhà cung cấp cloud có thể phát hành
tính năng theo nhịp độ khác với dự án Kubernetes chính.

cloud-controller-manager được cấu trúc theo cơ chế plugin, cho phép các
nhà cung cấp cloud khác nhau tích hợp nền tảng của họ với Kubernetes.

## Thiết kế (Design)

![Các thành phần của Kubernetes](https://kubernetes.io/images/docs/components-of-kubernetes.svg)

Cloud controller manager chạy trong control plane dưới dạng một tập các tiến trình
được nhân bản (thường là các container trong Pod). Mỗi cloud-controller-manager
triển khai nhiều controller trong một tiến trình duy nhất.

> **Ghi chú:**
> Bạn cũng có thể chạy cloud controller manager như một addon của Kubernetes
> thay vì như một phần của control plane.

## Các chức năng của cloud controller manager (Cloud controller manager functions) {#functions-of-the-ccm}

Các controller bên trong cloud controller manager bao gồm:

### Node controller

Node controller chịu trách nhiệm cập nhật các đối tượng Node khi có máy chủ mới
được tạo trong hạ tầng cloud của bạn. Node controller lấy thông tin về các host
đang chạy bên trong phạm vi thuê (tenancy) của bạn với nhà cung cấp cloud.
Node controller thực hiện các chức năng sau:

1. Cập nhật đối tượng Node với định danh duy nhất (unique identifier) của máy chủ
   tương ứng, lấy được từ API của nhà cung cấp cloud.
1. Gắn annotation và label cho đối tượng Node với các thông tin đặc thù của cloud,
   chẳng hạn như region mà node được triển khai vào và các tài nguyên
   (CPU, bộ nhớ, v.v.) mà node đó có sẵn.
1. Lấy hostname và các địa chỉ mạng của node.
1. Xác minh tình trạng (health) của node. Trong trường hợp một node ngừng phản hồi,
   controller này kiểm tra với API của nhà cung cấp cloud để xem máy chủ đó
   đã bị vô hiệu hóa / xóa / chấm dứt (terminate) hay chưa.
   Nếu node đã bị xóa khỏi cloud, controller sẽ xóa đối tượng Node khỏi
   cluster Kubernetes của bạn.

Một số hiện thực (implementation) của nhà cung cấp cloud tách phần này thành
một node controller và một node lifecycle controller riêng biệt.

### Route controller

Route controller chịu trách nhiệm cấu hình các route trong cloud một cách phù hợp,
để các container trên những node khác nhau trong cluster Kubernetes của bạn
có thể giao tiếp với nhau.

Tùy theo nhà cung cấp cloud, route controller cũng có thể cấp phát các khối
địa chỉ IP cho mạng Pod.

### Service controller

Các Service tích hợp với các thành phần hạ tầng cloud như bộ cân bằng tải
(load balancer) được quản lý, địa chỉ IP, lọc gói tin mạng (network packet filtering)
và kiểm tra tình trạng đích (target health checking). Service controller tương tác
với các API của nhà cung cấp cloud để thiết lập bộ cân bằng tải và các thành phần
hạ tầng khác khi bạn khai báo một tài nguyên Service cần đến chúng.

## Phân quyền (Authorization)

Phần này phân tích các quyền truy cập mà cloud controller manager cần
trên nhiều đối tượng API khác nhau để thực hiện các thao tác của nó.

### Node controller {#authorization-node-controller}

Node controller chỉ làm việc với các đối tượng Node. Nó cần quyền truy cập đầy đủ
để đọc và sửa đổi các đối tượng Node.

`v1/Node`:

- get
- list
- create
- update
- patch
- watch
- delete

### Route controller {#authorization-route-controller}

Route controller lắng nghe sự kiện tạo đối tượng Node và cấu hình các route
một cách phù hợp. Nó cần quyền Get trên các đối tượng Node.

`v1/Node`:

- get

### Service controller {#authorization-service-controller}

Service controller theo dõi các sự kiện **create**, **update** và **delete**
của đối tượng Service, rồi cấu hình bộ cân bằng tải cho các Service đó
một cách phù hợp.

Để truy cập các Service, nó cần quyền **list** và **watch**. Để cập nhật các Service,
nó cần quyền **patch** và **update** trên subresource `status`.

`v1/Service`:

- list
- get
- watch
- patch
- update

### Các quyền khác (Others) {#authorization-miscellaneous}

Phần hiện thực lõi (core) của cloud controller manager cần quyền tạo các đối tượng
Event, và để đảm bảo hoạt động an toàn, nó cần quyền tạo các ServiceAccount.

`v1/Event`:

- create
- patch
- update

`v1/ServiceAccount`:

- create

ClusterRole RBAC cho cloud controller manager trông như sau:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cloud-controller-manager
rules:
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch
  - update
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - '*'
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - services/status
  verbs:
  - patch
  - update
- apiGroups:
  - ""
  resources:
  - serviceaccounts
  verbs:
  - create
- apiGroups:
  - ""
  resources:
  - persistentvolumes
  verbs:
  - get
  - list
  - update
  - watch
```

## Tiếp theo (What's next)

* [Quản trị Cloud Controller Manager](https://kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/#cloud-controller-manager)
  có hướng dẫn về cách chạy và quản lý cloud controller manager.

* Để nâng cấp một control plane có tính sẵn sàng cao (HA) sang dùng
  cloud controller manager, xem
  [Di chuyển control plane được nhân bản sang dùng Cloud Controller Manager](https://kubernetes.io/docs/tasks/administer-cluster/controller-manager-leader-migration/).

* Bạn muốn biết cách tự hiện thực cloud controller manager của riêng mình,
  hoặc mở rộng một dự án sẵn có?

  - Cloud controller manager dùng các interface của Go, cụ thể là interface
    `CloudProvider` được định nghĩa trong
    [`cloud.go`](https://github.com/kubernetes/cloud-provider/blob/release-1.21/cloud.go#L42-L69)
    từ [kubernetes/cloud-provider](https://github.com/kubernetes/cloud-provider),
    để cho phép cắm vào (plug in) các hiện thực từ bất kỳ cloud nào.
  - Hiện thực của các controller dùng chung được nêu trong tài liệu này
    (Node, Route và Service), cùng một số khung sườn (scaffolding) và interface
    cloudprovider dùng chung, thuộc phần lõi của Kubernetes. Các hiện thực đặc thù
    cho từng nhà cung cấp cloud nằm ngoài phần lõi của Kubernetes và hiện thực
    interface `CloudProvider`.
  - Để biết thêm thông tin về việc phát triển plugin, xem
    [Phát triển Cloud Controller Manager](https://kubernetes.io/docs/tasks/administer-cluster/developing-cloud-controller-manager/).

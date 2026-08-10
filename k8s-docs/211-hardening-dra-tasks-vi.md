# Tăng cường bảo mật cho Cấp phát tài nguyên động trong cluster của bạn (Harden Dynamic Resource Allocation in Your Cluster)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/hardening-dra/>

Trang này hướng dẫn quản trị viên cluster cách tăng cường bảo mật cho việc phân quyền
(authorization) đối với Cấp phát tài nguyên động (Dynamic Resource Allocation - DRA), tập trung
vào quyền truy cập tối thiểu (least-privilege) cho các cập nhật trạng thái (status) của
`ResourceClaim`.

## Trước khi bạn bắt đầu (Before you begin)

- Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
  với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
  vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
  [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các
  sân chơi (playground) Kubernetes sau:

  - [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
  - [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
  - [KodeKloud](https://kodekloud.com/public-playgrounds)

  Phiên bản Kubernetes server của bạn phải bằng hoặc mới hơn v1.36. Để kiểm tra phiên bản, nhập
  `kubectl version`.
- Cấp phát tài nguyên động đã được cấu hình trong cluster của bạn.
- Bạn có thể chỉnh sửa các resource RBAC và khởi động lại hoặc roll out các thành phần DRA.

## Xác định các thành phần DRA ghi vào status (Identify DRA components that write status)

Hãy ghi lại danh sách những danh tính (identity) — thường là các ServiceAccount — đang cập nhật
status của ResourceClaim trong cluster của bạn. Các bên ghi (writer) điển hình gồm:

- kube-scheduler hoặc một controller cấp phát (allocation controller) tùy chỉnh
- các DRA driver cục bộ trên node (node-local)
- các controller trạng thái DRA đa node (multi-node)

## Cấp quyền tối thiểu cho các subresource tổng hợp (Grant least-privilege permissions for synthetic subresources)

Kể từ Kubernetes v1.36, các cập nhật status của DRA yêu cầu quyền trên các subresource tổng hợp
(synthetic subresource) bên cạnh quyền trên `resourceclaims/status`.

### Cấp quyền cho scheduler và controller cấp phát (Grant scheduler and allocation-controller permissions)

Áp dụng một role cho phép các cập nhật liên quan tới binding:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: dra-binding-updater
rules:
  - apiGroups: ["resource.k8s.io"]
    resources: ["resourceclaims/status"]
    verbs: ["get", "patch", "update"]
  - apiGroups: ["resource.k8s.io"]
    resources: ["resourceclaims/binding"]
    verbs: ["patch", "update"]
```

### Cấp quyền cho driver cục bộ trên node (Grant node-local driver permissions)

Dùng các verb gắn với node (node-aware) cho các driver cục bộ trên node:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: dra-node-driver-status-updater
rules:
  - apiGroups: ["resource.k8s.io"]
    resources: ["resourceclaims/status"]
    verbs: ["get", "patch", "update"]
  - apiGroups: ["resource.k8s.io"]
    resources: ["resourceclaims/driver"]
    verbs: ["associated-node:patch", "associated-node:update"]
    resourceNames: ["dra.example.com"]
```

### Chỉ cấp quyền cho controller đa node khi thật sự cần (Grant multi-node controller permissions only when needed)

Chỉ dùng `arbitrary-node:*` cho những thành phần bắt buộc phải cập nhật từ bất kỳ node nào:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: dra-multinode-status-updater
rules:
  - apiGroups: ["resource.k8s.io"]
    resources: ["resourceclaims/status"]
    verbs: ["get", "patch", "update"]
  - apiGroups: ["resource.k8s.io"]
    resources: ["resourceclaims/driver"]
    verbs: ["arbitrary-node:patch", "arbitrary-node:update"]
    resourceNames: ["dra.example.com"]
```

## Gắn role với các danh tính tường minh (Bind roles to explicit identities)

Tạo các đối tượng `ClusterRoleBinding` cho từng danh tính thành phần, và tránh dùng chung một
role rộng cho nhiều thành phần DRA không liên quan tới nhau.

Khi có thể, hãy giới hạn các rule trên `resourceclaims/driver` bằng `resourceNames` để một danh
tính chỉ có thể ghi status cho đúng DRA driver mà nó vận hành.

## Kiểm chứng và giám sát (Validate and monitor)

1. Kiểm tra rằng mỗi danh tính chỉ có đúng những verb và subresource cần thiết.
1. Xác nhận các cập nhật status của DRA vẫn hoạt động sau khi roll out.
1. Theo dõi các sự kiện audit của API server để phát hiện những request bị từ chối đối với
   `resourceclaims/binding` và `resourceclaims/driver`.

## Tiếp theo (What's next)

- [Hướng dẫn tăng cường bảo mật - Cấp phát tài nguyên động (Hardening Guide - Dynamic Resource Allocation)](125-hardening-dra-vi.md)
- [Bảo vệ cluster của bạn (Securing a Cluster)](https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/)
- [Phân quyền (Authorization)](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)

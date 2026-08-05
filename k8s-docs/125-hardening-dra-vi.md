# Hướng dẫn tăng cường bảo mật - Cấp phát tài nguyên động (Hardening Guide - Dynamic Resource Allocation)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/hardening-guide/dynamic-resource-allocation/>
>
> Thông tin về việc tăng cường bảo mật cho phân quyền và các mẫu truy cập của Cấp phát tài nguyên động (Dynamic Resource Allocation - DRA).

Cấp phát tài nguyên động (Dynamic Resource Allocation - DRA) bổ sung các khả năng lập lịch và quản lý thiết bị
mạnh mẽ. Vì các thành phần DRA cập nhật trạng thái (status) của `ResourceClaim`, quản trị viên
cluster nên cấu hình phân quyền cho các cập nhật đó bằng RBAC tường minh
theo nguyên tắc đặc quyền tối thiểu (least-privilege).

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [beta]`

Bắt đầu từ Kubernetes v1.36, các cập nhật trạng thái của DRA sử dụng các subresource tổng hợp (synthetic subresource) và,
trong một số trường hợp, các verb chuyên biệt có nhận biết node (node-aware).

## Tăng cường bảo mật cho quyền cập nhật trạng thái DRA (Harden DRA status update permissions)

Đối với các cập nhật trạng thái DRA, ngoài việc cấp quyền `update` trên
subresource `resourceclaims/status`, quản trị viên cluster phải cấp quyền trên
các subresource "tổng hợp" (synthetic) cụ thể dựa trên chính xác những trường mà một thành phần cần sửa đổi.
Điều này thực thi nguyên tắc đặc quyền tối thiểu giữa scheduler, các controller tùy chỉnh,
và các driver DRA.

Các kiểm tra phân quyền của DRA được chia thành hai subresource tổng hợp:

- **`resourceclaims/binding`**
  - Bắt buộc phải có để sửa đổi `status.allocation` và `status.reservedFor`.
  - Thường được cấp cho kube-scheduler và các controller cấp phát tùy chỉnh.
  - Sử dụng các verb `update` và `patch` tiêu chuẩn.
- **`resourceclaims/driver`**
  - Bắt buộc phải có để sửa đổi `status.devices`.
  - Kiểm tra này được thực hiện theo từng driver để ngăn các driver can thiệp vào thiết bị trên các node khác
  và/hoặc thiết bị của các driver khác.
  - Sử dụng các verb có nhận biết node để giới hạn phạm vi chặt chẽ hơn.

## Các verb DRA có nhận biết node (Node-aware DRA verbs)

Khi phân quyền cho các cập nhật trên `resourceclaims/driver`, hãy sử dụng tiền tố verb
chuyên biệt phù hợp:

- **`associated-node:<verb>`** (ví dụ: `associated-node:update`)
  - Dành cho các driver cục bộ trên node (node-local).
  - API server xác minh mối liên kết với node của driver đang gửi yêu cầu.
- **`arbitrary-node:<verb>`** (ví dụ: `arbitrary-node:patch`)
  - Dành cho các controller ở control plane hoặc controller đa node có thể cập nhật claim từ
    bất kỳ node nào.

## Các mẫu RBAC ví dụ (Example RBAC patterns)

### Quyền cho scheduler và controller cấp phát (Scheduler and allocation controller permissions)

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

### Quyền cho driver DRA cục bộ trên node (Node-local DRA driver permissions)

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

### Quyền cho controller trạng thái đa node (Multi-node status controller permissions)

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

## Tác vụ liên quan dành cho quản trị viên cluster (Related cluster administrator task)

Để áp dụng các mẫu này trong một cluster đang chạy, hãy xem
[Tăng cường bảo mật cho Cấp phát tài nguyên động trong cluster của bạn (Harden Dynamic Resource Allocation in Your Cluster)](https://kubernetes.io/docs/tasks/administer-cluster/hardening-dra/).

## Tiếp theo (What's next)

- [Phân quyền (Authorization)](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
- [Thiết lập DRA trong một Cluster (Set Up DRA in a Cluster)](https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/set-up-dra-cluster/)
- [Cấp phát tài nguyên động (Dynamic Resource Allocation)](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)

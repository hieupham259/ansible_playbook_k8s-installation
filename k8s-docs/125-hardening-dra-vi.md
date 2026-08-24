# Hướng dẫn tăng cường bảo mật - Cấp phát tài nguyên động (Hardening Guide - Dynamic Resource Allocation)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/hardening-guide/dynamic-resource-allocation/>
>
> Thông tin về việc tăng cường bảo mật cho phân quyền và các mẫu truy cập của Cấp phát tài nguyên động (Dynamic Resource Allocation - DRA).

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 13](00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao),
bài 3/15 · Kiểm chứng ở Lab 13 (tùy chọn, chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

**Giai đoạn 13 không bắt buộc với admin mới.** Phần lớn giai đoạn này là tính năng alpha/beta
hoặc dành cho nền tảng chuyên biệt (AI/HPC, GPU). Chỉ đọc khi đã vững giai đoạn 1–12 hoặc khi
công việc thực sự cần. Cluster lab 3 VM trên VMware, không GPU, **không chạy được** bài này:
không có DRA driver thì không có thành phần nào cần những quyền được mô tả ở đây. Đọc để hiểu
mô hình phân quyền, không phải để áp dụng.

Đây là **phần hoãn lại từ giai đoạn 9**. Lộ trình cố tình đẩy bài này xuống đây thay vì đọc
cùng cụm hardening: nó chỉ có nghĩa sau khi bạn đã biết ResourceClaim là gì và ai ghi vào
`status` của nó, tức là sau [bài 149](149-dynamic-resource-allocation-vi.md). Nền RBAC thì
ngược lại — đã học ở giai đoạn 9 với
[bài 119](119-controlling-access-vi.md) và [bài 120](120-rbac-good-practices-vi.md); bài này
chỉ thêm một lớp đặc thù DRA lên trên.

Cơ chế mô tả ở đây là `beta` từ v1.36 và **API có thể còn đổi** — kiểm tra lại tài liệu đúng
phiên bản cluster trước khi viết ClusterRole thật.

**Phải hiểu ở lần đọc này:**

- Vì sao DRA cần lớp phân quyền riêng: nhiều thành phần khác nhau cùng ghi vào `status` của
  ResourceClaim, nên chỉ cấp `update` trên `resourceclaims/status` là quá rộng.
- Hai **subresource tổng hợp (synthetic)** và ranh giới giữa chúng: `resourceclaims/binding`
  cho `status.allocation` và `status.reservedFor` (scheduler, controller cấp phát);
  `resourceclaims/driver` cho `status.devices` (driver DRA).
- Vì sao kiểm tra trên `resourceclaims/driver` được thực hiện **theo từng driver**: để một
  driver không can thiệp được vào thiết bị trên node khác hoặc thiết bị của driver khác — cụ
  thể hóa bằng `resourceNames` trong ClusterRole.
- Hai tiền tố verb có nhận biết node và khi nào dùng cái nào: `associated-node:<verb>` cho
  driver cục bộ trên node (API server xác minh liên kết node của bên gửi yêu cầu), và
  `arbitrary-node:<verb>` cho controller ở control plane hoặc controller đa node.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ba manifest ClusterRole ví dụ | là mẫu để chép khi thực sự triển khai driver | khi công việc thực sự cần |
| Khác biệt giữa verb `update` và `patch` trên subresource | thuộc nền RBAC chung | giai đoạn 9 — [bài 120](120-rbac-good-practices-vi.md) |
| Vì sao `status.devices` tách khỏi `status.allocation` | cần vòng đời cấp phát của DRA | [bài 149](149-dynamic-resource-allocation-vi.md) — mục *Trạng thái thiết bị trong ResourceClaim* |

---

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
[Tăng cường bảo mật cho Cấp phát tài nguyên động trong cluster của bạn (Harden Dynamic Resource Allocation in Your Cluster)](211-hardening-dra-tasks-vi.md).

## Tiếp theo (What's next)

- [Phân quyền (Authorization)](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
- [Thiết lập DRA trong một Cluster (Set Up DRA in a Cluster)](271-set-up-dra-cluster-vi.md)
- [Cấp phát tài nguyên động (Dynamic Resource Allocation)](149-dynamic-resource-allocation-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho một giai đoạn tùy chọn:

1. Thành phần nào cần `resourceclaims/binding` và thành phần nào cần `resourceclaims/driver`?
   Ranh giới giữa hai subresource này nằm ở những trường nào của `status`?
2. Bạn đã cấp cho DRA driver quyền `update` trên `resourceclaims/status`. Như vậy đã đủ để nó
   ghi `status.devices` chưa? Vì sao?
3. Một driver chạy dạng DaemonSet trên từng node nên dùng tiền tố verb nào, và điều gì bị mất
   nếu bạn cấp nhầm tiền tố kia?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **`resourceclaims/binding` — cho kube-scheduler và các controller cấp phát tùy chỉnh**, bắt
   buộc phải có để sửa `status.allocation` và `status.reservedFor`; dùng verb `update`/`patch`
   tiêu chuẩn. **`resourceclaims/driver` — cho driver DRA**, bắt buộc phải có để sửa
   `status.devices`; dùng verb có nhận biết node. Ranh giới vì thế là: **ai quyết định cấp
   phát và giữ chỗ** (binding) so với **ai báo cáo trạng thái thiết bị** (driver).
2. **Chưa đủ.** Bài nói rõ: *ngoài* quyền `update` trên `resourceclaims/status`, quản trị viên
   **phải cấp thêm quyền trên đúng subresource tổng hợp** ứng với những trường mà thành phần
   đó cần sửa. Trực giác "status là một tài nguyên, có quyền trên status là ghi được mọi
   trường" chính là thứ cơ chế này phá bỏ: từ v1.36 các kiểm tra phân quyền của DRA được tách
   thành `binding` và `driver` để thực thi đặc quyền tối thiểu giữa scheduler, các controller
   tùy chỉnh và các driver.
3. Driver cục bộ trên node dùng **`associated-node:<verb>`**, ví dụ `associated-node:update` —
   khi đó **API server xác minh mối liên kết với node của driver đang gửi yêu cầu**. Cấp nhầm
   `arbitrary-node:` (vốn dành cho controller ở control plane hoặc controller đa node) sẽ **mất
   đúng phép kiểm tra đó**: driver trên một node cập nhật được claim từ bất kỳ node nào. Kết
   hợp với `resourceNames` giới hạn theo tên driver, hai lớp này chính là thứ ngăn một driver
   can thiệp vào thiết bị trên node khác hoặc thiết bị của driver khác.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

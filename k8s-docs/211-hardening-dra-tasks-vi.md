# Tăng cường bảo mật cho Cấp phát tài nguyên động trong cluster của bạn (Harden Dynamic Resource Allocation in Your Cluster)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/hardening-dra/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 13 — Lập lịch và workload nâng cao](00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao)
→ dòng **Thực hành**, bài 1/5 · Kiểm chứng ở [Lab 13 — DRA](labs/LAB-13-DRA.md) phần B10.

**Giai đoạn 13 không bắt buộc với admin mới**, và Lab 13 là **lab tùy chọn**. Ba VM của lab không
có GPU và không có DRA driver nào, nên Lab 13 chạy **nhánh đọc-hiểu**: phần B10 đọc quyền **thật
đang có** trên `resourceclaims` thay vì tạo quyền mới. Đây là mặt thi công của bài
[125](125-hardening-dra-vi.md) đã đọc ở mạch chính cùng giai đoạn: 125 giải thích *vì sao* tách hai
subresource tổng hợp, bài này nói *bạn phải làm gì* với chúng.

**Phải hiểu ở lần đọc này:**

- Ba nhóm danh tính ghi vào `status` của ResourceClaim mà mục *Xác định các thành phần DRA ghi vào
  status* liệt kê: kube-scheduler hoặc controller cấp phát tùy chỉnh, DRA driver cục bộ trên node,
  và controller trạng thái DRA đa node. Việc đầu tiên của hardening là **lập được danh sách này
  cho cluster của mình**, không phải viết ClusterRole.
- Từ Kubernetes v1.36, cập nhật status của DRA cần quyền trên **subresource tổng hợp** *bên cạnh*
  `resourceclaims/status` — cấp mỗi `resourceclaims/status` là chưa đủ để thành phần chạy được.
- Ba ClusterRole ví dụ khác nhau đúng ở dòng subresource và verb: scheduler/controller cấp phát
  dùng `resourceclaims/binding`; driver node-local dùng `resourceclaims/driver` với
  `associated-node:patch` và `associated-node:update`; controller đa node dùng
  `arbitrary-node:patch`/`arbitrary-node:update` — và bài nói rõ **chỉ dùng `arbitrary-node:*` cho
  những thành phần bắt buộc phải cập nhật từ bất kỳ node nào**.
- Hai biện pháp thu hẹp phạm vi ở mục *Gắn role với các danh tính tường minh*: mỗi thành phần một
  `ClusterRoleBinding` riêng thay vì dùng chung một role rộng, và `resourceNames` để một danh tính
  chỉ ghi được status cho đúng DRA driver mà nó vận hành.
- Ba việc của mục *Kiểm chứng và giám sát*, theo đúng thứ tự: soát verb và subresource của từng
  danh tính, **xác nhận cập nhật status vẫn hoạt động** sau khi roll out, rồi theo dõi các request
  bị từ chối.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Áp ba manifest ClusterRole vào cluster và roll out lại các thành phần DRA | ba role chỉ có nghĩa khi gắn vào ServiceAccount của một driver thật; cluster lab không có driver nào | [Lab 13](labs/LAB-13-DRA.md) phần B10 — đọc quyền thật đang có thay vì tạo quyền mới |
| Bước 3 của *Kiểm chứng và giám sát* — theo dõi sự kiện audit của API server | cluster baseline chưa bật audit policy | [giai đoạn 22](00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu) |
| Vì sao `status` của ResourceClaim lại bị tách làm hai subresource tổng hợp | ở đây chỉ cần biết **phải cấp cái nào cho ai**, chưa cần lý do thiết kế | bài [125](125-hardening-dra-vi.md), đã đọc ở mạch chính giai đoạn 13 |
| Điều kiện tiên quyết "Cấp phát tài nguyên động đã được cấu hình trong cluster của bạn" | đó là việc của quản trị viên, bài này giả định đã xong | bài [271](271-set-up-dra-cluster-vi.md), bài cuối của chính nhóm Thực hành này |

---

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
- [Bảo vệ cluster của bạn (Securing a Cluster)](256-securing-a-cluster-vi.md)
- [Phân quyền (Authorization)](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 13:

1. Bài bắt bạn "ghi lại danh sách" gì trước khi viết bất kỳ ClusterRole nào, ba nhóm danh tính
   điển hình nào được liệt kê, và vì sao thứ tự đó không đảo được?
2. **Câu bẫy.** Bạn cấp cho một DRA driver quyền `get`, `patch`, `update` trên
   `resourceclaims/status` rồi kết luận "đủ rồi". Kể từ phiên bản nào thì kết luận đó sai, và còn
   thiếu chính xác cái gì?
3. `associated-node:patch` và `arbitrary-node:patch` khác nhau ở phạm vi nào, bài đặt điều kiện gì
   cho việc dùng cái thứ hai, và `resourceNames` trong cùng rule đó siết thêm theo chiều nào?
4. Cluster lab của bạn — `lab-k8s-master` cộng `lab-k8s-worker1` và `lab-k8s-worker2`, không GPU,
   chưa cài DRA driver nào — thì bước *Xác định các thành phần DRA ghi vào status* cho ra danh
   sách gồm những ai? Và theo mục *Kiểm chứng và giám sát*, sau khi roll out một role mới bạn phải
   xác nhận điều gì, chứ không chỉ là "không thấy lỗi"?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Danh sách **những danh tính — thường là ServiceAccount — đang cập nhật status của ResourceClaim
   trong cluster của bạn**. Ba nhóm điển hình: **kube-scheduler hoặc một controller cấp phát tùy
   chỉnh**, **các DRA driver cục bộ trên node**, và **các controller trạng thái DRA đa node**. Thứ
   tự không đảo được vì **mỗi nhóm cần một subresource và một bộ verb khác nhau**: chưa biết ai
   đang ghi thì hoặc cấp thừa quyền, hoặc roll out xong mới phát hiện một thành phần đã mất quyền.
2. Sai **kể từ Kubernetes v1.36**: từ bản đó, cập nhật status của DRA yêu cầu quyền trên các
   **subresource tổng hợp bên cạnh `resourceclaims/status`**. Với driver thì còn thiếu
   **`resourceclaims/driver`**; với scheduler và controller cấp phát thì thiếu
   **`resourceclaims/binding`**. Chỗ dễ sai là role cũ vẫn "trông đúng" — `resourceclaims/status`
   vẫn phải có, nó chỉ **không còn đủ một mình**.
3. `associated-node:*` dành cho **driver cục bộ trên node**, tức bên ghi status gắn với node mà nó
   chạy. `arbitrary-node:*` cho phép cập nhật **từ bất kỳ node nào**, nên bài đặt điều kiện: **chỉ
   dùng cho những thành phần bắt buộc phải cập nhật từ bất kỳ node nào** — các controller đa node.
   `resourceNames` siết theo chiều **driver**: giới hạn rule vào đúng tên driver, ví dụ
   `["dra.example.com"]`, để một danh tính **chỉ ghi được status cho đúng driver mà nó vận hành**.
4. **Không có driver nào trong danh sách**: không cài DRA driver thì không có driver cục bộ trên
   node và không có controller trạng thái đa node; ứng viên duy nhất còn lại thuộc nhóm thứ nhất là
   **kube-scheduler**. Đó chính là lý do bài này đọc để hiểu mô hình chứ không áp vào cluster lab.
   Sau khi roll out, bài yêu cầu **xác nhận các cập nhật status của DRA vẫn hoạt động** — tức phải
   kiểm chứng mặt khẳng định, vì siết quyền quá tay thì triệu chứng là status **ngừng được cập
   nhật**, không phải một lỗi hiện ra ngay lúc apply role.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

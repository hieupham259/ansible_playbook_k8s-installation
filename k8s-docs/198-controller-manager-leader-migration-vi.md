# Di chuyển control plane được nhân bản sang dùng Cloud Controller Manager (Migrate Replicated Control Plane To Use Cloud Controller Manager)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/controller-manager-leader-migration/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** tài liệu tra cứu thuộc nhánh `/docs/tasks/`
([Checkpoint tiếp nối](LO-TRINH-ADMIN.md#checkpoint-tiếp-nối--nhánh-docstasks)), là phần
"Tiếp theo" của bài [34 — Cloud Controller Manager](34-cloud-controller-vi.md) và có họ hàng
gần nhất với [CP2 — Nâng cấp cluster](LO-TRINH-ADMIN.md#cp2--nâng-cấp-cluster).

Bài này chỉ áp dụng khi cluster chạy **trên một cloud provider** và có **control plane nhân
bản (HA)** đang chạy các cloud controller trong `kube-controller-manager`. Cluster lab
on-premise của bạn không đặt `--cloud-provider` nên **không bao giờ chạy migration này** —
đọc để hiểu cơ chế phối hợp qua Lease, thứ bạn đã học ở bài [35](35-leases-vi.md) và
[167](167-coordinated-leader-election-vi.md).

**Phải hiểu ở lần đọc này:**

- Bài giải quyết vấn đề gì: chuyển các controller đặc thù cloud (`route`, `service`,
  `cloud-node-lifecycle`) từ `kube-controller-manager` sang `cloud-controller-manager`
  **trong lúc nâng cấp cuốn chiếu (rolling) control plane HA**, sao cho không controller nào
  chạy đồng thời ở hai nơi.
- Cơ chế: hai controller manager chia sẻ **một Lease di trú** (resource lock chung, tên ví dụ
  `cloud-provider-extraction-migration`); manager nào giữ Lease thì chạy nhóm controller được
  khai báo trong `LeaderMigrationConfiguration`.
- Khi nào **không** cần Leader Migration: control plane một node, hoặc chấp nhận controller
  manager không sẵn sàng trong lúc nâng cấp.
- Trình tự chính: cấp quyền RBAC vào Lease di trú → bật `--enable-leader-migration` trên
  `kube-controller-manager` phiên bản N → dựng node N + 1 chạy `cloud-controller-manager` có
  leader migration và `kube-controller-manager` với `--cloud-provider=external` (node N + 1
  **không** bật migration cho kube-controller-manager) → thay thế cuốn chiếu → tắt migration
  sau khi xong.
- Giá trị của wildcard `component: *`: một file cấu hình dùng được cho cả hai phía của cuộc
  di trú; từ Kubernetes 1.22 còn có **cấu hình mặc định** (chỉ cần
  `--enable-leader-migration`, không cần file).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Trường hợp đặc biệt: di trú Node IPAM controller | chỉ áp dụng khi cloud provider có bản Node IPAM riêng | khi vận hành trên cloud provider cụ thể đó |
| Ràng buộc phiên bản thư viện `k8s.io/cloud-provider` v0.21.0/v0.22.0 và feature gate alpha | chi tiết lịch sử dành cho người build `cloud-controller-manager` | không cần cho người vận hành |

---

cloud-controller-manager là một thành phần thuộc control plane của Kubernetes, nhúng logic
điều khiển đặc thù cho từng cloud. Cloud controller manager cho phép bạn kết nối cluster của
mình với API của nhà cung cấp cloud (cloud provider), đồng thời tách các thành phần tương tác
với nền tảng cloud đó khỏi các thành phần chỉ tương tác với cluster của bạn.

Bằng cách tách rời logic tương tác giữa Kubernetes và hạ tầng cloud bên dưới, thành phần
cloud-controller-manager giúp các nhà cung cấp cloud có thể phát hành tính năng theo nhịp độ
khác với dự án Kubernetes chính.

## Bối cảnh (Background)

Trong khuôn khổ [nỗ lực tách cloud provider](https://kubernetes.io/blog/2019/04/17/the-future-of-cloud-providers-in-kubernetes/),
tất cả các controller đặc thù cloud phải được chuyển ra khỏi `kube-controller-manager`. Mọi
cluster hiện có đang chạy các cloud controller trong `kube-controller-manager` phải di chuyển
(migrate) sang chạy các controller đó trong một `cloud-controller-manager` đặc thù của từng
nhà cung cấp cloud.

Leader Migration (di trú leader) cung cấp một cơ chế để các cluster HA có thể di chuyển an
toàn các controller "đặc thù cloud" giữa `kube-controller-manager` và
`cloud-controller-manager` thông qua một resource lock dùng chung giữa hai thành phần, trong
khi nâng cấp control plane được nhân bản (replicated). Với control plane một node, hoặc nếu
có thể chấp nhận việc các controller manager không sẵn sàng trong quá trình nâng cấp, thì
không cần Leader Migration và có thể bỏ qua hướng dẫn này.

Leader Migration có thể được bật bằng cách đặt `--enable-leader-migration` trên
`kube-controller-manager` hoặc `cloud-controller-manager`. Leader Migration chỉ có tác dụng
trong quá trình nâng cấp và có thể được tắt an toàn hoặc để nguyên trạng thái bật sau khi
nâng cấp hoàn tất.

Hướng dẫn này dẫn dắt bạn qua quy trình thủ công để nâng cấp control plane từ
`kube-controller-manager` với cloud provider tích hợp sẵn (in-tree) sang chạy đồng thời cả
`kube-controller-manager` và `cloud-controller-manager`. Nếu bạn dùng một công cụ để triển
khai và quản lý cluster, vui lòng tham khảo tài liệu của công cụ đó và của nhà cung cấp cloud
để có hướng dẫn di trú cụ thể.

## Trước khi bạn bắt đầu (Before you begin)

Giả định rằng control plane đang chạy Kubernetes phiên bản N và sẽ được nâng cấp lên phiên
bản N + 1. Mặc dù có thể di trú trong cùng một phiên bản, lý tưởng nhất là việc di trú nên
được thực hiện như một phần của đợt nâng cấp, để các thay đổi cấu hình có thể gắn với từng
bản phát hành. Giá trị chính xác của N và N + 1 phụ thuộc vào từng nhà cung cấp cloud. Ví dụ,
nếu một nhà cung cấp cloud xây dựng `cloud-controller-manager` hoạt động với Kubernetes 1.24,
thì N có thể là 1.23 và N + 1 có thể là 1.24.

Các node control plane nên chạy `kube-controller-manager` với Leader Election (bầu chọn
leader) được bật — đây là thiết lập mặc định. Ở phiên bản N, một cloud provider in-tree phải
được đặt qua flag `--cloud-provider` và `cloud-controller-manager` chưa được triển khai.

Cloud provider out-of-tree phải có bản build `cloud-controller-manager` với phần hiện thực
Leader Migration. Nếu cloud provider import `k8s.io/cloud-provider` và
`k8s.io/controller-manager` phiên bản v0.21.0 trở lên, Leader Migration sẽ khả dụng. Tuy
nhiên, với các phiên bản trước v0.22.0, Leader Migration ở mức alpha và yêu cầu bật feature
gate `ControllerManagerLeaderMigration` trong `cloud-controller-manager`.

Hướng dẫn này giả định rằng kubelet của mỗi node control plane khởi động
`kube-controller-manager` và `cloud-controller-manager` dưới dạng static pod được định nghĩa
bởi manifest của chúng. Nếu các thành phần này chạy theo cách khác, vui lòng điều chỉnh các
bước cho phù hợp.

Về phân quyền, hướng dẫn này giả định cluster dùng RBAC. Nếu một chế độ phân quyền
(authorization mode) khác cấp quyền cho các thành phần `kube-controller-manager` và
`cloud-controller-manager`, hãy cấp các quyền cần thiết theo cách phù hợp với chế độ đó.

### Cấp quyền truy cập Lease di trú (Grant access to Migration Lease)

Quyền mặc định của controller manager chỉ cho phép truy cập Lease chính của nó. Để việc di
trú hoạt động, cần quyền truy cập vào một Lease khác.

Bạn có thể cấp cho `kube-controller-manager` quyền truy cập đầy đủ vào API leases bằng cách
sửa role `system::leader-locking-kube-controller-manager`. Hướng dẫn này giả định tên của
Lease di trú là `cloud-provider-extraction-migration`.

```shell
kubectl patch -n kube-system role 'system::leader-locking-kube-controller-manager' -p '{"rules": [ {"apiGroups":[ "coordination.k8s.io"], "resources": ["leases"], "resourceNames": ["cloud-provider-extraction-migration"], "verbs": ["create", "list", "get", "update"] } ]}' --type=merge
```

Làm điều tương tự với role `system::leader-locking-cloud-controller-manager`.

```shell
kubectl patch -n kube-system role 'system::leader-locking-cloud-controller-manager' -p '{"rules": [ {"apiGroups":[ "coordination.k8s.io"], "resources": ["leases"], "resourceNames": ["cloud-provider-extraction-migration"], "verbs": ["create", "list", "get", "update"] } ]}' --type=merge
```

### Cấu hình Leader Migration ban đầu (Initial Leader Migration configuration)

Leader Migration có thể nhận (tùy chọn) một file cấu hình biểu diễn trạng thái gán
controller-cho-manager (controller-to-manager assignment). Ở thời điểm này, với cloud
provider in-tree, `kube-controller-manager` chạy `route`, `service`, và
`cloud-node-lifecycle`. Ví dụ cấu hình sau thể hiện cách gán đó.

Leader Migration có thể được bật mà không cần cấu hình. Xem
[Cấu hình mặc định](#default-configuration) để biết chi tiết.

```yaml
kind: LeaderMigrationConfiguration
apiVersion: controllermanager.config.k8s.io/v1
leaderName: cloud-provider-extraction-migration
resourceLock: leases
controllerLeaders:
  - name: route
    component: kube-controller-manager
  - name: service
    component: kube-controller-manager
  - name: cloud-node-lifecycle
    component: kube-controller-manager
```

Một cách khác, vì các controller có thể chạy dưới bất kỳ controller manager nào, đặt
`component` thành `*` cho cả hai phía sẽ giúp file cấu hình nhất quán giữa hai bên tham gia
cuộc di trú.

```yaml
# phiên bản wildcard
kind: LeaderMigrationConfiguration
apiVersion: controllermanager.config.k8s.io/v1
leaderName: cloud-provider-extraction-migration
resourceLock: leases
controllerLeaders:
  - name: route
    component: *
  - name: service
    component: *
  - name: cloud-node-lifecycle
    component: *
```

Trên mỗi node control plane, lưu nội dung trên vào `/etc/leadermigration.conf`, và cập nhật
manifest của `kube-controller-manager` sao cho file này được mount vào bên trong container
tại cùng vị trí. Đồng thời, cập nhật chính manifest đó để thêm các đối số sau:

- `--enable-leader-migration` để bật Leader Migration trên controller manager
- `--leader-migration-config=/etc/leadermigration.conf` để đặt file cấu hình

Khởi động lại `kube-controller-manager` trên từng node. Tại thời điểm này,
`kube-controller-manager` đã bật leader migration và sẵn sàng cho cuộc di trú.

### Triển khai Cloud Controller Manager (Deploy Cloud Controller Manager)

Ở phiên bản N + 1, trạng thái mong muốn của việc gán controller-cho-manager có thể được biểu
diễn bằng một file cấu hình mới, như dưới đây. Lưu ý trường `component` của mỗi phần tử
`controllerLeaders` đổi từ `kube-controller-manager` thành `cloud-controller-manager`. Hoặc
dùng phiên bản wildcard đã nêu ở trên, có cùng hiệu quả.

```yaml
kind: LeaderMigrationConfiguration
apiVersion: controllermanager.config.k8s.io/v1
leaderName: cloud-provider-extraction-migration
resourceLock: leases
controllerLeaders:
  - name: route
    component: cloud-controller-manager
  - name: service
    component: cloud-controller-manager
  - name: cloud-node-lifecycle
    component: cloud-controller-manager
```

Khi tạo các node control plane phiên bản N + 1, nội dung trên cần được triển khai vào
`/etc/leadermigration.conf`. Manifest của `cloud-controller-manager` cần được cập nhật để
mount file cấu hình theo cùng cách như `kube-controller-manager` phiên bản N. Tương tự, thêm
`--enable-leader-migration` và `--leader-migration-config=/etc/leadermigration.conf` vào các
đối số của `cloud-controller-manager`.

Tạo một node control plane mới phiên bản N + 1 với manifest `cloud-controller-manager` đã cập
nhật, và với flag `--cloud-provider` đặt thành `external` cho `kube-controller-manager`.
`kube-controller-manager` phiên bản N + 1 **KHÔNG ĐƯỢC** bật Leader Migration, bởi vì với
cloud provider external, nó không còn chạy các controller đã được di trú nữa, và do đó nó
không tham gia vào cuộc di trú.

Vui lòng tham khảo [Quản trị Cloud Controller Manager](https://kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/)
để biết thêm chi tiết về cách triển khai `cloud-controller-manager`.

### Nâng cấp control plane (Upgrade Control Plane)

Control plane bây giờ chứa các node của cả phiên bản N và N + 1. Các node phiên bản N chỉ
chạy `kube-controller-manager`, còn các node phiên bản N + 1 chạy cả
`kube-controller-manager` và `cloud-controller-manager`. Các controller đã được di trú, theo
như khai báo trong cấu hình, chạy dưới hoặc `kube-controller-manager` phiên bản N hoặc
`cloud-controller-manager` phiên bản N + 1, tùy vào controller manager nào đang giữ Lease di
trú. Không controller nào chạy dưới cả hai controller manager tại bất kỳ thời điểm nào.

Theo cách cuốn chiếu (rolling), tạo một node control plane mới phiên bản N + 1 và hạ một node
phiên bản N xuống, cho đến khi control plane chỉ còn các node phiên bản N + 1. Nếu cần
rollback từ phiên bản N + 1 về N, hãy thêm các node phiên bản N có bật Leader Migration cho
`kube-controller-manager` trở lại control plane, mỗi lần thay thế một node phiên bản N + 1,
cho đến khi chỉ còn các node phiên bản N.

### (Tùy chọn) Tắt Leader Migration (Disable Leader Migration) {#disable-leader-migration}

Giờ đây khi control plane đã được nâng cấp để chạy cả `kube-controller-manager` và
`cloud-controller-manager` phiên bản N + 1, Leader Migration đã hoàn thành nhiệm vụ và có thể
được tắt an toàn để tiết kiệm một tài nguyên Lease. Việc bật lại Leader Migration cho mục
đích rollback trong tương lai là an toàn.

Theo cách cuốn chiếu, cập nhật manifest của `cloud-controller-manager` để bỏ cả hai flag
`--enable-leader-migration` và `--leader-migration-config=`, đồng thời gỡ mount của
`/etc/leadermigration.conf`, và cuối cùng xóa `/etc/leadermigration.conf`. Để bật lại Leader
Migration, tạo lại file cấu hình và thêm lại mount của nó cùng các flag bật Leader Migration
vào `cloud-controller-manager`.

### Cấu hình mặc định (Default Configuration) {#default-configuration}

Kể từ Kubernetes 1.22, Leader Migration cung cấp một cấu hình mặc định phù hợp với cách gán
controller-cho-manager mặc định. Cấu hình mặc định có thể được bật bằng cách đặt
`--enable-leader-migration` nhưng không đặt `--leader-migration-config=`.

Với `kube-controller-manager` và `cloud-controller-manager`, nếu không có flag nào bật một
cloud provider in-tree hay thay đổi quyền sở hữu các controller, có thể dùng cấu hình mặc
định để tránh phải tự tạo file cấu hình.

### Trường hợp đặc biệt: di trú Node IPAM controller (Special case: migrating the Node IPAM controller) {#node-ipam-controller-migration}

Nếu nhà cung cấp cloud của bạn cung cấp một hiện thực của Node IPAM controller, bạn nên
chuyển sang dùng hiện thực trong `cloud-controller-manager`. Tắt Node IPAM controller trong
`kube-controller-manager` phiên bản N + 1 bằng cách thêm `--controllers=*,-nodeipam` vào các
flag của nó. Sau đó thêm `nodeipam` vào danh sách các controller được di trú.

```yaml
# phiên bản wildcard, có nodeipam
kind: LeaderMigrationConfiguration
apiVersion: controllermanager.config.k8s.io/v1
leaderName: cloud-provider-extraction-migration
resourceLock: leases
controllerLeaders:
  - name: route
    component: *
  - name: service
    component: *
  - name: cloud-node-lifecycle
    component: *
  - name: nodeipam
-   component: *
```

## Tiếp theo (What's next)

- Đọc đề xuất cải tiến (enhancement proposal)
  [Controller Manager Leader Migration](https://github.com/kubernetes/enhancements/tree/master/keps/sig-cloud-provider/2436-controller-manager-leader-migration).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc này:

1. Cluster lab của bạn có một control plane duy nhất trên `k8s-master` và không đặt
   `--cloud-provider`. Khi nâng cấp cluster này, bạn có cần Leader Migration không? Nêu hai
   điều kiện khiến một cluster thực sự cần nó.
2. Trong lúc control plane còn lẫn node phiên bản N và N + 1, cơ chế nào bảo đảm controller
   `service` không bao giờ chạy đồng thời ở cả `kube-controller-manager` (N) lẫn
   `cloud-controller-manager` (N + 1)?
3. **Câu bẫy.** Trên node phiên bản N + 1, `kube-controller-manager` có được bật
   `--enable-leader-migration` không? Vì sao — trong khi chính nó ở phiên bản N lại phải bật?
4. Sau khi toàn bộ control plane đã lên N + 1, việc tắt Leader Migration có an toàn không, và
   nếu sau đó cần rollback về N thì làm thế nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không cần.** Bài nói rõ: với control plane một node, hoặc nếu chấp nhận được việc
   controller manager không sẵn sàng trong lúc nâng cấp, thì không cần Leader Migration. Một
   cluster thực sự cần nó khi hội đủ: **(a)** control plane được nhân bản (HA) đang chạy các
   cloud controller in-tree trong `kube-controller-manager` (có đặt `--cloud-provider`), và
   **(b)** cần di chuyển các controller đó sang `cloud-controller-manager` trong lúc nâng cấp
   mà không gián đoạn.
2. **Lease di trú dùng chung** (resource lock giữa hai thành phần, ví dụ
   `cloud-provider-extraction-migration`). Các controller được liệt kê trong
   `LeaderMigrationConfiguration` chỉ chạy dưới controller manager **đang giữ Lease đó** —
   hoặc `kube-controller-manager` phiên bản N, hoặc `cloud-controller-manager` phiên bản
   N + 1 — nên không controller nào chạy dưới cả hai cùng lúc.
3. **Không — bài viết nhấn mạnh KHÔNG ĐƯỢC bật.** Ở N + 1, `kube-controller-manager` chạy với
   `--cloud-provider=external` nên **không còn chạy các controller đã di trú nữa**, tức nó
   không tham gia cuộc di trú; bật migration cho nó là vô nghĩa. Chỗ dễ nhầm: ở phiên bản N
   thì chính `kube-controller-manager` đang giữ các cloud controller, nên nó phải bật
   migration để tham gia tranh Lease; sang N + 1 vai trò đó đã chuyển hẳn cho
   `cloud-controller-manager`.
4. **An toàn.** Leader Migration chỉ có tác dụng trong quá trình nâng cấp; tắt đi giúp tiết
   kiệm một tài nguyên Lease. Cách tắt: cập nhật cuốn chiếu manifest của
   `cloud-controller-manager` để bỏ `--enable-leader-migration` và
   `--leader-migration-config=`, gỡ mount rồi xóa `/etc/leadermigration.conf`. Khi cần
   rollback về N: bật lại Leader Migration (tạo lại file cấu hình, mount và flag), rồi thêm
   các node phiên bản N có bật migration cho `kube-controller-manager` trở lại, mỗi lần thay
   một node N + 1 cho đến khi chỉ còn node phiên bản N.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

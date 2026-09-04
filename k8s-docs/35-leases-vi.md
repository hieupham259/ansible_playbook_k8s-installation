# Các Lease (Leases)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/architecture/leases/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 1 → nhóm [1c](00-ALO-TRINH-ADMIN.md#1c-vòng-đời-và-cơ-chế-nền-của-object),
bài 4/7 · Kiểm chứng ở [Lab 1c](labs/LAB-1C-VONG-DOI-VA-CO-CHE-NEN-CUA-OBJECT.md).

Bạn đã **dùng** Lease ở Lab 1a phần B6 mà chưa biết nó là gì: theo dõi `renewTime` của
`lab-k8s-worker2` rồi dừng kubelet để xem nó đứng lại. Đây là bài giải thích thứ bạn đã quan sát.

**Phải hiểu ở lần đọc này:**

- Lease là một object thật, thuộc API group `coordination.k8s.io`. Nó phục vụ **hai việc**
  hoàn toàn khác nhau: heartbeat của node, và bầu chọn leader.
- Heartbeat: mỗi Node có một Lease **trùng tên** trong namespace `kube-node-lease`. Mỗi
  heartbeat là một request **update** làm mới `spec.renewTime`.
- Bầu leader: `kube-controller-manager` và `kube-scheduler` trong cấu hình HA dùng Lease để
  chỉ **một** instance hoạt động, các instance khác chờ. Điều này khớp với bảng ba mô hình HA
  ở bài [22](22-architecture-vi.md).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Feature gate `ControllerManagerReleaseLeaderElectionLockOnExit` (alpha) | tinh chỉnh độ trễ chuyển giao leader | giai đoạn 12 |
| *Định danh của API server* và các Lease `apiserver-<hash>` | chỉ có ý nghĩa khi có nhiều API server | giai đoạn 8 |
| *Workload* — tự dùng Lease trong controller của bạn | dành cho người viết controller | giai đoạn 14 |

---

Các hệ thống phân tán thường có nhu cầu về _lease_ (hợp đồng thuê), cơ chế cung cấp cách khóa
các tài nguyên dùng chung và điều phối hoạt động giữa các thành viên trong một nhóm.
Trong Kubernetes, khái niệm lease được biểu diễn bởi các đối tượng
[Lease](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/lease-v1/)
trong API Group `coordination.k8s.io`, được dùng cho những khả năng thiết yếu của hệ thống
như heartbeat (nhịp tim) của node và bầu chọn leader (leader election) ở cấp thành phần.

## Nhịp tim của node (Node heartbeats) {#node-heart-beats}

Kubernetes dùng Lease API để truyền các heartbeat của kubelet trên node tới API server của Kubernetes.
Với mỗi `Node`, có một đối tượng `Lease` trùng tên trong namespace `kube-node-lease`.
Bên dưới, mỗi heartbeat của kubelet là một request **update** tới đối tượng `Lease` này,
cập nhật trường `spec.renewTime` của Lease. Control plane của Kubernetes dùng dấu thời gian (time stamp)
của trường này để xác định tính khả dụng của `Node` đó.

Xem [các đối tượng Lease của Node](./23-nodes-vi.md#nhịp-tim-của-node-node-heartbeats) để biết thêm chi tiết.

## Bầu chọn leader (Leader election)

Kubernetes cũng dùng Lease để đảm bảo tại mỗi thời điểm chỉ có một instance của một thành phần đang chạy.
Cơ chế này được các thành phần control plane như `kube-controller-manager` và `kube-scheduler`
sử dụng trong cấu hình HA (tính sẵn sàng cao — high availability), khi chỉ một instance của
thành phần được chạy chủ động còn các instance khác ở trạng thái chờ (stand-by).

Đọc [bầu chọn leader có điều phối (coordinated leader election)](167-coordinated-leader-election-vi.md)
để tìm hiểu cách Kubernetes xây dựng trên nền Lease API nhằm chọn ra instance nào
của thành phần đóng vai trò leader.

> **Ghi chú giải thích bổ sung (nội dung bên ngoài tài liệu Kubernetes gốc):**
>
> Trong một cluster control plane, có thể quan sát thấy các Lease như
> `kube-controller-manager`, `kube-scheduler` và `apiserver-<hash>`. Chúng không
> cùng thực hiện một nhiệm vụ:
>
> | Lease | Mục đích | Thành phần gia hạn |
> | --- | --- | --- |
> | `kube-controller-manager` | Bầu leader giữa các instance `kube-controller-manager` | Instance controller manager đang giữ vai trò leader |
> | `kube-scheduler` | Bầu leader giữa các instance `kube-scheduler` | Instance scheduler đang giữ vai trò leader |
> | `apiserver-<hash>` | Công bố định danh của một instance `kube-apiserver`; **không** dùng để bầu leader | Chính instance `kube-apiserver` tương ứng |
>
> Với hai Lease bầu leader, `spec.holderIdentity` (`HOLDER`) nhận diện instance
> đang giữ leadership, `spec.renewTime` (`RENEW`) là thời điểm gia hạn gần nhất,
> còn `spec.leaseDurationSeconds` (`DURATION`) là khoảng thời gian leadership còn
> hiệu lực nếu holder không tiếp tục gia hạn. Trong cấu hình HA, khi Lease hết hạn,
> instance khác có thể tranh quyền leader.
>
> Với Lease `apiserver-<hash>`, `HOLDER` là định danh do kube-apiserver công bố;
> label `kubernetes.io/hostname` (`HOST`) cho biết hostname của instance đó. Mỗi
> kube-apiserver công bố một Lease định danh riêng, vì vậy cluster chỉ có một
> kube-apiserver thường có đúng một Lease loại này. Nói ngắn gọn, hai Lease đầu
> trả lời câu hỏi **instance nào đang là leader**, còn Lease thứ ba giúp nhận biết
> **những kube-apiserver nào đang tồn tại**.

### Giải phóng khóa của kube-controller-manager khi thoát (Kube controller manager lock release on exit)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Khi feature gate `ControllerManagerReleaseLeaderElectionLockOnExit` được bật,
`kube-controller-manager` chủ động giải phóng khóa bầu chọn leader của nó trong
quá trình chuyển giao leader, thay vì chờ TTL của khóa hết hạn. Điều này cho phép
leader mới được bầu nhanh hơn, giảm độ trễ khi chuyển giao leader.

## Định danh của API server (API server identity)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.26 [beta]`

Bắt đầu từ Kubernetes v1.26, mỗi `kube-apiserver` dùng Lease API để công bố định danh (identity)
của nó cho phần còn lại của hệ thống. Tuy bản thân điều này chưa thực sự hữu ích, nó cung cấp
một cơ chế để các client khám phá xem có bao nhiêu instance `kube-apiserver` đang vận hành
control plane của Kubernetes. Sự tồn tại của các lease kube-apiserver mở đường cho những khả năng
trong tương lai có thể cần sự điều phối giữa các kube-apiserver.

Bạn có thể xem các Lease thuộc sở hữu của từng kube-apiserver bằng cách kiểm tra các đối tượng lease
trong namespace `kube-system` với tên `apiserver-<sha256-hash>`. Hoặc bạn có thể dùng
label selector `apiserver.kubernetes.io/identity=kube-apiserver`:

```shell
kubectl -n kube-system get lease -l apiserver.kubernetes.io/identity=kube-apiserver
```
```
NAME                                        HOLDER                                                                           AGE
apiserver-07a5ea9b9b072c4a5f3d1c3702        apiserver-07a5ea9b9b072c4a5f3d1c3702_0c8914f7-0f35-440e-8676-7844977d3a05        5m33s
apiserver-7be9e061c59d368b3ddaf1376e        apiserver-7be9e061c59d368b3ddaf1376e_84f2a85d-37c1-4b14-b6b9-603e62e4896f        4m23s
apiserver-1dfef752bcb36637d2763d1868        apiserver-1dfef752bcb36637d2763d1868_c5ffa286-8a9a-45d4-91e7-61118ed58d2e        4m43s

```

Giá trị hash SHA256 dùng trong tên lease dựa trên hostname của hệ điều hành mà API server đó nhìn thấy.
Mỗi kube-apiserver nên được cấu hình để dùng một hostname duy nhất trong phạm vi cluster.
Các instance mới của kube-apiserver dùng cùng hostname sẽ tiếp quản các Lease hiện có
bằng một holder identity mới, thay vì tạo ra các đối tượng Lease mới. Bạn có thể kiểm tra
hostname mà kube-apiserver sử dụng bằng cách xem giá trị của label `kubernetes.io/hostname`:

```shell
kubectl -n kube-system get lease apiserver-07a5ea9b9b072c4a5f3d1c3702 -o yaml
```
```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  creationTimestamp: "2023-07-02T13:16:48Z"
  labels:
    apiserver.kubernetes.io/identity: kube-apiserver
    kubernetes.io/hostname: master-1
  name: apiserver-07a5ea9b9b072c4a5f3d1c3702
  namespace: kube-system
  resourceVersion: "334899"
  uid: 90870ab5-1ba9-4523-b215-e4d4e662acb1
spec:
  holderIdentity: apiserver-07a5ea9b9b072c4a5f3d1c3702_0c8914f7-0f35-440e-8676-7844977d3a05
  leaseDurationSeconds: 3600
  renewTime: "2023-07-04T21:58:48.065888Z"
```

Các lease đã hết hạn của những kube-apiserver không còn tồn tại sẽ được các kube-apiserver mới
thu gom rác (garbage collect) sau 1 giờ.

Bạn có thể tắt các lease định danh API server bằng cách tắt
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) `APIServerIdentity`.

## Workload {#custom-workload}

Workload của riêng bạn có thể tự định nghĩa cách sử dụng Lease. Ví dụ, bạn có thể chạy một
controller tùy chỉnh trong đó một thành viên chính (primary) hoặc leader thực hiện những thao tác
mà các thành viên ngang hàng của nó không làm. Bạn định nghĩa một Lease để các bản sao (replica)
của controller có thể chọn hoặc bầu một leader, dùng Kubernetes API để điều phối.
Nếu dùng Lease, thực hành tốt là đặt tên cho Lease sao cho có liên hệ rõ ràng với
sản phẩm hoặc thành phần. Ví dụ, nếu bạn có thành phần tên là Example Foo, hãy dùng Lease
tên là `example-foo`.

Nếu một người vận hành cluster hoặc một người dùng cuối khác có thể triển khai nhiều instance
của một thành phần, hãy chọn một tiền tố tên và một cơ chế (chẳng hạn hash của tên Deployment)
để tránh xung đột tên giữa các Lease.

Bạn có thể dùng một cách tiếp cận khác miễn là đạt được cùng kết quả: các sản phẩm phần mềm
khác nhau không xung đột với nhau.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

1. Lease phục vụ hai mục đích rất khác nhau trong Kubernetes. Đó là hai mục đích nào?
2. Ở Lab 1a bạn theo dõi `spec.renewTime` của Lease `lab-k8s-worker2`. Field đó do thành phần nào
   cập nhật, và bằng thao tác API gì?
3. Lease của một Node nằm ở namespace nào và có tên là gì?
4. Ba control-plane node đều chạy `kube-scheduler`. Vì sao chỉ một bản được hoạt động, và cơ
   chế nào bảo đảm điều đó?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Heartbeat của node** (kubelet báo mình còn sống) và **bầu chọn leader** giữa các instance
   của một thành phần control plane.
2. **Kubelet** trên chính node đó cập nhật, bằng một request **update** tới object Lease. Bài
   nói rõ: bên dưới, mỗi heartbeat của kubelet là một request update làm mới `spec.renewTime`.
   Control plane dùng dấu thời gian này để xác định node còn khả dụng hay không.
3. Namespace **`kube-node-lease`**, và Lease **trùng tên với Node**.
4. Vì logic của scheduler phải là "một người quyết" — hai scheduler cùng gán Pod sẽ giẫm chân
   nhau. Cơ chế là **leader election dựa trên Lease**: chỉ instance giữ được Lease mới hoạt
   động, các instance khác ở trạng thái chờ.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

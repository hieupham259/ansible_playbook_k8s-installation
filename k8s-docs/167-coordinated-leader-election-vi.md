# Bầu chọn leader có phối hợp (Coordinated Leader Election)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/concepts/cluster-administration/coordinated-leader-election/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 12](LO-TRINH-ADMIN.md#giai-đoạn-12--quản-trị-cluster-nâng-cao), bài 7/8 ·
Kiểm chứng ở Lab 12 (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài này nối hai thứ bạn đã có: [Lease](35-leases-vi.md) từ nhóm 1c, và
[phiên bản giả lập](168-compatibility-version-vi.md) vừa đọc ở bài trước. Phần dài nhất của bài
thực ra là **ôn lại cơ chế bầu chọn leader thông thường**; phần mới chỉ là cách chọn ra ai thắng.
Đọc theo đúng thứ tự đó: nắm cơ chế Lease trước, rồi mới xem "có phối hợp" thêm gì vào.

**Phải hiểu ở lần đọc này:**

- Cơ chế nền: **Lease là một khóa phân tán gọn nhẹ do API server giữ**. Các instance watch hoặc
  định kỳ đọc Lease để biết ai đang là leader; khi Lease không tồn tại hoặc hết hạn (hiện tại >
  `renewTime` + `leaseDurationSeconds`), các ứng viên cùng thử cập nhật nó.
- Vì sao chỉ một bên thắng: **điều khiển đồng thời lạc quan qua `resourceVersion`** — các lần cập
  nhật đồng thời bị lệch phiên bản nên chỉ một lần thành công. Leader sau đó gia hạn `renewTime`
  định kỳ (ví dụ mỗi `leaseDurationSeconds` ÷ 2); ngừng gia hạn thì Lease hết hạn và có cuộc bầu
  chọn mới.
- Các trường của Lease và ý nghĩa: `holderIdentity`, `acquireTime`, `renewTime`,
  `leaseDurationSeconds`, `leaseTransitions`.
- Điểm mới của **bầu chọn có phối hợp**: leader được chọn **một cách xác định** thay vì ai giành
  được trước. Chiến lược dựng sẵn duy nhất hiện nay là **`OldestEmulationVersion`** — ưu tiên
  emulation version thấp nhất, rồi binary version, rồi creation timestamp. Nó phục vụ ràng buộc
  độ lệch phiên bản khi nâng cấp cluster.
- Cách hiện thực: các thành phần **đăng ký ứng viên bằng đối tượng LeaseCandidate** mang danh
  tính, binary version và emulation version; rồi các ứng viên vẫn phối hợp qua **một Lease dùng
  chung**. Tính năng ở mức beta, **mặc định tắt**, cần feature gate `CoordinatedLeaderElection`
  **và** API group `coordination.k8s.io/v1beta1`; trong Kubernetes 1.36 chỉ
  kube-controller-manager và kube-scheduler tự dùng nó.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Hai flag bật tính năng: `--feature-gates` và `--runtime-config` trên API Server | phải sửa manifest static Pod của kube-apiserver | CP5 cấu hình lại cluster đang chạy |
| Ràng buộc độ lệch phiên bản (version skew) mà tính năng này phục vụ | chỉ thành vấn đề thật trong một lần nâng cấp cluster nhiều control plane | CP2 nâng cấp cluster |
| Đặc tả đầy đủ của API LeaseCandidate `v1beta1` | còn ở mức beta, các trường có thể đổi | CP2 nâng cấp cluster |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.33 [beta]` (mặc định tắt)

Kubernetes 1.36 bao gồm một tính năng beta cho phép các thành phần
control plane lựa chọn leader một cách xác định (deterministic) thông qua
_bầu chọn leader có phối hợp_ (coordinated leader election).
Điều này hữu ích để thỏa mãn các ràng buộc về độ lệch phiên bản (version skew) của Kubernetes trong quá trình nâng cấp cluster.
Hiện tại, chiến lược lựa chọn dựng sẵn duy nhất là `OldestEmulationVersion`,
ưu tiên leader có emulation version thấp nhất, tiếp theo là binary
version, rồi đến thời điểm tạo (creation timestamp).

## Bật bầu chọn leader có phối hợp (Enabling coordinated leader election)

Hãy đảm bảo rằng [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`CoordinatedLeaderElection` được bật khi bạn khởi động API Server,
và API group `coordination.k8s.io/v1beta1` cũng được bật.

Việc này có thể được thực hiện bằng cách đặt các flag `--feature-gates="CoordinatedLeaderElection=true"` và
`--runtime-config="coordination.k8s.io/v1beta1=true"`.

## Cấu hình thành phần (Component configuration)

Với điều kiện bạn đã bật feature gate `CoordinatedLeaderElection` _và_
đã bật API group `coordination.k8s.io/v1beta1`, các thành phần control plane
tương thích sẽ tự động sử dụng các API LeaseCandidate và Lease để bầu chọn leader
khi cần.

Đối với Kubernetes 1.36, hai thành phần control plane
(kube-controller-manager và kube-scheduler) tự động sử dụng bầu chọn
leader có phối hợp khi feature gate và API group được bật.

## Lựa chọn leader cho các thành phần Kubernetes (Leader selection for Kubernetes components)

Kubernetes sử dụng [Lease API](./35-leases-vi.md) để thực hiện bầu chọn leader giữa nhiều instance của cùng một thành phần control plane trong một cluster có tính sẵn sàng cao (high availability), chẳng hạn `kube-controller-manager` hoặc `kube-scheduler`.

Một [Lease](./35-leases-vi.md) hoạt động như một khóa phân tán (distributed lock) gọn nhẹ, được lưu bởi [Kubernetes API server](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/).
Tất cả các instance đang chạy của một thành phần sẽ watch hoặc định kỳ đọc đối tượng Lease liên quan
để xác định instance nào hiện đang đóng vai trò leader.

[Lease API](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/lease-v1/) định nghĩa các trường
như:

`holderIdentity`
: danh tính (ví dụ: tên pod hoặc chuỗi dựa trên hostname) của leader hiện tại.

`acquireTime`
: mốc thời gian (timestamp) khi giành được quyền leader.

`renewTime`
: mốc thời gian của lần gia hạn gần nhất bởi leader.

`leaseDurationSeconds`
: khoảng thời gian hiệu lực của lease (các ứng viên nên chờ hết khoảng thời gian này cộng thêm một khoảng ân hạn (grace period) nhỏ trước khi thử giành một lease đã hết hạn).

`leaseTransitions`
: bộ đếm số lần quyền leader đã đổi chủ.

Các trường này cho biết instance nào đang giữ quyền leader và quyền leader đó còn hiệu lực trong bao lâu.

Khi [Lease](./35-leases-vi.md) không tồn tại hoặc đã hết hạn (thời gian hiện tại > `renewTime` + `leaseDurationSeconds`), các instance ứng viên sẽ cố gắng cập nhật Lease với danh tính của mình. Kubernetes dựa vào _điều khiển đồng thời lạc quan_ (optimistic concurrency control) thông qua `resourceVersion` của đối tượng: chỉ một lần cập nhật thành công do các lần thử đồng thời bị lệch phiên bản (version mismatch). Instance nào có lần cập nhật được chấp nhận sẽ trở thành _leader_.

Kubernetes sử dụng API [LeaseCandidate](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/lease-candidate-v1beta1/)
để quản lý các cuộc bầu chọn leader. Các thành phần control plane như `kube-controller-manager` và `kube-scheduler` đăng ký vai trò ứng viên của mình bằng cách tạo các đối tượng LeaseCandidate; các đối tượng này theo dõi tất cả các instance đang cạnh tranh quyền leader và mang theo metadata bao gồm danh tính của ứng viên, binary version và emulation version.

Trong một cuộc bầu chọn, các ứng viên phối hợp với nhau thông qua một [Lease](./35-leases-vi.md) dùng chung.
Control plane của Kubernetes đảm bảo rằng chỉ một ứng viên giành được [Lease](./35-leases-vi.md) thành công và đảm nhận vai trò _leader_, trong khi tất cả các ứng viên còn lại vẫn là follower. Nếu _leader_ hiện tại không gia hạn được [Lease](./35-leases-vi.md) trong khoảng thời gian timeout đã chọn, các ứng viên còn lại sẽ cạnh tranh để giành quyền leader và bầu ra một _leader_ mới.

Sau khi được bầu, leader định kỳ gia hạn Lease của mình bằng cách cập nhật trường `renewTime`

(ví dụ, thực hiện gia hạn mỗi `leaseDurationSeconds` ÷ 2, nhằm tránh xung đột khi [Lease](./35-leases-vi.md) sắp hết hạn).
Chừng nào việc gia hạn còn diễn ra trước khi lease hết hạn, instance leader hiện tại vẫn giữ quyền leader.
Nếu leader gặp sự cố (crash), trở nên không thể liên lạc được, hoặc ngừng gia hạn Lease, Lease đó sẽ hết hạn. Các instance khỏe mạnh khác phát hiện Lease đã hết hạn và tiến hành một cuộc bầu chọn mới.

Cơ chế này đảm bảo rằng mặc dù nhiều bản sao (replica) của một thành phần có thể đang chạy để phục vụ tính ổn định và khả năng phục hồi, _chỉ một instance chủ động thực hiện các tác vụ điều khiển tại một thời điểm_, trong khi các instance còn lại ở trạng thái chờ (standby), theo dõi Lease và sẵn sàng tiếp quản nhanh chóng khi cần.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 12:

1. Cluster lab chỉ có một control plane `k8s-master`, nên kube-scheduler và kube-controller-manager
   mỗi cái đúng một instance. Theo lập luận của bài, cơ chế Lease bảo đảm điều gì — và điều đó chỉ
   thực sự có giá trị trong hoàn cảnh nào?
2. Hai instance cùng phát hiện một Lease đã hết hạn và cùng lúc thử ghi danh tính của mình vào đó.
   Cái gì bảo đảm chỉ một bên trở thành leader?
3. **Câu bẫy.** Bật feature gate `CoordinatedLeaderElection` rồi thì cơ chế Lease có bị thay thế
   không? Khác biệt thật giữa bầu chọn thông thường và bầu chọn có phối hợp nằm ở đâu?
4. `OldestEmulationVersion` xếp thứ tự ứng viên theo ba tiêu chí nào, và vì sao lại chọn "cũ
   nhất" thay vì "mới nhất"?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Nó bảo đảm **chỉ một instance chủ động thực hiện các tác vụ điều khiển tại một thời điểm**,
   các instance còn lại ở trạng thái chờ và theo dõi Lease để tiếp quản. Với một instance duy nhất
   thì cơ chế vẫn chạy nhưng **không có gì để tranh**: giá trị thật của nó chỉ xuất hiện ở
   **cluster có tính sẵn sàng cao**, tức khi mỗi thành phần có nhiều replica — trên lộ trình là từ
   các lab HA của giai đoạn 8 trở đi.
2. **Điều khiển đồng thời lạc quan qua `resourceVersion` của đối tượng.** Cả hai đọc cùng một
   `resourceVersion` rồi cùng gửi cập nhật; API server chỉ chấp nhận lần cập nhật khớp phiên bản,
   lần còn lại bị từ chối vì lệch phiên bản. **Instance có lần cập nhật được chấp nhận trở thành
   leader.**
3. **Không bị thay thế.** Bài nói rõ các ứng viên "phối hợp với nhau thông qua **một Lease dùng
   chung**"; cái được thêm vào là **API LeaseCandidate**, nơi mỗi instance đăng ký danh tính kèm
   **binary version và emulation version** của mình. Khác biệt thật nằm ở **cách chọn ai thắng**:
   bầu chọn thông thường về bản chất là ai giành được Lease trước thì thắng, còn bầu chọn có phối
   hợp chọn leader **một cách xác định** theo một chiến lược đã khai báo. Chỗ dễ nhầm là tưởng
   tính năng mới thay hẳn cơ chế cũ; nó chồng lên trên cơ chế cũ.
4. Theo thứ tự: **emulation version thấp nhất → binary version → thời điểm tạo (creation
   timestamp)**. Chọn "cũ nhất" vì mục đích của tính năng là **thỏa mãn các ràng buộc về độ lệch
   phiên bản trong quá trình nâng cấp cluster**: leader là instance có hành vi cũ nhất thì nó
   không dùng những khả năng mà các thành phần chưa nâng cấp còn lại không hiểu.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

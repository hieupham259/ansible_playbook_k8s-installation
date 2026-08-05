# Bầu chọn leader có phối hợp (Coordinated Leader Election)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/concepts/cluster-administration/coordinated-leader-election/

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

# Các Node (Nodes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/architecture/nodes/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 1 → nhóm [1a](00-ALO-TRINH-ADMIN.md#1a-kiến-trúc-và-mô-hình-điều-khiển),
bài 6/8 · Kiểm chứng ở [Lab 1a](labs/LAB-1A-KIEN-TRUC-VA-MO-HINH-DIEU-KHIEN.md) phần B6.

Nửa sau của bài nói về eviction, taint và availability zone — thuộc giai đoạn 7 và 12. Ở đây
chỉ cần biết những cơ chế đó tồn tại.

**Phải hiểu ở lần đọc này:**

- Hai cách một Node vào cluster: **kubelet tự đăng ký** (mặc định, và là cách cluster lab của
  bạn dùng) hoặc bạn tạo object thủ công.
- Tên Node phải là DNS subdomain hợp lệ và **duy nhất tại một thời điểm**; đổi máy mà giữ tên
  thì phải xóa object Node cũ trước.
- Bốn nhóm thông tin trong Node status: `Addresses`, `Conditions`, `Capacity`/`Allocatable`,
  `Info`.
- **Hai dạng heartbeat**: cập nhật `.status` của Node, và đối tượng Lease trong namespace
  `kube-node-lease`. Phân biệt được hai dạng này là trọng tâm của bài.
- Node controller làm gì khi mất heartbeat: đặt condition `Ready` thành `Unknown`, rồi nếu vẫn
  mất liên lạc thì kích hoạt eviction.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| `--register-with-taints` | taint là chủ đề lập lịch | giai đoạn 7 |
| `--node-labels` và node selector | label là nhóm bài kế tiếp | nhóm 1b |
| Node authorization mode, NodeRestriction | thuộc kiểm soát truy cập | giai đoạn 9 |
| `kubectl cordon`, drain, DaemonSet tolerate | quy trình bảo trì node | giai đoạn 12 và giai đoạn 16 |
| *Giới hạn tốc độ trục xuất*, availability zone | eviction là chủ đề riêng | giai đoạn 7 |
| *Theo dõi dung lượng tài nguyên*, `requests` của container | cần hiểu tài nguyên Pod trước | giai đoạn 3 |
| *Topology của node* | tính năng nâng cao của kubelet | giai đoạn 7 |

Đừng học thuộc các con số mặc định (5 giây, 5 phút, `0.1`/giây). Chúng là giá trị cấu hình
được, không phải cam kết của hệ thống.

---

Kubernetes chạy workload của bạn bằng cách đặt các container vào trong các Pod để chạy trên các _Node_.
Một node có thể là máy ảo hoặc máy vật lý, tùy thuộc vào cluster. Mỗi node
được quản lý bởi control plane
và chứa các service cần thiết để chạy các Pod.

Thông thường bạn có nhiều node trong một cluster; trong môi trường học tập hoặc bị giới hạn
tài nguyên, bạn có thể chỉ có một node duy nhất.

Các [thành phần](./22-architecture-vi.md#node-components) trên một node bao gồm
kubelet, một container runtime, và kube-proxy.

## Quản lý (Management)

Có hai cách chính để thêm các Node vào API server:

1. kubelet trên một node tự đăng ký (self-register) với control plane
2. Bạn (hoặc một người dùng khác) thêm thủ công một đối tượng Node

Sau khi bạn tạo một đối tượng (object) Node,
hoặc kubelet trên một node tự đăng ký, control plane sẽ kiểm tra xem đối tượng Node mới
có hợp lệ hay không. Ví dụ, nếu bạn thử tạo một Node từ manifest JSON sau:

```json
{
  "kind": "Node",
  "apiVersion": "v1",
  "metadata": {
    "name": "10.240.79.157",
    "labels": {
      "name": "my-first-k8s-node"
    }
  }
}
```

Kubernetes tạo một đối tượng Node trong nội bộ (phần biểu diễn — representation). Kubernetes kiểm tra
rằng đã có một kubelet đăng ký với API server khớp với trường `metadata.name`
của Node. Nếu node khỏe mạnh (healthy — tức là tất cả các service cần thiết đang chạy),
thì nó đủ điều kiện để chạy Pod. Ngược lại, node đó sẽ bị bỏ qua trong mọi hoạt động của cluster
cho đến khi nó trở nên khỏe mạnh.

> **Ghi chú:**
>
> Kubernetes vẫn giữ lại đối tượng của Node không hợp lệ và tiếp tục kiểm tra xem
> nó có trở nên khỏe mạnh hay không.
>
> Bạn, hoặc một controller, phải xóa đối tượng Node một cách tường minh
> để dừng việc kiểm tra sức khỏe (health checking) đó.

Tên của một đối tượng Node phải là một
[tên DNS subdomain hợp lệ](17-names-vi.md#dns-subdomain-names).

### Tính duy nhất của tên Node (Node name uniqueness) {#node-name-uniqueness}

[Tên](./17-names-vi.md#names) định danh một Node. Hai Node
không thể có cùng tên tại cùng một thời điểm. Kubernetes cũng giả định rằng một tài nguyên có cùng
tên là cùng một đối tượng. Trong trường hợp của Node, hệ thống ngầm giả định rằng một instance dùng
cùng tên sẽ có cùng trạng thái (ví dụ: cấu hình mạng, nội dung đĩa gốc) và cùng các thuộc tính như
label của node. Điều này có thể dẫn đến sự không nhất quán nếu một instance bị thay đổi mà không đổi tên.
Nếu Node cần được thay thế hoặc cập nhật đáng kể, đối tượng Node hiện có cần được
xóa khỏi API server trước, rồi thêm lại sau khi cập nhật.

### Tự đăng ký của Node (Self-registration of Nodes) {#self-registration-of-nodes}

Khi cờ (flag) `--register-node` của kubelet là true (mặc định), kubelet sẽ cố gắng
tự đăng ký với API server. Đây là mẫu hình được ưu tiên, được hầu hết các bản phân phối (distro) sử dụng.

Để tự đăng ký, kubelet được khởi động với các tùy chọn sau:

- `--kubeconfig` - Đường dẫn tới thông tin xác thực (credentials) để tự xác thực với API server.
- `--cloud-provider` - Cách giao tiếp với một nhà cung cấp cloud (cloud provider)
  để đọc metadata về chính node đó.
- `--register-node` - Tự động đăng ký với API server.
- `--register-with-taints` - Đăng ký node với danh sách các
  taint cho trước (phân tách bằng dấu phẩy, dạng `<key>=<value>:<effect>`).

  Không có tác dụng (no-op) nếu `register-node` là false.
- `--node-ip` - Danh sách tùy chọn các địa chỉ IP cho node, phân tách bằng dấu phẩy.
  Bạn chỉ có thể chỉ định một địa chỉ duy nhất cho mỗi họ địa chỉ (address family).
  Ví dụ, trong một cluster IPv4 single-stack, bạn đặt giá trị này là địa chỉ IPv4 mà
  kubelet nên dùng cho node.
  Xem [cấu hình dual stack IPv4/IPv6](85-dual-stack-vi.md#configure-ipv4-ipv6-dual-stack)
  để biết chi tiết về việc chạy cluster dual-stack.

  Nếu bạn không cung cấp đối số này, kubelet sẽ dùng địa chỉ IPv4 mặc định của node, nếu có;
  nếu node không có địa chỉ IPv4 nào thì kubelet dùng địa chỉ IPv6 mặc định của node.
- `--node-labels` - Các label để thêm vào khi đăng ký node
  trong cluster (xem các hạn chế về label được
  [NodeRestriction admission plugin](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction) áp đặt).
- `--node-status-update-frequency` - Chỉ định tần suất kubelet gửi trạng thái node của nó lên API server.

Khi [chế độ ủy quyền Node (Node authorization mode)](https://kubernetes.io/docs/reference/access-authn-authz/node/) và
[NodeRestriction admission plugin](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction)
được bật, các kubelet chỉ được phép tạo/sửa tài nguyên Node của chính chúng.

> **Ghi chú:**
>
> Như đã đề cập trong mục [Tính duy nhất của tên Node](#node-name-uniqueness),
> khi cần cập nhật cấu hình của Node, một thực hành tốt là đăng ký lại
> node đó với API server. Ví dụ, nếu kubelet được khởi động lại với
> một tập `--node-labels` mới nhưng vẫn dùng cùng tên Node, thay đổi sẽ
> không có hiệu lực, vì các label chỉ được thiết lập (hoặc thay đổi) khi Node đăng ký với API server.
>
> Các Pod đã được lập lịch trên Node có thể hoạt động sai hoặc gây ra sự cố nếu cấu hình Node
> bị thay đổi khi kubelet khởi động lại. Ví dụ, một Pod đang chạy
> có thể xung khắc (tainted) với các label mới được gán cho Node, trong khi các
> Pod khác vốn không tương thích với Pod đó lại được lập lịch dựa trên label
> mới này. Việc đăng ký lại node đảm bảo tất cả các Pod sẽ được rút cạn (drain) và
> được lập lịch lại một cách đúng đắn.

### Quản trị Node thủ công (Manual Node administration) {#manual-node-administration}

Bạn có thể tạo và sửa các đối tượng Node bằng kubectl.

Khi bạn muốn tạo các đối tượng Node thủ công, hãy đặt cờ kubelet `--register-node=false`.

Bạn có thể sửa các đối tượng Node bất kể giá trị của `--register-node` là gì.
Ví dụ, bạn có thể gán label cho một Node hiện có hoặc đánh dấu nó là không thể lập lịch (unschedulable).

Bạn có thể đặt (các) vai trò (role) tùy chọn cho node bằng cách thêm một hoặc nhiều label dạng `node-role.kubernetes.io/<role>: <role>` cho node, trong đó các ký tự của `<role>`
bị giới hạn bởi các quy tắc [cú pháp](18-labels-vi.md#syntax-and-character-set) dành cho label.

Kubernetes bỏ qua giá trị label đối với vai trò của node; theo quy ước, bạn có thể đặt nó thành cùng chuỗi mà bạn đã dùng cho vai trò node trong khóa (key) của label.

Bạn có thể dùng label trên các Node kết hợp với node selector trên các Pod để kiểm soát
việc lập lịch. Ví dụ, bạn có thể ràng buộc một Pod chỉ đủ điều kiện chạy trên
một tập con các node hiện có.

Đánh dấu một node là unschedulable sẽ ngăn scheduler đặt các pod mới lên
Node đó, nhưng không ảnh hưởng đến các Pod đang tồn tại trên Node. Điều này hữu ích như một
bước chuẩn bị trước khi khởi động lại node hoặc thực hiện việc bảo trì khác.

Để đánh dấu một Node là unschedulable, chạy:

```shell
kubectl cordon $NODENAME
```

Xem [Rút cạn một Node an toàn (Safely Drain a Node)](255-safely-drain-node-vi.md)
để biết thêm chi tiết.

> **Ghi chú:**
>
> Các Pod thuộc một DaemonSet chấp nhận (tolerate)
> việc chạy trên một Node unschedulable. DaemonSet thường cung cấp các service cục bộ của node (node-local)
> cần tiếp tục chạy trên Node ngay cả khi Node đó đang được rút cạn các ứng dụng workload.

## Trạng thái của Node (Node status)

Trạng thái (status) của một Node chứa các thông tin sau:

* [Addresses (Địa chỉ)](https://kubernetes.io/docs/reference/node/node-status/#addresses)
* [Conditions (Điều kiện)](https://kubernetes.io/docs/reference/node/node-status/#condition)
* [Capacity và Allocatable (Dung lượng và phần cấp phát được)](https://kubernetes.io/docs/reference/node/node-status/#capacity)
* [Info (Thông tin)](https://kubernetes.io/docs/reference/node/node-status/#info)

Bạn có thể dùng `kubectl` để xem trạng thái cùng các chi tiết khác của một Node:

```shell
kubectl describe node <insert-node-name-here>
```

Xem [Trạng thái Node (Node Status)](https://kubernetes.io/docs/reference/node/node-status/) để biết thêm chi tiết.

## Nhịp tim của Node (Node heartbeats) {#node-heartbeats}

Các nhịp tim (heartbeat), do các node Kubernetes gửi đi, giúp cluster của bạn xác định
mức độ khả dụng của từng node, và thực hiện hành động khi phát hiện sự cố.

Đối với các node, có hai dạng heartbeat:

* Cập nhật vào [`.status`](https://kubernetes.io/docs/reference/node/node-status/) của một Node.
* Các đối tượng [Lease](35-leases-vi.md)
  bên trong namespace `kube-node-lease`.
  Mỗi Node có một đối tượng Lease tương ứng.

## Node controller

Node controller là một
thành phần control plane của Kubernetes, quản lý nhiều khía cạnh khác nhau của các node.

Node controller có nhiều vai trò trong vòng đời của một node. Vai trò thứ nhất là gán một
khối CIDR cho node khi node được đăng ký (nếu tính năng gán CIDR được bật).

Vai trò thứ hai là giữ cho danh sách node nội bộ của node controller luôn khớp với
danh sách các máy khả dụng của nhà cung cấp cloud. Khi chạy trong môi trường
cloud và mỗi khi một node không khỏe mạnh (unhealthy), node controller sẽ hỏi nhà cung cấp
cloud xem VM của node đó có còn khả dụng hay không. Nếu không, node
controller sẽ xóa node đó khỏi danh sách node của nó.

Vai trò thứ ba là giám sát sức khỏe của các node. Node controller chịu
trách nhiệm:

- Trong trường hợp một node trở nên không thể truy cập được (unreachable), cập nhật điều kiện `Ready`
  trong trường `.status` của Node. Trong trường hợp này, node controller đặt
  điều kiện `Ready` thành `Unknown`.
- Nếu node vẫn tiếp tục không thể truy cập được: kích hoạt
  [trục xuất khởi phát qua API (API-initiated eviction)](143-api-eviction-vi.md)
  cho tất cả các Pod trên node không thể truy cập đó. Theo mặc định, node controller
  đợi 5 phút giữa thời điểm đánh dấu node là `Unknown` và thời điểm gửi
  yêu cầu trục xuất (eviction) đầu tiên.

Theo mặc định, node controller kiểm tra trạng thái của từng node mỗi 5 giây.
Chu kỳ này có thể được cấu hình bằng cờ `--node-monitor-period` trên
thành phần `kube-controller-manager`.

### Giới hạn tốc độ trục xuất (Rate limits on eviction)

Trong hầu hết các trường hợp, node controller giới hạn tốc độ trục xuất ở mức
`--node-eviction-rate` (mặc định 0.1) mỗi giây, nghĩa là nó sẽ không trục xuất pod
từ nhiều hơn 1 node trong mỗi 10 giây.

Hành vi trục xuất của node thay đổi khi một node trong một availability zone (vùng khả dụng) nào đó
trở nên không khỏe mạnh. Node controller kiểm tra tỷ lệ phần trăm các node trong zone đó
không khỏe mạnh (điều kiện `Ready` là `Unknown` hoặc `False`) tại cùng một thời điểm:

- Nếu tỷ lệ node không khỏe mạnh tối thiểu là `--unhealthy-zone-threshold`
  (mặc định 0.55), thì tốc độ trục xuất được giảm xuống.
- Nếu cluster nhỏ (tức là có số node nhỏ hơn hoặc bằng
  `--large-cluster-size-threshold` — mặc định 50), thì việc trục xuất bị dừng lại.
- Ngược lại, tốc độ trục xuất được giảm xuống `--secondary-node-eviction-rate`
  (mặc định 0.01) mỗi giây.

Lý do các chính sách này được áp dụng theo từng availability zone là vì một
availability zone có thể bị chia cắt (partitioned) khỏi control plane trong khi các zone khác vẫn
duy trì kết nối. Nếu cluster của bạn không trải trên nhiều availability zone của nhà cung cấp cloud,
thì cơ chế trục xuất sẽ không xét đến tính bất khả dụng theo từng zone.

Một lý do quan trọng để phân bổ các node của bạn trên nhiều availability zone là để
workload có thể được chuyển sang các zone khỏe mạnh khi toàn bộ một zone bị sập.
Do đó, nếu tất cả các node trong một zone đều không khỏe mạnh, node controller trục xuất với
tốc độ bình thường `--node-eviction-rate`. Trường hợp biên (corner case) là khi tất cả các zone đều
hoàn toàn không khỏe mạnh (không node nào trong cluster khỏe mạnh). Trong trường hợp
đó, node controller giả định rằng có vấn đề nào đó về kết nối
giữa control plane và các node, và không thực hiện bất kỳ trục xuất nào.
(Nếu đã xảy ra sự cố và một số node xuất hiện trở lại, node controller sẽ
trục xuất pod khỏi các node còn lại đang không khỏe mạnh hoặc không thể truy cập được).

Node controller cũng chịu trách nhiệm trục xuất các pod chạy trên những node có
taint `NoExecute`, trừ khi các pod đó chấp nhận (tolerate) taint ấy.
Node controller cũng thêm các taint
tương ứng với các sự cố của node, như node không thể truy cập được hoặc chưa sẵn sàng (not ready). Điều này có nghĩa
là scheduler sẽ không đặt Pod lên các node không khỏe mạnh.

## Theo dõi dung lượng tài nguyên (Resource capacity tracking) {#node-capacity}

Các đối tượng Node theo dõi thông tin về dung lượng tài nguyên của Node: ví dụ, lượng
bộ nhớ khả dụng và số lượng CPU.
Các node [tự đăng ký](#self-registration-of-nodes) sẽ báo cáo dung lượng của chúng trong quá trình
đăng ký. Nếu bạn thêm một Node theo cách [thủ công](#manual-node-administration), thì
bạn cần tự thiết lập thông tin dung lượng của node khi thêm nó.

Scheduler của Kubernetes đảm bảo rằng
có đủ tài nguyên cho tất cả các Pod trên một Node. Scheduler kiểm tra rằng tổng
các request của các container trên node không lớn hơn dung lượng của node.
Tổng request đó bao gồm tất cả các container do kubelet quản lý, nhưng loại trừ mọi
container được container runtime khởi động trực tiếp, và cũng loại trừ mọi
tiến trình chạy ngoài tầm kiểm soát của kubelet.

> **Ghi chú:**
>
> Nếu bạn muốn dành riêng (reserve) tài nguyên một cách tường minh cho các tiến trình không phải Pod, xem
> [dành riêng tài nguyên cho các system daemon](253-reserve-compute-resources-vi.md#system-reserved).

## Topology của node (Node topology)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.27 [stable]`

Nếu bạn đã bật [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`TopologyManager`, thì
kubelet có thể sử dụng các gợi ý topology (topology hints) khi đưa ra quyết định gán tài nguyên.
Xem [Kiểm soát các chính sách quản lý topology trên một Node (Control Topology Management Policies on a Node)](259-topology-manager-vi.md)
để biết thêm thông tin.

## Tiếp theo (What's next)

Tìm hiểu thêm về các chủ đề sau:

* [Các thành phần](./22-architecture-vi.md#node-components) tạo nên một node.
* [Định nghĩa API của Node](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#node-v1-core).
* Mục [Node](https://git.k8s.io/design-proposals-archive/architecture/architecture.md#the-kubernetes-node)
  trong tài liệu thiết kế kiến trúc.
* [Tắt node êm thấm/không êm thấm (Graceful/non-graceful node shutdown)](169-node-shutdown-vi.md).
* [Tự động co giãn node (Node autoscaling)](171-node-autoscaling-vi.md) để
  quản lý số lượng và kích cỡ các node trong cluster của bạn.
* [Taint và Toleration](139-taint-and-toleration-vi.md).
* [Node Resource Managers](https://kubernetes.io/docs/concepts/policy/node-resource-managers/).
* [Quản lý tài nguyên cho các node Windows (Resource Management for Windows nodes)](112-windows-resource-management-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 1:

1. Hai worker trong cluster lab của bạn vào cluster bằng cách nào — bạn tạo object Node, hay
   kubelet tự đăng ký?
2. Kể bốn nhóm thông tin trong Node status, và cho một ví dụ cụ thể của từng nhóm.
3. Node có hai dạng heartbeat. Chúng là gì, và vì sao lại cần tới hai dạng thay vì một?
4. Bạn dừng `kubelet` trên một worker. Condition `Ready` của Node đó đổi thành gì, và **ai**
   là người đổi nó?
5. Bạn thay một máy worker bằng máy mới nhưng giữ nguyên hostname. Vì sao nên xóa object Node
   cũ trước?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Kubelet tự đăng ký.** Cờ `--register-node` mặc định là `true` và đây là mẫu hình được ưu
   tiên; lệnh `kubeadm join` ở Lab 00 chính là khiến kubelet tự đăng ký với API server. Bạn
   không hề tạo object Node nào bằng tay.
2. **Addresses** — ví dụ `InternalIP` của node. **Conditions** — ví dụ `Ready`.
   **Capacity/Allocatable** — ví dụ `cpu`, `memory`. **Info** — ví dụ `kubeletVersion`,
   `containerRuntimeVersion`, phiên bản kernel.
3. Cập nhật vào **`.status` của Node**, và object **Lease** trong namespace `kube-node-lease`.
   Cần hai vì `.status` mang nhiều thông tin nên cập nhật tốn kém; Lease rất nhẹ nên cập nhật
   được dày, giúp phát hiện node chết nhanh mà không tạo tải lớn lên API server và etcd.
4. Đổi thành **`Unknown`**, và người đổi là **node controller** trong
   `kube-controller-manager` — không phải kubelet, vì kubelet đã chết. Nếu node vẫn không liên
   lạc được, node controller tiếp tục kích hoạt eviction các Pod trên node đó.
5. Vì Kubernetes **giả định cùng tên là cùng một object với cùng trạng thái** — cấu hình mạng,
   nội dung đĩa gốc, các label của node. Máy mới mang tên cũ sẽ thừa hưởng thông tin không còn
   đúng, dẫn tới không nhất quán. Bài khuyên xóa object Node hiện có trước rồi thêm lại sau khi
   cập nhật.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

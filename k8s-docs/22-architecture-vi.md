# Kiến trúc cluster (Cluster Architecture)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/architecture/>
>
> Các khái niệm kiến trúc đằng sau Kubernetes.

Một cluster Kubernetes bao gồm một control plane cùng với một tập các máy worker, gọi là các node,
chuyên chạy các ứng dụng đóng gói trong container (containerized applications). Mỗi cluster cần
ít nhất một worker node để chạy các Pod.

Các worker node lưu trữ (host) các Pod — những thành phần cấu thành workload của ứng dụng.
Control plane quản lý các worker node và các Pod trong cluster. Trong môi trường
production, control plane thường chạy trên nhiều máy tính và cluster
thường chạy nhiều node, nhằm cung cấp khả năng chịu lỗi (fault-tolerance) và tính sẵn sàng cao (high availability).

Tài liệu này phác thảo các thành phần bạn cần có để có một cluster Kubernetes hoàn chỉnh và hoạt động được.

![Control plane (kube-apiserver, etcd, kube-controller-manager, kube-scheduler) và một số node. Mỗi node chạy một kubelet và kube-proxy.](https://kubernetes.io/images/docs/kubernetes-cluster-architecture.svg)

*Hình 1. Các thành phần của cluster Kubernetes.*

> **Về kiến trúc này (About this architecture)**
>
> Sơ đồ trong Hình 1 trình bày một kiến trúc tham chiếu ví dụ cho một cluster Kubernetes.
> Cách phân bố thực tế của các thành phần có thể khác nhau tùy theo cách thiết lập và yêu cầu
> cụ thể của từng cluster.
>
> Trong sơ đồ, mỗi node đều chạy thành phần [`kube-proxy`](#kube-proxy). Bạn cần một
> thành phần network proxy trên mỗi node để đảm bảo API Service và các hành vi liên quan
> khả dụng trên mạng của cluster. Tuy nhiên, một số network plugin tự cung cấp
> hiện thực proxy riêng của bên thứ ba. Khi bạn dùng loại network plugin đó,
> node không cần chạy `kube-proxy`.

## Các thành phần của control plane (Control plane components) {#control-plane-components}

Các thành phần của control plane đưa ra những quyết định toàn cục về cluster (ví dụ: lập lịch — scheduling),
đồng thời phát hiện và phản ứng với các sự kiện của cluster (ví dụ: khởi động một
pod mới khi trường `replicas` của một Deployment chưa được thỏa mãn).

Các thành phần control plane có thể chạy trên bất kỳ máy nào trong cluster. Tuy nhiên, để đơn giản,
các script cài đặt thường khởi động tất cả các thành phần control plane trên cùng một máy,
và không chạy container của người dùng trên máy này.
Xem [Tạo cluster có tính sẵn sàng cao với kubeadm](./08-high-availability-vi.md)
để có ví dụ về một thiết lập control plane chạy trên nhiều máy.

### kube-apiserver

API server là một thành phần của control plane Kubernetes, chịu trách nhiệm cung cấp (expose) API của Kubernetes.
API server là mặt tiền (front end) của control plane Kubernetes.

Hiện thực chính của API server Kubernetes là [kube-apiserver](https://kubernetes.io/docs/reference/generated/kube-apiserver/).
kube-apiserver được thiết kế để mở rộng theo chiều ngang (scale horizontally) — nghĩa là nó mở rộng bằng cách triển khai thêm nhiều instance.
Bạn có thể chạy nhiều instance của kube-apiserver và cân bằng lưu lượng giữa các instance đó.

### etcd

Kho lưu trữ key-value nhất quán (consistent) và có tính sẵn sàng cao, được dùng làm nơi lưu trữ nền (backing store) cho toàn bộ dữ liệu cluster của Kubernetes.

Nếu cluster Kubernetes của bạn dùng etcd làm backing store, hãy đảm bảo bạn có kế hoạch
[sao lưu (back up)](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster)
cho dữ liệu này.

Bạn có thể tìm thông tin chuyên sâu về etcd trong [tài liệu](https://etcd.io/docs/) chính thức.

### kube-scheduler

Thành phần control plane theo dõi các Pod mới được tạo mà chưa được gán
node, và chọn một node để chúng chạy trên đó.

Các yếu tố được cân nhắc trong quyết định lập lịch bao gồm:
yêu cầu tài nguyên (resource) của từng Pod và của tập hợp Pod, các ràng buộc về phần cứng/phần mềm/chính sách (policy),
các đặc tả affinity và anti-affinity, tính cục bộ của dữ liệu (data locality),
sự ảnh hưởng lẫn nhau giữa các workload, và các thời hạn (deadline).

### kube-controller-manager

Thành phần control plane chạy các tiến trình controller.

Về mặt logic, mỗi controller là một tiến trình riêng biệt, nhưng để giảm độ phức tạp,
tất cả chúng được biên dịch vào một binary duy nhất và chạy trong một tiến trình duy nhất.

Có nhiều loại controller khác nhau. Một vài ví dụ trong số đó là:

- Node controller: Chịu trách nhiệm phát hiện và phản ứng khi các node gặp sự cố (down).
- Job controller: Theo dõi các đối tượng Job đại diện cho các tác vụ chạy một lần (one-off task), sau đó tạo các Pod để chạy các tác vụ đó cho đến khi hoàn thành.
- EndpointSlice controller: Tạo và cập nhật các đối tượng EndpointSlice (để cung cấp mối liên kết giữa Service và Pod).
- ServiceAccount controller: Tạo các ServiceAccount mặc định cho những namespace mới.

Danh sách trên chưa phải là đầy đủ.

### cloud-controller-manager

Một thành phần control plane của Kubernetes nhúng logic điều khiển đặc thù cho từng nền tảng cloud.
Cloud controller manager cho phép bạn kết nối cluster của mình với API của nhà cung cấp cloud (cloud provider),
đồng thời tách các thành phần tương tác với nền tảng cloud đó khỏi
các thành phần chỉ tương tác với cluster của bạn.

cloud-controller-manager chỉ chạy những controller đặc thù cho nhà cung cấp cloud của bạn.
Nếu bạn chạy Kubernetes trên hạ tầng của riêng mình (on-premises), hoặc trong môi trường học tập ngay trên
PC cá nhân, cluster sẽ không có cloud controller manager.

Tương tự kube-controller-manager, cloud-controller-manager kết hợp nhiều vòng lặp điều khiển (control loop)
độc lập về mặt logic vào một binary duy nhất mà bạn chạy như một tiến trình duy nhất. Bạn có thể mở rộng
theo chiều ngang (chạy nhiều hơn một bản sao) để cải thiện hiệu năng hoặc tăng khả năng chịu lỗi.

Các controller sau có thể có phụ thuộc vào nhà cung cấp cloud:

- Node controller: Kiểm tra với nhà cung cấp cloud để xác định xem một node đã bị
  xóa trên cloud hay chưa sau khi node đó ngừng phản hồi
- Route controller: Thiết lập các route trong hạ tầng cloud bên dưới
- Service controller: Tạo, cập nhật và xóa các bộ cân bằng tải (load balancer) của nhà cung cấp cloud

---

## Các thành phần của node (Node components) {#node-components}

Các thành phần node chạy trên mọi node, duy trì các pod đang chạy và cung cấp môi trường runtime của Kubernetes.

### kubelet

Một agent chạy trên mỗi node trong cluster. Nó đảm bảo rằng các container đang chạy trong một Pod.

[kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/) nhận một tập các PodSpec
được cung cấp qua nhiều cơ chế khác nhau và đảm bảo các container được mô tả trong những
PodSpec đó đang chạy và khỏe mạnh (healthy). kubelet không quản lý các container không do
Kubernetes tạo ra.

### kube-proxy (tùy chọn) {#kube-proxy}

kube-proxy là một network proxy chạy trên mỗi node trong cluster của bạn,
hiện thực một phần của khái niệm Service trong Kubernetes.

[kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)
duy trì các network rule trên các node. Những network rule này cho phép các phiên mạng (network session)
từ bên trong hoặc bên ngoài cluster giao tiếp với các Pod của bạn.

kube-proxy sử dụng tầng lọc gói tin (packet filtering) của hệ điều hành nếu tầng này tồn tại
và khả dụng. Nếu không, kube-proxy sẽ tự mình chuyển tiếp lưu lượng.

Nếu bạn dùng một [network plugin](#network-plugins) tự hiện thực việc chuyển tiếp gói tin cho các Service
và cung cấp hành vi tương đương kube-proxy, thì bạn không cần chạy
kube-proxy trên các node trong cluster của mình.

### Container runtime

Một thành phần nền tảng giúp Kubernetes chạy các container một cách hiệu quả.
Nó chịu trách nhiệm quản lý việc thực thi và vòng đời của các container trong môi trường Kubernetes.

Kubernetes hỗ trợ các container runtime như
containerd, CRI-O,
và bất kỳ hiện thực nào khác của [Kubernetes CRI (Container Runtime Interface)](https://github.com/kubernetes/community/blob/main/contributors/devel/sig-node/container-runtime-interface.md).

## Addons

Addons sử dụng các tài nguyên Kubernetes (DaemonSet,
Deployment, v.v.) để hiện thực các tính năng của cluster.
Vì chúng cung cấp các tính năng ở cấp cluster, những tài nguyên thuộc namespace của
addons nằm trong namespace `kube-system`.

Một số addons chọn lọc được mô tả bên dưới; để xem danh sách mở rộng các addons hiện có,
vui lòng xem [Addons](https://kubernetes.io/docs/concepts/cluster-administration/addons/).

### DNS

Mặc dù các addons khác không thực sự bắt buộc, mọi cluster Kubernetes đều nên có
[DNS cho cluster (cluster DNS)](./10-dns-pod-service-vi.md), vì nhiều ví dụ phụ thuộc vào nó.

Cluster DNS là một DNS server, hoạt động bổ sung cho các DNS server khác trong môi trường của bạn,
chuyên phục vụ các bản ghi DNS cho các service của Kubernetes.

Các container do Kubernetes khởi động sẽ tự động đưa DNS server này vào danh sách tìm kiếm DNS của chúng.

### Web UI (Dashboard)

[Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) là một giao diện web
đa dụng cho các cluster Kubernetes. Nó cho phép người dùng quản lý và khắc phục sự cố (troubleshoot)
cho các ứng dụng đang chạy trong cluster, cũng như cho chính cluster đó.

### Giám sát tài nguyên container (Container resource monitoring)

[Giám sát tài nguyên container](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-usage-monitoring/)
ghi lại các số liệu (metrics) chuỗi thời gian tổng quát về các container vào một cơ sở dữ liệu trung tâm, và cung cấp giao diện để duyệt dữ liệu đó.

### Ghi log cấp cluster (Cluster-level Logging)

Cơ chế [ghi log cấp cluster (cluster-level logging)](https://kubernetes.io/docs/concepts/cluster-administration/logging/) chịu trách nhiệm
lưu log của container vào một kho log trung tâm có giao diện tìm kiếm/duyệt.

### Network plugins {#network-plugins}

[Network plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins)
là các thành phần phần mềm hiện thực đặc tả container network interface (CNI).
Chúng chịu trách nhiệm cấp phát địa chỉ IP cho các pod và cho phép các pod
giao tiếp với nhau bên trong cluster.

## Các biến thể kiến trúc (Architecture variations)

Mặc dù các thành phần cốt lõi của Kubernetes luôn nhất quán, cách chúng được triển khai và
quản lý có thể khác nhau. Hiểu rõ các biến thể này là điều then chốt để thiết kế và vận hành
các cluster Kubernetes đáp ứng những nhu cầu vận hành cụ thể.

### Các phương án triển khai control plane (Control plane deployment options)

Các thành phần control plane có thể được triển khai theo một số cách:

**Triển khai truyền thống (Traditional deployment)**
: Các thành phần control plane chạy trực tiếp trên các máy hoặc VM chuyên dụng, thường được quản lý dưới dạng các service của systemd.

**Static Pod**
: Các thành phần control plane được triển khai dưới dạng static Pod, do kubelet trên các node cụ thể quản lý.
  Đây là cách tiếp cận phổ biến được các công cụ như kubeadm sử dụng.

**Tự lưu trữ (Self-hosted)**
: Control plane chạy dưới dạng các Pod ngay bên trong chính cluster Kubernetes, được quản lý bởi các Deployment
  và StatefulSet hoặc các nguyên thủy (primitive) khác của Kubernetes.

**Dịch vụ Kubernetes được quản lý (Managed Kubernetes services)**
: Các nhà cung cấp cloud thường trừu tượng hóa control plane, quản lý các thành phần của nó như một phần trong gói dịch vụ mà họ cung cấp.

### Cân nhắc về vị trí đặt workload (Workload placement considerations)

Vị trí đặt các workload, bao gồm cả các thành phần control plane, có thể thay đổi tùy theo quy mô cluster,
yêu cầu hiệu năng và chính sách vận hành:

- Trong các cluster nhỏ hoặc cluster phát triển (development), các thành phần control plane và workload của người dùng có thể chạy trên cùng các node.
- Các cluster production lớn thường dành riêng một số node cụ thể cho các thành phần control plane,
  tách biệt chúng khỏi workload của người dùng.
- Một số tổ chức chạy các add-on quan trọng hoặc các công cụ giám sát trên những node control plane.

### Công cụ quản lý cluster (Cluster management tools)

Các công cụ như kubeadm, kops và Kubespray đưa ra những cách tiếp cận khác nhau để triển khai và quản lý cluster,
mỗi công cụ có phương pháp bố trí và quản lý thành phần riêng.

### Tùy biến và khả năng mở rộng (Customization and extensibility)

Kiến trúc Kubernetes cho phép tùy biến ở mức độ đáng kể:

- Có thể triển khai các scheduler tùy chỉnh để hoạt động song song với scheduler mặc định của Kubernetes hoặc thay thế hoàn toàn nó.
- Có thể mở rộng API server bằng CustomResourceDefinition và API Aggregation.
- Các nhà cung cấp cloud có thể tích hợp sâu với Kubernetes thông qua cloud-controller-manager.

Tính linh hoạt của kiến trúc Kubernetes cho phép các tổ chức điều chỉnh cluster theo những nhu cầu cụ thể,
cân bằng các yếu tố như độ phức tạp vận hành, hiệu năng và chi phí quản lý.

## Tiếp theo (What's next)

Tìm hiểu thêm về các chủ đề sau:

- [Các Node](./23-nodes-vi.md) và
  [giao tiếp giữa chúng](./24-control-plane-node-communication-vi.md)
  với control plane.
- Các [controller](./25-controllers-vi.md) của Kubernetes.
- [Thu gom rác (Garbage collection)](https://kubernetes.io/docs/concepts/architecture/garbage-collection/) đối với các đối tượng của cluster.
- [kube-scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/) — scheduler mặc định của Kubernetes.
- [Tài liệu](https://etcd.io/docs/) chính thức của etcd.
- Một số [container runtime](./00-container-runtimes-vi.md) trong Kubernetes.
- Tích hợp với các nhà cung cấp cloud bằng [cloud-controller-manager](https://kubernetes.io/docs/concepts/architecture/cloud-controller/).
- Các lệnh [kubectl](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands).

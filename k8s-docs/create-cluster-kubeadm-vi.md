# Tạo một cluster với kubeadm (Creating a cluster with kubeadm)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/>
>
> Tài liệu gốc thuộc Kubernetes Documentation, phát hành theo giấy phép CC BY 4.0.

<img src="https://kubernetes.io/images/kubeadm-stacked-color.png" align="right" width="150px"></img>
Với `kubeadm`, bạn có thể tạo một Kubernetes cluster tối thiểu khả dụng (minimum viable) tuân theo các thực hành tốt nhất.
Trên thực tế, bạn có thể dùng `kubeadm` để dựng một cluster vượt qua được các
[bài kiểm tra tương thích Kubernetes (Kubernetes Conformance tests)](https://kubernetes.io/blog/2017/10/software-conformance-certification/).
`kubeadm` cũng hỗ trợ các chức năng khác trong vòng đời cluster, chẳng hạn như
[bootstrap token](https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/) và nâng cấp cluster.

Công cụ `kubeadm` phù hợp nếu bạn cần:

- Một cách đơn giản để bạn dùng thử Kubernetes, có thể là lần đầu tiên.
- Một cách để những người dùng hiện tại tự động hóa việc dựng cluster và kiểm thử ứng dụng của họ.
- Một khối dựng (building block) trong các công cụ hệ sinh thái và/hoặc công cụ cài đặt khác
  với phạm vi lớn hơn.

Bạn có thể cài đặt và sử dụng `kubeadm` trên nhiều loại máy: laptop của bạn, một nhóm
máy chủ đám mây (cloud server), một chiếc Raspberry Pi, v.v. Dù bạn triển khai lên
đám mây hay tại chỗ (on-premises), bạn đều có thể tích hợp `kubeadm` vào các hệ thống
cung cấp hạ tầng (provisioning) như Ansible hoặc Terraform.

## Trước khi bạn bắt đầu (Before you begin)

Để làm theo hướng dẫn này, bạn cần:

- Một hoặc nhiều máy chạy hệ điều hành Linux tương thích deb/rpm; ví dụ: Ubuntu hoặc CentOS.
- Mỗi máy có từ 2 GiB RAM trở lên — ít hơn sẽ không còn nhiều chỗ cho các ứng dụng của bạn.
- Ít nhất 2 CPU trên máy mà bạn dùng làm control-plane node.
- Kết nối mạng đầy đủ giữa tất cả các máy trong cluster. Bạn có thể dùng mạng
  công cộng (public) hoặc mạng riêng (private).

Bạn cũng cần dùng một phiên bản `kubeadm` có khả năng triển khai phiên bản
Kubernetes mà bạn muốn sử dụng trong cluster mới của mình.

[Chính sách hỗ trợ phiên bản và chênh lệch phiên bản của Kubernetes](https://kubernetes.io/docs/setup/release/version-skew-policy/#supported-versions)
áp dụng cho `kubeadm` cũng như cho toàn bộ Kubernetes.
Hãy xem chính sách đó để biết những phiên bản Kubernetes và `kubeadm` nào
được hỗ trợ. Trang này được viết cho Kubernetes v1.36.

Trạng thái tính năng tổng thể của công cụ `kubeadm` là Sẵn sàng chung (General Availability - GA).
Một số tính năng con vẫn đang được phát triển tích cực. Cách hiện thực việc tạo cluster
có thể thay đổi đôi chút khi công cụ tiến hóa, nhưng cách hiện thực tổng thể sẽ khá ổn định.

> **Ghi chú:**
> Mọi lệnh thuộc `kubeadm alpha`, theo định nghĩa, chỉ được hỗ trợ ở mức alpha.

## Mục tiêu (Objectives)

* Cài đặt một Kubernetes cluster với một control-plane duy nhất
* Cài đặt một Pod network trên cluster để các Pod của bạn có thể
  giao tiếp với nhau

## Hướng dẫn (Instructions)

### Chuẩn bị các máy chủ (Preparing the hosts)

#### Cài đặt thành phần (Component installation)

Cài đặt container runtime và kubeadm trên tất cả các máy chủ.
Để có hướng dẫn chi tiết và các điều kiện tiên quyết khác, xem
[Cài đặt kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/).

> **Ghi chú:**
> Nếu bạn đã cài đặt kubeadm rồi, hãy xem hai bước đầu tiên của tài liệu
> [Nâng cấp các Linux node](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-linux-nodes)
> để biết cách nâng cấp kubeadm.
>
> Khi bạn nâng cấp, kubelet sẽ khởi động lại sau mỗi vài giây vì nó chờ trong một vòng lặp lỗi
> (crashloop) để kubeadm chỉ cho nó biết phải làm gì. Vòng lặp lỗi này là điều bình thường
> và được mong đợi. Sau khi bạn khởi tạo control-plane, kubelet sẽ chạy bình thường.

#### Thiết lập mạng (Network setup)

kubeadm, tương tự như các thành phần Kubernetes khác, cố gắng tìm một địa chỉ IP khả dụng
trên các giao diện mạng (network interface) gắn với default gateway trên máy chủ. Địa chỉ
IP đó sau đó được dùng cho việc quảng bá (advertise) và/hoặc lắng nghe do một thành phần thực hiện.

Để biết địa chỉ IP này trên một máy Linux, bạn có thể dùng:

```shell
ip route show # Tìm dòng bắt đầu bằng "default via"
```

> **Ghi chú:**
> Nếu trên máy chủ có từ hai default gateway trở lên, một thành phần Kubernetes sẽ
> cố gắng dùng gateway đầu tiên mà nó gặp có địa chỉ global unicast IP phù hợp.
> Khi thực hiện lựa chọn này, thứ tự chính xác của các gateway có thể khác nhau
> giữa các hệ điều hành và phiên bản kernel khác nhau.

Các thành phần Kubernetes không chấp nhận tùy chọn chỉ định giao diện mạng tùy biến,
do đó phải truyền một địa chỉ IP tùy biến dưới dạng cờ (flag) cho tất cả các thể hiện (instance)
của thành phần cần cấu hình tùy biến như vậy.

> **Ghi chú:**
> Nếu máy chủ không có default gateway và không truyền địa chỉ IP tùy biến
> cho một thành phần Kubernetes, thành phần đó có thể thoát ra kèm theo lỗi.

Để cấu hình địa chỉ quảng bá (advertise address) của API server cho các control plane node
được tạo bằng cả `init` và `join`, có thể dùng cờ `--apiserver-advertise-address`.
Tốt hơn, tùy chọn này có thể được đặt trong [kubeadm API](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4)
dưới dạng `InitConfiguration.localAPIEndpoint` và `JoinConfiguration.controlPlane.localAPIEndpoint`.

Với kubelet trên tất cả các node, tùy chọn `--node-ip` có thể được truyền trong
`.nodeRegistration.kubeletExtraArgs` bên trong file cấu hình kubeadm
(`InitConfiguration` hoặc `JoinConfiguration`).

Với dual-stack, xem
[Hỗ trợ dual-stack với kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/dual-stack-support).

Các địa chỉ IP mà bạn gán cho các thành phần control plane sẽ trở thành một phần trong
các trường subject alternative name của certificate X.509 của chúng. Việc thay đổi các
địa chỉ IP này sẽ đòi hỏi phải ký các certificate mới và khởi động lại các thành phần
bị ảnh hưởng, để thay đổi trong các file certificate được phản ánh. Xem
[Gia hạn certificate thủ công](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#manual-certificate-renewal)
để biết thêm chi tiết về chủ đề này.

> **Cảnh báo:**
> Dự án Kubernetes khuyến nghị không dùng cách tiếp cận này (cấu hình tất cả các thể hiện
> của thành phần với địa chỉ IP tùy biến). Thay vào đó, những người bảo trì Kubernetes
> khuyến nghị thiết lập mạng của máy chủ sao cho địa chỉ IP của default gateway chính là
> địa chỉ mà các thành phần Kubernetes tự động phát hiện và sử dụng.
> Trên các Linux node, bạn có thể dùng các lệnh như `ip route` để cấu hình mạng; hệ điều hành
> của bạn cũng có thể cung cấp các công cụ quản lý mạng ở cấp cao hơn. Nếu default gateway
> của node là một địa chỉ IP công cộng, bạn nên cấu hình lọc gói tin (packet filtering)
> hoặc các biện pháp bảo mật khác để bảo vệ các node và cluster của bạn.

### Chuẩn bị các container image cần thiết (Preparing the required container images)

Bước này là tùy chọn và chỉ áp dụng trong trường hợp bạn muốn `kubeadm init` và `kubeadm join`
không tải về các container image mặc định vốn được lưu trữ tại `registry.k8s.io`.

Kubeadm có các lệnh giúp bạn kéo (pull) trước các image cần thiết
khi tạo một cluster mà các node của nó không có kết nối internet.
Xem [Chạy kubeadm khi không có kết nối internet](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init#without-internet-connection)
để biết thêm chi tiết.

Kubeadm cho phép bạn dùng một kho image (image repository) tùy biến cho các image cần thiết.
Xem [Sử dụng image tùy biến](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init#custom-images)
để biết thêm chi tiết.

### Khởi tạo control-plane node (Initializing your control-plane node)

Control-plane node là máy nơi các thành phần control plane chạy, bao gồm
etcd (cơ sở dữ liệu của cluster) và API Server
(nơi công cụ dòng lệnh kubectl giao tiếp).

1. (Khuyến nghị) Nếu bạn có kế hoạch nâng cấp cluster `kubeadm` một control-plane này
   lên [tính sẵn sàng cao (high availability)](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/),
   bạn nên chỉ định `--control-plane-endpoint` để đặt endpoint dùng chung cho tất cả các control-plane node.
   Endpoint như vậy có thể là một tên DNS hoặc một địa chỉ IP của load-balancer.
1. Chọn một Pod network add-on, và kiểm tra xem nó có yêu cầu tham số nào cần
   truyền cho `kubeadm init` hay không. Tùy vào nhà cung cấp
   bên thứ ba mà bạn chọn, bạn có thể cần đặt `--pod-network-cidr` thành
   một giá trị đặc thù của nhà cung cấp đó. Xem [Cài đặt Pod network add-on](#pod-network).
1. (Tùy chọn) `kubeadm` cố gắng phát hiện container runtime bằng một danh sách các
   endpoint phổ biến. Để dùng một container runtime khác hoặc nếu có nhiều hơn một runtime
   được cài trên node đã chuẩn bị, hãy chỉ định tham số `--cri-socket` cho `kubeadm`. Xem
   [Cài đặt runtime](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-runtime).

Để khởi tạo control-plane node, chạy:

```bash
kubeadm init <args>
```

### Những lưu ý về apiserver-advertise-address và ControlPlaneEndpoint (Considerations about apiserver-advertise-address and ControlPlaneEndpoint)

Trong khi `--apiserver-advertise-address` có thể được dùng để đặt địa chỉ quảng bá cho API server
của riêng control-plane node này, `--control-plane-endpoint` có thể được dùng để đặt endpoint
dùng chung cho tất cả các control-plane node.

`--control-plane-endpoint` chấp nhận cả địa chỉ IP lẫn tên DNS có thể ánh xạ tới địa chỉ IP.
Hãy liên hệ với quản trị viên mạng của bạn để đánh giá các giải pháp khả thi cho việc ánh xạ như vậy.

Đây là một ví dụ ánh xạ:

```
192.168.0.102 cluster-endpoint
```

Trong đó `192.168.0.102` là địa chỉ IP của node này và `cluster-endpoint` là một tên DNS tùy biến ánh xạ tới IP đó.
Điều này cho phép bạn truyền `--control-plane-endpoint=cluster-endpoint` cho `kubeadm init` và truyền cùng tên DNS đó cho
`kubeadm join`. Sau này bạn có thể sửa `cluster-endpoint` để trỏ tới địa chỉ của load-balancer trong
kịch bản tính sẵn sàng cao (high availability).

Việc chuyển một cluster với một control plane duy nhất được tạo mà không có `--control-plane-endpoint`
thành một cluster có tính sẵn sàng cao không được kubeadm hỗ trợ.

### Thông tin thêm (More information)

Để biết thêm thông tin về các tham số của `kubeadm init`, xem [tài liệu tham khảo kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/).

Để cấu hình `kubeadm init` bằng một file cấu hình, xem
[Sử dụng kubeadm init với file cấu hình](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/#config-file).

Để tùy biến các thành phần control plane, bao gồm việc gán tùy chọn địa chỉ IPv6 cho liveness probe
của các thành phần control plane và etcd server, hãy cung cấp thêm tham số cho từng thành phần như được mô tả trong
[tham số tùy biến](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/control-plane-flags/).

Để cấu hình lại một cluster đã được tạo từ trước, xem
[Cấu hình lại một kubeadm cluster](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-reconfigure).

Để chạy lại `kubeadm init`, trước tiên bạn phải [gỡ bỏ cluster](#tear-down).

Nếu bạn join một node có kiến trúc khác vào cluster, hãy đảm bảo các DaemonSet đã triển khai
có container image hỗ trợ kiến trúc đó.

`kubeadm init` trước tiên chạy một loạt các kiểm tra sơ bộ (precheck) để đảm bảo máy
đã sẵn sàng chạy Kubernetes. Các kiểm tra sơ bộ này hiển thị cảnh báo và thoát khi có lỗi. Sau đó `kubeadm init`
tải về và cài đặt các thành phần control plane của cluster. Việc này có thể mất vài phút.
Sau khi hoàn tất, bạn sẽ thấy:

```none
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  /docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

Để kubectl hoạt động cho người dùng không phải root của bạn, hãy chạy các lệnh sau,
những lệnh này cũng nằm trong output của `kubeadm init`:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Hoặc, nếu bạn là người dùng `root`, bạn có thể chạy:

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

> **Cảnh báo:**
> File kubeconfig `admin.conf` mà `kubeadm init` sinh ra chứa một certificate với
> `Subject: O = kubeadm:cluster-admins, CN = kubernetes-admin`. Nhóm `kubeadm:cluster-admins`
> được gắn (bind) với ClusterRole dựng sẵn `cluster-admin`.
> Không chia sẻ file `admin.conf` với bất kỳ ai.
>
> `kubeadm init` sinh ra một file kubeconfig khác là `super-admin.conf` chứa một certificate với
> `Subject: O = system:masters, CN = kubernetes-super-admin`.
> `system:masters` là một nhóm siêu người dùng dùng trong trường hợp khẩn cấp (break-glass),
> bỏ qua lớp phân quyền (ví dụ RBAC).
> Không chia sẻ file `super-admin.conf` với bất kỳ ai. Khuyến nghị chuyển file này đến một nơi lưu trữ an toàn.
>
> Xem
> [Sinh file kubeconfig cho người dùng bổ sung](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs#kubeconfig-additional-users)
> để biết cách dùng `kubeadm kubeconfig user` sinh file kubeconfig cho những người dùng bổ sung.

Hãy ghi lại lệnh `kubeadm join` mà `kubeadm init` xuất ra. Bạn
cần lệnh này để [join các node vào cluster của bạn](#join-nodes).

Token được dùng để xác thực lẫn nhau (mutual authentication) giữa control-plane node và các node
tham gia (join). Token có trong output là bí mật. Hãy giữ nó an toàn, vì bất kỳ ai có
token này đều có thể thêm các node đã được xác thực vào cluster của bạn. Các token này có thể được
liệt kê, tạo và xóa bằng lệnh `kubeadm token`. Xem
[tài liệu tham khảo kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/).

### Cài đặt Pod network add-on (Installing a Pod network add-on) {#pod-network}

> **Thận trọng:**
> Mục này chứa thông tin quan trọng về thiết lập mạng và
> thứ tự triển khai.
> Hãy đọc kỹ toàn bộ lời khuyên này trước khi tiếp tục.
>
> **Bạn phải triển khai một Pod network add-on dựa trên
> Container Network Interface
> (CNI) để các Pod của bạn có thể giao tiếp với nhau.
> DNS của cluster (CoreDNS) sẽ không khởi động trước khi một mạng được cài đặt.**
>
> - Lưu ý rằng Pod network của bạn không được trùng lặp (overlap) với bất kỳ mạng nào của
>   máy chủ: bạn rất có thể sẽ gặp sự cố nếu có bất kỳ sự trùng lặp nào.
>   (Nếu bạn phát hiện xung đột giữa dải Pod network ưa thích của
>   network plugin với một số mạng của máy chủ, bạn nên nghĩ đến một khối CIDR
>   phù hợp để dùng thay thế, rồi dùng nó khi chạy `kubeadm init` với
>   `--pod-network-cidr`, đồng thời dùng nó thay thế trong YAML của network plugin).
>
> - Theo mặc định, `kubeadm` thiết lập cluster của bạn để dùng và bắt buộc dùng
>   [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) (điều khiển truy cập
>   dựa trên vai trò - role based access control).
>   Hãy đảm bảo Pod network plugin của bạn hỗ trợ RBAC, và mọi manifest
>   bạn dùng để triển khai nó cũng vậy.
>
> - Nếu bạn muốn dùng IPv6 — dual-stack, hoặc mạng single-stack chỉ dùng IPv6 —
>   cho cluster của mình, hãy đảm bảo Pod network plugin của bạn
>   hỗ trợ IPv6.
>   Hỗ trợ IPv6 đã được thêm vào CNI từ [v0.6.0](https://github.com/containernetworking/cni/releases/tag/v0.6.0).

> **Ghi chú:**
> Kubeadm được thiết kế để không phụ thuộc vào CNI cụ thể nào (CNI agnostic) và việc kiểm định
> các nhà cung cấp CNI nằm ngoài phạm vi kiểm thử e2e hiện tại của chúng tôi.
> Nếu bạn gặp vấn đề liên quan tới một CNI plugin, bạn nên tạo ticket trong trình theo dõi
> lỗi (issue tracker) của chính plugin đó thay vì trình theo dõi lỗi của kubeadm hay kubernetes.

Một số dự án bên ngoài cung cấp Kubernetes Pod network sử dụng CNI, trong đó một số cũng
hỗ trợ [Network Policy](https://kubernetes.io/docs/concepts/services-networking/network-policies/).

Xem danh sách các add-on hiện thực
[mô hình mạng Kubernetes](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-network-model).

Vui lòng tham khảo trang [Cài đặt Addons](https://kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy)
để có danh sách (chưa đầy đủ) các addon mạng được Kubernetes hỗ trợ.
Bạn có thể cài một Pod network add-on bằng lệnh sau trên
control-plane node hoặc một node có thông tin xác thực kubeconfig:

```bash
kubectl apply -f <add-on.yaml>
```

> **Ghi chú:**
> Chỉ một số ít CNI plugin hỗ trợ Windows. Thông tin chi tiết hơn và hướng dẫn thiết lập có tại
> [Thêm các Windows worker node](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/adding-windows-nodes/#network-config).

Bạn chỉ có thể cài một Pod network cho mỗi cluster.

Khi một Pod network đã được cài đặt, bạn có thể xác nhận nó hoạt động bằng cách
kiểm tra rằng Pod CoreDNS đang ở trạng thái `Running` trong output của `kubectl get pods --all-namespaces`.
Và khi Pod CoreDNS đã khởi động và chạy, bạn có thể tiếp tục bằng cách join các node của mình.

Nếu mạng của bạn không hoạt động hoặc CoreDNS không ở trạng thái `Running`, hãy xem
[hướng dẫn xử lý sự cố](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/)
cho `kubeadm`.

### Các label node được quản lý (Managed node labels)

Theo mặc định, kubeadm bật admission controller [NodeRestriction](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction),
giới hạn những label mà kubelet có thể tự gán cho chính nó khi đăng ký node.
Tài liệu về admission controller mô tả những label nào được phép dùng với tùy chọn
`--node-labels` của kubelet.

> **Thận trọng:**
> Do admission controller `NodeRestriction`, bạn **không thể** dùng cờ `--node-labels`
> của kubelet để gán các label bị giới hạn (chẳng hạn `node-role.kubernetes.io/*`) trong quá trình khởi tạo.
>
> Nếu bạn cố thêm các label bị giới hạn bằng cờ kubelet này, node sẽ không đăng ký được
> với API server.

Để gán các label này một cách thủ công, bạn phải dùng `kubectl label` sau khi node đã join vào cluster.
Hãy đảm bảo bạn đang dùng một kubeconfig có đặc quyền, chẳng hạn file `/etc/kubernetes/admin.conf` do kubeadm quản lý.

### Cách ly control plane node (Control plane node isolation)

Theo mặc định, cluster của bạn sẽ không lập lịch (schedule) Pod trên các control plane node
vì lý do bảo mật. Nếu bạn muốn có thể lập lịch Pod trên các control plane node,
ví dụ với một Kubernetes cluster chỉ gồm một máy duy nhất, hãy chạy:

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

Output sẽ trông giống như:

```
node "test-01" untainted
...
```

Lệnh này sẽ gỡ taint `node-role.kubernetes.io/control-plane:NoSchedule`
khỏi mọi node đang có nó, bao gồm cả các control plane node, nghĩa là
scheduler khi đó sẽ có thể lập lịch Pod ở mọi nơi.

Ngoài ra, bạn có thể thực thi lệnh sau để gỡ label
[`node.kubernetes.io/exclude-from-external-load-balancers`](https://kubernetes.io/docs/reference/labels-annotations-taints/#node-kubernetes-io-exclude-from-external-load-balancers)
khỏi control plane node, label này loại node đó khỏi danh sách các máy chủ backend:

```bash
kubectl label nodes --all node.kubernetes.io/exclude-from-external-load-balancers-
```

### Thêm control plane node (Adding more control plane nodes)

Xem [Tạo cluster có tính sẵn sàng cao với kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)
để biết các bước tạo một kubeadm cluster có tính sẵn sàng cao bằng cách thêm các control plane node.

### Thêm worker node (Adding worker nodes) {#join-nodes}

Worker node là nơi các workload của bạn chạy.

Các trang sau đây hướng dẫn cách thêm Linux worker node và Windows worker node vào cluster bằng
lệnh `kubeadm join`:

* [Thêm các Linux worker node](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/adding-linux-nodes/)
* [Thêm các Windows worker node](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/adding-windows-nodes/)

### (Tùy chọn) Điều khiển cluster từ các máy khác ngoài control-plane node ((Optional) Controlling your cluster from machines other than the control-plane node)

Để kubectl trên một máy tính khác (ví dụ laptop) có thể nói chuyện với
cluster của bạn, bạn cần sao chép file kubeconfig của quản trị viên từ control-plane node
về máy làm việc của bạn như sau:

```bash
scp root@<control-plane-host>:/etc/kubernetes/admin.conf .
kubectl --kubeconfig ./admin.conf get nodes
```

> **Ghi chú:**
> Ví dụ trên giả định rằng truy cập SSH cho root đã được bật. Nếu không phải
> vậy, bạn có thể sao chép file `admin.conf` sao cho một người dùng khác truy cập được
> và dùng `scp` với người dùng đó thay thế.
>
> File `admin.conf` trao cho người dùng đặc quyền _siêu người dùng_ (superuser) trên cluster.
> File này nên được dùng một cách hạn chế. Với người dùng thông thường, khuyến nghị
> sinh một thông tin xác thực (credential) riêng biệt mà bạn cấp quyền cho nó. Bạn có thể làm
> điều này với lệnh `kubeadm kubeconfig user --client-name <CN>`.
> Lệnh đó sẽ in một file KubeConfig ra STDOUT mà bạn
> nên lưu vào một file và phân phối cho người dùng của mình. Sau đó, cấp
> quyền bằng cách dùng `kubectl create (cluster)rolebinding`.

### (Tùy chọn) Proxy API Server về localhost ((Optional) Proxying API Server to localhost)

Nếu bạn muốn kết nối tới API Server từ bên ngoài cluster, bạn có thể dùng
`kubectl proxy`:

```bash
scp root@<control-plane-host>:/etc/kubernetes/admin.conf .
kubectl --kubeconfig ./admin.conf proxy
```

Giờ bạn có thể truy cập API Server cục bộ tại `http://localhost:8001/api/v1`

## Dọn dẹp (Clean up) {#tear-down}

Nếu bạn dùng các máy chủ dùng một lần (disposable) cho cluster của mình, cho mục đích thử nghiệm, bạn có thể
tắt chúng đi và không cần dọn dẹp gì thêm. Bạn có thể dùng
`kubectl config delete-cluster` để xóa các tham chiếu cục bộ tới
cluster.

Tuy nhiên, nếu bạn muốn thu hồi (deprovision) cluster của mình một cách gọn gàng hơn, trước tiên bạn nên
[drain node](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#drain)
và đảm bảo rằng node đã trống, sau đó gỡ cấu hình node.

### Gỡ bỏ node (Remove the node)

Kết nối tới control-plane node với thông tin xác thực phù hợp, chạy:

```bash
kubectl drain <node name> --delete-emptydir-data --force --ignore-daemonsets
```

Trước khi gỡ bỏ node, hãy đặt lại (reset) trạng thái đã được `kubeadm` cài đặt:

```bash
kubeadm reset
```

Quá trình reset không đặt lại hay dọn dẹp các quy tắc iptables hoặc các bảng IPVS.
Nếu bạn muốn đặt lại iptables, bạn phải làm thủ công:

```bash
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

Nếu bạn muốn đặt lại các bảng IPVS, bạn phải chạy lệnh sau:

```bash
ipvsadm -C
```

Bây giờ gỡ bỏ node:

```bash
kubectl delete node <node name>
```

Nếu bạn muốn bắt đầu lại từ đầu, hãy chạy `kubeadm init` hoặc `kubeadm join` với
các tham số phù hợp.

### Dọn dẹp control plane (Clean up the control plane)

Bạn có thể dùng `kubeadm reset` trên máy chủ control plane để kích hoạt việc dọn dẹp
ở mức nỗ lực tốt nhất có thể (best-effort).

Xem tài liệu tham khảo [`kubeadm reset`](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/)
để biết thêm thông tin về lệnh con này và các tùy chọn
của nó.

## Chính sách chênh lệch phiên bản (Version skew policy) {#version-skew-policy}

Mặc dù kubeadm cho phép chênh lệch phiên bản (version skew) với một số thành phần mà nó quản lý, khuyến nghị là bạn nên
dùng phiên bản kubeadm khớp với phiên bản của các thành phần control plane, kube-proxy và kubelet.

### Chênh lệch phiên bản giữa kubeadm và Kubernetes (kubeadm's skew against the Kubernetes version)

kubeadm có thể được dùng với các thành phần Kubernetes có cùng phiên bản với kubeadm
hoặc cũ hơn một phiên bản. Phiên bản Kubernetes có thể được chỉ định cho kubeadm bằng
cờ `--kubernetes-version` của `kubeadm init` hoặc trường
[`ClusterConfiguration.kubernetesVersion`](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/)
khi dùng `--config`. Tùy chọn này sẽ điều khiển phiên bản
của kube-apiserver, kube-controller-manager, kube-scheduler và kube-proxy.

Ví dụ:

* kubeadm ở phiên bản v1.36
* `kubernetesVersion` phải ở phiên bản v1.36 hoặc v1.35

### Chênh lệch phiên bản giữa kubeadm và kubelet (kubeadm's skew against the kubelet)

Tương tự như với phiên bản Kubernetes, kubeadm có thể được dùng với phiên bản kubelet
bằng với phiên bản của kubeadm hoặc cũ hơn tối đa ba phiên bản.

Ví dụ:

* kubeadm ở phiên bản v1.36
* kubelet trên máy phải ở phiên bản v1.36, v1.35,
  v1.34 hoặc v1.33

### Chênh lệch phiên bản giữa kubeadm và kubeadm (kubeadm's skew against kubeadm)

Có một số hạn chế về cách các lệnh kubeadm có thể thao tác trên các node hiện có hoặc toàn bộ cluster
do kubeadm quản lý.

Nếu có node mới được join vào cluster, binary kubeadm dùng cho `kubeadm join` phải khớp
với phiên bản kubeadm cuối cùng đã được dùng để tạo cluster bằng `kubeadm init` hoặc để nâng cấp
chính node đó bằng `kubeadm upgrade`. Các quy tắc tương tự áp dụng cho các lệnh kubeadm còn lại,
ngoại trừ `kubeadm upgrade`.

Ví dụ cho `kubeadm join`:

* kubeadm phiên bản v1.36 đã được dùng để tạo cluster bằng `kubeadm init`
* Các node join phải dùng binary kubeadm ở phiên bản v1.36

Các node đang được nâng cấp phải dùng một phiên bản kubeadm có cùng phiên bản MINOR
hoặc mới hơn một phiên bản MINOR so với phiên bản kubeadm đã được dùng để quản lý
node đó.

Ví dụ cho `kubeadm upgrade`:

* kubeadm phiên bản v1.35 đã được dùng để tạo hoặc nâng cấp node
* Phiên bản kubeadm dùng để nâng cấp node phải ở phiên bản v1.35
  hoặc v1.36

Để tìm hiểu thêm về chênh lệch phiên bản giữa các thành phần Kubernetes khác nhau, xem
[Chính sách chênh lệch phiên bản (Version Skew Policy)](https://kubernetes.io/releases/version-skew-policy/).

## Hạn chế (Limitations) {#limitations}

### Khả năng chống chịu của cluster (Cluster resilience) {#resilience}

Cluster được tạo ở đây có một control-plane node duy nhất, với một cơ sở dữ liệu etcd duy nhất
chạy trên đó. Điều này nghĩa là nếu control-plane node gặp sự cố, cluster của bạn có thể mất
dữ liệu và có thể phải được tạo lại từ đầu.

Các giải pháp khắc phục:

* Thường xuyên [sao lưu etcd](https://etcd.io/docs/v3.5/op-guide/recovery/). Thư mục
  dữ liệu etcd do kubeadm cấu hình nằm tại `/var/lib/etcd` trên control-plane node.

* Dùng nhiều control-plane node. Bạn có thể đọc
  [Các lựa chọn cho topology tính sẵn sàng cao](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/) để chọn một topology
  cluster cung cấp [tính sẵn sàng cao](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/).

### Tương thích nền tảng (Platform compatibility) {#multi-platform}

Các gói deb/rpm và binary của kubeadm được build cho amd64, arm (32-bit), arm64, ppc64le và s390x
theo [đề xuất đa nền tảng](https://git.k8s.io/design-proposals-archive/multi-platform.md).

Các container image đa nền tảng cho control plane và các addon cũng được hỗ trợ kể từ v1.12.

Chỉ một số nhà cung cấp giải pháp mạng cung cấp giải pháp cho tất cả các nền tảng. Vui lòng tham khảo danh sách
các nhà cung cấp mạng ở trên hoặc tài liệu của từng nhà cung cấp để biết liệu nhà cung cấp đó
có hỗ trợ nền tảng bạn chọn hay không.

## Xử lý sự cố (Troubleshooting) {#troubleshooting}

Nếu bạn đang gặp khó khăn với kubeadm, vui lòng tham khảo
[tài liệu xử lý sự cố](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/) của chúng tôi.

## Tiếp theo (What's next)

* Kiểm tra rằng cluster của bạn đang chạy đúng với [Sonobuoy](https://github.com/heptio/sonobuoy)
* <a id="lifecycle" />Xem [Nâng cấp các kubeadm cluster](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
  để biết chi tiết về việc nâng cấp cluster của bạn bằng `kubeadm`.
* Tìm hiểu cách sử dụng `kubeadm` nâng cao trong [tài liệu tham khảo kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/)
* Tìm hiểu thêm về các [khái niệm](https://kubernetes.io/docs/concepts/) Kubernetes và [`kubectl`](https://kubernetes.io/docs/reference/kubectl/).
* Xem trang [Mạng của cluster (Cluster Networking)](https://kubernetes.io/docs/concepts/cluster-administration/networking/) để có danh sách
  đầy đủ hơn các Pod network add-on.
* <a id="other-addons" />Xem [danh sách các add-on](https://kubernetes.io/docs/concepts/cluster-administration/addons/) để
  khám phá các add-on khác, bao gồm các công cụ ghi log (logging), giám sát (monitoring), network policy, trực quan hóa và
  điều khiển Kubernetes cluster của bạn.
* Cấu hình cách cluster của bạn xử lý log cho các sự kiện của cluster và từ
  các ứng dụng chạy trong Pod.
  Xem [Kiến trúc ghi log (Logging Architecture)](https://kubernetes.io/docs/concepts/cluster-administration/logging/) để có
  cái nhìn tổng quan về những gì liên quan.

### Phản hồi (Feedback) {#feedback}

* Với lỗi (bug), truy cập [trình theo dõi lỗi của kubeadm trên GitHub](https://github.com/kubernetes/kubeadm/issues)
* Để được hỗ trợ, truy cập kênh Slack
  [#kubeadm](https://kubernetes.slack.com/messages/kubeadm/)
* Kênh Slack chung về phát triển của SIG Cluster Lifecycle:
  [#sig-cluster-lifecycle](https://kubernetes.slack.com/messages/sig-cluster-lifecycle/)
* [Thông tin SIG](https://github.com/kubernetes/community/tree/main/sig-cluster-lifecycle#readme) của SIG Cluster Lifecycle
* Danh sách thư (mailing list) của SIG Cluster Lifecycle:
  [kubernetes-sig-cluster-lifecycle](https://groups.google.com/forum/#!forum/kubernetes-sig-cluster-lifecycle)

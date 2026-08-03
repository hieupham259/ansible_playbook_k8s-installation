# Tạo cluster có tính sẵn sàng cao với kubeadm (Creating Highly Available Clusters with kubeadm)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/>
>
> (Trang gốc không có phần description trong frontmatter)

Trang này giải thích hai cách tiếp cận khác nhau để thiết lập một cluster Kubernetes có tính sẵn sàng cao (highly available) bằng kubeadm:

- Với các node control plane xếp chồng (stacked). Cách tiếp cận này đòi hỏi ít hạ tầng hơn. Các thành viên etcd và các node control plane được đặt cùng nhau (co-located).
- Với một cluster etcd bên ngoài (external). Cách tiếp cận này đòi hỏi nhiều hạ tầng hơn. Các node control plane và các thành viên etcd được tách riêng.

Trước khi tiếp tục, bạn nên cân nhắc kỹ xem cách tiếp cận nào đáp ứng tốt nhất nhu cầu của ứng dụng và môi trường của bạn. Trang [Các lựa chọn cho topology có tính sẵn sàng cao](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/) trình bày ưu điểm và nhược điểm của từng cách.

Nếu bạn gặp vấn đề khi thiết lập cluster HA, vui lòng báo cáo trong [trình theo dõi issue](https://github.com/kubernetes/kubeadm/issues/new) của kubeadm.

Xem thêm [tài liệu về nâng cấp](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/).

> **Thận trọng:** Trang này không đề cập đến việc chạy cluster của bạn trên một nhà cung cấp cloud. Trong môi trường cloud, cả hai cách tiếp cận được mô tả ở đây đều không hoạt động với các đối tượng Service loại LoadBalancer, hoặc với PersistentVolume động (dynamic PersistentVolumes).

## Trước khi bạn bắt đầu (Before you begin)

Các điều kiện tiên quyết phụ thuộc vào topology mà bạn đã chọn cho control plane của cluster:

#### Stacked etcd

Bạn cần:

- Ba máy trở lên đáp ứng [các yêu cầu tối thiểu của kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin) cho các node control plane. Việc có số lẻ node control plane có thể giúp ích cho việc bầu chọn leader (leader selection) trong trường hợp máy hoặc zone gặp sự cố.
  - bao gồm một container runtime đã được thiết lập và hoạt động
- Ba máy trở lên đáp ứng [các yêu cầu tối thiểu của kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin) cho các worker
  - bao gồm một container runtime đã được thiết lập và hoạt động
- Kết nối mạng đầy đủ giữa tất cả các máy trong cluster (mạng công cộng hoặc mạng riêng)
- Quyền superuser trên tất cả các máy thông qua `sudo`
  - Bạn có thể dùng một công cụ khác; hướng dẫn này dùng `sudo` trong các ví dụ.
- Truy cập SSH từ một thiết bị đến tất cả các node trong hệ thống
- `kubeadm` và `kubelet` đã được cài đặt trên tất cả các máy.

_Xem [Topology stacked etcd](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/#stacked-etcd-topology) để hiểu thêm ngữ cảnh._

#### External etcd

Bạn cần:

- Ba máy trở lên đáp ứng [các yêu cầu tối thiểu của kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin) cho các node control plane. Việc có số lẻ node control plane có thể giúp ích cho việc bầu chọn leader (leader selection) trong trường hợp máy hoặc zone gặp sự cố.
  - bao gồm một container runtime đã được thiết lập và hoạt động
- Ba máy trở lên đáp ứng [các yêu cầu tối thiểu của kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin) cho các worker
  - bao gồm một container runtime đã được thiết lập và hoạt động
- Kết nối mạng đầy đủ giữa tất cả các máy trong cluster (mạng công cộng hoặc mạng riêng)
- Quyền superuser trên tất cả các máy thông qua `sudo`
  - Bạn có thể dùng một công cụ khác; hướng dẫn này dùng `sudo` trong các ví dụ.
- Truy cập SSH từ một thiết bị đến tất cả các node trong hệ thống
- `kubeadm` và `kubelet` đã được cài đặt trên tất cả các máy.

Và bạn còn cần thêm:

- Ba máy bổ sung trở lên, sẽ trở thành các thành viên của cluster etcd.
  Việc có số lẻ thành viên trong cluster etcd là một yêu cầu để đạt được
  quorum bỏ phiếu (voting quorum) tối ưu.
  - Các máy này cũng cần được cài đặt `kubeadm` và `kubelet`.
  - Các máy này cũng cần một container runtime đã được thiết lập và hoạt động.

_Xem [Topology external etcd](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/#external-etcd-topology) để hiểu thêm ngữ cảnh._

### Các container image (Container images)

Mỗi host cần có quyền truy cập để đọc và lấy các image từ container image registry của Kubernetes,
`registry.k8s.io`. Nếu bạn muốn triển khai một cluster có tính sẵn sàng cao mà các host không có
quyền truy cập để pull image, điều này vẫn khả thi. Bạn phải đảm bảo bằng một cách khác rằng các
container image cần thiết đã có sẵn trên các host liên quan.

### Giao diện dòng lệnh (Command line interface) {#kubectl}

Để quản lý Kubernetes sau khi cluster của bạn được thiết lập, bạn nên
[cài đặt kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) trên PC của mình. Việc cài đặt
công cụ `kubectl` trên mỗi node control plane cũng hữu ích, vì nó có thể
giúp ích cho việc xử lý sự cố (troubleshooting).

## Các bước đầu tiên cho cả hai phương pháp (First steps for both methods)

### Tạo bộ cân bằng tải cho kube-apiserver (Create load balancer for kube-apiserver)

> **Ghi chú:** Có rất nhiều cấu hình cho bộ cân bằng tải (load balancer). Ví dụ sau đây chỉ là một
> lựa chọn. Yêu cầu của cluster của bạn có thể cần một cấu hình khác.

1. Tạo một load balancer cho kube-apiserver với tên phân giải được qua DNS.

   - Trong môi trường cloud, bạn nên đặt các node control plane phía sau một load balancer
     chuyển tiếp TCP (TCP forwarding). Load balancer này phân phối lưu lượng đến tất cả
     các node control plane khỏe mạnh trong danh sách đích (target list) của nó. Health check
     cho một apiserver là một kiểm tra TCP trên cổng mà kube-apiserver lắng nghe
     (giá trị mặc định `:6443`).

   - Không khuyến nghị sử dụng trực tiếp địa chỉ IP trong môi trường cloud.

   - Load balancer phải có khả năng giao tiếp với tất cả các node control plane
     trên cổng của apiserver. Nó cũng phải cho phép lưu lượng đến trên cổng
     lắng nghe của nó.

   - Đảm bảo địa chỉ của load balancer luôn khớp với
     địa chỉ `ControlPlaneEndpoint` của kubeadm.

   - Đọc hướng dẫn [Các lựa chọn cho cân bằng tải bằng phần mềm](https://git.k8s.io/kubeadm/docs/ha-considerations.md#options-for-software-load-balancing)
     để biết thêm chi tiết.

1. Thêm node control plane đầu tiên vào load balancer, và kiểm tra
   kết nối:

   ```shell
   nc -zv -w 2 <LOAD_BALANCER_IP> <PORT>
   ```

   Lỗi connection refused là điều được mong đợi vì API server chưa
   chạy. Tuy nhiên, timeout có nghĩa là load balancer không thể giao tiếp
   với node control plane. Nếu xảy ra timeout, hãy cấu hình lại load
   balancer để giao tiếp được với node control plane.

1. Thêm các node control plane còn lại vào target group của load balancer.

## Các node control plane và etcd xếp chồng (Stacked control plane and etcd nodes)

### Các bước cho node control plane đầu tiên (Steps for the first control plane node)

1. Khởi tạo control plane:

   ```sh
   sudo kubeadm init --control-plane-endpoint "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" --upload-certs
   ```

   - Bạn có thể dùng cờ `--kubernetes-version` để đặt phiên bản Kubernetes muốn sử dụng.
     Khuyến nghị rằng phiên bản của kubeadm, kubelet, kubectl và Kubernetes nên khớp nhau.
   - Cờ `--control-plane-endpoint` nên được đặt thành địa chỉ hoặc DNS và cổng của load balancer.

   - Cờ `--upload-certs` được dùng để tải lên (upload) các certificate cần được chia sẻ
     giữa tất cả các instance control-plane trong cluster. Nếu thay vào đó, bạn muốn tự sao chép
     certificate giữa các node control-plane theo cách thủ công hoặc bằng công cụ tự động hóa, hãy bỏ cờ này
     và tham khảo phần [Phân phối certificate thủ công](#manual-certs) bên dưới.

   > **Ghi chú:** Các cờ `--config` và `--certificate-key` của `kubeadm init` không thể dùng chung với nhau,
   > do đó nếu bạn muốn dùng [cấu hình kubeadm](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/)
   > bạn phải thêm trường `certificateKey` vào các vị trí cấu hình phù hợp
   > (dưới `InitConfiguration` và `JoinConfiguration: controlPlane`).

   > **Ghi chú:** Một số plugin mạng CNI yêu cầu cấu hình bổ sung, ví dụ như chỉ định pod IP CIDR, trong khi một số khác thì không.
   > Xem [tài liệu về mạng CNI](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network).
   > Để thêm một pod CIDR, hãy truyền cờ `--pod-network-cidr`, hoặc nếu bạn đang dùng file cấu hình kubeadm
   > thì đặt trường `podSubnet` dưới đối tượng `networking` của `ClusterConfiguration`.

   Output trông tương tự như sau:

   ```sh
   ...
   You can now join any number of control-plane node by running the following command on each as a root:
       kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 --control-plane --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07

   Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
   As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use kubeadm init phase upload-certs to reload certs afterward.

   Then you can join any number of worker nodes by running the following on each as root:
       kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866
   ```

   - Sao chép output này vào một file văn bản. Bạn sẽ cần nó sau này để join các node control plane
     và worker vào cluster.
   - Khi `--upload-certs` được dùng cùng `kubeadm init`, các certificate của control plane chính
     được mã hóa và tải lên trong Secret `kubeadm-certs`.
   - Để tải lên lại các certificate và tạo một khóa giải mã (decryption key) mới, hãy dùng lệnh sau trên
     một node control plane
     đã được join vào cluster:

     ```sh
     sudo kubeadm init phase upload-certs --upload-certs
     ```

   - Bạn cũng có thể chỉ định một `--certificate-key` tùy chỉnh trong lúc `init` để sau này có thể dùng
     cho `join`. Để tạo một khóa như vậy, bạn có thể dùng lệnh sau:

     ```sh
     kubeadm certs certificate-key
     ```

   Certificate key là một chuỗi được mã hóa hex, là một khóa AES có kích thước 32 byte.

   > **Ghi chú:** Secret `kubeadm-certs` và khóa giải mã sẽ hết hạn sau hai giờ.

   > **Thận trọng:** Như đã nêu trong output của lệnh, certificate key cho phép truy cập vào dữ liệu nhạy cảm của cluster, hãy giữ bí mật nó!

1. Áp dụng plugin CNI mà bạn chọn:
   [Làm theo các hướng dẫn này](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network)
   để cài đặt CNI provider. Đảm bảo cấu hình tương ứng với Pod CIDR đã được chỉ định trong
   file cấu hình kubeadm (nếu có).

   > **Ghi chú:** Bạn phải chọn một network plugin phù hợp với trường hợp sử dụng của mình và triển khai nó trước khi chuyển sang bước tiếp theo.
   > Nếu bạn không làm điều này, bạn sẽ không thể khởi chạy cluster của mình một cách bình thường.

1. Gõ lệnh sau và theo dõi các pod của các thành phần control plane được khởi động:

   ```sh
   kubectl get pod -n kube-system -w
   ```

### Các bước cho các node control plane còn lại (Steps for the rest of the control plane nodes)

Với mỗi node control plane bổ sung, bạn cần:

1. Thực thi lệnh join mà output của `kubeadm init` trên node đầu tiên đã cung cấp cho bạn trước đó.
   Lệnh này trông giống như sau:

   ```sh
   sudo kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 --control-plane --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07
   ```

   - Cờ `--control-plane` yêu cầu `kubeadm join` tạo một control plane mới.
   - Cờ `--certificate-key ...` sẽ khiến các certificate của control plane được tải xuống
     từ Secret `kubeadm-certs` trong cluster và được giải mã bằng khóa đã cho.

> **Ghi chú:** Vì các node của cluster thường được khởi tạo tuần tự, các Pod CoreDNS nhiều khả năng sẽ đều chạy
> trên node control plane đầu tiên. Để cung cấp tính sẵn sàng cao hơn, hãy cân bằng lại các Pod CoreDNS
> bằng lệnh `kubectl -n kube-system rollout restart deployment coredns` sau khi có ít nhất một node mới được join.

## Các node etcd bên ngoài (External etcd nodes)

Việc thiết lập một cluster với các node etcd bên ngoài tương tự với quy trình dùng cho stacked etcd,
ngoại trừ việc bạn cần thiết lập etcd trước, và bạn cần truyền thông tin etcd
vào file cấu hình kubeadm.

### Thiết lập cluster etcd (Set up the etcd cluster)

1. Làm theo các [hướng dẫn này](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/setup-ha-etcd-with-kubeadm/) để thiết lập cluster etcd.

1. Thiết lập SSH như được mô tả [tại đây](#manual-certs).

1. Sao chép các file sau từ bất kỳ node etcd nào trong cluster đến node control plane đầu tiên:

   ```sh
   export CONTROL_PLANE="ubuntu@10.0.0.7"
   scp /etc/kubernetes/pki/etcd/ca.crt "${CONTROL_PLANE}":
   scp /etc/kubernetes/pki/apiserver-etcd-client.crt "${CONTROL_PLANE}":
   scp /etc/kubernetes/pki/apiserver-etcd-client.key "${CONTROL_PLANE}":
   ```

   - Thay giá trị của `CONTROL_PLANE` bằng `user@host` của node control-plane đầu tiên.

### Thiết lập node control plane đầu tiên (Set up the first control plane node)

1. Tạo một file tên là `kubeadm-config.yaml` với nội dung sau:

   ```yaml
   ---
   apiVersion: kubeadm.k8s.io/v1beta4
   kind: ClusterConfiguration
   kubernetesVersion: stable
   controlPlaneEndpoint: "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" # thay đổi giá trị này (xem bên dưới)
   etcd:
     external:
       endpoints:
         - https://ETCD_0_IP:2379 # thay ETCD_0_IP cho phù hợp
         - https://ETCD_1_IP:2379 # thay ETCD_1_IP cho phù hợp
         - https://ETCD_2_IP:2379 # thay ETCD_2_IP cho phù hợp
       caFile: /etc/kubernetes/pki/etcd/ca.crt
       certFile: /etc/kubernetes/pki/apiserver-etcd-client.crt
       keyFile: /etc/kubernetes/pki/apiserver-etcd-client.key
   ```

   > **Ghi chú:** Sự khác biệt giữa stacked etcd và external etcd ở đây là thiết lập external etcd yêu cầu
   > một file cấu hình với các endpoint của etcd nằm dưới đối tượng `external` của `etcd`.
   > Trong trường hợp topology stacked etcd, điều này được quản lý tự động.

   - Thay các biến sau trong mẫu cấu hình bằng các giá trị phù hợp cho cluster của bạn:

     - `LOAD_BALANCER_DNS`
     - `LOAD_BALANCER_PORT`
     - `ETCD_0_IP`
     - `ETCD_1_IP`
     - `ETCD_2_IP`

Các bước sau đây tương tự với thiết lập stacked etcd:

1. Chạy `sudo kubeadm init --config kubeadm-config.yaml --upload-certs` trên node này.

1. Ghi các lệnh join trong output trả về vào một file văn bản để dùng sau.

1. Áp dụng plugin CNI mà bạn chọn.

   > **Ghi chú:** Bạn phải chọn một network plugin phù hợp với trường hợp sử dụng của mình và triển khai nó trước khi chuyển sang bước tiếp theo.
   > Nếu bạn không làm điều này, bạn sẽ không thể khởi chạy cluster của mình một cách bình thường.

### Các bước cho các node control plane còn lại (Steps for the rest of the control plane nodes)

Các bước giống như với thiết lập stacked etcd:

- Đảm bảo node control plane đầu tiên đã được khởi tạo hoàn toàn.
- Join từng node control plane bằng lệnh join mà bạn đã lưu vào file văn bản. Khuyến nghị
  join các node control plane từng node một.
- Đừng quên rằng khóa giải mã từ `--certificate-key` mặc định sẽ hết hạn sau hai giờ.

## Các tác vụ thường gặp sau khi khởi tạo control plane (Common tasks after bootstrapping control plane)

### Cài đặt các worker (Install workers)

Các worker node có thể được join vào cluster bằng lệnh mà bạn đã lưu trước đó
từ output của lệnh `kubeadm init`:

```sh
sudo kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866
```

## Phân phối certificate thủ công (Manual certificate distribution) {#manual-certs}

Nếu bạn chọn không dùng `kubeadm init` với cờ `--upload-certs`, điều này có nghĩa là
bạn sẽ phải tự sao chép các certificate từ node control plane chính đến các
node control plane sắp join.

Có nhiều cách để làm việc này. Ví dụ sau đây dùng `ssh` và `scp`:

SSH là bắt buộc nếu bạn muốn điều khiển tất cả các node từ một máy duy nhất.

1. Bật ssh-agent trên thiết bị chính của bạn — thiết bị có quyền truy cập đến tất cả các node khác
   trong hệ thống:

   ```shell
   eval $(ssh-agent)
   ```

1. Thêm SSH identity của bạn vào phiên làm việc:

   ```shell
   ssh-add ~/.ssh/path_to_private_key
   ```

1. SSH giữa các node để kiểm tra rằng kết nối hoạt động bình thường.

   - Khi bạn SSH đến bất kỳ node nào, hãy thêm cờ `-A`. Cờ này cho phép node mà bạn
     đã đăng nhập qua SSH truy cập SSH agent trên PC của bạn. Hãy cân nhắc các phương pháp
     thay thế nếu bạn không hoàn toàn tin tưởng tính bảo mật của phiên người dùng trên node.

     ```shell
     ssh -A 10.0.0.7
     ```

   - Khi dùng sudo trên bất kỳ node nào, hãy đảm bảo giữ nguyên môi trường (environment) để SSH
     forwarding hoạt động:

     ```shell
     sudo -E -s
     ```

1. Sau khi cấu hình SSH trên tất cả các node, bạn nên chạy script sau trên node
   control plane đầu tiên sau khi đã chạy `kubeadm init`. Script này sẽ sao chép các certificate từ
   node control plane đầu tiên đến các node control plane khác:

   Trong ví dụ sau, hãy thay `CONTROL_PLANE_IPS` bằng địa chỉ IP của các
   node control plane khác.

   ```sh
   USER=ubuntu # có thể tùy chỉnh
   CONTROL_PLANE_IPS="10.0.0.7 10.0.0.8"
   for host in ${CONTROL_PLANE_IPS}; do
       scp /etc/kubernetes/pki/ca.crt "${USER}"@$host:
       scp /etc/kubernetes/pki/ca.key "${USER}"@$host:
       scp /etc/kubernetes/pki/sa.key "${USER}"@$host:
       scp /etc/kubernetes/pki/sa.pub "${USER}"@$host:
       scp /etc/kubernetes/pki/front-proxy-ca.crt "${USER}"@$host:
       scp /etc/kubernetes/pki/front-proxy-ca.key "${USER}"@$host:
       scp /etc/kubernetes/pki/etcd/ca.crt "${USER}"@$host:etcd-ca.crt
       # Bỏ qua dòng tiếp theo nếu bạn đang dùng external etcd
       scp /etc/kubernetes/pki/etcd/ca.key "${USER}"@$host:etcd-ca.key
   done
   ```

   > **Thận trọng:** Chỉ sao chép các certificate trong danh sách ở trên. kubeadm sẽ lo việc tạo phần certificate còn lại
   > với các SAN cần thiết cho các instance control-plane sắp join. Nếu bạn sao chép nhầm tất cả các certificate,
   > việc tạo các node bổ sung có thể thất bại do thiếu các SAN cần thiết.

1. Sau đó, trên mỗi node control plane sắp join, bạn phải chạy script sau trước khi chạy `kubeadm join`.
   Script này sẽ di chuyển các certificate đã được sao chép trước đó từ thư mục home vào `/etc/kubernetes/pki`:

   ```sh
   USER=ubuntu # có thể tùy chỉnh
   mkdir -p /etc/kubernetes/pki/etcd
   mv /home/${USER}/ca.crt /etc/kubernetes/pki/
   mv /home/${USER}/ca.key /etc/kubernetes/pki/
   mv /home/${USER}/sa.pub /etc/kubernetes/pki/
   mv /home/${USER}/sa.key /etc/kubernetes/pki/
   mv /home/${USER}/front-proxy-ca.crt /etc/kubernetes/pki/
   mv /home/${USER}/front-proxy-ca.key /etc/kubernetes/pki/
   mv /home/${USER}/etcd-ca.crt /etc/kubernetes/pki/etcd/ca.crt
   # Bỏ qua dòng tiếp theo nếu bạn đang dùng external etcd
   mv /home/${USER}/etcd-ca.key /etc/kubernetes/pki/etcd/ca.key
   ```

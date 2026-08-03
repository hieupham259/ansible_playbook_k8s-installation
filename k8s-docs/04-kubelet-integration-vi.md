# Cấu hình từng kubelet trong cluster bằng kubeadm (Configuring each kubelet in your cluster using kubeadm)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/>
>
> Trang này mô tả cách kubeadm quản lý cấu hình kubelet: các mẫu cấu hình cấp cluster và cấu hình riêng cho từng máy, quy trình khi chạy `kubeadm init` / `kubeadm join`, file drop-in của systemd cho kubelet, và nội dung các gói DEB/RPM của Kubernetes.

> **Ghi chú:** Dockershim đã bị xóa khỏi dự án Kubernetes kể từ bản phát hành 1.24. Đọc [Câu hỏi thường gặp về việc xóa Dockershim (Dockershim Removal FAQ)](https://kubernetes.io/dockershim) để biết thêm chi tiết.

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.11 [stable]`

Vòng đời của công cụ dòng lệnh kubeadm được tách rời khỏi
[kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet), vốn là một daemon chạy
trên mỗi node trong cluster Kubernetes. Công cụ dòng lệnh kubeadm được người dùng thực thi khi Kubernetes được
khởi tạo hoặc nâng cấp, trong khi kubelet luôn chạy nền.

Vì kubelet là một daemon, nó cần được duy trì bởi một loại hệ thống init
hoặc trình quản lý dịch vụ (service manager) nào đó. Khi kubelet được cài đặt bằng gói DEB hoặc RPM,
systemd được cấu hình để quản lý kubelet. Bạn có thể dùng một trình quản lý dịch vụ
khác, nhưng bạn cần tự cấu hình nó thủ công.

Một số chi tiết cấu hình kubelet cần giống nhau trên mọi kubelet tham gia cluster, trong khi
các khía cạnh cấu hình khác cần được thiết lập riêng cho từng kubelet để phù hợp với các
đặc điểm khác nhau của từng máy (chẳng hạn như hệ điều hành, lưu trữ và mạng). Bạn có thể quản lý cấu hình
của các kubelet một cách thủ công, nhưng kubeadm hiện cung cấp kiểu API `KubeletConfiguration` để
[quản lý tập trung cấu hình kubelet của bạn](#configure-kubelets-using-kubeadm).

## Các mẫu cấu hình kubelet (Kubelet configuration patterns)

Các phần sau mô tả những mẫu (pattern) cấu hình kubelet được đơn giản hóa nhờ
sử dụng kubeadm, thay vì phải quản lý cấu hình kubelet cho từng Node một cách thủ công.

### Lan truyền cấu hình cấp cluster tới từng kubelet (Propagating cluster-level configuration to each kubelet) {#propagating-cluster-level-configuration-to-each-kubelet}

Bạn có thể cung cấp cho kubelet các giá trị mặc định sẽ được các lệnh `kubeadm init` và `kubeadm join`
sử dụng. Các ví dụ đáng chú ý gồm việc dùng một container runtime khác hoặc đặt subnet mặc định
mà các Service sử dụng.

Nếu bạn muốn các Service của mình dùng subnet `10.96.0.0/12` làm mặc định cho Service, bạn có thể truyền
tham số `--service-cidr` cho kubeadm:

```bash
kubeadm init --service-cidr 10.96.0.0/12
```

Các IP ảo (Virtual IP) cho Service giờ đây được cấp phát từ subnet này. Bạn cũng cần đặt địa chỉ DNS mà
kubelet sử dụng, thông qua cờ `--cluster-dns`. Thiết lập này cần giống nhau cho mọi kubelet
trên mọi máy quản lý (manager) và Node trong cluster. Kubelet cung cấp một đối tượng API có cấu trúc và được đánh phiên bản
cho phép cấu hình hầu hết các tham số trong kubelet và đẩy cấu hình này ra từng
kubelet đang chạy trong cluster. Đối tượng này được gọi là
[`KubeletConfiguration`](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/).
`KubeletConfiguration` cho phép người dùng chỉ định các cờ, chẳng hạn như các địa chỉ IP DNS của cluster được biểu diễn dưới dạng
một danh sách giá trị cho một khóa viết theo kiểu camelCase, minh họa bằng ví dụ sau:

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
clusterDNS:
- 10.96.0.10
```

Để biết thêm chi tiết về `KubeletConfiguration`, hãy xem [phần này](#configure-kubelets-using-kubeadm).

### Cung cấp chi tiết cấu hình riêng cho từng máy (Providing instance-specific configuration details) {#providing-instance-specific-configuration-details}

Một số máy chủ (host) cần cấu hình kubelet riêng biệt do khác biệt về phần cứng, hệ điều hành,
mạng, hoặc các tham số đặc thù khác của máy. Danh sách sau đưa ra một vài ví dụ.

- Đường dẫn tới file phân giải DNS, được chỉ định bởi cờ cấu hình kubelet
  `--resolv-conf`, có thể khác nhau giữa các hệ điều hành, hoặc tùy thuộc vào việc bạn có dùng
  `systemd-resolved` hay không. Nếu đường dẫn này sai, việc phân giải DNS sẽ thất bại trên Node có kubelet
  được cấu hình không đúng.

- Đối tượng API Node có trường `.metadata.name` được đặt mặc định theo hostname của máy,
  trừ khi bạn dùng một nhà cung cấp đám mây (cloud provider). Bạn có thể dùng cờ `--hostname-override` để ghi đè
  hành vi mặc định nếu bạn cần chỉ định tên Node khác với hostname của máy.

- Hiện tại, kubelet không thể tự động phát hiện cgroup driver mà container runtime đang dùng,
  nhưng giá trị của `--cgroup-driver` phải khớp với cgroup driver mà container runtime sử dụng để đảm bảo
  kubelet hoạt động khỏe mạnh.

- Để chỉ định container runtime, bạn phải đặt endpoint của nó bằng cờ
`--container-runtime-endpoint=<path>`.

Cách được khuyến nghị để áp dụng những cấu hình riêng cho từng máy như vậy là sử dụng
[các bản vá (patch) `KubeletConfiguration`](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/control-plane-flags#patches).

## Cấu hình kubelet bằng kubeadm (Configure kubelets using kubeadm) {#configure-kubelets-using-kubeadm}

Có thể cấu hình kubelet mà kubeadm sẽ khởi động nếu một đối tượng API
[`KubeletConfiguration`](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)
tùy chỉnh được truyền vào thông qua một file cấu hình như sau: `kubeadm ... --config some-config-file.yaml`.

Bằng cách gọi `kubeadm config print init-defaults --component-configs KubeletConfiguration`, bạn có thể
xem tất cả các giá trị mặc định cho cấu trúc này.

Cũng có thể áp dụng các bản vá riêng cho từng máy đè lên `KubeletConfiguration` cơ sở.
Hãy xem [Tùy chỉnh kubelet](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/control-plane-flags#customizing-the-kubelet)
để biết thêm chi tiết.

### Quy trình khi dùng `kubeadm init` (Workflow when using `kubeadm init`)

Khi bạn gọi `kubeadm init`, cấu hình kubelet được ghi (marshal) ra đĩa
tại `/var/lib/kubelet/config.yaml`, đồng thời được tải lên một ConfigMap `kubelet-config`
trong namespace `kube-system` của cluster.
Ngoài ra, công cụ kubeadm phát hiện CRI socket trên node và ghi các chi tiết của nó
(bao gồm đường dẫn socket) vào một cấu hình cục bộ, `/var/lib/kubelet/instance-config.yaml`.
Một file cấu hình kubelet cũng được ghi vào `/etc/kubernetes/kubelet.conf`
chứa cấu hình cơ sở áp dụng toàn cluster cho mọi kubelet trong cluster. File cấu hình này
trỏ tới các chứng chỉ client cho phép kubelet giao tiếp với API server. Điều này
giải quyết nhu cầu
[lan truyền cấu hình cấp cluster tới từng kubelet](#propagating-cluster-level-configuration-to-each-kubelet).

Để giải quyết mẫu thứ hai —
[cung cấp chi tiết cấu hình riêng cho từng máy](#providing-instance-specific-configuration-details) —
kubeadm ghi một file môi trường (environment file) vào `/var/lib/kubelet/kubeadm-flags.env`, chứa danh sách
các cờ sẽ được truyền cho kubelet khi nó khởi động. Các cờ được trình bày trong file như sau:

```bash
KUBELET_KUBEADM_ARGS="--flag1=value1 --flag2=value2 ..."
```

Bên cạnh các cờ dùng khi khởi động kubelet, file này còn chứa các tham số
động như cgroup driver.

Sau khi ghi hai file này ra đĩa, kubeadm sẽ cố gắng chạy hai
lệnh sau, nếu bạn đang dùng systemd:

```bash
systemctl daemon-reload && systemctl restart kubelet
```

Nếu việc reload và restart thành công, quy trình `kubeadm init` bình thường sẽ tiếp tục.

### Quy trình khi dùng `kubeadm join` (Workflow when using `kubeadm join`)

Khi bạn chạy `kubeadm join`, kubeadm dùng thông tin xác thực Bootstrap Token để thực hiện
TLS bootstrap, qua đó lấy được thông tin xác thực cần thiết để tải xuống
ConfigMap `kubelet-config` và ghi nó vào `/var/lib/kubelet/config.yaml`.
Ngoài ra, công cụ kubeadm phát hiện CRI socket trên node và ghi các chi tiết của nó
(bao gồm đường dẫn socket) vào một cấu hình cục bộ, `/var/lib/kubelet/instance-config.yaml`.
File môi trường động được sinh ra theo cách hoàn toàn giống với `kubeadm init`.

Tiếp theo, `kubeadm` chạy hai lệnh sau để nạp cấu hình mới vào kubelet:

```bash
systemctl daemon-reload && systemctl restart kubelet
```

Sau khi kubelet nạp cấu hình mới, kubeadm ghi file KubeConfig
`/etc/kubernetes/bootstrap-kubelet.conf`, chứa một chứng chỉ CA và Bootstrap
Token. Chúng được kubelet dùng để thực hiện TLS Bootstrap và lấy được một thông tin xác thực
duy nhất, được lưu tại `/etc/kubernetes/kubelet.conf`.

Khi file `/etc/kubernetes/kubelet.conf` được ghi xong, kubelet đã hoàn tất quá trình TLS Bootstrap.
Kubeadm xóa file `/etc/kubernetes/bootstrap-kubelet.conf` sau khi hoàn thành TLS Bootstrap.

## File drop-in của kubelet cho systemd (The kubelet drop-in file for systemd) {#the-kubelet-drop-in-file-for-systemd}

`kubeadm` đi kèm với cấu hình quy định cách systemd nên chạy kubelet.
Lưu ý rằng lệnh kubeadm CLI không bao giờ đụng đến file drop-in này.

File cấu hình này, được cài đặt bởi [gói](https://github.com/kubernetes/release/blob/cd53840/cmd/krel/templates/latest/kubeadm/10-kubeadm.conf) `kubeadm`, được ghi vào
`/usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf` và được systemd sử dụng.
Nó bổ sung cho file
[`kubelet.service`](https://github.com/kubernetes/release/blob/cd53840/cmd/krel/templates/latest/kubelet/kubelet.service) cơ bản.

Nếu bạn muốn ghi đè thêm nữa, bạn có thể tạo một thư mục `/etc/systemd/system/kubelet.service.d/`
(không phải `/usr/lib/systemd/system/kubelet.service.d/`) và đặt các tùy chỉnh của riêng bạn vào một file trong đó.
Ví dụ, bạn có thể thêm một file cục bộ mới `/etc/systemd/system/kubelet.service.d/local-overrides.conf`
để ghi đè các thiết lập unit mà `kubeadm` đã cấu hình.

Dưới đây là những gì bạn thường thấy trong `/usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf`:

> **Ghi chú:** Nội dung bên dưới chỉ là một ví dụ. Nếu bạn không muốn dùng trình quản lý gói (package manager),
> hãy làm theo hướng dẫn trong phần ([Không dùng trình quản lý gói](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#k8s-install-2)).

```none
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# Đây là file mà "kubeadm init" và "kubeadm join" sinh ra lúc chạy (runtime), điền giá trị
# cho biến KUBELET_KUBEADM_ARGS một cách động
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# Đây là file mà người dùng có thể dùng để ghi đè các đối số của kubelet như một phương án cuối cùng. Tốt hơn hết,
# người dùng nên dùng đối tượng .NodeRegistration.KubeletExtraArgs trong các file cấu hình thay thế.
# KUBELET_EXTRA_ARGS nên được nạp (source) từ file này.
EnvironmentFile=-/etc/default/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
```

File này chỉ định các vị trí mặc định cho tất cả các file mà kubeadm quản lý cho kubelet.

- File KubeConfig dùng cho TLS Bootstrap là `/etc/kubernetes/bootstrap-kubelet.conf`,
  nhưng nó chỉ được dùng nếu `/etc/kubernetes/kubelet.conf` không tồn tại.
- File KubeConfig chứa định danh (identity) duy nhất của kubelet là `/etc/kubernetes/kubelet.conf`.
- File chứa ComponentConfig của kubelet là `/var/lib/kubelet/config.yaml`.
- File môi trường động chứa `KUBELET_KUBEADM_ARGS` được nạp từ `/var/lib/kubelet/kubeadm-flags.env`.
- File có thể chứa các cờ ghi đè do người dùng chỉ định với `KUBELET_EXTRA_ARGS` được nạp từ
  `/etc/default/kubelet` (đối với gói DEB), hoặc `/etc/sysconfig/kubelet` (đối với gói RPM). `KUBELET_EXTRA_ARGS`
  đứng cuối cùng trong chuỗi cờ và có độ ưu tiên cao nhất khi có xung đột thiết lập.

## Các file thực thi và nội dung gói của Kubernetes (Kubernetes binaries and package contents)

Các gói DEB và RPM đi kèm với các bản phát hành Kubernetes gồm:

| Tên gói | Mô tả |
|--------------|-------------|
| `kubeadm`    | Cài đặt công cụ dòng lệnh `/usr/bin/kubeadm` và [file drop-in của kubelet](#the-kubelet-drop-in-file-for-systemd) cho kubelet. |
| `kubelet`    | Cài đặt file thực thi `/usr/bin/kubelet`. |
| `kubectl`    | Cài đặt file thực thi `/usr/bin/kubectl`. |
| `cri-tools` | Cài đặt file thực thi `/usr/bin/crictl` từ [kho git cri-tools](https://github.com/kubernetes-sigs/cri-tools). |
| `kubernetes-cni` | Cài đặt các file thực thi trong `/opt/cni/bin` từ [kho git plugins](https://github.com/containernetworking/plugins). |

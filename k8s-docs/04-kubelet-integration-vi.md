# Cấu hình từng kubelet trong cluster bằng kubeadm (Configuring each kubelet in your cluster using kubeadm)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/>
>
> Trang này mô tả cách kubeadm quản lý cấu hình kubelet: các mẫu cấu hình cấp cluster và cấu hình riêng cho từng máy, quy trình khi chạy `kubeadm init` / `kubeadm join`, file drop-in của systemd cho kubelet, và nội dung các gói DEB/RPM của Kubernetes.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 8](LO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm), bài 4/9 ·
Kiểm chứng ở Lab 8a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài này trả lời một câu hỏi rất cụ thể: **`kubeadm init` và `kubeadm join` thực sự ghi những
file nào cho kubelet, và ai thắng khi các file đó mâu thuẫn nhau**. Nó là bài giải thích hậu
trường của hai lệnh bạn đã chạy ở
[A5.1](labs/LAB-00-MOI-TRUONG-1.35.7.md#a51-init-control-plane) và
[A5.3](labs/LAB-00-MOI-TRUONG-1.35.7.md#a53-join-hai-worker) của Lab 00 — và cũng giải thích vì sao
A4.3 dặn rằng kubelet restart liên tục **trước** `kubeadm init/join` là trạng thái dự kiến.
Đọc kèm một phiên SSH mở sẵn vào `k8s-master` để `ls` từng đường dẫn được nhắc tên.

**Phải hiểu ở lần đọc này:**

- Hai mẫu cấu hình đối lập nhau, ở hai mục đầu bài: phần **giống nhau toàn cluster** đi qua
  `KubeletConfiguration` (ví dụ `clusterDNS`), còn phần **riêng từng máy** — `--resolv-conf`,
  `--hostname-override`, `--cgroup-driver`, `--container-runtime-endpoint` — không thể nhét vào
  đó. Bài khuyến nghị dùng **patch `KubeletConfiguration`** cho nhóm thứ hai.
- *Quy trình khi dùng `kubeadm init`* ghi ra bốn thứ: `/var/lib/kubelet/config.yaml` (đồng thời
  tải lên ConfigMap `kubelet-config` trong namespace `kube-system`),
  `/var/lib/kubelet/instance-config.yaml` (CRI socket phát hiện được),
  `/etc/kubernetes/kubelet.conf`, và `/var/lib/kubelet/kubeadm-flags.env`. Rồi mới
  `systemctl daemon-reload && systemctl restart kubelet`.
- *Quy trình khi dùng `kubeadm join`* khác ở chiều dữ liệu: node **tải xuống** ConfigMap
  `kubelet-config` thay vì sinh ra nó. `/etc/kubernetes/bootstrap-kubelet.conf` chỉ là bước
  đệm và **bị kubeadm xóa** sau khi TLS Bootstrap sinh ra `/etc/kubernetes/kubelet.conf`.
- *File drop-in của kubelet cho systemd*: thứ tự biến trong `ExecStart` chính là thứ tự ưu
  tiên — `KUBELET_EXTRA_ARGS` (nạp từ `/etc/default/kubelet`) đứng **cuối cùng** nên **thắng
  khi xung đột**. Lệnh kubeadm CLI **không bao giờ đụng** vào file drop-in này; muốn ghi đè thì
  đặt file riêng trong `/etc/systemd/system/kubelet.service.d/`, **không phải** `/usr/lib/...`.
- *Các file thực thi và nội dung gói*: gói `kubeadm` cài cả binary lẫn **file drop-in cho
  kubelet**; `cri-tools` cài `crictl`; `kubernetes-cni` cài binary vào `/opt/cni/bin`. Đây đúng
  là năm gói mà A4.3 của Lab 00 ghim version.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ghi chú Dockershim ở đầu bài | cluster lab nói CRI thẳng với containerd | CP12 di chuyển khỏi dockershim |
| Cơ chế TLS Bootstrap sinh ra "thông tin xác thực duy nhất" thế nào | chưa học cách cấp và xoay certificate | giai đoạn 9, CP3 certificate |
| Toàn bộ field của `KubeletConfiguration` (`kubeadm config print init-defaults --component-configs`) | hàng chục field, chỉ tra khi cần | CP5 cấu hình lại cluster đang chạy |
| File `kubelet.service` cơ bản và cách cài không dùng package manager | Lab 00 cài bằng gói DEB | không cần |

---

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

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 8:

1. Bạn muốn `k8s-worker1` và `k8s-worker2` mỗi máy dùng một `--resolv-conf` khác nhau. Nhét
   giá trị đó vào `KubeletConfiguration` cấp cluster có được không? Bài khuyến nghị cách nào?
2. Sau `kubeadm join`, bạn SSH vào `k8s-worker2` và không tìm thấy
   `/etc/kubernetes/bootstrap-kubelet.conf`. Có phải node đã join hỏng không? File nào phải tồn
   tại thay cho nó?
3. Bạn thêm một cờ vào `/var/lib/kubelet/kubeadm-flags.env` rồi restart kubelet, nhưng kubelet
   vẫn chạy với giá trị cũ vì cùng cờ đó cũng có trong `/etc/default/kubelet`. File nào thắng,
   và bài giải thích bằng cái gì?
4. `kubeadm init` trên `k8s-master` ghi những file nào cho kubelet, và file nào trong số đó
   được đưa lên chính cluster để node join sau này lấy về?
5. Bạn cần ghi đè một thiết lập systemd unit của kubelet mà kubeadm đã cấu hình. Đặt file ở đâu,
   và vì sao bài dặn không sửa thẳng `10-kubeadm.conf`?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không.** `--resolv-conf` nằm đúng trong danh sách ví dụ của mục *Cung cấp chi tiết cấu hình
   riêng cho từng máy*: đường dẫn tới file phân giải DNS khác nhau giữa các hệ điều hành và tùy
   việc có dùng `systemd-resolved` hay không; đặt sai thì **phân giải DNS thất bại trên node
   đó**. `KubeletConfiguration` là chỗ cho phần **giống nhau trên mọi kubelet**. Cách được
   **khuyến nghị** cho phần riêng từng máy là **các bản vá `KubeletConfiguration`** — chính cơ
   chế patch của bài [03](03-control-plane-flags-vi.md), với target `kubeletconfiguration`.
2. **Không hỏng — đó là kết thúc bình thường.** `bootstrap-kubelet.conf` chỉ chứa CA certificate
   và Bootstrap Token, dùng để kubelet thực hiện TLS Bootstrap. Khi TLS Bootstrap xong, kubelet
   đã có thông tin xác thực **duy nhất** của riêng nó, lưu tại **`/etc/kubernetes/kubelet.conf`**,
   và **kubeadm xóa `bootstrap-kubelet.conf` sau khi hoàn thành**. Drop-in cũng nói cùng điều
   theo hướng khác: file bootstrap **chỉ được dùng nếu `/etc/kubernetes/kubelet.conf` không tồn tại**.
3. **`/etc/default/kubelet` thắng.** Bài giải thích bằng chính dòng `ExecStart` của drop-in:
   `$KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS`.
   `KUBELET_EXTRA_ARGS` được nạp từ `/etc/default/kubelet` (gói DEB) hoặc `/etc/sysconfig/kubelet`
   (gói RPM), **đứng cuối cùng trong chuỗi cờ và có độ ưu tiên cao nhất khi có xung đột thiết
   lập**. Còn `kubeadm-flags.env` là **file môi trường động** do kubeadm sinh lúc chạy — sửa tay
   vào đó cũng dễ bị `kubeadm init/join` ghi đè.
4. Bốn file: **`/var/lib/kubelet/config.yaml`** (ComponentConfig của kubelet),
   **`/var/lib/kubelet/instance-config.yaml`** (chi tiết CRI socket phát hiện được trên node),
   **`/etc/kubernetes/kubelet.conf`** (KubeConfig trỏ tới client certificate để nói chuyện với
   API server), và **`/var/lib/kubelet/kubeadm-flags.env`** (các cờ động, gồm cả cgroup driver).
   Thứ được đưa lên cluster là nội dung của `config.yaml`, dưới dạng **ConfigMap `kubelet-config`
   trong namespace `kube-system`** — đúng thứ mà `kubeadm join` tải xuống trên node mới.
5. Tạo thư mục **`/etc/systemd/system/kubelet.service.d/`** và đặt tùy chỉnh vào một file trong
   đó, ví dụ `local-overrides.conf`. Bài nêu rõ đường dẫn này **không phải**
   `/usr/lib/systemd/system/kubelet.service.d/` — nơi chứa `10-kubeadm.conf` do **gói `kubeadm`
   cài đặt**. Bẫy nằm ở chỗ dễ đọc câu "kubeadm CLI không bao giờ đụng đến file drop-in này"
   thành "sửa file đó là an toàn": nó nói CLI không ghi đè, chứ file vẫn thuộc quyền quản lý của
   gói, và cơ chế ghi đè đúng mà bài chỉ ra là đặt file riêng dưới `/etc/systemd/system/`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

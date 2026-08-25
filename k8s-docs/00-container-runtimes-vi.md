# Các container runtime (Container Runtimes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/setup/production-environment/container-runtimes/>
>
> Bạn cần cài đặt một container runtime (môi trường thực thi container) vào mỗi node trong cluster để các Pod có thể chạy ở đó. Trang này trình bày những việc liên quan và mô tả các tác vụ đi kèm khi thiết lập node.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 2](00-ALO-TRINH-ADMIN.md#giai-đoạn-2--container-và-runtime), bài 8/8 ·
Kiểm chứng ở [Lab 2](labs/LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md).

**Bài này được lộ trình tách làm hai lần dùng.** Lần này ở giai đoạn 2 bạn **chỉ đọc lý
thuyết**, đặc biệt là mục *Các cgroup driver*. Phần thao tác cài đặt để dành cho
[giai đoạn 8](00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm), khi bạn dựng lại
cluster có hiểu — xem điều chỉnh số 4 ở cuối lộ trình.

Bạn đã **chạy** phần lớn nội dung bài này ở [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) mục A4 mà chưa
biết vì sao. Đây là lúc đọc để hiểu những dòng lệnh đã gõ.

**Phải hiểu ở lần đọc này:**

- **Mục *Các cgroup driver* là trọng tâm duy nhất.** Kubelet và container runtime **bắt buộc
  phải dùng cùng một cgroup driver**. Đây là lỗi cấu hình kinh điển nhất khi tự dựng cluster.
- Vì sao `systemd` là lựa chọn đúng khi init system là systemd: dùng `cgroupfs` trong hoàn cảnh
  đó tạo ra **hai trình quản lý cgroup**, hai cách nhìn khác nhau về tài nguyên, và node trở
  nên **không ổn định khi chịu áp lực tài nguyên**.
- kubeadm từ v1.22 mặc định đặt `cgroupDriver: systemd` cho kubelet — nên phần bạn phải tự lo
  là **phía runtime** (`SystemdCgroup = true` trong `config.toml`, đúng như Lab 00 đã làm).
- **Đổi cgroup driver của node đã tham gia cluster là thao tác nhạy cảm**; cách an toàn là thay
  hoặc cài lại node, không phải sửa tại chỗ.
- Runtime bắt buộc hỗ trợ CRI `v1`; không có thì kubelet không đăng ký được node — nối với bài
  [44](44-cri-vi.md).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Toàn bộ các bước cài đặt containerd, CRI-O, Docker Engine, MCR | là thao tác dựng cluster | giai đoạn 8 |
| Bật chuyển tiếp gói tin IPv4, tham số sysctl | đã chạy ở Lab 00; hiểu khi học mạng | giai đoạn 5 và 8 |
| Ghi đè image sandbox (pause) | tinh chỉnh khi cần registry riêng | giai đoạn 8 |
| Feature gate `KubeletCgroupDriverFromCRI` | tự phát hiện driver, còn đang chuyển tiếp | giai đoạn 12 |
| Toàn bộ phần dockershim và di chuyển khỏi nó | chỉ cần khi tiếp quản cluster đời cũ | giai đoạn 27 |

---

> **Ghi chú:** Dockershim đã bị loại bỏ khỏi dự án Kubernetes kể từ bản phát hành 1.24. Đọc [Câu hỏi thường gặp về việc loại bỏ Dockershim](https://kubernetes.io/dockershim) để biết thêm chi tiết.

Bạn cần cài đặt một container runtime vào mỗi node trong cluster để các Pod có thể chạy ở đó.
Trang này trình bày những việc liên quan và mô tả các tác vụ đi kèm khi thiết lập node.

Kubernetes v1.36 yêu cầu bạn sử dụng một runtime tuân theo
Container Runtime Interface (CRI).

Xem [Hỗ trợ phiên bản CRI](#cri-versions) để biết thêm thông tin.

Trang này cung cấp hướng dẫn khái quát về cách sử dụng một số container runtime phổ biến với Kubernetes.

- [containerd](#containerd)
- [CRI-O](#cri-o)
- [Docker Engine](#docker)
- [Mirantis Container Runtime](#mcr)

> **Ghi chú:** Các bản phát hành Kubernetes trước v1.24 có tích hợp trực tiếp với Docker Engine,
> thông qua một thành phần tên là _dockershim_. Tích hợp trực tiếp đặc biệt đó không còn
> là một phần của Kubernetes (việc loại bỏ này đã được
> [công bố](https://kubernetes.io/blog/2020/12/08/kubernetes-1-20-release-announcement/#dockershim-deprecation)
> trong bản phát hành v1.20).
> Bạn có thể đọc
> [Kiểm tra xem việc loại bỏ Dockershim có ảnh hưởng đến bạn không](238-check-dockershim-removal-vi.md)
> để hiểu việc loại bỏ này có thể ảnh hưởng đến bạn như thế nào. Để tìm hiểu về việc di chuyển khỏi dockershim, xem
> [Di chuyển khỏi dockershim](236-migrating-from-dockershim-vi.md).
>
> Nếu bạn đang chạy một phiên bản Kubernetes khác v1.36, hãy xem tài liệu của phiên bản đó.

## Cài đặt và cấu hình các điều kiện tiên quyết (Install and configure prerequisites)

### Cấu hình mạng (Network configuration)

Theo mặc định, Linux kernel không cho phép các gói tin IPv4 được định tuyến
giữa các interface. Hầu hết các triển khai mạng cho cluster Kubernetes
sẽ thay đổi thiết lập này (nếu cần), nhưng một số có thể kỳ vọng
quản trị viên tự làm việc đó. (Một số còn có thể yêu cầu thiết lập các tham số
sysctl khác, nạp các kernel module, v.v.; hãy tham khảo
tài liệu của triển khai mạng cụ thể mà bạn dùng.)

### Bật chuyển tiếp gói tin IPv4 (Enable IPv4 packet forwarding) {#prerequisite-ipv4-forwarding-optional}

Để bật chuyển tiếp gói tin (packet forwarding) IPv4 thủ công:

```bash
# Các tham số sysctl cần cho việc cài đặt, được giữ nguyên qua các lần khởi động lại
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Áp dụng các tham số sysctl mà không cần khởi động lại
sudo sysctl --system
```

Xác minh rằng `net.ipv4.ip_forward` được đặt thành 1 bằng lệnh:

```bash
sysctl net.ipv4.ip_forward
```

## Các cgroup driver (cgroup drivers) {#cgroup-drivers}

Trên Linux, control group (cgroup) được dùng để giới hạn tài nguyên
cấp phát cho các tiến trình.

Cả kubelet lẫn container runtime bên dưới đều cần tương tác với control group
để thực thi
[quản lý tài nguyên cho pod và container](110-manage-resources-containers-vi.md)
và thiết lập các tài nguyên như yêu cầu (request) và giới hạn (limit) về CPU/bộ nhớ. Để tương tác
với control group, kubelet và container runtime cần dùng một *cgroup driver*.
Điều tối quan trọng là kubelet và container runtime phải dùng cùng một cgroup driver
và được cấu hình giống nhau.

Có hai cgroup driver:

* [`cgroupfs`](#cgroupfs-cgroup-driver)
* [`systemd`](#systemd-cgroup-driver)

### cgroupfs driver {#cgroupfs-cgroup-driver}

`cgroupfs` là [cgroup driver mặc định của kubelet](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1).
Khi dùng driver `cgroupfs`, kubelet và container runtime tương tác trực tiếp với
hệ thống tệp cgroup (cgroup filesystem) để cấu hình các cgroup.

Driver `cgroupfs` **không** được khuyến nghị khi
[systemd](https://www.freedesktop.org/wiki/Software/systemd/) là
init system, vì systemd kỳ vọng chỉ có một trình quản lý cgroup duy nhất
trên hệ thống. Ngoài ra, nếu bạn dùng [cgroup v2](33-cgroups-vi.md),
hãy dùng cgroup driver `systemd` thay cho `cgroupfs`.

### systemd cgroup driver {#systemd-cgroup-driver}

Khi [systemd](https://www.freedesktop.org/wiki/Software/systemd/) được chọn làm init
system cho một bản phân phối Linux, tiến trình init tạo ra và sử dụng một control group
(`cgroup`) gốc, đồng thời đóng vai trò trình quản lý cgroup.

systemd tích hợp chặt chẽ với cgroup và cấp phát một cgroup cho mỗi systemd
unit. Do đó, nếu bạn dùng `systemd` làm init system cùng với driver `cgroupfs`,
hệ thống sẽ có hai trình quản lý cgroup khác nhau.

Hai trình quản lý cgroup dẫn đến hai cách nhìn khác nhau về tài nguyên khả dụng và đang
sử dụng trên hệ thống. Trong một số trường hợp, các node được cấu hình dùng `cgroupfs`
cho kubelet và container runtime, nhưng dùng `systemd` cho phần còn lại của các tiến trình,
trở nên không ổn định khi chịu áp lực tài nguyên (resource pressure).

Cách giảm thiểu sự không ổn định này là dùng `systemd` làm cgroup driver cho
kubelet và container runtime khi systemd là init system được chọn.

Để đặt `systemd` làm cgroup driver, chỉnh sửa tùy chọn `cgroupDriver`
trong [`KubeletConfiguration`](224-kubelet-config-file-vi.md)
và đặt nó thành `systemd`. Ví dụ:

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
...
cgroupDriver: systemd
```

> **Ghi chú:** Từ v1.22 trở đi, khi tạo cluster bằng kubeadm, nếu người dùng không đặt
> trường `cgroupDriver` trong `KubeletConfiguration`, kubeadm sẽ mặc định đặt nó thành `systemd`.

Nếu bạn cấu hình `systemd` làm cgroup driver cho kubelet, bạn cũng phải
cấu hình `systemd` làm cgroup driver cho container runtime. Hãy tham khảo
tài liệu của container runtime bạn dùng để biết hướng dẫn. Ví dụ:

* [containerd](#containerd-systemd)
* [CRI-O](#cri-o)

Trong Kubernetes v1.36, khi [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`KubeletCgroupDriverFromCRI` được bật và container runtime hỗ trợ RPC CRI `RuntimeConfig`,
kubelet sẽ tự động phát hiện cgroup driver phù hợp từ runtime
và bỏ qua thiết lập `cgroupDriver` trong cấu hình của kubelet.

Tuy nhiên, các phiên bản container runtime cũ hơn (cụ thể là
containerd 1.y trở xuống) không hỗ trợ RPC CRI `RuntimeConfig` và
có thể không phản hồi đúng truy vấn này, khi đó kubelet sẽ quay về (fall back) dùng
giá trị trong cờ `--cgroup-driver` của chính nó.

Trong Kubernetes 1.38, hành vi dự phòng này sẽ bị loại bỏ, và các phiên bản containerd
cũ sẽ không hoạt động được với các kubelet mới hơn.

> **Thận trọng:** Thay đổi cgroup driver của một Node đã tham gia cluster là một thao tác nhạy cảm.
> Nếu kubelet đã tạo các Pod theo ngữ nghĩa của một cgroup driver, việc chuyển container
> runtime sang một cgroup driver khác có thể gây lỗi khi cố tạo lại Pod sandbox
> cho những Pod hiện có đó. Khởi động lại kubelet có thể không giải quyết được các lỗi này.
>
> Nếu bạn có hệ thống tự động hóa cho phép, hãy thay thế node bằng một node khác dùng
> cấu hình đã cập nhật, hoặc cài đặt lại node đó bằng công cụ tự động hóa.

### Di chuyển sang driver `systemd` trong các cluster do kubeadm quản lý (Migrating to the `systemd` driver in kubeadm managed clusters)

Nếu bạn muốn chuyển sang cgroup driver `systemd` trong các cluster hiện có do kubeadm quản lý,
hãy làm theo [cấu hình một cgroup driver](218-configure-cgroup-driver-vi.md).

## Hỗ trợ phiên bản CRI (CRI version support) {#cri-versions}

Container runtime của bạn phải hỗ trợ v1 của Container Runtime Interface.

Kubernetes [kể từ v1.26](https://kubernetes.io/blog/2022/11/18/upcoming-changes-in-kubernetes-1-26/#cri-api-removal)
_chỉ hoạt động_ với v1 của CRI API. Nếu container runtime không hỗ trợ API v1,
kubelet sẽ không đăng ký được thành node.

## Các container runtime (Container runtimes)

> **Ghi chú:** Mục này liên kết đến các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án này, vốn được liệt kê theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content) trước khi gửi thay đổi.

### containerd

Mục này trình bày các bước cần thiết để sử dụng containerd làm CRI runtime.

Để cài đặt containerd trên hệ thống của bạn, hãy làm theo hướng dẫn tại
[getting started with containerd](https://github.com/containerd/containerd/blob/main/docs/getting-started.md).
Quay lại bước này sau khi bạn đã tạo được file cấu hình `config.toml` hợp lệ.

#### Linux

Bạn có thể tìm thấy file này tại đường dẫn `/etc/containerd/config.toml`.

#### Windows

Bạn có thể tìm thấy file này tại đường dẫn `C:\Program Files\containerd\config.toml`.

Trên Linux, CRI socket mặc định của containerd là `/run/containerd/containerd.sock`.
Trên Windows, CRI endpoint mặc định là `npipe://./pipe/containerd-containerd`.

#### Cấu hình cgroup driver `systemd` (Configuring the `systemd` cgroup driver) {#containerd-systemd}

Để dùng cgroup driver `systemd` trong `/etc/containerd/config.toml` với `runc`,
đặt cấu hình sau tùy theo phiên bản containerd của bạn

Containerd phiên bản 1.x:

```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

Containerd phiên bản 2.x:

```
[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc]
  ...
  [plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]
    SystemdCgroup = true
```

Cgroup driver `systemd` được khuyến nghị nếu bạn dùng [cgroup v2](33-cgroups-vi.md).

> **Ghi chú:** Nếu bạn cài containerd từ gói (ví dụ RPM hoặc `.deb`), bạn có thể thấy
> plugin tích hợp CRI bị tắt theo mặc định.
>
> Bạn cần bật hỗ trợ CRI để dùng containerd với Kubernetes. Hãy đảm bảo `cri`
> không nằm trong danh sách `disabled_plugins` trong `/etc/containerd/config.toml`;
> nếu bạn có sửa file đó, hãy khởi động lại `containerd`.
>
> Nếu bạn gặp tình trạng container bị crash lặp đi lặp lại (crash loop) sau khi cài cluster lần đầu
> hoặc sau khi cài một CNI, cấu hình containerd đi kèm gói cài đặt có thể chứa
> các tham số cấu hình không tương thích. Cân nhắc đặt lại cấu hình containerd
> bằng `containerd config default > /etc/containerd/config.toml` như hướng dẫn trong
> [getting-started.md](https://github.com/containerd/containerd/blob/main/docs/getting-started.md#advanced-topics)
> rồi thiết lập lại các tham số cấu hình nêu trên cho phù hợp.

Nếu bạn áp dụng thay đổi này, nhớ khởi động lại containerd:

```shell
sudo systemctl restart containerd
```

Khi dùng kubeadm, hãy tự cấu hình
[cgroup driver cho kubelet](218-configure-cgroup-driver-vi.md#configuring-the-kubelet-cgroup-driver).

Trong Kubernetes v1.28, bạn có thể bật tính năng tự động phát hiện
cgroup driver dưới dạng tính năng alpha. Xem [systemd cgroup driver](#systemd-cgroup-driver)
để biết thêm chi tiết.

#### Ghi đè image sandbox (pause) (Overriding the sandbox (pause) image) {#override-pause-image-containerd}

Trong [cấu hình containerd](https://github.com/containerd/containerd/blob/main/docs/cri/config.md), bạn có thể ghi đè
image sandbox bằng cách đặt cấu hình sau:

```toml
[plugins."io.containerd.grpc.v1.cri"]
  sandbox_image = "registry.k8s.io/pause:3.10"
```

Bạn cũng có thể cần khởi động lại `containerd` sau khi cập nhật file cấu hình: `systemctl restart containerd`.

### CRI-O

Mục này trình bày các bước cần thiết để cài đặt CRI-O làm container runtime.

Để cài đặt CRI-O, hãy làm theo [Hướng dẫn cài đặt CRI-O](https://github.com/cri-o/packaging/blob/main/README.md#usage).

#### cgroup driver

CRI-O dùng cgroup driver systemd theo mặc định, và điều này nhiều khả năng phù hợp
với bạn. Để chuyển sang cgroup driver `cgroupfs`, hãy chỉnh sửa
`/etc/crio/crio.conf` hoặc đặt một file cấu hình bổ sung (drop-in) tại
`/etc/crio/crio.conf.d/02-cgroup-manager.conf`, ví dụ:

```toml
[crio.runtime]
conmon_cgroup = "pod"
cgroup_manager = "cgroupfs"
```

Bạn cũng nên lưu ý giá trị `conmon_cgroup` đã thay đổi, nó phải được đặt thành
`pod` khi dùng CRI-O với `cgroupfs`. Nói chung, cần giữ cấu hình
cgroup driver của kubelet (thường được thực hiện qua kubeadm) và của CRI-O
đồng bộ với nhau.

Trong Kubernetes v1.28, bạn có thể bật tính năng tự động phát hiện
cgroup driver dưới dạng tính năng alpha. Xem [systemd cgroup driver](#systemd-cgroup-driver)
để biết thêm chi tiết.

Với CRI-O, CRI socket mặc định là `/var/run/crio/crio.sock`.

#### Ghi đè image sandbox (pause) (Overriding the sandbox (pause) image) {#override-pause-image-cri-o}

Trong [cấu hình CRI-O](https://github.com/cri-o/cri-o/blob/main/docs/crio.conf.5.md), bạn có thể đặt
giá trị cấu hình sau:

```toml
[crio.image]
pause_image="registry.k8s.io/pause:3.10"
```

Tùy chọn cấu hình này hỗ trợ nạp lại cấu hình ngay khi đang chạy (live configuration reload) để áp dụng thay đổi: `systemctl reload crio` hoặc gửi
tín hiệu `SIGHUP` đến tiến trình `crio`.

### Docker Engine {#docker}

> **Ghi chú:** Các hướng dẫn này giả định bạn dùng adapter
> [`cri-dockerd`](https://mirantis.github.io/cri-dockerd/) để tích hợp
> Docker Engine với Kubernetes.

1. Trên mỗi node của bạn, cài đặt Docker cho bản phân phối Linux của bạn theo
   [Install Docker Engine](https://docs.docker.com/engine/install/#server).

2. Cài đặt [`cri-dockerd`](https://mirantis.github.io/cri-dockerd/usage/install), theo hướng dẫn trong mục cài đặt của tài liệu.

Với `cri-dockerd`, CRI socket mặc định là `/run/cri-dockerd.sock`.

### Mirantis Container Runtime {#mcr}

[Mirantis Container Runtime](https://docs.mirantis.com/mcr/25.0/overview.html) (MCR) là một container runtime
thương mại, trước đây có tên là Docker Enterprise Edition.

Bạn có thể dùng Mirantis Container Runtime với Kubernetes thông qua thành phần mã nguồn mở
[`cri-dockerd`](https://mirantis.github.io/cri-dockerd/), được đóng gói kèm MCR.

Để tìm hiểu thêm về cách cài đặt Mirantis Container Runtime,
xem [MCR Deployment Guide](https://docs.mirantis.com/mcr/25.0/install.html).

Kiểm tra systemd unit tên `cri-docker.socket` để tìm đường dẫn đến CRI
socket.

#### Ghi đè image sandbox (pause) (Overriding the sandbox (pause) image) {#override-pause-image-cri-dockerd-mcr}

Adapter `cri-dockerd` chấp nhận một đối số dòng lệnh để
chỉ định container image dùng làm Pod infrastructure container ("pause image").
Đối số dòng lệnh cần dùng là `--pod-infra-container-image`.

## Tiếp theo (What's next)

Bên cạnh container runtime, cluster của bạn còn cần một
[network plugin](157-networking-vi.md#how-to-implement-the-kubernetes-network-model)
hoạt động.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

1. Điều gì xảy ra nếu kubelet dùng cgroup driver `systemd` còn container runtime dùng
   `cgroupfs`? Vì sao triệu chứng chỉ lộ ra khi node chịu tải?
2. Ở Lab 00 bạn đặt `SystemdCgroup = true` cho containerd nhưng **không** đụng gì tới cấu hình
   kubelet. Vì sao vẫn khớp?
3. Một node đã chạy trong cluster và bạn muốn đổi cgroup driver của nó. Cách làm an toàn là gì,
   và vì sao không nên sửa tại chỗ rồi restart kubelet?
4. Bạn cài containerd từ gói `.deb` và cluster không lên. Bài nêu một nguyên nhân rất hay gặp
   liên quan tới plugin — đó là gì?
5. Vì sao lộ trình bắt bạn đọc bài này ở giai đoạn 2 nhưng để phần cài đặt lại tới giai đoạn 8?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Hệ thống có **hai trình quản lý cgroup**, dẫn tới **hai cách nhìn khác nhau về tài nguyên
   khả dụng và đang dùng**. Khi node rảnh thì chẳng ai nhận ra; chỉ khi có **áp lực tài nguyên**
   thì việc thống kê và áp giới hạn mới mâu thuẫn, và node trở nên không ổn định.
2. Vì **kubeadm từ v1.22 mặc định đặt `cgroupDriver: systemd`** cho kubelet khi người dùng
   không tự khai báo trong `KubeletConfiguration`. Nên phần duy nhất bạn phải tự lo là phía
   runtime — đúng thứ Lab 00 làm.
3. **Thay node bằng node mới với cấu hình đã cập nhật, hoặc cài lại node.** Sửa tại chỗ nguy
   hiểm vì kubelet đã tạo Pod theo ngữ nghĩa của driver cũ; đổi driver có thể gây lỗi khi tạo
   lại Pod sandbox cho những Pod hiện có, và **restart kubelet không sửa được** các lỗi đó.
4. **Plugin CRI bị tắt mặc định** trong cấu hình đi kèm gói. Phải bảo đảm `cri` không nằm trong
   `disabled_plugins` của `/etc/containerd/config.toml` rồi restart containerd. Bài cũng gợi ý
   nếu container crash loop sau khi cài cluster hoặc cài CNI thì đặt lại cấu hình bằng
   `containerd config default > /etc/containerd/config.toml`.
5. Để **cài runtime xong không bỏ máy không dùng suốt sáu giai đoạn**. Ở giai đoạn 2 bạn cần
   *hiểu* cgroup driver và CRI để nắm tầng dưới Pod; đến giai đoạn 8 bạn mới *dựng* cluster, và
   lúc đó mỗi bước cài đặt đều có nghĩa thay vì gõ theo hướng dẫn.

</details>

Đây là bài cuối của **Giai đoạn 2**. Trả lời được hết tám bài thì bạn sẵn sàng vào Lab 2.

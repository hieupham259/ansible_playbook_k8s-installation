# Chạy các thành phần Node của Kubernetes dưới người dùng không phải root (Running Kubernetes Node Components as a Non-root User)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/kubelet-in-userns/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 3 — Pod và cấu hình](00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình)
→ nhóm [3a. Pod và vòng đời](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời), bài thực hành 1/11 ·
[Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md) **không kiểm chứng bài này**: mục 1.1 của lab ghi rõ lý
do — rootless mode cần feature gate `KubeletInUserNamespace` và phải dựng lại toàn bộ node, lệch
khỏi baseline của Lab 00, mà lab thì không cài phần mềm.

Đây là bài **lệch mạch** của nhóm 3a: mọi bài khác nói về Pod, bài này nói về **thành phần Node**
— kubelet, CRI, OCI runtime và CNI. Đọc để tách bạch hai chuyện rất dễ lẫn, và để biết rootless
mode tồn tại. Phần lớn nội dung là con trỏ sang dự án bên thứ ba, không phải hướng dẫn làm được
ngay.

**Phải hiểu ở lần đọc này:**

- Đối tượng của bài là **các thành phần Node chạy dưới người dùng không phải root** (_rootless
  mode_), không phải Pod. Ghi chú ngay đầu bài tách bạch: nếu chỉ muốn chạy một Pod dưới người
  dùng không phải root thì đó là [SecurityContext](291-security-context-vi.md), việc khác hẳn.
- Điều kiện tiên quyết ở mục *Trước khi bạn bắt đầu*: **cgroup v2** (mục *Tạo cây cgroup được ủy
  quyền* nói thẳng cgroup v1 **không được hỗ trợ**), systemd với user session, một số giá trị
  sysctl, người dùng không đặc quyền phải có mặt trong `/etc/subuid` và `/etc/subgid`, và feature
  gate `KubeletInUserNamespace` — alpha từ v1.22.
- Feature gate đó làm đúng một việc dễ hình dung: cho kubelet **bỏ qua lỗi** khi đặt một danh
  sách sysctl (`vm.overcommit_memory`, `vm.panic_on_oom`, `kernel.panic`, …), bỏ qua lỗi khi mở
  `/dev/kmsg`, và cho kube-proxy bỏ qua lỗi khi đặt `RLIMIT_NOFILE`.
- Cây cgroup phải được systemd **ủy quyền** bằng `Delegate=yes`, nên cả containerd/CRI-O lẫn
  kubelet đều được cấu hình dùng `cgroupfs`, **không** dùng driver `systemd` — chính các dòng
  chú thích trong ba khối cấu hình nói lý do.
- Các hạn chế ở mục *Caveats*: volume "không cục bộ" như `nfs` và `iscsi` **không hoạt động**;
  chỉ `local`, `hostPath`, `emptyDir`, `configMap`, `secret` và `downwardAPI` được biết là chạy
  tốt. Một số CNI plugin cũng không chạy, Flannel (VXLAN) thì chạy tốt.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Chạy Kubernetes trong Rootless Docker/Podman bằng kind hoặc minikube | lộ trình không dùng kind hay minikube; cluster lab dựng bằng kubeadm | [giai đoạn 8](00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm) |
| sysbox, K3s, Usernetes | dự án bên thứ ba, chính bài cũng ghi Kubernetes không chịu trách nhiệm | không nằm trong lộ trình; cách dựng cluster mà lộ trình dùng ở [giai đoạn 8](00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm) |
| Cả mục *Tự tay triển khai một node chạy kubelet trong user namespace* — `unshare(2)`, cây cgroup ủy quyền, cấu hình mạng, cấu hình CRI | chính bài ghi mục này "dành cho các nhà phát triển bản phân phối, không dành cho người dùng cuối" | [giai đoạn 8](00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm), khi bạn dựng node thật |
| Hai khối YAML `KubeletConfiguration` và `KubeProxyConfiguration` | sửa cấu hình kubelet và kube-proxy của cluster đang chạy là chủ đề riêng | [giai đoạn 20](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.22 [alpha]`

Tài liệu này mô tả cách chạy các thành phần Node của Kubernetes như kubelet, CRI, OCI và CNI
mà không cần đặc quyền root, bằng cách sử dụng một user namespace.

Kỹ thuật này còn được gọi là _rootless mode_ (chế độ không cần root).

> **Ghi chú:**
>
> Tài liệu này mô tả cách chạy các thành phần Node của Kubernetes (và do đó cả các Pod) dưới
> một người dùng không phải root.
>
> Nếu bạn chỉ đang tìm cách chạy một Pod dưới người dùng không phải root, hãy xem
> [SecurityContext](291-security-context-vi.md).

## Trước khi bạn bắt đầu (Before you begin)

Kubernetes server của bạn phải ở phiên bản 1.22 hoặc mới hơn.
Để kiểm tra phiên bản, nhập lệnh `kubectl version`.

* [Bật Cgroup v2](https://rootlesscontaine.rs/getting-started/common/cgroup2/)
* [Bật systemd với user session](https://rootlesscontaine.rs/getting-started/common/login/)
* [Cấu hình một số giá trị sysctl, tùy theo bản phân phối Linux của host](https://rootlesscontaine.rs/getting-started/common/sysctl/)
* [Đảm bảo người dùng không đặc quyền của bạn được liệt kê trong `/etc/subuid` và `/etc/subgid`](https://rootlesscontaine.rs/getting-started/common/subuid/)
* Bật [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) `KubeletInUserNamespace`

## Chạy Kubernetes bên trong Rootless Docker/Podman (Running Kubernetes inside Rootless Docker/Podman)

### kind

[kind](https://kind.sigs.k8s.io/) hỗ trợ chạy Kubernetes bên trong Rootless Docker hoặc
Rootless Podman.

Xem [Chạy kind với Rootless Docker](https://kind.sigs.k8s.io/docs/user/rootless/).

### minikube

[minikube](https://minikube.sigs.k8s.io/) cũng hỗ trợ chạy Kubernetes bên trong Rootless Docker
hoặc Rootless Podman.

Xem tài liệu của Minikube:

* [Rootless Docker](https://minikube.sigs.k8s.io/docs/drivers/docker/)
* [Rootless Podman](https://minikube.sigs.k8s.io/docs/drivers/podman/)

## Chạy Kubernetes bên trong container không đặc quyền (Running Kubernetes inside Unprivileged Containers)

> **Ghi chú:** Mục này liên kết đến các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần.
> Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án này, vốn được liệt kê
> theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc
> [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content)
> trước khi gửi thay đổi.

### sysbox

[Sysbox](https://github.com/nestybox/sysbox) là một container runtime mã nguồn mở (tương tự
"runc") hỗ trợ chạy các workload cấp hệ thống như Docker và Kubernetes bên trong các container
không đặc quyền được cô lập bằng user namespace của Linux.

Xem [Sysbox Quick Start Guide: Kubernetes-in-Docker](https://github.com/nestybox/sysbox/blob/master/docs/quickstart/kind.md)
để biết thêm thông tin.

Sysbox hỗ trợ chạy Kubernetes bên trong các container không đặc quyền mà không yêu cầu Cgroup v2
và không cần feature gate `KubeletInUserNamespace`. Nó làm được điều này bằng cách phơi bày các
filesystem `/proc` và `/sys` được chế tác đặc biệt bên trong container, cộng với một số kỹ thuật
ảo hóa hệ điều hành nâng cao khác.

## Chạy Rootless Kubernetes trực tiếp trên host (Running Rootless Kubernetes directly on a host)

> **Ghi chú:** Mục này liên kết đến các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần.
> Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án này, vốn được liệt kê
> theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc
> [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content)
> trước khi gửi thay đổi.

### K3s

[K3s](https://k3s.io/) hỗ trợ rootless mode ở mức thử nghiệm (experimental).

Xem [Chạy K3s với Rootless mode](https://rancher.com/docs/k3s/latest/en/advanced/#running-k3s-with-rootless-mode-experimental)
để biết cách sử dụng.

### Usernetes

[Usernetes](https://github.com/rootless-containers/usernetes) là một bản phân phối tham chiếu
của Kubernetes có thể được cài đặt trong thư mục `$HOME` mà không cần đặc quyền root.

Usernetes hỗ trợ cả containerd và CRI-O làm CRI runtime.
Usernetes hỗ trợ cluster nhiều node bằng Flannel (VXLAN).

Xem [repo của Usernetes](https://github.com/rootless-containers/usernetes) để biết cách sử dụng.

## Tự tay triển khai một node chạy kubelet trong user namespace (Manually deploy a node that runs the kubelet in a user namespace) {#userns-the-hard-way}

Mục này cung cấp các gợi ý để chạy Kubernetes trong một user namespace một cách thủ công.

> **Ghi chú:**
>
> Mục này dành cho các nhà phát triển bản phân phối Kubernetes, không dành cho người dùng cuối.

### Tạo user namespace (Creating a user namespace)

Bước đầu tiên là tạo một user namespace.

Nếu bạn đang cố chạy Kubernetes trong một container đã có user namespace như Rootless
Docker/Podman hoặc LXC/LXD, thì bạn đã sẵn sàng và có thể chuyển sang tiểu mục tiếp theo.

Nếu không, bạn phải tự tạo user namespace bằng cách gọi `unshare(2)` với `CLONE_NEWUSER`.

Một user namespace cũng có thể được tách (unshare) bằng các công cụ dòng lệnh như:

- [`unshare(1)`](https://man7.org/linux/man-pages/man1/unshare.1.html)
- [RootlessKit](https://github.com/rootless-containers/rootlesskit)
- [become-root](https://github.com/giuseppe/become-root)

Sau khi tách user namespace, bạn cũng phải tách các namespace khác, chẳng hạn như mount
namespace.

Bạn *không* cần gọi `chroot()` hay `pivot_root()` sau khi tách mount namespace, tuy nhiên bạn
phải mount các filesystem ghi được (writable) lên một số thư mục *bên trong* namespace.

Ít nhất, các thư mục sau cần ghi được *bên trong* namespace (không phải *bên ngoài* namespace):

- `/etc`
- `/run`
- `/var/logs`
- `/var/lib/kubelet`
- `/var/lib/cni`
- `/var/lib/containerd` (với containerd)
- `/var/lib/containers` (với CRI-O)

### Tạo cây cgroup được ủy quyền (Creating a delegated cgroup tree)

Ngoài user namespace, bạn cũng cần có một cây cgroup ghi được với cgroup v2.

> **Ghi chú:**
>
> Việc Kubernetes hỗ trợ chạy các thành phần Node trong user namespace yêu cầu cgroup v2.
> Cgroup v1 không được hỗ trợ.

Nếu bạn đang cố chạy Kubernetes trong Rootless Docker/Podman hoặc LXC/LXD trên một host dùng
systemd, thì bạn đã sẵn sàng.

Nếu không, bạn phải tạo một systemd unit với thuộc tính `Delegate=yes` để ủy quyền
(delegate) một cây cgroup với quyền ghi.

Trên node của bạn, systemd phải được cấu hình sẵn để cho phép ủy quyền; để biết thêm chi tiết,
xem [cgroup v2](https://rootlesscontaine.rs/getting-started/common/cgroup2/) trong tài liệu của
Rootless Containers.

### Cấu hình mạng (Configuring network)

> **Ghi chú:** Mục này liên kết đến các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần.
> Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án này, vốn được liệt kê
> theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc
> [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content)
> trước khi gửi thay đổi.

Network namespace của các thành phần Node phải có một interface không phải loopback, interface
này có thể được cấu hình bằng, ví dụ,
[slirp4netns](https://github.com/rootless-containers/slirp4netns),
[VPNKit](https://github.com/moby/vpnkit), hoặc
[lxc-user-nic(1)](https://www.man7.org/linux/man-pages/man1/lxc-user-nic.1.html).

Các network namespace của các Pod có thể được cấu hình bằng các CNI plugin thông thường.
Với mạng nhiều node, Flannel (VXLAN, 8472/UDP) được biết là hoạt động tốt.

Các port như port của kubelet (10250/TCP) và các port của Service kiểu `NodePort` phải được
phơi ra (expose) từ network namespace của Node tới host bằng một bộ chuyển tiếp port (port
forwarder) bên ngoài, chẳng hạn như RootlessKit, slirp4netns, hoặc
[socat(1)](https://linux.die.net/man/1/socat).

Bạn có thể dùng port forwarder của K3s. Xem
[Chạy K3s ở Rootless Mode](https://rancher.com/docs/k3s/latest/en/advanced/#known-issues-with-rootless-mode)
để biết thêm chi tiết. Phần triển khai có thể được tìm thấy trong
[package `pkg/rootlessports`](https://github.com/k3s-io/k3s/blob/v1.22.3+k3s1/pkg/rootlessports/controller.go)
của k3s.

### Cấu hình CRI (Configuring CRI)

kubelet phụ thuộc vào một container runtime. Bạn nên triển khai một container runtime như
containerd hoặc CRI-O và đảm bảo nó đang chạy bên trong user namespace trước khi kubelet khởi
động.

#### containerd

Việc chạy CRI plugin của containerd trong user namespace được hỗ trợ từ containerd 1.4.

Chạy containerd trong một user namespace yêu cầu các cấu hình sau.

```toml
version = 2

[plugins."io.containerd.grpc.v1.cri"]
# Tắt AppArmor
  disable_apparmor = true
# Bỏ qua lỗi trong lúc đặt giá trị oom_score_adj
  restrict_oom_score_adj = true
# Tắt controller hugetlb của cgroup v2 (vì systemd không hỗ trợ ủy quyền controller hugetlb)
  disable_hugetlb_controller = true

[plugins."io.containerd.grpc.v1.cri".containerd]
# Dùng overlayfs không qua fuse cũng khả thi với kernel >= 5.11, nhưng yêu cầu tắt SELinux
  snapshotter = "fuse-overlayfs"

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
# Chúng ta dùng cgroupfs được systemd ủy quyền, nên không dùng driver SystemdCgroup
# (trừ khi bạn chạy một systemd khác bên trong namespace)
  SystemdCgroup = false
```

Đường dẫn mặc định của file cấu hình là `/etc/containerd/config.toml`.
Đường dẫn có thể được chỉ định bằng `containerd -c /path/to/containerd/config.toml`.

#### CRI-O

Việc chạy CRI-O trong user namespace được hỗ trợ từ CRI-O 1.22.

CRI-O yêu cầu biến môi trường `_CRIO_ROOTLESS=1` phải được đặt.

Các cấu hình sau cũng được khuyến nghị:

```toml
[crio]
  storage_driver = "overlay"
# Dùng overlayfs không qua fuse cũng khả thi với kernel >= 5.11, nhưng yêu cầu tắt SELinux
  storage_option = ["overlay.mount_program=/usr/local/bin/fuse-overlayfs"]

[crio.runtime]
# Chúng ta dùng cgroupfs được systemd ủy quyền, nên không dùng driver "systemd"
# (trừ khi bạn chạy một systemd khác bên trong namespace)
  cgroup_manager = "cgroupfs"
```

Đường dẫn mặc định của file cấu hình là `/etc/crio/crio.conf`.
Đường dẫn có thể được chỉ định bằng `crio --config /path/to/crio/crio.conf`.

### Cấu hình kubelet (Configuring kubelet)

Chạy kubelet trong một user namespace yêu cầu cấu hình sau:

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
featureGates:
  KubeletInUserNamespace: true
# Chúng ta dùng cgroupfs được systemd ủy quyền, nên không dùng driver "systemd"
# (trừ khi bạn chạy một systemd khác bên trong namespace)
cgroupDriver: "cgroupfs"
```

Khi feature gate `KubeletInUserNamespace` được bật, kubelet sẽ bỏ qua các lỗi có thể xảy ra
trong lúc đặt các giá trị sysctl sau trên node.

- `vm.overcommit_memory`
- `vm.panic_on_oom`
- `kernel.panic`
- `kernel.panic_on_oops`
- `kernel.keys.root_maxkeys`
- `kernel.keys.root_maxbytes`.

Bên trong user namespace, kubelet cũng bỏ qua mọi lỗi phát sinh khi cố mở `/dev/kmsg`.
Feature gate này cũng cho phép kube-proxy bỏ qua lỗi trong lúc đặt `RLIMIT_NOFILE`.

Feature gate `KubeletInUserNamespace` được giới thiệu trong Kubernetes v1.22 với trạng thái
"alpha".

Việc chạy kubelet trong user namespace mà không dùng feature gate này cũng khả thi bằng cách
mount một proc filesystem được chế tác đặc biệt (như cách
[Sysbox](https://github.com/nestybox/sysbox) làm), nhưng không được hỗ trợ chính thức.

### Cấu hình kube-proxy (Configuring kube-proxy)

Chạy kube-proxy trong một user namespace yêu cầu cấu hình sau:

```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "iptables" # hoặc "userspace"
conntrack:
# Bỏ qua việc đặt giá trị sysctl "net.netfilter.nf_conntrack_max"
  maxPerCore: 0
# Bỏ qua việc đặt "net.netfilter.nf_conntrack_tcp_timeout_established"
  tcpEstablishedTimeout: 0s
# Bỏ qua việc đặt "net.netfilter.nf_conntrack_tcp_timeout_close"
  tcpCloseWaitTimeout: 0s
```

## Các hạn chế (Caveats)

- Hầu hết các volume driver "không cục bộ" như `nfs` và `iscsi` không hoạt động.
  Các volume cục bộ như `local`, `hostPath`, `emptyDir`, `configMap`, `secret` và `downwardAPI`
  được biết là hoạt động tốt.

- Một số CNI plugin có thể không hoạt động. Flannel (VXLAN) được biết là hoạt động tốt.

Để biết thêm về điều này, xem trang
[Caveats and Future work](https://rootlesscontaine.rs/caveats/) trên website
rootlesscontaine.rs.

## Xem thêm (See Also)

- [rootlesscontaine.rs](https://rootlesscontaine.rs/)
- [Rootless Containers 2020 (KubeCon NA 2020)](https://www.slideshare.net/AkihiroSuda/kubecon-na-2020-containerd-rootless-containers-2020)
- [Chạy kind với Rootless Docker](https://kind.sigs.k8s.io/docs/user/rootless/)
- [Usernetes](https://github.com/rootless-containers/usernetes)
- [Chạy K3s với rootless mode](https://rancher.com/docs/k3s/latest/en/advanced/#running-k3s-with-rootless-mode-experimental)
- [KEP-2033: Kubelet-in-UserNS (aka Rootless mode)](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/2033-kubelet-in-userns-aka-rootless)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. **Câu bẫy.** Hai chuyện cùng có chữ "không phải root" và rất dễ lẫn: chạy **thành phần Node**
   dưới người dùng không phải root, và chạy **một Pod** dưới người dùng không phải root. Bài này
   nói về cái nào, và nó chỉ đường sang đâu cho cái còn lại?
2. Bạn muốn bật rootless mode cho `lab-k8s-worker2` của cluster lab. Theo mục *Trước khi bạn bắt
   đầu*, phải có sẵn những gì trên node đó trước khi kubelet khởi động được trong user namespace?
   Trong đó, thứ nào bài nói thẳng là **bắt buộc**, không có phương án thay thế?
3. Bật feature gate `KubeletInUserNamespace` thì kubelet đổi hành vi ở chỗ nào? Kể ba nhóm lỗi
   mà nó cho phép bỏ qua.
4. Cả containerd/CRI-O lẫn kubelet trong bài đều được cấu hình dùng `cgroupfs` chứ không dùng
   driver `systemd`. Lý do bài đưa ra là gì?
5. Cluster rootless chạy được rồi, bạn định cho workload dùng volume `nfs` và một CNI plugin bất
   kỳ. Mục *Caveats* cảnh báo gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Bài nói về **thành phần Node**: kubelet, CRI, OCI runtime và CNI — tức _rootless mode_, chạy
   cả cái nền của node dưới một người dùng không đặc quyền. Ghi chú ngay đầu bài chặn trước sự
   nhầm lẫn: **nếu bạn chỉ đang tìm cách chạy một Pod dưới người dùng không phải root, hãy xem
   [SecurityContext](291-security-context-vi.md)**. Chỗ dễ sai là tưởng hai việc chỉ khác nhau về
   quy mô; thật ra chúng nằm ở hai tầng khác nhau — một tầng là cách node được dựng, một tầng là
   trường trong manifest Pod.
2. Bài liệt kê năm thứ: **cgroup v2** đã bật, **systemd với user session**, một số **giá trị
   sysctl** tùy bản phân phối, người dùng không đặc quyền **có mặt trong `/etc/subuid` và
   `/etc/subgid`**, và feature gate **`KubeletInUserNamespace`**. Thứ bắt buộc không có đường lùi
   là **cgroup v2**: mục *Tạo cây cgroup được ủy quyền* ghi rõ **cgroup v1 không được hỗ trợ**.
   Ngoài ra còn cần một **cây cgroup ghi được**, ủy quyền qua systemd unit có `Delegate=yes`.
3. Nó cho kubelet **bỏ qua lỗi** thay vì chết. Ba nhóm: **một, lỗi khi đặt các sysctl**
   `vm.overcommit_memory`, `vm.panic_on_oom`, `kernel.panic`, `kernel.panic_on_oops`,
   `kernel.keys.root_maxkeys`, `kernel.keys.root_maxbytes`; **hai, lỗi khi cố mở `/dev/kmsg`**;
   **ba, lỗi của kube-proxy khi đặt `RLIMIT_NOFILE`**. Tất cả đều là những thao tác mà một tiến
   trình không đặc quyền không được phép làm.
4. Vì cây cgroup ở đây là **cây do systemd ủy quyền cho người dùng không đặc quyền**, chứ không
   phải cây mà một systemd đang chạy bên trong namespace quản lý. Chú thích trong cả ba khối cấu
   hình đều nói cùng một câu: *"Chúng ta dùng cgroupfs được systemd ủy quyền, nên không dùng
   driver systemd (trừ khi bạn chạy một systemd khác bên trong namespace)."*
5. Cảnh báo hai điều. Một, **hầu hết volume driver "không cục bộ" như `nfs` và `iscsi` không hoạt
   động**; chỉ các volume cục bộ — `local`, `hostPath`, `emptyDir`, `configMap`, `secret`,
   `downwardAPI` — được biết là chạy tốt. Hai, **một số CNI plugin có thể không hoạt động**, và
   thứ bài xác nhận chạy tốt là **Flannel (VXLAN)**.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

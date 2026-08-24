# Cài đặt kubeadm (Installing kubeadm)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/>
>
> Trang này hướng dẫn cách cài đặt bộ công cụ `kubeadm`.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 8](00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm), bài 1/9 ·
Kiểm chứng ở Lab 8a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bạn **đã chạy đúng các bước của bài này rồi** — mục A4 của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a4-chuẩn-bị-os-và-container-runtime) là bản copy-paste của
nó: [A4.1](labs/LAB-00-MOI-TRUONG-1.35.7.md#a41-cập-nhật-os-tắt-swap-và-bật-kernel-prerequisites) tắt
swap và bật sysctl, [A4.2](labs/LAB-00-MOI-TRUONG-1.35.7.md#a42-cài-containerd-và-runc-đúng-version)
cài containerd, [A4.3](labs/LAB-00-MOI-TRUONG-1.35.7.md#a43-cài-kubeadm-kubelet-kubectl-và-crictl) cài
ba binary. Lần này bạn đọc để **biết vì sao từng bước tồn tại**, không phải để gõ lại lệnh.
Trang gốc viết cho v1.36; cluster lab khóa ở v1.35.6, nên số minor trong URL kho `pkgs.k8s.io`
khác nhau — đó là khác biệt duy nhất về nội dung.

**Phải hiểu ở lần đọc này:**

- Ba gói có ba vai trò tách biệt (*Cài đặt kubeadm, kubelet và kubectl*): `kubeadm` chỉ bootstrap,
  `kubelet` là daemon chạy trên mọi máy, `kubectl` là CLI. kubeadm **không** cài và **không**
  quản lý `kubelet`/`kubectl`, nên việc khớp phiên bản là trách nhiệm của bạn.
- Quy tắc lệch phiên bản ở mục đó: lệch một minor giữa kubelet và control plane thì được, nhưng
  **kubelet không bao giờ được cao hơn API server**.
- *Cấu hình swap*: kubelet mặc định **không khởi động** khi thấy swap. Hai lối thoát là tắt swap
  hoặc `failSwapOn: false`; và ngay cả khi đã tolerate, workload vẫn không dùng được swap trừ
  khi đổi `swapBehavior` khác mặc định `NoSwap`. `swapoff -a` chỉ có tác dụng tới lần reboot kế.
- *Cấu hình cgroup driver*: cgroup driver của container runtime và của kubelet **bắt buộc khớp**,
  lệch là kubelet fail. Đây là lý do Lab 00 kiểm tra `SystemdCgroup` sau khi cài containerd.
- *Cài đặt container runtime*: kubeadm tự dò runtime bằng cách quét danh sách endpoint đã biết;
  phát hiện **nhiều hơn một** hoặc **không có cái nào** thì nó báo lỗi và bắt bạn chỉ định
  `--cri-socket`. Cộng với điều kiện ở *Trước khi bạn bắt đầu*: 2 GB RAM, 2 CPU cho control
  plane, và hostname / MAC / `product_uuid` phải duy nhất vì Kubernetes dùng chúng để định danh
  node.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Kiểm tra phiên bản hệ điều hành* — phần Windows | node Windows là giai đoạn riêng | giai đoạn 15 |
| *Các bản phân phối dựa trên Red Hat*, SELinux `permissive` | lab chạy Ubuntu 24.04.4 | không cần |
| *Không dùng trình quản lý gói* | Lab 00 cài bằng `apt` có ghim version | không cần |
| Cảnh báo "chú ý đặc biệt khi nâng cấp" và `apt-mark hold` | nâng cấp là quy trình riêng | CP2 nâng cấp |
| Docker Engine và `cri-dockerd` | cluster lab nói CRI thẳng với containerd | CP12 di chuyển khỏi dockershim |
| *Kiểm tra các network adapter* (nhiều adapter, thêm IP route) | ba VM lab chỉ có một default route | không cần |

---

<img src="https://kubernetes.io/images/kubeadm-stacked-color.png" align="right" width="150px"></img>
Trang này hướng dẫn cách cài đặt bộ công cụ `kubeadm`.
Để biết thông tin về cách tạo một cluster bằng kubeadm sau khi bạn đã hoàn tất quá trình cài đặt này,
hãy xem trang [Tạo cluster với kubeadm](02-create-cluster-kubeadm-vi.md).

Hướng dẫn cài đặt này dành cho Kubernetes v1.36. Nếu bạn muốn sử dụng một phiên bản Kubernetes khác, vui lòng tham khảo các trang sau:

* [Installing kubeadm (Kubernetes v1.35)](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
* [Installing kubeadm (Kubernetes v1.34)](https://v1-34.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
* [Installing kubeadm (Kubernetes v1.33)](https://v1-33.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
* [Installing kubeadm (Kubernetes v1.32)](https://v1-32.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

## Trước khi bạn bắt đầu (Before you begin) {#before-you-begin}

* Một máy chủ Linux tương thích. Dự án Kubernetes cung cấp hướng dẫn chung cho các bản phân phối Linux
  dựa trên Debian và Red Hat, cũng như các bản phân phối không có trình quản lý gói (package manager).
* Mỗi máy có từ 2 GB RAM trở lên (ít hơn sẽ còn rất ít dung lượng cho các ứng dụng của bạn).
* Từ 2 CPU trở lên đối với các máy control plane.
* Kết nối mạng đầy đủ giữa tất cả các máy trong cluster (mạng public hay private đều được).
* Hostname, địa chỉ MAC và product_uuid là duy nhất cho mỗi node. Xem [tại đây](#verify-mac-address) để biết thêm chi tiết.
* Một số port nhất định phải được mở trên các máy của bạn. Xem [tại đây](#check-required-ports) để biết thêm chi tiết.

> **Ghi chú:** Việc cài đặt `kubeadm` được thực hiện thông qua các tệp nhị phân (binary) sử dụng liên kết động (dynamic linking) và giả định rằng hệ thống đích của bạn cung cấp `glibc`.
> Đây là một giả định hợp lý trên nhiều bản phân phối Linux (bao gồm Debian, Ubuntu, Fedora, CentOS, v.v.)
> nhưng không phải lúc nào cũng đúng với các bản phân phối tùy biến và gọn nhẹ vốn không bao gồm `glibc` theo mặc định, chẳng hạn như Alpine Linux.
> Kỳ vọng là bản phân phối hoặc đã bao gồm `glibc`, hoặc có một
> [lớp tương thích](https://wiki.alpinelinux.org/wiki/Running_glibc_programs)
> cung cấp các symbol cần thiết.

## Kiểm tra phiên bản hệ điều hành (Check your OS version)

> **Ghi chú:** Mục này liên kết đến các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án này, vốn được liệt kê theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content) trước khi gửi thay đổi.

#### Linux

* Dự án kubeadm hỗ trợ các kernel LTS. Xem [Danh sách các kernel LTS](https://www.kernel.org/category/releases.html).
* Bạn có thể xem phiên bản kernel bằng lệnh `uname -r`

Để biết thêm thông tin, xem [Yêu cầu về Linux Kernel](https://kubernetes.io/docs/reference/node/kernel-version-requirements/).

#### Windows

* Dự án kubeadm hỗ trợ các phiên bản kernel gần đây. Để xem danh sách các kernel gần đây, xem [Thông tin phát hành Windows Server](https://learn.microsoft.com/en-us/windows/release-health/windows-server-release-info).
* Bạn có thể xem phiên bản kernel (còn gọi là phiên bản hệ điều hành) bằng lệnh `systeminfo`

Để biết thêm thông tin, xem [Khả năng tương thích các phiên bản hệ điều hành Windows](175-windows-intro-vi.md#windows-os-version-support).

Một cluster Kubernetes do kubeadm tạo ra phụ thuộc vào các phần mềm sử dụng những tính năng của kernel.
Các phần mềm này bao gồm, nhưng không giới hạn ở,
container runtime,
kubelet và một plugin Container Network Interface.

Để giúp bạn tránh các lỗi không mong muốn do phiên bản kernel không được hỗ trợ, kubeadm chạy bước kiểm tra tiền điều kiện (pre-flight check) `SystemVerification`.
Bước kiểm tra này sẽ thất bại nếu phiên bản kernel không được hỗ trợ.

Bạn có thể chọn bỏ qua bước kiểm tra này, nếu bạn biết rằng kernel của mình
cung cấp các tính năng cần thiết, ngay cả khi kubeadm không hỗ trợ phiên bản đó.

## Xác minh địa chỉ MAC và product_uuid là duy nhất trên mỗi node (Verify the MAC address and product_uuid are unique for every node) {#verify-mac-address}

* Bạn có thể xem địa chỉ MAC của các giao diện mạng (network interface) bằng lệnh `ip link` hoặc `ifconfig -a`
* Có thể kiểm tra product_uuid bằng lệnh `sudo cat /sys/class/dmi/id/product_uuid`

Rất có khả năng các thiết bị phần cứng sẽ có địa chỉ duy nhất, mặc dù một số máy ảo có thể có
các giá trị trùng nhau. Kubernetes sử dụng những giá trị này để định danh duy nhất các node trong cluster.
Nếu các giá trị này không phải là duy nhất cho mỗi node, quá trình cài đặt
có thể [thất bại](https://github.com/kubernetes/kubeadm/issues/31).

## Kiểm tra các network adapter (Check network adapters)

Nếu bạn có nhiều hơn một network adapter, và các thành phần Kubernetes không thể truy cập được qua
tuyến đường mặc định (default route), chúng tôi khuyên bạn nên thêm (các) IP route để các địa chỉ của cluster Kubernetes đi qua adapter phù hợp.

## Kiểm tra các port cần thiết (Check required ports) {#check-required-ports}

Các [port cần thiết](https://kubernetes.io/docs/reference/networking/ports-and-protocols/)
này phải được mở để các thành phần Kubernetes có thể giao tiếp với nhau.
Bạn có thể dùng các công cụ như [netcat](https://netcat.sourceforge.net) để kiểm tra một port có đang mở hay không. Ví dụ:

```shell
nc 127.0.0.1 6443 -zv -w 2
```

Plugin mạng cho pod (pod network plugin) mà bạn sử dụng cũng có thể yêu cầu mở một số port
nhất định. Vì điều này khác nhau đối với từng pod network plugin, vui lòng xem
tài liệu của các plugin về (các) port mà chúng cần.

## Cấu hình swap (Swap configuration) {#swap-configuration}

Hành vi mặc định của kubelet là không khởi động nếu phát hiện bộ nhớ swap trên node.
Điều này có nghĩa là swap hoặc phải bị vô hiệu hóa, hoặc phải được kubelet chấp nhận (tolerated).

* Để chấp nhận swap, thêm `failSwapOn: false` vào cấu hình của kubelet hoặc dưới dạng đối số dòng lệnh.
  Lưu ý: ngay cả khi đã cung cấp `failSwapOn: false`, các workload theo mặc định vẫn không có quyền truy cập swap.
  Điều này có thể thay đổi bằng cách đặt `swapBehavior`, cũng trong tệp cấu hình của kubelet. Để sử dụng swap,
  hãy đặt `swapBehavior` khác với giá trị mặc định `NoSwap`.
  Xem [Quản lý bộ nhớ swap](170-swap-memory-management-vi.md) để biết thêm chi tiết.
* Để vô hiệu hóa swap, có thể dùng `sudo swapoff -a` để tắt swap tạm thời.
  Để thay đổi này được duy trì qua các lần khởi động lại, hãy đảm bảo swap bị vô hiệu hóa trong
  các tệp cấu hình như `/etc/fstab`, `systemd.swap`, tùy vào cách nó được cấu hình trên hệ thống của bạn.

## Cài đặt container runtime (Installing a container runtime) {#installing-runtime}

Để chạy các container trong Pod, Kubernetes sử dụng một
container runtime.

Theo mặc định, Kubernetes sử dụng
Container Runtime Interface (CRI)
để giao tiếp với container runtime mà bạn đã chọn.

Nếu bạn không chỉ định một runtime, kubeadm sẽ tự động cố gắng phát hiện container runtime
đã cài đặt bằng cách quét qua một danh sách các endpoint đã biết.

Nếu phát hiện nhiều container runtime hoặc không phát hiện được cái nào, kubeadm sẽ báo lỗi
và yêu cầu bạn chỉ định runtime muốn sử dụng.

Xem [container runtimes](00-container-runtimes-vi.md)
để biết thêm thông tin.

> **Ghi chú:** Docker Engine không triển khai [CRI](https://kubernetes.io/docs/concepts/architecture/cri/),
> vốn là một yêu cầu để container runtime hoạt động được với Kubernetes.
> Vì lý do đó, cần cài đặt thêm một service là [cri-dockerd](https://mirantis.github.io/cri-dockerd/).
> cri-dockerd là một dự án dựa trên phần hỗ trợ Docker Engine tích hợp sẵn trước đây,
> vốn đã bị [loại bỏ](https://kubernetes.io/dockershim) khỏi kubelet ở phiên bản 1.24.

Các bảng dưới đây liệt kê các endpoint đã biết cho những hệ điều hành được hỗ trợ:

#### Linux

*Các container runtime trên Linux*

| Runtime                              | Đường dẫn đến Unix domain socket             |
|--------------------------------------|----------------------------------------------|
| containerd                           | `unix:///var/run/containerd/containerd.sock` |
| CRI-O                                | `unix:///var/run/crio/crio.sock`             |
| Docker Engine (sử dụng cri-dockerd)  | `unix:///var/run/cri-dockerd.sock`           |

#### Windows

*Các container runtime trên Windows*

| Runtime                              | Đường dẫn đến Windows named pipe             |
|--------------------------------------|----------------------------------------------|
| containerd                           | `npipe:////./pipe/containerd-containerd`     |
| Docker Engine (sử dụng cri-dockerd)  | `npipe:////./pipe/cri-dockerd`               |

## Cài đặt kubeadm, kubelet và kubectl (Installing kubeadm, kubelet and kubectl)

Bạn sẽ cài đặt các gói này trên tất cả các máy của mình:

* `kubeadm`: lệnh để khởi tạo (bootstrap) cluster.

* `kubelet`: thành phần chạy trên tất cả các máy trong cluster của bạn
  và thực hiện những việc như khởi động các pod và container.

* `kubectl`: công cụ dòng lệnh để giao tiếp với cluster của bạn.

kubeadm **sẽ không** cài đặt hay quản lý `kubelet` hoặc `kubectl` cho bạn, vì vậy bạn sẽ
cần đảm bảo chúng khớp với phiên bản của Kubernetes control plane mà bạn muốn
kubeadm cài đặt cho mình. Nếu không, sẽ có nguy cơ xảy ra lệch phiên bản (version skew),
có thể dẫn đến hành vi lỗi và không mong muốn. Tuy nhiên, lệch _một_ phiên bản minor giữa
kubelet và control plane là được hỗ trợ, nhưng phiên bản kubelet không bao giờ được vượt quá phiên bản
của API server. Ví dụ, kubelet chạy phiên bản 1.7.0 sẽ hoàn toàn tương thích với API server phiên bản 1.8.0,
nhưng điều ngược lại thì không.

Để biết thông tin về cách cài đặt `kubectl`, xem [Cài đặt và thiết lập kubectl](185-tools-vi.md).

> **Cảnh báo:** Các hướng dẫn này loại trừ tất cả các gói Kubernetes khỏi mọi đợt nâng cấp hệ thống.
> Lý do là kubeadm và Kubernetes đòi hỏi
> [sự chú ý đặc biệt khi nâng cấp](221-kubeadm-upgrade-vi.md).

Để biết thêm thông tin về lệch phiên bản, xem:

* [Chính sách phiên bản và lệch phiên bản](https://kubernetes.io/docs/setup/release/version-skew-policy/) của Kubernetes
* [Chính sách lệch phiên bản](02-create-cluster-kubeadm-vi.md#version-skew-policy) riêng của kubeadm

> **Ghi chú:** Các kho gói cũ (`apt.kubernetes.io` và `yum.kubernetes.io`) đã bị
> [ngưng sử dụng và đóng băng kể từ ngày 13-09-2023](https://kubernetes.io/blog/2023/08/31/legacy-package-repository-deprecation/).
> **Việc sử dụng [các kho gói mới được lưu trữ tại `pkgs.k8s.io`](https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/)
> được khuyến nghị mạnh mẽ và là bắt buộc để cài đặt các phiên bản Kubernetes phát hành sau ngày 13-09-2023.**
> Các kho cũ đã ngưng sử dụng, cùng nội dung của chúng, có thể bị xóa bất cứ lúc nào trong tương lai mà không cần
> thông báo thêm. Các kho gói mới cung cấp bản tải xuống cho các phiên bản Kubernetes kể từ v1.24.0.

> **Ghi chú:** Có một kho gói riêng cho mỗi phiên bản minor của Kubernetes. Nếu bạn muốn cài đặt
> một phiên bản minor khác với v1.36, vui lòng xem hướng dẫn cài đặt của
> phiên bản minor mà bạn mong muốn.

#### Các bản phân phối dựa trên Debian (Debian-based distributions)

Các hướng dẫn này dành cho Kubernetes v1.36.

1. Cập nhật chỉ mục gói `apt` và cài đặt các gói cần thiết để sử dụng kho `apt` của Kubernetes:

   ```shell
   sudo apt-get update
   # apt-transport-https có thể là một gói giả (dummy package); nếu vậy, bạn có thể bỏ qua gói đó
   sudo apt-get install -y apt-transport-https ca-certificates curl gpg
   ```

2. Tải xuống khóa ký công khai (public signing key) cho các kho gói của Kubernetes.
   Cùng một khóa ký được dùng cho tất cả các kho, vì vậy bạn có thể bỏ qua phần phiên bản trong URL:

   ```shell
   # Nếu thư mục `/etc/apt/keyrings` chưa tồn tại, cần tạo nó trước khi chạy lệnh curl, hãy đọc ghi chú bên dưới.
   # sudo mkdir -p -m 755 /etc/apt/keyrings
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
   ```

   > **Ghi chú:** Ở các bản phát hành cũ hơn Debian 12 và Ubuntu 22.04, thư mục `/etc/apt/keyrings` không
   > tồn tại theo mặc định và cần được tạo trước khi chạy lệnh curl.

3. Thêm kho `apt` phù hợp của Kubernetes. Xin lưu ý rằng kho này chỉ có các gói
   cho Kubernetes v1.36; đối với các phiên bản minor khác của Kubernetes, bạn cần
   thay đổi phiên bản minor của Kubernetes trong URL cho khớp với phiên bản minor mong muốn
   (bạn cũng nên kiểm tra rằng mình đang đọc tài liệu cho đúng phiên bản Kubernetes
   mà bạn dự định cài đặt).

   ```shell
   # Lệnh này ghi đè mọi cấu hình hiện có trong /etc/apt/sources.list.d/kubernetes.list
   echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
   ```

4. Cập nhật chỉ mục gói `apt`, cài đặt kubelet, kubeadm và kubectl, rồi ghim (pin) phiên bản của chúng:

   ```shell
   sudo apt-get update
   sudo apt-get install -y kubelet kubeadm kubectl
   sudo apt-mark hold kubelet kubeadm kubectl
   ```

5. (Tùy chọn) Kích hoạt service kubelet trước khi chạy kubeadm:

   ```shell
   sudo systemctl enable --now kubelet
   ```

#### Các bản phân phối dựa trên Red Hat (Red Hat-based distributions)

1. Đặt SELinux ở chế độ `permissive`:

   Các hướng dẫn này dành cho Kubernetes v1.36.

   ```shell
   # Đặt SELinux ở chế độ permissive (thực chất là vô hiệu hóa nó)
   sudo setenforce 0
   sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
   ```

   > **Thận trọng:**
   > - Việc đặt SELinux ở chế độ permissive bằng cách chạy `setenforce 0` và `sed ...`
   >   thực chất là vô hiệu hóa nó. Điều này là bắt buộc để cho phép các container truy cập
   >   filesystem của máy chủ; ví dụ, một số plugin mạng của cluster yêu cầu điều đó. Bạn phải
   >   làm như vậy cho đến khi phần hỗ trợ SELinux trong kubelet được cải thiện.
   > - Bạn có thể để SELinux bật nếu biết cách cấu hình nó, nhưng có thể sẽ cần
   >   những thiết lập mà kubeadm không hỗ trợ.

2. Thêm kho `yum` của Kubernetes. Tham số `exclude` trong định nghĩa
   kho đảm bảo rằng các gói liên quan đến Kubernetes
   không bị nâng cấp khi chạy `yum update`, vì có một quy trình đặc biệt
   phải tuân theo khi nâng cấp Kubernetes. Xin lưu ý rằng kho này
   chỉ có các gói cho Kubernetes v1.36; đối với các phiên bản
   minor khác của Kubernetes, bạn cần thay đổi phiên bản minor của Kubernetes
   trong URL cho khớp với phiên bản minor mong muốn (bạn cũng nên kiểm tra rằng
   mình đang đọc tài liệu cho đúng phiên bản Kubernetes mà bạn
   dự định cài đặt).

   ```shell
   # Lệnh này ghi đè mọi cấu hình hiện có trong /etc/yum.repos.d/kubernetes.repo
   cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
   [kubernetes]
   name=Kubernetes
   baseurl=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/
   enabled=1
   gpgcheck=1
   gpgkey=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/repodata/repomd.xml.key
   exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
   EOF
   ```

3. Cài đặt kubelet, kubeadm và kubectl:

   Đối với các hệ thống dùng DNF:
   ```shell
   sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
   ```
   Đối với các hệ thống dùng DNF5:
   ```shell
   sudo yum install -y kubelet kubeadm kubectl --setopt=disable_excludes=kubernetes
   ```

4. (Tùy chọn) Kích hoạt service kubelet trước khi chạy kubeadm:

   ```shell
   sudo systemctl enable --now kubelet
   ```

#### Không dùng trình quản lý gói (Without a package manager)

Cài đặt các plugin CNI (bắt buộc đối với hầu hết các pod network):

```bash
CNI_PLUGINS_VERSION="v1.3.0"
ARCH="amd64"
DEST="/opt/cni/bin"
sudo mkdir -p "$DEST"
curl -L "https://github.com/containernetworking/plugins/releases/download/${CNI_PLUGINS_VERSION}/cni-plugins-linux-${ARCH}-${CNI_PLUGINS_VERSION}.tgz" | sudo tar -C "$DEST" -xz
```

Định nghĩa thư mục để tải xuống các tệp lệnh:

> **Ghi chú:** Biến `DOWNLOAD_DIR` phải được đặt là một thư mục có quyền ghi.
> Nếu bạn đang chạy Flatcar Container Linux, hãy đặt `DOWNLOAD_DIR="/opt/bin"`.

```bash
DOWNLOAD_DIR="/usr/local/bin"
sudo mkdir -p "$DOWNLOAD_DIR"
```

Cài đặt crictl (tùy chọn) (bắt buộc khi tương tác với Container Runtime Interface (CRI), tùy chọn đối với kubeadm):

```bash
CRICTL_VERSION="v1.31.0"
ARCH="amd64"
curl -L "https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VERSION}/crictl-${CRICTL_VERSION}-linux-${ARCH}.tar.gz" | sudo tar -C $DOWNLOAD_DIR -xz
```

Cài đặt `kubeadm`, `kubelet` và thêm một systemd service cho `kubelet`:

```bash
RELEASE="$(curl -sSL https://dl.k8s.io/release/stable.txt)"
ARCH="amd64"
cd $DOWNLOAD_DIR
sudo curl -L --remote-name-all https://dl.k8s.io/release/${RELEASE}/bin/linux/${ARCH}/{kubeadm,kubelet}
sudo chmod +x {kubeadm,kubelet}

RELEASE_VERSION="v0.16.2"
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/krel/templates/latest/kubelet/kubelet.service" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /usr/lib/systemd/system/kubelet.service
sudo mkdir -p /usr/lib/systemd/system/kubelet.service.d
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/krel/templates/latest/kubeadm/10-kubeadm.conf" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
```

> **Ghi chú:** Vui lòng tham khảo ghi chú trong mục [Trước khi bạn bắt đầu](#before-you-begin) đối với các bản phân phối Linux
> không bao gồm `glibc` theo mặc định.

Cài đặt `kubectl` theo hướng dẫn trên [trang Cài đặt công cụ](185-tools-vi.md#kubectl).

Tùy chọn, kích hoạt service kubelet trước khi chạy kubeadm:

```bash
sudo systemctl enable --now kubelet
```

> **Ghi chú:** Bản phân phối Flatcar Container Linux mount thư mục `/usr` dưới dạng filesystem chỉ đọc (read-only).
> Trước khi khởi tạo (bootstrap) cluster của bạn, bạn cần thực hiện thêm các bước để cấu hình một thư mục có quyền ghi.
> Xem [Hướng dẫn xử lý sự cố kubeadm](09-troubleshooting-kubeadm-vi.md#usr-mounted-read-only)
> để biết cách thiết lập một thư mục có quyền ghi.

kubelet lúc này sẽ khởi động lại sau mỗi vài giây, do nó chờ trong vòng lặp crashloop để
kubeadm ra lệnh cho nó phải làm gì.

## Cấu hình cgroup driver (Configuring a cgroup driver)

Cả container runtime và kubelet đều có một thuộc tính gọi là
["cgroup driver"](00-container-runtimes-vi.md#cgroup-drivers), thuộc tính này rất quan trọng
đối với việc quản lý các cgroup trên máy Linux.

> **Cảnh báo:** Cgroup driver của container runtime và của kubelet bắt buộc phải khớp nhau, nếu không tiến trình kubelet sẽ thất bại.
>
> Xem [Cấu hình cgroup driver](218-configure-cgroup-driver-vi.md) để biết thêm chi tiết.

## Xử lý sự cố (Troubleshooting)

Nếu bạn gặp khó khăn với kubeadm, vui lòng tham khảo
[tài liệu xử lý sự cố](09-troubleshooting-kubeadm-vi.md) của chúng tôi.

## Tiếp theo (What's next)

* [Sử dụng kubeadm để tạo cluster](02-create-cluster-kubeadm-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 8:

1. Bạn cài `kubeadm` v1.35.6 trên một máy mới. Nó có tự kéo về `kubelet` và `kubectl` đúng
   phiên bản không? Nếu bạn để `apt` tự chọn version cho ba gói này thì rủi ro là gì?
2. Control plane của cluster lab chạy v1.35.6. Bạn được phép để kubelet trên `k8s-worker2` ở
   v1.36.0 không? Còn v1.34 thì sao?
3. Trên một VM mới bạn chạy `swapoff -a` rồi `kubeadm init` thành công. Sau lần reboot đầu
   tiên kubelet không lên nữa. Chuyện gì đã xảy ra, và
   [A4.1 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a41-cập-nhật-os-tắt-swap-và-bật-kernel-prerequisites)
   đã làm thêm gì để tránh?
4. Bạn để containerd ở `SystemdCgroup = false` nhưng kubelet dùng cgroup driver `systemd`. Node
   vẫn có RAM và CPU dư. Bài này nói điều gì sẽ xảy ra?
5. `kubeadm init` dừng lại và yêu cầu bạn chỉ định container runtime. Trước đó nó đã cố làm gì,
   và có mấy tình huống dẫn tới thông báo này?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không.** Bài nói thẳng: kubeadm **sẽ không** cài đặt hay quản lý `kubelet` hoặc `kubectl`
   cho bạn, vì vậy bạn phải tự đảm bảo chúng khớp với phiên bản control plane mà kubeadm sắp
   dựng. Để `apt` tự chọn thì ba gói có thể lệch nhau và lệch với control plane, dẫn tới
   **version skew** — bài mô tả hậu quả là "hành vi lỗi và không mong muốn". Đó là lý do
   [A4.3 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a43-cài-kubeadm-kubelet-kubectl-và-crictl) ghim
   cả ba gói ở đúng một chuỗi version rồi `apt-mark hold`.
2. **v1.36.0: không. v1.34: được.** Bài cho phép lệch *một* phiên bản minor giữa kubelet và
   control plane, nhưng nêu rõ **phiên bản kubelet không bao giờ được vượt quá phiên bản của
   API server**. Ví dụ của chính bài: kubelet 1.7.0 chạy được với API server 1.8.0, chiều ngược
   lại thì không. Trực giác "cứ mới hơn là an toàn" sai ở đây vì quan hệ này **bất đối xứng**.
3. `swapoff -a` chỉ **tắt swap tạm thời**; sau reboot swap bật lại, và **hành vi mặc định của
   kubelet là không khởi động nếu phát hiện swap trên node**. Bài yêu cầu đảm bảo swap bị vô
   hiệu hóa trong các file cấu hình như `/etc/fstab` để thay đổi được duy trì qua các lần khởi
   động lại — đúng việc mà A4.1 làm bằng cách comment dòng swap trong `/etc/fstab`, và gate của
   nó kiểm tra `swapon --show` rỗng **sau khi đã reboot**.
4. **Tiến trình kubelet sẽ thất bại.** Bài đặt cảnh báo riêng cho việc này: cgroup driver của
   container runtime và của kubelet *bắt buộc* phải khớp nhau. Đây là thuộc tính quản lý cgroup
   trên máy Linux, không phải chuyện đủ hay thiếu tài nguyên — nên RAM/CPU còn dư không cứu được.
5. Nó vừa **quét qua một danh sách các endpoint đã biết** (các Unix domain socket trong bảng
   *Các container runtime trên Linux*) để tự phát hiện runtime. Có **hai** tình huống làm nó
   báo lỗi: **phát hiện nhiều hơn một** runtime, hoặc **không phát hiện được cái nào**. Cả hai
   đều được xử lý bằng cách chỉ định `--cri-socket`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

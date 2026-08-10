# Thay đổi package repository của Kubernetes (Changing The Kubernetes Package Repository)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/>

Trang này giải thích cách bật package repository (kho gói phần mềm) cho phiên bản minor
Kubernetes mong muốn khi nâng cấp một cluster. Việc này chỉ cần thiết đối với người dùng các
package repository do cộng đồng quản lý, được lưu trữ tại `pkgs.k8s.io`. Khác với các package
repository cũ (legacy), các package repository do cộng đồng quản lý được tổ chức theo cách mỗi
phiên bản minor của Kubernetes có một package repository riêng.

> **Ghi chú:**
>
> Hướng dẫn này chỉ bao gồm một phần của quy trình nâng cấp Kubernetes. Vui lòng xem
> [hướng dẫn nâng cấp](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
> để biết thêm thông tin về việc nâng cấp cluster Kubernetes.

> **Ghi chú:**
>
> Bước này chỉ cần thiết khi nâng cấp cluster lên một phiên bản **minor** khác. Nếu bạn nâng cấp
> lên một phiên bản patch khác trong cùng phiên bản minor (ví dụ: v1.36.5 lên v1.36.7), bạn không
> cần làm theo hướng dẫn này. Tuy nhiên, nếu bạn vẫn đang dùng các package repository cũ, bạn sẽ
> cần chuyển sang các package repository mới do cộng đồng quản lý trước khi nâng cấp (xem mục
> tiếp theo để biết chi tiết cách thực hiện).

## Trước khi bạn bắt đầu (Before you begin)

Tài liệu này giả định rằng bạn đã đang sử dụng các package repository do cộng đồng quản lý
(`pkgs.k8s.io`). Nếu chưa, bạn được khuyến nghị mạnh mẽ nên chuyển sang các package repository
do cộng đồng quản lý theo mô tả trong
[thông báo chính thức](https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/).

> **Ghi chú:** Các package repository cũ (`apt.kubernetes.io` và `yum.kubernetes.io`) đã
> [ngưng sử dụng (deprecated) và bị đóng băng kể từ ngày 13/09/2023](https://kubernetes.io/blog/2023/08/31/legacy-package-repository-deprecation/).
> **Việc sử dụng [các package repository mới được lưu trữ tại `pkgs.k8s.io`](https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/)
> được khuyến nghị mạnh mẽ và là bắt buộc để cài đặt các phiên bản Kubernetes phát hành sau ngày
> 13/09/2023.** Các repository cũ đã ngưng sử dụng, cùng với nội dung của chúng, có thể bị gỡ bỏ
> bất cứ lúc nào trong tương lai mà không cần thông báo trước. Các package repository mới cung cấp
> bản tải xuống cho các phiên bản Kubernetes bắt đầu từ v1.24.0.

### Kiểm tra xem bạn có đang dùng các package repository của Kubernetes hay không (Verifying if the Kubernetes package repositories are used)

Nếu bạn không chắc mình đang dùng các package repository do cộng đồng quản lý hay các package
repository cũ, hãy thực hiện các bước sau để kiểm tra:

#### Ubuntu, Debian hoặc HypriotOS

In nội dung của file định nghĩa `apt` repository của Kubernetes:

```shell
# Trên hệ thống của bạn, file cấu hình này có thể có tên khác
pager /etc/apt/sources.list.d/kubernetes.list
```

Nếu bạn thấy một dòng tương tự như:

```
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /
```

**Bạn đang dùng các package repository của Kubernetes và hướng dẫn này áp dụng cho bạn.**
Nếu không, bạn được khuyến nghị mạnh mẽ nên chuyển sang các package repository của Kubernetes
theo mô tả trong
[thông báo chính thức](https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/).

#### CentOS, RHEL hoặc Fedora

In nội dung của file định nghĩa `yum` repository của Kubernetes:

```shell
# Trên hệ thống của bạn, file cấu hình này có thể có tên khác
cat /etc/yum.repos.d/kubernetes.repo
```

Nếu bạn thấy một `baseurl` tương tự như `baseurl` trong kết quả đầu ra bên dưới:

```
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.35/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.35/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl
```

**Bạn đang dùng các package repository của Kubernetes và hướng dẫn này áp dụng cho bạn.**
Nếu không, bạn được khuyến nghị mạnh mẽ nên chuyển sang các package repository của Kubernetes
theo mô tả trong
[thông báo chính thức](https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/).

#### openSUSE hoặc SLES

In nội dung của file định nghĩa `zypper` repository của Kubernetes:

```shell
# Trên hệ thống của bạn, file cấu hình này có thể có tên khác
cat /etc/zypp/repos.d/kubernetes.repo
```

Nếu bạn thấy một `baseurl` tương tự như `baseurl` trong kết quả đầu ra bên dưới:

```
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.35/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.35/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl
```

**Bạn đang dùng các package repository của Kubernetes và hướng dẫn này áp dụng cho bạn.**
Nếu không, bạn được khuyến nghị mạnh mẽ nên chuyển sang các package repository của Kubernetes
theo mô tả trong
[thông báo chính thức](https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/).

> **Ghi chú:**
>
> URL dùng cho các package repository của Kubernetes không giới hạn ở `pkgs.k8s.io`,
> nó cũng có thể là một trong các URL sau:
>
> - `pkgs.k8s.io`
> - `pkgs.kubernetes.io`
> - `packages.kubernetes.io`

## Chuyển sang một package repository khác của Kubernetes (Switching to another Kubernetes package repository)

Bước này cần được thực hiện khi nâng cấp từ một phiên bản minor Kubernetes này sang phiên bản
minor khác, để có quyền truy cập vào các gói của phiên bản minor Kubernetes mong muốn.

#### Ubuntu, Debian hoặc HypriotOS

1. Mở file định nghĩa `apt` repository của Kubernetes bằng trình soạn thảo văn bản tùy chọn:

   ```shell
   nano /etc/apt/sources.list.d/kubernetes.list
   ```

   Bạn sẽ thấy một dòng duy nhất chứa URL với phiên bản minor Kubernetes hiện tại của bạn.
   Ví dụ, nếu bạn đang dùng v1.35, bạn sẽ thấy như sau:

   ```
   deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /
   ```

1. Đổi phiên bản trong URL thành **phiên bản minor khả dụng kế tiếp**, ví dụ:

   ```
   deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /
   ```

1. Lưu file và thoát trình soạn thảo. Tiếp tục làm theo hướng dẫn nâng cấp tương ứng.

#### CentOS, RHEL hoặc Fedora

1. Mở file định nghĩa `yum` repository của Kubernetes bằng trình soạn thảo văn bản tùy chọn:

   ```shell
   nano /etc/yum.repos.d/kubernetes.repo
   ```

   Bạn sẽ thấy một file có hai URL chứa phiên bản minor Kubernetes hiện tại của bạn.
   Ví dụ, nếu bạn đang dùng v1.35, bạn sẽ thấy như sau:

   ```
   [kubernetes]
   name=Kubernetes
   baseurl=https://pkgs.k8s.io/core:/stable:/v1.35/rpm/
   enabled=1
   gpgcheck=1
   gpgkey=https://pkgs.k8s.io/core:/stable:/v1.35/rpm/repodata/repomd.xml.key
   exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
   ```

1. Đổi phiên bản trong các URL này thành **phiên bản minor khả dụng kế tiếp**, ví dụ:

   ```
   [kubernetes]
   name=Kubernetes
   baseurl=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/
   enabled=1
   gpgcheck=1
   gpgkey=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/repodata/repomd.xml.key
   exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
   ```

1. Lưu file và thoát trình soạn thảo. Tiếp tục làm theo hướng dẫn nâng cấp tương ứng.

## Tiếp theo (What's next)

* Xem cách [Nâng cấp node Linux](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-linux-nodes/).
* Xem cách [Nâng cấp node Windows](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-windows-nodes/).

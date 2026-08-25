# Thay đổi package repository của Kubernetes (Changing The Kubernetes Package Repository)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Checkpoint tiếp nối — nhánh `/docs/tasks/`](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 17 — Nâng cấp cluster](00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster), bài 4/5 · Kiểm chứng trên
cluster lab: in file định nghĩa repository trên `k8s-master` và xác định đúng minor version mà
cluster đang lấy gói từ đó.

Bài này là **một bước bên trong** quy trình của bài [221](221-kubeadm-upgrade-vi.md), không phải
một quy trình độc lập. Nó chỉ áp dụng cho người dùng repository cộng đồng `pkgs.k8s.io` — tức là
cả cluster lab của bạn, vì [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) cài kubeadm từ đó.

Phiên bản trong ví dụ là của trang gốc; minor version mà cluster lab đang khóa nằm ở [bảng A1.3 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa).

**Phải hiểu ở lần đọc này:**

- Điểm cốt lõi: **mỗi minor version Kubernetes có một repository riêng**. Đây là khác biệt với
  repository cũ, và là lý do bước này tồn tại.
- **Chỉ cần khi nâng minor** (1.n → 1.n+1). Nâng patch trong cùng minor (ví dụ v1.36.5 →
  v1.36.7) thì không phải đổi gì.
- Bỏ qua bước này khi nâng minor sẽ dẫn tới hiện tượng rất dễ hiểu nhầm: package manager báo
  "đã mới nhất" trong khi thật ra nó chỉ nhìn thấy repository của minor cũ.
- Cách kiểm tra mình đang dùng repository nào: đọc `/etc/apt/sources.list.d/kubernetes.list`
  (Debian/Ubuntu) hoặc `/etc/yum.repos.d/kubernetes.repo`, tìm chuỗi `core:/stable:/v1.NN`.
- Repository cũ (`apt.kubernetes.io`, `yum.kubernetes.io`) đã **đóng băng từ 13/09/2023** và có
  thể bị xóa bất cứ lúc nào; bắt buộc chuyển sang `pkgs.k8s.io` trước khi nâng cấp.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Cú pháp cho CentOS/RHEL/Fedora và openSUSE/SLES | cluster lab dùng Ubuntu theo Lab 00 | khi vận hành cluster trên distro khác |
| Danh sách tên miền thay thế (`pkgs.kubernetes.io`, `packages.kubernetes.io`) | chỉ là bí danh của cùng một hạ tầng | không cần cho lần đọc này |

---

Trang này giải thích cách bật package repository (kho gói phần mềm) cho phiên bản minor
Kubernetes mong muốn khi nâng cấp một cluster. Việc này chỉ cần thiết đối với người dùng các
package repository do cộng đồng quản lý, được lưu trữ tại `pkgs.k8s.io`. Khác với các package
repository cũ (legacy), các package repository do cộng đồng quản lý được tổ chức theo cách mỗi
phiên bản minor của Kubernetes có một package repository riêng.

> **Ghi chú:**
>
> Hướng dẫn này chỉ bao gồm một phần của quy trình nâng cấp Kubernetes. Vui lòng xem
> [hướng dẫn nâng cấp](221-kubeadm-upgrade-vi.md)
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

### Kiểm tra xem bạn có đang dùng các package repository của Kubernetes hay không (Verifying if the Kubernetes package repositories are used) {#verifying-if-the-kubernetes-package-repositories-are-used}

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

* Xem cách [Nâng cấp node Linux](222-upgrading-linux-nodes-vi.md).
* Xem cách [Nâng cấp node Windows](223-upgrading-windows-nodes-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 17:

1. Trên `k8s-master`, bạn mở `/etc/apt/sources.list.d/kubernetes.list` và thấy một dòng chứa
   `core:/stable:/v1.35`. Bạn muốn nâng cluster lên minor kế tiếp. Phải sửa gì, và nếu **không**
   sửa thì `apt-get install kubeadm=<phiên bản mới>` sẽ báo gì?
2. **Câu bẫy.** Bạn nâng cluster từ một bản patch lên bản patch khác trong cùng minor. Có phải
   làm bước của bài này không? Vì sao?
3. Vì sao repository cộng đồng lại tách riêng theo từng minor version, trong khi repository cũ
   thì không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Phải **đổi `v1.35` trong URL thành minor khả dụng kế tiếp** (`v1.36`), lưu file, rồi tiếp tục
   quy trình nâng cấp. Nếu không sửa, repository đang bật **chỉ chứa các gói của minor cũ**, nên
   apt sẽ báo không tìm thấy phiên bản bạn yêu cầu — hoặc tệ hơn về mặt hiểu nhầm, một lệnh
   `apt-get upgrade` chung chung sẽ báo hệ thống đã ở bản mới nhất, trong khi thực ra nó chỉ đang
   nhìn vào kho của minor cũ.
2. **Không cần.** Bài nói rõ bước này chỉ cần khi nâng lên một **minor version khác**. Trong cùng
   một minor, mọi bản patch nằm chung một repository nên không phải sửa gì. Ngoại lệ duy nhất:
   nếu bạn vẫn đang dùng repository cũ đã ngưng sử dụng thì phải chuyển sang `pkgs.k8s.io`
   trước, bất kể nâng patch hay minor.
3. Vì việc tách kho theo minor biến **lựa chọn phiên bản thành một hành động có chủ đích**: máy
   chỉ nhìn thấy các gói thuộc đúng minor đang bật, nên không thể vô tình nâng vọt sang minor
   khác và phá vỡ version skew. Kho cũ gộp chung mọi phiên bản nên không có lớp bảo vệ đó.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi sang bài
[195](195-cluster-upgrade-vi.md) — bài cuối của [Giai đoạn 17 — Nâng cấp cluster](00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster).

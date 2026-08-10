# Nâng cấp cluster kubeadm (Upgrading kubeadm clusters)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/>

Trang này giải thích cách nâng cấp một cluster Kubernetes được tạo bằng kubeadm từ phiên bản
1.35.x lên phiên bản 1.36.x, và từ phiên bản 1.36.x lên 1.36.y (với `y > x`). Việc bỏ qua
(skip) các phiên bản MINOR khi nâng cấp không được hỗ trợ. Để biết thêm chi tiết, vui lòng xem
[Chính sách lệch phiên bản (Version Skew Policy)](https://kubernetes.io/releases/version-skew-policy/).

Để xem thông tin về việc nâng cấp các cluster được tạo bằng những phiên bản kubeadm cũ hơn,
vui lòng tham khảo các trang sau đây thay thế:

- [Nâng cấp cluster kubeadm từ 1.34 lên 1.35](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [Nâng cấp cluster kubeadm từ 1.33 lên 1.34](https://v1-34.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [Nâng cấp cluster kubeadm từ 1.32 lên 1.33](https://v1-33.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [Nâng cấp cluster kubeadm từ 1.31 lên 1.32](https://v1-32.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

Dự án Kubernetes khuyến nghị bạn nâng cấp kịp thời lên các bản vá (patch release) mới nhất,
và đảm bảo rằng bạn đang chạy một bản phát hành minor của Kubernetes còn được hỗ trợ.
Tuân theo khuyến nghị này giúp bạn duy trì được sự an toàn.

Quy trình nâng cấp ở mức tổng quan như sau:

1. Nâng cấp node control plane chính (primary).
1. Nâng cấp các node control plane còn lại.
1. Nâng cấp các node worker.

## Trước khi bạn bắt đầu (Before you begin)

- Hãy chắc chắn rằng bạn đã đọc kỹ [ghi chú phát hành (release notes)](https://git.k8s.io/kubernetes/CHANGELOG).
- Cluster nên sử dụng control plane và etcd dạng static Pod, hoặc etcd bên ngoài (external etcd).
- Nhớ sao lưu (back up) mọi thành phần quan trọng, chẳng hạn như trạng thái ở mức ứng dụng được
  lưu trong cơ sở dữ liệu. `kubeadm upgrade` không đụng đến workload của bạn, chỉ tác động đến
  các thành phần nội bộ của Kubernetes, nhưng sao lưu luôn là một thực hành tốt (best practice).
- [Swap phải được tắt](https://serverfault.com/questions/684771/best-way-to-disable-swap-in-linux).

### Thông tin bổ sung (Additional information)

- Các hướng dẫn bên dưới nêu rõ thời điểm cần drain từng node trong quá trình nâng cấp.
  Nếu bạn đang thực hiện nâng cấp phiên bản **minor** cho bất kỳ kubelet nào, bạn **bắt buộc**
  phải drain node (hoặc các node) đang được nâng cấp trước tiên. Đối với các node control plane,
  chúng có thể đang chạy các Pod CoreDNS hoặc những workload quan trọng khác. Để biết thêm thông
  tin, xem [Drain node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/).
- Dự án Kubernetes khuyến nghị bạn dùng phiên bản kubelet và kubeadm trùng nhau.
  Thay vào đó, bạn cũng có thể dùng một phiên bản kubelet cũ hơn kubeadm, miễn là nó nằm trong
  phạm vi các phiên bản được hỗ trợ.
  Để biết thêm chi tiết, vui lòng xem [Độ lệch phiên bản của kubeadm so với kubelet](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#kubeadm-s-skew-against-the-kubelet).
- Tất cả các container sẽ được khởi động lại sau khi nâng cấp, vì giá trị hash của container
  spec đã thay đổi.
- Để xác nhận rằng dịch vụ kubelet đã khởi động lại thành công sau khi kubelet được nâng cấp,
  bạn có thể chạy `systemctl status kubelet` hoặc xem log của dịch vụ bằng `journalctl -xeu kubelet`.
- `kubeadm upgrade` hỗ trợ cờ `--config` với [kiểu API `UpgradeConfiguration`](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4)
  có thể được dùng để cấu hình quá trình nâng cấp.
- `kubeadm upgrade` không hỗ trợ cấu hình lại (reconfiguration) một cluster đang tồn tại.
  Thay vào đó, hãy làm theo các bước trong
  [Cấu hình lại cluster kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-reconfigure).

### Những điểm cần cân nhắc khi nâng cấp etcd (Considerations when upgrading etcd)

Vì static Pod `kube-apiserver` luôn luôn chạy (kể cả khi bạn đã drain node), nên khi bạn thực
hiện một lần nâng cấp kubeadm có bao gồm nâng cấp etcd, các request đang xử lý dở (in-flight)
tới server sẽ bị treo trong lúc static Pod etcd mới khởi động lại. Một cách khắc phục
(workaround) là chủ động dừng tiến trình `kube-apiserver` vài giây trước khi bắt đầu chạy lệnh
`kubeadm upgrade apply`. Điều này cho phép hoàn tất các request đang xử lý dở và đóng các kết
nối hiện có, đồng thời giảm thiểu hậu quả của khoảng thời gian etcd ngừng hoạt động (downtime).
Có thể thực hiện như sau trên các node control plane:

```shell
killall -s SIGTERM kube-apiserver # kích hoạt việc tắt kube-apiserver một cách êm thấm (graceful)
sleep 20 # chờ một chút để các request đang xử lý dở được hoàn tất
kubeadm upgrade ... # thực thi một lệnh kubeadm upgrade
```

## Thay đổi kho gói (Changing the package repository)

Nếu bạn đang dùng các kho gói do cộng đồng sở hữu (`pkgs.k8s.io`), bạn cần kích hoạt kho gói
cho bản phát hành minor Kubernetes mong muốn. Điều này được giải thích trong tài liệu
[Thay đổi kho gói Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/).

> **Ghi chú:** Các kho gói cũ (`apt.kubernetes.io` và `yum.kubernetes.io`) đã bị
> [ngưng sử dụng và đóng băng kể từ ngày 13-09-2023](https://kubernetes.io/blog/2023/08/31/legacy-package-repository-deprecation/).
> **Việc sử dụng [các kho gói mới được lưu trữ tại `pkgs.k8s.io`](https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/)
> được khuyến nghị mạnh mẽ và là bắt buộc để cài đặt các phiên bản Kubernetes phát hành sau ngày 13-09-2023.**
> Các kho cũ đã ngưng sử dụng, cùng nội dung của chúng, có thể bị xóa bất cứ lúc nào trong tương
> lai mà không cần thông báo trước. Các kho gói mới cung cấp bản tải về cho các phiên bản
> Kubernetes bắt đầu từ v1.24.0.

## Xác định phiên bản cần nâng cấp lên (Determine which version to upgrade to)

Tìm bản vá mới nhất của Kubernetes 1.36 bằng trình quản lý gói của hệ điều hành:

#### Ubuntu, Debian hoặc HypriotOS

```shell
# Tìm phiên bản 1.36 mới nhất trong danh sách.
# Nó sẽ có dạng 1.36.x-*, trong đó x là bản vá mới nhất.
sudo apt update
sudo apt-cache madison kubeadm
```

#### CentOS, RHEL hoặc Fedora

Với các hệ thống dùng DNF:
```shell
# Tìm phiên bản 1.36 mới nhất trong danh sách.
# Nó sẽ có dạng 1.36.x-*, trong đó x là bản vá mới nhất.
sudo yum list --showduplicates kubeadm --disableexcludes=kubernetes
```
Với các hệ thống dùng DNF5:
```shell
# Tìm phiên bản 1.36 mới nhất trong danh sách.
# Nó sẽ có dạng 1.36.x-*, trong đó x là bản vá mới nhất.
sudo yum list --showduplicates kubeadm --setopt=disable_excludes=kubernetes
```

Nếu bạn không thấy phiên bản mà bạn dự định nâng cấp lên, hãy
[kiểm tra xem các kho gói Kubernetes có đang được sử dụng hay không](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/#verifying-if-the-kubernetes-package-repositories-are-used).

## Nâng cấp các node control plane (Upgrading control plane nodes)

Thủ tục nâng cấp trên các node control plane nên được thực hiện từng node một tại một thời điểm.
Chọn node control plane mà bạn muốn nâng cấp trước tiên. Node đó phải có file
`/etc/kubernetes/admin.conf`.

### Gọi "kubeadm upgrade" (Call "kubeadm upgrade")

**Đối với node control plane đầu tiên**

1. Nâng cấp kubeadm:

   #### Ubuntu, Debian hoặc HypriotOS

   ```shell
   # thay x trong 1.36.x-* bằng phiên bản vá mới nhất
   sudo apt-mark unhold kubeadm && \
   sudo apt-get update && sudo apt-get install -y kubeadm='1.36.x-*' && \
   sudo apt-mark hold kubeadm
   ```

   #### CentOS, RHEL hoặc Fedora

   Với các hệ thống dùng DNF:
   ```shell
   # thay x trong 1.36.x-* bằng phiên bản vá mới nhất
   sudo yum install -y kubeadm-'1.36.x-*' --disableexcludes=kubernetes
   ```
   Với các hệ thống dùng DNF5:
   ```shell
   # thay x trong 1.36.x-* bằng phiên bản vá mới nhất
   sudo yum install -y kubeadm-'1.36.x-*' --setopt=disable_excludes=kubernetes
   ```

1. Kiểm tra rằng việc tải về đã thành công và đúng phiên bản mong đợi:

   ```shell
   kubeadm version
   ```

1. Kiểm tra kế hoạch nâng cấp (upgrade plan):

   ```shell
   sudo kubeadm upgrade plan
   ```

   Lệnh này kiểm tra xem cluster của bạn có thể được nâng cấp hay không, và lấy về các phiên
   bản mà bạn có thể nâng cấp lên. Nó cũng hiển thị một bảng chứa trạng thái phiên bản của các
   cấu hình thành phần (component config).

   > **Ghi chú:**
   >
   > `kubeadm upgrade` cũng tự động gia hạn (renew) các certificate mà nó quản lý trên node này.
   > Để từ chối việc gia hạn certificate, có thể dùng cờ `--certificate-renewal=false`.
   > Để biết thêm thông tin, xem [hướng dẫn quản lý certificate](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs).

1. Chọn một phiên bản để nâng cấp lên, và chạy lệnh tương ứng. Ví dụ:

   ```shell
   # thay x bằng phiên bản vá mà bạn đã chọn cho lần nâng cấp này
   sudo kubeadm upgrade apply v1.36.x
   ```

   Khi lệnh hoàn tất, bạn sẽ thấy:

   ```
   [upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.36.x". Enjoy!

   [upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
   ```

   > **Ghi chú:**
   >
   > Với các phiên bản trước v1.28, kubeadm mặc định dùng chế độ nâng cấp các addon (bao gồm
   > CoreDNS và kube-proxy) ngay lập tức trong khi chạy `kubeadm upgrade apply`, bất kể còn
   > các instance control plane khác chưa được nâng cấp hay không. Điều này có thể gây ra các
   > vấn đề về tính tương thích. Kể từ v1.28, kubeadm mặc định dùng chế độ kiểm tra xem tất cả
   > các instance control plane đã được nâng cấp hay chưa trước khi bắt đầu nâng cấp các addon.
   > Bạn phải thực hiện nâng cấp các instance control plane một cách tuần tự, hoặc ít nhất đảm
   > bảo rằng việc nâng cấp instance control plane cuối cùng không được bắt đầu cho tới khi tất
   > cả các instance control plane khác đã được nâng cấp hoàn toàn; khi đó việc nâng cấp addon
   > sẽ được thực hiện sau khi instance control plane cuối cùng được nâng cấp.

1. Nâng cấp thủ công plugin CNI của bạn.

   Nhà cung cấp Container Network Interface (CNI) của bạn có thể có hướng dẫn nâng cấp riêng
   cần tuân theo. Hãy kiểm tra trang [addons](https://kubernetes.io/docs/concepts/cluster-administration/addons/)
   để tìm nhà cung cấp CNI của bạn và xem có cần thêm bước nâng cấp bổ sung nào hay không.

   Bước này không bắt buộc trên các node control plane còn lại nếu nhà cung cấp CNI chạy dưới
   dạng DaemonSet.

**Đối với các node control plane còn lại**

Làm giống như node control plane đầu tiên nhưng dùng:

```shell
sudo kubeadm upgrade node
```

thay vì:

```shell
sudo kubeadm upgrade apply
```

Ngoài ra, việc gọi `kubeadm upgrade plan` và nâng cấp plugin CNI cũng không còn cần thiết nữa.

### Drain node (Drain the node)

Chuẩn bị node cho việc bảo trì bằng cách đánh dấu node là không thể lập lịch (unschedulable)
và trục xuất (evict) các workload:

```shell
# thay <node-to-drain> bằng tên của node mà bạn đang drain
kubectl drain <node-to-drain> --ignore-daemonsets
```

### Nâng cấp kubelet và kubectl (Upgrade kubelet and kubectl)

> **Ghi chú:**
>
> Trên các node Linux, kubelet mặc định chỉ hỗ trợ cgroups v2.
> Với Kubernetes 1.36, tùy chọn cấu hình kubelet `FailCgroupV1` được đặt là `true` theo mặc định.
>
> Để tìm hiểu thêm, tham khảo [tài liệu về việc ngưng hỗ trợ cgroup v1 của Kubernetes](https://kubernetes.io/docs/concepts/architecture/cgroups/#deprecation-of-cgroup-v1).

1. Nâng cấp kubelet và kubectl:

   #### Ubuntu, Debian hoặc HypriotOS

   ```shell
   # thay x trong 1.36.x-* bằng phiên bản vá mới nhất
   sudo apt-mark unhold kubelet kubectl && \
   sudo apt-get update && sudo apt-get install -y kubelet='1.36.x-*' kubectl='1.36.x-*' && \
   sudo apt-mark hold kubelet kubectl
   ```

   #### CentOS, RHEL hoặc Fedora

   Với các hệ thống dùng DNF:
   ```shell
   # thay x trong 1.36.x-* bằng phiên bản vá mới nhất
   sudo yum install -y kubelet-'1.36.x-*' kubectl-'1.36.x-*' --disableexcludes=kubernetes
   ```
   Với các hệ thống dùng DNF5:
   ```shell
   # thay x trong 1.36.x-* bằng phiên bản vá mới nhất
   sudo yum install -y kubelet-'1.36.x-*' kubectl-'1.36.x-*' --setopt=disable_excludes=kubernetes
   ```

1. Khởi động lại kubelet:

   ```shell
   sudo systemctl daemon-reload
   sudo systemctl restart kubelet
   ```

### Uncordon node (Uncordon the node)

Đưa node trở lại hoạt động bằng cách đánh dấu node là có thể lập lịch (schedulable):

```shell
# thay <node-to-uncordon> bằng tên node của bạn
kubectl uncordon <node-to-uncordon>
```

## Nâng cấp các node worker (Upgrade worker nodes)

Thủ tục nâng cấp trên các node worker nên được thực hiện từng node một, hoặc vài node tại một
thời điểm, mà không làm ảnh hưởng đến năng lực tối thiểu cần thiết để chạy các workload của bạn.

Các trang sau đây hướng dẫn cách nâng cấp node worker Linux và Windows:

* [Nâng cấp node Linux](222-upgrading-linux-nodes-vi.md)
* [Nâng cấp node Windows](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-windows-nodes/)

## Kiểm tra trạng thái của cluster (Verify the status of the cluster)

Sau khi kubelet đã được nâng cấp trên tất cả các node, hãy xác nhận rằng tất cả các node đều
khả dụng trở lại bằng cách chạy lệnh sau từ bất cứ nơi nào mà kubectl có thể truy cập cluster:

```shell
kubectl get nodes
```

Cột `STATUS` phải hiển thị `Ready` cho tất cả các node của bạn, và số phiên bản phải được cập nhật.

## Khôi phục từ trạng thái lỗi (Recovering from a failure state)

Nếu `kubeadm upgrade` thất bại và không tự rollback, ví dụ do máy bị tắt đột ngột trong khi
thực thi, bạn có thể chạy lại `kubeadm upgrade`. Lệnh này có tính lũy đẳng (idempotent) và cuối
cùng sẽ đảm bảo rằng trạng thái thực tế trùng với trạng thái mong muốn mà bạn khai báo.

Để khôi phục từ một trạng thái hỏng, bạn cũng có thể chạy `sudo kubeadm upgrade apply --force`
mà không thay đổi phiên bản cluster đang chạy.

Trong quá trình nâng cấp, kubeadm ghi các thư mục sao lưu sau vào `/etc/kubernetes/tmp`:

- `kubeadm-backup-etcd-<date>-<time>`
- `kubeadm-backup-manifests-<date>-<time>`

`kubeadm-backup-etcd` chứa bản sao lưu dữ liệu của thành viên etcd cục bộ (local etcd member)
trên node control plane này. Trong trường hợp nâng cấp etcd thất bại và cơ chế rollback tự động
không hoạt động, nội dung của thư mục này có thể được khôi phục thủ công vào `/var/lib/etcd`.
Trong trường hợp dùng etcd bên ngoài (external etcd), thư mục sao lưu này sẽ rỗng.

`kubeadm-backup-manifests` chứa bản sao lưu các file manifest static Pod của node control plane
này. Trong trường hợp nâng cấp thất bại và cơ chế rollback tự động không hoạt động, nội dung
của thư mục này có thể được khôi phục thủ công vào `/etc/kubernetes/manifests`. Nếu vì lý do
nào đó mà file manifest trước và sau nâng cấp của một thành phần không có gì khác biệt, thì
file sao lưu cho thành phần đó sẽ không được ghi ra.

> **Ghi chú:**
>
> Sau khi nâng cấp cluster bằng kubeadm, thư mục sao lưu `/etc/kubernetes/tmp` vẫn còn đó và
> các file sao lưu này cần được dọn dẹp thủ công.

## Cách thức hoạt động (How it works)

`kubeadm upgrade apply` thực hiện những việc sau:

- Kiểm tra rằng cluster của bạn đang ở trạng thái có thể nâng cấp:
  - API server có thể truy cập được
  - Tất cả các node đang ở trạng thái `Ready`
  - Control plane khỏe mạnh (healthy)
- Thực thi các chính sách lệch phiên bản (version skew policies).
- Đảm bảo các image của control plane đã có sẵn hoặc có thể kéo (pull) về máy.
- Sinh ra các bản thay thế và/hoặc sử dụng các giá trị ghi đè do người dùng cung cấp nếu các
  cấu hình thành phần (component config) yêu cầu nâng cấp phiên bản.
- Nâng cấp các thành phần control plane, hoặc rollback nếu bất kỳ thành phần nào không khởi
  động được.
- Áp dụng các manifest `CoreDNS` và `kube-proxy` mới và đảm bảo rằng tất cả các quy tắc RBAC
  cần thiết được tạo ra.
- Tạo file certificate và khóa (key) mới cho API server, đồng thời sao lưu các file cũ nếu
  chúng sắp hết hạn trong vòng 180 ngày.

`kubeadm upgrade node` thực hiện những việc sau trên các node control plane còn lại:

- Lấy `ClusterConfiguration` của kubeadm từ cluster.
- Tùy chọn sao lưu certificate của kube-apiserver.
- Nâng cấp các manifest static Pod của các thành phần control plane.
- Nâng cấp cấu hình kubelet cho node này.

`kubeadm upgrade node` thực hiện những việc sau trên các node worker:

- Lấy `ClusterConfiguration` của kubeadm từ cluster.
- Nâng cấp cấu hình kubelet cho node này.

# Nâng cấp node Linux (Upgrading Linux nodes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-linux-nodes/>

Trang này giải thích cách nâng cấp các node worker Linux được tạo bằng kubeadm.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có quyền truy cập shell vào tất cả các node, và công cụ dòng lệnh kubectl phải được
cấu hình để giao tiếp với cluster của bạn. Khuyến nghị chạy hướng dẫn này trên một cluster có
ít nhất hai node không đóng vai trò control plane. Để kiểm tra phiên bản, nhập `kubectl version`.

* Hãy làm quen với [quy trình nâng cấp phần còn lại của cluster kubeadm](221-kubeadm-upgrade-vi.md).
  Bạn nên nâng cấp các node control plane trước khi nâng cấp các node worker Linux.

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

## Nâng cấp các node worker (Upgrading worker nodes)

### Nâng cấp kubeadm (Upgrade kubeadm)

Nâng cấp kubeadm:

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

### Gọi "kubeadm upgrade" (Call "kubeadm upgrade")

Đối với các node worker, lệnh này nâng cấp cấu hình kubelet cục bộ:

```shell
sudo kubeadm upgrade node
```

### Drain node (Drain the node)

Chuẩn bị node cho việc bảo trì bằng cách đánh dấu node là không thể lập lịch (unschedulable)
và trục xuất (evict) các workload:

```shell
# chạy lệnh này trên một node control plane
# thay <node-to-drain> bằng tên của node mà bạn đang drain
kubectl drain <node-to-drain> --ignore-daemonsets
```

### Nâng cấp kubelet và kubectl (Upgrade kubelet and kubectl)

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
# chạy lệnh này trên một node control plane
# thay <node-to-uncordon> bằng tên node của bạn
kubectl uncordon <node-to-uncordon>
```

## Tiếp theo (What's next)

* Xem cách [Nâng cấp node Windows](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-windows-nodes/).

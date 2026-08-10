# Nâng cấp các node Windows (Upgrading Windows nodes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-windows-nodes/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.18 [beta]`

Trang này giải thích cách nâng cấp một node Windows được tạo bằng kubeadm.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có quyền truy cập shell vào tất cả các node, và công cụ dòng lệnh kubectl phải được
cấu hình để giao tiếp với cluster của bạn. Khuyến nghị chạy hướng dẫn này trên một cluster có
ít nhất hai node không đóng vai trò máy chủ control plane.

Máy chủ Kubernetes của bạn phải ở phiên bản bằng hoặc mới hơn 1.17. Để kiểm tra phiên bản,
hãy chạy `kubectl version`.

* Hãy làm quen với
  [quy trình nâng cấp phần còn lại của cluster kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade).
  Bạn nên nâng cấp các node control plane trước khi nâng cấp các node Windows.

## Nâng cấp các node worker (Upgrading worker nodes)

### Nâng cấp kubeadm (Upgrade kubeadm)

1. Từ node Windows, nâng cấp kubeadm:

   ```powershell
   # thay 1.36.0 bằng phiên bản mà bạn mong muốn
   curl.exe -Lo <path-to-kubeadm.exe>  "https://dl.k8s.io/v1.36.0/bin/windows/amd64/kubeadm.exe"
   ```

### Drain node (Drain the node)

1. Từ một máy có quyền truy cập tới API Kubernetes, chuẩn bị node cho việc bảo trì bằng cách
   đánh dấu node là không thể lập lịch (unschedulable) và trục xuất (evict) các workload:

   ```shell
   # thay <node-to-drain> bằng tên node mà bạn đang drain
   kubectl drain <node-to-drain> --ignore-daemonsets
   ```

   Bạn sẽ thấy đầu ra tương tự như sau:

   ```
   node/ip-172-31-85-18 cordoned
   node/ip-172-31-85-18 drained
   ```

### Nâng cấp cấu hình kubelet (Upgrade the kubelet configuration)

1. Từ node Windows, gọi lệnh sau để đồng bộ cấu hình kubelet mới:

   ```powershell
   kubeadm upgrade node
   ```

### Nâng cấp kubelet và kube-proxy (Upgrade kubelet and kube-proxy)

1. Từ node Windows, nâng cấp và khởi động lại kubelet:

   ```powershell
   stop-service kubelet
   curl.exe -Lo <path-to-kubelet.exe> "https://dl.k8s.io/v1.36.0/bin/windows/amd64/kubelet.exe"
   restart-service kubelet
   ```

2. Từ node Windows, nâng cấp và khởi động lại kube-proxy.

   ```powershell
   stop-service kube-proxy
   curl.exe -Lo <path-to-kube-proxy.exe> "https://dl.k8s.io/v1.36.0/bin/windows/amd64/kube-proxy.exe"
   restart-service kube-proxy
   ```

> **Ghi chú:**
>
> Nếu bạn đang chạy kube-proxy trong một container HostProcess bên trong một Pod, thay vì chạy
> như một Windows Service, bạn có thể nâng cấp kube-proxy bằng cách apply phiên bản mới hơn của
> các manifest kube-proxy.

### Uncordon node (Uncordon the node)

1. Từ một máy có quyền truy cập tới API Kubernetes, đưa node hoạt động trở lại bằng cách đánh
   dấu node là có thể lập lịch (schedulable):

   ```shell
   # thay <node-to-drain> bằng tên node của bạn
   kubectl uncordon <node-to-drain>
   ```

## Tiếp theo (What's next)

* Xem cách [Nâng cấp các node Linux](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-linux-nodes/).

# Cấu hình cgroup driver (Configuring a cgroup driver)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/>

Trang này giải thích cách cấu hình cgroup driver của kubelet sao cho khớp với cgroup driver của
container runtime trong các cluster dựng bằng kubeadm.

## Trước khi bạn bắt đầu (Before you begin)

Bạn nên nắm được các
[yêu cầu về container runtime](00-container-runtimes-vi.md)
của Kubernetes.

## Cấu hình cgroup driver của container runtime (Configuring the container runtime cgroup driver)

Trang [Container runtimes](00-container-runtimes-vi.md)
giải thích rằng driver `systemd` được khuyến nghị cho các hệ thống dựng bằng kubeadm, thay vì
driver `cgroupfs` [mặc định](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1)
của kubelet, bởi vì kubeadm quản lý kubelet như một
[systemd service](04-kubelet-integration-vi.md).

Trang đó cũng cung cấp chi tiết về cách thiết lập một số container runtime khác nhau với driver
`systemd` theo mặc định.

## Cấu hình cgroup driver của kubelet (Configuring the kubelet cgroup driver) {#configuring-the-kubelet-cgroup-driver}

kubeadm cho phép bạn truyền một cấu trúc `KubeletConfiguration` trong lúc chạy `kubeadm init`.
`KubeletConfiguration` này có thể bao gồm trường `cgroupDriver`, trường này điều khiển cgroup
driver của kubelet.

> **Ghi chú:**
>
> Từ v1.22 trở đi, nếu người dùng không đặt trường `cgroupDriver` trong `KubeletConfiguration`,
> kubeadm sẽ mặc định nó là `systemd`.
>
> Trong Kubernetes v1.28, bạn có thể bật tính năng tự động phát hiện cgroup driver dưới dạng
> tính năng alpha. Xem
> [systemd cgroup driver](00-container-runtimes-vi.md#systemd-cgroup-driver)
> để biết thêm chi tiết.

Một ví dụ tối giản về việc cấu hình trường này một cách tường minh:

```yaml
# kubeadm-config.yaml
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta4
kubernetesVersion: v1.21.0
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
```

File cấu hình như vậy sau đó có thể được truyền cho lệnh kubeadm:

```shell
kubeadm init --config kubeadm-config.yaml
```

> **Ghi chú:**
>
> Kubeadm dùng cùng một `KubeletConfiguration` cho tất cả các node trong cluster.
> `KubeletConfiguration` được lưu trong một object
> [ConfigMap](108-configmap-vi.md)
> thuộc namespace `kube-system`.
>
> Việc thực thi các lệnh con `init`, `join` và `upgrade` sẽ khiến kubeadm ghi
> `KubeletConfiguration` ra một file tại `/var/lib/kubelet/config.yaml`
> và truyền nó cho kubelet của node cục bộ.
>
> Trên mỗi node, kubeadm phát hiện CRI socket và lưu thông tin chi tiết của nó vào file
> `/var/lib/kubelet/instance-config.yaml`. Khi thực thi các lệnh con `init`, `join` hoặc
> `upgrade`, kubeadm sẽ vá (patch) giá trị `containerRuntimeEndpoint` từ file cấu hình instance
> này vào `/var/lib/kubelet/config.yaml`.

## Sử dụng driver `cgroupfs` (Using the `cgroupfs` driver)

Để dùng `cgroupfs` và để ngăn `kubeadm upgrade` sửa đổi cgroup driver trong
`KubeletConfiguration` trên các hệ thống hiện có, bạn phải khai báo tường minh giá trị của nó.
Điều này áp dụng cho trường hợp bạn không muốn các phiên bản kubeadm trong tương lai áp dụng
driver `systemd` theo mặc định.

Xem mục "[Sửa ConfigMap của kubelet](#modify-the-kubelet-configmap)" bên dưới để biết chi tiết
về cách khai báo tường minh giá trị này.

Nếu bạn muốn cấu hình container runtime dùng driver `cgroupfs`, bạn phải tham khảo tài liệu của
container runtime mà bạn chọn.

## Chuyển sang driver `systemd` (Migrating to the `systemd` driver)

Để thay đổi cgroup driver của một cluster kubeadm hiện có từ `cgroupfs` sang `systemd` tại chỗ
(in-place), cần một quy trình tương tự như khi nâng cấp kubelet. Quy trình này phải bao gồm cả
hai bước được nêu dưới đây.

> **Ghi chú:**
>
> Một cách khác là thay các node cũ trong cluster bằng các node mới sử dụng driver `systemd`.
> Cách này chỉ yêu cầu thực hiện bước đầu tiên bên dưới trước khi join các node mới, đồng thời
> đảm bảo các workload có thể di chuyển an toàn sang các node mới trước khi xóa các node cũ.

### Sửa ConfigMap của kubelet (Modify the kubelet ConfigMap) {#modify-the-kubelet-configmap}

- Chạy `kubectl edit cm kubelet-config -n kube-system`.
- Sửa giá trị `cgroupDriver` hiện có hoặc thêm một trường mới trông như sau:

  ```yaml
  cgroupDriver: systemd
  ```

  Trường này phải nằm dưới mục `kubelet:` của ConfigMap.

### Cập nhật cgroup driver trên tất cả các node (Update the cgroup driver on all nodes)

Với từng node trong cluster:

- [Drain node](255-safely-drain-node-vi.md) bằng lệnh
  `kubectl drain <node-name> --ignore-daemonsets`
- Dừng kubelet bằng lệnh `systemctl stop kubelet`
- Dừng container runtime
- Sửa cgroup driver của container runtime thành `systemd`
- Đặt `cgroupDriver: systemd` trong `/var/lib/kubelet/config.yaml`
- Khởi động container runtime
- Khởi động kubelet bằng lệnh `systemctl start kubelet`
- [Uncordon node](255-safely-drain-node-vi.md) bằng
  lệnh `kubectl uncordon <node-name>`

Hãy thực hiện các bước này trên từng node một, để đảm bảo các workload có đủ thời gian được lập
lịch (schedule) sang các node khác.

Khi quy trình hoàn tất, hãy đảm bảo tất cả các node và workload đều khỏe mạnh.

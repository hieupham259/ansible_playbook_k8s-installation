# Pod tĩnh (Static Pods)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/pods/static-pods/>

_Static Pod_ (Pod tĩnh) được quản lý trực tiếp bởi kubelet daemon trên một node cụ thể,
mà không có API server quan sát chúng.
Khác với các Pod được quản lý bởi control plane (ví dụ như một Deployment),
kubelet theo dõi từng static Pod và khởi động lại nó nếu nó gặp sự cố.

Static Pod luôn được gắn với một kubelet trên một node cụ thể.

Công dụng chính của static Pod là để chạy một control plane tự lưu trữ (self-hosted):
nói cách khác, dùng kubelet để giám sát từng
[thành phần control plane](./15-components-vi.md) riêng lẻ.
Ví dụ, [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) dùng static Pod để chạy
`kube-apiserver`, `kube-controller-manager`, `kube-scheduler` và `etcd` trên các node control plane.

> **Ghi chú:**
> Nếu cluster của bạn chạy các thành phần control plane dưới dạng Pod, nhiều khả năng
> chúng là static Pod. Bạn có thể nhận ra các mirror Pod của chúng trong namespace
> `kube-system` thông qua annotation `kubernetes.io/config.mirror`.

## Mirror Pod (Mirror Pods) {#mirror-pods}

Kubelet tự động cố gắng tạo một mirror Pod (Pod phản chiếu)
trên Kubernetes API server cho mỗi static Pod.
Điều này có nghĩa là các Pod đang chạy trên một node sẽ được nhìn thấy trên API server,
nhưng không thể bị điều khiển từ đó.
Tên của Pod sẽ có hậu tố là hostname của node, ngăn cách bằng một dấu gạch nối ở đầu.

Kubelet lan truyền (propagate) các label từ static Pod sang mirror Pod. Bạn có thể
dùng các label đó như bình thường thông qua các selector.

Nếu bạn thử dùng `kubectl` để xóa mirror Pod khỏi API server,
kubelet _không_ xóa static Pod. Kubelet sẽ tạo lại
mirror Pod.

## Giới hạn (Limitations) {#limitations}

Spec của một static Pod không thể tham chiếu đến các đối tượng API khác,
chẳng hạn như ServiceAccount, ConfigMap hay Secret.

Static Pod không hỗ trợ [ephemeral container](./52-ephemeral-containers-vi.md).

## Static Pod so với DaemonSet (Static Pods vs DaemonSets) {#static-pods-vs-daemonsets}

Nếu bạn đang vận hành Kubernetes dạng cluster và dùng static Pod để chạy một Pod
trên mọi node, thì có lẽ bạn nên dùng
DaemonSet thay thế.

Static Pod không được control plane quản lý, vì vậy chúng không thể được triển khai
cuốn chiếu (rolled out), hoàn tác (rolled back) hay co giãn (scaled) bằng các cơ chế
tiêu chuẩn của Kubernetes. DaemonSet cung cấp các khả năng này và là cách tiếp cận
được khuyến nghị để chạy các workload ở cấp node.

Static Pod được kubelet khởi động trước khi API server sẵn sàng, điều này
khiến chúng phù hợp cho việc khởi tạo (bootstrap) các thành phần control plane.
DaemonSet yêu cầu một control plane đang chạy.

## Tiếp theo (What's next)

- Tìm hiểu cách [tạo static Pod](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/).
- Tìm hiểu về [các thành phần Kubernetes](./15-components-vi.md) và cách control plane sử dụng static Pod.
- Tìm hiểu về [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) như một lựa chọn thay thế cho static Pod.

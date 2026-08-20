# Lab 1A — B1 Overview

## 1. Kubernetes đang quản lý những máy nào?

Kubernetes đang quản lý ba VM ở vai trò Node của cluster:
`lab-k8s-master`, `lab-k8s-worker1` và `lab-k8s-worker2`.
Output `kubectl get nodes -o wide` cho thấy cả ba đã đăng ký và đều Ready.

Kubernetes theo dõi trạng thái các Node và dùng tài nguyên của chúng để chạy
workload. Kubernetes không quản lý vòng đời VM ở tầng VMware; việc tạo VM,
cấp CPU/RAM và vận hành hypervisor vẫn thuộc người quản trị.

## 2. Những component nào Kubernetes giữ chạy mà không cần khởi động thủ công từng container?

Output Pod cho thấy cluster đang duy trì:

- `kube-apiserver`, `etcd`, `kube-controller-manager` và `kube-scheduler`
  trên master.
- `kube-proxy` trên cả ba node.
- `kube-flannel` trên cả ba node.
- Hai Pod CoreDNS trong cluster.

Tôi không phải tự khởi động từng container bằng containerd. Kubelet và các
controller của Kubernetes duy trì Pod/container theo trạng thái mong muốn.

Containerd chỉ là container runtime trực tiếp chạy container; nó không phải
orchestrator và không tự điều phối cluster.

## 3. Kubernetes tự động hóa việc gì và việc gì vẫn thuộc người quản trị?

Kubernetes tự động hóa ở mức orchestration: triển khai workload đã khai báo,
chọn node cho Pod, duy trì số replica theo cấu hình, khởi động lại hoặc thay thế
workload bị lỗi và phân phối configuration do người quản trị cung cấp.

Người quản trị vẫn phải chọn ứng dụng và image; khai báo replica, scaling rule
và policy; thiết kế bảo mật và mạng; cung cấp configuration đúng; lập và kiểm
thử backup; bảo đảm capacity, giám sát, nâng cấp và sửa lỗi ứng dụng.

Kubernetes không tự xây ứng dụng, không tự chọn policy kinh doanh và không tự
bảo đảm backup dữ liệu.

## 4. Vì sao ba VM không tự trở thành cluster chỉ vì đã cài container runtime?

Container runtime như containerd chỉ thực thi container trên từng máy cục bộ.
Ba máy có containerd vẫn là ba runtime độc lập: chưa có API chung, nơi lưu
trạng thái cluster, scheduler, controller, node registration hoặc mạng Pod
xuyên node.

Để trở thành Kubernetes cluster, control plane phải được cấu hình, các worker
phải đăng ký qua kubelet và mạng cluster phải được thiết lập. Kubernetes quyết
định và điều phối workload; kubelet yêu cầu containerd trên từng node thực thi
container tương ứng.

# Tài liệu Kubernetes — Bản dịch tiếng Việt

Các bản dịch tiếng Việt của tài liệu chính thức trên <https://kubernetes.io/docs/>, giữ nguyên cấu trúc trang gốc. Mỗi file đều có link trang nguồn ở đầu. Tài liệu Kubernetes phát hành theo giấy phép CC BY 4.0.

Các file được đánh số theo **thứ tự học**: mỗi tài liệu chỉ dựa trên kiến thức của các tài liệu đứng trước nó, không yêu cầu kiến thức của phần sau.

## Phần 1 — Dựng cluster với kubeadm

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 01 | [Cài đặt kubeadm](01-install-kubeadm-vi.md) | — (điểm bắt đầu) |
| 02 | [Tạo một cluster với kubeadm](02-create-cluster-kubeadm-vi.md) | 01 (kubeadm/kubelet/kubectl đã cài) |
| 03 | [Tùy chỉnh các thành phần với kubeadm API](03-control-plane-flags-vi.md) | 02 (`kubeadm init`, file cấu hình `ClusterConfiguration`) |
| 04 | [Cấu hình từng kubelet trong cluster bằng kubeadm](04-kubelet-integration-vi.md) | 02–03 (quy trình init/join, kubeadm API) |
| 05 | [Hỗ trợ dual-stack với kubeadm](05-dual-stack-support-vi.md) | 02–03 (tạo cluster bằng file cấu hình) |
| 06 | [Các lựa chọn topology cho tính sẵn sàng cao](06-ha-topology-vi.md) | 02 (hiểu control plane, etcd là gì) |
| 07 | [Thiết lập cluster etcd có tính sẵn sàng cao với kubeadm](07-setup-ha-etcd-with-kubeadm-vi.md) | 06 (topology external etcd) |
| 08 | [Tạo cluster có tính sẵn sàng cao với kubeadm](08-high-availability-vi.md) | 06–07 (chọn topology; etcd external nếu dùng phải dựng trước) |
| 09 | [Xử lý sự cố kubeadm](09-troubleshooting-kubeadm-vi.md) | 01–08 (tài liệu tra cứu cho toàn bộ phần 1) |

## Phần 2 — Services & Networking

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 10 | [DNS cho Service và Pod](10-dns-pod-service-vi.md) | Phần 1 (cluster đang chạy), khái niệm Service |
| 11 | [Ingress](11-ingress-vi.md) | 10 (Service, DNS, hostname); định nghĩa khái niệm IngressClass |
| 12 | [Ingress Controllers](12-ingress-controllers-vi.md) | 11 (IngressClass, `ingressClassName`, annotation cũ) |
| 13 | [Gateway API](13-gateway-vi.md) | 11–12 (là hậu duệ của Ingress, có mục di chuyển từ Ingress) |

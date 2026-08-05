# Tài liệu Kubernetes — Bản dịch tiếng Việt

Các bản dịch tiếng Việt của tài liệu chính thức trên <https://kubernetes.io/docs/>, giữ nguyên cấu trúc trang gốc. Mỗi file đều có link trang nguồn ở đầu. Tài liệu Kubernetes phát hành theo giấy phép CC BY 4.0.

Các file được đánh số theo **thứ tự học**: mỗi tài liệu chỉ dựa trên kiến thức của các tài liệu đứng trước nó, không yêu cầu kiến thức của phần sau.

## Phần 1 — Dựng cluster với kubeadm

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 00 | [Các container runtime](00-container-runtimes-vi.md) | — (điểm bắt đầu) |
| 01 | [Cài đặt kubeadm](01-install-kubeadm-vi.md) | 00 (container runtime đã cài trên mỗi node) |
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

## Phần 3 — Khái niệm nền tảng (Overview & Cluster Architecture)

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 14 | [Tổng quan](14-overview-vi.md) | — (có thể đọc song song với Phần 1) |
| 15 | [Các thành phần của Kubernetes](15-components-vi.md) | 14 (Kubernetes là gì) |
| 16 | [Các đối tượng trong Kubernetes](16-working-with-objects-vi.md) | 14–15 (khái niệm cluster, control plane) |
| 17 | [Tên và ID của đối tượng](17-names-vi.md) | 16 (đối tượng là gì) |
| 18 | [Label và Selector](18-labels-vi.md) | 16–17 (đối tượng, metadata) |
| 19 | [Namespaces](19-namespaces-vi.md) | 16–18 (đối tượng, tên, label) |
| 20 | [Annotations](20-annotations-vi.md) | 16–18 (đối tượng, metadata, label) |
| 21 | [Kubernetes API](21-kubernetes-api-vi.md) | 15–16 (kube-apiserver, đối tượng) |
| 22 | [Kiến trúc cluster](22-architecture-vi.md) | 15, 21 (các thành phần, API) |
| 23 | [Các Node](23-nodes-vi.md) | 22 (kiến trúc tổng thể) |
| 24 | [Giao tiếp giữa Node và Control Plane](24-control-plane-node-communication-vi.md) | 22–23 (kiến trúc, node) |
| 25 | [Các Controller](25-controllers-vi.md) | 21–22 (API, kiến trúc) |
| 26 | [Công cụ dòng lệnh kubectl](26-kubectl-vi.md) | 16 (đối tượng); dùng xuyên suốt |
| 27 | [Quản lý đối tượng Kubernetes](27-object-management-vi.md) | 16, 26 (đối tượng, kubectl) |
| 28 | [Field Selectors](28-field-selectors-vi.md) | 16–18, 26 (đối tượng, label, kubectl) |
| 29 | [Finalizers](29-finalizers-vi.md) | 16 (đối tượng, vòng đời xóa) |
| 30 | [Owners và Dependents](30-owners-dependents-vi.md) | 16, 29 (đối tượng, finalizer) |
| 31 | [Các label được khuyến nghị](31-common-labels-vi.md) | 18 (label và selector) |
| 32 | [Storage Version](32-storage-version-vi.md) | 21 (Kubernetes API) |
| 33 | [Về cgroup v2](33-cgroups-vi.md) | 00, 23 (container runtime, node) |
| 34 | [Cloud Controller Manager](34-cloud-controller-vi.md) | 22, 25 (kiến trúc, controller) |
| 35 | [Leases](35-leases-vi.md) | 22–23, 25 (kiến trúc, node, controller) |
| 36 | [Thu gom rác](36-garbage-collection-vi.md) | 25, 29–30 (controller, finalizer, owner) |
| 37 | [Mixed Version Proxy](37-mixed-version-proxy-vi.md) | 21–22 (API, kiến trúc) |
| 38 | [Tự phục hồi](38-self-healing-vi.md) | 22–25 (kiến trúc, node, controller) |

## Phần 4 — Containers

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 39 | [Containers](39-containers-vi.md) | 00 (container runtime), 14–15 |
| 40 | [Images](40-images-vi.md) | 39 (container, image là gì) |
| 41 | [Môi trường container](41-container-environment-vi.md) | 39 |
| 42 | [Container Lifecycle Hooks](42-container-lifecycle-hooks-vi.md) | 39, 41 |
| 43 | [Runtime Class](43-runtime-class-vi.md) | 00, 39 (các runtime) |
| 44 | [Container Runtime Interface (CRI)](44-cri-vi.md) | 00, 22 (runtime, kiến trúc) |

## Phần 5 — Workloads

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 45 | [Workloads](45-workloads-vi.md) | 16 (đối tượng), 39 (container) |
| 46 | [Pods](46-pods-vi.md) | 45 (workload là gì) |
| 47 | [Vòng đời của Pod](47-pod-lifecycle-vi.md) | 46 (Pod) |
| 48 | [Pod Conditions](48-pod-condition-vi.md) | 47 (vòng đời Pod) |
| 49 | [Probes](49-probes-vi.md) | 47–48 (vòng đời, trạng thái Pod) |
| 50 | [Init Containers](50-init-containers-vi.md) | 46–47 (Pod, vòng đời) |
| 51 | [Sidecar Containers](51-sidecar-containers-vi.md) | 50 (init container) |
| 52 | [Ephemeral Containers](52-ephemeral-containers-vi.md) | 46 (Pod) |
| 53 | [Disruptions](53-disruptions-vi.md) | 47 (vòng đời Pod) |
| 54 | [Pod QoS](54-pod-qos-vi.md) | 46 (Pod, tài nguyên container) |
| 55 | [User Namespaces](55-user-namespaces-vi.md) | 46 (Pod) |
| 56 | [Downward API](56-downward-api-vi.md) | 46, 18 (Pod, label) |
| 57 | [Pod Hostname](57-pod-hostname-vi.md) | 46, 10 (Pod, DNS) |
| 58 | [Static Pods](58-static-pods-vi.md) | 46, 23 (Pod, kubelet/node) |
| 59 | [Scheduling Group](59-scheduling-group-vi.md) | 46 (Pod) |
| 60 | [Cấu hình Pod nâng cao](60-advanced-pod-config-vi.md) | 46–47 (Pod, vòng đời) |
| 61 | [Quản lý workload](61-management-vi.md) | 45–46, 26 (workload, kubectl) |
| 62 | [Workload Management](62-controllers-index-vi.md) | 45, 25 (workload, controller) |
| 63 | [Deployment](63-deployment-vi.md) | 62 (đọc kèm ReplicaSet 64) |
| 64 | [ReplicaSet](64-replicaset-vi.md) | 62, 18 (controller, label selector) |
| 65 | [StatefulSet](65-statefulset-vi.md) | 62–63 (dùng thêm PV ở Phần Storage) |
| 66 | [DaemonSet](66-daemonset-vi.md) | 62, 23 (controller, node) |

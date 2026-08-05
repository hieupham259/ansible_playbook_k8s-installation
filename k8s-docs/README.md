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
| 67 | [Jobs](67-job-vi.md) | 62 (workload management) |
| 68 | [TTL cho Job đã hoàn tất](68-ttlafterfinished-vi.md) | 67 (Job) |
| 69 | [CronJob](69-cron-jobs-vi.md) | 67 (Job) |
| 70 | [ReplicationController](70-replicationcontroller-vi.md) | 62, 64 (tiền thân của ReplicaSet) |
| 71 | [Autoscaling Workloads](71-autoscaling-vi.md) | 62–63 (các workload controller) |
| 72 | [Horizontal Pod Autoscaler](72-horizontal-pod-autoscale-vi.md) | 71 (autoscaling) |
| 73 | [Vertical Pod Autoscaler](73-vertical-pod-autoscale-vi.md) | 71–72 |
| 74 | [Resource Managers](74-resource-managers-vi.md) | 23, 54 (node/kubelet, tài nguyên Pod) |
| 75 | [PodGroup API](75-podgroup-api-vi.md) | 46, 59 (Pod, scheduling group) |
| 76 | [Vòng đời PodGroup](76-podgroup-lifecycle-vi.md) | 75 |
| 77 | [Workload API](77-workload-api-vi.md) | 45, 62 |
| 78 | [Workload: Disruption và Priority](78-workload-disruption-priority-vi.md) | 77, 53 (disruption) |
| 79 | [Workload Policies](79-workload-policies-vi.md) | 77 |
| 80 | [Workload: Topology-aware Scheduling](80-workload-topology-scheduling-vi.md) | 77 |

## Phần 6 — Services, Load Balancing và Networking (phần còn lại)

Ghi chú: các bài 10–13 (DNS, Ingress, Ingress Controllers, Gateway API) cũng thuộc mục này, đã dịch ở Phần 2.

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 81 | [Services, Load Balancing và Networking](81-services-networking-vi.md) | 22 (mô hình mạng cluster) |
| 82 | [Service](82-service-vi.md) | 81, 46, 18 (networking, Pod, selector) |
| 83 | [EndpointSlices](83-endpoint-slices-vi.md) | 82 (Service) |
| 84 | [Network Policies](84-network-policies-vi.md) | 82, 19 (Service, namespace) |
| 85 | [IPv4/IPv6 dual-stack](85-dual-stack-vi.md) | 82 (Service; liên quan 05) |
| 86 | [Topology Aware Routing](86-topology-aware-routing-vi.md) | 82–83 (Service, EndpointSlices) |
| 87 | [Service Internal Traffic Policy](87-service-traffic-policy-vi.md) | 82 (Service) |
| 88 | [Cấp phát địa chỉ ClusterIP](88-cluster-ip-allocation-vi.md) | 82 (Service) |
| 89 | [Networking trên Windows](89-windows-networking-vi.md) | 81–82 |

## Phần 7 — Storage

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 90 | [Storage](90-storage-vi.md) | — (trang mục) |
| 91 | [Volumes](91-volumes-vi.md) | 46, 90 (Pod, tổng quan storage) |
| 92 | [Persistent Volumes](92-persistent-volumes-vi.md) | 91 (volume) |
| 93 | [Projected Volumes](93-projected-volumes-vi.md) | 91 |
| 94 | [Ephemeral Volumes](94-ephemeral-volumes-vi.md) | 91–92 |
| 95 | [Ephemeral Storage](95-ephemeral-storage-vi.md) | 91, 54 (volume, tài nguyên Pod) |
| 96 | [Storage Classes](96-storage-classes-vi.md) | 92 (PV/PVC) |
| 97 | [Volume Attributes Classes](97-volume-attributes-classes-vi.md) | 92, 96 |
| 98 | [Dynamic Volume Provisioning](98-dynamic-provisioning-vi.md) | 92, 96 |
| 99 | [Volume Snapshots](99-volume-snapshots-vi.md) | 92 (PV/PVC) |
| 100 | [Volume Snapshot Classes](100-volume-snapshot-classes-vi.md) | 99 |
| 101 | [PVC làm DataSource (CSI Cloning)](101-volume-pvc-datasource-vi.md) | 92, 99 |
| 102 | [Volume Populators và Data Sources](102-volume-populators-vi.md) | 99, 101 |
| 103 | [Storage Capacity](103-storage-capacity-vi.md) | 92, 98 |
| 104 | [Giới hạn Volume theo Node](104-storage-limits-vi.md) | 91, 23 (volume, node) |
| 105 | [Giám sát sức khỏe Volume](105-volume-health-monitoring-vi.md) | 91–92 |
| 106 | [Storage trên Windows](106-windows-storage-vi.md) | 91 |

## Phần 8 — Configuration

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 107 | [Configuration](107-configuration-vi.md) | — (trang mục) |
| 108 | [ConfigMap](108-configmap-vi.md) | 46, 91 (Pod, volume) |
| 109 | [Secrets](109-secret-vi.md) | 108, 46 (ConfigMap, Pod) |
| 110 | [Quản lý tài nguyên cho Pod và Container](110-manage-resources-containers-vi.md) | 46, 54 (Pod, QoS) |
| 111 | [Tổ chức truy cập cluster bằng kubeconfig](111-kubeconfig-vi.md) | 26, 21 (kubectl, API) |
| 112 | [Quản lý tài nguyên trên Windows](112-windows-resource-management-vi.md) | 110 |

## Phần 9 — Security

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 113 | [Security](113-security-vi.md) | — (trang mục) |
| 114 | [Cloud Native Security](114-cloud-native-security-vi.md) | 113 |
| 115 | [Pod Security Standards](115-pod-security-standards-vi.md) | 46, 113 (Pod) |
| 116 | [Pod Security Admission](116-pod-security-admission-vi.md) | 115, 19 (PSS, namespace) |
| 117 | [Pod Security Policy (đã gỡ bỏ)](117-pod-security-policy-vi.md) | 115–116 |
| 118 | [Service Accounts](118-service-accounts-vi.md) | 16, 109 (đối tượng, Secret) |
| 119 | [Kiểm soát truy cập Kubernetes API](119-controlling-access-vi.md) | 21, 118 (API, service account) |
| 120 | [RBAC Good Practices](120-rbac-good-practices-vi.md) | 119 (kiểm soát truy cập) |
| 121 | [Secrets Good Practices](121-secrets-good-practices-vi.md) | 109, 120 |
| 122 | [Multi-tenancy](122-multi-tenancy-vi.md) | 19, 84, 119 (namespace, NetworkPolicy, RBAC) |
| 123 | [Hardening: Cơ chế xác thực](123-hardening-authentication-vi.md) | 119 (kiểm soát truy cập) |
| 124 | [Hardening: kube-scheduler](124-hardening-scheduler-vi.md) | 119 (đọc kèm Phần 11) |
| 125 | [Hardening: Dynamic Resource Allocation](125-hardening-dra-vi.md) | 119 (liên quan 149) |
| 126 | [Bảo mật Linux](126-linux-security-vi.md) | 113 (trang mục con) |
| 127 | [Ràng buộc bảo mật kernel Linux](127-linux-kernel-security-vi.md) | 126, 115 |
| 128 | [Rủi ro bypass API Server](128-api-server-bypass-risks-vi.md) | 119, 58 (static pod) |
| 129 | [Security Checklist](129-security-checklist-vi.md) | 113–128 (tổng kết mục Security) |
| 130 | [Application Security Checklist](130-application-security-checklist-vi.md) | 115, 129 |
| 131 | [Bảo mật Windows](131-windows-security-vi.md) | 113 |

## Phần 10 — Policies

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 132 | [Policies](132-policies-vi.md) | — (trang mục) |
| 133 | [Limit Ranges](133-limit-range-vi.md) | 110, 19 (tài nguyên, namespace) |
| 134 | [Resource Quotas](134-resource-quotas-vi.md) | 110, 133 |
| 135 | [PID Limiting](135-pid-limiting-vi.md) | 110, 23 (tài nguyên, node) |

## Phần 11 — Scheduling, Preemption và Eviction

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 136 | [Scheduling, Preemption và Eviction](136-scheduling-eviction-vi.md) | — (trang mục) |
| 137 | [kube-scheduler](137-kube-scheduler-vi.md) | 136, 22 (kiến trúc) |
| 138 | [Gán Pod vào Node](138-assign-pod-node-vi.md) | 137, 18, 23 (scheduler, label, node) |
| 139 | [Taints và Tolerations](139-taint-and-toleration-vi.md) | 138 |
| 140 | [Topology Spread Constraints](140-topology-spread-constraints-vi.md) | 138, 18 |
| 141 | [Pod Priority và Preemption](141-pod-priority-preemption-vi.md) | 137, 53 (scheduler, disruption) |
| 142 | [Node-pressure Eviction](142-node-pressure-eviction-vi.md) | 23, 54, 110 (node, QoS, tài nguyên) |
| 143 | [API-initiated Eviction](143-api-eviction-vi.md) | 53, 142 |
| 144 | [Pod Overhead](144-pod-overhead-vi.md) | 110, 43 (tài nguyên, RuntimeClass) |
| 145 | [Pod Scheduling Readiness](145-pod-scheduling-readiness-vi.md) | 137 |
| 146 | [Scheduler Performance Tuning](146-scheduler-perf-tuning-vi.md) | 137 |
| 147 | [Scheduling Framework](147-scheduling-framework-vi.md) | 137 |
| 148 | [Resource Bin Packing](148-resource-bin-packing-vi.md) | 137, 147 |
| 149 | [Dynamic Resource Allocation](149-dynamic-resource-allocation-vi.md) | 110, 137 |
| 150 | [Gang Scheduling](150-gang-scheduling-vi.md) | 59, 75, 137 (scheduling group, PodGroup) |
| 151 | [PodGroup Scheduling](151-podgroup-scheduling-vi.md) | 75, 150 |
| 152 | [Workload-aware Preemption](152-workload-aware-preemption-vi.md) | 141, 77 |
| 153 | [Topology-aware Scheduling](153-topology-aware-scheduling-vi.md) | 80, 138 |
| 154 | [Node Declared Features](154-node-declared-features-vi.md) | 23, 137 |

## Phần 12 — Cluster Administration

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 155 | [Cluster Administration](155-cluster-administration-vi.md) | — (trang mục) |
| 156 | [Chứng chỉ](156-certificates-vi.md) | — (trang trỏ hướng sang tài liệu tasks) |
| 157 | [Networking cho quản trị cluster](157-networking-vi.md) | 22, 81 (kiến trúc, networking) |
| 158 | [Kiến trúc Logging](158-logging-vi.md) | 46, 23 (Pod, node) |
| 159 | [System Logs](159-system-logs-vi.md) | 158 |
| 160 | [System Metrics](160-system-metrics-vi.md) | 155 |
| 161 | [Traces](161-system-traces-vi.md) | 155 |
| 162 | [Observability](162-observability-vi.md) | 158–161 (log, metric, trace) |
| 163 | [kube-state-metrics](163-kube-state-metrics-vi.md) | 160 |
| 164 | [Proxies](164-proxies-vi.md) | 82 (Service) |

# Tài liệu Kubernetes — Bản dịch tiếng Việt

Các bản dịch tiếng Việt của tài liệu chính thức trên <https://kubernetes.io/docs/>, giữ nguyên cấu trúc trang gốc. Mỗi file đều có link trang nguồn ở đầu. Tài liệu Kubernetes phát hành theo giấy phép CC BY 4.0.

**Tổng cộng 398 bài**: số `00`–`185` là nhánh khái niệm (`/docs/concepts/`, `/docs/setup/`), số `186` trở lên là nhánh thực hành (`/docs/tasks/`).

> **Muốn học theo lộ trình?** Xem [00-ALO-TRINH-ADMIN.md](00-ALO-TRINH-ADMIN.md) — giáo trình 30 giai đoạn dành cho người muốn trở thành Kubernetes administrator, chia hai phần: **Phần I — Nền tảng Kubernetes** (giai đoạn 1–15) và **Phần II — Vận hành cluster** (giai đoạn 16–30). Mỗi giai đoạn có mục tiêu và checkpoint thực hành riêng.
>
> **Muốn thực hành?** Xem [labs/](labs/README.md) — runbook chạy được cho từng nhóm bài, bắt đầu ở [Lab 00 — Môi trường](labs/LAB-00-MOI-TRUONG-1.35.7.md).
>
> File README này là **mục lục tra cứu theo chủ đề**. Số trong tên file bám theo cấu trúc mục của kubernetes.io để dễ đối chiếu khi trang gốc cập nhật — **không phải thứ tự đọc**; thứ tự đọc nằm ở file lộ trình.

Các file trong cùng một phần được sắp xếp sao cho tài liệu sau chỉ dựa trên kiến thức của các tài liệu đứng trước nó trong phần đó.

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
| 165 | [Addons](165-addons-vi.md) | 155, 157 (quản trị cluster, networking) |
| 166 | [API Priority và Fairness](166-flow-control-vi.md) | 21, 119 (API, kiểm soát truy cập) |
| 167 | [Coordinated Leader Election](167-coordinated-leader-election-vi.md) | 35 (Leases) |
| 168 | [Compatibility Version](168-compatibility-version-vi.md) | 22 (control plane) |
| 169 | [Node Shutdown](169-node-shutdown-vi.md) | 23, 47 (node, vòng đời Pod) |
| 170 | [Quản lý bộ nhớ Swap](170-swap-memory-management-vi.md) | 00, 110 (swap khi cài, tài nguyên) |
| 171 | [Node Autoscaling](171-node-autoscaling-vi.md) | 23, 71 (node, autoscaling workload) |
| 172 | [DRA cho quản trị viên cluster](172-cluster-admin-dra-vi.md) | 149 (DRA) |
| 173 | [Admission Webhooks Good Practices](173-admission-webhooks-vi.md) | 119, 166 (kiểm soát truy cập, APF) |

## Phần 13 — Windows in Kubernetes

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 174 | [Windows trong Kubernetes](174-windows-vi.md) | — (trang mục) |
| 175 | [Container Windows trong Kubernetes](175-windows-intro-vi.md) | 174, 39 (container) |
| 176 | [Hướng dẫn lập lịch Windows container](176-windows-user-guide-vi.md) | 175, 63 (Deployment) |

## Phần 14 — Extending Kubernetes

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 177 | [Mở rộng Kubernetes](177-extend-kubernetes-vi.md) | 21–22 (API, kiến trúc) |
| 178 | [Mở rộng API Kubernetes](178-api-extension-vi.md) | 177 |
| 179 | [Custom Resources](179-custom-resources-vi.md) | 178, 16 (mở rộng API, đối tượng) |
| 180 | [Aggregation Layer của API Server](180-apiserver-aggregation-vi.md) | 178–179 |
| 181 | [Mẫu Operator](181-operator-vi.md) | 179, 25 (custom resource, controller) |
| 182 | [Mở rộng Compute, Storage và Networking](182-compute-storage-net-vi.md) | 177 |
| 183 | [Network Plugins](183-network-plugins-vi.md) | 182, 157 (networking) |
| 184 | [Device Plugins](184-device-plugins-vi.md) | 182, 110 (tài nguyên) |

## Phần 15 — Tasks: Install Tools

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 185 | [Install Tools](185-tools-vi.md) | — (trang mục) |
| 186 | [Cài kubectl trên Linux](186-install-kubectl-linux-vi.md) | 26 (kubectl là gì) |
| 187 | [Cài kubectl trên macOS](187-install-kubectl-macos-vi.md) | 26 |
| 188 | [Cài kubectl trên Windows](188-install-kubectl-windows-vi.md) | 26 |

## Phần 16 — Tasks: Administer a Cluster

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 189 | [Administer a Cluster](189-administer-cluster-vi.md) | — (trang mục) |
| 190 | [Truy cập Kubernetes API của cluster](190-access-cluster-api-vi.md) | 21, 111 (API, kubeconfig) |
| 191 | [Tạo certificate thủ công](191-certificates-manual-vi.md) | Giai đoạn 0 (TLS/openssl), 119 |
| 192 | [Đổi StorageClass mặc định](192-change-default-storage-class-vi.md) | 96 (StorageClass) |
| 193 | [Đổi access mode PV sang ReadWriteOncePod](193-change-pv-access-mode-vi.md) | 92 (PV/PVC) |
| 194 | [Đổi Reclaim Policy của PersistentVolume](194-change-pv-reclaim-policy-vi.md) | 92 |
| 195 | [Nâng cấp một cluster](195-cluster-upgrade-vi.md) | 02, 22 (chi tiết kubeadm ở 221) |
| 196 | [Cấu hình Feature Gates](196-configure-feature-gates-vi.md) | 15, 21 |
| 197 | [Vận hành cluster etcd — backup/restore/upgrade](197-configure-upgrade-etcd-vi.md) | 06–07, 22 (etcd, HA) |
| 198 | [Migration leader của Controller Manager](198-controller-manager-leader-migration-vi.md) | 34–35 (CCM, lease) |
| 199 | [Sử dụng CoreDNS cho Service Discovery](199-coredns-vi.md) | 10, 15 (DNS, components) |
| 200 | [CPU Management Policies](200-cpu-management-policies-vi.md) | 74 (resource managers) |
| 201 | [Khai báo Network Policy](201-declare-network-policy-vi.md) | 84 (NetworkPolicy) |
| 202 | [Giải mã dữ liệu đã mã hóa at rest](202-decrypt-data-vi.md) | 208 (encrypt-data) |
| 203 | [Phát triển Cloud Controller Manager](203-developing-cloud-controller-manager-vi.md) | 34 (CCM) |
| 204 | [Tùy chỉnh DNS Service (CoreDNS)](204-dns-custom-nameservers-vi.md) | 10, 199 |
| 205 | [Debug phân giải DNS](205-dns-debugging-resolution-vi.md) | 10, 204 |
| 206 | [Tự động co giãn DNS Service](206-dns-horizontal-autoscaling-vi.md) | 10, 63 (DNS, Deployment) |
| 207 | [Bật/tắt một Kubernetes API](207-enable-disable-api-vi.md) | 21 (API groups) |
| 208 | [Mã hóa dữ liệu bí mật at rest](208-encrypt-data-vi.md) | 109, 121 (Secret) |
| 209 | [Quảng bá Extended Resources cho Node](209-extended-resource-node-vi.md) | 110 (tài nguyên) |
| 210 | [Guaranteed Scheduling cho Critical Add-on Pods](210-guaranteed-scheduling-critical-addon-pods-vi.md) | 141 (priority) |
| 211 | [Tăng cường bảo mật DRA (tasks)](211-hardening-dra-tasks-vi.md) | 149, 125 (DRA) |
| 212 | [IP Masquerade Agent](212-ip-masq-agent-vi.md) | 157, 84 (mạng cluster) |
| 213 | [KMS provider cho mã hóa dữ liệu](213-kms-provider-vi.md) | 208 (encrypt-data) |
| 214 | [Quản trị với kubeadm](214-kubeadm-tasks-vi.md) | — (trang mục) |
| 215 | [Thêm node Linux](215-adding-linux-nodes-vi.md) | 02 (kubeadm join) |
| 216 | [Thêm node Windows](216-adding-windows-nodes-vi.md) | 175–176, 215 |
| 217 | [Đổi kho gói Kubernetes](217-change-package-repository-vi.md) | 01 (cài kubeadm) |
| 218 | [Cấu hình cgroup driver bằng kubeadm](218-configure-cgroup-driver-vi.md) | 00, 33 (runtime, cgroup) |
| 219 | [Quản lý certificate với kubeadm](219-kubeadm-certs-vi.md) | 191, Giai đoạn 0 (TLS) |
| 220 | [Cấu hình lại cluster kubeadm](220-kubeadm-reconfigure-vi.md) | 03–04 (kubeadm API) |
| 221 | [Nâng cấp cluster kubeadm](221-kubeadm-upgrade-vi.md) | 195, 02 |
| 222 | [Nâng cấp node Linux](222-upgrading-linux-nodes-vi.md) | 221 |
| 223 | [Nâng cấp node Windows](223-upgrading-windows-nodes-vi.md) | 221 |
| 224 | [Cấu hình kubelet qua file cấu hình](224-kubelet-config-file-vi.md) | 04 (kubelet) |
| 225 | [Kubelet credential provider](225-kubelet-credential-provider-vi.md) | 40 (kéo image) |
| 226 | [Chạy kubelet trong user namespace](226-kubelet-in-userns-vi.md) | 55 (user namespaces) |
| 227 | [Giới hạn tiêu thụ lưu trữ](227-limit-storage-consumption-vi.md) | 134, 92 (quota, PVC) |
| 228 | [Quản lý tài nguyên Memory/CPU/API](228-manage-resources-tasks-vi.md) | — (trang mục) |
| 229 | [Ràng buộc CPU min/max cho namespace](229-cpu-constraint-namespace-vi.md) | 133 (LimitRange) |
| 230 | [CPU request/limit mặc định cho namespace](230-cpu-default-namespace-vi.md) | 133 |
| 231 | [Ràng buộc memory min/max cho namespace](231-memory-constraint-namespace-vi.md) | 133 |
| 232 | [Memory request/limit mặc định cho namespace](232-memory-default-namespace-vi.md) | 133 |
| 233 | [Quota memory/CPU cho namespace](233-quota-memory-cpu-namespace-vi.md) | 134 (ResourceQuota) |
| 234 | [Quota số Pod cho namespace](234-quota-pod-namespace-vi.md) | 134 |
| 235 | [Memory Manager (NUMA)](235-memory-manager-vi.md) | 74, 200 (resource managers) |
| 236 | [Di chuyển khỏi dockershim](236-migrating-from-dockershim-vi.md) | 00 (trang mục nhóm) |
| 237 | [Đổi runtime sang containerd](237-change-runtime-containerd-vi.md) | 236, 00 |
| 238 | [Kiểm tra ảnh hưởng của việc gỡ dockershim](238-check-dockershim-removal-vi.md) | 236 |
| 239 | [Tìm container runtime đang dùng trên node](239-find-out-runtime-vi.md) | 236, 44 (CRI) |
| 240 | [Di chuyển agent telemetry và bảo mật](240-migrating-telemetry-agents-vi.md) | 236 |
| 241 | [Xử lý lỗi liên quan CNI plugin](241-troubleshooting-cni-errors-vi.md) | 236, 183 (CNI) |
| 242 | [Thao tác với Namespaces](242-namespaces-tasks-vi.md) | 19 (namespace) |
| 243 | [Network Policy Providers](243-network-policy-provider-vi.md) | — (trang mục) |
| 244 | [Antrea cho NetworkPolicy](244-antrea-network-policy-vi.md) | 84, 243 |
| 245 | [Calico cho NetworkPolicy](245-calico-network-policy-vi.md) | 84, 243 |
| 246 | [Cilium cho NetworkPolicy](246-cilium-network-policy-vi.md) | 84, 243 |
| 247 | [kube-router cho NetworkPolicy](247-kube-router-network-policy-vi.md) | 84, 243 |
| 248 | [Romana cho NetworkPolicy](248-romana-network-policy-vi.md) | 84, 243 |
| 249 | [Weave Net cho NetworkPolicy](249-weave-network-policy-vi.md) | 84, 243 |
| 250 | [Node Overprovisioning](250-node-overprovisioning-vi.md) | 141, 171 (priority, autoscaling) |
| 251 | [NodeLocal DNSCache](251-nodelocaldns-vi.md) | 10, 204 (DNS) |
| 252 | [Quota cho API object](252-quota-api-object-vi.md) | 134 (ResourceQuota) |
| 253 | [Dự trữ tài nguyên cho system daemons](253-reserve-compute-resources-vi.md) | 110, 142 |
| 254 | [Chạy Cloud Controller Manager](254-running-cloud-controller-vi.md) | 34 (CCM) |
| 255 | [Drain node an toàn](255-safely-drain-node-vi.md) | 53, 143 (disruption, eviction API) |
| 256 | [Bảo mật một cluster](256-securing-a-cluster-vi.md) | 129, 119 (checklist, kiểm soát truy cập) |
| 257 | [Chuyển sang Evented PLEG](257-switch-to-evented-pleg-vi.md) | 23, 44 (kubelet, CRI) |
| 258 | [Sử dụng sysctl trong cluster](258-sysctl-cluster-vi.md) | 46, 127 (Pod, kernel) |
| 259 | [Topology Manager](259-topology-manager-vi.md) | 74, 200, 235 (resource managers) |
| 260 | [Cascading deletion](260-use-cascading-deletion-vi.md) | 30, 36 (owner, thu gom rác) |
| 261 | [Xác minh artifact có chữ ký](261-verify-signed-artifacts-vi.md) | 40 (image) |

## Phần 17 — Tasks: Configure Pods and Containers

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 262 | [Configure Pods and Containers](262-configure-pod-container-vi.md) | — (trang mục) |
| 263 | [Gán tài nguyên CPU cho container](263-assign-cpu-resource-vi.md) | 110 (requests/limits) |
| 264 | [Gán tài nguyên memory cho container](264-assign-memory-resource-vi.md) | 110 |
| 265 | [Tài nguyên cấp Pod](265-assign-pod-level-resources-vi.md) | 110 |
| 266 | [Gán Pod vào Node bằng Node Affinity](266-assign-pods-nodes-node-affinity-vi.md) | 138 (affinity) |
| 267 | [Gán Pod vào Node](267-assign-pods-nodes-vi.md) | 138 (nodeSelector) |
| 268 | [Assign Resources (DRA)](268-assign-resources-vi.md) | 149 (trang mục nhóm) |
| 269 | [Đọc metadata thiết bị DRA](269-access-dra-device-metadata-vi.md) | 149, 268 |
| 270 | [Cấp phát thiết bị cho workload bằng DRA](270-allocate-devices-dra-vi.md) | 271 |
| 271 | [Dựng cluster có DRA](271-set-up-dra-cluster-vi.md) | 149, 268 |
| 272 | [Gắn handler vào lifecycle event](272-attach-handler-lifecycle-event-vi.md) | 42 (hooks) |
| 273 | [Cấu hình GMSA cho Windows Pod](273-configure-gmsa-vi.md) | 175–176 (Windows) |
| 274 | [Cấu hình Liveness/Readiness/Startup Probes](274-configure-probes-vi.md) | 49 (probes) |
| 275 | [Cấu hình Pod dùng ConfigMap](275-configure-pod-configmap-vi.md) | 108 (ConfigMap) |
| 276 | [Cấu hình khởi tạo Pod](276-configure-pod-initialization-vi.md) | 50 (init containers) |
| 277 | [Cấu hình projected volume](277-configure-projected-volume-vi.md) | 93 |
| 278 | [Cấu hình RunAsUserName (Windows)](278-configure-runasusername-vi.md) | 175 |
| 279 | [Cấu hình Service Account cho Pod](279-configure-service-account-vi.md) | 118 |
| 280 | [Cấu hình volume cho lưu trữ](280-configure-volume-storage-vi.md) | 91 |
| 281 | [Tạo HostProcess Pod (Windows)](281-create-hostprocess-pod-vi.md) | 175–176 |
| 282 | [Áp Pod Security Standards ở admission controller](282-enforce-standards-admission-controller-vi.md) | 116 |
| 283 | [Áp PSS bằng label namespace](283-enforce-standards-namespace-labels-vi.md) | 116 |
| 284 | [Gán extended resource cho container](284-extended-resource-vi.md) | 209 |
| 285 | [Dùng image volume trong Pod](285-image-volumes-vi.md) | 91, 40 |
| 286 | [Di chuyển từ PodSecurityPolicy](286-migrate-from-psp-vi.md) | 117, 116 |
| 287 | [Kéo image từ private registry](287-pull-image-private-registry-vi.md) | 40, 109 |
| 288 | [Cấu hình QoS cho Pod](288-quality-service-pod-vi.md) | 54, 110 |
| 289 | [Thay đổi tài nguyên container tại chỗ](289-resize-container-resources-vi.md) | 110 |
| 290 | [Thay đổi tài nguyên cấp Pod](290-resize-pod-resources-vi.md) | 265, 289 |
| 291 | [Cấu hình Security Context](291-security-context-vi.md) | 115, 127 |
| 292 | [Chia sẻ process namespace giữa các container](292-share-process-namespace-vi.md) | 46 |
| 293 | [Tạo static Pod](293-static-pod-tasks-vi.md) | 58 |
| 294 | [Chuyển Docker Compose sang Kubernetes (Kompose)](294-translate-compose-kubernetes-vi.md) | 63, 82 |
| 295 | [Dùng user namespace cho Pod](295-user-namespaces-tasks-vi.md) | 55 |

## Phần 18 — Tasks: Monitoring, Logging và Debugging

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 296 | [Monitoring, Logging và Debugging](296-debug-vi.md) | — (trang mục) |
| 297 | [Debug ứng dụng](297-debug-application-vi.md) | — (trang mục nhóm) |
| 298 | [Debug Init Containers](298-debug-init-containers-vi.md) | 50, 276 |
| 299 | [Debug Pods](299-debug-pods-vi.md) | 47, 110 (vòng đời, tài nguyên) |
| 300 | [Debug Pod đang chạy](300-debug-running-pod-vi.md) | 52, 299 (ephemeral container) |
| 301 | [Debug Services](301-debug-service-vi.md) | 82, 10 (Service, DNS) |
| 302 | [Debug StatefulSet](302-debug-statefulset-vi.md) | 65, 299 |
| 303 | [Xác định lý do Pod fail](303-determine-reason-pod-failure-vi.md) | 47–48 |
| 304 | [Lấy shell vào container đang chạy](304-get-shell-running-container-vi.md) | 26, 46 |
| 305 | [Debug Cluster](305-debug-cluster-vi.md) | 09, 23 (troubleshooting, node) |
| 306 | [Auditing](306-audit-vi.md) | 119, 256 (kiểm soát truy cập) |
| 307 | [Debug node bằng crictl](307-crictl-vi.md) | 44, 00 (CRI, runtime) |
| 308 | [Debug node bằng kubectl](308-kubectl-node-debug-vi.md) | 300, 23 |
| 309 | [Local debugging](309-local-debugging-vi.md) | 26, 82 |
| 310 | [Giám sát sức khỏe của Node](310-monitor-node-health-vi.md) | 23, 305 (node, debug cluster) |
| 311 | [Pipeline metrics tài nguyên](311-resource-metrics-pipeline-vi.md) | 160, 72 (metric hệ thống, HPA) |
| 312 | [Các công cụ giám sát tài nguyên](312-resource-usage-monitoring-vi.md) | 162, 311 |
| 313 | [Khắc phục sự cố Topology Management](313-debug-topology-vi.md) | 74, 259 (topology manager) |
| 314 | [Khắc phục sự cố kubectl](314-troubleshoot-kubectl-vi.md) | 26, 111 (kubectl, kubeconfig) |
| 315 | [Mẹo debug Windows](315-debug-windows-vi.md) | 174–176 (Windows) |
| 316 | [Ghi log trong Kubernetes](316-debug-logging-vi.md) | 158–159 (kiến trúc log) |
| 317 | [Giám sát trong Kubernetes](317-debug-monitoring-vi.md) | 162, 160 |

## Phần 19 — Tasks: Quản lý object bằng kubectl

Nhóm thực hành đi kèm bài [27](27-object-management-vi.md) — ba kỹ thuật quản lý object.

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 318 | [Quản lý các đối tượng Kubernetes](318-manage-kubernetes-objects-vi.md) | — (trang mục) |
| 319 | [Quản lý theo kiểu khai báo bằng file cấu hình](319-declarative-config-vi.md) | 27, 26 |
| 320 | [Quản lý bằng lệnh imperative](320-imperative-command-vi.md) | 27, 26 |
| 321 | [Quản lý theo kiểu imperative bằng file cấu hình](321-imperative-config-vi.md) | 27, 320 |
| 322 | [Quản lý theo kiểu khai báo bằng Kustomize](322-kustomization-vi.md) | 319, 108–109 |
| 323 | [Di trú object bằng Storage Version Migration](323-storage-version-migration-vi.md) | 32 (phiên bản lưu trữ) |
| 324 | [Cập nhật object tại chỗ bằng kubectl patch](324-kubectl-patch-vi.md) | 27, 63 (Deployment) |

## Phần 20 — Tasks: ConfigMap, Secret và đưa dữ liệu vào ứng dụng

Nhóm thực hành đi kèm bài [108](108-configmap-vi.md), [109](109-secret-vi.md) và [56](56-downward-api-vi.md).

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 325 | [Quản lý Secret](325-configmap-secret-vi.md) | 109 (trang mục nhóm) |
| 326 | [Quản lý Secret bằng file cấu hình](326-secret-config-file-vi.md) | 109, 321 |
| 327 | [Quản lý Secret bằng kubectl](327-secret-kubectl-vi.md) | 109, 320 |
| 328 | [Quản lý Secret bằng Kustomize](328-secret-kustomize-vi.md) | 109, 322 |
| 329 | [Đưa dữ liệu vào ứng dụng](329-inject-data-application-vi.md) | — (trang mục nhóm) |
| 330 | [Định nghĩa command và argument cho container](330-define-command-argument-vi.md) | 39, 41 |
| 331 | [Định nghĩa biến môi trường cho Container](331-define-environment-variable-vi.md) | 41 |
| 332 | [Định nghĩa giá trị biến môi trường bằng Init Container](332-define-env-via-file-vi.md) | 50, 331 |
| 333 | [Định nghĩa các biến môi trường phụ thuộc](333-interdependent-env-variables-vi.md) | 331 |
| 334 | [Phân phối credential an toàn bằng Secret](334-distribute-credentials-secure-vi.md) | 109, 93 |
| 335 | [Expose thông tin Pod qua file](335-downward-api-volume-vi.md) | 56, 93 (Downward API) |
| 336 | [Expose thông tin Pod qua biến môi trường](336-env-variable-expose-pod-info-vi.md) | 56, 331 |

## Phần 21 — Tasks: Chạy ứng dụng

Nhóm thực hành đi kèm giai đoạn 4 — workload controller.

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 337 | [Chạy ứng dụng](337-run-application-vi.md) | — (trang mục nhóm) |
| 338 | [Truy cập Kubernetes API từ một Pod](338-access-api-from-pod-vi.md) | 118, 21 (ServiceAccount, API) |
| 339 | [Chỉ định Disruption Budget cho ứng dụng](339-configure-pdb-vi.md) | 53 (disruption, PDB) |
| 340 | [Xóa một StatefulSet](340-delete-stateful-set-vi.md) | 65 |
| 341 | [Xóa cưỡng bức Pod của StatefulSet](341-force-delete-stateful-set-pod-vi.md) | 65, 340 |
| 342 | [Hướng dẫn từng bước về HorizontalPodAutoscaler](342-hpa-walkthrough-vi.md) | 72, 311 (**cần metrics-server**) |
| 343 | [Chạy ứng dụng có trạng thái được nhân bản](343-run-replicated-stateful-application-vi.md) | 65, 92, 96 |
| 344 | [Chạy ứng dụng có trạng thái đơn thực thể](344-run-single-instance-stateful-vi.md) | 63, 92 |
| 345 | [Chạy ứng dụng Stateless bằng Deployment](345-run-stateless-application-vi.md) | 63 |
| 346 | [Scale thủ công theo chiều ngang cho Deployment](346-scale-deployment-vi.md) | 63, 64 |
| 347 | [Scale một StatefulSet](347-scale-stateful-set-vi.md) | 65 |
| 348 | [Cập nhật Deployment không gây gián đoạn](348-update-deployment-rolling-vi.md) | 63, 49 (rollout, probe) |

## Phần 22 — Tasks: Job và CronJob

Nhóm thực hành đi kèm bài [67](67-job-vi.md) và [69](69-cron-jobs-vi.md).

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 349 | [Chạy Job](349-job-tasks-vi.md) | 67 (trang mục nhóm) |
| 350 | [Chạy tác vụ tự động với CronJob](350-automated-tasks-cron-jobs-vi.md) | 69 |
| 351 | [Xử lý song song thô bằng hàng đợi công việc](351-coarse-parallel-work-queue-vi.md) | 67 |
| 352 | [Xử lý song song mịn bằng hàng đợi công việc](352-fine-parallel-work-queue-vi.md) | 67, 351 |
| 353 | [Indexed Job phân công việc tĩnh](353-indexed-parallel-processing-vi.md) | 67 |
| 354 | [Job với giao tiếp Pod-đến-Pod](354-job-pod-to-pod-communication-vi.md) | 67, 82 (headless Service) |
| 355 | [Xử lý song song bằng khai triển template](355-parallel-processing-expansion-vi.md) | 67, 320 |
| 383 | [Xử lý các lần Pod thất bại có thể thử lại và không thể thử lại bằng Pod failure policy](383-pod-failure-policy-vi.md) | 67, 47 (Job, vòng đời Pod) |

## Phần 23 — Tasks: Truy cập ứng dụng trong cluster

Nhóm thực hành đi kèm giai đoạn 5 — mạng nền tảng.

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 359 | [Truy cập cluster](359-access-cluster-vi.md) | 111, 26 (kubeconfig) |
| 360 | [Giao tiếp giữa các Container bằng Volume dùng chung](360-containers-shared-volume-vi.md) | 46, 91 (`emptyDir`) |
| 361 | [Cấu hình truy cập nhiều cluster](361-configure-access-multiple-clusters-vi.md) | 111 (context, cluster, user) |
| 362 | [Cấu hình DNS cho một cluster](362-configure-dns-cluster-vi.md) | 10, 204 (DNS, CoreDNS) |
| 363 | [Kết nối Frontend với Backend bằng Service](363-connecting-frontend-backend-vi.md) | 82, 63 |
| 364 | [Tạo bộ cân bằng tải bên ngoài](364-create-external-load-balancer-vi.md) | 82 (LoadBalancer) |
| 365 | [Liệt kê mọi Container image đang chạy](365-list-running-container-images-vi.md) | 26 (`-o jsonpath`) |
| 366 | [Port Forwarding để truy cập ứng dụng](366-port-forward-vi.md) | 26, 82 |
| 368 | [Truy cập ứng dụng trong một cluster](368-access-application-cluster-index-vi.md) | — (trang mục nhóm) |
| 369 | [Truy cập các Service đang chạy trên cluster](369-access-cluster-services-vi.md) | 190, 82 (truy cập API, Service) |
| 370 | [Dùng Service để truy cập một ứng dụng trong cluster](370-service-access-application-cluster-vi.md) | 82, 63 (Service, Deployment) |
| 371 | [Triển khai và Truy cập Kubernetes Dashboard](371-web-ui-dashboard-vi.md) | 165, 120 (add-on, RBAC) |

## Phần 24 — Tasks: Quản lý DaemonSet

Nhóm thực hành đi kèm bài [66](66-daemonset-vi.md). Thứ tự dưới đây theo đúng thứ tự kubernetes.io hiển thị, không theo số file.

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 384 | [Quản lý các daemon của cluster](384-manage-daemon-index-vi.md) | 66 (trang mục nhóm) |
| 385 | [Xây dựng một DaemonSet cơ bản](385-create-daemon-set-vi.md) | 66, 50 (DaemonSet, init container) |
| 388 | [Thực hiện rolling update trên một DaemonSet](388-update-daemon-set-vi.md) | 66, 63 (rolling update) |
| 387 | [Thực hiện rollback trên một DaemonSet](387-rollback-daemon-set-vi.md) | 388, 63 (rollback, revision) |
| 386 | [Chỉ chạy Pod trên một số Node nhất định](386-pods-some-nodes-vi.md) | 66, 138 (nodeSelector, affinity) |

## Phần 25 — Tasks: Mạng

Nhóm thực hành đi kèm bài [82](82-service-vi.md), [85](85-dual-stack-vi.md) và [88](88-cluster-ip-allocation-vi.md).

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 391 | [Mạng](391-network-index-vi.md) | — (trang mục nhóm) |
| 392 | [Thêm entry vào file /etc/hosts của Pod bằng HostAliases](392-customize-hosts-file-for-pods-vi.md) | 10, 57 (DNS, hostname Pod) |
| 393 | [Mở rộng dải IP của Service](393-extend-service-ip-ranges-vi.md) | 88, 82 (ServiceCIDR, Service) |
| 394 | [Cấu hình lại ServiceCIDR mặc định của Kubernetes](394-reconfigure-default-service-ip-ranges-vi.md) | 393, 88 |
| 395 | [Kiểm chứng dual-stack IPv4/IPv6](395-validate-dual-stack-vi.md) | 85, 05 (dual-stack) |

## Phần 26 — Tasks: TLS và vòng đời certificate

Nhóm thực hành của [giai đoạn 18](00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ). Đi kèm bài [156](156-certificates-vi.md) và [219](219-kubeadm-certs-vi.md).

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 396 | [TLS](396-tls-index-vi.md) | — (trang mục nhóm) |
| 398 | [Cấu hình xoay vòng certificate cho kubelet](398-certificate-rotation-vi.md) | 219 (certificate kubelet) |
| 399 | [Quản lý TLS Certificate trong một Cluster](399-managing-tls-in-a-cluster-vi.md) | 219, 191 (CSR API, signer) |
| 397 | [Cấp certificate cho một client của Kubernetes API bằng CertificateSigningRequest](397-certificate-issue-client-csr-vi.md) | 399, 120 (CSR, RBAC) |
| 400 | [Xoay CA certificate thủ công](400-manual-rotation-of-ca-certificates-vi.md) | 219, 191 (xoay CA — thao tác nguy hiểm) |

## Phần 27 — Tasks: Mở rộng Kubernetes

Nhóm thực hành đi kèm giai đoạn 14. Thứ tự theo đúng thứ tự kubernetes.io hiển thị.

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 373 | [Mở rộng Kubernetes](373-extend-kubernetes-index-vi.md) | 177 (trang mục nhóm) |
| 374 | [Cấu hình tầng tổng hợp](374-configure-aggregation-layer-vi.md) | 180, 119 (aggregation layer) |
| 376 | [Sử dụng Custom Resource](376-custom-resources-index-vi.md) | 179 (trang mục con) |
| 378 | [Mở rộng Kubernetes API bằng CustomResourceDefinition](378-custom-resource-definitions-vi.md) | 179 (CRD — bài xương sống) |
| 377 | [Các phiên bản trong CustomResourceDefinition](377-custom-resource-definition-versioning-vi.md) | 378, 32 (nhiều version, conversion webhook) |
| 380 | [Thiết lập một Extension API Server](380-setup-extension-api-server-vi.md) | 374, 180 |
| 375 | [Cấu hình nhiều Scheduler](375-configure-multiple-schedulers-vi.md) | 137, 147 (scheduler) |
| 379 | [Dùng HTTP Proxy để truy cập Kubernetes API](379-http-proxy-access-api-vi.md) | 164, 190 (kubectl proxy) |
| 382 | [Dùng SOCKS5 Proxy để truy cập Kubernetes API](382-socks5-proxy-access-api-vi.md) | 164, 111 (SOCKS5, kubeconfig) |
| 381 | [Thiết lập dịch vụ Konnectivity](381-setup-konnectivity-vi.md) | 24, 164 (konnectivity) |
| 372 | [Mở rộng kubectl bằng plugin](372-kubectl-plugins-vi.md) | 26, 177 (plugin kubectl) |

## Phần 28 — Tasks: GPU, HugePages và trang mục gốc

Hai bài thiết bị chuyên dụng, cộng trang mục gốc của toàn nhánh `/docs/tasks/`.

| # | Tài liệu | Kiến thức cần trước |
|---|---|---|
| 389 | [Lập lịch GPU](389-scheduling-gpus-vi.md) | 184, 149 (device plugin, DRA) |
| 390 | [Quản lý HugePages](390-scheduling-hugepages-vi.md) | 110, 74 (tài nguyên, hugepages) |
| 367 | [Tác vụ](367-tasks-index-vi.md) | — (trang mục gốc /docs/tasks/) |
## Checkpoint — Những phần còn thiếu quan trọng

Phần này theo dõi các khoảng trống cần bổ sung để bộ tài liệu không chỉ bao phủ
Kubernetes Concepts mà còn đủ dùng cho việc quản trị cluster thực tế. Đánh dấu
`[x]` khi đã có tài liệu hướng dẫn đầy đủ và có bài thực hành kiểm chứng; việc một
khái niệm chỉ được nhắc đến hoặc trỏ sang tài liệu bên ngoài chưa được coi là hoàn tất.

> **Cập nhật:** phần lớn *tài liệu* cho các mục dưới đây **đã có bản dịch** trong thư mục (nhóm số `186` trở lên) và đã được nối vào lộ trình ở [giai đoạn 16–27](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster). Cái còn thiếu là **bài lab kiểm chứng** — theo đúng tiêu chí của mục này, có tài liệu mà chưa có thực hành thì vẫn để `[ ]`. Xem [bản đồ lab](labs/README.md#4-bản-đồ-lab).

### A. Vòng đời cluster và quản trị bằng kubeadm

- [ ] Nâng cấp control plane và worker node bằng `kubeadm upgrade`.
- [ ] Giải thích version skew và thứ tự nâng cấp các component.
- [ ] Cấu hình lại một cluster do kubeadm quản lý.
- [ ] Thay đổi hoặc di chuyển container runtime trên node.
- [ ] Thêm, xóa và thay thế node an toàn.
- [ ] Thực hành `cordon`, `drain`, `uncordon` và xử lý PodDisruptionBudget khi bảo trì node.
- [ ] Xử lý và khôi phục khi quá trình nâng cấp cluster thất bại.

Tài liệu tham chiếu: <https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/>

### B. Vận hành etcd và khôi phục thảm họa

- [ ] Kiểm tra endpoint health, endpoint status và cảnh báo mất quorum.
- [ ] Tạo và xác minh snapshot bằng `etcdctl`.
- [ ] Khôi phục stacked etcd từ snapshot.
- [ ] Khôi phục external etcd từ snapshot.
- [ ] Cập nhật static Pod manifest và data directory sau khi restore.
- [ ] Bảo trì database: compaction, defragmentation và theo dõi dung lượng.
- [ ] Xây dựng và kiểm thử runbook disaster recovery cho control plane.

Tài liệu tham chiếu: <https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/>

### C. Certificate và PKI

- [ ] Giải thích cấu trúc PKI và các certificate do kubeadm tạo.
- [ ] Kiểm tra thời hạn bằng `kubeadm certs check-expiration`.
- [ ] Gia hạn certificate bằng `kubeadm certs renew` và khởi động lại component an toàn.
- [ ] Cấu hình và xử lý lỗi kubelet certificate rotation.
- [ ] Tạo, phê duyệt hoặc từ chối `CertificateSigningRequest`.
- [ ] Xoay vòng CA và phân phối lại kubeconfig/certificate khi cần.
- [ ] Chẩn đoán các lỗi do certificate hết hạn hoặc sai SAN.

Tài liệu tham chiếu: <https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/>

### D. Troubleshooting có hệ thống

- [ ] Xây dựng quy trình triage chung bằng Events, `kubectl describe`, log và trạng thái component.
- [ ] Debug Pod `Pending`, `CrashLoopBackOff`, `ImagePullBackOff`, OOMKilled và probe thất bại.
- [ ] Debug Deployment rollout, StatefulSet, DaemonSet, Job và scheduling failure.
- [ ] Debug Service không có endpoint, DNS/CoreDNS, NetworkPolicy, CNI và kube-proxy.
- [ ] Debug node `NotReady`, kubelet, container runtime và node pressure.
- [ ] Debug static Pod của API server, scheduler, controller-manager và etcd.
- [ ] Debug PVC `Pending`, lỗi attach/mount, StorageClass và CSI driver.
- [ ] Debug lỗi authentication, RBAC `Forbidden`, admission và certificate.
- [ ] Thực hành `kubectl debug`, `journalctl`, `systemctl`, `crictl`, `curl`, `openssl`, `ss`, `ip route` và `nsenter`.
- [ ] Chuẩn bị các lab cố ý làm hỏng cluster và runbook khôi phục tương ứng.

Tài liệu tham chiếu: <https://kubernetes.io/docs/tasks/debug/>

### E. Security vận hành

- [ ] Authentication và authorization căn bản, bao gồm luồng xử lý một API request.
- [ ] Thực hành `Role`, `ClusterRole`, `RoleBinding`, `ClusterRoleBinding` và `kubectl auth can-i`.
- [ ] Cấu hình audit policy, audit backend và chính sách lưu giữ audit log.
- [ ] Mã hóa Secret tại rest bằng `EncryptionConfiguration`.
- [ ] Cấu hình và vận hành KMS provider.
- [ ] Cấu hình kubelet authentication và authorization.
- [ ] Quản lý TLS/CSR cho user, node và component.
- [ ] Cấu hình admission controller và `ValidatingAdmissionPolicy`.
- [ ] Hardening API server, etcd và quyền truy cập host/control-plane node.

Tài liệu tham chiếu:

- <https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/>
- <https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/>

### F. Networking vận hành

- [ ] Cài đặt, nâng cấp và rollback ít nhất một CNI plugin cụ thể.
- [ ] Chẩn đoán CNI plugin, IPAM, routing, overlay, MTU và lỗi cấp IP cho Pod.
- [ ] Cấu hình CoreDNS/Corefile và kiểm tra luồng phân giải DNS end-to-end.
- [ ] Cài đặt, cấu hình và chẩn đoán NodeLocal DNSCache.
- [ ] Giải thích và kiểm tra các mode của kube-proxy: iptables, IPVS và nftables.
- [ ] Debug Service, EndpointSlice, NodePort, LoadBalancer, Ingress và Gateway.
- [ ] Thực hành NetworkPolicy với một network policy provider thực tế.
- [ ] Chẩn đoán SNAT, conntrack, firewall và route giữa Pod, Service, Node và mạng ngoài.

Tài liệu tham chiếu: <https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/>

### G. Observability và vận hành production

- [ ] Cài đặt và xác minh Metrics Server/resource metrics pipeline.
- [ ] Thu thập metric của node, control plane, kubelet, kube-proxy, CoreDNS và etcd.
- [ ] Xây dựng alert cho API latency/error, etcd, node pressure, Pod failure và certificate expiry.
- [ ] Triển khai log aggregation, log rotation, retention và truy vấn log tập trung.
- [ ] Thu thập và phân tích distributed trace khi phù hợp.
- [ ] Xây dựng dashboard, SLI/SLO và runbook liên kết với từng cảnh báo.
- [ ] Thực hành capacity planning cho CPU, memory, storage, Pod và API server.
- [ ] Kiểm thử backup/restore cho cả trạng thái cluster và dữ liệu workload.

Tài liệu tham chiếu: <https://kubernetes.io/docs/tasks/debug/>

### H. Công cụ quản lý cấu hình và package

- [ ] Quản lý manifest theo kiểu declarative với Kustomize, bao gồm base và overlay.
- [ ] Cài đặt, nâng cấp, rollback và kiểm tra release bằng Helm.
- [ ] Quản lý khác biệt cấu hình giữa dev, staging và production.
- [ ] Thiết lập quy trình kiểm tra manifest trước khi áp dụng vào cluster.

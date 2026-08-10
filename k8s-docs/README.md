# Tài liệu Kubernetes — Bản dịch tiếng Việt

Các bản dịch tiếng Việt của tài liệu chính thức trên <https://kubernetes.io/docs/>, giữ nguyên cấu trúc trang gốc. Mỗi file đều có link trang nguồn ở đầu. Tài liệu Kubernetes phát hành theo giấy phép CC BY 4.0.

> **Muốn học theo lộ trình?** Xem [LO-TRINH-ADMIN.md](LO-TRINH-ADMIN.md) — giáo trình 15 giai đoạn dành cho người muốn trở thành Kubernetes administrator, kèm mục tiêu và checkpoint thực hành cho từng giai đoạn, và phần tiếp nối sang nhánh `/docs/tasks/` để vận hành thực tế.
>
> **Muốn thực hành?** Xem [labs/](labs/README.md) — runbook chạy được cho từng nhóm bài, bắt đầu ở [Lab 00 — Môi trường](labs/LAB-00-MOI-TRUONG.md).
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

## Checkpoint — Những phần còn thiếu quan trọng

Phần này theo dõi các khoảng trống cần bổ sung để bộ tài liệu không chỉ bao phủ
Kubernetes Concepts mà còn đủ dùng cho việc quản trị cluster thực tế. Đánh dấu
`[x]` khi đã có tài liệu hướng dẫn đầy đủ và có bài thực hành kiểm chứng; việc một
khái niệm chỉ được nhắc đến hoặc trỏ sang tài liệu bên ngoài chưa được coi là hoàn tất.

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

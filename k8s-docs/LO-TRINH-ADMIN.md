# Lộ trình học Kubernetes Administrator

Giáo trình đọc **185 bài dịch** trong thư mục này theo thứ tự dành cho người muốn trở thành Kubernetes administrator.

> **Số trong tên file KHÔNG phải thứ tự đọc.** Số chỉ là mã định danh bám theo cấu trúc mục của kubernetes.io (để dễ đối chiếu khi trang gốc cập nhật). Thứ tự đọc là thứ tự các bài xuất hiện trong file này. Xem [README.md](README.md) nếu muốn tra cứu theo chủ đề thay vì theo lộ trình.

Mỗi giai đoạn có **Mục tiêu** (học xong phải trả lời được gì) và **Checkpoint** (phải làm được gì trước khi sang giai đoạn sau). Đọc mà không làm checkpoint thì kiến thức không trụ được.

Phần thực hành vận hành thực tế (upgrade, backup etcd, drain node, xử lý sự cố…) nằm ở [Checkpoint tiếp nối](#checkpoint-tiếp-nối--nhánh-docstasks) ở cuối file.

---

## Môi trường lab

Lộ trình này yêu cầu một cluster thật để thực hiện checkpoint ngay từ Giai đoạn 1.
Không sử dụng minikube. Chọn một trong hai phương án sau:

### Phương án 1 — Dùng cluster đã dựng

Có thể dùng cluster Kubernetes đã dựng trước đó nếu bạn có toàn quyền quản trị và
cluster không phục vụ production hoặc workload quan trọng. Cluster cần đáp ứng tối thiểu:

- Có ít nhất một control plane node và một worker node.
- Được dựng bằng kubeadm hoặc có kiến trúc đủ tương đồng để quan sát control plane,
  kubelet, container runtime, CNI, CoreDNS và kube-proxy.
- CNI đang hoạt động và có hỗ trợ NetworkPolicy để làm checkpoint Giai đoạn 5.
- Có StorageClass và dynamic provisioner hoạt động để làm checkpoint Giai đoạn 6.
- Có quyền SSH vào node, quyền `sudo` và kubeconfig có quyền quản trị cluster.
- Có thể chủ động cordon, drain, restart component, thay đổi cấu hình và khôi phục cluster.

Không thực hiện các bài phá lỗi, nâng cấp, restore etcd, xoay certificate hoặc thay đổi
control plane trên cluster đang phục vụ người dùng thật.

### Phương án 2 — Dựng một cluster lab khác tương tự

Nếu cluster hiện tại đang được sử dụng hoặc không được phép thử nghiệm phá lỗi, hãy dựng
một cluster riêng có kiến trúc tương tự. Nên dùng các máy ảo để có thể snapshot và khôi phục:

- Tối thiểu cho phần lớn bài học: 1 control plane + 2 worker.
- Cho bài HA: 3 control plane + 2 worker và một load balancer phía trước API server.
- Nếu học external etcd: bổ sung 3 node etcd riêng hoặc một nhóm VM tách biệt tương đương.
- Dùng cùng hệ điều hành, container runtime, CNI và dải mạng gần giống môi trường cần quản trị.
- Tạo snapshot VM trước các bài upgrade, restore etcd, certificate và troubleshooting phá lỗi.

Có thể tạo lại cluster nhiều lần trong quá trình học. Việc dựng, phá và khôi phục cluster
là một phần của bài thực hành, không phải thao tác chuẩn bị chỉ làm một lần.

### Quy ước an toàn

- Dùng namespace riêng cho bài tập workload thông thường.
- Ghi lại trạng thái ban đầu và sao lưu manifest/cấu hình trước khi chỉnh sửa.
- Chỉ làm bài gây gián đoạn trên cluster lab hoặc cluster có thể bỏ và dựng lại.
- Không đưa credential, private key, Secret hoặc snapshot etcd thật vào repository.
- Với bài làm đầy disk, chỉ dùng filesystem/volume dành riêng cho lab và đặt giới hạn rõ ràng;
  không làm đầy filesystem của máy host.

**Checkpoint môi trường:** chạy được `kubectl get nodes`, tất cả node ở trạng thái `Ready`;
SSH được vào từng node; xác định được container runtime, CNI, CoreDNS, kube-proxy và
StorageClass đang sử dụng; đồng thời có phương án snapshot hoặc dựng lại cluster khi bị hỏng.

---

## Giai đoạn 0 — Kiến thức nền ngoài thư mục này

Không có tài liệu trong thư mục. Thiếu phần này thì mọi giai đoạn sau đều học vẹt.

- **Linux**: systemd (unit, `systemctl`, `journalctl`), process, filesystem, permission, user/group.
- **Mạng**: TCP/IP, subnet, routing, DNS, NAT, firewall (iptables/nftables), load balancer L4 vs L7.
- **YAML và JSON**: cấu trúc lồng nhau, list vs map, multi-document (`---`).
- **TLS**: certificate, CA, chain of trust, SAN, thời hạn và gia hạn.
- **Công cụ**: `bash`, `curl`, `openssl`, `ss`, `ip`, `journalctl`, `systemctl`, `dig`.

**Checkpoint:** tự dựng được 2 máy Linux nối mạng với nhau, tự ký một certificate bằng `openssl` và giải thích được vì sao trình duyệt tin/không tin nó.

---

## Giai đoạn 1 — Mô hình Kubernetes

**Mục tiêu:** hiểu control plane gồm gì, API server đóng vai trò gì, object và desired state là gì, kubectl nói chuyện với cluster ra sao.

### 1a. Kiến trúc và mô hình điều khiển

1. [Tổng quan](14-overview-vi.md) — Kubernetes giải quyết bài toán gì.
2. [Các thành phần của Kubernetes](15-components-vi.md) — trọng tâm: phân biệt thành phần control plane và thành phần chạy trên mọi node.
3. [Kiến trúc cluster](22-architecture-vi.md) — ghép các thành phần ở bài 2 thành bức tranh tổng thể.
4. [Các đối tượng trong Kubernetes](16-working-with-objects-vi.md) — trọng tâm: `spec` (mong muốn) vs `status` (thực tế).
5. [Kubernetes API](21-kubernetes-api-vi.md) — trọng tâm: API group, versioning (alpha/beta/stable); phần API aggregation đọc lướt, sẽ quay lại ở giai đoạn 14.
6. [Các Node](23-nodes-vi.md) — trọng tâm: kubelet đăng ký node, node condition, heartbeat.
7. [Giao tiếp giữa Node và Control Plane](24-control-plane-node-communication-vi.md) — chiều giao tiếp nào đi qua API server, chiều nào không.
8. [Các Controller](25-controllers-vi.md) — vòng lặp điều khiển; đây là ý tưởng cốt lõi của toàn bộ Kubernetes.

### 1b. Làm việc với object và kubectl

9. [Tên và ID của đối tượng](17-names-vi.md) — quy tắc đặt tên DNS subdomain/label, UID.
10. [Label và Selector](18-labels-vi.md) — bài quan trọng nhất nhóm này; selector là cơ chế mọi controller và Service dùng để tìm Pod.
11. [Annotations](20-annotations-vi.md) — phân biệt rõ với label: annotation không dùng để chọn object.
12. [Namespaces](19-namespaces-vi.md) — trọng tâm: tài nguyên nào có namespace, tài nguyên nào cấp cluster.
13. [Các label được khuyến nghị](31-common-labels-vi.md) — quy ước `app.kubernetes.io/*`.
14. [Công cụ dòng lệnh kubectl](26-kubectl-vi.md) — cú pháp, các động từ chính.
15. [Tổ chức quyền truy cập cluster bằng file kubeconfig](111-kubeconfig-vi.md) — context, cluster, user; cần cho mọi thao tác về sau.
16. [Quản lý object trong Kubernetes](27-object-management-vi.md) — trọng tâm: khác biệt giữa imperative, declarative (`apply`) và khi nào dùng cái nào.
17. [Field selector](28-field-selectors-vi.md) — bổ sung cho label selector khi lọc theo trường.

### 1c. Vòng đời và cơ chế nền của object

18. [Finalizers](29-finalizers-vi.md) — vì sao một object xóa mãi không đi.
19. [Đối tượng sở hữu và đối tượng phụ thuộc](30-owners-dependents-vi.md) — owner reference.
20. [Thu gom rác](36-garbage-collection-vi.md) — xóa cascade foreground/background; dựa trên hai bài trên.
21. [Các Lease](35-leases-vi.md) — heartbeat của node và bầu leader của control plane.
22. [Các phiên bản lưu trữ](32-storage-version-vi.md) — object được lưu trong etcd ở version nào.
23. [Proxy phiên bản hỗn hợp](37-mixed-version-proxy-vi.md) — đọc lướt, chỉ cần biết tồn tại khi cluster có nhiều version apiserver.
24. [Cloud Controller Manager](34-cloud-controller-vi.md) — nếu chạy on-premise thì đọc để biết phần nào **không** có.

**Checkpoint:** giải thích được đường đi của `kubectl apply -f pod.yaml` từ lúc gõ lệnh đến khi container chạy, kể tên từng thành phần tham gia. Dùng `kubectl explain`, `kubectl get -o yaml`, label selector và `-n` thành thạo trên cluster lab đã chuẩn bị ở đầu lộ trình.

---

## Giai đoạn 2 — Container và runtime

**Mục tiêu:** hiểu tầng dưới Pod: image, runtime, CRI, cgroup — trước khi cấu hình runtime thật.

1. [Các Container](39-containers-vi.md)
2. [Các Image](40-images-vi.md) — trọng tâm: tag vs digest, `imagePullPolicy`, `imagePullSecrets`; đây là nguồn lỗi vận hành rất phổ biến.
3. [Môi trường Container](41-container-environment-vi.md)
4. [Các hook vòng đời của Container](42-container-lifecycle-hooks-vi.md) — `postStart`, `preStop`; `preStop` liên quan trực tiếp đến shutdown êm ở giai đoạn 3.
5. [Container Runtime Interface (CRI)](44-cri-vi.md) — hợp đồng giữa kubelet và runtime.
6. [Giới thiệu về cgroup v2](33-cgroups-vi.md) — nền tảng của mọi giới hạn tài nguyên học ở giai đoạn 3.
7. [Runtime Class](43-runtime-class-vi.md) — chọn runtime khác nhau cho từng workload.
8. [Các container runtime](00-container-runtimes-vi.md) — **đọc lý thuyết ở đây** (đặc biệt mục cgroup driver: kubelet và runtime phải khớp nhau). Phần cài đặt thực tế để dành làm cùng giai đoạn 8.

**Checkpoint:** trên một máy Linux, giải thích được `containerd` và `runc` khác nhau chỗ nào, kiểm tra được cgroup version của máy, và nói được hậu quả khi kubelet dùng `systemd` còn runtime dùng `cgroupfs`.

---

## Giai đoạn 3 — Pod và cấu hình

**Mục tiêu:** Pod là đơn vị nhỏ nhất — phải nắm vòng đời, probe, và cách cấp phát tài nguyên trước khi đụng tới controller.

### 3a. Pod và vòng đời

1. [Workload](45-workloads-vi.md)
2. [Pod](46-pods-vi.md)
3. [Vòng đời của Pod](47-pod-lifecycle-vi.md) — bài xương sống: phase, trạng thái container, `restartPolicy`, chấm dứt êm và `terminationGracePeriodSeconds`.
4. [Các Condition của Pod](48-pod-condition-vi.md)
5. [Các probe Liveness, Readiness và Startup](49-probes-vi.md) — trọng tâm: phân biệt ba loại; cấu hình sai liveness là nguyên nhân kinh điển của restart loop.
6. [Container khởi tạo](50-init-containers-vi.md)
7. [Các container sidecar](51-sidecar-containers-vi.md) — sidecar hiện là init container có `restartPolicy: Always`.
8. [Các container tạm thời](52-ephemeral-containers-vi.md) — công cụ debug, dùng nhiều ở giai đoạn xử lý sự cố.
9. [Không gian tên người dùng](55-user-namespaces-vi.md)
10. [Downward API](56-downward-api-vi.md)
11. [Cấu hình Pod nâng cao](60-advanced-pod-config-vi.md)

### 3b. Cấu hình và tài nguyên

12. [Cấu hình](107-configuration-vi.md)
13. [ConfigMap](108-configmap-vi.md)
14. [Secret](109-secret-vi.md) — trọng tâm: Secret **chỉ mã hóa base64**, không phải mã hóa thật; encryption at rest sẽ làm ở phần tasks.
15. [Quản lý tài nguyên cho Pod và Container](110-manage-resources-containers-vi.md) — **bài bắt buộc phải chắc**: `requests` quyết định lập lịch, `limits` quyết định giới hạn thực thi. Toàn bộ QoS, eviction và scheduling phía sau đều dựa vào bài này.
16. [Các lớp chất lượng dịch vụ của Pod](54-pod-qos-vi.md) — Guaranteed/Burstable/BestEffort suy ra trực tiếp từ requests và limits.
17. [Sự gián đoạn](53-disruptions-vi.md) — gián đoạn tự nguyện vs không tự nguyện, PodDisruptionBudget.
18. [Pod tĩnh](58-static-pods-vi.md) — kubelet tự quản; chính là cách control plane của kubeadm chạy, cần cho giai đoạn 8.

**Checkpoint:** viết tay một Pod manifest có init container, sidecar, readiness + liveness probe, requests/limits, mount ConfigMap và Secret. Cố ý đặt request vượt sức node để thấy Pod `Pending`, rồi đọc `kubectl describe` tìm lý do. Xác định QoS class của 3 Pod khác nhau chỉ bằng cách nhìn manifest.

---

## Giai đoạn 4 — Workload controller

**Mục tiêu:** hiểu các controller vận hành Pod thay bạn, và cơ chế rollout/rollback.

1. [Quản lý Workload — trang mục các controller](62-controllers-index-vi.md)
2. [ReplicaSet](64-replicaset-vi.md) — **đọc trước Deployment**, vì Deployment vận hành thông qua ReplicaSet.
3. [Deployment](63-deployment-vi.md) — bài dài nhất bộ tài liệu. Trọng tâm: rollout, rollback, chiến lược RollingUpdate/Recreate, `maxSurge`/`maxUnavailable`, revision history.
4. [StatefulSets](65-statefulset-vi.md) — trọng tâm: định danh ổn định, thứ tự khởi tạo, `volumeClaimTemplates` (phần volume sẽ rõ hơn sau giai đoạn 6).
5. [DaemonSet](66-daemonset-vi.md) — mô hình mọi node một Pod; CNI và log agent đều chạy kiểu này.
6. [Jobs](67-job-vi.md) — trọng tâm: `completions`, `parallelism`, `backoffLimit`; các mục Pod failure policy và Elastic Indexed Job đọc lướt lần đầu.
7. [Tự động dọn dẹp các Job đã hoàn thành](68-ttlafterfinished-vi.md)
8. [CronJob](69-cron-jobs-vi.md) — trọng tâm: `concurrencyPolicy`, `startingDeadlineSeconds`.
9. [Khả năng tự phục hồi của Kubernetes](38-self-healing-vi.md) — đọc ở đây (không phải giai đoạn 1) vì nội dung dựa trên Deployment, ReplicaSet, StatefulSet vừa học.
10. [Quản lý Workload — vận hành bằng kubectl](61-management-vi.md) — tổ chức manifest, `kubectl apply` theo nhóm, canary thủ công.
11. [Tự động co giãn Workload](71-autoscaling-vi.md)
12. [Tự động co giãn Pod theo chiều ngang](72-horizontal-pod-autoscale-vi.md) — cần metrics server; phần triển khai nằm ở giai đoạn 11.
13. [Tự động co giãn Pod theo chiều dọc](73-vertical-pod-autoscale-vi.md)

**Đọc như tài liệu lịch sử:** [ReplicationController](70-replicationcontroller-vi.md) — tiền thân của ReplicaSet, không dùng cho hệ thống mới. Chỉ cần biết nó tồn tại khi gặp cluster cũ.

**Checkpoint:** tạo Deployment 3 replica, thực hiện rolling update, theo dõi `kubectl rollout status`, rồi rollback về revision trước. Xóa thủ công một Pod và quan sát ReplicaSet tạo lại. Giải thích được vì sao StatefulSet không thể thay bằng Deployment cho database.

---

## Giai đoạn 5 — Mạng nền tảng

**Mục tiêu:** hiểu Pod nói chuyện với nhau và với bên ngoài thế nào. Service học trước DNS và Ingress.

1. [Service, cân bằng tải và mạng](81-services-networking-vi.md) — trọng tâm: mô hình mạng Kubernetes (mọi Pod thấy nhau không qua NAT).
2. [Service](82-service-vi.md) — bài quan trọng nhất giai đoạn. Trọng tâm: ClusterIP, NodePort, LoadBalancer, ExternalName, headless Service, và cơ chế virtual IP.
3. [EndpointSlices](83-endpoint-slices-vi.md) — Service tìm Pod qua đây.
4. [DNS cho Service và Pod](10-dns-pod-service-vi.md) — trọng tâm: dạng FQDN `svc.namespace.svc.cluster.local`, `search` domain và `ndots:5` trong `/etc/resolv.conf`.
5. [Hostname của Pod](57-pod-hostname-vi.md) — `hostname`/`subdomain`, dùng với headless Service.
6. [Chính sách mạng](84-network-policies-vi.md) — trọng tâm: mặc định là cho phép tất cả; policy chỉ có tác dụng khi CNI hỗ trợ.
7. [Định tuyến nhận biết topology](86-topology-aware-routing-vi.md)
8. [Chính sách lưu lượng nội bộ của Service](87-service-traffic-policy-vi.md)
9. [Cấp phát ClusterIP cho Service](88-cluster-ip-allocation-vi.md)
10. [Ingress](11-ingress-vi.md) — trọng tâm: rule, path type, IngressClass, TLS.
11. [Ingress Controllers](12-ingress-controllers-vi.md) — không có controller thì Ingress vô nghĩa.
12. [Gateway API](13-gateway-vi.md) — hướng thay thế Ingress; đọc để biết định hướng tương lai.
13. [Dual-stack IPv4/IPv6](85-dual-stack-vi.md)

### Tầng hạ tầng mạng của cluster

14. [Mạng trong cluster](157-networking-vi.md) — mô hình mạng ở góc nhìn quản trị và các cách hiện thực.
15. [Network Plugin](183-network-plugins-vi.md) — CNI; cần trước khi cài CNI thật ở giai đoạn 8.
16. [Các loại proxy trong Kubernetes](164-proxies-vi.md) — kube-proxy và các loại proxy khác.

**Checkpoint:** expose một Deployment bằng ClusterIP rồi NodePort; từ trong một Pod dùng `nslookup`/`curl` gọi Service bằng tên ngắn và FQDN. Viết một NetworkPolicy chặn toàn bộ ingress vào một namespace rồi mở đúng một cổng. Giải thích được khác biệt giữa Service, Ingress và load balancer bên ngoài.

---

## Giai đoạn 6 — Lưu trữ

**Mục tiêu:** cấp phát và quản lý dữ liệu bền vững. ConfigMap/Secret đã học ở giai đoạn 3 nên phần projected volume không còn phụ thuộc ngược.

1. [Lưu trữ](90-storage-vi.md)
2. [Các Volume](91-volumes-vi.md) — trọng tâm: `emptyDir`, `hostPath` (và rủi ro bảo mật của nó), `configMap`, `secret`, `persistentVolumeClaim`. Các driver in-tree đã bị loại bỏ chỉ cần đọc lướt.
3. [Volume bền vững](92-persistent-volumes-vi.md) — bài xương sống: vòng đời PV/PVC, binding, access mode, reclaim policy, mở rộng dung lượng.
4. [Lớp lưu trữ](96-storage-classes-vi.md) — trọng tâm: `provisioner`, `reclaimPolicy`, `volumeBindingMode`, default StorageClass.
5. [Cấp phát Volume động](98-dynamic-provisioning-vi.md)
6. [Volume dạng projected](93-projected-volumes-vi.md) — gộp ConfigMap, Secret, downwardAPI, service account token vào một mount.
7. [Volume tạm thời](94-ephemeral-volumes-vi.md)
8. [Lưu trữ tạm thời cục bộ](95-ephemeral-storage-vi.md) — liên quan trực tiếp tới eviction ở giai đoạn 7.
9. [Lớp thuộc tính Volume](97-volume-attributes-classes-vi.md)
10. [Ảnh chụp nhanh Volume](99-volume-snapshots-vi.md)
11. [Các lớp Volume Snapshot](100-volume-snapshot-classes-vi.md)
12. [Nhân bản CSI Volume](101-volume-pvc-datasource-vi.md)
13. [Volume Populator và Nguồn dữ liệu](102-volume-populators-vi.md)
14. [Dung lượng lưu trữ](103-storage-capacity-vi.md)
15. [Giới hạn volume theo từng Node](104-storage-limits-vi.md)
16. [Giám sát tình trạng volume](105-volume-health-monitoring-vi.md)

**Checkpoint:** tạo StorageClass, xin một PVC và mount vào Pod; xóa Pod rồi tạo lại, chứng minh dữ liệu còn nguyên. Thử `reclaimPolicy: Retain` và `Delete` để thấy khác biệt khi xóa PVC. Chạy một StatefulSet có `volumeClaimTemplates` và quan sát PVC sinh ra theo từng replica.

---

## Giai đoạn 7 — Lập lịch và chính sách tài nguyên

**Mục tiêu:** điều khiển Pod chạy ở đâu, và bảo vệ cluster khỏi workload ngốn tài nguyên.

### 7a. Scheduling và eviction

1. [Lập lịch, Preemption và Eviction](136-scheduling-eviction-vi.md)
2. [Bộ lập lịch của Kubernetes](137-kube-scheduler-vi.md) — chu trình filter rồi score.
3. [Gán Pod cho Node](138-assign-pod-node-vi.md) — bài dài, trọng tâm: `nodeSelector`, node affinity (required vs preferred), inter-pod affinity/anti-affinity.
4. [Taint và Toleration](139-taint-and-toleration-vi.md) — mặt đối ngẫu của affinity; control plane node bị taint chính là cơ chế này.
5. [Ràng buộc phân bố Pod theo topology](140-topology-spread-constraints-vi.md) — trải Pod đều theo zone/node.
6. [Độ ưu tiên và Preemption của Pod](141-pod-priority-preemption-vi.md) — PriorityClass.
7. [Eviction do áp lực node](142-node-pressure-eviction-vi.md) — trọng tâm: ngưỡng eviction, thứ tự trục xuất theo QoS.
8. [Eviction khởi phát qua API](143-api-eviction-vi.md) — cơ chế đứng sau `kubectl drain`.
9. [Pod Overhead](144-pod-overhead-vi.md)
10. [Mức sẵn sàng lập lịch của Pod](145-pod-scheduling-readiness-vi.md)
11. [Tinh chỉnh hiệu năng bộ lập lịch](146-scheduler-perf-tuning-vi.md) — đọc lướt, chỉ cần khi cluster rất lớn.
12. [Scheduling Framework](147-scheduling-framework-vi.md) — các điểm mở rộng của scheduler.
13. [Đóng gói tài nguyên](148-resource-bin-packing-vi.md)

### 7b. Chính sách giới hạn tài nguyên

14. [Chính sách](132-policies-vi.md)
15. [Khoảng giới hạn tài nguyên](133-limit-range-vi.md) — đặt mặc định và trần cho từng Pod/container trong namespace.
16. [Hạn ngạch tài nguyên](134-resource-quotas-vi.md) — trần tổng cho cả namespace; công cụ chính khi chia cluster cho nhiều nhóm.
17. [Giới hạn và dự trữ Process ID](135-pid-limiting-vi.md)
18. [Các trình quản lý tài nguyên](74-resource-managers-vi.md) — CPU manager, memory manager, topology manager của kubelet.
19. [Tính năng do Node khai báo](154-node-declared-features-vi.md) — đọc lướt.

**Checkpoint:** taint một node và chứng minh Pod thường không lên đó, rồi thêm toleration để lên được. Đặt ResourceQuota + LimitRange cho một namespace, thử tạo Pod vượt quota và đọc thông báo từ chối. Tạo tình huống node hết đĩa để quan sát eviction và thứ tự Pod bị trục xuất.

---

## Giai đoạn 8 — Dựng cluster bằng kubeadm

**Mục tiêu:** tự tay dựng cluster. Đến đây bạn đã hiểu component, Pod, Service, CNI và storage nên mỗi bước cài đặt đều có nghĩa, không phải gõ theo hướng dẫn.

1. [Cài đặt kubeadm](01-install-kubeadm-vi.md) — kèm phần thực hành cài container runtime ở [bài 00](00-container-runtimes-vi.md) đã đọc lý thuyết tại giai đoạn 2. Trọng tâm: port cần mở, swap, cgroup driver.
2. [Tạo một cluster với kubeadm](02-create-cluster-kubeadm-vi.md) — `kubeadm init`, cài CNI, join worker, taint control plane.
3. [Tùy chỉnh các thành phần với kubeadm API](03-control-plane-flags-vi.md) — `ClusterConfiguration` và patch.
4. [Cấu hình từng kubelet trong cluster bằng kubeadm](04-kubelet-integration-vi.md)
5. [Các lựa chọn topology cho tính sẵn sàng cao](06-ha-topology-vi.md) — stacked etcd vs external etcd; **quyết định trước khi dựng HA**.
6. [Thiết lập cluster etcd có tính sẵn sàng cao với kubeadm](07-setup-ha-etcd-with-kubeadm-vi.md) — chỉ cần nếu chọn external etcd; phải dựng xong trước bước sau.
7. [Tạo cluster có tính sẵn sàng cao với kubeadm](08-high-availability-vi.md) — cần load balancer đứng trước các API server.
8. [Hỗ trợ dual-stack với kubeadm](05-dual-stack-support-vi.md)
9. [Xử lý sự cố kubeadm](09-troubleshooting-kubeadm-vi.md) — **tài liệu tra cứu, không đọc tuần tự**. Đọc lướt mục lục một lần để biết có gì, rồi quay lại khi gặp lỗi.

**Checkpoint — bắt buộc làm thật, không xem video:**
- Dựng cluster 1 control plane + 2 worker, cài CNI, chạy được một Deployment có Service.
- Dựng lại thành HA 3 control plane với load balancer phía trước.
- Dựng một lần với stacked etcd, một lần với external etcd.
- Join thêm node, remove node đúng quy trình (drain trước).
- **Phá rồi khôi phục**: tắt một control plane node và chứng minh cluster vẫn phục vụ; xóa `/etc/kubernetes/manifests/kube-apiserver.yaml` rồi khôi phục.
- Đọc được `crictl ps`, `journalctl -u kubelet` khi node không lên `Ready`.

---

## Giai đoạn 9 — Bảo mật và multi-tenancy

**Mục tiêu:** kiểm soát ai làm được gì, và cô lập workload.

1. [Bảo mật](113-security-vi.md)
2. [Bảo mật Cloud Native và Kubernetes](114-cloud-native-security-vi.md) — mô hình 4C.
3. [Tài khoản dịch vụ](118-service-accounts-vi.md) — danh tính của Pod khi gọi API.
4. [Kiểm soát truy cập vào Kubernetes API](119-controlling-access-vi.md) — **bài xương sống**: authentication → authorization (RBAC) → admission control, đúng thứ tự ba chặng.
5. [Các thực hành tốt về kiểm soát truy cập dựa trên vai trò](120-rbac-good-practices-vi.md) — Role/ClusterRole, binding, nguyên tắc quyền tối thiểu.
6. [Chuẩn bảo mật Pod](115-pod-security-standards-vi.md) — ba profile Privileged/Baseline/Restricted.
7. [Cơ chế admission bảo mật Pod](116-pod-security-admission-vi.md) — áp ba profile trên vào namespace bằng label, chế độ enforce/audit/warn.
8. [Các thực hành tốt cho Kubernetes Secrets](121-secrets-good-practices-vi.md)
9. [Đa người thuê](122-multi-tenancy-vi.md) — ghép namespace + RBAC + quota + NetworkPolicy thành mô hình cô lập.
10. [Hướng dẫn tăng cường bảo mật — Các cơ chế xác thực](123-hardening-authentication-vi.md)
11. [Bảo mật cho node Linux](126-linux-security-vi.md)
12. [Các ràng buộc bảo mật của Linux kernel cho Pod và container](127-linux-kernel-security-vi.md) — capabilities, seccomp, AppArmor, SELinux.
13. [Rủi ro vượt qua Kubernetes API Server](128-api-server-bypass-risks-vi.md) — vì sao static pod và quyền truy cập kubelet nguy hiểm.
14. [Danh sách kiểm tra bảo mật](129-security-checklist-vi.md) — dùng như checklist đối chiếu cluster của bạn, không phải bài đọc.
15. [Danh sách kiểm tra bảo mật ứng dụng](130-application-security-checklist-vi.md)
16. [Ưu tiên và Công bằng cho API](166-flow-control-vi.md) — bảo vệ API server khỏi quá tải; FlowSchema và PriorityLevelConfiguration.
17. [Thực hành tốt cho Admission Webhook](173-admission-webhooks-vi.md) — webhook hỏng có thể làm chết cả cluster; đọc kỹ phần failure policy.

**Đọc như tài liệu lịch sử:** [Chính sách bảo mật Pod](117-pod-security-policy-vi.md) — PodSecurityPolicy đã bị gỡ khỏi Kubernetes, thay bằng bài 6–7 ở trên. Chỉ đọc khi tiếp quản cluster rất cũ.

**Để sau:** [Hướng dẫn tăng cường bảo mật — Cấu hình Scheduler](124-hardening-scheduler-vi.md) và [— Cấp phát tài nguyên động](125-hardening-dra-vi.md) nằm ở giai đoạn 13, sau khi đã học DRA và scheduler nâng cao.

**Checkpoint:** tạo một ServiceAccount + Role + RoleBinding cho phép một ứng dụng chỉ đọc được ConfigMap trong đúng một namespace, rồi tự kiểm chứng bằng `kubectl auth can-i --as=...`. Bật Pod Security Admission mức `restricted` cho một namespace và xem Pod đặc quyền bị từ chối. Rà cluster của mình theo checklist bài 14.

---

## Giai đoạn 10 — Vận hành day-2

Đây là phần **không có tài liệu trong thư mục này** vì toàn bộ nằm ở nhánh `/docs/tasks/` của kubernetes.io. Theo thiết kế lộ trình, bạn học hết lý thuyết (giai đoạn 11–15) rồi chuyển sang phần thực hành ở [Checkpoint tiếp nối](#checkpoint-tiếp-nối--nhánh-docstasks) cuối file.

Nếu đang cần vận hành gấp một cluster production, có thể nhảy tới phần đó ngay sau giai đoạn 9 — đặc biệt là ba nhóm: nâng cấp cluster, vòng đời chứng chỉ, và backup/restore etcd.

---

## Giai đoạn 11 — Observability

**Mục tiêu:** biết cluster đang khỏe hay ốm, và tìm được nguyên nhân.

1. [Khả năng quan sát](162-observability-vi.md) — đọc trước để có bức tranh chung ba trụ cột.
2. [Metric cho các thành phần hệ thống Kubernetes](160-system-metrics-vi.md) — endpoint `/metrics`, định dạng Prometheus.
3. [Metrics cho trạng thái đối tượng Kubernetes](163-kube-state-metrics-vi.md) — khác biệt với metric tài nguyên.
4. [Kiến trúc ghi log](158-logging-vi.md) — log ở mức container, node, cluster; mô hình sidecar và node agent.
5. [Log hệ thống](159-system-logs-vi.md) — log của kubelet và các thành phần control plane, mức verbosity.
6. [Trace cho các thành phần hệ thống Kubernetes](161-system-traces-vi.md)

**Checkpoint:** triển khai metrics-server và chạy được `kubectl top node`/`kubectl top pod` (đây cũng là điều kiện để HPA ở giai đoạn 4 hoạt động). Triển khai một stack Prometheus + Grafana, thu metric từ kubelet và kube-state-metrics, dựng một dashboard và một alert (ví dụ node NotReady hoặc Pod CrashLoopBackOff). Gom log tập trung bằng một agent chạy dạng DaemonSet.

---

## Giai đoạn 12 — Quản trị cluster nâng cao

**Mục tiêu:** các chủ đề vận hành ở tầng cluster.

1. [Quản trị cluster](155-cluster-administration-vi.md)
2. [Tắt node](169-node-shutdown-vi.md) — graceful node shutdown; quan trọng khi bảo trì phần cứng.
3. [Quản lý bộ nhớ swap](170-swap-memory-management-vi.md) — swap từng bị cấm hoàn toàn, nay có hỗ trợ; đọc kỹ nếu node của bạn bật swap.
4. [Tự động mở rộng Node](171-node-autoscaling-vi.md)
5. [Cài đặt các Add-on](165-addons-vi.md) — danh mục add-on theo nhóm chức năng.
6. [Phiên bản tương thích cho các thành phần Control Plane](168-compatibility-version-vi.md) — `--emulated-version`, hữu ích khi nâng cấp thận trọng.
7. [Bầu chọn leader có phối hợp](167-coordinated-leader-election-vi.md)

**Lưu ý:** [Chứng chỉ](156-certificates-vi.md) trong thư mục chỉ là **trang trỏ hướng 6 dòng**, không thay thế được module quản lý certificate — phần đó nằm ở checkpoint tasks cuối file.

**Checkpoint:** mô phỏng bảo trì một node: cordon → drain → tắt máy → bật lại → uncordon, và quan sát workload dịch chuyển. Kiểm tra graceful node shutdown có được kích hoạt không.

---

## Giai đoạn 13 — Lập lịch và workload nâng cao

**Không bắt buộc với admin mới.** Phần lớn là tính năng alpha/beta hoặc dành cho nền tảng chuyên biệt (AI/HPC, GPU). Đọc khi đã vững giai đoạn 1–12 hoặc khi công việc thực sự cần.

1. [Cấp phát tài nguyên động](149-dynamic-resource-allocation-vi.md) — DRA; cách cấp phát GPU và thiết bị chuyên dụng thế hệ mới.
2. [Các thực hành tốt về DRA dành cho quản trị viên cluster](172-cluster-admin-dra-vi.md)
3. [Hướng dẫn tăng cường bảo mật — Cấp phát tài nguyên động](125-hardening-dra-vi.md) — phần hoãn lại từ giai đoạn 9.
4. [Nhóm lập lịch](59-scheduling-group-vi.md)
5. [PodGroup API](75-podgroup-api-vi.md)
6. [Vòng đời của PodGroup](76-podgroup-lifecycle-vi.md)
7. [Workload API](77-workload-api-vi.md)
8. [Gián đoạn và độ ưu tiên của Pod Group](78-workload-disruption-priority-vi.md)
9. [Các chính sách lập lịch PodGroup](79-workload-policies-vi.md)
10. [Lập lịch workload nhận biết topology (Workload API)](80-workload-topology-scheduling-vi.md)
11. [Lập lịch theo nhóm](150-gang-scheduling-vi.md) — hoặc chạy hết cả nhóm, hoặc không chạy gì; thiết yếu cho huấn luyện phân tán.
12. [Lập lịch PodGroup](151-podgroup-scheduling-vi.md)
13. [Preemption nhận biết workload](152-workload-aware-preemption-vi.md)
14. [Lập lịch workload nhận biết topology (scheduling)](153-topology-aware-scheduling-vi.md)
15. [Hướng dẫn tăng cường bảo mật — Cấu hình Scheduler](124-hardening-scheduler-vi.md) — phần hoãn lại từ giai đoạn 9.

**Checkpoint:** nếu cluster có GPU, cấp phát một GPU cho Pod bằng DRA. Nếu không, chỉ cần giải thích được DRA khác device plugin truyền thống ở điểm nào.

---

## Giai đoạn 14 — Khả năng mở rộng

**Dành cho platform administrator / người phát triển operator.**

1. [Mở rộng Kubernetes](177-extend-kubernetes-vi.md) — bản đồ tất cả các điểm mở rộng.
2. [Mở rộng Kubernetes API](178-api-extension-vi.md)
3. [Tài nguyên tùy chỉnh](179-custom-resources-vi.md) — CRD; trọng tâm: bảng so sánh CRD với aggregated API để biết khi nào dùng cái nào.
4. [Tầng tổng hợp API của Kubernetes](180-apiserver-aggregation-vi.md)
5. [Mẫu Operator](181-operator-vi.md) — CRD + controller; mô hình vận hành ứng dụng stateful phức tạp.
6. [Các phần mở rộng về Tính toán, Lưu trữ và Mạng](182-compute-storage-net-vi.md)
7. [Device Plugin](184-device-plugins-vi.md) — cách cũ để expose GPU/thiết bị, so sánh với DRA ở giai đoạn 13.

**Đã đọc ở giai đoạn 5:** [Network Plugin](183-network-plugins-vi.md) — nếu cần xem lại trong ngữ cảnh mở rộng thì quay lại bài đó.

**Checkpoint:** tạo một CRD đơn giản, `kubectl apply` một custom resource và đọc lại bằng `kubectl get`. Giải thích được vì sao CRD không có controller thì chỉ là kho lưu dữ liệu.

---

## Giai đoạn 15 — Windows, nếu môi trường có node Windows

Bỏ qua hoàn toàn nếu cluster chỉ có Linux.

1. [Windows trong Kubernetes](174-windows-vi.md)
2. [Windows containers trong Kubernetes](175-windows-intro-vi.md) — trọng tâm: những gì **không** tương đương Linux.
3. [Hướng dẫn chạy Windows container trong Kubernetes](176-windows-user-guide-vi.md)
4. [Mạng trên Windows](89-windows-networking-vi.md)
5. [Lưu trữ trên Windows](106-windows-storage-vi.md)
6. [Quản lý tài nguyên cho các node Windows](112-windows-resource-management-vi.md)
7. [Bảo mật cho các node Windows](131-windows-security-vi.md)

**Checkpoint:** join một node Windows vào cluster và chạy được một workload Windows có Service.

---

## Checkpoint tiếp nối — nhánh `/docs/tasks/`

Học hết 15 giai đoạn trên là bạn có **nền lý thuyết**. Phần dưới là **thực hành vận hành** — kỹ năng thực sự phân biệt người biết Kubernetes với người vận hành được Kubernetes, và cũng là phần chiếm tỷ trọng lớn nhất trong kỳ thi CKA.

Các trang này **chưa được dịch** trong thư mục (thuộc nhánh `/docs/tasks/` của kubernetes.io, ~90 trang). Làm theo thứ tự checkpoint dưới đây, mỗi checkpoint làm trên cluster thật rồi mới sang checkpoint kế.

### CP1 — Vòng đời node

- [Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/) — cordon, drain, uncordon; liên hệ bài [53](53-disruptions-vi.md) và [143](143-api-eviction-vi.md).
- [Adding Linux nodes](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/adding-linux-nodes/)
- [Adding Windows nodes](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/adding-windows-nodes/)
- [Node Overprovisioning](https://kubernetes.io/docs/tasks/administer-cluster/node-overprovisioning/)

### CP2 — Nâng cấp cluster

- [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/) — quy trình chuẩn, liên hệ bảng version skew ở bài [02](02-create-cluster-kubeadm-vi.md).
- [Upgrading Linux nodes](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-linux-nodes/)
- [Upgrading Windows nodes](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-windows-nodes/)
- [Changing the Kubernetes package repository](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/)
- [Cluster Upgrade](https://kubernetes.io/docs/tasks/administer-cluster/cluster-upgrade/)

### CP3 — Vòng đời chứng chỉ

- [Certificate Management with kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/) — kiểm tra hạn, gia hạn, xoay CA.
- [PKI certificates and requirements](https://kubernetes.io/docs/setup/best-practices/certificates/)
- [Manage TLS Certificates in a Cluster](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/)

Đây là phần thay thế cho trang trỏ hướng [156](156-certificates-vi.md).

### CP4 — etcd, backup và khôi phục thảm họa

- [Operating etcd clusters for Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/) — snapshot `etcdctl snapshot save`, khôi phục, nâng cấp etcd.

**Bài tập bắt buộc:** backup etcd → cố ý xóa vài Deployment → restore từ snapshot → chứng minh cluster trở về trạng thái cũ. Không làm được bài này thì chưa nên vận hành production.

### CP5 — Cấu hình lại cluster đang chạy

- [Reconfiguring a kubeadm cluster](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-reconfigure/)
- [Configuring a cgroup driver](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/) — nối tiếp bài [00](00-container-runtimes-vi.md).
- [Set Kubelet Parameters Via A Configuration File](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/) — nối tiếp bài [04](04-kubelet-integration-vi.md).
- [Configure Feature Gates](https://kubernetes.io/docs/tasks/administer-cluster/configure-feature-gates/)
- [Enable Or Disable A Kubernetes API](https://kubernetes.io/docs/tasks/administer-cluster/enable-disable-api/)
- [Reserve Compute Resources for System Daemons](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/) — nối tiếp bài [110](110-manage-resources-containers-vi.md) và [142](142-node-pressure-eviction-vi.md).

### CP6 — DNS, CNI và kube-proxy

- [Customizing DNS Service](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/) — cấu hình CoreDNS, nối tiếp bài [10](10-dns-pod-service-vi.md).
- [Debugging DNS Resolution](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)
- [Using NodeLocal DNSCache](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/)
- [Autoscale the DNS Service](https://kubernetes.io/docs/tasks/administer-cluster/dns-horizontal-autoscaling/)
- [Declare Network Policy](https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/) — nối tiếp bài [84](84-network-policies-vi.md).
- [Network Policy Providers](https://kubernetes.io/docs/tasks/administer-cluster/network-policy-provider/) — Calico, Cilium, Antrea…
- [IP Masquerade Agent](https://kubernetes.io/docs/tasks/administer-cluster/ip-masq-agent/)

### CP7 — Audit và mã hóa dữ liệu

- [Auditing](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/) — audit policy và backend.
- [Encrypting Confidential Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) — phần còn thiếu của bài [109](109-secret-vi.md).
- [Decrypt Confidential Data that is Already Encrypted at Rest](https://kubernetes.io/docs/tasks/administer-cluster/decrypt-data/)
- [Using a KMS provider for data encryption](https://kubernetes.io/docs/tasks/administer-cluster/kms-provider/)
- [Securing a Cluster](https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/) — nối tiếp bài [129](129-security-checklist-vi.md).
- [Verify Signed Kubernetes Artifacts](https://kubernetes.io/docs/tasks/administer-cluster/verify-signed-artifacts/)

### CP8 — Giám sát và cảnh báo

- [Resource metrics pipeline](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/) — metrics-server, điều kiện cho HPA ở bài [72](72-horizontal-pod-autoscale-vi.md).
- [Tools for Monitoring Resources](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-usage-monitoring/)
- [Monitor Node Health](https://kubernetes.io/docs/tasks/debug/debug-cluster/monitor-node-health/) — node-problem-detector.

### CP9 — Xử lý sự cố

- [Troubleshooting Clusters](https://kubernetes.io/docs/tasks/debug/debug-cluster/) — nối tiếp bài [09](09-troubleshooting-kubeadm-vi.md).
- [Debugging Kubernetes nodes with crictl](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/) — công cụ thay `docker` khi debug node.
- [Debugging Kubernetes Nodes With Kubectl](https://kubernetes.io/docs/tasks/debug/debug-cluster/kubectl-node-debug/)
- [Troubleshooting kubectl](https://kubernetes.io/docs/tasks/debug/debug-cluster/troubleshoot-kubectl/)
- [Debug Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
- [Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/) — quy trình lần từ Service về Pod.
- [Debug Running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/) — dùng ephemeral container ở bài [52](52-ephemeral-containers-vi.md).
- [Determine the Reason for Pod Failure](https://kubernetes.io/docs/tasks/debug/debug-application/determine-reason-pod-failure/)
- [Debug Init Containers](https://kubernetes.io/docs/tasks/debug/debug-application/debug-init-containers/)
- [Debug a StatefulSet](https://kubernetes.io/docs/tasks/debug/debug-application/debug-statefulset/)

### CP10 — Quản trị tài nguyên theo namespace

- [Share a Cluster with Namespaces](https://kubernetes.io/docs/tasks/administer-cluster/namespaces/)
- [Manage Memory, CPU, and API Resources](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/) — loạt bài thực hành LimitRange và ResourceQuota, nối tiếp bài [133](133-limit-range-vi.md) và [134](134-resource-quotas-vi.md).
- [Configure Quotas for API Objects](https://kubernetes.io/docs/tasks/administer-cluster/quota-api-object/)
- [Control CPU Management Policies](https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/) — nối tiếp bài [74](74-resource-managers-vi.md).
- [Utilizing the NUMA-aware Memory Manager](https://kubernetes.io/docs/tasks/administer-cluster/memory-manager/)
- [Control Topology Management Policies](https://kubernetes.io/docs/tasks/administer-cluster/topology-manager/)
- [Advertise Extended Resources for a Node](https://kubernetes.io/docs/tasks/administer-cluster/extended-resource-node/)

### CP11 — Vận hành lưu trữ

- [Change the default StorageClass](https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/)
- [Change the Reclaim Policy of a PersistentVolume](https://kubernetes.io/docs/tasks/administer-cluster/change-pv-reclaim-policy/) — nối tiếp bài [92](92-persistent-volumes-vi.md).
- [Limit Storage Consumption](https://kubernetes.io/docs/tasks/administer-cluster/limit-storage-consumption/)

### CP12 — Di chuyển khỏi dockershim (cluster cũ)

Chỉ cần khi tiếp quản cluster đời cũ còn dùng Docker Engine:

- [Migrating from dockershim](https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/)
- [Find Out What Container Runtime is Used on a Node](https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/find-out-runtime-you-use/)
- [Changing the Container Runtime to containerd](https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/change-runtime-containerd/)
- [Troubleshooting CNI plugin-related errors](https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/troubleshooting-cni-plugin-related-errors/)

---

## Điều chỉnh so với bản phác thảo ban đầu

Bốn thay đổi có chủ đích so với thứ tự phác thảo, để không còn bài nào phải tham chiếu kiến thức của giai đoạn sau:

1. **Bài [38](38-self-healing-vi.md) (Tự phục hồi) chuyển từ giai đoạn 1 xuống giai đoạn 4.** Nội dung bài này nói về cách Deployment, ReplicaSet và StatefulSet phục hồi workload — đọc ở giai đoạn 1 sẽ gặp toàn khái niệm chưa học.
2. **Bài [172](172-cluster-admin-dra-vi.md) (DRA cho quản trị viên) chỉ đặt ở giai đoạn 13**, bỏ khỏi giai đoạn 12. Nội dung phụ thuộc hoàn toàn vào DRA ở bài 149.
3. **Bài [183](183-network-plugins-vi.md) (Network Plugin) đặt chính ở giai đoạn 5**, giai đoạn 14 chỉ tham chiếu lại. Cần hiểu CNI trước khi cài CNI ở giai đoạn 8.
4. **Bài [00](00-container-runtimes-vi.md) tách làm hai lần dùng**: đọc lý thuyết ở giai đoạn 2 (cùng CRI và cgroup), thực hành cài đặt ở giai đoạn 8 (cùng kubeadm) — tránh việc cài runtime xong bỏ máy không dùng suốt sáu giai đoạn.

Ngoài ra bài [70](70-replicationcontroller-vi.md) và [117](117-pod-security-policy-vi.md) được đánh dấu là tài liệu lịch sử, không nằm trong mạch bắt buộc.

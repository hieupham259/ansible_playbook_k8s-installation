# Lab thực hành cho lộ trình Kubernetes Administrator

Thư mục này chứa các bài lab đi kèm [00-ALO-TRINH-ADMIN.md](../00-ALO-TRINH-ADMIN.md). Mỗi lab là một
runbook chạy được trên cluster thật, có gate `PASS:` ở từng bước và checkpoint vấn đáp ở cuối.

Bắt đầu ở [Lab 00 — Môi trường](LAB-00-MOI-TRUONG-1.35.7.md). Không lab nào khác lặp lại phần dựng
môi trường; tất cả đều bắt đầu từ một snapshot có tên.

Lab 00 khóa baseline ở bảng A1.3 của chính nó, và gate `01-cluster-ready` gồm bảy tầng —
kiểm tới Pod xuyên node, DNS, Service/kube-proxy và `logs`/`exec`/`port-forward`. (Bản gốc
cũ của Lab 00 khóa baseline thấp hơn đã xoá 16/08/2026; xem lịch sử git nếu cần.)

---

## 1. Nguyên tắc chia lab

**Một lab = một nhóm bài trong lộ trình**, không phải một giai đoạn. Tách lab khi vi phạm
**một trong ba** điều sau:

1. **Trạng thái cluster đổi.** Lab cần hạ tầng mới (CNI khác, StorageClass, ingress
   controller, metrics-server, topology HA) thì phải là lab riêng, vì nó tạo snapshot mới.
2. **Thời lượng vượt 2–4 giờ.** Quá ngưỡng này thì không ai làm hết trong một phiên, và làm
   dở thì không có gate để quay lại.
3. **Bộ "kết quả phải đạt" không còn kiểm chứng độc lập được.** Khoảng 8–12 gạch đầu dòng
   checkpoint là giới hạn trên của thứ người học tự vấn đáp được một lần.

Giai đoạn nhỏ và đồng nhất thì một lab là đủ — giai đoạn 2, 12, 14 chỉ có một lab.

## 2. Nguyên tắc không nhảy cóc

Lab **không được** dùng khái niệm chưa dạy trong lộ trình để làm cho bài chạy được. Khi một
bài cần thứ thuộc giai đoạn sau, xử lý theo đúng thứ tự ưu tiên:

1. Thực hành **phần đạt được** bằng kiến thức đã có.
2. Ghi phần còn lại vào **sổ nợ lab** ở mục 5, kèm lab sẽ trả nợ.
3. Chỉ khi cả hai cách trên bất khả thi mới cân nhắc đổi thứ tự lộ trình.

Không cài trước hạ tầng của giai đoạn sau rồi im lặng dùng. Ví dụ: HPA đọc ở giai đoạn 4
nhưng cần metrics-server của giai đoạn 11, nên phần thực hành HPA nằm ở lab 11b — giai đoạn 4
không cài metrics-server sớm.

**Ngoại lệ duy nhất: [Lab 00](LAB-00-MOI-TRUONG-1.35.7.md).** Nó dùng kubeadm — nội dung của giai
đoạn 8 — vì lộ trình cần cluster thật ngay từ giai đoạn 1. Lab 00 được miễn trừ vì nó không
phải bài học: không có checkpoint vấn đáp, người học chạy copy-paste và được dặn rõ là chưa
cần hiểu. Phần học thật nằm ở Lab 8a. Không lab nào khác được viện dẫn ngoại lệ này.

## 3. Chuỗi snapshot

Mỗi lab khai báo ở đầu file: **điểm bắt đầu** (snapshot nào) và **điểm kết thúc** (trả cluster
về snapshot cũ, hay tạo snapshot mới).

| Snapshot | Tạo bởi | Nội dung thêm |
| --- | --- | --- |
| `01-cluster-ready` | [Lab 00](LAB-00-MOI-TRUONG-1.35.7.md) | 1 control plane + 2 worker, Flannel, không workload |
| `02-net-ready` | Lab 5b | CNI hỗ trợ NetworkPolicy thay Flannel, ingress controller |
| `03-storage-ready` | Lab 6a | StorageClass mặc định và provisioner |
| `04-metrics-ready` | Lab 11a | metrics-server |

Lab không tạo snapshot mới thì phải cleanup đưa cluster về đúng trạng thái đầu vào, và gate
cuối phải chứng minh điều đó.

Ngoài chuỗi chính còn hai nhánh phụ: hai lab HA của giai đoạn 8 dùng **bộ VM riêng** với mốc
tiền tố `8x-` (`8x-ha-stacked`, `8x-ha-external`), và lab 15 thêm một VM Windows vào chuỗi
chính rồi chụp `15-windows-ready`. Cả hai nhánh này không ảnh hưởng tới chuỗi chính.

## 4. Bản đồ lab

Cột **Bắt đầu từ** là snapshot phải khôi phục trước khi mở lab. Cột **Kết thúc** cho biết lab
đó có tạo mốc mới hay không: *trả về* nghĩa là cleanup đưa cluster về đúng snapshot đầu vào và
không snapshot lại; *tạo* nghĩa là lab thay đổi hạ tầng vĩnh viễn và phải chụp mốc mới trước
khi sang lab sau.

| Lab | Giai đoạn / nhóm bài | Bắt đầu từ | Kết thúc | Giờ | Trạng thái |
| --- | --- | --- | --- | --- | --- |
| [00 — Môi trường](LAB-00-MOI-TRUONG-1.35.7.md) | chuẩn bị | chưa có cluster | **tạo** `01-cluster-ready` | 2–4 | ✅ đã viết |
| [1a — Kiến trúc và mô hình điều khiển](LAB-1A-KIEN-TRUC-VA-MO-HINH-DIEU-KHIEN.md) | 1a (8 bài) | `01-cluster-ready` | trả về `01-cluster-ready` | 2–3 | ✅ đã viết |
| [1b — Object, label, kubectl và kubeconfig](LAB-1B-OBJECT-LABEL-KUBECTL-VA-KUBECONFIG.md) | 1b (9 bài) | `01-cluster-ready` | trả về `01-cluster-ready` | 3–4 | ✅ đã viết |
| [1c — Vòng đời và cơ chế nền của object](LAB-1C-VONG-DOI-VA-CO-CHE-NEN-CUA-OBJECT.md) | 1c (7 bài) | `01-cluster-ready` | trả về `01-cluster-ready` | 2–3 | ✅ đã viết |
| 2 — Container, image, CRI và cgroup | 2 (8 bài) | `01-cluster-ready` | trả về `01-cluster-ready` | 2–3 | ⬜ chưa viết |
| 3a — Pod và vòng đời | 3a (11 bài + 11 bài thực hành) | `01-cluster-ready` | trả về `01-cluster-ready` | 3–4 | ⬜ chưa viết |
| 3b — Cấu hình ứng dụng | 3b (3 bài + 11 bài thực hành) | `01-cluster-ready` | trả về `01-cluster-ready` | 3–4 | ⬜ chưa viết |
| 3c — Tài nguyên, QoS và gián đoạn | 3c (4 bài + 8 bài thực hành) | `01-cluster-ready` | trả về `01-cluster-ready` | 2–3 | ⬜ chưa viết |
| 4a — ReplicaSet, Deployment và rollout | 4a (6 bài + 4 bài thực hành) | `01-cluster-ready` | trả về `01-cluster-ready` | 2–3 | ⬜ chưa viết |
| 4b — StatefulSet, DaemonSet và Job | 4b (9 bài + 9 bài thực hành) | `01-cluster-ready` | trả về `01-cluster-ready` | 3–4 | ⬜ chưa viết |
| 5a — Service, EndpointSlice và DNS | 5 (phần Service/DNS) | `01-cluster-ready` | trả về `01-cluster-ready` | 3–4 | ⬜ chưa viết |
| 5b — NetworkPolicy, Ingress và CNI | 5 (phần policy/ingress) | `01-cluster-ready` | **tạo** `02-net-ready` | 3–4 | ⬜ chưa viết |
| 6a — PV, PVC và StorageClass | 6 (phần cốt lõi) | `02-net-ready` | **tạo** `03-storage-ready` | 3–4 | ⬜ chưa viết |
| 6b — Snapshot và volume nâng cao | 6 (phần còn lại) | `03-storage-ready` | trả về `03-storage-ready` | 2–3 | ⬜ chưa viết |
| 7a — Lập lịch và eviction | 7a (13 bài) | `03-storage-ready` | trả về `03-storage-ready` | 3–4 | ⬜ chưa viết |
| 7b — Quota và giới hạn tài nguyên | 7b (6 bài) | `03-storage-ready` | trả về `03-storage-ready` | 2–3 | ⬜ chưa viết |
| 8a — Dựng cluster bằng kubeadm | 8 (bài 01–05, 09) | `03-storage-ready` | phá cluster rồi **restore** `03-storage-ready` | 4 | ⬜ chưa viết |
| 8b — HA với stacked etcd | 8 (bài 06, 08) | bộ VM riêng, dựng mới | **tạo** `8x-ha-stacked` | 4 | ⬜ chưa viết |
| 8c — HA với external etcd | 8 (bài 06, 07, 08) | bộ VM riêng của lab 8b, reset | **tạo** `8x-ha-external` | 4 | ⬜ chưa viết |
| 9a — ServiceAccount, authn/authz và RBAC | 9 (phần truy cập) | `03-storage-ready` | trả về `03-storage-ready` | 3–4 | ⬜ chưa viết |
| 9b — Pod Security và hardening | 9 (phần policy) | `03-storage-ready` | trả về `03-storage-ready` | 3–4 | ⬜ chưa viết |
| 11a — Observability | 11 (6 bài) | `03-storage-ready` | **tạo** `04-metrics-ready` | 3–4 | ⬜ chưa viết |
| 11b — HPA và VPA | trả nợ giai đoạn 4 | `04-metrics-ready` | trả về `04-metrics-ready` | 2–3 | ⬜ chưa viết |
| 12 — Vận hành vòng đời node | 12 (8 bài) | `04-metrics-ready` | trả về `04-metrics-ready` | 2–3 | ⬜ chưa viết |
| 13 — DRA (tùy chọn) | 13 | `04-metrics-ready` | trả về `04-metrics-ready` | 2–3 | ⬜ chưa viết |
| 14 — CRD và Operator | 14 (7 bài) | `04-metrics-ready` | trả về `04-metrics-ready` | 2–3 | ⬜ chưa viết |
| 15 — Node Windows (tùy chọn) | 15 | `04-metrics-ready` + 1 VM Windows | **tạo** `15-windows-ready` | 4 | ⬜ chưa viết |

Trên chuỗi chính chỉ có **bốn lab tạo mốc mới**: 00, 5b, 6a và 11a. Hai lab HA (8b, 8c) dùng
bộ VM riêng với mốc tiền tố `8x-`, và lab 15 thêm một VM Windows vào chuỗi chính rồi chụp mốc
riêng. Toàn bộ lab còn lại trả cluster về mốc cũ, nên bạn không cần chụp snapshot sau mỗi bài.

Giai đoạn 10 không có lab riêng: toàn bộ nội dung nằm ở phần
[Checkpoint tiếp nối](../00-ALO-TRINH-ADMIN.md#checkpoint-tiếp-nối--nhánh-docstasks) và được thực
hành theo CP1–CP12 trên cluster đã có ở `04-metrics-ready`.

## 5. Sổ nợ lab

Bảng này ghi những phần **cố ý chưa làm** vì kiến thức cần cho nó thuộc giai đoạn sau. Lab
phát sinh nợ phải nói rõ trong phần checkpoint rằng nợ chưa trả; lab trả nợ phải nhắc lại
bài gốc.

| # | Nợ | Phát sinh ở | Vì cần | Trả ở |
| --- | --- | --- | --- | --- |
| 1 | Thực hành HPA và VPA | giai đoạn 4, bài [72](../72-horizontal-pod-autoscale-vi.md), [73](../73-vertical-pod-autoscale-vi.md) | metrics-server (giai đoạn 11) | Lab 11b |
| 2 | `volumeClaimTemplates` của StatefulSet | giai đoạn 4, bài [65](../65-statefulset-vi.md) | StorageClass + provisioner (giai đoạn 6) | Lab 6a |
| 3 | Service quản trị headless cho StatefulSet | giai đoạn 4, bài [65](../65-statefulset-vi.md) | Service headless (giai đoạn 5) | Lab 5a |
| 4 | NetworkPolicy được thực thi thật | giai đoạn 5, bài [84](../84-network-policies-vi.md) | CNI hỗ trợ policy thay Flannel | Lab 5b |
| 5 | Ảnh chụp nhanh và nhân bản volume | giai đoạn 6, bài [99](../99-volume-snapshots-vi.md)–[101](../101-volume-pvc-datasource-vi.md) | CSI driver có hỗ trợ snapshot | Lab 6b |
| 6 | Mã hóa Secret at rest | giai đoạn 3, bài [109](../109-secret-vi.md) | sửa cấu hình apiserver | [CP7](../00-ALO-TRINH-ADMIN.md#cp7--audit-và-mã-hóa-dữ-liệu) |
| 7 | Quản lý vòng đời certificate | giai đoạn 12, bài [156](../156-certificates-vi.md) | quy trình `kubeadm certs` | [CP3](../00-ALO-TRINH-ADMIN.md#cp3--vòng-đời-chứng-chỉ) |
| 8 | Backup và restore etcd | giai đoạn 8 | `etcdctl` và quy trình khôi phục | [CP4](../00-ALO-TRINH-ADMIN.md#cp4--etcd-backup-và-khôi-phục-thảm-họa) |
| 9 | Hai khối *Đọc bài này thế nào* và *Tự kiểm tra* cho 135 bài nhánh `/docs/tasks/` | mọi mục có dấu ⏳ trong lộ trình | công sức viết, không phải kiến thức | trả tại chỗ — xem [hướng dẫn trả nợ #9](../00-ALO-TRINH-ADMIN.md#nợ-9--hai-khối-hướng-dẫn-đọc-cho-nhánh-docstasks) |

Số hiệu nợ ở cột đầu **khớp với** [Sổ nợ lộ trình](../00-ALO-TRINH-ADMIN.md#sổ-nợ-lộ-trình). Trong file lộ trình, mỗi món nợ được đánh dấu ngay tại chỗ: `⏳ Nợ #N` ở nơi phát sinh và `✅ Trả nợ #N` ở nơi trả. Sửa một bảng thì phải sửa bảng kia.

## 6. Quy ước chung trong mọi lab

- Bằng chứng ghi vào `~/lab-evidence/<mã lab>/`, ví dụ `~/lab-evidence/1a/`.
- Fault injection chỉ chạy trên `lab-k8s-worker2`.
- Dòng bắt đầu bằng `PASS:` là điều kiện phải đạt; không đi tiếp khi gate fail.
- Số phiên bản chỉ tồn tại ở [bảng A1.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa);
  lab khác link về đó thay vì chép lại con số.
- Checkpoint là vấn đáp không nhìn tài liệu, không phải danh sách lệnh đã chạy.

---

## 7. Bài `/docs/tasks/` mỗi lab phải phủ

Nhánh `/docs/tasks/` của kubernetes.io là **bài tập có lời giải sẵn**. Trong repo này chúng
**không** phải một luồng thực hành song song: chỉ lab mới có snapshot, gate `PASS:`, thư mục
bằng chứng và giới hạn fault injection ở `k8s-worker2`. Vì vậy mỗi bài task được gắn vào
**đúng một lab**, và lab đó phải hấp thụ nội dung của nó.

**Cách gán:** lấy mốc muộn hơn giữa (a) lab của nhóm bài mà task đó thực hành, và (b) lab sớm
nhất mà **mọi object xuất hiện trong ví dụ của task** đã được dạy. Nhờ (b) mà không lab nào
phải dùng Deployment, Service hay ConfigMap trước khi lộ trình dạy chúng — ví dụ
[Kustomize](../322-kustomization-vi.md) thực hành khái niệm của giai đoạn 1b nhưng ví dụ dùng
Deployment, ConfigMap và Service nên chỉ khả thi từ Lab 5a.

Cột **Nợ #9** là số bài chưa có hai khối hướng dẫn đọc — viết khi mở bài đó, xem
[cách trả nợ #9](../00-ALO-TRINH-ADMIN.md#nợ-9--hai-khối-hướng-dẫn-đọc-cho-nhánh-docstasks).

| Lab | Số bài | Nợ #9 | Danh sách |
| --- | ---: | ---: | --- |
| Lab 1b | 7 | 4 | [Cài đặt và thiết lập kubectl trên Linux](../186-install-kubectl-linux-vi.md) · [Cài đặt và thiết lập kubectl trên macOS](../187-install-kubectl-macos-vi.md) · [Cài đặt và thiết lập kubectl trên Windows](../188-install-kubectl-windows-vi.md) · [Quản lý các đối tượng Kubernetes](../318-manage-kubernetes-objects-vi.md) · [Quản lý object Kubernetes theo kiểu imperative bằng file cấu hình](../321-imperative-config-vi.md) · [Cấu hình truy cập nhiều cluster](../361-configure-access-multiple-clusters-vi.md) · [Liệt kê tất cả Container image đang chạy trong Cluster](../365-list-running-container-images-vi.md) |
| Lab 1c | 1 | 0 | [Phát triển Cloud Controller Manager](../203-developing-cloud-controller-manager-vi.md) |
| Lab 2 | 2 | 2 | [Cấu hình một kubelet image credential provider](../225-kubelet-credential-provider-vi.md) · [Chuyển từ polling sang cập nhật trạng thái container dựa trên sự kiện CRI](../257-switch-to-evented-pleg-vi.md) |
| Lab 3a | 12 | 12 | [Chạy các thành phần Node của Kubernetes dưới người dùng không phải root](../226-kubelet-in-userns-vi.md) · [Cấu hình Pod và Container](../262-configure-pod-container-vi.md) · [Gắn handler vào các sự kiện vòng đời của Container](../272-attach-handler-lifecycle-event-vi.md) · [Cấu hình các probe Liveness, Readiness và Startup](../274-configure-probes-vi.md) · [Cấu hình khởi tạo Pod](../276-configure-pod-initialization-vi.md) · [Sử dụng Image Volume với một Pod](../285-image-volumes-vi.md) · [Chia sẻ Process Namespace giữa các Container trong một Pod](../292-share-process-namespace-vi.md) · [Tạo static Pod](../293-static-pod-tasks-vi.md) · [Sử dụng user namespace với Pod](../295-user-namespaces-tasks-vi.md) · [Expose thông tin Pod cho container thông qua file](../335-downward-api-volume-vi.md) · [Expose thông tin Pod cho container thông qua biến môi trường](../336-env-variable-expose-pod-info-vi.md) · [Giao tiếp giữa các Container trong cùng Pod bằng Volume dùng chung](../360-containers-shared-volume-vi.md) |
| Lab 3b | 12 | 12 | [Cấu hình một Pod để sử dụng ConfigMap](../275-configure-pod-configmap-vi.md) · [Pull image từ một private registry](../287-pull-image-private-registry-vi.md) · [Quản lý Secret](../325-configmap-secret-vi.md) · [Quản lý Secret bằng file cấu hình](../326-secret-config-file-vi.md) · [Quản lý Secret bằng kubectl](../327-secret-kubectl-vi.md) · [Quản lý Secret bằng Kustomize](../328-secret-kustomize-vi.md) · [Đưa dữ liệu vào ứng dụng](../329-inject-data-application-vi.md) · [Định nghĩa command và argument cho container](../330-define-command-argument-vi.md) · [Định nghĩa biến môi trường cho một Container](../331-define-environment-variable-vi.md) · [Định nghĩa giá trị biến môi trường bằng một Init Container](../332-define-env-via-file-vi.md) · [Định nghĩa các biến môi trường phụ thuộc](../333-interdependent-env-variables-vi.md) · [Phân phối thông tin xác thực một cách an toàn bằng Secret](../334-distribute-credentials-secure-vi.md) |
| Lab 3c | 8 | 8 | [Gán tài nguyên CPU cho Container và Pod](../263-assign-cpu-resource-vi.md) · [Gán tài nguyên memory cho Container và Pod](../264-assign-memory-resource-vi.md) · [Gán tài nguyên CPU và memory ở cấp Pod](../265-assign-pod-level-resources-vi.md) · [Gán Extended Resource cho một Container](../284-extended-resource-vi.md) · [Cấu hình Quality of Service cho Pod](../288-quality-service-pod-vi.md) · [Thay đổi kích thước tài nguyên CPU và Memory được gán cho Container](../289-resize-container-resources-vi.md) · [Thay đổi kích thước tài nguyên CPU và Memory được gán cho Pod](../290-resize-pod-resources-vi.md) · [Chỉ định Disruption Budget cho ứng dụng của bạn](../339-configure-pdb-vi.md) |
| Lab 4a | 7 | 7 | [Sử dụng xóa theo tầng trong Cluster](../260-use-cascading-deletion-vi.md) · [Quản lý object Kubernetes theo kiểu khai báo bằng file cấu hình](../319-declarative-config-vi.md) · [Cập nhật đối tượng API tại chỗ bằng kubectl patch](../324-kubectl-patch-vi.md) · [Chạy ứng dụng](../337-run-application-vi.md) · [Chạy một ứng dụng Stateless bằng Deployment](../345-run-stateless-application-vi.md) · [Scale thủ công theo chiều ngang cho một Deployment](../346-scale-deployment-vi.md) · [Cập nhật một Deployment mà không gây gián đoạn](../348-update-deployment-rolling-vi.md) |
| Lab 4b | 7 | 7 | [Xóa cưỡng bức Pod của StatefulSet](../341-force-delete-stateful-set-pod-vi.md) · [Scale một StatefulSet](../347-scale-stateful-set-vi.md) · [Chạy Job](../349-job-tasks-vi.md) · [Chạy các tác vụ tự động với CronJob](../350-automated-tasks-cron-jobs-vi.md) · [Xử lý song song thô sử dụng hàng đợi công việc](../351-coarse-parallel-work-queue-vi.md) · [Indexed Job để xử lý song song với phân công việc tĩnh](../353-indexed-parallel-processing-vi.md) · [Xử lý song song bằng cách khai triển template](../355-parallel-processing-expansion-vi.md) |
| Lab 5a | 10 | 9 | [Chuyển đổi file Docker Compose thành tài nguyên Kubernetes](../294-translate-compose-kubernetes-vi.md) · [Quản lý object Kubernetes bằng lệnh imperative](../320-imperative-command-vi.md) · [Quản lý object Kubernetes theo kiểu khai báo bằng Kustomize](../322-kustomization-vi.md) · [Xóa một StatefulSet](../340-delete-stateful-set-vi.md) · [Xử lý song song mịn sử dụng hàng đợi công việc](../352-fine-parallel-work-queue-vi.md) · [Job với giao tiếp Pod-đến-Pod](../354-job-pod-to-pod-communication-vi.md) · [Cấu hình DNS cho một cluster](../362-configure-dns-cluster-vi.md) · [Kết nối Frontend với Backend bằng Service](../363-connecting-frontend-backend-vi.md) · [Tạo bộ cân bằng tải bên ngoài](../364-create-external-load-balancer-vi.md) · [Sử dụng Port Forwarding để truy cập ứng dụng trong Cluster](../366-port-forward-vi.md) |
| Lab 6a | 4 | 4 | [Cấu hình Pod sử dụng projected Volume cho lưu trữ](../277-configure-projected-volume-vi.md) · [Cấu hình Pod sử dụng Volume để lưu trữ](../280-configure-volume-storage-vi.md) · [Chạy ứng dụng có trạng thái được nhân bản](../343-run-replicated-stateful-application-vi.md) · [Chạy ứng dụng có trạng thái đơn thực thể](../344-run-single-instance-stateful-vi.md) |
| Lab 7a | 3 | 3 | [Bảo đảm lập lịch cho các Pod add-on quan trọng](../210-guaranteed-scheduling-critical-addon-pods-vi.md) · [Gán Pod vào Node bằng Node Affinity](../266-assign-pods-nodes-node-affinity-vi.md) · [Gán Pod vào Node](../267-assign-pods-nodes-vi.md) |
| Lab 8a | 1 | 1 | [Quản trị với kubeadm](../214-kubeadm-tasks-vi.md) |
| Lab 9a | 9 | 9 | [Quản trị Cloud Controller Manager](../254-running-cloud-controller-vi.md) · [Sử dụng sysctl trong một cluster Kubernetes](../258-sysctl-cluster-vi.md) · [Cấu hình Service Account cho Pod](../279-configure-service-account-vi.md) · [Thực thi Pod Security Standards bằng cách cấu hình Admission Controller tích hợp sẵn](../282-enforce-standards-admission-controller-vi.md) · [Thực thi Pod Security Standards bằng nhãn Namespace](../283-enforce-standards-namespace-labels-vi.md) · [Di chuyển từ PodSecurityPolicy sang PodSecurity Admission Controller tích hợp sẵn](../286-migrate-from-psp-vi.md) · [Cấu hình Security Context cho Pod hoặc Container](../291-security-context-vi.md) · [Truy cập Kubernetes API từ một Pod](../338-access-api-from-pod-vi.md) · [Truy cập cluster](../359-access-cluster-vi.md) |
| Lab 11a | 7 | 6 | [Giám sát, ghi log và gỡ lỗi](../296-debug-vi.md) · [Xử lý sự cố ứng dụng](../297-debug-application-vi.md) · [Truy cập shell của một container đang chạy](../304-get-shell-running-container-vi.md) · [Phát triển và debug service cục bộ bằng telepresence](../309-local-debugging-vi.md) · [Ghi log trong Kubernetes](../316-debug-logging-vi.md) · [Giám sát trong Kubernetes](../317-debug-monitoring-vi.md) · [Hướng dẫn từng bước về HorizontalPodAutoscaler](../342-hpa-walkthrough-vi.md) |
| Lab 12 | 3 | 2 | [Quản trị một Cluster](../189-administer-cluster-vi.md) · [Truy cập cluster bằng Kubernetes API](../190-access-cluster-api-vi.md) · [Di chuyển control plane được nhân bản sang dùng Cloud Controller Manager](../198-controller-manager-leader-migration-vi.md) |
| Lab 13 | 5 | 5 | [Tăng cường bảo mật cho Cấp phát tài nguyên động trong cluster của bạn](../211-hardening-dra-tasks-vi.md) · [Gán thiết bị cho Pod và Container](../268-assign-resources-vi.md) · [Truy cập metadata thiết bị DRA](../269-access-dra-device-metadata-vi.md) · [Cấp phát thiết bị cho workload bằng DRA](../270-allocate-devices-dra-vi.md) · [Thiết lập DRA trong một cluster](../271-set-up-dra-cluster-vi.md) |
| Lab 14 | 2 | 0 | [Khắc phục sự cố Topology Management](../313-debug-topology-vi.md) · [Di trú object Kubernetes bằng Storage Version Migration](../323-storage-version-migration-vi.md) |
| Lab 15 | 4 | 3 | [Cấu hình GMSA cho Pod và container Windows](../273-configure-gmsa-vi.md) · [Cấu hình RunAsUserName cho Pod và container Windows](../278-configure-runasusername-vi.md) · [Tạo một Windows HostProcess Pod](../281-create-hostprocess-pod-vi.md) · [Mẹo debug Windows](../315-debug-windows-vi.md) |

48 bài task còn lại thuộc nhóm vận hành cluster, nằm ở
[CP1–CP12](../00-ALO-TRINH-ADMIN.md#checkpoint-tiếp-nối--nhánh-docstasks) chứ không thuộc lab nào.

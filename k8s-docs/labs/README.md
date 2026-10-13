# Lab thực hành cho lộ trình Kubernetes Administrator

Thư mục này chứa các bài lab đi kèm [00-ALO-TRINH-ADMIN.md](../00-ALO-TRINH-ADMIN.md). Mỗi lab chạy
trên cluster thật, có gate `PASS:` và checkpoint vấn đáp ở cuối.

Các lab có hai dạng:

- **Lab 00 → Lab 2 và Lab 15** là runbook: đưa sẵn lệnh, gate sau từng bước.
- **Lab 3a → Lab 14** là các **sprint** của [dự án Bookstore](DU-AN-BOOKSTORE.md) — đưa một ứng dụng
  ba tầng (nginx → PostgREST → PostgreSQL) lên Kubernetes và hoàn thiện nó qua từng nhóm bài. Lab
  giao **ticket** (yêu cầu, ràng buộc, mục tài liệu cần đọc lại, tiêu chí nghiệm thu), người học tự
  viết manifest và tự chứng minh bằng script nghiệm thu; gợi ý và lời giải nằm trong khối gập. Mục
  đích và cách thiết kế của nhóm lab này ở
  [mục 7](#7-lab-dạng-dự-án-3a--14-mục-đích-và-cách-thiết-kế); đọc mục đó và
  [file dự án](DU-AN-BOOKSTORE.md) trước khi mở Lab 3a.

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
2. **Thời lượng vượt 2–4 giờ cho một phiên.** Quá ngưỡng này thì không ai làm hết trong một
   phiên, và làm dở thì không có gate để quay lại. Lab dạng dự án (3a → 14) được phép **hai
   phiên**, cả lab 4–6 giờ, vì giữa hai phiên có gate quay lại được định nghĩa ở
   [mục 7.5](#75-cách-làm-một-ticket). Cần quá hai phiên thì tách lab.
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
| [1c — Vòng đời và cơ chế nền của object](LAB-1C-VONG-DOI-VA-CO-CHE-NEN-CUA-OBJECT.md) | 1c (7 bài + 1 bài thực hành) | `01-cluster-ready` | trả về `01-cluster-ready` | 2–3 | ✅ đã viết |
| [2 — Container, image, CRI và cgroup](LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md) | 2 (8 bài + 2 bài thực hành) | `01-cluster-ready` | trả về `01-cluster-ready` | 2–3 | ✅ đã viết |
| [3a — Pod và vòng đời](LAB-3A-POD-VA-VONG-DOI.md) · Sprint 1 | 3a (11 bài + 11 bài thực hành) | `01-cluster-ready` | trả về `01-cluster-ready` | 4–6 | ✅ đã viết |
| [3b — Cấu hình ứng dụng](LAB-3B-CAU-HINH-UNG-DUNG.md) · Sprint 2 | 3b (3 bài + 11 bài thực hành) | `01-cluster-ready` | trả về `01-cluster-ready` | 4–6 | ✅ đã viết |
| [3c — Tài nguyên, QoS và gián đoạn](LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md) · Sprint 3 | 3c (4 bài + 9 bài thực hành) | `01-cluster-ready` | trả về `01-cluster-ready` | 4–5 | ✅ đã viết |
| [4a — ReplicaSet, Deployment và rollout](LAB-4A-REPLICASET-DEPLOYMENT-VA-ROLLOUT.md) · Sprint 4 | 4a (5 bài + 7 bài thực hành) | `01-cluster-ready` | trả về `01-cluster-ready` | 4–5 | ✅ đã viết |
| [4b — StatefulSet, DaemonSet và Job](LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) · Sprint 5 | 4b (9 bài + 9 bài thực hành) | `01-cluster-ready` | trả về `01-cluster-ready` | 4–5 | ✅ đã viết |
| [5a — Service, EndpointSlice và DNS](LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md) · Sprint 6 | 5 (phần Service/DNS) | `01-cluster-ready` | trả về `01-cluster-ready` | 4–5 | ✅ đã viết |
| [5b — NetworkPolicy, Ingress và CNI](LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) · Sprint 7 | 5 (phần policy/ingress) | `01-cluster-ready` | **tạo** `02-net-ready` | 4–5 | ✅ đã viết |
| [6a — PV, PVC và StorageClass](LAB-6A-PV-PVC-VA-STORAGECLASS.md) · Sprint 8 | 6 (phần cốt lõi) | `02-net-ready` | **tạo** `03-storage-ready` | 4–5 | ✅ đã viết |
| [6b — Snapshot và volume nâng cao](LAB-6B-SNAPSHOT-VA-VOLUME-NANG-CAO.md) · Sprint 9 | 6 (phần còn lại) | `03-storage-ready` | trả về `03-storage-ready` | 4–5 | ✅ đã viết |
| [7a — Lập lịch và eviction](LAB-7A-LAP-LICH-VA-EVICTION.md) · Sprint 10 | 7a (13 bài) | `03-storage-ready` | trả về `03-storage-ready` | 4–5 | ✅ đã viết |
| [7b — Quota và giới hạn tài nguyên](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md) · Sprint 11 | 7b (6 bài) | `03-storage-ready` | trả về `03-storage-ready` | 4–5 | ✅ đã viết |
| [8a — Dựng cluster bằng kubeadm](LAB-8A-DUNG-CLUSTER-BANG-KUBEADM.md) · Sprint 12 | 8 (bài 01–05, 09) | `03-storage-ready` | phá cluster rồi **restore** `03-storage-ready` | 5–6 | ✅ đã viết |
| [8b — HA với stacked etcd](LAB-8B-HA-VOI-STACKED-ETCD.md) · Sprint 13 | 8 (bài 06, 08) | bộ VM riêng, dựng mới | **tạo** `8x-ha-stacked` | 4–5, chưa kể dựng VM | ✅ đã viết |
| [8c — HA với external etcd](LAB-8C-HA-VOI-EXTERNAL-ETCD.md) · Sprint 14 | 8 (bài 06, 07, 08) | bộ VM riêng của lab 8b, reset | **tạo** `8x-ha-external` | 4–5 | ✅ đã viết |
| [9a — ServiceAccount, authn/authz và RBAC](LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md) · Sprint 15 | 9 (phần truy cập) | `03-storage-ready` | trả về `03-storage-ready` | 4–5 | ✅ đã viết |
| [9b — Pod Security và hardening](LAB-9B-POD-SECURITY-VA-HARDENING.md) · Sprint 16 | 9 (phần policy) | `03-storage-ready` | trả về `03-storage-ready` | 4–5 | ✅ đã viết |
| [11a — Observability](LAB-11A-OBSERVABILITY.md) · Sprint 17 | 11 (6 bài) | `03-storage-ready` | **tạo** `04-metrics-ready` | 4–6 | ✅ đã viết |
| [11b — HPA và VPA](LAB-11B-HPA-VA-VPA.md) · Sprint 18 | trả nợ giai đoạn 4 | `04-metrics-ready` | trả về `04-metrics-ready` | 4–5 | ✅ đã viết |
| [12 — Vận hành vòng đời node](LAB-12-VAN-HANH-VONG-DOI-NODE.md) · Sprint 19 | 12 (8 bài) | `04-metrics-ready` | trả về `04-metrics-ready` | 4–5 | ✅ đã viết |
| [13 — DRA (tùy chọn)](LAB-13-DRA.md) · Sprint 20 | 13 | `04-metrics-ready` | trả về `04-metrics-ready` | 4–5 | ✅ đã viết |
| [14 — CRD và Operator](LAB-14-CRD-VA-OPERATOR.md) · Sprint 21 | 14 (7 bài) | `04-metrics-ready` | trả về `04-metrics-ready` | 4–5 | ✅ đã viết |
| [15 — Node Windows (tùy chọn)](LAB-15-NODE-WINDOWS.md) | 15 | `04-metrics-ready` (nhánh thực hành cần thêm 1 VM Windows) | nhánh A **tạo** `15-windows-ready`; nhánh B **trả về** `04-metrics-ready` | 4 | ✅ đã viết |

Trên chuỗi chính chỉ có **bốn lab tạo mốc mới**: 00, 5b, 6a và 11a. Hai lab HA (8b, 8c) dùng
bộ VM riêng với mốc tiền tố `8x-`, và lab 15 thêm một VM Windows vào chuỗi chính rồi chụp mốc
riêng. Toàn bộ lab còn lại trả cluster về mốc cũ, nên bạn không cần chụp snapshot sau mỗi bài.

Giai đoạn 10 không có lab riêng: toàn bộ nội dung nằm ở phần
[Checkpoint tiếp nối](../00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster) và được thực
hành theo giai đoạn 16–27 trên cluster đã có ở `04-metrics-ready`.

## 5. Sổ nợ lab

Bảng này ghi những phần **cố ý chưa làm** vì kiến thức cần cho nó thuộc giai đoạn sau. Lab
phát sinh nợ phải nói rõ trong phần checkpoint rằng nợ chưa trả; lab trả nợ phải nhắc lại
bài gốc.

| # | Nợ | Phát sinh ở | Vì cần | Trả ở |
| --- | --- | --- | --- | --- |
| 1 | Thực hành HPA và VPA | [Giai đoạn 4](../00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller), bài [72](../72-horizontal-pod-autoscale-vi.md), [73](../73-vertical-pod-autoscale-vi.md) | metrics-server (giai đoạn 11); riêng VPA còn cần **add-on ngoài Kubernetes** chưa có trong A1.4 | **HPA đã trả** ở Lab 11b; **VPA chưa trả** |
| 2 | `volumeClaimTemplates` của StatefulSet | giai đoạn 4, bài [65](../65-statefulset-vi.md) | StorageClass + provisioner (giai đoạn 6) | Lab 6a |
| 3 | Service quản trị headless cho StatefulSet | giai đoạn 4, bài [65](../65-statefulset-vi.md) | Service headless (giai đoạn 5) | Lab 5a |
| 4 | NetworkPolicy được thực thi thật | giai đoạn 5, bài [84](../84-network-policies-vi.md) | CNI hỗ trợ policy thay Flannel | Lab 5b |
| 5 | Ảnh chụp nhanh và nhân bản volume | [Giai đoạn 6](../00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), bài [99](../99-volume-snapshots-vi.md)–[101](../101-volume-pvc-datasource-vi.md) | **CSI driver hỗ trợ snapshot** — `local-path-provisioner` của baseline dùng `rancher.io/local-path`, không phải CSI, nên **chưa đủ điều kiện** | Lab 6b, nhánh A — chạy lại sau khi cluster có CSI driver hỗ trợ snapshot |
| 6 | Mã hóa Secret at rest | giai đoạn 3, bài [109](../109-secret-vi.md) | sửa cấu hình apiserver | [giai đoạn 22](../00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu) |
| 7 | Quản lý vòng đời certificate | giai đoạn 12, bài [156](../156-certificates-vi.md) | quy trình `kubeadm certs` | [giai đoạn 18](../00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ) |
| 8 | Backup và restore etcd | giai đoạn 8 | `etcdctl` và quy trình khôi phục | [giai đoạn 19](../00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa) |

Số hiệu nợ ở cột đầu **khớp với** [Sổ nợ lộ trình](../00-ALO-TRINH-ADMIN.md#sổ-nợ-lộ-trình). Trong file lộ trình, mỗi món nợ được đánh dấu ngay tại chỗ: `⏳ Nợ #N` ở nơi phát sinh và `✅ Trả nợ #N` ở nơi trả. Sửa một bảng thì phải sửa bảng kia.

## 6. Quy ước chung trong mọi lab

- Bằng chứng ghi vào `~/lab-evidence/<mã lab>/`, ví dụ `~/lab-evidence/1a/`.
- Fault injection chỉ chạy trên `lab-k8s-worker2`. Ngoại lệ (8a, 8b, 8c) liệt kê ở
  [mục 7.3, nguyên tắc 8](#73-nguyên-tắc-thiết-kế).
- Dòng bắt đầu bằng `PASS:` là điều kiện phải đạt; không đi tiếp khi gate fail.
- Số phiên bản chỉ tồn tại ở [bảng A1.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa);
  lab khác link về đó thay vì chép lại con số.
- Checkpoint là vấn đáp không nhìn tài liệu, không phải danh sách lệnh đã chạy.

Các lab dự án (3a → 14) có thêm quy ước riêng về repo, nhịp mở đầu – kết thúc sprint và nợ kỹ
thuật — xem [mục 7](#7-lab-dạng-dự-án-3a--14-mục-đích-và-cách-thiết-kế).

---

## 7. Lab dạng dự án (3a → 14): mục đích và cách thiết kế

Mục này giải thích **vì sao** 21 lab từ 3a tới 14 được viết thành các sprint của một dự án, chúng
được **thiết kế** theo những nguyên tắc nào, và người học **làm** một lab như thế nào. Chi tiết kỹ
thuật — kiến trúc, hợp đồng, bố cục repo, kế hoạch từng sprint — nằm ở
[DU-AN-BOOKSTORE.md](DU-AN-BOOKSTORE.md); mục này là bản đồ để đọc file đó đúng cách.

### 7.1. Vì sao đổi sang dạng dự án

Lab dạng runbook đưa sẵn từng lệnh để gõ theo. Gõ theo thì lab chạy được và mọi gate đều PASS,
nhưng người học không phải đọc lại tài liệu, không phải chọn giữa hai cách làm, không phải giải
thích vì sao. Kiến thức vì thế không trụ lại — đúng thứ mà khối *Tự kiểm tra* trong các bài dịch
đã làm được, còn lab thì chưa.

Dạng dự án nhắm tới ba điều:

1. **Mỗi việc trong lab đòi đúng kiểu suy nghĩ của khối *Tự kiểm tra*:** đọc lại một mục cụ thể,
   suy ra cơ chế, chọn field hay cách làm, rồi chứng minh bằng bằng chứng trên cluster thật.
2. **Kiến thức nối vào nhau.** Mọi nhóm bài áp lên cùng một hệ thống, nên sprint sau dựa vào kết quả
   sprint trước. Ví dụ: readiness probe viết ở 3a là thứ quyết định ở 5a Pod có nằm trong tập
   backend sẵn sàng nhận lưu lượng hay không — endpoint vẫn còn trong EndpointSlice, chỉ mang
   `ready=false`; PodDisruptionBudget đặt ở 3c là thứ chặn hay cho phép drain ở 12; requests chốt ở 3c là mẫu số
   của HPA ở 11b.
3. **Giống công việc thật của người quản trị:** nhận yêu cầu, điều tra, thay đổi hệ thống, diễn
   tập sự cố, viết quyết định thiết kế và postmortem, bàn giao cho sprint sau.

Lab 00 → Lab 2 vẫn là runbook vì chúng dựng môi trường và quan sát nền tảng — chưa có workload nào
để xây. Lab 15 (node Windows) cần thêm VM riêng nên nằm ngoài dự án và giữ dạng runbook.

### 7.2. Một hệ thống, 21 sprint

Dự án giả định một hiệu sách nhỏ đang chạy bằng `docker compose` trên một VM, với tám vấn đề đã ghi
nhận (một máy, không health check, mật khẩu trong file, phát hành dừng cả hệ thống…). Mục tiêu là
đưa nó lên Kubernetes và đạt 19 yêu cầu `R1`–`R19` — xem
[mục 2 của file dự án](DU-AN-BOOKSTORE.md#2-bài-toán). Mỗi lab là **một sprint** phủ **đúng một nhóm
bài** của lộ trình, và để lại hệ thống ở một trạng thái rõ ràng:

| Giai đoạn | Lab (sprint) | Hệ thống có thêm |
| --- | --- | --- |
| 3 — Pod và cấu hình | 3a (1), 3b (2), 3c (3) | ba Pod trần chạy đúng vòng đời; cấu hình và bí mật ra khỏi manifest; tài nguyên và QoS được khai báo |
| 4 — Workload controller | 4a (4), 4b (5) | Deployment, phát hành và rollback; StatefulSet cho DB, Job migration, CronJob backup, DaemonSet agent |
| 5 — Mạng | 5a (6), 5b (7) | các tầng gọi nhau bằng tên; mạng zero-trust trên Calico; HTTPS qua Ingress |
| 6 — Lưu trữ | 6a (8), 6b (9) | dữ liệu và bản backup bền; quy trình khôi phục đã diễn tập |
| 7 — Lập lịch và tài nguyên | 7a (10), 7b (11) | đặt workload đúng node, preemption; hạn mức cho namespace |
| 8 — kubeadm | 8a (12), 8b (13), 8c (14) | dựng lại cluster từ repo; đánh giá HA stacked và external etcd |
| 9 — Bảo mật | 9a (15), 9b (16) | danh tính và quyền tối thiểu; Pod Security `restricted`, tenant thứ hai |
| 11 — Observability | 11a (17), 11b (18) | metric, log theo node, metrics-server; HPA cho `web` |
| 12, 13, 14 | 12 (19), 13 (20, tùy chọn), 14 (21) | bảo trì node có kế hoạch; đánh giá DRA; operator sao lưu và nghiệm thu cuối |

Trạng thái đích sau Lab 14 là một hệ thống gần với production: ba tầng có controller, mạng
zero-trust, dữ liệu bền, quyền tối thiểu, quan sát được, tự co giãn, bảo trì được, và một API khai
báo riêng cho việc sao lưu — xem [mục 3 của file dự án](DU-AN-BOOKSTORE.md#3-kiến-trúc-đích).

### 7.3. Nguyên tắc thiết kế

1. **Một sprint = một nhóm bài.** Giữ nguyên nguyên tắc chia lab ở [mục 1](#1-nguyên-tắc-chia-lab).
   Bảng 1.1 của mỗi lab ánh xạ từng bài sang ticket; bài nào không thực hành được thì ghi rõ lý do.
2. **Không nhảy cóc, chỗ thiếu thành nợ.** Ticket chỉ dùng kiến thức của nhóm bài hiện tại và các
   nhóm trước ([mục 2](#2-nguyên-tắc-không-nhảy-cóc)). Hệ thống vì thế cố ý có điểm yếu ở mỗi sprint,
   và điểm yếu đó là **động lực** của sprint sau. Có hai sổ nợ, không lẫn nhau:

   | Sổ | Ghi gì | Ví dụ |
   | --- | --- | --- |
   | Nợ lab `#N` ([mục 5](#5-sổ-nợ-lab)) | phần **bài học** chưa thực hành được | `#4` — NetworkPolicy chưa được thực thi vì Flannel |
   | Nợ kỹ thuật `TD-nn` ([mục 8 của file dự án](DU-AN-BOOKSTORE.md#8-sổ-nợ-kỹ-thuật-dự-án)) | điểm yếu của **hệ thống** | `TD-03` — `api` tìm `db` bằng IP Pod, trả ở 5a bằng Service |

3. **Ba lớp trong một ticket: hợp đồng, cơ chế bắt buộc, lựa chọn của người học.**

   | Lớp | Là gì | Ticket nêu ở đâu | Script nghiệm thu |
   | --- | --- | --- | --- |
   | Hợp đồng giao tiếp | Tên object, label, cổng, đường dẫn, dữ liệu mẫu, bố cục repo — chốt ở [mục 4 và 5.5 của file dự án](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật) | **Yêu cầu**, nguyên văn | kiểm |
   | Cơ chế bắt buộc của bài học | Cơ chế mà nhóm bài đang dạy và ticket tồn tại để thực hành — `nodeSelector` thay `nodeName` ở 3a, ba loại probe, PDB `minAvailable` ở 3c | **Yêu cầu** được gọi tên, kèm câu hỏi thiết kế hỏi vì sao chọn nó và phương án thay thế là gì | kiểm |
   | Lựa chọn của người học | Cách viết manifest, cách tổ chức file, lệnh dùng để đạt hành vi | không nêu; chỉ ở *Gợi ý* và *Lời giải tham khảo* | không kiểm |

   Mọi cách làm đạt hợp đồng và dùng đúng cơ chế bắt buộc đều PASS. Script chỉ xác nhận đã bàn giao
   và hành vi đúng; nó không chứng minh lập luận đúng — phần đó do câu hỏi thiết kế và checkpoint vấn
   đáp đánh giá.
4. **Repo là nguồn sự thật, cluster là môi trường tạm.** Mỗi lab mở đầu bằng việc dựng lại toàn bộ
   hệ thống chỉ từ repo và kết thúc bằng teardown. Thứ gì chỉ tồn tại trên cluster cũ — một lệnh
   `kubectl edit` quên đưa vào repo — sẽ lộ ra ngay ở lab sau. Snapshot chỉ chứa hạ tầng, không chứa
   dự án.
5. **Yêu cầu nói "phải đạt gì", không nói "gõ gì".** Ngoài hợp đồng và cơ chế bắt buộc ở nguyên
   tắc 3, tên field và lệnh chỉ xuất hiện trong khối gập *Gợi ý* và *Lời giải tham khảo*.
6. **Kiểm chứng bằng bằng chứng, không bằng cảm nhận.** Mỗi ticket có dòng `PASS:` và script nghiệm
   thu; mỗi lab kết thúc bằng `tests/smoke.sh` **tích lũy** — bộ kiểm thử hồi quy cho mọi hợp đồng
   còn hiệu lực tới sprint đó.
7. **Lập luận là sản phẩm bàn giao.** Mỗi ticket có câu hỏi thiết kế; câu trả lời ghi vào repo dưới
   dạng ADR hoặc postmortem, như một đội vận hành thật.
8. **An toàn như các lab cũ.** Diễn tập sự cố mặc định chỉ chạy trên `lab-k8s-worker2`; không cài
   thêm phần mềm ngoài [bảng A1.3, A1.4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00);
   chuỗi snapshot ở [mục 3](#3-chuỗi-snapshot) giữ nguyên. Ngoại lệ đã định, mỗi lab tự khai phạm vi
   phá hoại ở mục 2 của nó:

   | Lab | Đích diễn tập | Vì sao |
   | --- | --- | --- |
   | 8a | `lab-k8s-master` và cả ba node | Bài học là dựng lại cluster từ con số không; đường lùi là snapshot `03-storage-ready` |
   | 8b, 8c | Bộ VM `lab-ha-*` riêng, không chạm cluster chính | Quorum và topology HA chỉ thử được khi tắt control plane hay etcd; `lab-ha-1` giữ repo nên không bao giờ bị tắt |
   | 12 | Bảo trì `lab-k8s-worker1` là **vận hành có kế hoạch**, không phải diễn tập | Mọi diễn tập sự cố của Lab 12 vẫn chỉ trên `lab-k8s-worker2` |

### 7.4. Giải phẫu một lab dự án

Khung 12 mục của mọi lab (tiêu đề, điểm bắt đầu và kết thúc, kết quả phải đạt, ánh xạ tài liệu,
quy ước, phần B, checkpoint, troubleshooting, nguồn) giữ nguyên. Phần khác nằm ở phần B:

| Mục | Vai trò |
| --- | --- |
| Blockquote đầu file | snapshot **và tag repo** đầu vào, đầu ra; tên sprint |
| `B0. Nhận bàn giao sprint` | gate A5.5 → kiểm tag repo của lab trước → `./scripts/deploy.sh` → `./tests/smoke.sh` |
| `B1`… `B<n>` | các ticket `[BS-<mã>-<nn>]`, theo thứ tự trong bảng 1.1 |
| Mục cuối phần B | đóng sprint: `smoke.sh` mới PASS → commit → tag `lab-<mã>` → bundle ra máy host → teardown → gate cuối; chụp snapshot nếu lab tạo mốc mới |
| `## 3. Checkpoint` | vấn đáp không nhìn tài liệu, gắn với hệ thống của chính người học; nêu nợ lab và nợ kỹ thuật còn mở |

Mỗi ticket có cùng một khuôn:

| Phần | Nội dung |
| --- | --- |
| **Loại** | một trong bốn loại ở bảng dưới |
| **Bối cảnh** | vì sao dự án cần việc này lúc này |
| **Yêu cầu** | điều phải đúng khi xong, viết theo hành vi |
| **Ràng buộc** | giới hạn về kiến thức, node, tên, cách làm |
| **Đọc lại** | link tới đúng mục trong bài dịch |
| **Bàn giao** | file trong repo, file bằng chứng |
| **Nghiệm thu** | các dòng `PASS:` |
| **Câu hỏi thiết kế** | trả lời vào `docs/adr/` hoặc `docs/postmortem/` của repo |
| Ba khối gập | *Gợi ý* → *Script nghiệm thu* → *Lời giải tham khảo* |

Bốn loại ticket, mô phỏng bốn kiểu việc của người vận hành:

| Loại | Làm gì | Thay đổi repo `~/bookstore`? |
| --- | --- | --- |
| *Tính năng* | thay đổi hệ thống để đạt một yêu cầu | có |
| *Điều tra* | thí nghiệm trên Pod thử nghiệm để trả lời một câu hỏi thiết kế | chỉ `docs/` |
| *Diễn tập sự cố* | cố ý gây lỗi trên `lab-k8s-worker2`, chẩn đoán, viết postmortem | chỉ `docs/` |
| *Vận hành* | thay đổi hạ tầng nền của cluster (CNI, ingress, storage, kubeadm) | ghi chép ở `platform/` |

Manifest của Pod thử nghiệm đặt ở `~/lab-work/<mã>/` và bị xóa khi xong ticket — chúng không bao giờ
vào `k8s/`.

### 7.5. Cách làm một ticket

1. Đọc ticket, rồi đọc lại đúng các mục ở **Đọc lại**.
2. Tự viết manifest hoặc chạy thí nghiệm; tự kiểm bằng `kubectl get`, `describe`, `logs`.
3. Mở **Script nghiệm thu**, chạy, đối chiếu từng dòng `PASS:`. Mở trước khi làm sẽ lộ lời giải.
4. Trả lời **Câu hỏi thiết kế** bằng lời của mình vào repo.
5. Mở **Lời giải tham khảo** để so. Khác cách không có nghĩa là sai — miễn script PASS và bạn giải
   thích được lựa chọn.

Bí quá khoảng 20 phút thì mở *Gợi ý*; quá khoảng 45 phút thì mở *Lời giải*. Ticket và mục có dấu ⭐
là tùy chọn. Phần cốt lõi của mỗi lab khoảng 4 giờ; cả lab 4–6 giờ, chia **hai phiên**. Dừng giữa
chừng không cần teardown, nhưng phải đi qua gate giữa phiên:

- **Cuối phiên 1** — dừng đúng **điểm dừng mà mục 1.2 của lab chỉ ra**; lab không ghi thì dừng ở
  ranh giới ticket, không dừng giữa một ticket *Diễn tập sự cố* hay khi node đang cordon hoặc mang
  taint thử nghiệm. Commit phần đã làm (`wip: ...` nếu dở), ghi mã ticket cuối đã PASS vào
  `~/lab-evidence/<mã lab>/phien.txt`. Tắt máy được nếu mục 1.2 cho phép; không chụp snapshot.
- **Đầu phiên 2** — chạy **gate tiếp tục ghi ở mục 1.2 của chính lab**. Gate đó khớp với trạng thái mà
  lab cố ý để lại ở điểm dừng; gate chung không thay được nó. Mọi gate tiếp tục đều có `PASS:`
  `git status` sạch và HEAD là commit cuối phiên 1. Phần còn lại theo mốc:

  | Trạng thái ở điểm dừng | Gate tiếp tục | Lab |
  | --- | --- | --- |
  | Cluster chính, Bookstore đang chạy | [A5.5 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab) rồi `./scripts/deploy.sh` thành công. Không chạy smoke của tag trước: ticket phiên 1 có thể đã đổi hợp đồng | mặc định — 3a → 5a, 6a → 7b, 9a → 11b |
  | Như trên, cộng kiểm của riêng lab | 12, 13: A5.5 và `./tests/smoke.sh`; 14: A5.5, `deploy.sh` và `./scripts/backup-operator.sh status` | 12, 13, 14 |
  | Cluster chính, nền mạng mới, Bookstore **đã gỡ** | A5.5 và gate chỉ đọc ở mục 1.2 của 5b. Không `deploy.sh`, không smoke: dự án chỉ được dựng lại ở B5 | 5b |
  | Cluster chính trước khi phá — phiên 2 không dừng giữa chừng | A5.5 và `./tests/smoke.sh`; nghỉ lâu thì làm lại B5 (RPO tính từ lúc dump) | 8a |
  | Bộ VM HA, cluster stacked đã dựng, chưa có Bookstore | Trên `lab-ha-1`: chạy lại Script nghiệm thu của B6 (chỉ đọc). A5.5 và smoke là của cluster chính, không áp dụng | 8b (giữa phiên 2 và 3) |
  | Bộ VM HA, mới có cụm etcd, **chưa có Kubernetes** | Trên `lab-ha-1`: chạy lại Script nghiệm thu của B6 (chỉ đọc). Chưa có API server nên A5.5, `kubectl` và smoke đều không áp dụng | 8c |

  Ticket cuối trong `phien.txt` là *Tính năng* thì chạy lại Script nghiệm thu của nó để chắc phần đã
  commit tái tạo được hành vi; ticket *Điều tra* hay *Diễn tập* thì không chạy lại — bằng chứng đã
  nằm trong `~/lab-evidence/`. Gate nào fail thì sửa trước khi mở ticket mới.

### 7.6. Người học bàn giao gì sau mỗi sprint

| Bàn giao | Ở đâu |
| --- | --- |
| Manifest, script, ghi chép hạ tầng | repo `~/bookstore`, commit và tag `lab-<mã>` |
| Bản sao repo ngoài VM | file bundle trên máy host ([mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm)) |
| Quyết định thiết kế, postmortem | `docs/adr/`, `docs/postmortem/` trong repo |
| Sổ nợ kỹ thuật đang sống | `docs/tech-debt.md` trong repo, mã `TD-nn` cố định theo file dự án |
| Bằng chứng chạy lệnh | `~/lab-evidence/<mã lab>/` |
| Câu trả lời checkpoint | vấn đáp không nhìn tài liệu, như mọi lab |

### 7.7. Repo, snapshot và nhịp sprint

- Repo dự án ở `~/bookstore` trên `lab-k8s-master`. Lab bắt đầu từ tag của lab trước, dựng lại toàn
  bộ hệ thống bằng `./scripts/deploy.sh` rồi `./tests/smoke.sh`; kết thúc bằng tag `lab-<mã>`, bundle
  chép ra máy host và `./scripts/teardown.sh`. Xem [mục 6 của file dự án](DU-AN-BOOKSTORE.md#6-nhịp-của-một-lab-dự-án).
- Ba script dùng chung là **công cụ**, không phải nội dung học: lab nào đổi cách triển khai thì đưa
  nguyên văn phiên bản mới, và mỗi phiên bản giữ đủ tính năng của phiên bản trước.
- Snapshot chỉ chứa hạ tầng: dự án luôn được teardown trước khi chụp mốc mới.
- Restore snapshot đưa cả `~/bookstore` về quá khứ — luôn có bundle mới nhất ngoài VM
  ([mục 5.4](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm)).
- Lab 8b, 8c chạy trên bộ VM riêng: repo được mang sang và mang về bằng bundle; chuỗi chính tiếp tục
  ở Lab 9a với tag `lab-8c`.
- Lab 13 là sprint tùy chọn; bỏ qua thì Lab 14 bắt đầu từ tag `lab-12`.

### 7.8. Giới hạn hiện tại

- 21 lab dạng dự án được viết ngày 13/10/2026 và **chưa được chạy thử trọn vẹn trên cluster thật**.
  Manifest và lệnh dựa trên tài liệu chính thức của đúng phiên bản baseline và trên các lab runbook
  cũ đã kiểm chứng, nhưng thời gian chờ, chuỗi thông báo lỗi và vài hành vi biên có thể lệch. Gặp
  chỗ không khớp thì ghi lại bằng chứng và sửa lab, không sửa tay trên cluster cho qua.
- Thời lượng dài hơn lab runbook vì phần suy luận và đọc lại tài liệu. Cột **Giờ** ở
  [mục 4](#4-bản-đồ-lab) là **ước lượng**, chưa đo với người học. Lần pilot đầu ghi riêng ba phần:
  đọc và suy luận, thao tác và chờ, bàn giao (ADR, postmortem, bundle) — rồi sửa cột này và ranh giới
  hai phiên theo số đo, không theo số dòng tài liệu.
- Ticket đổi Flannel sang Calico ở Lab 5b (B3) là **thí nghiệm migration**, không phải quy trình Calico
  hỗ trợ: tài liệu yêu cầu hệ thống của Calico nói cluster đã dùng CNI khác thì không migrate được.
  Ticket chỉ chạy trên ba VM có snapshot `01-cluster-ready` làm đường lùi; xem
  [mục 2.1 của Lab 5b](LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md#21-đổi-cni-là-thao-tác-phá-hoại).

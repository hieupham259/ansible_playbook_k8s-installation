# Lab thực hành cho lộ trình Kubernetes Administrator

Thư mục này chứa các bài lab đi kèm [LO-TRINH-ADMIN.md](../LO-TRINH-ADMIN.md). Mỗi lab là một
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
| 1b — Object, label, kubectl và kubeconfig | 1b (9 bài) | `01-cluster-ready` | trả về `01-cluster-ready` | 3–4 | ⬜ chưa viết |
| 1c — Vòng đời và cơ chế nền của object | 1c (7 bài) | `01-cluster-ready` | trả về `01-cluster-ready` | 2–3 | ⬜ chưa viết |
| 2 — Container, image, CRI và cgroup | 2 (8 bài) | `01-cluster-ready` | trả về `01-cluster-ready` | 2–3 | ⬜ chưa viết |
| 3a — Pod và vòng đời | 3a (11 bài) | `01-cluster-ready` | trả về `01-cluster-ready` | 3–4 | ⬜ chưa viết |
| 3b — Cấu hình và tài nguyên | 3b (7 bài) | `01-cluster-ready` | trả về `01-cluster-ready` | 3–4 | ⬜ chưa viết |
| 4 — Workload controller | 4 (11 bài thực hành) | `01-cluster-ready` | trả về `01-cluster-ready` | 3–4 | ⬜ chưa viết |
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
[Checkpoint tiếp nối](../LO-TRINH-ADMIN.md#checkpoint-tiếp-nối--nhánh-docstasks) và được thực
hành theo CP1–CP12 trên cluster đã có ở `04-metrics-ready`.

## 5. Sổ nợ lab

Bảng này ghi những phần **cố ý chưa làm** vì kiến thức cần cho nó thuộc giai đoạn sau. Lab
phát sinh nợ phải nói rõ trong phần checkpoint rằng nợ chưa trả; lab trả nợ phải nhắc lại
bài gốc.

| Nợ | Phát sinh ở | Vì cần | Trả ở |
| --- | --- | --- | --- |
| Thực hành HPA và VPA | giai đoạn 4, bài [72](../72-horizontal-pod-autoscale-vi.md), [73](../73-vertical-pod-autoscale-vi.md) | metrics-server (giai đoạn 11) | Lab 11b |
| `volumeClaimTemplates` của StatefulSet | giai đoạn 4, bài [65](../65-statefulset-vi.md) | StorageClass + provisioner (giai đoạn 6) | Lab 6a |
| Service quản trị headless cho StatefulSet | giai đoạn 4, bài [65](../65-statefulset-vi.md) | Service headless (giai đoạn 5) | Lab 5a |
| NetworkPolicy được thực thi thật | giai đoạn 5, bài [84](../84-network-policies-vi.md) | CNI hỗ trợ policy thay Flannel | Lab 5b |
| Ảnh chụp nhanh và nhân bản volume | giai đoạn 6, bài [99](../99-volume-snapshots-vi.md)–[101](../101-volume-pvc-datasource-vi.md) | CSI driver có hỗ trợ snapshot | Lab 6b |
| Mã hóa Secret at rest | giai đoạn 3, bài [109](../109-secret-vi.md) | sửa cấu hình apiserver | CP7 |
| Quản lý vòng đời certificate | giai đoạn 12, bài [156](../156-certificates-vi.md) | quy trình kubeadm certs | CP3 |
| Backup và restore etcd | giai đoạn 8 | `etcdctl` và quy trình khôi phục | CP4 |

## 6. Quy ước chung trong mọi lab

- Bằng chứng ghi vào `~/lab-evidence/<mã lab>/`, ví dụ `~/lab-evidence/1a/`.
- Fault injection chỉ chạy trên `lab-k8s-worker2`.
- Dòng bắt đầu bằng `PASS:` là điều kiện phải đạt; không đi tiếp khi gate fail.
- Số phiên bản chỉ tồn tại ở [bảng A1.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa);
  lab khác link về đó thay vì chép lại con số.
- Checkpoint là vấn đáp không nhìn tài liệu, không phải danh sách lệnh đã chạy.

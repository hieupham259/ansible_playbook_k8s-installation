# Dự án Bookstore — đưa một ứng dụng ba tầng lên Kubernetes qua từng sprint

> **Phạm vi:** Lab 3a → Lab 14. Lab 13 là sprint tùy chọn. Lab 00, 1a, 1b, 1c, 2 là phần nền,
> chưa thuộc dự án. Lab 15 (node Windows) nằm ngoài dự án.
> **Nơi làm việc:** repo git `~/bookstore` trên `lab-k8s-master`, cluster VM của
> [Lab 00](LAB-00-MOI-TRUONG-1.35.7.md).
>
> **Cập nhật và đối chiếu phiên bản:** 13/10/2026.

File này là **bản thiết kế tổng** của dự án xuyên suốt các lab từ giai đoạn 3 trở đi: bài toán,
kiến trúc đích, hợp đồng kỹ thuật cố định, cấu trúc repo và kế hoạch từng sprint. Mỗi file lab
từ [Lab 3a](LAB-3A-POD-VA-VONG-DOI.md) trở đi là **một sprint** của dự án này. Thứ tự đọc bài
vẫn nằm ở [lộ trình](../00-ALO-TRINH-ADMIN.md); bản đồ lab, chuỗi snapshot và sổ nợ lab nằm ở
[labs/README.md](README.md).

Đọc hết file này **một lần** trước khi mở Lab 3a. Sau đó chỉ cần quay lại mục 4 (hợp đồng) và
mục 7 (kế hoạch sprint) khi bắt đầu lab mới.

---

## 1. Vì sao lab chuyển sang dạng dự án

Các lab cũ đưa sẵn từng lệnh để gõ theo. Gõ theo thì lab chạy được nhưng người học không phải
đọc lại tài liệu, không phải chọn giữa hai cách làm, không phải giải thích vì sao. Kiến thức vì
thế không trụ lại.

Từ Lab 3a, mỗi lab giao **ticket** như trong một dự án thật:

- Ticket nói **cần đạt điều gì và vì sao**, không nói gõ lệnh gì.
- Người học tự đọc lại đúng mục tài liệu, tự viết manifest, tự chọn field.
- Ticket kết thúc bằng **tiêu chí nghiệm thu** (`PASS:`) kiểm chứng được và một **script nghiệm
  thu** chạy tự động. Script không quan tâm bạn viết manifest thế nào, chỉ kiểm hành vi.
- Mỗi ticket có **câu hỏi thiết kế**. Câu trả lời ghi vào repo dự án, giống một ADR (Architecture
  Decision Record) ngắn. Đây là phần tương đương với khối *Tự kiểm tra* trong các bài dịch.
- Gợi ý và lời giải tham khảo nằm trong khối gập. Mở theo thứ tự ở [mục 6.2](#62-làm-một-ticket).

Kiến thức của từng sprint **chỉ lấy từ nhóm bài tương ứng** trong lộ trình và các nhóm đã học
trước đó. Thứ gì cần kiến thức của giai đoạn sau thì thành **nợ kỹ thuật** của dự án
([mục 8](#8-sổ-nợ-kỹ-thuật-dự-án)) hoặc **nợ lab** ([sổ nợ lab](README.md#5-sổ-nợ-lab)), không
làm trước.

---

## 2. Bài toán

### 2.1. Hiện trạng

Một hiệu sách nhỏ bán sách qua web. Hệ thống hiện chạy bằng `docker compose` trên **một VM**:

```yaml
# docker-compose.yml — hệ thống cũ, chỉ để đọc
services:
  db:
    image: postgres
    environment:
      POSTGRES_DB: bookstore
      POSTGRES_PASSWORD: doi-mat-khau-nay
    volumes:
      - dbdata:/var/lib/postgresql/data
      - ./sql:/docker-entrypoint-initdb.d:ro
  api:
    image: postgrest/postgrest
    environment:
      PGRST_DB_URI: postgres://authenticator:doi-mat-khau-nay@db:5432/bookstore
      PGRST_DB_SCHEMAS: api
      PGRST_DB_ANON_ROLE: web_anon
      PGRST_ADMIN_SERVER_PORT: "3001"
    depends_on: [db]
  web:
    image: nginxinc/nginx-unprivileged
    ports: ["80:8080"]
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - ./html:/usr/share/nginx/html:ro
    depends_on: [api]
  backup:
    image: postgres
    entrypoint: ["sh", "-c", "while true; do pg_dump -h db -U postgres bookstore > /backup/$$(date +%F-%H%M).sql; sleep 3600; done"]
    environment:
      PGPASSWORD: doi-mat-khau-nay
    volumes:
      - ./backup:/backup
volumes:
  dbdata: {}
```

Các vấn đề đã ghi nhận:

| # | Vấn đề | Hệ quả |
| --- | --- | --- |
| P1 | Một VM duy nhất | VM chết là mất dịch vụ; bảo trì OS là downtime |
| P2 | Image không ghim version | Mỗi lần `pull` có thể ra một bản khác |
| P3 | Mật khẩu nằm thẳng trong file compose | Ai đọc được repo là có mật khẩu DB |
| P4 | Không có health check | Container treo vẫn được coi là đang chạy |
| P5 | Phát hành bằng tay, dừng cả hệ thống | Mỗi lần cập nhật là một lần gián đoạn |
| P6 | Backup bằng vòng lặp shell, cùng máy với DB | Mất VM là mất cả DB lẫn bản backup |
| P7 | Mọi thứ chạy root, chung một mạng phẳng | Một container bị chiếm là chạm được mọi thứ |
| P8 | Không đo được tải, không tự co giãn | Giờ cao điểm chậm, giờ thấp điểm lãng phí |

### 2.2. Yêu cầu của hệ thống mới

| Mã | Yêu cầu | Đáp ứng ở |
| --- | --- | --- |
| R1 | Mọi thành phần có health check đúng vai trò; container hỏng tự được thay | Lab 3a, 4a |
| R2 | Cấu hình và bí mật tách khỏi manifest và image | Lab 3b |
| R3 | Mỗi thành phần có tài nguyên được khai báo; ứng dụng chịu được gián đoạn tự nguyện | Lab 3c |
| R4 | Phát hành không gián đoạn, rollback được trong vài phút | Lab 4a |
| R5 | DB có định danh ổn định; migration, backup và agent node chạy như workload riêng | Lab 4b |
| R6 | Các tầng tìm nhau bằng tên, không bằng IP | Lab 5a |
| R7 | Mạng zero-trust giữa các tầng; truy cập từ ngoài qua HTTPS | Lab 5b |
| R8 | Dữ liệu và bản backup sống sót qua việc xóa Pod, xóa workload | Lab 6a |
| R9 | Có quy trình khôi phục dữ liệu đã diễn tập | Lab 6b, 8a |
| R10 | Bản sao trải đều trên các node; tác vụ nền không chiếm chỗ của dịch vụ chính | Lab 7a |
| R11 | Namespace của dự án có hạn mức; không workload nào ăn hết cluster | Lab 7b |
| R12 | Dựng lại được toàn bộ cluster và hệ thống từ repo; đánh giá được topology HA | Lab 8a, 8b, 8c |
| R13 | Quyền tối thiểu cho từng thành phần, cho CI và cho đội phát triển | Lab 9a |
| R14 | Mọi Pod đạt Pod Security Standard `restricted`; có mô hình nhiều tenant | Lab 9b |
| R15 | Có metric, log tập trung theo node, và quy trình chẩn đoán | Lab 11a |
| R16 | Tầng web tự co giãn theo tải | Lab 11b |
| R17 | Bảo trì node không làm gián đoạn dịch vụ | Lab 12 |
| R18 | Đánh giá được khả năng dùng thiết bị chuyên dụng cho tính năng tương lai | Lab 13 (tùy chọn) |
| R19 | Backup được vận hành bằng API khai báo của chính dự án | Lab 14 |

---

## 3. Kiến trúc đích

Đây là trạng thái **sau Lab 14**. Các sprint đi tới đây từng bước; [mục 7](#7-lộ-trình-sprint)
cho biết sau mỗi lab hệ thống trông như thế nào.

### 3.1. Sơ đồ tổng thể

```text
 Trình duyệt trên máy host Windows
   │  https://bookstore.lab.local   (file hosts trỏ tới IP một worker)
   ▼
 ┌─ namespace traefik ──────────────────────────┐
 │  Traefik (ingress controller, NodePort)       │  Lab 5b
 └──────────────────┬────────────────────────────┘
                    │ Ingress `bookstore`, TLS secret `bookstore-tls`
 ┌─ namespace bookstore  (PSS restricted, ResourceQuota, LimitRange, NetworkPolicy deny-all) ─┐
 │                  ▼                                                                         │
 │  Service web (ClusterIP :80) ──► Deployment web ×2..5  [HPA, PDB, topology spread]          │
 │                                   ├─ nginx (nginx-unprivileged) :8080                       │
 │                                   └─ sidecar status-writer (busybox)                        │
 │                  │ /api/  →  http://api:3000/                                               │
 │  Service api (ClusterIP :3000) ─► Deployment api ×2  [PDB, anti-affinity]                   │
 │                                   └─ postgrest :3000, admin :3001 (/live /ready /metrics)   │
 │                  │ postgres://authenticator@db-0.db:5432/bookstore                          │
 │  Service db (headless) ─────────► StatefulSet db ×1  [PriorityClass bookstore-critical]     │
 │                                   └─ postgres :5432, PVC data-db-0 (StorageClass            │
 │                                      bookstore-retain)                                     │
 │  Job db-migrate ───────────────► db         CronJob db-backup ──► PVC backup-store          │
 │  DbBackup (custom resource) ──► backup-operator tạo Job pg_dump        Lab 14               │
 └────────────────────────────────────────────────────────────────────────────────────────────┘
 ┌─ namespace bookstore-system (PSS privileged) ┐   ┌─ namespace bookstore-staging ──────────┐
 │  DaemonSet node-agent: log + heartbeat       │   │  tenant thứ hai, cô lập bằng RBAC,     │
 │  trên cả ba node                             │   │  quota, NetworkPolicy        Lab 9b   │
 └──────────────────────────────────────────────┘   └────────────────────────────────────────┘
 Nền cluster: Calico (5b) · Traefik (5b) · local-path-provisioner (6a) · metrics-server (11a)
```

### 3.2. Thành phần ứng dụng

| Thành phần | Workload cuối | Image | Vai trò | Xuất hiện |
| --- | --- | --- | --- | --- |
| `web` | Deployment `web`, 2–5 bản sao do HPA điều khiển | nginx-unprivileged + busybox | Phục vụ trang tĩnh, reverse proxy `/api/` | Pod ở 3a, Deployment ở 4a |
| `api` | Deployment `api`, 2 bản sao | PostgREST | REST API sinh từ schema `api` của DB | Pod ở 3a, Deployment ở 4a |
| `db` | StatefulSet `db`, 1 bản sao | PostgreSQL | Lưu sách và đơn hàng | Pod ở 3a, StatefulSet ở 4b |
| `db-migrate` | Job | PostgreSQL (dùng `psql`) | Chạy migration có tính bất biến kết quả | 4b |
| `db-backup` | CronJob | PostgreSQL (dùng `pg_dump`) | Backup logic định kỳ vào PVC | 4b, có PVC ở 6a |
| `node-agent` | DaemonSet | busybox | Phục vụ `/node.json` (tên node, tải) cho Pod cùng node; từ 11a đọc log node | 4b, chuyển namespace ở 11a |
| `backup-operator` | Vòng lặp controller dưới ServiceAccount riêng | — (xem Lab 14) | Hiện thực custom resource `DbBackup` | 14 |

Phiên bản image chỉ ghi ở [bảng A1.4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00).
Manifest dùng đúng tag ở bảng đó; không dùng `latest`.

### 3.3. Hạ tầng nền của cluster

Hạ tầng nền thuộc **cluster**, không thuộc repo ứng dụng. Nó được cài bởi lab tạo snapshot và
nằm sẵn trong snapshot cho các lab sau.

| Thành phần | Namespace | Cài ở | Snapshot chứa nó |
| --- | --- | --- | --- |
| Calico (CNI có thực thi NetworkPolicy) | theo manifest của Calico | Lab 5b | `02-net-ready` trở đi |
| Helm, Traefik, IngressClass `traefik` | `traefik` | Lab 5b | `02-net-ready` trở đi |
| local-path-provisioner, StorageClass mặc định `local-path` | `local-path-storage` | Lab 6a | `03-storage-ready` trở đi |
| metrics-server | `kube-system` | Lab 11a | `04-metrics-ready` |

Repo dự án giữ **ghi chép** cách cài các thành phần này trong thư mục `platform/` (xem
[mục 5.1](#51-vị-trí-và-cấu-trúc)) để Lab 8a dựng lại cluster được từ repo.

### 3.4. Luồng request và luồng dữ liệu

Luồng đọc danh sách sách:

```text
trình duyệt ─HTTPS─► Traefik ─HTTP─► Service web ─► Pod web (nginx)
   nginx: location /api/ ─► Service api ─► Pod api (PostgREST)
   PostgREST: SET ROLE web_anon; SELECT * FROM api.books ─► db-0 ─► JSON
```

Luồng dữ liệu bền:

```text
db-0 ──ghi──► PVC data-db-0 (StorageClass bookstore-retain, reclaimPolicy Retain)
CronJob db-backup ─pg_dump─► PVC backup-store ─(sao ra ngoài cluster bằng tay)─► máy host
DbBackup (CR) ─► backup-operator ─► Job pg_dump ─► PVC backup-store ─► status của CR
```

### 3.5. Thiết kế bảo mật

**Namespace và Pod Security Standards:**

| Namespace | `enforce` | Lý do |
| --- | --- | --- |
| `bookstore` | `restricted` | Mọi workload ứng dụng phải chạy non-root, không leo thang đặc quyền |
| `bookstore-staging` | `restricted` | Tenant thứ hai, cùng chuẩn |
| `bookstore-system` | `privileged` | `node-agent` cần `hostPath` chỉ đọc tới log của node |

**Danh tính và quyền** (Lab 9a):

| Chủ thể | Loại | Quyền |
| --- | --- | --- |
| `web`, `api`, `db`, `db-migrate` | ServiceAccount, `automountServiceAccountToken: false` | Không gọi API |
| `db-backup` | ServiceAccount | `get`, `patch` đúng ConfigMap `backup-status` |
| `ci-deployer` | ServiceAccount trong namespace `bookstore`, token có thời hạn | Cập nhật Deployment, ConfigMap trong `bookstore`; không đọc Secret |
| nhóm `bookstore-devs` | Group (kiểm bằng impersonation) | `view` trong `bookstore`; không đọc Secret. Trong `bookstore-staging` (từ 9b): Role `staging-developer`, quyền kiểu `edit` |
| nhóm `bookstore-oncall` | Group (kiểm bằng impersonation) | như `bookstore-devs`, thêm `exec` và gắn ephemeral container để gỡ lỗi — giới hạn `TD-07` |
| `backup-operator` | ServiceAccount | `DbBackup` (+ `status`), tạo và đọc Job; `get`/`list` Pod và `get` `pods/log` để đọc kết quả |

**Ma trận NetworkPolicy** (Lab 5b). Mặc định chặn cả hai chiều trong `bookstore`; chỉ mở:

| Từ | Tới | Cổng |
| --- | --- | --- |
| Pod của Traefik (namespace `traefik`) | `web` | 8080 |
| `web` | `api` | 3000 |
| `api`, `db-migrate`, `db-backup`, Job của `backup-operator` (mang `component: backup`) | `db` | 5432 |
| mọi Pod trong `bookstore` | CoreDNS trong `kube-system` | 53 UDP/TCP |
| `db-backup` (từ 9a) | API server (IP endpoint của Service `kubernetes`) | 6443 |
| `web` (sidecar `status-writer`) | `node-agent` (từ 11a ở namespace `bookstore-system`) | 9100 |

Từ 11a, `bookstore-system` cũng chặn mặc định hai chiều và chỉ mở ingress 9100 từ `web` của
`bookstore`.

Probe của kubelet đi từ chính node đó nên không bị NetworkPolicy chặn — xem lại bài
[84](../84-network-policies-vi.md) trước khi viết policy.

### 3.6. Ngân sách tài nguyên và lập lịch

Hai worker mỗi máy 2 vCPU, 6 GB ([A1.2 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a12-ba-vm)).
Control plane mang taint và **không** chạy workload ứng dụng, trừ DaemonSet `node-agent`.

| Container | `requests` gợi ý | `limits` gợi ý | QoS của Pod |
| --- | --- | --- | --- |
| `web/nginx` | 50m, 32Mi | 200m, 64Mi | Burstable |
| `web/status-writer` | 10m, 8Mi | 50m, 16Mi | (cùng Pod web) |
| `api/postgrest` | 50m, 64Mi | 500m, 256Mi | Burstable |
| `db/postgres` | 500m, 512Mi | 500m, 512Mi | Guaranteed |
| `node-agent/agent` | 10m, 16Mi | 50m, 32Mi | Burstable |

Đây là **điểm xuất phát**. Lab 3c đo mức dùng thật trong cgroup rồi chốt con số và ghi lý do
vào repo. Lab 7b đặt ResourceQuota dựa trên tổng đã chốt.

Ràng buộc lập lịch đích:

- `db` ở `lab-k8s-worker2`. Từ 3a tới 6b ghim bằng `nodeSelector` theo label
  `kubernetes.io/hostname`; từ 7a đổi sang node affinity theo label riêng của dự án. Volume
  `local-path` gắn chặt với node, nên đây cũng là ràng buộc của dữ liệu.
- `web` trải đều hai worker bằng topology spread; `api` dùng anti-affinity mềm.
- Job và CronJob dùng PriorityClass `bookstore-batch`; `db` và `api` dùng `bookstore-critical`.

### 3.7. Chịu lỗi và vận hành

| Tình huống | Hành vi mong đợi | Kiểm chứng ở |
| --- | --- | --- |
| Container `db` chết | `api` thành NotReady nhưng **không** bị khởi động lại; tự Ready lại khi DB lên | 3a |
| Pod `web` bị xóa | Controller tạo lại, Service không gửi request tới Pod chưa Ready | 4a, 5a |
| Phát hành hỏng | Rollout dừng, bản cũ vẫn phục vụ; rollback trong vài phút | 4a |
| Xóa nhầm StatefulSet hoặc PVC | Dữ liệu còn trên PV `Retain`; khôi phục được | 6a |
| Mất toàn bộ cluster | Dựng lại bằng kubeadm, cài nền, triển khai từ repo, khôi phục từ bản dump | 8a |
| Mất một control plane (HA) | API vẫn trả lời; mất quorum thì workload đang chạy vẫn phục vụ | 8b, 8c |
| Bảo trì `lab-k8s-worker1` | `web`, `api` không gián đoạn nhờ PDB và bản sao trên node kia | 12 |
| Bảo trì `lab-k8s-worker2` | `db` gián đoạn **có kế hoạch** trong cửa sổ bảo trì: PV local gắn node nên `db-0` không chuyển được | 12 |

---

## 4. Hợp đồng kỹ thuật

Mục này là **bất biến** giữa các lab. Script nghiệm thu của mọi lab dựa vào nó, nên bạn được tự
do viết manifest theo cách mình chọn **miễn là giữ đúng hợp đồng**. Đổi tên hay đổi cổng ở đây là
làm hỏng nghiệm thu của các lab sau.

### 4.1. Tên và label

Namespace ứng dụng: **`bookstore`**. Namespace phụ: `bookstore-system` (từ 11a),
`bookstore-staging` (từ 9b). Namespace tạm dùng trong diễn tập — `bookstore-restore-test` (6b),
`bookstore-thu` (11a) — cũng mang label dự án để teardown dọn được.

Mọi object của dự án — kể cả object cấp cluster như PriorityClass, StorageClass, ClusterRole, CRD
và chính các Namespace — mang label:

```yaml
app.kubernetes.io/part-of: bookstore
```

Object thuộc một thành phần mang thêm `app.kubernetes.io/component` và `app.kubernetes.io/name`:

| `component` | `name` | Object chính |
| --- | --- | --- |
| `web` | `nginx` | Pod `web` (3a) → Deployment `web` (4a); Service `web` (5a) |
| `api` | `postgrest` | Pod `api` (3a) → Deployment `api` (4a); Service `api` (5a) |
| `db` | `postgresql` | Pod `db` (3a) → StatefulSet `db`, Pod `db-0` (4b); Service headless `db` (5a) |
| `migrate` | `postgresql` | Job `db-migrate` (4b) |
| `backup` | `postgresql` | CronJob `db-backup` (4b); PVC `backup-store` (6a) |
| `agent` | `busybox` | DaemonSet `node-agent` (4b) |
| `loadtest` | `busybox` | Job sinh tải trong `tests/load/` (11b), không thuộc `k8s/` |
| `operator` | — | `backup-operator` (14) |

Selector của Service, controller và NetworkPolicy dùng cặp
`app.kubernetes.io/part-of=bookstore` + `app.kubernetes.io/component=<thành phần>`.

Tên container cố định: `nginx` và sidecar `status-writer` trong Pod web; `postgrest` trong Pod
api; `postgres` trong Pod db.

Các object có tên cố định khác:

| Object | Tên | Từ lab |
| --- | --- | --- |
| PriorityClass | `bookstore-critical` | 3a |
| PriorityClass | `bookstore-batch` | 7a |
| ConfigMap | `web-config`, `api-config`, `db-init` | 3b |
| Secret | `db-credentials` (key `postgres-password`, `authenticator-password`) | 3b |
| PodDisruptionBudget | `web`, `api` — từ 4a là `maxUnavailable: 1` để drain được khi HPA giữ `web` ở min | 3c |
| Ingress | `bookstore` (host `bookstore.lab.local`) | 5b |
| Secret TLS | `bookstore-tls` | 5b |
| StorageClass | `bookstore-retain` | 6a |
| PVC | `data-db-0` (từ template `data`), `backup-store` | 6a |
| ResourceQuota, LimitRange | `bookstore-quota`, `bookstore-limits` | 7b |
| ConfigMap | `backup-status` (key `last-run`, `file`, `bytes`, `pod`) | 9a |
| HorizontalPodAutoscaler | `web` | 11b |
| CRD | `dbbackups.bookstore.example.com`, kind `DbBackup`; ClusterRole `bookstore-dbbackup-viewer`; custom resource `baseline`; Job của operator tên `dbbackup-<tên CR>` | 14 |

Từ Lab 5a, ConfigMap có thể được sinh bằng `configMapGenerator` của Kustomize và mang hậu tố
hash. Script nghiệm thu từ đó **không** giả định tên ConfigMap tuyệt đối mà đọc tên từ spec của
workload.

### 4.2. Cổng, đường dẫn và điểm kiểm tra sức khỏe

| Thành phần | Cổng container (tên) | Đường dẫn | Ý nghĩa |
| --- | --- | --- | --- |
| `web` | 8080 (`http`) | `/` | Trang HTML có chữ `Bookstore` |
| | | `/healthz` | `200` khi nginx phục vụ được |
| | | `/status.json` | JSON do sidecar `status-writer` cập nhật định kỳ: `pod`, `node`, `version`, `updated`; từ 5a thêm `agent` (nội dung `/node.json` của `node-agent` cùng node) |
| | | `/api/` | Từ 5a: reverse proxy sang `api`, bỏ tiền tố `/api` |
| `api` | 3000 (`http`) | `/books` | Mảng JSON các cuốn sách |
| | 3001 (`admin`) | `/live` | `200` khi tiến trình PostgREST sống, **không** phụ thuộc DB |
| | | `/ready` | `200` khi kết nối được DB và đã nạp schema; `503` nếu không |
| | | `/metrics` | Metric dạng Prometheus (dùng ở 11a) |
| `db` | 5432 (`postgres`) | — | PostgreSQL |
| `node-agent` | 9100 (`http`) | `/node.json` | JSON `node`, `load`, `updated` của node mà agent đang chạy |
| | | `/logs.txt` | Từ 11a: văn bản thuần liệt kê file log của Pod `bookstore` trên node đó |
| `db-backup` | — | — | Mỗi lần chạy thành công in một dòng log bắt đầu bằng `db-backup: OK` |

Service (từ 5a): `web` cổng 80 → `http`; `api` cổng 3000 → `http`; `db` headless, cổng 5432;
`node-agent` cổng 9100 → `http` với `internalTrafficPolicy: Local`.
NodePort của `web` ở 5a là `30080`; từ 5b `web` trở về ClusterIP và người dùng đi qua Ingress.
Traefik (namespace `traefik`) mở NodePort cố định `30081` cho HTTP và `30443` cho HTTPS, khai trong
`platform/traefik/values.yaml`.

Hành vi của `/live` và `/ready` được tài liệu PostgREST mô tả ở mục
[Admin Server](https://docs.postgrest.org/en/v14/references/admin_server.html).

### 4.3. Dữ liệu

- Database `bookstore`, superuser `postgres`.
- Schema `api`; bảng `api.books` (`id`, `title`, `author`, `price_vnd`, `stock`) và
  `api.orders` (`id`, `book_id`, `quantity`, `created_at`).
- Role `web_anon` (`NOLOGIN`): `SELECT` trên `api.books`; `SELECT`, `INSERT` trên `api.orders`.
- Role `authenticator` (`LOGIN NOINHERIT`), được `GRANT web_anon`. PostgREST đăng nhập bằng role
  này rồi chuyển sang `web_anon` cho request ẩn danh.
- Dữ liệu mẫu tối thiểu: **5 cuốn sách** sau lần khởi tạo đầu. Job `db-migrate` từ 4b thêm ít nhất
  3 cuốn; từ 4b mọi `smoke.sh` kiểm `>= 8`.

Script khởi tạo DB chỉ chạy khi thư mục dữ liệu **rỗng** — đó là hành vi của entrypoint image
PostgreSQL với `/docker-entrypoint-initdb.d`. Ghi nhớ điều này khi dữ liệu bắt đầu bền ở 6a.

### 4.4. Image

| Thành phần | Image | Ghi chú đã kiểm |
| --- | --- | --- |
| `web` | `nginxinc/nginx-unprivileged` (biến thể `-alpine`) | `USER 101`, lắng nghe 8080, `STOPSIGNAL SIGQUIT` |
| `api` | `postgrest/postgrest` | `USER 1000`, một layer chỉ chứa binary — không có shell |
| `db`, `db-migrate`, `db-backup` | `postgres` | Entrypoint chạy root rồi hạ quyền xuống `postgres` (uid 999); `STOPSIGNAL SIGINT`; `PGDATA=/var/lib/postgresql/data` |
| sidecar, init container, `node-agent`, Pod thử nghiệm | `busybox` | Đã dùng từ Lab 00 |

Tag chính xác của cả bốn image nằm ở [A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa)
(busybox) và [A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) (ba image còn
lại). Cột "Ghi chú đã kiểm" đọc từ config của image trên registry ở ngày đối chiếu đầu file.

---

## 5. Repo dự án

### 5.1. Vị trí và cấu trúc

Repo là git repository ở `~/bookstore` trên `lab-k8s-master`. Cấu trúc **đích**:

```text
~/bookstore/
├── README.md               mô tả hệ thống, cách triển khai, cách kiểm thử
├── .gitignore              loại secrets/*.env
├── docs/
│   ├── adr/                quyết định thiết kế: NNNN-<chu-de>.md
│   ├── postmortem/         báo cáo sau mỗi diễn tập sự cố
│   ├── security-review.md  rà soát bảo mật (9b)
│   └── tech-debt.md        nợ kỹ thuật còn mở và đã trả
├── k8s/
│   ├── cluster/            object cấp cluster của dự án (PriorityClass, StorageClass, CRD, …)
│   └── app/                namespace bookstore và mọi object trong đó
├── platform/               ghi chép cách cài hạ tầng nền (Lab 5b, 6a, 11a, 8a)
├── secrets/                file .env sinh tại chỗ, KHÔNG commit (từ 3b)
├── scripts/
│   ├── deploy.sh           triển khai toàn bộ dự án từ repo
│   └── teardown.sh         gỡ toàn bộ object của dự án khỏi cluster
└── tests/
    └── smoke.sh            kiểm thử khói của trạng thái hiện tại
```

Lab 3a dựng khung này. Các lab sau thêm file vào đúng chỗ theo [mục 5.5](#55-bố-cục-repo-theo-sprint):
`scripts/deploy.sh` được viết sẵn cho bố cục đó, nên tên **thư mục và file manifest** là một phần
của hợp đồng. Nội dung bên trong manifest do bạn viết.

### 5.2. Ba script dùng chung

| Script | Hợp đồng |
| --- | --- |
| `scripts/deploy.sh` | Từ một cluster đang ở snapshot đầu vào của lab, triển khai **toàn bộ** dự án và chờ tới khi mọi thành phần Ready. Thoát `0` khi thành công. Chạy lại nhiều lần không gây lỗi. |
| `scripts/teardown.sh` | Gỡ **mọi** object mang label `app.kubernetes.io/part-of=bookstore`, cả cấp cluster, cùng dữ liệu còn sót mà dự án tạo trên node. Sau khi chạy, cluster trở về đúng snapshot đầu vào. |
| `tests/smoke.sh` | Kiểm hợp đồng của trạng thái hiện tại; in một dòng `PASS:` hoặc `FAIL:` cho từng kiểm tra; thoát khác `0` nếu có `FAIL:`. |

Ba script là **công cụ**, không phải nội dung học. Lab nào đổi cách triển khai thì đưa nguyên
văn phiên bản mới của script trong ticket tương ứng. `tests/smoke.sh` được thay bằng phiên bản
mới ở cuối **mỗi** lab, nên nó luôn là bộ kiểm thử hồi quy của sprint vừa xong.

### 5.3. Quy tắc làm việc với repo

1. **Repo là nguồn sự thật duy nhất.** Mọi thay đổi trên cluster phải nằm trong repo trước khi
   đóng sprint. Lệnh imperative (`kubectl edit`, `kubectl scale`, `kubectl set image`) chỉ dùng
   trong diễn tập hoặc thử nghiệm, và phải được dọn hoặc chuyển vào repo.
2. **Pod và object thử nghiệm** của ticket loại *Điều tra* hay *Diễn tập* không vào `k8s/`. Đặt
   manifest của chúng ở `~/lab-work/<mã lab>/` như các lab trước, xóa khi xong ticket.
3. **Không commit bí mật.** Trước 3b mật khẩu nằm trong manifest — đó là nợ `TD-02`, được ghi
   rõ. Từ 3b, bí mật sinh tại chỗ vào `secrets/`, và `.gitignore` chặn chúng.
4. **Mỗi lab kết thúc bằng một commit và một tag** `lab-<mã>`, ví dụ `lab-3a`. Lab sau bắt đầu
   từ tag đó.
5. **Câu trả lời câu hỏi thiết kế** ghi vào `docs/adr/` hoặc `docs/postmortem/` theo chỉ dẫn của
   ticket. Ghi bằng lời của bạn; không chép lời giải tham khảo. Số ADR ghi trong lab chỉ là gợi ý:
   luôn dùng số kế tiếp trong `docs/adr/` của bạn.

### 5.4. Sao lưu repo ra ngoài VM

Restore snapshot VM sẽ đưa đĩa của `lab-k8s-master` về quá khứ, **kể cả `~/bookstore`**. Lab 8a
chủ động restore snapshot; ngoài ra bạn có thể phải restore khi một lab hỏng. Vì vậy cuối mỗi lab
đóng gói repo thành một file và chép ra máy host:

```bash
# Trên lab-k8s-master
cd ~/bookstore
git bundle create ~/bookstore-lab-3a.bundle --all
git bundle verify ~/bookstore-lab-3a.bundle
```

```powershell
# Trên máy host Windows, PowerShell; sửa user và thư mục đích cho đúng máy của bạn
scp <user>@192.168.100.221:~/bookstore-lab-3a.bundle "$HOME\lab-backup\"
```

Snapshot chụp ở cuối Lab 5b, 6a, 11a chứa `~/bookstore` ở đúng tag của lab đó: restore mốc nào là
đưa repo về tag tương ứng. Khôi phục repo sau khi restore snapshot: chép file bundle mới nhất về master rồi
`git clone ~/bookstore-lab-<mã>.bundle ~/bookstore`. Nếu bạn có remote git riêng (GitHub,
GitLab) thì `git push --tags` lên đó cũng đạt yêu cầu này.

Bundle **không** chứa `secrets/` vì thư mục đó bị `.gitignore` loại trừ. Đó là chủ đích:
`scripts/deploy.sh` từ Lab 3b tự sinh bí mật mới khi thiếu, và vì `teardown.sh` xóa luôn dữ liệu,
DB được khởi tạo lại với bí mật mới. Ngoại lệ duy nhất: khi **thư mục dữ liệu** của DB được giữ lại
(PV `Retain` gắn lại ở 6a), mật khẩu của role nằm trong thư mục đó là mật khẩu cũ — phải dùng lại
đúng `secrets/db.env` cũ, hoặc đặt lại mật khẩu role. Nạp bản dump (6b, 8a) thì không vướng: bản
`pg_dump` của một database không chứa role hay mật khẩu; role do script khởi tạo tạo với bí mật
hiện hành.

### 5.5. Bố cục repo theo sprint

Cột cuối là cơ chế mà `scripts/deploy.sh` của lab đó dùng. Lab nào đổi cơ chế thì đưa nguyên văn
`deploy.sh` và `teardown.sh` mới.

Từ 3b, Kustomize sinh Secret từ file trong `secrets/` ở gốc repo — **ngoài** thư mục gốc của
kustomization. Kustomize mặc định từ chối đọc file như vậy và `kubectl apply -k` không có cờ nới
lỏng, nên `deploy.sh` luôn render bằng `kubectl kustomize --load-restrictor=LoadRestrictionsNone`
rồi `kubectl apply -f -`. Gõ tay `kubectl apply -k k8s/app` sẽ lỗi — đó là chủ đích, mọi triển
khai đi qua `deploy.sh` (Lab 3b, ADR 0007).

| Sau lab | Thêm hoặc đổi trong repo | Cơ chế triển khai |
| --- | --- | --- |
| 3a | `k8s/cluster/*.yaml`; `k8s/app/00-namespace.yaml`; `k8s/app/db/pod.yaml`; `k8s/app/api/pod.yaml.tpl`; `k8s/app/web/pod.yaml` | `kubectl apply -f` từng thư mục; file `.tpl` được thay `__DB_HOST__` bằng IP Pod `db` |
| 3b | `k8s/app/kustomization.yaml` gom namespace, `db`, `web`, ConfigMap và `secretGenerator` (Secret giữ tên cố định, không hậu tố hash); `.gitignore` đổi thành `secrets/*` trừ `secrets/.gitkeep` | sinh `secrets/db.env` nếu thiếu → `kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/app \| kubectl apply -f -` → chờ `db` → render `.tpl` của `api` → apply |
| 3c | `k8s/app/web/pdb.yaml`, `k8s/app/api/pdb.yaml` | như 3b |
| 4a | `k8s/app/web/deployment.yaml` thay `pod.yaml`; `k8s/app/api/deployment.yaml.tpl` thay `pod.yaml.tpl`; PDB đổi sang `maxUnavailable: 1` | như 3b; chờ bằng `kubectl rollout status`; xóa Pod trần `web`/`api` còn sót. Phát hành nội dung bằng cách tăng label `app.kubernetes.io/version` trong Pod template (tới 5a) |
| 4b | `k8s/app/db/statefulset.yaml` thay `pod.yaml`; `k8s/app/jobs/db-migrate.yaml.tpl`, `k8s/app/jobs/db-backup.yaml.tpl`; `k8s/app/agent/daemonset.yaml`; bản thiết kế `docs/hpa-web.draft.yaml` (chưa apply) | mọi file `k8s/app/**/*.tpl` được render với IP của `db-0` |
| 5a | không còn file `.tpl`: các tầng gọi nhau bằng tên DNS; `k8s/app/*/service.yaml`; ConfigMap sinh bằng `configMapGenerator` có hậu tố hash | render Kustomize như 3b rồi `kubectl apply -f -`, chờ rollout |
| 5b | `k8s/app/network/` (NetworkPolicy), `k8s/app/ingress/`; Secret TLS sinh từ `secrets/tls.crt`, `secrets/tls.key`; `platform/versions.env`, `platform/calico/README.md`, `platform/traefik/values.yaml` và `README.md` | như 5a; `deploy.sh` tự tạo certificate tự ký nếu thiếu |
| 6a | `k8s/cluster/storageclass-retain.yaml`; `volumeClaimTemplates` của `db`; `k8s/app/backup/pvc.yaml`; `k8s/app/web/status-writer.env`; `platform/local-path-provisioner/` | `deploy.sh` kiểm provisioner, không sinh `secrets/db.env` mới khi dữ liệu cũ còn, apply hai pha (`-l 'app.kubernetes.io/component!=migrate'` trước, `db-migrate` sau); `teardown.sh` dọn PV của dự án và thư mục `/opt/local-path-provisioner/*_bookstore_*` trên node qua `ssh` |
| 6b | chỉ `docs/` (ADR backup, runbook chép bản dump ra ngoài) | không đổi |
| 7a | `k8s/cluster/priorityclass-batch.yaml`; Job báo cáo chạy theo yêu cầu `docs/bao-cao-ton-kho.job.yaml` (mang `component: backup` để qua NetworkPolicy vào `db`); `web` spread dùng `nodeTaintsPolicy: Honor` | như 5a; `deploy.sh` gắn label node `bookstore.example.com/data=true` cho `lab-k8s-worker2`, `teardown.sh` gỡ |
| 7b | `k8s/app/policy/` (ResourceQuota, LimitRange); `docs/ngan-sach-namespace.md` | như 7a; Kustomize tự xếp quota và LimitRange trước workload |
| 8a | `platform/README.md`, `platform/kubeadm/`; bổ sung `platform/versions.env` | không đổi |
| 8b | `k8s/overlays/ha-lab/` (vá `services.nodeports: "1"`, gỡ ghim node của `db` và `db-backup`, `emptyDir` thay PVC); `platform/ha-lab/`; `scripts/deploy-ha-lab.sh`, `scripts/teardown-ha-lab.sh`, `tests/smoke-ha-lab.sh`; công cụ đo `scripts/do-san-sang.ps1` (CSV `ms,app,api`) | chỉ dùng trên bộ VM HA; `deploy.sh`, `teardown.sh`, `tests/smoke.sh` của chuỗi chính không đổi |
| 9a | `k8s/app/rbac/`; `k8s/app/backup/status-configmap.yaml`; `k8s/app/network/backup-to-apiserver.yaml`; `scripts/ci-kubeconfig.sh` | như 7a; kubeconfig của CI sinh vào `secrets/` bằng `scripts/ci-kubeconfig.sh` |
| 9b | label Pod Security trên namespace (bước trung gian `enforce: baseline` rồi `restricted`); `k8s/tenants/staging/` (chỉ chạy `web`, `staging-quota`, `staging-limits`, Role `staging-developer`, NetworkPolicy cô lập); `docs/security-review.md`, `docs/pss-kiem-ke.md` | áp thêm `k8s/tenants/*`; `deploy.sh` coi cảnh báo Pod Security là lỗi |
| 11a | `k8s/system/` (namespace `bookstore-system`, `node-agent` cùng ServiceAccount của nó chuyển sang đây); `platform/metrics-server/` | áp `k8s/system` trước `k8s/app` |
| 11b | `k8s/app/web/hpa.yaml`; bỏ `replicas` khỏi Deployment `web`; công cụ tải `tests/load/` | không đổi; `smoke.sh` chờ HPA đưa `web` lên min |
| 12 | `tests/availability-probe.sh`, `tests/availability-report.sh`; `docs/runbook-bao-tri-node.md`; ⭐ `preStop` cho `api` | không đổi |
| 13 | chỉ `docs/` (ADR, bản nháp `docs/dra/*.draft.yaml`) | không đổi |
| 14 | `k8s/cluster/crd-dbbackup.yaml`; `scripts/backup-operator.sh` (`run` mặc định, `once`, `start`, `stop`, `restart`, `status`); `k8s/app/rbac/backup-operator.yaml`; DbBackup `baseline` trong `k8s/app`; quota nâng `count/jobs.batch` lên 16, thêm `count/dbbackups.bookstore.example.com` | CRD được áp và chờ `Established` trước mọi custom resource; `deploy.sh` khởi động operator và chờ `baseline` thành công; `teardown.sh` dừng operator, xóa custom resource trước CRD |

---

## 6. Nhịp của một lab dự án

### 6.1. Mở đầu

Mỗi lab từ 3b trở đi bắt đầu bằng mục `B0. Nhận bàn giao sprint` gồm bốn bước:

1. Chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab) trên
   snapshot đầu vào của lab.
2. Kiểm repo: đúng tag của lab trước, không có thay đổi chưa commit.
3. `./scripts/deploy.sh` rồi `./tests/smoke.sh` — hệ thống của sprint trước dựng lại được chỉ từ
   repo.
4. Tạo `~/lab-evidence/<mã lab>/` và `~/lab-work/<mã lab>/`.

Bước 3 fail nghĩa là repo của sprint trước **chưa đủ** — có thứ chỉ tồn tại trên cluster cũ. Sửa
repo trước, không sửa tay trên cluster.

### 6.2. Làm một ticket

Mỗi ticket là một mục `B<n>` có cấu trúc cố định:

| Phần | Nội dung |
| --- | --- |
| Tiêu đề | `## B<n>. [BS-<mã lab>-<nn>] <tên ticket>` |
| **Loại** | *Tính năng* (thay đổi hệ thống), *Điều tra* (thí nghiệm để trả lời một câu hỏi thiết kế), *Diễn tập sự cố* (cố ý gây lỗi trên `lab-k8s-worker2` rồi chẩn đoán), hoặc *Vận hành* (thay đổi hạ tầng nền) |
| **Bối cảnh** | Vì sao dự án cần việc này lúc này |
| **Yêu cầu** | Điều phải đúng khi xong, viết theo hành vi |
| **Ràng buộc** | Giới hạn về kiến thức, node, tên, cách làm |
| **Đọc lại** | Mục cụ thể trong bài dịch cần đọc lại trước khi làm |
| **Bàn giao** | File trong repo, file bằng chứng |
| **Nghiệm thu** | Các dòng `PASS:` |
| **Câu hỏi thiết kế** | Trả lời vào repo |
| Khối gập *Gợi ý* | Mở khi bí quá 20 phút; gợi ý đi từ "đọc ở đâu" tới "dùng field nào" |
| Khối gập *Script nghiệm thu* | Chạy sau khi tự làm xong; đọc trước sẽ lộ lời giải |
| Khối gập *Lời giải tham khảo* | Mở sau khi đã nghiệm thu, hoặc khi bí quá 45 phút |

Trình tự làm:

1. Đọc ticket, đọc lại các mục ở **Đọc lại**.
2. Viết manifest hoặc chạy thí nghiệm. Tự kiểm bằng `kubectl get`, `describe`, `logs`.
3. Mở **Script nghiệm thu**, chạy, đối chiếu từng dòng `PASS:`.
4. Trả lời **Câu hỏi thiết kế** vào repo.
5. Mở **Lời giải tham khảo**, so với cách của bạn. Khác nhau không có nghĩa là sai — miễn script
   nghiệm thu PASS và bạn giải thích được lựa chọn.

### 6.3. Đóng sprint

Mục cuối của phần B trong mỗi lab là `Đóng sprint: cleanup và gate cuối`:

1. Thay `tests/smoke.sh` bằng phiên bản của lab, chạy, phải PASS toàn bộ.
2. Cập nhật `docs/tech-debt.md`: nợ vừa trả, nợ vừa phát sinh.
3. `git commit`, `git tag lab-<mã>`, đóng gói và chép bundle ra máy host
   ([mục 5.4](#54-sao-lưu-repo-ra-ngoài-vm)).
4. Dọn `~/lab-work/<mã lab>/` và mọi object thử nghiệm.
5. `./scripts/teardown.sh`.
6. Gate cuối: không còn object nào mang label `app.kubernetes.io/part-of=bookstore`, gate ngắn
   A5.5 PASS. Lab tạo snapshot mới thì chụp **sau** bước này.

Teardown sau mỗi sprint là chủ đích: cluster lab là môi trường dùng chung và phải trả về đúng
snapshot. Dự án sống trong repo; việc dựng lại toàn bộ hệ thống chỉ bằng repo ở đầu mỗi lab là
bài kiểm tra liên tục rằng hệ thống được khai báo đầy đủ.

### 6.4. Khuôn file lab

Mỗi file lab giữ khung bắt buộc của chuỗi lab, thêm phần dự án:

1. `# Lab <mã> — <tên>`.
2. Blockquote: điểm bắt đầu (snapshot + tag repo), điểm kết thúc, sprint, ngày đối chiếu.
3. Câu dẫn tới nhóm bài trong lộ trình, tới file này và tới A5.5.
4. `## 1. Kết quả phải đạt`, `### 1.1. Ánh xạ tài liệu sang bài thực hành` (bài → ticket),
   `### 1.2. Thời lượng`.
5. `## 2. Quy ước và an toàn`.
6. `# Phần B — Sprint <n>: <tên sprint>` với `B0` nhận bàn giao, các ticket `B1`…, mục cuối đóng
   sprint.
7. `## 3. Checkpoint <mã>` — vấn đáp không nhìn tài liệu, kèm *Bài giải thích cuối cùng*.
8. `## 4. Troubleshooting của lab này`.
9. `## 5. Nguồn chính thức`.

---

## 7. Lộ trình sprint

### 7.1. Bảng tổng

| Lab | Sprint | Bắt đầu từ | Kết thúc | Hệ thống sau sprint |
| --- | --- | --- | --- | --- |
| [3a](LAB-3A-POD-VA-VONG-DOI.md) | 1 — Pod hóa ba thành phần | `01-cluster-ready`, repo mới | `01-cluster-ready`, tag `lab-3a` | Pod trần `db`, `api`, `web`; probe, init, sidecar, tắt êm |
| [3b](LAB-3B-CAU-HINH-UNG-DUNG.md) | 2 — Cấu hình ngoài code | `01-cluster-ready`, `lab-3a` | `01-cluster-ready`, `lab-3b` | ConfigMap, Secret, Kustomize cho secret |
| [3c](LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md) | 3 — Định cỡ và chịu gián đoạn | `01-cluster-ready`, `lab-3b` | `01-cluster-ready`, `lab-3c` | requests/limits, QoS, PDB, resize tại chỗ |
| [4a](LAB-4A-REPLICASET-DEPLOYMENT-VA-ROLLOUT.md) | 4 — Lên Deployment | `01-cluster-ready`, `lab-3c` | `01-cluster-ready`, `lab-4a` | Deployment `web`, `api`; rollout, rollback |
| [4b](LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) | 5 — Trạng thái và tác vụ nền | `01-cluster-ready`, `lab-4a` | `01-cluster-ready`, `lab-4b` | StatefulSet `db`, Job, CronJob, DaemonSet |
| [5a](LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md) | 6 — Nối các tầng | `01-cluster-ready`, `lab-4b` | `01-cluster-ready`, `lab-5a` | Service, DNS, Kustomize đầy đủ, NodePort |
| [5b](LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) | 7 — Mạng zero-trust và HTTPS | `01-cluster-ready`, `lab-5a` | **tạo** `02-net-ready`, `lab-5b` | Calico, NetworkPolicy, Traefik, Ingress TLS |
| [6a](LAB-6A-PV-PVC-VA-STORAGECLASS.md) | 8 — Dữ liệu bền | `02-net-ready`, `lab-5b` | **tạo** `03-storage-ready`, `lab-6a` | PVC cho DB và backup, StorageClass Retain |
| [6b](LAB-6B-SNAPSHOT-VA-VOLUME-NANG-CAO.md) | 9 — Chiến lược sao lưu | `03-storage-ready`, `lab-6a` | `03-storage-ready`, `lab-6b` | Diễn tập khôi phục từ dump, ADR về snapshot |
| [7a](LAB-7A-LAP-LICH-VA-EVICTION.md) | 10 — Đặt workload đúng chỗ | `03-storage-ready`, `lab-6b` | `03-storage-ready`, `lab-7a` | Spread, affinity, PriorityClass, gate lập lịch |
| [7b](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md) | 11 — Hạn mức namespace | `03-storage-ready`, `lab-7a` | `03-storage-ready`, `lab-7b` | ResourceQuota, LimitRange |
| [8a](LAB-8A-DUNG-CLUSTER-BANG-KUBEADM.md) | 12 — Diễn tập dựng lại cluster | `03-storage-ready`, `lab-7b` | restore `03-storage-ready`, `lab-8a` | Cấu hình kubeadm và `platform/` trong repo |
| [8b](LAB-8B-HA-VOI-STACKED-ETCD.md) | 13 — HA stacked etcd | bộ VM riêng, `lab-8a` | **tạo** `8x-ha-stacked`, `lab-8b` | Overlay `ha-lab`, đo sẵn sàng khi mất control plane |
| [8c](LAB-8C-HA-VOI-EXTERNAL-ETCD.md) | 14 — HA external etcd | bộ VM riêng, `lab-8b` | **tạo** `8x-ha-external`, `lab-8c` | ADR chọn topology cho production |
| [9a](LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md) | 15 — Danh tính và quyền | `03-storage-ready`, `lab-8c` | `03-storage-ready`, `lab-9a` | ServiceAccount, Role, CI identity |
| [9b](LAB-9B-POD-SECURITY-VA-HARDENING.md) | 16 — Hardening | `03-storage-ready`, `lab-9a` | `03-storage-ready`, `lab-9b` | PSS `restricted`, tenant staging |
| [11a](LAB-11A-OBSERVABILITY.md) | 17 — Quan sát hệ thống | `03-storage-ready`, `lab-9b` | **tạo** `04-metrics-ready`, `lab-11a` | Metric, log node, metrics-server |
| [11b](LAB-11B-HPA-VA-VPA.md) | 18 — Tự co giãn | `04-metrics-ready`, `lab-11a` | `04-metrics-ready`, `lab-11b` | HPA cho `web` |
| [12](LAB-12-VAN-HANH-VONG-DOI-NODE.md) | 19 — Bảo trì node | `04-metrics-ready`, `lab-11b` | `04-metrics-ready`, `lab-12` | Quy trình bảo trì đã diễn tập |
| [13](LAB-13-DRA.md) | 20 — Đánh giá DRA (tùy chọn) | `04-metrics-ready`, `lab-12` | `04-metrics-ready`, `lab-13` | ADR về thiết bị chuyên dụng |
| [14](LAB-14-CRD-VA-OPERATOR.md) | 21 — Operator sao lưu | `04-metrics-ready`, `lab-13` hoặc `lab-12` | `04-metrics-ready`, `lab-14` | CRD `DbBackup` và controller; nghiệm thu cuối |

### 7.2. Chi tiết từng sprint

Danh sách ticket dưới đây là **kế hoạch**. File lab là nơi định nghĩa đầy đủ từng ticket.

#### Sprint 1 — Lab 3a: Pod hóa ba thành phần

Nhóm bài [3a](../00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời). Chưa có ConfigMap, Secret,
controller, Service, PVC.

- Khởi tạo repo, namespace, ba script.
- Pod `db`: init container ghi schema SQL vào `emptyDir` cho entrypoint của PostgreSQL; readiness
  qua TCP; ghim `lab-k8s-worker2`.
- Pod `api`: init container chờ cổng DB; startup, readiness `/ready`, liveness `/live` trên cổng
  admin; địa chỉ DB là **IP của Pod db**, chèn lúc triển khai.
- Diễn tập: tắt PostgreSQL trong Pod db — `api` NotReady nhưng không bị khởi động lại.
- Pod `web`: init container sinh trang có tên Pod và node (downward API qua env); sidecar
  `status-writer` đọc label qua volume downward API và ghi `/status.json`.
- Tắt êm cho ba thành phần: `preStop`, `terminationGracePeriodSeconds`, `STOPSIGNAL` của image.
- Diễn tập: phase và condition — `CrashLoopBackOff`, init thất bại, `Pending`.
- Gỡ lỗi `api` không có shell bằng ephemeral container.
- securityContext, user namespace, PriorityClass `bookstore-critical`.
- Sprint review: giới hạn của Pod trần, lập danh sách nợ.

**Sau sprint:** ba Pod trần chạy đúng vòng đời. Nợ mở: `TD-01` → `TD-07`.

#### Sprint 2 — Lab 3b: Cấu hình ngoài code

Nhóm bài [3b](../00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod).

- ConfigMap `web-config` (cấu hình nginx, trang HTML mẫu), `db-init` (schema), `api-config`
  (biến `PGRST_*` không bí mật, nạp bằng `envFrom`).
- Secret `db-credentials` sinh từ `secrets/db.env` bằng `secretGenerator` của Kustomize;
  `deploy.sh` chuyển sang render Kustomize ([mục 5.5](#55-bố-cục-repo-theo-sprint)).
- DB đọc mật khẩu qua file (`POSTGRES_PASSWORD_FILE`); `api` ghép URI bằng biến môi trường phụ
  thuộc `$(VAR)`.
- Điều tra: cập nhật ConfigMap lan tới Pod qua volume, env, `subPath` khác nhau thế nào;
  ConfigMap bất biến; Secret chỉ là base64.
- `command`/`args` ghi đè cho PostgreSQL; chuẩn bị `imagePullSecrets` cho registry riêng.

**Sau sprint:** không còn mật khẩu trong git. Trả `TD-02`. Nợ lab #6 vẫn mở.

#### Sprint 3 — Lab 3c: Định cỡ và chịu gián đoạn

Nhóm bài [3c](../00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn).

- Đo mức dùng thật, đặt `requests`/`limits`, đọc lại `cpu.max`, `memory.max` trên node.
- QoS: `db` Guaranteed, `web`/`api` Burstable — ghi ADR. Request hiệu dụng của init container
  và sidecar (trả nợ lab của 3a).
- Diễn tập: `OOMKilled`, Pod `Pending` vì vượt `Allocatable`.
- Resize CPU của `api` tại chỗ, không khởi động lại.
- PDB `web`, `api`; Eviction API bị PDB chặn.
- Điều tra: static Pod và mirror Pod (trả nợ lab của 3a); extended resource (tùy chọn).

**Sau sprint:** mọi container có tài nguyên khai báo. Trả `TD-05`.

#### Sprint 4 — Lab 4a: Lên Deployment

Nhóm bài [4a](../00-ALO-TRINH-ADMIN.md#4a-replicaset-deployment-và-rollout).

- `web` (3 bản sao) và `api` (2 bản sao) thành Deployment; chuỗi sở hữu Pod → ReplicaSet →
  Deployment; tự phục hồi khi xóa Pod.
- Chiến lược RollingUpdate không gián đoạn cho `web`; phát hành nội dung mới; ghi lý do thay đổi.
- Diễn tập: bản phát hành hỏng, rollout kẹt, rollback.
- Điều tra: Recreate so với RollingUpdate; `apply` so với `patch` và `scale` (trôi cấu hình);
  xóa theo tầng `orphan`.
- Canary thủ công cho `web` — dựng cấu trúc; chia lưu lượng thật cần Service, nợ sang 5a.

**Sau sprint:** `web`, `api` có controller. Trả một phần `TD-01` (còn `db`).

#### Sprint 5 — Lab 4b: Trạng thái và tác vụ nền

Nhóm bài [4b](../00-ALO-TRINH-ADMIN.md#4b-statefulset-daemonset-job-và-autoscaling).

- `db` thành StatefulSet `db` (Pod `db-0`), `serviceName: db` khai trước (Service headless là nợ
  lab #3), dữ liệu vẫn `emptyDir` (nợ lab #2).
- Job `db-migrate` có tính bất biến kết quả; `backoffLimit`, `activeDeadlineSeconds`,
  `ttlSecondsAfterFinished`.
- CronJob `db-backup`: `concurrencyPolicy`, `startingDeadlineSeconds`, giới hạn lịch sử; bản dump
  chưa có chỗ bền (`TD-08`).
- DaemonSet `node-agent` trên cả ba node (toleration cho control plane), phục vụ `/node.json`
  bằng `httpd` của busybox ở cổng 9100.
- Indexed Job xử lý song song; điều tra thứ tự của StatefulSet trên một StatefulSet thử nghiệm.
- Thiết kế HPA cho `web` dưới dạng tài liệu — nợ lab #1, thực hành ở 11b.

**Sau sprint:** mọi thành phần có controller. Trả xong `TD-01`.

#### Sprint 6 — Lab 5a: Nối các tầng

Nhóm bài [5 — phần Service/DNS](../00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng) (bài 81, 82,
83, 10, 57, 84, 86, 87, 88 và khối thực hành).

- Service headless `db` (trả nợ lab #3), `api` dùng `db-0.db`; Service `api`; nginx proxy `/api/`;
  Service `web` NodePort `30080`. Bỏ hẳn việc chèn IP lúc triển khai — trả `TD-03`.
- EndpointSlice: Pod NotReady bị gỡ khỏi Service (trả nợ lab của 3a).
- DNS: `ndots:5`, FQDN, `hostname`/`subdomain`.
- Kustomize đầy đủ, `configMapGenerator` gây rollout khi cấu hình đổi.
- Canary chia lưu lượng thật (trả nợ của 4a); Service `node-agent` với
  `internalTrafficPolicy: Local` — `status-writer` của `web` đọc `/node.json` của agent **cùng node**.
- Viết NetworkPolicy cho dự án — chưa có hiệu lực (nợ lab #4).
- Điều tra: Service `LoadBalancer` treo `Pending`; `port-forward`; đối chiếu với file compose cũ.

**Sau sprint:** trang web gọi được API qua NodePort từ máy host.

#### Sprint 7 — Lab 5b: Mạng zero-trust và HTTPS

Nhóm bài 5 — phần policy/ingress (bài 11, 12, 13, 85, 157, 183, 164).

- Chứng minh NetworkPolicy của sprint 6 không chặn gì trên Flannel.
- Vận hành: gỡ dự án, đổi CNI sang Calico, cài Helm và Traefik, chụp `02-net-ready`.
- Triển khai lại; default deny hai chiều, mở theo ma trận ở [mục 3.5](#35-thiết-kế-bảo-mật); trả
  nợ lab #4.
- Ingress `bookstore` với TLS tự ký; `web` về ClusterIP.
- Điều tra: Gateway API, dual-stack, chế độ kube-proxy, cấu hình CNI trên node.

#### Sprint 8 — Lab 6a: Dữ liệu bền

Nhóm bài [6 — phần cốt lõi](../00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ).

- Vận hành: cài local-path-provisioner, chụp `03-storage-ready`.
- StorageClass `bookstore-retain`, `volumeClaimTemplates` cho `db` (trả nợ lab #2, `TD-04`),
  `PGDATA` trong thư mục con.
- PVC `backup-store` cho `db-backup` (trả `TD-08`).
- Diễn tập: xóa Pod, xóa StatefulSet, xóa PVC — khôi phục dữ liệu từ PV `Released`.
- Projected volume cho sidecar; `emptyDir.sizeLimit` và eviction vì ephemeral storage.
- `teardown.sh` mới: dọn cả PV `Retain` và thư mục trên node.

#### Sprint 9 — Lab 6b: Chiến lược sao lưu

Phần còn lại của giai đoạn 6.

- Kiểm năng lực CSI: snapshot, VolumeAttributesClass, storage capacity, giới hạn volume theo node,
  health monitoring.
- ADR: vì sao dự án dùng backup logic `pg_dump` thay cho VolumeSnapshot trên cluster này (nợ lab
  #5 vẫn mở).
- Diễn tập khôi phục: dựng DB mới trong namespace tạm, nạp bản dump, đối chiếu dữ liệu, đo thời
  gian.

#### Sprint 10 — Lab 7a: Đặt workload đúng chỗ

Nhóm bài [7a](../00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction).

- `web` trải đều bằng topology spread; `api` anti-affinity mềm.
- `db` chuyển sang node affinity theo label riêng của dự án trên `lab-k8s-worker2`; label được
  teardown gỡ.
- PriorityClass `bookstore-batch`; diễn tập preemption trên `lab-k8s-worker2`.
- `schedulingGates` cho một Job báo cáo chờ duyệt; taint/toleration khi bảo trì.
- Điều tra: ngưỡng eviction thật của kubelet, Pod overhead, scheduler profile.

#### Sprint 11 — Lab 7b: Hạn mức namespace

Nhóm bài [7b](../00-ALO-TRINH-ADMIN.md#7b-chính-sách-giới-hạn-tài-nguyên).

- LimitRange `bookstore-limits`, ResourceQuota `bookstore-quota` theo ngân sách đã chốt ở 3c.
- Diễn tập: rollout bị quota chặn vì `maxSurge`; Pod thiếu `requests` bị từ chối.
- Điều tra: `podPidsLimit`, resource manager, tính năng do node khai báo.

#### Sprint 12 — Lab 8a: Diễn tập dựng lại cluster

[Giai đoạn 8](../00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm), bài 01–05, 09.

- Trước khi phá: bundle repo và bản dump DB ra máy host.
- Đọc cấu hình kubeadm thật, viết file `--config` vào `platform/kubeadm/`.
- Phá có kiểm soát static Pod control plane rồi khôi phục.
- `kubeadm reset`, `init --config`, join; cài lại nền theo `platform/`; triển khai từ repo; nạp
  bản dump; đo thời gian khôi phục (RTO).
- Restore `03-storage-ready` để quay về chuỗi chính.

#### Sprint 13, 14 — Lab 8b, 8c: Kiểm chứng HA

Bộ VM riêng của giai đoạn 8.

- Overlay Kustomize `ha-lab` cho cluster HA: không NetworkPolicy, không Ingress, `db` dùng
  `emptyDir`, `web` NodePort.
- Đo sẵn sàng từ máy host trong lúc tắt control plane; quan sát mất quorum.
- 8c: lặp lại với external etcd; ADR chọn topology cho production.

#### Sprint 15 — Lab 9a: Danh tính và quyền

Giai đoạn 9 — phần truy cập.

- ServiceAccount cho từng thành phần, tắt automount khi không cần.
- `db-backup` ghi kết quả vào ConfigMap `backup-status` qua API với Role tối thiểu.
- Danh tính CI `ci-deployer` với token có thời hạn và kubeconfig riêng.
- Nhóm `bookstore-devs` chỉ đọc; kiểm bằng `kubectl auth can-i --as`.
- Diễn tập: request bị chặn ở từng chặng authentication, authorization, admission.
- securityContext và sysctl an toàn.

#### Sprint 16 — Lab 9b: Hardening

Giai đoạn 9 — phần policy.

- `warn`/`audit` rồi `enforce: restricted` cho `bookstore`; sửa từng vi phạm.
- `db` non-root (uid 999), `readOnlyRootFilesystem` cho `web`, `api`; drop mọi capability;
  seccomp `RuntimeDefault`. Trả `TD-06`.
- Tenant `bookstore-staging` theo mô hình multi-tenancy.
- Rà Secret, RBAC, checklist bảo mật; ghi báo cáo vào `docs/`.

#### Sprint 17 — Lab 11a: Quan sát hệ thống

Nhóm bài [11](../00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability).

- Metric của PostgREST, kubelet, apiserver; khác biệt metric tài nguyên với metric trạng thái.
- `node-agent` chuyển sang `bookstore-system`, đọc log node qua `hostPath` chỉ đọc.
- Log hệ thống, log rotation, verbosity.
- Vận hành: cài metrics-server (hỏng vì certificate trước, sửa sau), chụp `04-metrics-ready`.
- Diễn tập chẩn đoán ứng dụng theo quy trình của bài thực hành.

#### Sprint 18 — Lab 11b: Tự co giãn

- HPA `web` theo CPU tính trên `requests`; bỏ `replicas` khỏi manifest khi HPA quản lý.
- Job sinh tải có trần; quan sát scale lên, scale xuống, `behavior`.
- Xung đột với `kubectl scale` và với `kubectl apply`. Trả nợ lab #1 phần HPA.

#### Sprint 19 — Lab 12: Bảo trì node

- Bảo trì `lab-k8s-worker1`: cordon → drain → tắt → bật → uncordon; đo sẵn sàng qua Ingress.
- Bảo trì `lab-k8s-worker2` nơi có `db` và PV cục bộ: cửa sổ bảo trì, ADR.
- Graceful node shutdown, swap, add-on, Lease; phase `Unknown` (trả nợ lab của 3a). Nợ lab #7
  vẫn mở.

#### Sprint 20 — Lab 13: Đánh giá DRA (tùy chọn)

- Tính năng giả định "gợi ý sách bằng GPU": kiểm năng lực DRA của cluster; so sánh extended
  resource, device plugin, DRA; ADR.

#### Sprint 21 — Lab 14: Operator sao lưu

Nhóm bài [14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng).

- CRD `DbBackup` có validation, `status` subresource, printer column.
- Controller dạng vòng lặp dưới ServiceAccount riêng: thấy `DbBackup` mới thì tạo Job `pg_dump`,
  cập nhật `status`.
- Chứng minh CRD không có controller chỉ là kho dữ liệu; chuyển storage version của CRD.
- **Nghiệm thu cuối dự án:** chạy toàn bộ kiểm thử, đối chiếu từng yêu cầu R1–R19, đóng sổ nợ.

---

## 8. Sổ nợ kỹ thuật dự án

Nợ kỹ thuật (`TD`) là chỗ **hệ thống** còn yếu vì chưa học tới cơ chế cần thiết. Nó khác
[nợ lab](README.md#5-sổ-nợ-lab): nợ lab là phần **bài học** chưa thực hành được. Một chỗ có thể
vừa là nợ lab vừa là nợ kỹ thuật — ví dụ dữ liệu trên `emptyDir` là `TD-04` đồng thời gắn với
nợ lab #2.

| Mã | Nợ | Phát sinh | Trả ở |
| --- | --- | --- | --- |
| `TD-01` | Pod trần: không ai tạo lại khi Pod mất | 3a | 4a (`web`, `api`), 4b (`db`) |
| `TD-02` | Mật khẩu và cấu hình nằm thẳng trong manifest | 3a | 3b |
| `TD-03` | `api` tìm `db` bằng IP Pod chèn lúc triển khai | 3a | 5a |
| `TD-04` | Dữ liệu DB nằm trên `emptyDir`, mất khi Pod bị xóa | 3a | 6a |
| `TD-05` | Chưa khai báo `requests`/`limits` | 3a | 3c |
| `TD-06` | Container `db` khởi động bằng root | 3a | 9b |
| `TD-07` | Ai `exec` hay gắn debug container được vào Pod `api` là đọc được mật khẩu trong biến môi trường | 3a | 3b (giảm), 9a (giới hạn quyền debug) |
| `TD-08` | Bản backup chưa có nơi lưu bền | 4b | 6a |
| `TD-09` | Lưu lượng giữa các tầng chưa bị giới hạn | 5a | 5b |
| `TD-10` | Bản backup vẫn nằm trong cùng cluster với DB | 6a | chấp nhận có điều kiện ở 6b: chép ra ngoài bằng tay, ghi trong runbook |
| `TD-11` | Certificate tự ký, client không tin được; đoạn sau Traefik là plaintext | 5b | chấp nhận có điều kiện ở 14 |
| `TD-12` | Sau bảo trì, `web` và `api` không tự trải đều lại trên hai worker | 7a | 12 (bước cân bằng lại trong quy trình bảo trì) |
| `TD-13` | Kubelet chưa giới hạn PID cho Pod, chưa dự trữ PID cho node | 7b | Phần II, giai đoạn 20 |
| `TD-14` | Cluster và nền dựng lại bằng tay theo `platform/README.md`; không có backup etcd | 8a | Phần II, giai đoạn 19 |
| `TD-15` | `ci-deployer` sửa được Pod template nên gián tiếp đọc được Secret, mượn được ServiceAccount | 9a | giới hạn ở 9b; chấp nhận có điều kiện ở 14 |
| `TD-16` | metrics-server bỏ kiểm certificate của kubelet (`--kubelet-insecure-tls`) | 11a | Phần II, giai đoạn 20 |
| `TD-17` | Chưa có nơi lưu metric và log dài hạn | 11a | Phần II, giai đoạn 23 |
| `TD-18` | Đường vào chưa HA: file hosts trỏ một IP, Traefik một bản sao | 12 | chấp nhận có điều kiện ở 14 |
| `TD-19` | DB một instance trên PV local của worker2: bảo trì worker2 là downtime DB | 12 | chấp nhận có điều kiện (cửa sổ bảo trì) |
| `TD-20` | `backup-operator` chạy ngoài cluster như tiến trình nền trên master | 14 | Phần II, giai đoạn 28 |

Mã trong bảng là **cố định**: khi ghi vào `docs/tech-debt.md` của repo, dùng đúng mã này để script
nghiệm thu tìm được. `docs/tech-debt.md` là bản đang sống của bảng — trạng thái, bằng chứng, ngày trả.
Lab 14 đóng sổ: mọi dòng phải ở `đã trả`, `chấp nhận có điều kiện` hoặc `chuyển Phần II`.

---

## 9. Phạm vi, ngoại lệ và quan hệ với chuỗi snapshot

- **Chuỗi snapshot không đổi.** Lab vẫn bắt đầu và kết thúc ở các mốc trong
  [bản đồ lab](README.md#4-bản-đồ-lab). Snapshot chỉ chứa **hạ tầng**; dự án luôn được teardown
  trước khi chụp.
- **Nguyên tắc không nhảy cóc giữ nguyên.** Ticket chỉ dùng kiến thức của nhóm bài hiện tại và
  các nhóm trước. Thứ còn thiếu thì thành nợ, không làm trước.
- **Fault injection chỉ trên `lab-k8s-worker2`.** Đó cũng là lý do `db` được đặt ở worker 2: mọi
  diễn tập chạm vào DB đều nằm trên node được phép phá. Bảo trì **có kế hoạch** — cordon, drain,
  tắt máy rồi bật lại theo quy trình — không phải fault injection; Lab 12 làm điều đó trên cả hai
  worker.
- **Lab 8b, 8c** chạy trên bộ VM riêng. Repo được mang sang bằng bundle và mang về bằng bundle.
- **Lab 13** là sprint tùy chọn; bỏ qua thì Lab 14 bắt đầu từ tag `lab-12`.
- **Lab 15** (node Windows) không thuộc dự án và giữ dạng lab cũ.
- **Lab 00, 1a, 1b, 1c, 2** giữ dạng lab cũ. Chúng là phần nền trước khi có workload để xây.

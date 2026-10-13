# Lab 13 — DRA

> **Điểm bắt đầu:** snapshot `04-metrics-ready` — mốc do Lab 11a tạo, xem
> [chuỗi snapshot](README.md#3-chuỗi-snapshot). Repo `~/bookstore` ở tag `lab-12` (hoặc bundle
> `~/bookstore-lab-12.bundle` nếu bạn vừa restore snapshot).
> **Điểm kết thúc:** teardown trả cluster về đúng `04-metrics-ready`, không tạo snapshot mới. Repo
> `~/bookstore` có tag `lab-13` và một bản bundle nằm ngoài VM. Sprint này **chỉ đổi `docs/`**:
> `k8s/`, `scripts/`, `tests/` và `platform/` giữ nguyên như ở tag `lab-12`.
> **Sprint dự án:** Sprint 20 (tùy chọn) — Đánh giá DRA của [dự án Bookstore](DU-AN-BOOKSTORE.md).
> **Sprint tùy chọn.** Bỏ qua lab này thì [Lab 14](LAB-14-CRD-VA-OPERATOR.md) bắt đầu thẳng từ tag
> `lab-12`. Hệ thống không thiếu gì vì tag `lab-13` chỉ thêm tài liệu.
> **Lab trước:** [Lab 12 — Vận hành vòng đời node](LAB-12-VAN-HANH-VONG-DOI-NODE.md).
> **Lab sau:** [Lab 14 — CRD và Operator](LAB-14-CRD-VA-OPERATOR.md).
>
> **Cập nhật và đối chiếu phiên bản:** 13/10/2026.

Lab này đi cùng
[Giai đoạn 13 — Lập lịch và workload nâng cao](../00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao):
mười lăm bài khái niệm và năm bài thực hành liệt kê ở mục 1.1. Mọi ticket dựa vào hợp đồng kỹ thuật
ở [mục 4 của file dự án](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật). Kế hoạch gốc của sprint nằm ở
[mục 7.2](DU-AN-BOOKSTORE.md#sprint-20--lab-13-đánh-giá-dra-tùy-chọn), yêu cầu tương ứng là `R18`
ở [mục 2.2](DU-AN-BOOKSTORE.md#22-yêu-cầu-của-hệ-thống-mới).

**Bối cảnh của sprint.** Đội sản phẩm đề xuất hai thứ cho quý sau:

1. Tính năng **gợi ý sách bằng GPU** — một dịch vụ suy luận đứng cạnh `api`, mỗi bản sao cần một
   GPU riêng.
2. Một batch đêm **huấn luyện mô hình gợi ý** gồm ba worker. Ba worker phải chạy **cùng lúc** mới
   tiến triển được; hai worker chạy mà thiếu worker thứ ba thì chỉ ngồi giữ tài nguyên.

Trước khi ai mua GPU, đội vận hành phải trả lời bằng bằng chứng: cluster làm được gì hôm nay, thiếu
chính xác cái gì, nếu làm thì làm bằng cơ chế nào, và làm sao giữ nó an toàn. Đây là sprint **điều
tra**: không ticket nào thêm workload vào hệ thống. Sản phẩm là ADR và các bản thiết kế dạng nháp
trong `docs/`, mọi kết luận dẫn về file bằng chứng trong `~/lab-evidence/13/`.

**Phiên bản tài liệu khác phiên bản cluster.** Phần lớn bài dịch của nhóm này là bản tài liệu
mới nhất của kubernetes.io — nhiều mục ghi `Kubernetes v1.36 [alpha]` hoặc `[beta]`. Cluster chạy
đúng baseline ở [bảng A1.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa), và
lab này không chép lại con số đó. Đừng suy tính năng từ tài liệu: **đo trên cluster**. Đó là kỹ năng
chính của sprint, và là lý do mọi ticket đều có phép đo trước khi có kết luận.

Ba VM của [Lab 00](LAB-00-MOI-TRUONG-1.35.7.md) không có GPU hay thiết bị chuyên dụng nào. Lab
không cài driver, không bật feature gate, không sửa cấu hình control plane. B1 đo năng lực thật
rồi rẽ nhánh; nhánh dành cho cluster không có thiết bị vẫn kiểm chứng được phần lớn nhóm bài.

Quan hệ với các sprint trước, để không làm lại:
[Lab 3c](LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md) đã quảng bá và tiêu thụ một extended resource thật —
đó là con đường thứ nhất mà B4 đem ra so sánh. [Lab 7a](LAB-7A-LAP-LICH-VA-EVICTION.md) đã làm
PriorityClass, preemption, chu trình filter/score; B5 chỉ đo phần nhóm PodGroup *thêm vào*.
[Lab 7b](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md) hoãn mục *Hạn ngạch cho DRA resource claim* của bài
134 tới giai đoạn này; B7 trả phần đó. [Lab 9a](LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md) dựng
danh tính `ci-deployer` và nhóm `bookstore-devs` mà B6, B7 dùng để kiểm quyền.
[Lab 12](LAB-12-VAN-HANH-VONG-DOI-NODE.md) dựng quy trình bảo trì node mà ADR ở B8 phải bổ sung.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`. B0 bên dưới chạy lại đúng gate đó.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi `docs/` của repo có ADR chốt quyết định về DRA cho Bookstore, mọi kết luận dẫn
về bằng chứng, **và** bạn chứng minh, giải thích được:

- Năng lực DRA thật của cluster: nhóm API `resource.k8s.io`, DeviceClass, ResourceSlice, driver,
  feature gate — kết luận bằng số đo, kể cả khi kết luận là "chưa dùng được", và chỉ ra chính xác
  điều kiện nào thiếu.
- Bốn kind API của DRA: ai tạo cái nào, phạm vi của từng cái đọc từ chính cluster, và vì sao phạm
  vi đó cũng là ranh giới phân quyền.
- Vì sao một DeviceClass mới tạo không sinh ra thiết bị nào, và ai mới là bên công bố phần cứng.
- Pod của tính năng gợi ý dừng ở **hai bước khác nhau** của luồng cấp phát khi claim chưa tồn tại
  và khi claim tồn tại nhưng không có thiết bị — phân biệt bằng trạng thái object.
- ResourceClaim so với ResourceClaimTemplate: ai sinh claim, ở đâu trong control plane, vòng đời
  của mỗi loại; vì sao **sinh claim** khác **cấp phát thiết bị**.
- Ba con đường đưa thiết bị vào Pod — extended resource, device plugin, DRA — nằm ở đâu trong Pod
  spec, hạch toán ở đâu, và vì sao scheduler báo Pending bằng hai thông điệp khác nhau.
- "API có trường" khác "tính năng đang bật": API server âm thầm gỡ trường của tính năng chưa bật.
- Batch huấn luyện cần gang scheduling vì sao; khởi động một phần trông ra sao trên
  `lab-k8s-worker2`; cluster thiếu chính xác những gì để dùng PodGroup.
- Cấu hình kube-scheduler đang chạy so với khuyến nghị của bài 124, và vì sao quyền gán nhãn node
  là vấn đề bảo mật của chính Bookstore — nơi `db` chạy phụ thuộc vào một nhãn node.
- Mô hình quyền của DRA ở bài 125 và 211: điều kiện phiên bản của nó so với cluster; vì sao
  `kubectl auth can-i` trả `yes` không chứng minh một subresource tồn tại; hạn mức thiết bị theo
  DeviceClass chặn ngay lúc tạo claim, trước mọi lần cấp phát.
- Quyết định "Bookstore có nên dùng DRA, và khi nào" với điều kiện xem lại đo được.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 13 | Ticket kiểm chứng |
| --- | --- |
| [149 — Cấp phát tài nguyên động](../149-dynamic-resource-allocation-vi.md) | B1 — bốn kind, phạm vi, bước 1 của luồng; B2 — DeviceClass là danh mục, ResourceSlice là nguồn cung, luồng năm bước tới điểm dừng; B3 — ResourceClaim so với ResourceClaimTemplate; B4 — lợi ích của DRA, ba thiếu sót của device plugin, trường `extendedResourceName` bị gỡ khi gate tắt |
| [172 — Thực hành tốt về DRA cho quản trị viên](../172-cluster-admin-dra-vi.md) | B3 — controller ResourceClaim nằm trong kube-controller-manager; B4 ⭐ — metric `dra_operations_duration_seconds` của kubelet; B7 — tách quyền bốn API cho danh tính của dự án; B8 — driver dạng DaemonSet, nâng cấp liền mạch, drain driver muộn nhất vào quy trình bảo trì |
| [125 — Hardening DRA](../125-hardening-dra-vi.md) | B7 — `resourceclaims/binding` và `resourceclaims/driver`, hai tiền tố verb, bản nháp ClusterRole cho driver giả định |
| [59 — Nhóm lập lịch](../59-scheduling-group-vi.md) | B5 — trường liên kết Pod với nhóm không có trong schema Pod của cluster |
| [75 — PodGroup API](../75-podgroup-api-vi.md) | B5 — kind PodGroup không được phục vụ; API server không nhận bản nháp PodGroup |
| [76 — Vòng đời của PodGroup](../76-podgroup-lifecycle-vi.md) | B5 — thứ tự Workload → PodGroup → Pod trong bản nháp; câu hỏi thiết kế về `minCount` bất biến và `schedulerName` chung |
| [77 — Workload API](../77-workload-api-vi.md) | B5 — kind Workload không được phục vụ; gate `WorkloadWithJob`; Job Indexed có `parallelism` bằng `completions` là thiết kế của batch |
| [78 — Gián đoạn và độ ưu tiên của PodGroup](../78-workload-disruption-priority-vi.md) | B5 — không có PriorityClass `globalDefault`; `bookstore-batch` ở cấp nhóm chỉ có tác dụng trong workload-aware preemption |
| [79 — Các chính sách lập lịch PodGroup](../79-workload-policies-vi.md) | B5 — `gang` với `minCount: 3` cho batch, so với `basic` |
| [80 — Lập lịch nhận biết topology (Workload API)](../80-workload-topology-scheduling-vi.md) | B5 — cluster chỉ có trục topology `kubernetes.io/hostname`; không gán nhãn zone |
| [150 — Lập lịch theo nhóm](../150-gang-scheduling-vi.md) | B5 — thí nghiệm khởi động một phần trên `lab-k8s-worker2`: đúng tình huống mà gang ngăn |
| [151 — Lập lịch PodGroup](../151-podgroup-scheduling-vi.md) | B5 — kube-scheduler không nạp file cấu hình; hạn chế với nhóm không đồng nhất là câu hỏi thiết kế |
| [152 — Preemption nhận biết workload](../152-workload-aware-preemption-vi.md) | B5 — gate `WorkloadAwarePreemption`; đường preemption mặc định là đường duy nhất của cluster |
| [153 — Lập lịch nhận biết topology (scheduling)](../153-topology-aware-scheduling-vi.md) | B5 — plugin TAS chỉ cấu hình được qua `KubeSchedulerConfiguration`; B6 — cờ `--config` |
| [124 — Hardening scheduler](../124-hardening-scheduler-vi.md) | B6 — mười hai cờ đọc từ manifest thật, quyền file kubeconfig, không danh tính nào của dự án gán được nhãn node |
| [211 — Tăng cường bảo mật DRA trong cluster](../211-hardening-dra-tasks-vi.md) | B7 — bước *Xác định các thành phần DRA ghi vào status* trên cluster thật, điều kiện phiên bản ở *Trước khi bạn bắt đầu*, *Gắn role với các danh tính tường minh* trong bản nháp |
| [268 — Gán thiết bị cho Pod và Container](../268-assign-resources-vi.md) | Trang mục lục, không có thao tác riêng. Thứ tự vai trò của nó — 271 rồi 270 rồi 269 — là thứ tự B1 → B2, B3 |
| [269 — Truy cập metadata thiết bị DRA](../269-access-dra-device-metadata-vi.md) | B3 ⭐ — vế phủ định: container không yêu cầu thiết bị thì không có thư mục metadata. Vế khẳng định cần thiết bị thật và phiên bản server mà bài đòi |
| [270 — Cấp phát thiết bị cho workload bằng DRA](../270-allocate-devices-dra-vi.md) | B2 — claim tham chiếu trực tiếp, hai chỗ khai báo trong Pod spec, claim chưa tồn tại; B3 — ResourceClaimTemplate |
| [271 — Thiết lập DRA trong một cluster](../271-set-up-dra-cluster-vi.md) | B1 — mục *Xác minh rằng DRA đã được bật*, `kubectl get resourceslices` là phép xác minh driver; B2 — tạo DeviceClass bằng selector CEL; đóng sprint — mục *Dọn dẹp* |

Ngoài nhóm 13, B7 đọc và thực hành mục
[*Hạn ngạch cho DRA resource claim*](../134-resource-quotas-vi.md#hạn-ngạch-cho-dra-resource-claim-quota-for-dra-resource-claims)
của bài 134 — mục mà Lab 7b đã cố ý để lại cho giai đoạn này.

Các phần dưới đây **không kiểm chứng được** trên cluster này, ghi rõ lý do thay vì bỏ trống:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| Bài 149, mục *Hạn chế* — scheduler không preempt cho tài nguyên DRA | Cần hai Pod tranh nhau một thiết bị đã cấp phát. Không thiết bị thì không có tranh chấp, và không có cách tạo tranh chấp mà không dựng thiết bị giả. B2 đưa nó vào câu hỏi thiết kế |
| Bài 149, toàn bộ *Các tính năng beta của DRA* và *Các tính năng alpha của DRA* | Mỗi mục cần một feature gate riêng, có khi thêm một nhóm API riêng, cộng một driver công bố ResourceSlice đúng cấu trúc. Bật gate là sửa control plane — việc của [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy). B1 đo trạng thái gate; B4 chứng minh một trường alpha bị gỡ |
| Bài 149, *Khả năng quan sát tài nguyên động* và *Pod được lập lịch sẵn* | Ba nguồn quan sát đều do driver ghi. Pod đặt sẵn `nodeName` mà kẹt claim thì giữ CPU, RAM trên node trong lúc chờ — đúng điều bài cảnh báo; Bookstore không bao giờ đặt `nodeName` (dùng `nodeSelector` từ 3a, affinity từ 7a) |
| Bài 172, truy vấn PromQL và tinh chỉnh QPS/burst | Cần stack giám sát và tải thật; histogram rỗng không phải kiểm chứng. Đường đọc `/metrics` có ủy quyền đã làm ở [Lab 11a](LAB-11A-OBSERVABILITY.md) |
| Bài 172, nâng cấp liền mạch, liveness probe của driver, thứ tự drain driver | Ba việc đều thao tác trên một DaemonSet driver đang chạy. B8 đưa chúng vào quy trình bảo trì trong ADR |
| Bài 125 và 211, áp ClusterRole thật rồi roll out thành phần DRA | Role chỉ có nghĩa khi gắn vào ServiceAccount của một driver thật, và bài 211 đòi phiên bản server cao hơn baseline. B7 đọc quyền thật đang có, viết bản nháp và kiểm bằng server dry-run |
| Bài 211, theo dõi audit event | Cluster chưa bật audit policy — [giai đoạn 22](../00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu) |
| Bài 59, 75–80, 150–153, phần **chạy thật** PodGroup và Workload | Cần nhóm API alpha và feature gate; cluster còn ở phiên bản thấp hơn phiên bản mà tài liệu mô tả. B5 đo bề mặt API, bảng gate, và dựng lại đúng tình huống mà gang scheduling giải |
| Bài 124, `KubeSchedulerConfiguration` tắt plugin cho scheduler tùy chỉnh | Phải chạy một profile scheduler khác, tức sửa manifest control plane |
| Bài 269 vế khẳng định, bài 270 phần cấp phát thật | Chỉ chạy ở nhánh A, trên cluster có thiết bị và driver |

Sprint này **không phát sinh nợ lab**. [Sổ nợ lab](README.md#5-sổ-nợ-lab) ghi phần bị chặn bởi
*kiến thức của giai đoạn sau*; ở đây không thiếu kiến thức, chỉ thiếu phần cứng — và đó là lý do
lab được đánh dấu tùy chọn. Riêng [bài 184 — Device Plugin](../184-device-plugins-vi.md) thuộc
giai đoạn 14 và bạn chưa đọc: B4 so sánh device plugin chỉ bằng những gì bài 149 và bài 110 nói.
Khi đọc bài 184, quay lại bảng so sánh trong ADR.

Sprint này cũng không trả và không phát sinh nợ kỹ thuật của ứng dụng (`TD-nn`). Phát hiện về cấu
hình kube-scheduler ở B6 thuộc hạ tầng cluster; nó được ghi vào ADR và trỏ từ `docs/tech-debt.md`.

### 1.2. Thời lượng

4–5 giờ. Phần cốt lõi (B0–B8, bỏ các mục ⭐) khoảng 4 giờ; nhánh A dài hơn nhánh B khoảng nửa giờ.
Có thể chia hai phiên: phiên 1 làm B0–B4, phiên 2 làm B5–B9. Giữa hai phiên cứ để nguyên cluster
và namespace `lab-13`; đầu phiên 2 chạy lại gate ngắn A5.5 và `./tests/smoke.sh`.

Mọi bước chờ controller hay scheduler viết dưới dạng vòng lặp có điều kiện dừng. Thời gian chờ
thực tế **phụ thuộc cấu hình** cluster.

---

## 2. Quy ước và an toàn

- Mọi lệnh chạy trên `lab-k8s-master` trừ khi ghi rõ node khác. Lệnh `sudo` chỉ dùng để **đọc**
  file trong `/etc/kubernetes` của chính `lab-k8s-master`.
- Repo dự án: `~/bookstore`. Manifest thử nghiệm: `~/lab-work/13/`. Bằng chứng: `~/lab-evidence/13/`.
- **Repo chỉ đổi `docs/`.** Bản thiết kế dạng nháp đặt ở `docs/dra/*.draft.yaml` — đuôi `.draft` và
  vị trí ngoài `k8s/` bảo đảm `scripts/deploy.sh` không bao giờ áp chúng, giống
  `docs/hpa-web.draft.yaml` của Lab 4b. Bản nháp được kiểm bằng `--dry-run=server`, không apply.
- Object thử nghiệm nằm trong namespace **`lab-13`** và mang label `lab=lab-13`, **không** mang label
  `app.kubernetes.io/part-of`. Namespace riêng để thí nghiệm không đụng quota, LimitRange và
  NetworkPolicy của `bookstore`. Object cấp cluster duy nhất mà lab tạo là DeviceClass `lab-13-gpu`
  (label `lab=lab-13`); đóng sprint xóa nó.
- **Lab tuyệt đối không cài và không sửa:** không DRA driver, không device plugin, không CRD, không
  bật feature gate hay nhóm API, không sửa file trong `/etc/kubernetes/manifests`, không sửa cấu
  hình kubelet, không gán nhãn node, không tạo ResourceSlice bằng tay, không tạo Role hay
  ClusterRole thật. Thiếu điều kiện thì ghi nhận thiếu, không đi vá.
- **Thí nghiệm có tải chỉ chạy trên `lab-k8s-worker2`** (B5). Pod thí nghiệm không đặt
  `priorityClassName`, nên mang độ ưu tiên 0 và không chiếm chỗ của Pod nào.
- B0 chụp **ảnh nền** của cluster vào `~/lab-evidence/13/b0-anh-nen.env` trước khi triển khai dự án.
  Gate cuối so từng giá trị với ảnh nền đó; các script sau đọc lại file bằng lệnh `.`, nên mở shell
  mới giữa chừng không làm mất gate.
- Image dùng cho Pod thử nghiệm là `busybox:1.37`, tag lấy từ
  [A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa).
- Dòng `PASS:` là gate; không đi tiếp khi gate fail. Dòng `FAIL:` nghĩa là ticket chưa xong. Dòng
  `GHI NHAN:` là kết quả **hợp lệ nhưng khác mặc định** — giữ nguyên văn trong evidence, đọc lại
  phần Lời giải trước khi kết luận. Dòng `STOP:` nghĩa là dừng đúng bước đó và ghi lại.
- **Nhánh A** (cluster có thiết bị): các dòng `PASS:` về `status.allocation` rỗng đảo ngược. Script
  của B2 có khối riêng cho nhánh A; các ticket khác ghi rõ chỗ khác biệt.
- **Quay lui khi hỏng:** tắt **cả ba VM**, restore **cả ba** về `04-metrics-ready`, bật lại theo thứ
  tự master → worker 1 → worker 2, clone lại repo từ `~/bookstore-lab-12.bundle`, chạy lại từ B0.

---

# Phần B — Sprint 20: Đánh giá DRA (tùy chọn)

## B0. Nhận bàn giao sprint

**Mục đích:** xác nhận cluster ở đúng `04-metrics-ready`, repo ở đúng tag `lab-12`, chụp ảnh nền
của cluster **trước** khi có dự án, rồi dựng lại hệ thống của sprint 19 chỉ từ repo.

Gate ngắn A5.5, chạy nguyên văn:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

Mốc `04-metrics-ready` nghĩa là metrics-server trả số liệu cho cả ba node:

```bash
test "$(kubectl top node --no-headers 2>/dev/null | wc -l)" -eq 3 \
  && echo 'PASS: metrics-server tra so lieu ca ba node'
```

Repo và thư mục làm việc. Nếu vừa restore snapshot, repo đã mất — khôi phục từ bundle theo
[mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm) trước:

```bash
mkdir -p ~/lab-work/13 ~/lab-evidence/13
test -d ~/bookstore/.git || git clone ~/bookstore-lab-12.bundle ~/bookstore
cd ~/bookstore
git describe --tags --exact-match HEAD 2>/dev/null | grep -qx 'lab-12' && echo 'PASS: HEAD dung tag lab-12'
test -z "$(git status --porcelain)" && echo 'PASS: repo khong co thay doi chua commit'
test "$(kubectl get namespace -l app.kubernetes.io/part-of=bookstore --no-headers 2>/dev/null | wc -l)" -eq 0 \
  && echo 'PASS: cluster chua co du an' || echo 'FAIL: con namespace du an tu lan truoc'
```

Ảnh nền của cluster — chụp **trước** `deploy.sh`, vì gate cuối so với trạng thái sau teardown:

```bash
EV=~/lab-evidence/13
kubectl get --raw /metrics > "$EV/b0-apiserver-metrics.txt"
{
  echo "API_BEFORE=$(kubectl api-resources -o name 2>/dev/null | sort -u | wc -l)"
  echo "CRD_BEFORE=$(kubectl get crd -o name 2>/dev/null | wc -l)"
  echo "FG_ALL_BEFORE=$(grep -c '^kubernetes_feature_enabled{' "$EV/b0-apiserver-metrics.txt")"
  echo "FG_ON_BEFORE=$(grep '^kubernetes_feature_enabled{' "$EV/b0-apiserver-metrics.txt" | grep -c ' 1$')"
  echo "NS_BEFORE='$(kubectl get namespace -o name | sort | tr '\n' ' ')'"
  echo "PC_BEFORE='$(kubectl get priorityclass -o name | sort | tr '\n' ' ')'"
  echo "LBL_BEFORE=$(kubectl get nodes --no-headers --show-labels | awk '{print $1, $NF}' | sort | md5sum | cut -d' ' -f1)"
  echo "DC_BEFORE=$(kubectl get deviceclasses --no-headers 2>/dev/null | wc -l)"
  echo "SLICE_BEFORE=$(kubectl get resourceslices --no-headers 2>/dev/null | wc -l)"
} > "$EV/b0-anh-nen.env"
cat "$EV/b0-anh-nen.env"
. "$EV/b0-anh-nen.env"
test "$FG_ALL_BEFORE" -gt 0 && echo "PASS: apiserver cong bo $FG_ALL_BEFORE feature gate, dang bat $FG_ON_BEFORE"
test "$(grep -c '=' "$EV/b0-anh-nen.env")" -eq 9 && echo 'PASS: anh nen du chin gia tri'
```

Dựng lại hệ thống của sprint 19 bằng script của chính sprint đó:

```bash
cd ~/bookstore
./scripts/deploy.sh > ~/lab-evidence/13/b0-deploy.txt 2>&1; echo "deploy exit=$?"
./tests/smoke.sh > ~/lab-evidence/13/b0-smoke-lab-12.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/13/b0-smoke-lab-12.txt
```

Namespace thử nghiệm của sprint, và image mà mọi Pod thử nghiệm dùng:

```bash
kubectl create namespace lab-13 --dry-run=client -o yaml | kubectl apply -f -
kubectl label namespace lab-13 lab=lab-13 --overwrite
test "$(kubectl get namespace lab-13 -o jsonpath='{.status.phase}')" = 'Active' \
  && test -z "$(kubectl get namespace lab-13 -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}')" \
  && echo 'PASS: namespace lab-13 Active, khong mang label du an'
for N in lab-k8s-worker1 lab-k8s-worker2; do
  ssh "$N" "sudo crictl pull docker.io/library/busybox:1.37" >/dev/null \
    && echo "PASS: $N co busybox:1.37" || echo "FAIL: $N khong keo duoc busybox:1.37"
done
```

**PASS:** gate A5.5 đạt như mô tả ở Lab 00; dòng `PASS: metrics-server...`; ba dòng `PASS:` về repo
và cluster; hai dòng `PASS:` của ảnh nền; `deploy exit=0`, `smoke exit=0`, file smoke không có dòng
`FAIL:`; namespace `lab-13` sẵn sàng; hai dòng `PASS: ... co busybox:1.37`.

`deploy.sh` hoặc `smoke.sh` fail nghĩa là repo của sprint 19 **chưa đủ**. Sửa repo ở tag `lab-12`
trước, không sửa tay trên cluster ([mục 6.1 của file dự án](DU-AN-BOOKSTORE.md#61-mở-đầu)).

## B1. [BS-13-01] Điều tra: năng lực DRA của cluster

**Loại:** Điều tra.

**Bối cảnh:** câu hỏi đầu tiên của tính năng gợi ý không phải "viết YAML thế nào" mà là "cluster
này có cấp phát được thiết bị không". Trả lời bằng lệnh, không bằng trí nhớ, và không bằng tài
liệu — tài liệu của nhóm bài này mô tả một phiên bản Kubernetes mới hơn cluster của bạn.

**Yêu cầu:**

1. Ghi phiên bản server và đối chiếu với phiên bản mà các mục *Trạng thái tính năng* trong bài dịch
   nhắc tới.
2. Xác định nhóm API của DRA có được phục vụ không, ở những version nào; bốn kind của nó và phạm vi
   (cấp cluster hay cấp namespace) của từng kind — đọc từ cluster, không chép từ bài.
3. Năm phép kiểm năng lực: nhóm API được phục vụ; có DeviceClass nào; có ResourceSlice nào; driver
   nào đang công bố thiết bị; feature gate liên quan đang ở trạng thái nào.
4. Lập bảng mười ba feature gate mà bài 149 và bài 271 nhắc tên: API server **biết tên** gate nào,
   đang **bật** gate nào. Đối chiếu thêm: control plane có khai cờ feature gate riêng không, kubelet
   trên `lab-k8s-worker2` có khai gate riêng không.
5. Chốt nhánh. **Nhánh A** khi nhóm API được phục vụ **và** có ít nhất một ResourceSlice **và** có
   ít nhất một driver công bố thiết bị; thiếu một điều kiện là **nhánh B**. Ghi kết quả vào
   `~/lab-evidence/13/b1-nang-luc.env` để các ticket sau đọc lại.
6. Mở ADR `docs/adr/NNNN-dra-bookstore.md` (số tiếp theo trong `docs/adr/`) với mục *Năng lực
   cluster*.

**Ràng buộc:** chỉ đọc. Không bật nhóm API hay feature gate, không cài driver, không tạo
ResourceSlice để "có dữ liệu". Nhánh B là kết quả hợp lệ, không phải thất bại của sprint.

**Đọc lại:**

- Bài [271](../271-set-up-dra-cluster-vi.md):
  [*Xác minh rằng DRA đã được bật*](../271-set-up-dra-cluster-vi.md#verify),
  [*Cài đặt driver thiết bị*](../271-set-up-dra-cluster-vi.md#install-drivers) — phép xác minh driver,
  [*Tùy chọn: bật thêm các API group của DRA*](../271-set-up-dra-cluster-vi.md#enable-dra).
- Bài [149](../149-dynamic-resource-allocation-vi.md):
  [*Thuật ngữ DRA*](../149-dynamic-resource-allocation-vi.md#terminology),
  [*Các nhóm người dùng DRA*](../149-dynamic-resource-allocation-vi.md#dra-user-types),
  [*Luồng công việc của Kubernetes*](../149-dynamic-resource-allocation-vi.md#kubernetes-workflow) —
  bước 1.
- Bài [268](../268-assign-resources-vi.md) — ba vai trò và thứ tự phụ thuộc giữa chúng.
- [Lab 11a](LAB-11A-OBSERVABILITY.md) — `/metrics` của API server; metric `kubernetes_feature_enabled`
  là bảng feature gate do chính binary công bố.

**Bàn giao:** mục *Năng lực cluster* trong `docs/adr/NNNN-dra-bookstore.md`;
`~/lab-evidence/13/b1-*`.

**Nghiệm thu:**

- `PASS:` đọc được phiên bản server.
- `PASS:` DeviceClass và ResourceSlice ở cấp cluster; ResourceClaim và ResourceClaimTemplate ở cấp
  namespace — đọc từ cluster.
- `PASS:` đủ mười ba dòng gate, mỗi dòng kết luận `0`, `1` hoặc "không biết tên".
- `PASS:` hai phép đối chiếu cấu hình in `PASS:` hoặc `GHI NHAN:` và đã vào evidence.
- `PASS:` file `b1-nang-luc.env` chốt `NHANH=A` hoặc `NHANH=B` khớp với số đo.
- `PASS:` ADR có mục *Năng lực cluster*.

**Câu hỏi thiết kế** — trả lời trong mục *Năng lực cluster* của ADR:

1. `kubectl get deviceclasses` in `No resources found`. DRA của cluster hỏng hay chạy? Output nào
   mới là output của cluster chưa bật DRA, và nó chỉ ra chuyện gì?
2. Cluster của bạn thiếu điều kiện nào trong bốn điều kiện của nhánh A, và thiếu vì lý do gì —
   phần cứng, feature gate, hay phiên bản? Vì sao câu trả lời đó **không** tạo ra nợ lab?
3. Kể ít nhất ba gate mà bài 149 ghi là beta hoặc bật mặc định, nhưng cluster của bạn đang tắt hoặc
   không biết tên. Điều đó nói gì về việc viết manifest theo tài liệu "bản mới nhất"?

<details>
<summary>Gợi ý</summary>

1. `kubectl version` in một dòng `Server Version:`. `kubectl api-versions` liệt kê mọi group/version
   đang được phục vụ.
2. `kubectl api-resources --api-group=resource.k8s.io -o wide` có cột `NAMESPACED`. Dạng kiểm được
   bằng script: `kubectl api-resources --namespaced=false -o name`.
3. Tên driver nằm ở `.spec.driver` của mỗi ResourceSlice.
4. B0 đã lưu `/metrics` của API server. Mỗi gate là một dòng
   `kubernetes_feature_enabled{name="<gate>",stage="<giai đoạn>"} 0|1`. Không có dòng nghĩa là
   binary không biết tên gate đó.
5. Cấu hình hiệu lực của kubelet đọc qua node proxy: `kubectl get --raw
   /api/v1/nodes/lab-k8s-worker2/proxy/configz`. Cờ của control plane nằm trong manifest static Pod
   ở `/etc/kubernetes/manifests/`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
EV=~/lab-evidence/13; W2=lab-k8s-worker2
. "$EV/b0-anh-nen.env"

# 1. Phiên bản server
SV="$(kubectl version 2>/dev/null | sed -n 's/^Server Version: //p')"
echo "server=$SV" > "$EV/b1-phien-ban.txt"
test -n "$SV" && echo "PASS: doc duoc phien ban server ($SV)"

# 2. Nhóm API, bốn kind, phạm vi
kubectl api-versions | grep '^resource\.k8s\.io/' > "$EV/b1-api-versions.txt"
kubectl api-resources --api-group=resource.k8s.io -o wide > "$EV/b1-api-resource.txt" 2>&1
cat "$EV/b1-api-versions.txt" "$EV/b1-api-resource.txt"
DRA_API=0; grep -qx 'resource.k8s.io/v1' "$EV/b1-api-versions.txt" && DRA_API=1
sc() { kubectl api-resources --namespaced="$1" -o name 2>/dev/null | grep -qx "$2.resource.k8s.io"; }
{ sc false deviceclasses && sc false resourceslices && sc true resourceclaims && sc true resourceclaimtemplates; } \
  && echo 'PASS: DeviceClass, ResourceSlice cap cluster; ResourceClaim, ResourceClaimTemplate cap namespace'

# 3. Danh mục, nguồn cung, driver
kubectl get deviceclasses > "$EV/b1-deviceclass.txt" 2>&1
kubectl get resourceslices -o wide > "$EV/b1-resourceslice.txt" 2>&1
kubectl get resourceslices -o jsonpath='{range .items[*]}{.spec.driver}{"\n"}{end}' 2>/dev/null \
  | sed '/^$/d' | sort -u > "$EV/b1-drivers.txt"
cat "$EV/b1-deviceclass.txt" "$EV/b1-resourceslice.txt" "$EV/b1-drivers.txt"
DC_N="$(kubectl get deviceclasses --no-headers 2>/dev/null | wc -l)"
SLICE_N="$(kubectl get resourceslices --no-headers 2>/dev/null | wc -l)"
DRV_N="$(wc -l < "$EV/b1-drivers.txt")"

# 4. Bảng feature gate — đọc từ /metrics của API server đã lưu ở B0
for g in DynamicResourceAllocation DRAAdminAccess DRAResourceClaimDeviceStatus ResourceHealthStatus \
         DRAExtendedResource DRAPartitionableDevices DRADeviceTaints DRADeviceTaintRules \
         DRADeviceBindingConditions DRAResourcePoolStatus DRANodeAllocatableResources \
         DRAListTypeAttributes DRAWorkloadResourceClaims; do
  v="$(grep "^kubernetes_feature_enabled{name=\"$g\"," "$EV/b0-apiserver-metrics.txt" | awk '{print $NF}' | head -1)"
  echo "$g ${v:-khong-biet-ten}"
done > "$EV/b1-feature-gates.txt"
cat "$EV/b1-feature-gates.txt"
{ test "$(wc -l < "$EV/b1-feature-gates.txt")" -eq 13 && \
  test "$(grep -cE ' (0|1|khong-biet-ten)$' "$EV/b1-feature-gates.txt")" -eq 13; } \
  && echo 'PASS: du 13 gate, moi gate co ket luan'
DRA_GATE="$(awk '$1=="DynamicResourceAllocation"{print $2}' "$EV/b1-feature-gates.txt")"

sudo grep -n -- '--feature-gates' /etc/kubernetes/manifests/*.yaml > "$EV/b1-manifest-gates.txt" 2>/dev/null
if [ -s "$EV/b1-manifest-gates.txt" ]; then
  echo 'GHI NHAN: co thanh phan control plane khai --feature-gates'; cat "$EV/b1-manifest-gates.txt"
else
  echo 'PASS: control plane khong khai --feature-gates — dung mac dinh cua binary'
fi
kubectl get --raw "/api/v1/nodes/$W2/proxy/configz" > "$EV/b1-configz-$W2.json" 2>/dev/null
if grep -q '"featureGates"' "$EV/b1-configz-$W2.json"; then
  echo "GHI NHAN: kubelet tren $W2 khai featureGates rieng"
else
  echo "PASS: kubelet tren $W2 khong khai featureGates rieng"
fi

# 5. Chốt nhánh
NHANH=B
[ "$DRA_API" -eq 1 ] && [ "$SLICE_N" -ge 1 ] && [ "$DRV_N" -ge 1 ] && NHANH=A
printf 'SV=%s\nDRA_API=%s\nDC_N=%s\nSLICE_N=%s\nDRV_N=%s\nDRA_GATE=%s\nNHANH=%s\n' \
  "$SV" "$DRA_API" "$DC_N" "$SLICE_N" "$DRV_N" "$DRA_GATE" "$NHANH" > "$EV/b1-nang-luc.env"
cat "$EV/b1-nang-luc.env"
grep -qE '^NHANH=(A|B)$' "$EV/b1-nang-luc.env" && echo "PASS: chot nhanh $NHANH bang so do"

# 6. ADR
cd ~/bookstore
ADR="$(ls docs/adr/*-dra-bookstore.md 2>/dev/null | head -1)"
{ test -n "$ADR" && grep -q 'Năng lực cluster' "$ADR"; } && echo "PASS: $ADR co muc Nang luc cluster"
```

Nếu `DRA_API=0`: đó đúng là tình huống mà bài 271 mô tả — nhóm API không được phục vụ. Ghi lại,
bỏ qua phần DRA của B2, B3, B4, B7 và đi tiếp từ B5. Không bật nhóm API.

</details>

<details>
<summary>Lời giải tham khảo</summary>

Không có manifest nào: ticket chỉ đọc. Kết quả kỳ vọng trên ba VM của Lab 00, nếu baseline giữ
nguyên:

| Đo | Kỳ vọng | Ý nghĩa |
| --- | --- | --- |
| `resource.k8s.io` | chỉ `resource.k8s.io/v1` | DRA đã stable; bài 271 nói v1beta1, v1beta2, v1alpha3 chỉ bật khi thật sự cần |
| DeviceClass | `No resources found` | Chưa ai tạo danh mục — không phải lỗi |
| ResourceSlice, driver | 0 và 0 | Không driver nào công bố phần cứng; bước 1 của luồng chưa từng xảy ra |
| Gate | 9 gate biết tên, 4 gate không biết tên; bật `DynamicResourceAllocation`, `DRAAdminAccess`, `DRAResourceClaimDeviceStatus` | Đối chiếu [bảng feature gate của tài liệu đúng phiên bản baseline](https://v1-35.docs.kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) |
| `NHANH` | `B` | Thiếu thiết bị và driver |

Bốn gate không biết tên là `DRAResourcePoolStatus`, `DRANodeAllocatableResources`,
`DRAListTypeAttributes`, `DRAWorkloadResourceClaims`: binary của bạn ra đời trước các tính năng
đó. Con số của bạn là con số đo được; khác bảng trên thì ghi nguyên văn vào ADR.

Ý chính cho mục *Năng lực cluster*:

1. **Đang chạy đúng.** `No resources found` chỉ nghĩa là chưa có DeviceClass nào. Cluster chưa bật
   DRA in `error: the server doesn't have a resource type "deviceclasses"` — API server không phục vụ
   kiểu đó. Bài 271 phân biệt đúng hai trường hợp này.
2. Thiếu **thiết bị** và **driver** — hai điều kiện phần cứng. Nhóm API có, gate chính đã bật. Sổ nợ
   lab ghi phần bị chặn bởi kiến thức của giai đoạn sau; ở đây kiến thức đủ, chỉ thiếu phần cứng.
3. Ví dụ: `DRAExtendedResource`, `DRAPartitionableDevices`, `DRADeviceTaints`,
   `DRADeviceBindingConditions` — bài 149 ghi beta, có cái bật mặc định; trên cluster của bạn chúng
   còn alpha và tắt. Còn `DRAWorkloadResourceClaims` thì binary không biết tên. Viết manifest theo
   tài liệu mới nhất sẽ dùng trường mà API server của bạn gỡ âm thầm hoặc không hiểu — B4 chứng minh.

</details>

## B2. [BS-13-02] Điều tra: lớp thiết bị và claim của tính năng gợi ý

**Loại:** Điều tra.

**Bối cảnh:** nếu tính năng gợi ý được duyệt, đội vận hành tạo một lớp thiết bị cho GPU và dịch vụ
gợi ý xin GPU qua claim. Trước khi có phần cứng, đội cần biết luồng đó đi được tới đâu và dừng ở
đâu — vì `deploy.sh` sẽ báo thành công ngay cả khi tính năng không bao giờ chạy được.

**Yêu cầu:**

1. DeviceClass thử nghiệm `lab-13-gpu`, label `lab=lab-13`, chọn mọi thiết bị của driver giả định
   `gpu.bookstore.example.com`. Chứng minh: API server nhận nó, số DeviceClass tăng một, số
   ResourceSlice **không đổi**.
2. Pod `goi-y-gpu` trong `lab-13` (label `lab=lab-13`, `busybox`, ngủ dài) xin một thiết bị qua
   ResourceClaim tên `goi-y-gpu`. Tạo Pod **trước khi** claim tồn tại. Chứng minh Pod không được gán
   node; ghi lại condition và sự kiện của nó ở thời điểm này.
3. Tạo ResourceClaim `goi-y-gpu` xin một thiết bị từ lớp `lab-13-gpu`. Chứng minh: API server nhận
   claim; scheduler bắt đầu xét Pod và ghi sự kiện không lập lịch được; claim không có kết quả cấp
   phát, không ai giữ chỗ; Pod vẫn chưa có node.
4. Nhánh A: thêm một cặp claim và Pod `goi-y-gpu-that` trỏ tới một DeviceClass có thật. Chứng minh
   scheduler ghi kết quả cấp phát vào claim và ghi Pod vào danh sách giữ chỗ.
5. Viết `docs/dra/deviceclass-gpu.draft.yaml`: DeviceClass `bookstore-gpu` dự kiến cho tính năng,
   mang label dự án. Kiểm bằng server dry-run, **không apply**.
6. **Giữ Pod và claim `goi-y-gpu` lại** cho B4 và B7.

**Ràng buộc:** không tạo ResourceSlice. Không chạm namespace `bookstore` ngoài server dry-run.

**Đọc lại:**

- Bài [271](../271-set-up-dra-cluster-vi.md#create-deviceclasses), mục *Tạo DeviceClass* — selector
  CEL theo driver.
- Bài [270](../270-allocate-devices-dra-vi.md): [*Claim tài nguyên*](../270-allocate-devices-dra-vi.md#claim-resources)
  — đoạn về ResourceClaim chưa tồn tại;
  [*Yêu cầu thiết bị trong workload bằng DRA*](../270-allocate-devices-dra-vi.md#request-devices-workloads)
  — hai chỗ khai báo trong Pod spec.
- Bài [149](../149-dynamic-resource-allocation-vi.md):
  [*ResourceSlice*](../149-dynamic-resource-allocation-vi.md#resourceslice) — đoạn controller của
  driver ghi đè thay đổi thủ công;
  [*Luồng công việc của Kubernetes*](../149-dynamic-resource-allocation-vi.md#kubernetes-workflow) —
  năm bước; [*Hạn chế*](../149-dynamic-resource-allocation-vi.md#hạn-chế-limitations).

**Bàn giao:** `~/lab-work/13/b2-*.yaml`; `docs/dra/deviceclass-gpu.draft.yaml`; mục *Luồng claim*
trong ADR; `~/lab-evidence/13/b2-*`.

**Nghiệm thu:**

- `PASS:` DeviceClass tăng đúng một, ResourceSlice không đổi.
- `PASS:` chưa có claim — Pod `Pending`, chưa có node.
- `PASS:` có claim — API server nhận claim; có sự kiện `FailedScheduling` cho Pod.
- `PASS:` (nhánh B) claim không có `status.allocation`, không có `status.reservedFor`; Pod chưa có
  node.
- `PASS:` (nhánh A) claim `goi-y-gpu-that` có kết quả cấp phát và `reservedFor` ghi đúng Pod.
- `PASS:` bản nháp DeviceClass qua server dry-run, mang label dự án, chưa tồn tại trên cluster.

**Câu hỏi thiết kế** — trả lời trong mục *Luồng claim* của ADR:

1. Trong năm bước của *Luồng công việc của Kubernetes*, Pod `goi-y-gpu` dừng ở bước nào trước khi
   có claim, và ở bước nào sau khi có claim? Tín hiệu nào trên object phân biệt hai điểm dừng?
2. API server nhận DeviceClass và claim mà không hỏi có thiết bị khớp không. Hệ quả với quy trình
   phát hành của Bookstore là gì, và `tests/smoke.sh` phải kiểm thêm gì nếu tính năng ra mắt?
3. Nếu ai đó tự tạo một ResourceSlice để "có dữ liệu", chuyện gì xảy ra khi cluster có driver, và
   khi không có?
4. Sau này có GPU: một Pod gợi ý mang `bookstore-critical` có giành được GPU từ batch huấn luyện
   đang giữ nó không?

<details>
<summary>Gợi ý</summary>

1. DeviceClass chỉ cần một selector CEL trên `device.driver`. Bài 271 có mẫu.
2. Pod khai claim ở `spec.resourceClaims` (mỗi mục đúng một trong `resourceClaimName` hoặc
   `resourceClaimTemplateName`), container gọi tên claim ở `resources.claims`.
3. Ba file riêng — DeviceClass, Pod, claim — để apply theo đúng thứ tự yêu cầu.
4. Sự kiện của scheduler cho đúng Pod này, lọc theo UID để không lẫn với lần chạy trước:
   `kubectl -n lab-13 get events --field-selector involvedObject.uid=<uid>,reason=FailedScheduling`.
5. Server dry-run trả lại object như API server sẽ lưu: `kubectl create --dry-run=server -f <file>
   -o yaml`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script apply theo đúng thứ tự</summary>

```bash
NS=lab-13; EV=~/lab-evidence/13; W=~/lab-work/13
. "$EV/b0-anh-nen.env"; . "$EV/b1-nang-luc.env"
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
fs() { U="$(jp "pod/$1" '{.metadata.uid}')"
       kubectl -n "$NS" get events --no-headers \
         --field-selector "involvedObject.uid=$U,reason=FailedScheduling" 2>/dev/null | wc -l; }

# 1. Danh mục không sinh ra hàng
kubectl apply -f "$W/b2-deviceclass.yaml"
DC_NOW="$(kubectl get deviceclasses --no-headers 2>/dev/null | wc -l)"
SLICE_NOW="$(kubectl get resourceslices --no-headers 2>/dev/null | wc -l)"
echo "DeviceClass $DC_BEFORE -> $DC_NOW ; ResourceSlice $SLICE_BEFORE -> $SLICE_NOW" | tee "$EV/b2-dem.txt"
{ [ "$DC_NOW" -eq $((DC_BEFORE + 1)) ] && [ "$SLICE_NOW" -eq "$SLICE_BEFORE" ]; } \
  && echo 'PASS: them mot DeviceClass, khong them ResourceSlice nao'

# 2. Pod đến trước claim
kubectl -n "$NS" delete pod goi-y-gpu --ignore-not-found --wait=true
kubectl -n "$NS" delete resourceclaim goi-y-gpu --ignore-not-found --wait=true
kubectl apply -f "$W/b2-pod.yaml"
sleep 30
P1="$(jp pod/goi-y-gpu '{.status.phase}')"; N1="$(jp pod/goi-y-gpu '{.spec.nodeName}')"
C1="$(jp pod/goi-y-gpu '{.status.conditions[?(@.type=="PodScheduled")].status}')"; E1="$(fs goi-y-gpu)"
kubectl -n "$NS" describe pod goi-y-gpu > "$EV/b2-truoc-claim.txt"
echo "truoc claim: phase=$P1 node=${N1:-<chua gan>} PodScheduled=${C1:-<chua co>} FailedScheduling=$E1" \
  | tee -a "$EV/b2-truoc-claim.txt"
{ [ "$P1" = 'Pending' ] && [ -z "$N1" ]; } && echo 'PASS: chua co claim — Pod Pending, chua gan node'
echo "GHI NHAN: truoc khi co claim PodScheduled=${C1:-<chua co>}, su kien FailedScheduling=$E1"

# 3. Có claim: scheduler xét và từ chối; claim không được cấp phát
kubectl apply -f "$W/b2-claim.yaml"
for i in $(seq 1 24); do [ "$(fs goi-y-gpu)" -ge 1 ] && break; sleep 5; done
E2="$(fs goi-y-gpu)"; N2="$(jp pod/goi-y-gpu '{.spec.nodeName}')"
ALLOC="$(jp resourceclaim/goi-y-gpu '{.status.allocation}')"; RESV="$(jp resourceclaim/goi-y-gpu '{.status.reservedFor}')"
kubectl -n "$NS" get resourceclaim goi-y-gpu -o yaml > "$EV/b2-claim.yaml"
kubectl -n "$NS" describe pod goi-y-gpu > "$EV/b2-sau-claim.txt"
echo "sau claim: FailedScheduling=$E2 allocation=${ALLOC:-<rong>} reservedFor=${RESV:-<rong>} node=${N2:-<chua gan>}"
{ kubectl -n "$NS" get resourceclaim goi-y-gpu >/dev/null 2>&1 && [ "$E2" -ge 1 ]; } \
  && echo 'PASS: API server nhan claim; scheduler xet Pod va ghi FailedScheduling'
if [ "$NHANH" = B ]; then
  { [ -z "$ALLOC" ] && [ -z "$RESV" ] && [ -z "$N2" ]; } \
    && echo 'PASS: claim khong duoc cap phat, khong ai giu cho, Pod chua co node'
fi

# 4. Chỉ nhánh A: lớp thiết bị có thật
if [ "$NHANH" = A ]; then
  kubectl apply -f "$W/b2-that.yaml"
  for i in $(seq 1 60); do [ -n "$(jp resourceclaim/goi-y-gpu-that '{.status.allocation}')" ] && break; sleep 5; done
  kubectl -n "$NS" get resourceclaim goi-y-gpu-that -o yaml > "$EV/b2-claim-that.yaml"
  { [ -n "$(jp resourceclaim/goi-y-gpu-that '{.status.allocation.devices.results[0].device}')" ] && \
    [ "$(jp resourceclaim/goi-y-gpu-that '{.status.reservedFor[0].name}')" = 'goi-y-gpu-that' ]; } \
    && echo 'PASS: (nhanh A) scheduler ghi ket qua cap phat va reservedFor dung Pod'
fi

# 5. Bản nháp cho dự án
cd ~/bookstore
L="$(kubectl create --dry-run=server -f docs/dra/deviceclass-gpu.draft.yaml \
      -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}' 2>"$EV/b2-draft-loi.txt")"
D="$(kubectl create --dry-run=client -f docs/dra/deviceclass-gpu.draft.yaml -o jsonpath='{.metadata.name}')"
{ [ "$L" = 'bookstore' ] && ! kubectl get deviceclass "$D" >/dev/null 2>&1; } \
  && echo "PASS: ban nhap DeviceClass $D qua server dry-run, mang label du an, chua apply"
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/13/b2-deviceclass.yaml
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: lab-13-gpu
  labels:
    lab: lab-13
spec:
  selectors:
  - cel:
      expression: device.driver == "gpu.bookstore.example.com"
```

```yaml
# ~/lab-work/13/b2-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: goi-y-gpu
  namespace: lab-13
  labels:
    lab: lab-13
spec:
  restartPolicy: Never
  terminationGracePeriodSeconds: 1
  containers:
  - name: goi-y
    image: busybox:1.37
    command: ["sleep", "3600"]
    resources:
      claims:
      - name: gpu
  resourceClaims:
  - name: gpu
    resourceClaimName: goi-y-gpu
```

```yaml
# ~/lab-work/13/b2-claim.yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: goi-y-gpu
  namespace: lab-13
  labels:
    lab: lab-13
spec:
  devices:
    requests:
    - name: gpu
      exactly:
        deviceClassName: lab-13-gpu
```

Nhánh A: `~/lab-work/13/b2-that.yaml` là bản sao của hai file trên, đổi tên thành `goi-y-gpu-that`
và `deviceClassName` thành một DeviceClass có sẵn của driver thật (`kubectl get deviceclasses`).

```yaml
# docs/dra/deviceclass-gpu.draft.yaml — BẢN NHÁP, không apply. Chỉ có hiệu lực khi có driver thật.
# Thuộc tính cho selector phải đọc từ ResourceSlice thật (bài 271, mục Tạo DeviceClass);
# hiện chỉ chọn theo tên driver giả định.
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: bookstore-gpu
  labels:
    app.kubernetes.io/part-of: bookstore
spec:
  selectors:
  - cel:
      expression: device.driver == "gpu.bookstore.example.com"
```

Ý chính cho mục *Luồng claim*:

1. Trước khi có claim, Pod dừng ở **bước 2** (*Tạo workload*): Kubernetes kiểm claim được tham
   chiếu có tồn tại không, và không tìm thấy. Pod chưa vào được lượt xét của scheduler — trên
   cluster của bạn có thể không có condition `PodScheduled` lẫn sự kiện nào, `describe` trống trơn.
   Sau khi có claim, Pod tiến tới **bước 3** (*Lọc ResourceSlice*): scheduler xét và không tìm được
   thiết bị khớp, nên ghi `FailedScheduling`, còn `status.allocation` của claim rỗng — bước 4 chưa
   xảy ra. Tín hiệu phân biệt: có hay không sự kiện của scheduler, và claim có tồn tại hay không.
2. Mọi manifest đều được nhận, không lỗi, không cảnh báo; tính năng chỉ không bao giờ chạy. Một
   `deploy.sh` chờ rollout sẽ treo tới timeout, còn một `deploy.sh` không chờ sẽ báo thành công
   giả. Smoke phải kiểm Pod gợi ý Ready **và** claim có `status.allocation`. Đó cũng là lý do năng
   lực phải đo trước (B1), không phải sau.
3. Có driver: controller của driver đối chiếu ResourceSlice và **ghi đè** mọi thay đổi thủ công —
   slice tự tạo bị xóa hoặc sửa ở lần đối chiếu kế tiếp. Không driver: slice là dữ liệu chết, có
   thể khiến scheduler "cấp phát" một thiết bị không tồn tại; Pod lên node rồi kẹt vì không ai
   chuẩn bị thiết bị. Cả hai trường hợp đều là dựng dữ liệu giả.
4. **Không.** Mục *Hạn chế* của bài 149: scheduler không preempt cho tài nguyên DRA. Pod gợi ý
   nằm Pending tới khi batch kết thúc hoặc bị xóa tay. ADR phải ghi điều này vào cách lên lịch batch.

</details>

## B3. [BS-13-03] Điều tra: mỗi bản sao một GPU — ResourceClaimTemplate

**Loại:** Điều tra.

**Bối cảnh:** dịch vụ gợi ý sẽ chạy nhiều bản sao, mỗi bản sao cần một GPU **riêng** nhưng cấu hình
giống nhau. Claim tạo tay ở B2 chỉ có một tên, nên mọi Pod tham chiếu nó sẽ **chia sẻ** cùng một
nhóm thiết bị. Đội cần biết cách xin thiết bị theo từng Pod và vòng đời của claim khi Pod mất.

**Yêu cầu:**

1. ResourceClaimTemplate `goi-y-gpu-mau` (một thiết bị từ lớp `lab-13-gpu`) và Pod `goi-y-gpu-mau`
   dùng nó, cả hai trong `lab-13` với label `lab=lab-13`. Chứng minh: một claim riêng được sinh cho
   Pod; tên claim tra được từ status của Pod; claim thuộc sở hữu của Pod; claim chưa được cấp phát.
2. Chứng minh thành phần sinh claim là một controller của kube-controller-manager và đang được bật —
   đọc cấu hình, chỉ đọc.
3. Xóa Pod `goi-y-gpu-mau`. Chứng minh claim sinh ra biến mất theo, trong khi claim tạo tay
   `goi-y-gpu` của B2 vẫn còn.
4. ⭐ Một Pod `khong-thiet-bi` không yêu cầu thiết bị: chứng minh trong container không có thư mục
   metadata thiết bị theo quy ước của bài 269. Xóa Pod khi xong.
5. Viết `docs/dra/claim-goi-y.draft.yaml`: ResourceClaimTemplate dự kiến cho dịch vụ gợi ý trong
   namespace `bookstore`, xin một thiết bị lớp `bookstore-gpu`, mang label dự án. Kiểm bằng server
   dry-run, không apply.

**Ràng buộc:** không tham chiếu trực tiếp claim sinh tự động. Không xóa finalizer của claim nào.

**Đọc lại:**

- Bài [149](../149-dynamic-resource-allocation-vi.md):
  [*Trường hợp sử dụng ResourceClaim và ResourceClaimTemplate*](../149-dynamic-resource-allocation-vi.md#when-to-use-rc-rct),
  [*Thuật ngữ DRA*](../149-dynamic-resource-allocation-vi.md#terminology) — đoạn ResourceClaimTemplate,
  [*Luồng công việc của Kubernetes*](../149-dynamic-resource-allocation-vi.md#kubernetes-workflow) —
  bước 2, `resourceclaim-controller`.
- Bài [270](../270-allocate-devices-dra-vi.md#claim-resources) — tiêu chí chọn giữa hai cách, và lời
  khuyên không tham chiếu claim sinh tự động.
- Bài [172](../172-cluster-admin-dra-vi.md), mục *Giám sát và tinh chỉnh các thành phần cho mức tải
  cao hơn* — controller ResourceClaim nội bộ do kube-controller-manager điều phối.
- Bài [269](../269-access-dra-device-metadata-vi.md#access-metadata-resourceclaim) — đường dẫn
  `/var/run/kubernetes.io/dra-device-attributes`, và mục *Trước khi bạn bắt đầu*.

**Bàn giao:** `~/lab-work/13/b3-*.yaml`; `docs/dra/claim-goi-y.draft.yaml`; mục *Claim theo từng
Pod* trong ADR; `~/lab-evidence/13/b3-*`.

**Nghiệm thu:**

- `PASS:` status của Pod trỏ tới một claim mà `ownerReferences` là Pod `goi-y-gpu-mau`.
- `PASS:` (nhánh B) claim sinh ra chưa có `status.allocation`.
- `PASS:` kube-controller-manager chạy bộ controller mặc định, không loại controller ResourceClaim.
- `PASS:` xóa Pod thì claim sinh ra bị xóa; claim `goi-y-gpu` vẫn còn.
- `PASS:` (⭐) không có thư mục metadata trong container không yêu cầu thiết bị.
- `PASS:` bản nháp ResourceClaimTemplate qua server dry-run trong `bookstore`.

**Câu hỏi thiết kế** — trả lời trong mục *Claim theo từng Pod* của ADR:

1. Dịch vụ gợi ý chạy hai bản sao, mỗi bản sao một GPU: claim tạo tay hay template? Còn sidecar
   và container chính trong **cùng một** Pod cần dùng chung GPU của Pod đó thì khai thế nào?
2. Claim đã sinh ra nhưng thiết bị chưa được cấp phát. Hai việc đó do hai thành phần nào làm, và vì
   sao trên cluster này việc thứ nhất chạy được còn việc thứ hai thì không?
3. Vì sao bài 270 khuyên không tham chiếu trực tiếp một claim sinh tự động? Áp vào một Deployment
   cuộn bản mới (Lab 4a).

<details>
<summary>Gợi ý</summary>

1. Template có `spec.spec` — tầng `spec` thứ hai là spec của claim sẽ được sinh ra. Pod tham chiếu
   bằng `resourceClaimTemplateName`.
2. Tên claim sinh ra không đoán trước được; nó nằm ở `status.resourceClaimStatuses` của Pod.
3. Danh sách controller của kube-controller-manager nằm ở cờ `--controllers` trong manifest static
   Pod. `*` là mọi controller bật mặc định; tiền tố `-` là loại trừ.
4. `kubectl wait --for=delete resourceclaim/<tên>` chờ object biến mất.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
NS=lab-13; EV=~/lab-evidence/13; W=~/lab-work/13
. "$EV/b1-nang-luc.env"
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }

# 1. Template sinh claim riêng cho Pod
kubectl apply -f "$W/b3-template-pod.yaml"
GEN=''
for i in $(seq 1 24); do
  GEN="$(jp pod/goi-y-gpu-mau '{.status.resourceClaimStatuses[0].resourceClaimName}')"
  [ -n "$GEN" ] && break; sleep 5
done
echo "claim sinh ra: ${GEN:-<khong co>}" | tee "$EV/b3-claim-sinh-ra.txt"
kubectl -n "$NS" get resourceclaim "$GEN" -o yaml >> "$EV/b3-claim-sinh-ra.txt" 2>&1
OWNER="$(jp "resourceclaim/$GEN" '{.metadata.ownerReferences[0].kind} {.metadata.ownerReferences[0].name}')"
{ [ -n "$GEN" ] && [ "$OWNER" = 'Pod goi-y-gpu-mau' ]; } \
  && echo 'PASS: claim rieng duoc sinh cho Pod va thuoc so huu cua Pod'
if [ "$NHANH" = B ]; then
  [ -z "$(jp "resourceclaim/$GEN" '{.status.allocation}')" ] \
    && echo 'PASS: claim da sinh nhung chua cap phat — sinh claim khac cap phat thiet bi'
fi

# 2. Ai sinh claim
CTRL="$(sudo grep -o -- '--controllers=[^ "]*' /etc/kubernetes/manifests/kube-controller-manager.yaml)"
echo "${CTRL:-<khong khai --controllers>}" | tee "$EV/b3-kcm-controllers.txt"
{ { [ -z "$CTRL" ] || echo "$CTRL" | grep -q '[=,]\*'; } && ! echo "$CTRL" | grep -q -- '-resourceclaim'; } \
  && echo 'PASS: kube-controller-manager chay bo controller mac dinh, co controller ResourceClaim'

# 3. Vòng đời
kubectl -n "$NS" delete pod goi-y-gpu-mau --wait=true
kubectl -n "$NS" wait --for=delete "resourceclaim/$GEN" --timeout=120s \
  && echo 'PASS: Pod mat thi claim sinh ra cung mat'
kubectl -n "$NS" get resourceclaim goi-y-gpu >/dev/null 2>&1 \
  && echo 'PASS: claim tao tay goi-y-gpu van con — vong doi do nguoi tao quan'

# 4. ⭐ Vế phủ định của metadata thiết bị
kubectl apply -f "$W/b3-khong-thiet-bi.yaml"
kubectl -n "$NS" wait --for=condition=Ready pod/khong-thiet-bi --timeout=120s
kubectl -n "$NS" exec khong-thiet-bi -- test -d /var/run/kubernetes.io/dra-device-attributes \
  && echo 'GHI NHAN: co thu muc metadata du container khong yeu cau thiet bi' \
  || echo 'PASS: khong yeu cau thiet bi thi khong co thu muc metadata'
kubectl -n "$NS" delete pod khong-thiet-bi --wait=true

# 5. Bản nháp cho dự án
cd ~/bookstore
kubectl create --dry-run=server -f docs/dra/claim-goi-y.draft.yaml > "$EV/b3-draft.txt" 2>&1 \
  && echo 'PASS: ban nhap ResourceClaimTemplate qua server dry-run trong bookstore'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/13/b3-template-pod.yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: goi-y-gpu-mau
  namespace: lab-13
  labels:
    lab: lab-13
spec:
  spec:
    devices:
      requests:
      - name: gpu
        exactly:
          deviceClassName: lab-13-gpu
---
apiVersion: v1
kind: Pod
metadata:
  name: goi-y-gpu-mau
  namespace: lab-13
  labels:
    lab: lab-13
spec:
  restartPolicy: Never
  terminationGracePeriodSeconds: 1
  containers:
  - name: goi-y
    image: busybox:1.37
    command: ["sleep", "3600"]
    resources:
      claims:
      - name: gpu
  resourceClaims:
  - name: gpu
    resourceClaimTemplateName: goi-y-gpu-mau
```

```yaml
# ~/lab-work/13/b3-khong-thiet-bi.yaml
apiVersion: v1
kind: Pod
metadata:
  name: khong-thiet-bi
  namespace: lab-13
  labels:
    lab: lab-13
spec:
  terminationGracePeriodSeconds: 1
  containers:
  - name: shell
    image: busybox:1.37
    command: ["sleep", "3600"]
```

```yaml
# docs/dra/claim-goi-y.draft.yaml — BẢN NHÁP, không apply.
# Component recommender chưa có trong hợp đồng (DU-AN-BOOKSTORE.md mục 4.1) — đề xuất ở ADR.
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: goi-y-gpu
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: recommender
spec:
  spec:
    devices:
      requests:
      - name: gpu
        exactly:
          deviceClassName: bookstore-gpu
```

Trong Deployment gợi ý tương lai, Pod template khai `resourceClaims: [{name: gpu,
resourceClaimTemplateName: goi-y-gpu}]` và container gọi `resources.claims: [{name: gpu}]`.

Ý chính cho mục *Claim theo từng Pod*:

1. Hai bản sao, mỗi bản một GPU → **template**: Kubernetes sinh một claim cho mỗi Pod. Hai
   container trong cùng Pod muốn dùng chung GPU của Pod đó thì **cùng gọi một tên** trong
   `resources.claims` — đúng cách `container1` và `container2` chia sẻ `shared-gpu-claim` ở bài 270.
2. **Sinh claim** là việc của `resourceclaim-controller` trong kube-controller-manager, xảy ra ngay
   khi Pod ra đời và không cần thiết bị. **Cấp phát** là việc của scheduler ở bước 3–4 và cần
   ResourceSlice. Cluster có controller nhưng không có ResourceSlice, nên `status.allocation` rỗng.
3. Claim sinh tự động gắn vào vòng đời của Pod đã sinh ra nó. Deployment cuộn bản mới thì Pod cũ
   bị xóa, claim của nó bị xóa theo; Pod khác tham chiếu tên claim đó sẽ nằm Pending vì claim không
   còn — đúng tình huống của B2 trước khi có claim.

</details>

## B4. [BS-13-04] Điều tra: ba con đường cấp phát thiết bị

**Loại:** Điều tra.

**Bối cảnh:** đội phát triển hỏi: "Sao không viết `example.com/gpu: 1` vào `limits` như ở Lab 3c cho
nhanh?" ADR phải trả lời câu đó bằng số đo trên cluster, không bằng khẩu hiệu.

**Yêu cầu:**

1. Pod `goi-y-er` trong `lab-13` xin `example.com/gpu: 1` theo kiểu extended resource. Chứng minh
   Pod Pending và scheduler nói rõ node thiếu **con số** đó. Đặt thông điệp này cạnh thông điệp
   của Pod `goi-y-gpu` (B2) — hai cách hạch toán khác nhau.
2. Chứng minh `status.allocatable` của `lab-k8s-worker2` chỉ có tài nguyên chuẩn: không extended
   resource nào, không dấu vết nào của DRA.
3. Chứng minh trường của tính năng *Cấp phát extended resource bằng DRA* có trong schema của
   DeviceClass nhưng API server **gỡ nó** khi lưu object, vì gate tương ứng đang tắt (đối chiếu B1).
   Dùng server dry-run, không tạo object thật.
4. ⭐ Đọc `/metrics` của kubelet trên `lab-k8s-worker2`: số dòng metric đăng ký device plugin và số
   dòng metric thao tác DRA của kubelet.
5. Mục *Ba con đường* trong ADR: bảng so sánh cho tính năng gợi ý — nơi khai báo trong Pod spec,
   đơn vị, lọc theo thuộc tính, chia sẻ, nơi hạch toán, đã kiểm ở đâu.

**Ràng buộc:** không vá `status.capacity` của node — Lab 3c đã làm, lần này chỉ đọc. Không đọc bài
184 ở giai đoạn này; so sánh device plugin bằng bài 149 và bài 110.

**Đọc lại:**

- Bài [149](../149-dynamic-resource-allocation-vi.md):
  [*Lợi ích của DRA*](../149-dynamic-resource-allocation-vi.md#dra-benefits) — đoạn cuối về device
  plugin; [*Cấp phát extended resource bằng DRA*](../149-dynamic-resource-allocation-vi.md#extended-resource).
- Bài [110](../110-manage-resources-containers-vi.md):
  [*Tài nguyên mở rộng*](../110-manage-resources-containers-vi.md#extended-resources),
  [*Tài nguyên do device plugin quản lý*](../110-manage-resources-containers-vi.md#device-plugin-managed-resources),
  [*Tiêu thụ tài nguyên mở rộng*](../110-manage-resources-containers-vi.md#consuming-extended-resources).
- Bài [284](../284-extended-resource-vi.md), mục
  [*Thử tạo một Pod thứ hai*](../284-extended-resource-vi.md#thử-tạo-một-pod-thứ-hai-attempt-to-create-a-second-pod).
- Bài [172](../172-cluster-admin-dra-vi.md), mục
  [*Các metric của `kubelet`*](../172-cluster-admin-dra-vi.md#các-metric-của-kubelet-kubelet-metrics) và
  [*Các thao tác DRA kubeletplugin*](../172-cluster-admin-dra-vi.md#các-thao-tác-dra-kubeletplugin-dra-kubeletplugin-operations).

**Bàn giao:** `~/lab-work/13/b4-*.yaml`; mục *Ba con đường* trong ADR; `~/lab-evidence/13/b4-*`.

**Nghiệm thu:**

- `PASS:` `goi-y-er` Pending, sự kiện scheduler có `Insufficient example.com/gpu`.
- `PASS:` sự kiện của `goi-y-gpu` không nói `Insufficient` — DRA không hạch toán bằng con số
  trên node.
- `PASS:` `allocatable` của `lab-k8s-worker2` chỉ có `cpu`, `memory`, `pods`, `ephemeral-storage`,
  `hugepages-*`.
- `PASS:` schema DeviceClass có `extendedResourceName`, nhưng object qua server dry-run không còn
  trường đó.
- `PASS:` (⭐) đã chốt số liệu metric kubelet vào evidence.

**Câu hỏi thiết kế** — trả lời trong mục *Ba con đường* của ADR:

1. Vì sao viết `example.com/gpu: 1` vào `limits` không phải cách làm việc của DRA, dù bạn đã làm
   đúng động tác đó ở Lab 3c? Áp ba thiếu sót mà bài 149 gán cho device plugin vào tính năng gợi ý:
   chọn GPU theo bộ nhớ, dùng chung GPU giữa hai container, khai ở đâu.
2. Hai thông điệp Pending nói lên hai cách hạch toán nào? Pod dùng DRA "ăn" gì trong `allocatable`
   của node?
3. "API có trường" khác "tính năng đang bật" ở đâu? Hậu quả nếu đội chép một DeviceClass có
   `extendedResourceName` từ tài liệu mới nhất vào repo?

<details>
<summary>Gợi ý</summary>

1. Extended resource không overcommit được: khai `limits` là đủ, `requests` mặc định bằng `limits`,
   giá trị phải là số nguyên.
2. In `allocatable` của node: `kubectl get node lab-k8s-worker2 -o jsonpath='{.status.allocatable}'`.
3. `kubectl explain deviceclass.spec.extendedResourceName` đọc schema. Server dry-run trả về object
   như API server sẽ lưu: so trường bạn gửi với trường nhận lại.
4. Metric kubelet đọc qua node proxy: `kubectl get --raw /api/v1/nodes/lab-k8s-worker2/proxy/metrics`.
   Metric `dra_grpc_operations_duration_seconds` do thư viện `kubeletplugin` **trong driver** phát ra,
   không nằm ở kubelet.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
NS=lab-13; EV=~/lab-evidence/13; W=~/lab-work/13; W2=lab-k8s-worker2
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
msg() { U="$(jp "pod/$1" '{.metadata.uid}')"
        kubectl -n "$NS" get events --field-selector "involvedObject.uid=$U,reason=FailedScheduling" \
          -o jsonpath='{.items[*].message}' 2>/dev/null; }

# 1. Hai thông điệp Pending
kubectl apply -f "$W/b4-goi-y-er.yaml"
for i in $(seq 1 24); do [ -n "$(msg goi-y-er)" ] && break; sleep 5; done
M_ER="$(msg goi-y-er)"; M_DRA="$(msg goi-y-gpu)"
{ echo "extended resource: $M_ER"; echo "DRA              : $M_DRA"; } | tee "$EV/b4-hai-thong-diep.txt"
{ [ "$(jp pod/goi-y-er '{.status.phase}')" = 'Pending' ] && echo "$M_ER" | grep -q 'Insufficient example.com/gpu'; } \
  && echo 'PASS: extended resource — Pending vi node khong co du con so'
{ [ -n "$M_DRA" ] && ! echo "$M_DRA" | grep -q 'Insufficient'; } \
  && echo 'PASS: DRA — Pending vi ly do khac, khong phai thieu con so tren node'

# 2. allocatable của worker2
kubectl get node "$W2" -o jsonpath='{.status.allocatable}' | tr ',' '\n' | tr -d '{}"' | sed '/^$/d' \
  > "$EV/b4-allocatable.txt"
cat "$EV/b4-allocatable.txt"
[ "$(grep -cvE '^(cpu|memory|pods|ephemeral-storage|hugepages-[^:]+):' "$EV/b4-allocatable.txt")" -eq 0 ] \
  && echo 'PASS: allocatable cua worker2 chi co tai nguyen chuan'

# 3. Trường có trong schema nhưng bị gỡ khi lưu
kubectl explain deviceclass.spec.extendedResourceName > "$EV/b4-extended-resource-name.txt" 2>&1 \
  && SCHEMA=co || SCHEMA=khong
if KEPT="$(kubectl create --dry-run=server -f "$W/b4-deviceclass-er.yaml" \
             -o jsonpath='{.spec.extendedResourceName}' 2>>"$EV/b4-extended-resource-name.txt")"; then
  DRY=ok; else DRY=loi; fi
echo "schema=$SCHEMA dry-run=$DRY gia-tri-sau-khi-luu='${KEPT}'" | tee -a "$EV/b4-extended-resource-name.txt"
if [ "$SCHEMA" = co ] && [ "$DRY" = ok ] && [ -z "$KEPT" ]; then
  echo 'PASS: schema co truong nhung API server go no khi luu — gate dang tat'
else
  echo 'GHI NHAN: khac ky vong — doi chieu dong DRAExtendedResource trong b1-feature-gates.txt'
fi

# 4. ⭐ Metric kubelet
kubectl get --raw "/api/v1/nodes/$W2/proxy/metrics" > "$EV/b4-kubelet-metrics.txt"
DP="$(grep -c '^kubelet_device_plugin_registration_total' "$EV/b4-kubelet-metrics.txt")"
OPS="$(grep -c '^dra_operations_duration_seconds' "$EV/b4-kubelet-metrics.txt")"
echo "device plugin dang ky: $DP dong ; DRA PrepareResources/UnprepareResources: $OPS dong" \
  | tee "$EV/b4-metric-kubelet.txt"
test -s "$EV/b4-kubelet-metrics.txt" && echo 'PASS: da chot so lieu metric kubelet'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/13/b4-goi-y-er.yaml
apiVersion: v1
kind: Pod
metadata:
  name: goi-y-er
  namespace: lab-13
  labels:
    lab: lab-13
spec:
  restartPolicy: Never
  terminationGracePeriodSeconds: 1
  containers:
  - name: goi-y
    image: busybox:1.37
    command: ["sleep", "3600"]
    resources:
      limits:
        example.com/gpu: 1
```

```yaml
# ~/lab-work/13/b4-deviceclass-er.yaml — chỉ dùng với --dry-run=server
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: lab-13-gpu-er
  labels:
    lab: lab-13
spec:
  selectors:
  - cel:
      expression: device.driver == "gpu.bookstore.example.com"
  extendedResourceName: example.com/gpu
```

Bảng cho mục *Ba con đường*:

| | Extended resource | Device plugin | DRA |
| --- | --- | --- | --- |
| Ai công bố | quản trị viên vá `status.capacity` của Node | plugin trên node báo số lượng cho kubelet | driver tạo ResourceSlice |
| Đơn vị | một số nguyên dưới một tên tự đặt | một số nguyên dưới tên do plugin đặt | danh sách thiết bị kèm thuộc tính, dung lượng |
| Pod xin ở đâu | `resources.limits` của từng container | `resources.limits` của từng container | `spec.resourceClaims` của Pod, container gọi tên ở `resources.claims` |
| Lọc theo thuộc tính | không | không | CEL trong DeviceClass và claim |
| Chia sẻ | không | không | có, qua cùng một claim |
| Hạch toán | `allocatable` của Node | `allocatable` của Node | ResourceSlice (cung), ResourceClaim (cầu) |
| Kiểm ở đâu | Lab 3c, B4 bước 1 | bài 110, bài 149; bài 184 ở giai đoạn 14 | B2, B3 |

Kỳ vọng ở bước 4 (⭐): cả hai con số bằng 0 trên cluster này — chưa plugin nào đăng ký, kubelet
chưa từng gọi xuống driver DRA nào.

Ý chính:

1. Bài 149 nêu ba thiếu sót của device plugin: **khai báo theo từng container**, **không chia sẻ**,
   **không lọc theo biểu thức**. Áp vào tính năng gợi ý: không chọn được "GPU có ít nhất 16 GiB bộ
   nhớ", sidecar và container chính không dùng chung được một GPU, mỗi container tự khai con số của
   mình. Một con số trong `limits` chính là mô hình cũ mà DRA thay thế.
2. `Insufficient example.com/gpu` là phép trừ trên `allocatable` của node. Thông điệp của Pod DRA
   nói về việc không cấp phát được claim — DRA hạch toán bằng object riêng (ResourceSlice và
   ResourceClaim), và Pod dùng DRA không "ăn" gì trong `allocatable` của node.
3. Schema là tập trường mà binary **biết**; gate quyết định trường đó có được **lưu** không. Chép
   DeviceClass có `extendedResourceName` vào repo thì `kubectl apply` thành công, trường biến mất
   không báo lỗi, và mọi Pod xin `example.com/gpu` vẫn Pending. Đó là lý do mỗi bản nháp trong
   `docs/dra/` phải qua server dry-run và đọc lại object trả về.

</details>

## B5. [BS-13-05] Điều tra: batch huấn luyện mô hình gợi ý chạy theo nhóm

**Loại:** Điều tra.

**Bối cảnh:** batch đêm có ba worker trao đổi dữ liệu với nhau sau mỗi bước, nên phải cùng chạy.
Nếu cluster chỉ đủ chỗ cho hai, hai worker đó lên, giữ CPU, và chờ worker thứ ba mãi. Đội cần biết
Kubernetes có cơ chế gì cho việc này, cluster có chưa, và nếu chưa thì thiết kế batch thế nào để
khi cluster có, chỉ cần bật lên.

**Yêu cầu:**

1. Đo bề mặt API lập lịch theo nhóm: nhóm `scheduling.k8s.io` phục vụ những gì; có kind PodGroup
   hay Workload không; schema Pod có trường liên kết Pod với nhóm như bài 59 mô tả không; API server
   phản ứng thế nào với bản nháp PodGroup viết theo tài liệu.
2. Lập bảng bốn gate `GenericWorkload`, `GangScheduling`, `WorkloadAwarePreemption`,
   `WorkloadWithJob`: biết tên hay không, bật hay tắt.
3. Đo ba điều kiện nền: có PriorityClass nào là `globalDefault` không; node mang nhãn topology nào;
   kube-scheduler có nạp file cấu hình không.
4. Thí nghiệm trên `lab-k8s-worker2`: Job Indexed `huan-luyen-thu` trong `lab-13`, ba worker chạy
   cùng lúc, ghim `lab-k8s-worker2`, mỗi worker xin một lượng CPU sao cho **chỉ hai** worker vừa.
   Chứng minh hai worker chạy và giữ CPU còn worker thứ ba Pending vì thiếu CPU. Xóa Job khi xong.
5. Hai bản nháp:
   - `docs/dra/huan-luyen-job.draft.yaml` — Job Indexed cho batch thật trong `bookstore`, ba worker
     chạy cùng lúc, PriorityClass `bookstore-batch`, đạt Pod Security `restricted`. Server dry-run
     phải nhận và không cảnh báo PodSecurity.
   - `docs/dra/huan-luyen-podgroup.draft.yaml` — Workload và PodGroup `gang` với `minCount: 3` viết
     đúng theo tài liệu. Server dry-run phải **bị từ chối**; ghi lý do.
6. ADR `docs/adr/NNNN-batch-huan-luyen-theo-nhom.md`.

**Ràng buộc:** không bật gate hay nhóm API; không gán nhãn zone cho node. Job thí nghiệm không đặt
`priorityClassName` — độ ưu tiên 0, không chiếm chỗ Pod nào. Có `globalDefault` thì **STOP**, không
chạy thí nghiệm.

**Đọc lại:**

- Bài [150](../150-gang-scheduling-vi.md#cách-hoạt-động-how-it-works) — ba bước, hai điều kiện rời
  `PreEnqueue`.
- Bài [79](../79-workload-policies-vi.md):
  [*Chính sách basic*](../79-workload-policies-vi.md#chính-sách-basic-basic-policy),
  [*Chính sách gang*](../79-workload-policies-vi.md#chính-sách-gang-gang-policy).
- Bài [77](../77-workload-api-vi.md):
  [*Workload là gì?*](../77-workload-api-vi.md#workload-là-gì-what-is-a-workload),
  [*Gang scheduling với Job*](../77-workload-api-vi.md#gang-scheduling-với-job-gang-scheduling-with-jobs).
- Bài [75](../75-podgroup-api-vi.md):
  [*PodGroup là gì?*](../75-podgroup-api-vi.md#podgroup-là-gì-what-is-a-podgroup),
  [*Cách các thành phần khớp với nhau*](../75-podgroup-api-vi.md#cách-các-thành-phần-khớp-với-nhau-how-it-fits-together).
- Bài [59](../59-scheduling-group-vi.md#chỉ-định-nhóm-lập-lịch-specifying-a-scheduling-group).
- Bài [76](../76-podgroup-lifecycle-vi.md): [*Thứ tự tạo*](../76-podgroup-lifecycle-vi.md#thứ-tự-tạo-creation-ordering),
  [*Giới hạn*](../76-podgroup-lifecycle-vi.md#limitations).
- Bài [78](../78-workload-disruption-priority-vi.md#độ-ưu-tiên-của-pod-group-pod-group-priority) và
  [152](../152-workload-aware-preemption-vi.md#cách-hoạt-động-how-it-works) — khối *Ghi chú* của
  cả hai bài.
- Bài [80](../80-workload-topology-scheduling-vi.md#ràng-buộc-topology-topology-constraint).
- Bài [151](../151-podgroup-scheduling-vi.md):
  [*Chu trình lập lịch PodGroup*](../151-podgroup-scheduling-vi.md#podgroup-scheduling-cycle),
  [*Hạn chế*](../151-podgroup-scheduling-vi.md#hạn-chế-limitations).
- Bài [153](../153-topology-aware-scheduling-vi.md#scheduling-framework-cấu-hình-các-plugin-tas-scheduling-framework-tas-plugins-configuration).
- [Lab 4b](LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) — Indexed Job, kỹ thuật render `.tpl` bằng `sed`;
  [Lab 9b](LAB-9B-POD-SECURITY-VA-HARDENING.md) — các trường mà `restricted` đòi.

**Bàn giao:** `~/lab-work/13/b5-*`; hai bản nháp `docs/dra/huan-luyen-*.draft.yaml`;
`docs/adr/NNNN-batch-huan-luyen-theo-nhom.md`; `~/lab-evidence/13/b5-*`.

**Nghiệm thu:**

- `PASS:` `scheduling.k8s.io` có PriorityClass, không có PodGroup hay Workload.
- `PASS:` schema Pod của cluster không có `schedulingGroup`.
- `PASS:` API server không biết kind trong bản nháp PodGroup.
- `PASS:` đủ bốn dòng gate.
- `PASS:` không có `globalDefault`; không node nào mang nhãn zone hay region; kube-scheduler không nạp
  file cấu hình (hoặc `GHI NHAN:` kèm file đó).
- `PASS:` thí nghiệm — đúng hai worker Running, một worker Pending với `Insufficient cpu`; xóa Job thì
  không còn Pod nào.
- `PASS:` bản nháp Job qua server dry-run trong `bookstore`, không cảnh báo PodSecurity.
- `PASS:` có ADR batch.

**Câu hỏi thiết kế** — trả lời trong ADR batch:

1. Nếu ba worker thuộc một PodGroup `gang` với `minCount: 3`, thí nghiệm ở bước 4 diễn ra thế nào?
   Hai điều kiện nào phải đúng để Pod rời `PreEnqueue`, và hai worker đã có chỗ thì sao?
2. Để dùng PodGroup, cluster thiếu chính xác những gì? API server biết tên bao nhiêu trong bốn gate,
   và "không biết tên" khác "đang tắt" ở điểm nào?
3. Gắn `bookstore-batch` ở cấp PodGroup có làm batch được xếp trước trong hàng đợi không? Một Pod
   `api` mang `bookstore-critical` cần chỗ và chiếm chỗ một worker của nhóm `disruptionMode:
   PodGroup` — scheduler có tôn trọng chế độ đó không?
4. Ràng buộc topology trên ba VM này có ý nghĩa không? Vì sao không gán nhãn zone cho node để "có
   nhiều miền"?
5. Thuật toán lập lịch PodGroup chỉ được kỳ vọng tìm ra placement với nhóm đồng nhất. Batch của bạn
   có đồng nhất không? Thêm một Pod "điều phối" có yêu cầu khác thì sao?

<details>
<summary>Gợi ý</summary>

1. `kubectl api-versions` và `kubectl api-resources --api-group=scheduling.k8s.io -o name`.
   `kubectl explain pod.spec.<trường>` thất bại nghĩa là schema không có trường đó.
2. Bốn gate đọc từ `b0-apiserver-metrics.txt` giống B1.
3. CPU còn trống của worker2 = `allocatable.cpu` trừ tổng `requests` mà `kubectl describe node`
   in ở phần `Allocated resources`. Mỗi worker xin khoảng 45% phần còn trống thì hai worker vừa, ba
   worker không vừa. Viết manifest dạng `.tpl` với chỗ giữ `__CPU__`, render bằng `sed`.
4. Pod của Job mang label `batch.kubernetes.io/job-name=<tên Job>`.
5. Job Indexed: `completionMode: Indexed`, `completions` bằng `parallelism`.
6. `restricted` đòi `runAsNonRoot`, `allowPrivilegeEscalation: false`, bỏ mọi capability,
   `seccompProfile` `RuntimeDefault`. Cảnh báo của Pod Security hiện ở output của server dry-run.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; bước 4 là thí nghiệm, chạy liền mạch</summary>

```bash
NS=lab-13; EV=~/lab-evidence/13; W=~/lab-work/13; W2=lab-k8s-worker2

# 1. Bề mặt API lập lịch theo nhóm
kubectl api-versions | grep '^scheduling\.k8s\.io/' | tee "$EV/b5-scheduling-versions.txt"
kubectl api-resources --api-group=scheduling.k8s.io -o name | tee "$EV/b5-scheduling-resources.txt"
{ grep -qx 'priorityclasses.scheduling.k8s.io' "$EV/b5-scheduling-resources.txt" && \
  ! grep -qE '^(podgroups|workloads)\.' "$EV/b5-scheduling-resources.txt"; } \
  && echo 'PASS: scheduling.k8s.io chi co PriorityClass — khong co PodGroup, Workload'
kubectl explain pod.spec.schedulingGroup > "$EV/b5-explain-schedulinggroup.txt" 2>&1 \
  && echo 'GHI NHAN: schema Pod co schedulingGroup' \
  || echo 'PASS: schema Pod cua cluster khong co truong schedulingGroup'
cd ~/bookstore
kubectl create --dry-run=server -f docs/dra/huan-luyen-podgroup.draft.yaml > "$EV/b5-podgroup-dryrun.txt" 2>&1
grep -q 'no matches for kind' "$EV/b5-podgroup-dryrun.txt" \
  && echo 'PASS: API server khong biet kind cua ban nhap PodGroup'

# 2. Bốn gate
for g in GenericWorkload GangScheduling WorkloadAwarePreemption WorkloadWithJob; do
  v="$(grep "^kubernetes_feature_enabled{name=\"$g\"," "$EV/b0-apiserver-metrics.txt" | awk '{print $NF}' | head -1)"
  echo "$g ${v:-khong-biet-ten}"
done > "$EV/b5-feature-gates.txt"
cat "$EV/b5-feature-gates.txt"
[ "$(grep -cE ' (0|1|khong-biet-ten)$' "$EV/b5-feature-gates.txt")" -eq 4 ] && echo 'PASS: du bon gate cua nhom PodGroup'

# 3. Điều kiện nền
GD="$(kubectl get priorityclass -o jsonpath='{range .items[?(@.globalDefault==true)]}{.metadata.name}{"\n"}{end}' | sed '/^$/d' | wc -l)"
kubectl get priorityclass > "$EV/b5-priorityclass.txt"
[ "$GD" -eq 0 ] && echo 'PASS: khong PriorityClass nao la globalDefault'
Z="$(kubectl get nodes -l topology.kubernetes.io/zone --no-headers 2>/dev/null | wc -l)"
R="$(kubectl get nodes -l topology.kubernetes.io/region --no-headers 2>/dev/null | wc -l)"
kubectl get nodes --show-labels > "$EV/b5-node-labels.txt"
{ [ "$Z" -eq 0 ] && [ "$R" -eq 0 ]; } && echo 'PASS: khong node nao co nhan zone, region — chi co truc hostname'
sudo grep -q -- '--config=' /etc/kubernetes/manifests/kube-scheduler.yaml \
  && echo 'GHI NHAN: kube-scheduler nap KubeSchedulerConfiguration — doc file do truoc khi ket luan' \
  || echo 'PASS: kube-scheduler khong nap file cau hinh — khong co cho cau hinh plugin placement'

# 4. Thí nghiệm khởi động một phần trên worker2
to_m() { case "$1" in *m) echo "${1%m}" ;; *) echo $(( $1 * 1000 )) ;; esac; }
ALLOC_M="$(to_m "$(kubectl get node "$W2" -o jsonpath='{.status.allocatable.cpu}')")"
USED_M="$(to_m "$(kubectl describe node "$W2" | awk '/^Allocated resources:/{f=1} f && $1=="cpu"{print $2; exit}')")"
FREE_M=$(( ALLOC_M - USED_M )); REQ_M=$(( FREE_M * 45 / 100 ))
echo "allocatable=${ALLOC_M}m da-xin=${USED_M}m con-trong=${FREE_M}m moi-worker=${REQ_M}m" | tee "$EV/b5-cpu.txt"
SEL='batch.kubernetes.io/job-name=huan-luyen-thu'
dem() { kubectl -n "$NS" get pods -l "$SEL" --field-selector="status.phase=$1" --no-headers 2>/dev/null | wc -l; }
if [ "$GD" -ne 0 ] || [ "$REQ_M" -lt 100 ]; then
  echo 'STOP: co globalDefault hoac worker2 qua chat — khong chay thi nghiem, ghi lai b5-cpu.txt'
else
  sed "s/__CPU__/${REQ_M}m/" "$W/b5-huan-luyen-thu.yaml.tpl" | kubectl apply -f -
  for i in $(seq 1 40); do [ "$(dem Running)" -ge 2 ] && break; sleep 3; done
  sleep 15
  kubectl -n "$NS" get pods -l "$SEL" -o wide | tee "$EV/b5-mot-phan.txt"
  RUN="$(dem Running)"; PEN="$(dem Pending)"
  U="$(kubectl -n "$NS" get pods -l "$SEL" --field-selector=status.phase=Pending -o jsonpath='{.items[0].metadata.uid}')"
  M="$(kubectl -n "$NS" get events --field-selector "involvedObject.uid=$U,reason=FailedScheduling" -o jsonpath='{.items[*].message}')"
  echo "$M" | tee -a "$EV/b5-mot-phan.txt"
  { [ "$RUN" -eq 2 ] && [ "$PEN" -eq 1 ] && echo "$M" | grep -q 'Insufficient cpu'; } \
    && echo 'PASS: hai worker chay va giu CPU, worker thu ba Pending vi thieu CPU — khoi dong mot phan'
  kubectl -n "$NS" delete job huan-luyen-thu --wait=true
  for i in $(seq 1 40); do [ "$(kubectl -n "$NS" get pods -l "$SEL" --no-headers 2>/dev/null | wc -l)" -eq 0 ] && break; sleep 3; done
  [ "$(kubectl -n "$NS" get pods -l "$SEL" --no-headers 2>/dev/null | wc -l)" -eq 0 ] && echo 'PASS: da xoa Job thi nghiem'
fi

# 5. Bản nháp Job cho batch thật
cd ~/bookstore
kubectl get namespace bookstore -o jsonpath='{.metadata.labels}' > "$EV/b5-pss-bookstore.txt"
if kubectl create --dry-run=server -f docs/dra/huan-luyen-job.draft.yaml > "$EV/b5-job-dryrun.txt" 2>&1; then
  grep -qi 'would violate PodSecurity' "$EV/b5-job-dryrun.txt" \
    && echo 'FAIL: ban nhap Job vi pham PodSecurity restricted' \
    || echo 'PASS: ban nhap Job qua server dry-run trong bookstore, khong canh bao PodSecurity'
else
  echo 'FAIL: ban nhap Job bi tu choi — doc b5-job-dryrun.txt'
fi

# 6. ADR
ls docs/adr/*-batch-huan-luyen-theo-nhom.md >/dev/null 2>&1 && echo 'PASS: co ADR batch huan luyen'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/13/b5-huan-luyen-thu.yaml.tpl — __CPU__ được script thay bằng 45% CPU còn trống
apiVersion: batch/v1
kind: Job
metadata:
  name: huan-luyen-thu
  namespace: lab-13
  labels:
    lab: lab-13
spec:
  completionMode: Indexed
  completions: 3
  parallelism: 3
  backoffLimit: 0
  template:
    metadata:
      labels:
        lab: lab-13
    spec:
      restartPolicy: Never
      terminationGracePeriodSeconds: 1
      nodeSelector:
        kubernetes.io/hostname: lab-k8s-worker2
      containers:
      - name: worker
        image: busybox:1.37
        command: ["sh", "-c", "echo worker $JOB_COMPLETION_INDEX; sleep 3600"]
        resources:
          requests:
            cpu: __CPU__
            memory: 16Mi
          limits:
            cpu: __CPU__
            memory: 32Mi
```

```yaml
# docs/dra/huan-luyen-job.draft.yaml — BẢN NHÁP, không apply.
# parallelism = completions là điều kiện để Job controller tự tạo Workload và PodGroup gang
# khi cluster có gate WorkloadWithJob (bài 77). Component training chưa có trong hợp đồng.
apiVersion: batch/v1
kind: Job
metadata:
  name: huan-luyen-goi-y
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: training
spec:
  completionMode: Indexed
  completions: 3
  parallelism: 3
  backoffLimit: 0
  activeDeadlineSeconds: 7200
  ttlSecondsAfterFinished: 3600
  template:
    metadata:
      labels:
        app.kubernetes.io/part-of: bookstore
        app.kubernetes.io/component: training
    spec:
      priorityClassName: bookstore-batch
      restartPolicy: Never
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        runAsGroup: 65534
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: worker
        image: busybox:1.37          # chỗ giữ cho image huấn luyện thật
        command: ["sh", "-c", "echo worker $JOB_COMPLETION_INDEX; sleep 30"]
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
```

```yaml
# docs/dra/huan-luyen-podgroup.draft.yaml — BẢN NHÁP theo tài liệu bài 75, 77, 78, 79.
# Server dry-run trên cluster hiện tại bị từ chối: nhóm API và gate chưa có. Thứ tự tạo bắt buộc:
# Workload -> PodGroup -> Pod (bài 76).
apiVersion: scheduling.k8s.io/v1alpha2
kind: Workload
metadata:
  name: huan-luyen-goi-y
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
spec:
  podGroupTemplates:
  - name: worker
    schedulingPolicy:
      gang:
        minCount: 3
---
apiVersion: scheduling.k8s.io/v1alpha2
kind: PodGroup
metadata:
  name: huan-luyen-goi-y-0
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
spec:
  podGroupTemplateRef:
    workload:
      workloadName: huan-luyen-goi-y
      podGroupTemplateName: worker
  schedulingPolicy:
    gang:
      minCount: 3
  priorityClassName: bookstore-batch
```

Kỳ vọng ở bước 2, theo [bảng feature gate của tài liệu đúng phiên bản baseline](https://v1-35.docs.kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/):
`GenericWorkload` và `GangScheduling` biết tên, đang tắt; `WorkloadAwarePreemption` và
`WorkloadWithJob` không biết tên. Tài liệu của đúng phiên bản baseline mô tả một tiền thân khác của
cùng ý tưởng — Workload ở `scheduling.k8s.io/v1alpha1` và trường Pod `spec.workloadRef`, xem
[Workload Reference](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/workload-reference/).
Tài liệu mới nhất đã đổi sang PodGroup và `spec.schedulingGroup`: đây là ví dụ cụ thể của câu "API
có thể đổi giữa các phiên bản" lặp lại trong cả nhóm bài.

Ý chính cho ADR batch:

1. Với `gang` và `minCount: 3`: Pod chỉ rời `PreEnqueue` khi PodGroup tồn tại **và** đã có ít nhất
   ba Pod được tạo. Chu trình lập lịch PodGroup tìm được chỗ cho hai, không đạt `minCount`, nên
   **không Pod nào được bind**; cả ba về hàng đợi không thể lập lịch, CPU trên worker2 để trống cho
   workload khác. Thí nghiệm của bạn là đúng phương án ngược lại: hai worker giữ gần hết CPU còn
   trống mà không làm được gì.
2. Thiếu nhóm API alpha của PodGroup/Workload và gate `GenericWorkload` đang tắt; hai gate còn lại
   binary chưa biết tên. "Đang tắt" nghĩa là bật lên là có; "không biết tên" nghĩa là phải nâng
   phiên bản Kubernetes — và trần ở
   [A1.4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) chặn điều đó.
3. **Không.** Bài 78: `priority` của PodGroup chỉ được workload-aware preemption tôn trọng, không
   xét trong pha lập lịch. Bài 152: Pod đơn lẻ đi đường preemption mặc định và **không** tôn trọng
   `priority` hay `disruptionMode` của nhóm nạn nhân — `api` có thể đá một worker, làm cả batch
   hỏng.
4. Không. Mọi node có giá trị `kubernetes.io/hostname` riêng, nên mỗi miền chỉ có một node; một
   nhóm ba worker ghim một miền thì chỉ vừa khi một node chứa đủ ba. Gán nhãn zone giả là dựng dữ
   liệu giả, và B6 cho thấy quyền gán nhãn node là quyền nhạy cảm.
5. Ba worker cùng image, cùng `requests`, không affinity — nhóm **đồng nhất**, thuật toán được kỳ
   vọng tìm ra placement nếu có. Thêm một Pod điều phối có yêu cầu khác thì nhóm thành không đồng
   nhất và không còn bảo đảm đó (bài 151, *Hạn chế*). Phương án tạm thời hôm nay: chạy batch ở cửa
   sổ thấp điểm, đặt `requests` đủ để scheduler không xếp được nửa vời, và đặt
   `activeDeadlineSeconds` để Job tự hủy thay vì giữ chỗ cả đêm.

</details>

## B6. [BS-13-06] Điều tra: hardening kube-scheduler

**Loại:** Điều tra.

**Bối cảnh:** Bookstore phụ thuộc vào scheduler theo một cách rất cụ thể: từ Lab 7a, `db` chỉ được
đặt lên node mang nhãn riêng của dự án, do `deploy.sh` gắn cho `lab-k8s-worker2`. Ai sửa được nhãn
node hay cấu hình scheduler là đổi được nơi DB chạy. Bài 124 đưa ra checklist; ticket này đối chiếu
nó với cluster đang chạy.

**Yêu cầu:**

1. Bảng mười hai cờ mà bài 124 nhắc tên: cluster khai cờ nào, giá trị gì. Với cờ không khai, tra
   giá trị mặc định của binary trong
   [trang tham chiếu kube-scheduler đúng phiên bản baseline](https://v1-35.docs.kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)
   và ghi nó có trùng khuyến nghị không.
2. Quyền file của kubeconfig mà scheduler dùng cho `authentication-kubeconfig`.
3. Chứng minh không danh tính thường nào của dự án gán được nhãn node: ServiceAccount `ci-deployer`
   trong `bookstore`, một thành viên nhóm `bookstore-devs`, ServiceAccount `default` của `lab-13`,
   một user đã xác thực bất kỳ.
4. Chứng minh nhãn node đúng là đầu vào lập lịch của Bookstore: spec của `db` phụ thuộc khóa nhãn
   nào, và node nào đang mang nhãn đó.
5. ADR `docs/adr/NNNN-hardening-scheduler-va-dra.md`, mục *Scheduler*: cờ nào đạt, cờ nào lệch, lệch
   thì sửa ở đâu và thuộc giai đoạn nào.

**Ràng buộc:** chỉ đọc. Không sửa manifest control plane, không đổi quyền file, không gán nhãn.

**Đọc lại:**

- Bài [124](../124-hardening-scheduler-vi.md):
  [*Các tùy chọn dòng lệnh về xác thực và phân quyền*](../124-hardening-scheduler-vi.md#các-tùy-chọn-dòng-lệnh-về-xác-thực-và-phân-quyền-của-scheduler-scheduler-authentication--authorization-command-line-options),
  [*… về mạng*](../124-hardening-scheduler-vi.md#các-tùy-chọn-dòng-lệnh-về-mạng-của-scheduler-scheduler-networking-command-line-options),
  [*… về TLS*](../124-hardening-scheduler-vi.md#các-tùy-chọn-dòng-lệnh-về-tls-của-scheduler-scheduler-tls-command-line-options),
  [*Không cho phép gán nhãn node*](../124-hardening-scheduler-vi.md#không-cho-phép-gán-nhãn-node-disallow-labeling-nodes).
- [Lab 7a](LAB-7A-LAP-LICH-VA-EVICTION.md) — node affinity của `db`; [Lab 9a](LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md)
  — `ci-deployer`, `bookstore-devs`, `kubectl auth can-i --as`.

**Bàn giao:** mục *Scheduler* trong `docs/adr/NNNN-hardening-scheduler-va-dra.md`;
`~/lab-evidence/13/b6-*`.

**Nghiệm thu:**

- `PASS:` đủ mười hai dòng cờ.
- `PASS:` `bind-address` là loopback.
- `PASS:` không truyền `--requestheader-client-ca-file`.
- `PASS:` kubeconfig xác thực của scheduler chỉ chủ sở hữu đọc được (hoặc `GHI NHAN:` kèm quyền thật).
- `PASS:` bốn danh tính × hai verb `patch`, `update` trên `nodes` đều `no`.
- `PASS:` `db` phụ thuộc nhãn `bookstore.example.com/data`; chỉ `lab-k8s-worker2` mang nhãn đó.
- `PASS:` ADR có mục *Scheduler*.

**Câu hỏi thiết kế** — trả lời trong mục *Scheduler* của ADR:

1. Hai cờ `authentication-tolerate-lookup-failure` và `profiling` cluster không khai. Mặc định của
   chúng là gì, có trùng khuyến nghị không, và sửa thuộc giai đoạn nào của lộ trình?
2. Vì sao "không khai" không đồng nghĩa với "cấu hình sai"? Cho một ví dụ từ bảng của bạn.
3. Một danh tính có quyền `patch nodes`. Kể hai cách nó làm hỏng Bookstore mà không cần chạm vào
   scheduler hay namespace `bookstore`.

<details>
<summary>Gợi ý</summary>

1. Manifest của scheduler: `/etc/kubernetes/manifests/kube-scheduler.yaml`, đọc bằng `sudo`. Một
   cờ là một dòng `- --<tên>=<giá trị>`; `grep -o -- '--<tên>=[^ "]*'` lấy đúng cờ đó.
2. `sudo stat -c '%a %U:%G %n' <file>`.
3. Gán nhãn node là `patch` hoặc `update` trên tài nguyên `nodes` — không có verb riêng tên "label".
   `kubectl auth can-i patch nodes --as=<user> --as-group=<nhóm>`.
4. Ràng buộc của `db` nằm trong `spec.template.spec` của StatefulSet: `affinity` hoặc `nodeSelector`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
EV=~/lab-evidence/13
sudo cat /etc/kubernetes/manifests/kube-scheduler.yaml > "$EV/b6-scheduler-manifest.yaml"
M="$EV/b6-scheduler-manifest.yaml"

# 1. Mười hai cờ
for f in authentication-kubeconfig authorization-kubeconfig authentication-tolerate-lookup-failure \
         authentication-skip-lookup authorization-always-allow-paths profiling \
         requestheader-client-ca-file bind-address permit-address-sharing permit-port-sharing \
         tls-cipher-suites config; do
  v="$(grep -o -- "--$f=[^ \"]*" "$M" | head -1)"
  echo "${v:-khong khai: --$f}"
done > "$EV/b6-co-scheduler.txt"
cat "$EV/b6-co-scheduler.txt"
[ "$(wc -l < "$EV/b6-co-scheduler.txt")" -eq 12 ] && echo 'PASS: doi chieu du 12 co cua bai 124'
B="$(grep -o -- '--bind-address=[^ "]*' "$M" | cut -d= -f2)"
case "$B" in
  127.0.0.1|localhost|::1) echo "PASS: bind-address=$B — scheduler khong lo ra ngoai node" ;;
  *) echo "GHI NHAN: bind-address=${B:-<mac dinh cua binary>}" ;;
esac
grep -q -- '--requestheader-client-ca-file' "$M" \
  && echo 'GHI NHAN: scheduler nhan --requestheader-client-ca-file' \
  || echo 'PASS: khong truyen --requestheader-client-ca-file'

# 2. Quyền file kubeconfig
KC="$(grep -o -- '--authentication-kubeconfig=[^ "]*' "$M" | cut -d= -f2)"
P="$(sudo stat -c '%a %U:%G %n' "$KC" 2>/dev/null)"; echo "${P:-<khong doc duoc>}" | tee "$EV/b6-kubeconfig-perm.txt"
case "${P%% *}" in
  600|400) echo 'PASS: kubeconfig xac thuc cua scheduler chi chu so huu doc duoc' ;;
  *) echo "GHI NHAN: quyen file ${P:-<khong doc duoc>}" ;;
esac

# 3. Ai gán được nhãn node
for who in '--as=system:serviceaccount:bookstore:ci-deployer' \
           '--as=dev-thu --as-group=bookstore-devs' \
           '--as=system:serviceaccount:lab-13:default' \
           '--as=nguoi-dung-thu --as-group=system:authenticated'; do
  for v in patch update; do
    echo "$who $v nodes -> $(kubectl auth can-i $v nodes $who)"
  done
done > "$EV/b6-gan-nhan-node.txt"
cat "$EV/b6-gan-nhan-node.txt"
[ "$(grep -c -- '-> no$' "$EV/b6-gan-nhan-node.txt")" -eq 8 ] \
  && echo 'PASS: khong danh tinh thuong nao cua du an gan duoc nhan node'

# 4. Nhãn node là đầu vào lập lịch của db
kubectl -n bookstore get statefulset db \
  -o jsonpath='{.spec.template.spec.affinity}{"\n"}{.spec.template.spec.nodeSelector}' > "$EV/b6-db-lap-lich.txt"
grep -q 'bookstore.example.com/data' "$EV/b6-db-lap-lich.txt" && echo 'PASS: db phu thuoc nhan node bookstore.example.com/data'
[ "$(kubectl get nodes -l bookstore.example.com/data=true -o name)" = 'node/lab-k8s-worker2' ] \
  && echo 'PASS: chi lab-k8s-worker2 mang nhan du lieu cua du an'

# 5. ADR
cd ~/bookstore
ADR="$(ls docs/adr/*-hardening-scheduler-va-dra.md 2>/dev/null | head -1)"
{ test -n "$ADR" && grep -q 'Scheduler' "$ADR"; } && echo "PASS: $ADR co muc Scheduler"
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Không có manifest: ticket chỉ đọc. Bảng kỳ vọng trên cluster dựng bằng kubeadm ở Lab 00:

| Cờ | Cluster | Mặc định của binary | So với bài 124 |
| --- | --- | --- | --- |
| `authentication-kubeconfig`, `authorization-kubeconfig` | khai, trỏ `/etc/kubernetes/scheduler.conf` | không có | đạt; file quyền `600` |
| `authentication-tolerate-lookup-failure` | không khai | `true` | **lệch** — bài khuyên `false` |
| `authentication-skip-lookup` | không khai | `false` | đạt |
| `authorization-always-allow-paths` | không khai | `/healthz,/readyz,/livez` | đạt — đúng mặc định bài nêu |
| `profiling` | không khai | `true` | **lệch** — bài khuyên tắt |
| `requestheader-client-ca-file` | không khai | không có | đạt |
| `bind-address` | `127.0.0.1` (kubeadm đặt) | `0.0.0.0` | đạt nhờ kubeadm |
| `permit-address-sharing`, `permit-port-sharing` | không khai | `false` | đạt |
| `tls-cipher-suites` | không khai | bộ mã mặc định của Go | **lệch** — bài đòi khai tường minh |
| `config` | không khai | không có | không có chỗ cấu hình plugin |

Ý chính cho mục *Scheduler*:

1. `authentication-tolerate-lookup-failure` mặc định `true`, `profiling` mặc định `true` — cả hai
   trái khuyến nghị. Sửa là sửa manifest static Pod của control plane:
   [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy), làm qua
   cấu hình kubeadm ở `platform/kubeadm/` của repo để lần dựng lại sau (Lab 8a) giữ được.
2. Mỗi cờ có mặc định riêng. `bind-address` mặc định `0.0.0.0` là lệch, nhưng kubeadm khai
   `127.0.0.1` nên đạt; `permit-port-sharing` không khai nhưng mặc định đã là `false`, đúng khuyến
   nghị. Kết luận chỉ đúng sau khi tra mặc định của đúng binary.
3. Gỡ nhãn `bookstore.example.com/data` khỏi `lab-k8s-worker2`: lần `db-0` bị thay kế tiếp nằm
   Pending, dịch vụ mất DB. Hoặc gán nhãn cho node khác: Pod mới của `db` có thể chọn sai node — với
   PV cục bộ thì kẹt, không có PV thì chạy với dữ liệu trống. Kẻ tấn công không cần phá scheduler,
   chỉ cần nói dối scheduler về node — đúng câu cuối của bài 124.

</details>

## B7. [BS-13-07] Điều tra: hardening DRA — quyền và hạn mức thiết bị

**Loại:** Điều tra.

**Bối cảnh:** nếu tính năng gợi ý ra mắt: ai được tạo lớp thiết bị, ai được xin thiết bị, xin tối
đa bao nhiêu, và driver được ghi gì vào status của claim. `bookstore-staging` (Lab 9b) là tenant thứ
hai — không được để nó xin hết GPU. Ticket này đo hiện trạng và viết bản nháp chính sách.

**Yêu cầu:**

1. Theo bài 172: chứng minh hiện trạng — ServiceAccount `ci-deployer`, một thành viên
   `bookstore-devs`, ServiceAccount `default` của `lab-13` đều **không** tạo được DeviceClass hay
   ResourceSlice. Ghi lại họ có tạo được ResourceClaim, ResourceClaimTemplate trong `bookstore` không.
2. Theo bài 211, bước đầu tiên: lập danh sách danh tính đang có quyền ghi status của ResourceClaim
   — đọc rule DRA **thật** của ClusterRole `system:kube-scheduler`. Ghi rule đó có hay không có
   `resourceclaims/binding`. Đối chiếu điều kiện phiên bản ở mục *Trước khi bạn bắt đầu* của bài
   211 với phiên bản server ở B1.
3. Chứng minh `kubectl auth can-i` của quản trị viên trả `yes` cho `resourceclaims/binding` **và**
   cho một subresource bịa ra — rồi giải thích vì sao `yes` không chứng minh subresource tồn tại.
4. Hạn mức: ResourceQuota `gpu-thu` trong `lab-13` cho phép tối đa **hai** thiết bị lớp
   `lab-13-gpu` trong cả namespace. Claim `goi-y-gpu` của B2 đã chiếm một. Chứng minh: quota đếm
   claim sẵn có; claim xin hai thiết bị bị từ chối ngay lúc tạo; claim xin một được nhận; claim
   thứ hai xin một bị từ chối; `status.used` của quota bằng hai — tất cả **không cần thiết bị nào**.
5. Viết `docs/dra/rbac-quota-gpu.draft.yaml`: ClusterRole cho driver giả định theo mẫu node-local
   của bài 125 (giới hạn bằng `resourceNames`); Role và RoleBinding cho `ci-deployer` quản lý
   ResourceClaimTemplate trong `bookstore`; ResourceQuota giới hạn thiết bị lớp `bookstore-gpu`
   trong `bookstore`. Mọi object mang label dự án. Kiểm bằng server dry-run, không apply.
6. Mục *DRA* trong ADR hardening.

**Ràng buộc:** không tạo Role, ClusterRole hay binding thật. Quota thật chỉ trong `lab-13`.

**Đọc lại:**

- Bài [172](../172-cluster-admin-dra-vi.md#tách-quyền-truy-cập-các-api-liên-quan-đến-dra-separate-permissions-to-dra-related-apis).
- Bài [125](../125-hardening-dra-vi.md):
  [*Tăng cường bảo mật cho quyền cập nhật trạng thái DRA*](../125-hardening-dra-vi.md#tăng-cường-bảo-mật-cho-quyền-cập-nhật-trạng-thái-dra-harden-dra-status-update-permissions),
  [*Các verb DRA có nhận biết node*](../125-hardening-dra-vi.md#các-verb-dra-có-nhận-biết-node-node-aware-dra-verbs),
  [*Các mẫu RBAC ví dụ*](../125-hardening-dra-vi.md#các-mẫu-rbac-ví-dụ-example-rbac-patterns).
- Bài [211](../211-hardening-dra-tasks-vi.md):
  [*Trước khi bạn bắt đầu*](../211-hardening-dra-tasks-vi.md#trước-khi-bạn-bắt-đầu-before-you-begin),
  [*Xác định các thành phần DRA ghi vào status*](../211-hardening-dra-tasks-vi.md#xác-định-các-thành-phần-dra-ghi-vào-status-identify-dra-components-that-write-status),
  [*Gắn role với các danh tính tường minh*](../211-hardening-dra-tasks-vi.md#gắn-role-với-các-danh-tính-tường-minh-bind-roles-to-explicit-identities),
  [*Kiểm chứng và giám sát*](../211-hardening-dra-tasks-vi.md#kiểm-chứng-và-giám-sát-validate-and-monitor).
- Bài [134](../134-resource-quotas-vi.md#hạn-ngạch-cho-dra-resource-claim-quota-for-dra-resource-claims)
  — mục mà Lab 7b để lại cho giai đoạn này.
- [Lab 9a](LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md) — Role, ClusterRole, binding, `can-i`.

**Bàn giao:** `~/lab-work/13/b7-*.yaml`; `docs/dra/rbac-quota-gpu.draft.yaml`; mục *DRA* trong ADR
hardening; `~/lab-evidence/13/b7-*`.

**Nghiệm thu:**

- `PASS:` ba danh tính × hai API cấp cluster đều `no`.
- `PASS:` đã ghi kết quả của ba danh tính với hai API cấp namespace (`PASS:` nếu đều `no`,
  `GHI NHAN:` nếu không).
- `PASS:` đọc được rule DRA thật của `system:kube-scheduler`; một dòng `GHI NHAN:` cho
  `resourceclaims/binding`.
- `PASS:` server thấp hơn phiên bản mà bài 211 đòi (hoặc `GHI NHAN:` nếu không).
- `PASS:` quản trị viên được `yes` cho cả `binding` lẫn subresource bịa.
- `PASS:` quota đếm claim sẵn có; claim xin hai bị từ chối; claim xin một được nhận; claim thứ hai
  bị từ chối; `status.used` bằng 2.
- `PASS:` bản nháp qua server dry-run; mọi object trong bản nháp mang label dự án.

**Câu hỏi thiết kế** — trả lời trong mục *DRA* của ADR hardening:

1. Đặc điểm nào khiến ranh giới của bài 172 gọn với cách dự án chia namespace? Nếu tính năng ra
   mắt, `ci-deployer` cần thêm quyền gì, và vì sao **không** được cấp quyền trên DeviceClass?
2. Cluster của bạn có thực thi kiểm tra trên `resourceclaims/binding` và `resourceclaims/driver`
   không? Bằng chứng nào? Muốn có thì cần gì, và trần ở A1.4 chặn điều đó ra sao?
3. Driver chạy dạng DaemonSet dùng tiền tố verb nào? Cấp nhầm `arbitrary-node:` mất gì, và
   `resourceNames` siết thêm theo chiều nào?
4. Hạn mức thiết bị chặn ở thời điểm nào so với lúc cấp phát? Vì sao điều đó hữu ích cho hai tenant
   `bookstore` và `bookstore-staging` dùng chung GPU?

<details>
<summary>Gợi ý</summary>

1. `kubectl auth can-i create deviceclasses.resource.k8s.io --as=...`; với API cấp namespace thêm
   `-n bookstore`. Thêm `--subresource=<tên>` để hỏi về subresource.
2. In rule của một ClusterRole, mỗi rule một dòng:

   ```bash
   kubectl get clusterrole <tên> \
     -o jsonpath='{range .rules[*]}{.apiGroups}{" "}{.resources}{" "}{.verbs}{"\n"}{end}'
   ```

3. Khóa hạn mức thiết bị: `<tên DeviceClass>.deviceclass.resource.k8s.io/devices`. Claim xin nhiều
   thiết bị bằng `exactly.count`.
4. ResourceQuota mới tạo chỉ có hiệu lực khi controller đã điền `status.hard`. Chờ trường đó trước
   khi tạo claim.
5. Đọc một khóa có dấu chấm và gạch chéo bằng jsonpath:
   `'{.status.used.lab-13-gpu\.deviceclass\.resource\.k8s\.io/devices}'`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; bước 4 apply theo đúng thứ tự</summary>

```bash
NS=lab-13; EV=~/lab-evidence/13; W=~/lab-work/13
. "$EV/b1-nang-luc.env"
WHO=('--as=system:serviceaccount:bookstore:ci-deployer' '--as=dev-thu --as-group=bookstore-devs' \
     '--as=system:serviceaccount:lab-13:default')

# 1. Ranh giới bài 172 với danh tính của dự án
for who in "${WHO[@]}"; do
  for r in deviceclasses resourceslices; do echo "$who create $r -> $(kubectl auth can-i create $r.resource.k8s.io $who)"; done
done > "$EV/b7-ranh-gioi-cluster.txt"
for who in "${WHO[@]}"; do
  for r in resourceclaims resourceclaimtemplates; do
    echo "$who create $r -n bookstore -> $(kubectl auth can-i create $r.resource.k8s.io -n bookstore $who)"
  done
done > "$EV/b7-ranh-gioi-namespace.txt"
cat "$EV/b7-ranh-gioi-cluster.txt" "$EV/b7-ranh-gioi-namespace.txt"
[ "$(grep -c -- '-> no$' "$EV/b7-ranh-gioi-cluster.txt")" -eq 6 ] \
  && echo 'PASS: khong danh tinh nao cua du an tao duoc DeviceClass hay ResourceSlice'
[ "$(grep -c -- '-> no$' "$EV/b7-ranh-gioi-namespace.txt")" -eq 6 ] \
  && echo 'PASS: chua danh tinh nao cua du an tao duoc claim trong bookstore' \
  || echo 'GHI NHAN: co danh tinh tao duoc claim trong bookstore — ghi vao ADR kem role cap quyen do'

# 2. Ai đang ghi status của ResourceClaim
kubectl get clusterrole system:kube-scheduler \
  -o jsonpath='{range .rules[*]}{.apiGroups}{" "}{.resources}{" "}{.verbs}{"\n"}{end}' \
  | grep 'resource.k8s.io' > "$EV/b7-scheduler-dra-rules.txt"
cat "$EV/b7-scheduler-dra-rules.txt"
test -s "$EV/b7-scheduler-dra-rules.txt" && echo 'PASS: doc duoc rule DRA that cua system:kube-scheduler'
grep -q 'resourceclaims/binding' "$EV/b7-scheduler-dra-rules.txt" \
  && echo 'GHI NHAN: scheduler co quyen tren resourceclaims/binding' \
  || echo 'GHI NHAN: scheduler chi co quyen tren resourceclaims/status — chua co subresource tong hop'
MINOR="$(echo "$SV" | sed -n 's/^v1\.\([0-9]*\)\..*/\1/p')"
if [ -n "$MINOR" ] && [ "$MINOR" -lt 36 ]; then
  echo "PASS: server $SV thap hon v1.36 — dieu kien tien quyet cua bai 211 chua dat"
else
  echo "GHI NHAN: server $SV — doc lai muc Truoc khi ban bat dau cua bai 211"
fi

# 3. can-i đánh giá rule trên chuỗi tên
A1="$(kubectl auth can-i update resourceclaims.resource.k8s.io --subresource=binding -n "$NS")"
A2="$(kubectl auth can-i update resourceclaims.resource.k8s.io --subresource=khong-ton-tai -n "$NS")"
echo "admin binding=$A1 ; admin khong-ton-tai=$A2" | tee "$EV/b7-can-i-chuoi.txt"
{ [ "$A1" = yes ] && [ "$A2" = yes ]; } \
  && echo 'PASS: can-i tra yes ca cho subresource bia — no danh gia rule, khong kiem su ton tai'

# 4. Hạn mức thiết bị theo DeviceClass
K='{.status.used.lab-13-gpu\.deviceclass\.resource\.k8s\.io/devices}'
kubectl -n "$NS" delete resourceclaim quota-2 quota-1a quota-1b --ignore-not-found --wait=true
kubectl -n "$NS" delete resourcequota gpu-thu --ignore-not-found
kubectl apply -f "$W/b7-quota.yaml"
for i in $(seq 1 30); do [ -n "$(kubectl -n "$NS" get resourcequota gpu-thu -o jsonpath='{.status.hard}')" ] && break; sleep 2; done
U0="$(kubectl -n "$NS" get resourcequota gpu-thu -o jsonpath="$K")"
echo "used truoc khi tao claim moi=$U0" | tee "$EV/b7-quota.txt"
[ "$U0" = 1 ] && echo 'PASS: quota dem ca claim goi-y-gpu da co tu B2'
kubectl apply -f "$W/b7-claim-2.yaml" >> "$EV/b7-quota.txt" 2>&1
kubectl apply -f "$W/b7-claim-1a.yaml" >> "$EV/b7-quota.txt" 2>&1 && OK1A=1 || OK1A=0
kubectl apply -f "$W/b7-claim-1b.yaml" >> "$EV/b7-quota.txt" 2>&1
U1="$(kubectl -n "$NS" get resourcequota gpu-thu -o jsonpath="$K")"
echo "used sau=$U1" | tee -a "$EV/b7-quota.txt"
cat "$EV/b7-quota.txt"
{ ! kubectl -n "$NS" get resourceclaim quota-2 >/dev/null 2>&1 && \
  [ "$OK1A" -eq 1 ] && kubectl -n "$NS" get resourceclaim quota-1a >/dev/null 2>&1 && \
  ! kubectl -n "$NS" get resourceclaim quota-1b >/dev/null 2>&1 && \
  [ "$(grep -c 'exceeded quota' "$EV/b7-quota.txt")" -eq 2 ] && [ "$U1" = 2 ]; } \
  && echo 'PASS: han muc chan claim ngay luc tao — xin 2 bi tu choi, 1 duoc nhan, them 1 bi tu choi, used=2'

# 5. Bản nháp cho dự án
cd ~/bookstore
F=docs/dra/rbac-quota-gpu.draft.yaml
kubectl create --dry-run=server -f "$F" > "$EV/b7-draft.txt" 2>&1 && echo 'PASS: ban nhap RBAC va quota qua server dry-run'
N="$(kubectl create --dry-run=client -f "$F" -o name | wc -l)"
L="$(kubectl create --dry-run=client -f "$F" -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}{"\n"}' | grep -cx bookstore)"
{ [ "$N" -ge 4 ] && [ "$L" -eq "$N" ]; } && echo "PASS: ca $N object trong ban nhap mang label du an"

# 6. ADR
ADR="$(ls docs/adr/*-hardening-scheduler-va-dra.md 2>/dev/null | head -1)"
{ test -n "$ADR" && grep -q 'DRA' "$ADR"; } && echo "PASS: $ADR co muc DRA"
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/13/b7-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: gpu-thu
  namespace: lab-13
  labels:
    lab: lab-13
spec:
  hard:
    lab-13-gpu.deviceclass.resource.k8s.io/devices: "2"
```

```yaml
# ~/lab-work/13/b7-claim-2.yaml — b7-claim-1a.yaml và b7-claim-1b.yaml chỉ đổi tên, bỏ dòng count
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: quota-2
  namespace: lab-13
  labels:
    lab: lab-13
spec:
  devices:
    requests:
    - name: gpu
      exactly:
        deviceClassName: lab-13-gpu
        count: 2
```

```yaml
# docs/dra/rbac-quota-gpu.draft.yaml — BẢN NHÁP, không apply.
# ClusterRole cho driver chỉ có tác dụng trên server từ v1.36 (bài 211) — xem ADR hardening.
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: bookstore-gpu-driver-status
  labels:
    app.kubernetes.io/part-of: bookstore
rules:
- apiGroups: ["resource.k8s.io"]
  resources: ["resourceclaims/status"]
  verbs: ["get", "patch", "update"]
- apiGroups: ["resource.k8s.io"]
  resources: ["resourceclaims/driver"]
  verbs: ["associated-node:patch", "associated-node:update"]
  resourceNames: ["gpu.bookstore.example.com"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ci-deployer-gpu-claims
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
rules:
- apiGroups: ["resource.k8s.io"]
  resources: ["resourceclaimtemplates"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deployer-gpu-claims
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
subjects:
- kind: ServiceAccount
  name: ci-deployer
  namespace: bookstore
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ci-deployer-gpu-claims
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: bookstore-gpu-quota
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
spec:
  hard:
    bookstore-gpu.deviceclass.resource.k8s.io/devices: "2"
```

Kỳ vọng — verb cụ thể đọc từ file của bạn: các rule DRA của `system:kube-scheduler` cho đọc
DeviceClass, ResourceSlice, ResourceClaim và ghi `resourceclaims/status`, **không** có
`resourceclaims/binding`. Cơ chế subresource tổng hợp có từ v1.36 (bài 125, bài 211), trong khi tài
liệu đúng phiên bản baseline không có trang hardening DRA nào. Danh sách của bước 211 trên cluster
này chỉ có một tên: kube-scheduler.

Ý chính cho mục *DRA*:

1. Hai API của người triển khai Pod — ResourceClaim, ResourceClaimTemplate — **cấp namespace**,
   nên cấp đúng cho `bookstore` là đủ, `bookstore-staging` không bị ảnh hưởng. DeviceClass và
   ResourceSlice cấp cluster: mở cho CI là mở cho toàn cluster, và CI có thể tạo một lớp chọn thiết
   bị của tenant khác. `ci-deployer` chỉ cần quản lý ResourceClaimTemplate trong `bookstore`.
2. **Không thực thi.** Bằng chứng: phiên bản server thấp hơn v1.36 và ClusterRole của scheduler
   không có `resourceclaims/binding`. `can-i` trả `yes` cho cả subresource bịa: nó chỉ so chuỗi với
   rule RBAC — quản trị viên có `*` nên khớp mọi chuỗi. Muốn có kiểm tra này phải lên v1.36, nhưng
   trần ở [A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) (chart Rancher) chặn
   việc nâng phiên bản. Hardening DRA của bài 125 vì thế là **điều kiện tiên quyết** của tính năng.
3. `associated-node:<verb>` — API server xác minh driver chỉ ghi claim của node mà nó chạy. Cấp nhầm
   `arbitrary-node:` thì mất đúng phép kiểm đó: driver trên một node bị chiếm ghi được claim của mọi
   node. `resourceNames` siết theo chiều **driver**: danh tính chỉ ghi được status cho đúng driver
   `gpu.bookstore.example.com`.
4. Quota chặn ở **admission**, lúc tạo claim — trước mọi lần cấp phát, không cần thiết bị. Claim
   chưa cấp phát vẫn bị đếm. Hai tenant mỗi bên một quota theo DeviceClass thì không bên nào xin
   vượt phần của mình, kể cả khi GPU đang rảnh.

</details>

## B8. [BS-13-08] ADR: Bookstore có nên dùng DRA, và khi nào

**Loại:** Điều tra — tổng hợp.

**Bối cảnh:** đội sản phẩm cần một câu trả lời đọc trong năm phút. Bảy ticket trước cho số đo; ticket
này chốt quyết định, điều kiện để xem lại, và những gì vận hành phải đổi nếu quyết định thay đổi.

**Yêu cầu:** ADR `docs/adr/NNNN-dra-bookstore.md` có đủ năm mục `## Bối cảnh`, `## Quyết định`,
`## Lý do`, `## Hệ quả`, `## Điều kiện xem lại`, cộng các mục điều tra đã viết ở B1–B4. Nội dung
phải trả lời:

1. Tính năng gợi ý: làm hay không làm hôm nay; nếu làm sau thì bằng con đường nào trong ba con
   đường, vì sao theo bài 149.
2. Batch huấn luyện: chờ gang scheduling hay chạy tạm bằng cách nào, rủi ro của cách tạm.
3. Điều kiện xem lại **đo được**: những con số nào trong `b1-nang-luc.env`, `b1-feature-gates.txt`,
   `b5-feature-gates.txt` phải đổi, và đổi thành gì.
4. Tác động lên vận hành nếu duyệt: quy trình bảo trì node của Lab 12 (drain driver muộn nhất, nâng
   cấp liền mạch, liveness probe của driver — bài 172); quota và RBAC từ B7; metric phải theo dõi.
5. Ràng buộc nền tảng: phiên bản tối thiểu cho hardening DRA so với trần ở A1.4.
6. Hợp đồng dự án sẽ đổi gì nếu duyệt — component mới, DeviceClass, quota. Ghi ở dạng đề xuất;
   **không** sửa `DU-AN-BOOKSTORE.md`.
7. Dẫn ít nhất bốn file bằng chứng trong `~/lab-evidence/13/`.

**Ràng buộc:** không sửa `k8s/`, `scripts/`, `tests/`, `platform/`. Viết bằng lời của bạn.

**Đọc lại:**

- Bài [172](../172-cluster-admin-dra-vi.md):
  [*Triển khai và bảo trì DRA driver*](../172-cluster-admin-dra-vi.md#triển-khai-và-bảo-trì-dra-driver-dra-driver-deployment-and-maintenance)
  và ba mục con.
- Bài [149](../149-dynamic-resource-allocation-vi.md#hạn-chế-limitations), mục *Hạn chế*.
- Bài [211](../211-hardening-dra-tasks-vi.md#trước-khi-bạn-bắt-đầu-before-you-begin).
- [A1.4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) — đoạn về trần
  phiên bản.
- [Mục 2.2 của file dự án](DU-AN-BOOKSTORE.md#22-yêu-cầu-của-hệ-thống-mới) — yêu cầu `R18`.

**Bàn giao:** `docs/adr/NNNN-dra-bookstore.md` hoàn chỉnh; năm bản nháp trong `docs/dra/`.

**Nghiệm thu:**

- `PASS:` ADR có đủ năm mục bắt buộc.
- `PASS:` ADR dẫn ít nhất bốn file bằng chứng khác nhau trong `~/lab-evidence/13/`.
- `PASS:` đủ năm bản nháp trong `docs/dra/` và hai ADR còn lại.
- `PASS:` ngoài `docs/`, working tree không khác tag `lab-12`.

**Câu hỏi thiết kế** — trả lời trong mục *Hệ quả* của ADR:

1. Ngày mai cluster có GPU. Liệt kê theo đúng thứ tự các việc phải làm trước khi dịch vụ gợi ý chạy
   được, và với mỗi việc, phép kiểm nào của sprint này cho biết việc đó đã xong.
2. Vì sao một ADR kết luận "chưa dùng" vẫn có giá trị? Nó khác một nợ kỹ thuật `TD-nn` ở điểm nào?

<details>
<summary>Gợi ý</summary>

- Bắt đầu từ mẫu `docs/adr/0000-mau.md` của Lab 3a, thêm mục `## Điều kiện xem lại`.
- Điều kiện xem lại tốt có dạng "`SLICE_N` ≥ 1 và `DRV_N` ≥ 1 trong `b1-nang-luc.env`", không có
  dạng "khi có GPU".
- Thứ tự việc của câu hỏi 1 lấy từ bài 271 (vai trò quản trị viên) rồi bài 270 (vai trò người vận
  hành workload).

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
cd ~/bookstore
ADR="$(ls docs/adr/*-dra-bookstore.md 2>/dev/null | head -1)"
BAD=0
for H in '## Bối cảnh' '## Quyết định' '## Lý do' '## Hệ quả' '## Điều kiện xem lại'; do
  grep -q "^$H" "$ADR" || { echo "FAIL: ADR thieu muc $H"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo "PASS: $ADR du nam muc bat buoc"
[ "$(grep -o 'lab-evidence/13/[A-Za-z0-9._-]*' "$ADR" | sort -u | wc -l)" -ge 4 ] \
  && echo 'PASS: ADR dan it nhat bon file bang chung'
BAD=0
for F in docs/dra/deviceclass-gpu.draft.yaml docs/dra/claim-goi-y.draft.yaml docs/dra/huan-luyen-job.draft.yaml \
         docs/dra/huan-luyen-podgroup.draft.yaml docs/dra/rbac-quota-gpu.draft.yaml; do
  test -f "$F" || { echo "FAIL: thieu $F"; BAD=1; }
done
for F in 'docs/adr/*-batch-huan-luyen-theo-nhom.md' 'docs/adr/*-hardening-scheduler-va-dra.md'; do
  ls $F >/dev/null 2>&1 || { echo "FAIL: thieu $F"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: du nam ban nhap va hai ADR con lai'
{ git diff --quiet lab-12 -- . ':(exclude)docs' && \
  test -z "$(git status --porcelain -- . ':(exclude)docs')"; } \
  && echo 'PASS: ngoai docs/, working tree khong khac lab-12'
```

</details>

<details>
<summary>Lời giải tham khảo — khung ADR</summary>

```markdown
# NNNN — Bookstore và cấp phát thiết bị bằng DRA

- Trạng thái: chấp nhận
- Sprint: Lab 13, ticket BS-13-01 → BS-13-08

## Bối cảnh

Đề xuất tính năng gợi ý sách bằng GPU và batch huấn luyện ba worker (R18). Năng lực cluster
(~/lab-evidence/13/b1-nang-luc.env): nhóm API DRA có, không ResourceSlice, không driver — nhánh B.
Luồng claim dừng ở bước Lọc ResourceSlice (~/lab-evidence/13/b2-sau-claim.txt). PodGroup không có
trên cluster (~/lab-evidence/13/b5-feature-gates.txt). Server chưa tới phiên bản mà hardening DRA
đòi (~/lab-evidence/13/b7-scheduler-dra-rules.txt).

## Quyết định

Chưa triển khai tính năng gợi ý bằng GPU. Khi triển khai, dùng DRA với ResourceClaimTemplate theo
bản nháp docs/dra/, không dùng extended resource hay device plugin. Batch huấn luyện chạy tạm bằng
Job Indexed (docs/dra/huan-luyen-job.draft.yaml) ở cửa sổ thấp điểm.

## Lý do

Bài 149: DRA lọc thiết bị bằng CEL, chia sẻ qua claim, Pod không tự khai số lượng — ba điều device
plugin không làm được. Extended resource chỉ là một con số (~/lab-evidence/13/b4-hai-thong-diep.txt).
Không có phần cứng thì không có gì để cấp phát; mọi manifest vẫn được nhận nên triển khai bây giờ
là phát hành một tính năng không bao giờ chạy.

## Hệ quả

- Quy trình bảo trì node (Lab 12) thêm bước: drain Pod có claim trước, driver DRA sau cùng; chọn
  driver có nâng cấp liền mạch; DaemonSet driver phải có liveness probe qua socket gRPC.
- Trước khi ra mắt: áp quota và RBAC theo docs/dra/rbac-quota-gpu.draft.yaml; theo dõi metric
  workqueue resource_claim của kube-controller-manager và dra_operations_duration_seconds của kubelet.
- Đề xuất đổi hợp đồng: component recommender và training, DeviceClass bookstore-gpu,
  ResourceQuota bookstore-gpu-quota. Chưa sửa DU-AN-BOOKSTORE.md.
- Scheduler không preempt cho tài nguyên DRA: batch giữ GPU thì Pod gợi ý chờ.

## Điều kiện xem lại

- SLICE_N ≥ 1 và DRV_N ≥ 1 trong b1-nang-luc.env (có thiết bị và driver).
- Phiên bản server ≥ v1.36 và ClusterRole của scheduler có resourceclaims/binding — đòi gỡ trần
  phiên bản ở A1.4.
- Với batch: GenericWorkload, GangScheduling, WorkloadWithJob biết tên và bật trong
  b5-feature-gates.txt.
```

Ý chính cho câu hỏi thiết kế:

1. Gắn thiết bị vào node → (nếu cần) bật nhóm API → xác minh `kubectl get deviceclasses` (B1) → cài
   driver, xác minh bằng `kubectl get resourceslices` có `DRIVER` (B1) → đọc thuộc tính trong
   ResourceSlice, sửa selector của `bookstore-gpu`, apply (B2) → áp quota và RBAC (B7) → tạo
   ResourceClaimTemplate, Deployment gợi ý (B3), kiểm claim có `status.allocation` (B2 nhánh A).
2. Nó biến một ý kiến thành một tập điều kiện đo được, nên lần sau chỉ cần chạy lại các phép đo
   thay vì tranh luận lại. Nợ kỹ thuật là chỗ hệ thống **đang dùng** còn yếu; ở đây hệ thống không
   dùng thiết bị nào nên không có gì yếu — chỉ có một tính năng chưa đủ điều kiện ra đời.

</details>

## B9. Đóng sprint: cleanup và gate cuối

**Mục đích:** bàn giao repo có tag `lab-13` chỉ khác `lab-12` ở `docs/`, dọn mọi object thử nghiệm
cả trong namespace lẫn cấp cluster, rồi chứng minh bằng so sánh giá trị rằng cluster trở về đúng
`04-metrics-ready` và lab không cài thêm gì.

1. **`tests/smoke.sh` giữ nguyên bản Lab 12.** Sprint này không thêm, không đổi hợp đồng nào của hệ
   thống, nên bộ kiểm thử khói tích lũy của sprint 19 vẫn là bộ đầy đủ. Không thêm kiểm tra DRA vào
   smoke: tính năng chưa tồn tại. Kiểm rằng file không đổi:

```bash
cd ~/bookstore
git diff --quiet lab-12 -- tests/smoke.sh && echo 'PASS: tests/smoke.sh giu nguyen ban Lab 12'
```

2. Cập nhật `docs/tech-debt.md`: thêm một dòng ghi chú *Lab 13 — không trả, không phát sinh nợ của
   ứng dụng*, trỏ tới ADR hardening cho phát hiện về cờ kube-scheduler (sửa thuộc giai đoạn 20).

3. Dựng lại sạch từ repo và chạy kiểm thử khói:

```bash
cd ~/bookstore
./scripts/teardown.sh && ./scripts/deploy.sh > ~/lab-evidence/13/b9-deploy.txt 2>&1
./tests/smoke.sh > ~/lab-evidence/13/b9-smoke.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/13/b9-smoke.txt
```

4. Commit, tag, đóng gói, chép ra ngoài VM theo
   [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm):

```bash
cd ~/bookstore
git status --short
git add -A && git commit -m 'lab-13: sprint 20 — danh gia DRA, ADR va ban nhap trong docs/'
git tag lab-13
git bundle create ~/bookstore-lab-13.bundle --all && git bundle verify ~/bookstore-lab-13.bundle
```

Chép `~/bookstore-lab-13.bundle` về máy host bằng `scp` trước khi đi tiếp.

5. Dọn object thử nghiệm — xóa Pod trước để claim được gỡ finalizer, rồi namespace, rồi object cấp
   cluster — và trả cluster:

```bash
kubectl -n lab-13 delete pod --all --wait=true --timeout=180s
kubectl delete namespace lab-13 --wait=true --timeout=300s
kubectl delete deviceclass -l lab=lab-13 --ignore-not-found
rm -rf ~/lab-work/13
cd ~/bookstore && ./scripts/teardown.sh
```

6. Gate cuối:

```bash
EV=~/lab-evidence/13; . "$EV/b0-anh-nen.env"
cd ~/bookstore
test -z "$(git status --porcelain)" && echo 'PASS: repo khong con thay doi chua commit'
git describe --tags --exact-match HEAD | grep -qx 'lab-13' && echo 'PASS: HEAD mang tag lab-13'
test -s ~/bookstore-lab-13.bundle && echo 'PASS: co bundle'
test -z "$(git diff --name-only lab-12 lab-13 -- . ':(exclude)docs')" && echo 'PASS: lab-13 chi doi docs/ so voi lab-12'
{ [ "$(kubectl get namespace,priorityclass,storageclass -l app.kubernetes.io/part-of=bookstore --no-headers 2>/dev/null | wc -l)" -eq 0 ] && \
  [ "$(kubectl get all -A -l app.kubernetes.io/part-of=bookstore --no-headers 2>/dev/null | wc -l)" -eq 0 ]; } \
  && echo 'PASS: khong con object nao cua du an'
[ "$(kubectl get namespace,deviceclass -l lab=lab-13 --no-headers 2>/dev/null | wc -l)" -eq 0 ] \
  && echo 'PASS: khong con object thu nghiem cua lab 13'
test ! -e ~/lab-work/13 && echo 'PASS: da don lab-work'
{ [ "$(kubectl get deviceclasses --no-headers 2>/dev/null | wc -l)" -eq "$DC_BEFORE" ] && \
  [ "$(kubectl get resourceslices --no-headers 2>/dev/null | wc -l)" -eq "$SLICE_BEFORE" ]; } \
  && echo "PASS: DeviceClass, ResourceSlice ve dung muc dau ($DC_BEFORE, $SLICE_BEFORE)"
[ "$(kubectl api-resources -o name 2>/dev/null | sort -u | wc -l)" -eq "$API_BEFORE" ] \
  && echo "PASS: so API resource khong doi ($API_BEFORE) — khong bat nhom API nao"
[ "$(kubectl get crd -o name 2>/dev/null | wc -l)" -eq "$CRD_BEFORE" ] && echo "PASS: so CRD khong doi ($CRD_BEFORE)"
kubectl get --raw /metrics > "$EV/b9-apiserver-metrics.txt"
{ [ "$(grep -c '^kubernetes_feature_enabled{' "$EV/b9-apiserver-metrics.txt")" -eq "$FG_ALL_BEFORE" ] && \
  [ "$(grep '^kubernetes_feature_enabled{' "$EV/b9-apiserver-metrics.txt" | grep -c ' 1$')" -eq "$FG_ON_BEFORE" ]; } \
  && echo "PASS: bang feature gate khong doi ($FG_ON_BEFORE/$FG_ALL_BEFORE)"
[ "$(kubectl get namespace -o name | sort | tr '\n' ' ')" = "$NS_BEFORE" ] && echo 'PASS: danh sach namespace khong doi'
[ "$(kubectl get priorityclass -o name | sort | tr '\n' ' ')" = "$PC_BEFORE" ] && echo 'PASS: danh sach PriorityClass khong doi'
[ "$(kubectl get nodes --no-headers --show-labels | awk '{print $1, $NF}' | sort | md5sum | cut -d' ' -f1)" = "$LBL_BEFORE" ] \
  && echo 'PASS: nhan cua ca ba node khong doi'
test "$(kubectl top node --no-headers 2>/dev/null | wc -l)" -eq 3 && echo 'PASS: metrics-server van tra so lieu ca ba node'

kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

**PASS:** dòng `PASS:` ở bước 1; `smoke exit=0` và file smoke ở bước 3 in toàn `PASS:`; mười lăm
dòng `PASS:` của gate cuối trước khối A5.5 xuất hiện; dòng `PASS: readyz ok`; ba node `Ready`; lệnh
field selector trả `No resources found`; CoreDNS đủ replica; namespace `default` không có Pod. Thư
mục `~/lab-evidence/13/` **giữ lại**. Cluster trở về `04-metrics-ready`; **không tạo snapshot mới**.

Ba phép so `API resource`, `CRD`, `feature gate` biến câu "lab không cài gì" từ một lời hứa thành
một phép đo. Phép so nhãn node đồng thời kiểm `teardown.sh` của Lab 12 đã gỡ nhãn mà `deploy.sh` gắn.

---

## 3. Checkpoint 13

Không đánh dấu lab hoàn tất chỉ vì script nghiệm thu PASS. Tự trả lời, không nhìn tài liệu:

- [ ] Bốn điều kiện để cluster cấp phát được thiết bị bằng DRA là gì? Cluster Bookstore thiếu điều
      kiện nào, đọc ra từ file nào, và vì sao thiếu đó không thành nợ lab? `No resources found`
      khác `the server doesn't have a resource type` ở đâu?
- [ ] Bốn kind API của DRA: ai tạo cái nào, cái nào cấp cluster, cái nào cấp namespace? Vì sao
      phạm vi đó cũng chính là ranh giới quyền giữa đội vận hành và `ci-deployer`?
- [ ] Pod `goi-y-gpu` trước và sau khi có claim dừng ở hai bước nào của luồng năm bước? Bạn phân
      biệt hai điểm dừng bằng tín hiệu nào, và vì sao API server nhận mọi manifest mà không báo lỗi
      lại là rủi ro cho quy trình phát hành?
- [ ] Dịch vụ gợi ý hai bản sao: claim tạo tay hay template, vì sao? Ai sinh claim từ template, nằm
      ở đâu trong control plane? Claim đã sinh mà chưa cấp phát nghĩa là gì? Xóa Pod thì claim nào
      mất, claim nào còn?
- [ ] Ba con đường đưa GPU vào Pod nằm ở những trường nào của Pod spec và hạch toán ở đâu? Hai thông
      điệp Pending của `goi-y-er` và `goi-y-gpu` khác nhau thế nào? Kể ba thiếu sót của device
      plugin theo bài 149.
- [ ] "API có trường" khác "tính năng đang bật" ra sao? Bạn chứng minh bằng trường nào, lệnh nào?
      Vì sao viết manifest theo tài liệu bản mới nhất là nguy hiểm trên cluster của bạn?
- [ ] Thí nghiệm `huan-luyen-thu` cho thấy gì? Với PodGroup `gang` `minCount: 3`, cùng tình huống
      diễn ra thế nào, và hai điều kiện nào phải đúng để Pod rời `PreEnqueue`?
- [ ] Cluster thiếu chính xác những gì để dùng PodGroup? "Gate không biết tên" khác "gate đang tắt"
      ở điểm nào? `bookstore-batch` gắn ở cấp nhóm có làm batch được xếp trước không?
- [ ] Ràng buộc topology trên ba VM có ý nghĩa không? Vì sao không gán nhãn zone? Một danh tính có
      `patch nodes` làm hỏng `db` của Bookstore bằng cách nào?
- [ ] Trong mười hai cờ của bài 124, cờ nào cluster đạt nhờ kubeadm, cờ nào lệch vì mặc định của
      binary? Sửa ở giai đoạn nào, qua file nào của repo?
- [ ] `resourceclaims/binding` và `resourceclaims/driver` gác những trường nào? Cluster của bạn có
      thực thi chúng không, bằng chứng nào, và trần ở A1.4 liên quan ra sao? Vì sao `can-i` trả
      `yes` cho một subresource bịa?
- [ ] Hạn mức thiết bị theo DeviceClass chặn ở thời điểm nào? Vì sao claim `goi-y-gpu` chưa cấp phát
      vẫn bị đếm? Lab không tạo snapshot mới — bạn chứng minh cluster về đúng `04-metrics-ready`
      bằng những phép so nào?

Nợ còn mở sau lab này — nói được **vì sao chưa trả**:

- Sprint này không phát sinh và không trả nợ lab nào. Nợ lab còn mở theo
  [sổ nợ lab](README.md#5-sổ-nợ-lab): #1 phần VPA (add-on ngoài A1.4), #5 snapshot volume (cần CSI
  driver), #6 mã hóa Secret at rest (giai đoạn 22), #7 vòng đời certificate (giai đoạn 18), #8 backup
  etcd (giai đoạn 19).
- Nợ kỹ thuật của ứng dụng không đổi — xem `docs/tech-debt.md` của repo. Phát hiện về cờ
  kube-scheduler là việc của hạ tầng cluster, đã ghi trong ADR hardening.
- Phần cấp phát thiết bị thật, PodGroup chạy thật và hardening DRA thực thi được không phải nợ: chúng
  chờ phần cứng và phiên bản, với điều kiện xem lại đã ghi trong ADR.

### Bài giải thích cuối cùng

Trong vài phút, kể lại **một yêu cầu "cho dịch vụ gợi ý một GPU" đi được tới đâu trên cluster
Bookstore, dừng ở đâu, và vì sao**: từ phép đo năng lực ở B1; DeviceClass `lab-13-gpu` được nhận mà
không có thiết bị nào; Pod đến trước claim, rồi claim đến và scheduler xét mà không cấp phát được;
template sinh claim cho từng Pod nhưng sinh claim không phải cấp phát. Đặt cạnh đó cách làm cũ — một
con số trong `limits` — và thông điệp Pending khác hẳn của nó. Rồi kể batch huấn luyện: hai worker
giữ CPU chờ worker thứ ba, gang scheduling lẽ ra ngăn điều đó thế nào, và vì sao cluster chưa có
nó. Kết thúc bằng hai lớp an toàn — scheduler và quyền DRA, quota thiết bị — và quyết định trong ADR
kèm điều kiện xem lại đo được. Nói rõ ở mỗi chặng: object nào đang tồn tại, trường nào rỗng, thành
phần nào đã làm việc và thành phần nào chưa từng được gọi.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường xem [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Dưới đây chỉ là sự cố phát sinh trong nội dung bài học.

| Triệu chứng | Nguyên nhân thường gặp | Xử lý |
| --- | --- | --- |
| B0: `kubectl top node` báo lỗi | Cluster không ở `04-metrics-ready` | Restore cả ba VM về `04-metrics-ready` |
| B0: `git describe` không ra `lab-12` | Repo ở commit khác, hoặc clone từ bundle cũ | `git checkout lab-12` hoặc clone lại từ `~/bookstore-lab-12.bundle` mới nhất |
| B0: `deploy.sh` hay `smoke.sh` fail | Repo của sprint 19 chưa đủ | Sửa repo ở tag `lab-12`, không sửa tay trên cluster |
| B1: `kubectl get deviceclasses` báo `the server doesn't have a resource type` | Nhóm API `resource.k8s.io` không được phục vụ | **Không phải lỗi của bạn.** Ghi lại, bỏ qua phần DRA của B2, B3, B4, B7, đi tiếp từ B5. Không bật nhóm API |
| B1: không có dòng `kubernetes_feature_enabled` nào | Không đọc được `/metrics` lúc B0, hoặc user không đủ quyền | Chạy lại phần ảnh nền của B0 bằng kubeconfig quản trị; nếu vẫn rỗng, dùng manifest và `configz`, ghi rõ vào ADR |
| B1: `configz` hoặc `/metrics` qua node proxy báo `403` | Sai context hoặc user | Chạy bằng kubeconfig quản trị trên `lab-k8s-master`; không tạo binding để chữa |
| B2: `apply` DeviceClass báo không nhận ra `apiVersion` | Cluster phục vụ version khác | Đọc cột `APIVERSION` trong `~/lab-evidence/13/b1-api-resource.txt`, sửa manifest cho khớp; không đoán |
| B2: Pod `goi-y-gpu` Running dù claim không cấp phát | Scheduler hoặc kubelet bỏ qua claim — gate `DynamicResourceAllocation` bị tắt ở một thành phần | Đọc bước khắc phục 2 trong mục *Xác minh* của bài 271; ghi lại, không sửa control plane |
| B2: không có `FailedScheduling` sau khi tạo claim | Pod đang trong backoff của scheduler | Chờ thêm; `kubectl -n lab-13 describe pod goi-y-gpu`. Vòng chờ của script là 2 phút |
| B2: `status.allocation` của `goi-y-gpu` không rỗng ở nhánh B | Cluster có thiết bị mà B1 đọc thiếu | Chạy lại B1, cập nhật `NHANH`, đọc lại kết luận theo nhánh A |
| B3: không có claim nào sinh từ template | `--controllers` của kube-controller-manager loại controller ResourceClaim | Cluster lệch baseline — ghi lại, không sửa manifest |
| B3: claim sinh ra không biến mất sau khi xóa Pod | Finalizer của claim chờ controller gỡ | Chờ; `kubectl -n lab-13 get resourceclaim -o yaml` xem `finalizers`. Không xóa finalizer bằng tay |
| B4: `goi-y-er` không có `Insufficient example.com/gpu` mà Running | Một node còn quảng bá `example.com/gpu` từ lab trước | Đọc `allocatable` của các node; gỡ đúng cách ở mục cleanup của Lab 3c, hoặc restore snapshot |
| B4: dry-run DeviceClass giữ nguyên `extendedResourceName` | Gate `DRAExtendedResource` đang bật | Đối chiếu `b1-feature-gates.txt`; ghi `GHI NHAN` vào ADR |
| B5: `STOP` vì worker2 quá chật | Pod của dự án trên worker2 đã xin gần hết CPU | Không thu nhỏ dự án để chữa; ghi lại `b5-cpu.txt` và trả lời câu hỏi thiết kế bằng lý thuyết |
| B5: cả ba worker Running | Phép tính CPU còn trống sai — `describe node` in số dạng khác | Đọc `b5-cpu.txt` và phần `Allocated resources` của `kubectl describe node lab-k8s-worker2`; xóa Job, chạy lại |
| B5: dry-run bản nháp Job cảnh báo `would violate PodSecurity` | Thiếu trường mà `restricted` đòi | Bổ sung `securityContext` theo Lab 9b |
| B5: dry-run bản nháp Job bị quota hoặc LimitRange của `bookstore` từ chối | `requests`/`limits` của bản nháp vượt `bookstore-limits` hoặc quota | Đọc thông báo, chỉnh con số trong bản nháp cho khớp chính sách đã chốt ở Lab 7b |
| B6: `sudo cat` báo không có file manifest | Đang ở worker | B6 chỉ chạy trên `lab-k8s-master` |
| B6: một danh tính của dự án `patch nodes` được | RBAC của Lab 9a cấp quá rộng | Đây là phát hiện thật: ghi vào ADR hardening, sửa ở repo trong sprint kế tiếp — không sửa ở sprint chỉ đổi `docs/` |
| B6: không tìm thấy nhãn `bookstore.example.com/data` | Lab 7a hiện thực ràng buộc `db` bằng khóa khác | Đọc `b6-db-lap-lich.txt`, dùng đúng khóa mà repo của bạn dùng |
| B7: claim xin hai thiết bị vẫn được nhận | Quota tạo xong nhưng `status.hard` chưa được điền khi claim tới | Xóa các claim `quota-*`, chờ `status.hard` có giá trị, chạy lại bước 4 |
| B7: `status.used` không đếm claim `goi-y-gpu` | Claim của B2 đã bị xóa | Tạo lại claim theo B2 rồi chạy lại bước 4 |
| B7: dry-run RBAC báo `attempting to grant RBAC permissions not currently held` | Không chạy bằng kubeconfig quản trị | Chạy bằng kubeconfig quản trị trên `lab-k8s-master` |
| B9: namespace `lab-13` kẹt `Terminating` | Còn claim mang finalizer | `kubectl -n lab-13 get pods,resourceclaims`; xóa Pod trước rồi chờ; không cưỡng chế finalizer |
| B9: số API resource khác ảnh nền | API tổng hợp của metrics-server tạm thời không trả lời | `kubectl get apiservice v1beta1.metrics.k8s.io`; chờ `Available=True` rồi chạy lại gate. Vẫn khác thì restore snapshot |
| B9: nhãn node khác ảnh nền | Teardown không gỡ nhãn `bookstore.example.com/data`, hoặc có nhãn bị thêm trong lab | So `kubectl get nodes --show-labels` với `b5-node-labels.txt`; gỡ đúng nhãn thừa, ghi lỗi của `teardown.sh` vào báo cáo sprint |

---

## 5. Nguồn chính thức

- [Kubernetes — Dynamic Resource Allocation](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)
- [Kubernetes — Good practices for Dynamic Resource Allocation as a Cluster Admin](https://kubernetes.io/docs/concepts/cluster-administration/dra/)
- [Kubernetes — Hardening Guide: Dynamic Resource Allocation](https://kubernetes.io/docs/concepts/security/hardening-guide/dynamic-resource-allocation/)
- [Kubernetes — Hardening Guide: Scheduler Configuration](https://kubernetes.io/docs/concepts/security/hardening-guide/scheduler/)
- [Kubernetes — Scheduling Group](https://kubernetes.io/docs/concepts/workloads/pods/scheduling-group/)
- [Kubernetes — PodGroup API](https://kubernetes.io/docs/concepts/workloads/podgroup-api/)
- [Kubernetes — PodGroup Lifecycle](https://kubernetes.io/docs/concepts/workloads/podgroup-api/lifecycle/)
- [Kubernetes — Workload API](https://kubernetes.io/docs/concepts/workloads/workload-api/)
- [Kubernetes — Pod Group Disruption and Priority](https://kubernetes.io/docs/concepts/workloads/workload-api/disruption-and-priority/)
- [Kubernetes — PodGroup Scheduling Policies](https://kubernetes.io/docs/concepts/workloads/workload-api/policies/)
- [Kubernetes — Topology-Aware Workload Scheduling (Workload API)](https://kubernetes.io/docs/concepts/workloads/workload-api/topology-aware-scheduling/)
- [Kubernetes — Gang Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/)
- [Kubernetes — PodGroup Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/podgroup-scheduling/)
- [Kubernetes — Workload-Aware Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/workload-aware-preemption/)
- [Kubernetes — Topology-Aware Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-aware-scheduling/)
- [Kubernetes — Assign Devices to Pods and Containers](https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/)
- [Kubernetes — Set Up DRA in a Cluster](https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/set-up-dra-cluster/)
- [Kubernetes — Allocate Devices to Workloads with DRA](https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/allocate-devices-dra/)
- [Kubernetes — Access DRA Device Metadata](https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/access-dra-device-metadata/)
- [Kubernetes — Harden Dynamic Resource Allocation in Your Cluster](https://kubernetes.io/docs/tasks/administer-cluster/hardening-dra/)
- [Kubernetes — Resource Quotas, mục Quota for DRA resource claims](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Kubernetes v1.35 — Feature Gates](https://v1-35.docs.kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
- [Kubernetes v1.35 — kube-scheduler](https://v1-35.docs.kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)
- [Kubernetes v1.35 — Workload API](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/workload-api/)
- [Kubernetes v1.35 — Workload Reference](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/workload-reference/)
- [Kubernetes — Device Plugins — đọc ở giai đoạn 14, dùng để so sánh với DRA](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)

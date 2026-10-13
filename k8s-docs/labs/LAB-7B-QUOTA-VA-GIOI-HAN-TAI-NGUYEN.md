# Lab 7b — Quota và giới hạn tài nguyên

> **Điểm bắt đầu:** snapshot `03-storage-ready` — mốc do [Lab 6a](LAB-6A-PV-PVC-VA-STORAGECLASS.md)
> tạo, xem [chuỗi snapshot](README.md#3-chuỗi-snapshot). Repo `~/bookstore` ở tag `lab-7a` (hoặc
> bundle `~/bookstore-lab-7a.bundle` nếu bạn vừa restore snapshot).
> **Điểm kết thúc:** teardown trả cluster về đúng `03-storage-ready`, không tạo snapshot mới. Repo
> `~/bookstore` có tag `lab-7b` và một bản bundle nằm ngoài VM.
> **Sprint dự án:** Sprint 11 — Hạn mức namespace của [dự án Bookstore](DU-AN-BOOKSTORE.md).
> **Lab trước:** [Lab 7a — Lập lịch và eviction](LAB-7A-LAP-LICH-VA-EVICTION.md).
> **Lab sau:** [Lab 8a — Dựng cluster bằng kubeadm](LAB-8A-DUNG-CLUSTER-BANG-KUBEADM.md).
>
> **Cập nhật và đối chiếu phiên bản:** 13/10/2026.

Lab này đi cùng mục
[7b. Chính sách giới hạn tài nguyên](../00-ALO-TRINH-ADMIN.md#7b-chính-sách-giới-hạn-tài-nguyên)
của [Giai đoạn 7](../00-ALO-TRINH-ADMIN.md#giai-đoạn-7--lập-lịch-và-chính-sách-tài-nguyên) — sáu bài
[132](../132-policies-vi.md), [133](../133-limit-range-vi.md), [134](../134-resource-quotas-vi.md),
[135](../135-pid-limiting-vi.md), [74](../74-resource-managers-vi.md) và
[154](../154-node-declared-features-vi.md). Nhóm 7b **không có dòng "Thực hành"** trong lộ trình:
loạt trang task về quota và LimitRange nằm ở
[giai đoạn 25](../00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace), sau lab
này, nên các ticket bám thẳng vào sáu bài khái niệm.

Mọi ticket dựa vào hợp đồng kỹ thuật ở [mục 4 của file dự án](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật)
và bố cục repo ở [mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint). Kế hoạch gốc của sprint
nằm ở [mục 7.2](DU-AN-BOOKSTORE.md#sprint-11--lab-7b-hạn-mức-namespace); ngân sách xuất phát ở
[mục 3.6](DU-AN-BOOKSTORE.md#36-ngân-sách-tài-nguyên-và-lập-lịch).

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); image và hạ tầng
nền của dự án nằm ở [bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00). Lab
này không chép lại con số phiên bản nào trong phần văn bản và **không cài thêm gì** vào cluster.

**Sprint này nói về chính sách ở cấp namespace và cấp node.** `requests`/`limits` của từng
container, QoS và cgroup đã làm ở [Lab 3c](LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md). Ở đây câu hỏi
khác: ai đặt trần cho **cả namespace `bookstore`**, ai điền giá trị cho Pod **không khai gì**, và
những chính sách nào **không đi qua API server** mà nằm trên kubelet của từng node.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`. B0 bên dưới chạy lại đúng gate đó, thêm hai dòng riêng của lab này.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi namespace `bookstore` có hạn mức đúng với ngân sách thật của hệ thống —
LimitRange `bookstore-limits` và ResourceQuota `bookstore-quota` trong `k8s/app/policy/` — **và**
bạn chứng minh, giải thích được:

- Chính sách của Kubernetes nằm ở **hai mặt phẳng** (bài 132): object API trong namespace —
  NetworkPolicy của sprint 7 và hai object của sprint này — và cấu hình kubelet trên từng node, thứ
  không `kubectl get` được.
- Con số của quota không đoán: nó là tổng request/limit **hiệu dụng** của mọi Pod chưa kết thúc,
  cộng phần trội của rollout (`maxSurge` và Pod đang kết thúc), Job, CronJob, Pod tạm của người vận
  hành, và chỗ cho HPA đưa `web` lên 5 bản sao ở Lab 11b.
- LimitRange là trần và mặc định cho **từng container**: nó điền giá trị ở admission, trước khi
  quota soi; nó không kiểm giá trị mặc định có khớp `requests` của client; nó không đụng Pod đang
  chạy — nên một lần siết sai chỉ phát nổ ở lần thay Pod kế tiếp.
- ResourceQuota là trần **tổng** của namespace, độc lập với dung lượng cluster, không ràng buộc
  node. Ba kiểu thông báo từ chối — *thiếu khai báo*, *vượt trần*, *vi phạm LimitRange* — mỗi kiểu
  chỉ tới một chỗ phải sửa khác nhau.
- Quota theo **số lượng object** (`services.nodeports`, `persistentvolumeclaims`,
  `count/jobs.batch`) bảo vệ quyết định kiến trúc và bộ lưu trữ của control plane, thứ trần
  CPU/memory không chạm tới — và cái giá: đặt quá chặt thì `db-backup` im lặng ngừng chạy.
- Bẫy Deployment: `kubectl` báo thành công, rollout kẹt vì quota không đủ chỗ cho `maxSurge`; đọc
  nguyên nhân ở sự kiện `FailedCreate` của ReplicaSet và condition `ReplicaFailure`; vì sao kẹt mà
  `web` vẫn phục vụ.
- Giới hạn PID không khai được trong Pod spec; cluster lab chưa bật giới hạn nào; đọc nó ở cgroup
  **cấp Pod** trên node chứ không ở trong container.
- Policy của CPU, memory và topology manager trên worker; vì sao `db-0` không có CPU độc quyền dưới
  policy hiện tại và kể cả khi đổi; vì sao topology manager không có gì để căn chỉnh trên VM một
  NUMA node; tính năng do node khai báo chưa có trên baseline.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 7b | Ticket kiểm chứng |
| --- | --- |
| [132 — Chính sách](../132-policies-vi.md) | B1 — kiểm kê chính sách đang ràng buộc `bookstore`: NetworkPolicy (5b) là object API phạm vi namespace, chưa có LimitRange hay ResourceQuota; B7, B8 — nhánh cấu hình kubelet: không có object API nào, chỉ đọc được trên node |
| [133 — Khoảng giới hạn tài nguyên](../133-limit-range-vi.md) | B2 — `bookstore-limits`: mặc định chèn vào Pod tạm kèm dấu vết, `max` và `min` từ chối ngay lúc tạo, bẫy `default` nhỏ hơn `requests` của client; B6 — gỡ LimitRange, siết `max`: Pod đang chạy không bị đụng, lần thay Pod kế tiếp bị chặn |
| [134 — Hạn ngạch tài nguyên](../134-resource-quotas-vi.md) | B1 — quota tạm làm máy đếm `used`; B3 — `bookstore-quota` trần tính toán, độc lập dung lượng, không ràng buộc node, bẫy Deployment khi scale nhầm; B4 — số lượng object và lưu trữ; B5 — rollout kẹt với `FailedCreate`; B6 — *thiếu khai báo* so với *vượt trần*, quota đếm đúng giá trị LimitRange chèn; B9 ⭐ — phạm vi `PriorityClass` |
| [135 — Giới hạn và dự trữ Process ID](../135-pid-limiting-vi.md) | B7 — `pids` không phải tài nguyên của Pod spec; `podPidsLimit` thật trên ba node; `pids.max` ở cgroup cấp Pod; số tiến trình của `db-0`; `PIDPressure` và `pid.available`; nợ kỹ thuật mới |
| [74 — Các trình quản lý tài nguyên](../74-resource-managers-vi.md) | B8 — policy thật của CPU/memory/topology manager; `db-0` Guaranteed nhưng CPU không nguyên; CFS quota; policy không nằm trong object Node; một NUMA node; ⭐ Pod CPU nguyên vẫn ở pool dùng chung |
| [154 — Tính năng do Node khai báo](../154-node-declared-features-vi.md) | B8 — bài được lộ trình đánh dấu **đọc lướt**; ticket kiểm đúng phần đọc được: `.status.declaredFeatures` chưa có trên Node của baseline |

Các phần dưới đây **không thực hành trong sprint này**, ghi rõ lý do thay vì bỏ trống:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| Bài 132: admission controller tích hợp, `ValidatingAdmissionPolicy`, admission webhook | Cần chuỗi authn → authz → admission và cờ của kube-apiserver — [giai đoạn 9](../00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), bài [119](../119-controlling-access-vi.md) và [173](../173-admission-webhooks-vi.md) |
| Bài 133: hai LimitRange trong một namespace | Bài nói kết quả **không xác định**; không viết được gate `PASS:` cho hành vi không xác định. B2 chỉ đòi **đúng một** LimitRange và hỏi vì sao |
| Bài 133: `maxLimitRequestRatio`, `min`/`max` cho `type: PersistentVolumeClaim` | Bài xếp vào *Đọc lướt*, hoãn tới nhóm task [giai đoạn 25](../00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace) |
| Bài 134: phạm vi `BestEffort`, `Terminating`, `CrossNamespacePodAffinity`, `VolumeAttributesClass`, và hai ví dụ cần cờ `--admission-control-config-file` | Bài xếp *Phạm vi hạn ngạch* vào *Đọc lướt*; sửa cờ kube-apiserver thuộc [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy). Riêng phạm vi `PriorityClass` cơ bản — không cần cờ, và bài 141 hẹn tới nhóm 7b — làm ở B9 ⭐ |
| Bài 134: quota cho tài nguyên mở rộng, DRA resource claim, `hugepages-<size>`, lưu trữ tạm thời | DRA thuộc [giai đoạn 13](../00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao); device plugin thuộc giai đoạn 14; hai worker VM không cấu hình hugepages nên `hugepages-2Mi` của node bằng 0, không có gì để đo; quota lưu trữ tạm thời còn alpha |
| Bài 134: `count/<resource>.<group>` cho custom resource | Cần CRD — [giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng), Lab 14 thêm `DbBackup`. B4 dùng cú pháp `count/` cho tài nguyên chuẩn |
| Bài 135: đặt `podPidsLimit`, dự trữ `pid=<number>` trong `--system-reserved`/`--kube-reserved` | Phải sửa cấu hình kubelet và restart kubelet trên từng node — sprint này **cấm sửa node**; việc đó thuộc [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy), bài [224](../224-kubelet-config-file-vi.md) và [253](../253-reserve-compute-resources-vi.md). B7 chỉ đọc và ghi thành nợ kỹ thuật |
| Bài 135: chạy fork bomb | Cluster chưa bật giới hạn PID nào (B7 chứng minh), nên fork bomb chỉ làm mất ổn định `lab-k8s-worker2` — nơi có `db-0` — mà không chứng minh được gì; `pid.available` lại **không cưỡng chế** giới hạn |
| Bài 74: đổi CPU manager sang `static` | Đòi sửa cấu hình kubelet, restart kubelet và xóa state file. Worker là VM 2 vCPU một NUMA node. Đọc tiếp ở giai đoạn 20 và giai đoạn 25, bài [200](../200-cpu-management-policies-vi.md), [235](../235-memory-manager-vi.md), [259](../259-topology-manager-vi.md) |
| Bài 74: tùy chọn của chính sách static, trình quản lý tài nguyên cấp Pod, Device Manager | Tùy chọn static là tinh chỉnh theo NUMA/socket/uncore cache của phần cứng thật; phần cấp Pod là alpha ở bản sau baseline; Device Manager cần device plugin — giai đoạn 14, bài [184](../184-device-plugins-vi.md) |
| Bài 154: plugin `NodeDeclaredFeatures` và admission controller `NodeDeclaredFeatureValidator` | Tính năng ra sau baseline và cần feature gate trên `kube-apiserver`, `kube-scheduler`, `kubelet` — [giai đoạn 8](../00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm) bài [03](../03-control-plane-flags-vi.md), giai đoạn 20 bài [196](../196-configure-feature-gates-vi.md) |

> Trang [313 — Khắc phục sự cố Topology Management](../313-debug-topology-vi.md) **không thuộc
> nhóm 7b** và chỉ dùng được khi Topology Manager chạy policy khác `none`.

Sprint này **không phát sinh và không trả nợ lab** nào. Nợ lab #1 (HPA, VPA) vẫn mở; sprint này chỉ
**giữ chỗ** cho HPA của Lab 11b trong quota. Nợ kỹ thuật: **phát sinh một nợ mới** về giới hạn PID
của node (B7, mã `TD-13` theo sổ nợ của file dự án); không
trả nợ kỹ thuật nào. Xem [sổ nợ lab](README.md#5-sổ-nợ-lab) và
[sổ nợ kỹ thuật của file dự án](DU-AN-BOOKSTORE.md#8-sổ-nợ-kỹ-thuật-dự-án).

### 1.2. Thời lượng

4–5 giờ. Phần cốt lõi (B0–B8, B10) khoảng 4 giờ; B9 và các mục ⭐ trong B4, B8 là tùy chọn. Phần
lớn thời gian nằm ở việc đọc kỹ từng thông báo từ chối và tính ngân sách. Có thể chia hai phiên:
phiên 1 làm B0–B4, phiên 2 làm B5–B10. Giữa hai phiên cứ để nguyên cluster; đầu phiên 2 chạy lại
gate ngắn A5.5 và `./scripts/deploy.sh`.

---

## 2. Quy ước và an toàn

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` trừ khi ghi rõ node khác. Lệnh đọc file trên
  worker chạy qua `ssh <node> 'sudo ...'`.
- **Fault injection chỉ trên `lab-k8s-worker2`.** Diễn tập B6 xóa `db-0` — Pod nằm trên node này.
  Pod thử nghiệm của diễn tập ghim `lab-k8s-worker2`. Diễn tập B5 siết **chính sách của namespace**,
  không tiêm lỗi vào node nào; Pod mới do controller của `web` tạo theo lịch bình thường.
- Repo dự án: `~/bookstore`. Manifest và script thử nghiệm: `~/lab-work/7b/`. Bằng chứng:
  `~/lab-evidence/7b/`. Object thử nghiệm trong `bookstore` **không** mang label
  `app.kubernetes.io/part-of`; Job và PVC thử nghiệm mang label `lab=...` để dọn theo nhóm.
- **Sprint này chỉ ĐỌC cấu hình node.** Không sửa `/var/lib/kubelet/config.yaml`, không restart
  kubelet, không xóa state file trong `/var/lib/kubelet/`. B0 ghi checksum cấu hình kubelet của cả ba
  node; gate cuối so lại.
- Lab chỉ dùng kiến thức nhóm 7b và các nhóm trước: Pod, probe, init container, sidecar (3a);
  ConfigMap, Secret, Kustomize (3b, 5a); `requests`/`limits`, QoS, PDB (3c); Deployment, rollout
  (4a); StatefulSet, Job, CronJob, DaemonSet (4b); Service, DNS, NetworkPolicy, Ingress (5a, 5b);
  PVC, StorageClass (6a); affinity, taint, PriorityClass (7a). **Không** dùng RBAC (9a), admission
  webhook hay `ValidatingAdmissionPolicy` (9), HPA hay metrics-server (11), `kubectl top`.
- Lệnh imperative (`kubectl patch`, `scale`, `rollout restart`, `delete limitrange`) chỉ dùng trong
  script nghiệm thu và diễn tập; script luôn trả hệ thống về đúng repo trước khi kết thúc.
- Từ B2, mọi Pod tạo trong `bookstore` đi qua LimitRange; từ B3, qua cả quota. Pod tạm của bạn —
  `kubectl run` để thử một lệnh — nhận giá trị mặc định của LimitRange và chịu NetworkPolicy chặn
  mặc định của sprint 7.
- Pod `busybox` chạy `sleep` làm tiến trình 1 bỏ qua `SIGTERM`. Pod thử nghiệm đặt
  `terminationGracePeriodSeconds: 1` để xóa nhanh; **không** áp điều này cho Pod của dự án.
- Namespace thử nghiệm duy nhất là `bookstore-thu` (⭐ B8), mang label của dự án để teardown dọn
  được nếu bạn quên. LimitRange và ResourceQuota là object phạm vi namespace: sprint này không thêm
  object cấp cluster nào vào repo.
- Mỗi script nghiệm thu chạy trong một shell liền mạch và nạp hàm dùng chung ở
  `~/lab-work/7b/lib.sh` (tạo ở B0).
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail. Dòng `FAIL:` trong script nghiệm
  thu nghĩa là ticket chưa xong.

**Cách quay lui khi hỏng:** tắt **cả ba VM**, restore **cả ba** về `03-storage-ready`, khôi phục
repo từ bundle `lab-7a` theo [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm),
rồi làm lại từ B0. Không bao giờ restore riêng một VM — xem
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).

---

# Phần B — Sprint 11: Hạn mức namespace

## B0. Nhận bàn giao sprint

**Mục đích:** xác nhận cluster ở đúng `03-storage-ready` và chưa có chính sách tài nguyên nào, repo ở
đúng tag `lab-7a`, hệ thống của sprint 10 dựng lại được **chỉ từ repo**, và ghi lại cấu hình kubelet
để cuối sprint chứng minh bạn không sửa gì trên node.

Gate ngắn A5.5, chạy nguyên văn, cộng hai dòng riêng của sprint này:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default

# Riêng Lab 7b: chưa namespace nào có chính sách tài nguyên; label dữ liệu của 7a đã được gỡ.
kubectl get resourcequota,limitrange -A
kubectl get nodes -l bookstore.example.com/data
```

Repo và thư mục làm việc. Nếu vừa restore snapshot, repo đã mất — khôi phục từ bundle theo
[mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm) trước:

```bash
mkdir -p ~/lab-work/7b ~/lab-evidence/7b
test -d ~/bookstore/.git || git clone ~/bookstore-lab-7a.bundle ~/bookstore
cd ~/bookstore
git describe --tags --exact-match HEAD 2>/dev/null | grep -qx 'lab-7a' && echo 'PASS: HEAD dung tag lab-7a'
test -z "$(git status --porcelain)" && echo 'PASS: repo khong co thay doi chua commit'
kubectl get namespace bookstore >/dev/null 2>&1 \
  && echo 'FAIL: namespace bookstore con sot tu lan truoc' || echo 'PASS: cluster chua co du an'
```

Dựng lại hệ thống của sprint 10 bằng script của chính sprint đó:

```bash
cd ~/bookstore
./scripts/deploy.sh > ~/lab-evidence/7b/b0-deploy.txt 2>&1; echo "deploy exit=$?"
./tests/smoke.sh > ~/lab-evidence/7b/b0-smoke-lab-7a.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/7b/b0-smoke-lab-7a.txt
```

Hàm dùng chung cho mọi script nghiệm thu của sprint — chép nguyên văn. Đây là công cụ, không phải
nội dung học:

```bash
cat > ~/lab-work/7b/lib.sh <<'EOF'
# Hàm dùng chung cho script nghiệm thu Lab 7b. Nạp bằng: . ~/lab-work/7b/lib.sh
NS=bookstore; EV=~/lab-evidence/7b; W=~/lab-work/7b
SEL='app.kubernetes.io/part-of=bookstore'
k()  { kubectl -n "$NS" "$@"; }
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
# Quantity -> số nguyên để so sánh: cpu_m ra millicore, mem_mi ra MiB.
cpu_m() { awk -v q="$1" 'BEGIN { if (q == "") { print 0; exit }
  if (q ~ /m$/) { sub(/m$/, "", q); printf "%d\n", q; exit }
  printf "%d\n", q * 1000 }'; }
mem_mi() { awk -v q="$1" 'BEGIN { if (q == "") { print 0; exit }
  n = q; u = q; sub(/[A-Za-z]+$/, "", n); sub(/^[0-9.]+/, "", u)
  f["Ki"] = 1/1024; f["Mi"] = 1; f["Gi"] = 1024; f["Ti"] = 1048576; f[""] = 1/1048576
  f["k"] = 1000/1048576; f["M"] = 1000000/1048576; f["G"] = 1000000000/1048576
  printf "%d\n", n * f[u] }'; }
norm() { case "$1" in *cpu) cpu_m "$2" ;; *memory|*storage) mem_mi "$2" ;; *) echo "${2:-0}" ;; esac; }
# qv <quota> hard|used <tài nguyên>: một giá trị trong status của ResourceQuota (rỗng nếu chưa có)
qv() { kubectl -n "$NS" get resourcequota "$1" \
  -o go-template="{{with .status.$2}}{{with index . \"$3\"}}{{.}}{{end}}{{end}}" 2>/dev/null; }
# res_lines <đường dẫn tới PodSpec>: "tên=reqCPU,reqMem,limCPU,limMem" cho mọi container, cả init
res_lines() {
  local f='{.name}={.resources.requests.cpu},{.resources.requests.memory},{.resources.limits.cpu},{.resources.limits.memory}{"\n"}'
  printf '{range %s.initContainers[*]}%s{end}{range %s.containers[*]}%s{end}' "$1" "$f" "$1" "$f"; }
# lr <đường dẫn>: một giá trị của mục type Container trong LimitRange bookstore-limits
lr() { jp limitrange/bookstore-limits "{.spec.limits[?(@.type==\"Container\")].$1}"; }
# kz <file configz> <khóa>: một khóa trong cấu hình hiệu lực của kubelet
kz() { python3 -c 'import json, sys
c = json.load(open(sys.argv[1]))["kubeletconfig"]; v = c.get(sys.argv[2])
print(json.dumps(v) if isinstance(v, (dict, list)) else v)' "$1" "$2"; }
# podcg <node> <uid Pod> <file>: đọc một file cgroup CẤP POD trên node — chỉ đọc
podcg() { local u="${2//-/_}" c
  c="d=\$(find /sys/fs/cgroup -maxdepth 4 -type d \\( -name '*pod${u}.slice' -o -name 'pod$2' \\) | head -n 1); cat \"\$d/$3\""
  if [ "$1" = lab-k8s-master ]; then sh -c "$c"; else ssh "$1" "$c"; fi; }
# ev <kind> <tên> <reason>: message của các sự kiện tương ứng trong namespace
ev() { kubectl -n "$NS" get events \
  --field-selector "involvedObject.kind=$1,involvedObject.name=$2,reason=$3" \
  -o jsonpath='{range .items[*]}{.message}{"\n"}{end}' 2>/dev/null; }
EOF
```

Ảnh chụp "trước" của kubelet. Cấu hình **hiệu lực** — gồm cả giá trị mặc định mà file không viết
ra — đọc qua endpoint `configz`, đúng đường control plane → kubelet mà
[tầng 6 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a547-tầng-6--đường-control-plane--kubelet) đã kiểm. B7
và B8 đọc lại các file này:

```bash
. ~/lab-work/7b/lib.sh
OKZ=0
for n in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  kubectl get --raw "/api/v1/nodes/$n/proxy/configz" > "$EV/b0-configz-$n.json"
  kz "$EV/b0-configz-$n.json" cgroupDriver >/dev/null 2>&1 && OKZ=$((OKZ + 1))
done
test "$OKZ" -eq 3 && echo 'PASS: doc duoc cau hinh hieu luc cua ca ba kubelet'

{ echo "lab-k8s-master $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for n in lab-k8s-worker1 lab-k8s-worker2; do
    echo "$n $(ssh "$n" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done; } > "$EV/b0-kubelet-sha.txt"
test "$(grep -c ' [0-9a-f]\{64\}$' "$EV/b0-kubelet-sha.txt")" -eq 3 \
  && echo 'PASS: ghi checksum cau hinh kubelet cua ca ba node'
```

**PASS:** gate A5.5 đạt như mô tả ở Lab 00; hai lệnh riêng của 7b đều trả `No resources found`; ba
dòng `PASS:` về repo và cluster; `deploy exit=0`, `smoke exit=0` và file smoke không có dòng
`FAIL:`; hai dòng `PASS:` về kubelet.

`deploy.sh` hoặc `smoke.sh` fail nghĩa là repo của sprint 10 **chưa đủ** — có thứ chỉ tồn tại trên
cluster cũ. Sửa repo ở tag `lab-7a` trước, không sửa tay trên cluster
([mục 6.1 của file dự án](DU-AN-BOOKSTORE.md#61-mở-đầu)). `configz` không đọc được thì xem
[mục 4](#4-troubleshooting-của-lab-này) trước khi đi tiếp — B7 và B8 dựa vào nó.

`scripts/deploy.sh` và `scripts/teardown.sh` **giữ nguyên** bản của Lab 7a trong cả sprint. Hai
object mới nằm trong namespace `bookstore`, đi cùng lần render Kustomize hiện có, và biến mất cùng
namespace khi teardown.

## B1. [BS-7B-01] Kiểm kê: hệ thống thật cần bao nhiêu

**Loại:** Điều tra, kèm tính năng.

**Bối cảnh:** yêu cầu `R11` của dự án: namespace có hạn mức, không workload nào ăn hết cluster.
Đặt trần theo cảm giác thì hoặc chặn chính hệ thống ở lần phát hành tới, hoặc rộng tới mức vô
nghĩa. Trước khi viết một dòng chính sách, đội phải biết hệ thống **thật** đang giữ bao nhiêu và lúc
đông nhất cần bao nhiêu. [Ngân sách mục 3.6](DU-AN-BOOKSTORE.md#36-ngân-sách-tài-nguyên-và-lập-lịch)
là điểm xuất phát; con số đã chốt ở Lab 3c và trạng thái sau Lab 7a mới là sự thật.

**Yêu cầu:**

1. Liệt kê mọi chính sách đang ràng buộc namespace `bookstore` hôm nay, chia theo hai mặt phẳng
   của bài 132: object API trong namespace, và cấu hình trên kubelet. Ghi vào ADR.
2. **Mọi** container của dự án — kể cả init container, sidecar, container của Job, CronJob,
   DaemonSet — khai **tường minh** request và limit CPU, memory trong repo. Container nào còn thiếu
   thì bổ sung theo ngân sách rồi triển khai lại. Sprint này sắp đặt một trần mà Pod không khai sẽ
   bị từ chối; hệ thống không được phụ thuộc vào giá trị mặc định của ai.
3. Đo tổng thật ở trạng thái thường bằng **chính quota controller**: một ResourceQuota **tạm** có
   trần rất lớn trong `bookstore`; đọc `used` của năm tài nguyên `requests.cpu`, `requests.memory`,
   `limits.cpu`, `limits.memory`, `pods`; rồi xóa nó. Lúc đo không có Pod Job nào đang chạy.
4. Tính **đỉnh** cho từng thành phần: `web` 5 bản sao (HPA ở Lab 11b) cộng phần trội rollout; `api`
   cộng phần trội rollout; Pod đang kết thúc trong rollout; `db` (StatefulSet thay Pod thế nào?);
   `node-agent` trên mọi node; một Pod `db-migrate`; hai Pod backup (theo lịch và chạy tay); một Pod
   tạm của người vận hành.
5. `docs/ngan-sach-namespace.md`: một bảng theo thành phần — request và limit hiệu dụng của một
   Pod, số Pod thường, số Pod đỉnh, lý do — và ở cuối file một khối số liệu **đúng định dạng** dưới
   đây. Script nghiệm thu của B1 và B3 đọc khối này:

```text
thuong.requests.cpu=<giá trị>
thuong.requests.memory=<giá trị>
thuong.limits.cpu=<giá trị>
thuong.limits.memory=<giá trị>
thuong.pods=<giá trị>
dinh.requests.cpu=<giá trị>
dinh.requests.memory=<giá trị>
dinh.limits.cpu=<giá trị>
dinh.limits.memory=<giá trị>
dinh.pods=<giá trị>
```

**Ràng buộc:** chưa tạo LimitRange hay ResourceQuota cố định ở ticket này. Quota tạm ở
`~/lab-work/7b/`, không mang label dự án, xóa ngay khi đọc xong. Không đổi `replicas` của thành phần
nào trong repo; không đổi chiến lược rollout.

**Đọc lại:**

- Bài [132](../132-policies-vi.md):
  [*Áp dụng chính sách bằng các đối tượng API*](../132-policies-vi.md#áp-dụng-chính-sách-bằng-các-đối-tượng-api-apply-policies-using-api-objects),
  [*Áp dụng chính sách bằng cấu hình Kubelet*](../132-policies-vi.md#áp-dụng-chính-sách-bằng-cấu-hình-kubelet-apply-policies-using-kubelet-configurations).
- Bài [134](../134-resource-quotas-vi.md):
  [*Cách ResourceQuota trong Kubernetes hoạt động*](../134-resource-quotas-vi.md#cách-resourcequota-trong-kubernetes-hoạt-động-how-kubernetes-resourcequotas-work),
  bảng [*Hạn ngạch cho tài nguyên hạ tầng*](../134-resource-quotas-vi.md#compute-resource-quota) —
  chú ý cụm "các pod ở trạng thái chưa kết thúc".
- Bài [51](../51-sidecar-containers-vi.md#chia-sẻ-tài-nguyên-giữa-các-container-resource-sharing-within-containers)
  — request hiệu dụng khi có sidecar, và câu cuối mục về quota; bài
  [50](../50-init-containers-vi.md#resource-sharing-within-containers).
- Bài [63](../63-deployment-vi.md): [*Max Surge*](../63-deployment-vi.md#max-surge),
  [*Max Unavailable*](../63-deployment-vi.md#max-unavailable),
  [*Pod đang kết thúc*](../63-deployment-vi.md#terminating-pods).
- Trong repo: `docs/hpa-web.draft.yaml` (Lab 4b) — số bản sao tối đa của `web`.

**Bàn giao:** manifest đã bổ sung tài nguyên (nếu có); `docs/ngan-sach-namespace.md`;
`docs/adr/NNNN-ngan-sach-namespace.md` (`NNNN` là số tiếp theo trong `docs/adr/`);
`~/lab-evidence/7b/b1-*.txt`.

**Nghiệm thu:**

- `PASS:` mọi container — cả init và sidecar — của mọi Pod dự án và của template CronJob `db-backup`
  có đủ bốn giá trị request/limit CPU, memory.
- `PASS:` quota tạm đo được `used` của năm tài nguyên, và đã bị xóa.
- `PASS:` khối số liệu đủ mười khóa; `thuong.*` khớp số đo; mọi `dinh.*` lớn hơn hoặc bằng `thuong.*`
  tương ứng; `dinh.pods` lớn hơn `thuong.pods` ít nhất 4.
- `PASS:` có ADR ngân sách.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-ngan-sach-namespace.md`:

1. Trước sprint này, `bookstore` đã chịu những chính sách nào ở mỗi mặt phẳng của bài 132? Vì sao
   `kubectl get` không cho bạn biết một worker có đang giới hạn PID hay không?
2. Viết công thức request hiệu dụng của Pod `web` (init `render-html`, sidecar `status-writer`,
   container `nginx`). Nếu `render-html` không khai gì và sau này nhận 500m CPU từ một giá trị mặc
   định, Pod `web` giữ bao nhiêu CPU — và quota đếm bao nhiêu?
3. Vì sao đỉnh phải tính cả Pod **đang kết thúc**? Dẫn câu của bài 63 và cột mô tả trong bảng của
   bài 134.
4. `db` là StatefulSet một bản sao. Khi rollout, có lúc nào hai Pod `db` cùng bị đếm không? So với
   `web`.

<details>
<summary>Gợi ý</summary>

1. Object chính sách phạm vi namespace:
   `kubectl api-resources --namespaced=true | grep -E '^(limitranges|resourcequotas|networkpolicies) '`,
   rồi `kubectl -n bookstore get networkpolicy,limitrange,resourcequota`.
2. Tài nguyên của mọi container, cả init: hàm `res_lines` trong `lib.sh` sinh sẵn biểu thức
   jsonpath — `kubectl -n bookstore get pods -o jsonpath="{range .items[*]}$(res_lines .spec){end}"`.
   Ô trống là container chưa khai.
3. Quota tạm: một ResourceQuota với `hard` cỡ 1000 CPU, 1000Gi, 1000 Pod — đủ lớn để không chặn
   gì, chỉ để đếm. Đọc bằng `kubectl -n bookstore describe quota <tên>`.
4. Chiến lược thật: `kubectl -n bookstore get deploy web -o jsonpath='{.spec.strategy}'`. Giá trị
   phần trăm thì tính trên 5 bản sao: `maxSurge` làm tròn lên, `maxUnavailable` làm tròn xuống.
5. Pod đang kết thúc vẫn ở phase `Running` cho tới khi container dừng; bảng của bài 134 chỉ loại
   Pod ở trạng thái kết thúc, tức phase `Succeeded` hoặc `Failed`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
. ~/lab-work/7b/lib.sh
cd ~/bookstore
dv() { sed -n "s/^$1=//p" docs/ngan-sach-namespace.md | head -n 1 | tr -d ' \r'; }

# 1. Mọi container khai tường minh — Pod đang có và template CronJob
{ k get pods -l "$SEL" -o jsonpath="{range .items[*]}$(res_lines .spec){end}"
  k get cronjob db-backup -o jsonpath="$(res_lines .spec.jobTemplate.spec.template.spec)"
} | sort -u > "$EV/b1-container.txt"
cat "$EV/b1-container.txt"
MISS="$(grep -E '=,|,,|,$' "$EV/b1-container.txt")"
{ [ -s "$EV/b1-container.txt" ] && [ -z "$MISS" ]; } \
  && echo 'PASS: moi container, ca init va sidecar, khai du request va limit' \
  || echo "FAIL: container thieu tai nguyen: $(echo $MISS)"

# 2. Đo lại bằng quota tạm, khi không có Pod Job nào đang chạy
for i in $(seq 1 60); do
  k get pods -l "$SEL" --field-selector=status.phase=Running \
    -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].kind}{"\n"}{end}' | grep -qx Job || break
  sleep 5
done
k delete resourcequota kiem-ke --ignore-not-found >/dev/null
cat <<'Y' | kubectl apply -f - >/dev/null
apiVersion: v1
kind: ResourceQuota
metadata: {name: kiem-ke, namespace: bookstore}
spec:
  hard: {requests.cpu: "1000", requests.memory: 1000Gi, limits.cpu: "1000", limits.memory: 1000Gi, pods: "1000"}
Y
for i in $(seq 1 30); do [ -n "$(qv kiem-ke used pods)" ] && break; sleep 2; done
: > "$EV/b1-do-quota.txt"
for K in requests.cpu requests.memory limits.cpu limits.memory pods; do
  echo "thuong.$K=$(qv kiem-ke used "$K")" >> "$EV/b1-do-quota.txt"
done
k delete resourcequota kiem-ke >/dev/null
cat "$EV/b1-do-quota.txt"
{ [ "$(grep -c '=.\+$' "$EV/b1-do-quota.txt")" -eq 5 ] && ! k get resourcequota kiem-ke >/dev/null 2>&1; } \
  && echo 'PASS: quota tam do duoc used cua nam tai nguyen va da bi xoa'

# 3. Khối số liệu của ngân sách
BAD=0
for K in requests.cpu requests.memory limits.cpu limits.memory pods; do
  T="$(dv "thuong.$K")"; D="$(dv "dinh.$K")"; M="$(sed -n "s/^thuong\.$K=//p" "$EV/b1-do-quota.txt")"
  { [ -n "$T" ] && [ -n "$D" ]; } || { echo "FAIL: thieu thuong.$K hoac dinh.$K"; BAD=1; continue; }
  [ "$(norm "$K" "$T")" -eq "$(norm "$K" "$M")" ] || { echo "FAIL: thuong.$K=$T, do duoc $M"; BAD=1; }
  [ "$(norm "$K" "$D")" -ge "$(norm "$K" "$T")" ] || { echo "FAIL: dinh.$K=$D nho hon thuong"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: thuong.* khop so do, dinh.* khong nho hon thuong.*'
[ "$(dv dinh.pods)" -ge $(( $(dv thuong.pods) + 4 )) ] 2>/dev/null \
  && echo 'PASS: dinh.pods chua it nhat 4 Pod troi (HPA web, surge web, surge api)'
ls docs/adr/*-ngan-sach-namespace.md >/dev/null 2>&1 && echo 'PASS: co ADR ngan sach'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Nếu init container `render-html` của `web` (và `wait-for-db` của `api`, nếu còn) chưa khai tài
nguyên, thêm vào đúng init container đó trong manifest ở tag `lab-7a` của bạn:

```yaml
        resources:
          requests: {cpu: 10m, memory: 16Mi}
          limits: {cpu: 50m, memory: 32Mi}
```

Rồi `./scripts/deploy.sh`. Quota tạm dùng để đo:

```yaml
# ~/lab-work/7b/kiem-ke.yaml — chỉ để đếm, xóa ngay sau khi đọc
apiVersion: v1
kind: ResourceQuota
metadata:
  name: kiem-ke
  namespace: bookstore
spec:
  hard:
    requests.cpu: "1000"
    requests.memory: 1000Gi
    limits.cpu: "1000"
    limits.memory: 1000Gi
    pods: "1000"
```

`docs/ngan-sach-namespace.md` theo ngân sách mục 3.6, chiến lược `web` `maxSurge: 1`,
`maxUnavailable: 0`, `api` theo mặc định 25% — **thay bằng con số và chiến lược thật của bạn**:

````markdown
# Ngân sách tài nguyên của namespace bookstore

Nguồn: manifest ở tag lab-7a; số thường đo bằng quota tạm `kiem-ke` (~/lab-evidence/7b/b1-do-quota.txt).

| Thành phần | Request hiệu dụng / Pod | Limit hiệu dụng / Pod | Pod thường | Pod đỉnh | Lý do đỉnh |
| --- | --- | --- | --- | --- | --- |
| web | 60m, 40Mi | 250m, 80Mi | 3 | 7 | HPA 5 + maxSurge 1 + 1 Pod đang kết thúc |
| api | 50m, 64Mi | 500m, 256Mi | 2 | 4 | 2 + maxSurge 1 + 1 Pod đang kết thúc |
| db | 500m, 512Mi | 500m, 512Mi | 1 | 1 | StatefulSet xóa xong Pod cũ mới tạo Pod mới |
| node-agent | 10m, 16Mi | 50m, 32Mi | 3 | 3 | một Pod mỗi node, rollout không trội |
| db-migrate | 50m, 64Mi | 200m, 128Mi | 0 | 1 | chạy ở mỗi lần deploy, xong là Succeeded |
| db-backup | 50m, 64Mi | 200m, 128Mi | 0 | 2 | một lần theo lịch + một lần chạy tay (smoke) |
| Pod tạm | 10m, 16Mi | 100m, 64Mi | 0 | 1 | giá trị mặc định của LimitRange (B2) |

Request hiệu dụng của web = max(render-html, nginx + status-writer) = max(10m, 50m + 10m) = 60m.

```text
thuong.requests.cpu=810m
thuong.requests.memory=808Mi
thuong.limits.cpu=2400m
thuong.limits.memory=1360Mi
thuong.pods=9
dinh.requests.cpu=1310m
dinh.requests.memory=1304Mi
dinh.limits.cpu=5100m
dinh.limits.memory=2640Mi
dinh.pods=19
```
````

Ý chính cho ADR:

1. Mặt phẳng API: NetworkPolicy chặn mặc định và các luật mở của Lab 5b — object phạm vi namespace,
   `kubectl get networkpolicy` thấy được. Chưa có LimitRange, ResourceQuota. Mặt phẳng kubelet: giới
   hạn PID, policy của resource manager, ngưỡng eviction — nằm trong file và cấu hình hiệu lực của
   kubelet trên từng node; bài 132 xếp chúng vào *cấu hình Kubelet trên mỗi worker node*, không có
   object API nào để `get`.
2. Request hiệu dụng = max(request init lớn nhất, tổng container chính + sidecar) =
   max(render-html, nginx + status-writer). Nếu `render-html` nhận 500m thì Pod `web` giữ 500m thay
   vì 60m, và bài 51 nói quota áp theo request **hiệu dụng** — quota đếm 500m cho mỗi bản sao `web`.
   Đó là lý do mọi init container phải khai nhỏ và tường minh.
3. Bài 63 mục *Pod đang kết thúc*: Pod bị xóa hay bị scale down có thể mất thời gian để chấm dứt,
   nên tổng số Pod tạm thời vượt `.spec.replicas`. Bảng của bài 134 tính "trên tất cả các pod ở
   trạng thái chưa kết thúc" — Pod đang kết thúc vẫn `Running`, vẫn bị đếm.
4. Không. StatefulSet chỉ tạo `db-0` mới sau khi `db-0` cũ đã biến mất — đúng ngữ nghĩa "nhiều nhất
   một" của Lab 4b. `web` thì Deployment tạo Pod mới song song với Pod cũ.

</details>

## B2. [BS-7B-02] LimitRange `bookstore-limits`

**Loại:** Tính năng.

**Bối cảnh:** ticket sau đặt quota cho `cpu` và `memory`; từ lúc đó mọi Pod mới không khai tài
nguyên bị từ chối. Hệ thống đã khai đủ (B1), nhưng người vận hành vẫn tạo Pod tạm trong `bookstore`
— `kubectl run` để gọi thử một Service, kiểm một tên DNS. Namespace cần giá trị mặc định cho những
Pod đó, và một trần cho **từng container** để một manifest gõ nhầm — `cpu: 20` thay vì `200m` — không
lọt.

**Yêu cầu:**

1. Đúng **một** LimitRange tên `bookstore-limits` trong `bookstore`, label theo hợp đồng
   ([mục 4.1](DU-AN-BOOKSTORE.md#41-tên-và-label)), ở `k8s/app/policy/limitrange.yaml`, thêm vào
   `k8s/app/kustomization.yaml`, triển khai qua `deploy.sh`. Manifest là manifest thuần có
   `namespace` — B6 cần apply riêng nó được.
2. Ràng buộc theo **từng container**, cho cả CPU và memory:
   - Giá trị mặc định cho container không khai: cỡ một container phụ trợ busybox — request không quá
     50m CPU, 64Mi; limit không quá 200m, 128Mi.
   - Trần mỗi container: không nhỏ hơn limit của container lớn nhất trong hệ thống (B1), không lớn
     hơn nửa allocatable của một worker.
   - Sàn mỗi container: không lớn hơn request của container nhỏ nhất trong hệ thống.
3. Sau khi triển khai lại từ sạch, **không** Pod nào của dự án nhận giá trị mặc định — LimitRange
   chỉ là lưới an toàn cho Pod ngoài repo.
4. Một Pod tạm không khai tài nguyên được tạo, mang đúng giá trị mặc định, và có dấu vết cho biết ai
   đã điền.
5. Một container xin vượt trần, và một container xin dưới sàn, bị từ chối ngay lúc tạo; mỗi thông
   báo chỉ đúng ràng buộc của nó.
6. Dựng lại cái bẫy của bài 133 bằng Pod tạm: khai `requests` CPU lớn hơn limit mặc định, không khai
   `limits`. Ghi thông báo; chứng minh nó **không** phải `Forbidden`; sửa từ phía Pod.

**Ràng buộc:** chưa tạo ResourceQuota ở ticket này. Pod thử nghiệm ghim `lab-k8s-worker2`, không
mang label dự án, xóa khi xong. Không đặt `maxLimitRequestRatio`, không ràng buộc PVC (đọc lướt).

**Đọc lại:**

- Bài [133](../133-limit-range-vi.md): đoạn mở đầu (bốn khả năng của LimitRange);
  [*Ràng buộc đối với limit và request tài nguyên*](../133-limit-range-vi.md#constraints-on-resource-limits-and-requests)
  — trình tự "trước hết điền mặc định, sau đó soi min/max", và câu về hai LimitRange;
  [*LimitRange và kiểm tra admission cho Pod*](../133-limit-range-vi.md#limitrange-và-kiểm-tra-admission-cho-pod-limitrange-and-admission-checks-for-pods).
- Bài [132](../132-policies-vi.md#áp-dụng-chính-sách-bằng-các-đối-tượng-api-apply-policies-using-api-objects)
  — LimitRange "quản lý các ràng buộc cấp phát tài nguyên trên nhiều loại đối tượng".
- [A1.2 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a12-ba-vm) — cỡ của một worker.

**Bàn giao:** `k8s/app/policy/limitrange.yaml`; `k8s/app/kustomization.yaml`;
`docs/adr/NNNN-limitrange.md`; `~/lab-evidence/7b/b2-*.txt`.

**Nghiệm thu:**

- `PASS:` `bookstore` có đúng một LimitRange, tên `bookstore-limits`, mang label dự án, có mục
  `type: Container` với `default`, `defaultRequest`, `max`, `min` cho cả `cpu` và `memory`.
- `PASS:` giá trị mặc định trong giới hạn yêu cầu; `max` ≥ container lớn nhất và ≤ nửa worker;
  `min` ≤ container nhỏ nhất.
- `PASS:` teardown rồi deploy thành công; không Pod dự án nào mang annotation
  `kubernetes.io/limit-ranger`; smoke của Lab 7a vẫn PASS.
- `PASS:` Pod tạm Running với request và limit bằng giá trị mặc định, mang annotation
  `kubernetes.io/limit-ranger`.
- `PASS:` container vượt `max` bị `Forbidden` với `maximum cpu usage per Container`; container dưới
  `min` bị từ chối với `minimum cpu usage per Container`; không Pod nào được tạo.
- `PASS:` bẫy: thông báo `must be less than or equal to cpu limit`, không có `Forbidden`; khai cả
  `requests` lẫn `limits` thì Pod lên được.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-limitrange.md`:

1. Vì sao giá trị mặc định phải nhỏ? Nếu đặt 500m và một init container quên khai, chuyện gì xảy ra
   với request hiệu dụng của Pod đó và với quota ở B3?
2. `max` 1 CPU có ngăn `bookstore` chiếm hết cả cluster không? Dẫn đoạn mở đầu của bài 133.
3. Annotation `kubernetes.io/limit-ranger` cho người vận hành biết gì? Vì sao ta muốn **không** Pod
   nào của dự án mang nó?
4. Vì sao chỉ một LimitRange trong namespace? Nếu một sprint sau thêm LimitRange thứ hai vào
   `bookstore` thì Pod tạm nhận giá trị nào?
5. Pod debug xin `cpu: 150m` mà quên limit: vì sao lỗi không phải `Forbidden`, và sửa thế nào mà
   không đụng LimitRange?

<details>
<summary>Gợi ý</summary>

1. Khung: `spec.limits` là một danh sách; mỗi mục có `type` và các khối `default`, `defaultRequest`,
   `max`, `min`. `kubectl explain limitrange.spec.limits`.
2. API server kiểm tính hợp lệ của **chính** LimitRange: `min` ≤ `defaultRequest` ≤ `default` ≤
   `max`. Sai thứ tự thì `apply` bị từ chối.
3. Pod tạm ghim node: `kubectl run ... --overrides='{"apiVersion":"v1","spec":{"nodeSelector":{"kubernetes.io/hostname":"lab-k8s-worker2"}}}'`.
4. Dấu vết của LimitRange nằm trong `metadata.annotations` của Pod — đọc bằng
   `kubectl get pod <tên> -o yaml`.
5. Container lớn nhất và nhỏ nhất: file `~/lab-evidence/7b/b1-container.txt` của B1.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script teardown rồi deploy lại</summary>

```bash
. ~/lab-work/7b/lib.sh
cd ~/bookstore
./scripts/teardown.sh > "$EV/b2-teardown.txt" 2>&1
./scripts/deploy.sh > "$EV/b2-deploy.txt" 2>&1 && ./tests/smoke.sh > "$EV/b2-smoke.txt" 2>&1 \
  && echo 'PASS: teardown + deploy thanh cong, smoke cua Lab 7a van PASS' || echo 'FAIL: xem b2-deploy.txt, b2-smoke.txt'

# 1. Đúng một LimitRange, đủ bốn khối
k get limitrange bookstore-limits -o yaml > "$EV/b2-limitrange.yaml"
BAD=0
for F in default.cpu default.memory defaultRequest.cpu defaultRequest.memory max.cpu max.memory min.cpu min.memory; do
  [ -n "$(lr "$F")" ] || { echo "FAIL: thieu $F"; BAD=1; }
done
{ [ "$BAD" -eq 0 ] && [ "$(k get limitrange -o name)" = 'limitrange/bookstore-limits' ] && \
  [ "$(jp limitrange/bookstore-limits '{.metadata.labels.app\.kubernetes\.io/part-of}')" = bookstore ]; } \
  && echo 'PASS: dung mot LimitRange bookstore-limits, du default/defaultRequest/max/min cho cpu va memory'

# 2. Con số đúng yêu cầu
BIG_C=0; BIG_M=0; SML_C=999999; SML_M=999999
while IFS='=,' read -r n rc rm lc lm; do
  [ -z "$n" ] && continue
  c=$(cpu_m "$lc"); m=$(mem_mi "$lm"); [ "$c" -gt "$BIG_C" ] && BIG_C=$c; [ "$m" -gt "$BIG_M" ] && BIG_M=$m
  c=$(cpu_m "$rc"); m=$(mem_mi "$rm"); [ "$c" -lt "$SML_C" ] && SML_C=$c; [ "$m" -lt "$SML_M" ] && SML_M=$m
done < "$EV/b1-container.txt"
HALF_C=999999; HALF_M=999999
for n in lab-k8s-worker1 lab-k8s-worker2; do
  c=$(( $(cpu_m "$(kubectl get node "$n" -o jsonpath='{.status.allocatable.cpu}')") / 2 ))
  m=$(( $(mem_mi "$(kubectl get node "$n" -o jsonpath='{.status.allocatable.memory}')") / 2 ))
  [ "$c" -lt "$HALF_C" ] && HALF_C=$c; [ "$m" -lt "$HALF_M" ] && HALF_M=$m
done
echo "container lon nhat ${BIG_C}m/${BIG_M}Mi ; nho nhat ${SML_C}m/${SML_M}Mi ; nua worker ${HALF_C}m/${HALF_M}Mi" | tee "$EV/b2-nguong.txt"
{ [ "$(cpu_m "$(lr defaultRequest.cpu)")" -le 50 ] && [ "$(mem_mi "$(lr defaultRequest.memory)")" -le 64 ] && \
  [ "$(cpu_m "$(lr default.cpu)")" -le 200 ] && [ "$(mem_mi "$(lr default.memory)")" -le 128 ]; } \
  && echo 'PASS: gia tri mac dinh co mot container phu tro'
{ [ "$(cpu_m "$(lr max.cpu)")" -ge "$BIG_C" ] && [ "$(mem_mi "$(lr max.memory)")" -ge "$BIG_M" ] && \
  [ "$(cpu_m "$(lr max.cpu)")" -le "$HALF_C" ] && [ "$(mem_mi "$(lr max.memory)")" -le "$HALF_M" ] && \
  [ "$(cpu_m "$(lr min.cpu)")" -le "$SML_C" ] && [ "$(mem_mi "$(lr min.memory)")" -le "$SML_M" ]; } \
  && echo 'PASS: max bao duoc container lon nhat va duoi nua worker; min duoi container nho nhat'

# 3. Không Pod dự án nào phải nhờ giá trị mặc định
LRP="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{.metadata.name}={.metadata.annotations.kubernetes\.io/limit-ranger}{"\n"}{end}' | grep -v '=$')"
[ -z "$LRP" ] && echo 'PASS: khong Pod du an nao mang annotation kubernetes.io/limit-ranger' \
  || echo "FAIL: Pod nhan gia tri mac dinh: $(echo $LRP)"

# 4. Pod tạm nhận giá trị mặc định
k delete pod lr-tam --ignore-not-found --grace-period=1 >/dev/null
k run lr-tam --image=busybox:1.37 --restart=Never \
  --overrides='{"apiVersion":"v1","spec":{"nodeSelector":{"kubernetes.io/hostname":"lab-k8s-worker2"},"terminationGracePeriodSeconds":1}}' \
  -- sleep 600 >/dev/null
k wait --for=condition=Ready pod/lr-tam --timeout=120s >/dev/null
k get pod lr-tam -o yaml > "$EV/b2-lr-tam.yaml"
{ [ "$(cpu_m "$(jp pod/lr-tam '{.spec.containers[0].resources.requests.cpu}')")" -eq "$(cpu_m "$(lr defaultRequest.cpu)")" ] && \
  [ "$(cpu_m "$(jp pod/lr-tam '{.spec.containers[0].resources.limits.cpu}')")" -eq "$(cpu_m "$(lr default.cpu)")" ] && \
  [ "$(mem_mi "$(jp pod/lr-tam '{.spec.containers[0].resources.limits.memory}')")" -eq "$(mem_mi "$(lr default.memory)")" ] && \
  [ -n "$(jp pod/lr-tam '{.metadata.annotations.kubernetes\.io/limit-ranger}')" ]; } \
  && echo 'PASS: Pod tam nhan dung gia tri mac dinh, co dau vet kubernetes.io/limit-ranger'
k delete pod lr-tam --grace-period=1 >/dev/null

# 5. Vượt max và dưới min
pod() { printf 'apiVersion: v1\nkind: Pod\nmetadata: {name: %s, namespace: bookstore}\nspec:\n  nodeSelector: {kubernetes.io/hostname: lab-k8s-worker2}\n  terminationGracePeriodSeconds: 1\n  containers:\n  - name: app\n    image: busybox:1.37\n    command: [sleep, "600"]\n    resources: %s\n' "$1" "$2"; }
MAXC="$(cpu_m "$(lr max.cpu)")"; MINC="$(cpu_m "$(lr min.cpu)")"
pod lr-qua-max "{requests: {cpu: ${MINC}m}, limits: {cpu: $((MAXC * 2))m}}" | kubectl create -f - > "$EV/b2-qua-max.txt" 2>&1
pod lr-duoi-min "{requests: {cpu: $((MINC / 2 > 0 ? MINC / 2 : 1))m}, limits: {cpu: ${MINC}m}}" | kubectl create -f - > "$EV/b2-duoi-min.txt" 2>&1
cat "$EV/b2-qua-max.txt" "$EV/b2-duoi-min.txt"
{ grep -qi forbidden "$EV/b2-qua-max.txt" && grep -q 'maximum cpu usage per Container' "$EV/b2-qua-max.txt" && \
  grep -q 'minimum cpu usage per Container' "$EV/b2-duoi-min.txt" && \
  ! k get pod lr-qua-max >/dev/null 2>&1 && ! k get pod lr-duoi-min >/dev/null 2>&1; } \
  && echo 'PASS: vuot max va duoi min deu bi tu choi ngay luc tao, moi thong bao chi dung rang buoc'

# 6. Bẫy của bài 133
TRAP=$(( $(cpu_m "$(lr default.cpu)") + 50 ))
pod lr-bay "{requests: {cpu: ${TRAP}m}}" | kubectl create -f - > "$EV/b2-bay.txt" 2>&1
cat "$EV/b2-bay.txt"
{ grep -q 'must be less than or equal to cpu limit' "$EV/b2-bay.txt" && ! grep -qi forbidden "$EV/b2-bay.txt"; } \
  && echo 'PASS: bay — Pod bi tu choi vi request lon hon limit LimitRange vua chen; khong phai Forbidden'
pod lr-bay-sua "{requests: {cpu: ${TRAP}m}, limits: {cpu: ${TRAP}m}}" | kubectl create -f - >/dev/null
k wait --for=condition=Ready pod/lr-bay-sua --timeout=120s >/dev/null && echo 'PASS: khai ca requests va limits thi Pod len duoc'
k delete pod lr-bay-sua --grace-period=1 >/dev/null
ls docs/adr/*-limitrange.md >/dev/null 2>&1 && echo 'PASS: co ADR limitrange'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/policy/limitrange.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: bookstore-limits
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
spec:
  limits:
  - type: Container
    defaultRequest:      # cho container không khai: Pod tạm, Pod debug của người vận hành
      cpu: 10m
      memory: 16Mi
    default:             # limit mặc định, cỡ một container busybox
      cpu: 100m
      memory: 64Mi
    max:                 # trần mỗi container: gấp đôi db (500m, 512Mi), dưới nửa một worker
      cpu: "1"
      memory: 1Gi
    min:                 # sàn mỗi container: dưới status-writer (10m, 8Mi)
      cpu: 5m
      memory: 4Mi
```

Trong `k8s/app/kustomization.yaml`, mục `resources`, thêm một dòng:

```yaml
  - policy/limitrange.yaml
```

```bash
mkdir -p ~/bookstore/k8s/app/policy
cd ~/bookstore && ./scripts/deploy.sh && kubectl -n bookstore describe limitrange bookstore-limits
```

Ý chính cho ADR:

1. Giá trị mặc định rơi vào **mọi** container không khai, kể cả init container. Request hiệu dụng
   lấy giá trị lớn nhất của các init container (bài 51), nên một init quên khai mà nhận 500m sẽ
   kéo cả Pod lên 500m — scheduler giữ chỗ và quota ở B3 đếm 500m dù init chạy vài giây. Mặc định
   nhỏ làm sai lầm đó rẻ; B1 bắt khai tường minh để nó không xảy ra.
2. Không. Bài 133: LimitRange có mặt để **một đối tượng đơn lẻ** không chiếm toàn bộ tài nguyên;
   trần cho cả namespace là việc của ResourceQuota. Hai mươi container mỗi cái 1 CPU vẫn hợp lệ.
3. Annotation ghi container nào nhận request, limit nào từ LimitRange. Pod dự án mang nó nghĩa là
   repo đang dựa vào một mặc định nằm ngoài manifest của thành phần đó — đổi LimitRange là đổi
   ngầm tài nguyên của hệ thống.
4. Bài 133: hai LimitRange trở lên thì giá trị mặc định nào được áp là **không xác định**. Mọi thay
   đổi về mặc định phải sửa đúng `bookstore-limits`.
5. LimitRange không kiểm tính nhất quán của mặc định nó áp: nó chèn limit 100m cạnh request 150m;
   Pod tự mâu thuẫn bị **validation** của object từ chối (`Invalid`), không phải admission của
   LimitRange (`Forbidden`). Sửa từ phía Pod: khai cả `requests` lẫn `limits`.

</details>

## B3. [BS-7B-03] ResourceQuota `bookstore-quota`: trần tài nguyên tính toán

**Loại:** Tính năng.

**Bối cảnh:** cluster lab là môi trường dùng chung; các sprint sau còn thêm tenant `bookstore-staging`
(9b) chạy chung hai worker. `bookstore` cần một trần **tổng** — đủ cho hệ thống lúc đông nhất, đủ
chặt để một sai lầm như scale nhầm không nuốt cả cluster. Con số lấy từ ngân sách của B1.

**Yêu cầu:**

1. ResourceQuota `bookstore-quota` ở `k8s/app/policy/resourcequota.yaml`, label dự án, có
   `namespace`, thêm vào `k8s/app/kustomization.yaml`, triển khai qua `deploy.sh`.
2. Trần cho `requests.cpu`, `requests.memory`, `limits.cpu`, `limits.memory` và số Pod — không nhỏ
   hơn đỉnh trong `docs/ngan-sach-namespace.md`, có dự phòng; giải thích con số trong ADR.
3. Trần `requests.cpu` và `requests.memory` **nhỏ hơn** tổng allocatable của hai worker: quota phải
   là thứ chặn trước scheduler.
4. Hệ thống dựng lại từ sạch với quota có hiệu lực; smoke PASS; `used` nằm dưới `hard`.
5. Chỗ cho Lab 11b: `web` ở 5 bản sao, đang phát hành lại, cùng lúc một lần backup chạy tay — tất cả
   xong mà không cần ai nới quota.
6. Một sự cố nhân rộng — `web` bị scale nhầm lên 20 — bị quota chặn trước khi đủ 20 Pod. `kubectl`
   không báo lỗi; nguyên nhân đọc được ở đâu?
7. Quota không ràng buộc node: `lab-k8s-worker2` chạy Pod của `bookstore` cùng Pod của namespace
   khác.

**Ràng buộc:** không đổi `replicas` trong repo. `kubectl scale` chỉ chạy trong script nghiệm thu;
`deploy.sh` đưa hệ thống về đúng repo. Chưa đặt quota theo số lượng object (B4).

**Đọc lại:**

- Bài [134](../134-resource-quotas-vi.md):
  [*Cách ResourceQuota trong Kubernetes hoạt động*](../134-resource-quotas-vi.md#cách-resourcequota-trong-kubernetes-hoạt-động-how-kubernetes-resourcequotas-work)
  — đoạn về Deployment "thành công" mà không đưa đủ Pod vào tồn tại;
  [*Hạn ngạch cho tài nguyên hạ tầng*](../134-resource-quotas-vi.md#compute-resource-quota);
  [*Xem và thiết lập hạn ngạch*](../134-resource-quotas-vi.md#viewing-and-setting-quotas);
  [*Hạn ngạch và dung lượng cluster*](../134-resource-quotas-vi.md#hạn-ngạch-và-dung-lượng-cluster-quota-and-cluster-capacity).
- Bài [63](../63-deployment-vi.md#failed-deployment), mục *Deployment thất bại* — ví dụ thiếu quota
  với condition `ReplicaFailure`.

**Bàn giao:** `k8s/app/policy/resourcequota.yaml`; `k8s/app/kustomization.yaml`;
`docs/adr/NNNN-resourcequota.md`; `~/lab-evidence/7b/b3-*.txt`.

**Nghiệm thu:**

- `PASS:` teardown rồi deploy thành công với quota; smoke PASS.
- `PASS:` `bookstore-quota` mang label dự án; mỗi trần trong năm tài nguyên ≥ `dinh.*` của ngân sách;
  `used` ≤ `hard`.
- `PASS:` trần `requests.cpu`, `requests.memory` nhỏ hơn tổng allocatable của hai worker.
- `PASS:` `web` 5 bản sao phát hành lại xong trong 300 giây; backup chạy tay cùng lúc `Complete`.
- `PASS:` `web` scale lên 20: lệnh thành công; Deployment có `ReplicaFailure` lý do `FailedCreate`,
  thông điệp có `exceeded quota: bookstore-quota`; số bản sao Ready dưới 20.
- `PASS:` `deploy.sh` đưa `web` về 3 bản sao Ready.
- `PASS:` `lab-k8s-worker2` chạy Pod của `bookstore` và của ít nhất một namespace khác.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-resourcequota.md`:

1. Bạn để dự phòng bao nhiêu trên đỉnh, vì sao không đặt bằng đúng đỉnh — và vì sao không đặt gấp
   mười cho yên tâm?
2. Vì sao `requests.*` phải dưới allocatable mà `limits.cpu` lại có thể vượt dung lượng thật của
   cluster? Bài 134 nói gì khi tổng hạn ngạch vượt dung lượng?
3. Thêm worker thứ ba: quota có tự nới không? Pod của `bookstore` có bị giữ trên node nào không?
4. Khi `web` bị scale lên 20: lệnh báo gì, bao nhiêu Pod tồn tại, nguyên nhân nằm ở object nào?
5. Bài 134 khuyên hạn chế quyền xóa, sửa ResourceQuota. Hôm nay ai sửa được `bookstore-quota`, và
   sprint nào của dự án đóng lỗ hổng đó?

<details>
<summary>Gợi ý</summary>

1. Tài nguyên trong `spec.hard` dùng đúng tên ở bảng của bài 134: `requests.cpu`, `limits.memory`,
   …; số Pod là `pods`.
2. Đọc trạng thái: `kubectl -n bookstore describe quota bookstore-quota`, hoặc
   `kubectl -n bookstore get resourcequota bookstore-quota -o yaml` rồi xem `status.hard`,
   `status.used`.
3. Allocatable của node: `kubectl get node <tên> -o jsonpath='{.status.allocatable}'`.
4. Kustomize xếp ResourceQuota ngay sau Namespace và LimitRange trước mọi workload khi render, nên
   chính sách có mặt trước khi Pod đầu tiên được tạo.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script teardown, deploy, rồi scale thử</summary>

```bash
. ~/lab-work/7b/lib.sh
cd ~/bookstore
dv() { sed -n "s/^$1=//p" docs/ngan-sach-namespace.md | head -n 1 | tr -d ' \r'; }

./scripts/teardown.sh > "$EV/b3-teardown.txt" 2>&1
{ ./scripts/deploy.sh > "$EV/b3-deploy.txt" 2>&1 && ./tests/smoke.sh > "$EV/b3-smoke.txt" 2>&1; } \
  && echo 'PASS: he thong dung lai tu sach voi quota, smoke PASS' || echo 'FAIL: xem b3-deploy.txt, b3-smoke.txt'

for i in $(seq 1 30); do [ -n "$(qv bookstore-quota used pods)" ] && break; sleep 2; done
k get resourcequota bookstore-quota -o yaml > "$EV/b3-quota.yaml"
[ "$(jp resourcequota/bookstore-quota '{.metadata.labels.app\.kubernetes\.io/part-of}')" = bookstore ] \
  && echo 'PASS: bookstore-quota mang label du an'
BAD=0
for K in requests.cpu requests.memory limits.cpu limits.memory pods; do
  H="$(qv bookstore-quota hard "$K")"; U="$(qv bookstore-quota used "$K")"; D="$(dv "dinh.$K")"
  echo "$K used=$U hard=$H dinh=$D" | tee -a "$EV/b3-so-sanh.txt"
  [ -n "$H" ] || { echo "FAIL: thieu tran $K"; BAD=1; continue; }
  [ "$(norm "$K" "$H")" -ge "$(norm "$K" "$D")" ] || { echo "FAIL: $K hard=$H nho hon dinh=$D"; BAD=1; }
  [ "$(norm "$K" "$U")" -le "$(norm "$K" "$H")" ] || { echo "FAIL: $K used=$U vuot hard=$H"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: moi tran nam tren dinh cua ngan sach, used nam duoi tran'

ALC=0; ALM=0
for n in lab-k8s-worker1 lab-k8s-worker2; do
  ALC=$(( ALC + $(cpu_m "$(kubectl get node "$n" -o jsonpath='{.status.allocatable.cpu}')") ))
  ALM=$(( ALM + $(mem_mi "$(kubectl get node "$n" -o jsonpath='{.status.allocatable.memory}')") ))
done
{ [ "$(cpu_m "$(qv bookstore-quota hard requests.cpu)")" -lt "$ALC" ] && \
  [ "$(mem_mi "$(qv bookstore-quota hard requests.memory)")" -lt "$ALM" ]; } \
  && echo "PASS: tran requests nho hon tong allocatable hai worker (${ALC}m, ${ALM}Mi)"

# Chỗ cho Lab 11b: web 5 bản sao, phát hành lại, cùng lúc một lần backup chạy tay
k scale deployment web --replicas=5 >/dev/null
k rollout status deployment/web --timeout=300s >/dev/null
k delete job b3-backup --ignore-not-found --wait=true >/dev/null
k create job b3-backup --from=cronjob/db-backup >/dev/null
k rollout restart deployment/web >/dev/null
k rollout status deployment/web --timeout=300s > "$EV/b3-rollout-5.txt" 2>&1 \
  && echo 'PASS: web 5 ban sao phat hanh lai xong trong 300 giay'
k wait --for=condition=Complete job/b3-backup --timeout=300s >/dev/null \
  && echo 'PASS: backup chay tay Complete cung luc'
k delete job b3-backup --wait=true >/dev/null
echo "su kien FailedCreate thoang qua trong lan phat hanh: $(k get events --field-selector reason=FailedCreate --no-headers 2>/dev/null | wc -l)"

# Sự cố nhân rộng: web bị scale nhầm lên 20
k scale deployment web --replicas=20 > "$EV/b3-scale-20.txt" 2>&1 \
  && echo 'PASS: lenh scale len 20 thanh cong — kubectl khong bao loi'
for i in $(seq 1 60); do
  [ "$(jp deployment/web '{.status.conditions[?(@.type=="ReplicaFailure")].reason}')" = FailedCreate ] && break; sleep 3
done
sleep 20
R="$(jp deployment/web '{.status.readyReplicas}')"
k get deployment web -o yaml > "$EV/b3-web-20.yaml"
{ [ "$(jp deployment/web '{.status.conditions[?(@.type=="ReplicaFailure")].reason}')" = FailedCreate ] && \
  grep -q 'exceeded quota: bookstore-quota' "$EV/b3-web-20.yaml" && [ "${R:-0}" -lt 20 ]; } \
  && echo "PASS: quota chan o $R ban sao — Deployment ghi ReplicaFailure/FailedCreate, exceeded quota"

./scripts/deploy.sh > "$EV/b3-deploy-2.txt" 2>&1
[ "$(jp deployment/web '{.status.readyReplicas}')" = 3 ] && echo 'PASS: deploy.sh dua web ve 3 ban sao'

NSS="$(kubectl get pods -A --field-selector spec.nodeName=lab-k8s-worker2 -o jsonpath='{range .items[*]}{.metadata.namespace}{"\n"}{end}' | sort -u)"
echo "$NSS" > "$EV/b3-worker2-namespaces.txt"
{ echo "$NSS" | grep -qx bookstore && [ "$(echo "$NSS" | grep -vxc bookstore)" -ge 1 ]; } \
  && echo 'PASS: worker2 chay Pod cua bookstore lan namespace khac — quota khong rang buoc node'
ls docs/adr/*-resourcequota.md >/dev/null 2>&1 && echo 'PASS: co ADR resourcequota'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/policy/resourcequota.yaml — sau B3; B4 thêm phần số lượng object và lưu trữ
apiVersion: v1
kind: ResourceQuota
metadata:
  name: bookstore-quota
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
spec:
  hard:
    # Đỉnh ở docs/ngan-sach-namespace.md: 1310m, 1304Mi, 5100m, 2640Mi, 19 Pod — cộng dự phòng
    requests.cpu: 1500m
    requests.memory: 1536Mi
    limits.cpu: "6"
    limits.memory: 3Gi
    pods: "20"
```

Trong `k8s/app/kustomization.yaml`, mục `resources`, thêm `- policy/resourcequota.yaml`, rồi
`./scripts/deploy.sh`.

Ý chính cho ADR:

1. Đỉnh là ước lượng — thêm khoảng 15% cho Pod tạm thứ hai, một lần rollout trùng lúc chạy
   migration. Bằng đúng đỉnh thì lần phát hành đầu tiên lệch khỏi giả định sẽ kẹt (B5). Gấp mười
   thì quota không chặn được scale nhầm và namespace có thể chiếm hết hai worker.
2. Request là thứ scheduler giữ chỗ: tổng request vượt allocatable thì Pod `Pending` — quota phải
   chặn trước. Limit chỉ là trần lúc chạy, được phép overcommit; `limits.cpu` 6 trên hai worker 2
   vCPU là overcommit có chủ đích. Bài 134: tổng hạn ngạch vượt dung lượng thì tranh chấp xử lý theo
   ai đến trước được phục vụ trước.
3. Không và không. Bài 134: ResourceQuota là đơn vị tuyệt đối, độc lập dung lượng — thêm node không
   tự nới; và nó không tạo ràng buộc nào về node. Muốn giữ Pod trên node nào là việc của affinity,
   taint của Lab 7a.
4. `kubectl scale` báo thành công; Deployment và ReplicaSet tồn tại; chỉ một phần Pod được tạo.
   Nguyên nhân nằm ở condition `ReplicaFailure` của Deployment và sự kiện `FailedCreate` của
   ReplicaSet — đúng bẫy Deployment của bài 134 và ví dụ thiếu quota của bài 63.
5. Hôm nay ai có kubeconfig quản trị đều sửa được. Lab 9a cho nhóm `bookstore-devs` quyền `view`
   và `ci-deployer` chỉ cập nhật Deployment, ConfigMap — quota nằm ngoài quyền của cả hai.

</details>

## B4. [BS-7B-04] Quota theo số lượng object và lưu trữ

**Loại:** Tính năng.

**Bối cảnh:** trần CPU/memory không bảo vệ được ba thứ. Một là **quyết định kiến trúc** của Lab 5b:
người dùng vào qua Ingress HTTPS — một Service `NodePort` trong `bookstore` sẽ mở thẳng một cổng trên
mọi node, vòng qua TLS. Hai là **dữ liệu**: `db` chỉ có một bản sao, PVC đặt tên theo ordinal — scale
nhầm `db` lên 2 là sinh `data-db-1` và một DB rỗng thứ hai. Ba là **control plane**: một CronJob cấu
hình hỏng có thể đẻ Job tới mức từ chối dịch vụ (bài 134).

**Yêu cầu:**

1. Bổ sung vào **cùng** `bookstore-quota`:
   - Không Service `NodePort` hay `LoadBalancer` nào được tồn tại trong `bookstore`.
   - Trần số PVC và tổng dung lượng PVC: đủ cho PVC hiện có cộng **tối đa hai** PVC dự phòng.
   - Trần số Job: tính từ giới hạn lịch sử của CronJob `db-backup`, cộng Job `db-migrate`, một lần
     chạy tay và một lần đang chạy — dự phòng thêm không quá 10.
2. Chứng minh: Service `NodePort` bị từ chối còn `ClusterIP` vẫn tạo được; PVC vượt trần bị từ chối;
   Job vượt trần bị từ chối — và trong lúc trần Job đầy, một lần backup chạy tay **cũng** bị từ chối.
3. ⭐ Nếu mọi PVC của dự án dùng StorageClass `bookstore-retain`: cấm PVC dùng StorageClass mặc
   định trong `bookstore`, bằng quota theo StorageClass.

**Ràng buộc:** object thử nghiệm không mang label dự án và bị xóa khi xong. PVC thử nghiệm không
được sinh PV — dùng một StorageClass không tồn tại. Job thử nghiệm không được tạo Pod nào. Service thử
nghiệm chỉ gửi bằng server dry-run.

**Đọc lại:**

- Bài [134](../134-resource-quotas-vi.md):
  [*Hạn ngạch cho lưu trữ*](../134-resource-quotas-vi.md#hạn-ngạch-cho-lưu-trữ-quota-for-storage),
  [*Hạn ngạch theo số lượng đối tượng*](../134-resource-quotas-vi.md#quota-on-object-count) — đoạn
  về Secret và CronJob, [*Cú pháp chuyên biệt*](../134-resource-quotas-vi.md#resource-quota-object-count-specialized).
- Bài [69](../69-cron-jobs-vi.md#giới-hạn-lịch-sử-job-jobs-history-limits) — giới hạn lịch sử Job.
- Bài [67](../67-job-vi.md#kiểm-soát-mức-song-song-controlling-parallelism) — `parallelism: 0`.
- [Mục 4.2 của file dự án](DU-AN-BOOKSTORE.md#42-cổng-đường-dẫn-và-điểm-kiểm-tra-sức-khỏe) — từ 5b
  `web` là ClusterIP, người dùng đi qua Ingress.

**Bàn giao:** `k8s/app/policy/resourcequota.yaml`; `docs/adr/NNNN-quota-so-luong-object.md`;
`~/lab-evidence/7b/b4-*.txt`.

**Nghiệm thu:**

- `PASS:` `services.nodeports` bằng 0; Service `NodePort` bị từ chối với `exceeded quota`; Service
  `ClusterIP` vẫn qua.
- `PASS:` có trần `persistentvolumeclaims` (dư so với hiện có từ 0 tới 2) và `requests.storage`
  (không nhỏ hơn đang dùng).
- `PASS:` lấp đầy PVC: `used` bằng `hard`, PVC tiếp theo bị từ chối; dọn xong `used` về như cũ.
- `PASS:` `count/jobs.batch` nằm trong khoảng tính từ giới hạn lịch sử của `db-backup`; lấp đầy:
  Job tiếp theo bị từ chối, backup chạy tay cũng bị từ chối; dọn xong `used` về như cũ.
- `PASS:` smoke vẫn PASS.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-quota-so-luong-object.md`:

1. `services.nodeports: 0` bảo vệ quyết định nào? Lab 8b dùng overlay `ha-lab` với `web` NodePort
   trên cụm HA — overlay đó phải làm gì với `bookstore-quota`?
2. Trần Job của bạn tính thế nào? Nếu đặt quá chặt, `db-backup` ngừng chạy thì ai, ở đâu phát hiện?
3. PVC thử nghiệm nằm `Pending`, không bind vào PV nào — nó có bị tính vào quota không, vì sao?
4. Trần PVC chặn được việc scale `db` lên hai bản sao thế nào? Bài 134 nói họ quota số lượng object
   tồn tại để bảo vệ cái gì?

<details>
<summary>Gợi ý</summary>

1. Tên tài nguyên: `services.nodeports`, `services.loadbalancers`, `persistentvolumeclaims`,
   `requests.storage`, `count/jobs.batch` — chuỗi, nên đặt trong dấu nháy.
2. PVC hiện có và dung lượng: `kubectl -n bookstore get pvc -o custom-columns=NAME:.metadata.name,SIZE:.spec.resources.requests.storage,SC:.spec.storageClassName`.
3. Giới hạn lịch sử của CronJob: `.spec.successfulJobsHistoryLimit`, `.spec.failedJobsHistoryLimit`
   — mặc định 3 và 1 nếu bạn không khai.
4. Gửi object mà không tạo: `kubectl create --dry-run=server -f -`. Quota vẫn soi yêu cầu dry-run,
   chỉ không cộng vào `used`.
5. ⭐ Tên quota theo StorageClass: `<tên class>.storageclass.storage.k8s.io/persistentvolumeclaims`.
   Tên class mặc định đọc từ annotation `storageclass.kubernetes.io/is-default-class`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
. ~/lab-work/7b/lib.sh
cd ~/bookstore
./scripts/deploy.sh > "$EV/b4-deploy.txt" 2>&1 || echo 'FAIL: deploy.sh that bai'
for i in $(seq 1 30); do [ -n "$(qv bookstore-quota used count/jobs.batch)" ] && break; sleep 2; done
k describe quota bookstore-quota > "$EV/b4-quota.txt"; cat "$EV/b4-quota.txt"

# 1. NodePort bị cấm, ClusterIP vẫn qua
svc() { printf 'apiVersion: v1\nkind: Service\nmetadata: {name: %s, namespace: bookstore}\nspec:\n  type: %s\n  selector: {lab: khong-co}\n  ports: [{port: 80}]\n' "$1" "$2"; }
svc np-thu NodePort | kubectl create --dry-run=server -f - > "$EV/b4-nodeport.txt" 2>&1
{ [ "$(qv bookstore-quota hard services.nodeports)" = 0 ] && \
  grep -q 'exceeded quota: bookstore-quota' "$EV/b4-nodeport.txt" && grep -q 'services.nodeports' "$EV/b4-nodeport.txt"; } \
  && echo 'PASS: services.nodeports=0 — Service NodePort bi tu choi'
svc ci-thu ClusterIP | kubectl create --dry-run=server -f - >/dev/null 2>&1 && echo 'PASS: Service ClusterIP van qua'

# 2. PVC
PH="$(qv bookstore-quota hard persistentvolumeclaims)"; PU="$(qv bookstore-quota used persistentvolumeclaims)"
SH="$(qv bookstore-quota hard requests.storage)"; SU="$(qv bookstore-quota used requests.storage)"
echo "pvc used=$PU hard=$PH ; storage used=$SU hard=$SH" | tee "$EV/b4-pvc.txt"
{ [ -n "$PH" ] && [ $((PH - PU)) -ge 0 ] && [ $((PH - PU)) -le 2 ] && [ -n "$SH" ] && \
  [ "$(mem_mi "$SH")" -ge "$(mem_mi "$SU")" ]; } && echo 'PASS: tran PVC du hien co cong toi da hai du phong'
pvc() { printf 'apiVersion: v1\nkind: PersistentVolumeClaim\nmetadata: {name: %s, namespace: bookstore, labels: {lab: 7b-dem}}\nspec:\n  storageClassName: quota-thu-khong-co\n  accessModes: [ReadWriteOnce]\n  resources: {requests: {storage: 1Mi}}\n' "$1"; }
i=0
while pvc "pvc-thu-$i" | kubectl create -f - > "$EV/b4-pvc-$i.txt" 2>&1; do i=$((i + 1)); [ "$i" -gt 5 ] && break; done
{ [ "$(qv bookstore-quota used persistentvolumeclaims)" = "$PH" ] && grep -q 'exceeded quota: bookstore-quota' "$EV/b4-pvc-$i.txt"; } \
  && echo "PASS: tao duoc $i PVC thu thi day tran; PVC tiep theo bi tu choi"
k delete pvc -l lab=7b-dem --wait=true >/dev/null
for i in $(seq 1 30); do [ "$(qv bookstore-quota used persistentvolumeclaims)" = "$PU" ] && break; sleep 2; done
[ "$(qv bookstore-quota used persistentvolumeclaims)" = "$PU" ] && echo 'PASS: don PVC thu, used tro ve nhu cu'

# 3. Job
H="$(jp cronjob/db-backup '{.spec.successfulJobsHistoryLimit}')"; F="$(jp cronjob/db-backup '{.spec.failedJobsHistoryLimit}')"
LO=$(( ${H:-3} + ${F:-1} + 3 ))
JH="$(qv bookstore-quota hard count/jobs.batch)"; JU="$(qv bookstore-quota used count/jobs.batch)"
echo "count/jobs.batch used=$JU hard=$JH ; toi thieu=$LO" | tee "$EV/b4-job.txt"
{ [ -n "$JH" ] && [ "$JH" -ge "$LO" ] && [ "$JH" -le $((LO + 10)) ]; } \
  && echo "PASS: count/jobs.batch=$JH tinh tu lich su cua db-backup"
job() { printf 'apiVersion: batch/v1\nkind: Job\nmetadata: {name: %s, namespace: bookstore, labels: {lab: 7b-dem}}\nspec:\n  parallelism: 0\n  template:\n    spec:\n      restartPolicy: Never\n      containers: [{name: x, image: "busybox:1.37", command: ["true"]}]\n' "$1"; }
i=0
while job "job-thu-$i" | kubectl create -f - > "$EV/b4-job-$i.txt" 2>&1; do i=$((i + 1)); [ "$i" -gt 30 ] && break; done
{ [ "$(qv bookstore-quota used count/jobs.batch)" = "$JH" ] && grep -q 'count/jobs.batch' "$EV/b4-job-$i.txt"; } \
  && echo "PASS: tao duoc $i Job thu thi day tran; Job tiep theo bi tu choi"
if k create job b4-backup --from=cronjob/db-backup > "$EV/b4-backup-bi-chan.txt" 2>&1; then
  echo 'FAIL: backup chay tay van tao duoc'; k delete job b4-backup >/dev/null
else
  grep -q 'exceeded quota' "$EV/b4-backup-bi-chan.txt" && echo 'PASS: tran Job day thi backup chay tay cung bi tu choi'
fi
k delete job -l lab=7b-dem --wait=true >/dev/null
for i in $(seq 1 30); do [ "$(qv bookstore-quota used count/jobs.batch)" -le "$JU" ] && break; sleep 2; done
[ "$(qv bookstore-quota used count/jobs.batch)" -le "$JU" ] && echo 'PASS: don Job thu, used tro ve nhu cu'

# ⭐ Quota theo StorageClass
DSC="$(kubectl get storageclass -o jsonpath='{range .items[?(@.metadata.annotations.storageclass\.kubernetes\.io/is-default-class=="true")]}{.metadata.name}{end}')"
[ "$(qv bookstore-quota hard "$DSC.storageclass.storage.k8s.io/persistentvolumeclaims")" = 0 ] \
  && echo "PASS: (sao) cam PVC dung StorageClass mac dinh $DSC trong bookstore"

./tests/smoke.sh > "$EV/b4-smoke.txt" 2>&1 && echo 'PASS: smoke van PASS'
ls docs/adr/*-quota-so-luong-object.md >/dev/null 2>&1 && echo 'PASS: co ADR quota so luong object'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/policy/resourcequota.yaml — bản cuối của sprint
apiVersion: v1
kind: ResourceQuota
metadata:
  name: bookstore-quota
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
spec:
  hard:
    # Tài nguyên tính toán (B3) — đỉnh của docs/ngan-sach-namespace.md cộng dự phòng
    requests.cpu: 1500m
    requests.memory: 1536Mi
    limits.cpu: "6"
    limits.memory: 3Gi
    pods: "20"
    # Quyết định kiến trúc của Lab 5b: vào từ ngoài chỉ qua Ingress
    services.nodeports: "0"
    services.loadbalancers: "0"
    # Dữ liệu: data-db-0 và backup-store, cộng một PVC dự phòng
    persistentvolumeclaims: "3"
    requests.storage: 4Gi        # tổng PVC thật của bạn + dung lượng một PVC; 4Gi ứng với hai PVC 1Gi
    # Control plane: 3 Job thành công + 3 thất bại của db-backup + 1 đang chạy + db-migrate
    # + 1 chạy tay, cộng dự phòng
    count/jobs.batch: "12"
    # ⭐ chỉ khi mọi PVC của dự án dùng bookstore-retain
    # local-path.storageclass.storage.k8s.io/persistentvolumeclaims: "0"
```

Ý chính cho ADR:

1. Kiến trúc của Lab 5b: người dùng vào qua Ingress `bookstore` với TLS, NetworkPolicy chỉ mở cho
   Traefik vào `web`. `services.nodeports: 0` biến quyết định đó thành luật mà API server cưỡng chế,
   không chỉ là quy ước. Overlay `ha-lab` của Lab 8b phải vá `bookstore-quota` (cho
   `services.nodeports` ít nhất 1) hoặc bỏ quota khỏi overlay, nếu không Service `web` NodePort bị
   từ chối.
2. Trần = Job thành công + Job thất bại được giữ + Job đang chạy + `db-migrate` + một lần chạy tay +
   dự phòng. Quá chặt thì CronJob controller không tạo được Job, ghi sự kiện cảnh báo trên
   `cronjob/db-backup`; `status.lastSuccessfulTime` ngừng tăng. Không ai được báo — đó là lý do
   giám sát backup là việc của Lab 9a (`backup-status`) và Lab 11a.
3. Có. Quota đếm **object** và giá trị khai trong `spec` lúc admission, không đếm đĩa thật đã dùng;
   bài 134 tách `persistentvolumeclaims` (số claim) và `requests.storage` (tổng dung lượng xin).
4. Scale `db` lên 2 thì StatefulSet phải tạo PVC `data-db-1`; trần PVC còn dư không quá hai nên một
   lần nhầm còn lọt, hai lần thì bị chặn — và trần càng sát thì càng sớm. Bài 134: họ quota số lượng
   object tồn tại để bảo vệ **bộ lưu trữ của control plane** khỏi bị cạn — ở đây thêm một lớp bảo vệ
   dữ liệu.

</details>

## B5. [BS-7B-05] Diễn tập: rollout kẹt vì quota

**Loại:** Diễn tập sự cố — siết chính sách của namespace; không tiêm lỗi vào node nào.

**Bối cảnh:** một ngày quota bị hạ để nhường chỗ cho tenant khác, và đúng hôm đó có phát hành.
`kubectl` báo thành công, pipeline xanh, nhưng phiên bản mới không lên. Diễn tập để người trực nhận
ra tình huống trong vài phút và biết vì sao người dùng không bị ảnh hưởng.

**Yêu cầu:**

1. Ghi lại chiến lược rollout thật của `web` và số bản sao.
2. Siết `requests.cpu` của `bookstore-quota` xuống **đúng** mức đang dùng — bằng lệnh, không sửa
   repo.
3. Phát hành lại `web` (khởi động lại rollout, không đổi repo). Chứng minh: lệnh thành công;
   ReplicaSet mới không tạo được Pod nào và sự kiện `FailedCreate` của nó có đủ ba mảnh
   `requested`/`used`/`limited`; Deployment có condition `ReplicaFailure`; rollout không xong.
4. Chứng minh trong lúc kẹt `web` vẫn đủ bản sao cũ và vẫn phục vụ trang lẫn `/api/books`.
5. Trả quota về đúng repo; rollout hoàn tất; `deploy.sh` và smoke xanh.
6. Postmortem.

**Ràng buộc:** chỉ đổi quota, chỉ bằng lệnh trong diễn tập. Không đổi chiến lược rollout hay số bản
sao của `web`.

**Đọc lại:**

- Bài [63](../63-deployment-vi.md): [*Deployment thất bại*](../63-deployment-vi.md#failed-deployment)
  — ví dụ thiếu quota, ba condition và cách xử lý;
  [*Max Unavailable*](../63-deployment-vi.md#max-unavailable),
  [*Max Surge*](../63-deployment-vi.md#max-surge),
  [*Progress Deadline Seconds*](../63-deployment-vi.md#progress-deadline-seconds).
- Bài [134](../134-resource-quotas-vi.md#cách-resourcequota-trong-kubernetes-hoạt-động-how-kubernetes-resourcequotas-work)
  — Deployment "thành công" mà không đưa đủ Pod vào tồn tại; ghi chú *Thận trọng* đầu bài.

**Bàn giao:** `docs/postmortem/7b-rollout-ket-vi-quota.md`; `~/lab-evidence/7b/b5-*.txt`.

**Nghiệm thu:**

- `PASS:` `hard` của `requests.cpu` bằng `used` sau khi siết.
- `PASS:` lệnh phát hành thành công; sự kiện `FailedCreate` của ReplicaSet mới có
  `exceeded quota: bookstore-quota`, `requested: requests.cpu=`, `limited: requests.cpu=`.
- `PASS:` Deployment `web` có condition `ReplicaFailure`, lý do `FailedCreate`.
- `PASS:` (với `maxUnavailable` bằng 0) `rollout status` hết giờ; vẫn đủ bản sao cũ Ready, ReplicaSet
  mới 0 Pod; `web` phục vụ `/healthz` và `/api/books`.
- `PASS:` sau khi trả quota: `hard` về giá trị của repo, rollout xong, `deploy.sh` và smoke xanh.

**Câu hỏi thiết kế** — trả lời trong postmortem:

1. Vì sao kẹt mà không mất dịch vụ? Nếu `web` dùng `maxSurge: 0`, `maxUnavailable: 1` thì phát hành
   có qua được quota bị siết không, và trả giá bằng gì?
2. Người trực bắt đầu từ đâu khi `apply` và `rollout restart` đều báo thành công? Chỉ ra ba chỗ đọc
   được nguyên nhân.
3. Sau khi trả quota, vì sao rollout có thể không chạy tiếp **ngay**? Script làm gì để không phải chờ?
4. Bài học cho ngân sách B1: dự phòng phải đủ cho thứ gì?

<details>
<summary>Gợi ý</summary>

- Sửa một khóa của `spec.hard` bằng lệnh: `kubectl patch resourcequota ... --type=merge -p '{"spec":{"hard":{"requests.cpu":"<giá trị>"}}}'`.
- Phát hành lại không đổi manifest: `kubectl rollout restart deployment/web`.
- ReplicaSet mới là ReplicaSet có annotation `deployment.kubernetes.io/revision` bằng revision hiện
  tại của Deployment. Sự kiện của nó:
  `kubectl -n bookstore get events --field-selector involvedObject.kind=ReplicaSet,involvedObject.name=<tên>,reason=FailedCreate`.
- Trả quota về repo mà không chờ `deploy.sh` đứng đợi rollout: render Kustomize rồi `apply`, như bước
  đầu của `deploy.sh`.

</details>

<details>
<summary>Script nghiệm thu — script này chính là cuộc diễn tập; chạy liền một mạch</summary>

```bash
. ~/lab-work/7b/lib.sh
cd ~/bookstore
WSEL="$SEL,app.kubernetes.io/component=web"
REP="$(jp deployment/web '{.spec.replicas}')"
MU="$(jp deployment/web '{.spec.strategy.rollingUpdate.maxUnavailable}')"; MU="${MU:-25%}"
case "$MU" in *%) MU=$(( REP * ${MU%\%} / 100 )) ;; esac
H0="$(qv bookstore-quota hard requests.cpu)"; U0="$(qv bookstore-quota used requests.cpu)"
{ echo "replicas=$REP maxUnavailable(tinh ra)=$MU strategy=$(jp deployment/web '{.spec.strategy}')"
  echo "requests.cpu used=$U0 hard=$H0"; } | tee "$EV/b5-truoc.txt"

# 1. Siết quota xuống đúng mức đang dùng
k patch resourcequota bookstore-quota --type=merge -p "{\"spec\":{\"hard\":{\"requests.cpu\":\"$U0\"}}}" >/dev/null
for i in $(seq 1 30); do [ "$(qv bookstore-quota hard requests.cpu)" = "$U0" ] && break; sleep 2; done
[ "$(cpu_m "$(qv bookstore-quota hard requests.cpu)")" -eq "$(cpu_m "$U0")" ] \
  && echo "PASS: requests.cpu bi siet con $U0 — khong con cho trong"

# 2. Phát hành lại
R0="$(jp deployment/web '{.metadata.annotations.deployment\.kubernetes\.io/revision}')"
k rollout restart deployment/web && echo 'PASS: lenh phat hanh thanh cong — terminal khong bao loi'
for i in $(seq 1 30); do
  R1="$(jp deployment/web '{.metadata.annotations.deployment\.kubernetes\.io/revision}')"; [ "$R1" != "$R0" ] && break; sleep 2
done
NEWRS="$(k get rs -l "$WSEL" -o jsonpath='{range .items[*]}{.metadata.name} {.metadata.annotations.deployment\.kubernetes\.io/revision}{"\n"}{end}' | awk -v r="$R1" '$2 == r {print $1}')"
for i in $(seq 1 40); do [ -n "$(ev ReplicaSet "$NEWRS" FailedCreate)" ] && break; sleep 3; done
ev ReplicaSet "$NEWRS" FailedCreate > "$EV/b5-failedcreate.txt"; cat "$EV/b5-failedcreate.txt"
{ grep -q 'exceeded quota: bookstore-quota' "$EV/b5-failedcreate.txt" && \
  grep -q 'requested: requests.cpu=' "$EV/b5-failedcreate.txt" && grep -q 'limited: requests.cpu=' "$EV/b5-failedcreate.txt"; } \
  && echo "PASS: ReplicaSet moi $NEWRS ghi FailedCreate du ba manh requested/used/limited"
for i in $(seq 1 20); do
  [ "$(jp deployment/web '{.status.conditions[?(@.type=="ReplicaFailure")].reason}')" = FailedCreate ] && break; sleep 3
done
k get deployment web -o yaml > "$EV/b5-web.yaml"
[ "$(jp deployment/web '{.status.conditions[?(@.type=="ReplicaFailure")].reason}')" = FailedCreate ] \
  && echo 'PASS: Deployment web co condition ReplicaFailure/FailedCreate'

# 3. Kẹt nhưng vẫn phục vụ
if [ "$MU" -eq 0 ]; then
  k rollout status deployment/web --timeout=30s > "$EV/b5-rollout-status.txt" 2>&1 \
    && echo 'FAIL: rollout van xong' || echo 'PASS: rollout ket — rollout status het gio'
  { [ "$(jp deployment/web '{.status.readyReplicas}')" = "$REP" ] && [ "$(jp "rs/$NEWRS" '{.status.replicas}')" = 0 ]; } \
    && echo "PASS: van du $REP ban cu Ready, ReplicaSet moi 0 Pod"
  WEB="$(k get pod -l "$WSEL" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}')"
  { k exec "$WEB" -c status-writer -- wget -qO- http://127.0.0.1:8080/healthz >/dev/null && \
    [ "$(k exec "$WEB" -c status-writer -- wget -qO- http://127.0.0.1:8080/api/books | grep -o '"title"' | wc -l)" -ge 8 ]; } \
    && echo 'PASS: web van phuc vu trang va /api/books trong luc ket'
else
  echo "GHI CHU: maxUnavailable=$MU — Deployment duoc bot ban cu de di tiep; ghi dien bien vao postmortem"
fi

# 4. Trả quota về repo, phát hành lại để controller thử ngay
kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/app | kubectl apply -f - > "$EV/b5-tra-quota.txt"
for i in $(seq 1 30); do [ "$(qv bookstore-quota hard requests.cpu)" = "$H0" ] && break; sleep 2; done
[ "$(cpu_m "$(qv bookstore-quota hard requests.cpu)")" -eq "$(cpu_m "$H0")" ] && echo "PASS: requests.cpu ve $H0 nhu repo"
k rollout restart deployment/web >/dev/null
k rollout status deployment/web --timeout=300s >/dev/null && echo 'PASS: rollout hoan tat sau khi tra quota'
{ ./scripts/deploy.sh > "$EV/b5-deploy.txt" 2>&1 && ./tests/smoke.sh > "$EV/b5-smoke.txt" 2>&1; } \
  && echo 'PASS: deploy.sh va smoke xanh'
ls docs/postmortem/7b-rollout-ket-vi-quota.md >/dev/null 2>&1 && echo 'PASS: co postmortem'
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính của postmortem</summary>

1. Với `maxUnavailable: 0`, Deployment chỉ bớt Pod cũ khi Pod mới đã Ready. Pod mới không tạo được,
   nên Pod cũ đứng yên — người dùng không thấy gì (bài 63, *Max Unavailable*). Với `maxSurge: 0`,
   `maxUnavailable: 1`, Deployment xóa một Pod cũ trước; khi Pod đó hết bị đếm (hết thời gian kết
   thúc), chỗ vừa trả đủ cho Pod mới — phát hành đi qua quota bị siết, nhưng suốt rollout `web` chỉ
   còn `replicas - 1` bản sao và PDB không bảo vệ trường hợp này vì rollout không đi qua Eviction API.
2. (a) Condition `ReplicaFailure` của Deployment, thông điệp nằm trong `kubectl get deploy -o yaml`;
   (b) sự kiện `FailedCreate` trên ReplicaSet mới — `kubectl describe rs`; (c) `kubectl describe quota`
   thấy `used` bằng `hard`. Sau `progressDeadlineSeconds` (mặc định 600, phụ thuộc cấu hình),
   `Progressing` thành `False` với `ProgressDeadlineExceeded` — bài 63 nói Kubernetes **không** tự làm
   gì thêm.
3. ReplicaSet controller thử tạo Pod lại theo nhịp lùi dần sau mỗi lần thất bại; nới quota không phát
   sinh sự kiện nào cho ReplicaSet, nên lần thử kế tiếp có thể đến muộn — khoảng chờ phụ thuộc số lần
   đã thất bại. Script phát hành lại một lần nữa để controller xử lý ngay.
4. Dự phòng phải chứa phần trội của rollout **cộng** Pod đang kết thúc, ở số bản sao lớn nhất (HPA 5
   ở 11b), và không được ai "lấy lại" khi quota bị hạ — đó là lý do ADR B3 ghi rõ dự phòng dùng cho
   việc gì.

</details>

## B6. [BS-7B-06] Diễn tập: Pod thiếu khai báo và chính sách siết sai

**Loại:** Diễn tập sự cố — trên `lab-k8s-worker2`, nơi `db-0` chạy.

**Bối cảnh:** hai sự cố hay gặp khi namespace đã có chính sách. (A) LimitRange bị xóa nhầm khi dọn
dẹp: mọi Pod tạm của người vận hành bị từ chối với một thông báo lạ. (B) Ai đó hạ trần mỗi container
của LimitRange "cho chặt": không có gì hỏng ngay — cho tới lần `db-0` bị thay kế tiếp, ví dụ khi bảo
trì node ở Lab 12.

**Yêu cầu:**

1. Kịch bản A: gỡ `bookstore-limits`. Chứng minh Pod tạm không khai tài nguyên bị quota từ chối kiểu
   **thiếu khai báo** — khác kiểu vượt trần của B3, B5; workload của dự án (một lần backup chạy tay)
   vẫn chạy được vì đã khai tường minh.
2. Trả LimitRange về từ repo. Cùng Pod tạm đó giờ được tạo; `used.requests.cpu` của quota tăng
   **đúng** bằng giá trị LimitRange vừa chèn.
3. Kịch bản B: hạ trần CPU mỗi container của LimitRange xuống dưới limit CPU của `db` — vẫn không
   thấp hơn limit mặc định, để LimitRange còn hợp lệ. Chứng minh `db-0` đang chạy không bị đụng.
4. Xóa êm `db-0`. Chứng minh StatefulSet không tạo lại được, sự kiện chỉ đúng ràng buộc của
   LimitRange, và `api` mất Ready.
5. Trả LimitRange về từ repo; `db-0` lên lại trên `lab-k8s-worker2` với dữ liệu còn nguyên; `api`
   Ready; `deploy.sh` và smoke xanh.
6. Postmortem.

**Ràng buộc:** không xóa cưỡng bức `db-0`. LimitRange chỉ đổi bằng lệnh trong diễn tập. Pod tạm ghim
`lab-k8s-worker2`.

**Đọc lại:**

- Bài [133](../133-limit-range-vi.md#constraints-on-resource-limits-and-requests) — hai gạch đầu dòng
  cuối: kiểm tra chỉ diễn ra ở admission; thêm hay sửa LimitRange không đụng Pod đang chạy.
- Bài [134](../134-resource-quotas-vi.md#cách-resourcequota-trong-kubernetes-hoạt-động-how-kubernetes-resourcequotas-work)
  — "người dùng phải chỉ định request hoặc limit", và ghi chú về LimitRange "để người dùng không phải
  nhớ tự làm việc đó".
- Bài [65](../65-statefulset-vi.md#pod-identity) — `db-0` mới mang cùng định danh, cùng PVC.

**Bàn giao:** `docs/postmortem/7b-chinh-sach-siet-sai.md`; `~/lab-evidence/7b/b6-*.txt`.

**Nghiệm thu:**

- `PASS:` A — Pod tạm bị từ chối với `failed quota: bookstore-quota: must specify`, không có
  `exceeded quota`; không Pod nào được tạo.
- `PASS:` A — backup chạy tay `Complete` khi không có LimitRange.
- `PASS:` sau khi trả LimitRange: Pod tạm mang request bằng `defaultRequest`; `used.requests.cpu`
  tăng đúng bằng giá trị đó.
- `PASS:` B — LimitRange đã siết vẫn hợp lệ; `db-0` cùng UID, cùng `restartCount`, vẫn `Running`.
- `PASS:` B — sau khi xóa `db-0`: sự kiện `FailedCreate` của StatefulSet có
  `maximum cpu usage per Container`; không có `db-0`; `api` mất Ready.
- `PASS:` khôi phục: `db-0` Ready trên `lab-k8s-worker2`, ít nhất 8 cuốn sách, `api` Ready,
  `deploy.sh` và smoke xanh.

**Câu hỏi thiết kế** — trả lời trong postmortem:

1. Ba thông báo — `must specify`, `exceeded quota`, `maximum ... per Container` — mỗi cái do cơ chế
   nào sinh ra, và người nhận phải sửa ở đâu?
2. Vì sao quota đếm đúng giá trị mà LimitRange vừa chèn, chứ không đếm 0? Hai admission controller
   chạy theo thứ tự nào?
3. Vì sao hạ trần LimitRange không làm gì `db-0` đang chạy — và vì sao chính điều đó lại nguy hiểm?
   Lab 12 drain `lab-k8s-worker2` sẽ gặp gì nếu chính sách sai còn đó?
4. Đề xuất quy trình đổi chính sách cho đội: ngay sau khi đổi, làm gì để lỗi lộ ra trong giờ làm
   việc chứ không lúc 3 giờ sáng?

<details>
<summary>Gợi ý</summary>

- Gỡ: `kubectl -n bookstore delete limitrange bookstore-limits`. Trả: render Kustomize rồi `apply`.
- Sửa một phần tử của danh sách `spec.limits` bằng JSON patch:
  `--type=json -p '[{"op":"replace","path":"/spec/limits/<chỉ số>/max/cpu","value":"<giá trị>"}]'`.
- Sự kiện của StatefulSet:
  `kubectl -n bookstore get events --field-selector involvedObject.kind=StatefulSet,involvedObject.name=db,reason=FailedCreate`.
- StatefulSet thử tạo lại theo nhịp lùi dần; trả LimitRange ngay khi thấy sự kiện đầu tiên để nhịp
  chờ còn ngắn.

</details>

<details>
<summary>Script nghiệm thu — script này chính là cuộc diễn tập; chạy liền một mạch</summary>

```bash
. ~/lab-work/7b/lib.sh
cd ~/bookstore
render_apply() { kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/app | kubectl apply -f - >/dev/null; }
tam() { k run "$1" --image=busybox:1.37 --restart=Never \
  --overrides='{"apiVersion":"v1","spec":{"nodeSelector":{"kubernetes.io/hostname":"lab-k8s-worker2"},"terminationGracePeriodSeconds":1}}' \
  -- sleep 600; }
no_job_running() { ! k get pods -l "$SEL" --field-selector=status.phase=Running \
  -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].kind}{"\n"}{end}' | grep -qx Job; }

# Kịch bản A — LimitRange bị xóa nhầm
k delete limitrange bookstore-limits >/dev/null
tam drill-tam > "$EV/b6-a-tam.txt" 2>&1; cat "$EV/b6-a-tam.txt"
{ grep -q 'failed quota: bookstore-quota' "$EV/b6-a-tam.txt" && grep -q 'must specify' "$EV/b6-a-tam.txt" && \
  ! grep -q 'exceeded quota' "$EV/b6-a-tam.txt" && ! k get pod drill-tam >/dev/null 2>&1; } \
  && echo 'PASS: Pod tam bi tu choi kieu thieu khai bao, khong Pod nao duoc tao'
k create job drill-backup --from=cronjob/db-backup >/dev/null
k wait --for=condition=Complete job/drill-backup --timeout=300s >/dev/null \
  && echo 'PASS: workload cua du an van chay khi khong co LimitRange'
k delete job drill-backup --wait=true >/dev/null

# Trả LimitRange, cùng Pod tạm giờ được nhận
render_apply
for i in $(seq 1 60); do no_job_running && break; sleep 5; done
U0="$(cpu_m "$(qv bookstore-quota used requests.cpu)")"
tam drill-tam > "$EV/b6-a-tam-2.txt" 2>&1
DR="$(cpu_m "$(lr defaultRequest.cpu)")"
U1="$(cpu_m "$(qv bookstore-quota used requests.cpu)")"
RQ="$(cpu_m "$(jp pod/drill-tam '{.spec.containers[0].resources.requests.cpu}')")"
echo "defaultRequest=${DR}m ; Pod nhan ${RQ}m ; used truoc=${U0}m sau=${U1}m" | tee "$EV/b6-a-dem.txt"
{ [ "$RQ" -eq "$DR" ] && [ $((U1 - U0)) -eq "$DR" ]; } \
  && echo 'PASS: LimitRange chen defaultRequest, quota cong dung gia tri do'
k delete pod drill-tam --grace-period=1 >/dev/null

# Kịch bản B — siết trần mỗi container dưới limit của db
DBL="$(cpu_m "$(jp pod/db-0 '{.spec.containers[?(@.name=="postgres")].resources.limits.cpu}')")"
DEFL="$(cpu_m "$(lr default.cpu)")"; T=$(( DBL - 100 ))
IDX=$(( $(jp limitrange/bookstore-limits '{range .spec.limits[*]}{.type}{"\n"}{end}' | grep -n -x Container | cut -d: -f1) - 1 ))
echo "limit cpu cua db=${DBL}m ; default limit=${DEFL}m ; tran se siet=${T}m" | tee "$EV/b6-b-truoc.txt"
UID0="$(jp pod/db-0 '{.metadata.uid}')"; RC0="$(jp pod/db-0 '{.status.containerStatuses[?(@.name=="postgres")].restartCount}')"
[ "$T" -ge "$DEFL" ] && k patch limitrange bookstore-limits --type=json \
  -p "[{\"op\":\"replace\",\"path\":\"/spec/limits/$IDX/max/cpu\",\"value\":\"${T}m\"}]" >/dev/null \
  && echo "PASS: LimitRange van hop le voi tran ${T}m"
sleep 10
{ [ "$(jp pod/db-0 '{.metadata.uid}')" = "$UID0" ] && [ "$(jp pod/db-0 '{.status.phase}')" = Running ] && \
  [ "$(jp pod/db-0 '{.status.containerStatuses[?(@.name=="postgres")].restartCount}')" = "$RC0" ]; } \
  && echo 'PASS: db-0 dang chay khong bi dung toi'

k delete pod db-0 --wait=true --timeout=180s >/dev/null
for i in $(seq 1 40); do ev StatefulSet db FailedCreate | grep -q 'maximum cpu usage per Container' && break; sleep 3; done
ev StatefulSet db FailedCreate > "$EV/b6-b-failedcreate.txt"; cat "$EV/b6-b-failedcreate.txt"
{ grep -q 'maximum cpu usage per Container' "$EV/b6-b-failedcreate.txt" && ! k get pod db-0 >/dev/null 2>&1; } \
  && echo 'PASS: StatefulSet khong tao lai duoc db-0 — FailedCreate chi dung rang buoc LimitRange'
k wait --for=condition=Ready=false pod -l "$SEL,app.kubernetes.io/component=api" --timeout=120s >/dev/null \
  && echo 'PASS: api mat Ready vi DB khong con'

# Khôi phục từ repo
render_apply
for i in $(seq 1 120); do k get pod db-0 >/dev/null 2>&1 && break; sleep 5; done
k wait --for=condition=Ready pod/db-0 --timeout=300s >/dev/null && echo 'PASS: db-0 len lai sau khi tra LimitRange'
N="$(k exec db-0 -c postgres -- psql -U postgres -d bookstore -tAc 'SELECT count(*) FROM api.books')"
{ [ "$(jp pod/db-0 '{.spec.nodeName}')" = lab-k8s-worker2 ] && [ "${N:-0}" -ge 8 ]; } \
  && echo "PASS: db-0 tren lab-k8s-worker2, du lieu con nguyen ($N cuon)"
k wait --for=condition=Ready pod -l "$SEL,app.kubernetes.io/component=api" --timeout=300s >/dev/null && echo 'PASS: api Ready lai'
{ ./scripts/deploy.sh > "$EV/b6-deploy.txt" 2>&1 && ./tests/smoke.sh > "$EV/b6-smoke.txt" 2>&1; } \
  && echo 'PASS: deploy.sh va smoke xanh'
ls docs/postmortem/7b-chinh-sach-siet-sai.md >/dev/null 2>&1 && echo 'PASS: co postmortem'
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính của postmortem</summary>

1. `failed quota: ...: must specify ...` — ResourceQuota, Pod **thiếu khai báo** tài nguyên mà quota
   đang theo dõi: sửa manifest, hoặc trả LimitRange về. `exceeded quota: ..., requested, used,
   limited` — ResourceQuota, namespace **hết chỗ**: dọn bớt hoặc xin nới quota, sửa manifest vô ích.
   `maximum ... usage per Container` — LimitRange, **một container** vượt trần: sửa manifest của
   container đó, thêm node hay nới quota vô ích.
2. LimitRange là admission **biến đổi** (mutating): nó chèn `requests`/`limits` vào object trước.
   ResourceQuota **kiểm** (validating) sau, trên object đã được chèn, nên cộng đúng giá trị mặc định
   vào `used` — bài 134 gọi đó là để "người dùng không phải nhớ tự làm việc đó".
3. Bài 133: LimitRange chỉ kiểm ở admission của Pod; Pod đã tồn tại chạy tiếp không đổi. Chính sách
   sai vì thế nằm im, không ai biết, cho tới lần Pod bị thay: drain ở Lab 12 sẽ đuổi `db-0` khỏi
   `lab-k8s-worker2` và StatefulSet không tạo lại được — một cuộc bảo trì thường biến thành sự cố
   mất DB.
4. Đổi chính sách qua repo và `deploy.sh`, rồi **ngay lập tức** phát hành lại có kiểm soát từng
   workload (`rollout restart` cho Deployment, StatefulSet) và chạy smoke — Pod mới đi qua admission
   mới, lỗi lộ ra khi người đổi còn ở đó. Không bao giờ xóa LimitRange như một bước "dọn dẹp".

</details>

## B7. [BS-7B-07] Điều tra: giới hạn PID trên node

**Loại:** Điều tra — chỉ đọc, không sửa node.

**Bối cảnh:** PostgreSQL tạo một tiến trình cho mỗi kết nối. Một lỗi ở `api` mở kết nối mà không
đóng, hay một script fork loạn trong Pod tạm, có thể ăn hết PID của `lab-k8s-worker2` — nơi có
`db-0` và dữ liệu cục bộ. LimitRange và quota vừa làm không có dòng nào cho PID. Đội cần biết cluster
đang bảo vệ tới đâu, trước khi đề xuất gì.

**Yêu cầu:**

1. Chứng minh không khai được giới hạn PID trong Pod spec: API server từ chối — dùng server dry-run,
   không tạo object.
2. Đọc `podPidsLimit` trên cả ba node ở hai nơi: file cấu hình kubelet, và giá trị hiệu lực của
   kubelet đang chạy (B0). Giải thích vì sao hai nơi có thể khác nhau.
3. Đọc `pids.max` ở cgroup **cấp Pod** trên node, cho Pod `node-agent` của từng node; đọc
   `pids.max` bên trong container và giải thích vì sao con số trong container **không** đủ để kết
   luận.
4. Đo số tiến trình hiện có trong Pod `db-0` và giới hạn PID của hệ điều hành trên
   `lab-k8s-worker2`.
5. Đọc condition `PIDPressure` của hai worker và ngưỡng eviction cứng của kubelet: có trục PID không.
6. Ghi một nợ kỹ thuật mới vào `docs/tech-debt.md`; ADR đề xuất mức giới hạn PID cho node của dự án.

**Ràng buộc:** không sửa file cấu hình kubelet, không restart kubelet, không chạy fork bomb. Lệnh qua
`ssh` chỉ đọc.

**Đọc lại:**

- Bài [135](../135-pid-limiting-vi.md): đoạn mở đầu — PID là tài nguyên nền tảng, khai trên kubelet
  chứ không trong `.spec` của Pod, ghi chú *Thận trọng*;
  [*Giới hạn PID của node*](../135-pid-limiting-vi.md#giới-hạn-pid-của-node-node-pid-limits),
  [*Giới hạn PID của Pod*](../135-pid-limiting-vi.md#giới-hạn-pid-của-pod-pod-pid-limits),
  [*Eviction dựa trên PID*](../135-pid-limiting-vi.md#eviction-dựa-trên-pid-pid-based-eviction).
- Bài [132](../132-policies-vi.md#áp-dụng-chính-sách-bằng-cấu-hình-kubelet-apply-policies-using-kubelet-configurations)
  — nhánh cấu hình kubelet.
- Bài [142](../142-node-pressure-eviction-vi.md#node-conditions) — condition `PIDPressure` (Lab 7a).

**Bàn giao:** `docs/tech-debt.md`; `docs/adr/NNNN-gioi-han-pid.md`; `~/lab-evidence/7b/b7-*.txt`.

**Nghiệm thu:**

- `PASS:` server dry-run một Pod có `limits.pids` bị từ chối là không hợp lệ, thông báo nhắc `pids`.
- `PASS:` giá trị hiệu lực của `podPidsLimit` là `-1` trên cả ba node.
- `PASS:` cgroup cấp Pod của `node-agent` trên mỗi node có `pids.max` là `max`.
- `PASS:` `db-0` đang giữ ít nhất 5 tiến trình, không có trần riêng; đọc được `pid_max` của
  `lab-k8s-worker2`.
- `PASS:` hai worker không có `PIDPressure`; không node nào đặt ngưỡng `pid.available`.
- `PASS:` `docs/tech-debt.md` có dòng nợ nhắc `podPidsLimit`; có ADR.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-gioi-han-pid.md`:

1. Vì sao `pids` không có trong `resources` của container? Câu *Thận trọng* của bài 135 — "giới hạn
   phụ thuộc nơi Pod được lập lịch" — nghĩa là gì với `db` đã ghim `lab-k8s-worker2`?
2. Đề xuất `podPidsLimit` cho node của dự án dựa trên số tiến trình đo được của `db-0`; đặt ở node
   nào, và vì sao bài 135 khuyên mọi node cùng một mức?
3. Giới hạn PID theo Pod có bảo vệ được kubelet và containerd không? Cơ chế nào bảo vệ chúng, và
   trên cluster của bạn nó đang bật không?
4. Vì sao ngưỡng eviction `pid.available` không thay được giới hạn cứng?
5. Nợ này trả ở đâu, bằng thao tác nào — và vì sao không trả ngay trong sprint?

<details>
<summary>Gợi ý</summary>

- Server dry-run một manifest: `kubectl create --dry-run=server -f <file>`.
- Giá trị hiệu lực: hàm `kz` trong `lib.sh` đọc file `configz` mà B0 đã lưu —
  `kz ~/lab-evidence/7b/b0-configz-<node>.json podPidsLimit`.
- kubelet đặt giới hạn PID lên cgroup **của Pod**, thư mục cha của các container. Bên trong container
  bạn chỉ thấy cgroup của chính container. Hàm `podcg` tìm thư mục cgroup của Pod trên node theo UID
  của Pod.
- Ngưỡng eviction: khóa `evictionHard` trong cấu hình hiệu lực.

</details>

<details>
<summary>Script nghiệm thu — chỉ đọc; chạy sau khi đã ghi nợ và ADR</summary>

```bash
. ~/lab-work/7b/lib.sh

# 1. pids không phải tài nguyên của Pod spec
cat <<'Y' | kubectl create --dry-run=server -f - > "$EV/b7-pids-spec.txt" 2>&1
apiVersion: v1
kind: Pod
metadata: {name: pid-thu, namespace: bookstore}
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: [sleep, "60"]
    resources:
      requests: {cpu: 10m, memory: 16Mi}
      limits: {cpu: 50m, memory: 32Mi, pids: "100"}
Y
cat "$EV/b7-pids-spec.txt"
{ grep -qi invalid "$EV/b7-pids-spec.txt" && grep -q pids "$EV/b7-pids-spec.txt"; } \
  && echo 'PASS: API server tu choi pids trong resources — gioi han PID khong khai o Pod spec'

# 2. podPidsLimit: file và giá trị hiệu lực
{ echo "lab-k8s-master file: $(sudo grep -i podPidsLimit /var/lib/kubelet/config.yaml || echo '<khong khai>')"
  for n in lab-k8s-worker1 lab-k8s-worker2; do
    echo "$n file: $(ssh "$n" 'sudo grep -i podPidsLimit /var/lib/kubelet/config.yaml' || echo '<khong khai>')"
  done
  for n in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
    echo "$n hieu luc: podPidsLimit=$(kz "$EV/b0-configz-$n.json" podPidsLimit)"
  done; } | tee "$EV/b7-pid.txt"
[ "$(grep -c 'hieu luc: podPidsLimit=-1$' "$EV/b7-pid.txt")" -eq 3 ] \
  && echo 'PASS: ca ba kubelet chay voi podPidsLimit=-1 — khong co tran PID cho Pod'

# 3. cgroup cấp Pod so với trong container — Pod node-agent trên từng node
BAD=0
for L in $(kubectl -n "$NS" get pods -l "$SEL,app.kubernetes.io/component=agent" \
             -o jsonpath='{range .items[*]}{.spec.nodeName}={.metadata.uid}={.metadata.name}{"\n"}{end}'); do
  N="${L%%=*}"; R="${L#*=}"; U="${R%%=*}"; P="${R#*=}"
  PM="$(podcg "$N" "$U" pids.max)"; CM="$(k exec "$P" -c agent -- cat /sys/fs/cgroup/pids.max)"
  echo "$N $P: pids.max cap Pod=$PM ; trong container=$CM" | tee -a "$EV/b7-pid.txt"
  [ "$PM" = max ] || BAD=1
done
[ "$BAD" -eq 0 ] && echo 'PASS: cgroup cap Pod cua node-agent tren moi node khong co tran PID'

# 4. db-0 và giới hạn của hệ điều hành
DU="$(jp pod/db-0 '{.metadata.uid}')"
DC="$(podcg lab-k8s-worker2 "$DU" pids.current)"; DM="$(podcg lab-k8s-worker2 "$DU" pids.max)"
KPM="$(ssh lab-k8s-worker2 'cat /proc/sys/kernel/pid_max')"
echo "db-0: pids.current=$DC pids.max=$DM ; kernel pid_max worker2=$KPM" | tee -a "$EV/b7-pid.txt"
{ [ "${DC:-0}" -ge 5 ] && [ "$DM" = max ] && [ "${KPM:-0}" -gt 0 ]; } \
  && echo "PASS: db-0 giu $DC tien trinh, khong tran rieng — chi co tran $KPM cua ca node"

# 5. PIDPressure và ngưỡng eviction
PP=0
for n in lab-k8s-worker1 lab-k8s-worker2; do
  [ "$(kubectl get node "$n" -o jsonpath='{.status.conditions[?(@.type=="PIDPressure")].status}')" = False ] && PP=$((PP + 1))
done
[ "$PP" -eq 2 ] && echo 'PASS: hai worker khong chiu PIDPressure'
EH=0
for n in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  E="$(kz "$EV/b0-configz-$n.json" evictionHard)"; echo "$n evictionHard=$E" >> "$EV/b7-pid.txt"
  echo "$E" | grep -q 'pid.available' || EH=$((EH + 1))
done
[ "$EH" -eq 3 ] && echo 'PASS: khong node nao dat nguong eviction pid.available'

# 6. Nợ và ADR
cd ~/bookstore
grep -q 'podPidsLimit' docs/tech-debt.md && echo 'PASS: tech-debt.md ghi no gioi han PID'
ls docs/adr/*-gioi-han-pid.md >/dev/null 2>&1 && echo 'PASS: co ADR gioi han PID'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Dòng thêm vào `docs/tech-debt.md` — mã `TD-13`, cố định theo [mục 8 của file dự án](DU-AN-BOOKSTORE.md#8-sổ-nợ-kỹ-thuật-dự-án):

```markdown
| TD-13 | Kubelet chưa giới hạn PID cho Pod (podPidsLimit = -1) và chưa dự trữ PID cho node: một Pod fork loạn hay rò kết nối DB có thể làm mất ổn định worker | 7b | giai đoạn 20 (cấu hình lại kubelet); đề xuất ghi vào cấu hình kubelet trong platform/kubeadm/ ở Lab 8a | mở | ~/lab-evidence/7b/b7-pid.txt |
```

Ý chính cho ADR:

1. Bài 135: giới hạn PID cấu hình **trên kubelet** (`--pod-max-pids` hoặc `PodPidsLimit` trong file
   cấu hình), giới hạn ở cấp Pod chưa được hỗ trợ. Mỗi node một mức nên cùng một Pod có thể chịu mức
   khác nhau tùy node. `db` ghim `lab-k8s-worker2`, nên **mức của worker2** là mức thật của DB — đặt
   chỗ khác không bảo vệ nó; và nếu sau này `db` được dời node (bảo trì), mức đổi theo.
2. Ví dụ: `db-0` lúc nhàn giữ khoảng chục tiến trình; PostgreSQL mặc định nhận tối đa 100 kết nối,
   mỗi kết nối một tiến trình — vài trăm là trần hợp lý cho Pod lớn nhất. Đề xuất `podPidsLimit`
   1024 trên **cả ba node**: dư cho `db`, thấp xa `pid_max` của node. Bài 135: mọi node cùng mức thì
   giới hạn không phụ thuộc nơi Pod rơi vào.
3. Không. Bài 135: giới hạn theo Pod chỉ bảo vệ Pod này khỏi Pod khác, không bảo vệ agent của node.
   Thứ bảo vệ kubelet, containerd là **dự trữ PID cho node**: `pid=<number>` trong
   `--system-reserved`, `--kube-reserved`. Cluster lab không đặt.
4. Tín hiệu eviction được tính **định kỳ** và **không cưỡng chế** giới hạn: PID tăng nhanh giữa hai
   lần đo thì node đã mất ổn định trước khi kubelet kịp đuổi Pod nào.
5. Trả bằng việc sửa cấu hình kubelet và restart kubelet trên từng node — thao tác của giai đoạn 20,
   ngoài phạm vi sprint (sprint này cấm sửa node). Lab 8a viết cấu hình kubeadm vào `platform/`: đó là
   chỗ ghi mức đề xuất để cluster dựng lại từ repo có giới hạn ngay từ đầu.

</details>

## B8. [BS-7B-08] Điều tra: resource manager và tính năng do node khai báo

**Loại:** Điều tra — chỉ đọc; phần ⭐ tùy chọn.

**Bối cảnh:** có đề xuất "cho `db` CPU riêng để truy vấn bớt chậm giờ cao điểm". Trước khi đổi gì
trên node, đội đọc xem kubelet đang làm gì với CPU của `db-0`, và đề xuất đó cần điều kiện gì. Cùng
lúc, rà một tính năng mới của bản sau — tính năng do node khai báo — để biết cluster có dùng chưa.

**Yêu cầu:**

1. Đọc policy hiệu lực của CPU manager, memory manager, topology manager (và scope) trên hai worker.
2. Với `db-0`: QoS, CPU request; tập CPU mà tiến trình PostgreSQL được phép chạy; cơ chế cưỡng chế
   limit CPU của container. Giải thích vì sao `db-0` không có CPU riêng — và vì sao kể cả khi đổi
   sang `static` nó vẫn không có.
3. Chứng minh các policy này không nằm trong object Node.
4. Đếm NUMA node của `lab-k8s-worker2`; giải thích vì sao topology manager không có gì để căn chỉnh.
5. Chứng minh không Node nào của cluster khai `.status.declaredFeatures`.
6. ⭐ Một Pod `Guaranteed` xin đúng 1 CPU nguyên trong namespace thử nghiệm `bookstore-thu`: chứng
   minh nó vẫn ở pool dùng chung dưới policy hiện tại. Xóa namespace khi xong.
7. ADR: có nên cấp CPU độc quyền cho `db` không.

**Ràng buộc:** không đổi policy, không restart kubelet, không xóa state file trong
`/var/lib/kubelet/`. Pod ⭐ chạy ở node còn đủ một CPU chưa cấp phát — `lab-k8s-worker1` trong lời
giải, vì `db-0` đã giữ nửa CPU của `lab-k8s-worker2`; Pod này chỉ để đọc, không phải diễn tập.

**Đọc lại:**

- Bài [74](../74-resource-managers-vi.md):
  [*Trình quản lý topology*](../74-resource-managers-vi.md#trình-quản-lý-topology-topology-manager),
  [*Các chính sách gán CPU cho Pod*](../74-resource-managers-vi.md#các-chính-sách-gán-cpu-cho-pod-policies-for-assigning-cpus-to-pods),
  [*Chính sách static*](../74-resource-managers-vi.md#chính-sách-static-static-policy) — sáu ví dụ
  YAML, [*Các chính sách gán memory cho Pod*](../74-resource-managers-vi.md#memory-management-policies).
- Bài [54](../54-pod-qos-vi.md#guaranteed) — QoS `Guaranteed`.
- Bài [154](../154-node-declared-features-vi.md):
  [*Cách hoạt động*](../154-node-declared-features-vi.md#cách-hoạt-động-how-it-works),
  [*Bật tính năng do node khai báo*](../154-node-declared-features-vi.md#bật-tính-năng-do-node-khai-báo-enabling-node-declared-features).

**Bàn giao:** `docs/adr/NNNN-resource-manager.md`; `~/lab-evidence/7b/b8-*.txt`.

**Nghiệm thu:**

- `PASS:` hai worker chạy `cpuManagerPolicy: none`, `memoryManagerPolicy: None`,
  `topologyManagerPolicy: none`, scope `container`.
- `PASS:` `db-0` là `Guaranteed`; PostgreSQL được chạy trên mọi CPU của node; limit CPU cưỡng chế
  bằng CFS quota (`cpu.max` không phải `max`).
- `PASS:` object Node không mang tên policy nào của kubelet.
- `PASS:` `lab-k8s-worker2` có đúng một NUMA node.
- `PASS:` không Node nào khai `.status.declaredFeatures`.
- `PASS:` (⭐) Pod `Guaranteed` 1 CPU vẫn thấy mọi CPU của node; namespace thử nghiệm đã xóa.
- `PASS:` có ADR.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-resource-manager.md`:

1. Ba điều kiện để một container được CPU độc quyền. `db-0` thiếu điều kiện nào?
2. Worker có 2 vCPU. Bật `static` thì kubelet đòi dự trữ CPU lớn hơn 0 — còn bao nhiêu CPU cấp độc
   quyền được, và `web`, `api` trên cùng node trả giá gì?
3. `cpu.max` của `db-0` nói gì về cách giới hạn 500m được cưỡng chế? Dưới `static`, với một container
   được CPU độc quyền, cơ chế đó còn dùng không?
4. Tính năng do node khai báo giải quyết vấn đề gì, và khi nào cluster của bạn sẽ gặp vấn đề đó?

<details>
<summary>Gợi ý</summary>

- Policy hiệu lực: hàm `kz` với các khóa `cpuManagerPolicy`, `memoryManagerPolicy`,
  `topologyManagerPolicy`, `topologyManagerScope`.
- Tập CPU của một tiến trình: dòng `Cpus_allowed_list` trong `/proc/<pid>/status`. Tiến trình 1 của
  container `postgres` chính là PostgreSQL.
- Giới hạn CPU của container: `/sys/fs/cgroup/cpu.max` bên trong container — hai số là quota và chu
  kỳ, tính bằng micro giây.
- NUMA node: `ls -d /sys/devices/system/node/node*` trên worker.

</details>

<details>
<summary>Script nghiệm thu — chỉ đọc; phần ⭐ tạo rồi xóa namespace thử nghiệm</summary>

```bash
. ~/lab-work/7b/lib.sh
: > "$EV/b8-policy.txt"
OK=0
for n in lab-k8s-worker1 lab-k8s-worker2; do
  F="$EV/b0-configz-$n.json"
  L="cpuManager=$(kz "$F" cpuManagerPolicy) memoryManager=$(kz "$F" memoryManagerPolicy) topologyManager=$(kz "$F" topologyManagerPolicy) scope=$(kz "$F" topologyManagerScope)"
  echo "$n $L" | tee -a "$EV/b8-policy.txt"
  [ "$(echo "$L" | tr 'A-Z' 'a-z')" = 'cpumanager=none memorymanager=none topologymanager=none scope=container' ] && OK=$((OK + 1))
done
[ "$OK" -eq 2 ] && echo 'PASS: hai worker chay policy mac dinh cua ca ba trinh quan ly'

NCPU="$(kubectl get node lab-k8s-worker2 -o jsonpath='{.status.capacity.cpu}')"
if [ "$NCPU" -eq 1 ]; then EXP=0; else EXP="0-$((NCPU - 1))"; fi
QOS="$(jp pod/db-0 '{.status.qosClass}')"
RC="$(jp pod/db-0 '{.spec.containers[?(@.name=="postgres")].resources.requests.cpu}')"
ALW="$(k exec db-0 -c postgres -- cat /proc/1/status | awk '/^Cpus_allowed_list/ {print $2}')"
CMX="$(k exec db-0 -c postgres -- cat /sys/fs/cgroup/cpu.max)"
echo "db-0 qos=$QOS requests.cpu=$RC Cpus_allowed_list=$ALW cpu.max=$CMX" | tee "$EV/b8-db.txt"
{ [ "$QOS" = Guaranteed ] && [ "$ALW" = "$EXP" ] && [ "$CMX" != max ]; } \
  && echo "PASS: db-0 Guaranteed nhung chay tren moi CPU ($EXP), limit cuong che bang CFS quota ($CMX)"
[ $(( $(cpu_m "$RC") % 1000 )) -ne 0 ] \
  && echo "GHI CHU: requests.cpu=$RC khong nguyen — khong du dieu kien CPU doc quyen ke ca duoi static"

kubectl get node lab-k8s-worker2 -o yaml > "$EV/b8-node.yaml"
grep -qiE 'cpuManagerPolicy|memoryManagerPolicy|topologyManagerPolicy|podPidsLimit' "$EV/b8-node.yaml" \
  && echo 'FAIL: Node mang chinh sach kubelet' || echo 'PASS: object Node khong mang chinh sach cua kubelet'
NUMA="$(ssh lab-k8s-worker2 'ls -d /sys/devices/system/node/node[0-9]* | wc -l')"
[ "$NUMA" -eq 1 ] && echo 'PASS: worker2 chi co mot NUMA node — topology manager khong co ranh gioi nao de can'
DF=0
for n in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  DF=$(( DF + $(kubectl get node "$n" -o json | grep -c declaredFeatures) ))
done
[ "$DF" -eq 0 ] && echo 'PASS: khong Node nao khai .status.declaredFeatures tren baseline'

# ⭐ Pod Guaranteed xin đúng 1 CPU
cat <<'Y' | kubectl apply -f - >/dev/null
apiVersion: v1
kind: Namespace
metadata:
  name: bookstore-thu
  labels: {app.kubernetes.io/part-of: bookstore}
---
apiVersion: v1
kind: Pod
metadata: {name: cpu-nguyen, namespace: bookstore-thu}
spec:
  nodeSelector: {kubernetes.io/hostname: lab-k8s-worker1}
  terminationGracePeriodSeconds: 1
  containers:
  - name: app
    image: busybox:1.37
    command: [sleep, "600"]
    resources:
      requests: {cpu: "1", memory: 64Mi}
      limits: {cpu: "1", memory: 64Mi}
Y
if kubectl -n bookstore-thu wait --for=condition=Ready pod/cpu-nguyen --timeout=180s >/dev/null; then
  N1="$(kubectl get node lab-k8s-worker1 -o jsonpath='{.status.capacity.cpu}')"
  A="$(kubectl -n bookstore-thu exec cpu-nguyen -- awk '/^Cpus_allowed_list/ {print $2}' /proc/self/status)"
  echo "cpu-nguyen qos=$(kubectl -n bookstore-thu get pod cpu-nguyen -o jsonpath='{.status.qosClass}') Cpus_allowed_list=$A" | tee -a "$EV/b8-db.txt"
  [ "$A" = "0-$((N1 - 1))" ] && echo 'PASS: (sao) Pod Guaranteed 1 CPU nguyen van o pool dung chung duoi policy none'
else
  echo 'GHI CHU: cpu-nguyen chua Ready — xem muc 4 (worker khong con du 1 CPU chua cap phat)'
fi
kubectl delete namespace bookstore-thu --wait=true >/dev/null && echo 'PASS: (sao) da xoa namespace thu nghiem'
ls ~/bookstore/docs/adr/*-resource-manager.md >/dev/null 2>&1 && echo 'PASS: co ADR resource manager'
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính của ADR</summary>

1. Ba điều kiện của bài 74: CPU manager chạy chính sách `static`; Pod thuộc QoS `Guaranteed`;
   CPU `requests` của container là **số nguyên**. `db-0` thỏa điều thứ hai nhưng thiếu điều thứ nhất
   (policy là `none`) **và** điều thứ ba (500m là số lẻ) — giống ví dụ `cpu: "1.5"` của bài: Guaranteed
   vẫn chạy trong pool dùng chung. Đổi policy thôi là chưa đủ.
2. Bài 74: số CPU cấp độc quyền = tổng CPU trừ phần dự trữ, và dự trữ phải lớn hơn 0 để pool dùng
   chung không rỗng. Trên 2 vCPU, dự trữ một CPU thì chỉ còn đúng một CPU độc quyền; nếu `db` lấy nó,
   mọi Pod khác trên `lab-k8s-worker2` — `web`, `api`, `node-agent`, các Pod hệ thống — dồn vào CPU
   còn lại. Đề xuất bị từ chối: lợi ích nhỏ, cái giá lớn; xem lại khi có node vật lý nhiều core.
3. `cpu.max` dạng `50000 100000` nghĩa là được dùng 50 ms CPU mỗi chu kỳ 100 ms — CFS quota, đúng
   cơ chế mặc định bài 74 mô tả cho policy `none`. Với container được cấp CPU độc quyền dưới
   `static`, CFS quota không dùng nữa: số CPU trong cpuset đã giới hạn mức dùng.
4. Bài 154: tính năng do node khai báo xử lý **chênh lệch phiên bản** giữa các node — kubelet báo
   tính năng đang bật, scheduler loại node thiếu tính năng Pod cần. Cluster ba node cùng phiên bản
   chưa gặp; nó xuất hiện khi nâng cấp từng node một
   ([giai đoạn 17](../00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster)), và tính năng này ra
   sau baseline của bạn.

</details>

## B9. [BS-7B-09] ⭐ Quota riêng cho tác vụ nền theo PriorityClass

**Loại:** Điều tra — tùy chọn.

**Bối cảnh:** từ Lab 7a, Job và CronJob của dự án chạy với PriorityClass `bookstore-batch`. Bài 141
mở đầu bằng cảnh báo về PriorityClass và ResourceQuota, và hẹn tới nhóm 7b. Câu hỏi của đội: có nên
cho tác vụ nền một trần riêng, để một loạt Job không ăn vào phần dự phòng mà `web` cần khi phát hành?

**Yêu cầu:** một ResourceQuota thử nghiệm `batch-thu` trong `bookstore` chỉ đếm Pod dùng
`bookstore-batch`, trần một Pod. Chứng minh: Pod `web`, `api`, `db`, `node-agent` không bị nó đếm;
Pod batch đầu tiên được tạo và bị đếm; Pod batch thứ hai bị từ chối và Job ghi `FailedCreate`. Ghi ADR
có đưa quota này vào repo không. Xóa quota và Job thử nghiệm.

**Ràng buộc:** không thêm `batch-thu` vào repo trong sprint này — nếu ADR chọn "có", ghi thành đề
xuất cho sprint sau. Không dùng cờ `--admission-control-config-file` (giai đoạn 20).

**Đọc lại:** bài [134](../134-resource-quotas-vi.md): [*Phạm vi hạn ngạch*](../134-resource-quotas-vi.md#quota-scopes),
[*Phạm vi PriorityClass*](../134-resource-quotas-vi.md#resource-quota-per-priorityclass) và ví dụ ba
quota `pods-high`, `pods-medium`, `pods-low`; bài [141](../141-pod-priority-preemption-vi.md) — cảnh
báo đầu bài, mục [*PriorityClass*](../141-pod-priority-preemption-vi.md#priorityclass).

**Bàn giao:** `~/lab-work/7b/quota-batch.yaml`; `docs/adr/NNNN-quota-priorityclass.md`;
`~/lab-evidence/7b/b9-*.txt`.

**Nghiệm thu:**

- `PASS:` template CronJob `db-backup` dùng `bookstore-batch`.
- `PASS:` `batch-thu` đếm 0 Pod dù namespace đang chạy các Pod của dự án.
- `PASS:` Pod batch đầu tiên được đếm; Pod batch thứ hai bị `batch-thu` từ chối, Job ghi
  `FailedCreate`.
- `PASS:` đã xóa `batch-thu` và Job thử nghiệm; có ADR.

<details>
<summary>Script nghiệm thu và lời giải tham khảo</summary>

```yaml
# ~/lab-work/7b/quota-batch.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: batch-thu
  namespace: bookstore
spec:
  hard:
    pods: "1"
    requests.cpu: 200m
    requests.memory: 256Mi
  scopeSelector:
    matchExpressions:
    - scopeName: PriorityClass
      operator: In
      values: ["bookstore-batch"]
```

```bash
. ~/lab-work/7b/lib.sh
[ "$(jp cronjob/db-backup '{.spec.jobTemplate.spec.template.spec.priorityClassName}')" = bookstore-batch ] \
  && echo 'PASS: tac vu nen cua du an dung bookstore-batch'
kubectl apply -f "$W/quota-batch.yaml" >/dev/null
for i in $(seq 1 30); do [ -n "$(qv batch-thu used pods)" ] && break; sleep 2; done
{ [ "$(qv batch-thu used pods)" = 0 ] && [ "$(qv bookstore-quota used pods)" -ge 9 ]; } \
  && echo 'PASS: batch-thu dem 0 Pod — web, api, db, node-agent nam ngoai pham vi'
bj() { printf 'apiVersion: batch/v1\nkind: Job\nmetadata: {name: %s, namespace: bookstore, labels: {lab: 7b-batch}}\nspec:\n  backoffLimit: 0\n  template:\n    metadata: {labels: {lab: 7b-batch}}\n    spec:\n      priorityClassName: bookstore-batch\n      restartPolicy: Never\n      terminationGracePeriodSeconds: 1\n      nodeSelector: {kubernetes.io/hostname: lab-k8s-worker2}\n      containers:\n      - {name: x, image: "busybox:1.37", command: [sleep, "120"], resources: {requests: {cpu: 10m, memory: 16Mi}, limits: {cpu: 50m, memory: 32Mi}}}\n' "$1"; }
bj batch-a | kubectl apply -f - >/dev/null
for i in $(seq 1 30); do [ "$(qv batch-thu used pods)" = 1 ] && break; sleep 2; done
[ "$(qv batch-thu used pods)" = 1 ] && echo 'PASS: Pod batch dau tien duoc tao va bi dem'
bj batch-b | kubectl apply -f - >/dev/null
for i in $(seq 1 30); do [ -n "$(ev Job batch-b FailedCreate)" ] && break; sleep 2; done
ev Job batch-b FailedCreate | tee "$EV/b9-batch-b.txt" | grep -q 'exceeded quota: batch-thu' \
  && echo 'PASS: Pod batch thu hai bi batch-thu tu choi — Job ghi FailedCreate'
k delete job -l lab=7b-batch --wait=true >/dev/null
kubectl delete -f "$W/quota-batch.yaml" >/dev/null
{ ! k get resourcequota batch-thu >/dev/null 2>&1 && ls ~/bookstore/docs/adr/*-quota-priorityclass.md >/dev/null 2>&1; } \
  && echo 'PASS: da xoa batch-thu va Job thu nghiem; co ADR'
```

Ý chính cho ADR: quota có phạm vi chỉ khớp Pod mang đúng PriorityClass trong `scopeSelector` (bài
134), nên trần của tác vụ nền tách khỏi phần của `web`, `api` — một loạt Job hỏng không thể chiếm
phần dự phòng rollout. Cái giá: thêm một con số phải giữ đúng; trần quá chặt thì backup và migration
chờ nhau. Với hai tác vụ nền nhỏ hiện tại, `count/jobs.batch` của B4 đã đủ; đề xuất xem lại ở Lab 9b
khi có tenant thứ hai. Cảnh báo của bài 141 — người dùng tự gán PriorityClass cao cho Pod của mình —
cần cơ chế `limitedResources` qua cờ của kube-apiserver, thuộc giai đoạn 20.

</details>

## B10. Đóng sprint: cleanup và gate cuối

**Mục đích:** bàn giao một repo dựng lại được toàn bộ hệ thống của sprint 11 kèm hạn mức namespace,
chứng minh cấu hình node không bị đụng, rồi trả cluster về đúng `03-storage-ready`.

1. Thay `tests/smoke.sh` bằng phiên bản của Lab 7b — kiểm thử khói **tích lũy**, chép nguyên văn:

```bash
cat > ~/bookstore/tests/smoke.sh <<'EOF'
#!/usr/bin/env bash
# Kiểm thử khói — trạng thái sau Lab 7b (sprint 11). Kiểm mọi hợp đồng còn hiệu lực tới sprint 11:
# controller (4a, 4b), cấu hình và bí mật (3b, 5a), Service và DNS (5a), NetworkPolicy và Ingress
# (5b), dữ liệu bền (6a), đặt chỗ và độ ưu tiên (7a), hạn mức namespace (7b).
set -u
NS=bookstore
SEL='app.kubernetes.io/part-of=bookstore'
FAIL=0
pass() { echo "PASS: $*"; }
fail() { echo "FAIL: $*"; FAIL=1; }
k()  { kubectl -n "$NS" "$@"; }
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
cpu_m() { awk -v q="$1" 'BEGIN { if (q == "") { print 0; exit }
  if (q ~ /m$/) { sub(/m$/, "", q); printf "%d\n", q; exit }
  printf "%d\n", q * 1000 }'; }
mem_mi() { awk -v q="$1" 'BEGIN { if (q == "") { print 0; exit }
  n = q; u = q; sub(/[A-Za-z]+$/, "", n); sub(/^[0-9.]+/, "", u)
  f["Ki"] = 1/1024; f["Mi"] = 1; f["Gi"] = 1024; f["Ti"] = 1048576; f[""] = 1/1048576
  f["k"] = 1000/1048576; f["M"] = 1000000/1048576; f["G"] = 1000000000/1048576
  printf "%d\n", n * f[u] }'; }
qv() { k get resourcequota bookstore-quota -o go-template="{{with .status.$1}}{{with index . \"$2\"}}{{.}}{{end}}{{end}}" 2>/dev/null; }
cm_ok() { local n; for n in $(jp "$1" "$2"); do
  case "$n" in "$3"*) k get configmap "$n" >/dev/null 2>&1 && return 0 ;; esac; done; return 1; }

# 1. Namespace và object cấp cluster (3a, 6a, 7a)
[ "$(kubectl get namespace "$NS" -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}' 2>/dev/null)" = bookstore ] \
  && pass 'namespace bookstore mang label du an' || fail 'namespace bookstore thieu hoac thieu label'
CRIT="$(kubectl get priorityclass bookstore-critical -o jsonpath='{.value}' 2>/dev/null)"
BATCH="$(kubectl get priorityclass bookstore-batch -o jsonpath='{.value}' 2>/dev/null)"
{ [ -n "$CRIT" ] && [ -n "$BATCH" ] && [ "$BATCH" -lt "$CRIT" ]; } \
  && pass "PriorityClass bookstore-critical ($CRIT) tren bookstore-batch ($BATCH)" || fail 'thieu PriorityClass hoac sai thu tu'
[ "$(kubectl get storageclass bookstore-retain -o jsonpath='{.reclaimPolicy}' 2>/dev/null)" = Retain ] \
  && pass 'StorageClass bookstore-retain giu du lieu (Retain)' || fail 'thieu StorageClass bookstore-retain'
[ "$(kubectl get node lab-k8s-worker2 -o jsonpath='{.metadata.labels.bookstore\.example\.com/data}' 2>/dev/null)" = true ] \
  && pass 'lab-k8s-worker2 mang label bookstore.example.com/data=true' || fail 'thieu label du lieu tren lab-k8s-worker2'

# 2. Hạn mức namespace (7b)
[ "$(k get limitrange -o name 2>/dev/null)" = 'limitrange/bookstore-limits' ] \
  && pass 'dung mot LimitRange: bookstore-limits' || fail 'LimitRange khong dung bookstore-limits duy nhat'
{ [ "$(jp limitrange/bookstore-limits '{.metadata.labels.app\.kubernetes\.io/part-of}')" = bookstore ] && \
  [ "$(jp resourcequota/bookstore-quota '{.metadata.labels.app\.kubernetes\.io/part-of}')" = bookstore ]; } \
  && pass 'bookstore-limits va bookstore-quota mang label du an' || fail 'thieu bookstore-limits/bookstore-quota hoac thieu label'
BAD=0
for K in requests.cpu requests.memory limits.cpu limits.memory pods services.nodeports persistentvolumeclaims count/jobs.batch; do
  H="$(qv hard "$K")"; U="$(qv used "$K")"
  case "$K" in *cpu) h=$(cpu_m "$H"); u=$(cpu_m "$U") ;; *memory) h=$(mem_mi "$H"); u=$(mem_mi "$U") ;; *) h="$H"; u="${U:-0}" ;; esac
  if [ -z "$H" ]; then echo "FAIL: bookstore-quota thieu tran $K"; BAD=1
  elif [ "$u" -gt "$h" ]; then echo "FAIL: $K used=$U vuot hard=$H"; BAD=1; fi
done
[ "$BAD" -eq 0 ] && pass 'bookstore-quota du tran tinh toan va so luong object, used duoi hard' || FAIL=1
[ "$(qv hard services.nodeports)" = 0 ] && pass 'services.nodeports = 0 — vao tu ngoai chi qua Ingress' || fail 'services.nodeports khac 0'

# 3. Cấu hình, bí mật, PDB (3b, 3c, 5a, 5b)
cm_ok deployment/web '{..configMap.name}' web-config && pass 'web dung ConfigMap web-config*' || fail 'web khong dung web-config*'
cm_ok deployment/api '{..configMapRef.name}' api-config && pass 'api nap ConfigMap api-config*' || fail 'api khong nap api-config*'
cm_ok statefulset/db '{..configMap.name}' db-init && pass 'db dung ConfigMap db-init*' || fail 'db khong dung db-init*'
D="$(jp secret/db-credentials '{.data}')"
{ echo "$D" | grep -q '"postgres-password"' && echo "$D" | grep -q '"authenticator-password"'; } \
  && pass 'Secret db-credentials du hai key' || fail 'Secret db-credentials thieu key'
[ "$(jp secret/bookstore-tls '{.type}')" = kubernetes.io/tls ] && pass 'Secret bookstore-tls loai TLS' || fail 'thieu Secret bookstore-tls'
for P in web api; do k get pdb "$P" >/dev/null 2>&1 && pass "co PodDisruptionBudget $P" || fail "thieu PodDisruptionBudget $P"; done

# 4. Controller, đặt chỗ, độ ưu tiên (4a, 4b, 7a)
{ [ "$(jp statefulset/db '{.status.readyReplicas}')" = 1 ] && [ "$(jp statefulset/db '{.spec.serviceName}')" = db ]; } \
  && pass 'StatefulSet db 1/1 Ready, serviceName db' || fail 'StatefulSet db chua Ready hoac sai serviceName'
{ [ "$(jp pod/db-0 '{.spec.nodeName}')" = lab-k8s-worker2 ] && \
  jp statefulset/db '{.spec.template.spec.affinity.nodeAffinity}' | grep -q 'bookstore.example.com/data'; } \
  && pass 'db-0 tren lab-k8s-worker2 theo node affinity cua du an' || fail 'db-0 sai cho hoac khong dung node affinity'
{ [ "$(jp pod/db-0 '{.status.qosClass}')" = Guaranteed ] && [ "$(jp pod/db-0 '{.spec.priority}')" = "$CRIT" ]; } \
  && pass 'db-0 Guaranteed, bookstore-critical' || fail 'db-0 sai QoS hoac priority'
for X in web:3 api:2; do
  N="${X%%:*}"; WANT="${X##*:}"
  [ "$(jp "deployment/$N" '{.status.readyReplicas}')" = "$WANT" ] && pass "Deployment $N $WANT/$WANT Ready" \
    || fail "Deployment $N chua du $WANT ban sao Ready"
done
[ "$(k get pods -l "$SEL,app.kubernetes.io/component=api" -o jsonpath='{.items[0].spec.priority}' 2>/dev/null)" = "$CRIT" ] \
  && pass 'api dung bookstore-critical' || fail 'api sai priority'
WN="$(k get pods -l "$SEL,app.kubernetes.io/component=web" -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' 2>/dev/null | sort -u | grep -c .)"
[ "$WN" -ge 2 ] && pass "web trai tren $WN node" || fail 'web don ve mot node'
[ "$(jp cronjob/db-backup '{.spec.jobTemplate.spec.template.spec.priorityClassName}')" = bookstore-batch ] \
  && pass 'db-backup dung bookstore-batch' || fail 'db-backup khong dung bookstore-batch'
ORPHAN="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{.metadata.name}:{.metadata.ownerReferences[0].kind}{"\n"}{end}' 2>/dev/null | grep -c ':$')"
[ "$ORPHAN" -eq 0 ] && pass 'moi Pod cua du an deu co controller' || fail "$ORPHAN Pod tran"

# 5. Tài nguyên khai tường minh (3c, 7b)
F='{.name}={.resources.requests.cpu},{.resources.requests.memory},{.resources.limits.cpu},{.resources.limits.memory}{"\n"}'
MISS="$(k get pods -l "$SEL" -o jsonpath="{range .items[*]}{range .spec.initContainers[*]}$F{end}{range .spec.containers[*]}$F{end}{end}" 2>/dev/null | grep -E '=,|,,|,$' | sort -u)"
[ -z "$MISS" ] && pass 'moi container, ca init va sidecar, khai du request va limit' || fail "container thieu tai nguyen: $(echo $MISS)"
LRP="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{.metadata.name}={.metadata.annotations.kubernetes\.io/limit-ranger}{"\n"}{end}' 2>/dev/null | grep -v '=$')"
[ -z "$LRP" ] && pass 'khong Pod du an nao phai nho gia tri mac dinh cua LimitRange' || fail "Pod nhan mac dinh: $(echo $LRP)"

# 6. Service (5a)
[ "$(jp service/web '{.spec.type}:{.spec.ports[0].port}:{.spec.ports[0].targetPort}')" = ClusterIP:80:http ] \
  && pass 'Service web ClusterIP 80 -> http' || fail 'Service web sai'
[ "$(jp service/api '{.spec.ports[0].port}:{.spec.ports[0].targetPort}')" = 3000:http ] && pass 'Service api 3000 -> http' || fail 'Service api sai'
[ "$(jp service/db '{.spec.clusterIP}:{.spec.ports[0].port}')" = None:5432 ] && pass 'Service db headless 5432' || fail 'Service db sai'
[ "$(jp service/node-agent '{.spec.ports[0].port}:{.spec.internalTrafficPolicy}')" = 9100:Local ] \
  && pass 'Service node-agent 9100, internalTrafficPolicy Local' || fail 'Service node-agent sai'

# 7. Dữ liệu bền (4b, 6a)
# backup-store (WaitForFirstConsumer) chỉ Bound sau lần chạy thử db-backup ở mục 8 (6a).
[ "$(jp pvc/data-db-0 '{.status.phase}')" = Bound ] && pass 'PVC data-db-0 Bound' || fail 'PVC data-db-0 chua Bound'
[ "$(jp pvc/data-db-0 '{.spec.storageClassName}')" = bookstore-retain ] && pass 'data-db-0 tren bookstore-retain' || fail 'data-db-0 sai StorageClass'
N="$(k exec db-0 -c postgres -- psql -U postgres -d bookstore -tAc 'SELECT count(*) FROM api.books' 2>/dev/null)"
[ "${N:-0}" -ge 8 ] && pass "db co $N cuon sach" || fail "db co ${N:-0} cuon sach, can >= 8"
if k get job db-migrate >/dev/null 2>&1; then
  [ "$(jp job/db-migrate '{.status.conditions[?(@.type=="Complete")].status}')" = True ] && pass 'Job db-migrate Complete' || fail 'Job db-migrate chua Complete'
else
  pass 'Job db-migrate da duoc TTL don, ket qua nam trong db'
fi

# 8. Đường đi của request trong cluster (5a, 5b)
WEB="$(k get pod -l "$SEL,app.kubernetes.io/component=web" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)"
WNODE="$(jp "pod/$WEB" '{.spec.nodeName}')"
get() { k exec "$WEB" -c status-writer -- wget -qO- -T 5 "$1" 2>/dev/null; }
T="$(get http://api:3000/books | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && pass "api:3000/books tra $T cuon sach qua DNS" || fail "api:3000/books tra $T cuon sach"
get http://127.0.0.1:8080/ | grep -q Bookstore && pass 'web phuc vu trang chu' || fail 'web khong phuc vu trang chu'
T="$(get http://127.0.0.1:8080/api/books | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && pass "nginx proxy /api/books tra $T cuon sach" || fail "nginx proxy /api/books tra $T cuon sach"
get http://127.0.0.1:8080/status.json | grep -Eq "\"pod\" *: *\"$WEB\"" && pass 'status.json dung ten Pod' || fail 'status.json sai'
get http://node-agent:9100/node.json | grep -Eq "\"node\" *: *\"$WNODE\"" \
  && pass 'node-agent qua Service tra dung node cua Pod goi' || fail 'node-agent khong tra dung node'

# 9. Vào từ ngoài qua Ingress HTTPS (5b)
{ [ "$(jp ingress/bookstore '{.spec.rules[0].host}')" = bookstore.lab.local ] && \
  [ "$(jp ingress/bookstore '{.spec.tls[0].secretName}')" = bookstore-tls ]; } \
  && pass 'Ingress bookstore: host bookstore.lab.local, TLS bookstore-tls' || fail 'Ingress bookstore sai'
TIP="$(kubectl -n traefik get svc -l app.kubernetes.io/name=traefik -o jsonpath='{.items[0].spec.clusterIP}' 2>/dev/null)"
TP="$(kubectl -n traefik get svc -l app.kubernetes.io/name=traefik -o jsonpath='{.items[0].spec.ports[?(@.name=="websecure")].port}' 2>/dev/null)"
T="$(curl -sk --max-time 10 --resolve "bookstore.lab.local:${TP:-443}:$TIP" "https://bookstore.lab.local:${TP:-443}/api/books" 2>/dev/null | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && pass "HTTPS qua Traefik tra $T cuon sach" || fail "HTTPS qua Traefik tra $T cuon sach"

# 10. Pod tạm: LimitRange điền mặc định, NetworkPolicy chặn (5b, 7b)
P="smoke-tam-$(date +%s)"
if k run "$P" --restart=Never --image=busybox:1.37 --command -- \
     sh -c 'wget -T 3 -qO- http://api:3000/books >/dev/null 2>&1 && echo MO || echo CHAN' >/dev/null 2>&1; then
  [ -n "$(jp "pod/$P" '{.metadata.annotations.kubernetes\.io/limit-ranger}')" ] \
    && pass 'Pod tam khong khai tai nguyen duoc nhan, LimitRange dien mac dinh' || fail 'Pod tam khong nhan mac dinh'
  k wait --for=jsonpath='{.status.phase}'=Succeeded "pod/$P" --timeout=90s >/dev/null 2>&1
  [ "$(k logs "$P" 2>/dev/null)" = CHAN ] && pass 'Pod la bi NetworkPolicy chan toi api' || fail 'Pod la goi duoc api'
  k delete pod "$P" --wait=true >/dev/null 2>&1
else
  fail 'Pod tam bi tu choi — kiem LimitRange bookstore-limits'
fi

# 11. Tác vụ nền (4b, 6a)
[ "$(jp cronjob/db-backup '{.spec.concurrencyPolicy}')" = Forbid ] && pass 'db-backup Forbid' || fail 'db-backup khong Forbid'
[ "$(jp cronjob/db-backup '{.spec.suspend}')" != true ] && pass 'db-backup khong bi tam ngung' || fail 'db-backup dang suspend'
jp cronjob/db-backup '{..persistentVolumeClaim.claimName}' | grep -qw backup-store \
  && pass 'db-backup ghi vao PVC backup-store' || fail 'db-backup khong dung PVC backup-store'
J="smoke-backup-$(date +%s)"
if k create job "$J" --from=cronjob/db-backup >/dev/null 2>&1 \
   && k wait --for=condition=Complete "job/$J" --timeout=180s >/dev/null 2>&1 \
   && k logs "job/$J" 2>/dev/null | grep -q '^db-backup: OK'; then
  pass 'chay thu db-backup thanh cong'
else
  fail 'chay thu db-backup that bai'
fi
k delete job "$J" --ignore-not-found --wait=true >/dev/null 2>&1
# Hợp đồng ghi file backup (6a): đọc thẳng PVC, không tin log — tối đa 10 bản mỗi loại,
# không còn file tạm cũ hơn một giờ.
RD="smoke-doc-backup-$(date +%s)"
k apply -f - >/dev/null 2>&1 <<POD
apiVersion: v1
kind: Pod
metadata:
  name: $RD
spec:
  restartPolicy: Never
  securityContext:
    runAsNonRoot: true
    runAsUser: 65534
    seccompProfile: {type: RuntimeDefault}
  containers:
  - name: doc
    image: busybox:1.37
    command: ["sh", "-c", "ls -1 /backup; echo ---; find /backup -maxdepth 1 -name '*.tmp' -mmin +60"]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities: {drop: [ALL]}
    resources:
      requests: {cpu: 10m, memory: 16Mi}
      limits: {cpu: 50m, memory: 32Mi}
    volumeMounts:
    - {name: b, mountPath: /backup, readOnly: true}
  volumes:
  - name: b
    persistentVolumeClaim: {claimName: backup-store, readOnly: true}
POD
if k wait --for=jsonpath='{.status.phase}'=Succeeded "pod/$RD" --timeout=120s >/dev/null 2>&1; then
  OUT="$(k logs "$RD" 2>/dev/null)"
  NB="$(echo "$OUT" | sed '/^---$/,$d' | grep -c '^bookstore-.*\.sql$')"
  NO="$(echo "$OUT" | sed '/^---$/,$d' | grep -Ec '^dbbackup-.*\.(sql|dump)$')"
  NT="$(echo "$OUT" | sed '1,/^---$/d' | grep -c .)"
  { [ "$NB" -le 10 ] && [ "$NO" -le 10 ] && [ "$NT" -eq 0 ]; } \
    && pass "backup-store: $NB ban db-backup, $NO ban operator, khong file tam cu" \
    || fail "backup-store sai hop dong 6a ($NB ban db-backup, $NO ban operator, $NT file tam cu)"
else
  fail 'khong doc duoc noi dung backup-store'
fi
k delete pod "$RD" --ignore-not-found --wait=false >/dev/null 2>&1
[ "$(jp pvc/backup-store '{.status.phase}')" = Bound ] && pass 'PVC backup-store Bound (sau lan chay thu)' || fail 'PVC backup-store chua Bound'
NODES="$(kubectl get nodes --no-headers | wc -l)"
[ "$(jp daemonset/node-agent '{.status.numberReady}')" = "$NODES" ] && pass "node-agent Ready tren $NODES node" || fail 'node-agent chua du node'

exit "$FAIL"
EOF
chmod +x ~/bookstore/tests/smoke.sh
```

2. Cập nhật `docs/tech-debt.md`: dòng nợ giới hạn PID đã thêm ở B7. Sprint này không trả nợ nào; ghi
   chú ở dòng nợ quyền truy cập (`TD-07` hoặc mục tương ứng của bạn) rằng `bookstore-quota` cũng cần
   được bảo vệ bằng RBAC ở Lab 9a.

3. Dựng lại sạch từ repo và chạy kiểm thử khói:

```bash
cd ~/bookstore
./scripts/teardown.sh && ./scripts/deploy.sh > ~/lab-evidence/7b/b10-deploy.txt 2>&1
./tests/smoke.sh > ~/lab-evidence/7b/b10-smoke.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/7b/b10-smoke.txt
```

4. Commit, tag, đóng gói, chép ra ngoài VM theo
   [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm):

```bash
cd ~/bookstore
git status --short
git add -A && git commit -m 'lab-7b: sprint 11 — LimitRange, ResourceQuota, ngan sach namespace'
git tag lab-7b
git bundle create ~/bookstore-lab-7b.bundle --all && git bundle verify ~/bookstore-lab-7b.bundle
```

Chép `~/bookstore-lab-7b.bundle` về máy host bằng `scp` trước khi đi tiếp.

5. Dọn object thử nghiệm và trả cluster:

```bash
kubectl -n bookstore delete job,pvc -l lab --ignore-not-found --wait=true
kubectl -n bookstore delete pod -l '!app.kubernetes.io/part-of' --ignore-not-found --wait=true
kubectl -n bookstore delete resourcequota kiem-ke batch-thu --ignore-not-found
kubectl delete namespace bookstore-thu --ignore-not-found --wait=true
rm -rf ~/lab-work/7b
cd ~/bookstore && ./scripts/teardown.sh
```

6. Gate cuối:

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
cd ~/bookstore
test -z "$(git status --porcelain)" && echo 'PASS: repo khong con thay doi chua commit'
git describe --tags --exact-match HEAD | grep -qx 'lab-7b' && echo 'PASS: HEAD mang tag lab-7b'
test -s ~/bookstore-lab-7b.bundle && echo 'PASS: co bundle'
{ test -f k8s/app/policy/limitrange.yaml && test -f k8s/app/policy/resourcequota.yaml && \
  grep -q 'policy' k8s/app/kustomization.yaml; } && echo 'PASS: bo cuc repo dung muc 5.5 (k8s/app/policy/)'
grep -q 'podPidsLimit' docs/tech-debt.md && echo 'PASS: tech-debt.md co no gioi han PID'
BAD=0
for F in ngan-sach-namespace.md 'adr/*-ngan-sach-namespace.md' 'adr/*-limitrange.md' 'adr/*-resourcequota.md' \
         'adr/*-quota-so-luong-object.md' 'adr/*-gioi-han-pid.md' 'adr/*-resource-manager.md' \
         'postmortem/7b-rollout-ket-vi-quota.md' 'postmortem/7b-chinh-sach-siet-sai.md'; do
  ls docs/$F >/dev/null 2>&1 || { echo "FAIL: thieu docs/$F"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: du ngan sach, ADR va postmortem cua sprint 11'
{ [ "$(sl kubectl get namespace,priorityclass,storageclass -l app.kubernetes.io/part-of=bookstore --no-headers)" -eq 0 ] && \
  [ "$(sl kubectl get all -A -l app.kubernetes.io/part-of=bookstore --no-headers)" -eq 0 ]; } \
  && echo 'PASS: khong con object nao cua du an'
[ "$(sl kubectl get resourcequota,limitrange -A --no-headers)" -eq 0 ] \
  && echo 'PASS: khong con ResourceQuota, LimitRange nao trong cluster'
[ -z "$(kubectl get nodes -l bookstore.example.com/data -o name)" ] && echo 'PASS: label du lieu da go khoi node'
[ "$(sl kubectl get pv --no-headers)" -eq 0 ] && echo 'PASS: khong con PV nao — dung 03-storage-ready'
test ! -e ~/lab-work/7b && echo 'PASS: da don lab-work'
{ echo "lab-k8s-master $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for n in lab-k8s-worker1 lab-k8s-worker2; do
    echo "$n $(ssh "$n" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done; } > ~/lab-evidence/7b/b10-kubelet-sha.txt
diff -q ~/lab-evidence/7b/b0-kubelet-sha.txt ~/lab-evidence/7b/b10-kubelet-sha.txt >/dev/null \
  && echo 'PASS: cau hinh kubelet cua ca ba node khong doi trong suot sprint' \
  || echo 'FAIL: cau hinh kubelet da bi sua — xem muc 4'

kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

**PASS:** `smoke exit=0` và file smoke ở bước 3 in toàn `PASS:`; mười hai dòng `PASS:` đầu của gate
cuối xuất hiện, không dòng `FAIL:` nào; dòng `PASS: readyz ok`; ba node `Ready`; lệnh field selector
trả `No resources found`; CoreDNS đủ replica; namespace `default` không có Pod. Thư mục
`~/lab-evidence/7b/` **giữ lại**. Cluster trở về `03-storage-ready`; **không tạo snapshot mới**.

---

## 3. Checkpoint 7b

Không đánh dấu lab hoàn tất chỉ vì script nghiệm thu PASS. Tự trả lời, không nhìn tài liệu:

- [ ] Bài 132 chia chính sách theo nơi cưỡng chế. Với hệ thống Bookstore, kể object nào nằm ở mặt
      phẳng API và thứ gì nằm trên kubelet. Vì sao `kubectl get` không bao giờ cho bạn biết
      `lab-k8s-worker2` đang chạy `cpuManagerPolicy` nào hay có giới hạn PID không?
- [ ] Công thức request hiệu dụng của Pod `web`. Quota đếm con số nào, và vì sao một init container
      quên khai cộng một giá trị mặc định lớn có thể làm cả namespace hết chỗ?
- [ ] Đỉnh trong ngân sách của bạn gồm những gì? Vì sao phải tính Pod đang kết thúc, và vì sao `db`
      không có phần trội?
- [ ] Pod tạm không khai tài nguyên vào `bookstore`: ai điền giá trị, ở bước nào của yêu cầu, dấu vết
      nằm ở đâu? Vì sao không Pod nào của dự án được mang dấu vết đó?
- [ ] Pod debug xin `cpu: 150m` mà không khai limit: thông báo gì, vì sao không phải `Forbidden`, sửa
      từ phía Pod thế nào?
- [ ] Ba thông báo `must specify`, `exceeded quota`, `maximum ... per Container`: mỗi cái do cơ chế
      nào, và người nhận phải sửa ở đâu? Vì sao quota đếm đúng giá trị LimitRange vừa chèn?
- [ ] Rollout `web` kẹt vì quota: lệnh báo gì, ba chỗ đọc nguyên nhân là đâu, vì sao người dùng không
      thấy gì? `maxSurge: 0`, `maxUnavailable: 1` đổi kết cục thế nào?
- [ ] Hạ trần LimitRange dưới limit của `db`: chuyện gì xảy ra ngay, chuyện gì xảy ra ở lần thay
      `db-0` kế tiếp? Quy trình đổi chính sách an toàn của đội là gì?
- [ ] `services.nodeports: 0`, trần PVC và `count/jobs.batch` mỗi cái bảo vệ điều gì mà trần
      CPU/memory không bảo vệ được? Trần Job quá chặt thì `db-backup` ra sao, ai phát hiện?
- [ ] Thêm worker thứ ba: `bookstore-quota` có nới không, có giữ Pod của `bookstore` trên node nào
      không? Vì sao `requests.*` phải dưới allocatable còn `limits.cpu` thì không cần?
- [ ] Giới hạn PID của Pod khai ở đâu? Trên cluster lab giá trị là bao nhiêu, đọc ở cgroup nào, và vì
      sao `pids.max` trong container không đủ để kết luận? `pid.available` có thay được giới hạn cứng
      không?
- [ ] `db-0` có CPU độc quyền không? Ba điều kiện của bài 74, `db-0` thiếu điều kiện nào, và vì sao
      đổi sang `static` trên worker 2 vCPU là cái giá không đáng? Tính năng do node khai báo chống lại
      loại sự cố nào?

Nợ còn mở sau lab này — nói được **vì sao chưa trả**:

- Nợ lab: sprint này không phát sinh, không trả. Nợ lab #1 vẫn mở — HPA trả ở Lab 11b, và quota của
  sprint này đã giữ chỗ cho `web` 5 bản sao; VPA chưa có lab trả. Các nợ lab khác của lộ trình xem
  [sổ nợ lab](README.md#5-sổ-nợ-lab).
- Nợ kỹ thuật: **mới** — giới hạn PID của node (`TD-13` trong lời giải), chờ giai đoạn 20 và đề xuất
  ghi vào `platform/kubeadm/` ở Lab 8a. Vẫn mở từ trước: `TD-06` (`db` khởi động bằng root — Lab 9b),
  `TD-07` (quyền debug và exec — Lab 9a, nay thêm quyền sửa quota), `TD-10` (bản backup trong cùng
  cluster — chấp nhận có điều kiện ở 6b), cùng nợ mà Lab 7a ghi thêm nếu có.

### Bài giải thích cuối cùng

Trong vài phút, kể lại hành trình của **một bản sao `web` mới** khi bạn phát hành lại ở 5 bản sao:
Deployment tạo ReplicaSet mới; yêu cầu tạo Pod đi qua LimitRange — nó chèn gì nếu thiếu, nó soi gì —
rồi qua ResourceQuota — nó cộng con số nào vào `used`, và khi nào nó trả về `FailedCreate`; nếu lọt
cả hai, scheduler đặt Pod ở đâu và vì sao quota không can thiệp vào việc đó; trên node, kubelet giới
hạn CPU của container bằng gì, PID của Pod bằng gì — và hôm nay cái nào đang không giới hạn. Rồi kể
cuộc diễn tập B6 bằng chính hành trình đó: vì sao `db-0` mới không qua được cửa đầu tiên. Kết thúc
bằng việc chỉ ra Lab 11b (HPA) và Lab 12 (drain node) sẽ đặt hạn mức của sprint này vào thử thách ở
điểm nào.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường xem [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Dưới đây chỉ là sự cố phát sinh trong nội dung bài học.

| Triệu chứng | Nguyên nhân thường gặp | Xử lý |
| --- | --- | --- |
| B0: `git describe` không ra `lab-7a` | Repo ở commit khác, hoặc clone từ bundle cũ | `git checkout lab-7a` hoặc clone lại từ `~/bookstore-lab-7a.bundle` mới nhất |
| B0: `configz` không đọc được trên một node | Đường control plane → kubelet hỏng; ảnh hưởng cả `logs`, `exec` | Chạy lại [tầng 6 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a547-tầng-6--đường-control-plane--kubelet); không sửa gì trên node. Vẫn hỏng thì restore cả ba VM |
| B0: `kubectl get resourcequota,limitrange -A` có kết quả | Sprint trước để sót, hoặc cluster lệch mốc | Đọc namespace của object; nếu không thuộc baseline thì restore `03-storage-ready` |
| B1: `qv kiem-ke used pods` mãi rỗng | Quota controller chưa đồng bộ object mới | Chờ thêm; `kubectl -n bookstore describe quota kiem-ke`. Không xóa rồi tạo lại liên tục |
| B1: `thuong.*` lệch số đo | Lúc đo có Pod Job (`db-backup`, `db-migrate`) đang chạy, hoặc một Pod tạm còn sót | Chờ Job xong, xóa Pod tạm, đo lại; script đã chờ Job nhưng không biết Pod tạm của bạn |
| B1: container thiếu tài nguyên là init container | 3c chỉ khai cho container chính | Thêm `resources` vào đúng init container trong manifest, `./scripts/deploy.sh` |
| B2: `apply` LimitRange báo `default value ... is greater than max value` hoặc `min value ... is greater than` | API server kiểm thứ tự `min` ≤ `defaultRequest` ≤ `default` ≤ `max` của chính LimitRange | Sửa con số trong manifest; LimitRange không bao giờ được lưu ở trạng thái mâu thuẫn |
| B2: `deploy.sh` treo ở `rollout status` sau khi thêm LimitRange | `min` cao hơn request của một container (ví dụ `status-writer` 8Mi) — ReplicaSet `FailedCreate` với `minimum ... usage per Container` | `kubectl -n bookstore describe rs`; hạ `min` dưới container nhỏ nhất |
| B2: bẫy ra `Forbidden` thay vì `must be less than or equal to cpu limit` | Request của Pod bẫy vượt `max` hoặc dưới `min`, LimitRange chặn trước | Script lấy `default` + 50m; nếu `max` của bạn nhỏ hơn số đó thì hạ request bẫy |
| B3: `deploy.sh` treo sau khi thêm quota | Trần nhỏ hơn tổng thường; hoặc một container chưa khai tài nguyên nên `must specify` | `kubectl -n bookstore describe rs`, `describe quota`; quay lại ngân sách B1 |
| B3: phát hành ở 5 bản sao chậm, có `FailedCreate` thoáng qua | Dự phòng không đủ cho Pod đang kết thúc | Chấp nhận nếu xong trong 300 giây; nếu không, nới dự phòng và ghi vào ADR |
| B3: `web` vẫn 20 bản sao sau `deploy.sh` | `replicas` không nằm trong manifest | Sprint này `replicas: 3` phải có trong repo; bỏ `replicas` là việc của Lab 11b |
| B4: dry-run NodePort vẫn qua | Thiếu `services.nodeports` hoặc `status.hard` chưa cập nhật | Đọc `kubectl -n bookstore get resourcequota bookstore-quota -o yaml`; chờ controller rồi chạy lại |
| B4: PVC thử nghiệm được cấp PV | Dùng nhầm StorageClass có thật | Xóa PVC; dùng `storageClassName` không tồn tại như script |
| B4: `deploy.sh` hỏng ở `db-migrate` sau B4 | Job thử nghiệm còn sót, `count/jobs.batch` đầy | `kubectl -n bookstore delete job -l lab=7b-dem` rồi chạy lại |
| B5: không thấy `FailedCreate` | `status.hard` chưa nhận giá trị siết; hoặc `maxUnavailable` > 0 nên Deployment bớt Pod cũ để đi tiếp | Chờ `hard` đổi rồi phát hành lại; đọc chiến lược ở `b5-truoc.txt`, ghi diễn biến vào postmortem |
| B5: rollout không chạy tiếp sau khi trả quota | ReplicaSet controller đang chờ theo nhịp lùi sau nhiều lần thất bại | `kubectl -n bookstore rollout restart deployment/web` — script đã làm |
| B6: Pod tạm vẫn được nhận khi đã gỡ LimitRange | Quota thiếu khóa `cpu`/`memory`; hoặc còn một LimitRange khác | `kubectl -n bookstore get limitrange`; kiểm `spec.hard` có `requests.cpu` và `limits.cpu` |
| B6: patch LimitRange bị từ chối | Trần mới nhỏ hơn `default` | Script chỉ patch khi trần mới ≥ `default`; nếu `db` của bạn có limit nhỏ, đọc dòng `b6-b-truoc.txt` |
| B6: `db-0` chưa lên lại sau khi trả LimitRange | StatefulSet controller chờ theo nhịp lùi; hoặc PV `local-path` chỉ gắn được `lab-k8s-worker2` mà node đang bận | Chờ, đọc sự kiện của StatefulSet; **không** xóa cưỡng bức, không xóa PVC |
| B6: `api` vẫn Ready sau khi xóa `db-0` | PostgREST chưa phát hiện mất kết nối | Chờ thêm; `kubectl -n bookstore describe pod -l app.kubernetes.io/component=api` |
| B7: `podcg` không in gì | Không tìm thấy thư mục cgroup của Pod: driver khác `systemd` hoặc UID sai | `ssh <node> 'stat -fc %T /sys/fs/cgroup'` phải là `cgroup2fs`; xem cgroup driver ở `b0-configz-*.json` |
| B7: `podPidsLimit` khác `-1` | Node đã được cấu hình trần PID — cluster lệch baseline | **Không sửa file để "về chuẩn"**; ghi giá trị vào evidence và ADR, kỳ vọng `pids.max` cấp Pod bằng đúng số đó |
| B8: `Cpus_allowed_list` hẹp hơn mọi CPU | Policy đã là `static` — cluster lệch baseline | Đối chiếu `b8-policy.txt`; restore `03-storage-ready` nếu không phải chủ đích |
| B8 ⭐: `cpu-nguyen` nằm `Pending` | Worker không còn đủ 1 CPU chưa cấp phát | `kubectl describe node lab-k8s-worker1`, phần *Allocated resources*; bỏ qua phần ⭐ hoặc chạy sau khi teardown dự án |
| B10: `teardown.sh` chờ lâu | `db-0` dùng hết ân hạn khi thu về 0 | Chờ; script của Lab 7a có thời hạn riêng |
| B10: `diff` checksum kubelet báo khác | Cấu hình kubelet đã bị sửa trong sprint | Tắt cả ba VM, restore `03-storage-ready`; không sang lab sau khi gate này chưa PASS |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Policies](https://v1-35.docs.kubernetes.io/docs/concepts/policy/)
- [Kubernetes v1.35 — Limit Ranges](https://v1-35.docs.kubernetes.io/docs/concepts/policy/limit-range/)
- [Kubernetes v1.35 — Resource Quotas](https://v1-35.docs.kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Kubernetes v1.35 — Process ID Limits And Reservations](https://v1-35.docs.kubernetes.io/docs/concepts/policy/pid-limiting/)
- [Kubernetes v1.35 — Resource Managers](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/resource-managers/)
- [Kubernetes — Node Declared Features](https://kubernetes.io/docs/concepts/scheduling-eviction/node-declared-features/)
  — tính năng ra sau baseline của Lab 00, nên dùng link không gắn version
- [Kubernetes v1.35 — Deployments, mục Failed Deployment](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/deployment/#failed-deployment)
- [Kubernetes v1.35 — Sidecar Containers, mục Resource sharing within containers](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/#resource-sharing-within-containers)
- [Kubernetes v1.35 — Pod Priority and Preemption](https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
- [Kubernetes v1.35 — ResourceQuota API reference](https://v1-35.docs.kubernetes.io/docs/reference/kubernetes-api/policy-resources/resource-quota-v1/)
- [Kubernetes v1.35 — LimitRange API reference](https://v1-35.docs.kubernetes.io/docs/reference/kubernetes-api/policy-resources/limit-range-v1/)
- [Kubernetes v1.35 — Admission Control, mục LimitRanger và ResourceQuota](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- [Kubernetes v1.35 — Kubelet Configuration (v1beta1)](https://v1-35.docs.kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)
- [Kubernetes v1.35 — Set Kubelet Parameters Via A Configuration File](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/)
- [PostgreSQL 17 — Connections and Authentication (`max_connections`)](https://www.postgresql.org/docs/17/runtime-config-connection.html)

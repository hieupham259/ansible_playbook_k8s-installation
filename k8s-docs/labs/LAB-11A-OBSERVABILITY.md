# Lab 11a — Observability

> **Điểm bắt đầu:** snapshot `03-storage-ready` — mốc do [Lab 6a](LAB-6A-PV-PVC-VA-STORAGECLASS.md)
> tạo, xem [chuỗi snapshot](README.md#3-chuỗi-snapshot). Repo `~/bookstore` ở tag `lab-9b` (hoặc
> bundle `~/bookstore-lab-9b.bundle` nếu bạn vừa restore snapshot).
> **Điểm kết thúc:** lab **tạo mốc mới `04-metrics-ready`** — cài metrics-server vào `kube-system` và
> để lại nó cho mọi lab sau. Snapshot chụp **sau** teardown, chỉ chứa hạ tầng nền (Calico, Traefik,
> local-path-provisioner, metrics-server). Repo có tag `lab-11a` và một bản bundle nằm ngoài VM.
> **Sprint dự án:** Sprint 17 — Quan sát hệ thống của [dự án Bookstore](DU-AN-BOOKSTORE.md).
> **Lab trước:** [Lab 9b — Pod Security và hardening](LAB-9B-POD-SECURITY-VA-HARDENING.md).
> **Lab sau:** [Lab 11b — HPA và VPA](LAB-11B-HPA-VA-VPA.md).
>
> **Cập nhật và đối chiếu phiên bản:** 13/10/2026.

Lab này đi cùng mục
[Giai đoạn 11 — Observability](../00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability): sáu bài khái
niệm [162](../162-observability-vi.md), [160](../160-system-metrics-vi.md),
[163](../163-kube-state-metrics-vi.md), [158](../158-logging-vi.md),
[159](../159-system-logs-vi.md), [161](../161-system-traces-vi.md) và khối thực hành
[296](../296-debug-vi.md), [297](../297-debug-application-vi.md),
[304](../304-get-shell-running-container-vi.md), [309](../309-local-debugging-vi.md),
[316](../316-debug-logging-vi.md), [317](../317-debug-monitoring-vi.md),
[342](../342-hpa-walkthrough-vi.md). Mọi ticket dựa vào
[hợp đồng kỹ thuật](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật) và hàng "11a" của
[bố cục repo](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint). Kế hoạch gốc của sprint nằm ở
[mục 7.2](DU-AN-BOOKSTORE.md#sprint-17--lab-11a-quan-sát-hệ-thống).

Sprint 16 để lại một hệ thống đã cứng: `bookstore` và `bookstore-staging` thực thi Pod Security
`restricted`, `web` và `api` chạy root filesystem chỉ đọc, `db` chạy non-root, mạng zero-trust,
quyền tối thiểu. Nhưng người trực vẫn **mù**: không ai đọc metric, `kubectl top` báo lỗi vì cluster
chưa có Metrics API, log chỉ xem được từng container một bằng `kubectl logs`, và không có quy trình
chẩn đoán nào được viết ra. Sprint này đáp ứng [R15](DU-AN-BOOKSTORE.md#22-yêu-cầu-của-hệ-thống-mới):
có metric, log tập trung theo node, và quy trình chẩn đoán — đồng thời dựng điều kiện đầu vào của
Lab 11b.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa). Thành phần duy nhất lab
cài thêm là metrics-server; version của nó đọc ở
[bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00). Lab không chép lại con
số phiên bản nào trong phần văn bản.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`. B0 bên dưới chạy lại đúng gate đó.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi hệ thống Bookstore đạt trạng thái sau sprint 17 — **có metric đọc được, có agent
log trên mọi node, có Metrics API, có quy trình chẩn đoán** — **và** bạn chứng minh, giải thích được:

- Ba trụ cột metrics, logs, traces trên chính hệ thống của bạn: thành phần nào đã phát tín hiệu, khúc
  nào của pipeline "nguồn phát → bộ thu thập → kho lưu trữ → dashboard" còn trống, và vì sao.
- Metric nằm ở endpoint `/metrics` của từng thành phần, theo định dạng Prometheus: đọc được metric
  của PostgREST ở cổng admin, của kube-apiserver, và bốn endpoint của kubelet; chỉ ra mức ổn định
  của một metric ngay trong dòng `HELP`; giải thích vì sao các endpoint của kubelet **không cùng
  vòng đời**.
- Ranh giới của bài 163: "`db-0` dùng bao nhiêu bộ nhớ" là metric **tài nguyên**, có sẵn từ kubelet;
  "`web` có mấy bản sao sẵn sàng" là **trạng thái object**, không có ở dạng metric trên cluster này.
- Đọc metric là hành động **được ủy quyền**: một ServiceAccount có token hợp lệ vẫn bị từ chối ở
  `/metrics` của API server cho tới khi có ClusterRole với `nonResourceURLs` — và ClusterRole đó mở
  đúng một đường dẫn, không hơn. Còn `/metrics` của PostgREST thì được bảo vệ bằng **mạng**, không
  bằng RBAC.
- Cài metrics-server trên cluster kubeadm, chẩn đoán vì sao nó không `Ready` từ log của chính nó,
  chọn đường sửa và nói được **đánh đổi**; `kubectl top` chạy cho cả node lẫn Pod của Bookstore; nói
  được metrics-server **không** thay được Prometheus.
- Kiến trúc agent ghi log cấp node: `node-agent` rời `bookstore` sang `bookstore-system` vì cần
  `hostPath` mà profile `restricted` cấm; đọc log container của Pod `bookstore` trên đúng node của nó,
  không chạy root; đường dẫn thật `/var/log/pods/<namespace>_<pod>_<uid>/<container>/<n>.log`.
- Kubelet xoay vòng log và `kubectl logs` chỉ đọc file mới nhất — chứng minh **bằng con số** trên
  `lab-k8s-worker2`; log sống theo Pod.
- Log của chính Kubernetes nằm ở hai chỗ theo nơi thành phần chạy; thang `-v`; tracing đang tắt ở
  cả apiserver lẫn kubelet — chứng minh bằng cấu hình thật mà **không sửa gì**.
- Chẩn đoán một dịch vụ không trả lời bằng bộ công cụ của giai đoạn 11 — `kubectl logs`,
  `--previous`, `kubectl exec` — theo đúng cách bài 296, 297 khoanh vùng, và biết chỗ nào dừng lại
  để nhường cho giai đoạn 24.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 11 | Ticket kiểm chứng |
| --- | --- |
| [162 — Khả năng quan sát](../162-observability-vi.md) | B1 — ba trụ cột trên hệ thống thật, nguồn phát nào có sẵn, khúc nào của pipeline còn trống; B3 — metrics-server lấp khúc "bộ thu thập" cho metric tài nguyên; B4 — agent cấp node là khúc thu thập của trụ cột log; B6 — trụ cột trace chưa có nguồn |
| [160 — Metric cho các thành phần hệ thống](../160-system-metrics-vi.md) | B1 — `/metrics` của kube-apiserver, bốn endpoint của kubelet, định dạng Prometheus, mức ổn định trong `HELP`, ⭐ chữ ký deprecated; B2 — đọc metric cần ủy quyền, `nonResourceURLs`; B3 — `/metrics/resource` là nguồn mà metrics-server tổng hợp |
| [163 — Metric cho trạng thái đối tượng](../163-kube-state-metrics-vi.md) | B1 — metric tài nguyên của `db-0` có, metric `kube_*` không có ở nguồn nào, trạng thái Deployment `web` chỉ đọc được qua API, metric kiểu *info*. Không cài kube-state-metrics — lý do ở bảng dưới |
| [158 — Kiến trúc ghi log](../158-logging-vi.md) | B4 — agent cấp node dạng `DaemonSet` đọc `/var/log/pods` qua `hostPath` chỉ đọc, thấy đúng node của nó; B5 — kubelet xoay vòng log, `kubectl logs` chỉ trả file mới nhất, log sống theo Pod; B6 — vị trí log của thành phần hệ thống; B7 — `kubectl logs`, `--previous`, `-c` |
| [159 — Log hệ thống](../159-system-logs-vi.md) | B6 — journald cho kubelet, file `.log` cho thành phần chạy trong Pod, header klog, thang `-v`, Log Query chỉ đọc cấu hình |
| [161 — Trace cho các thành phần hệ thống](../161-system-traces-vi.md) | B6 — tracing tắt ở kube-apiserver và kubelet, không ai nghe cổng OTLP; không sửa gì |
| [296 — Giám sát, ghi log và gỡ lỗi](../296-debug-vi.md) | B7 — xếp sự cố vào đúng nhánh trước khi chẩn đoán, và ranh giới với giai đoạn 24; B6 — hai nhánh *Ghi log*, *Giám sát* ứng với ticket nào của sprint |
| [297 — Xử lý sự cố ứng dụng](../297-debug-application-vi.md) | B7 — ba nhóm chủ đề của nhánh; dùng phần làm được ở giai đoạn 11 để khoanh một Service không có endpoint và một container khởi động lại liên tục |
| [304 — Truy cập shell của container đang chạy](../304-get-shell-running-container-vi.md) | B7 — shell tương tác, ghi trang gốc rồi tự gọi từ bên trong, lệnh rời và dấu `--`, `-c` cho Pod nhiều container; B1 — `api` không có shell nên đọc metric của nó bằng port-forward |
| [309 — Debug service cục bộ bằng telepresence](../309-local-debugging-vi.md) | **Không chạy công cụ.** Telepresence cài sidecar traffic-agent vào Pod ứng dụng, tức sửa workload trên chuỗi snapshot. B7 ⭐ — câu hỏi thiết kế đối chiếu nó với cách thủ công của bài 304 |
| [316 — Ghi log trong Kubernetes](../316-debug-logging-vi.md) | B6 — trang trỏ hướng tới 158 và 159; B4, B5, B6 phủ đủ hai trang con |
| [317 — Giám sát trong Kubernetes](../317-debug-monitoring-vi.md) | B6 — trang trỏ hướng tới 160 và 161; B1, B2 phủ phần metric, B6 phủ phần trace |
| [342 — HPA walkthrough](../342-hpa-walkthrough-vi.md) | **Thực hành ở Lab 11b.** Sprint này chỉ dựng đúng điều kiện ở mục *Trước khi bạn bắt đầu* của bài — Metrics Server phục vụ qua một APIService (B3). Phần HPA là [nợ lab #1](README.md#5-sổ-nợ-lab), trả ở sprint 18 |

Các phần dưới đây **không kiểm chứng được trong lab này**; lý do ghi rõ thay vì bỏ trống:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| Stack Prometheus + Grafana, kho log tập trung, backend tracing | Hai worker 2 vCPU ([A1.2](LAB-00-MOI-TRUONG-1.35.7.md#a12-ba-vm)); mốc `04-metrics-ready` là điểm xuất phát của năm lab sau, một stack giám sát đầy đủ ăn hết headroom và làm snapshot phình. Chart của chúng cũng không có trong [A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00). Thuộc [giai đoạn 23](../00-ALO-TRINH-ADMIN.md#giai-đoạn-23--giám-sát-và-cảnh-báo). B1 và B3 kiểm chứng **khoảng trống** mà stack đó lấp; dự án ghi nó thành nợ `TD-17` |
| Cài kube-state-metrics | Bài 163 nói đây là add-on bên thứ ba; không có Prometheus thì nó chỉ là một endpoint không ai đọc. B1 kiểm chứng ranh giới mà bài dạy bằng metric có sẵn |
| PromQL và alert rule của bài 163; metric PSI và mục *Yêu cầu* của bài 160 | Cần Prometheus; chính các bài xếp vào *Đọc lướt*, hoãn tới giai đoạn 23 và 24 |
| `--show-hidden-metrics-for-version`, `--disabled-metrics`, `--allow-metric-labels`, `/metrics/resources` của kube-scheduler | Đều là cờ dòng lệnh của control plane — sửa manifest static Pod, việc của [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy). Lab **cấm** sửa `/etc/kubernetes/manifests`; gate cuối kiểm bằng checksum |
| Metric của cloud provider (`cloudprovider_*`) | Cluster lab chạy trên VM tự dựng, không có cloud provider |
| Sidecar chạy agent ghi log thật (fluentd) của bài 158 | Image cũ, cần backend ngoài, tốn tài nguyên như bài cảnh báo. Ranh giới "log không qua `stdout` thì `kubectl logs` không thấy" là câu hỏi thiết kế của B4 |
| `PodLogsQuerySplitStreams`, ghi log có cấu trúc, `--logging-format=json` | Feature gate alpha hoặc cờ của control plane — giai đoạn 20; JSON chỉ có giá trị khi đã có pipeline phân tích log |
| **Bật** Log Query và **bật** tracing | Sửa cấu hình kubelet trên node và manifest của kube-apiserver — giai đoạn 20, cộng một collector ngoài baseline. B6 chỉ **đọc** giá trị hiệu lực |
| Chạy `telepresence connect`, `telepresence intercept` (bài 309) | Binary bên thứ ba cài lên máy trạm, và nó cài sidecar vào Pod ứng dụng — sửa workload trên chuỗi snapshot |
| Nhánh *Gỡ lỗi cluster* của bài 296; quy trình chẩn đoán Pod, Service, StatefulSet chi tiết; termination message | Nội dung [giai đoạn 24](../00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố) (bài 299, 301, 302, 303, 305). B7 chỉ dùng `logs`, `exec` cộng `get`/`describe` đã học, và vẽ ranh giới |
| Exporter cho PostgreSQL, `stub_status` của nginx | Image không có trong A1.4; nginx và PostgreSQL không có endpoint `/metrics` sẵn. B1 ghi khoảng trống này vào ADR |

Nợ lab và nợ kỹ thuật sau sprint:

- **Không phát sinh nợ lab mới.** [Nợ #1](README.md#5-sổ-nợ-lab) (HPA, VPA) **vẫn mở**: sprint này
  chỉ dựng Metrics API; HPA trả ở Lab 11b, VPA chưa có lab trả.
- **Nợ kỹ thuật mới:** `TD-16` — metrics-server bỏ kiểm tra certificate của kubelet (B3); `TD-17` —
  metric và log chưa có nơi lưu trữ dài hạn (B5). Cả hai **chấp nhận có điều kiện** trên mạng lab cô
  lập, trả ở phần vận hành ([giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy),
  [23](../00-ALO-TRINH-ADMIN.md#giai-đoạn-23--giám-sát-và-cảnh-báo)). `TD-10` vẫn chấp nhận có điều
  kiện như Lab 6b; các nợ còn lại đã trả — xem
  [sổ nợ của file dự án](DU-AN-BOOKSTORE.md#8-sổ-nợ-kỹ-thuật-dự-án).

### 1.2. Thời lượng

4–6 giờ. Phần cốt lõi (B0–B8 trừ các mục ⭐) khoảng 4 giờ 30 nếu không bí lâu ở ticket nào. Ba đoạn
chờ đáng kể: metrics-server hỏng đủ lâu để log nói ra nguyên nhân (B3), kubelet xoay vòng khối log
lớn (B5), và bảy tầng gate cộng chụp snapshot (B8). Mọi bước chờ viết thành vòng lặp có điều kiện
thoát — chu kỳ thu thập của metrics-server và chu kỳ giám sát log của kubelet đều **phụ thuộc cấu
hình**. Có thể chia hai phiên: phiên 1 làm B0–B3, phiên 2 làm B4–B8. Giữa hai phiên cứ để nguyên
cluster; đầu phiên 2 chạy lại gate ngắn A5.5 và `./scripts/deploy.sh`.

---

## 2. Quy ước và an toàn

- **Kiểm snapshot đầu vào trước khi bắt đầu.** Lab cài metrics-server và **không** gỡ nó. Nếu B3 hỏng
  giữa chừng, đường lui duy nhất là restore `03-storage-ready`. Trên máy host, PowerShell:

  ```powershell
  $vmrun = 'C:\Program Files\VMware\VMware Workstation\vmrun.exe'
  $vmx = @(
    'E:\Virtual Machines\lab-k8s-master\lab-k8s-master.vmx'
    'E:\Virtual Machines\lab-k8s-worker1\lab-k8s-worker1.vmx'
    'E:\Virtual Machines\lab-k8s-worker2\lab-k8s-worker2.vmx'
  )
  foreach ($f in $vmx) {
    $names = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
    if ($names -ccontains '03-storage-ready') { "PASS: $f" } else { "FAIL: $f" }
  }
  ```

  **PASS:** đúng ba dòng `PASS:`. Không mở B0 khi còn dòng `FAIL:`.

- **Quay lui khi hỏng:** tắt **cả ba VM**, restore **cả ba** về `03-storage-ready` (không bao giờ
  restore riêng một VM — xem ghi chú cuối [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường)),
  khôi phục repo từ bundle `lab-9b`, rồi làm lại từ B0.
- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` trừ khi ghi rõ node khác. Lệnh đọc file trên worker đi
  qua `ssh <node> 'sudo ...'`, **chỉ đọc**.
- **Fault injection chỉ trên `lab-k8s-worker2`.** Pod ghi log hàng chục MB ở B5 và sự cố dựng sẵn ở
  B7 đều ghim vào node này.
- Repo dự án: `~/bookstore`. Manifest thử nghiệm: `~/lab-work/11a/`. Bằng chứng: `~/lab-evidence/11a/`.
  Namespace thử nghiệm duy nhất là `bookstore-thu`, mang label dự án để `teardown.sh` dọn được nếu bạn
  quên; object thử nghiệm đặt tên bắt đầu bằng `thu-` hoặc `bookstore-thu-`.
- **Lab chỉ ĐỌC cấu hình node và control plane.** Không sửa `/var/lib/kubelet/config.yaml`, không sửa
  file nào trong `/etc/kubernetes/manifests`, không restart kubelet. B0 ghi checksum sáu file đó, gate
  cuối ở B8 đối chiếu lại. Thứ duy nhất lab thêm vĩnh viễn vào cluster là metrics-server, cài bằng
  object API chứ không bằng cách sửa file trên node.
- metrics-server là **hạ tầng nền**, không phải một phần của ứng dụng: nó **không** mang label
  `app.kubernetes.io/part-of=bookstore`, `teardown.sh` không gỡ nó, và repo chỉ giữ **ghi chép** cách
  cài trong `platform/` ([mục 3.3 của file dự án](DU-AN-BOOKSTORE.md#33-hạ-tầng-nền-của-cluster)).
- Image kéo từ mạng trong lab: chỉ image của metrics-server. Mọi Pod thử nghiệm dùng `busybox:1.37`.
- Token sinh ra ở B2 chỉ nằm trong biến shell, **không** ghi vào `~/lab-evidence/`.
- Lab chỉ dùng kiến thức nhóm 11 và các nhóm trước. **Không** tạo HorizontalPodAutoscaler (Lab 11b),
  không `kubectl drain` (Lab 12), không sửa cấu hình kubelet hay kube-apiserver (giai đoạn 20), không
  cài Prometheus, kube-state-metrics hay agent log bên thứ ba (giai đoạn 23), không dùng `kubectl
  debug` để chẩn đoán ở B7 (quy trình của giai đoạn 24).
- `kubectl patch`, `kubectl set`, sửa tay object chỉ dùng trên object thử nghiệm và trên
  metrics-server ở B3. Repo là nguồn sự thật duy nhất của hệ thống.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail. Dòng `FAIL:` trong script nghiệm thu
  nghĩa là ticket chưa xong.

---

# Phần B — Sprint 17: Quan sát hệ thống

## B0. Nhận bàn giao sprint

**Mục đích:** xác nhận cluster ở đúng `03-storage-ready` (chưa có Metrics API), repo ở đúng tag
`lab-9b`, hệ thống của sprint 16 dựng lại được **chỉ từ repo**, và chụp lại cấu hình node cùng
manifest control plane để gate cuối chứng minh lab không sửa chúng.

Gate ngắn A5.5, chạy nguyên văn:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

Ba kiểm tra riêng của mốc đầu vào — cluster **chưa** có nguồn metric tài nguyên nào:

```bash
kubectl get apiservice 2>/dev/null | grep -i 'metrics.k8s.io' || echo 'PASS: chua co APIService metrics.k8s.io'
test -z "$(kubectl -n kube-system get deployment metrics-server --ignore-not-found -o name)" \
  && echo 'PASS: chua co metrics-server'
kubectl top node >/dev/null 2>&1 && echo 'FAIL: kubectl top da chay duoc' || echo 'PASS: kubectl top chua chay duoc'
```

Repo, công cụ và thư mục làm việc. Nếu vừa restore snapshot, repo đã mất — chép bundle `lab-9b` từ máy
host lên master theo [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm) trước:

```bash
mkdir -p ~/lab-work/11a ~/lab-evidence/11a
test -d ~/bookstore/.git || git clone ~/bookstore-lab-9b.bundle ~/bookstore
cd ~/bookstore
git describe --tags --exact-match HEAD 2>/dev/null | grep -qx 'lab-9b' && echo 'PASS: HEAD dung tag lab-9b'
test -z "$(git status --porcelain)" && echo 'PASS: repo khong co thay doi chua commit'
kubectl get namespace bookstore >/dev/null 2>&1 \
  && echo 'FAIL: namespace bookstore con sot tu lan truoc' || echo 'PASS: cluster chua co du an'
for T in git openssl curl; do command -v "$T" >/dev/null && echo "PASS: co $T"; done
```

Dựng lại hệ thống của sprint 16 bằng script của chính sprint đó:

```bash
cd ~/bookstore
./scripts/deploy.sh > ~/lab-evidence/11a/b0-deploy.txt 2>&1; echo "deploy exit=$?"
./tests/smoke.sh > ~/lab-evidence/11a/b0-smoke-lab-9b.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/11a/b0-smoke-lab-9b.txt
```

Kéo sẵn `busybox` lên **cả ba** node — agent chạy cả trên control plane, Pod thử nghiệm ở worker 2:

```bash
sudo crictl pull docker.io/library/busybox:1.37 >/dev/null && echo 'PASS: lab-k8s-master co busybox'
for N in lab-k8s-worker1 lab-k8s-worker2; do
  ssh "$N" 'sudo crictl pull docker.io/library/busybox:1.37' >/dev/null && echo "PASS: $N co busybox"
done
```

Cấu hình **hiệu lực** của ba kubelet — gồm cả giá trị mặc định mà file trên đĩa không ghi — đọc qua
endpoint `configz`, đúng đường control plane → kubelet mà
[tầng 6 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a547-tầng-6--đường-control-plane--kubelet) đã kiểm. B5 và
B6 dùng lại ba file này:

```bash
EV=~/lab-evidence/11a
for n in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  kubectl get --raw "/api/v1/nodes/$n/proxy/configz" > "$EV/b0-configz-$n.json"
  grep -q '"kubeletconfig"' "$EV/b0-configz-$n.json" && echo "PASS: doc duoc configz cua $n"
done
```

Checksum của cấu hình kubelet ba node và ba manifest control plane — lời hứa "chỉ đọc" sẽ được kiểm ở
B8:

```bash
EV=~/lab-evidence/11a
{
  echo "lab-k8s-master kubelet $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for n in lab-k8s-worker1 lab-k8s-worker2; do
    echo "$n kubelet $(ssh "$n" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done
  for f in kube-apiserver kube-scheduler kube-controller-manager; do
    echo "lab-k8s-master $f $(sudo sha256sum /etc/kubernetes/manifests/$f.yaml | awk '{print $1}')"
  done
} > "$EV/b0-config-sha.txt"
test "$(awk '{print $3}' "$EV/b0-config-sha.txt" | grep -c '^[0-9a-f]\{64\}$')" -eq 6 \
  && echo 'PASS: ghi duoc checksum cua 3 cau hinh kubelet va 3 manifest control plane'
```

Namespace thử nghiệm của sprint — một file công cụ, chép nguyên văn; các ticket sau `apply` lại nó
khi cần (`teardown.sh` xóa nó cùng dự án):

```bash
cat > ~/lab-work/11a/ns-thu.yaml <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: bookstore-thu
  labels:
    app.kubernetes.io/part-of: bookstore
EOF
```

**PASS:** gate A5.5 đạt như mô tả ở Lab 00; ba dòng `PASS:` của mốc đầu vào; các dòng `PASS:` về repo,
cluster và công cụ; `deploy exit=0`, `smoke exit=0` và file smoke không có dòng `FAIL:`; ba dòng
`PASS: ... co busybox`; ba dòng `PASS: doc duoc configz`; dòng checksum.

`deploy.sh` hoặc `smoke.sh` fail nghĩa là repo của sprint 16 **chưa đủ** — có thứ chỉ tồn tại trên
cluster cũ. Sửa repo ở tag `lab-9b` trước, không sửa tay trên cluster
([mục 6.1 của file dự án](DU-AN-BOOKSTORE.md#61-mở-đầu)).

## B1. [BS-11A-01] Điều tra: ba nguồn metric của hệ thống

**Loại:** Điều tra.

**Bối cảnh:** trước khi cài thêm bất cứ thứ gì, đội phải biết hệ thống **đã** phát những tín hiệu
nào, ở đâu, theo định dạng gì, và ai đọc được. Bookstore có ba tầng nhưng chỉ một tầng — PostgREST —
có endpoint metric của riêng nó. Mức dùng tài nguyên của mọi container thì kubelet đo sẵn. Control
plane phát metric của chính nó. Còn câu hỏi "Deployment có đủ bản sao không" thì chưa ai trả lời ở
dạng metric.

**Yêu cầu:**

1. Lấy được nội dung `/metrics` ở cổng `admin` của một Pod `api` **mà không nới NetworkPolicy và không
   thêm container nào vào Pod `api`**. Chứng minh đồng thời: từ Pod `web`, cổng `admin` của `api` bị
   chặn.
2. Trong metric của PostgREST, chỉ ra metric về **pool kết nối DB** và metric về **số lần nạp schema
   cache**; nói metric nào là gauge, metric nào là counter; chứng minh schema cache đã nạp thành công
   ít nhất một lần.
3. Đọc `/metrics` của kube-apiserver; chỉ ra mức ổn định của metric đọc ở đâu, với ít nhất một ví dụ
   `STABLE` và một ví dụ `ALPHA`.
4. Gọi **bốn** endpoint metric của kubelet trên node đang chạy `db-0`, qua đường proxy của API server.
   Từ endpoint gọn nhất, rút mức CPU tích lũy và bộ nhớ working set của container `postgres` trong
   `db-0`. Trên node đang chạy một Pod `api`, chỉ ra kết quả readiness probe của container `postgrest`.
   Chứng minh endpoint dành cho pipeline tài nguyên gọn hơn hẳn endpoint chi tiết.
5. Chứng minh không nguồn nào ở trên phát metric **trạng thái đối tượng** (họ `kube_*`), trong khi số
   bản sao sẵn sàng của Deployment `web` vẫn đọc được — qua API.
6. Chỉ ra một metric kiểu *info* có sẵn: giá trị là `1`, thông tin nằm ở label.
7. ⭐ Rà chữ ký deprecated trên mọi nguồn đã lấy.
8. ADR: ma trận quan sát của Bookstore — mỗi thành phần × mỗi trụ cột: nguồn nào, ai đọc được, khúc
   nào của pipeline còn trống.

**Ràng buộc:** chỉ đọc. Không sửa NetworkPolicy, không gắn ephemeral container (không gỡ ra được, bài
52), không cài exporter, kube-state-metrics hay Prometheus.

**Đọc lại:**

- Bài [162](../162-observability-vi.md): mục [*Metrics*](../162-observability-vi.md#metrics) — pipeline
  nguồn phát → scraper → kho chuỗi thời gian → dashboard.
- Bài [160](../160-system-metrics-vi.md): [*Metric trong Kubernetes*](../160-system-metrics-vi.md#metric-trong-kubernetes-metrics-in-kubernetes),
  [*Vòng đời metric*](../160-system-metrics-vi.md#vòng-đời-metric-metric-lifecycle).
- Bài [163](../163-kube-state-metrics-vi.md) — cả bài, nhất là đoạn về `kube_pod_container_info`.
- Bài [366](../366-port-forward-vi.md), mục
  [*Cân nhắc về ủy quyền và bảo mật*](../366-port-forward-vi.md#cân-nhắc-về-ủy-quyền-và-bảo-mật-authorization-and-security-considerations).
- PostgREST v14: [Observability — Metrics](https://docs.postgrest.org/en/v14/references/observability.html).

**Bàn giao:** `docs/adr/NNNN-nguon-metric.md` (`NNNN` là số tiếp theo trong `docs/adr/`);
`~/lab-evidence/11a/b1-*.txt`.

**Nghiệm thu:**

- `PASS:` từ `web` không gọi được cổng `admin` của `api`; vẫn lấy được `/metrics` của PostgREST.
- `PASS:` đọc được một gauge và một counter của PostgREST; schema cache đã nạp thành công.
- `PASS:` `/metrics` của apiserver có `apiserver_request_total` và có cả metric `STABLE` lẫn `ALPHA`.
- `PASS:` bốn endpoint của kubelet đều trả dữ liệu; `/metrics/resource` có CPU và bộ nhớ của
  `postgres` trong `db-0`; `/metrics/probes` có readiness của `postgrest`; `/metrics/resource` ít họ
  metric hơn `/metrics/cadvisor`.
- `PASS:` không có chuỗi `kube_*` nào trong mọi nguồn; trạng thái `web` đọc được qua API.
- `PASS:` `kubelet_node_name` là metric kiểu info, giá trị `1`.
- `PASS:` ADR tồn tại.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-nguon-metric.md`:

1. `web` không gọi được cổng `3001` của `api`, nhưng `kubectl port-forward` thì lấy được `/metrics`.
   Traffic của port-forward đi đường nào, và bài 366 cảnh báo gì về quyền `pods/portforward`?
2. PostgREST **không** có metric đếm request. Vậy câu hỏi "`api` đang phục vụ bao nhiêu request mỗi
   giây" trả lời bằng nguồn nào trên hệ thống hiện tại — hay không trả lời được?
3. Một panel dựng trên `container_memory_working_set_bytes` của kubelet và một panel dựng trên
   `apiserver_request_total`: panel nào có cam kết ổn định, đọc cam kết đó ở đâu? Dẫn câu "không cùng
   vòng đời" của bài 160.
4. "`db-0` đang dùng bao nhiêu bộ nhớ" và "`web` có mấy bản sao sẵn sàng" — mỗi câu đến từ nguồn nào?
   Câu nào không có dạng metric trên cluster này, và cần thêm thành phần gì (bài 163)?

<details>
<summary>Gợi ý</summary>

1. `postgrest` không có shell (Lab 3a). Đường ra còn lại là đường mà kubelet mở thẳng vào network
   namespace của Pod: `kubectl port-forward`. Chạy nó ở nền, gọi `curl` vào cổng cục bộ, rồi `kill`.
2. Đường proxy tới kubelet: `kubectl get --raw "/api/v1/nodes/<node>/proxy/<đường dẫn>"`.
3. Trong `/metrics/resource`, label của một mẫu container gồm `container`, `namespace`, `pod`. Lọc bằng
   `grep` hai lần thay vì đoán thứ tự label.
4. Đếm họ metric: số dòng `# TYPE`.
5. Gọi một địa chỉ có giới hạn thời gian từ container có `wget`: `wget -q -T 3 -O /dev/null <url>`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
NS=bookstore; EV=~/lab-evidence/11a
SEL='app.kubernetes.io/part-of=bookstore'
pod() { kubectl -n "$NS" get pod -l "$SEL,app.kubernetes.io/component=$1" \
          --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}'; }
WEB="$(pod web)"; API="$(pod api)"
API_IP="$(kubectl -n "$NS" get pod "$API" -o jsonpath='{.status.podIP}')"
API_NODE="$(kubectl -n "$NS" get pod "$API" -o jsonpath='{.spec.nodeName}')"
DB_NODE="$(kubectl -n "$NS" get pod db-0 -o jsonpath='{.spec.nodeName}')"

# 1-2. Metric của ứng dụng
kubectl -n "$NS" exec "$WEB" -c status-writer -- wget -q -T 3 -O /dev/null "http://${API_IP}:3001/metrics" \
  && echo 'FAIL: web goi duoc cong admin cua api' || echo 'PASS: web khong goi duoc cong admin 3001 cua api'
kubectl -n "$NS" port-forward "pod/$API" 13001:3001 > "$EV/b1-port-forward.log" 2>&1 & PF=$!
for i in $(seq 1 20); do curl -fsS http://127.0.0.1:13001/metrics > "$EV/b1-api-metrics.txt" 2>/dev/null && break; sleep 1; done
kill "$PF" 2>/dev/null; wait "$PF" 2>/dev/null
grep -q '^pgrst_db_pool_max' "$EV/b1-api-metrics.txt" && echo 'PASS: lay duoc /metrics cua PostgREST'
{ grep -q '^# TYPE pgrst_db_pool_available gauge' "$EV/b1-api-metrics.txt" && \
  grep -q '^# TYPE pgrst_schema_cache_loads_total counter' "$EV/b1-api-metrics.txt"; } \
  && echo 'PASS: doc duoc mot gauge va mot counter cua PostgREST'
grep -E '^pgrst_schema_cache_loads_total\{' "$EV/b1-api-metrics.txt" | grep 'status="SUCCESS"' \
  | grep -Eq ' [1-9]' && echo 'PASS: schema cache da nap thanh cong'

# 3. kube-apiserver
kubectl get --raw /metrics > "$EV/b1-apiserver-metrics.txt"
{ grep -q '^apiserver_request_total{' "$EV/b1-apiserver-metrics.txt" && \
  grep -q '^# HELP .*\[STABLE\]' "$EV/b1-apiserver-metrics.txt" && \
  grep -q '^# HELP .*\[ALPHA\]' "$EV/b1-apiserver-metrics.txt"; } \
  && echo 'PASS: apiserver phat metric, muc on dinh nam trong dong HELP'

# 4. Bốn endpoint của kubelet
for EP in metrics metrics/cadvisor metrics/resource; do
  kubectl get --raw "/api/v1/nodes/$DB_NODE/proxy/$EP" > "$EV/b1-kubelet-$(echo "$EP" | tr / -).txt"
done
kubectl get --raw "/api/v1/nodes/$API_NODE/proxy/metrics/probes" > "$EV/b1-kubelet-metrics-probes.txt"
RES="$EV/b1-kubelet-metrics-resource.txt"; CAD="$EV/b1-kubelet-metrics-cadvisor.txt"
grep -q '^kubelet_node_name{' "$EV/b1-kubelet-metrics.txt" && echo 'PASS: /metrics cua kubelet'
{ grep '^container_cpu_usage_seconds_total{' "$RES" | grep 'container="postgres"' | grep -q 'pod="db-0"' && \
  grep '^container_memory_working_set_bytes{' "$RES" | grep 'container="postgres"' | grep -q 'pod="db-0"'; } \
  && echo 'PASS: /metrics/resource co CPU va bo nho cua postgres trong db-0'
grep '^prober_probe_total{' "$EV/b1-kubelet-metrics-probes.txt" | grep "pod=\"$API\"" \
  | grep 'probe_type="Readiness"' | grep -q 'result="successful"' && echo 'PASS: /metrics/probes co readiness cua postgrest'
R_N="$(grep -c '^# TYPE ' "$RES")"; C_N="$(grep -c '^# TYPE ' "$CAD")"
echo "ho metric: resource=$R_N cadvisor=$C_N"
test "$R_N" -lt "$C_N" && echo 'PASS: /metrics/resource gon hon /metrics/cadvisor'

# 5-6. Metric trạng thái đối tượng và metric kiểu info
cat "$EV"/b1-*metrics*.txt > "$EV/b1-moi-nguon.txt"
KS="$(grep -cE '^kube_(pod|deployment|statefulset|node)_' "$EV/b1-moi-nguon.txt")"
test "$KS" -eq 0 && echo 'PASS: khong nguon nao phat metric trang thai doi tuong kube_*'
kubectl -n "$NS" get deploy web -o jsonpath='{.metadata.name} {.spec.replicas} {.status.readyReplicas}{"\n"}' \
  | tee "$EV/b1-web-qua-api.txt" | grep -Eq '^web [0-9]+ [0-9]+$' && echo 'PASS: trang thai web doc duoc qua API'
grep -q "^kubelet_node_name{node=\"$DB_NODE\"} 1$" "$EV/b1-kubelet-metrics.txt" \
  && echo 'PASS: kubelet_node_name la metric kieu info, gia tri 1'

ls ~/bookstore/docs/adr/*-nguon-metric.md >/dev/null 2>&1 && echo 'PASS: co ADR nguon metric'

# ⭐ 7. Chữ ký deprecated
for f in "$EV"/b1-*metrics*.txt; do echo "$(basename "$f"): $(grep -c '(Deprecated since' "$f")"; done \
  | tee "$EV/b1-deprecated.txt"
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```bash
NS=bookstore; EV=~/lab-evidence/11a
API="$(kubectl -n "$NS" get pod -l app.kubernetes.io/component=api -o jsonpath='{.items[0].metadata.name}')"
kubectl -n "$NS" port-forward "pod/$API" 13001:3001 &
curl -s http://127.0.0.1:13001/metrics | grep -E '^(# TYPE )?pgrst_'
kill %1

DB_NODE="$(kubectl -n "$NS" get pod db-0 -o jsonpath='{.spec.nodeName}')"
kubectl get --raw "/api/v1/nodes/$DB_NODE/proxy/metrics/resource" | grep 'pod="db-0"'
grep -E '^# (HELP|TYPE) apiserver_request_total ' "$EV/b1-apiserver-metrics.txt"
```

Khung ADR — ma trận quan sát sau B1 (điền bằng chứng của bạn):

| Thành phần | Metric | Log | Trace |
| --- | --- | --- | --- |
| `web` (nginx) | không có endpoint metric; tài nguyên qua kubelet | access log ra `stdout` | không |
| `api` (PostgREST) | `/metrics` cổng `admin`: pool, schema cache, JWT cache; **không** có số request | `stdout` | không |
| `db` (PostgreSQL) | không có endpoint; cần exporter (ngoài A1.4) | `stderr`, có `log_connections` | không |
| `node-agent` | không có | `stdout` | không |
| kube-apiserver, scheduler, controller-manager | `/metrics` theo định dạng Prometheus | file `.log` trong `/var/log/pods` | tắt (B6) |
| kubelet | `/metrics`, `/metrics/cadvisor`, `/metrics/resource`, `/metrics/probes` | journald (B6) | tắt (B6) |
| Trạng thái object (Deployment, Pod) | **không** có dạng metric — chỉ đọc qua API | — | — |

Khúc còn trống của pipeline bài 162: không có scraper, không có kho chuỗi thời gian, không có
dashboard. B3 lấp riêng khúc "bộ thu thập" cho metric **tài nguyên** — không lấp gì khác.

Ý chính cho ADR:

1. Port-forward đi API server → kubelet → runtime mở kết nối **bên trong** network namespace của Pod
   tới `localhost:3001`, không đi qua giao diện mạng của Pod nên NetworkPolicy không áp dụng. Bài 366:
   quyền `create` trên `pods/portforward` có thể vượt qua các biện pháp kiểm soát ở tầng mạng — cấp nó
   cẩn trọng như cấp `exec`.
2. Không có nguồn metric nào. Gần nhất là đếm dòng access log của nginx — tức trụ cột **log**, không
   phải metric. Ghi khoảng trống này vào ADR.
3. `apiserver_request_total` mang `[STABLE]` trong `HELP`: không đổi tên, không đổi kiểu. Các endpoint
   phụ của kubelet "không có cùng vòng đời" với `/metrics` — panel dựng trên chúng phải rà lại mỗi lần
   nâng cấp.
4. Bộ nhớ là metric **tài nguyên** — kubelet đo trên node. Số bản sao sẵn sàng là **trạng thái
   object** — nằm ở API server; muốn nó thành metric phải có kube-state-metrics, kết nối tới API server
   rồi expose endpoint riêng.

</details>

## B2. [BS-11A-02] Điều tra: đọc metric là hành động được ủy quyền

**Loại:** Điều tra.

**Bối cảnh:** khi dự án có scraper thật (giai đoạn 23), nó sẽ chạy dưới một danh tính riêng. Trước
khi thiết kế danh tính đó, đội cần biết từng nguồn ở B1 được bảo vệ bằng gì: endpoint của thành phần
Kubernetes đòi **ủy quyền**, còn `/metrics` của PostgREST thì không có xác thực nào — chỉ có mạng.

**Yêu cầu:**

1. Trong namespace thử nghiệm `bookstore-thu`, một ServiceAccount `scraper`.
2. Chứng minh token của `scraper` được **xác thực** nhưng bị **từ chối** ở `/metrics` của
   kube-apiserver.
3. Một ClusterRole và một ClusterRoleBinding cùng tên `bookstore-thu-metrics`, mang label dự án, cho
   `scraper` đọc **đúng** `/metrics` của API server. Chứng minh: `/metrics` trả dữ liệu thật;
   `/metrics/resource`, `/metrics/cadvisor`, `/debug/pprof` vẫn bị từ chối.
4. Xóa ServiceAccount, ClusterRole, ClusterRoleBinding thử nghiệm khi xong.

**Ràng buộc:** tự viết ClusterRole, không bind vào ClusterRole có sẵn. Token chỉ nằm trong biến shell.
Không đổi cờ nào của control plane.

**Đọc lại:**

- Bài [160](../160-system-metrics-vi.md#metric-trong-kubernetes-metrics-in-kubernetes) — đoạn "nếu
  cluster của bạn dùng RBAC" và manifest ví dụ ngay sau đó.
- Bài [119](../119-controlling-access-vi.md) — ba chặng authentication → authorization → admission
  (giai đoạn 9, Lab 9a).
- Bài [120](../120-rbac-good-practices-vi.md) — quyền tối thiểu.

**Bàn giao:** `docs/adr/NNNN-quyen-doc-metric.md`; `~/lab-evidence/11a/b2-*.txt`.

**Nghiệm thu:**

- `PASS:` trước khi bind: `can-i` trả `no`, API server trả mã khác `200` cho token của `scraper`.
- `PASS:` sau khi bind: `can-i` trả `yes`, API server trả `200` và nội dung có `apiserver_request_total`.
- `PASS:` trong bốn đường dẫn `/metrics`, `/metrics/resource`, `/metrics/cadvisor`, `/debug/pprof`,
  đúng một đường được mở.
- `PASS:` object thử nghiệm đã xóa; ADR tồn tại.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-quyen-doc-metric.md`:

1. Mã trả về là `403` chứ không phải `401`. Chặng nào đã cho qua, chặng nào chặn?
2. Vì sao quyền trên `/metrics` phải là ClusterRole, không thể là Role trong một namespace?
3. Viết bảng quyền cho một scraper tương lai của dự án với ba nguồn ở B1: `/metrics` của API server,
   `/metrics/resource` của kubelet, `/metrics` của PostgREST. Nguồn nào cần RBAC, nguồn nào chỉ cần
   NetworkPolicy? Vì sao **không** nên cấp `nodes/proxy` cho scraper (đọc cảnh báo ở mục *Truy vấn
   log* của bài [159](../159-system-logs-vi.md#truy-vấn-log-log-query))?

<details>
<summary>Gợi ý</summary>

- Kiểm quyền không cần token: `kubectl auth can-i get /metrics --as=system:serviceaccount:<ns>:<sa>`.
- Token có thời hạn: `kubectl -n bookstore-thu create token scraper` (Lab 9a).
- Địa chỉ API server lấy từ kubeconfig:
  `kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'`.
- `nonResourceURLs` khớp **chuỗi đường dẫn**. Đừng chọn `/healthz`, `/livez`, `/readyz`, `/version` để
  thử phạm vi: mọi danh tính đã xác thực vốn được đọc chúng qua ClusterRole `system:public-info-viewer`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã tạo ServiceAccount, CHƯA tạo ClusterRole</summary>

Phần 1 — trước khi bind:

```bash
EV=~/lab-evidence/11a; AS=system:serviceaccount:bookstore-thu:scraper
SRV="$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
kubectl auth can-i get /metrics --as="$AS" > "$EV/b2-can-i-truoc.txt" 2>&1
TOK="$(kubectl -n bookstore-thu create token scraper)"
C1="$(curl -sk -o /dev/null -w '%{http_code}' -H "Authorization: Bearer $TOK" "$SRV/metrics")"
echo "truoc khi bind: HTTP $C1" | tee "$EV/b2-http-truoc.txt"
{ grep -qx 'no' "$EV/b2-can-i-truoc.txt" && test "$C1" != '200'; } \
  && echo "PASS: token hop le nhung bi tu choi (HTTP $C1)"
```

Phần 2 — sau khi apply ClusterRole và ClusterRoleBinding:

```bash
EV=~/lab-evidence/11a; AS=system:serviceaccount:bookstore-thu:scraper
SRV="$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
TOK="$(kubectl -n bookstore-thu create token scraper)"
C2="$(curl -sk -o "$EV/b2-metrics-qua-sa.txt" -w '%{http_code}' -H "Authorization: Bearer $TOK" "$SRV/metrics")"
{ test "$(kubectl auth can-i get /metrics --as="$AS")" = 'yes' && test "$C2" = '200' && \
  grep -q '^apiserver_request_total{' "$EV/b2-metrics-qua-sa.txt"; } \
  && echo 'PASS: sau khi bind, scraper doc duoc /metrics that'
for U in /metrics /metrics/resource /metrics/cadvisor /debug/pprof; do
  printf '%-20s %s\n' "$U" "$(kubectl auth can-i get "$U" --as="$AS")"
done | tee "$EV/b2-pham-vi.txt"
test "$(grep -c ' yes$' "$EV/b2-pham-vi.txt")" -eq 1 && echo 'PASS: dung mot duong dan duoc mo'
```

Phần 3 — sau khi dọn:

```bash
test "$(kubectl get clusterrole,clusterrolebinding --no-headers 2>/dev/null | grep -c 'bookstore-thu-metrics')" -eq 0 \
  && ! kubectl -n bookstore-thu get serviceaccount scraper >/dev/null 2>&1 \
  && echo 'PASS: da xoa object thu nghiem cua B2'
ls ~/bookstore/docs/adr/*-quyen-doc-metric.md >/dev/null 2>&1 && echo 'PASS: co ADR quyen doc metric'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/11a/b2-rbac.yaml — ví dụ của bài 160, đổi tên theo quy ước của sprint
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: bookstore-thu-metrics
  labels:
    app.kubernetes.io/part-of: bookstore
rules:
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: bookstore-thu-metrics
  labels:
    app.kubernetes.io/part-of: bookstore
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: bookstore-thu-metrics
subjects:
- kind: ServiceAccount
  name: scraper
  namespace: bookstore-thu
```

```bash
kubectl apply -f ~/lab-work/11a/ns-thu.yaml
kubectl -n bookstore-thu create serviceaccount scraper
# chạy phần 1 của script nghiệm thu, rồi:
kubectl apply -f ~/lab-work/11a/b2-rbac.yaml
# chạy phần 2, rồi dọn:
kubectl delete -f ~/lab-work/11a/b2-rbac.yaml
kubectl -n bookstore-thu delete serviceaccount scraper
```

Ý chính cho ADR:

1. API server nhận ra token là của `system:serviceaccount:bookstore-thu:scraper` — authentication đã
   qua. Nó chặn ở authorization: RBAC không có rule nào cho `get` trên `/metrics`. `401` mới là lỗi xác
   thực.
2. `/metrics` không thuộc namespace nào — nó không phải resource. `nonResourceURLs` chỉ có nghĩa trong
   ClusterRole, bind bằng ClusterRoleBinding.
3. API server: ClusterRole `nonResourceURLs: ["/metrics"]`. Kubelet: quyền trên subresource
   `nodes/metrics` — đúng như ClusterRole mà metrics-server mang theo ở B3. PostgREST: không RBAC nào
   áp dụng; bảo vệ duy nhất là NetworkPolicy chặn cổng `admin`, nên scraper tương lai cần một rule
   ingress riêng tới cổng đó. Không cấp `nodes/proxy`: bài 159 cảnh báo nó mở cả các API kubelet có
   thể thực thi lệnh trong container.

</details>

## B3. [BS-11A-03] Vận hành: cài metrics-server và chữa lỗi certificate

**Loại:** Vận hành — thay đổi hạ tầng nền, định nghĩa mốc `04-metrics-ready`.

**Bối cảnh:** B1 cho thấy kubelet đã phát `/metrics/resource` nhưng **không ai tổng hợp** nó, nên
`kubectl top` không có gì để hiển thị và sprint 18 (HPA cho `web`) không có số đo. metrics-server lấp
đúng khúc "bộ thu thập" cho metric tài nguyên và đăng ký Metrics API vào API server. Trên cluster
kubeadm, nó gần như luôn **không `Ready` ở lần cài đầu** — chẩn đoán được nguyên nhân từ log của chính
nó là phần học thật của ticket này, nên **không** đưa sẵn cờ chữa lỗi vào trước.

**Yêu cầu:**

1. Cài metrics-server đúng version ở [bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00),
   bằng manifest phát hành chính thức. Lưu manifest nguyên trạng vào
   `platform/metrics-server/components.yaml`.
2. **Trước khi apply**, đối chiếu manifest: đúng image, có đăng ký APIService của Metrics API, có cờ
   chọn kiểu địa chỉ khi gọi kubelet, và **không** tắt kiểm tra certificate.
3. Apply nguyên trạng. Chứng minh ba triệu chứng — Pod không `Ready`, APIService không `Available`,
   `kubectl top` báo lỗi — và tìm nguyên nhân trong **log của metrics-server**.
4. Chọn đường sửa, ghi đánh đổi vào ADR, rồi sửa bằng **một thay đổi có ghi lại trong repo**
   (`platform/metrics-server/`), không sửa file manifest đã lưu.
5. Sau khi sửa: APIService `Available`; `kubectl top node` có đủ ba node với số liệu khác 0; `kubectl top
   pod -n bookstore` có mọi Pod đang chạy của dự án, và theo từng container cho `db-0`; Metrics API thô
   có trường cửa sổ thời gian.
6. Chứng minh metrics-server **không** phải Prometheus: Metrics API chỉ có hai loại tài nguyên, `kubectl
   top` không có cờ khoảng thời gian.
7. `platform/metrics-server/README.md`: cách cài lại từ repo (dùng ở lần dựng lại cluster), version —
   link về A1.4, không chép số — và vì sao có bước vá. Ghi nợ `TD-16` vào `docs/tech-debt.md`.
8. ⭐ Đối chiếu bộ nhớ của `postgres` trong `kubectl top pod db-0 --containers` với
   `container_memory_working_set_bytes` của kubelet: cùng một nguồn.

**Ràng buộc:** không sửa cấu hình kubelet, không bật `serverTLSBootstrap`, không restart kubelet
(giai đoạn 20). Không cài bằng Helm, không dùng manifest khác. metrics-server **không** mang label dự
án; `teardown.sh` không được gỡ nó.

**Đọc lại:**

- Bài [160](../160-system-metrics-vi.md#metric-trong-kubernetes-metrics-in-kubernetes) — `/metrics/resource`
  của kubelet.
- Bài [162](../162-observability-vi.md#metrics) — vị trí của bộ thu thập trong pipeline.
- Bài [342](../342-hpa-walkthrough-vi.md#trước-khi-bạn-bắt-đầu-before-you-begin) — Metrics Server thu
  metric từ kubelet và phục vụ qua một APIService.
- Bài [180](../180-apiserver-aggregation-vi.md#tầng-tổng-hợp-aggregation-layer) — tầng tổng hợp API.
- metrics-server: [README, bảng tương thích](https://github.com/kubernetes-sigs/metrics-server#compatibility-matrix),
  [FAQ — certificate của kubelet](https://github.com/kubernetes-sigs/metrics-server/blob/master/FAQ.md).

**Bàn giao:** `platform/metrics-server/{components.yaml,patch-kubelet-insecure-tls.json,README.md}`;
`docs/adr/NNNN-metrics-server.md`; `docs/tech-debt.md` có `TD-16`; `~/lab-evidence/11a/b3-*.txt`.

**Nghiệm thu:**

- `PASS:` bốn gate đối chiếu manifest trước khi apply.
- `PASS:` (sau khi apply nguyên trạng) Pod chưa `Ready`; log có lỗi certificate `x509`; APIService
  `Available=False`; `kubectl top` lỗi.
- `PASS:` (sau khi sửa) cờ sửa xuất hiện đúng một lần; metrics-server `Ready` đúng image đã khóa; log
  mới không còn `x509`; APIService `Available=True`.
- `PASS:` `kubectl top node` đủ ba node, CPU và bộ nhớ khác 0; `kubectl top pod -n bookstore` đủ mọi Pod
  đang chạy; `db-0` có dòng cho container `postgres`; Metrics API thô có `window`.
- `PASS:` nhóm `metrics.k8s.io` có đúng hai tài nguyên; `kubectl top` không có cờ thời gian.
- `PASS:` ba file trong `platform/metrics-server/`, ADR, và `TD-16` trong sổ nợ.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-metrics-server.md`:

1. Nguyên nhân gốc là gì? Hai đường sửa là gì, mỗi đường đánh đổi gì, vì sao sprint này chọn đường đó
   và đường còn lại thuộc giai đoạn nào?
2. Kể chuỗi phụ thuộc từ `kubectl top pod db-0` ngược về nơi đo. Khi `kubectl top` lỗi, bạn đọc log
   của thành phần nào **đầu tiên**, vì sao không bắt đầu từ chính `kubectl top`?
3. Nêu ba câu hỏi giám sát về Bookstore mà metrics-server **không** trả lời được, mỗi câu chỉ ra thành
   phần nào mới trả lời được.
4. Vì sao metrics-server không mang label dự án và không nằm trong `deploy.sh`? Nó khác Deployment
   `web` ở điểm nào về vòng đời ([mục 3.3 của file dự án](DU-AN-BOOKSTORE.md#33-hạ-tầng-nền-của-cluster))?

<details>
<summary>Gợi ý</summary>

1. Manifest phát hành có dạng
   `https://github.com/kubernetes-sigs/metrics-server/releases/download/<version>/components.yaml`.
2. Log của Deployment: `kubectl -n kube-system logs deploy/metrics-server --tail=40`. Chờ ít nhất một
   chu kỳ thu thập trước khi kết luận log không có gì.
3. Trạng thái APIService: `.status.conditions[?(@.type=="Available")]`.
4. Một JSON patch `add` vào cuối mảng `args` của container đầu tiên: đường dẫn kết thúc bằng `/args/-`.
   `kubectl patch` nhận patch từ file bằng `--patch-file`. Chạy **một lần** — `add` vào `-` luôn nối
   thêm.
5. Metrics API thô: `kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes/<node>`.

</details>

<details>
<summary>Script nghiệm thu — ba phần, chạy đúng thời điểm ghi ở đầu mỗi phần</summary>

Phần 1 — sau khi tải manifest về `platform/metrics-server/components.yaml`, **trước** khi apply.
Điền `MS_VERSION` bằng giá trị đọc ở A1.4:

```bash
MS_VERSION='<điền version từ A1.4>'
F=~/bookstore/platform/metrics-server/components.yaml
case "$MS_VERSION" in
  v[0-9]*.[0-9]*.[0-9]*) echo "PASS: MS_VERSION = $MS_VERSION" ;;
  *) echo 'FAIL: chua dien MS_VERSION theo A1.4' ;;
esac
grep -q "image: registry.k8s.io/metrics-server/metrics-server:${MS_VERSION}" "$F" && echo 'PASS: image dung version A1.4'
grep -q 'name: v1beta1.metrics.k8s.io' "$F" && echo 'PASS: manifest dang ky APIService v1beta1.metrics.k8s.io'
grep -q -- '--kubelet-preferred-address-types=' "$F" && echo 'PASS: co co chon kieu dia chi kubelet'
grep -q -- '--kubelet-insecure-tls' "$F" && echo 'FAIL: manifest da tat kiem tra certificate' \
  || echo 'PASS: manifest thuong nguon KHONG tat kiem tra certificate'
```

Phần 2 — ngay sau khi apply nguyên trạng, **trước** khi vá:

```bash
EV=~/lab-evidence/11a
for i in $(seq 1 24); do
  kubectl -n kube-system logs deploy/metrics-server --tail=40 > "$EV/b3-log-hong.txt" 2>&1
  grep -qiE 'x509|failed to verify certificate' "$EV/b3-log-hong.txt" && break
  sleep 5
done
R="$(kubectl -n kube-system get deploy metrics-server -o jsonpath='{.status.readyReplicas}')"
test "${R:-0}" -eq 0 && echo 'PASS: metrics-server chua Ready'
grep -qiE 'x509|failed to verify certificate' "$EV/b3-log-hong.txt" && echo 'PASS: log chi ra certificate cua kubelet'
AV="$(kubectl get apiservice v1beta1.metrics.k8s.io -o jsonpath='{.status.conditions[?(@.type=="Available")].status}')"
test "$AV" = 'False' && echo 'PASS: APIService chua Available'
kubectl top node >/dev/null 2>&1 && echo 'FAIL: kubectl top da chay' || echo 'PASS: kubectl top van loi'
```

Nếu `readyReplicas` đã là `1` ngay lần đầu, kubelet của bạn đã có certificate do CA của cluster ký —
tốt hơn baseline. Ghi lý do vào ADR, bỏ bước vá, đi thẳng tới phần 3.

Phần 3 — sau khi vá:

```bash
NS=bookstore; EV=~/lab-evidence/11a; MS_VERSION='<điền version từ A1.4>'
kubectl -n kube-system rollout status deploy/metrics-server --timeout=300s
kubectl -n kube-system get deploy metrics-server \
  -o jsonpath='{range .spec.template.spec.containers[0].args[*]}{@}{"\n"}{end}' > "$EV/b3-args.txt"
test "$(grep -cx -- '--kubelet-insecure-tls' "$EV/b3-args.txt")" -eq 1 && echo 'PASS: co sua xuat hien dung mot lan'
{ test "$(kubectl -n kube-system get deploy metrics-server -o jsonpath='{.status.readyReplicas}')" -ge 1 && \
  test "$(kubectl -n kube-system get deploy metrics-server -o jsonpath='{.spec.template.spec.containers[0].image}')" \
       = "registry.k8s.io/metrics-server/metrics-server:${MS_VERSION}"; } && echo 'PASS: metrics-server Ready dung image'
for i in $(seq 1 30); do
  AV="$(kubectl get apiservice v1beta1.metrics.k8s.io -o jsonpath='{.status.conditions[?(@.type=="Available")].status}')"
  test "$AV" = 'True' && break; sleep 10
done
test "$AV" = 'True' && echo 'PASS: APIService Available'
kubectl -n kube-system logs deploy/metrics-server --tail=20 > "$EV/b3-log-sau.txt" 2>&1
grep -qiE 'x509|failed to verify certificate' "$EV/b3-log-sau.txt" && echo 'FAIL: van loi certificate' \
  || echo 'PASS: log moi khong con loi certificate'

cpu_m() { case "$1" in *m) echo "${1%m}" ;; *) echo "$(( $1 * 1000 ))" ;; esac; }
mem_mi() { case "$1" in *Ki) echo "$(( ${1%Ki} / 1024 ))" ;; *Mi) echo "${1%Mi}" ;; *Gi) echo "$(( ${1%Gi} * 1024 ))" ;; *) echo 0 ;; esac; }
for i in $(seq 1 30); do kubectl top node --no-headers > "$EV/b3-top-node.txt" 2>/dev/null && break; sleep 10; done
OK=0
while read -r n c cp m mp; do
  [ "$(cpu_m "$c")" -gt 0 ] && [ "$(mem_mi "$m")" -gt 0 ] && OK=$((OK + 1))
done < "$EV/b3-top-node.txt"
test "$OK" -eq "$(kubectl get nodes --no-headers | wc -l)" && echo 'PASS: kubectl top node du moi node, so lieu khac 0'

RUN="$(kubectl -n "$NS" get pods --field-selector=status.phase=Running --no-headers | wc -l)"
for i in $(seq 1 24); do
  kubectl top pod -n "$NS" --no-headers > "$EV/b3-top-pod.txt" 2>/dev/null
  test "$(wc -l < "$EV/b3-top-pod.txt")" -ge "$RUN" && break; sleep 5
done
test "$(wc -l < "$EV/b3-top-pod.txt")" -ge "$RUN" && echo "PASS: kubectl top pod co du $RUN Pod dang chay cua bookstore"
kubectl top pod db-0 -n "$NS" --containers --no-headers | tee "$EV/b3-top-db.txt" | grep -q ' postgres ' \
  && echo 'PASS: top theo container cua db-0 co postgres'
N1="$(kubectl get nodes -o jsonpath='{.items[1].metadata.name}')"
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes/$N1" | grep -q '"window"' && echo 'PASS: Metrics API co cua so thoi gian'

test "$(kubectl api-resources --api-group=metrics.k8s.io --no-headers | wc -l)" -eq 2 \
  && echo 'PASS: Metrics API chi co nodes va pods'
test "$(kubectl top node --help 2>&1 | grep -ciE -- '--since|--until|--range')" -eq 0 \
  && echo 'PASS: kubectl top khong co co khoang thoi gian'

cd ~/bookstore
{ test -s platform/metrics-server/components.yaml && test -s platform/metrics-server/patch-kubelet-insecure-tls.json \
  && test -s platform/metrics-server/README.md; } && echo 'PASS: du ghi chep trong platform/metrics-server'
{ ls docs/adr/*-metrics-server.md >/dev/null 2>&1 && grep -q 'TD-16' docs/tech-debt.md; } \
  && echo 'PASS: co ADR va TD-16'
```

⭐ Đối chiếu bộ nhớ — cùng nguồn, lệch nhau một chút vì hai lần đọc ở hai thời điểm:

```bash
NS=bookstore
TOPM="$(kubectl top pod db-0 -n "$NS" --containers --no-headers | awk '$2=="postgres"{print $4}')"
DBN="$(kubectl -n "$NS" get pod db-0 -o jsonpath='{.spec.nodeName}')"
RAWM="$(kubectl get --raw "/api/v1/nodes/$DBN/proxy/metrics/resource" | grep '^container_memory_working_set_bytes{' \
  | grep 'container="postgres"' | grep 'pod="db-0"' | awk '{printf "%d", $2/1048576}')"
echo "kubectl top=$TOPM | kubelet=${RAWM}Mi"
T="${TOPM%Mi}"; test "$T" -gt 0 && test $((RAWM * 2)) -ge "$T" && test $((T * 2)) -ge "$RAWM" \
  && echo 'PASS: hai con so cung bac — cung mot nguon'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```bash
cd ~/bookstore && mkdir -p platform/metrics-server
MS_VERSION='<điền version từ A1.4>'
curl -fsSL -o platform/metrics-server/components.yaml \
  "https://github.com/kubernetes-sigs/metrics-server/releases/download/${MS_VERSION}/components.yaml"
# phần 1 của script nghiệm thu, rồi:
kubectl apply -f platform/metrics-server/components.yaml
# phần 2, rồi:
cat > platform/metrics-server/patch-kubelet-insecure-tls.json <<'EOF'
[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]
EOF
kubectl -n kube-system patch deployment metrics-server --type=json \
  --patch-file platform/metrics-server/patch-kubelet-insecure-tls.json
```

Khung `platform/metrics-server/README.md`:

```markdown
# metrics-server — hạ tầng nền, cài ở Lab 11a

- Version: dòng `metrics-server` ở bảng A1.4 của Lab 00. `components.yaml` là bản phát hành
  nguyên trạng của đúng version đó.
- Cài lại: `kubectl apply -f components.yaml`, rồi vá đúng MỘT lần:
  `kubectl -n kube-system patch deployment metrics-server --type=json --patch-file patch-kubelet-insecure-tls.json`
- Vì sao vá: certificate phục vụ của kubelet do kubeadm cấp là tự ký — ADR NNNN-metrics-server, nợ TD-16.
- Kiểm: `kubectl get apiservice v1beta1.metrics.k8s.io` Available=True; `kubectl top node` đủ ba node.
- Không mang label dự án; `scripts/teardown.sh` không gỡ nó.
```

Thêm vào `docs/tech-debt.md`:

```markdown
| TD-16 | metrics-server bỏ kiểm tra certificate của kubelet (`--kubelet-insecure-tls`) | 11a | giai đoạn 20 (`serverTLSBootstrap` + duyệt CSR); chấp nhận có điều kiện trên mạng lab cô lập | mở | docs/adr/NNNN-metrics-server.md |
```

Ý chính cho ADR:

1. kubeadm cấp cho kubelet mỗi node một certificate phục vụ **tự ký** trên cổng `10250`, không do CA
   của cluster ký. metrics-server gọi kubelet qua địa chỉ IP, kiểm certificate đúng chuẩn và từ chối.
   Đường A — `--kubelet-insecure-tls`: kết nối vẫn mã hóa nhưng không xác thực được danh tính kubelet;
   chấp nhận được trên mạng lab cô lập, không phải ở production. Đường B — bật `serverTLSBootstrap`
   trong cấu hình kubelet cả ba node, restart kubelet, duyệt CSR: đúng đắn, nhưng sửa cấu hình node —
   giai đoạn 20, bài [224](../224-kubelet-config-file-vi.md); lab này cấm và gate cuối kiểm bằng
   checksum.
2. `kubectl top` → API server → APIService `v1beta1.metrics.k8s.io` → Service `metrics-server` → Pod
   metrics-server → kubelet `/metrics/resource` trên từng node. Ba triệu chứng ở phần 2 là **một**
   nguyên nhân lan xuôi chuỗi; đọc log của metrics-server trước vì đó là mắt xích đầu tiên báo lỗi
   thật, `kubectl top` chỉ thấy hậu quả.
3. Ví dụ: "bộ nhớ của `db-0` tuần trước" — cần kho chuỗi thời gian (Prometheus); "`web` có mấy bản sao
   sẵn sàng" — trạng thái object, kube-state-metrics; "`api` phục vụ bao nhiêu request" — metric của
   ứng dụng, không có nguồn (B1). metrics-server chỉ có CPU và bộ nhớ, trong một cửa sổ gần nhất.
4. metrics-server thuộc **cluster**, sống lâu hơn mọi sprint và nằm trong snapshot; dự án chỉ ghi lại
   cách cài để dựng lại cluster. `web` thuộc **ứng dụng**, sinh ra và mất đi theo `deploy.sh` /
   `teardown.sh` mỗi sprint.

</details>

## B4. [BS-11A-04] `node-agent` sang `bookstore-system` và đọc log trên node

**Loại:** Tính năng.

**Bối cảnh:** hôm nay log của Bookstore chỉ xem được từng container một bằng `kubectl logs`, chỉ file
mới nhất, và mất khi Pod mất. Bài 158 khuyến nghị **agent ghi log cấp node** chạy dạng DaemonSet, có
quyền vào thư mục log của mọi container trên node, không đòi ứng dụng thay đổi gì. Dự án đã có sẵn một
DaemonSet trên mọi node — `node-agent`. Nhưng muốn đọc log node nó cần `hostPath`, thứ mà profile
`restricted` của `bookstore` cấm, nên nó phải chuyển sang namespace riêng `bookstore-system` theo
[thiết kế bảo mật](DU-AN-BOOKSTORE.md#35-thiết-kế-bảo-mật). Chuyển namespace kéo theo Service, địa chỉ
mà `status-writer` của `web` gọi, NetworkPolicy, và hai script dùng chung.

**Yêu cầu:**

1. Namespace `bookstore-system` mang label dự án và Pod Security `enforce: privileged`, ở
   `k8s/system/00-namespace.yaml`. `k8s/system/kustomization.yaml` gom **mọi** object của namespace này.
2. DaemonSet và Service `node-agent` chuyển từ `k8s/app/agent/` sang `k8s/system/agent/`, namespace
   `bookstore-system`, giữ nguyên hợp đồng ([mục 4](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật)): label,
   container `agent`, cổng `http` 9100, `/node.json`, một Pod trên **mỗi** node kể cả control plane,
   Service với `internalTrafficPolicy: Local`, tài nguyên theo [mục 3.6](DU-AN-BOOKSTORE.md#36-ngân-sách-tài-nguyên-và-lập-lịch).
   Danh tính riêng của agent từ Lab 9a — ServiceAccount `node-agent`, không mount token — chuyển cùng
   sang `bookstore-system` và được khai trong `k8s/system/`. Không còn `node-agent` nào trong `bookstore`,
   kể cả ServiceAccount.
3. Agent đọc log container của mọi Pod namespace `bookstore` **trên node của nó**, qua `hostPath`
   `/var/log/pods` mount **chỉ đọc** ở đúng đường dẫn đó, và phục vụ `GET /logs.txt` — văn bản thuần,
   làm mới vài giây một lần, đúng ba loại dòng:
   - `file <số byte> <đường dẫn>` — mỗi file log (kể cả file đã xoay vòng) của container thuộc Pod
     namespace `bookstore` trên node;
   - `last <đường dẫn> <thông điệp>` — mỗi file log **hiện hành** (`<n>.log`): thông điệp của dòng cuối,
     đã bỏ phần tiền tố của định dạng log CRI;
   - dòng **cuối cùng**: `node=<tên node> files=<số file> bytes=<tổng byte> updated=<thời điểm>`.
4. Agent vẫn chạy **non-root**, không leo thang đặc quyền, bỏ mọi capability, root filesystem chỉ đọc,
   seccomp `RuntimeDefault` — và vẫn đọc được file log mà container runtime tạo ra.
5. `status-writer` của `web` lấy `/node.json` của agent **cùng node** qua tên DNS của Service ở
   namespace mới.
6. Mạng: NetworkPolicy của `bookstore` cho `web` đi ra `node-agent` ở `bookstore-system` cổng 9100. Trong
   `bookstore-system`: chặn mặc định hai chiều, chỉ mở ingress 9100 cho `web` của `bookstore`.
7. `scripts/deploy.sh`, `scripts/teardown.sh` thay bằng phiên bản Lab 11a — chép nguyên văn từ khối bên
   dưới.
8. Chứng minh vì sao agent không thể ở lại `bookstore`: một Pod dùng `hostPath` bị từ chối ở đó.

**Ràng buộc:** không `privileged: true`, không `hostNetwork`, `hostPID`, `hostPort`; không mount cả
`/var/log`, chỉ `/var/log/pods`; không ghi gì lên node. Không đổi tên Service, cổng hay đường dẫn của hợp
đồng.

**Đọc lại:**

- Bài [158](../158-logging-vi.md): [*Cách node xử lý log của container*](../158-logging-vi.md#cách-node-xử-lý-log-của-container-how-nodes-handle-container-logs),
  [*Vị trí log*](../158-logging-vi.md#log-location-node) (đoạn `/var/log/pods`),
  [*Các kiến trúc ghi log cấp cluster*](../158-logging-vi.md#cluster-level-logging-architectures),
  [*Sử dụng agent ghi log cấp node*](../158-logging-vi.md#sử-dụng-agent-ghi-log-cấp-node-using-a-node-logging-agent),
  [*Sử dụng sidecar container cùng agent ghi log*](../158-logging-vi.md#sidecar-container-with-logging-agent).
- Bài [91](../91-volumes-vi.md#hostpath) — cảnh báo về `hostPath`, mount chỉ đọc, file trên host chỉ root
  đọc được.
- Bài [115](../115-pod-security-standards-vi.md#baseline) — mục *HostPath Volumes*; bài
  [116](../116-pod-security-admission-vi.md#pod-security-admission-labels-for-namespaces) và
  [*Tài nguyên workload và Pod template*](../116-pod-security-admission-vi.md#workload-resources-and-pod-templates).
- Bài [291](../291-security-context-vi.md#set-the-security-context-for-a-pod) — `runAsGroup`,
  `supplementalGroups`.
- Bài [84](../84-network-policies-vi.md#nhắm-đến-một-namespace-theo-tên-targeting-a-namespace-by-its-name),
  [*Hành vi của các bộ chọn `to` và `from`*](../84-network-policies-vi.md#hành-vi-của-các-bộ-chọn-to-và-from-behavior-of-to-and-from-selectors).
- Bài [87](../87-service-traffic-policy-vi.md#cách-hoạt-động-how-it-works); bài
  [10](../10-dns-pod-service-vi.md#namespace-của-service-namespaces-of-services).

**Bàn giao:** `k8s/system/` (namespace, `agent/` gồm cả ServiceAccount, NetworkPolicy,
`kustomization.yaml`); `k8s/app/kustomization.yaml` không còn agent; `k8s/app/rbac/serviceaccounts.yaml`
không còn `node-agent`; NetworkPolicy của `web` trong `k8s/app/network/`;
`status-writer` của `web`; hai script mới; `docs/adr/NNNN-node-agent-log.md`;
`~/lab-evidence/11a/b4-*.txt`.

**Nghiệm thu:**

- `PASS:` `teardown.sh` gỡ sạch; `deploy.sh` hai lần liên tiếp đều thành công.
- `PASS:` `bookstore-system` mang label dự án và `enforce: privileged`; `bookstore` không còn
  `node-agent`, kể cả ServiceAccount.
- `PASS:` ServiceAccount `node-agent` ở `bookstore-system`, không mount token, DaemonSet dùng nó.
- `PASS:` DaemonSet đủ một Pod mỗi node; Service `Local`, cổng 9100.
- `PASS:` agent chạy uid khác 0, bounding set capability rỗng, không ghi được vào root filesystem lẫn
  `/var/log/pods`.
- `PASS:` một Pod dùng `hostPath` bị Pod Security từ chối ở `bookstore`.
- `PASS:` mỗi Pod `web` đọc được `/node.json` của agent **cùng node** qua tên Service mới; spec của
  `web` tham chiếu tên mới.
- `PASS:` `/logs.txt` của agent cạnh một Pod `web` có dòng tổng đúng node, dòng `file` và dòng `last`
  cho container `nginx` của Pod đó; thông điệp `last` có trong `kubectl logs` của chính container đó.
- `PASS:` mọi file trong `/logs.txt` thuộc Pod chạy trên đúng node đó; agent trên `lab-k8s-worker2`
  thấy log của `db-0`; agent trên `lab-k8s-master` báo `files=0`.
- `PASS:` một Pod ở `bookstore-thu` không gọi được agent.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-node-agent-log.md`:

1. Nếu để DaemonSet có `hostPath` ở lại `bookstore`, API server từ chối ở đâu — lúc tạo DaemonSet hay
   lúc DaemonSet controller tạo Pod? Bạn thấy lỗi ở chỗ nào (bài 116)?
2. File log do container runtime tạo có quyền `0640`, chủ `root:root`. Có ba cách cho agent đọc chúng:
   chạy uid 0, thêm group 0, `privileged`. Bạn chọn cách nào, mỗi cách cho agent thêm quyền gì trên
   node?
3. Mount `/var/log/pods` cho agent thấy log của **mọi** namespace trên node — `bookstore-staging`,
   `kube-system` — dù `/logs.txt` chỉ công bố `bookstore`. Đó là giới hạn kỹ thuật hay giới hạn tự đặt?
   Hệ quả với mô hình nhiều tenant của Lab 9b, và vì sao NetworkPolicy của `bookstore-system` là bắt
   buộc chứ không phải trang trí?
4. Vì sao mount ở đúng `/var/log/pods` mà không đổi đường dẫn bên trong container, và vì sao đọc
   `/var/log/containers` sẽ hỏng khi chỉ mount `/var/log/pods`?
5. So với hai kiểu sidecar của bài 158: vì sao agent cấp node hợp với Bookstore hơn? Một sidecar chạy
   agent đẩy log thẳng ra backend thì `kubectl logs` còn thấy gì?

Hai script, chép nguyên văn. Khối đánh dấu `[11a]` là phần mới của sprint này; phần còn lại dựng lại cơ
chế tích lũy của các sprint trước theo [mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint). Nếu
`deploy.sh` ở tag `lab-9b` của bạn có bước mà bản dưới không có (ví dụ sinh kubeconfig của CI vào
`secrets/` ở Lab 9a), giữ bước đó ở đúng chỗ của nó.

```bash
cat > ~/bookstore/scripts/deploy.sh <<'EOF'
#!/usr/bin/env bash
# Triển khai dự án Bookstore từ repo — phiên bản Lab 11a.
# Thứ tự: kiểm hạ tầng nền -> bí mật tại chỗ -> object cấp cluster, label node -> k8s/system
# (bookstore-system, node-agent) -> k8s/app (db trước, migration sau) -> k8s/tenants/*
# -> chờ mọi controller -> HTTPS qua Traefik.
set -euo pipefail
cd "$(dirname "$0")/.."
NS=bookstore
SYS=bookstore-system
HOST=bookstore.lab.local
DATA_NODE=lab-k8s-worker2
DATA_LABEL=bookstore.example.com/data
ERR="$(mktemp)"
APP="$(mktemp)"
trap 'rm -f "$ERR" "$APP"' EXIT
# secrets/ nằm ngoài thư mục gốc của kustomization nên phải nới load restrictor (Lab 3b, ADR 0007).
render() { kubectl kustomize --load-restrictor=LoadRestrictionsNone "$1"; }
# Cảnh báo "would violate PodSecurity" lúc apply là lỗi triển khai (9b): namespace enforce sẽ không tạo
# được Pod của workload đó, nên dừng ngay thay vì chờ rollout hết hạn.
apply_ps() {   # tham số chuyển nguyên cho kubectl apply
  if ! kubectl apply "$@" 2>"$ERR"; then cat "$ERR" >&2; return 1; fi
  cat "$ERR" >&2
  if grep -q 'would violate PodSecurity' "$ERR"; then
    echo 'deploy: co workload vi pham Pod Security — xem canh bao o tren' >&2
    return 1
  fi
}

# 0. Hạ tầng nền thuộc cluster, không thuộc repo: thiếu thì dừng, không tự cài.
#    Calico, IngressClass traefik, Service traefik (02-net-ready, 5b); provisioner
#    rancher.io/local-path đang chạy (03-storage-ready, 6a).
for X in 'daemonset/calico-node -n kube-system' 'ingressclass/traefik' 'service/traefik -n traefik'; do
  kubectl get $X >/dev/null 2>&1 || { echo "deploy: thieu $X - cluster chua o moc 02-net-ready"; exit 1; }
done
kubectl get storageclass -o jsonpath='{range .items[*]}{.provisioner}{"\n"}{end}' \
  | grep -qx 'rancher.io/local-path' \
  || { echo 'deploy: cluster chua co provisioner rancher.io/local-path (moc 03-storage-ready)'; exit 1; }
kubectl -n local-path-storage rollout status deployment/local-path-provisioner --timeout=120s

# 1. Bí mật sinh tại chỗ khi thiếu; secrets/ không vào git (3b, 5b). Dữ liệu DB cũ còn trên cluster
#    thì không sinh mới: mật khẩu thật nằm trong chính dữ liệu đó (6a).
mkdir -p secrets
if [ ! -s secrets/db.env ]; then
  CU="$( { kubectl -n "$NS" get pvc data-db-0 -o name 2>/dev/null || true
           kubectl get pv -o jsonpath='{range .items[*]}{.spec.claimRef.namespace}/{.spec.claimRef.name}{"\n"}{end}' \
             | grep -x "$NS/data-db-0" || true; } | wc -l)"
  if [ "$CU" -gt 0 ]; then
    echo 'deploy: thieu secrets/db.env trong khi du lieu db cu con tren cluster.'
    echo 'deploy: chep lai secrets/db.env cu, hoac chay scripts/teardown.sh de xoa du lieu cu.'
    exit 1
  fi
  ( umask 077
    printf 'postgres-password=%s\nauthenticator-password=%s\n' \
      "$(openssl rand -hex 24)" "$(openssl rand -hex 24)" > secrets/db.env )
  echo 'deploy: da sinh secrets/db.env moi'
fi
if [ ! -s secrets/tls.crt ] || [ ! -s secrets/tls.key ]; then
  ( umask 077
    openssl req -x509 -newkey rsa:2048 -nodes -days 365 -subj '/CN=bookstore.lab.local' \
      -addext 'subjectAltName=DNS:bookstore.lab.local' \
      -keyout secrets/tls.key -out secrets/tls.crt 2>/dev/null )
  echo 'deploy: da sinh certificate tu ky moi'
fi

# 2. Object cấp cluster, rồi label node giữ dữ liệu của db: đúng một node mang label (7a).
if compgen -G 'k8s/cluster/*.yaml' >/dev/null; then kubectl apply -f k8s/cluster/; fi
kubectl label node "$DATA_NODE" "${DATA_LABEL}=true" --overwrite
for N in $(kubectl get nodes -l "$DATA_LABEL" -o jsonpath='{.items[*].metadata.name}'); do
  [ "$N" = "$DATA_NODE" ] || kubectl label node "$N" "${DATA_LABEL}-"
done

# 3. [11a] Namespace hệ thống và node-agent, áp TRƯỚC ứng dụng để status-writer của web gọi được
#    agent ngay khi lên. node-agent của các sprint trước (namespace bookstore) nhường chỗ.
render k8s/system | kubectl apply -f -
kubectl -n "$NS" delete daemonset,service,serviceaccount node-agent --ignore-not-found --wait=true >/dev/null 2>&1 || true

# 4. Ứng dụng. Render một lần; áp mọi thứ trừ migration, chờ db-0, rồi mới chạy migration.
#    Job db-migrate chạy lại ở MỖI lần triển khai nên phải bất biến kết quả; pod template của Job
#    không sửa được, nên xóa Job cũ trước (4b).
render k8s/app > "$APP"
apply_ps -f "$APP" -l 'app.kubernetes.io/component!=migrate'
kubectl -n "$NS" rollout status statefulset/db --timeout=300s
kubectl -n "$NS" wait --for=condition=Ready pod/db-0 --timeout=300s

wait_job() {   # thành công khi Job Complete, thất bại ngay khi Job Failed
  local c=''
  for _ in $(seq 1 100); do
    c="$(kubectl -n "$NS" get job "$1" \
          -o jsonpath='{.status.conditions[?(@.status=="True")].type}' 2>/dev/null || true)"
    case " $c " in
      *' Complete '*) return 0 ;;
      *' Failed '*) kubectl -n "$NS" logs "job/$1" --tail=20 || true; return 1 ;;
    esac
    sleep 3
  done
  return 1
}
kubectl -n "$NS" delete job db-migrate --ignore-not-found --wait=true
OUT="$(kubectl apply -f "$APP" -l 'app.kubernetes.io/component=migrate' 2>&1)" \
  || echo "$OUT" | grep -q 'no objects passed to apply' || { echo "$OUT"; exit 1; }
echo "$OUT"
if grep -q 'would violate PodSecurity' <<<"$OUT"; then   # (9b)
  echo 'deploy: db-migrate vi pham Pod Security — xem canh bao o tren' >&2; exit 1
fi
if kubectl -n "$NS" get job db-migrate >/dev/null 2>&1; then
  wait_job db-migrate || { echo 'deploy: db-migrate that bai'; exit 1; }
fi

# 5. Tenant (9b): mỗi thư mục con của k8s/tenants là một tenant, namespace bookstore-<tên>, có ít
#    nhất một Deployment. Apply qua kiểm Pod Security, rồi chờ Available.
for D in k8s/tenants/*/; do
  [ -d "$D" ] || continue
  if [ -f "${D}kustomization.yaml" ]; then render "$D" | apply_ps -f -; else apply_ps -f "$D"; fi
  kubectl -n "bookstore-$(basename "$D")" wait --for=condition=Available deployment --all --timeout=180s
done

# 6. Chờ các controller còn lại.
for D in web api; do
  kubectl -n "$NS" rollout status "deployment/$D" --timeout=300s
done
kubectl -n "$SYS" rollout status daemonset/node-agent --timeout=300s

# 7. Đường vào (5b): Traefik phải phục vụ HTTPS bằng đúng certificate của dự án.
NP="$(kubectl -n traefik get service traefik -o jsonpath='{.spec.ports[?(@.name=="websecure")].nodePort}')"
IP="$(kubectl get node lab-k8s-worker1 -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
OK=0
for _ in $(seq 1 30); do
  if curl -fsS --max-time 5 --cacert secrets/tls.crt --resolve "${HOST}:${NP}:${IP}" \
       "https://${HOST}:${NP}/healthz" >/dev/null 2>&1; then OK=1; break; fi
  sleep 2
done
[ "$OK" -eq 1 ] || { echo "deploy: https://${HOST}:${NP}/ chua phuc vu qua Traefik"; exit 1; }

kubectl -n "$NS" get statefulset,deployment,cronjob,job,pvc -o wide
kubectl -n "$SYS" get daemonset,service -o wide
EOF

cat > ~/bookstore/scripts/teardown.sh <<'EOF'
#!/usr/bin/env bash
# Gỡ toàn bộ dự án Bookstore khỏi cluster — phiên bản Lab 11a.
# Gỡ mọi object mang label dự án (cả bookstore-system và namespace thử nghiệm), object cấp cluster,
# PV cùng thư mục dữ liệu trên node, label và taint node; rồi tự kiểm. KHÔNG đụng hạ tầng nền (Calico,
# Traefik, local-path-provisioner, metrics-server): chúng không mang label dự án.
set -euo pipefail
NS=bookstore
SEL='app.kubernetes.io/part-of=bookstore'
LP_ROOT=/opt/local-path-provisioner
WORKERS='lab-k8s-worker1 lab-k8s-worker2'
# Kind namespaced được dọn theo label ở mọi namespace (3b, 4a, 5a, 5b, 6a, 9b); 11a thêm ServiceAccount
# vì ServiceAccount node-agent chuyển namespace cùng agent.
KINDS='statefulsets,deployments,daemonsets,cronjobs,jobs,replicasets,pods,poddisruptionbudgets,services,networkpolicies,ingresses,configmaps,secrets,persistentvolumeclaims,roles,rolebindings,resourcequotas,limitranges,serviceaccounts'
# Thư mục volume tên <PV>_<namespace>_<PVC>: khớp namespace bookstore và bookstore-* (6a).
LP_FIND="sudo find $LP_ROOT -mindepth 1 -maxdepth 1 \( -name '*_bookstore_*' -o -name '*_bookstore-*_*' \)"
# Lệnh chạy trên worker. Thư mục gốc chưa từng có nghĩa là không có gì để dọn; lỗi ssh, sudo hay find
# thì trả mã khác 0 — "không kiểm được" không bao giờ được hiểu thành "không còn gì".
LP_ON_NODE="sudo -n test ! -e $LP_ROOT || $LP_FIND"

# PV do provisioner sinh không mang label dự án: nhận diện theo namespace của claim hoặc theo class;
# PV tạo tay thì mang label dự án (6a).
# Hai truy vấn chạy riêng: một cái lỗi thì hàm trả lỗi, không trả về một danh sách thiếu.
project_pvs() {
  local a b
  a="$(kubectl get pv -o jsonpath='{range .items[*]}{.metadata.name} {.spec.claimRef.namespace} {.spec.storageClassName}{"\n"}{end}')" || return 1
  b="$(kubectl get pv -l "$SEL" -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}')" || return 1
  { printf '%s\n' "$a" | awk '$2 == "bookstore" || $2 ~ /^bookstore-/ || $3 == "bookstore-retain" {print $1}'
    printf '%s\n' "$b"; } | awk 'NF' | sort -u
}
# Liệt kê thư mục dữ liệu còn lại trên một worker; mã thoát khác 0 = không kiểm được worker đó.
lp_left() { ssh -o BatchMode=yes "$1" "$LP_ON_NODE"; }

# 1. db tắt êm trước khi xóa namespace (4b).
if kubectl -n "$NS" get statefulset db >/dev/null 2>&1; then
  kubectl -n "$NS" scale statefulset db --replicas=0
  kubectl -n "$NS" wait --for=delete pod/db-0 --timeout=180s || true
fi

# 2. Namespace của dự án: bookstore, bookstore-staging, bookstore-system, bookstore-thu.
kubectl delete namespace -l "$SEL" --ignore-not-found --wait=true --timeout=300s
kubectl delete "$KINDS" --all-namespaces -l "$SEL" --ignore-not-found --wait=true --timeout=120s

# 3. PV còn lại và thư mục dữ liệu trên node — reclaimPolicy Retain giữ lại cả hai (6a).
PVS="$(project_pvs)" || { echo 'teardown: STOP - khong liet ke duoc PV cua du an'; exit 1; }
for PV in $PVS; do
  P="$(kubectl get pv "$PV" -o jsonpath='{.spec.hostPath.path}{.spec.local.path}' 2>/dev/null || true)"
  N="$(kubectl get pv "$PV" -o jsonpath='{.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0]}' 2>/dev/null || true)"
  kubectl delete pv "$PV" --ignore-not-found --wait=true --timeout=120s
  case "$P" in
    "$LP_ROOT"/?*)
      if [ "$N" = "$(hostname)" ]; then sudo rm -rf -- "$P"
      elif [ -n "$N" ]; then ssh "$N" "sudo rm -rf -- '$P'"; fi ;;
  esac
done
#    Thư mục còn sót trên mọi worker, kể cả của PV đã mất từ trước (6a).
for W in $WORKERS; do
  ssh -o BatchMode=yes "$W" "$LP_ON_NODE -exec rm -rf {} +" \
    || { echo "teardown: STOP - khong xoa duoc thu muc du lieu tren $W (ssh, sudo hoac find loi)"; exit 1; }
done

# 4. Object cấp cluster mang label dự án (StorageClass của dự án xóa cả theo tên, 6a), rồi label và
#    taint mà dự án đặt lên node (7a).
kubectl delete clusterrolebinding,clusterrole,priorityclass,storageclass -l "$SEL" --ignore-not-found
kubectl delete storageclass bookstore-retain --ignore-not-found
kubectl label node --all bookstore.example.com/data- >/dev/null 2>&1 || true
for N in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do
  for K in $(kubectl get node "$N" -o jsonpath='{range .spec.taints[*]}{.key}{"\n"}{end}' \
             | grep '^bookstore\.example\.com/' | sort -u || true); do
    kubectl taint node "$N" "${K}-"
  done
done

# 5. Tự kiểm.
# Mỗi truy vấn kiểm kê chạy riêng và phải thành công: API không trả lời hay Forbidden thì dừng —
# "không hỏi được" không bao giờ được đếm thành "không còn gì". dem <biến> <lệnh...> cộng số dòng
# output của lệnh vào biến; hàm chạy trong shell chính nên exit dừng cả script.
dem() {
  local v="$1" out; shift
  out="$("$@" 2>/dev/null)" || { echo "teardown: STOP - truy van loi, chua ket luan sach: $*"; exit 1; }
  printf -v "$v" '%d' "$(( ${!v:-0} + $(printf '%s\n' "$out" | awk 'NF' | wc -l) ))"
}
taints() {    # key của taint mà dự án đặt lên node (7a); lấy dữ liệu trước, lọc sau
  local t; t="$(kubectl get nodes -o jsonpath='{range .items[*]}{range .spec.taints[*]}{.key}{"\n"}{end}{end}')" || return 1
  printf '%s\n' "$t" | awk '/^bookstore\.example\.com\//'
}
LEFT=0; LEFT_NS=0; LEFT_PV=0; LEFT_NODE=0; LEFT_TAINT=0
dem LEFT kubectl get namespace,priorityclass,storageclass,clusterrole,clusterrolebinding -l "$SEL" --no-headers
dem LEFT kubectl get storageclass bookstore-retain --no-headers --ignore-not-found
dem LEFT_NS kubectl get all -A -l "$SEL" --no-headers
dem LEFT_NS kubectl get "$KINDS,endpointslices" -A -l "$SEL" --no-headers
dem LEFT_PV project_pvs
dem LEFT_NODE kubectl get nodes -l bookstore.example.com/data --no-headers
dem LEFT_TAINT taints
#    Mỗi worker cho một trong ba kết quả: còn thư mục, đã xác nhận không còn, hoặc không kiểm được.
#    Kết quả thứ ba dừng script: chưa kết luận sạch thì không được chụp snapshot.
LEFT_DIR=0
for W in $WORKERS; do
  L="$(lp_left "$W")" \
    || { echo "teardown: STOP - khong kiem duoc thu muc du lieu tren $W; chua ket luan sach, KHONG chup snapshot"; exit 1; }
  LEFT_DIR=$(( LEFT_DIR + $(printf '%s\n' "$L" | grep -c . || true) ))
done
if [ $(( LEFT + LEFT_NS + LEFT_PV + LEFT_NODE + LEFT_TAINT + LEFT_DIR )) -eq 0 ]; then
  echo 'teardown: sach'
else
  echo "teardown: con $LEFT object cap cluster, $LEFT_NS object co label, $LEFT_PV PV, $LEFT_NODE node co label, $LEFT_TAINT taint, $LEFT_DIR thu muc du lieu"
  exit 1
fi
# Hạ tầng nền không mang label dự án và phải còn nguyên (5b).
{ kubectl get ingressclass traefik >/dev/null 2>&1 && kubectl -n kube-system get daemonset calico-node >/dev/null 2>&1; } \
  || echo 'teardown: CANH BAO - ha tang 02-net-ready khong con nguyen (khong do script nay go)'
EOF

chmod +x ~/bookstore/scripts/*.sh
```

`deploy.sh` áp migration tách riêng bằng bộ lọc label `app.kubernetes.io/component=migrate` của
[hợp đồng](DU-AN-BOOKSTORE.md#41-tên-và-label). Giả định từ Lab 5a: mọi manifest của `k8s/app` nằm
trong `k8s/app/kustomization.yaml`, và Job `db-migrate` mang label đó ở `metadata` của chính Job. Thiếu
Job trong bản render thì bước migration được bỏ qua, không làm hỏng triển khai.

<details>
<summary>Gợi ý</summary>

1. Trình tự an toàn: `git mv k8s/app/agent k8s/system/agent`, đổi `namespace` trong hai file, bỏ hai
   dòng agent khỏi `k8s/app/kustomization.yaml`, chuyển khối ServiceAccount `node-agent` từ
   `k8s/app/rbac/serviceaccounts.yaml` sang `k8s/system/agent/serviceaccount.yaml`, viết
   `k8s/system/kustomization.yaml`, sửa NetworkPolicy và `status-writer`, chép hai script, rồi
   `./scripts/teardown.sh && ./scripts/deploy.sh`. DaemonSet vẫn giữ `serviceAccountName: node-agent`
   của Lab 9a: ServiceAccount thiếu ở namespace mới thì DaemonSet controller báo `FailedCreate`.
2. Xem quyền thật của file log từ một node:
   `ssh lab-k8s-worker1 'sudo ls -ln /var/log/pods/*/*/ | head'`.
3. `stat -c %s <file>` của busybox cho kích thước mà không cần đọc nội dung; `tail -n 1` thì cần quyền
   đọc. Một dòng log CRI có dạng `<thời điểm> <stdout|stderr> <F|P> <thông điệp>` — bỏ ba trường đầu bằng
   `cut -d' ' -f4-`.
4. Glob trong shell: `/var/log/pods/"$LOG_NAMESPACE"_*/*/*` — tên thư mục là `<namespace>_<pod>_<uid>`,
   và tên namespace không chứa `_`.
5. Label do control plane đặt sẵn cho mọi namespace: `kubernetes.io/metadata.name`. Một phần tử `to`
   có **cả** `namespaceSelector` lẫn `podSelector` nghĩa là "Pod này trong namespace kia".
6. Thử Pod Security mà không tạo gì: `kubectl -n bookstore apply --dry-run=server -f <pod>.yaml`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script teardown rồi deploy lại</summary>

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
NS=bookstore; SYS=bookstore-system; EV=~/lab-evidence/11a
SEL='app.kubernetes.io/part-of=bookstore'
cd ~/bookstore
./scripts/teardown.sh > "$EV/b4-teardown.txt" 2>&1; grep -q 'teardown: sach' "$EV/b4-teardown.txt" && echo 'PASS: teardown.sh go sach'
./scripts/deploy.sh > "$EV/b4-deploy-1.txt" 2>&1 && echo 'PASS: deploy.sh lan 1 thanh cong'
./scripts/deploy.sh > "$EV/b4-deploy-2.txt" 2>&1 && echo 'PASS: deploy.sh lan 2 thanh cong'

lbl() { kubectl get namespace "$SYS" -o jsonpath="{.metadata.labels.$1}"; }
{ [ "$(lbl 'app\.kubernetes\.io/part-of')" = 'bookstore' ] && [ "$(lbl 'pod-security\.kubernetes\.io/enforce')" = 'privileged' ]; } \
  && echo 'PASS: bookstore-system mang label du an, enforce privileged'
[ "$(sl kubectl -n "$NS" get daemonset,service,serviceaccount --field-selector metadata.name=node-agent --no-headers)" -eq 0 ] \
  && echo 'PASS: bookstore khong con node-agent, ke ca ServiceAccount'
{ [ "$(kubectl -n "$SYS" get ds node-agent -o jsonpath='{.spec.template.spec.serviceAccountName}')" = 'node-agent' ] \
  && [ "$(kubectl -n "$SYS" get sa node-agent -o jsonpath='{.automountServiceAccountToken}|{.metadata.labels.app\.kubernetes\.io/component}')" = 'false|agent' ]; } \
  && echo 'PASS: ServiceAccount node-agent o bookstore-system, khong mount token, DaemonSet dung no'

NODES="$(kubectl get nodes --no-headers | wc -l)"
[ "$(kubectl -n "$SYS" get ds node-agent -o jsonpath='{.status.numberReady}')" -eq "$NODES" ] && echo "PASS: node-agent Ready tren $NODES node"
[ "$(kubectl -n "$SYS" get svc node-agent -o jsonpath='{.spec.internalTrafficPolicy}/{.spec.ports[0].port}')" = 'Local/9100' ] \
  && echo 'PASS: Service node-agent Local, cong 9100'

agent() { kubectl -n "$SYS" get pod -l "$SEL,app.kubernetes.io/component=agent" --field-selector "spec.nodeName=$1" \
            -o jsonpath="{.items[0].$2}"; }
AG2="$(agent lab-k8s-worker2 metadata.name)"
ax() { kubectl -n "$SYS" exec "$AG2" -c agent -- "$@"; }
[ "$(ax id -u)" != '0' ] && echo 'PASS: agent chay uid khac 0'
ax grep -q '^CapBnd:[[:space:]]*0000000000000000$' /proc/1/status && echo 'PASS: bounding set capability rong'
ax touch /thu-ghi >/dev/null 2>&1 && echo 'FAIL: ghi duoc root filesystem' || echo 'PASS: root filesystem chi doc'
ax touch /var/log/pods/thu-ghi >/dev/null 2>&1 && echo 'FAIL: ghi duoc /var/log/pods' || echo 'PASS: /var/log/pods mount chi doc'

cat <<'YAML' | kubectl -n "$NS" apply --dry-run=server -f - > "$EV/b4-pss.txt" 2>&1
apiVersion: v1
kind: Pod
metadata: {name: thu-hostpath}
spec:
  securityContext: {runAsNonRoot: true, runAsUser: 65534, seccompProfile: {type: RuntimeDefault}}
  containers:
  - name: c
    image: busybox:1.37
    command: ["sleep", "60"]
    securityContext: {allowPrivilegeEscalation: false, capabilities: {drop: ["ALL"]}}
    volumeMounts: [{name: pods, mountPath: /var/log/pods, readOnly: true}]
  volumes:
  - name: pods
    hostPath: {path: /var/log/pods, type: Directory}
YAML
grep -q 'violates PodSecurity' "$EV/b4-pss.txt" && grep -q 'hostPath' "$EV/b4-pss.txt" \
  && echo 'PASS: Pod dung hostPath bi Pod Security tu choi o bookstore'

BAD=0
for P in $(kubectl -n "$NS" get pod -l "$SEL,app.kubernetes.io/component=web" --field-selector=status.phase=Running -o jsonpath='{.items[*].metadata.name}'); do
  N="$(kubectl -n "$NS" get pod "$P" -o jsonpath='{.spec.nodeName}')"
  J="$(kubectl -n "$NS" exec "$P" -c status-writer -- wget -qO- -T 5 "http://node-agent.$SYS.svc.cluster.local:9100/node.json")"
  echo "$P@$N -> $J" >> "$EV/b4-node-json.txt"
  echo "$J" | grep -Eq "\"node\" *: *\"$N\"" || BAD=1
done
[ "$BAD" -eq 0 ] && echo 'PASS: moi Pod web doc /node.json cua agent cung node qua ten moi'
{ kubectl -n "$NS" get deploy web -o yaml; kubectl -n "$NS" get configmap -o yaml; } | grep -q "node-agent\.$SYS" \
  && echo 'PASS: spec cua web tham chieu ten moi'

WEB="$(kubectl -n "$NS" get pod -l "$SEL,app.kubernetes.io/component=web" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}')"
WN="$(kubectl -n "$NS" get pod "$WEB" -o jsonpath='{.spec.nodeName}')"
WUID="$(kubectl -n "$NS" get pod "$WEB" -o jsonpath='{.metadata.uid}')"
F="/var/log/pods/${NS}_${WEB}_${WUID}/nginx/"
for i in $(seq 1 12); do
  kubectl -n "$NS" exec "$WEB" -c status-writer -- wget -qO- -T 5 "http://node-agent.$SYS.svc.cluster.local:9100/logs.txt" > "$EV/b4-logs-$WN.txt"
  grep -q "^last $F" "$EV/b4-logs-$WN.txt" && break; sleep 5
done
{ tail -n 1 "$EV/b4-logs-$WN.txt" | grep -q "^node=$WN files=[1-9]" && grep -q "^file [0-9]* $F" "$EV/b4-logs-$WN.txt"; } \
  && echo "PASS: /logs.txt tren $WN co dong tong va dong file cua nginx trong $WEB"
MSG="$(grep "^last $F" "$EV/b4-logs-$WN.txt" | head -1 | cut -d' ' -f3-)"
{ [ -n "$MSG" ] && kubectl -n "$NS" logs "$WEB" -c nginx --tail=300 | grep -Fq -- "$MSG"; } \
  && echo 'PASS: thong diep dong last co trong kubectl logs cua chinh container do'
BAD=0
for D in $(awk '$1=="file"{print $3}' "$EV/b4-logs-$WN.txt" | cut -d/ -f5 | sort -u); do
  P="$(echo "$D" | cut -d_ -f2)"
  [ "$(kubectl -n "$NS" get pod "$P" -o jsonpath='{.spec.nodeName}' 2>/dev/null)" = "$WN" ] || { echo "FAIL: $P khong o $WN"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo "PASS: moi file trong /logs.txt thuoc Pod chay tren $WN"

AG2_IP="$(agent lab-k8s-worker2 status.podIP)"; AGM_IP="$(agent lab-k8s-master status.podIP)"
kubectl -n "$NS" exec "$WEB" -c status-writer -- wget -qO- -T 5 "http://${AG2_IP}:9100/logs.txt" > "$EV/b4-logs-lab-k8s-worker2.txt"
grep -q "^file [0-9]* /var/log/pods/${NS}_db-0_" "$EV/b4-logs-lab-k8s-worker2.txt" && echo 'PASS: agent tren worker2 thay log cua db-0'
kubectl -n "$NS" exec "$WEB" -c status-writer -- wget -qO- -T 5 "http://${AGM_IP}:9100/logs.txt" | tail -n 1 \
  | grep -q '^node=lab-k8s-master files=0 ' && echo 'PASS: agent tren master khong co log nao cua bookstore'

kubectl apply -f ~/lab-work/11a/ns-thu.yaml >/dev/null
kubectl -n bookstore-thu run thu-goi-agent --rm -i --restart=Never --image=busybox:1.37 \
  -- wget -q -T 3 -O /dev/null "http://${AG2_IP}:9100/node.json" > /dev/null 2>&1 \
  && echo 'FAIL: Pod ngoai bookstore goi duoc agent' || echo 'PASS: Pod o bookstore-thu khong goi duoc agent'
ls docs/adr/*-node-agent-log.md >/dev/null 2>&1 && echo 'PASS: co ADR node-agent'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/system/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- 00-namespace.yaml
- agent/serviceaccount.yaml
- agent/daemonset.yaml
- agent/service.yaml
- network/networkpolicy.yaml
```

```yaml
# k8s/system/agent/serviceaccount.yaml — chuyển từ k8s/app/rbac/serviceaccounts.yaml (Lab 9a)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: node-agent
  namespace: bookstore-system
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: agent
    app.kubernetes.io/name: busybox
automountServiceAccountToken: false
```

```yaml
# k8s/system/00-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: bookstore-system
  labels:
    app.kubernetes.io/part-of: bookstore
    pod-security.kubernetes.io/enforce: privileged   # node-agent cần hostPath (mục 3.5 file dự án)
```

```yaml
# k8s/system/agent/daemonset.yaml — chuyển từ k8s/app/agent/ ở Lab 11a
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent
  namespace: bookstore-system
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: agent
    app.kubernetes.io/name: busybox
spec:
  selector:
    matchLabels:
      app.kubernetes.io/part-of: bookstore
      app.kubernetes.io/component: agent
  template:
    metadata:
      labels:
        app.kubernetes.io/part-of: bookstore
        app.kubernetes.io/component: agent
        app.kubernetes.io/name: busybox
    spec:
      serviceAccountName: node-agent     # Lab 9a; ServiceAccount nằm trong k8s/system/agent/
      automountServiceAccountToken: false
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      terminationGracePeriodSeconds: 10
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        runAsGroup: 65534
        supplementalGroups: [0]        # file log của runtime là 0640 root:root — đọc bằng quyền group
        seccompProfile:
          type: RuntimeDefault
      volumes:
      - name: www
        emptyDir: {}
      - name: pod-logs
        hostPath:
          path: /var/log/pods
          type: Directory
      containers:
      - name: agent
        image: busybox:1.37
        env:
        - name: NODE_NAME
          valueFrom: {fieldRef: {fieldPath: spec.nodeName}}
        - name: LOG_NAMESPACE
          value: bookstore
        command:
        - sh
        - -c
        - |
          trap 'echo "node-agent: nhan TERM"; kill "$HTTPD" 2>/dev/null; exit 0' TERM
          write_node() {
            printf '{"node":"%s","load":"%s","updated":"%s"}\n' \
              "$NODE_NAME" "$(cut -d' ' -f1 /proc/loadavg)" "$(date -Iseconds)" > /www/node.json.tmp
            mv /www/node.json.tmp /www/node.json
          }
          write_logs() {
            N=0; B=0
            {
              for F in /var/log/pods/"$LOG_NAMESPACE"_*/*/*; do
                [ -f "$F" ] || continue
                S="$(stat -c %s "$F")"; N=$((N + 1)); B=$((B + S))
                echo "file $S $F"
              done
              for F in /var/log/pods/"$LOG_NAMESPACE"_*/*/*.log; do
                [ -f "$F" ] || continue
                echo "last $F $(tail -n 1 "$F" | cut -d' ' -f4-)"
              done
              echo "node=$NODE_NAME files=$N bytes=$B updated=$(date -Iseconds)"
            } > /www/logs.txt.tmp
            mv /www/logs.txt.tmp /www/logs.txt
          }
          write_node; write_logs
          httpd -f -p 9100 -h /www &
          HTTPD=$!
          while kill -0 "$HTTPD" 2>/dev/null; do
            write_node; write_logs
            sleep 5 & wait $!
          done
          echo "node-agent: httpd da dung"
          exit 1
        ports:
        - {name: http, containerPort: 9100}
        readinessProbe:
          httpGet: {path: /node.json, port: http}
          periodSeconds: 10
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
        resources:
          requests: {cpu: 10m, memory: 16Mi}
          limits: {cpu: 50m, memory: 32Mi}
        volumeMounts:
        - {name: www, mountPath: /www}
        - {name: pod-logs, mountPath: /var/log/pods, readOnly: true}
```

```yaml
# k8s/system/agent/service.yaml — chuyển từ k8s/app/agent/, chỉ đổi namespace
apiVersion: v1
kind: Service
metadata:
  name: node-agent
  namespace: bookstore-system
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: agent
    app.kubernetes.io/name: busybox
spec:
  selector:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: agent
  internalTrafficPolicy: Local
  ports:
  - {name: http, port: 9100, targetPort: http}
```

```yaml
# k8s/system/network/networkpolicy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: bookstore-system
  labels: {app.kubernetes.io/part-of: bookstore}
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: agent-from-web
  namespace: bookstore-system
  labels: {app.kubernetes.io/part-of: bookstore}
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/part-of: bookstore
      app.kubernetes.io/component: agent
  policyTypes: ["Ingress"]
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: bookstore
      podSelector:
        matchLabels:
          app.kubernetes.io/part-of: bookstore
          app.kubernetes.io/component: web
    ports:
    - {protocol: TCP, port: 9100}
```

Trong NetworkPolicy egress của `web` ở `k8s/app/network/`, thay rule cũ trỏ tới agent trong cùng
namespace bằng rule này:

```yaml
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: bookstore-system
      podSelector:
        matchLabels:
          app.kubernetes.io/part-of: bookstore
          app.kubernetes.io/component: agent
    ports:
    - {protocol: TCP, port: 9100}
```

Trong script của `status-writer` (Deployment `web`, hoặc ConfigMap chứa script nếu bạn tách ra ở Lab
5a), đổi địa chỉ agent sang FQDN của namespace mới:

```yaml
        - name: NODE_AGENT_URL
          value: http://node-agent.bookstore-system.svc.cluster.local:9100/node.json
```

```bash
cd ~/bookstore
git mv k8s/app/agent k8s/system/agent
sed -i 's/^  namespace: bookstore$/  namespace: bookstore-system/' k8s/system/agent/*.yaml
# sửa tay: bỏ agent khỏi k8s/app/kustomization.yaml; cắt khối ServiceAccount node-agent khỏi
# k8s/app/rbac/serviceaccounts.yaml sang k8s/system/agent/serviceaccount.yaml (khối ở trên);
# viết k8s/system/kustomization.yaml, 00-namespace.yaml, network/networkpolicy.yaml, rule egress của
# web, địa chỉ trong status-writer
! grep -q 'name: node-agent' k8s/app/rbac/serviceaccounts.yaml && echo 'OK: k8s/app khong con ServiceAccount node-agent'
./scripts/teardown.sh && ./scripts/deploy.sh
```

Ý chính cho ADR:

1. Pod Security `enforce` chỉ chặn **Pod**. DaemonSet có `hostPath` vẫn được tạo (nếu namespace có
   `warn`, `kubectl` in cảnh báo); lỗi hiện ở sự kiện `FailedCreate` của DaemonSet controller khi nó tạo
   Pod — dễ bị bỏ qua. Bài 116, mục *Tài nguyên workload và Pod template*.
2. uid 0 với mọi capability đã drop: đọc được vì là chủ file, nhưng mọi file root sở hữu trên mount đều
   mở. `privileged`: gần như toàn quyền trên node — quá tay cho việc đọc. `supplementalGroups: [0]`: tiến
   trình vẫn uid 65534, chỉ thêm quyền group `root` trên đúng mount chỉ đọc — hẹp nhất. Cái giá chung:
   mọi file group-readable của `root` dưới `/var/log/pods` đều đọc được.
3. Giới hạn tự đặt: `hostPath` không lọc theo namespace, thư mục Pod sinh động theo UID. Agent nhìn
   xuyên tenant — ranh giới của Lab 9b (namespace, RBAC, quota, NetworkPolicy) bị thủng ở tầng node.
   Vì vậy agent nằm ở namespace riêng, ai được vào cổng 9100 phải bị chặn bằng NetworkPolicy, và
   `/logs.txt` chỉ công bố namespace `bookstore`.
4. `/var/log/containers/*.log` là symlink tuyệt đối trỏ vào `/var/log/pods/...`. Mount `/var/log/pods` ở
   đúng đường dẫn thì mọi đường dẫn mà kubelet và runtime dùng vẫn đúng bên trong container; đọc
   `/var/log/containers` mà không mount nó thì không có gì, mount nó mà không mount đúng `/var/log/pods`
   thì symlink trỏ vào khoảng không.
5. Agent cấp node: một agent mỗi node, ứng dụng không đổi gì, log vẫn đi qua `stdout`/`stderr` nên
   `kubectl logs` vẫn dùng được. Sidecar truyền luồng nhân đôi dung lượng log trên node; sidecar chạy
   agent đẩy thẳng ra backend nên log **không** do kubelet kiểm soát — `kubectl logs` không thấy gì.

</details>

## B5. [BS-11A-05] Diễn tập: xoay vòng log trên `lab-k8s-worker2`

**Loại:** Diễn tập sự cố — chỉ trên `lab-k8s-worker2`.

**Bối cảnh:** một tình huống trực có thật ở mọi đội: ứng dụng in một tràng lỗi dài, người trực gõ
`kubectl logs` và chỉ thấy phần đuôi, phần đầu — nơi có nguyên nhân — không còn. Diễn tập dựng lại tình
huống đó bằng một Pod thử nghiệm ghi vài chục MB log, rồi trả lời bằng **con số**: ai cắt, cắt ở đâu,
phần bị cắt nằm ở đâu, và ai còn đọc được nó.

**Yêu cầu:**

1. Đọc hai ngưỡng xoay vòng thật của kubelet trên `lab-k8s-worker2` từ cấu hình **hiệu lực**, không đọc
   file trên đĩa.
2. Pod `thu-log-flood` trong `bookstore-thu`, ghim `lab-k8s-worker2`, ghi khoảng 40 000 dòng chừng 1 KB
   ra `stdout` trong vài chục giây (không dồn hết vào một nhịp), rồi cứ vài giây in một dòng
   `FLOOD-DONE` cho tới khi bị xóa.
3. Chứng minh kubelet đã xoay vòng: thư mục log của container có từ hai file trở lên và không quá số file
   tối đa — quan sát **từ chính `node-agent` trên `lab-k8s-worker2`**, không `ssh`.
4. Chứng minh bằng số: `kubectl logs` trả không quá một file, và mất phần đầu của luồng log.
5. Xóa Pod: thư mục log của nó biến mất khỏi node — kiểm từ agent.
6. Postmortem `docs/postmortem/11a-xoay-vong-log.md`; ghi nợ `TD-17` vào `docs/tech-debt.md`.

**Ràng buộc:** không sửa cấu hình kubelet; không xóa tay file trong `/var/log/pods` — vùng đó kubelet
quản lý. Pod thử nghiệm không mang label dự án và bị xóa khi xong.

**Đọc lại:** bài [158](../158-logging-vi.md):
[*Log của Pod và container*](../158-logging-vi.md#basic-logging-in-kubernetes),
[*Cách node xử lý log của container*](../158-logging-vi.md#cách-node-xử-lý-log-của-container-how-nodes-handle-container-logs),
[*Xoay vòng log*](../158-logging-vi.md#xoay-vòng-log-log-rotation) — ghi chú về 40 MiB; tài liệu cấu hình
kubelet cho `containerLogMaxSize`, `containerLogMaxFiles` (mục 5).

**Bàn giao:** `docs/postmortem/11a-xoay-vong-log.md`; `docs/tech-debt.md` có `TD-17`;
`~/lab-evidence/11a/b5-*.txt`.

**Nghiệm thu:**

- `PASS:` đọc được cả hai ngưỡng từ `configz` của `lab-k8s-worker2`.
- `PASS:` Pod ghi xong khối log.
- `PASS:` agent trên `lab-k8s-worker2` thấy từ hai file trở lên trong thư mục log của container, không
  quá `containerLogMaxFiles`.
- `PASS:` `kubectl logs` trả không quá `containerLogMaxSize` byte, ít hơn số dòng đã ghi, và dòng đầu
  không phải dòng số `0`.
- `PASS:` xóa Pod thì thư mục log của nó biến mất.
- `PASS:` có postmortem và `TD-17`.

**Câu hỏi thiết kế** — trả lời trong postmortem:

1. Ai quyết định cắt, ai ghi file, `kubectl logs` đọc từ đâu? Với cấu hình của node bạn, `kubectl logs`
   trả nhiều nhất bao nhiêu byte?
2. Phần đầu của luồng log đang nằm ở đâu trong lúc Pod còn sống? Ai trong hệ thống của bạn đọc được nó,
   và vì sao `/logs.txt` của agent vẫn không công bố nó? Xóa Pod thì sao?
3. Để sự cố này không mất log nữa, agent còn thiếu khúc nào của pipeline bài 162? Ghi nó thành `TD-17`.

<details>
<summary>Gợi ý</summary>

- Hàm rút một trường khỏi JSON `configz` mà không cần `jq` (node lab không có `jq`):
  `cfgz() { grep -o "\"$1\":[^,}]*" "$2" | head -1 | cut -d: -f2- | tr -d '"'; }`.
- Một dòng 1000 ký tự trong busybox: `P=$(dd if=/dev/zero bs=1000 count=1 2>/dev/null | tr '\0' 'x')`.
- Vì sao phải ghi rải ra và giữ nhịp `FLOOD-DONE`: kubelet chỉ xoay vòng ở mỗi chu kỳ kiểm. Ghi dồn 40 MB
  trong vài giây thì chỉ có **một** lần xoay vòng với một file khổng lồ; container im lặng sau đó thì file
  mới nhất **rỗng** và `kubectl logs` không trả gì — kể cả dòng báo xong. Nghỉ một giây sau mỗi nghìn dòng
  là đủ để thấy nhiều lần xoay vòng.
- Đường dẫn thư mục log: `/var/log/pods/<namespace>_<pod>_<uid>/<container>/`. Lệnh trong agent:
  `kubectl -n bookstore-system exec <Pod agent> -c agent -- ls -l <thư mục>`.
- Kubelet kiểm tra xoay vòng theo chu kỳ — chờ bằng vòng lặp có điều kiện thoát.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã apply Pod thử nghiệm</summary>

```bash
EV=~/lab-evidence/11a; SYS=bookstore-system
cfgz() { grep -o "\"$1\":[^,}]*" "$2" | head -1 | cut -d: -f2- | tr -d '"'; }
to_bytes() { case "$1" in *Ki) echo $(( ${1%Ki} * 1024 )) ;; *Mi) echo $(( ${1%Mi} * 1048576 )) ;; *Gi) echo $(( ${1%Gi} * 1073741824 )) ;; *[0-9]) echo "$1" ;; *) echo 0 ;; esac; }
CLMS="$(cfgz containerLogMaxSize "$EV/b0-configz-lab-k8s-worker2.json")"
CLMF="$(cfgz containerLogMaxFiles "$EV/b0-configz-lab-k8s-worker2.json")"
MAXB="$(to_bytes "$CLMS")"
echo "containerLogMaxSize=$CLMS ($MAXB byte) containerLogMaxFiles=$CLMF" | tee "$EV/b5-nguong.txt"
{ test "$MAXB" -gt 0 && test "${CLMF:-0}" -ge 2; } && echo 'PASS: doc duoc hai nguong tu cau hinh hieu luc'

for i in $(seq 1 60); do kubectl -n bookstore-thu logs thu-log-flood --tail=1 2>/dev/null | grep -q 'FLOOD-DONE' && break; sleep 5; done
kubectl -n bookstore-thu logs thu-log-flood --tail=1 | grep -q 'FLOOD-DONE' && echo 'PASS: Pod da ghi xong khoi log'

AG2="$(kubectl -n "$SYS" get pod -l app.kubernetes.io/component=agent --field-selector spec.nodeName=lab-k8s-worker2 -o jsonpath='{.items[0].metadata.name}')"
U="$(kubectl -n bookstore-thu get pod thu-log-flood -o jsonpath='{.metadata.uid}')"
DIR="/var/log/pods/bookstore-thu_thu-log-flood_$U/flood"
for i in $(seq 1 60); do
  NF="$(kubectl -n "$SYS" exec "$AG2" -c agent -- ls -1 "$DIR" | wc -l)"
  test "$NF" -ge 2 && break; sleep 5
done
kubectl -n "$SYS" exec "$AG2" -c agent -- ls -l "$DIR" | tee "$EV/b5-thu-muc-log.txt"
{ test "$NF" -ge 2 && test "$NF" -le "$CLMF"; } && echo "PASS: agent thay $NF file da xoay vong, khong qua $CLMF"

# File hiện hành có thể vừa vượt ngưỡng giữa hai chu kỳ kiểm — chờ kubelet xoay nốt.
for i in $(seq 1 24); do
  kubectl -n bookstore-thu logs thu-log-flood > "$EV/b5-kubectl-logs.txt"
  test "$(wc -c < "$EV/b5-kubectl-logs.txt")" -le "$MAXB" && break; sleep 5
done
GOT_B="$(wc -c < "$EV/b5-kubectl-logs.txt")"; GOT_L="$(wc -l < "$EV/b5-kubectl-logs.txt")"
FIRST="$(head -1 "$EV/b5-kubectl-logs.txt" | cut -d' ' -f1)"
echo "kubectl logs: $GOT_B byte, $GOT_L dong, dong dau so $FIRST" | tee -a "$EV/b5-nguong.txt"
{ test "$GOT_B" -le "$MAXB" && test "$GOT_L" -lt 40001 && test "$FIRST" != '0'; } \
  && echo 'PASS: kubectl logs chi tra file moi nhat, phan dau da roi khoi tam voi'

kubectl -n bookstore-thu delete pod thu-log-flood --wait=true
for i in $(seq 1 30); do kubectl -n "$SYS" exec "$AG2" -c agent -- test -d "$DIR" || break; sleep 5; done
kubectl -n "$SYS" exec "$AG2" -c agent -- test -d "$DIR" && echo 'FAIL: thu muc log van con' \
  || echo 'PASS: xoa Pod la kubelet don thu muc log cua no'
{ test -s ~/bookstore/docs/postmortem/11a-xoay-vong-log.md && grep -q 'TD-17' ~/bookstore/docs/tech-debt.md; } \
  && echo 'PASS: co postmortem va TD-17'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/11a/thu-log-flood.yaml
apiVersion: v1
kind: Pod
metadata:
  name: thu-log-flood
  namespace: bookstore-thu
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  restartPolicy: Never
  containers:
  - name: flood
    image: busybox:1.37
    command:
    - sh
    - -c
    - |
      P=$(dd if=/dev/zero bs=1000 count=1 2>/dev/null | tr '\0' 'x')
      i=0
      while [ $i -lt 40000 ]; do
        echo "$i $P"; i=$((i+1))
        if [ $((i % 1000)) -eq 0 ]; then sleep 1; fi
      done
      trap 'exit 0' TERM
      while true; do echo "FLOOD-DONE $i"; sleep 2 & wait $!; done
```

```bash
kubectl apply -f ~/lab-work/11a/ns-thu.yaml -f ~/lab-work/11a/thu-log-flood.yaml
```

Thêm vào `docs/tech-debt.md`:

```markdown
| TD-17 | Metric và log chưa có nơi lưu dài hạn: không có kho chuỗi thời gian, agent chỉ công bố chỉ mục log trên node | 11a | giai đoạn 23; chấp nhận có điều kiện ở quy mô lab | mở | docs/postmortem/11a-xoay-vong-log.md |
```

Ý chính cho postmortem:

1. Kubelet quyết định — theo `containerLogMaxSize` và `containerLogMaxFiles` — và chỉ thị qua CRI để
   runtime ghi vào `/var/log/pods/...`. `kubectl logs` đi API server → kubelet → **file mới nhất**. Với
   giá trị mặc định mà bài 158 nêu, nhiều nhất 10 MiB.
2. Ở các file đã xoay vòng cạnh file hiện hành, trên đĩa `lab-k8s-worker2`. Agent của B4 **đọc được** vì
   mount cả `/var/log/pods`, nhưng `/logs.txt` chỉ công bố namespace `bookstore` — giới hạn tự đặt ở B4.
   Xóa Pod thì kubelet dọn cả thư mục: log sống theo Pod.
3. Khúc "kho lưu trữ" có vòng đời riêng, độc lập với node, Pod, container — bài 158 gọi đó là ghi log
   cấp cluster, và Kubernetes không cung cấp backend nào. Agent hiện chỉ là khúc "bộ thu thập" chưa có
   đích để đẩy tới.

</details>

## B6. [BS-11A-06] Điều tra: log của chính Kubernetes và trace

**Loại:** Điều tra — chỉ đọc.

**Bối cảnh:** không phải sự cố nào của Bookstore cũng nằm trong container của nó. Khi kubelet không
chạy được container, khi API server chậm, người trực phải biết log của **chính Kubernetes** nằm đâu, đọc
thế nào, và trace — trụ cột thứ ba — có dùng được không. Ticket này viết runbook đó bằng bằng chứng trên
node thật, và **không sửa gì**: mọi thay đổi cấu hình node và control plane thuộc giai đoạn 20.

**Yêu cầu:**

1. Trên `lab-k8s-worker1`: log của kubelet đọc ở đâu; chứng minh `/var/log/kubelet.log` không tồn tại.
2. Trên `lab-k8s-master`: log của kube-apiserver, kube-scheduler, kube-controller-manager nằm ở đâu trên
   đĩa; có symlink trong `/var/log/containers`; chúng không phải unit systemd.
3. Đọc header klog: mức nghiêm trọng, thời điểm, file nguồn.
4. Đo hướng của thang `-v` mà không đụng control plane; đọc mức `-v` thật của ba thành phần control
   plane.
5. Log Query: đọc giá trị hiệu lực của các tùy chọn liên quan trên `lab-k8s-worker1` và chứng minh hành
   vi của endpoint khớp với giá trị đó.
6. Trace: kube-apiserver không có file cấu hình tracing; không kubelet nào có đoạn `tracing`; không node
   nào nghe cổng OTLP mặc định.
7. `docs/runbook-log.md`: bảng "triệu chứng → đọc ở đâu → bằng lệnh gì" cho Bookstore, ít nhất năm dòng
   (container ứng dụng, container đã khởi động lại, file đã xoay vòng, kubelet, control plane), kèm một
   bảng ánh xạ bốn nhánh của bài 296 → trang trỏ hướng 316/317 → bài con → ticket của sprint này.
8. ADR: dự án **chưa** bật tracing; điều kiện để bật.

**Ràng buộc:** chỉ đọc. Checksum ở B0 phải khớp ở gate cuối. Không đổi `-v` của thành phần nào, không
bật Log Query, không bật tracing.

**Đọc lại:**

- Bài [158](../158-logging-vi.md#log-của-các-thành-phần-hệ-thống-system-component-logs) và mục
  [*Vị trí log*](../158-logging-vi.md#log-location-node).
- Bài [159](../159-system-logs-vi.md): [*Klog*](../159-system-logs-vi.md#klog),
  [*Output của klog*](../159-system-logs-vi.md#output-của-klog-klog-output),
  [*Mức chi tiết của log*](../159-system-logs-vi.md#mức-chi-tiết-của-log-log-verbosity-level),
  [*Vị trí log*](../159-system-logs-vi.md#vị-trí-log-log-location),
  [*Truy vấn log*](../159-system-logs-vi.md#truy-vấn-log-log-query).
- Bài [161](../161-system-traces-vi.md): [*Thu thập trace*](../161-system-traces-vi.md#thu-thập-trace-trace-collection),
  [*Trace của kube-apiserver*](../161-system-traces-vi.md#trace-của-kube-apiserver-kube-apiserver-traces),
  [*Trace của kubelet*](../161-system-traces-vi.md#trace-của-kubelet-kubelet-traces),
  [*Tính ổn định*](../161-system-traces-vi.md#tính-ổn-định-stability).
- Bài [296](../296-debug-vi.md), [316](../316-debug-logging-vi.md), [317](../317-debug-monitoring-vi.md).

**Bàn giao:** `docs/runbook-log.md`; `docs/adr/NNNN-log-he-thong-va-trace.md`;
`~/lab-evidence/11a/b6-*.txt`.

**Nghiệm thu:**

- `PASS:` kubelet trên `lab-k8s-worker1` ghi journald; không có `/var/log/kubelet.log`.
- `PASS:` ba thành phần control plane có thư mục log trong `/var/log/pods` và symlink trong
  `/var/log/containers`; không có unit systemd mang tên chúng.
- `PASS:` log của scheduler và apiserver khớp header klog.
- `PASS:` tăng `-v` thì số dòng log tăng; từ `-v=6` thấy từng request HTTP; đọc được `-v` của ba thành
  phần control plane.
- `PASS:` hành vi của Log Query khớp cấu hình hiệu lực.
- `PASS:` không có cấu hình tracing ở apiserver, ở ba kubelet; không node nào nghe cổng `4317`.
- `PASS:` runbook và ADR tồn tại.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-log-he-thong-va-trace.md`:

1. Vì sao log của kubelet và log của kube-scheduler — hai thành phần của cùng một cluster — nằm ở hai
   chỗ khác nhau? Log trong `/var/log` của node do ai xoay vòng?
2. Một đồng đội muốn viết cảnh báo khớp đúng một câu trong log của kube-controller-manager. Bạn trả lời
   gì, dẫn câu cảnh báo nào của bài 159?
3. Bật Log Query cho người trực đổi lấy quyền gì của họ trên node? Liên hệ với câu 3 của B2.
4. Bật tracing cho kubelet trên `lab-k8s-worker2` mang lại thứ gì mà log của kubelet không có? Vì sao
   client gửi kèm trace context tới API server vẫn không nối được vào trace của nó? `samplingRatePerMillion:
   100` nghĩa là gì?
5. Vì sao dự án không dựng dashboard dài hạn trên tên span (mục *Tính ổn định*)?

<details>
<summary>Gợi ý</summary>

- klog luôn ghi ra **stderr**; khi đo `-v` của `kubectl`, chuyển hướng `2>` vào file.
- Lệnh của static Pod control plane: `kubectl -n kube-system get pod kube-apiserver-lab-k8s-master
  -o jsonpath='{range .spec.containers[0].command[*]}{@}{"\n"}{end}'`.
- Endpoint của Log Query: `/api/v1/nodes/<node>/proxy/logs/?query=kubelet&tailLines=5`. Đọc cả
  `enableSystemLogHandler` lẫn `enableSystemLogQuery` trong `configz` đã lưu ở B0.
- Cổng đang nghe trên một node: `ssh <node> 'ss -lnt'`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
EV=~/lab-evidence/11a; M=lab-k8s-master; W1=lab-k8s-worker1
cfgz() { grep -o "\"$1\":[^,}]*" "$2" | head -1 | cut -d: -f2- | tr -d '"'; }
cmd() { kubectl -n kube-system get pod "$1-$M" -o jsonpath='{range .spec.containers[0].command[*]}{@}{"\n"}{end}'; }

# 1-2. Hai loại thành phần, hai vị trí
ssh "$W1" 'sudo journalctl -u kubelet --no-pager -n 10' > "$EV/b6-journal-kubelet.txt"
test -s "$EV/b6-journal-kubelet.txt" && echo 'PASS: kubelet tren worker1 ghi journald'
case "$(ssh -o BatchMode=yes "$W1" 'sudo -n true && if sudo test -e /var/log/kubelet.log; then echo CO; else echo KHONG; fi')" in
  KHONG) echo 'PASS: khong co /var/log/kubelet.log' ;;
  CO)    echo 'FAIL: co /var/log/kubelet.log' ;;
  *)     echo 'STOP: khong kiem duoc worker1 (ssh hoac sudo loi) — chua ket luan' ;;
esac
BAD=0
for c in kube-apiserver kube-scheduler kube-controller-manager; do
  sudo ls -d /var/log/pods/kube-system_${c}-* >/dev/null 2>&1 || BAD=1
  sudo ls /var/log/containers/ | grep -q "^${c}-" || BAD=1
done
[ "$BAD" -eq 0 ] && echo 'PASS: ba thanh phan control plane co log trong /var/log/pods va symlink'
test "$(systemctl list-units --all --type=service --no-legend --no-pager | grep -cE 'kube-(apiserver|scheduler|controller-manager)')" -eq 0 \
  && echo 'PASS: khong co unit systemd nao cua control plane'

# 3. Header klog
{ kubectl -n kube-system logs "kube-scheduler-$M" --tail=5; kubectl -n kube-system logs "kube-apiserver-$M" --tail=5; } > "$EV/b6-klog.txt" 2>&1
test "$(grep -cE '^[IWEF][0-9]{4} [0-9]{2}:[0-9]{2}:[0-9]{2}\.[0-9]+ +[0-9]+ [^ ]+:[0-9]+\]' "$EV/b6-klog.txt")" -ge 2 \
  && echo 'PASS: doc duoc header klog — muc, thoi diem, file nguon'

# 4. Thang -v
for v in 2 6 8; do kubectl get --raw /version -v=$v > /dev/null 2> "$EV/b6-v$v.txt"; done
N2="$(wc -l < "$EV/b6-v2.txt")"; N6="$(wc -l < "$EV/b6-v6.txt")"; N8="$(wc -l < "$EV/b6-v8.txt")"
{ test "$N6" -gt "$N2" && test "$N8" -gt "$N6"; } && echo "PASS: tang -v thi log tang ($N2 < $N6 < $N8)"
grep -q 'GET https' "$EV/b6-v6.txt" && echo 'PASS: tu -v=6 thay tung request HTTP'
for c in kube-apiserver kube-scheduler kube-controller-manager; do
  echo "$c: $(cmd "$c" | grep -E '^--v=' || echo 'khong khai --v')"
done | tee "$EV/b6-verbosity.txt"
test "$(wc -l < "$EV/b6-verbosity.txt")" -eq 3 && echo 'PASS: doc duoc muc -v cua ba thanh phan'

# 5. Log Query
LH="$(cfgz enableSystemLogHandler "$EV/b0-configz-$W1.json")"; LQ="$(cfgz enableSystemLogQuery "$EV/b0-configz-$W1.json")"
kubectl get --raw "/api/v1/nodes/$W1/proxy/logs/?query=kubelet&tailLines=5" > "$EV/b6-log-query.txt" 2>&1 && OK=1 || OK=0
echo "enableSystemLogHandler=${LH:-<khong khai>} enableSystemLogQuery=${LQ:-<khong khai>} goi duoc=$OK" | tee "$EV/b6-log-query-cfg.txt"
if [ "$LH" = 'true' ] && [ "$LQ" = 'true' ]; then
  [ "$OK" -eq 1 ] && echo 'PASS: Log Query bat va endpoint tra log' || echo 'FAIL: Log Query bat ma khong tra log'
else
  [ "$OK" -eq 0 ] && echo 'PASS: Log Query tat va endpoint khong tra log' || echo 'FAIL: Log Query tat ma van tra log'
fi

# 6. Trace
test "$(cmd kube-apiserver | grep -c -- '--tracing-config-file=')" -eq 0 && echo 'PASS: apiserver khong co --tracing-config-file'
test "$(cat "$EV"/b0-configz-*.json | grep -c '"tracing"')" -eq 0 && echo 'PASS: khong kubelet nao cau hinh tracing'
L=0
for n in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  ssh "$n" "ss -lnt | grep -q ':4317 '" && L=$((L + 1))
done
test "$L" -eq 0 && echo 'PASS: khong node nao nghe cong 4317'

cd ~/bookstore
{ test -s docs/runbook-log.md && ls docs/adr/*-log-he-thong-va-trace.md >/dev/null 2>&1; } && echo 'PASS: co runbook va ADR'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Khung `docs/runbook-log.md`:

```markdown
# Runbook — đọc log của Bookstore và của cluster

| Triệu chứng | Đọc ở đâu | Lệnh |
| --- | --- | --- |
| Container ứng dụng báo lỗi | stdout/stderr của container, qua kubelet | `kubectl -n bookstore logs <pod> -c <container>` |
| Container vừa khởi động lại | log của lần chạy trước | `kubectl -n bookstore logs <pod> -c <container> --previous` |
| Cần phần log đã bị xoay vòng | file `<n>.log.*` trên node, agent thấy | `node-agent` `/logs.txt` (chỉ mục); file trên node |
| Pod không được tạo container | kubelet trên node đó, journald | `ssh <node> 'sudo journalctl -u kubelet --since -15m'` |
| Runtime không kéo được image | containerd, journald | `ssh <node> 'sudo journalctl -u containerd --since -15m'` |
| API server, scheduler, controller-manager | Pod static trong `kube-system`; file trong `/var/log/pods` | `kubectl -n kube-system logs kube-apiserver-lab-k8s-master` |

| Nhánh của bài 296 | Trang trỏ hướng | Bài con | Ticket sprint 17 |
| --- | --- | --- | --- |
| Gỡ lỗi ứng dụng | 297 | 304 (299, 301, 303 ở giai đoạn 24) | B7 |
| Gỡ lỗi cluster | 305 | giai đoạn 24 | — |
| Ghi log | 316 | 158, 159 | B4, B5, B6 |
| Giám sát | 317 | 160, 161 | B1, B2, B6 |
```

Ý chính cho ADR:

1. kubelet và container runtime không chạy trong container — là service của hệ điều hành, có systemd
   thì ghi journald. Scheduler, controller-manager, API server chạy trong static Pod, ghi file `.log`
   dưới `/var/log`, bỏ qua journald. Log trong `/var/log` cần xoay vòng và Kubernetes không quản lý việc
   đó — thuộc `logrotate` của OS hoặc công cụ triển khai.
2. Nội dung log **không** thuộc phạm vi bảo đảm ổn định của Kubernetes API: từng dòng và định dạng có thể
   đổi giữa các bản phát hành. Cảnh báo khớp chuỗi sẽ im lặng hỏng sau một lần nâng cấp; dùng metric.
3. Endpoint đi qua `nodes/proxy` — quyền mà bài 159 cảnh báo mở cả API kubelet thực thi được lệnh trong
   container. Đổi tiện lợi lấy một quyền gần với root trên node.
4. Liên kết cha–con giữa span của kubelet và span của containerd, vì kubelet lan truyền trace context qua
   gRPC — log không dựng lại được điều đó. API server không dùng trace context đính kèm request đến vì nó
   thường là endpoint công khai. `100` là ghi span cho 1 trong mỗi 10 000 request, chỉ áp cho span không
   có cha.
5. Bài 161: phần đo đạc tracing còn đang phát triển, tên span và thuộc tính chưa có bảo đảm tương thích
   ngược; tracing kèm chi phí mạng và CPU — công cụ cho đợt chẩn đoán có thời hạn. Điều kiện bật: có
   collector và backend (giai đoạn 23), sửa cấu hình qua quy trình của giai đoạn 20.

</details>

## B7. [BS-11A-07] Diễn tập: chẩn đoán một dịch vụ không trả lời

**Loại:** Diễn tập sự cố — mọi Pod ghim `lab-k8s-worker2`, trong namespace `bookstore-thu`.

**Bối cảnh:** sprint này để lại cho người trực một quy trình chẩn đoán ứng dụng. Trước khi ghi nó vào
repo, phải chạy nó trên một sự cố thật. Đội dựng sẵn một dịch vụ phụ `thu-catalog` mô phỏng cách
Bookstore phục vụ web — Deployment, Service, một client gọi định kỳ — kèm một tác vụ nền `thu-report`.
Cả hai đang hỏng. Manifest sự cố được bàn giao nguyên văn; **đừng đọc kỹ nó trước khi chẩn đoán xong**
— trong sự cố thật bạn cũng không có ai chỉ chỗ sai.

```bash
kubectl apply -f ~/lab-work/11a/ns-thu.yaml
cat > ~/lab-work/11a/su-co.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: thu-catalog
  namespace: bookstore-thu
spec:
  replicas: 2
  selector:
    matchLabels:
      app: thu-catalog
  template:
    metadata:
      labels:
        app: thu-catalog
        tier: frontend
    spec:
      nodeSelector:
        kubernetes.io/hostname: lab-k8s-worker2
      volumes:
      - name: www
        emptyDir: {}
      containers:
      - name: httpd
        image: busybox:1.37
        command: ["sh", "-c", "echo catalog-ok > /www/index.html; exec httpd -f -v -p 8080 -h /www"]
        ports:
        - {name: http, containerPort: 8080}
        readinessProbe:
          httpGet: {path: /index.html, port: http}
          periodSeconds: 5
        volumeMounts:
        - {name: www, mountPath: /www}
      - name: content-sync
        image: busybox:1.37
        command: ["sh", "-c", "trap 'exit 0' TERM; while true; do date -Iseconds > /www/updated.txt; sleep 5 & wait $!; done"]
        volumeMounts:
        - {name: www, mountPath: /www}
---
apiVersion: v1
kind: Service
metadata:
  name: thu-catalog
  namespace: bookstore-thu
spec:
  selector:
    app: thu-catalog
    tier: front-end
  ports:
  - {name: http, port: 80, targetPort: http}
---
apiVersion: v1
kind: Pod
metadata:
  name: thu-client
  namespace: bookstore-thu
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
  - name: client
    image: busybox:1.37
    command: ["sh", "-c", "trap 'exit 0' TERM; while true; do if wget -q -T 2 -O /dev/null http://thu-catalog/index.html; then echo \"$(date -Iseconds) OK\"; else echo \"$(date -Iseconds) LOI\"; fi; sleep 3 & wait $!; done"]
---
apiVersion: v1
kind: Pod
metadata:
  name: thu-report
  namespace: bookstore-thu
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
  - name: report
    image: busybox:1.37
    command: ["sh", "-c", "echo \"report: bat dau $(date -Iseconds)\"; if [ -z \"$REPORT_TARGET\" ]; then echo 'report: thieu bien REPORT_TARGET, thoat'; exit 3; fi; echo \"report: gui bao cao toi $REPORT_TARGET\"; trap 'exit 0' TERM; while true; do sleep 30 & wait $!; done"]
EOF
kubectl apply -f ~/lab-work/11a/su-co.yaml
```

**Yêu cầu:**

1. Trước khi đụng vào gì: xếp hai triệu chứng ("client báo `LOI`", "`thu-report` khởi động lại liên
   tục") vào đúng nhánh của bài 296 và đúng nhóm chủ đề của bài 297; ghi vào postmortem kèm lý do.
2. Thu bằng chứng bằng bộ công cụ của giai đoạn 11 — `kubectl logs` (kể cả `--previous`, `-c`),
   `kubectl exec` (shell tương tác **và** lệnh rời) — cộng `get`/`describe` đã học. Lưu đúng bốn file:
   - `~/lab-evidence/11a/b7-client.txt` — log của client cho thấy lỗi;
   - `~/lab-evidence/11a/b7-trong-pod.txt` — `thu-catalog` trả `catalog-ok` khi gọi **từ bên trong** Pod
     của nó;
   - `~/lab-evidence/11a/b7-endpointslice-truoc.yaml` — EndpointSlice của Service `thu-catalog` lúc lỗi,
     dạng YAML;
   - `~/lab-evidence/11a/b7-report-previous.txt` — log của lần chạy **trước** của `thu-report`.
3. Sửa cả hai lỗi bằng cách sửa manifest ở `~/lab-work/11a/` rồi apply: client in `OK`; `thu-report`
   chạy ổn định.
4. Sau khi sửa: từ shell tương tác trong container `httpd` của một Pod `thu-catalog`, ghi một trang gốc
   mới có chữ `Hello shell demo` vào volume dùng chung rồi tự gọi lại từ bên trong; container **kia** của
   cùng Pod cũng thấy trang đó.
5. Postmortem `docs/postmortem/11a-chan-doan-ung-dung.md`: dòng thời gian, bằng chứng từng bước, nguyên
   nhân gốc, cách sửa, và bước nào trong quy trình đầy đủ thuộc giai đoạn 24.
6. Xóa object sự cố.

**Ràng buộc:** không xóa và tạo lại Service hay Deployment "để thử" trước khi có bằng chứng. Không dùng
`kubectl debug` và ephemeral container — quy trình đó thuộc giai đoạn 24. Không đổi image.

**Đọc lại:**

- Bài [296](../296-debug-vi.md) — bốn nhánh và tiêu chí chia; bài [297](../297-debug-application-vi.md)
  — ba nhóm chủ đề.
- Bài [304](../304-get-shell-running-container-vi.md):
  [*Truy cập shell của một container*](../304-get-shell-running-container-vi.md#truy-cập-shell-của-một-container-getting-a-shell-to-a-container),
  [*Ghi trang gốc cho nginx*](../304-get-shell-running-container-vi.md#ghi-trang-gốc-cho-nginx-writing-the-root-page-for-nginx),
  [*Chạy từng lệnh riêng lẻ*](../304-get-shell-running-container-vi.md#chạy-từng-lệnh-riêng-lẻ-trong-container-running-individual-commands-in-a-container),
  [*Mở shell khi Pod có nhiều hơn một container*](../304-get-shell-running-container-vi.md#mở-shell-khi-pod-có-nhiều-hơn-một-container-opening-a-shell-when-a-pod-has-more-than-one-container).
- Bài [158](../158-logging-vi.md#basic-logging-in-kubernetes) — `--previous`, `-c`.
- Bài [82](../82-service-vi.md#định-nghĩa-một-service-defining-a-service),
  [83](../83-endpoint-slices-vi.md#endpointslice-resource), [18](../18-labels-vi.md#label-selectors).
- ⭐ Bài [309](../309-local-debugging-vi.md#telepresence-hoạt-động-như-thế-nào-how-does-telepresence-work).

**Bàn giao:** `docs/postmortem/11a-chan-doan-ung-dung.md`; bốn file bằng chứng ở yêu cầu 2.

**Nghiệm thu:**

- `PASS:` bốn file bằng chứng có đúng nội dung mô tả.
- `PASS:` EndpointSlice của `thu-catalog` có hai địa chỉ sẵn sàng; ba dòng log gần nhất của client đều
  `OK`.
- `PASS:` `thu-report` `Running`, không khởi động lại thêm trong lúc quan sát, log có đích báo cáo.
- `PASS:` một Pod `thu-catalog` phục vụ `Hello shell demo`, đọc được từ container `content-sync`.
- `PASS:` có postmortem; object sự cố đã xóa.

**Câu hỏi thiết kế** — trả lời trong postmortem:

1. Vì sao lỗi `LOI` của client là sự cố **ứng dụng** chứ không phải sự cố **cluster**, dù bạn là admin?
   Bằng chứng nào loại trừ "mạng của cluster hỏng"?
2. Bằng chứng nào cho thấy ứng dụng **không** có lỗi, và lỗi nằm giữa Service và Pod? Kể đúng thứ tự
   bạn đã đi.
3. Lúc `thu-report` đang chờ khởi động lại, `kubectl logs` và `kubectl logs --previous` khác nhau thế nào?
   Vì sao `--previous` là công cụ đầu tiên với `CrashLoopBackOff`?
4. `kubectl exec` vào `thu-catalog` mà không có `-c` thì chuyện gì xảy ra? Trong `kubectl exec <pod> --
   ls -l /`, bỏ dấu `--` thì sao?
5. ⭐ Nếu dùng Telepresence intercept `thu-catalog`, nó để lại gì trong Pod trên cluster? Vì sao lab
   không chạy nó, và cách thủ công tương đương là gì?

<details>
<summary>Gợi ý</summary>

1. Bắt đầu từ phía bị ảnh hưởng: log của client. Rồi đi ngược về đích: Pod của dịch vụ có chạy không, có
   phục vụ được **bên trong** chính nó không.
2. Service chuyển request tới đâu là do EndpointSlice quyết định; EndpointSlice được dựng từ selector của
   Service. So selector với label thật của Pod: `kubectl get pods --show-labels`.
3. Biến môi trường của Pod trần không sửa được trên Pod đang chạy (Lab 3a) — xóa rồi tạo lại.
4. Shell tương tác: `kubectl exec --stdin --tty <pod> -c <container> -- sh`. Thoát bằng `exit`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã sửa xong, TRƯỚC khi xóa object sự cố</summary>

```bash
NS=bookstore-thu; EV=~/lab-evidence/11a
grep -q ' LOI$' "$EV/b7-client.txt" && echo 'PASS: bang chung client bao loi'
grep -qx 'catalog-ok' "$EV/b7-trong-pod.txt" && echo 'PASS: bang chung ung dung phuc vu ben trong Pod'
{ grep -q 'kind: EndpointSlice' "$EV/b7-endpointslice-truoc.yaml" && ! grep -q 'addresses:' "$EV/b7-endpointslice-truoc.yaml"; } \
  && echo 'PASS: bang chung EndpointSlice rong luc loi'
grep -q 'thieu bien REPORT_TARGET' "$EV/b7-report-previous.txt" && echo 'PASS: bang chung log lan chay truoc cua thu-report'

N="$(kubectl -n "$NS" get endpointslice -l kubernetes.io/service-name=thu-catalog \
     -o jsonpath='{range .items[*].endpoints[?(@.conditions.ready==true)]}{.addresses[0]}{"\n"}{end}' | grep -c .)"
test "$N" -eq 2 && echo 'PASS: EndpointSlice co hai dia chi san sang'
sleep 10
test "$(kubectl -n "$NS" logs thu-client --tail=3 | grep -c ' OK$')" -eq 3 && echo 'PASS: ba lan goi gan nhat cua client deu OK'

R0="$(kubectl -n "$NS" get pod thu-report -o jsonpath='{.status.containerStatuses[0].restartCount}')"
sleep 30
{ test "$(kubectl -n "$NS" get pod thu-report -o jsonpath='{.status.phase}')" = 'Running' && \
  test "$(kubectl -n "$NS" get pod thu-report -o jsonpath='{.status.containerStatuses[0].restartCount}')" = "$R0" && \
  kubectl -n "$NS" logs thu-report | grep -q 'gui bao cao toi'; } && echo 'PASS: thu-report chay on dinh'

BAD=1
for P in $(kubectl -n "$NS" get pod -l app=thu-catalog -o jsonpath='{.items[*].metadata.name}'); do
  kubectl -n "$NS" exec "$P" -c content-sync -- wget -qO- http://127.0.0.1:8080/ 2>/dev/null | grep -q 'Hello shell demo' && BAD=0
done
[ "$BAD" -eq 0 ] && echo 'PASS: trang goc moi doc duoc tu container content-sync cung Pod'
test -s ~/bookstore/docs/postmortem/11a-chan-doan-ung-dung.md && echo 'PASS: co postmortem'
```

Sau đó dọn và kiểm:

```bash
kubectl delete -f ~/lab-work/11a/su-co.yaml --ignore-not-found --wait=true
test -z "$(kubectl -n bookstore-thu get deploy,svc,pod -o name 2>/dev/null)" && echo 'PASS: da xoa object su co'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```bash
NS=bookstore-thu; EV=~/lab-evidence/11a
# 1. Phía bị ảnh hưởng
kubectl -n "$NS" logs thu-client --tail=20 > "$EV/b7-client.txt"
# 2. Đích có chạy, có phục vụ bên trong không — Pod có hai container, chỉ đích danh bằng -c
P="$(kubectl -n "$NS" get pod -l app=thu-catalog -o jsonpath='{.items[0].metadata.name}')"
kubectl -n "$NS" logs "$P" -c httpd --tail=5
kubectl -n "$NS" exec "$P" -c httpd -- wget -qO- http://127.0.0.1:8080/index.html > "$EV/b7-trong-pod.txt"
# 3. Giữa Service và Pod
kubectl -n "$NS" get endpointslice -l kubernetes.io/service-name=thu-catalog -o yaml > "$EV/b7-endpointslice-truoc.yaml"
kubectl -n "$NS" get svc thu-catalog -o jsonpath='{.spec.selector}{"\n"}'
kubectl -n "$NS" get pods -l app=thu-catalog --show-labels
# 4. Tác vụ khởi động lại
kubectl -n "$NS" get pod thu-report
kubectl -n "$NS" logs thu-report --previous > "$EV/b7-report-previous.txt"
```

Sửa trong `~/lab-work/11a/su-co.yaml`: selector của Service `tier: front-end` → `tier: frontend`; Pod
`thu-report` thêm `env: [{name: REPORT_TARGET, value: bookstore-thu}]`. Rồi:

```bash
kubectl -n bookstore-thu delete pod thu-report --wait=true
kubectl apply -f ~/lab-work/11a/su-co.yaml
# Shell tương tác (bài 304) vào container httpd; trong shell gõ:
#   echo 'Hello shell demo' > /www/index.html; wget -qO- http://localhost:8080/; exit
P="$(kubectl -n bookstore-thu get pod -l app=thu-catalog -o jsonpath='{.items[0].metadata.name}')"
kubectl -n bookstore-thu exec --stdin --tty "$P" -c httpd -- sh
```

Ý chính cho postmortem:

1. Bài 296 chia nhánh theo **thứ đang hỏng**, không theo chức danh: thứ hỏng là một ứng dụng vừa triển
   khai. Client vẫn phân giải được tên `thu-catalog` và nhận lỗi ngay (Service không có endpoint) chứ
   không timeout vì DNS; cluster vẫn phục vụ mọi thứ khác — không có dấu hiệu nào ở tầng cluster.
2. Client `LOI` → Pod `thu-catalog` `Running`, `Ready`, trả `catalog-ok` từ bên trong → EndpointSlice
   rỗng → selector `tier: front-end` không khớp label `tier: frontend`. Ứng dụng đúng; Service không chọn
   được Pod nào. Quy trình lần từ Service về Pod đầy đủ là bài 301, giai đoạn 24.
3. Khi container đang chờ backoff, `kubectl logs` có thể trống hoặc chỉ có lần chạy dở; `--previous` trả
   log của lần chạy đã kết thúc — chỗ có câu "thoát". Kubelet giữ container đã kết thúc cùng log của nó
   khi container khởi động lại (bài 158).
4. Không có `-c`, `kubectl` chọn container mặc định (container đầu, kèm dòng thông báo) — tiện nhưng dễ
   nhầm. Bỏ `--`, `kubectl` nuốt `-l` thành cờ của chính nó và báo lỗi.
5. Telepresence cài sidecar traffic-agent cạnh container ứng dụng — sửa workload; lab nằm trên chuỗi
   snapshot nên không chạy. Cách thủ công mà chính bài 309 nhắc là lấy shell vào container đang chạy —
   bước 2 và lệnh `exec --stdin --tty` ở trên.

</details>

## B8. Đóng sprint: cleanup, gate cuối và snapshot `04-metrics-ready`

**Mục đích:** bàn giao một repo dựng lại được toàn bộ hệ thống của sprint 17, gỡ dự án, chứng minh cấu
hình node và control plane không đổi, giữ lại **đúng** phần hạ tầng mà mốc mới được định nghĩa là có —
metrics-server — rồi chụp `04-metrics-ready`.

1. Thay `tests/smoke.sh` bằng phiên bản của Lab 11a — kiểm thử khói **tích lũy**, chép nguyên văn:

```bash
cat > ~/bookstore/tests/smoke.sh <<'EOF'
#!/usr/bin/env bash
# Kiểm thử khói — trạng thái sau Lab 11a (sprint 17). Tích lũy mọi hợp đồng còn hiệu lực: giữ mọi kiểm
# tra của smoke Lab 9b (node-agent và ServiceAccount của nó nay ở bookstore-system), thêm node-agent ở
# bookstore-system, /logs.txt, NetworkPolicy của bookstore-system và metric. Không in mật khẩu.
set -u
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
cd "$(dirname "$0")/.."
NS=bookstore
SYS=bookstore-system
TNS=bookstore-staging
SEL='app.kubernetes.io/part-of=bookstore'
FAIL=0
pass() { echo "PASS: $*"; }
fail() { echo "FAIL: $*"; FAIL=1; }
k() { kubectl -n "$NS" "$@"; }
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
lbl() { kubectl get "$1" -o jsonpath="{.metadata.labels.$2}" 2>/dev/null; }
can() { kubectl auth can-i "$@" -n "$NS" 2>/dev/null; }
psa() { kubectl get namespace "$1" -o jsonpath="{.metadata.labels.pod-security\.kubernetes\.io/$2}" 2>/dev/null; }
pst() { kubectl -n "$1" exec "$2" -c "$3" -- sh -c "awk '/^$4:/{print \$2}' /proc/1/status" 2>/dev/null; }
dry_priv() {
  printf '%s\n' 'apiVersion: v1' 'kind: Pod' 'metadata: {name: smoke-dac-quyen}' 'spec:' \
    '  containers: [{name: c, image: busybox:1.37, securityContext: {privileged: true}}]' \
    | kubectl -n "$1" create --dry-run=server -f - 2>&1
}
V="$(kubectl get --raw /version 2>/dev/null)"
PSA_VER="v$(echo "$V" | sed -n 's/.*"major": *"\([0-9]*\).*/\1/p').$(echo "$V" | sed -n 's/.*"minor": *"\([0-9]*\).*/\1/p')"

# 1. Namespace, label dự án, Pod Security (9b, 11a)
for X in "$NS:restricted" "$TNS:restricted" "$SYS:privileged"; do
  N="${X%%:*}"; L="${X##*:}"
  { [ "$(lbl "namespace/$N" 'app\.kubernetes\.io/part-of')" = 'bookstore' ] && \
    [ "$(lbl "namespace/$N" 'pod-security\.kubernetes\.io/enforce')" = "$L" ]; } \
    && pass "namespace $N, PSS enforce=$L" || fail "namespace $N thieu, thieu label du an hoac PSS khac $L"
done
for N in "$NS" "$TNS"; do
  OK=1
  for M in enforce warn audit; do
    { [ "$(psa "$N" "$M")" = restricted ] && [ "$(psa "$N" "$M-version")" = "$PSA_VER" ]; } || OK=0
  done
  [ "$OK" -eq 1 ] && pass "$N restricted o ca ba che do, version $PSA_VER" || fail "$N sai nhan Pod Security"
  dry_priv "$N" | grep -q "violates PodSecurity \"restricted:$PSA_VER\"" \
    && pass "$N tu choi Pod dac quyen" || fail "$N khong tu choi Pod dac quyen"
done
# Lệnh dry-run phải thành công thì "không có cảnh báo" mới có nghĩa: Forbidden, timeout hay lỗi
# chứng chỉ là FAIL, không phải "sạch".
if PSW="$(kubectl label --dry-run=server --overwrite ns "$NS" pod-security.kubernetes.io/enforce-version=latest 2>&1)"; then
  echo "$PSW" | grep -q 'violate' && fail 'co Pod dang chay vi pham restricted' || pass 'moi Pod dang chay dat restricted'
else
  fail "khong hoi duoc Pod Security bang dry-run: $PSW"
fi

# 2. Object cấp cluster, label node (3a, 6a, 7a, 9a)
PRIO="$(kubectl get priorityclass bookstore-critical -o jsonpath='{.value}' 2>/dev/null)"
for PC in bookstore-critical bookstore-batch; do
  [ "$(lbl "priorityclass/$PC" 'app\.kubernetes\.io/part-of')" = bookstore ] \
    && pass "co PriorityClass $PC" || fail "thieu PriorityClass $PC"
done
[ "$(kubectl get storageclass bookstore-retain -o jsonpath='{.reclaimPolicy}' 2>/dev/null)" = 'Retain' ] \
  && pass 'StorageClass bookstore-retain, Retain' || fail 'StorageClass bookstore-retain thieu hoac khong Retain'
[ "$(kubectl get nodes -l bookstore.example.com/data=true -o jsonpath='{.items[*].metadata.name}')" = lab-k8s-worker2 ] \
  && pass 'chi lab-k8s-worker2 mang label du lieu' || fail 'label node bookstore.example.com/data sai'
N_CL="$(kubectl get clusterrole,clusterrolebinding -l "$SEL" --no-headers 2>/dev/null | wc -l)"
[ "$N_CL" -eq 0 ] && pass 'du an khong co ClusterRole, ClusterRoleBinding' || fail "co $N_CL object RBAC cap cluster"

# 3. Cấu hình, bí mật, PDB, hạn mức, trạng thái backup (3b, 3c, 5b, 7b, 9a)
for P in web-config api-config db-init; do
  k get configmap -o name 2>/dev/null | grep -q "^configmap/$P" && pass "co ConfigMap $P" || fail "thieu ConfigMap $P"
done
D="$(jp secret/db-credentials '{.data}')"
{ echo "$D" | grep -q '"postgres-password"' && echo "$D" | grep -q '"authenticator-password"'; } \
  && pass 'Secret db-credentials du hai key' || fail 'Secret db-credentials thieu key'
[ "$(jp secret/bookstore-tls '{.type}')" = 'kubernetes.io/tls' ] && pass 'Secret bookstore-tls kieu TLS' || fail 'thieu Secret TLS bookstore-tls'
for R in pdb/web pdb/api resourcequota/bookstore-quota limitrange/bookstore-limits configmap/backup-status; do
  k get "$R" >/dev/null 2>&1 && pass "co $R" || fail "thieu $R"
done

# 4. Controller, vị trí, bảo mật (4a, 4b, 7a, 9b)
[ "$(jp statefulset/db '{.status.readyReplicas}')" = '1' ] && pass 'StatefulSet db 1/1 Ready' || fail 'StatefulSet db chua Ready'
{ [ "$(jp pod/db-0 '{.spec.nodeName}')" = 'lab-k8s-worker2' ] && [ "$(jp pod/db-0 '{.status.qosClass}')" = 'Guaranteed' ] && \
  [ -n "$PRIO" ] && [ "$(jp pod/db-0 '{.spec.priority}')" = "$PRIO" ]; } \
  && pass 'db-0 tren lab-k8s-worker2, Guaranteed, bookstore-critical' || fail 'db-0 sai node, QoS hoac priority'
for D in web api; do
  W="$(jp "deployment/$D" '{.spec.replicas}')"; R="$(jp "deployment/$D" '{.status.readyReplicas}')"
  { [ -n "$W" ] && [ "$W" -ge 2 ] && [ "${R:-0}" = "$W" ]; } && pass "Deployment $D $R/$W Ready" || fail "Deployment $D ${R:-0}/${W:-?} Ready"
done
ORPHAN="$(kubectl get pods -A -l "$SEL" -o jsonpath='{range .items[*]}{.metadata.name}:{.metadata.ownerReferences[0].kind}{"\n"}{end}' 2>/dev/null | grep -c ':$')"
[ "$ORPHAN" -eq 0 ] && pass 'moi Pod cua du an deu co controller' || fail "$ORPHAN Pod tran"
MISS="$(kubectl get pods -A -l "$SEL" -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.name}={.resources.requests.cpu},{.resources.limits.memory}{"\n"}{end}{end}' 2>/dev/null | grep -E '=,|,$' | sort -u)"
[ -z "$MISS" ] && pass 'moi container co requests va limits' || fail "container thieu tai nguyen: $(echo $MISS)"

# 5. Hardening (9b): db non-root, không capability, rootfs chỉ đọc
[ "$(pst "$NS" db-0 postgres Uid)" = 999 ] && pass 'db chay uid 999' || fail 'db khong chay uid 999'
C="$(pst "$NS" db-0 postgres CapEff)"
[ "$((0x${C:-1}))" -eq 0 ] && pass 'db khong giu capability nao' || fail 'db con capability'
NODROP="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.name}={.securityContext.capabilities.drop}{"\n"}{end}{end}' 2>/dev/null \
          | grep -v 'ALL' | sort -u)"
[ -z "$NODROP" ] && pass 'moi container cua du an drop ALL capability' || fail "container chua drop ALL: $(echo $NODROP)"
RO() { jp "deployment/$1" "{.spec.template.spec.containers[?(@.name==\"$2\")].securityContext.readOnlyRootFilesystem}"; }
{ [ "$(RO web nginx)" = true ] && [ "$(RO api postgrest)" = true ]; } \
  && pass 'nginx, postgrest khai rootfs chi doc' || fail 'nginx hoac postgrest chua rootfs chi doc'
SC='{.spec.jobTemplate.spec.template.spec.securityContext'
{ [ "$(jp cronjob/db-backup "$SC.runAsUser}")" = 999 ] && [ "$(jp cronjob/db-backup "$SC.runAsNonRoot}")" = true ]; } \
  && pass 'db-backup chay uid 999, kubelet kiem non-root' || fail 'db-backup chua non-root'
jp deployment/api '{range .spec.template.spec.securityContext.sysctls[*]}{.name}={.value}{" "}{end}' \
  | grep -q 'net.ipv4.tcp_keepalive_time=60' && pass 'api dat sysctl keepalive' || fail 'api thieu sysctl keepalive'

# 6. Dữ liệu bền (4b, 6a)
# backup-store (WaitForFirstConsumer) chỉ Bound sau lần chạy thử db-backup bên dưới (6a).
[ "$(jp pvc/data-db-0 '{.status.phase}')" = 'Bound' ] && pass 'PVC data-db-0 Bound' || fail 'PVC data-db-0 chua Bound'
[ "$(jp pvc/data-db-0 '{.spec.storageClassName}')" = 'bookstore-retain' ] && pass 'data-db-0 dung bookstore-retain' || fail 'data-db-0 sai StorageClass'
N="$(k exec db-0 -c postgres -- psql -U postgres -d bookstore -tAc 'SELECT count(*) FROM api.books' 2>/dev/null)"
[ "${N:-0}" -ge 8 ] && pass "db co $N cuon sach" || fail "db co ${N:-0} cuon sach, can >= 8"

# 7. Service, DNS, đường đi của request (5a, 9b)
[ "$(jp service/db '{.spec.clusterIP}')" = 'None' ] && pass 'Service db headless' || fail 'Service db khong headless'
[ "$(jp service/web '{.spec.type}/{.spec.ports[0].port}')" = 'ClusterIP/80' ] && pass 'Service web ClusterIP:80' || fail 'Service web sai'
[ "$(jp service/api '{.spec.ports[0].port}')" = '3000' ] && pass 'Service api cong 3000' || fail 'Service api sai cong'
WEB="$(k get pod -l "$SEL,app.kubernetes.io/component=web" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)"
WNODE="$(jp "pod/$WEB" '{.spec.nodeName}')"
get() { k exec "$WEB" -c status-writer -- wget -qO- -T 5 "$1" 2>/dev/null; }
get http://127.0.0.1:8080/ | grep -q 'Bookstore' && pass 'web phuc vu trang chu' || fail 'web khong phuc vu trang chu'
get http://127.0.0.1:8080/status.json | grep -Eq "\"pod\" *: *\"$WEB\"" && pass 'status.json dung ten Pod' || fail 'status.json sai hoac thieu'
T="$(get http://127.0.0.1:8080/api/books | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && pass "web /api/books tra $T cuon sach" || fail "web /api/books tra $T cuon sach"
DB_IP="$(jp pod/db-0 '{.status.podIP}')"
k exec "$WEB" -c status-writer -- nslookup "db-0.db.$NS.svc.cluster.local" 2>/dev/null | grep -qwF "$DB_IP" \
  && pass 'ten db-0.db phan giai ve IP cua db-0' || fail 'ten db-0.db khong phan giai dung'
k exec "$WEB" -c nginx -- sh -c 'touch /thu-ghi' >/dev/null 2>&1 \
  && fail 'nginx ghi duoc root filesystem' || pass 'root filesystem cua nginx chi doc'

# 8. Ingress HTTPS qua Traefik (5b)
{ [ "$(jp ingress/bookstore '{.spec.rules[0].host}')" = 'bookstore.lab.local' ] && \
  [ "$(jp ingress/bookstore '{.spec.tls[0].secretName}')" = 'bookstore-tls' ]; } \
  && pass 'Ingress bookstore dung host va TLS' || fail 'Ingress bookstore sai host hoac TLS'
NP="$(kubectl -n traefik get svc traefik -o jsonpath='{.spec.ports[?(@.name=="websecure")].nodePort}' 2>/dev/null)"
IP="$(kubectl get node lab-k8s-worker1 -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
if [ -n "$NP" ] && curl -ks --max-time 5 --resolve "bookstore.lab.local:$NP:$IP" "https://bookstore.lab.local:$NP/" | grep -q 'Bookstore'; then
  pass "HTTPS qua Traefik (NodePort $NP) tra trang chu"
else
  fail 'khong vao duoc trang chu qua Ingress HTTPS'
fi

# 9. Chính sách mạng (5b, 9a, 11a)
for N in "$NS" "$SYS"; do
  P="$(kubectl -n "$N" get networkpolicy -o jsonpath='{range .items[*]}{.spec.podSelector}|{.spec.policyTypes}{"\n"}{end}' 2>/dev/null | grep '^{}|')"
  { echo "$P" | grep -q 'Ingress' && echo "$P" | grep -q 'Egress'; } \
    && pass "moi Pod trong $N bi co lap hai chieu" || fail "namespace $N thieu policy co lap hai chieu"
done
# web không tới được API server. Đối chứng dương cùng đường đi: Pod busybox ở namespace thử
# bookstore-thu (không NetworkPolicy; chỉ xóa nếu smoke tự tạo) gọi được đúng URL đó bằng đúng lệnh đó.
# Phủ định chỉ tính khi wget hết giờ hoặc bị từ chối kết nối; lỗi khác là "không kết luận được".
WGET_API='timeout 10 wget -q -T 5 -O /dev/null --no-check-certificate https://kubernetes.default.svc/version'
PNS=bookstore-thu; MADE=0
if ! kubectl get namespace "$PNS" >/dev/null 2>&1; then
  kubectl create namespace "$PNS" >/dev/null 2>&1 \
    && kubectl label namespace "$PNS" app.kubernetes.io/part-of=bookstore >/dev/null 2>&1 && MADE=1
fi
CTRL="$(kubectl -n "$PNS" run "smoke-api-$(date +%s)" --rm -i --restart=Never --image=busybox:1.37 --command -- \
        sh -c "$WGET_API && echo TOI-DUOC || echo KHONG-TOI" 2>/dev/null)"
[ "$MADE" -eq 1 ] && kubectl delete namespace "$PNS" --ignore-not-found --wait=true --timeout=120s >/dev/null 2>&1
OUT="$(k exec "$WEB" -c status-writer -- sh -c "$WGET_API" 2>&1)"; RC=$?
if ! echo "$CTRL" | grep -q 'TOI-DUOC'; then fail 'doi chung: Pod ngoai NetworkPolicy khong goi duoc API server — khong ket luan duoc'
elif [ "$RC" -eq 0 ]; then fail 'web toi duoc API server'
elif echo "$OUT" | grep -Eq 'timed out|Connection refused|exit code 143'; then pass 'web bi NetworkPolicy chan toi API server'
else fail "web toi API server loi khac chan mang — khong ket luan duoc: $(echo $OUT)"; fi

# 10. CronJob db-backup và backup-status (4b, 6a, 7a, 9a)
[ "$(jp cronjob/db-backup '{.spec.concurrencyPolicy}')" = Forbid ] && pass 'db-backup Forbid' || fail 'db-backup khong Forbid'
SDS="$(jp cronjob/db-backup '{.spec.startingDeadlineSeconds}')"
[ "${SDS:-0}" -ge 10 ] && pass "db-backup startingDeadlineSeconds=$SDS" || fail 'db-backup thieu startingDeadlineSeconds'
[ "$(jp cronjob/db-backup '{.spec.suspend}')" != true ] && pass 'db-backup khong bi tam ngung' || fail 'db-backup dang suspend'
[ "$(jp cronjob/db-backup '{.spec.jobTemplate.spec.template.spec.priorityClassName}')" = bookstore-batch ] \
  && pass 'db-backup dung bookstore-batch' || fail 'db-backup khong dung bookstore-batch'
J="smoke-backup-$(date +%s)"
if k create job "$J" --from=cronjob/db-backup >/dev/null 2>&1 \
   && k wait --for=condition=Complete "job/$J" --timeout=300s >/dev/null 2>&1; then
  LINE="$(k logs "job/$J" 2>/dev/null | grep '^db-backup: OK' | tail -1)"
  F="$(echo "$LINE" | awk '{print $3}')"; B="$(echo "$LINE" | awk '{print $4}')"
  [ -n "$LINE" ] && pass "chay thu db-backup: $F ($B byte)" || fail 'chay thu db-backup khong in db-backup: OK'
  { [ -n "$F" ] && [ "$(jp configmap/backup-status '{.data.file}')" = "$F" ] && \
    [ "$(jp configmap/backup-status '{.data.bytes}')" = "$B" ]; } \
    && pass 'backup-status ghi dung lan chay vua roi' || fail 'backup-status khong khop lan chay'
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
[ "$(jp pvc/backup-store '{.status.phase}')" = 'Bound' ] && pass 'PVC backup-store Bound (sau lan chay thu)' || fail 'PVC backup-store chua Bound'

# 11. Danh tính và quyền (9a, 9b); ServiceAccount node-agent chuyển sang bookstore-system (11a)
BAD=0
for X in deployment/web:web deployment/api:api statefulset/db:db; do
  [ "$(jp "${X%%:*}" '{.spec.template.spec.serviceAccountName}')" = "${X##*:}" ] || { echo "FAIL: ${X%%:*} sai ServiceAccount"; BAD=1; }
done
[ "$(jp cronjob/db-backup '{.spec.jobTemplate.spec.template.spec.serviceAccountName}')" = db-backup ] \
  || { echo 'FAIL: db-backup sai ServiceAccount'; BAD=1; }
for SA in default web api db db-migrate db-backup ci-deployer; do
  [ "$(jp "serviceaccount/$SA" '{.automountServiceAccountToken}')" = false ] \
    || { echo "FAIL: ServiceAccount $SA thieu hoac tu mount token"; BAD=1; }
done
{ [ "$(kubectl -n "$SYS" get daemonset node-agent -o jsonpath='{.spec.template.spec.serviceAccountName}' 2>/dev/null)" = node-agent ] && \
  [ "$(kubectl -n "$SYS" get serviceaccount node-agent -o jsonpath='{.automountServiceAccountToken}' 2>/dev/null)" = false ]; } \
  || { echo 'FAIL: node-agent khong dung ServiceAccount node-agent khong token o bookstore-system'; BAD=1; }
[ "$BAD" -eq 0 ] && pass 'moi thanh phan mot ServiceAccount, khong ai tu mount token' || FAIL=1
TOKP="$(kubectl get pods -A -l "$SEL" -o jsonpath='{range .items[*]}{range .spec.volumes[*]}{.name}{"\n"}{end}{end}' 2>/dev/null | grep -c '^kube-api-access-')"
[ "$TOKP" -eq 0 ] && pass 'khong Pod nao cua du an mang token tu dong' || fail "$TOKP volume token tu dong"
B=system:serviceaccount:bookstore:db-backup
{ [ "$(can patch configmaps/backup-status --as="$B")" = yes ] && [ "$(can list configmaps --as="$B")" = no ] && \
  [ "$(can get secrets --as="$B")" = no ]; } && pass 'db-backup chi sua duoc backup-status' || fail 'quyen db-backup sai'
CI=system:serviceaccount:bookstore:ci-deployer
{ [ "$(can patch deployments --as="$CI")" = yes ] && [ "$(can get secrets --as="$CI")" = no ] && \
  [ "$(can create pods --subresource=exec --as="$CI")" = no ] && [ "$(can patch namespaces/bookstore --as="$CI")" = no ]; } \
  && pass 'ci-deployer sua Deployment, khong Secret, exec hay Namespace' || fail 'quyen ci-deployer sai'
{ [ "$(can list pods --as=dev --as-group=bookstore-devs)" = yes ] && [ "$(can get secrets --as=dev --as-group=bookstore-devs)" = no ] && \
  [ "$(can patch pods --subresource=ephemeralcontainers --as=dev --as-group=bookstore-devs)" = no ]; } \
  && pass 'bookstore-devs chi doc' || fail 'quyen bookstore-devs sai'
{ [ "$(can patch pods --subresource=ephemeralcontainers --as=truc --as-group=bookstore-oncall)" = yes ] && \
  [ "$(can get secrets --as=truc --as-group=bookstore-oncall)" = no ]; } \
  && pass 'bookstore-oncall go loi duoc, khong doc Secret' || fail 'quyen bookstore-oncall sai'

# 12. Tenant bookstore-staging (9b)
{ kubectl -n "$TNS" get resourcequota staging-quota >/dev/null 2>&1 && \
  kubectl -n "$TNS" get limitrange staging-limits >/dev/null 2>&1; } \
  && pass 'staging co quota va LimitRange' || fail 'staging thieu quota hoac LimitRange'
kubectl -n "$TNS" get networkpolicy -o jsonpath='{range .items[*]}{.spec.podSelector}|{.spec.policyTypes}{"\n"}{end}' 2>/dev/null \
  | grep -q '^{}|.*Ingress.*Egress' && pass 'staging co default deny hai chieu' || fail 'staging thieu default deny'
SW="$(kubectl -n "$TNS" get pod -l "$SEL,app.kubernetes.io/component=web" --field-selector=status.phase=Running \
        -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)"
kubectl -n "$TNS" exec "$SW" -- wget -qO- -T 5 http://web/ 2>/dev/null | grep -q 'Bookstore staging' \
  && pass 'staging phuc vu trang qua Service cua chinh no' || fail 'staging khong phuc vu duoc trang'
kubectl -n "$TNS" exec "$SW" -- wget -qO- -T 3 http://api.bookstore.svc.cluster.local:3000/books >/dev/null 2>&1 \
  && fail 'staging goi duoc api cua bookstore' || pass 'staging bi chan khi goi bookstore'
DEV='--as=dev --as-group=bookstore-devs'
{ [ "$(kubectl auth can-i create deployments -n "$TNS" $DEV 2>/dev/null)" = yes ] && \
  [ "$(kubectl auth can-i get secrets -n "$TNS" $DEV 2>/dev/null)" = no ] && \
  [ "$(kubectl auth can-i patch "namespace/$TNS" $DEV 2>/dev/null)" = no ] && \
  [ "$(can create deployments $DEV)" = no ]; } \
  && pass 'bookstore-devs trien khai duoc o staging; khong Secret, Namespace, bookstore' || fail 'quyen staging sai'

# 13. node-agent ở bookstore-system (4b, 5a, 11a)
NODES="$(kubectl get nodes --no-headers | wc -l)"
R="$(kubectl -n "$SYS" get daemonset node-agent -o jsonpath='{.status.numberReady}' 2>/dev/null)"
[ "${R:-0}" -eq "$NODES" ] && pass "node-agent Ready tren $R/$NODES node" || fail "node-agent ${R:-0}/$NODES node"
[ "$(sl k get daemonset,service,serviceaccount --field-selector metadata.name=node-agent --no-headers)" -eq 0 ] \
  && pass 'khong con node-agent trong bookstore, ke ca ServiceAccount' || fail 'con node-agent trong bookstore'
[ "$(kubectl -n "$SYS" get svc node-agent -o jsonpath='{.spec.internalTrafficPolicy}/{.spec.ports[0].port}' 2>/dev/null)" = 'Local/9100' ] \
  && pass 'Service node-agent Local, cong 9100' || fail 'Service node-agent sai'
get "http://node-agent.$SYS.svc.cluster.local:9100/node.json" | grep -Eq "\"node\" *: *\"$WNODE\"" \
  && pass "web doc /node.json cua agent cung node ($WNODE)" || fail 'web khong doc duoc agent cung node'
BAD=0
for L in $(kubectl -n "$SYS" get pod -l "$SEL,app.kubernetes.io/component=agent" -o jsonpath='{range .items[*]}{.spec.nodeName}={.status.podIP}{"\n"}{end}' 2>/dev/null); do
  get "http://${L##*=}:9100/node.json" | grep -Eq "\"node\" *: *\"${L%%=*}\"" || { echo "FAIL: node-agent tren ${L%%=*} sai /node.json"; BAD=1; }
done
[ "$BAD" -eq 0 ] && pass 'moi node-agent tra /node.json dung ten node' || FAIL=1
LOGS=''
for i in $(seq 1 12); do
  LOGS="$(get "http://node-agent.$SYS.svc.cluster.local:9100/logs.txt")"
  echo "$LOGS" | grep -q "^last /var/log/pods/${NS}_${WEB}_" && break
  sleep 5
done
{ echo "$LOGS" | tail -n 1 | grep -q "^node=$WNODE files=[1-9]" && \
  echo "$LOGS" | grep -q "^file [0-9]* /var/log/pods/${NS}_${WEB}_" && \
  echo "$LOGS" | grep -q "^last /var/log/pods/${NS}_${WEB}_"; } \
  && pass "/logs.txt cua agent tren $WNODE liet ke log cua $WEB" || fail '/logs.txt thieu hoac sai'

# 14. Metric (11a)
[ "$(kubectl get apiservice v1beta1.metrics.k8s.io -o jsonpath='{.status.conditions[?(@.type=="Available")].status}' 2>/dev/null)" = 'True' ] \
  && pass 'APIService v1beta1.metrics.k8s.io Available' || fail 'Metrics API chua Available'
[ "$(sl kubectl top node --no-headers)" -eq "$NODES" ] && pass 'kubectl top node du moi node' || fail 'kubectl top node thieu node'
TOP=''
for i in $(seq 1 18); do
  TOP="$(kubectl top pod -n "$NS" --no-headers 2>/dev/null)"
  echo "$TOP" | grep -q '^db-0 ' && break
  sleep 5
done
echo "$TOP" | grep -q '^db-0 ' && pass 'kubectl top pod co db-0' || fail 'kubectl top pod chua co db-0'
DBN="$(jp pod/db-0 '{.spec.nodeName}')"
kubectl get --raw "/api/v1/nodes/$DBN/proxy/metrics/resource" 2>/dev/null | grep '^container_memory_working_set_bytes{' \
  | grep 'container="postgres"' | grep -q 'pod="db-0"' \
  && pass 'kubelet /metrics/resource co bo nho cua postgres' || fail 'kubelet /metrics/resource thieu db-0'
API="$(k get pod -l "$SEL,app.kubernetes.io/component=api" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)"
kubectl -n "$NS" port-forward "pod/$API" 13001:3001 >/dev/null 2>&1 & PF=$!
M=''
for i in $(seq 1 20); do M="$(curl -fsS http://127.0.0.1:13001/metrics 2>/dev/null)" && [ -n "$M" ] && break; sleep 1; done
kill "$PF" 2>/dev/null; wait "$PF" 2>/dev/null
echo "$M" | grep -q '^pgrst_db_pool_max' && pass 'PostgREST /metrics doc duoc qua port-forward' || fail 'khong doc duoc /metrics cua PostgREST'

# 15. Repo và bí mật (3b, 9a)
git check-ignore -q secrets/ci-deployer.kubeconfig \
  && pass 'kubeconfig CI nam ngoai git' || fail 'secrets/ khong bi .gitignore chan'
PGPW="$(k get secret db-credentials -o go-template='{{index .data "postgres-password"}}' 2>/dev/null | base64 -d 2>/dev/null)"
if [ -n "$PGPW" ]; then
  git grep --untracked -qF -e "$PGPW" && fail 'mat khau nam trong file se vao git' || pass 'repo khong chua mat khau'
  kubectl get pods -A -l "$SEL" -o yaml 2>/dev/null | grep -qF -e "$PGPW" \
    && fail 'mat khau xuat hien trong spec Pod' || pass 'spec Pod khong chua mat khau'
else
  fail 'khong doc duoc Secret db-credentials'
fi

exit "$FAIL"
EOF
chmod +x ~/bookstore/tests/smoke.sh
```

2. Cập nhật `docs/tech-debt.md`: `TD-16` (B3) và `TD-17` (B5) ở trạng thái `mở — chấp nhận có điều kiện`;
   không nợ nào được trả ở sprint này.

3. Dựng lại sạch từ repo và chạy kiểm thử khói:

```bash
cd ~/bookstore
kubectl delete namespace bookstore-thu --ignore-not-found --wait=true
./scripts/teardown.sh && ./scripts/deploy.sh > ~/lab-evidence/11a/b8-deploy.txt 2>&1
./tests/smoke.sh > ~/lab-evidence/11a/b8-smoke.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/11a/b8-smoke.txt
```

4. Commit, tag, đóng gói, chép ra ngoài VM theo
   [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm):

```bash
cd ~/bookstore
git status --short
git add -A && git commit -m 'lab-11a: sprint 17 — metric, node-agent doc log node, metrics-server'
git tag lab-11a
git bundle create ~/bookstore-lab-11a.bundle --all && git bundle verify ~/bookstore-lab-11a.bundle
```

Chép `~/bookstore-lab-11a.bundle` về máy host bằng `scp` **trước** khi đi tiếp — bước 9 tắt máy để chụp
snapshot.

5. Dọn và gỡ dự án:

```bash
rm -rf ~/lab-work/11a
cd ~/bookstore && ./scripts/teardown.sh
```

6. Gate cuối về repo, dự án và tính toàn vẹn của node:

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
cd ~/bookstore; EV=~/lab-evidence/11a
test -z "$(git status --porcelain)" && echo 'PASS: repo khong con thay doi chua commit'
git describe --tags --exact-match HEAD | grep -qx 'lab-11a' && echo 'PASS: HEAD mang tag lab-11a'
test -s ~/bookstore-lab-11a.bundle && echo 'PASS: co bundle'
{ test -f k8s/system/kustomization.yaml && test -f k8s/system/agent/daemonset.yaml && test ! -e k8s/app/agent \
  && test -s platform/metrics-server/README.md; } && echo 'PASS: bo cuc repo dung muc 5.5'
BAD=0
for F in 'adr/*-nguon-metric.md' 'adr/*-quyen-doc-metric.md' 'adr/*-metrics-server.md' 'adr/*-node-agent-log.md' \
         'adr/*-log-he-thong-va-trace.md' 'postmortem/11a-xoay-vong-log.md' 'postmortem/11a-chan-doan-ung-dung.md' 'runbook-log.md'; do
  ls docs/$F >/dev/null 2>&1 || { echo "FAIL: thieu docs/$F"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: du ADR, postmortem va runbook cua sprint 17'
grep -q 'TD-16' docs/tech-debt.md && grep -q 'TD-17' docs/tech-debt.md && echo 'PASS: tech-debt.md co TD-16, TD-17'
test "$(sl kubectl get namespace,priorityclass,storageclass,clusterrole,clusterrolebinding -l app.kubernetes.io/part-of=bookstore --no-headers)" -eq 0 \
  && echo 'PASS: khong con object nao cua du an'
test ! -e ~/lab-work/11a && echo 'PASS: da don lab-work'

{
  echo "lab-k8s-master kubelet $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for n in lab-k8s-worker1 lab-k8s-worker2; do
    echo "$n kubelet $(ssh "$n" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done
  for f in kube-apiserver kube-scheduler kube-controller-manager; do
    echo "lab-k8s-master $f $(sudo sha256sum /etc/kubernetes/manifests/$f.yaml | awk '{print $1}')"
  done
} > "$EV/b8-config-sha.txt"
diff -u "$EV/b0-config-sha.txt" "$EV/b8-config-sha.txt" \
  && echo 'PASS: cau hinh kubelet va manifest control plane khong doi trong suot lab' \
  || echo 'FAIL: co file cau hinh da bi sua — KHONG chup snapshot, xem muc 4'
```

7. Gate trạng thái của mốc mới — có metrics-server, không còn gì của dự án, tầng lưu trữ đúng như
   `03-storage-ready`:

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
R="$(kubectl -n kube-system get deploy metrics-server -o jsonpath='{.status.readyReplicas}')"
AV="$(kubectl get apiservice v1beta1.metrics.k8s.io -o jsonpath='{.status.conditions[?(@.type=="Available")].status}')"
{ test "${R:-0}" -ge 1 && test "$AV" = 'True'; } && echo 'PASS: metrics-server Ready, Metrics API Available'
test -z "$(kubectl -n kube-system get deploy metrics-server -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}')" \
  && echo 'PASS: metrics-server khong mang label du an'
test "$(kubectl top node --no-headers | wc -l)" -eq 3 && echo 'PASS: kubectl top node — dieu kien dau vao cua Lab 11b'
test "$(kubectl top pod -n kube-system --no-headers | wc -l)" -gt 0 && echo 'PASS: kubectl top pod chay duoc'
{ test "$(sl kubectl get pv --no-headers)" -eq 0 && test "$(sl kubectl get pvc -A --no-headers)" -eq 0; } \
  && echo 'PASS: khong con PV, PVC nao'
SC="$(kubectl get sc -o jsonpath='{range .items[*]}{.metadata.name}={.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}{"\n"}{end}')"
{ test "$(echo "$SC" | grep -c .)" -eq 1 && echo "$SC" | grep -q '=true$'; } && echo 'PASS: dung mot StorageClass mac dinh nhu 03-storage-ready'
test -z "$(kubectl get nodes -l bookstore.example.com/data -o name)" && echo 'PASS: khong node nao mang label du lieu'

kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

8. Lab đổi hạ tầng nên **không** dừng ở gate ngắn: chạy **toàn bộ bảy tầng** của
   [A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready), từ
   [tầng 0](LAB-00-MOI-TRUONG-1.35.7.md#a541-tầng-0--prereq-os-còn-đúng-sau-khi-cluster-chạy) tới
   [tầng 6](LAB-00-MOI-TRUONG-1.35.7.md#a547-tầng-6--đường-control-plane--kubelet), rồi dọn resource test
   theo [A5.4.8](LAB-00-MOI-TRUONG-1.35.7.md#a548-dọn-resource-test-ghi-evidence-và-chụp-snapshot) —
   **dừng trước** bước tắt máy của A5.4.8.

| Tầng | Điều chỉnh cho mốc `04-metrics-ready` |
| --- | --- |
| 0, 1, 4, 5 | không đổi |
| 2 | Không còn `kube-flannel`: CNI là Calico do [Lab 5b](LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) cài. Cột `Memory Requests` có thêm phần của metrics-server — vẫn phải dưới ngưỡng của tầng này |
| 3 | Đường hầm giữa các node do Calico dựng, không còn VXLAN cổng `8472` của Flannel; nguyên nhân fail đọc theo Lab 5b |
| 6 | Quan trọng gấp đôi: metrics-server đi đúng đường này tới kubelet; tầng 6 hỏng là `kubectl top` chết theo |

Tập namespace hệ thống hợp lệ ở mốc này: `kube-system` (có thêm metrics-server), `traefik`,
`local-path-storage`, cộng namespace của Calico nếu manifest của Lab 5b tạo riêng.

Ghi hồ sơ của mốc:

```bash
EV=~/lab-evidence/11a
{
  date -Is; echo '=== 04-metrics-ready ==='
  kubectl -n kube-system get deploy metrics-server -o wide
  kubectl -n kube-system get deploy metrics-server -o jsonpath='{range .spec.template.spec.containers[0].args[*]}{@}{"\n"}{end}'
  kubectl get apiservice v1beta1.metrics.k8s.io
  kubectl top node
  kubectl get sc; kubectl get pv; kubectl get namespaces
  kubectl get pods -A -o wide
} > "$EV/b8-ho-so-04-metrics-ready.txt" 2>&1
grep -q 'metrics-server' "$EV/b8-ho-so-04-metrics-ready.txt" && ! grep -qE '^bookstore' "$EV/b8-ho-so-04-metrics-ready.txt" \
  && echo 'PASS: ho so moc moi co metrics-server, khong con gi cua du an'
```

9. Tắt máy và chụp. Chạy trên **từng node** theo thứ tự worker 2 → worker 1 → master:

```bash
sudo shutdown -h now
```

Chờ VMware Workstation hiển thị cả ba VM *Powered off*. Chụp trên **cả ba VM**: chuột phải VM →
**Snapshot → Take Snapshot** → ô *Name* điền đúng nguyên văn:

```text
04-metrics-ready
```

Ô *Description*: `dựng bằng LAB-11A-OBSERVABILITY.md, repo tag lab-11a, chụp <ngày>`. Quy tắc tên giống
Lab 00: đúng nguyên văn trên cả ba VM, không hậu tố theo VM, không thêm ngày, không đổi hoa thường. **Giữ
nguyên `03-storage-ready`** — nó vẫn là điểm bắt đầu của các lab từ 6b tới 9b và là điểm quay lui của mốc
mới. Kiểm từ máy host:

```powershell
$vmrun = 'C:\Program Files\VMware\VMware Workstation\vmrun.exe'
$vmx = @(
  'E:\Virtual Machines\lab-k8s-master\lab-k8s-master.vmx'
  'E:\Virtual Machines\lab-k8s-worker1\lab-k8s-worker1.vmx'
  'E:\Virtual Machines\lab-k8s-worker2\lab-k8s-worker2.vmx'
)
foreach ($f in $vmx) {
  $names = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  if (($names -ccontains '04-metrics-ready') -and ($names -ccontains '03-storage-ready')) { "PASS: $f" }
  else { "FAIL: $f -> $($names -join ', ')" }
}
```

**PASS:** `smoke exit=0` và file smoke ở bước 3 toàn `PASS:`; mọi dòng `PASS:` của bước 6 và 7, không dòng
`FAIL:` — đặc biệt dòng checksum; bảy tầng A5.4 PASS; dòng hồ sơ; ba dòng `PASS:` của `vmrun`, cả ba VM có
**cả hai** mốc. Thư mục `~/lab-evidence/11a/` **giữ lại**. Lab 11b bật máy theo
[A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab) từ mốc mới.

---

## 3. Checkpoint 11a

Không đánh dấu lab hoàn tất chỉ vì script nghiệm thu PASS. Tự trả lời, không nhìn tài liệu:

- [ ] Kể ba trụ cột quan sát. Trên Bookstore sau sprint này, mỗi trụ cột có nguồn phát nào, khúc nào
      của pipeline "nguồn → thu thập → lưu trữ → dashboard" đã có, khúc nào còn trống và vì sao?
- [ ] `/metrics` của `api` nằm ở đâu, ai đọc được? Vì sao `web` không gọi được cổng `admin` mà
      `kubectl port-forward` thì được, và điều đó nói gì về quyền `pods/portforward`?
- [ ] "`db-0` dùng bao nhiêu bộ nhớ" và "`web` có mấy bản sao sẵn sàng": mỗi câu đến từ nguồn nào, câu
      nào không có dạng metric trên cluster của bạn, cần thêm gì?
- [ ] Kubelet có bốn endpoint metric. Kể đủ, nói mỗi cái phục vụ gì; "không cùng vòng đời" ảnh hưởng
      dashboard ra sao; mức ổn định của một metric đọc ở đâu?
- [ ] Một scraper có token hợp lệ vẫn nhận `403` ở `/metrics` của API server. Chặng nào chặn, sửa bằng
      object nào, vì sao object đó không mở luôn `/metrics/resource`? `/metrics` của PostgREST được bảo
      vệ bằng gì?
- [ ] metrics-server vừa cài trên cluster kubeadm không `Ready`: nguyên nhân gốc, hai đường sửa, đánh
      đổi, đường lab chọn và nợ nào ghi lại. Kể chuỗi phụ thuộc từ `kubectl top` về nơi đo.
- [ ] Ba câu hỏi giám sát về Bookstore mà metrics-server không trả lời được, mỗi câu thành phần nào trả
      lời. Vì sao metrics-server không mang label dự án và nằm trong snapshot?
- [ ] `node-agent`: vì sao phải rời `bookstore`, chỗ nào từ chối nếu không rời; agent đọc được file log
      `0640 root:root` bằng cách nào mà không chạy root; nó thấy gì, công bố gì, và vì sao NetworkPolicy
      của `bookstore-system` là bắt buộc?
- [ ] `thu-log-flood` in 40 000 dòng: `kubectl logs` trả bao nhiêu, ai quyết định, phần còn lại ở đâu,
      ai đọc được nó, xóa Pod thì sao, thiếu khúc nào để giữ được (`TD-17`)?
- [ ] Trên `lab-k8s-worker1`, log của kubelet ở đâu, log của kube-scheduler ở đâu, vì sao? Tăng `-v` thì
      log thêm loại sự kiện nào? Vì sao không cảnh báo bằng cách khớp chuỗi trong log?
- [ ] Cluster của bạn có bật tracing không — kiểm ở ba chỗ nào? Bật thì sửa những gì, thuộc giai đoạn
      nào; kubelet có trace cho thêm gì mà log không có?
- [ ] Kể lại quy trình chẩn đoán `thu-catalog`: xếp nhánh, bằng chứng theo thứ tự, công cụ ở mỗi bước
      (`logs`, `--previous`, `exec` có `-c`, `--`), chỗ nào dừng để nhường giai đoạn 24.

Nợ còn mở sau lab này — nói được **vì sao chưa trả**:

- Nợ lab [#1](README.md#5-sổ-nợ-lab) — HPA trả ở Lab 11b trên chính mốc vừa chụp (**đọc lại bài
  [72](../72-horizontal-pod-autoscale-vi.md) trước khi mở Lab 11b**); VPA chưa có lab trả. Sprint này
  không phát sinh nợ lab mới.
- Nợ kỹ thuật: `TD-16` (metrics-server không xác thực kubelet) và `TD-17` (không có nơi lưu metric, log
  dài hạn) — mới, chấp nhận có điều kiện; `TD-10` vẫn chấp nhận có điều kiện. Các nợ còn lại đã trả.

### Bài giải thích cuối cùng

Trong vài phút, kể lại bằng lời hai luồng trên chính hệ thống Bookstore của bạn:

1. **Một con số đi từ container `postgres` tới màn hình `kubectl top pod db-0`.** Ai đo, số liệu xuất
   hiện ở endpoint nào của kubelet trên `lab-k8s-worker2`, ai gọi endpoint đó bằng quyền gì và qua
   certificate nào, kết quả được đăng ký vào Kubernetes API dưới hình thức nào, `kubectl top` gọi cái
   gì. Ở mỗi mắt xích nói một cách nó hỏng và triệu chứng. Rồi: muốn xem lại con số đó vào tuần sau thì
   thiếu mắt xích nào.
2. **Một dòng access log của nginx trong Pod `web` đi tới chỗ người trực đọc được.** Ai bắt nó, chuẩn hóa
   theo định dạng gì, nằm ở đường dẫn nào trên node, `kubectl logs` đọc nó qua đâu, `node-agent` đọc nó
   bằng quyền gì từ namespace nào, ba ranh giới nó không vượt qua được — xoay vòng, Pod bị xóa, node
   chết. Cuối cùng chỉ ra phần nào của kiến trúc ghi log cấp cluster đã có trong dự án, phần nào chưa,
   và phần nào nằm ngoài Kubernetes.

Khi mọi checkbox được đánh dấu và bạn không còn nhầm **metric tài nguyên** với **metric trạng thái đối
tượng**, **metrics-server** với **Prometheus**, **RBAC** với **NetworkPolicy** khi bảo vệ `/metrics`,
**journald** với **`/var/log/pods`**, hay **sự cố ứng dụng** với **sự cố cluster** — sprint 17 và
[giai đoạn 11](../00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability) hoàn tất.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường xem [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Dưới đây chỉ là sự cố phát sinh trong nội dung bài học.

| Triệu chứng | Nguyên nhân thường gặp | Xử lý |
| --- | --- | --- |
| B0: `git describe` không ra `lab-9b` | Repo ở commit khác, hoặc clone từ bundle cũ | `git checkout lab-9b` hoặc clone lại từ bundle `lab-9b` mới nhất |
| B0: `configz` không đọc được trên một node | Đường control plane → kubelet hỏng — cũng làm hỏng `logs`, `exec`, metrics-server | Chạy lại [tầng 6 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a547-tầng-6--đường-control-plane--kubelet); không sửa gì trên node; vẫn hỏng thì restore `03-storage-ready` |
| B1: `curl` vào cổng `13001` không trả gì | Port-forward chưa kịp mở, hoặc cổng cục bộ đang bị một port-forward cũ giữ | Vòng chờ trong script; `pkill -f 'port-forward'` rồi chạy lại |
| B1: `wget` từ `web` tới cổng `3001` của `api` **thành công** | NetworkPolicy của `api` mở cả Pod chứ không chỉ cổng `http` | Đúng là lỗi so với [ma trận mục 3.5](DU-AN-BOOKSTORE.md#35-thiết-kế-bảo-mật): sửa policy ở `k8s/app/network/`, không nới thêm |
| B1: không có dòng `# TYPE pgrst_...` | Không lấy được `/metrics` — file evidence rỗng hoặc là trang lỗi | Đọc `b1-port-forward.log`; kiểm `PGRST_ADMIN_SERVER_PORT` trong ConfigMap `api-config` |
| B1: `/metrics/probes` không có `prober_probe_total` của `postgrest` | Gọi nhầm node — endpoint chỉ có probe của Pod trên chính node đó | Đọc lại `API_NODE`; gọi trên đúng node của Pod `api` |
| B2: mã trả về `401` chứ không phải `403` | Token hết hạn hoặc lấy từ ServiceAccount khác | Sinh token mới ngay trước `curl`; gate chỉ đòi khác `200` nhưng hãy biết mình đang chẩn đoán gì |
| B2: `can-i` trả `yes` mà `curl` vẫn không `200` | Dùng token cũ, hoặc `SRV` trỏ nhầm endpoint | Sinh token mới; in lại `SRV` |
| **B3: metrics-server không `Ready`, log có `x509`** | **Hành vi đúng của bước này** — certificate phục vụ của kubelet tự ký | Đọc ý chính 1 của lời giải B3 rồi vá. Đừng bỏ qua phần 2 của script "cho nhanh" |
| B3: đã vá mà vẫn không `Ready`, log **không** có `x509` | Không tới được địa chỉ node | Đối chiếu `--kubelet-preferred-address-types` với `.status.addresses` của Node; chạy lại tầng 6 |
| B3: `ImagePullBackOff` cho metrics-server | Đây là image duy nhất lab kéo từ mạng; DNS ra ngoài hỏng, hoặc `MS_VERSION` sai | [Tầng 4 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a545-tầng-4--dns-trong-cluster-và-ra-internet); `MS_VERSION` phải khớp A1.4, không sửa tay tag trong manifest |
| B3: `--kubelet-insecure-tls` xuất hiện hai lần | Chạy `patch` hai lần — `add` vào `args/-` luôn nối thêm | `kubectl -n kube-system edit deploy metrics-server` xóa bớt một dòng; ghi chú vào `platform/metrics-server/README.md` |
| B3: `kubectl top node` lỗi dù APIService `Available` | Chưa xong chu kỳ thu thập đầu tiên — phụ thuộc cấu hình | Chạy lại vòng chờ; vẫn rỗng thì đọc log metrics-server |
| B3: `kubectl top pod -n bookstore` thiếu Pod | Pod mới lên chưa có đủ mẫu, hoặc đếm cả Pod `Completed` của Job | Chờ thêm; script đếm theo `status.phase=Running` |
| B4: `deploy.sh` báo `no objects passed to apply` | `k8s/app/kustomization.yaml` không còn object nào không mang `component=migrate`, hoặc render rỗng | `kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/app | head`; kiểm danh sách `resources` |
| B4: `deploy.sh` treo ở `wait_job db-migrate` | Job không mang label `component=migrate` ở `metadata`, hoặc migration lỗi | Đối chiếu [hợp đồng](DU-AN-BOOKSTORE.md#41-tên-và-label); `kubectl -n bookstore logs job/db-migrate` |
| B4: DaemonSet `node-agent` có `DESIRED` 3 nhưng không Pod nào, sự kiện `FailedCreate ... violates PodSecurity` | Namespace `bookstore-system` chưa có label `enforce: privileged`, hoặc DaemonSet vẫn nằm ở `bookstore` | Kiểm `kubectl get ns bookstore-system --show-labels` và `namespace` trong manifest |
| B4: `/logs.txt` có dòng `file` nhưng dòng `last` trống; log của agent có `Permission denied` | Agent không đọc được file `0640 root:root` | Kiểm `id -G` trong container agent có `0`; xem câu hỏi thiết kế 2 của B4. Không chuyển sang `privileged` |
| B4: `/logs.txt` có `files=0` trên worker | Mount sai đường dẫn, hoặc `LOG_NAMESPACE` sai | `kubectl -n bookstore-system exec <agent> -- ls /var/log/pods`; tên thư mục có dạng `bookstore_<pod>_<uid>` |
| B4: `status-writer` không lấy được `/node.json` | Còn dùng tên ngắn `node-agent` (giờ không phân giải trong `bookstore`), hoặc thiếu rule egress sang `bookstore-system` | Dùng FQDN; rule egress có **cả** `namespaceSelector` lẫn `podSelector` trong cùng một phần tử `to` |
| B4: Pod ở `bookstore-thu` vẫn gọi được agent | Thiếu policy chặn mặc định trong `bookstore-system`, hoặc `namespaceSelector` và `podSelector` tách thành hai phần tử | Đọc lại bài 84, mục *Hành vi của các bộ chọn `to` và `from`* |
| B4: `teardown.sh` báo còn PV | PV `Released` với thư mục ngoài `/opt/local-path-provisioner` | Script chỉ xóa thư mục dưới gốc đó; kiểm `kubectl get pv -o wide` và dọn theo [mục 4 của Lab 6a](LAB-6A-PV-PVC-VA-STORAGECLASS.md#4-troubleshooting-của-lab-này) |
| B5: chỉ có một file trong thư mục log | Khối log chưa vượt ngưỡng, hoặc kubelet chưa tới chu kỳ kiểm | Chờ hết vòng lặp; `containerLogMaxSize` của node lớn hơn mặc định thì tăng số dòng trong Pod, không sửa kubelet |
| B5: gate byte fail | `to_bytes` gặp đơn vị chưa có nhánh | Thêm nhánh vào hàm trong phiên shell |
| B5: thư mục log còn sau khi xóa Pod | Pod còn trong ân hạn | Chờ vòng lặp; **không** `rm -rf` trong `/var/log/pods` |
| B6: `journalctl -u kubelet` rỗng | journald giới hạn dung lượng | `--since '-1h'`; ghi vào evidence; gate thứ hai vẫn kiểm được ranh giới |
| B6: `N6` không lớn hơn `N2` | Ghi nhầm `stdout` thay vì `stderr` | klog luôn ghi `stderr`; giữ `2>` |
| B6: có `--tracing-config-file` trên apiserver | Cluster lệch baseline | **Không** gỡ để "đưa về chuẩn"; ghi vào ADR, đọc lại ý chính theo cấu hình thật |
| B7: client báo `LOI` cả sau khi sửa selector | Chưa apply lại Service, hoặc Pod `thu-catalog` chưa `Ready` | `kubectl -n bookstore-thu get endpointslice`; chờ readiness |
| B7: `kubectl exec -it` báo `Unable to use a TTY` | Chạy trong script không có terminal | Shell tương tác chỉ chạy tay; script dùng lệnh rời |
| B8: smoke fail ở mục RBAC | Tên ServiceAccount CI hoặc nhóm khác với [mục 3.5](DU-AN-BOOKSTORE.md#35-thiết-kế-bảo-mật) | Sửa repo cho khớp hợp đồng, không sửa smoke |
| **B8: `diff` checksum báo khác** | Một file cấu hình đã bị sửa | **Không chụp snapshot.** Tắt và restore cả ba VM về `03-storage-ready`, làm lại từ B0 |
| B8: `kubectl top` fail ở gate cuối | metrics-server vừa bị dời chỗ khi teardown, chưa thu thập lại | Chờ rồi chạy lại bước 7; **không chụp** khi gate này chưa PASS — Lab 11b sẽ không mở được |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Observability](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/observability/)
- [Kubernetes v1.35 — Metrics For Kubernetes System Components](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/system-metrics/)
- [Kubernetes v1.35 — Metrics for Kubernetes Object States](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/kube-state-metrics/)
- [Kubernetes v1.35 — Logging Architecture](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/logging/)
- [Kubernetes v1.35 — System Logs](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/system-logs/)
- [Kubernetes v1.35 — Traces For Kubernetes System Components](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/system-traces/)
- [Kubernetes v1.35 — Monitoring, Logging, and Debugging](https://v1-35.docs.kubernetes.io/docs/tasks/debug/)
- [Kubernetes v1.35 — Troubleshooting Applications](https://v1-35.docs.kubernetes.io/docs/tasks/debug/debug-application/)
- [Kubernetes v1.35 — Get a Shell to a Running Container](https://v1-35.docs.kubernetes.io/docs/tasks/debug/debug-application/get-shell-running-container/)
- [Kubernetes v1.35 — Developing and debugging services locally using telepresence](https://v1-35.docs.kubernetes.io/docs/tasks/debug/debug-cluster/local-debugging/)
- [Kubernetes v1.35 — Logging in Kubernetes](https://v1-35.docs.kubernetes.io/docs/tasks/debug/logging/)
- [Kubernetes v1.35 — Monitoring in Kubernetes](https://v1-35.docs.kubernetes.io/docs/tasks/debug/monitoring/)
- [Kubernetes v1.35 — HorizontalPodAutoscaler Walkthrough](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
- [Kubernetes v1.35 — Resource metrics pipeline](https://v1-35.docs.kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)
- [Kubernetes v1.35 — Kubelet Configuration (v1beta1)](https://v1-35.docs.kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)
  — `containerLogMaxSize`, `containerLogMaxFiles`, `enableSystemLogHandler`, `enableSystemLogQuery`, `tracing`
- [Kubernetes v1.35 — Using RBAC Authorization](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/rbac/)
  — `nonResourceURLs`
- [Kubernetes v1.35 — Kubelet authentication/authorization](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/)
  — `nodes/metrics`, cảnh báo về `nodes/proxy`
- [Kubernetes v1.35 — Pod Security Admission](https://v1-35.docs.kubernetes.io/docs/concepts/security/pod-security-admission/)
- [Kubernetes v1.35 — Volumes, mục hostPath](https://v1-35.docs.kubernetes.io/docs/concepts/storage/volumes/#hostpath)
- [Kubernetes v1.35 — Network Policies](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kubernetes v1.35 — Service Internal Traffic Policy](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/service-traffic-policy/)
- [Kubernetes v1.35 — Use Port Forwarding to Access Applications in a Cluster](https://v1-35.docs.kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)
- [metrics-server — README và bảng tương thích](https://github.com/kubernetes-sigs/metrics-server#compatibility-matrix)
- [metrics-server — FAQ, certificate của kubelet](https://github.com/kubernetes-sigs/metrics-server/blob/master/FAQ.md)
- [PostgREST v14 — Observability](https://docs.postgrest.org/en/v14/references/observability.html)
- [PostgREST v14 — Admin Server](https://docs.postgrest.org/en/v14/references/admin_server.html)
- [containerd — CRI, nơi runtime tạo file log container](https://github.com/containerd/containerd/tree/main/internal/cri)

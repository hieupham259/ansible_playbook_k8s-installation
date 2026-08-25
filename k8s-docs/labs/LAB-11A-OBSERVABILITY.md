# Lab 11a — Observability

> **Điểm bắt đầu:** snapshot `03-storage-ready` — mốc do Lab 6a tạo, xem
> [chuỗi snapshot](README.md#3-chuỗi-snapshot).
> **Điểm kết thúc:** lab **tạo mốc mới `04-metrics-ready`**. Đây là một trong bốn lab đổi hạ tầng
> vĩnh viễn trên chuỗi chính: nó cài **metrics-server** và để lại thành phần đó cho mọi lab sau.
> **Lab trước:** [Lab 7b — Quota và giới hạn tài nguyên](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md)
> đã cleanup và trả cluster về đúng `03-storage-ready`: có provisioner và StorageClass mặc định,
> không workload, không ResourceQuota, không LimitRange.
>
> **Cập nhật và đối chiếu phiên bản:** 25/08/2026.

Lab này đi cùng mục
[Giai đoạn 11 — Observability](../00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability) — sáu bài
khái niệm [162](../162-observability-vi.md), [160](../160-system-metrics-vi.md),
[163](../163-kube-state-metrics-vi.md), [158](../158-logging-vi.md),
[159](../159-system-logs-vi.md), [161](../161-system-traces-vi.md), cộng sáu bài thực hành
[296](../296-debug-vi.md), [297](../297-debug-application-vi.md),
[304](../304-get-shell-running-container-vi.md), [309](../309-local-debugging-vi.md),
[316](../316-debug-logging-vi.md), [317](../317-debug-monitoring-vi.md).

Bài [342 — Hướng dẫn từng bước về HorizontalPodAutoscaler](../342-hpa-walkthrough-vi.md) cũng nằm
ở dòng thực hành của giai đoạn 11 nhưng **không thuộc lab này**: nó là nội dung của Lab 11b, nơi
[nợ #1](README.md#5-sổ-nợ-lab) được trả. Lab 11a chỉ dựng điều kiện đầu vào cho nó.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab này **không chép
lại con số phiên bản Kubernetes nào**. Thành phần duy nhất lab cài thêm là metrics-server, và
version của nó phải đọc từ
[bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) — xem
[B5](#b5-cài-metrics-server-và-chữa-lỗi-certificate).

Lab dùng Pod, ConfigMap, downwardAPI của giai đoạn 3, Deployment/DaemonSet của giai đoạn 4,
`hostPath` của giai đoạn 6, toleration của nhóm 7a và **RBAC của giai đoạn 9** làm công cụ.
**Không** dùng HPA (Lab 11b), `kubectl drain` hay node autoscaling (giai đoạn 12), DRA
(giai đoạn 13), CRD hay Operator (giai đoạn 14).

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`, rồi thêm ba lệnh riêng của lab này:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default

# Ba lệnh riêng của lab 11a: cluster chưa có nguồn metric tài nguyên nào.
kubectl get apiservice 2>/dev/null | grep -i 'metrics.k8s.io' || echo 'chua co APIService metrics.k8s.io'
kubectl -n kube-system get deployment metrics-server --ignore-not-found
kubectl top node 2>&1 | head -2
```

**PASS:** ba node `Ready`; có dòng `PASS: readyz ok`; lệnh field selector trả `No resources found`;
CoreDNS đủ replica `READY`; namespace `default` không có Pod; **lệnh `apiservice` in dòng
`chua co APIService metrics.k8s.io`**, lệnh `deployment` không trả gì, và `kubectl top node` **báo
lỗi** — đây là trạng thái đúng của `03-storage-ready`. Nếu `kubectl top` đã chạy được thì cluster
của bạn không ở mốc đầu vào; restore cả ba VM về `03-storage-ready` trước khi tiếp tục.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi có thể tự chứng minh và giải thích được:

- Ba trụ cột **metrics, logs, traces** trả lời ba câu hỏi khác nhau: chỉ ra được trên chính cluster
  của mình trụ cột nào đang có nguồn, trụ cột nào chưa, và câu hỏi vận hành nào rơi vào khoảng
  trống đó.
- Metric của thành phần hệ thống nằm ở endpoint `/metrics` của từng thành phần: gọi được endpoint
  của kube-apiserver và của kubelet, đọc được **định dạng Prometheus** (`# HELP`, `# TYPE`, dòng
  mẫu), và chỉ ra **mức ổn định** của một metric ngay trong dòng `HELP` của nó.
- Kubelet có thêm `/metrics/cadvisor`, `/metrics/resource` và `/metrics/probes`: gọi được cả ba,
  nói đúng mỗi endpoint phục vụ mục đích gì, và vì sao bài
  [160](../160-system-metrics-vi.md) nhấn mạnh chúng **không cùng vòng đời** với `/metrics`.
- **Đọc metric là một hành động được ủy quyền**: chứng minh bằng chính cluster rằng một
  ServiceAccount không có quyền bị từ chối ở endpoint `/metrics` của kube-scheduler, và một
  ClusterRole với `nonResourceURLs` mở đúng cánh cửa đó — không hơn.
- Ranh giới của bài [163](../163-kube-state-metrics-vi.md): phân biệt **metric tài nguyên** (thứ
  kubelet đo được trên node của nó) với **metric trạng thái đối tượng** (thứ chỉ sinh ra được từ
  Kubernetes API), và chứng minh bằng lệnh rằng loại thứ hai **không tồn tại** trên cluster chưa
  cài add-on nào.
- Cài được **metrics-server** trên cluster kubeadm: chẩn đoán đúng vì sao nó không `Ready` ngay,
  đọc được nguyên nhân trong log của chính nó, sửa đúng chỗ, và giải thích được **đánh đổi** của
  cách sửa đó so với cách đúng đắn hơn.
- `kubectl top node` và `kubectl top pod` chạy được — **điều kiện đầu vào của Lab 11b** — và nói
  được metrics-server phục vụ cái gì mà **không** phục vụ cái gì.
- Đường đi của log container: `kubectl logs` lấy dữ liệu từ đâu, vì sao một container ghi ra file
  thì `kubectl logs` không thấy gì, và vì sao **sidecar truyền luồng** lại thấy được.
- **Kubelet chịu trách nhiệm xoay vòng log**: đọc `containerLogMaxSize`/`containerLogMaxFiles`
  thật của kubelet, làm tràn ngưỡng đó bằng một workload thật, rồi chứng minh **bằng con số** rằng
  `kubectl logs` chỉ trả về file log mới nhất.
- Kiến trúc **agent ghi log cấp node**: dựng được một agent dạng `DaemonSet` một Pod mỗi node,
  chứng minh nó thấy log của **mọi** container trên node của nó và **không thấy** của node khác.
- Log của thành phần hệ thống chia hai loại theo **nơi chúng chạy**: chứng minh trên node thật
  rằng kubelet ghi journald còn thành phần chạy trong Pod ghi file `.log`, và đọc được mức
  verbosity `-v` như một thang đo có hướng.
- **Trace**: xác định được cluster đang **không** bật tracing, chỉ ra bằng cấu hình thật của
  kube-apiserver và của kubelet, và nói được bật nó sẽ đổi cái gì — mà **không sửa gì**.
- Bộ công cụ gỡ lỗi ở mức giai đoạn 11: `kubectl logs --previous`, `kubectl exec` mở shell và chạy
  lệnh rời, `-c` cho Pod nhiều container, và ranh giới giữa phần này với giai đoạn 24.
- Cleanup đúng phạm vi: xóa mọi object của bài học nhưng **giữ lại metrics-server**, chứng minh
  cấu hình node và manifest control plane **không hề bị sửa**, rồi chụp mốc `04-metrics-ready`.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 11 | Kiểm chứng ở |
| --- | --- |
| [162 — Khả năng quan sát](../162-observability-vi.md) | B1 — trang bản đồ; B1.1 chỉ ra nguồn của từng trụ cột trên cluster thật, B1.2 chứng minh khoảng trống mà lab sẽ lấp. Về sau: B2–B6 là trụ cột metric, B7–B9 là trụ cột log, B10 là trụ cột trace |
| [160 — Metric cho các thành phần hệ thống](../160-system-metrics-vi.md) | B2 — endpoint `/metrics` của kube-apiserver (B2.1), định dạng Prometheus và mức ổn định đọc ngay trong `HELP` (B2.2), ba endpoint phụ của kubelet (B2.3), chữ ký deprecated (B2.4); B3 — đọc metric là hành động **được ủy quyền**, `nonResourceURLs` và cờ `--bind-address`; B5–B6 — `/metrics/resource` chính là nguồn mà metrics-server tổng hợp |
| [163 — Metrics cho trạng thái đối tượng](../163-kube-state-metrics-vi.md) | B4 — B4.1 chứng minh metric **tài nguyên** có sẵn, B4.2 chứng minh metric **trạng thái object** (`kube_*`) không tồn tại trên cluster này, B4.3 chứng minh thông tin đó vẫn đọc được qua API nhưng **không** ở dạng metric, B4.4 đọc một metric kiểu *info* có sẵn để thấy quy tắc "thông tin nằm ở label, không ở giá trị". Lab **không cài** kube-state-metrics; lý do ở bảng dưới |
| [158 — Kiến trúc ghi log](../158-logging-vi.md) | B7 — `kubectl logs` và `--previous` (B7.1, B7.3), ranh giới `stdout` so với file cùng hai kiểu sidecar (B7.2), đường đi thật của file log trên node (B7.4), agent ghi log cấp node dạng `DaemonSet` (B7.5); B8 — kubelet xoay vòng log và hệ quả "chỉ file mới nhất" |
| [159 — Log hệ thống](../159-system-logs-vi.md) | B9 — hai loại thành phần và hai vị trí log (B9.1, B9.2), định dạng klog (B9.3), mức verbosity `-v` (B9.4), tính năng Log Query và vì sao lab chỉ **đọc** cấu hình của nó (B9.5) |
| [161 — Trace cho các thành phần hệ thống](../161-system-traces-vi.md) | B10 — cấu hình tracing thật của kube-apiserver (B10.1) và của kubelet (B10.2), cổng OTLP im lặng trên cả ba node (B10.3). Lab **không sửa** cấu hình apiserver; phần bật tracing ở bảng dưới |
| [296 — Giám sát, ghi log và gỡ lỗi](../296-debug-vi.md) | B1.3 — trang mục lục chia bốn nhánh; lab đi hai nhánh thuộc giai đoạn 11 (*Ghi log*, *Giám sát*) và ghi rõ ranh giới của hai nhánh còn lại; B11.5 nhắc lại ranh giới đó trước khi sang giai đoạn 24 |
| [297 — Xử lý sự cố ứng dụng](../297-debug-application-vi.md) | B11 — trang mục lục; phần dùng được ở giai đoạn 11 là bộ ba `logs` / `logs --previous` / `exec`, làm ở B7.1, B7.3 và B11 |
| [304 — Truy cập shell của một container đang chạy](../304-get-shell-running-container-vi.md) | B11 — mở shell bằng `kubectl exec --stdin --tty` (B11.1), ghi trang gốc rồi tự gọi từ bên trong (B11.2), chạy lệnh rời và vai trò của dấu `--` (B11.3), chọn container bằng `-c` khi Pod có nhiều container (B11.4) |
| [309 — Debug service cục bộ bằng telepresence](../309-local-debugging-vi.md) | B11.5 — kiểm chứng đúng phần đọc được: chính bài 309 nói cách thủ công là "lấy một shell vào container đang chạy", tức B11; và nó **sửa workload trên cluster** nên không chạy trong chuỗi snapshot. Lý do đầy đủ ở bảng dưới |
| [316 — Ghi log trong Kubernetes](../316-debug-logging-vi.md) | B7–B9 — trang trỏ hướng, trỏ đúng vào hai bài 158 và 159; B1.3 kiểm chứng nó nằm ở nhánh nào của bài 296 |
| [317 — Giám sát trong Kubernetes](../317-debug-monitoring-vi.md) | B2 và B10 — trang trỏ hướng, trỏ đúng vào hai bài 160 và 161; B1.3 kiểm chứng nó nằm ở nhánh nào của bài 296 |

Các phần dưới đây **không kiểm chứng được trong lab này**, ghi rõ lý do thay vì bỏ trống:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| **Stack Prometheus + Grafana** mà checkpoint giai đoạn 11 nhắc tới | Ba VM lab là 4/2/2 vCPU (xem [A1.2](LAB-00-MOI-TRUONG-1.35.7.md#a12-ba-vm)), và mốc `04-metrics-ready` là điểm bắt đầu của **năm** lab sau (11b, 12, 13, 14, 15). Một stack Prometheus + Grafana + kube-state-metrics + node-exporter chiếm phần lớn headroom còn lại của hai worker và làm snapshot phình to, khiến chính các lab đó thiếu chỗ chạy. Nó cũng cần một chart chưa được khóa version ở [A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00). Lộ trình đã có chỗ cho nó: [giai đoạn 23](../00-ALO-TRINH-ADMIN.md#giai-đoạn-23--giám-sát-và-cảnh-báo) là phần vận hành thật, chạy trên cluster production chứ không trên ba VM lab. B1.2 và B6.4 kiểm chứng **khoảng trống** mà stack đó lấp, đó là phần học được ở đây |
| **kube-state-metrics** — cài thật rồi scrape | Cùng lý do headroom, và bản thân bài [163](../163-kube-state-metrics-vi.md) nói rõ đây là **agent add-on bên thứ ba**, không phải thành phần core. Cài nó mà không có Prometheus thì chỉ được một endpoint HTTP không ai đọc. B4 kiểm chứng **ranh giới** mà bài dạy — thứ thật sự phải hiểu — bằng metric có sẵn trên cluster: metric tài nguyên có, metric `kube_*` không |
| Câu truy vấn PromQL và quy tắc alert trong bài [163](../163-kube-state-metrics-vi.md) | Cần Prometheus. Chính bài xếp hai mục này vào *Đọc lướt* và hoãn tới [giai đoạn 23](../00-ALO-TRINH-ADMIN.md#giai-đoạn-23--giám-sát-và-cảnh-báo) |
| Bài [160](../160-system-metrics-vi.md): metric **PSI** của kubelet và mục *Yêu cầu* | Bài tự xếp phần này vào *Đọc lướt* và hoãn tới [giai đoạn 24](../00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố): đọc được số PSI là kỹ năng chẩn đoán, không phải kỹ năng dựng nguồn metric. B2.3 vẫn gọi endpoint chứa chúng, nhưng không diễn giải |
| Bài [160](../160-system-metrics-vi.md): `--show-hidden-metrics-for-version`, `--disabled-metrics`, `--allow-metric-labels` | Cả ba là cờ dòng lệnh của thành phần control plane; đặt chúng nghĩa là sửa manifest static Pod rồi chờ kubelet dựng lại — việc của [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy), bài [196](../196-configure-feature-gates-vi.md). Lab này **cấm sửa** `/etc/kubernetes/manifests`, và B12.2 chứng minh bằng checksum |
| Bài [160](../160-system-metrics-vi.md): metric của cloud provider và endpoint `/metrics/resources` của kube-scheduler | Cluster lab là bare-metal, không có cloud provider nào để sinh `cloudprovider_*`. Hai metric `kube_pod_resource_request`/`kube_pod_resource_limit` là **metric ẩn**, chỉ hiện khi bật cờ hidden metrics — cùng lý do như dòng trên |
| Bài [158](../158-logging-vi.md): sidecar chạy **agent ghi log** cụ thể (fluentd) | Image trong ví dụ của bài là bản cũ và cần mạng ra ngoài, cộng thêm chi phí tài nguyên mà chính bài cảnh báo là "đáng kể". Cơ chế mà bài dạy — log không đi qua `stdout` thì `kubectl logs` không thấy — **được kiểm chứng đầy đủ ở B7.2** bằng `busybox:1.37`, không cần agent thật |
| Bài [158](../158-logging-vi.md): **backend log tập trung** và chính sách lưu giữ | Kubernetes không cung cấp backend nào; đó là hệ thống ngoài cluster. B7.5 dựng đúng phần thuộc Kubernetes — agent cấp node dạng `DaemonSet` — còn phần đẩy ra kho tập trung thuộc [giai đoạn 23](../00-ALO-TRINH-ADMIN.md#giai-đoạn-23--giám-sát-và-cảnh-báo) |
| Bài [158](../158-logging-vi.md): feature gate `PodLogsQuerySplitStreams` để tách `Stdout`/`Stderr` | Tính năng alpha, phải bật feature gate trên kube-apiserver — [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) |
| Bài [159](../159-system-logs-vi.md): *Ghi log có cấu trúc*, *Ghi log theo ngữ cảnh*, `--logging-format=json` | Hai mục đầu bài tự xếp vào *Đọc lướt* vì viết cho người phát triển thành phần. Mục JSON đổi định dạng log của thành phần, tức lại là cờ dòng lệnh của control plane — [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy), và chỉ có giá trị khi đã có pipeline phân tích log |
| Bài [159](../159-system-logs-vi.md): **bật** Log Query (`enableSystemLogHandler`) và bảng tùy chọn `boot`/`pattern`/`sinceTime` | Bật nó là sửa cấu hình kubelet trên node đích rồi restart kubelet, và bài kèm cảnh báo an ninh về quyền `nodes/proxy`. Bài tự hoãn tới [giai đoạn 24](../00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố). B9.5 chỉ **đọc** giá trị hiệu lực và kiểm chứng hành vi khớp với giá trị đó |
| Bài [161](../161-system-traces-vi.md): **bật tracing** cho kube-apiserver và cho kubelet | `--tracing-config-file` là cờ của kube-apiserver và đoạn `tracing` là cấu hình kubelet — cả hai thuộc [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy). Bật xong còn cần một OpenTelemetry Collector và một backend tracing, tức thêm hạ tầng ngoài baseline. B10 kiểm chứng đúng phần đọc được |
| Bài [161](../161-system-traces-vi.md): apiserver **không dùng** trace context của request đến; `samplingRatePerMillion`; liên kết cha–con giữa span kubelet và containerd | Cả ba chỉ quan sát được khi tracing đang chạy và có backend nhận span. Không có cách nào viết gate `PASS:` cho chúng trên cluster tắt tracing, nên chúng bị loại theo đúng quy ước của thư mục lab |
| Bài [309](../309-local-debugging-vi.md): chạy `telepresence connect` và `telepresence intercept` | Telepresence là binary bên thứ ba phải cài lên máy trạm, và cơ chế của nó là **cài một sidecar traffic-agent vào Pod ứng dụng trong cluster** — tức nó sửa workload. Lab này nằm trên chuỗi snapshot và phải chụp `04-metrics-ready` sạch, nên không chạy công cụ có tác dụng phụ như vậy. B11 làm đúng cách thủ công mà chính bài 309 dẫn tới |
| Bài [296](../296-debug-vi.md) nhánh *Gỡ lỗi cluster*; bài [297](../297-debug-application-vi.md) phần chẩn đoán Pod chi tiết | Hai nhánh này là nội dung của [giai đoạn 24](../00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố) — `crictl`, `kubectl debug` với ephemeral container, quy trình lần từ Service về Pod. Làm ở đây là lặp lại và nhảy cóc cùng lúc. B11.5 chỉ **vẽ ranh giới** để bạn biết chỗ nào dừng |
| [Bài 342 — HorizontalPodAutoscaler](../342-hpa-walkthrough-vi.md), dù nằm ở dòng thực hành của giai đoạn 11 | Đây là [nợ #1](README.md#5-sổ-nợ-lab), trả ở **Lab 11b** trên chính mốc `04-metrics-ready` mà lab này tạo ra. Làm HPA ở đây là gộp hai nhóm bài vào một lab và đẩy thời lượng vượt ngưỡng |

### 1.2. Thời lượng

3–4 giờ, tính từ lúc gate mở đầu đã PASS. Ba khoảng thời gian đáng kể: chờ metrics-server hỏng đủ
lâu để log nói ra nguyên nhân (B5.3), sinh và chờ xoay vòng khối log lớn ở B8, và bảy tầng gate
cộng thao tác chụp snapshot ở B12. Mọi bước phải chờ đều viết dưới dạng vòng lặp có điều kiện
thoát, không phải con số cố định — chu kỳ thu thập của metrics-server và chu kỳ giám sát xoay vòng
log của kubelet đều **phụ thuộc cấu hình**.

---

## 2. Quy ước và an toàn

- **Chụp snapshot cả ba VM trước khi bắt đầu.** Lab này cài thêm một thành phần vào `kube-system`
  và **không** gỡ nó ra ở cuối. Nếu B5 hỏng giữa chừng, đường quay lui duy nhất là restore
  `03-storage-ready`. Trước khi chạy B0, xác nhận trên máy host rằng cả ba VM đều còn mốc đó:

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

  **PASS:** đúng ba dòng `PASS:`. Không mở B0 khi còn dòng `FAIL:` — bạn sẽ không có đường lui.

- **Cách quay lui khi hỏng:** tắt **cả ba VM**, restore **cả ba** về `03-storage-ready` — không bao
  giờ restore riêng một VM, xem ghi chú cuối
  [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường) — rồi bật lại theo
  thứ tự master → worker 1 → worker 2, chạy lại gate mở đầu và bắt đầu lại từ B0.
- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` bằng user quản trị có kubeconfig, **trừ khi ghi rõ
  node khác**. Lệnh cần `sudo` để đọc file trên node chạy trên chính node đó qua `ssh`.
- **Chạy toàn bộ phần B trong cùng một phiên shell.** Gần như mọi gate so sánh với biến đặt ở B0
  (`NODES`, `W1`, `W2`, `MASTER`, `EV`, `WK`) và ba hàm trợ giúp (`cpu_m`, `mem_mi`, `to_bytes`,
  `cfgz`); mở shell mới giữa chừng là mất hết.
- **Lab này chỉ ĐỌC cấu hình node và cấu hình control plane.** Tuyệt đối không sửa
  `/var/lib/kubelet/config.yaml`, không sửa file nào trong `/etc/kubernetes/manifests`, không
  restart kubelet. B0.4 ghi checksum của bốn file đó và B12.2 đối chiếu lại — đó là gate chứng minh
  bạn không đụng vào. Thứ duy nhất lab thay đổi vĩnh viễn là **metrics-server**, và nó được cài
  bằng object API chứ không bằng cách sửa file trên node.
- **Lab tạo object phạm vi cluster.** Ngoài namespace `lab-11a`, lab tạo một ClusterRole và một
  ClusterRoleBinding ở B3, cộng toàn bộ object của metrics-server ở B5. Ba nhóm này được xử lý
  khác nhau ở B12: hai cái đầu bị xóa, metrics-server **ở lại** vì nó là định nghĩa của mốc mới.
- **Fault injection chỉ trên `lab-k8s-worker2`.** Hai bước có thể làm phiền node — Pod crashloop ở
  B7.3 và khối log 40 000 dòng ở B8 — đều ghim vào `lab-k8s-worker2` bằng `nodeName`.
- Image dùng cho toàn bộ phần workload là `busybox:1.37` đã khóa ở
  [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) và đã có sẵn trên cả ba node từ
  Lab 00. Chỉ đúng **một** image mới được kéo về từ mạng trong lab này: image của metrics-server ở
  B5.
- **Mọi con số trong lab được đọc từ cluster thật** — tên node, `allocatable`, `containerLogMaxSize`
  của kubelet, cổng của kube-scheduler, số file log trên node. Không có con số nào viết cứng trong
  manifest hay trong gate.
- Manifest tạm ghi vào `~/lab-work/11a/`; bằng chứng ghi vào `~/lab-evidence/11a/`. Token sinh ra ở
  B3 chỉ tồn tại trong biến shell, **không ghi vào evidence**.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail.

---

# Phần B — Thực hành kiến thức 11a

## B0. Chuẩn bị workspace, namespace và ảnh chụp "trước"

**Mục đích:** dựng chỗ làm việc, khóa tên node vào biến, định nghĩa các hàm chuẩn hóa mà mọi gate
sau dùng lại, và chụp lại checksum của cấu hình node cùng manifest control plane để B12 chứng minh
lab **không sửa gì** ngoài việc thêm metrics-server.

### B0.1. Workspace, namespace và biến tên node

```bash
mkdir -p ~/lab-work/11a ~/lab-evidence/11a
WK=~/lab-work/11a
EV=~/lab-evidence/11a

kubectl config current-context
kubectl create namespace lab-11a

MASTER='lab-k8s-master'
W1='lab-k8s-worker1'
W2='lab-k8s-worker2'
NODES="$MASTER $W1 $W2"

for n in $NODES; do
  kubectl get node "$n" -o jsonpath='{.metadata.name}{"\t"}{.status.nodeInfo.kubeletVersion}{"\n"}'
done | tee "$EV/b0-nodes.txt"

test "$(wc -l < "$EV/b0-nodes.txt")" -eq 3 \
  && test "$(kubectl get namespace lab-11a -o jsonpath='{.status.phase}')" = 'Active' \
  && echo 'PASS: ba node doc duoc va namespace lab-11a da Active'
```

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B0.2. Ba hàm chuẩn hóa

Kubernetes viết cùng một giá trị theo nhiều dạng (`1` và `1000m`, `1Gi` và `1024Mi`), và
`kubectl top` cũng vậy. Mọi so sánh trong lab đi qua ba hàm dưới đây. Hai hàm đầu giống hệt
[Lab 7b](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md) — giữ nguyên để hai lab đọc số theo cùng một
cách. Định nghĩa chúng **một lần** và giữ tới hết phần B:

```bash
cpu_m()   { case "$1" in ''|'<none>') echo 0 ;; *m) echo "${1%m}" ;; *) echo "$(( $1 * 1000 ))" ;; esac; }
mem_mi()  { case "$1" in ''|'<none>') echo 0 ;; *Ki) echo "$(( ${1%Ki} / 1024 ))" ;; *Mi) echo "${1%Mi}" ;; *Gi) echo "$(( ${1%Gi} * 1024 ))" ;; *) echo 0 ;; esac; }
to_bytes(){ case "$1" in ''|'<none>') echo 0 ;; *Ki) echo "$(( ${1%Ki} * 1024 ))" ;; *Mi) echo "$(( ${1%Mi} * 1048576 ))" ;; *Gi) echo "$(( ${1%Gi} * 1073741824 ))" ;; *[0-9]) echo "$1" ;; *) echo 0 ;; esac; }

test "$(cpu_m 500m)" -eq 500 && test "$(cpu_m 2)" -eq 2000 \
  && test "$(mem_mi 1Gi)" -eq 1024 && test "$(mem_mi 2048Ki)" -eq 2 \
  && test "$(to_bytes 10Mi)" -eq 10485760 && test "$(to_bytes 4096)" -eq 4096 \
  && echo 'PASS: ba ham chuan hoa hoat dong dung'
```

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B0.3. Hàm đọc cấu hình hiệu lực của kubelet

Cấu hình *hiệu lực* của kubelet — gồm cả giá trị mặc định mà file trên đĩa không viết ra — đọc qua
endpoint `configz`, đi đúng đường control plane → kubelet mà
[tầng 6 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a547-tầng-6--đường-control-plane--kubelet) đã kiểm.
Lab dùng nó ở B8, B9 và B10, nên lấy về ngay bây giờ:

```bash
for n in $NODES; do
  kubectl get --raw "/api/v1/nodes/$n/proxy/configz" > "$EV/b0-configz-$n.json" 2>/dev/null || true
  echo "$n -> $(wc -c < "$EV/b0-configz-$n.json") byte"
done

OKZ=0
for n in $NODES; do
  grep -q '"kubeletconfig"' "$EV/b0-configz-$n.json" && OKZ=$(( OKZ + 1 ))
done
echo "configz doc duoc tren $OKZ/3 node"
test "$OKZ" -eq 3 && echo 'PASS: doc duoc cau hinh hieu luc cua ca ba kubelet'
```

Hàm rút một trường vô hướng ra khỏi file JSON đó, không cần `jq` — node lab không cài `jq`:

```bash
cfgz() { grep -o "\"$1\":[^,}]*" "$2" | head -1 | cut -d: -f2- | tr -d '"'; }

test -n "$(cfgz cgroupDriver "$EV/b0-configz-$W2.json")" \
  && echo "PASS: cfgz doc duoc — cgroupDriver = $(cfgz cgroupDriver "$EV/b0-configz-$W2.json")"
```

**PASS:** hai dòng `PASS:` của bước này xuất hiện. Nếu `configz` không đọc được, dừng lại và xem
[mục 4](#4-troubleshooting-của-lab-này) trước khi đi tiếp — B8, B9 và B10 dựa hẳn vào file này.

### B0.4. Checksum của cấu hình node và của manifest control plane

Lab này đọc rất nhiều thứ trên node, và cám dỗ "sửa thử một dòng để xem tracing chạy thế nào" là có
thật. Chụp lại checksum ngay bây giờ để B12.2 biến lời hứa "chỉ đọc" thành thứ kiểm chứng được.

```bash
{
  echo "$MASTER kubelet $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for n in "$W1" "$W2"; do
    echo "$n kubelet $(ssh "$n" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done
  for f in kube-apiserver kube-scheduler kube-controller-manager; do
    echo "$MASTER $f $(sudo sha256sum /etc/kubernetes/manifests/$f.yaml | awk '{print $1}')"
  done
} | tee "$EV/b0-config-sha.txt"

test "$(wc -l < "$EV/b0-config-sha.txt")" -eq 6 \
  && test "$(awk '{print $3}' "$EV/b0-config-sha.txt" | grep -c '^[0-9a-f]\{64\}$')" -eq 6 \
  && echo 'PASS: ghi duoc checksum cua 3 cau hinh kubelet va 3 manifest control plane'
```

Ghi luôn ảnh chụp "trước" của toàn cluster để B12 `diff` lại:

```bash
{
  echo "=== $(date -Is) — trang thai truoc Lab 11a ==="
  echo '--- apiservice'; kubectl get apiservice 2>&1 | grep -i metrics || echo 'khong co'
  echo '--- deployment kube-system'; kubectl -n kube-system get deployment
  echo '--- storageclass'; kubectl get storageclass
  echo '--- pv'; kubectl get pv 2>&1
  echo '--- clusterrole cua lab'; kubectl get clusterrole lab-11a-metrics-reader --ignore-not-found
  echo '--- namespaces'; kubectl get namespaces
} | tee "$EV/b0-truoc.txt"

test -s "$EV/b0-truoc.txt" && echo 'PASS: da ghi anh chup truoc cua cluster'
```

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

---

## B1. Ba trụ cột trên cluster thật

**Mục đích:** dựng khung của bài [162](../162-observability-vi.md) bằng lệnh, để suốt phần còn lại
bạn luôn biết mình đang đứng ở trụ cột nào. Bài chia tín hiệu quan sát thành ba loại — metrics,
logs, traces — và mô tả hình dạng chung của một pipeline: **nguồn phát → bộ thu thập → kho lưu trữ
→ dashboard, cảnh báo, tự động hóa**. Ở đây bạn đi tìm từng khúc đó trên cluster của mình.

### B1.1. Mỗi trụ cột, một câu hỏi, một nguồn

Ba lệnh dưới đây hỏi ba câu khác nhau về **cùng một** đối tượng: CoreDNS.

```bash
DNS_POD="$(kubectl -n kube-system get pods -l k8s-app=kube-dns \
  -o jsonpath='{.items[0].metadata.name}')"
echo "Pod duoc quan sat: $DNS_POD"

# Trụ cột METRIC — "so lan da xay ra", doc tu endpoint /metrics cua mot thanh phan
kubectl get --raw '/metrics' | grep -c '^apiserver_request_total{' \
  | xargs -I{} echo "so chuoi apiserver_request_total tren apiserver = {}"

# Trụ cột LOG — "ung dung da in ra gi", doc qua kubelet
kubectl -n kube-system logs "$DNS_POD" --tail=3 | tee "$EV/b1-log-coredns.txt"

# Trụ cột TRACE — "mot request di qua nhung dau, moi chang mat bao lau"
kubectl -n kube-system get pod "kube-apiserver-$MASTER" \
  -o jsonpath='{range .spec.containers[0].command[*]}{@}{"\n"}{end}' \
  | grep -c -- '--tracing-config-file=' \
  | xargs -I{} echo "so co --tracing-config-file tren kube-apiserver = {}"
```

```bash
M_OK="$(kubectl get --raw '/metrics' | grep -c '^apiserver_request_total{')"
L_OK="$(wc -l < "$EV/b1-log-coredns.txt")"
T_OK="$(kubectl -n kube-system get pod "kube-apiserver-$MASTER" \
  -o jsonpath='{range .spec.containers[0].command[*]}{@}{"\n"}{end}' \
  | grep -c -- '--tracing-config-file=')"
echo "metric=$M_OK log=$L_OK trace-config=$T_OK"

test "$M_OK" -gt 0 && echo 'PASS: nguon metric ton tai san tren cluster'
test "$L_OK" -gt 0 && echo 'PASS: nguon log ton tai san tren cluster'
test "$T_OK" -eq 0 && echo 'PASS: khong co nguon trace — cluster chua bat tracing'
```

**Ý nghĩa:** hai trụ cột đầu **có sẵn từ lúc `kubeadm init`** — không ai phải cài gì để kube-apiserver
phát metric hay để kubelet phục vụ log. Trụ cột thứ ba thì không: tracing phải được bật bằng cấu
hình, và B10 sẽ xác nhận điều đó ở cả hai phía apiserver lẫn kubelet.

**PASS:** đủ ba dòng `PASS:` của bước này.

### B1.2. Khúc còn thiếu của pipeline

Nguồn phát có sẵn, nhưng pipeline của bài 162 còn ba khúc nữa. Đây là khoảng trống mà lab sẽ lấp
một phần:

```bash
echo '--- co bo thu thap metric tai nguyen chua?'
kubectl get apiservice v1beta1.metrics.k8s.io --ignore-not-found -o name | tee "$EV/b1-apiservice.txt"
echo '--- co kho luu tru chuoi thoi gian nao chua?'
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"/"}{.metadata.name}{"\n"}{end}' \
  | grep -icE 'prometheus|grafana|thanos|loki|jaeger' \
  | xargs -I{} echo "so Pod cua cong cu quan sat ben thu ba = {}"

API_N="$(wc -l < "$EV/b1-apiservice.txt")"
TOOL_N="$(kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' \
  | grep -icE 'prometheus|grafana|thanos|loki|jaeger' || true)"
echo "apiservice metrics=$API_N | pod cong cu ben thu ba=$TOOL_N"

test "$API_N" -eq 0 && test "$TOOL_N" -eq 0 \
  && echo 'PASS: cluster chua co bo thu thap lan kho luu tru — dung moc 03-storage-ready'
kubectl top node >/dev/null 2>&1 \
  && echo 'FAIL: kubectl top da chay duoc truoc khi lab cai gi' \
  || echo 'PASS: kubectl top chua chay duoc — dung nhu du kien'
```

**Ý nghĩa:** đây là bản đồ của phần còn lại. **B5 lấp khúc "bộ thu thập"** cho metric tài nguyên
bằng metrics-server. **Khúc "kho lưu trữ chuỗi thời gian"** thì lab cố ý không lấp — lý do đầy đủ ở
[bảng mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành). Với trụ cột log, **B7.5 dựng khúc "bộ thu
thập"** dưới dạng agent cấp node, còn kho log tập trung cũng để lại cho vận hành thật.

**PASS:** hai dòng `PASS:` của bước này xuất hiện, không dòng `FAIL:` nào.

### B1.3. Bốn nhánh của trang mục lục 296

Bài [296](../296-debug-vi.md) chia tài liệu gỡ lỗi thành bốn nhánh. Hai nhánh thuộc giai đoạn 11,
hai nhánh thuộc giai đoạn 24. Ghi lại ranh giới đó ngay bây giờ, vì B11.5 sẽ dùng lại:

```bash
{
  echo 'Nhanh cua bai 296 — lam o dau'
  echo '1. Go loi ung dung cua ban (297) : phan logs/exec lam o B7 va B11; phan chan doan Pod -> giai doan 24'
  echo '2. Go loi cluster cua ban (305)  : giai doan 24, KHONG lam trong lab 11a'
  echo '3. Ghi log trong Kubernetes (316): lam o B7, B8, B9'
  echo '4. Giam sat trong Kubernetes (317): lam o B2, B3, B5, B6 va B10'
} | tee "$EV/b1-ranh-gioi-296.txt"

test "$(wc -l < "$EV/b1-ranh-gioi-296.txt")" -eq 5 \
  && echo 'PASS: da ghi ranh gioi bon nhanh cua bai 296'
```

**PASS:** dòng `PASS:` của bước này xuất hiện.

---

## B2. Endpoint `/metrics` và định dạng Prometheus

**Mục đích:** thực hành đúng câu mở đầu của bài [160](../160-system-metrics-vi.md) — "trong hầu hết
các trường hợp, metric khả dụng tại endpoint `/metrics` của HTTP server" — trên chính các thành
phần đang chạy, và đọc được định dạng văn bản mà chúng phát ra.

### B2.1. `/metrics` của kube-apiserver

`kubectl get --raw` gửi thẳng một đường dẫn tới API server bằng kubeconfig hiện tại, nên đây là
cách gọi endpoint `/metrics` mà không cần thêm công cụ nào:

```bash
kubectl get --raw '/metrics' > "$EV/b2-apiserver-metrics.txt"
wc -l "$EV/b2-apiserver-metrics.txt"

HELP_N="$(grep -c '^# HELP ' "$EV/b2-apiserver-metrics.txt")"
TYPE_N="$(grep -c '^# TYPE ' "$EV/b2-apiserver-metrics.txt")"
echo "dong HELP=$HELP_N | dong TYPE=$TYPE_N"

test "$HELP_N" -gt 100 && test "$TYPE_N" -gt 100 \
  && grep -q '^apiserver_request_total{' "$EV/b2-apiserver-metrics.txt" \
  && echo 'PASS: apiserver phat metric o /metrics theo dinh dang Prometheus'
```

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B2.2. Đọc một metric: ba dòng, ba vai trò

Định dạng Prometheus là văn bản thuần có cấu trúc, "được thiết kế để cả con người lẫn máy đều có
thể đọc được". Ba dòng dưới đây là toàn bộ hợp đồng của một metric:

```bash
grep -E '^# (HELP|TYPE) apiserver_request_total ' "$EV/b2-apiserver-metrics.txt"
grep -m 2 '^apiserver_request_total{' "$EV/b2-apiserver-metrics.txt"
```

Bài 160 dành hẳn mục *Vòng đời metric* cho chuỗi alpha → beta → stable → deprecated → hidden →
deleted. Trên các thành phần Kubernetes, **mức ổn định được in ngay trong dòng `HELP`**, nên bạn
không phải tra bảng ở đâu khác:

```bash
ST_N="$(grep -c '^# HELP .*\[STABLE\]' "$EV/b2-apiserver-metrics.txt")"
AL_N="$(grep -c '^# HELP .*\[ALPHA\]' "$EV/b2-apiserver-metrics.txt")"
BE_N="$(grep -c '^# HELP .*\[BETA\]' "$EV/b2-apiserver-metrics.txt" || true)"
echo "STABLE=$ST_N | BETA=$BE_N | ALPHA=$AL_N"

{
  echo '--- vai vi du STABLE'; grep -m 3 '^# HELP .*\[STABLE\]' "$EV/b2-apiserver-metrics.txt"
  echo '--- vai vi du ALPHA';  grep -m 3 '^# HELP .*\[ALPHA\]'  "$EV/b2-apiserver-metrics.txt"
} | tee "$EV/b2-muc-on-dinh.txt"

test "$ST_N" -gt 0 && test "$AL_N" -gt 0 \
  && echo 'PASS: doc duoc muc on dinh cua metric ngay trong dong HELP'
```

**Ý nghĩa:** đây là thứ quyết định dashboard của bạn sống được bao lâu. Panel dựng trên metric
`[STABLE]` được bảo đảm không đổi tên và không đổi kiểu; panel dựng trên `[ALPHA]` **không có bảo
đảm nào** và có thể bị sửa hoặc xóa bất cứ lúc nào.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B2.3. Bốn endpoint của kubelet

Bài 160 nói kubelet expose thêm `/metrics/cadvisor`, `/metrics/resource` và `/metrics/probes`, và
nhấn mạnh **"các metric đó không có cùng vòng đời"**. Gọi cả bốn qua đường proxy tới node — đây
cũng chính là đường mà một scraper bên ngoài sẽ đi:

```bash
for ep in metrics metrics/cadvisor metrics/resource metrics/probes; do
  f="$EV/b2-kubelet-$(echo "$ep" | tr '/' '-')-$MASTER.txt"
  kubectl get --raw "/api/v1/nodes/$MASTER/proxy/$ep" > "$f" 2>/dev/null || true
  echo "$ep -> $(wc -l < "$f") dong"
done
```

```bash
F_BASE="$EV/b2-kubelet-metrics-$MASTER.txt"
F_CAD="$EV/b2-kubelet-metrics-cadvisor-$MASTER.txt"
F_RES="$EV/b2-kubelet-metrics-resource-$MASTER.txt"
F_PRB="$EV/b2-kubelet-metrics-probes-$MASTER.txt"

grep -q '^kubelet_node_name{' "$F_BASE" \
  && echo 'PASS: /metrics — metric ve chinh kubelet'
grep -q '^container_memory_working_set_bytes{' "$F_CAD" \
  && echo 'PASS: /metrics/cadvisor — metric chi tiet tung container'
grep -q '^container_cpu_usage_seconds_total{' "$F_RES" \
  && echo 'PASS: /metrics/resource — bo metric tai nguyen gon, dung cho pipeline'
grep -q '^prober_probe_total{' "$F_PRB" \
  && echo 'PASS: /metrics/probes — ket qua liveness/readiness probe'

RES_N="$(grep -c '^# TYPE ' "$F_RES")"
CAD_N="$(grep -c '^# TYPE ' "$F_CAD")"
echo "so ho metric: /metrics/resource=$RES_N | /metrics/cadvisor=$CAD_N"
test "$RES_N" -lt "$CAD_N" \
  && echo 'PASS: /metrics/resource nho hon han /metrics/cadvisor'
```

**Ý nghĩa:** con số cuối là chỗ B5 nối vào. `/metrics/cadvisor` là kho chi tiết cho một hệ thống
giám sát đầy đủ; `/metrics/resource` là **bộ tối thiểu về CPU và bộ nhớ**, đủ cho một bộ tổng hợp
nhẹ. metrics-server dùng đúng endpoint nhỏ này, và đó là lý do nó chạy được trên ba VM lab trong
khi một stack giám sát đầy đủ thì không.

**PASS:** đủ năm dòng `PASS:` của bước này. Nếu `/metrics/probes` rỗng, xem
[mục 4](#4-troubleshooting-của-lab-này).

### B2.4. Chữ ký deprecated

Bài 160 mô tả cách một metric bị loại bỏ dần tự khai điều đó ngay trong dòng `HELP`, kèm phiên bản
bắt đầu. Đếm trên cluster của bạn và ghi lại — con số này sẽ khác nhau giữa các bản phát hành:

```bash
{
  for f in "$EV/b2-apiserver-metrics.txt" "$F_BASE" "$F_CAD" "$F_RES" "$F_PRB"; do
    echo "$f : $(grep -c '(Deprecated since' "$f" || true) metric mang chu ky deprecated"
    grep -m 2 '(Deprecated since' "$f" || true
  done
} | tee "$EV/b2-deprecated.txt"

test "$(wc -l < "$EV/b2-deprecated.txt")" -ge 5 \
  && echo 'PASS: da ra soat chu ky deprecated tren ca nam endpoint'
```

**Ý nghĩa:** đây là bước rà soát bạn phải làm **trước** mỗi lần nâng cấp minor, không phải sau. Một
metric `[STABLE]` mang chữ ký deprecated còn ít nhất ba bản phát hành trước khi bị ẩn; một metric
`[BETA]` chỉ còn một. Sau khi bị ẩn, `--show-hidden-metrics-for-version` mua thêm được **đúng một
chu kỳ** vì nó chỉ nhận phiên bản minor liền trước.

**PASS:** dòng `PASS:` của bước này xuất hiện.

---

## B3. Đọc metric là một hành động được ủy quyền

**Mục đích:** kiểm chứng câu mà bài [160](../160-system-metrics-vi.md) đặt ngay sau danh sách
endpoint: *"Nếu cluster của bạn dùng RBAC, việc đọc metric yêu cầu được ủy quyền qua một user,
group hoặc ServiceAccount với một ClusterRole cho phép truy cập `/metrics`"*. Bài kèm hẳn một
manifest ví dụ; ở đây bạn dựng đúng manifest đó và chứng minh cả hai chiều — không có nó thì bị từ
chối, có nó thì đi qua.

RBAC là nội dung [giai đoạn 9](../00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), tức
là kiến thức **đã học** khi bạn tới đây.

### B3.1. Chọn mục tiêu: kube-scheduler và cờ `--bind-address`

Không dùng kube-apiserver làm mục tiêu: kubeconfig quản trị của bạn có quyền `*` trên mọi thứ nên
sẽ không thấy được ranh giới. kube-scheduler là mục tiêu tốt hơn, và nó còn minh họa luôn câu
"với các thành phần không expose endpoint này theo mặc định, có thể bật bằng cờ `--bind-address`".

Đọc địa chỉ và cổng thật từ chính Pod đang chạy, **không đoán**:

```bash
SCHED_POD="kube-scheduler-$MASTER"
kubectl -n kube-system get pod "$SCHED_POD" \
  -o jsonpath='{range .spec.containers[0].command[*]}{@}{"\n"}{end}' \
  | tee "$EV/b3-scheduler-command.txt"

SCHED_BIND="$(grep -m1 '^--bind-address=' "$EV/b3-scheduler-command.txt" | cut -d= -f2)"
SCHED_PORT="$(kubectl -n kube-system get pod "$SCHED_POD" \
  -o jsonpath='{.spec.containers[0].livenessProbe.httpGet.port}')"
echo "kube-scheduler bind-address=${SCHED_BIND:-<khong khai>} port=${SCHED_PORT:-<khong doc duoc>}"

test -n "$SCHED_PORT" && echo 'PASS: doc duoc cong HTTPS that cua kube-scheduler'
```

**Ý nghĩa:** nếu `bind-address` là `127.0.0.1` — giá trị kubeadm đặt mặc định — thì endpoint
`/metrics` của scheduler **chỉ gọi được từ chính node master**. Đó là lý do phần còn lại của B3
chạy `curl` trên `lab-k8s-master` chứ không qua `kubectl`, và cũng là lý do một scraper thật thường
phải chạy dưới dạng Pod trên node đó hoặc phải đổi cờ này. Đổi cờ là việc của
[giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy); lab này chỉ
đọc.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B3.2. Một ServiceAccount trần bị từ chối

```bash
kubectl -n lab-11a create serviceaccount scraper
kubectl auth can-i get /metrics --as=system:serviceaccount:lab-11a:scraper \
  | tee "$EV/b3-can-i-truoc.txt"

TOK="$(kubectl -n lab-11a create token scraper)"
CODE1="$(curl -sk -o /dev/null -w '%{http_code}' \
  -H "Authorization: Bearer $TOK" \
  "https://${SCHED_BIND:-127.0.0.1}:$SCHED_PORT/metrics")"
echo "HTTP code khi chua uy quyen = $CODE1"

grep -qx 'no' "$EV/b3-can-i-truoc.txt" \
  && echo 'PASS: can-i tra "no" — RBAC chua cho phep'
test "$CODE1" != '200' \
  && echo "PASS: kube-scheduler tu choi that su — HTTP $CODE1 (thuong la 403)"
```

**Ý nghĩa:** token đã được **xác thực** — scheduler biết đây là ServiceAccount nào — nhưng bị chặn
ở bước **ủy quyền**. Hai bước đó khác nhau, và đây là chỗ dễ chẩn đoán nhầm nhất khi dựng một
scraper: mạng thông, token hợp lệ, mà vẫn không lấy được metric.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B3.3. Đúng một ClusterRole, đúng một đường

Manifest dưới đây là ví dụ trong bài 160, đổi tên cho khớp quy ước của lab:

```bash
cat > "$WK/b3-rbac.yaml" <<'YAML'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: lab-11a-metrics-reader
rules:
  - nonResourceURLs:
      - "/metrics"
    verbs:
      - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: lab-11a-metrics-reader
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: lab-11a-metrics-reader
subjects:
  - kind: ServiceAccount
    name: scraper
    namespace: lab-11a
YAML

kubectl apply -f "$WK/b3-rbac.yaml"
kubectl auth can-i get /metrics --as=system:serviceaccount:lab-11a:scraper \
  | tee "$EV/b3-can-i-sau.txt"

TOK="$(kubectl -n lab-11a create token scraper)"
CODE2="$(curl -sk -o /dev/null -w '%{http_code}' \
  -H "Authorization: Bearer $TOK" \
  "https://${SCHED_BIND:-127.0.0.1}:$SCHED_PORT/metrics")"
echo "HTTP code sau khi uy quyen = $CODE2"

grep -qx 'yes' "$EV/b3-can-i-sau.txt" && echo 'PASS: can-i tra "yes"'
test "$CODE2" = '200' && echo 'PASS: scraper doc duoc /metrics cua kube-scheduler'
```

Lấy về một ít nội dung để chắc chắn đây là metric thật, không phải trang lỗi:

```bash
curl -sk -H "Authorization: Bearer $(kubectl -n lab-11a create token scraper)" \
  "https://${SCHED_BIND:-127.0.0.1}:$SCHED_PORT/metrics" \
  | grep -m 5 -E '^# (HELP|TYPE) scheduler_' | tee "$EV/b3-scheduler-metrics.txt"

test -s "$EV/b3-scheduler-metrics.txt" \
  && echo 'PASS: lay ve duoc metric rieng cua scheduler'
```

### B3.4. Cánh cửa mở đúng bằng một đường dẫn

ClusterRole vừa tạo **chỉ** cấp `get` cho đúng chuỗi `/metrics`. Nó không cấp gì khác:

```bash
for url in /metrics /metrics/resource /metrics/cadvisor /debug/pprof; do
  printf '%-22s ' "$url"
  kubectl auth can-i get "$url" --as=system:serviceaccount:lab-11a:scraper
done | tee "$EV/b3-pham-vi.txt"

test "$(grep -c ' yes$' "$EV/b3-pham-vi.txt")" -eq 1 \
  && echo 'PASS: dung mot duong dan duoc mo, ba duong con lai van bi tu choi'
```

> Bốn đường dẫn trên được chọn có chủ đích: cả bốn đều **không** nằm trong ClusterRole
> `system:public-info-viewer` mà mọi danh tính đã xác thực đều được gán sẵn. Nếu bạn thử với
> `/healthz`, `/livez`, `/readyz` hay `/version` thì câu trả lời là `yes` ngay cả khi chưa tạo
> ClusterRole nào — và bạn sẽ kết luận sai về phạm vi.

**Ý nghĩa:** `nonResourceURLs` khớp **chuỗi đường dẫn**, không khớp tiền tố. Muốn scraper đọc thêm
`/metrics/resource` hay `/metrics/cadvisor` của kubelet thì phải liệt kê thêm — hoặc dùng quyền
`nodes/metrics` ở nhánh tài nguyên. Đây chính là lý do manifest RBAC của metrics-server ở B5 dài
hơn ví dụ bốn dòng trong bài 160.

**PASS:** dòng `PASS:` của bước này xuất hiện.

---

## B4. Metric tài nguyên so với metric trạng thái đối tượng

**Mục đích:** dựng bằng lệnh cái ranh giới mà bài [163](../163-kube-state-metrics-vi.md) tồn tại để
vẽ. Lab **không cài** kube-state-metrics (lý do ở [bảng mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành)),
nên cách kiểm chứng ở đây là chứng minh **khoảng trống** — thứ mà add-on đó sinh ra thì trên cluster
này hoàn toàn không có, dù mọi nguồn metric khác đều đang chạy.

### B4.1. Một Deployment để có gì đó mà quan sát

```bash
cat > "$WK/b4-web.yaml" <<'YAML'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: b4-web
  namespace: lab-11a
spec:
  replicas: 2
  selector:
    matchLabels:
      app: b4-web
  template:
    metadata:
      labels:
        app: b4-web
    spec:
      containers:
        - name: web
          image: busybox:1.37
          command:
            - sh
            - -c
            - 'mkdir -p /www && echo b4-web-ok > /www/index.html && httpd -f -p 8080 -h /www'
YAML

kubectl apply -f "$WK/b4-web.yaml"
kubectl -n lab-11a rollout status deploy/b4-web --timeout=180s
kubectl -n lab-11a get pods -l app=b4-web -o wide

AVAIL="$(kubectl -n lab-11a get deploy b4-web -o jsonpath='{.status.availableReplicas}')"
echo "b4-web availableReplicas doc tu API = ${AVAIL:-0}"
test "${AVAIL:-0}" -eq 2 && echo 'PASS: Deployment b4-web co du hai replica san sang'
```

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B4.2. Metric tài nguyên **có** — đo được từ node

```bash
POD1="$(kubectl -n lab-11a get pods -l app=b4-web \
  -o jsonpath='{.items[0].metadata.name}')"
NODE1="$(kubectl -n lab-11a get pod "$POD1" -o jsonpath='{.spec.nodeName}')"
echo "quan sat Pod $POD1 tren node $NODE1"

kubectl get --raw "/api/v1/nodes/$NODE1/proxy/metrics/resource" \
  > "$EV/b4-resource-$NODE1.txt"

grep -m 2 "^container_cpu_usage_seconds_total{.*pod=\"$POD1\"" "$EV/b4-resource-$NODE1.txt" \
  | tee "$EV/b4-cpu-cua-pod.txt"
grep -m 2 "^container_memory_working_set_bytes{.*pod=\"$POD1\"" "$EV/b4-resource-$NODE1.txt" \
  | tee -a "$EV/b4-cpu-cua-pod.txt"

test -s "$EV/b4-cpu-cua-pod.txt" \
  && echo 'PASS: kubelet cua node do da phat metric TAI NGUYEN cho dung Pod nay'
```

**Ý nghĩa:** kubelet chỉ biết những gì chạy **trên node của nó**, và thứ nó đo được là **mức tiêu
thụ**. Nó không biết Pod này thuộc Deployment nào, cũng không biết Deployment đó mong muốn mấy
replica — những thứ đó nằm trong object ở API server, không nằm trên node.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B4.3. Metric trạng thái đối tượng **không có**

Gom mọi nguồn metric mà cluster đang phát rồi tìm đúng những tên metric mà bài 163 nêu:

```bash
cat "$EV/b2-apiserver-metrics.txt" \
    "$EV/b2-kubelet-metrics-$MASTER.txt" \
    "$EV/b2-kubelet-metrics-cadvisor-$MASTER.txt" \
    "$EV/b2-kubelet-metrics-resource-$MASTER.txt" \
    "$EV/b2-kubelet-metrics-probes-$MASTER.txt" \
    "$EV/b4-resource-$NODE1.txt" > "$EV/b4-moi-nguon-metric.txt"
wc -l "$EV/b4-moi-nguon-metric.txt"

for m in kube_pod_container_info kube_pod_status_ready kube_pod_deletion_timestamp \
         kube_deployment_status_replicas_available; do
  printf '%-42s %s\n' "$m" "$(grep -c "^$m" "$EV/b4-moi-nguon-metric.txt" || true)"
done | tee "$EV/b4-thieu-kube-state.txt"

KS_N="$(awk '{s+=$2} END {print s+0}' "$EV/b4-thieu-kube-state.txt")"
echo "tong so chuoi kube_* tim thay tren toan bo nguon = $KS_N"

test "$KS_N" -eq 0 \
  && echo 'PASS: khong nguon nao phat metric TRANG THAI DOI TUONG'
```

Nhưng chính thông tin đó lại **có sẵn** ở dạng object, chỉ là không ở dạng metric:

```bash
kubectl -n lab-11a get deploy b4-web \
  -o custom-columns='NAME:.metadata.name,DESIRED:.spec.replicas,AVAILABLE:.status.availableReplicas' \
  | tee "$EV/b4-trang-thai-qua-api.txt"

test "$(grep -c '^b4-web' "$EV/b4-trang-thai-qua-api.txt")" -eq 1 \
  && echo 'PASS: trang thai object doc duoc qua API — nhung phai kubectl get roi doc bang mat'
```

**Ý nghĩa:** đây là toàn bộ lý do kube-state-metrics tồn tại. Nó **kết nối tới API server**, đọc
trạng thái từng object, rồi **expose một endpoint HTTP** để biến trạng thái đó thành thứ truy vấn
và cảnh báo được — thay vì phải `kubectl get` rồi nhìn bằng mắt. Hệ quả kèm theo: nguồn duy nhất
của nó là API server, nên mất kết nối tới đó là metric của nó **không còn phản ánh thực tế**; nó
không quan sát node hay container một cách độc lập.

Câu hỏi phân biệt cần thuộc: *"Pod này ngốn CPU"* là metric **tài nguyên** — kubelet trả lời.
*"Pod này kẹt `Terminating` mười phút"* là metric **trạng thái đối tượng** — chỉ kube-state-metrics
trả lời được.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B4.4. "Thông tin nằm ở label, không ở giá trị"

Bài 163 mô tả `kube_pod_container_info` là một metric mà giá trị số gần như vô nghĩa, còn toàn bộ
thông tin nằm ở label. Cluster này không có metric đó, nhưng có một metric **cùng kiểu** để bạn đọc
đúng quy tắc:

```bash
grep -m 1 '^kubelet_node_name{' "$EV/b2-kubelet-metrics-$MASTER.txt" | tee "$EV/b4-info-metric.txt"
grep -m 1 '^kubernetes_build_info{' "$EV/b2-apiserver-metrics.txt" | tee -a "$EV/b4-info-metric.txt"

grep -q "^kubelet_node_name{node=\"$MASTER\"}" "$EV/b4-info-metric.txt" \
  && echo 'PASS: metric kieu info — ten node nam o label, gia tri chi la 1'
```

**Ý nghĩa:** một metric kiểu *info* luôn có giá trị `1`; nó tồn tại để **các metric khác nhóm và
lọc theo label của nó**. Nắm quy tắc này thì lúc gặp `kube_pod_container_info` bạn đã biết phải đọc
nó thế nào rồi.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B4.5. Dọn phần workload của B4

```bash
kubectl -n lab-11a delete -f "$WK/b4-web.yaml" --wait=true --timeout=180s
kubectl -n lab-11a get pods -l app=b4-web

test "$(kubectl -n lab-11a get pods -l app=b4-web --no-headers 2>/dev/null | wc -l)" -eq 0 \
  && echo 'PASS: workload cua B4 da bien mat'
```

**PASS:** dòng `PASS:` của bước này xuất hiện.

---

## B5. Cài metrics-server và chữa lỗi certificate

**Mục đích:** đây là bước đổi hạ tầng của lab và là lý do nó tạo mốc mới. B2 đã cho thấy kubelet
phát sẵn `/metrics/resource`, nhưng **không ai đọc và tổng hợp nó**, nên `kubectl top` chưa có gì
để hiển thị. metrics-server lấp đúng khúc "bộ thu thập" đó, và nó là **điều kiện đầu vào của
Lab 11b**.

Bước này cố ý đi qua một lần hỏng. Trên cluster kubeadm, metrics-server gần như luôn **không
`Ready` ở lần cài đầu tiên**. Đừng đưa sẵn cờ chữa lỗi vào rồi đi tiếp — chẩn đoán được nguyên
nhân từ log của chính nó là phần học thật của B5.

### B5.1. Chốt version trước khi tải

Version của metrics-server **không nằm trong file này và không được chép vào đây**. Nó thuộc
[bảng A1.4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) — bảng giữ
con số của các thành phần ngoài Lab 00 ở đúng một chỗ.

> **Nếu bảng A1.4 chưa có dòng `metrics-server`**, dừng lại và bổ sung dòng đó **trước**, đúng như
> Lab 6a đã làm với `local-path-provisioner`. Cách chốt: mở bảng tương thích trong
> [README của metrics-server](https://github.com/kubernetes-sigs/metrics-server#compatibility-matrix),
> tìm dòng có cột *Kubernetes* bao trùm minor của
> [A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa), lấy bản vá mới nhất của dòng đó, ghi
> vào A1.4 kèm ghi chú "Lab 11a (metrics-server)". Sửa A1.4 rồi mới quay lại đây; **đừng** viết
> con số vào lab này.

Mở A1.4, đọc dòng `metrics-server`, rồi điền vào biến dưới đây:

```bash
# Điền giá trị đọc được từ bảng A1.4 của Lab 00, dòng metrics-server.
MS_VERSION='<điền version từ A1.4>'

case "$MS_VERSION" in
  ''|*'<'*|*'>'*|*'điền'*)
    echo 'FAIL: chưa điền MS_VERSION — lấy version ở bảng A1.4 của Lab 00 rồi chạy lại' ;;
  v[0-9]*.[0-9]*.[0-9]*)
    echo "PASS: MS_VERSION = $MS_VERSION" ;;
  *)
    echo "FAIL: MS_VERSION = $MS_VERSION không có dạng vX.Y.Z như A1.4 ghi" ;;
esac
```

**PASS:** dòng `PASS: MS_VERSION = …` xuất hiện. Không đi tiếp khi còn thấy `FAIL:` — mọi lệnh phía
dưới đều dùng lại biến này, và một placeholder chưa điền sẽ kéo theo cả chuỗi lỗi khó đọc.

### B5.2. Tải manifest và đọc trước khi apply

```bash
MS_URL="https://github.com/kubernetes-sigs/metrics-server/releases/download/${MS_VERSION}/components.yaml"
curl -fsSL -o "$WK/metrics-server.yaml" "$MS_URL"

grep -nE '^kind:|^  name:|^  namespace:|image:|^        - --' "$WK/metrics-server.yaml" \
  | tee "$EV/b5-manifest.txt"
```

Gate bốn thứ **trước khi** apply — không apply một manifest chưa đối chiếu:

```bash
MS_FILE="$WK/metrics-server.yaml"

grep -q "image: registry.k8s.io/metrics-server/metrics-server:${MS_VERSION}" "$MS_FILE" \
  && echo "PASS: image dung version ${MS_VERSION} da khoa o A1.4"
grep -q 'name: v1beta1.metrics.k8s.io' "$MS_FILE" \
  && echo 'PASS: manifest dang ky APIService v1beta1.metrics.k8s.io'
grep -q -- '--kubelet-preferred-address-types=' "$MS_FILE" \
  && echo 'PASS: co co chon kieu dia chi khi goi kubelet'
grep -q -- '--kubelet-insecure-tls' "$MS_FILE" \
  && echo 'FAIL: manifest thuong nguon da tat kiem tra certificate — doc lai B5.4 truoc khi apply' \
  || echo 'PASS: manifest thuong nguon KHONG tat kiem tra certificate'
```

**Ý nghĩa:** dòng gate cuối là mấu chốt của cả B5. Manifest thượng nguồn **kiểm tra certificate của
kubelet** như mọi client TLS đàng hoàng khác. Điều đó đúng về nguyên tắc, và nó sắp va vào một sự
thật của cluster kubeadm ở B5.3.

Manifest còn tạo: một ServiceAccount, hai ClusterRole cùng ClusterRoleBinding tương ứng, một
RoleBinding vào `kube-system`, một Service, một Deployment và một APIService — tất cả trong
namespace `kube-system`. Để ý phần RBAC của nó: khác với ClusterRole bốn dòng bạn viết ở B3, nó cần
thêm quyền trên `nodes/metrics`, đúng như B3.4 đã dự báo.

**PASS:** đủ bốn dòng `PASS:`, không dòng `FAIL:` nào.

### B5.3. Apply và quan sát nó **không** Ready

```bash
kubectl apply -f "$MS_FILE"
kubectl -n kube-system get deployment metrics-server -o wide

kubectl -n kube-system rollout status deploy/metrics-server --timeout=180s \
  || echo 'du kien: rollout khong hoan tat trong thoi gian cho'
```

```bash
READY="$(kubectl -n kube-system get deploy metrics-server -o jsonpath='{.status.readyReplicas}')"
echo "metrics-server readyReplicas = ${READY:-0}"
test "${READY:-0}" -eq 0 \
  && echo 'PASS: metrics-server chua Ready — dung nhu du kien, di tiep de tim nguyen nhan'
```

> Nếu `readyReplicas` đã là `1` ngay lần đầu, cluster của bạn đã có certificate kubelet do CA của
> cluster ký. Đó là trạng thái tốt hơn baseline; bỏ qua phần vá ở B5.5, ghi lý do vào evidence và
> đi thẳng tới B6 — nhưng vẫn đọc B5.4 vì nó giải thích chuyện gì đã không xảy ra.

Nguyên nhân nằm trong log của chính nó:

```bash
kubectl -n kube-system logs deploy/metrics-server --tail=40 | tee "$EV/b5-log-hong.txt"

grep -qiE 'x509|failed to verify certificate' "$EV/b5-log-hong.txt" \
  && echo 'PASS: log chi dung nguyen nhan la certificate cua kubelet'
grep -m 2 -iE 'x509|unable to fully scrape metrics' "$EV/b5-log-hong.txt"
```

Hệ quả lan ra tới tận API tổng hợp và tới `kubectl top`:

```bash
kubectl get apiservice v1beta1.metrics.k8s.io \
  -o custom-columns='NAME:.metadata.name,AVAILABLE:.status.conditions[?(@.type=="Available")].status,REASON:.status.conditions[?(@.type=="Available")].reason' \
  | tee "$EV/b5-apiservice-hong.txt"

AV="$(kubectl get apiservice v1beta1.metrics.k8s.io \
  -o jsonpath='{.status.conditions[?(@.type=="Available")].status}')"
echo "APIService Available = $AV"
test "$AV" = 'False' && echo 'PASS: APIService chua Available'

kubectl top node >/dev/null 2>&1 \
  && echo 'FAIL: kubectl top chay duoc trong khi metrics-server chua Ready' \
  || echo 'PASS: kubectl top van chua chay duoc'
```

**Ý nghĩa:** ba triệu chứng — Pod không `Ready`, APIService `Available=False`, `kubectl top` báo
lỗi — đều là **cùng một** nguyên nhân. Chuỗi phụ thuộc đi theo đúng chiều đó, nên khi gặp lại tình
huống này ở cluster thật, đọc log của metrics-server trước, đừng bắt đầu từ `kubectl top`.

**PASS:** đủ bốn dòng `PASS:` của bước này, không dòng `FAIL:` nào.

### B5.4. Vì sao hỏng, và hai đường sửa

kubeadm cấp cho kubelet của mỗi node một certificate **tự ký** để phục vụ endpoint HTTPS trên cổng
`10250`. Certificate đó không do CA của cluster ký và thường không có IP SAN. metrics-server gọi
kubelet qua địa chỉ IP, kiểm tra certificate theo đúng chuẩn, và từ chối kết nối. Đó chính là dòng
`x509` bạn vừa đọc.

Có **hai** đường sửa, và bạn phải biết cả hai:

| Đường | Làm gì | Đánh đổi | Thuộc giai đoạn nào |
| --- | --- | --- | --- |
| **A — bỏ kiểm tra certificate** | Thêm cờ `--kubelet-insecure-tls` cho metrics-server | Kết nối vẫn được mã hóa nhưng **không xác thực được danh tính kubelet**; một bên đứng giữa giả làm kubelet sẽ không bị phát hiện. Chấp nhận được trên mạng lab cô lập, **không** chấp nhận được trên production | Làm ngay ở đây |
| **B — cấp certificate hợp lệ cho kubelet** | Bật `serverTLSBootstrap` trong cấu hình kubelet của **cả ba node**, restart kubelet, rồi duyệt CSR mà kubelet gửi lên | Đúng đắn về bảo mật, nhưng phải sửa `/var/lib/kubelet/config.yaml` trên từng node và restart kubelet — hai việc lab này **cấm**, và B12.2 kiểm bằng checksum | [Giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy), bài [224](../224-kubelet-config-file-vi.md) |

Lab đi đường A và **ghi rõ đó là đánh đổi có ý thức**, không phải mặc định đúng.

```bash
{
  echo "=== $(date -Is) — quyet dinh cua B5.4 ==="
  echo 'Nguyen nhan : certificate phuc vu cua kubelet la tu ky, khong do CA cluster ky'
  echo 'Duong chon  : A — them --kubelet-insecure-tls cho metrics-server'
  echo 'Danh doi    : khong xac thuc duoc danh tinh kubelet; chi chap nhan tren mang lab co lap'
  echo 'Duong dung  : B — serverTLSBootstrap + duyet CSR, thuoc giai doan 20'
} | tee "$EV/b5-quyet-dinh.txt"

test "$(wc -l < "$EV/b5-quyet-dinh.txt")" -eq 5 \
  && echo 'PASS: da ghi lai quyet dinh va danh doi'
```

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B5.5. Vá và chờ nó lên

```bash
kubectl -n kube-system patch deployment metrics-server --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

kubectl -n kube-system rollout status deploy/metrics-server --timeout=300s
kubectl -n kube-system get pods -l k8s-app=metrics-server -o wide
```

```bash
kubectl -n kube-system get deploy metrics-server \
  -o jsonpath='{range .spec.template.spec.containers[0].args[*]}{@}{"\n"}{end}' \
  | tee "$EV/b5-args-sau.txt"

READY="$(kubectl -n kube-system get deploy metrics-server -o jsonpath='{.status.readyReplicas}')"
IMG="$(kubectl -n kube-system get deploy metrics-server \
  -o jsonpath='{.spec.template.spec.containers[0].image}')"
FLAG_N="$(grep -cx -- '--kubelet-insecure-tls' "$EV/b5-args-sau.txt")"
echo "readyReplicas=${READY:-0} | image=$IMG | so lan co xuat hien=$FLAG_N"

test "$FLAG_N" -eq 1 \
  && echo 'PASS: co da duoc them dung mot lan vao args'
test "${READY:-0}" -ge 1 \
  && test "$IMG" = "registry.k8s.io/metrics-server/metrics-server:${MS_VERSION}" \
  && echo 'PASS: metrics-server dang chay dung version da khoa'
```

Chờ APIService chuyển sang `Available` — thời gian phụ thuộc chu kỳ thu thập của metrics-server,
nên viết dưới dạng vòng lặp có điều kiện thoát chứ không phải một con số:

```bash
for i in $(seq 1 30); do
  AV="$(kubectl get apiservice v1beta1.metrics.k8s.io \
    -o jsonpath='{.status.conditions[?(@.type=="Available")].status}')"
  test "$AV" = 'True' && break
  sleep 10
done
echo "APIService Available = $AV sau $i vong cho"

test "$AV" = 'True' && echo 'PASS: APIService v1beta1.metrics.k8s.io da Available'

kubectl -n kube-system logs deploy/metrics-server --tail=20 | tee "$EV/b5-log-sau.txt"
grep -qiE 'x509|failed to verify certificate' "$EV/b5-log-sau.txt" \
  && echo 'FAIL: van con loi certificate trong log moi' \
  || echo 'PASS: log moi khong con loi certificate'
```

**PASS:** đủ bốn dòng `PASS:` của bước này, không dòng `FAIL:` nào.

---

## B6. `kubectl top` và ranh giới của metrics-server

**Mục đích:** thu hoạch của B5, và là **gate quan trọng nhất của lab** — Lab 11b không mở được nếu
bước này không PASS. Kèm theo là phần phải hiểu để không dùng sai công cụ: metrics-server phục vụ
cái gì và **không** phục vụ cái gì.

### B6.1. `kubectl top node`

```bash
for i in $(seq 1 30); do
  kubectl top node >/dev/null 2>&1 && break
  sleep 10
done
kubectl top node | tee "$EV/b6-top-node.txt"

TN="$(kubectl top node --no-headers | wc -l)"
echo "so dong kubectl top node = $TN"
test "$TN" -eq 3 && echo 'PASS: kubectl top node liet ke du ba node'
```

Kiểm **giá trị**, không kiểm chuỗi — mọi node phải báo con số khác 0 ở cả CPU lẫn bộ nhớ:

```bash
kubectl top node --no-headers > "$EV/b6-top-node-raw.txt"
OK_T=0
while read -r n c cp m mp; do
  CM="$(cpu_m "$c")"; MM="$(mem_mi "$m")"
  echo "$n -> ${CM}m CPU / ${MM}Mi memory"
  if [ "$CM" -gt 0 ] && [ "$MM" -gt 0 ]; then OK_T=$(( OK_T + 1 )); fi
done < "$EV/b6-top-node-raw.txt"
echo "so node co so lieu hop le = $OK_T/3"

test "$OK_T" -eq 3 \
  && echo 'PASS: ca ba node deu bao muc su dung that, khac 0'
```

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B6.2. `kubectl top pod`

```bash
kubectl top pod -A | tee "$EV/b6-top-pod-all.txt"
kubectl top pod -n kube-system --sort-by=cpu | tee "$EV/b6-top-pod-kube-system.txt"

TP="$(kubectl top pod -n kube-system --no-headers | wc -l)"
KS_POD="$(kubectl -n kube-system get pods --field-selector=status.phase=Running --no-headers | wc -l)"
echo "top pod=$TP | Pod dang Running trong kube-system=$KS_POD"

test "$TP" -gt 0 && test "$TP" -le "$KS_POD" \
  && echo 'PASS: top pod bao cao cho Pod dang chay, khong nhieu hon so Pod that'
grep -q 'metrics-server' "$EV/b6-top-pod-kube-system.txt" \
  && echo 'PASS: metrics-server dang do ca chinh no'
```

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B6.3. Số liệu đó đến từ đâu

`kubectl top` không nói chuyện với kubelet. Nó gọi Metrics API tổng hợp, và API đó do metrics-server
phục vụ bằng cách tổng hợp đúng endpoint mà bạn đã đọc tay ở B2.3:

```bash
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes/$W1" > "$EV/b6-raw-node.txt"
head -c 400 "$EV/b6-raw-node.txt"; echo

RAW_CPU="$(grep -o '"cpu":"[^"]*"' "$EV/b6-raw-node.txt" | head -1 | cut -d'"' -f4)"
WINDOW="$(grep -o '"window":"[^"]*"' "$EV/b6-raw-node.txt" | head -1 | cut -d'"' -f4)"
echo "cpu tho = $RAW_CPU | window = $WINDOW"

test -n "$RAW_CPU" && echo 'PASS: Metrics API tra ve so lieu tho cho node'
test -n "$WINDOW"  && echo 'PASS: moi mau kem theo mot cua so thoi gian'
```

**Ý nghĩa:** trường `window` là chỗ ranh giới lộ ra. Mỗi phép đo chỉ có nghĩa **trong một cửa sổ vừa
qua**; không có trường nào cho phép hỏi "cách đây một giờ thì bao nhiêu". Độ dài cửa sổ và chu kỳ
làm mới **phụ thuộc cấu hình** của metrics-server, nên đừng nhớ nó như một con số.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B6.4. metrics-server **không** phải Prometheus

```bash
kubectl api-resources --api-group=metrics.k8s.io | tee "$EV/b6-api-resources.txt"

MR="$(kubectl api-resources --api-group=metrics.k8s.io --no-headers | wc -l)"
echo "so tai nguyen trong nhom metrics.k8s.io = $MR"
test "$MR" -eq 2 \
  && echo 'PASS: dung hai tai nguyen — nodes va pods, khong co gi de truy van qua khu'

kubectl top node --help > "$EV/b6-top-help.txt" 2>&1
TFLAG="$(grep -ciE '\-\-since|\-\-until|\-\-range' "$EV/b6-top-help.txt" || true)"
echo "so co ve khoang thoi gian trong kubectl top node = $TFLAG"
test "$TFLAG" -eq 0 \
  && echo 'PASS: kubectl top khong co bat ky co thoi gian nao'
```

**Ý nghĩa:** metrics-server là **bộ tổng hợp metric tài nguyên phục vụ tự động hóa trong cluster** —
`kubectl top`, và ở Lab 11b là HorizontalPodAutoscaler. Nó **không** lưu lịch sử, **không** phát
metric của thành phần hệ thống, **không** biết trạng thái đối tượng, và **không** thay được một kho
chuỗi thời gian. Ba khoảng trống đó tương ứng đúng với ba thứ lab cố ý không cài: Prometheus (lịch
sử và truy vấn), một scraper cho `/metrics` (metric thành phần), kube-state-metrics (trạng thái đối
tượng). Xem [bảng mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành) cho lý do đầy đủ.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

---

## B7. Log của container — đường đi và ranh giới

**Mục đích:** đi hết xương sống của bài [158](../158-logging-vi.md) ở tầng container và tầng node.
Bốn câu hỏi phải trả lời xong ở đây: `kubectl logs` lấy dữ liệu từ đâu, cái gì quyết định một dòng
log có hiện ra ở đó hay không, file log nằm chỗ nào trên đĩa node, và một agent cấp node nhìn thấy
những gì.

### B7.1. Pod đếm số và `kubectl logs`

```bash
cat > "$WK/b7-counter.yaml" <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: counter
  namespace: lab-11a
spec:
  nodeName: lab-k8s-worker1
  containers:
    - name: count
      image: busybox:1.37
      args:
        - /bin/sh
        - -c
        - 'i=0; while true; do echo "$i: $(date)"; i=$((i+1)); sleep 1; done'
YAML

kubectl apply -f "$WK/b7-counter.yaml"
kubectl -n lab-11a wait --for=condition=Ready pod/counter --timeout=180s
kubectl -n lab-11a logs counter --tail=5 | tee "$EV/b7-counter.txt"
```

```bash
L1="$(kubectl -n lab-11a logs counter --tail=100 | wc -l)"
sleep 5
L2="$(kubectl -n lab-11a logs counter --tail=100 | wc -l)"
FIRST="$(kubectl -n lab-11a logs counter | head -1 | cut -d: -f1)"
echo "dong dau tien bat dau bang so '$FIRST' | so dong lan 1=$L1, lan 2=$L2"

test "$FIRST" = '0' \
  && echo 'PASS: kubectl logs tra ve tu dong dau tien cua container, khong phai tu luc ban goi'
test "$L2" -ge "$L1" \
  && echo 'PASS: log tiep tuc day len — day la mot luong dang chay, khong phai anh chup'
```

**Ý nghĩa:** container runtime bắt `stdout`/`stderr` của ứng dụng, chuẩn hóa theo **định dạng log
CRI**, và kubelet trên node phục vụ file đó cho `kubectl logs`. Không có agent nào tham gia ở đây,
và cũng không có kho lưu trữ nào — dữ liệu vẫn nằm trên đúng node đang chạy container.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B7.2. `stdout` hay file — thứ quyết định `kubectl logs` thấy gì

Đây là cái bẫy đắt nhất của bài 158. Pod dưới đây tái dựng đúng ví dụ *sidecar truyền luồng*: một
container ghi vào **hai file** trong một `emptyDir` dùng chung, hai sidecar `tail` từng file ra
`stdout` của chính chúng.

```bash
cat > "$WK/b7-sidecar.yaml" <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: two-files-counter
  namespace: lab-11a
spec:
  nodeName: lab-k8s-worker1
  containers:
    - name: count
      image: busybox:1.37
      args:
        - /bin/sh
        - -c
        - >
          i=0;
          while true;
          do
            echo "$i: $(date)" >> /var/log/1.log;
            echo "$(date) INFO $i" >> /var/log/2.log;
            i=$((i+1));
            sleep 1;
          done
      volumeMounts:
        - name: varlog
          mountPath: /var/log
    - name: count-log-1
      image: busybox:1.37
      args: [/bin/sh, -c, 'tail -n+1 -F /var/log/1.log']
      volumeMounts:
        - name: varlog
          mountPath: /var/log
    - name: count-log-2
      image: busybox:1.37
      args: [/bin/sh, -c, 'tail -n+1 -F /var/log/2.log']
      volumeMounts:
        - name: varlog
          mountPath: /var/log
  volumes:
    - name: varlog
      emptyDir: {}
YAML

kubectl apply -f "$WK/b7-sidecar.yaml"
kubectl -n lab-11a wait --for=condition=Ready pod/two-files-counter --timeout=180s
sleep 5
```

```bash
kubectl -n lab-11a logs two-files-counter -c count       > "$EV/b7-c-count.txt"
kubectl -n lab-11a logs two-files-counter -c count-log-1 > "$EV/b7-c-log1.txt"
kubectl -n lab-11a logs two-files-counter -c count-log-2 > "$EV/b7-c-log2.txt"

for f in b7-c-count b7-c-log1 b7-c-log2; do
  printf '%-14s %s byte\n' "$f" "$(wc -c < "$EV/$f.txt")"
done

test "$(wc -c < "$EV/b7-c-count.txt")" -eq 0 \
  && echo 'PASS: container ghi VAO FILE thi kubectl logs khong thay gi'
test "$(wc -c < "$EV/b7-c-log1.txt")" -gt 0 \
  && test "$(wc -c < "$EV/b7-c-log2.txt")" -gt 0 \
  && echo 'PASS: hai sidecar truyen luong deu hien ra o kubectl logs'
grep -q ' INFO ' "$EV/b7-c-log2.txt" \
  && ! grep -q ' INFO ' "$EV/b7-c-log1.txt" \
  && echo 'PASS: hai luong log tach roi han nhau, moi sidecar mot dinh dang'
```

**Ý nghĩa:** yếu tố quyết định **không phải** là "có sidecar hay không", mà là **log có đi qua
`stdout`/`stderr` của một container hay không**. Sidecar truyền luồng thì có, nên nó rơi vào đường
đi thông thường do kubelet xử lý. Sidecar chạy **agent ghi log** thì đọc file rồi đẩy thẳng ra
backend — log đó **không do kubelet kiểm soát** nên `kubectl logs` không bao giờ thấy. Container
`count` ở trên chính là mô hình thu nhỏ của trường hợp thứ hai.

Bài cũng cảnh báo cái giá: ghi vào file rồi truyền ra `stdout` có thể **tăng gấp đôi dung lượng
lưu trữ cần trên node**, nên khi ứng dụng chỉ ghi một file thì trỏ thẳng đích ghi vào `/dev/stdout`
gọn hơn hẳn.

**PASS:** đủ ba dòng `PASS:` của bước này.

### B7.3. `--previous` — log của lần chạy trước

Bước này là **fault injection**, nên nó chạy trên `lab-k8s-worker2` theo đúng quy ước:

```bash
cat > "$WK/b7-crash.yaml" <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: crash-once
  namespace: lab-11a
spec:
  nodeName: lab-k8s-worker2
  restartPolicy: Always
  containers:
    - name: app
      image: busybox:1.37
      command:
        - sh
        - -c
        - 'echo "LAN-CHAY-$(date +%s) bat dau"; sleep 5; echo "sap thoat voi ma 1"; exit 1'
YAML

kubectl apply -f "$WK/b7-crash.yaml"

for i in $(seq 1 60); do
  RC="$(kubectl -n lab-11a get pod crash-once \
    -o jsonpath='{.status.containerStatuses[0].restartCount}' 2>/dev/null)"
  test "${RC:-0}" -ge 1 && break
  sleep 5
done
kubectl -n lab-11a get pod crash-once -o wide
echo "restartCount = ${RC:-0}"
```

```bash
kubectl -n lab-11a logs crash-once             > "$EV/b7-crash-hien-tai.txt" 2>&1 || true
kubectl -n lab-11a logs crash-once --previous  > "$EV/b7-crash-truoc.txt"     2>&1 || true

grep -c 'bat dau' "$EV/b7-crash-truoc.txt"
NOW_ID="$(grep -o 'LAN-CHAY-[0-9]*' "$EV/b7-crash-hien-tai.txt" | head -1)"
PRE_ID="$(grep -o 'LAN-CHAY-[0-9]*' "$EV/b7-crash-truoc.txt"     | head -1)"
echo "lan chay hien tai=$NOW_ID | lan chay truoc=$PRE_ID"

test "${RC:-0}" -ge 1 && echo 'PASS: container da khoi dong lai it nhat mot lan'
test -n "$PRE_ID" && echo 'PASS: --previous doc duoc log cua lan chay truoc'
test -n "$NOW_ID" && test "$NOW_ID" != "$PRE_ID" \
  && echo 'PASS: hai lan chay la hai luong log khac nhau, phan biet duoc bang gia tri'
```

**Ý nghĩa:** kubelet **giữ lại container đã kết thúc cùng log của nó** khi container khởi động lại —
đó là toàn bộ cơ sở của `--previous`, và là công cụ đầu tiên bạn với tới khi gặp
`CrashLoopBackOff`. Nhưng bảo đảm đó dừng ở ranh giới Pod: **Pod bị trục xuất thì container và log
của chúng đi theo**, và mất node là mất luôn file. Đó là lý do tồn tại của ghi log cấp cluster ở
B7.5.

**PASS:** đủ ba dòng `PASS:` của bước này.

### B7.4. File log thật trên node

Chạy trên **`lab-k8s-worker1`** — node đang giữ Pod `counter`:

```bash
POD_UID="$(kubectl -n lab-11a get pod counter -o jsonpath='{.metadata.uid}')"
echo "$POD_UID"
ssh "$W1" "sudo ls -l /var/log/pods/lab-11a_counter_$POD_UID/count/" | tee "$EV/b7-file-node.txt"
ssh "$W1" "sudo ls -l /var/log/containers/ | grep counter_lab-11a" | tee -a "$EV/b7-file-node.txt"
```

So **nội dung** hai đầu — file trên node và output của `kubectl logs` — thay vì chỉ nhìn cho có:

```bash
NODE_LINE="$(ssh "$W1" "sudo tail -n 1 /var/log/pods/lab-11a_counter_$POD_UID/count/0.log" \
  | sed 's/.*stdout F //')"
K_LINES="$(kubectl -n lab-11a logs counter | tail -n 5)"
echo "dong cuoi tren node : $NODE_LINE"

echo "$K_LINES" | grep -Fq "$NODE_LINE" \
  && echo 'PASS: dong log tren dia node xuat hien y nguyen trong kubectl logs'
ssh "$W1" "sudo test -f /var/log/pods/lab-11a_counter_$POD_UID/count/0.log" \
  && echo 'PASS: kubelet chi thi runtime ghi log vao /var/log/pods nhu bai 158 mo ta'
ssh "$W1" "sudo ls /var/log/containers/ | grep -q '^counter_lab-11a'" \
  && echo 'PASS: /var/log/containers co symlink tro toi dung file do'
```

**Ý nghĩa:** ba dòng đó khép kín đường đi: ứng dụng in ra `stdout` → runtime ghi theo định dạng CRI
vào `/var/log/pods/<ns>_<pod>_<uid>/<container>/N.log` → `/var/log/containers` là lớp symlink có
tên phẳng để agent dễ đọc → kubelet đọc file rồi trả về cho `kubectl logs`. Tiền tố `stdout F`
trong file chính là phần định dạng CRI thêm vào, và `kubectl logs` gỡ nó ra trước khi trả về.

**PASS:** đủ ba dòng `PASS:` của bước này.

### B7.5. Agent ghi log cấp node

Bài 158 khuyến nghị chạy agent ghi log **dưới dạng `DaemonSet`**, một agent trên mỗi node, và nhấn
mạnh rằng cách này **không đòi hỏi thay đổi gì ở ứng dụng**. Dựng đúng hình dạng đó bằng
`busybox:1.37` — agent ở đây chỉ đếm và liệt kê, không đẩy đi đâu, vì backend log tập trung nằm
ngoài phạm vi Kubernetes.

Toleration lấy từ nhóm bài 7a: không có nó thì `lab-k8s-master` không nhận agent, và bạn sẽ mù
đúng chỗ có control plane.

```bash
cat > "$WK/b7-agent.yaml" <<'YAML'
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent-demo
  namespace: lab-11a
spec:
  selector:
    matchLabels:
      app: log-agent-demo
  template:
    metadata:
      labels:
        app: log-agent-demo
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      containers:
        - name: agent
          image: busybox:1.37
          env:
            - name: MY_NODE
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
          command:
            - sh
            - -c
            - >
              while true;
              do
                echo "node=$MY_NODE files=$(ls /varlog/containers 2>/dev/null | wc -l) apiserver=$(ls /varlog/containers 2>/dev/null | grep -c '^kube-apiserver') counter=$(ls /varlog/containers 2>/dev/null | grep -c '^counter_lab-11a')";
                sleep 20;
              done
          volumeMounts:
            - name: varlog
              mountPath: /varlog
              readOnly: true
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
            type: Directory
YAML

kubectl apply -f "$WK/b7-agent.yaml"
kubectl -n lab-11a rollout status ds/log-agent-demo --timeout=240s
kubectl -n lab-11a get pods -l app=log-agent-demo -o wide
```

```bash
DS_D="$(kubectl -n lab-11a get ds log-agent-demo -o jsonpath='{.status.desiredNumberScheduled}')"
DS_R="$(kubectl -n lab-11a get ds log-agent-demo -o jsonpath='{.status.numberReady}')"
echo "desiredNumberScheduled=$DS_D | numberReady=$DS_R"
test "$DS_D" -eq 3 && test "$DS_R" -eq 3 \
  && echo 'PASS: dung mot agent tren moi node, ke ca control plane'
```

Đọc báo cáo của từng agent — mỗi agent chỉ thấy log của **node nó đang đứng**:

```bash
: > "$EV/b7-agent-bao-cao.txt"
for n in $NODES; do
  P="$(kubectl -n lab-11a get pod -l app=log-agent-demo \
    -o jsonpath="{range .items[?(@.spec.nodeName=='$n')]}{.metadata.name}{end}")"
  kubectl -n lab-11a logs "$P" --tail=1 | tee -a "$EV/b7-agent-bao-cao.txt"
done
cat "$EV/b7-agent-bao-cao.txt"
```

```bash
A_MASTER="$(grep "node=$MASTER " "$EV/b7-agent-bao-cao.txt" | grep -o 'apiserver=[0-9]*' | cut -d= -f2)"
A_W1="$(grep "node=$W1 " "$EV/b7-agent-bao-cao.txt" | grep -o 'apiserver=[0-9]*' | cut -d= -f2)"
C_W1="$(grep "node=$W1 " "$EV/b7-agent-bao-cao.txt" | grep -o 'counter=[0-9]*' | cut -d= -f2)"
C_W2="$(grep "node=$W2 " "$EV/b7-agent-bao-cao.txt" | grep -o 'counter=[0-9]*' | cut -d= -f2)"
echo "apiserver: master=$A_MASTER worker1=$A_W1 | counter: worker1=$C_W1 worker2=$C_W2"

test "${A_MASTER:-0}" -ge 1 && test "${A_W1:-0}" -eq 0 \
  && echo 'PASS: chi agent tren master thay log cua kube-apiserver'
test "${C_W1:-0}" -ge 1 && test "${C_W2:-0}" -eq 0 \
  && echo 'PASS: chi agent tren worker1 thay log cua Pod counter'
test "$(grep -c 'files=' "$EV/b7-agent-bao-cao.txt")" -eq 3 \
  && echo 'PASS: ca ba agent deu doc duoc thu muc log cua node minh'
```

**Ý nghĩa:** đây là toàn bộ kiến trúc *agent cấp node* ở dạng nhỏ nhất chạy được. Nó minh họa cả
điểm mạnh — một agent mỗi node, không đụng vào ứng dụng, thấy hết mọi container trên node — lẫn
điểm giới hạn: **tầm nhìn của nó dừng ở biên node**. Một agent thật khác ở chỗ nó chuyển tiếp
những dòng này ra một kho log tập trung; phần đó nằm ngoài Kubernetes và thuộc
[giai đoạn 23](../00-ALO-TRINH-ADMIN.md#giai-đoạn-23--giám-sát-và-cảnh-báo).

**PASS:** đủ bốn dòng `PASS:` của bước này.

### B7.6. Dọn phần workload không còn dùng của B7

Giữ lại `counter` — B8 và B9 còn dùng nó.

```bash
kubectl -n lab-11a delete -f "$WK/b7-sidecar.yaml" --wait=true --timeout=180s
kubectl -n lab-11a delete -f "$WK/b7-crash.yaml"   --wait=true --timeout=180s
kubectl -n lab-11a delete -f "$WK/b7-agent.yaml"   --wait=true --timeout=180s
kubectl -n lab-11a get pods

test "$(kubectl -n lab-11a get pods --no-headers | grep -cvE '^counter ')" -eq 0 \
  && echo 'PASS: chi con Pod counter trong namespace lab-11a'
```

**PASS:** dòng `PASS:` của bước này xuất hiện.

---

## B8. Kubelet xoay vòng log

**Mục đích:** kiểm chứng câu quan trọng nhất mà bài [158](../158-logging-vi.md) đóng khung riêng:
**chỉ nội dung của file log mới nhất là truy cập được qua `kubectl logs`**. Bước này không tin lời —
nó đọc ngưỡng thật của kubelet, làm tràn ngưỡng đó bằng một workload thật, rồi so **bằng con số**.

### B8.1. Đọc hai ngưỡng thật của kubelet

```bash
CLMS="$(cfgz containerLogMaxSize "$EV/b0-configz-$W2.json")"
CLMF="$(cfgz containerLogMaxFiles "$EV/b0-configz-$W2.json")"
echo "tren $W2: containerLogMaxSize=${CLMS:-<khong khai>} containerLogMaxFiles=${CLMF:-<khong khai>}"

MAXB="$(to_bytes "$CLMS")"
echo "nguong mot file log = $MAXB byte"

test "$MAXB" -gt 0 && test "${CLMF:-0}" -ge 2 \
  && echo 'PASS: doc duoc ca hai nguong xoay vong tu cau hinh hieu luc cua kubelet'
```

**Ý nghĩa:** hai giá trị này là **cấu hình của kubelet**, không phải của ứng dụng và cũng không phải
của container runtime. Kubelet đọc chúng rồi chỉ thị cho runtime qua CRI ghi vào đúng vị trí và
xoay vòng khi cần. Lab **không sửa** chúng — đổi giá trị là việc của
[giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy).

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B8.2. Một workload ghi vượt ngưỡng

Bước này ghi vài chục MB log, nên nó ghim vào `lab-k8s-worker2` theo đúng quy ước fault injection.

```bash
cat > "$WK/b8-flood.yaml" <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: log-flood
  namespace: lab-11a
spec:
  nodeName: lab-k8s-worker2
  restartPolicy: Never
  containers:
    - name: flood
      image: busybox:1.37
      command:
        - sh
        - -c
        - >
          P=$(dd if=/dev/zero bs=1000 count=1 2>/dev/null | tr '\0' 'x');
          i=0;
          while [ $i -lt 40000 ];
          do
            echo "$i $P";
            i=$((i+1));
          done;
          echo "FLOOD-DONE";
          sleep 900
YAML

kubectl apply -f "$WK/b8-flood.yaml"
kubectl -n lab-11a wait --for=condition=Ready pod/log-flood --timeout=180s

for i in $(seq 1 60); do
  kubectl -n lab-11a logs log-flood --tail=1 2>/dev/null | grep -q 'FLOOD-DONE' && break
  sleep 5
done
kubectl -n lab-11a logs log-flood --tail=1

kubectl -n lab-11a logs log-flood --tail=1 | grep -q 'FLOOD-DONE' \
  && echo 'PASS: workload da ghi xong khoi log'
```

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B8.3. So bằng con số: `kubectl logs` chỉ trả file mới nhất

Kubelet giám sát và xoay vòng theo **chu kỳ**, độ dài chu kỳ **phụ thuộc cấu hình**, nên chờ bằng
vòng lặp có điều kiện thoát chứ không bằng một con số cố định:

```bash
FLOOD_UID="$(kubectl -n lab-11a get pod log-flood -o jsonpath='{.metadata.uid}')"
LOGDIR="/var/log/pods/lab-11a_log-flood_$FLOOD_UID/flood"

for i in $(seq 1 60); do
  NF="$(ssh "$W2" "sudo ls -1 $LOGDIR | wc -l")"
  test "$NF" -ge 2 && break
  sleep 5
done
ssh "$W2" "sudo ls -l $LOGDIR" | tee "$EV/b8-file-tren-node.txt"
echo "so file log trong thu muc container = $NF"
```

```bash
GOT_B="$(kubectl -n lab-11a logs log-flood | wc -c)"
GOT_L="$(kubectl -n lab-11a logs log-flood | wc -l)"
echo "kubectl logs tra ve $GOT_B byte / $GOT_L dong; nguong mot file = $MAXB byte; da ghi 40001 dong"

test "$NF" -ge 2 \
  && echo "PASS: kubelet da xoay vong — co $NF file trong thu muc container"
test "$NF" -le "${CLMF:-5}" \
  && echo "PASS: so file khong vuot containerLogMaxFiles = ${CLMF:-5}"
test "$GOT_B" -le "$MAXB" \
  && echo 'PASS: kubectl logs tra ve khong qua mot file — dung bang so, khong bang cam giac'
test "$GOT_L" -lt 40001 \
  && echo 'PASS: phan log cu da roi khoi tam voi cua kubectl logs'
```

**Ý nghĩa:** đây là cái bẫy vận hành hay gặp nhất khi truy sự cố. Ứng dụng đã in ra 40 000 dòng, các
dòng cũ **vẫn còn trên đĩa node** dưới dạng file đã xoay vòng, nhưng `kubectl logs` không chạm tới
chúng. Muốn giữ được toàn bộ, bạn cần đúng thứ B7.5 phác ra: một agent cấp node đọc `/var/log` và
đẩy ra kho lưu trữ có vòng đời riêng.

**PASS:** đủ bốn dòng `PASS:` của bước này.

### B8.4. Dọn khối log khỏi node

```bash
kubectl -n lab-11a delete -f "$WK/b8-flood.yaml" --wait=true --timeout=180s

for i in $(seq 1 30); do
  ssh "$W2" "sudo test -d $LOGDIR" || break
  sleep 5
done
ssh "$W2" "sudo test -d $LOGDIR" \
  && echo 'FAIL: thu muc log cua Pod van con tren node' \
  || echo 'PASS: xoa Pod la kubelet don luon thu muc log cua no'

ssh "$W2" "df -h /var | tail -1" | tee "$EV/b8-dia-sau.txt"
```

**Ý nghĩa:** log container **sống theo Pod**. Đây cũng chính là câu trả lời cho "node chết thì log
còn không": không còn, trừ khi đã có ghi log cấp cluster.

**PASS:** dòng `PASS:` của bước này xuất hiện, không dòng `FAIL:` nào.

---

## B9. Log của thành phần hệ thống

**Mục đích:** bài [158](../158-logging-vi.md) nói về log của **workload**; bài
[159](../159-system-logs-vi.md) nói về log của **chính Kubernetes**. Ranh giới của bài 159 rất gọn:
thành phần hệ thống chia hai loại theo **nơi chúng chạy**, và hai loại đó ghi log ở hai chỗ khác
nhau. Bước này chứng minh cả hai trên node thật.

### B9.1. Loại không chạy trong container: kubelet ghi journald

Chạy trên **`lab-k8s-worker1`**:

```bash
ssh "$W1" 'journalctl -u kubelet --no-pager -n 10' | tee "$EV/b9-journal-kubelet.txt"
ssh "$W1" 'sudo ls -l /var/log/kubelet.log 2>&1 | tail -1'
ssh "$W1" 'systemctl show -p FragmentPath kubelet'
```

```bash
J_LINES="$(wc -l < "$EV/b9-journal-kubelet.txt")"
echo "so dong doc duoc tu journald = $J_LINES"

test "$J_LINES" -gt 0 \
  && echo 'PASS: kubelet ghi vao journald tren node co systemd'
ssh "$W1" 'sudo test -f /var/log/kubelet.log' \
  && echo 'FAIL: bat ngo — co file /var/log/kubelet.log' \
  || echo 'PASS: khong co /var/log/kubelet.log — tim o do la huong sai'
```

**Ý nghĩa:** kubelet và container runtime **không chạy trong container**; chúng là service của hệ
điều hành, nên trên máy có systemd chúng ghi vào journald. Chỉ khi **không có** systemd chúng mới
ghi file `.log` trong `/var/log`. Node lab có systemd, nên đi tìm `/var/log/kubelet.log` là đi sai
hướng — và gate thứ hai chứng minh điều đó thay vì bắt bạn tin.

**PASS:** hai dòng `PASS:` của bước này xuất hiện, không dòng `FAIL:` nào.

### B9.2. Loại chạy trong container: scheduler ghi file `.log`

Chạy trên **`lab-k8s-master`**:

```bash
kubectl -n kube-system logs "kube-scheduler-$MASTER" --tail=5 | tee "$EV/b9-log-scheduler.txt"
sudo ls -d /var/log/pods/kube-system_kube-scheduler-*/ | tee "$EV/b9-thu-muc-scheduler.txt"
sudo ls -l /var/log/containers/ | grep -E 'kube-(apiserver|scheduler|controller-manager)' \
  | tee "$EV/b9-symlink-control-plane.txt"
```

```bash
SCHED_DIR="$(sudo ls -d /var/log/pods/kube-system_kube-scheduler-*/ | head -1)"
echo "thu muc log cua scheduler tren node = $SCHED_DIR"

SVC_N="$(systemctl list-units --all --type=service --no-legend --no-pager \
  | grep -cE 'kube-(apiserver|scheduler|controller-manager)' || true)"
echo "so unit systemd mang ten thanh phan control plane = $SVC_N"

test -n "$SCHED_DIR" \
  && echo 'PASS: thanh phan chay trong Pod ghi file .log duoi /var/log'
test "$(wc -l < "$EV/b9-symlink-control-plane.txt")" -ge 3 \
  && echo 'PASS: ca ba thanh phan control plane deu co file log tren node'
test "$SVC_N" -eq 0 \
  && echo 'PASS: chung khong co unit systemd nao — chung khong phai service cua OS'
```

**Ý nghĩa:** đây là lý do hai thành phần của **cùng một** cluster lại nằm hai chỗ khác nhau.
`kube-scheduler`, `kube-controller-manager` và `kube-apiserver` chạy bên trong Pod (static Pod), nên
chúng ghi file `.log` trong `/var/log`, **bỏ qua** cơ chế ghi log mặc định của container và không
vào journal của node. Nhớ ranh giới này thì lần sau bạn không mất mười phút gõ `journalctl -u
kube-apiserver` cho một thứ không tồn tại.

Kèm theo là một cảnh báo vận hành của bài 159: log trong `/var/log` **vẫn cần được xoay vòng**, và
Kubernetes **không quản lý** việc xoay vòng đó — nó thuộc `logrotate` của hệ điều hành hoặc thuộc
công cụ triển khai.

**PASS:** đủ ba dòng `PASS:` của bước này.

### B9.3. Định dạng klog

```bash
head -3 "$EV/b9-log-scheduler.txt"
kubectl -n kube-system logs "kube-apiserver-$MASTER" --tail=3 | tee "$EV/b9-log-apiserver.txt"

KLOG_N="$(grep -cE '^[IWEF][0-9]{4} [0-9]{2}:[0-9]{2}:[0-9]{2}\.[0-9]+' \
  "$EV/b9-log-scheduler.txt" "$EV/b9-log-apiserver.txt" | awk -F: '{s+=$2} END {print s+0}')"
echo "so dong khop dinh dang klog = $KLOG_N"

test "$KLOG_N" -ge 2 \
  && echo 'PASS: doc duoc header klog — muc nghiem trong, ngay gio, file nguon'
```

**Ý nghĩa:** ký tự đầu là mức nghiêm trọng (`I`, `W`, `E`, `F`), sau đó là ngày giờ rồi tên file
nguồn và số dòng. Nhưng bài 159 mở đầu bằng một cảnh báo phải nhớ hơn cả định dạng: **nội dung log
không thuộc phạm vi bảo đảm ổn định của Kubernetes API**. Cờ dòng lệnh thì có cam kết; từng dòng log
và định dạng của chúng **có thể đổi giữa các bản phát hành**. Đừng bao giờ viết một cảnh báo bằng
cách khớp đúng một chuỗi ký tự trong log của kube-controller-manager — nó sẽ im lặng hỏng sau một
lần nâng cấp.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B9.4. Mức chi tiết `-v` là một thang có hướng

Cờ `-v` của klog nằm trong cả `kubectl`, nên bạn đo được hướng của thang này **mà không đụng vào
control plane**:

```bash
kubectl get --raw '/version' -v=2 > /dev/null 2> "$EV/b9-v2.txt"
kubectl get --raw '/version' -v=6 > /dev/null 2> "$EV/b9-v6.txt"
kubectl get --raw '/version' -v=8 > /dev/null 2> "$EV/b9-v8.txt"

for f in b9-v2 b9-v6 b9-v8; do
  printf '%-8s %s dong\n' "$f" "$(wc -l < "$EV/$f.txt")"
done

N2="$(wc -l < "$EV/b9-v2.txt")"
N6="$(wc -l < "$EV/b9-v6.txt")"
N8="$(wc -l < "$EV/b9-v8.txt")"

test "$N6" -gt "$N2" && test "$N8" -gt "$N6" \
  && echo 'PASS: tang -v thi so su kien duoc ghi tang len, do bang so dong'
grep -q 'GET https' "$EV/b9-v6.txt" \
  && echo 'PASS: tu -v=6 moi thay tung request HTTP — su kien it nghiem trong hon'
```

Còn mức verbosity của **thành phần** thì chỉ đọc, không sửa:

```bash
for c in kube-apiserver kube-scheduler kube-controller-manager; do
  echo "--- $c"
  kubectl -n kube-system get pod "$c-$MASTER" \
    -o jsonpath='{range .spec.containers[0].command[*]}{@}{"\n"}{end}' | grep -E '^--v=' \
    || echo '(khong khai --v — dung gia tri mac dinh cua klog)'
done | tee "$EV/b9-verbosity-control-plane.txt"

test -s "$EV/b9-verbosity-control-plane.txt" \
  && echo 'PASS: doc duoc muc verbosity that cua ba thanh phan control plane'
```

**Ý nghĩa:** tăng `-v` **ghi thêm các sự kiện ngày càng ít nghiêm trọng hơn**; `-v=0` chỉ ghi sự
kiện nghiêm trọng. Đổi giá trị này cho một thành phần control plane nghĩa là sửa manifest static
Pod rồi chờ kubelet dựng lại Pod — việc của
[giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy), và B12.2 sẽ
chứng minh bằng checksum rằng lab này không làm.

**PASS:** đủ ba dòng `PASS:` của bước này.

### B9.5. Log Query — đọc cấu hình, không bật

Tính năng Log Query cho phép lấy log của service trên node qua API. Bài 159 xếp nó vào phần hoãn
tới [giai đoạn 24](../00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố), và kèm cảnh báo an ninh:
cấp quyền `nodes/proxy`, **kể cả chỉ `get`**, cũng đồng thời mở các API kubelet rất mạnh. Lab chỉ
**đọc giá trị hiệu lực** rồi kiểm tra hành vi có khớp với giá trị đó không.

```bash
LQ="$(cfgz enableSystemLogHandler "$EV/b0-configz-$W1.json")"
echo "enableSystemLogHandler tren $W1 = ${LQ:-<khong khai trong configz>}"

kubectl get --raw "/api/v1/nodes/$W1/proxy/logs/?query=kubelet&tailLines=5" \
  > "$EV/b9-log-query.txt" 2> "$EV/b9-log-query-err.txt" \
  && LQ_OK=1 || LQ_OK=0
echo "goi endpoint /logs -> LQ_OK=$LQ_OK"
```

```bash
if [ "$LQ" = 'true' ]; then
  test "$LQ_OK" -eq 1 && test -s "$EV/b9-log-query.txt" \
    && echo 'PASS: handler dang bat va endpoint tra ve log — khop voi cau hinh' \
    || echo 'FAIL: handler bat ma endpoint khong tra ve gi'
else
  test "$LQ_OK" -eq 0 \
    && echo 'PASS: handler dang tat va endpoint im lang — khop voi cau hinh' \
    || echo 'FAIL: handler tat ma endpoint van tra loi'
fi
```

**Ý nghĩa:** gate này đúng ở **cả hai chiều**, nên nó kiểm chứng được điều thật sự phải nắm: hành vi
của endpoint là **hệ quả trực tiếp của một tùy chọn kubelet**, không phải thứ ngẫu nhiên. Nếu bạn
cần bật nó để chẩn đoán, đó là thao tác sửa cấu hình kubelet — làm ở giai đoạn 20, và luôn tắt lại
sau khi xong.

**PASS:** dòng `PASS:` của nhánh tương ứng xuất hiện, không dòng `FAIL:` nào.

---

## B10. Trace của thành phần hệ thống

**Mục đích:** trụ cột thứ ba, và cũng là trụ cột **non nhất** — chính bài
[161](../161-system-traces-vi.md) nói phần đo đạc còn đang phát triển tích cực và chưa có bảo đảm
tương thích ngược. Trên cluster baseline, tracing **không được bật**. Việc của B10 là chứng minh
điều đó ở cả ba chỗ và đọc được cấu hình sẽ phải sửa nếu muốn bật — **không sửa gì**.

### B10.1. kube-apiserver không có file cấu hình tracing

```bash
kubectl -n kube-system get pod "kube-apiserver-$MASTER" \
  -o jsonpath='{range .spec.containers[0].command[*]}{@}{"\n"}{end}' \
  | tee "$EV/b10-apiserver-command.txt"

TC="$(grep -c -- '--tracing-config-file=' "$EV/b10-apiserver-command.txt" || true)"
echo "so co --tracing-config-file = $TC"

test "$TC" -eq 0 \
  && echo 'PASS: kube-apiserver khong duoc cap file cau hinh tracing'
sudo ls /etc/kubernetes/ | grep -i 'tracing' \
  && echo 'FAIL: co file cau hinh tracing tren node' \
  || echo 'PASS: khong co file TracingConfiguration nao tren master'
```

**Ý nghĩa:** bật tracing cho apiserver nghĩa là ghi một object `TracingConfiguration` ra file, thêm
cờ `--tracing-config-file` vào manifest static Pod, rồi chờ kubelet dựng lại kube-apiserver. Ba
thao tác đó đều nằm trong `/etc/kubernetes` và đều thuộc
[giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy). Lab này
**cấm** chúng, và B12.2 kiểm bằng checksum của chính manifest bạn vừa đọc.

**PASS:** hai dòng `PASS:` của bước này xuất hiện, không dòng `FAIL:` nào.

### B10.2. kubelet không có đoạn `tracing`

```bash
for n in $NODES; do
  printf '%-18s ' "$n"
  grep -o '"tracing":{[^}]*}' "$EV/b0-configz-$n.json" || echo '(khong co khoa tracing)'
done | tee "$EV/b10-kubelet-tracing.txt"

TR_N="$(grep -c '"tracing"' "$EV/b10-kubelet-tracing.txt" || true)"
echo "so node co khoa tracing trong cau hinh hieu luc = $TR_N"

test "$TR_N" -eq 0 \
  && echo 'PASS: khong kubelet nao duoc cau hinh tracing'
```

**Ý nghĩa:** giá trị mà bạn sẽ phải đặt nếu bật là `samplingRatePerMillion` — đọc đúng thang của nó:
`100` là ghi span cho **1 trong mỗi 10 000** request, `1000000` là mọi span. Và một quy tắc dễ nhầm:
**quyết định lấy mẫu của span cha luôn được tôn trọng**, tỷ lệ này chỉ áp dụng cho span **không có
cha**.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B10.3. Cổng OTLP im lặng trên cả ba node

Các thành phần Kubernetes xuất trace bằng gRPC exporter cho OTLP, mặc định tới cổng `4317`. Nếu
tracing đang chạy thì phải có ai đó lắng nghe ở đó — hoặc một OpenTelemetry Collector, hoặc backend
trực tiếp:

```bash
: > "$EV/b10-cong-4317.txt"
for n in $NODES; do
  echo "--- $n" >> "$EV/b10-cong-4317.txt"
  ssh "$n" "ss -lnt | grep ':4317' || echo 'khong ai lang nghe 4317'" >> "$EV/b10-cong-4317.txt"
done
cat "$EV/b10-cong-4317.txt"

L_N="$(grep -c ':4317' "$EV/b10-cong-4317.txt" || true)"
echo "so node dang lang nghe 4317 = $L_N"

test "$L_N" -eq 0 \
  && echo 'PASS: khong co collector nao — khop voi hai buoc tren'
test "$(grep -c 'khong ai lang nghe 4317' "$EV/b10-cong-4317.txt")" -eq 3 \
  && echo 'PASS: kiem du ca ba node'
```

**Ý nghĩa:** ba bằng chứng độc lập cùng chỉ một kết luận — **nguồn phát tắt, và không có bộ thu**.
Đó là lý do các phần còn lại của bài 161 (liên kết cha–con giữa span kubelet và span containerd,
việc apiserver **không dùng** trace context của request đến) không kiểm chứng được ở lab này; xem
[bảng mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành).

Điều phải mang theo khi rời B10: tracing luôn kèm **chi phí mạng và CPU**, và tên span cùng thuộc
tính **chưa ổn định**. Vì vậy nó là công cụ cho những đợt chẩn đoán có thời hạn, không phải nền cho
một dashboard trực dài hạn.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

---

## B11. Shell vào container đang chạy

**Mục đích:** bộ công cụ gỡ lỗi mà bài [304](../304-get-shell-running-container-vi.md) dạy, cộng
ranh giới mà hai trang mục lục [296](../296-debug-vi.md) và
[297](../297-debug-application-vi.md) vẽ ra. Lab dùng `busybox:1.37` thay cho image trong bài — cùng
cơ chế, khác chỗ shell là `sh` chứ không phải `bash`, và không phải kéo image mới về.

### B11.1. Mở shell

```bash
cat > "$WK/b11-shell-demo.yaml" <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: shell-demo
  namespace: lab-11a
spec:
  nodeName: lab-k8s-worker1
  volumes:
    - name: shared-data
      emptyDir: {}
  containers:
    - name: web
      image: busybox:1.37
      command:
        - sh
        - -c
        - 'mkdir -p /www && echo cho-den-khi-ban-ghi-de > /www/index.html && httpd -f -p 8080 -h /www'
      volumeMounts:
        - name: shared-data
          mountPath: /www
YAML

kubectl apply -f "$WK/b11-shell-demo.yaml"
kubectl -n lab-11a wait --for=condition=Ready pod/shell-demo --timeout=180s
kubectl -n lab-11a get pod shell-demo -o wide
```

Mở shell tương tác — gõ `exit` để thoát khi xong:

```bash
kubectl -n lab-11a exec --stdin --tty shell-demo -- /bin/sh
```

Bên trong shell, thử vài lệnh của bài 304 rồi thoát:

```bash
# Chạy các lệnh này bên trong container
ls /
cat /proc/mounts | head -5
ps
exit
```

Gate không phụ thuộc vào phiên tương tác — nó kiểm chính đường `exec`:

```bash
test "$(kubectl -n lab-11a exec shell-demo -- hostname)" = 'shell-demo' \
  && echo 'PASS: duong exec toi container hoat dong'
kubectl -n lab-11a exec shell-demo -- sh -c 'test -d /www' \
  && echo 'PASS: nhin duoc filesystem ben trong container'
```

**Ý nghĩa:** `exec` đi qua apiserver → kubelet cổng `10250`, đúng đường mà
[tầng 6 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a547-tầng-6--đường-control-plane--kubelet) kiểm. Nó
hỏng thì `logs` và `port-forward` cũng hỏng, vì cả ba dùng chung một đường.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B11.2. Ghi trang gốc rồi tự gọi từ bên trong

Đây là mục *Ghi trang gốc cho nginx* của bài 304, chuyển sang `httpd` của busybox:

```bash
kubectl -n lab-11a exec shell-demo -- \
  sh -c 'echo "Hello shell demo" > /www/index.html'
kubectl -n lab-11a exec shell-demo -- wget -q -O- http://localhost:8080/ \
  | tee "$EV/b11-trang-goc.txt"

grep -qx 'Hello shell demo' "$EV/b11-trang-goc.txt" \
  && echo 'PASS: thay doi tu ben trong container co hieu luc ngay voi tien trinh dang chay'
```

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B11.3. Chạy lệnh rời, và vai trò của dấu `--`

```bash
kubectl -n lab-11a exec shell-demo -- env | tee "$EV/b11-env.txt"
kubectl -n lab-11a exec shell-demo -- ls /
kubectl -n lab-11a exec shell-demo -- cat /proc/1/mounts | head -3
```

Bỏ dấu `--` đi thì `kubectl` nuốt mất tùy chọn của lệnh bên trong:

```bash
kubectl -n lab-11a exec shell-demo ls -l / > "$EV/b11-thieu-gach.txt" 2>&1 \
  && echo 'FAIL: lenh thieu -- lai chay duoc' \
  || echo 'PASS: thieu -- thi kubectl hieu -l la co cua chinh no va bao loi'
head -2 "$EV/b11-thieu-gach.txt"

grep -q 'KUBERNETES_SERVICE_HOST' "$EV/b11-env.txt" \
  && echo 'PASS: chay duoc lenh roi va doc duoc bien moi truong cua container'
```

**Ý nghĩa:** dấu gạch ngang kép **phân tách đối số của lệnh bên trong container với đối số của
`kubectl`**. Thiếu nó, mọi cờ ngắn quen thuộc (`-l`, `-a`, `-n`) đều bị `kubectl` giành lấy — và
thông báo lỗi thường không nói thẳng ra nguyên nhân.

**PASS:** hai dòng `PASS:` của bước này xuất hiện, không dòng `FAIL:` nào.

### B11.4. Pod nhiều container thì phải chỉ đích danh

```bash
cat > "$WK/b11-hai-container.yaml" <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: two-app
  namespace: lab-11a
spec:
  nodeName: lab-k8s-worker1
  containers:
    - name: main-app
      image: busybox:1.37
      command: [sh, -c, 'echo main-app > /tmp/who; sleep 3600']
    - name: helper-app
      image: busybox:1.37
      command: [sh, -c, 'echo helper-app > /tmp/who; sleep 3600']
YAML

kubectl apply -f "$WK/b11-hai-container.yaml"
kubectl -n lab-11a wait --for=condition=Ready pod/two-app --timeout=180s
```

```bash
WHO_DEFAULT="$(kubectl -n lab-11a exec two-app -- cat /tmp/who 2>/dev/null)"
WHO_MAIN="$(kubectl -n lab-11a exec two-app -c main-app -- cat /tmp/who)"
WHO_HELP="$(kubectl -n lab-11a exec two-app -c helper-app -- cat /tmp/who)"
echo "khong -c => $WHO_DEFAULT | -c main-app => $WHO_MAIN | -c helper-app => $WHO_HELP"

test "$WHO_MAIN" = 'main-app' && test "$WHO_HELP" = 'helper-app' \
  && echo 'PASS: -c chon dung container, khong doan'
test "$WHO_DEFAULT" = "$WHO_MAIN" \
  && echo 'PASS: khong -c thi kubectl chon container dau tien — tien nhung de nham'
```

**Ý nghĩa:** `-c` (dạng dài `--container`) là bắt buộc về mặt thói quen ngay khi Pod có từ hai
container trở lên — kể cả khi kubectl vẫn chạy được nhờ chọn giúp bạn. Cùng quy tắc đó áp cho
`kubectl logs`, và B7.2 đã dùng tới nó.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B11.5. Ranh giới: chỗ giai đoạn 11 dừng

Bộ công cụ ở giai đoạn 11 chỉ gồm ba thứ: `kubectl logs`, `kubectl logs --previous`, `kubectl exec`.
Ghi lại ranh giới trước khi rời lab, để bạn không sa vào phần của giai đoạn sau:

```bash
{
  echo 'Cong cu DUOC dung o giai doan 11 : kubectl logs / kubectl logs --previous / kubectl exec'
  echo 'Thuoc giai doan 24 — KHONG lam o day:'
  echo '  - kubectl debug voi ephemeral container (bai 300, khai niem o bai 52)'
  echo '  - crictl khi API server khong tra loi (bai 307)'
  echo '  - quy trinh lan tu Service ve Pod (bai 301)'
  echo '  - xac dinh nguyen nhan Pod that bai qua termination message (bai 303)'
  echo 'Thuoc cong cu ben thu ba — KHONG chay tren chuoi snapshot:'
  echo '  - telepresence (bai 309): no cai sidecar traffic-agent vao Pod ung dung,'
  echo '    tuc sua workload tren cluster; cach thu cong tuong duong la chinh B11.1'
} | tee "$EV/b11-ranh-gioi.txt"

test "$(wc -l < "$EV/b11-ranh-gioi.txt")" -eq 9 \
  && echo 'PASS: da ghi ranh gioi cong cu go loi cua giai doan 11'
```

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B11.6. Dọn phần workload của B11

```bash
kubectl -n lab-11a delete -f "$WK/b11-shell-demo.yaml"    --wait=true --timeout=180s
kubectl -n lab-11a delete -f "$WK/b11-hai-container.yaml" --wait=true --timeout=180s
kubectl -n lab-11a get pods

test "$(kubectl -n lab-11a get pods --no-headers | grep -cvE '^counter ')" -eq 0 \
  && echo 'PASS: chi con Pod counter trong namespace lab-11a'
```

**PASS:** dòng `PASS:` của bước này xuất hiện.

---

## B12. Cleanup và gate tạo snapshot `04-metrics-ready`

**Mục đích:** xóa mọi object của bài học, **giữ lại đúng phần hạ tầng** mà mốc mới được định nghĩa
là có — metrics-server — chứng minh cấu hình node và manifest control plane không hề bị sửa, rồi
chứng minh cluster khỏe trước khi chụp.

Định nghĩa của mốc `04-metrics-ready` theo [chuỗi snapshot](README.md#3-chuỗi-snapshot): mọi thứ của
`03-storage-ready`, **cộng** metrics-server đang chạy trong `kube-system` và APIService
`v1beta1.metrics.k8s.io` ở trạng thái `Available`. Không workload, không PV, không PVC, không object
phạm vi cluster nào của lab.

### B12.1. Xóa object của bài học

```bash
kubectl delete namespace lab-11a --wait=true --timeout=300s
kubectl delete -f "$WK/b3-rbac.yaml" --ignore-not-found

rm -f "$WK/b3-rbac.yaml" "$WK/b4-web.yaml" \
      "$WK/b7-counter.yaml" "$WK/b7-sidecar.yaml" "$WK/b7-crash.yaml" "$WK/b7-agent.yaml" \
      "$WK/b8-flood.yaml" \
      "$WK/b11-shell-demo.yaml" "$WK/b11-hai-container.yaml" \
      "$WK/metrics-server.yaml"
rmdir "$WK"
```

`rmdir` cố ý không dùng `-rf`: nó fail nếu thư mục còn file lạ, và `test ! -e` bên dưới biến điều đó
thành gate thay vì im lặng bỏ qua. Thư mục `~/lab-evidence/11a/` **giữ lại** — đó là bằng chứng.

```bash
NS_LEFT="$(kubectl get namespace lab-11a --ignore-not-found -o name | wc -l)"
CR_LEFT="$(kubectl get clusterrole lab-11a-metrics-reader --ignore-not-found -o name | wc -l)"
CRB_LEFT="$(kubectl get clusterrolebinding lab-11a-metrics-reader --ignore-not-found -o name | wc -l)"
echo "namespace=$NS_LEFT clusterrole=$CR_LEFT clusterrolebinding=$CRB_LEFT"

test "$NS_LEFT" -eq 0 && echo 'PASS: namespace lab-11a da bien mat'
test "$CR_LEFT" -eq 0 && test "$CRB_LEFT" -eq 0 \
  && echo 'PASS: khong con object pham vi cluster nao cua lab'
test ! -e "$WK" && echo 'PASS: manifest tam da xoa'

kubectl auth can-i get /metrics --as=system:serviceaccount:lab-11a:scraper | grep -qx 'no' \
  && echo 'PASS: quyen doc /metrics cua lab da bi thu hoi'
```

**PASS:** đủ bốn dòng `PASS:` của bước này.

### B12.2. Gate quan trọng nhất về tính toàn vẹn: cấu hình node và control plane không đổi

```bash
{
  echo "$MASTER kubelet $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for n in "$W1" "$W2"; do
    echo "$n kubelet $(ssh "$n" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done
  for f in kube-apiserver kube-scheduler kube-controller-manager; do
    echo "$MASTER $f $(sudo sha256sum /etc/kubernetes/manifests/$f.yaml | awk '{print $1}')"
  done
} | tee "$EV/b12-config-sha.txt"

diff -u "$EV/b0-config-sha.txt" "$EV/b12-config-sha.txt" \
  && echo 'PASS: cau hinh kubelet va manifest control plane khong he doi trong suot lab' \
  || echo 'FAIL: co file cau hinh da bi sua — xem muc 4'

RS_OK=0
for n in $NODES; do
  A="$(kubectl get node "$n" -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')"
  test "$A" = 'True' && RS_OK=$(( RS_OK + 1 ))
done
test "$RS_OK" -eq 3 && echo 'PASS: ca ba kubelet van Ready sau khi lab ket thuc'
```

**Ý nghĩa:** B5.4 mô tả một cách sửa "đúng đắn hơn" đòi sửa cấu hình kubelet, B9.4 nhắc tới việc đổi
`-v` của control plane, và B10 dừng ngay trước cửa file cấu hình tracing. Ba cám dỗ đó đều nằm trong
sáu file này. Gate ở đây biến lời hứa "chỉ đọc" thành thứ kiểm chứng được — và quan trọng gấp đôi vì
lab sắp **chụp snapshot**: một sửa đổi lén lút sẽ đi vào `04-metrics-ready` rồi theo năm lab sau.

**PASS:** hai dòng `PASS:` của bước này xuất hiện, không dòng `FAIL:` nào.

### B12.3. Gate trạng thái của mốc mới

```bash
MS_READY="$(kubectl -n kube-system get deploy metrics-server -o jsonpath='{.status.readyReplicas}')"
MS_IMG="$(kubectl -n kube-system get deploy metrics-server \
  -o jsonpath='{.spec.template.spec.containers[0].image}')"
AV="$(kubectl get apiservice v1beta1.metrics.k8s.io \
  -o jsonpath='{.status.conditions[?(@.type=="Available")].status}')"
TN="$(kubectl top node --no-headers | wc -l)"
echo "metrics-server readyReplicas=${MS_READY:-0} | image=$MS_IMG"
echo "APIService Available=$AV | so dong kubectl top node=$TN"

test "${MS_READY:-0}" -ge 1 && test "$AV" = 'True' \
  && echo 'PASS: metrics-server dang chay va Metrics API san sang'
test "$TN" -eq 3 \
  && echo 'PASS: kubectl top node chay duoc — dieu kien dau vao cua Lab 11b da co'
kubectl top pod -n kube-system --no-headers | head -3
test "$(kubectl top pod -n kube-system --no-headers | wc -l)" -gt 0 \
  && echo 'PASS: kubectl top pod chay duoc'
```

Tầng lưu trữ phải trở về đúng định nghĩa của mốc trước, không được nhiễm gì từ lab này:

```bash
PV_ALL="$(kubectl get pv --no-headers 2>/dev/null | wc -l)"
PVC_ALL="$(kubectl get pvc -A --no-headers 2>/dev/null | wc -l)"
SC_ALL="$(kubectl get sc --no-headers | wc -l)"
SC_NOW="$(kubectl get sc -o jsonpath='{.items[0].metadata.name}')"
SC_DEF="$(kubectl get sc "$SC_NOW" \
  -o jsonpath='{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}')"
echo "pv=$PV_ALL pvc=$PVC_ALL storageclass=$SC_ALL ($SC_NOW, default=$SC_DEF)"

test "$PV_ALL" -eq 0 && test "$PVC_ALL" -eq 0 \
  && echo 'PASS: khong con PV hay PVC nao'
test "$SC_ALL" -eq 1 && test "$SC_DEF" = 'true' \
  && echo 'PASS: van dung mot StorageClass mac dinh nhu 03-storage-ready quy dinh'
```

**Ý nghĩa:** hai nhóm gate này là **định nghĩa của mốc mới ở dạng lệnh**. Nhóm thứ nhất bảo đảm bạn
**có** chụp kèm phần hạ tầng mà Lab 11b, 12, 13, 14 và 15 sẽ dựa vào; nhóm thứ hai bảo đảm bạn
**không** chụp kèm rác của bài học hay làm hỏng thứ Lab 6a để lại.

**PASS:** đủ năm dòng `PASS:` của bước này.

### B12.4. Chạy trọn bảy tầng gate của A5.4

Lab này đổi hạ tầng, nên **không** được dừng ở gate ngắn A5.5. Chạy **toàn bộ bảy tầng** của
[A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready) theo đúng thứ tự từ dưới lên, từ
[tầng 0](LAB-00-MOI-TRUONG-1.35.7.md#a541-tầng-0--prereq-os-còn-đúng-sau-khi-cluster-chạy) tới
[tầng 6](LAB-00-MOI-TRUONG-1.35.7.md#a547-tầng-6--đường-control-plane--kubelet), rồi dọn resource
test theo [A5.4.8](LAB-00-MOI-TRUONG-1.35.7.md#a548-dọn-resource-test-ghi-evidence-và-chụp-snapshot).

| Tầng | Nội dung | Điều chỉnh cho mốc `04-metrics-ready` |
| --- | --- | --- |
| 0 | prereq OS trên cả ba node | không đổi |
| 1 | control plane khỏe thật | không đổi |
| 2 | node, condition, taint và PodCIDR | Không còn DaemonSet `kube-flannel-ds` và không còn namespace `kube-flannel`: đọc theo CNI do Lab 5b cài, đúng như [bảng điều chỉnh ở B11.2 của Lab 5b](LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md#b112-chạy-trọn-bảy-tầng-gate-a54). Cột `Memory Requests` giờ có thêm phần của metrics-server — vẫn phải dưới ngưỡng của tầng này |
| 3 | Pod networking xuyên node | Đường hầm giữa các node do CNI của Lab 5b dựng, không còn là VXLAN cổng `8472` của Flannel; ba nguyên nhân fail đọc lại theo bảng của Lab 5b |
| 4 | DNS trong cluster và ra Internet | không đổi |
| 5 | Service, EndpointSlice và kube-proxy | không đổi |
| 6 | đường control plane → kubelet | Quan trọng gấp đôi ở mốc này: metrics-server đi đúng đường này để gọi kubelet, nên tầng 6 hỏng là `kubectl top` chết theo |

Khi đối chiếu danh sách Pod, tập namespace hệ thống hợp lệ ở mốc này là: `kube-system` (nay **có
thêm** metrics-server), namespace của CNI và của ingress controller do Lab 5b cài, cộng
`local-path-storage` do Lab 6a cài.

```bash
kubectl get pods -A -o wide | tee "$EV/b12-pods.txt"
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl get pods -n default
kubectl get svc -n default

grep -q 'metrics-server' "$EV/b12-pods.txt" \
  && echo 'PASS: metrics-server nam trong danh sach Pod se duoc chup'
grep -qi 'lab-11a' "$EV/b12-pods.txt" \
  && echo 'FAIL: con Pod cua lab trong danh sach' \
  || echo 'PASS: khong con Pod nao cua lab 11a'
```

**PASS:** bảy tầng của A5.4 đều PASS; lệnh field selector trả `No resources found`; `default` không
có Pod và chỉ còn Service `kubernetes`; hai dòng `PASS:` ở trên xuất hiện, không dòng `FAIL:` nào.

### B12.5. Ghi hồ sơ của mốc mới

```bash
{
  date -Is
  echo '=== 04-metrics-ready ==='
  echo '--- metrics-server'
  kubectl -n kube-system get deploy metrics-server -o wide
  kubectl -n kube-system get deploy metrics-server \
    -o jsonpath='{range .spec.template.spec.containers[0].args[*]}{@}{"\n"}{end}'
  echo '--- Metrics API'
  kubectl get apiservice v1beta1.metrics.k8s.io -o wide
  kubectl api-resources --api-group=metrics.k8s.io
  echo '--- kubectl top'
  kubectl top node
  echo '--- tang luu tru (thua ke tu 03-storage-ready)'
  kubectl get sc -o wide
  kubectl get pv
  echo '--- namespaces'
  kubectl get namespaces
} | tee "$EV/b12-ho-so-04-metrics-ready.txt"

test -s "$EV/b12-ho-so-04-metrics-ready.txt" \
  && grep -q 'metrics-server' "$EV/b12-ho-so-04-metrics-ready.txt" \
  && echo 'PASS: da ghi ho so cua moc 04-metrics-ready'

diff -u "$EV/b0-truoc.txt" "$EV/b12-ho-so-04-metrics-ready.txt" \
  > "$EV/b12-diff.txt" 2>&1 || true
```

**PASS:** dòng `PASS:` của bước này xuất hiện. File `b12-diff.txt` là bản ghi **chính xác thứ lab
này đã thêm vào cluster** — đọc lại nó một lượt trước khi chụp; nếu thấy gì ngoài metrics-server,
dừng lại và tìm nguyên nhân.

### B12.6. Tắt máy và chụp `04-metrics-ready`

Chụp khi VM đã tắt để snapshot không kèm trạng thái RAM — cùng lý do như A5.4.8 của Lab 00. Chạy
trên **từng node** theo thứ tự worker 2 → worker 1 → master:

```bash
sudo shutdown -h now
```

Chờ VMware Workstation hiển thị cả ba VM ở trạng thái *Powered off*. Chụp trên **cả ba VM**: chuột
phải VM → **Snapshot → Take Snapshot** → ô *Name* điền đúng nguyên văn:

```text
04-metrics-ready
```

Ô *Description* ghi lab đã dựng mốc này và ngày chụp, ví dụ
`dựng bằng LAB-11A-OBSERVABILITY.md, chụp <ngày>`.

Quy tắc tên giống hệt Lab 00 và Lab 6a: đúng nguyên văn `04-metrics-ready` trên cả ba VM, không hậu
tố theo VM, không thêm ngày, không đổi hoa thường. **Giữ nguyên snapshot `03-storage-ready`** —
đừng xóa nó, đó vẫn là điểm quay lui của mốc này và là điểm bắt đầu của các lab 8a, 9a, 9b chưa
viết.

Verify từ PowerShell trên máy host:

```powershell
$vmrun = 'C:\Program Files\VMware\VMware Workstation\vmrun.exe'
$vmx = @(
  'E:\Virtual Machines\lab-k8s-master\lab-k8s-master.vmx'
  'E:\Virtual Machines\lab-k8s-worker1\lab-k8s-worker1.vmx'
  'E:\Virtual Machines\lab-k8s-worker2\lab-k8s-worker2.vmx'
)
foreach ($f in $vmx) {
  $names = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  if (($names -ccontains '04-metrics-ready') -and ($names -ccontains '03-storage-ready')) {
    "PASS: $f"
  } else {
    "FAIL: $f -> $($names -join ', ')"
  }
}
```

**PASS:** đúng ba dòng `PASS:`, không dòng `FAIL:` nào — cả ba VM có **cả hai** mốc, tên phân biệt
hoa thường chính xác. Lab 11a kết thúc ở đây; để ba VM ở trạng thái tắt, lab sau tự bật theo
[A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab).

---

## 3. Checkpoint 11a

Không đánh dấu lab hoàn tất chỉ vì command chạy hết. Tự trả lời, không nhìn tài liệu:

- [ ] Kể tên ba trụ cột quan sát và cho mỗi trụ cột một câu hỏi vận hành mà **chỉ** nó trả lời
      được. Trên cluster của bạn ngay sau khi `kubeadm init`, trụ cột nào đã có nguồn phát sẵn và
      trụ cột nào chưa? Bạn chứng minh điều đó bằng lệnh gì?
- [ ] Bạn cần biết một Pod trên `lab-k8s-worker2` đang ngốn bao nhiêu bộ nhớ, và cần biết một
      Deployment đang có mấy replica sẵn sàng. Hai con số đó đến từ hai nguồn khác nhau — nguồn
      nào, vì sao, và trên cluster lab thì con số thứ hai **có ở dạng metric** không? Bạn đã chứng
      minh bằng bước nào?
- [ ] kubelet expose bốn endpoint metric. Kể đủ bốn, nói mỗi cái phục vụ gì, và giải thích vì sao
      bài 160 nhấn mạnh chúng **không cùng vòng đời**. Điều đó ảnh hưởng thế nào tới một dashboard
      bạn sắp dựng?
- [ ] Một scraper đã thông mạng tới endpoint `/metrics` của kube-scheduler và cầm token của một
      ServiceAccount hợp lệ, nhưng vẫn không lấy được metric. Chuyện gì đang xảy ra, bạn sửa bằng
      object nào, và vì sao object đó **không** đồng thời mở luôn `/metrics/resource`?
- [ ] **`kubectl top node` trên cluster của bạn chạy được chưa?** Chuỗi phụ thuộc từ lệnh đó ngược
      về nguồn số liệu gồm những mắt xích nào? Nếu nó hỏng, bạn đọc log của thành phần nào **đầu
      tiên**, và vì sao không bắt đầu từ `kubectl top`?
- [ ] Bạn vừa cài metrics-server bằng manifest thượng nguồn trên một cluster kubeadm và nó không
      `Ready`. Nguyên nhân gốc là gì? Hai đường sửa là gì, mỗi đường đánh đổi cái gì, và lab đã
      chọn đường nào — vì sao đường còn lại không làm được ở đây?
- [ ] metrics-server đang chạy. Nêu **ba** câu hỏi giám sát mà nó **không** trả lời được, và với
      mỗi câu nói rõ thành phần nào mới trả lời được. Bạn đã chứng minh giới hạn "không lưu lịch
      sử" bằng cách nào?
- [ ] Một Pod có ba container: một ghi vào hai file trong `emptyDir`, hai container kia `tail` từng
      file ra `stdout` của chúng. `kubectl logs` cho từng container trả về gì? Yếu tố **duy nhất**
      quyết định điều đó là gì? Suy ra: sidecar chạy agent ghi log thì `kubectl logs` thấy gì?
- [ ] Một Pod trên `lab-k8s-worker2` đã in ra 40 000 dòng log. `kubectl logs` trả về nhiều nhất bao
      nhiêu, con số đó do ai quyết định, và phần còn lại đang ở đâu? Bạn xóa Pod thì phần đó ra
      sao? Muốn giữ được thì cần thêm gì?
- [ ] Trên `lab-k8s-worker1`, bạn tìm log của kubelet ở đâu và log của kube-scheduler ở đâu? Vì sao
      hai thành phần của cùng một cluster lại nằm hai chỗ khác nhau, và vì sao tìm
      `/var/log/kubelet.log` là hướng sai?
- [ ] Bạn dựng một agent ghi log dạng `DaemonSet`. Agent trên `lab-k8s-worker1` **thấy** gì và
      **không thấy** gì? Vì sao agent trên `lab-k8s-master` cần thêm một thứ mà hai agent kia không
      cần? Phần nào của kiến trúc ghi log cấp cluster vẫn còn thiếu sau bước đó?
- [ ] Cluster của bạn có đang bật tracing không? Bạn kiểm ở **ba** chỗ nào để chắc chắn? Bật nó lên
      sẽ phải sửa những file nào, thuộc giai đoạn nào, và vì sao lab này cố ý không làm?

### Bài giải thích cuối cùng

Trong vài phút, kể lại bằng lời hai luồng sau, không nhìn tài liệu:

1. **Luồng một con số đi từ container tới màn hình `kubectl top`.** Bắt đầu từ một container đang
   chạy trên `lab-k8s-worker2`. Kể đủ: ai đo, số liệu đó xuất hiện ở endpoint nào của kubelet, ai
   gọi endpoint đó và bằng quyền gì, kết quả được đăng ký vào Kubernetes API dưới hình thức nào, và
   `kubectl top` gọi cái gì. Ở mỗi mắt xích, nói ra **một cách nó có thể hỏng** và triệu chứng
   tương ứng. Rồi kể tiếp: cùng con số đó, nếu bạn muốn xem lại vào tuần sau thì thiếu mắt xích
   nào, và vì sao mắt xích đó không nằm trong lab này.
2. **Luồng một dòng log đi từ `echo` tới chỗ bạn đọc được nó.** Bắt đầu từ lệnh `echo` trong
   container. Kể đủ: ai bắt lấy nó, nó được chuẩn hóa theo cái gì, nằm ở đường dẫn nào trên node,
   ai đọc file đó khi bạn gõ `kubectl logs`, và ba ranh giới nó không vượt qua được — file đã xoay
   vòng, Pod bị xóa, node chết. Sau đó chuyển sang log của **chính Kubernetes**: hai loại thành
   phần, hai vị trí, và vì sao không có cái nào trong hai loại đó nằm ở nơi bạn hay đoán đầu tiên.
   Cuối cùng, nói rõ **kiến trúc nào** lấp được cả ba ranh giới ở trên, và những gì thuộc về
   Kubernetes trong kiến trúc đó so với những gì không.

Khi mọi checkbox được đánh dấu và bạn không còn nhầm **metric tài nguyên** với **metric trạng thái
đối tượng**, **metrics-server** với **Prometheus**, **sidecar truyền luồng** với **sidecar chạy
agent**, hay **journald** với **`/var/log/pods`** — Lab 11a và
[giai đoạn 11](../00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability) hoàn tất.

Lab này **không phát sinh nợ mới** và **không trả nợ nào**, nhưng nó **mở khóa** cho lab kế tiếp:
[nợ #1](README.md#5-sổ-nợ-lab) — thực hành HPA và VPA, phát sinh ở
[giai đoạn 4](../00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller) tại bài
[72](../72-horizontal-pod-autoscale-vi.md) và [73](../73-vertical-pod-autoscale-vi.md) — **vẫn chưa
được trả**. Nó chỉ được trả ở **Lab 11b**, trên chính mốc `04-metrics-ready` mà bạn vừa chụp, và
điều kiện đầu vào của nó là gate `kubectl top` ở [B12.3](#b123-gate-trạng-thái-của-mốc-mới). **Đọc
lại hai bài 72 và 73 trước khi mở Lab 11b.**

Những phần cố ý không làm trong lab này đều nằm trong bảng lý do ở
[mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành), và chúng thuộc đúng thứ tự lộ trình đã định —
[giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy),
[23](../00-ALO-TRINH-ADMIN.md#giai-đoạn-23--giám-sát-và-cảnh-báo) và
[24](../00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố) — chứ không phải nợ.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường nằm ở
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường). Bảng dưới chỉ liệt kê
sự cố phát sinh từ nội dung bài học 11a.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| **B5.3: metrics-server không `Ready`, log có `x509`** | `kubectl -n kube-system logs deploy/metrics-server --tail=40` | **Đây là hành vi đúng của bước này, không phải sự cố.** Certificate phục vụ của kubelet là tự ký nên metrics-server từ chối kết nối. Đọc B5.4 rồi vá theo B5.5. Đừng bỏ qua B5.3 để "cho nhanh" — chẩn đoán được từ log là phần học của B5 |
| B5.5: đã vá mà vẫn không `Ready`, log **không** có `x509` | `kubectl -n kube-system describe pod -l k8s-app=metrics-server`; `kubectl -n kube-system logs deploy/metrics-server --tail=60` | Nguyên nhân khác certificate. Nếu log báo không tới được địa chỉ node, kiểm `--kubelet-preferred-address-types` và đối chiếu với `.status.addresses` của Node. Nếu Pod `ImagePullBackOff`, xem dòng ngay dưới |
| B5.2 hoặc B5.3: `ImagePullBackOff` cho image metrics-server | `kubectl -n kube-system describe pod -l k8s-app=metrics-server \| tail -20`; `kubectl exec` một Pod busybox rồi `nslookup registry.k8s.io` | Đây là image **duy nhất** lab kéo từ mạng. Kiểm DNS ra ngoài bằng [tầng 4 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a545-tầng-4--dns-trong-cluster-và-ra-internet). Nếu `MS_VERSION` sai thì tag không tồn tại — sửa ở **bảng A1.4 của Lab 00** trước, rồi tải lại manifest |
| B5.2: gate image fail | `grep image "$WK/metrics-server.yaml"` | Bạn tải nhầm release hoặc điền `MS_VERSION` khác A1.4. `MS_VERSION` phải khớp bảng A1.4; sửa ở bảng đó trước, rồi tải lại. Không sửa tay tag trong manifest |
| B5.5: `patch` báo lỗi đường dẫn `args` | `kubectl -n kube-system get deploy metrics-server -o jsonpath='{.spec.template.spec.containers[0].args}'` | Manifest của release đó đặt cờ ở `command` thay vì `args`. Đọc lại field thật rồi đổi `path` trong lệnh `patch` cho khớp. **Không** sửa file manifest rồi apply lại — làm thế thì mất dấu vết bước hỏng của B5.3 |
| B5.5: `--kubelet-insecure-tls` xuất hiện **hai lần** | `kubectl -n kube-system get deploy metrics-server -o jsonpath='{range .spec.template.spec.containers[0].args[*]}{@}{"\n"}{end}'` | Bạn chạy lệnh `patch` hai lần; thao tác `add` với `args/-` luôn nối thêm. Xóa bớt bằng `kubectl -n kube-system edit deploy metrics-server` rồi chạy lại gate B5.5 |
| B6.1: `kubectl top node` vẫn báo lỗi dù APIService `Available=True` | `kubectl get --raw '/apis/metrics.k8s.io/v1beta1/nodes'` | metrics-server đã sẵn sàng nhưng chưa hoàn thành chu kỳ thu thập đầu tiên — thời gian **phụ thuộc cấu hình**. Chạy lại vòng lặp chờ ở B6.1. Nếu sau vài vòng vẫn rỗng, quay lại log ở B5.5 |
| B6.1: `kubectl top node` thiếu một node | `kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes"`; `kubectl -n kube-system logs deploy/metrics-server \| grep <ten node>` | metrics-server không gọi được kubelet của đúng node đó. Chạy lại [tầng 6 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a547-tầng-6--đường-control-plane--kubelet) cho node đó; nếu tầng 6 fail thì `logs`/`exec` cũng đang hỏng và đó mới là gốc |
| B2.3: `/metrics/probes` rỗng hoặc không có `prober_probe_total` | `kubectl -n kube-system get pod "kube-apiserver-$MASTER" -o jsonpath='{.spec.containers[0].livenessProbe}'` | Endpoint chỉ phát một họ metric khi họ đó đã có ít nhất một mẫu, tức đã có probe chạy. Chạy lại lệnh trên `lab-k8s-master`, nơi bốn static Pod control plane đều có liveness probe. Nếu vẫn rỗng, ghi lại vào evidence và đi tiếp — ba endpoint còn lại đủ cho B4 |
| B0.3 hoặc B9.5: `configz` không đọc được trên một node | `kubectl get --raw "/api/v1/nodes/<node>/proxy/healthz"`; [tầng 6 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a547-tầng-6--đường-control-plane--kubelet) | Đường control plane → kubelet đang hỏng, và nó ảnh hưởng cả `logs`/`exec` lẫn metrics-server. Chạy lại tầng 6; đừng đọc vòng bằng cách sửa gì trên node. Nếu vẫn hỏng, restore cả ba VM về `03-storage-ready` |
| B0.3: `cfgz` trả về rỗng cho một khóa | `grep -o '"<ten khoa>":[^,}]*' ~/lab-evidence/11a/b0-configz-<node>.json` | Khóa đó không có trong cấu hình hiệu lực (kubelet không khai và cũng không có mặc định hiển thị). Ghi `<khong khai>` vào evidence và đọc phần *Ý nghĩa* theo giá trị mặc định của phiên bản, **không** thêm khóa vào file cấu hình để "cho có" |
| B3.2: `curl` tới kube-scheduler báo `Connection refused` | In lại `SCHED_BIND` và `SCHED_PORT` ở B3.1; `ssh $MASTER 'ss -lnt \| grep 1025'` | Bạn đang chạy `curl` từ máy khác `lab-k8s-master` trong khi `--bind-address` là `127.0.0.1`. Chạy đúng trên master. **Không** đổi cờ `--bind-address` để "cho tiện" — đó là sửa control plane, và B12.2 sẽ bắt được |
| B3.2: mã trả về là `401` chứ không phải `403` | `kubectl -n lab-11a get serviceaccount scraper`; sinh lại token | Token đã hết hạn hoặc lấy từ ServiceAccount khác. Sinh lại bằng `kubectl -n lab-11a create token scraper` ngay trước khi `curl`. Gate của B3.2 chỉ đòi **khác `200`**, nên `401` vẫn PASS, nhưng bạn nên biết mình đang chẩn đoán cái gì |
| B3.3: `can-i` trả `yes` mà `curl` vẫn không `200` | `kubectl get clusterrolebinding lab-11a-metrics-reader -o yaml` | RBAC đã đúng ở API server nhưng token dùng cho `curl` là token cũ sinh trước khi bind. Sinh token mới rồi gọi lại |
| B7.3: `restartCount` mãi bằng 0 | `kubectl -n lab-11a describe pod crash-once \| tail -20` | Pod chưa kịp chạy hết vòng đời đầu tiên, hoặc bị `ImagePullBackOff`. Chờ hết vòng lặp trong bước đó; nếu là lỗi kéo image thì `busybox:1.37` chưa có trên `lab-k8s-worker2` — xem [tầng 4 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a545-tầng-4--dns-trong-cluster-và-ra-internet) |
| B7.4: không tìm thấy thư mục trong `/var/log/pods` | `kubectl -n lab-11a get pod counter -o jsonpath='{.spec.nodeName}{"\n"}{.metadata.uid}'` | Bạn đang `ssh` sang nhầm node, hoặc UID đọc sai. Đường dẫn có dạng `/var/log/pods/<namespace>_<pod>_<uid>/<container>/`. Nếu kubelet của node đó dùng `podLogsDir` khác mặc định, đọc giá trị thật bằng `cfgz podLogsDir` rồi thay vào |
| B7.5: DaemonSet chỉ có hai Pod | `kubectl -n lab-11a describe ds log-agent-demo \| tail -20`; `kubectl get node $MASTER -o jsonpath='{.spec.taints}'` | Toleration không khớp taint thật của control plane. Đọc taint thật rồi sửa `key`/`effect` trong manifest cho khớp. **Không** gỡ taint của master — làm thế là đổi hành vi scheduling của mọi lab sau |
| B7.5: agent trên worker báo `apiserver=` khác 0 | `ssh <worker> 'sudo ls /var/log/containers \| head'` | Node đó thật sự đang chạy một container tên bắt đầu bằng `kube-apiserver`, tức cluster của bạn không ở topology của Lab 00. Dừng lại và đối chiếu [tầng 2 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a543-tầng-2--node-condition-taint-và-podcidr) |
| B8.3: chỉ có đúng một file trong thư mục log của `log-flood` | `cfgz containerLogMaxSize ~/lab-evidence/11a/b0-configz-$W2.json`; `ssh $W2 "sudo du -sh $LOGDIR"` | Khối log chưa vượt ngưỡng, hoặc kubelet chưa tới chu kỳ giám sát. Chờ thêm vòng lặp ở B8.3; nếu `containerLogMaxSize` của node bạn lớn hơn baseline thì tăng số vòng lặp trong manifest B8.2 (`40000` → lớn hơn), **không** sửa cấu hình kubelet |
| B8.3: gate byte fail vì `kubectl logs` trả nhiều hơn `MAXB` | In lại `CLMS`, `MAXB` và `GOT_B` | `to_bytes` gặp đơn vị chưa có nhánh (ví dụ `M` thay `Mi`). Bổ sung nhánh tương ứng vào hàm **trong phiên shell**, không sửa cluster |
| B8.4: thư mục log vẫn còn sau khi xóa Pod | `kubectl -n lab-11a get pod log-flood`; `ssh $W2 "sudo ls -l $LOGDIR"` | Pod còn trong grace period hoặc container chưa bị dọn. Chờ hết vòng lặp; **không** `sudo rm -rf` thư mục trong `/var/log/pods` — đó là vùng kubelet quản lý |
| B9.1: `journalctl -u kubelet` không có dòng nào | `ssh $W1 'systemctl is-active kubelet'`; `ssh $W1 'sudo journalctl --disk-usage'` | Nếu kubelet đang `active` mà journal rỗng thì journald đã bị giới hạn dung lượng và xoay hết. Đọc `journalctl -u kubelet --since '-1h'`; nếu vẫn rỗng, ghi vào evidence và đi tiếp — gate thứ hai của B9.1 (không có `/var/log/kubelet.log`) vẫn kiểm chứng được ranh giới của bài |
| B9.4: `N6` không lớn hơn `N2` | `wc -l ~/lab-evidence/11a/b9-v*.txt` | Bạn ghi nhầm `stdout` thay vì `stderr`. klog **luôn ghi ra stderr**; lệnh trong bài dùng `2>` đúng chỗ, chạy lại nguyên khối |
| B10.1: có cờ `--tracing-config-file` trên apiserver | `sudo grep -n tracing /etc/kubernetes/manifests/kube-apiserver.yaml` | Cluster của bạn đã bật tracing từ trước, tức lệch baseline. **Không gỡ cờ để "đưa về chuẩn"** — ghi giá trị đọc được vào evidence, và đọc lại phần *Ý nghĩa* của B10 theo cấu hình thật. Việc đưa ba node và control plane về cùng một cấu hình là nội dung của giai đoạn 20 |
| B12.1: namespace `lab-11a` kẹt `Terminating` | `kubectl get all -n lab-11a`; `kubectl -n lab-11a get pods` | Thường là Pod còn trong grace period. Chờ; **không** cưỡng chế finalizer của Namespace. Nếu quá lâu, kiểm `kubectl -n lab-11a describe pod` xem có Pod nào bị kẹt ở `Terminating` vì node không phản hồi |
| **B12.2: `diff` báo checksum khác** | `diff -u ~/lab-evidence/11a/b0-config-sha.txt ~/lab-evidence/11a/b12-config-sha.txt` | Một file cấu hình đã bị sửa trong lúc chạy lab. **Tuyệt đối không chụp snapshot** — mốc `04-metrics-ready` sẽ mang theo sai lệch đó vào năm lab sau. Tắt cả ba VM, restore cả ba về `03-storage-ready`, và làm lại từ B0 |
| B12.3: `kubectl top` fail ở gate cuối dù B6 đã PASS | `kubectl -n kube-system get pods -l k8s-app=metrics-server -o wide` | Pod metrics-server bị lập lịch lại sau khi namespace `lab-11a` bị xóa và chưa kịp thu thập xong. Chờ rồi chạy lại B12.3. **Không chụp snapshot khi gate này chưa PASS** — Lab 11b sẽ không mở được |
| B12.3: còn PV sót lại | `kubectl get pv -o wide`; log của provisioner | Lab 11a không tạo PV nào, nên đây là dấu vết còn lại từ lab trước. Xử lý theo [mục 4 của Lab 6a](LAB-6A-PV-PVC-VA-STORAGECLASS.md#4-troubleshooting-của-lab-này) trước khi chụp — cả `03-storage-ready` lẫn `04-metrics-ready` đều được định nghĩa là **không có PV nào** |

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
- [Kubernetes v1.35 — Resource metrics pipeline](https://v1-35.docs.kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)
  — nguồn của mô tả metrics-server và Metrics API dùng ở B5–B6; bài dịch tương ứng
  ([311](../311-resource-metrics-pipeline-vi.md)) thuộc [giai đoạn 23](../00-ALO-TRINH-ADMIN.md#giai-đoạn-23--giám-sát-và-cảnh-báo),
  đọc sau lab này
- [Kubernetes v1.35 — Kubelet Configuration (v1beta1)](https://v1-35.docs.kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)
  — nguồn của `containerLogMaxSize`, `containerLogMaxFiles`, `podLogsDir`, `enableSystemLogHandler`,
  `tracing` dùng ở B8, B9 và B10
- [Kubernetes v1.35 — Using RBAC Authorization](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/rbac/)
  — nguồn của `nonResourceURLs` dùng ở B3
- [Kubernetes v1.35 — Kubelet authentication/authorization](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/)
  — nguồn của cảnh báo về quyền `nodes/proxy` nhắc ở B9.5
- [metrics-server — README và bảng tương thích](https://github.com/kubernetes-sigs/metrics-server#compatibility-matrix)
  (nguồn để chốt dòng `metrics-server` cho bảng A1.4, dùng ở B5.1)
- [metrics-server — FAQ, mục về certificate của kubelet](https://github.com/kubernetes-sigs/metrics-server/blob/master/FAQ.md)
  (nguồn của hai đường sửa ở B5.4)

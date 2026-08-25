# Giám sát sức khỏe của Node (Monitor Node Health)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/debug-cluster/monitor-node-health/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 23 — Giám sát và cảnh báo](00-ALO-TRINH-ADMIN.md#giai-đoạn-23--giám-sát-và-cảnh-báo),
bài 3/3 · Giai đoạn này của Phần II không có lab riêng: thực hành trực tiếp trên cluster lab ở mốc
`04-metrics-ready` (xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)), tự chấm bằng Checkpoint ghi ở
cuối mục giai đoạn trong lộ trình.

Đây là **bài cuối giai đoạn 23** và là bài duy nhất trong nhóm có phần **phải cài thêm thật**:
metrics-server thì bạn đã có từ [Lab 11a](labs/LAB-11A-OBSERVABILITY.md), còn Node Problem
Detector thì chưa. Đọc xong bài này là đủ để làm nốt vế thứ ba của Checkpoint — cài
node-problem-detector rồi tạo ra một điều kiện node bất thường trên `lab-k8s-worker2`.

Lưu ý khi đọc: manifest ví dụ trong bài ghi image `registry.k8s.io/node-problem-detector:v0.1`
còn các link cấu hình lại trỏ tag `v0.8.12` — bản dịch giữ nguyên như trang gốc. Bài nói rõ ngay
đoạn mở đầu rằng cách cài và dùng đầy đủ nằm ở tài liệu của chính dự án Node Problem Detector.

**Phải hiểu ở lần đọc này:**

- Node Problem Detector là **một daemon ngoài Kubernetes** chạy dưới dạng `DaemonSet` hoặc daemon
  độc lập (đoạn mở đầu). Việc của nó là **thu thập** thông tin vấn đề node từ nhiều daemon khác
  rồi **báo cáo** lên API server. Nó không tự sửa gì.
- Mục *Exporter*, phần Kubernetes exporter: **vấn đề tạm thời → Event; vấn đề vĩnh viễn → Node
  Condition**. Đây chính là "điều kiện node bất thường" mà Checkpoint yêu cầu bạn tạo ra, và là lý
  do phải biết mình đang chờ nhìn thấy gì trong `kubectl describe node`.
- Bốn loại **problem daemon** ở mục *Các Problem Daemon* và việc chúng dùng để bắt loại sự cố nào:
  `SystemLogMonitor` (đọc log hệ thống theo quy tắc định sẵn), `SystemStatsMonitor` (thống kê hệ
  thống thành metrics), `CustomPluginMonitor` (chạy script bạn tự viết), `HealthChecker` (kiểm tra
  sức khỏe của kubelet và container runtime).
- Mục *Ghi đè cấu hình*: cấu hình mặc định **nhúng sẵn trong image**. Muốn đổi thì tạo ConfigMap
  `node-problem-detector-config` từ thư mục `config/` rồi mount đè lên `/config` trong DaemonSet.
  Và ràng buộc quan trọng: cách này **chỉ áp dụng cho bản khởi động bằng `kubectl`** — chạy dưới
  dạng Addon của cluster thì không ghi đè được, vì addon manager không hỗ trợ `ConfigMap`.
- Mục *Hạn chế* và *Khuyến nghị và giới hạn*: NPD dựa vào **định dạng log của kernel** để báo cáo
  vấn đề kernel, muốn định dạng khác thì phải viết log watcher mới. Đổi lại, chi phí chạy nó trên
  mỗi node là chấp nhận được vì log kernel tăng chậm và bản thân NPD đã bị đặt resource limit.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Dùng Addon pod để bật Node Problem Detector* (`/etc/kubernetes/addons/...`) | dành cho giải pháp bootstrap tùy biến có addon manager; cluster lab dựng bằng kubeadm nên đi nhánh `kubectl` | không dùng — Checkpoint [giai đoạn 23](00-ALO-TRINH-ADMIN.md#giai-đoạn-23--giám-sát-và-cảnh-báo) chạy nhánh [Dùng kubectl](#using-kubectl) |
| Prometheus exporter và Stackdriver exporter ở mục *Exporter* | cluster lab không có hệ giám sát nào để nhận, cũng không dùng dịch vụ cloud | pipeline metrics đầy đủ ở bài [312](312-resource-usage-monitoring-vi.md#full-metrics-pipeline) vừa đọc (bài 2/3) |
| Viết custom plugin monitor mới và thêm log watcher cho định dạng log khác | là việc lập trình mở rộng NPD theo giao thức exit code và stdout, không phải thao tác vận hành | không thuộc lộ trình — quay lại khi cần luật phát hiện riêng cho cluster thật |
| Các con số resource `requests`/`limits` trong hai manifest ví dụ | chỉ là giá trị mẫu của trang gốc | cách đọc và đặt `requests`/`limits` đã học ở bài [110](110-manage-resources-containers-vi.md), [giai đoạn 3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn) |

---

*Node Problem Detector* là một daemon dùng để giám sát và báo cáo về sức khỏe (health) của một
node. Bạn có thể chạy Node Problem Detector dưới dạng `DaemonSet` hoặc dưới dạng một daemon độc
lập. Node Problem Detector thu thập thông tin về các vấn đề của node từ nhiều daemon khác nhau
và báo cáo các tình trạng này lên API server dưới dạng
[Condition](https://kubernetes.io/docs/concepts/architecture/nodes#condition) của Node hoặc
dưới dạng [Event](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/event-v1).

Để tìm hiểu cách cài đặt và sử dụng Node Problem Detector, xem
[tài liệu của dự án Node Problem Detector](https://github.com/kubernetes/node-problem-detector).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Hạn chế (Limitations)

* Node Problem Detector sử dụng định dạng log của kernel để báo cáo các vấn đề về kernel.
  Để tìm hiểu cách mở rộng hỗ trợ cho định dạng log khác của kernel, xem
  [Thêm hỗ trợ cho định dạng log khác](#support-other-log-format).

## Bật Node Problem Detector (Enabling Node Problem Detector)

Một số nhà cung cấp cloud bật sẵn Node Problem Detector dưới dạng một Addon.
Bạn cũng có thể bật Node Problem Detector bằng `kubectl` hoặc bằng cách tạo một Addon DaemonSet.

### Dùng kubectl để bật Node Problem Detector (Using kubectl to enable Node Problem Detector) {#using-kubectl}

`kubectl` cung cấp cách quản lý Node Problem Detector linh hoạt nhất.
Bạn có thể ghi đè cấu hình mặc định để phù hợp với môi trường của mình hoặc
để phát hiện các vấn đề node tùy biến. Ví dụ:

1. Tạo một cấu hình Node Problem Detector tương tự như `node-problem-detector.yaml`:

   ```yaml
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     name: node-problem-detector-v0.1
     namespace: kube-system
     labels:
       k8s-app: node-problem-detector
       version: v0.1
       kubernetes.io/cluster-service: "true"
   spec:
     selector:
       matchLabels:
         k8s-app: node-problem-detector  
         version: v0.1
         kubernetes.io/cluster-service: "true"
     template:
       metadata:
         labels:
           k8s-app: node-problem-detector
           version: v0.1
           kubernetes.io/cluster-service: "true"
       spec:
         hostNetwork: true
         containers:
         - name: node-problem-detector
           image: registry.k8s.io/node-problem-detector:v0.1
           securityContext:
             privileged: true
           resources:
             limits:
               cpu: "200m"
               memory: "100Mi"
             requests:
               cpu: "20m"
               memory: "20Mi"
           volumeMounts:
           - name: log
             mountPath: /log
             readOnly: true
         volumes:
         - name: log
           hostPath:
             path: /var/log/
   ```

   > **Ghi chú:** Bạn nên kiểm tra để chắc chắn rằng thư mục log hệ thống là đúng với bản phân
   > phối hệ điều hành của bạn.

1. Khởi động node problem detector bằng `kubectl`:

   ```shell
   kubectl apply -f https://k8s.io/examples/debug/node-problem-detector.yaml
   ```

### Dùng Addon pod để bật Node Problem Detector (Using an Addon pod to enable Node Problem Detector) {#using-addon-pod}

Nếu bạn đang sử dụng một giải pháp khởi tạo (bootstrap) cluster tùy biến và không cần
ghi đè cấu hình mặc định, bạn có thể tận dụng Addon pod để tự động hóa việc triển khai
hơn nữa.

Tạo file `node-problem-detector.yaml`, và lưu cấu hình này vào thư mục Addon pod
`/etc/kubernetes/addons/node-problem-detector` trên một node control plane.

## Ghi đè cấu hình (Overwrite the configuration)

[Cấu hình mặc định](https://github.com/kubernetes/node-problem-detector/tree/v0.8.12/config)
được nhúng sẵn khi build Docker image của Node Problem Detector.

Tuy nhiên, bạn có thể dùng một
[`ConfigMap`](275-configure-pod-configmap-vi.md)
để ghi đè cấu hình:

1. Thay đổi các file cấu hình trong `config/`
1. Tạo `ConfigMap` `node-problem-detector-config`:

   ```shell
   kubectl create configmap node-problem-detector-config --from-file=config/
   ```

1. Thay đổi `node-problem-detector.yaml` để sử dụng `ConfigMap`:

   ```yaml
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     name: node-problem-detector-v0.1
     namespace: kube-system
     labels:
       k8s-app: node-problem-detector
       version: v0.1
       kubernetes.io/cluster-service: "true"
   spec:
     selector:
       matchLabels:
         k8s-app: node-problem-detector  
         version: v0.1
         kubernetes.io/cluster-service: "true"
     template:
       metadata:
         labels:
           k8s-app: node-problem-detector
           version: v0.1
           kubernetes.io/cluster-service: "true"
       spec:
         hostNetwork: true
         containers:
         - name: node-problem-detector
           image: registry.k8s.io/node-problem-detector:v0.1
           securityContext:
             privileged: true
           resources:
             limits:
               cpu: "200m"
               memory: "100Mi"
             requests:
               cpu: "20m"
               memory: "20Mi"
           volumeMounts:
           - name: log
             mountPath: /log
             readOnly: true
           - name: config # Ghi đè thư mục config/ bằng volume ConfigMap
             mountPath: /config
             readOnly: true
         volumes:
         - name: log
           hostPath:
             path: /var/log/
         - name: config # Định nghĩa volume ConfigMap
           configMap:
             name: node-problem-detector-config
   ```

1. Tạo lại Node Problem Detector với file cấu hình mới:

   ```shell
   # Nếu bạn đang có một node-problem-detector chạy sẵn, hãy xóa nó trước khi tạo lại
   kubectl delete -f https://k8s.io/examples/debug/node-problem-detector.yaml
   kubectl apply -f https://k8s.io/examples/debug/node-problem-detector-configmap.yaml
   ```

> **Ghi chú:** Cách tiếp cận này chỉ áp dụng cho Node Problem Detector được khởi động bằng
> `kubectl`.

Việc ghi đè cấu hình không được hỗ trợ nếu Node Problem Detector chạy dưới dạng Addon của
cluster. Addon manager không hỗ trợ `ConfigMap`.

## Các Problem Daemon (Problem Daemons)

Problem daemon là một daemon con (sub-daemon) của Node Problem Detector. Nó giám sát những loại
vấn đề node cụ thể và báo cáo chúng cho Node Problem Detector.
Có một số loại problem daemon được hỗ trợ.

- Daemon loại `SystemLogMonitor` giám sát log hệ thống và báo cáo các vấn đề cùng metrics
  theo các quy tắc được định nghĩa trước. Bạn có thể tùy biến cấu hình cho các nguồn log
  khác nhau như
  [filelog](https://github.com/kubernetes/node-problem-detector/blob/v0.8.12/config/kernel-monitor-filelog.json),
  [kmsg](https://github.com/kubernetes/node-problem-detector/blob/v0.8.12/config/kernel-monitor.json),
  [kernel](https://github.com/kubernetes/node-problem-detector/blob/v0.8.12/config/kernel-monitor-counter.json),
  [abrt](https://github.com/kubernetes/node-problem-detector/blob/v0.8.12/config/abrt-adaptor.json),
  và [systemd](https://github.com/kubernetes/node-problem-detector/blob/v0.8.12/config/systemd-monitor-counter.json).

- Daemon loại `SystemStatsMonitor` thu thập nhiều số liệu thống kê hệ thống liên quan đến sức
  khỏe dưới dạng metrics. Bạn có thể tùy biến hành vi của nó bằng cách cập nhật
  [file cấu hình](https://github.com/kubernetes/node-problem-detector/blob/v0.8.12/config/system-stats-monitor.json)
  của nó.

- Daemon loại `CustomPluginMonitor` gọi và kiểm tra nhiều vấn đề node khác nhau bằng cách chạy
  các script do người dùng định nghĩa. Bạn có thể dùng các custom plugin monitor khác nhau để
  giám sát các vấn đề khác nhau và tùy biến hành vi của daemon bằng cách cập nhật
  [file cấu hình](https://github.com/kubernetes/node-problem-detector/blob/v0.8.12/config/custom-plugin-monitor.json).

- Daemon loại `HealthChecker` kiểm tra sức khỏe của kubelet và container runtime trên một node.

### Thêm hỗ trợ cho định dạng log khác (Adding support for other log format) {#support-other-log-format}

System log monitor hiện hỗ trợ log dạng file, journald, và kmsg.
Có thể thêm các nguồn khác bằng cách triển khai một
[log watcher](https://github.com/kubernetes/node-problem-detector/blob/v0.8.12/pkg/systemlogmonitor/logwatchers/types/log_watcher.go)
mới.

### Thêm các custom plugin monitor (Adding custom plugin monitors)

Bạn có thể mở rộng Node Problem Detector để thực thi bất kỳ script giám sát nào viết bằng bất
kỳ ngôn ngữ nào bằng cách phát triển một custom plugin. Các script giám sát phải tuân theo giao
thức plugin về mã thoát (exit code) và đầu ra chuẩn (standard output). Để biết thêm thông tin,
vui lòng tham khảo
[đề xuất giao diện plugin](https://docs.google.com/document/d/1jK_5YloSYtboj-DtfjmYKxfNnUxCAvohLnsH5aGCAYQ/edit#).

## Exporter

Một exporter báo cáo các vấn đề node và/hoặc metrics tới các backend nhất định.
Các exporter sau được hỗ trợ:

- **Kubernetes exporter**: exporter này báo cáo các vấn đề node lên Kubernetes API server.
  Các vấn đề tạm thời được báo cáo dưới dạng Event và các vấn đề vĩnh viễn được báo cáo dưới
  dạng Node Condition.

- **Prometheus exporter**: exporter này báo cáo các vấn đề node và metrics tại chỗ (locally)
  dưới dạng metrics Prometheus (hoặc OpenMetrics). Bạn có thể chỉ định địa chỉ IP và port cho
  exporter bằng các đối số dòng lệnh.

- **Stackdriver exporter**: exporter này báo cáo các vấn đề node và metrics tới Stackdriver
  Monitoring API. Hành vi xuất dữ liệu có thể được tùy biến bằng một
  [file cấu hình](https://github.com/kubernetes/node-problem-detector/blob/v0.8.12/config/exporter/stackdriver-exporter.json).

## Khuyến nghị và giới hạn (Recommendations and restrictions)

Bạn nên chạy Node Problem Detector trong cluster của mình để giám sát sức khỏe của node.
Khi chạy Node Problem Detector, bạn có thể dự liệu một phần tài nguyên phụ trội (overhead) trên
mỗi node. Thông thường điều này không đáng ngại, vì:

* Log của kernel tăng trưởng tương đối chậm.
* Node Problem Detector đã được đặt giới hạn tài nguyên (resource limit).
* Ngay cả khi tải cao, mức sử dụng tài nguyên vẫn ở mức chấp nhận được. Để biết thêm thông tin,
  xem [kết quả benchmark](https://github.com/kubernetes/node-problem-detector/issues/2#issuecomment-220255629)
  của Node Problem Detector.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 23:

1. Bạn cài Node Problem Detector dưới dạng DaemonSet lên cluster lab. Manifest ví dụ mount
   `hostPath` `/var/log/` vào `/log` ở chế độ chỉ đọc, và bài kèm ngay một ghi chú cảnh báo. Ghi
   chú đó cảnh báo điều gì, và vì sao nó quyết định việc bạn có nhìn thấy sự cố mình cố tình tạo
   ra trên `lab-k8s-worker2` hay không?
2. Kubernetes exporter báo cáo hai loại vấn đề bằng hai cơ chế khác nhau. Loại nào thành Event,
   loại nào thành Node Condition, và bạn đọc mỗi thứ ở đâu?
3. **Câu bẫy.** NPD vừa đặt một Node Condition bất thường lên `lab-k8s-worker2`. Cluster có tự
   động phản ứng — đuổi Pod đi hoặc ngừng lập lịch lên node đó — không?
4. Bạn cần NPD phát hiện ba thứ: (a) một dòng lỗi xuất hiện trong log hệ thống của node, (b)
   container runtime trên node chết, (c) một tình trạng chỉ script của riêng bạn kiểm được. Mỗi
   thứ dùng loại problem daemon nào?
5. Bạn sửa một file trong `config/` để thêm luật phát hiện mới. Đưa cấu hình đó vào NPD đang chạy
   bằng cách nào, và cách đó dùng được cho mọi kiểu triển khai NPD không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Ghi chú cảnh báo phải **kiểm tra để chắc chắn thư mục log hệ thống là đúng với bản phân phối hệ
   điều hành của bạn**. Nó quyết định tất cả vì `SystemLogMonitor` — daemon con đọc log — chỉ thấy
   được những gì nằm trong thư mục được mount vào. **Mount sai thư mục thì NPD chạy bình thường
   nhưng không bao giờ báo cáo gì**, và bạn sẽ tưởng là sự cố mình tạo ra không đủ nghiêm trọng.
2. **Vấn đề tạm thời → Event; vấn đề vĩnh viễn → Node Condition.** Đây là hành vi của Kubernetes
   exporter, exporter báo về API server. Event đọc bằng `kubectl get events` (hoặc khối `Events`
   cuối `kubectl describe node`), còn Node Condition đọc ở bảng `Conditions` của
   `kubectl describe node` — đúng chỗ mà Checkpoint yêu cầu bạn chỉ ra.
3. **Không — bài không nói NPD làm gì hơn ngoài giám sát và báo cáo.** Ngay câu đầu bài định nghĩa
   nó là "một daemon dùng để giám sát và báo cáo về sức khỏe của một node": nó thu thập thông tin
   từ các daemon khác rồi báo cáo lên API server dưới dạng Condition hoặc Event, hết. Trực giác
   "condition xấu thì node bị loại" đến từ các Condition do chính kubelet đặt, không phải từ việc
   NPD có quyền lực gì với scheduler hay với Pod.
4. (a) **`SystemLogMonitor`** — giám sát log hệ thống và báo cáo theo các quy tắc định nghĩa trước,
   cấu hình được cho nhiều nguồn log (filelog, kmsg, kernel, abrt, systemd). (b) **`HealthChecker`**
   — đúng việc của nó là kiểm tra sức khỏe của kubelet và container runtime trên node. (c)
   **`CustomPluginMonitor`** — chạy script do người dùng định nghĩa, script phải tuân theo giao
   thức plugin về mã thoát và đầu ra chuẩn. (Loại thứ tư, `SystemStatsMonitor`, thu thập thống kê
   hệ thống dưới dạng metrics, không hợp với cả ba ca trên.)
5. Tạo ConfigMap `node-problem-detector-config` từ thư mục `config/`
   (`kubectl create configmap node-problem-detector-config --from-file=config/`), sửa manifest
   DaemonSet để mount volume ConfigMap đó **đè lên `/config`**, rồi xóa NPD cũ và tạo lại bằng
   manifest mới — vì cấu hình mặc định được **nhúng sẵn khi build image**. **Không dùng được cho
   mọi kiểu triển khai:** bài nói rõ cách này chỉ áp dụng cho NPD khởi động bằng `kubectl`; chạy
   dưới dạng Addon của cluster thì không ghi đè được, vì addon manager không hỗ trợ `ConfigMap`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng, rồi làm
**Checkpoint của [Giai đoạn 23 — Giám sát và cảnh báo](00-ALO-TRINH-ADMIN.md#giai-đoạn-23--giám-sát-và-cảnh-báo)**:
chạy `kubectl top node` và `kubectl top pod`, giải thích metrics-server khác Prometheus ở điểm nào
và vì sao HPA cần nó, rồi cài node-problem-detector và tạo ra một điều kiện node bất thường trên
`lab-k8s-worker2` để thấy nó được báo cáo.

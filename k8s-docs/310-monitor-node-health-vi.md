# Giám sát sức khỏe của Node (Monitor Node Health)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/debug-cluster/monitor-node-health/>

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

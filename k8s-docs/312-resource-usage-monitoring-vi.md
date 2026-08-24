# Các công cụ giám sát tài nguyên (Tools for Monitoring Resources)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-usage-monitoring/>

Để co giãn (scale) một ứng dụng và cung cấp một dịch vụ đáng tin cậy, bạn cần hiểu ứng dụng
hành xử như thế nào khi nó được triển khai. Bạn có thể xem xét hiệu năng của ứng dụng trong một
cluster Kubernetes bằng cách kiểm tra các container,
[pod](46-pods-vi.md),
[service](82-service-vi.md), và
các đặc tính của toàn bộ cluster. Kubernetes cung cấp thông tin chi tiết về mức sử dụng tài
nguyên của ứng dụng ở từng cấp độ này. Thông tin này cho phép bạn đánh giá hiệu năng của ứng
dụng và xác định nơi có thể loại bỏ các điểm nghẽn (bottleneck) để cải thiện hiệu năng tổng thể.

Trong Kubernetes, việc giám sát (monitoring) ứng dụng không phụ thuộc vào một giải pháp giám
sát duy nhất. Trên các cluster mới, bạn có thể dùng pipeline
[metrics tài nguyên](#resource-metrics-pipeline) hoặc pipeline
[metrics đầy đủ](#full-metrics-pipeline) để thu thập số liệu giám sát.

## Pipeline metrics tài nguyên (Resource metrics pipeline) {#resource-metrics-pipeline}

Pipeline metrics tài nguyên cung cấp một tập metrics hạn chế liên quan đến các thành phần của
cluster như controller
[Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/),
cũng như tiện ích `kubectl top`.
Các metrics này được thu thập bởi
[metrics-server](https://github.com/kubernetes-sigs/metrics-server) — một thành phần gọn nhẹ,
lưu dữ liệu ngắn hạn trong bộ nhớ (in-memory) — và được cung cấp qua API `metrics.k8s.io`.

metrics-server khám phá tất cả các node trong cluster và truy vấn
[kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/) của từng
node để lấy mức sử dụng CPU và bộ nhớ. Kubelet đóng vai trò cầu nối giữa Kubernetes master và
các node, quản lý các pod và container chạy trên một máy. Kubelet phân giải mỗi pod thành các
container cấu thành của nó và lấy số liệu sử dụng của từng container từ container runtime thông
qua giao diện container runtime (container runtime interface). Nếu bạn dùng một container
runtime sử dụng cgroups và namespaces của Linux để triển khai container, và container runtime
đó không công bố số liệu sử dụng, thì kubelet có thể tự tra cứu trực tiếp các số liệu đó (dùng
mã nguồn từ [cAdvisor](https://github.com/google/cadvisor)).
Bất kể các số liệu đó đến bằng cách nào, kubelet sau đó cung cấp số liệu sử dụng tài nguyên
tổng hợp của pod thông qua Resource Metrics API của metrics-server.
API này được phục vụ tại `/metrics/resource` trên các port có xác thực (authenticated) và
chỉ đọc (read-only) của kubelet.

## Pipeline metrics đầy đủ (Full metrics pipeline) {#full-metrics-pipeline}

Một pipeline metrics đầy đủ cho bạn quyền truy cập vào các metrics phong phú hơn. Kubernetes có
thể phản ứng với các metrics này bằng cách tự động co giãn hoặc điều chỉnh cluster dựa trên
trạng thái hiện tại của nó, thông qua các cơ chế như Horizontal Pod Autoscaler. Pipeline giám
sát lấy metrics từ kubelet rồi cung cấp chúng cho Kubernetes qua một adapter bằng cách triển
khai API `custom.metrics.k8s.io` hoặc `external.metrics.k8s.io`.

Kubernetes được thiết kế để hoạt động với [OpenMetrics](https://openmetrics.io/), một trong các
[dự án Giám sát thuộc nhóm Observability và Phân tích của CNCF](https://landscape.cncf.io/?group=projects-and-products&view-mode=card#observability-and-analysis--monitoring),
được xây dựng dựa trên và mở rộng cẩn trọng
[định dạng phơi bày metrics của Prometheus (Prometheus exposition format)](https://prometheus.io/docs/instrumenting/exposition_formats/)
theo cách gần như tương thích ngược 100%.

Nếu bạn nhìn qua
[CNCF Landscape](https://landscape.cncf.io/?group=projects-and-products&view-mode=card#observability-and-analysis--monitoring),
bạn có thể thấy nhiều dự án giám sát có thể hoạt động với Kubernetes bằng cách _scrape_ (cào)
dữ liệu metric và dùng dữ liệu đó để giúp bạn quan sát cluster của mình. Việc chọn công cụ hay
các công cụ phù hợp với nhu cầu là tùy ở bạn. CNCF landscape cho observability và phân tích bao
gồm cả phần mềm mã nguồn mở, phần mềm dạng dịch vụ (software-as-a-service) trả phí, và các sản
phẩm thương mại khác.

Khi bạn thiết kế và triển khai một pipeline metrics đầy đủ, bạn có thể đưa dữ liệu giám sát đó
trở lại cho Kubernetes sử dụng. Ví dụ, một HorizontalPodAutoscaler có thể dùng các metrics đã
xử lý để tính ra cần chạy bao nhiêu Pod cho một thành phần trong workload của bạn.

Việc tích hợp một pipeline metrics đầy đủ vào hệ thống Kubernetes của bạn nằm ngoài phạm vi của
tài liệu Kubernetes, vì phạm vi các giải pháp khả dĩ là rất rộng.

Việc lựa chọn nền tảng giám sát phụ thuộc nhiều vào nhu cầu, ngân sách và nguồn lực kỹ thuật
của bạn. Kubernetes không khuyến nghị một pipeline metrics cụ thể nào;
[có rất nhiều lựa chọn](https://landscape.cncf.io/?group=projects-and-products&view-mode=card#observability-and-analysis--monitoring).
Hệ thống giám sát của bạn cần có khả năng xử lý chuẩn truyền tải metrics
[OpenMetrics](https://openmetrics.io/) và cần được chọn sao cho phù hợp nhất với thiết kế và
cách triển khai tổng thể của nền tảng hạ tầng của bạn.

## Tiếp theo (What's next)

Tìm hiểu về các công cụ gỡ lỗi (debug) bổ sung, bao gồm:

* [Logging](158-logging-vi.md)
* [Truy cập vào container qua `exec`](304-get-shell-running-container-vi.md)
* [Kết nối tới container qua proxy](https://kubernetes.io/docs/tasks/extend-kubernetes/http-proxy-access-api/)
* [Kết nối tới container qua chuyển tiếp port (port forwarding)](366-port-forward-vi.md)
* [Kiểm tra node Kubernetes bằng crictl](307-crictl-vi.md)

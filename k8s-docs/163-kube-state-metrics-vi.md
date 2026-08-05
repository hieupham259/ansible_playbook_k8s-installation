# Metrics cho trạng thái đối tượng Kubernetes (Metrics for Kubernetes Object States)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/concepts/cluster-administration/kube-state-metrics/
>
> kube-state-metrics, một agent add-on dùng để sinh ra và expose các metrics ở cấp cluster.

Trạng thái của các đối tượng Kubernetes trong Kubernetes API có thể được expose dưới dạng metrics.
Một agent add-on có tên [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) có thể kết nối đến Kubernetes API server và expose một endpoint HTTP với các metrics được sinh ra từ trạng thái của từng đối tượng riêng lẻ trong cluster.
Nó expose nhiều thông tin khác nhau về trạng thái của các đối tượng như label và annotation, thời điểm khởi động và kết thúc, trạng thái (status) hoặc phase mà đối tượng hiện đang ở.
Ví dụ, các container chạy trong pod tạo ra một metric `kube_pod_container_info`.
Metric này bao gồm tên của container, tên của pod chứa nó, namespace mà pod đang chạy trong đó, tên của container image, ID của image, tên image lấy từ spec của container, ID của container đang chạy và ID của pod — tất cả dưới dạng label.

> **Ghi chú:** Mục này đề cập đến một sản phẩm hoặc dự án bên thứ ba cung cấp chức năng mà Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về sản phẩm hoặc dự án bên thứ ba đó. Xem [hướng dẫn trên website của CNCF](https://github.com/cncf/foundation/blob/master/website-guidelines.md) để biết thêm chi tiết.

Một thành phần bên ngoài có đủ khả năng thu thập (scrape) endpoint của kube-state-metrics (ví dụ thông qua Prometheus) giờ đây có thể được dùng để hiện thực các trường hợp sử dụng sau.

## Ví dụ: dùng metrics từ kube-state-metrics để truy vấn trạng thái cluster (Example: using metrics from kube-state-metrics to query the cluster state) {#example-kube-state-metrics-query-1}

Các chuỗi metric (metric series) do kube-state-metrics sinh ra rất hữu ích để thu thập thêm hiểu biết sâu hơn về cluster, vì chúng có thể được dùng cho việc truy vấn.

Nếu bạn dùng Prometheus hoặc một công cụ khác sử dụng cùng ngôn ngữ truy vấn, câu truy vấn PromQL sau sẽ trả về số lượng pod chưa sẵn sàng (not ready):

```
count(kube_pod_status_ready{condition="false"}) by (namespace, pod)
```

## Ví dụ: cảnh báo dựa trên kube-state-metrics (Example: alerting based on from kube-state-metrics) {#example-kube-state-metrics-alert-1}

Các metrics được sinh ra từ kube-state-metrics cũng cho phép cảnh báo về các vấn đề trong cluster.

Nếu bạn dùng Prometheus hoặc một công cụ tương tự sử dụng cùng ngôn ngữ quy tắc cảnh báo (alert rule), cảnh báo sau sẽ được kích hoạt nếu có các pod đã ở trạng thái `Terminating` quá 5 phút:

```yaml
groups:
- name: Pod state
  rules:
  - alert: PodsBlockedInTerminatingState
    expr: count(kube_pod_deletion_timestamp) by (namespace, pod) * count(kube_pod_status_reason{reason="NodeLost"} == 0) by (namespace, pod) > 0
    for: 5m
    labels:
      severity: page
    annotations:
      summary: Pod {{$labels.namespace}}/{{$labels.pod}} blocked in Terminating state.
```

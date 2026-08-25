# Metrics cho trạng thái đối tượng Kubernetes (Metrics for Kubernetes Object States)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/concepts/cluster-administration/kube-state-metrics/
>
> kube-state-metrics, một agent add-on dùng để sinh ra và expose các metrics ở cấp cluster.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 11](00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability), bài 3/6 · Kiểm chứng
ở [Lab 11a](labs/LAB-11A-OBSERVABILITY.md).

Bài rất ngắn, nhưng nó vẽ đúng một ranh giới quan trọng: **metric tài nguyên** (thứ kubelet đo
được trên node của nó) khác **metric trạng thái đối tượng** (thứ chỉ đọc được từ Kubernetes
API). Đọc bài này chủ yếu để đặt kube-state-metrics vào đúng ô trong bức tranh của bài
[162](162-observability-vi.md).

**Phải hiểu ở lần đọc này:**

- kube-state-metrics là một **agent add-on**, không phải thành phần core: nó kết nối tới API
  server và expose một endpoint HTTP riêng để bên ngoài scrape.
- Nó sinh metric **từ trạng thái của từng đối tượng** — label, annotation, thời điểm khởi động
  và kết thúc, status hoặc phase — chứ không đo mức tiêu thụ CPU hay bộ nhớ.
- Thông tin nằm ở **label của metric**, không ở giá trị: `kube_pod_container_info` mang tên
  container, tên pod, namespace, tên image, ID image, ID container và ID pod.
- Vì đầu ra là các chuỗi metric nên trạng thái cluster trở thành thứ **truy vấn và cảnh báo
  được** bằng công cụ bên ngoài, thay vì phải `kubectl get` rồi đọc bằng mắt.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Câu truy vấn PromQL trong *Ví dụ: dùng metrics từ kube-state-metrics để truy vấn trạng thái cluster* | cú pháp PromQL, và cluster lab chưa có Prometheus | Lab 11a |
| Quy tắc alert `PodsBlockedInTerminatingState` trong *Ví dụ: cảnh báo dựa trên kube-state-metrics* | cú pháp alert rule và cách gắn severity | giai đoạn 23 giám sát và cảnh báo |
| Ghi chú miễn trừ trách nhiệm với dự án bên thứ ba | là tuyên bố pháp lý của dự án, không phải kiến thức kỹ thuật | không cần |

---

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

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 11:

1. **Câu bẫy.** Trên cluster lab, `lab-k8s-worker2` đang có hai hiện tượng: một Pod ngốn gần hết CPU
   của node, và một Pod khác kẹt ở trạng thái `Terminating` suốt mười phút. Hiện tượng nào là
   thứ kube-state-metrics sinh ra được metric, hiện tượng nào không? Vì sao?
2. kube-state-metrics lấy dữ liệu từ đâu, và nó đưa dữ liệu ra ngoài bằng cách nào? Nếu nó mất
   kết nối tới nguồn đó thì metric còn phản ánh thực tế nữa không?
3. Metric `kube_pod_container_info` có giá trị số gần như vô nghĩa. Vậy nó hữu ích ở chỗ nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Chỉ hiện tượng **kẹt `Terminating`** là thứ kube-state-metrics phủ được — đó là **trạng thái
   của đối tượng** trong Kubernetes API, đúng loại thông tin bài mô tả ("status hoặc phase mà
   đối tượng hiện đang ở"), và chính ví dụ cảnh báo trong bài dựng trên
   `kube_pod_deletion_timestamp` và `kube_pod_status_reason`. **Mức tiêu thụ CPU thì không**: nó
   không phải trạng thái đối tượng, mà là số đo tài nguyên trên node. Trực giác sai ở chỗ nghĩ
   "cứ metric về Pod là kube-state-metrics lo hết"; ranh giới thật là **trạng thái đối tượng** so
   với **mức sử dụng tài nguyên**.
2. Nó **kết nối tới Kubernetes API server** để đọc trạng thái các đối tượng, rồi **expose một
   endpoint HTTP** để một thành phần bên ngoài (ví dụ Prometheus) thu thập. Vì nguồn duy nhất là
   API server, mất kết nối tới đó nghĩa là metric **không còn phản ánh thực tế** — nó không quan
   sát node hay container một cách độc lập.
3. Vì **toàn bộ thông tin nằm ở label, không ở giá trị**. Bài liệt kê rõ: tên container, tên pod,
   namespace, tên image, ID image, tên image lấy từ spec, ID container và ID pod — tất cả dưới
   dạng label. Nhờ vậy bạn lọc và nhóm được các metric khác theo những chiều này, ví dụ tìm mọi
   Pod đang chạy một image cụ thể.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

# Các công cụ giám sát tài nguyên (Tools for Monitoring Resources)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-usage-monitoring/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 23 — Giám sát và cảnh báo](00-ALO-TRINH-ADMIN.md#giai-đoạn-23--giám-sát-và-cảnh-báo),
bài 2/3 · Giai đoạn này của Phần II không có lab riêng: thực hành trực tiếp trên cluster lab ở mốc
`04-metrics-ready` (xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)), tự chấm bằng Checkpoint ghi ở
cuối mục giai đoạn trong lộ trình.

Bài rất ngắn nhưng là bài **chốt ranh giới** của cả giai đoạn: nó đặt cạnh nhau hai pipeline mà
bài [311](311-resource-metrics-pipeline-vi.md) vừa mô tả một nửa. Đây là chỗ lấy câu trả lời cho
vế "metrics-server khác Prometheus ở điểm nào" trong Checkpoint giai đoạn 23.

**Phải hiểu ở lần đọc này:**

- Đoạn mở đầu: Kubernetes **không gắn với một giải pháp giám sát duy nhất**. Bạn có hai lựa chọn
  và chúng không thay thế nhau — *pipeline metrics tài nguyên* và *pipeline metrics đầy đủ*.
- Mục *Pipeline metrics tài nguyên*: đây là pipeline phục vụ **chính các cơ chế bên trong
  Kubernetes** — controller HorizontalPodAutoscaler và tiện ích `kubectl top` — qua API
  `metrics.k8s.io`. Bài mô tả metrics-server bằng đúng hai chữ quyết định: **gọn nhẹ** và **lưu dữ
  liệu ngắn hạn trong bộ nhớ (in-memory)**. Đó là toàn bộ khác biệt so với một hệ giám sát thật.
- Cũng ở mục đó, đường lấy số liệu: metrics-server khám phá mọi node rồi truy vấn kubelet từng
  node; kubelet phân giải mỗi Pod thành các container và lấy số liệu từ container runtime qua
  container runtime interface; **nếu runtime không công bố số liệu** thì kubelet tự tra cứu trực
  tiếp bằng mã nguồn cAdvisor. Kết quả phục vụ tại `/metrics/resource`, trên port có xác thực và
  chỉ đọc của kubelet.
- Mục *Pipeline metrics đầy đủ*: nó lấy metric từ kubelet rồi **đưa ngược trở lại** cho Kubernetes
  qua một **adapter** triển khai `custom.metrics.k8s.io` hoặc `external.metrics.k8s.io`. Nhờ vậy
  một HorizontalPodAutoscaler có thể tính số Pod từ metric đã xử lý, chứ không chỉ từ CPU/memory.
- Cũng mục đó: Kubernetes **không khuyến nghị** pipeline metrics cụ thể nào, và việc tích hợp một
  pipeline đầy đủ nằm **ngoài phạm vi** tài liệu Kubernetes. Ràng buộc kỹ thuật duy nhất bài đặt
  ra là hệ giám sát phải xử lý được chuẩn truyền tải [OpenMetrics](https://openmetrics.io/) — vốn
  mở rộng từ định dạng phơi bày của Prometheus theo cách gần như tương thích ngược 100%.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Duyệt CNCF Landscape và chọn nền tảng giám sát | bài nói thẳng là không có khuyến nghị chung, lựa chọn phụ thuộc nhu cầu, ngân sách và nguồn lực; cluster lab cũng không dựng stack giám sát | không thuộc lộ trình — [Lab 11a](labs/LAB-11A-OBSERVABILITY.md) đã cố ý để trống đúng khúc "kho lưu trữ chuỗi thời gian" này |
| Chi tiết định dạng phơi bày metrics của Prometheus / OpenMetrics | bạn đã đọc và đã tự gọi endpoint `/metrics` thật rồi | bài [160](160-system-metrics-vi.md) ở [giai đoạn 11](00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability) và [Lab 11a](labs/LAB-11A-OBSERVABILITY.md) |
| Danh sách *Tiếp theo* (logging, `exec`, proxy, port-forward, `crictl`) | là bảng chỉ đường sang công cụ khác, không phải nội dung của bài | bài [307](307-crictl-vi.md) ở [giai đoạn 24](00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố) và bài [158](158-logging-vi.md) |

---

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
* [Kết nối tới container qua proxy](379-http-proxy-access-api-vi.md)
* [Kết nối tới container qua chuyển tiếp port (port forwarding)](366-port-forward-vi.md)
* [Kiểm tra node Kubernetes bằng crictl](307-crictl-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 23:

1. Checkpoint giai đoạn 23 hỏi "metrics-server khác Prometheus ở điểm nào". Trả lời bằng đúng
   những gì bài mô tả về metrics-server, rồi nói tiếp vì sao HPA vẫn cần metrics-server dù một hệ
   giám sát đầy đủ có nhiều metric hơn hẳn.
2. **Câu bẫy.** Bạn muốn HPA co giãn một Deployment theo **số request mỗi giây** của nginx.
   metrics-server đã chạy sẵn trên cluster lab. Vậy là đủ chưa? Nếu chưa thì thiếu thành phần nào
   và API nào?
3. Trên `lab-k8s-worker1`, container chạy bằng containerd. Kể lại đường mà số liệu sử dụng của một
   container đi từ runtime tới metrics-server: kubelet lấy nó từ đâu, khi nào kubelet phải tự tra
   cứu bằng mã cAdvisor, và kết quả được phơi ra ở endpoint nào với tính chất gì?
4. Bài nói Kubernetes không khuyến nghị một pipeline metrics cụ thể nào. Vậy ràng buộc kỹ thuật
   duy nhất mà bài đặt ra cho hệ giám sát bạn chọn là gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Bài gọi metrics-server là **thành phần gọn nhẹ, lưu dữ liệu ngắn hạn trong bộ nhớ
   (in-memory)**, phục vụ **một tập metrics hạn chế** qua API `metrics.k8s.io`. Nói cách khác:
   **không có kho lưu trữ chuỗi thời gian, không có lịch sử, không có ngôn ngữ truy vấn** — đúng ba
   thứ làm nên một hệ giám sát đầy đủ. Nhưng HPA vẫn cần nó vì **HPA đọc `metrics.k8s.io`**: đó là
   pipeline metrics tài nguyên, thứ mà bài liệt kê phục vụ trực tiếp cho controller HPA và
   `kubectl top`. Một hệ giám sát đầy đủ không tự động phục vụ API đó.
2. **Chưa đủ.** metrics-server chỉ phục vụ pipeline metrics tài nguyên — CPU và memory. Request mỗi
   giây là metric của ứng dụng, thuộc **pipeline metrics đầy đủ**: bạn cần một hệ giám sát scrape
   được metric đó, **cộng một adapter** triển khai `custom.metrics.k8s.io` (hoặc
   `external.metrics.k8s.io` nếu nguồn nằm ngoài cluster) để đưa số liệu đã xử lý trở lại cho
   Kubernetes. Đúng như bài nói: khi đã có pipeline đầy đủ, HorizontalPodAutoscaler mới dùng được
   metric đã xử lý để tính cần chạy bao nhiêu Pod.
3. metrics-server **khám phá tất cả node** rồi **truy vấn kubelet** của từng node. Kubelet phân
   giải mỗi Pod thành các container cấu thành và lấy số liệu của từng container **từ container
   runtime qua container runtime interface**. Nếu runtime dùng cgroups và namespaces của Linux
   nhưng **không công bố số liệu sử dụng**, kubelet **tự tra cứu trực tiếp** bằng mã nguồn từ
   cAdvisor. Kết quả tổng hợp theo Pod được phơi tại **`/metrics/resource`**, trên các port
   **có xác thực (authenticated) và chỉ đọc (read-only)** của kubelet.
4. **Hệ giám sát phải xử lý được chuẩn truyền tải metrics OpenMetrics**, và phải được chọn sao cho
   phù hợp với thiết kế cùng cách triển khai tổng thể hạ tầng của bạn. Ngoài hai điều đó bài không
   ràng buộc gì thêm — nó nói thẳng là có rất nhiều lựa chọn và Kubernetes không khuyến nghị cái
   nào.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

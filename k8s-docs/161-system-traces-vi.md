# Trace cho các thành phần hệ thống Kubernetes (Traces For Kubernetes System Components)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/cluster-administration/system-traces/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 11](LO-TRINH-ADMIN.md#giai-đoạn-11--observability), bài 6/6 · Kiểm chứng
ở Lab 11a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Đây là trụ cột thứ ba và cũng là trụ cột **non nhất**: bài tự nói phần đo đạc còn đang phát triển
tích cực và chưa có bảo đảm tương thích ngược. Lần đọc này chỉ cần biết trace của Kubernetes trả
lời được câu hỏi gì mà metric và log không trả lời được, cùng những giới hạn phải nhớ trước khi
bật nó trên cluster thật.

**Phải hiểu ở lần đọc này:**

- Trace ghi **độ trễ và mối quan hệ giữa các thao tác**. Các thành phần phát trace bằng **OTLP
  qua gRPC exporter**, mặc định tới port **4317**, và có hai đường: đi thẳng tới backend (chỉ
  cần đặt trường `endpoint`) hoặc qua một **OpenTelemetry Collector**.
- kube-apiserver sinh span cho request đến, cho request đi ra tới webhook và etcd, và cho request
  tái nhập. Nó **lan truyền W3C Trace Context với request đi ra** nhưng **không dùng trace
  context đính kèm theo request đến**, vì nó thường là một endpoint công khai.
- kubelet đo đạc giao diện CRI và các HTTP server có xác thực; nó **lan truyền trace context
  xuống container runtime** qua gRPC, nên span của kubelet và span của containerd nối được thành
  quan hệ cha–con — đó là giá trị chính khi gỡ lỗi trên node.
- Cách đọc `samplingRatePerMillion`: `100` là ghi span cho 1 trong mỗi 10000 request, `1000000`
  là mọi span. **Quyết định lấy mẫu của span cha luôn được tôn trọng**, tỷ lệ này chỉ áp dụng cho
  span **không có cha**.
- Bật tracing luôn có **chi phí mạng và CPU**; cách giảm nhẹ là hạ `samplingRatePerMillion` hoặc
  tắt hẳn bằng cách gỡ cấu hình. Cộng thêm việc tên span và thuộc tính chưa ổn định, đây là tính
  năng để chẩn đoán theo đợt, không phải để dựng dashboard dài hạn.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| `--tracing-config-file` và nội dung `TracingConfiguration` của kube-apiserver | phải sửa manifest static Pod của API server | CP5 cấu hình lại cluster đang chạy |
| Đoạn cấu hình `tracing` trong `KubeletConfiguration` | phải sửa file cấu hình kubelet trên từng node rồi khởi động lại | CP5 cấu hình lại cluster đang chạy |
| Cấu hình receiver YAML của OpenTelemetry Collector | thuộc tài liệu OpenTelemetry, không phải Kubernetes | Lab 11a |
| `OTEL_EXPORTER_OTLP_HEADERS` và `OTEL_RESOURCE_ATTRIBUTES` | chỉ cần khi đã có backend tracing thật cần xác thực | CP8 giám sát và cảnh báo |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.27 [beta]`

Trace của các thành phần hệ thống ghi lại độ trễ và mối quan hệ giữa các thao tác trong cluster.

Các thành phần Kubernetes phát ra trace bằng
[OpenTelemetry Protocol](https://opentelemetry.io/docs/specs/otlp/)
với gRPC exporter và có thể được thu thập rồi định tuyến tới các backend truy vết (tracing
backend) bằng một
[OpenTelemetry Collector](https://github.com/open-telemetry/opentelemetry-collector#-opentelemetry-collector).

## Thu thập trace (Trace Collection)

Các thành phần Kubernetes có sẵn gRPC exporter tích hợp cho OTLP để xuất trace, hoặc thông
qua một OpenTelemetry Collector, hoặc không cần OpenTelemetry Collector.

Để có hướng dẫn đầy đủ về việc thu thập trace và sử dụng collector, xem
[Bắt đầu với OpenTelemetry Collector](https://opentelemetry.io/docs/collector/getting-started/).
Tuy nhiên, có một vài điểm cần lưu ý dành riêng cho các thành phần Kubernetes.

Mặc định, các thành phần Kubernetes xuất trace bằng grpc exporter cho OTLP trên
[cổng OpenTelemetry theo IANA](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=opentelemetry), 4317.
Ví dụ, nếu collector chạy như một sidecar của một thành phần Kubernetes,
cấu hình receiver sau đây sẽ thu thập các span và ghi chúng ra standard output:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
exporters:
  # Thay exporter này bằng exporter cho backend của bạn
  exporters:
    debug:
      verbosity: detailed
service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [debug]
```

Để phát trace trực tiếp tới một backend mà không cần dùng collector,
hãy chỉ định trường endpoint trong file cấu hình tracing của Kubernetes với địa chỉ trace
backend mong muốn.
Cách này loại bỏ nhu cầu về collector và đơn giản hóa cấu trúc tổng thể.

Để cấu hình header cho trace backend, bao gồm cả thông tin xác thực, có thể dùng biến môi
trường với `OTEL_EXPORTER_OTLP_HEADERS`,
xem [Cấu hình OTLP Exporter](https://opentelemetry.io/docs/languages/sdk-configuration/otlp-exporter/).

Ngoài ra, để cấu hình các thuộc tính tài nguyên (resource attribute) của trace như tên
cluster Kubernetes, namespace, tên Pod, v.v.,
cũng có thể dùng biến môi trường với `OTEL_RESOURCE_ATTRIBUTES`, xem
[OTLP Kubernetes Resource](https://opentelemetry.io/docs/specs/semconv/resource/k8s/).

## Trace của các thành phần (Component traces)

### Trace của kube-apiserver (kube-apiserver traces)

kube-apiserver sinh ra các span cho các HTTP request đến, và cho các request đi ra tới
webhook, etcd, cũng như các request tái nhập (re-entrant). Nó lan truyền (propagate)
[W3C Trace Context](https://www.w3.org/TR/trace-context/) với các request đi ra
nhưng không sử dụng trace context đính kèm theo các request đến,
vì kube-apiserver thường là một endpoint công khai.

#### Bật tracing trong kube-apiserver (Enabling tracing in the kube-apiserver)

Để bật tracing, cung cấp cho kube-apiserver một file cấu hình tracing
bằng `--tracing-config-file=<path-to-config>`. Đây là một ví dụ cấu hình ghi span
cho 1 trong mỗi 10000 request, và dùng endpoint OpenTelemetry mặc định:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: TracingConfiguration
# giá trị mặc định
#endpoint: localhost:4317
samplingRatePerMillion: 100
```

Để biết thêm thông tin về struct `TracingConfiguration`, xem
[API cấu hình API server (v1)](https://kubernetes.io/docs/reference/config-api/apiserver-config.v1/#apiserver-k8s-io-v1-TracingConfiguration).

### Trace của kubelet (kubelet traces)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.34 [stable]`

Giao diện CRI của kubelet và các HTTP server có xác thực của nó được đo đạc (instrument)
để sinh ra các trace span. Như với apiserver, endpoint và tỷ lệ lấy mẫu (sampling rate)
đều có thể cấu hình được.
Việc lan truyền trace context cũng được cấu hình. Quyết định lấy mẫu của span cha luôn được
tôn trọng. Tỷ lệ lấy mẫu trong cấu hình tracing được cung cấp sẽ áp dụng cho các span không
có cha.
Nếu được bật mà không cấu hình endpoint, địa chỉ receiver mặc định của OpenTelemetry
Collector là "localhost:4317" sẽ được sử dụng.

#### Bật tracing trong kubelet (Enabling tracing in the kubelet)

Để bật tracing, hãy áp dụng [cấu hình tracing](https://github.com/kubernetes/component-base/blob/release-1.27/tracing/api/v1/types.go).
Đây là một ví dụ đoạn cấu hình kubelet ghi span cho 1 trong mỗi 10000 request, và dùng
endpoint OpenTelemetry mặc định:

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
tracing:
  # giá trị mặc định
  #endpoint: localhost:4317
  samplingRatePerMillion: 100
```

Nếu `samplingRatePerMillion` được đặt là một triệu (`1000000`), thì mọi span
đều sẽ được gửi tới exporter.

kubelet trong Kubernetes v1.36 thu thập span từ hoạt động garbage collection, quy trình
đồng bộ hóa pod cũng như mọi phương thức gRPC. kubelet lan truyền trace context với các
request gRPC để các container runtime có đo đạc trace, chẳng hạn CRI-O và containerd,
có thể liên kết các span mà chúng xuất ra với trace context từ kubelet.
Các trace thu được sẽ có liên kết cha-con giữa các span của kubelet và của container
runtime, mang lại ngữ cảnh hữu ích khi gỡ lỗi các vấn đề trên node.

Xin lưu ý rằng việc xuất span luôn đi kèm một chi phí hiệu năng nhỏ
về phía mạng và CPU, tùy theo cấu hình tổng thể của hệ thống. Nếu có bất kỳ vấn đề nào như
vậy trong một cluster đang chạy với tracing được bật, hãy giảm nhẹ vấn đề bằng cách giảm
`samplingRatePerMillion` hoặc tắt hẳn tracing bằng cách gỡ bỏ cấu hình.

## Tính ổn định (Stability)

Phần đo đạc tracing vẫn đang được phát triển tích cực và có thể thay đổi
theo nhiều cách. Điều này bao gồm tên span, các thuộc tính đính kèm,
các endpoint được đo đạc, v.v. Cho tới khi tính năng này lên mức ổn định (stable),
không có bảo đảm nào về tương thích ngược cho phần đo đạc tracing.

## Tiếp theo (What's next)

* Đọc về [Bắt đầu với OpenTelemetry Collector](https://opentelemetry.io/docs/collector/getting-started/)
* Đọc về [Cấu hình OTLP Exporter](https://opentelemetry.io/docs/languages/sdk-configuration/otlp-exporter/)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 11:

1. Cluster lab dùng containerd làm container runtime. Bật tracing cho kubelet trên `k8s-worker1`
   mang lại thứ gì mà log của kubelet không có? Cơ chế nào làm được điều đó?
2. **Câu bẫy.** Client của bạn gửi request tới kube-apiserver kèm sẵn trace context, mong span
   của client nối liền vào trace của API server. Việc đó có xảy ra không? Vì sao Kubernetes chọn
   như vậy?
3. `samplingRatePerMillion: 100` nghĩa là gì? Một span có cha đã được quyết định lấy mẫu thì tỷ
   lệ này còn chi phối nó nữa không?
4. Bạn định dựng một dashboard dài hạn cho đội trực, dựa trên tên span mà kube-apiserver phát ra.
   Bài khuyên gì về ý định này?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Nó cho bạn **liên kết cha–con giữa span của kubelet và span do container runtime xuất ra**.
   Cơ chế: kubelet **lan truyền trace context theo các request gRPC**, nên một runtime có đo đạc
   trace như containerd hay CRI-O gắn được span của nó vào đúng trace của kubelet. Kết quả là
   thấy được một thao tác trên node đi qua hai tiến trình mất bao lâu ở từng chặng — thứ mà đọc
   log kubelet không dựng lại được.
2. **Không.** kube-apiserver **lan truyền W3C Trace Context với các request đi ra, nhưng không sử
   dụng trace context đính kèm theo các request đến**. Lý do bài nêu thẳng: kube-apiserver
   **thường là một endpoint công khai**, nên tin vào trace context do client bất kỳ gửi tới là
   không an toàn. Trực giác sai ở chỗ nghĩ tracing phân tán thì luôn nối được đầu-cuối; ở đây
   biên tin cậy cắt đúng tại API server.
3. Nghĩa là ghi span cho **1 trong mỗi 10000 request** (100 phần triệu). Và **không** — bài nói
   rõ **quyết định lấy mẫu của span cha luôn được tôn trọng**, còn tỷ lệ trong cấu hình tracing
   chỉ áp dụng cho **các span không có cha**. Nếu đặt `1000000` thì mọi span đều được gửi tới
   exporter.
4. **Đừng làm lúc này.** Bài dành hẳn mục *Tính ổn định* để nói rằng phần đo đạc tracing vẫn đang
   được phát triển tích cực và có thể thay đổi ở nhiều mặt — **tên span, các thuộc tính đính kèm,
   các endpoint được đo đạc** — và **không có bảo đảm nào về tương thích ngược** cho tới khi tính
   năng lên mức stable. Thêm vào đó, việc xuất span luôn kèm chi phí mạng và CPU, nên tracing hợp
   với các đợt chẩn đoán có thời hạn hơn là với giám sát thường trực.

</details>

Đây là bài cuối của giai đoạn 11. Trả lời trôi cả bốn câu thì chuyển sang **Lab 11a** (chưa viết,
xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)) — nơi bạn cài metrics-server, chụp snapshot
`04-metrics-ready` rồi dùng nó để trả nợ HPA/VPA ở Lab 11b, xem
[sổ nợ lab](labs/README.md#5-sổ-nợ-lab). Câu nào còn vướng thì quay lại đúng mục tương ứng trước
khi mở lab.

# Trace cho các thành phần hệ thống Kubernetes (Traces For Kubernetes System Components)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/cluster-administration/system-traces/>

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

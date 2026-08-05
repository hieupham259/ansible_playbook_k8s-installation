# Metric cho các thành phần hệ thống Kubernetes (Metrics For Kubernetes System Components)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/cluster-administration/system-metrics/>

Metric của các thành phần hệ thống có thể cho cái nhìn rõ hơn về những gì đang diễn ra bên
trong chúng. Metric đặc biệt hữu ích để xây dựng dashboard và cảnh báo (alert).

Các thành phần Kubernetes phát ra metric theo
[định dạng Prometheus](https://prometheus.io/docs/instrumenting/exposition_formats/).
Định dạng này là văn bản thuần có cấu trúc, được thiết kế để cả con người lẫn máy đều có
thể đọc được.

## Metric trong Kubernetes (Metrics in Kubernetes)

Trong hầu hết các trường hợp, metric khả dụng tại endpoint `/metrics` của HTTP server. Với
các thành phần không expose endpoint này theo mặc định, có thể bật bằng cờ `--bind-address`.

Ví dụ về các thành phần đó:

* kube-controller-manager
* kube-proxy
* kube-apiserver
* kube-scheduler
* kubelet

Trong môi trường production, bạn có thể muốn cấu hình [Prometheus Server](https://prometheus.io/)
hoặc một công cụ thu thập (scraper) metric khác để định kỳ thu thập các metric này và đưa
chúng vào một dạng cơ sở dữ liệu chuỗi thời gian (time series database).

Lưu ý rằng kubelet cũng expose metric tại các endpoint `/metrics/cadvisor`,
`/metrics/resource` và `/metrics/probes`. Các metric đó không có cùng vòng đời.

Nếu cluster của bạn dùng RBAC, việc đọc metric yêu cầu được ủy quyền qua một user, group
hoặc ServiceAccount với một ClusterRole cho phép truy cập `/metrics`. Ví dụ:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
  - nonResourceURLs:
      - "/metrics"
    verbs:
      - get
```

## Vòng đời metric (Metric lifecycle)

Metric alpha → Metric beta → Metric ổn định (stable) → Metric bị loại bỏ dần (deprecated) → Metric ẩn (hidden) → Metric bị xóa (deleted)

Metric alpha không có bảo đảm nào về tính ổn định. Các metric này có thể bị sửa đổi hoặc
xóa bất cứ lúc nào.

Metric beta tuân theo một hợp đồng API lỏng hơn so với metric ổn định. Không nhãn (label)
nào có thể bị xóa khỏi metric beta trong suốt vòng đời của nó, tuy nhiên nhãn có thể được
thêm vào khi metric còn ở giai đoạn beta.

Metric ổn định được bảo đảm không thay đổi. Điều này nghĩa là:

* Một metric ổn định không có chữ ký deprecated sẽ không bị xóa hay bị đổi tên
* Kiểu của một metric ổn định sẽ không bị sửa đổi

Metric bị loại bỏ dần là các metric đã được lên lịch để xóa, nhưng vẫn còn dùng được.
Các metric này kèm theo một chú thích về phiên bản mà chúng bắt đầu bị loại bỏ dần.

Ví dụ:

* Trước khi bị loại bỏ dần

  ```
  # HELP some_counter this counts things
  # TYPE some_counter counter
  some_counter 0
  ```

* Sau khi bị loại bỏ dần

  ```
  # HELP some_counter (Deprecated since 1.15.0) this counts things
  # TYPE some_counter counter
  some_counter 0
  ```

Metric ẩn không còn được công bố để thu thập (scrape) nữa, nhưng vẫn còn dùng được.
Một metric bị loại bỏ dần sẽ trở thành metric ẩn sau một khoảng thời gian, tùy theo mức ổn
định của nó:
* Metric **STABLE** trở thành ẩn sau tối thiểu 3 bản phát hành hoặc 9 tháng, tùy theo mốc nào dài hơn.
* Metric **BETA** trở thành ẩn sau tối thiểu 1 bản phát hành hoặc 4 tháng, tùy theo mốc nào dài hơn.
* Metric **ALPHA** có thể bị ẩn hoặc bị xóa ngay trong chính bản phát hành mà chúng bị loại bỏ dần.

Để dùng một metric ẩn, bạn phải bật nó lên. Để biết thêm chi tiết, xem mục
[Hiển thị metric ẩn](#show-hidden-metrics).

Metric bị xóa không còn được công bố và không thể dùng được nữa.

## Hiển thị metric ẩn (Show hidden metrics) {#show-hidden-metrics}

Như đã mô tả ở trên, quản trị viên có thể bật các metric ẩn thông qua một cờ dòng lệnh trên
một binary cụ thể. Cách này được dùng như một lối thoát cho quản trị viên khi họ lỡ bỏ sót
việc di trú khỏi các metric đã bị loại bỏ dần trong bản phát hành trước.

Cờ `show-hidden-metrics-for-version` nhận một phiên bản mà bạn muốn hiển thị các metric bị
loại bỏ dần trong bản phát hành đó. Phiên bản được biểu diễn dưới dạng x.y, trong đó x là
số phiên bản chính (major), y là số phiên bản phụ (minor). Không cần số phiên bản vá (patch)
dù một metric có thể bị loại bỏ dần trong một bản phát hành vá; lý do là chính sách loại bỏ
dần metric được áp dụng theo bản phát hành minor.

Cờ này chỉ có thể nhận phiên bản minor liền trước làm giá trị. Nếu bạn muốn hiển thị tất cả
metric bị ẩn trong bản phát hành trước, bạn có thể đặt cờ `show-hidden-metrics-for-version`
thành phiên bản trước đó. Không được phép dùng một phiên bản quá cũ vì như vậy vi phạm chính
sách loại bỏ dần metric.

Ví dụ, giả sử metric `A` bị loại bỏ dần trong `1.29`. Phiên bản mà metric `A` trở thành ẩn
phụ thuộc vào mức ổn định của nó:
* Nếu metric `A` là **ALPHA**, nó có thể bị ẩn ngay trong `1.29`.
* Nếu metric `A` là **BETA**, sớm nhất nó sẽ bị ẩn trong `1.30`. Nếu bạn nâng cấp lên `1.30` và vẫn cần `A`, bạn phải dùng cờ dòng lệnh `--show-hidden-metrics-for-version=1.29`.
* Nếu metric `A` là **STABLE**, sớm nhất nó sẽ bị ẩn trong `1.32`. Nếu bạn nâng cấp lên `1.32` và vẫn cần `A`, bạn phải dùng cờ dòng lệnh `--show-hidden-metrics-for-version=1.31`.

## Metric của các thành phần (Component metrics)

### Metric của kube-controller-manager (kube-controller-manager metrics)

Metric của controller manager cung cấp cái nhìn quan trọng về hiệu năng và tình trạng của
controller manager. Các metric này bao gồm các metric runtime thông dụng của ngôn ngữ Go
như số lượng go_routine và các metric riêng của controller như độ trễ request tới etcd hoặc
độ trễ API của Cloudprovider (AWS, GCE, OpenStack), có thể dùng để đánh giá tình trạng của
cluster.

Từ Kubernetes 1.7, các metric Cloudprovider chi tiết đã khả dụng cho các thao tác lưu trữ
đối với GCE, AWS, Vsphere và OpenStack.
Các metric này có thể được dùng để theo dõi tình trạng của các thao tác persistent volume.

Ví dụ, với GCE các metric này có tên:

```
cloudprovider_gce_api_request_duration_seconds { request = "instance_list"}
cloudprovider_gce_api_request_duration_seconds { request = "disk_insert"}
cloudprovider_gce_api_request_duration_seconds { request = "disk_delete"}
cloudprovider_gce_api_request_duration_seconds { request = "attach_disk"}
cloudprovider_gce_api_request_duration_seconds { request = "detach_disk"}
cloudprovider_gce_api_request_duration_seconds { request = "list_disk"}
```


### Metric của kube-scheduler (kube-scheduler metrics)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.21 [beta]`

Scheduler expose các metric tùy chọn báo cáo tài nguyên được yêu cầu (requested resources)
và giới hạn mong muốn (desired limits) của tất cả các pod đang chạy. Các metric này có thể
được dùng để xây dựng dashboard hoạch định dung lượng (capacity planning), đánh giá các
giới hạn lập lịch hiện tại hoặc trong quá khứ, nhanh chóng nhận diện các workload không thể
được lập lịch do thiếu tài nguyên, và so sánh mức sử dụng thực tế với request của pod.

kube-scheduler nhận diện [request và limit](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
tài nguyên được cấu hình cho từng Pod; khi request hoặc limit khác 0, kube-scheduler báo
cáo một chuỗi thời gian (timeseries) metric. Chuỗi thời gian này được gắn nhãn theo:

- namespace
- tên pod
- node nơi pod được lập lịch, hoặc chuỗi rỗng nếu pod chưa được lập lịch
- độ ưu tiên (priority)
- scheduler được gán cho pod đó
- tên của tài nguyên (ví dụ, `cpu`)
- đơn vị của tài nguyên nếu biết (ví dụ, `cores`)

Khi một pod đạt trạng thái hoàn tất (có `restartPolicy` là `Never` hoặc `OnFailure` và đang
ở pha `Succeeded` hoặc `Failed`, hoặc đã bị xóa và tất cả container đều ở trạng thái
terminated) thì chuỗi này không còn được báo cáo nữa, vì lúc này scheduler đã rảnh để lập
lịch cho các pod khác chạy.
Hai metric này có tên `kube_pod_resource_request` và `kube_pod_resource_limit`.

Các metric được expose tại endpoint HTTP `/metrics/resources`. Chúng yêu cầu được ủy quyền
cho endpoint `/metrics/resources`, thường được cấp bởi một ClusterRole với verb `get` cho
non-resource URL `/metrics/resources`.

Trên Kubernetes 1.21, bạn phải dùng cờ `--show-hidden-metrics-for-version=1.20` để expose
các metric ở mức ổn định alpha này.

### Metric Pressure Stall Information (PSI) của kubelet (kubelet Pressure Stall Information (PSI) metrics)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [stable]`

Khi kernel bật PSI (phiên bản 4.20 trở lên), kubelet thu thập
[Pressure Stall Information](https://docs.kernel.org/accounting/psi.html)
(PSI) cho mức sử dụng CPU, memory và I/O.
Thông tin này được thu thập ở cấp node, pod và container.

*Metric Prometheus*: Được expose tại endpoint `/metrics/cadvisor` dưới dạng các bộ đếm tích
lũy (tổng) biểu diễn tổng thời gian đình trệ (stall) tính bằng giây. Các metric được expose
tại endpoint này với các tên sau:

```
container_pressure_cpu_stalled_seconds_total
container_pressure_cpu_waiting_seconds_total
container_pressure_memory_stalled_seconds_total
container_pressure_memory_waiting_seconds_total
container_pressure_io_stalled_seconds_total
container_pressure_io_waiting_seconds_total
```
*Summary API*: Được expose tại endpoint `/stats/summary`, cung cấp cả tổng tích lũy
(`totals`) lẫn các trung bình trượt (`avg10`, `avg60`, `avg300`) ở định dạng JSON. Các
trung bình này biểu diễn phần trăm thời gian mà các tác vụ bị đình trệ trên một tài nguyên
trong các khoảng thời gian tương ứng 10 giây, 60 giây và 5 phút.

Các metric này cũng được xuất một cách tự nhiên (natively) qua file tương ứng của node trong
`/proc/pressure/` -- cpu, memory và io, theo định dạng sau:

```
some avg10=0.00 avg60=0.00 avg300=0.00 total=0
full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

Làm thế nào để diễn giải các metric này cùng với nhau? Lấy ví dụ truy vấn sau từ Summary API:
`kubectl get --raw "/api/v1/nodes/$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')/proxy/stats/summary" | jq '.pods[].containers[] | select(.name=="<CONTAINER_NAME>") | {name, cpu: .cpu.psi, memory: .memory.psi, io: .io.psi}'`.
Truy vấn này trả về thông tin ở định dạng json như sau.

```
{
  "name": "<CONTAINER_NAME>",
  "cpu": {
    "full": {
      "total": 0,
      "avg10": 0,
      "avg60": 0,
      "avg300": 0
    },
    "some": {
      "total": 35232438,
      "avg10": 0.74,
      "avg60": 0.52,
      "avg300": 0.21,
    },  
  },
  "memory": {
    "full": {
      "total": 539105,
      "avg10": 0,
      "avg60": 0,
      "avg300": 0
    },
    "some": {
      "total": 658164,
      "avg10": 0.01,
      "avg60": 0.01,
      "avg300": 0.00,
    },
    }
  },
  "io": {
    "full": {
      "total": 33190987,
      "avg10": 0.31,
      "avg60": 0.22,
      "avg300": 0.05,
    },
    "some": {
      "total": 40809937,
      "avg10": 0.52,
      "avg60": 0.45,
      "avg300": 0.12,
    }
  }
}
```

Đây là một kịch bản tăng đột biến (spike) đơn giản. Giá trị `avg10` của cpu.some là `0.74`
cho biết trong 10 giây gần nhất, có ít nhất một tác vụ trong container này bị đình trệ trên
CPU trong 0.74% thời gian (0.0074 giây, tức 74 mili-giây). Vì `avg10` (0.74) cao hơn đáng
kể so với `avg300` (0.21) trên cùng một tài nguyên, điều này gợi ý một đợt tăng tranh chấp
tài nguyên gần đây thay vì một điểm nghẽn kéo dài lâu nay. Nếu theo dõi liên tục và các
metric `avg300` cũng tăng theo, chúng ta có thể chẩn đoán một vấn đề nghiêm trọng và dai
dẳng hơn!

Ngoài ra, hãy để ý trong ví dụ này `cpu.some` cho thấy có áp lực (pressure), trong khi
`cpu.full` vẫn ở mức 0.00. Điều này cho biết dù một số tiến trình bị trễ do chờ đến lượt
CPU, container về tổng thể vẫn đang tiến triển. Một giá trị full khác 0 sẽ cho biết tất cả
các tác vụ không rảnh (non-idle) đều bị đình trệ đồng thời — một vấn đề lớn hơn nhiều.
Dù không dễ đọc với con người, giá trị `total` bằng 35232438 biểu diễn thời gian đình trệ
tích lũy tính bằng micro-giây, cho phép phát hiện các đợt tăng đột biến độ trễ mà có thể
không hiển thị trong các giá trị trung bình. Chúng cũng hữu ích cho các hệ thống giám sát,
như Prometheus, để tính toán chính xác tốc độ tăng trong các cửa sổ thời gian cụ thể.
Lưu ý cuối cùng, khi quan sát thấy I/O Pressure cao đi kèm Memory Pressure thấp, điều đó có
thể cho thấy ứng dụng đang chờ băng thông đĩa (disk throughput) chứ không phải thất bại do
thiếu RAM khả dụng. Node không bị cấp phát quá mức (over-committed) về memory, và có thể
điều tra một hướng chẩn đoán khác về mức tiêu thụ đĩa.

Metric PSI mở ra một cách thức mạnh mẽ hơn để giám sát tranh chấp tài nguyên theo thời gian
thực ở mọi cấp cho từng cgroup, mở ra cơ hội xử lý các workload một cách linh hoạt trên
toàn hệ thống. Bạn có thể đọc thêm về metric PSI tại
[Hiểu về metric PSI](https://kubernetes.io/docs/reference/instrumentation/understand-psi-metrics/).

#### Yêu cầu (Requirements)

Pressure Stall Information yêu cầu:

- [Kernel Linux phiên bản 4.20 trở lên](https://kubernetes.io/docs/reference/node/kernel-version-requirements#requirements-psi).
- [cgroup v2](https://kubernetes.io/docs/concepts/architecture/cgroups)

## Tắt metric (Disabling metrics)

Bạn có thể tắt tường minh các metric qua cờ dòng lệnh `--disabled-metrics`. Điều này có thể
hữu ích nếu, chẳng hạn, một metric đang gây ra vấn đề hiệu năng. Đầu vào là danh sách các
metric bị tắt (ví dụ `--disabled-metrics=metric1,metric2`).

## Cưỡng chế cardinality của metric (Metric cardinality enforcement)

Các metric với số chiều (dimension) không giới hạn có thể gây ra vấn đề bộ nhớ trong các
thành phần mà chúng đo đạc. Để giới hạn mức sử dụng tài nguyên, bạn có thể dùng tùy chọn
dòng lệnh `--allow-metric-labels` để cấu hình động một danh sách cho phép (allow-list) các
giá trị nhãn cho một metric.

Ở giai đoạn alpha, cờ này chỉ có thể nhận vào một loạt các ánh xạ (mapping) làm allow-list
nhãn của metric.
Mỗi ánh xạ có định dạng `<metric_name>,<label_name>=<allowed_labels>` trong đó
`<allowed_labels>` là danh sách các tên nhãn được chấp nhận, phân tách bằng dấu phẩy.

Định dạng tổng thể trông như sau:

```
--allow-metric-labels <metric_name>,<label_name>='<allow_value1>, <allow_value2>...', <metric_name2>,<label_name>='<allow_value1>, <allow_value2>...', ...
```

Đây là một ví dụ:

```none
--allow-metric-labels number_count_metric,odd_number='1,3,5', number_count_metric,even_number='2,4,6', date_gauge_metric,weekend='Saturday,Sunday'
```

Ngoài việc chỉ định từ CLI, việc này cũng có thể được thực hiện trong một file cấu hình. Bạn
có thể chỉ định đường dẫn tới file cấu hình đó bằng đối số dòng lệnh
`--allow-metric-labels-manifest` cho một thành phần. Đây là một ví dụ về nội dung của file
cấu hình đó:

```yaml
"metric1,label2": "v1,v2,v3"
"metric2,label1": "v1,v2,v3"
```

Ngoài ra, meta-metric `cardinality_enforcement_unexpected_categorizations_total` ghi nhận
số lần phân loại ngoài dự kiến trong quá trình cưỡng chế cardinality, tức là mỗi khi gặp
một giá trị nhãn không được phép theo các ràng buộc của allow-list.

## Tiếp theo (What's next)

* Đọc về [định dạng văn bản Prometheus](https://github.com/prometheus/docs/blob/main/docs/instrumenting/exposition_formats.md#text-based-format)
  cho metric
* Xem danh sách [các metric ổn định của Kubernetes](https://github.com/kubernetes/kubernetes/blob/master/test/instrumentation/testdata/stable-metrics-list.yaml)
* Đọc về [chính sách loại bỏ dần của Kubernetes](https://kubernetes.io/docs/reference/using-api/deprecation-policy/#deprecating-a-feature-or-behavior)

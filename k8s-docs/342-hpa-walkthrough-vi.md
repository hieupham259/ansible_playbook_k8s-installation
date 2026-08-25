# Hướng dẫn từng bước về HorizontalPodAutoscaler (HorizontalPodAutoscaler Walkthrough)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/>
>
> Tài liệu này dẫn bạn qua một ví dụ về việc bật HorizontalPodAutoscaler để tự động quản lý
> quy mô (scale) cho một ứng dụng web mẫu.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 11 — Observability](00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability)
→ dòng **Thực hành**, bài 7/7 — bài cuối của nhóm · Kiểm chứng ở
[Lab 11b — HPA và VPA](labs/LAB-11B-HPA-VA-VPA.md) phần B3 (workload đích và Service), B4 (tạo
HPA bằng hai cách, và các status condition lúc nghỉ), B6 (tăng tải rồi dừng tải) và B8 (HPA gặp
`kubectl scale`). Đây là bài duy nhất của nhóm không kiểm chứng ở
[Lab 11a](labs/LAB-11A-OBSERVABILITY.md), vì nó cần metrics-server mà Lab 11a mới cài xong.

Bài dài nhất nhóm và **hơn một nửa nội dung không dùng được trên cluster lab**: từ mục
*Autoscaling dựa trên nhiều metric và metric tùy chỉnh* trở đi, mọi thứ đều cần một adapter
`custom.metrics.k8s.io` hoặc `external.metrics.k8s.io` của bên thứ ba. Phần bạn thật sự làm là
bốn mục đầu cộng hai phụ lục. Đây cũng là chỗ **trả ⏳ [nợ #1](00-ALO-TRINH-ADMIN.md#sổ-nợ-lộ-trình)
phần HPA**: bạn đã đọc lý thuyết ở bài [72](72-horizontal-pod-autoscale-vi.md) từ
[giai đoạn 4](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller) nhưng phải hoãn thực hành
vì chưa có metrics-server. Nếu bài 72 đã mờ, đọc lại nó trước.

**Phải hiểu ở lần đọc này:**

- Trình tự tối thiểu của bài: dựng Deployment `php-apache` (mỗi Pod khai `requests.cpu: 200m`)
  cùng một Service, rồi `kubectl autoscale deployment php-apache --cpu=50% --min=1 --max=10`.
  Cùng một object đó có thể tạo theo cách khai báo bằng manifest `autoscaling/v2` ở mục
  *Tạo autoscaler theo cách khai báo*.
- Mục tiêu `50%` là **phần trăm của `requests`**, không phải của dung lượng node: bài tính ra
  mức trung bình mục tiêu là 100 milli-core trên mỗi Pod. Cột `TARGET` của `kubectl get hpa` là
  giá trị trung bình **trên tất cả các Pod** do Deployment quản lý.
- Điều kiện tiên quyết ở mục *Trước khi bạn bắt đầu*: cluster phải có **Metrics Server** — nó
  thu metric tài nguyên từ các kubelet và phục vụ qua [Kubernetes API](21-kubernetes-api-vi.md)
  bằng một [APIService](180-apiserver-aggregation-vi.md). Không có nó thì HPA không có số đo.
- Đường đi của một lần scale (mục *Tạo HorizontalPodAutoscaler*): HPA **cập nhật Deployment** →
  Deployment cập nhật ReplicaSet → ReplicaSet thêm hoặc bớt Pod. Ở mục *Tăng tải* và *Dừng tạo
  tải* bạn thấy nó chạy hai chiều, và bài nói rõ số replica cuối **có thể khác ví dụ** vì tải
  không được kiểm soát, cộng thêm việc ổn định mất vài phút.
- Ba status condition ở phụ lục *Các status condition*: `AbleToScale` (HPA lấy và cập nhật được
  scale, không bị backoff chặn), `ScalingActive` (tính được số replica mong muốn — `False`
  thường là **vấn đề lấy metric**), `ScalingLimited` (khuyến nghị đã bị `minReplicas` hoặc
  `maxReplicas` cắt). Cộng mục *Đại lượng*: `10500m` chính là `10.5`, nên một giá trị có thể
  hiện lúc là `1`, lúc là `1500m`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Autoscaling dựa trên nhiều metric và metric tùy chỉnh* — `type: Pods` và `type: Object` | cần một adapter `custom.metrics.k8s.io` của bên thứ ba; metrics-server không cung cấp API đó | [giai đoạn 23](00-ALO-TRINH-ADMIN.md#giai-đoạn-23--giám-sát-và-cảnh-báo) |
| Mục *Autoscaling dựa trên các metric cụ thể hơn* — label selector cho metric | cùng lý do: selector được chuyển cho pipeline metric bên ngoài, mà cluster lab không có | [giai đoạn 23](00-ALO-TRINH-ADMIN.md#giai-đoạn-23--giám-sát-và-cảnh-báo) |
| Mục *Autoscaling dựa trên các metric không liên quan đến object Kubernetes* — `type: External` | cần hệ thống giám sát ngoài cluster; chính bài khuyên hạn chế expose API này | [giai đoạn 23](00-ALO-TRINH-ADMIN.md#giai-đoạn-23--giám-sát-và-cảnh-báo) |
| Câu `minikube addons enable metrics-server` ở mục *Trước khi bạn bắt đầu* | lộ trình không dùng minikube; metrics-server được cài bằng tay và có chẩn đoán lỗi certificate đi kèm | [Lab 11a](labs/LAB-11A-OBSERVABILITY.md) phần B5 |
| Image `registry.k8s.io/hpa-example` và Pod `load-generator` chạy vòng lặp vô hạn | image nằm ngoài bộ đã khóa của baseline, và một vòng lặp không có trần rất dễ làm nghẽn hai worker nhỏ của cluster lab | [Lab 11b](labs/LAB-11B-HPA-VA-VPA.md) phần B3 và B6, nơi lab thay bằng image đã khóa và một Job có trần lẫn deadline |

---

Một [HorizontalPodAutoscaler](72-horizontal-pod-autoscale-vi.md)
(gọi tắt là HPA)
tự động cập nhật một tài nguyên workload (chẳng hạn một Deployment hoặc StatefulSet), với mục
tiêu tự động scale workload cho khớp với nhu cầu.

Scale theo chiều ngang (horizontal scaling) nghĩa là phản ứng trước tải tăng lên bằng cách
triển khai thêm Pod. Điều này khác với scale theo chiều *dọc* (vertical scaling), mà đối với
Kubernetes có nghĩa là gán thêm tài nguyên (ví dụ: bộ nhớ hoặc CPU) cho các Pod đang chạy sẵn
của workload đó.

Nếu tải giảm xuống, và số lượng Pod đang cao hơn mức tối thiểu đã cấu hình,
HorizontalPodAutoscaler sẽ chỉ thị cho tài nguyên workload (Deployment, StatefulSet,
hoặc tài nguyên tương tự khác) scale xuống trở lại.

Tài liệu này dẫn bạn qua một ví dụ về việc bật HorizontalPodAutoscaler để tự động quản lý
quy mô cho một ứng dụng web mẫu. Workload ví dụ này là Apache httpd chạy một đoạn mã PHP.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò máy chủ control plane. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
hoặc dùng một trong các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản 1.23 hoặc mới hơn. Để kiểm tra phiên bản, hãy chạy
`kubectl version`. Nếu bạn đang chạy một bản phát hành Kubernetes cũ hơn, hãy tham khảo phiên
bản tài liệu tương ứng với bản phát hành đó (xem
[các phiên bản tài liệu hiện có](https://kubernetes.io/docs/home/supported-doc-versions/)).

Để làm theo hướng dẫn này, bạn cũng cần dùng một cluster đã triển khai và cấu hình
[Metrics Server](https://github.com/kubernetes-sigs/metrics-server#readme).
Kubernetes Metrics Server thu thập các metric tài nguyên từ các kubelet trong cluster của bạn,
và cung cấp các metric đó thông qua
[Kubernetes API](21-kubernetes-api-vi.md),
sử dụng một [APIService](180-apiserver-aggregation-vi.md)
để thêm các loại tài nguyên mới đại diện cho các số đọc metric.

Để tìm hiểu cách triển khai Metrics Server, xem
[tài liệu metrics-server](https://github.com/kubernetes-sigs/metrics-server#deployment).

Nếu bạn đang chạy minikube, hãy chạy lệnh sau để bật metrics-server:

```shell
minikube addons enable metrics-server
```

## Chạy và expose server php-apache (Run and expose php-apache server)

Để minh họa một HorizontalPodAutoscaler, trước tiên bạn sẽ khởi động một Deployment chạy
container dùng image `hpa-example`, và expose nó thành một Service bằng manifest sau:

**`application/php-apache.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache
```

Để làm việc đó, hãy chạy lệnh sau:

```shell
kubectl apply -f https://k8s.io/examples/application/php-apache.yaml
```

```
deployment.apps/php-apache created
service/php-apache created
```

## Tạo HorizontalPodAutoscaler (Create the HorizontalPodAutoscaler) {#create-horizontal-pod-autoscaler}

Bây giờ server đã chạy, hãy tạo autoscaler bằng `kubectl`. Lệnh con
[`kubectl autoscale`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#autoscale),
một phần của `kubectl`, sẽ giúp bạn làm việc này.

Ngay sau đây bạn sẽ chạy một lệnh tạo ra một HorizontalPodAutoscaler duy trì từ 1 đến 10
replica của các Pod do Deployment php-apache — mà bạn đã tạo ở bước đầu tiên của hướng dẫn
này — quản lý.

Nói một cách đại khái, controller HPA sẽ tăng và giảm số lượng replica (bằng cách cập nhật
Deployment) để duy trì mức sử dụng CPU trung bình trên tất cả các Pod ở 50%.
Deployment sau đó cập nhật ReplicaSet — đây là một phần trong cách mọi Deployment hoạt động
trong Kubernetes — và rồi ReplicaSet thêm hoặc bớt Pod dựa trên thay đổi trong `.spec` của nó.

Vì mỗi Pod yêu cầu (request) 200 milli-core thông qua `kubectl run`, điều này đồng nghĩa với
mức sử dụng CPU trung bình là 100 milli-core.
Xem [Chi tiết thuật toán](72-horizontal-pod-autoscale-vi.md#algorithm-details)
để biết thêm chi tiết về thuật toán.

Tạo HorizontalPodAutoscaler:

```shell
kubectl autoscale deployment php-apache --cpu=50% --min=1 --max=10
```

```
horizontalpodautoscaler.autoscaling/php-apache autoscaled
```

Bạn có thể kiểm tra trạng thái hiện tại của HorizontalPodAutoscaler vừa tạo bằng cách chạy:

```shell
# Bạn có thể dùng "hpa" hoặc "horizontalpodautoscaler"; tên nào cũng được.
kubectl get hpa
```

Kết quả tương tự như:

```
NAME         REFERENCE                     TARGET    MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache/scale   0% / 50%  1         10        1          18s
```

(nếu bạn thấy các HorizontalPodAutoscaler khác với tên khác, điều đó nghĩa là chúng đã tồn tại
từ trước, và thường không phải là vấn đề).

Lưu ý rằng mức tiêu thụ CPU hiện tại là 0% vì chưa có client nào gửi request đến server
(cột ``TARGET`` hiển thị giá trị trung bình trên tất cả các Pod do Deployment tương ứng
quản lý).

## Tăng tải (Increase the load) {#increase-load}

Tiếp theo, hãy xem autoscaler phản ứng thế nào với tải tăng lên.
Để làm điều này, bạn sẽ khởi động một Pod khác đóng vai trò client. Container bên trong Pod
client chạy trong một vòng lặp vô hạn, gửi truy vấn đến service php-apache.

```shell
# Chạy lệnh này trong một terminal riêng
# để việc tạo tải tiếp diễn trong khi bạn thực hiện tiếp các bước còn lại
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

Bây giờ chạy:

```shell
# nhấn Ctrl+C để kết thúc việc theo dõi khi bạn đã sẵn sàng
kubectl get hpa php-apache --watch
```

Trong vòng khoảng một phút, bạn sẽ thấy tải CPU cao hơn; ví dụ:

```
NAME         REFERENCE                     TARGET      MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache/scale   305% / 50%  1         10        1          3m
```

và sau đó là nhiều replica hơn. Ví dụ:

```
NAME         REFERENCE                     TARGET      MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache/scale   305% / 50%  1         10        7          3m
```

Ở đây, mức tiêu thụ CPU đã tăng lên 305% so với request.
Kết quả là Deployment đã được thay đổi kích thước thành 7 replica:

```shell
kubectl get deployment php-apache
```

Bạn sẽ thấy số lượng replica khớp với con số từ HorizontalPodAutoscaler:

```
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
php-apache   7/7      7           7           19m
```

> **Ghi chú:**
> Có thể mất vài phút để số lượng replica ổn định. Vì lượng tải không được kiểm soát theo bất
> kỳ cách nào, số lượng replica cuối cùng có thể khác với ví dụ này.

## Dừng tạo tải (Stop generating load) {#stop-load}

Để kết thúc ví dụ, hãy dừng gửi tải.

Trong terminal nơi bạn đã tạo Pod chạy image `busybox`, kết thúc việc tạo tải bằng cách nhấn
`<Ctrl> + C`.

Sau đó xác minh trạng thái kết quả (sau khoảng một phút):

```shell
# nhấn Ctrl+C để kết thúc việc theo dõi khi bạn đã sẵn sàng
kubectl get hpa php-apache --watch
```

Kết quả tương tự như:

```
NAME         REFERENCE                     TARGET       MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache/scale   0% / 50%     1         10        1          11m
```

và Deployment cũng cho thấy nó đã scale xuống:

```shell
kubectl get deployment php-apache
```

```
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
php-apache   1/1     1            1           27m
```

Một khi mức sử dụng CPU giảm về 0, HPA tự động scale số lượng replica trở về 1.

Việc autoscale các replica có thể mất vài phút.

## Autoscaling dựa trên nhiều metric và metric tùy chỉnh (Autoscaling on multiple metrics and custom metrics) {#autoscaling-on-multiple-metrics-and-custom-metrics}

Bạn có thể đưa thêm các metric bổ sung để dùng khi autoscale Deployment `php-apache`
bằng cách sử dụng phiên bản API `autoscaling/v2`.

Trước tiên, lấy YAML của HorizontalPodAutoscaler của bạn ở dạng `autoscaling/v2`:

```shell
kubectl get hpa php-apache -o yaml > /tmp/hpa-v2.yaml
```

Mở file `/tmp/hpa-v2.yaml` trong một trình soạn thảo, và bạn sẽ thấy YAML trông như thế này:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
status:
  observedGeneration: 1
  lastScaleTime: <some-time>
  currentReplicas: 1
  desiredReplicas: 1
  currentMetrics:
  - type: Resource
    resource:
      name: cpu
      current:
        averageUtilization: 0
        averageValue: 0
```

Hãy để ý rằng trường `targetCPUUtilizationPercentage` đã được thay bằng một mảng có tên
`metrics`. Metric mức sử dụng CPU là một *metric tài nguyên* (resource metric), vì nó được biểu
diễn dưới dạng phần trăm của một tài nguyên được khai báo trên các container của Pod. Lưu ý
rằng bạn có thể chỉ định các metric tài nguyên khác ngoài CPU. Mặc định, metric tài nguyên duy
nhất khác được hỗ trợ là `memory`. Các tài nguyên này không đổi tên giữa các cluster, và luôn
sẵn có, miễn là API `metrics.k8s.io` khả dụng.

Bạn cũng có thể chỉ định các metric tài nguyên theo giá trị trực tiếp, thay vì theo phần trăm
của giá trị request, bằng cách dùng `target.type` là `AverageValue` thay cho `Utilization`, và
đặt trường `target.averageValue` tương ứng thay cho `target.averageUtilization`.

```
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 500Mi
```

Có hai loại metric khác nữa, cả hai đều được coi là *metric tùy chỉnh* (custom metrics):
metric theo Pod (pod metrics) và metric theo object (object metrics). Các metric này có thể
mang tên đặc thù cho từng cluster, và đòi hỏi một hệ thống giám sát (monitoring) cluster
nâng cao hơn.

Loại metric thay thế đầu tiên là *pod metrics*. Các metric này mô tả các Pod, được lấy trung
bình trên các Pod và so sánh với một giá trị mục tiêu để xác định số lượng replica.
Chúng hoạt động rất giống metric tài nguyên, ngoại trừ việc chúng *chỉ* hỗ trợ kiểu `target`
là `AverageValue`.

Pod metrics được chỉ định bằng một khối metric như sau:

```yaml
type: Pods
pods:
  metric:
    name: packets-per-second
  target:
    type: AverageValue
    averageValue: 1k
```

Loại metric thay thế thứ hai là *object metrics*. Các metric này mô tả một object khác trong
cùng namespace, thay vì mô tả các Pod. Các metric không nhất thiết được lấy từ chính object đó;
chúng chỉ mô tả object đó mà thôi. Object metrics hỗ trợ cả hai kiểu `target` là `Value` và
`AverageValue`. Với `Value`, giá trị mục tiêu được so sánh trực tiếp với metric trả về từ API.
Với `AverageValue`, giá trị trả về từ API metric tùy chỉnh được chia cho số lượng Pod trước khi
so sánh với giá trị mục tiêu. Ví dụ sau là biểu diễn YAML của metric `requests-per-second`.

```yaml
type: Object
object:
  metric:
    name: requests-per-second
  describedObject:
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    name: main-route
  target:
    type: Value
    value: 2k
```

Nếu bạn cung cấp nhiều khối metric như vậy, HorizontalPodAutoscaler sẽ xem xét lần lượt từng
metric. HorizontalPodAutoscaler sẽ tính số lượng replica đề xuất cho từng metric, rồi chọn
metric cho số lượng replica cao nhất.

Ví dụ, nếu hệ thống giám sát của bạn thu thập metric về lưu lượng mạng, bạn có thể cập nhật
định nghĩa ở trên bằng `kubectl edit` để trông như thế này:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Pods
    pods:
      metric:
        name: packets-per-second
      target:
        type: AverageValue
        averageValue: 1k
  - type: Object
    object:
      metric:
        name: requests-per-second
      describedObject:
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        name: main-route
      target:
        type: Value
        value: 10k
status:
  observedGeneration: 1
  lastScaleTime: <some-time>
  currentReplicas: 1
  desiredReplicas: 1
  currentMetrics:
  - type: Resource
    resource:
      name: cpu
    current:
      averageUtilization: 0
      averageValue: 0
  - type: Object
    object:
      metric:
        name: requests-per-second
      describedObject:
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        name: main-route
      current:
        value: 10k
```

Khi đó, HorizontalPodAutoscaler của bạn sẽ cố gắng bảo đảm rằng mỗi Pod tiêu thụ khoảng
50% CPU đã request, phục vụ 1000 gói tin mỗi giây, và tất cả các Pod đứng sau Ingress
main-route phục vụ tổng cộng 10000 request mỗi giây.

### Autoscaling dựa trên các metric cụ thể hơn (Autoscaling on more specific metrics)

Nhiều pipeline metric cho phép bạn mô tả metric theo tên hoặc theo một tập các bộ mô tả bổ
sung gọi là _label_. Với mọi loại metric không phải tài nguyên (pod, object, và external —
được mô tả bên dưới), bạn có thể chỉ định thêm một label selector, selector này được chuyển
cho pipeline metric của bạn. Chẳng hạn, nếu bạn thu thập metric `http_requests` kèm label
`verb`, bạn có thể chỉ định khối metric sau để chỉ scale dựa trên các request GET:

```yaml
type: Object
object:
  metric:
    name: http_requests
    selector: {matchLabels: {verb: GET}}
```

Selector này dùng cùng cú pháp với các label selector đầy đủ của Kubernetes. Pipeline giám sát
quyết định cách gộp nhiều chuỗi dữ liệu (series) thành một giá trị duy nhất, nếu tên và
selector khớp với nhiều series. Selector chỉ mang tính bổ sung (additive), và không thể chọn
các metric mô tả những object **không phải** object mục tiêu (các Pod mục tiêu trong trường
hợp loại `Pods`, và object được mô tả trong trường hợp loại `Object`).

### Autoscaling dựa trên các metric không liên quan đến object Kubernetes (Autoscaling on metrics not related to Kubernetes objects) {#autoscaling-on-metrics-not-related-to-kubernetes-objects}

Các ứng dụng chạy trên Kubernetes có thể cần autoscale dựa trên những metric không có mối liên
hệ rõ ràng với bất kỳ object nào trong cluster Kubernetes, chẳng hạn các metric mô tả một dịch
vụ được host bên ngoài, không có tương quan trực tiếp với các namespace Kubernetes. Trong
Kubernetes 1.10 trở lên, bạn có thể giải quyết trường hợp sử dụng này bằng *external metrics*
(metric bên ngoài).

Việc dùng external metrics đòi hỏi hiểu biết về hệ thống giám sát của bạn; cách thiết lập
tương tự như khi dùng metric tùy chỉnh. External metrics cho phép bạn autoscale cluster dựa
trên bất kỳ metric nào sẵn có trong hệ thống giám sát của bạn. Hãy cung cấp một khối `metric`
với `name` và `selector` như trên, và dùng loại metric `External` thay cho `Object`.
Nếu `metricSelector` khớp với nhiều chuỗi thời gian (time series),
HorizontalPodAutoscaler sẽ dùng tổng các giá trị của chúng.
External metrics hỗ trợ cả hai kiểu target `Value` và `AverageValue`, hoạt động giống hệt như
khi bạn dùng loại `Object`.

Ví dụ, nếu ứng dụng của bạn xử lý các tác vụ từ một dịch vụ hàng đợi được host bên ngoài, bạn
có thể thêm đoạn sau vào manifest HorizontalPodAutoscaler để chỉ định rằng bạn cần một worker
cho mỗi 30 tác vụ đang chờ xử lý.

```yaml
- type: External
  external:
    metric:
      name: queue_messages_ready
      selector:
        matchLabels:
          queue: "worker_tasks"
    target:
      type: AverageValue
      averageValue: 30
```

Khi có thể, nên ưu tiên dùng các kiểu target metric tùy chỉnh thay vì external metrics, vì
việc bảo mật API metric tùy chỉnh dễ dàng hơn đối với quản trị viên cluster. API external
metrics có khả năng cho phép truy cập vào bất kỳ metric nào, vì vậy quản trị viên cluster nên
cẩn trọng khi expose nó.

## Phụ lục: Các status condition của Horizontal Pod Autoscaler (Appendix: Horizontal Pod Autoscaler Status Conditions)

Khi dùng dạng `autoscaling/v2` của HorizontalPodAutoscaler, bạn sẽ có thể thấy các
*status condition* (điều kiện trạng thái) do Kubernetes đặt trên HorizontalPodAutoscaler.
Các status condition này cho biết HorizontalPodAutoscaler có khả năng scale hay không, và
hiện tại nó có đang bị hạn chế theo cách nào hay không.

Các condition xuất hiện trong trường `status.conditions`. Để xem các condition đang ảnh hưởng
đến một HorizontalPodAutoscaler, chúng ta có thể dùng `kubectl describe hpa`:

```shell
kubectl describe hpa cm-test
```

```
Name:                           cm-test
Namespace:                      prom
Labels:                         <none>
Annotations:                    <none>
CreationTimestamp:              Fri, 16 Jun 2017 18:09:22 +0000
Reference:                      ReplicationController/cm-test
Metrics:                        ( current / target )
  "http_requests" on pods:      66m / 500m
Min replicas:                   1
Max replicas:                   4
ReplicationController pods:     1 current / 1 desired
Conditions:
  Type                  Status  Reason                  Message
  ----                  ------  ------                  -------
  AbleToScale           True    ReadyForNewScale        the last scale time was sufficiently old as to warrant a new scale
  ScalingActive         True    ValidMetricFound        the HPA was able to successfully calculate a replica count from pods metric http_requests
  ScalingLimited        False   DesiredWithinRange      the desired replica count is within the acceptable range
Events:
```

Với HorizontalPodAutoscaler này, bạn có thể thấy một số condition ở trạng thái khỏe mạnh.
Condition đầu tiên, `AbleToScale`, cho biết HPA có thể lấy và cập nhật scale hay không, cũng
như liệu có condition nào liên quan đến backoff đang ngăn việc scale hay không. Condition thứ
hai, `ScalingActive`, cho biết HPA có đang được bật hay không (tức là số replica của mục tiêu
khác 0) và có thể tính toán scale mong muốn hay không. Khi nó là `False`, điều đó thường cho
thấy có vấn đề trong việc lấy metric. Cuối cùng, condition `ScalingLimited` cho biết scale
mong muốn đã bị giới hạn bởi giá trị tối đa hoặc tối thiểu của HorizontalPodAutoscaler. Đây
là dấu hiệu cho thấy bạn có thể muốn nâng hoặc hạ các ràng buộc về số replica tối thiểu hoặc
tối đa trên HorizontalPodAutoscaler của mình.

## Đại lượng (Quantities)

Tất cả các metric trong HorizontalPodAutoscaler và các API metric đều được chỉ định bằng một
ký pháp số nguyên đặc biệt, được gọi trong Kubernetes là quantity (đại lượng). Ví dụ, đại
lượng `10500m` sẽ được viết là `10.5` theo ký pháp thập phân. Các API metric sẽ trả về số
nguyên không có hậu tố khi có thể, và nói chung sẽ trả về các đại lượng theo đơn vị milli
trong các trường hợp còn lại. Điều này nghĩa là bạn có thể thấy giá trị metric của mình dao
động giữa `1` và `1500m`, hoặc giữa `1` và `1.5` khi viết theo ký pháp thập phân.

## Các kịch bản khả dĩ khác (Other possible scenarios)

### Tạo autoscaler theo cách khai báo (Creating the autoscaler declaratively)

Thay vì dùng lệnh `kubectl autoscale` để tạo HorizontalPodAutoscaler theo cách mệnh lệnh
(imperative), chúng ta có thể dùng manifest sau để tạo nó theo cách khai báo (declarative):

**`application/hpa/php-apache.yaml`**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Sau đó, tạo autoscaler bằng cách thực thi lệnh sau:

```shell
kubectl create -f https://k8s.io/examples/application/hpa/php-apache.yaml
```

```
horizontalpodautoscaler.autoscaling/php-apache created
```

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 11:

1. `kubectl autoscale deployment php-apache --cpu=50%` — 50% của cái gì? Với Deployment
   `php-apache` của bài, con số tuyệt đối tương ứng trên mỗi Pod là bao nhiêu, và cột `TARGET`
   của `kubectl get hpa` là giá trị của một Pod hay của cả nhóm?
2. **Câu bẫy.** HPA có tự tạo và tự xóa Pod không? Kể lại đúng chuỗi mắt xích từ HPA tới Pod.
3. Trên cluster lab, metrics-server chỉ được cài ở [Lab 11a](labs/LAB-11A-OBSERVABILITY.md) —
   trước đó `kubectl top node` chạy trên `lab-k8s-master` không trả về số liệu nào. Vì sao không
   thể chạy bài này ngay từ giai đoạn 4 lúc bạn đọc bài
   [72](72-horizontal-pod-autoscale-vi.md)? Mục *Trước khi bạn bắt đầu* nêu điều kiện gì, và
   Metrics Server phục vụ số liệu qua đường nào?
4. `kubectl describe hpa` cho `ScalingActive: False`. Bài nói dấu hiệu đó thường chỉ ra chuyện
   gì? Còn `ScalingLimited: True` nghĩa là gì và gợi ý bạn chỉnh trường nào?
5. Bài nói một giá trị metric có thể "dao động giữa `1` và `1500m`". `1500m` bằng bao nhiêu theo
   ký pháp thập phân, và vì sao cùng một đại lượng lại hiện ra ở hai dạng khác nhau?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. 50% của **lượng CPU mà container đã `request`**, không phải của dung lượng node. Mỗi Pod
   `php-apache` request **200 milli-core**, nên mục tiêu tuyệt đối là **100 milli-core mỗi Pod**.
   Cột `TARGET` là **giá trị trung bình trên tất cả các Pod** do Deployment tương ứng quản lý —
   bài nói thẳng điều này ngay sau ví dụ `kubectl get hpa`, chứ không phải số của một Pod.
2. **Không.** Đây là chỗ dễ sai vì nhìn `kubectl get hpa` thấy số replica đổi là tưởng HPA thao
   tác thẳng lên Pod. Chuỗi thật là: **HPA cập nhật Deployment → Deployment cập nhật ReplicaSet →
   ReplicaSet thêm hoặc bớt Pod dựa trên thay đổi trong `.spec` của nó**. Bài nhấn mạnh đây "là
   một phần trong cách mọi Deployment hoạt động trong Kubernetes", không phải cơ chế riêng của
   HPA.
3. Vì bài đòi **cluster đã triển khai và cấu hình Metrics Server**; không có nó thì HPA không có
   số đo nào để so với mục tiêu. Metrics Server **thu metric tài nguyên từ các kubelet** trong
   cluster và **phục vụ chúng qua Kubernetes API bằng một APIService** — tức là thêm loại tài
   nguyên mới đại diện cho các số đọc metric. Đó đúng là lý do lộ trình hoãn phần thực hành HPA
   từ giai đoạn 4 xuống đây (⏳ [nợ #1](00-ALO-TRINH-ADMIN.md#sổ-nợ-lộ-trình)).
4. `ScalingActive` cho biết HPA **có đang bật và có tính được scale mong muốn hay không**; khi nó
   là `False` thì điều đó **thường cho thấy có vấn đề trong việc lấy metric**. `ScalingLimited` là
   `True` nghĩa là **số replica mong muốn đã bị chặn bởi giá trị tối thiểu hoặc tối đa** của HPA
   — dấu hiệu bạn có thể muốn nâng hoặc hạ `minReplicas`/`maxReplicas`.
5. `1500m` là **`1.5`**. Mọi metric trong HPA và trong các API metric đều dùng ký pháp
   **quantity**: API trả **số nguyên không hậu tố khi có thể**, và trả **đơn vị milli** trong các
   trường hợp còn lại. Vì vậy cùng một đại lượng lúc hiện `1`, lúc hiện `1500m` — không phải hai
   giá trị khác nhau, chỉ là hai cách viết.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng. Đây là bài cuối của dòng **Thực hành**
giai đoạn 11: đọc xong thì mở [Lab 11b — HPA và VPA](labs/LAB-11B-HPA-VA-VPA.md), nơi
⏳ [nợ #1](00-ALO-TRINH-ADMIN.md#sổ-nợ-lộ-trình) được trả **phần HPA** trên chính mốc snapshot mà
[Lab 11a](labs/LAB-11A-OBSERVABILITY.md) vừa tạo. Trước khi vào lab, đọc lại bài
[72](72-horizontal-pod-autoscale-vi.md) — lab bám vào nó chứ không chỉ vào bài này. Phần VPA của
nợ #1 vẫn **chưa được trả**: add-on VPA không nằm trong bộ đã khóa của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md).

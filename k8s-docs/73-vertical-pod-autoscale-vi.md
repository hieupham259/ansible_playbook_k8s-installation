# Tự động co giãn Pod theo chiều dọc (Vertical Pod Autoscaling)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/autoscaling/vertical-pod-autoscale/>
>
> Tự động điều chỉnh resource request và limit dựa trên các mẫu sử dụng thực tế.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 4](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller), bài 13/14 ·
Kiểm chứng ở Lab 4 (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

**Như bài [72](72-horizontal-pod-autoscale-vi.md): ở đây bạn chỉ đọc lý thuyết.** VPA cần
hai thứ mà cluster baseline chưa có — chính add-on VPA (nó không đi kèm Kubernetes) và
Metrics Server của **giai đoạn 11**. Phần thực hành là [nợ lab](labs/README.md#5-sổ-nợ-lab),
trả ở **Lab 11b**. **Đừng cài metrics-server hay VPA sớm để chạy thử.** Lần đọc này chỉ cần
nắm ba thành phần của VPA và ranh giới giữa các `updateMode` — đó là thứ quyết định workload
của bạn có bị khởi động lại hay không.

**Phải hiểu ở lần đọc này:**

- VPA **không thuộc API lõi**: nó được định nghĩa dưới dạng một CustomResourceDefinition
  (`autoscaling.k8s.io/v1`) và phải cài riêng — khác hẳn HorizontalPodAutoscaler. Nó cũng
  **yêu cầu một nguồn metric**, các thành phần VPA lấy metric từ API `metrics.k8s.io`.
- Ba thành phần và việc của từng cái: **Recommender** phân tích mức sử dụng hiện tại và quá
  khứ rồi ghi khuyến nghị vào `.status.recommendation`; **Updater** so sánh request hiện tại
  với khuyến nghị và evict Pod hoặc sửa tại chỗ; **admission controller** là một mutating
  webhook chặn request tạo Pod và áp khuyến nghị **Target** lên Pod trước khi nó được tạo.
- Recommender sinh **ba mức** — Target, cận dưới, cận trên — và VPA chạy **theo từng đợt**,
  không phải một tiến trình liên tục.
- Ranh giới giữa các `updateMode`: `Off` chỉ khuyến nghị chứ không áp dụng; `Initial` chỉ đặt
  request lúc Pod được tạo lần đầu; `Recreate` **evict Pod** để áp giá trị mới;
  `InPlaceOrRecreate` thử sửa tại chỗ rồi mới quay về evict; `InPlace` **không bao giờ evict**
  mà hoãn lại và thử lại; `Auto` đã lỗi thời và hiện chỉ là bí danh của `Recreate`.
- `resourcePolicy` để chặn khuyến nghị đi quá xa: `minAllowed`/`maxAllowed` là biên cứng mà
  VPA không bao giờ vượt, `controlledResources` giới hạn loại tài nguyên (`cpu`, `memory`),
  và `controlledValues` quyết định VPA đặt cả request lẫn limit (`RequestsAndLimits`, mặc
  định) hay chỉ request (`RequestsOnly`).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Toàn bộ phần thao tác thật — cài VPA, tạo object VerticalPodAutoscaler | cần add-on VPA và Metrics Server | [nợ lab](labs/README.md#5-sổ-nợ-lab), trả ở Lab 11b |
| Mục *InPlace* — feature gate `InPlace` và `InPlacePodVerticalScaling`, VPA 1.7.0 | alpha, phải bật feature gate ở cả cluster lẫn hai thành phần VPA | không cần |
| *Tài nguyên LimitRange* | chưa học LimitRange | giai đoạn 7 |
| Cú pháp đầy đủ của `containerPolicies` trong ví dụ YAML | không cài VPA ở giai đoạn này | Lab 11b |

---

Trong Kubernetes, một _VerticalPodAutoscaler_ tự động cập nhật một tài nguyên quản lý
workload (chẳng hạn một Deployment hoặc StatefulSet), với mục tiêu tự động điều chỉnh
[request và limit](110-manage-resources-containers-vi.md#requests-and-limits)
tài nguyên hạ tầng cho khớp với mức sử dụng thực tế.

Co giãn theo chiều dọc (vertical scaling) nghĩa là phản ứng trước nhu cầu tài nguyên tăng
lên bằng cách gán thêm tài nguyên (ví dụ: memory hoặc CPU) cho các Pod đang chạy sẵn của
workload. Cách này còn được gọi là _rightsizing_ (định cỡ cho vừa), hoặc đôi khi là
_autopilot_. Nó khác với co giãn theo chiều ngang (horizontal scaling): với Kubernetes,
co giãn theo chiều ngang nghĩa là triển khai thêm Pod để phân tán tải.

Nếu mức sử dụng tài nguyên giảm xuống và resource request của Pod đang cao hơn mức tối
ưu, VerticalPodAutoscaler chỉ thị cho tài nguyên workload (Deployment, StatefulSet, hoặc
tài nguyên tương tự khác) điều chỉnh resource request giảm trở lại, tránh lãng phí tài
nguyên.

VerticalPodAutoscaler được hiện thực dưới dạng một tài nguyên API của Kubernetes và một
controller. Tài nguyên quyết định hành vi của controller. Controller tự động co giãn Pod
theo chiều dọc, chạy trong data plane của Kubernetes, định kỳ điều chỉnh resource request
và limit của đối tượng đích (ví dụ, một Deployment) dựa trên việc phân tích mức sử dụng
tài nguyên trong quá khứ, lượng tài nguyên còn khả dụng trong cluster, và các sự kiện
thời gian thực như tình trạng hết bộ nhớ (out-of-memory — OOM).

## Đối tượng API (API object)

VerticalPodAutoscaler được định nghĩa dưới dạng một Custom Resource Definition (CRD)
trong Kubernetes. Không giống HorizontalPodAutoscaler vốn là một phần của API lõi
Kubernetes, VPA phải được cài đặt riêng vào cluster của bạn.

Phiên bản API ổn định hiện tại là `autoscaling.k8s.io/v1`. Chi tiết thêm về việc cài đặt
VPA và API của nó có thể xem tại
[kho GitHub của VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler).

## VerticalPodAutoscaler hoạt động như thế nào? (How does a VerticalPodAutoscaler work?)

![Vertical Pod Autoscaling architecture](https://kubernetes.io/images/docs/concepts/vpa-architecture.svg)

*Hình 1. VerticalPodAutoscaler điều khiển resource request và limit của các Pod trong một Deployment*

Kubernetes hiện thực việc tự động co giãn Pod theo chiều dọc thông qua nhiều thành phần
phối hợp với nhau, chạy theo từng đợt (đây không phải là một tiến trình liên tục). VPA
gồm ba thành phần chính:

* _Recommender_ (bộ khuyến nghị), phân tích mức sử dụng tài nguyên và đưa ra các khuyến nghị.
* _Updater_ (bộ cập nhật), cập nhật resource request của Pod bằng cách evict (trục xuất)
  Pod hoặc sửa đổi chúng tại chỗ (in-place).
* Và webhook _admission controller_ của VPA, áp dụng các khuyến nghị tài nguyên cho các
  Pod mới hoặc Pod được tạo lại.

Mỗi chu kỳ một lần, Recommender truy vấn mức sử dụng tài nguyên của các Pod là đích của
từng định nghĩa VerticalPodAutoscaler. Recommender tìm tài nguyên đích được định nghĩa
bởi `targetRef`, sau đó chọn các pod dựa trên các label `.spec.selector` của tài nguyên
đích, và lấy metric từ resource metrics API để phân tích mức tiêu thụ CPU và memory
thực tế.

Recommender phân tích cả dữ liệu sử dụng tài nguyên hiện tại lẫn trong quá khứ (CPU và
memory) của từng Pod là đích của VerticalPodAutoscaler. Nó xem xét:
- Các mẫu tiêu thụ trong quá khứ theo thời gian để nhận diện xu hướng
- Mức sử dụng đỉnh và độ dao động để bảo đảm đủ khoảng dư (headroom)
- Các sự kiện hết bộ nhớ (OOM) và các sự cố khác liên quan đến tài nguyên

Dựa trên phân tích này, Recommender tính ra ba loại khuyến nghị:
- Khuyến nghị Target (mức tài nguyên tối ưu cho mức sử dụng thông thường)
- Cận dưới (mức tài nguyên tối thiểu chấp nhận được)
- Cận trên (mức tài nguyên hợp lý tối đa).

Các khuyến nghị này được lưu trong trường `.status.recommendation` của tài nguyên
VerticalPodAutoscaler.

Thành phần _updater_ theo dõi các tài nguyên VerticalPodAutoscaler và so sánh resource
request hiện tại của Pod với các khuyến nghị. Khi mức chênh lệch vượt quá các ngưỡng đã
cấu hình và update policy cho phép, updater có thể:

- Evict Pod, kích hoạt việc tạo lại chúng với resource request mới (cách tiếp cận truyền thống)
- Cập nhật tài nguyên của Pod tại chỗ mà không cần evict, khi cluster hỗ trợ cập nhật
  tài nguyên Pod tại chỗ

Phương thức được chọn phụ thuộc vào chế độ cập nhật đã cấu hình, khả năng của cluster, và
loại thay đổi tài nguyên cần thực hiện. Cập nhật tại chỗ, khi khả dụng, tránh làm gián
đoạn Pod nhưng có thể có hạn chế về việc những tài nguyên nào có thể được sửa đổi.
Updater tôn trọng PodDisruptionBudget để giảm thiểu ảnh hưởng đến dịch vụ.

_Admission controller_ hoạt động như một mutating webhook chặn các request tạo Pod. Nó
kiểm tra xem Pod có phải là đích của một VerticalPodAutoscaler hay không, và nếu có, áp
dụng resource request và limit được khuyến nghị trước khi Pod được tạo. Cụ thể hơn,
admission controller dùng khuyến nghị Target trong phần `.status.recommendation` của tài
nguyên VerticalPodAutoscaler làm resource request mới. Admission controller bảo đảm các
Pod mới khởi động với lượng tài nguyên được định cỡ phù hợp, dù chúng được tạo trong lần
triển khai ban đầu, sau khi bị updater evict, hay do các thao tác co giãn.

VerticalPodAutoscaler yêu cầu một nguồn metric, chẳng hạn add-on Metrics Server của
Kubernetes, được cài đặt trong cluster. Các thành phần VPA lấy metric từ API
`metrics.k8s.io`. Metrics Server cần được khởi chạy riêng vì nó không được triển khai
mặc định trong hầu hết các cluster. Để biết thêm thông tin về resource metrics, xem
[Metrics Server](311-resource-metrics-pipeline-vi.md#metrics-server).

## Các chế độ cập nhật (Update modes)

Một VerticalPodAutoscaler hỗ trợ nhiều _chế độ cập nhật_ (update mode) khác nhau, điều
khiển cách thức và thời điểm các khuyến nghị tài nguyên được áp dụng cho Pod của bạn. Bạn
cấu hình chế độ cập nhật bằng trường `updateMode` trong spec VPA, dưới `updatePolicy`:

```yaml
---
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Recreate"  # Off, Initial, Recreate, InPlaceOrRecreate, InPlace
```

### Off {#updateMode-Off}

Trong chế độ cập nhật _Off_, recommender của VPA vẫn phân tích mức sử dụng tài nguyên và
sinh ra các khuyến nghị, nhưng các khuyến nghị này không được tự động áp dụng cho Pod.
Các khuyến nghị chỉ được lưu trong trường `.status` của đối tượng VPA.

Bạn có thể dùng một công cụ như `kubectl` để xem `.status` và các khuyến nghị trong đó.

### Initial {#updateMode-Initial}

Trong chế độ _Initial_, VPA chỉ đặt resource request khi Pod được tạo lần đầu. Nó không
cập nhật tài nguyên cho các Pod đang chạy, kể cả khi khuyến nghị thay đổi theo thời gian.
Các khuyến nghị chỉ được áp dụng trong lúc tạo Pod.

### Recreate {#updateMode-Recreate}

Trong chế độ _Recreate_, VPA chủ động quản lý tài nguyên của Pod bằng cách evict Pod khi
resource request hiện tại của chúng chênh lệch đáng kể so với khuyến nghị. Khi một Pod bị
evict, workload controller (đang quản lý một Deployment, StatefulSet, v.v.) tạo một Pod
thay thế, và admission controller của VPA áp dụng resource request đã cập nhật cho Pod
mới.

### InPlaceOrRecreate {#updateMode-InPlaceOrRecreate}

Trong chế độ `InPlaceOrRecreate`, VPA cố gắng cập nhật resource request và limit của Pod
mà không khởi động lại Pod khi có thể. Tuy nhiên, nếu việc cập nhật tại chỗ không thể
thực hiện được cho một thay đổi tài nguyên cụ thể, VPA sẽ quay về (fall back) việc evict
Pod (tương tự chế độ `Recreate`) và để workload controller tạo một Pod thay thế với tài
nguyên đã cập nhật.

Trong chế độ này, updater áp dụng các khuyến nghị tại chỗ bằng tính năng
[Thay đổi tài nguyên container tại chỗ (Resize Container Resources In-Place)](289-resize-container-resources-vi.md).

### InPlace {#updateMode-InPlace}

Chế độ này khả dụng dưới dạng tính năng alpha trong VPA 1.7.0 và yêu cầu Kubernetes 1.33
trở lên với feature gate `InPlacePodVerticalScaling` của cluster được bật, cùng với
feature gate `InPlace` được bật trên updater và admission controller của VPA. Nó dùng
tính năng [resize Pod tại chỗ](./47-pod-lifecycle-vi.md#pod-resize) để áp dụng các cập
nhật mà không làm gián đoạn Pod.

Trong chế độ `InPlace`, VPA cố gắng cập nhật resource request và limit của Pod mà không
khởi động lại hay evict Pod. Không giống `InPlaceOrRecreate`, chế độ này **không bao giờ
quay về việc evict**. Nếu một cập nhật tại chỗ không thể áp dụng được (ví dụ, do node
không còn đủ dung lượng), VPA hoãn cập nhật đó lại và thử lại trong một vòng đối chiếu
(reconciliation loop) tiếp theo.

Để dùng chế độ `InPlace`, bật feature gate `InPlace` trên cả updater lẫn admission
controller của VPA:

```shell
--feature-gates=InPlace=true
```

Sau đó đặt `updateMode` thành `"InPlace"` trong spec VPA của bạn:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "InPlace"
```

**Điểm khác biệt chính so với `InPlaceOrRecreate`:** Khi một thao tác resize bị hoãn,
đang diễn ra, hoặc bất khả thi, chế độ `InPlace` luôn chờ và thử lại — nó không bao giờ
evict Pod, bất kể cập nhật đã ở trạng thái chờ bao lâu.

### Auto (đã lỗi thời — deprecated) {#updateMode-Auto}

> **Ghi chú:**
> Chế độ cập nhật `Auto` **đã lỗi thời (deprecated) kể từ VPA phiên bản 1.4.0**. Hãy dùng
> `Recreate` cho các cập nhật dựa trên evict, hoặc `InPlaceOrRecreate` cho các cập nhật
> tại chỗ có cơ chế quay về evict khi cần.

`Auto` hiện là một bí danh (alias) của chế độ `Recreate` và hoạt động y hệt. Nó được đưa
vào để cho phép mở rộng các chiến lược cập nhật tự động trong tương lai.

## Chính sách tài nguyên (Resource policies)

Resource policy cho phép bạn tinh chỉnh cách VerticalPodAutoscaler sinh khuyến nghị và áp
dụng cập nhật. Bạn có thể đặt các giới hạn cho khuyến nghị tài nguyên, chỉ định những tài
nguyên nào cần quản lý, và cấu hình các policy khác nhau cho từng container riêng lẻ
trong một Pod.

Bạn định nghĩa resource policy trong trường `resourcePolicy` của spec VPA:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Recreate"
  resourcePolicy:
    containerPolicies:
    - containerName: "application"
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2
        memory: 2Gi
      controlledResources:
      - cpu
      - memory
      controlledValues: RequestsAndLimits
```

#### minAllowed và maxAllowed

Các trường này đặt giới hạn cho khuyến nghị của VPA. VPA sẽ không bao giờ khuyến nghị tài
nguyên thấp hơn `minAllowed` hoặc cao hơn `maxAllowed`, kể cả khi dữ liệu sử dụng thực tế
gợi ý các giá trị khác.

#### controlledResources

Trường `controlledResources` chỉ định những loại tài nguyên nào VPA nên quản lý cho một
container trong Pod. Nếu không chỉ định, VPA mặc định quản lý cả CPU lẫn memory. Bạn có
thể giới hạn VPA chỉ quản lý các tài nguyên cụ thể. Các tên tài nguyên hợp lệ gồm `cpu`
và `memory`.

### controlledValues

Trường `controlledValues` quyết định VPA điều khiển resource request, limit, hay cả hai:

RequestsAndLimits
: VPA đặt cả request lẫn limit. Limit được co giãn theo tỷ lệ với request dựa trên tỷ lệ
  request-trên-limit định nghĩa trong spec của Pod. Đây là chế độ mặc định.

RequestsOnly
: VPA chỉ đặt request, giữ nguyên limit. Limit vẫn được tôn trọng và vẫn có thể gây
  throttling hoặc bị kill do hết bộ nhớ (out-of-memory kill) nếu mức sử dụng vượt quá
  chúng.

Xem [request và limit](110-manage-resources-containers-vi.md#requests-and-limits)
để tìm hiểu thêm về hai khái niệm này.

## Tài nguyên LimitRange (LimitRange resources)

Hai thành phần admission controller và updater của VPA hậu xử lý các khuyến nghị để tuân
thủ các ràng buộc được định nghĩa trong
[LimitRange](133-limit-range-vi.md). Các tài nguyên
LimitRange với `type` là Pod và Container được kiểm tra trong cluster Kubernetes.

Ví dụ, nếu trường `max` trong một tài nguyên LimitRange loại Container bị vượt quá, cả
hai thành phần VPA đều hạ limit xuống giá trị được định nghĩa trong trường `max`, và
request được giảm theo tỷ lệ tương ứng để duy trì tỷ lệ request-trên-limit trong spec của
Pod.

## Tiếp theo (What's next)

Nếu bạn cấu hình tự động co giãn (autoscaling) trong cluster, bạn cũng có thể cân nhắc
dùng [tự động co giãn node (node autoscaling)](171-node-autoscaling-vi.md)
để bảo đảm bạn đang chạy đúng số lượng node.
Bạn cũng có thể đọc thêm về
[tự động co giãn Pod theo chiều _ngang_ (horizontal Pod autoscaling)](72-horizontal-pod-autoscale-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 4. Đây là bài
chỉ đọc lý thuyết; phần thực hành nằm ở [Lab 11b](labs/README.md#5-sổ-nợ-lab).

1. **Câu bẫy.** VPA ở `updateMode: Recreate` quyết định tăng CPU request cho một Deployment.
   Pod có được đổi request mà không phải khởi động lại không? Và thành phần nào của VPA thực
   sự ghi giá trị mới lên Pod?
2. `updateMode: Off` có tác dụng gì, và bạn xem khuyến nghị ở đâu?
3. `InPlace` và `InPlaceOrRecreate` khác nhau đúng một điểm. Điểm đó là gì?
4. Trên cluster lab của bạn — chưa có metrics-server, chưa cài gì thêm — bạn `kubectl apply`
   một object `kind: VerticalPodAutoscaler`. Chuyện gì xảy ra, và khác gì so với khi bạn apply
   một HorizontalPodAutoscaler?
5. Bạn đặt `controlledValues: RequestsOnly`. Limit của container có còn tác dụng không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không — Pod bị evict.** Trong chế độ `Recreate`, VPA "chủ động quản lý tài nguyên của Pod
   bằng cách **evict Pod** khi resource request hiện tại của chúng chênh lệch đáng kể so với
   khuyến nghị". Và thành phần ghi giá trị mới **không phải updater**: updater chỉ evict, sau
   đó **workload controller** (Deployment ở đây) tạo Pod thay thế, rồi **admission controller
   của VPA** — một mutating webhook chặn request tạo Pod — mới áp resource request đã cập nhật
   lên Pod mới, dùng khuyến nghị Target trong `.status.recommendation`. Trực giác "VPA sửa
   thẳng Pod đang chạy" chỉ đúng với `InPlace` và `InPlaceOrRecreate`, không đúng với
   `Recreate`.
2. Ở `Off`, **recommender vẫn phân tích và vẫn sinh khuyến nghị, nhưng khuyến nghị không được
   tự động áp dụng cho Pod**. Chúng chỉ được lưu trong trường **`.status`** của object VPA, và
   bạn xem bằng `kubectl`. Đây là chế độ để đo trước khi dám cho VPA đụng vào workload.
3. Điểm khác duy nhất là **cách xử lý khi việc resize tại chỗ không thực hiện được**.
   `InPlaceOrRecreate` **quay về evict Pod** (giống `Recreate`) rồi để workload controller tạo
   Pod thay thế. `InPlace` thì **không bao giờ evict**: khi thao tác resize bị hoãn, đang diễn
   ra, hoặc bất khả thi — ví dụ node không còn đủ dung lượng — nó hoãn cập nhật lại và thử lại
   trong một vòng đối chiếu tiếp theo, bất kể đã chờ bao lâu.
4. **API server từ chối vì không biết kind đó.** VPA "được định nghĩa dưới dạng một Custom
   Resource Definition (CRD) trong Kubernetes. **Không giống HorizontalPodAutoscaler vốn là
   một phần của API lõi Kubernetes, VPA phải được cài đặt riêng** vào cluster của bạn". Với
   HorizontalPodAutoscaler thì ngược lại: object tạo được ngay vì kind đã có sẵn — nó chỉ
   không hoạt động vì thiếu nguồn metric. Đó là hai loại "không dùng được" khác nhau, và là lý
   do cả hai bài đều để phần thực hành ở Lab 11b.
5. **Có, limit vẫn còn nguyên tác dụng.** `RequestsOnly` chỉ nghĩa là **VPA chỉ đặt request và
   giữ nguyên limit** — bài nói rõ "Limit vẫn được tôn trọng và vẫn có thể gây throttling hoặc
   bị kill do hết bộ nhớ (out-of-memory kill) nếu mức sử dụng vượt quá chúng". Ở chế độ mặc
   định `RequestsAndLimits` thì limit được co giãn theo tỷ lệ với request, dựa trên tỷ lệ
   request-trên-limit định nghĩa trong spec của Pod.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

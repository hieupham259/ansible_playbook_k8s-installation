# Độ ưu tiên và Preemption của Pod (Pod Priority and Preemption)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 7 → nhóm [7a](LO-TRINH-ADMIN.md#7a-scheduling-và-eviction), bài 6/13 ·
Kiểm chứng ở Lab 7a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Đây là bài đầu tiên của nhóm nói về việc **lấy chỗ của Pod đang chạy**. Bốn bài trước chỉ mô
tả cách chọn chỗ trống; từ đây trở đi cluster bắt đầu chấm dứt Pod. Bài chia làm hai nửa rõ
rệt: nửa đầu là cấu hình (PriorityClass, `priorityClassName`), nửa sau là **các ranh giới của
preemption** — đó mới là phần đáng đọc kỹ, vì nó giải thích những hành vi trông như lỗi.

Mục cuối *Tương tác giữa độ ưu tiên của Pod và chất lượng dịch vụ* nối thẳng sang bài
[142](142-node-pressure-eviction-vi.md); đọc nó như phần mở đầu của bài kế tiếp.

**Phải hiểu ở lần đọc này:**

- PriorityClass là object **không thuộc namespace**, ánh xạ tên → số nguyên (tối đa 1 tỷ; số
  lớn hơn dành cho `system-cluster-critical` và `system-node-critical`). Pod trỏ tới bằng
  `priorityClassName`; Pod không có thì độ ưu tiên bằng 0, trừ khi có một PriorityClass đặt
  `globalDefault`.
- Priority tác động ở **hai chỗ khác nhau**: sắp xếp **hàng đợi lập lịch** (Pod ưu tiên cao
  đứng trước), và chỉ khi Pod đó vẫn không lập lịch được thì logic **preemption** mới khởi
  động.
- Cơ chế preemption: tìm một Node mà việc loại bỏ một hoặc nhiều Pod có độ ưu tiên **thấp hơn**
  sẽ khiến Pod đang chờ lập lịch được; các nạn nhân bị evict và nhận grace period của mình;
  `nominatedNodeName` ghi node được đề cử — nhưng Pod **không chắc** lên đúng node đó.
- Các ranh giới trong mục *Hạn chế của preemption*: PDB được **tôn trọng ở mức nỗ lực tốt
  nhất**, không đảm bảo; **không có preemption xuyên node**; và nếu Pod đang chờ có inter-pod
  affinity tới chính các Pod độ ưu tiên thấp trên node đó, scheduler **không preempt node đó**.
- Priority và QoS class là hai thứ **trực giao**: logic preemption của bộ lập lịch **không xem
  xét QoS**. Ngược lại, kubelet **có** dùng Priority khi xếp thứ tự eviction do áp lực node —
  và không evict Pod đang dùng dưới `requests` của nó.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Cảnh báo đầu bài về người dùng tạo Pod ưu tiên cao, và ResourceQuota giới hạn PriorityClass | cần ResourceQuota | nhóm [7b](LO-TRINH-ADMIN.md#7b-chính-sách-giới-hạn-tài-nguyên), bài [134](134-resource-quotas-vi.md) |
| *PriorityClass không preempt* (`preemptionPolicy: Never`) | dành cho workload dạng job/batch | giai đoạn 13, bài [150](150-gang-scheduling-vi.md) |
| *Lưu ý về PodPriority và các cluster hiện có* | chỉ liên quan khi tiếp quản cluster cũ | không cần |
| *Xử lý sự cố* — ba tình huống preempt bất thường | là tài liệu tra cứu khi gặp hiện tượng | tra khi cần |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.14 [stable]`

[Pod](https://kubernetes.io/docs/concepts/workloads/pods/) có thể có _độ ưu tiên_ (priority).
Độ ưu tiên thể hiện mức độ quan trọng của một Pod so với các Pod khác. Nếu một Pod
không thể được lập lịch, bộ lập lịch (scheduler) sẽ cố gắng preempt (chiếm chỗ — tức là evict)
các Pod có độ ưu tiên thấp hơn để việc lập lịch Pod đang chờ (pending) trở nên khả thi.

> **Cảnh báo:**
>
> Trong một cluster mà không phải mọi người dùng đều đáng tin cậy, một người dùng
> có ý đồ xấu có thể tạo các Pod với độ ưu tiên cao nhất có thể, khiến các Pod khác
> bị evict hoặc không được lập lịch.
>
> Quản trị viên có thể dùng ResourceQuota để ngăn người dùng tạo Pod với độ ưu tiên cao.
>
> Xem [giới hạn tiêu thụ PriorityClass theo mặc định](https://kubernetes.io/docs/concepts/policy/resource-quotas/#limit-priority-class-consumption-by-default)
> để biết chi tiết.

## Cách sử dụng độ ưu tiên và preemption (How to use priority and preemption)

Để sử dụng độ ưu tiên và preemption:

1.  Thêm một hoặc nhiều [PriorityClass](#priorityclass).

1.  Tạo các Pod có [`priorityClassName`](#pod-priority) được đặt bằng một trong các
    PriorityClass vừa thêm. Tất nhiên bạn không cần tạo các Pod trực tiếp;
    thông thường bạn sẽ thêm `priorityClassName` vào Pod template của một
    đối tượng tập hợp như Deployment.

Hãy đọc tiếp để biết thêm thông tin về các bước này.

> **Ghi chú:**
>
> Kubernetes đã có sẵn hai PriorityClass:
> `system-cluster-critical` và `system-node-critical`.
> Đây là các class dùng chung và được dùng để [đảm bảo các thành phần quan trọng luôn được lập lịch trước](https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/).

## PriorityClass

PriorityClass là một đối tượng không thuộc namespace (non-namespaced) định nghĩa
ánh xạ từ tên của priority class sang giá trị số nguyên của độ ưu tiên. Tên được
chỉ định trong trường `name` thuộc metadata của đối tượng PriorityClass. Giá trị
được chỉ định trong trường bắt buộc `value`. Giá trị càng cao thì độ ưu tiên càng cao.
Tên của một đối tượng PriorityClass phải là một
[tên miền con DNS hợp lệ (DNS subdomain name)](https://kubernetes.io/docs/concepts/overview/working-with-objects/names#dns-subdomain-names),
và không được có tiền tố `system-`.

Một đối tượng PriorityClass có thể mang bất kỳ giá trị số nguyên 32-bit nào nhỏ hơn
hoặc bằng 1 tỷ. Điều này có nghĩa là khoảng giá trị cho một đối tượng PriorityClass
là từ -2147483648 đến 1000000000 (bao gồm cả hai đầu). Các số lớn hơn được dành riêng
cho các PriorityClass tích hợp sẵn đại diện cho các Pod hệ thống quan trọng.
Quản trị viên cluster nên tạo một đối tượng PriorityClass cho mỗi ánh xạ như vậy mà họ muốn.

PriorityClass cũng có hai trường tùy chọn: `globalDefault` và `description`.
Trường `globalDefault` cho biết giá trị của PriorityClass này sẽ được dùng cho
các Pod không có `priorityClassName`. Chỉ có thể tồn tại một PriorityClass với
`globalDefault` được đặt là true trong hệ thống. Nếu không có PriorityClass nào
đặt `globalDefault`, độ ưu tiên của các Pod không có `priorityClassName` bằng 0.

Trường `description` là một chuỗi tùy ý. Nó dùng để cho người dùng của cluster
biết khi nào họ nên dùng PriorityClass này.

### Lưu ý về PodPriority và các cluster hiện có (Notes about PodPriority and existing clusters)

-   Nếu bạn nâng cấp một cluster hiện có chưa có tính năng này, độ ưu tiên
    của các Pod hiện có của bạn thực tế bằng 0.

-   Việc thêm một PriorityClass với `globalDefault` được đặt là `true` không làm
    thay đổi độ ưu tiên của các Pod hiện có. Giá trị của PriorityClass như vậy
    chỉ được dùng cho các Pod được tạo sau khi PriorityClass đó được thêm vào.

-   Nếu bạn xóa một PriorityClass, các Pod hiện có đang dùng tên của
    PriorityClass đã xóa vẫn giữ nguyên, nhưng bạn không thể tạo thêm Pod
    dùng tên của PriorityClass đã bị xóa.

### Ví dụ PriorityClass (Example PriorityClass)

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for XYZ service pods only."
```

## PriorityClass không preempt (Non-preempting PriorityClass) {#non-preempting-priority-class}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.24 [stable]`

Các Pod với `preemptionPolicy: Never` sẽ được xếp trong hàng đợi lập lịch (scheduling queue)
trước các pod có độ ưu tiên thấp hơn,
nhưng chúng không thể preempt các pod khác.
Một pod không preempt đang chờ được lập lịch sẽ ở lại trong hàng đợi lập lịch,
cho đến khi có đủ tài nguyên trống,
và nó có thể được lập lịch.
Các pod không preempt,
giống như các pod khác,
vẫn chịu cơ chế back-off của bộ lập lịch.
Điều này có nghĩa là nếu bộ lập lịch thử các pod này và chúng không thể được lập lịch,
chúng sẽ được thử lại với tần suất thấp hơn,
cho phép các pod khác có độ ưu tiên thấp hơn được lập lịch trước chúng.

Các pod không preempt vẫn có thể bị preempt bởi các pod khác
có độ ưu tiên cao.

`preemptionPolicy` mặc định là `PreemptLowerPriority`,
cho phép các pod thuộc PriorityClass đó preempt các pod có độ ưu tiên thấp hơn
(đây là hành vi mặc định hiện có).
Nếu `preemptionPolicy` được đặt là `Never`,
các pod trong PriorityClass đó sẽ không preempt.

Một trường hợp sử dụng ví dụ là các workload khoa học dữ liệu (data science).
Người dùng có thể gửi một job mà họ muốn được ưu tiên hơn các workload khác,
nhưng không muốn hủy bỏ công việc hiện tại bằng cách preempt các pod đang chạy.
Job có độ ưu tiên cao với `preemptionPolicy: Never` sẽ được lập lịch
trước các pod khác trong hàng đợi,
ngay khi có đủ tài nguyên cluster được giải phóng "một cách tự nhiên".

### Ví dụ PriorityClass không preempt (Example Non-preempting PriorityClass)

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-nonpreempting
value: 1000000
preemptionPolicy: Never
globalDefault: false
description: "This priority class will not cause other pods to be preempted."
```

## Độ ưu tiên của Pod (Pod priority) {#pod-priority}

Sau khi bạn có một hoặc nhiều PriorityClass, bạn có thể tạo các Pod chỉ định một
trong các tên PriorityClass đó trong đặc tả (specification) của chúng. Priority
admission controller sử dụng trường `priorityClassName` và điền vào giá trị số nguyên
của độ ưu tiên. Nếu không tìm thấy priority class, Pod sẽ bị từ chối.

YAML sau đây là một ví dụ về cấu hình Pod sử dụng PriorityClass đã được tạo
trong ví dụ trước. Priority admission controller kiểm tra đặc tả và
phân giải độ ưu tiên của Pod thành 1000000.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  priorityClassName: high-priority
```

### Ảnh hưởng của độ ưu tiên Pod đến thứ tự lập lịch (Effect of Pod priority on scheduling order)

Khi độ ưu tiên của Pod được bật, bộ lập lịch sắp xếp các Pod đang chờ theo
độ ưu tiên của chúng, và một Pod đang chờ được đặt trước các Pod đang chờ khác
có độ ưu tiên thấp hơn trong hàng đợi lập lịch. Kết quả là, Pod có độ ưu tiên
cao hơn có thể được lập lịch sớm hơn các Pod có độ ưu tiên thấp hơn nếu
các yêu cầu lập lịch của nó được đáp ứng. Nếu Pod đó không thể được lập lịch,
bộ lập lịch sẽ tiếp tục và thử lập lịch các Pod khác có độ ưu tiên thấp hơn.

## Preemption

Khi các Pod được tạo, chúng đi vào một hàng đợi và chờ được lập lịch. Bộ lập lịch
chọn một Pod từ hàng đợi và cố gắng lập lịch nó lên một Node. Nếu không tìm thấy
Node nào thỏa mãn tất cả các yêu cầu đã chỉ định của Pod, logic preemption được
kích hoạt cho Pod đang chờ. Hãy gọi Pod đang chờ là P.
Logic preemption cố gắng tìm một Node mà tại đó việc loại bỏ một hoặc nhiều Pod có
độ ưu tiên thấp hơn P sẽ giúp P có thể được lập lịch lên Node đó. Nếu tìm thấy
Node như vậy, một hoặc nhiều Pod có độ ưu tiên thấp hơn sẽ bị evict khỏi Node. Sau khi
các Pod đó biến mất, P có thể được lập lịch lên Node.

### Thông tin hiển thị cho người dùng (User exposed information)

Khi Pod P preempt một hoặc nhiều Pod trên Node N, trường `nominatedNodeName` trong
status của Pod P được đặt thành tên của Node N. Trường này giúp bộ lập lịch theo dõi
các tài nguyên được dành riêng cho Pod P và cũng cung cấp cho người dùng thông tin
về các lần preemption trong cluster của họ.

Xin lưu ý rằng Pod P không nhất thiết được lập lịch lên "Node được đề cử" (nominated Node).
Bộ lập lịch luôn thử "Node được đề cử" trước khi duyệt qua bất kỳ node nào khác.
Sau khi các Pod nạn nhân (victim) bị preempt, chúng nhận được khoảng thời gian chấm dứt
nhẹ nhàng (graceful termination period) của mình. Nếu một node khác trở nên khả dụng
trong khi bộ lập lịch đang chờ các Pod nạn nhân chấm dứt, bộ lập lịch có thể dùng
node khác đó để lập lịch Pod P. Do đó, `nominatedNodeName` và `nodeName` trong spec
của Pod không phải lúc nào cũng giống nhau. Ngoài ra, nếu bộ lập lịch preempt các Pod
trên Node N, nhưng sau đó một Pod có độ ưu tiên cao hơn Pod P xuất hiện, bộ lập lịch
có thể trao Node N cho Pod mới có độ ưu tiên cao hơn. Trong trường hợp như vậy,
bộ lập lịch xóa `nominatedNodeName` của Pod P. Bằng cách này, bộ lập lịch làm cho
Pod P đủ điều kiện preempt các Pod trên một Node khác.

### Hạn chế của preemption (Limitations of preemption)

#### Chấm dứt nhẹ nhàng cho các nạn nhân của preemption (Graceful termination of preemption victims)

Khi các Pod bị preempt, các nạn nhân nhận được
[khoảng thời gian chấm dứt nhẹ nhàng](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination) của mình.
Chúng có chừng đó thời gian để hoàn thành công việc và thoát. Nếu không, chúng sẽ
bị kill. Khoảng thời gian chấm dứt nhẹ nhàng này tạo ra một khoảng trống thời gian
giữa thời điểm bộ lập lịch preempt các Pod và thời điểm Pod đang chờ (P) có thể được
lập lịch lên Node (N). Trong thời gian đó, bộ lập lịch tiếp tục lập lịch các Pod
đang chờ khác. Khi các nạn nhân thoát hoặc bị chấm dứt, bộ lập lịch cố gắng lập lịch
các Pod trong hàng đợi chờ. Do đó, thường có một khoảng trống thời gian giữa thời điểm
bộ lập lịch preempt các nạn nhân và thời điểm Pod P được lập lịch. Để giảm thiểu
khoảng trống này, bạn có thể đặt khoảng thời gian chấm dứt nhẹ nhàng của các Pod
có độ ưu tiên thấp về 0 hoặc một số nhỏ.

#### PodDisruptionBudget được hỗ trợ, nhưng không được đảm bảo (PodDisruptionBudget is supported, but not guaranteed)

[PodDisruptionBudget](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/) (PDB)
cho phép chủ sở hữu ứng dụng giới hạn số lượng Pod của một ứng dụng được nhân bản
bị ngừng hoạt động đồng thời do các gián đoạn tự nguyện (voluntary disruption). Kubernetes hỗ trợ
PDB khi preempt các Pod, nhưng việc tôn trọng PDB chỉ ở mức nỗ lực tốt nhất (best effort).
Bộ lập lịch cố gắng tìm các nạn nhân mà PDB của chúng không bị vi phạm bởi preemption,
nhưng nếu không tìm thấy nạn nhân nào như vậy, preemption vẫn sẽ xảy ra, và các Pod
có độ ưu tiên thấp hơn sẽ bị loại bỏ bất chấp PDB của chúng bị vi phạm.

#### Inter-Pod affinity trên các Pod có độ ưu tiên thấp hơn (Inter-Pod affinity on lower-priority Pods)

Một Node chỉ được xem xét cho preemption khi câu trả lời cho câu hỏi này là
có: "Nếu tất cả các Pod có độ ưu tiên thấp hơn Pod đang chờ bị loại bỏ khỏi
Node, liệu Pod đang chờ có thể được lập lịch lên Node đó không?"

> **Ghi chú:**
>
> Preemption không nhất thiết loại bỏ tất cả các Pod có độ ưu tiên thấp hơn.
> Nếu Pod đang chờ có thể được lập lịch bằng cách loại bỏ ít hơn toàn bộ
> các Pod có độ ưu tiên thấp hơn, thì chỉ một phần các Pod có độ ưu tiên thấp hơn
> bị loại bỏ. Dù vậy, câu trả lời cho câu hỏi trên vẫn phải là có. Nếu câu trả lời
> là không, Node đó không được xem xét cho preemption.

Nếu một Pod đang chờ có inter-pod affinity với một hoặc nhiều Pod có độ ưu tiên
thấp hơn trên Node, quy tắc inter-Pod affinity đó không thể được thỏa mãn khi
thiếu các Pod có độ ưu tiên thấp hơn đó. Trong trường hợp này, bộ lập lịch không
preempt bất kỳ Pod nào trên Node đó. Thay vào đó, nó tìm một Node khác. Bộ lập lịch
có thể tìm thấy một Node phù hợp hoặc không. Không có gì đảm bảo rằng Pod đang chờ
sẽ được lập lịch.

Giải pháp được khuyến nghị của chúng tôi cho vấn đề này là chỉ tạo inter-Pod affinity
hướng tới các Pod có độ ưu tiên bằng hoặc cao hơn.

#### Preemption xuyên node (Cross node preemption)

Giả sử một Node N đang được xem xét cho preemption để một Pod đang chờ P có thể
được lập lịch lên N. P có thể chỉ trở nên khả thi trên N nếu một Pod trên một
Node khác bị preempt. Đây là một ví dụ:

*   Pod P đang được xem xét cho Node N.
*   Pod Q đang chạy trên một Node khác trong cùng Zone với Node N.
*   Pod P có anti-affinity phạm vi toàn Zone với Pod Q (`topologyKey:
    topology.kubernetes.io/zone`).
*   Không có trường hợp anti-affinity nào khác giữa Pod P và các Pod khác trong
    Zone.
*   Để lập lịch Pod P lên Node N, Pod Q có thể bị preempt, nhưng bộ lập lịch
    không thực hiện preemption xuyên node. Vì vậy, Pod P sẽ bị coi là
    không thể lập lịch trên Node N.

Nếu Pod Q bị loại bỏ khỏi Node của nó, vi phạm Pod anti-affinity sẽ không còn,
và Pod P có khả năng được lập lịch lên Node N.

Chúng tôi có thể cân nhắc thêm preemption xuyên node trong các phiên bản tương lai
nếu có đủ nhu cầu và nếu chúng tôi tìm được một thuật toán có hiệu năng hợp lý.

## Xử lý sự cố (Troubleshooting)

Độ ưu tiên và preemption của Pod có thể có các tác dụng phụ không mong muốn. Dưới đây
là một số ví dụ về các vấn đề tiềm ẩn và cách xử lý chúng.

### Pod bị preempt một cách không cần thiết (Pods are preempted unnecessarily)

Preemption loại bỏ các Pod hiện có khỏi cluster khi có áp lực tài nguyên để nhường
chỗ cho các Pod đang chờ có độ ưu tiên cao hơn. Nếu bạn vô tình gán độ ưu tiên cao
cho một số Pod nhất định, các Pod có độ ưu tiên cao ngoài ý muốn này có thể gây ra
preemption trong cluster của bạn. Độ ưu tiên của Pod được chỉ định bằng cách đặt
trường `priorityClassName` trong đặc tả của Pod. Giá trị số nguyên của độ ưu tiên
sau đó được phân giải và điền vào trường `priority` của `podSpec`.

Để giải quyết vấn đề, bạn có thể thay đổi `priorityClassName` của các Pod đó
để dùng các priority class thấp hơn, hoặc để trống trường đó. `priorityClassName`
trống được phân giải thành 0 theo mặc định.

Khi một Pod bị preempt, sẽ có các event được ghi lại cho Pod bị preempt đó.
Preemption chỉ nên xảy ra khi cluster không có đủ tài nguyên cho một Pod.
Trong những trường hợp như vậy, preemption chỉ xảy ra khi độ ưu tiên của Pod
đang chờ (preemptor) cao hơn các Pod nạn nhân. Preemption không được xảy ra khi
không có Pod đang chờ nào, hoặc khi các Pod đang chờ có độ ưu tiên bằng hoặc thấp hơn
các nạn nhân. Nếu preemption xảy ra trong những tình huống như vậy, vui lòng tạo một issue.

### Pod bị preempt, nhưng preemptor không được lập lịch (Pods are preempted, but the preemptor is not scheduled)

Khi các pod bị preempt, chúng nhận được khoảng thời gian chấm dứt nhẹ nhàng
đã yêu cầu, mặc định là 30 giây. Nếu các Pod nạn nhân không chấm dứt trong
khoảng thời gian này, chúng sẽ bị buộc chấm dứt. Khi tất cả các nạn nhân biến mất,
Pod preemptor mới có thể được lập lịch.

Trong khi Pod preemptor đang chờ các nạn nhân biến mất, một Pod có độ ưu tiên
cao hơn có thể được tạo và vừa với cùng Node đó. Trong trường hợp này, bộ lập lịch
sẽ lập lịch Pod có độ ưu tiên cao hơn thay vì preemptor.

Đây là hành vi được mong đợi: Pod có độ ưu tiên cao hơn sẽ chiếm lấy vị trí
của Pod có độ ưu tiên thấp hơn.

### Pod có độ ưu tiên cao hơn bị preempt trước pod có độ ưu tiên thấp hơn (Higher priority Pods are preempted before lower priority pods)

Bộ lập lịch cố gắng tìm các node có thể chạy một Pod đang chờ. Nếu không tìm thấy
node nào, bộ lập lịch cố gắng loại bỏ các Pod có độ ưu tiên thấp hơn khỏi một node
bất kỳ để nhường chỗ cho pod đang chờ.
Nếu một node có các Pod độ ưu tiên thấp không khả thi để chạy Pod đang chờ, bộ lập lịch
có thể chọn một node khác có các Pod độ ưu tiên cao hơn (so với các Pod trên
node kia) để thực hiện preemption. Các nạn nhân vẫn phải có độ ưu tiên thấp hơn
Pod preemptor.

Khi có nhiều node khả dụng cho preemption, bộ lập lịch cố gắng chọn node có
tập các Pod với độ ưu tiên thấp nhất. Tuy nhiên, nếu các Pod đó có
PodDisruptionBudget sẽ bị vi phạm nếu chúng bị preempt, thì bộ lập lịch
có thể chọn một node khác có các Pod độ ưu tiên cao hơn.

Khi tồn tại nhiều node cho preemption và không có kịch bản nào ở trên áp dụng,
bộ lập lịch chọn node có độ ưu tiên thấp nhất.

## Tương tác giữa độ ưu tiên của Pod và chất lượng dịch vụ (Interactions between Pod priority and quality of service) {#interactions-of-pod-priority-and-qos}

Độ ưu tiên của Pod và QoS class là hai tính năng trực giao (orthogonal) với ít
tương tác và không có ràng buộc mặc định nào đối với việc đặt độ ưu tiên của một Pod
dựa trên QoS class của nó. Logic preemption của bộ lập lịch không xem xét QoS
khi chọn các mục tiêu preemption. Preemption xem xét độ ưu tiên của Pod và cố gắng
chọn một tập các mục tiêu có độ ưu tiên thấp nhất. Các Pod có độ ưu tiên cao hơn
chỉ được xem xét cho preemption nếu việc loại bỏ các Pod có độ ưu tiên thấp nhất
là không đủ để bộ lập lịch có thể lập lịch Pod preemptor, hoặc nếu các Pod có
độ ưu tiên thấp nhất được bảo vệ bởi `PodDisruptionBudget`.

kubelet dùng Priority để xác định thứ tự các pod trong [eviction do áp lực node (node-pressure eviction)](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/).
Bạn có thể dùng QoS class để ước lượng thứ tự mà các pod có nhiều khả năng
bị evict nhất. kubelet xếp hạng các pod để evict dựa trên các yếu tố sau:

  1. Việc sử dụng tài nguyên đang cạn kiệt (starved resource) có vượt quá requests hay không
  1. Độ ưu tiên của Pod
  1. Lượng tài nguyên sử dụng so với requests

Xem [Lựa chọn Pod cho kubelet eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/#pod-selection-for-kubelet-eviction)
để biết thêm chi tiết.

Eviction do áp lực node của kubelet không evict các Pod khi mức sử dụng của chúng
không vượt quá requests của chúng. Nếu một Pod có độ ưu tiên thấp hơn không
vượt quá requests của nó, nó sẽ không bị evict. Một Pod khác có độ ưu tiên cao hơn
nhưng vượt quá requests của nó thì có thể bị evict.

## Tiếp theo (What's next)

* Đọc về việc sử dụng ResourceQuota kết hợp với PriorityClass: [giới hạn tiêu thụ PriorityClass theo mặc định](https://kubernetes.io/docs/concepts/policy/resource-quotas/#limit-priority-class-consumption-by-default)
* Tìm hiểu về [Pod Disruption](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
* Tìm hiểu về [Eviction khởi phát qua API (API-initiated Eviction)](https://kubernetes.io/docs/concepts/scheduling-eviction/api-eviction/)
* Tìm hiểu về [Eviction do áp lực node (Node-pressure Eviction)](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 7:

1. Độ ưu tiên của Pod can thiệp vào hai chỗ khác nhau của quá trình lập lịch. Kể ra, và cho
   biết chỗ nào xảy ra trước.
2. Một Pod có QoS class `Guaranteed` (requests bằng limits cho mọi container) nhưng
   `priorityClassName` để trống. Nó có được bảo vệ khỏi preemption không?
3. Trên cluster lab, hai worker 2 vCPU / 6 GB RAM đã bị lấp đầy bởi các Pod độ ưu tiên 0. Bạn
   tạo một Pod với `priorityClassName: high-priority`. Mô tả những gì xảy ra, và cho biết
   `nominatedNodeName` của Pod đó nghĩa là gì.
4. Ứng dụng của bạn có PodDisruptionBudget. PDB đó có chặn được preemption không?
5. Pod đang chờ P có `podAffinity` tới các Pod độ ưu tiên thấp đang chạy trên node N.
   Scheduler có preempt các Pod đó để nhét P vào N không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Trước hết là hàng đợi lập lịch**: bộ lập lịch sắp xếp các Pod đang chờ theo độ ưu tiên, và
   Pod ưu tiên cao được đặt trước các Pod ưu tiên thấp hơn trong hàng đợi — nhờ đó nó có thể
   được lập lịch sớm hơn **nếu** các yêu cầu lập lịch của nó được đáp ứng. **Sau đó mới tới
   preemption**: chỉ khi không tìm được Node nào thỏa mãn mọi yêu cầu của Pod thì logic
   preemption mới kích hoạt. Nếu Pod ưu tiên cao vẫn không lập lịch được, bộ lập lịch đi tiếp
   và thử các Pod ưu tiên thấp hơn.
2. **Không.** Độ ưu tiên và QoS class là **hai tính năng trực giao** — bài nói thẳng: "Logic
   preemption của bộ lập lịch **không xem xét QoS** khi chọn các mục tiêu preemption. Preemption
   xem xét độ ưu tiên của Pod". Pod này có độ ưu tiên 0, tức là nạn nhân dễ bị chọn nhất, dù
   QoS của nó là hạng cao nhất. Đây là chỗ trực giác sai: `Guaranteed` bảo vệ bạn khỏi **eviction
   do áp lực node**, chứ không bảo vệ khỏi **preemption**.
3. Pod không lập lịch được ngay, nên logic preemption khởi động: nó tìm một Node mà việc loại
   bỏ một hoặc vài Pod độ ưu tiên thấp hơn đủ để Pod mới vừa chỗ, rồi **evict các Pod nạn nhân
   đó**; các nạn nhân nhận grace period của mình rồi mới biến mất. `nominatedNodeName` được đặt
   bằng tên Node vừa bị preempt — nó **là node được đề cử, không phải cam kết**: bộ lập lịch
   luôn thử node đề cử trước, nhưng nếu trong lúc chờ nạn nhân chấm dứt mà một node khác trống
   ra, Pod có thể được lập lịch ở đó; và nếu xuất hiện một Pod ưu tiên còn cao hơn, node đề cử
   có thể bị trao cho Pod kia và `nominatedNodeName` bị xóa. Lưu ý preemption **không nhất
   thiết loại bỏ tất cả** Pod ưu tiên thấp trên node — chỉ loại đủ để Pod mới vừa chỗ.
4. **Không chắc chắn — PDB chỉ được tôn trọng ở mức nỗ lực tốt nhất (best effort).** Bộ lập
   lịch cố tìm nạn nhân sao cho PDB không bị vi phạm, nhưng nếu không tìm được nạn nhân nào như
   vậy thì **preemption vẫn xảy ra** và Pod độ ưu tiên thấp hơn vẫn bị loại bỏ bất chấp PDB bị
   vi phạm. Điều PDB thực sự làm được là khiến bộ lập lịch **ưu tiên chọn node khác**, kể cả
   node đó có các Pod độ ưu tiên cao hơn.
5. **Không.** Quy tắc inter-Pod affinity của P không thể được thỏa mãn khi thiếu chính các Pod
   độ ưu tiên thấp đó, nên bộ lập lịch **không preempt bất kỳ Pod nào trên node N** mà đi tìm
   node khác — và không có gì đảm bảo nó tìm được. Khuyến nghị của bài: chỉ tạo inter-Pod
   affinity hướng tới các Pod có độ ưu tiên **bằng hoặc cao hơn**.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

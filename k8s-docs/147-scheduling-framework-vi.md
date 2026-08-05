# Scheduling Framework

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.19 [stable]`

_Scheduling framework_ là một kiến trúc dạng plugin (pluggable architecture) cho scheduler của Kubernetes.
Nó bao gồm một tập các API "plugin" được biên dịch trực tiếp vào scheduler.
Các API này cho phép hầu hết các tính năng lập lịch được triển khai dưới dạng plugin,
trong khi vẫn giữ cho phần "lõi" (core) lập lịch gọn nhẹ và dễ bảo trì. Tham khảo
[đề xuất thiết kế của scheduling framework][kep] để biết thêm thông tin kỹ thuật về
thiết kế của framework này.

[kep]: https://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/624-scheduling-framework/README.md

## Luồng hoạt động của framework (Framework workflow)

Scheduling Framework định nghĩa một số điểm mở rộng (extension point). Các plugin của scheduler
đăng ký để được gọi tại một hoặc nhiều điểm mở rộng. Một số plugin trong đó
có thể thay đổi các quyết định lập lịch, còn một số chỉ mang tính cung cấp thông tin.

Mỗi lần thử lập lịch một Pod được chia thành hai giai đoạn:
**chu trình lập lịch (scheduling cycle)** và **chu trình binding (binding cycle)**.

### Chu trình lập lịch & chu trình binding (Scheduling cycle & binding cycle)

Chu trình lập lịch chọn một node cho Pod, và chu trình binding áp dụng
quyết định đó lên cluster. Gộp lại, một chu trình lập lịch và một chu trình binding được
gọi chung là một "ngữ cảnh lập lịch" (scheduling context).

Các chu trình lập lịch chạy tuần tự, trong khi các chu trình binding có thể chạy song song.

Một chu trình lập lịch hoặc binding có thể bị hủy bỏ nếu Pod được xác định là
không thể lập lịch (unschedulable) hoặc nếu xảy ra lỗi nội bộ. Pod sẽ được trả về
hàng đợi và được thử lại.

## Các interface (Interfaces)

Hình dưới đây thể hiện ngữ cảnh lập lịch của một Pod và các interface
mà scheduling framework cung cấp.

Một plugin có thể triển khai nhiều interface để thực hiện các tác vụ phức tạp hơn
hoặc có trạng thái (stateful).

Một số interface tương ứng với các điểm mở rộng của scheduler, có thể cấu hình qua
[Cấu hình Scheduler (Scheduler Configuration)](https://kubernetes.io/docs/reference/scheduling/config/#extension-points).

![Các điểm mở rộng của scheduling framework](https://kubernetes.io/images/docs/scheduling-framework-extensions.png)

*Các điểm mở rộng của scheduling framework (Scheduling framework extension points)*

### PreEnqueue {#pre-enqueue}

Các plugin này được gọi trước khi thêm Pod vào hàng đợi active nội bộ, nơi các Pod được đánh dấu là
sẵn sàng để lập lịch.

Chỉ khi tất cả các plugin PreEnqueue trả về `Success`, Pod mới được phép vào hàng đợi active.
Nếu không, nó được đặt vào danh sách nội bộ các Pod không thể lập lịch, và không nhận điều kiện (condition) `Unschedulable`.

Để biết thêm chi tiết về cách các hàng đợi nội bộ của scheduler hoạt động, hãy đọc
[Hàng đợi lập lịch trong kube-scheduler](https://github.com/kubernetes/community/blob/f03b6d5692bd979f07dd472e7b6836b2dad0fd9b/contributors/devel/sig-scheduling/scheduler_queues.md).

### EnqueueExtension

EnqueueExtension là interface mà qua đó plugin có thể kiểm soát
việc có thử lập lịch lại các Pod đã bị plugin này từ chối hay không, dựa trên các thay đổi trong cluster.
Các plugin triển khai PreEnqueue, PreFilter, Filter, Reserve hoặc Permit nên triển khai interface này.

### QueueingHint

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.34 [stable]`

QueueingHint là một hàm callback dùng để quyết định liệu một Pod có thể được đưa lại vào hàng đợi active hay hàng đợi backoff hay không.
Nó được thực thi mỗi khi một loại sự kiện hoặc thay đổi nhất định xảy ra trong cluster.
Khi QueueingHint nhận thấy sự kiện đó có thể làm cho Pod trở nên lập lịch được,
Pod được đưa vào hàng đợi active hoặc hàng đợi backoff
để scheduler thử lập lịch lại Pod đó.

### QueueSort {#queue-sort}

Các plugin này được dùng để sắp xếp các Pod trong hàng đợi lập lịch. Về bản chất, một plugin queue sort
cung cấp một hàm `Less(Pod1, Pod2)`. Chỉ có thể bật một plugin queue sort
tại một thời điểm.

### PreFilter {#pre-filter}

Các plugin này được dùng để tiền xử lý thông tin về Pod, hoặc để kiểm tra những
điều kiện nhất định mà cluster hoặc Pod phải đáp ứng. Nếu một plugin PreFilter trả về
lỗi, chu trình lập lịch bị hủy bỏ.

### Filter

Các plugin này được dùng để loại bỏ những node không thể chạy Pod. Với mỗi
node, scheduler sẽ gọi các plugin filter theo thứ tự đã cấu hình. Nếu bất kỳ
plugin filter nào đánh dấu node là không khả thi (infeasible), các plugin còn lại sẽ không được
gọi cho node đó. Các node có thể được đánh giá song song.

### PostFilter {#post-filter}

Các plugin này được gọi sau giai đoạn Filter, nhưng chỉ khi không tìm thấy node khả thi nào
cho pod. Các plugin được gọi theo thứ tự đã cấu hình. Nếu
bất kỳ plugin postFilter nào đánh dấu node là `Schedulable`, các plugin còn lại
sẽ không được gọi. Một triển khai PostFilter điển hình là preemption (chiếm chỗ), cơ chế
cố gắng làm cho pod trở nên lập lịch được bằng cách chiếm chỗ (preempt) các Pod khác.

### PreScore {#pre-score}

Các plugin này được dùng để thực hiện công việc "tiền chấm điểm" (pre-scoring), tạo ra một
trạng thái chia sẻ được cho các plugin Score sử dụng. Nếu một plugin PreScore trả về lỗi,
chu trình lập lịch bị hủy bỏ.

### Score {#scoring}

Các plugin này được dùng để xếp hạng các node đã vượt qua giai đoạn lọc (filtering).
Scheduler sẽ gọi từng plugin chấm điểm cho từng node. Sẽ có một khoảng
số nguyên được định nghĩa rõ ràng đại diện cho điểm tối thiểu và tối đa. Sau
giai đoạn [NormalizeScore](#normalize-scoring), scheduler sẽ kết hợp điểm số node
từ tất cả các plugin theo trọng số (weight) plugin đã cấu hình.

#### Chấm điểm theo dung lượng (Capacity scoring) {#scoring-capacity}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.33 [alpha]`

Feature gate `VolumeCapacityPriority` được dùng trong v1.32 để hỗ trợ lưu trữ
được cấp phát tĩnh (statically provisioned). Kể từ v1.33, feature gate mới `StorageCapacityScoring`
thay thế gate `VolumeCapacityPriority` cũ với hỗ trợ bổ sung cho lưu trữ được cấp phát động (dynamically provisioned).
Khi `StorageCapacityScoring` được bật, plugin VolumeBinding trong kube-scheduler được mở rộng
để chấm điểm các Node dựa trên dung lượng lưu trữ trên mỗi Node.
Tính năng này áp dụng cho các CSI volume có hỗ trợ [Dung lượng lưu trữ (Storage Capacity)](./103-storage-capacity-vi.md),
bao gồm cả lưu trữ cục bộ được cung cấp bởi một CSI driver.

### NormalizeScore {#normalize-scoring}

Các plugin này được dùng để điều chỉnh điểm số trước khi scheduler tính toán
xếp hạng cuối cùng của các Node. Một plugin đăng ký cho điểm mở rộng này sẽ được
gọi với kết quả [Score](#scoring) từ chính plugin đó. Việc này được gọi
một lần cho mỗi plugin trong mỗi chu trình lập lịch.

Ví dụ, giả sử một plugin `BlinkingLightScorer` xếp hạng các Node dựa trên số lượng
đèn nhấp nháy mà chúng có.

```go
func ScoreNode(_ *v1.pod, n *v1.Node) (int, error) {
    return getBlinkingLightCount(n)
}
```

Tuy nhiên, số lượng đèn nhấp nháy tối đa có thể nhỏ so với
`NodeScoreMax`. Để khắc phục điều này, `BlinkingLightScorer` cũng nên đăng ký cho
điểm mở rộng này.

```go
func NormalizeScores(scores map[string]int) {
    highest := 0
    for _, score := range scores {
        highest = max(highest, score)
    }
    for node, score := range scores {
        scores[node] = score*NodeScoreMax/highest
    }
}
```

Nếu bất kỳ plugin NormalizeScore nào trả về lỗi, chu trình lập lịch bị
hủy bỏ.

> **Ghi chú:**
>
> Các plugin muốn thực hiện công việc "tiền dự trữ" (pre-reserve) nên dùng
> điểm mở rộng NormalizeScore.

### Reserve {#reserve}

Một plugin triển khai interface Reserve có hai phương thức, là `Reserve`
và `Unreserve`, tương ứng với hai giai đoạn lập lịch mang tính cung cấp thông tin gọi là Reserve
và Unreserve. Các plugin duy trì trạng thái lúc chạy (còn gọi là "stateful
plugin") nên dùng các giai đoạn này để được scheduler thông báo khi tài nguyên
trên một node được dự trữ (reserve) và hủy dự trữ (unreserve) cho một Pod nhất định.

Giai đoạn Reserve diễn ra trước khi scheduler thực sự bind một Pod vào
node được chỉ định. Nó tồn tại để ngăn ngừa các tình huống tranh chấp (race condition) trong khi scheduler chờ
việc bind thành công. Phương thức `Reserve` của mỗi plugin Reserve có thể thành công
hoặc thất bại; nếu một lời gọi phương thức `Reserve` thất bại, các plugin tiếp theo sẽ không được thực thi
và giai đoạn Reserve được coi là thất bại. Nếu phương thức `Reserve` của
tất cả các plugin đều thành công, giai đoạn Reserve được coi là thành công và
phần còn lại của chu trình lập lịch cùng chu trình binding sẽ được thực thi.

Giai đoạn Unreserve được kích hoạt nếu giai đoạn Reserve hoặc một giai đoạn sau đó thất bại.
Khi điều này xảy ra, phương thức `Unreserve` của **tất cả** các plugin Reserve sẽ được
thực thi theo thứ tự ngược lại với các lời gọi phương thức `Reserve`. Giai đoạn này tồn tại để
dọn dẹp trạng thái gắn với Pod đã được dự trữ.

> **Thận trọng:**
>
> Việc triển khai phương thức `Unreserve` trong các plugin Reserve phải
> có tính lũy đẳng (idempotent) và không được phép thất bại.

### Permit

Các plugin _Permit_ được gọi vào cuối chu trình lập lịch của mỗi Pod, để
ngăn chặn hoặc trì hoãn việc binding tới node ứng viên. Một plugin permit có thể làm một trong
ba việc sau:

1.  **approve** (chấp thuận) \
    Khi tất cả các plugin Permit chấp thuận một Pod, Pod đó được gửi đi để binding.

1.  **deny** (từ chối) \
    Nếu bất kỳ plugin Permit nào từ chối một Pod, Pod đó được trả về hàng đợi lập lịch.
    Điều này sẽ kích hoạt giai đoạn Unreserve trong [các plugin Reserve](#reserve).

1.  **wait** (chờ — có timeout) \
    Nếu một plugin Permit trả về "wait", thì Pod được giữ trong danh sách nội bộ các Pod
    đang "chờ" (waiting), và chu trình binding của Pod này bắt đầu nhưng bị chặn (block) trực tiếp cho đến khi
    nó được chấp thuận. Nếu xảy ra timeout, **wait** trở thành **deny**
    và Pod được trả về hàng đợi lập lịch, kích hoạt
    giai đoạn Unreserve trong [các plugin Reserve](#reserve).

> **Ghi chú:**
>
> Mặc dù bất kỳ plugin nào cũng có thể truy cập danh sách các Pod đang "chờ" và chấp thuận chúng
> (xem [`FrameworkHandle`](https://git.k8s.io/enhancements/keps/sig-scheduling/624-scheduling-framework#frameworkhandle)),
> chúng tôi kỳ vọng chỉ các plugin permit mới chấp thuận việc binding của các Pod đã được dự trữ đang ở trạng thái "chờ".
> Khi một Pod được chấp thuận, nó được gửi đến giai đoạn [PreBind](#pre-bind).

### PreBind {#pre-bind}

Các plugin này được dùng để thực hiện bất kỳ công việc nào cần thiết trước khi một Pod được bind. Ví
dụ, một plugin pre-bind có thể cấp phát (provision) một network volume và mount nó lên
node đích trước khi cho phép Pod chạy ở đó.

Nếu bất kỳ plugin PreBind nào trả về lỗi, Pod bị [từ chối](#reserve) và
được trả về hàng đợi lập lịch.

### Bind

Các plugin này được dùng để bind một Pod vào một Node. Các plugin bind sẽ không được gọi
cho đến khi tất cả các plugin PreBind hoàn tất. Mỗi plugin bind được gọi theo
thứ tự đã cấu hình. Một plugin bind có thể chọn xử lý hoặc không xử lý Pod
được giao. Nếu một plugin bind chọn xử lý một Pod, **các plugin bind còn lại sẽ bị
bỏ qua**.

### PostBind {#post-bind}

Đây là một interface mang tính cung cấp thông tin. Các plugin post-bind được gọi sau khi một
Pod được bind thành công. Đây là điểm kết thúc của một chu trình binding, và có thể được dùng
để dọn dẹp các tài nguyên liên quan.

## Plugin API

Plugin API gồm hai bước. Thứ nhất, các plugin phải đăng ký và được
cấu hình, sau đó chúng sử dụng các interface của điểm mở rộng. Các interface
của điểm mở rộng có dạng sau.

```go
type Plugin interface {
    Name() string
}

type QueueSortPlugin interface {
    Plugin
    Less(*v1.pod, *v1.pod) bool
}

type PreFilterPlugin interface {
    Plugin
    PreFilter(context.Context, *framework.CycleState, *v1.pod) error
}

// ...
```

## Cấu hình plugin (Plugin configuration)

Bạn có thể bật hoặc tắt các plugin trong cấu hình scheduler. Nếu bạn đang dùng
Kubernetes v1.18 trở lên, hầu hết các
[plugin](https://kubernetes.io/docs/reference/scheduling/config/#scheduling-plugins) lập lịch đều đang được sử dụng và
được bật theo mặc định.

Ngoài các plugin mặc định, bạn cũng có thể triển khai các plugin lập lịch
của riêng mình và cấu hình chúng cùng với các plugin mặc định. Bạn có thể truy cập
[scheduler-plugins](https://github.com/kubernetes-sigs/scheduler-plugins) để biết thêm chi tiết.

Nếu bạn đang dùng Kubernetes v1.18 trở lên, bạn có thể cấu hình một tập các plugin thành
một scheduler profile rồi định nghĩa nhiều profile để phù hợp với nhiều loại workload khác nhau.
Tìm hiểu thêm tại [nhiều profile (multiple profiles)](https://kubernetes.io/docs/reference/scheduling/config/#multiple-profiles).

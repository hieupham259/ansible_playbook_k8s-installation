# Vòng đời của Pod (Pod Lifecycle)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 3 → nhóm [3a](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời), bài 3/11 · Kiểm chứng
ở [Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md).

Đây là **bài xương sống của nhóm**, và cũng là bài dài nhất. Phần lớn độ dài đến từ các mục nằm
sau feature gate beta hoặc alpha — những thứ cluster lab không bật. Bốn cơ chế thật sự phải nắm
chỉ chiếm chừng một phần ba bài: phase, trạng thái container, `restartPolicy`, và trình tự kết
thúc êm. Đọc kỹ bốn phần đó, phần còn lại đọc lướt để biết là có.

**Phải hiểu ở lần đọc này:**

- Năm giá trị của `phase`: `Pending`, `Running`, `Succeeded`, `Failed`, `Unknown` — và phase chỉ
  là **bản tóm tắt ở mức cao**, không phải máy trạng thái đầy đủ. `CrashLoopBackOff` và
  `Terminating` là **trường Status hiển thị của kubectl**, không phải phase.
- Ba trạng thái container `Waiting`, `Running`, `Terminated`, ý nghĩa của từng cái, và cách đọc
  chúng bằng `kubectl describe pod <tên-pod>`.
- `restartPolicy` cấp Pod (`Always` mặc định, `OnFailure`, `Never`) áp cho app container và init
  container thông thường, còn **sidecar bỏ qua nó**; sau lần crash đầu, kubelet khởi động lại với
  backoff theo hàm mũ 10s, 20s, 40s… trần 300 giây, và đặt lại bộ đếm khi container chạy êm 10
  phút. Đó chính là cơ chế đứng sau `CrashLoopBackOff`.
- Trình tự **kết thúc êm**: Pod bị đánh dấu đang kết thúc → chạy hook `preStop` → container
  runtime gửi TERM (hoặc `STOPSIGNAL` của image) tới tiến trình 1 → hết
  `terminationGracePeriodSeconds` (mặc định 30 giây) → `SIGKILL` → kubelet chuyển Pod sang phase
  kết thúc rồi buộc xóa khỏi API server. Nếu `preStop` còn chạy khi hết ân hạn, kubelet chỉ xin
  gia hạn **một lần 2 giây**.
- Pod chỉ được **lập lịch một lần** trong đời. Pod chết không bao giờ được "lập lịch lại"; nó chỉ
  có thể được **thay thế** bằng một Pod mới, có thể trùng `.metadata.name` nhưng khác
  `.metadata.uid`, và không có bảo đảm nào rằng bản thay thế lên đúng node cũ.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Pod condition* và *Độ sẵn sàng của Pod* (`readinessGates`) | bài kế trình bày đủ | bài [48](48-pod-condition-vi.md) |
| *Container probe* — startup, liveness, readiness | ở đây chỉ là tóm tắt | bài [49](49-probes-vi.md) |
| *Chính sách và quy tắc khởi động lại cho từng container* (`restartPolicyRules`), *Khởi động lại tất cả container*, *Định nghĩa tín hiệu dừng tùy chỉnh* | đều nằm sau feature gate beta/alpha, cluster lab không bật | giai đoạn 8 — bài [03](03-control-plane-flags-vi.md), khi học cách bật feature gate |
| *Giảm* và *Cấu hình độ trễ khởi động lại container* (`maxContainerRestartPeriod`) | là cấu hình kubelet theo từng node | giai đoạn 8 — bài [04](04-kubelet-integration-vi.md) |
| *Thay đổi kích thước Pod* và hai condition `PodResizePending`/`PodResizeInProgress` | dựa trên requests/limits chưa học | giai đoạn 3, nhóm 3b — bài [110](110-manage-resources-containers-vi.md) |
| Việc Pod đang kết thúc bị rút khỏi EndpointSlice, condition `serving` | chưa học Service | giai đoạn 5 — bài [83](83-endpoint-slices-vi.md) |
| *Thu gom rác cho Pod* (PodGC) và condition `DisruptionTarget` | gắn với gián đoạn và eviction | giai đoạn 3, nhóm 3b — bài [53](53-disruptions-vi.md); giai đoạn 7 |
| *Hành vi của Pod khi kubelet khởi động lại* | là tình huống vận hành node | giai đoạn 12 |

---

Trang này mô tả vòng đời của một Pod. Pod tuân theo một vòng đời được định nghĩa sẵn, bắt đầu
ở [phase](#pod-phase) `Pending`, chuyển sang `Running` nếu ít nhất một trong các container
chính của nó khởi động thành công, rồi sau đó chuyển sang phase `Succeeded` hoặc `Failed`
tùy vào việc có container nào trong Pod kết thúc do lỗi hay không.

Trong khi Pod chạy, kubelet quản lý các container và chuyển đổi spec của Pod
cho container runtime. Kubelet cũng quản lý việc thực thi các
[probe](#container-probes) theo dõi sức khỏe của ứng dụng của bạn.

Giống như các container ứng dụng riêng lẻ, Pod được xem là những thực thể tương đối
phù du (ephemeral) chứ không phải bền vững. Pod được tạo ra, được gán một định danh
duy nhất ([UID](17-names-vi.md#uids)),
và được lập lịch (schedule) chạy trên các node, nơi chúng tồn tại cho đến khi bị kết thúc
(theo chính sách khởi động lại — restart policy) hoặc bị xóa.
Nếu một node chết, các Pod đang chạy trên (hoặc đã được lập lịch để chạy trên) node đó
sẽ được [đánh dấu để xóa](#pod-garbage-collection). Control plane đánh dấu các Pod này
để loại bỏ sau một khoảng thời gian chờ (timeout).

## Thời gian sống của Pod (Pod lifetime) {#pod-lifetime}

Trong khi một Pod đang chạy, kubelet có khả năng khởi động lại các container để xử lý
một số loại lỗi. Bên trong một Pod, Kubernetes theo dõi các [trạng thái](#container-states)
khác nhau của container và quyết định hành động cần thực hiện để làm cho Pod khỏe mạnh
trở lại. Việc này được thực hiện trong một [vòng lặp kiểm tra định kỳ (polling loop)](https://kubernetes.io/docs/reference/node/kubelet-sync-loop/)
liên tục đối chiếu trạng thái mong muốn (spec của Pod) với trạng thái thực tế của các
container đang chạy.

Trong Kubernetes API, Pod có cả phần đặc tả (specification) lẫn trạng thái thực tế
(actual status). Trạng thái của một đối tượng Pod bao gồm một tập các
[Pod condition](#pod-conditions). Bạn cũng có thể chèn thêm
[thông tin sẵn sàng tùy chỉnh](#pod-readiness-gate) vào dữ liệu condition của một Pod,
nếu điều đó hữu ích cho ứng dụng của bạn.

Pod chỉ được [lập lịch](136-scheduling-eviction-vi.md) một lần
duy nhất trong vòng đời của chúng; việc gán một Pod cho một node cụ thể được gọi là
_binding_, và quá trình chọn node nào để sử dụng được gọi là _scheduling_ (lập lịch).
Một khi Pod đã được lập lịch và gắn (bound) với một node, Kubernetes sẽ cố gắng
chạy Pod đó trên node. Pod chạy trên node đó cho đến khi nó dừng, hoặc cho đến khi Pod
bị [kết thúc](#pod-termination); nếu Kubernetes không thể khởi động Pod trên node đã chọn
(ví dụ, nếu node bị sập trước khi Pod khởi động), thì Pod cụ thể đó
sẽ không bao giờ khởi động.

Bạn có thể dùng [Pod Scheduling Readiness](145-pod-scheduling-readiness-vi.md)
để trì hoãn việc lập lịch cho một Pod cho đến khi tất cả các _scheduling gate_ của nó được gỡ bỏ.
Ví dụ, bạn có thể muốn định nghĩa một tập các Pod nhưng chỉ kích hoạt việc lập lịch
khi tất cả các Pod đã được tạo xong.

### Pod và việc khôi phục sau lỗi (Pods and fault recovery) {#pod-fault-recovery}

Nếu một trong các container trong Pod bị lỗi, Kubernetes có thể thử khởi động lại
container cụ thể đó.
Đọc [Cách Pod xử lý sự cố với container](#container-restarts) để tìm hiểu thêm.

Tuy nhiên, Pod cũng có thể gặp lỗi theo cách mà cluster không thể khôi phục được, và trong
trường hợp đó Kubernetes không cố gắng chữa lành Pod thêm nữa; thay vào đó, Kubernetes xóa
Pod và dựa vào các thành phần khác để cung cấp khả năng tự phục hồi.

Nếu một Pod được lập lịch lên một node và node đó sau đó gặp sự cố, Pod được xem là
không khỏe mạnh và cuối cùng Kubernetes sẽ xóa Pod đó.
Một Pod sẽ không sống sót qua một lần trục xuất (eviction) do thiếu tài nguyên
hoặc do bảo trì Node.

Kubernetes sử dụng một tầng trừu tượng cấp cao hơn, gọi là
controller, đảm nhiệm công việc quản lý các thực thể Pod tương đối "dùng một lần" này.

Một Pod cho trước (được xác định bởi một UID) không bao giờ được "lập lịch lại" sang một
node khác; thay vào đó, Pod đó có thể được thay thế bằng một Pod mới, gần như giống hệt.
Nếu bạn tạo một Pod thay thế, nó thậm chí có thể mang cùng tên (trong `.metadata.name`)
mà Pod cũ đã có, nhưng bản thay thế sẽ có `.metadata.uid` khác với Pod cũ.

Kubernetes không đảm bảo rằng bản thay thế cho một Pod hiện có sẽ được lập lịch lên
đúng node mà Pod cũ đang được thay thế từng chạy.

### Các vòng đời gắn liền (Associated lifetimes) {#associated-lifetimes}

Khi một thứ gì đó được nói là có cùng thời gian sống với một Pod, chẳng hạn như một
volume, điều đó có nghĩa là thứ đó tồn tại chừng nào Pod cụ thể ấy (với đúng UID đó)
còn tồn tại. Nếu Pod đó bị xóa vì bất kỳ lý do gì, và ngay cả khi một bản thay thế
giống hệt được tạo ra, thứ liên quan (volume, trong ví dụ này) cũng bị hủy và
được tạo mới.

![Sơ đồ Pod nhiều container](https://kubernetes.io/images/docs/pod.svg)

*Hình 1. Một Pod nhiều container chứa một [sidecar](51-sidecar-containers-vi.md) kéo tệp (file puller) và một web server. Pod dùng một [volume `emptyDir` phù du](91-volumes-vi.md#emptydir) làm nơi lưu trữ chia sẻ giữa các container.*

## Phase của Pod (Pod phase) {#pod-phase}

Trường `status` của một Pod là một đối tượng
[PodStatus](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#podstatus-v1-core),
trong đó có trường `phase`.

Phase của một Pod là một bản tóm tắt đơn giản, ở mức cao về vị trí của Pod trong
vòng đời của nó. Phase không nhằm mục đích là một bản tổng hợp đầy đủ mọi quan sát
về trạng thái container hay trạng thái Pod, cũng không nhằm trở thành một máy trạng thái
(state machine) toàn diện.

Số lượng và ý nghĩa của các giá trị phase của Pod được kiểm soát chặt chẽ.
Ngoài những gì được ghi ở đây, không nên giả định gì thêm về các Pod
có một giá trị `phase` nhất định.

Dưới đây là các giá trị có thể có của `phase`:

Giá trị     | Mô tả
:-----------|:-----------
`Pending`   | Pod đã được Kubernetes cluster chấp nhận, nhưng một hoặc nhiều container chưa được thiết lập và sẵn sàng để chạy. Giai đoạn này bao gồm thời gian Pod chờ được lập lịch cũng như thời gian tải các container image qua mạng.
`Running`   | Pod đã được gắn với một node, và tất cả các container đã được tạo. Ít nhất một container vẫn đang chạy, hoặc đang trong quá trình khởi động hay khởi động lại.
`Succeeded` | Tất cả các container trong Pod đã kết thúc thành công, và sẽ không bị khởi động lại.
`Failed`    | Tất cả các container trong Pod đã kết thúc, và ít nhất một container kết thúc do lỗi. Nghĩa là, container hoặc thoát với mã trạng thái khác 0 hoặc bị hệ thống chấm dứt, và không được thiết lập để tự động khởi động lại.
`Unknown`   | Vì lý do nào đó, không thể lấy được trạng thái của Pod. Phase này thường xảy ra do lỗi giao tiếp với node nơi Pod đáng lẽ đang chạy.

> **Ghi chú:**
>
> Khi một Pod liên tục khởi động thất bại, `CrashLoopBackOff` có thể xuất hiện trong trường `Status` của một số lệnh kubectl.
> Tương tự, khi một Pod đang bị xóa, `Terminating` có thể xuất hiện trong trường `Status` của một số lệnh kubectl.
>
> Hãy chắc chắn không nhầm lẫn _Status_ — một trường hiển thị của kubectl để người dùng dễ hình dung — với `phase` của Pod.
> Pod phase là một phần tường minh của mô hình dữ liệu Kubernetes và của
> [Pod API](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/).
>
> ```
>   NAMESPACE               NAME               READY   STATUS             RESTARTS   AGE
>   alessandras-namespace   alessandras-pod    0/1     CrashLoopBackOff   200        2d9h
> ```
>
> Một Pod được cấp một khoảng thời gian để kết thúc một cách nhẹ nhàng (gracefully), mặc định là 30 giây.
> Bạn có thể dùng cờ `--force` để [buộc kết thúc một Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination-forced).

Kể từ Kubernetes 1.27, kubelet chuyển các Pod bị xóa sang một phase kết thúc
(`Failed` hoặc `Succeeded` tùy theo mã thoát của các container trong Pod)
trước khi chúng bị xóa khỏi API server, với hai ngoại lệ:

* [static Pod](293-static-pod-tasks-vi.md) (được
  quản lý trực tiếp bởi kubelet và được đại diện bởi các mirror Pod)
* [Pod bị buộc xóa (force-deleted)](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination-forced)
  không có finalizer

Nếu một node chết hoặc bị ngắt kết nối khỏi phần còn lại của cluster, Kubernetes
áp dụng một chính sách đặt `phase` của tất cả các Pod trên node bị mất thành Failed.

## Trạng thái container (Container states) {#container-states}

Bên cạnh [phase](#pod-phase) của Pod nói chung, Kubernetes còn theo dõi trạng thái của
từng container bên trong một Pod. Bạn có thể dùng
[container lifecycle hooks](42-container-lifecycle-hooks-vi.md) để
kích hoạt các sự kiện chạy tại những thời điểm nhất định trong vòng đời của một container.

Một khi scheduler gán một Pod cho một Node, kubelet bắt đầu tạo các container cho Pod đó
bằng một container runtime.
Có ba trạng thái container khả dĩ: `Waiting`, `Running`, và `Terminated`.

Để kiểm tra trạng thái các container của một Pod, bạn có thể dùng
`kubectl describe pod <tên-pod>`. Kết quả hiển thị trạng thái của từng container
trong Pod đó.

Mỗi trạng thái có một ý nghĩa cụ thể:

### `Waiting` {#container-state-waiting}

Nếu một container không ở trạng thái `Running` hay `Terminated`, thì nó đang ở `Waiting`.
Một container ở trạng thái `Waiting` vẫn đang thực hiện các thao tác cần thiết để
hoàn tất việc khởi động: ví dụ, kéo (pull) container image từ một container image
registry, hoặc áp dụng dữ liệu Secret.
Khi bạn dùng `kubectl` để truy vấn một Pod có container đang `Waiting`, bạn cũng thấy
một trường Reason tóm tắt lý do container ở trạng thái đó.

### `Running` {#container-state-running}

Trạng thái `Running` cho biết container đang thực thi mà không có vấn đề gì. Nếu có
một hook `postStart` được cấu hình, nó đã được thực thi và hoàn tất. Khi bạn dùng
`kubectl` để truy vấn một Pod có container đang `Running`, bạn cũng thấy thông tin
về thời điểm container bước vào trạng thái `Running`.

### `Terminated` {#container-state-terminated}

Một container ở trạng thái `Terminated` đã bắt đầu thực thi và sau đó hoặc chạy đến khi
hoàn tất, hoặc thất bại vì lý do nào đó. Khi bạn dùng `kubectl` để truy vấn một Pod có
container đang `Terminated`, bạn thấy lý do, mã thoát (exit code), cùng thời điểm bắt đầu
và kết thúc của khoảng thời gian thực thi của container đó.

Nếu một container có hook `preStop` được cấu hình, hook này chạy trước khi container bước vào
trạng thái `Terminated`.

## Cách Pod xử lý sự cố với container (How Pods handle problems with containers) {#container-restarts}

Kubernetes quản lý các lỗi container bên trong Pod bằng [`restartPolicy`](#restart-policy)
được định nghĩa trong `spec` của Pod. Chính sách này quyết định cách Kubernetes phản ứng khi
container thoát do lỗi hoặc do lý do khác, diễn ra theo trình tự sau:

1. **Sự cố ban đầu (Initial crash)**: Kubernetes thử khởi động lại ngay lập tức dựa trên
   `restartPolicy` của Pod.
1. **Sự cố lặp lại (Repeated crashes)**: Sau sự cố ban đầu, Kubernetes áp dụng độ trễ
   backoff theo hàm mũ (exponential backoff) cho các lần khởi động lại tiếp theo, được mô tả trong
   [`restartPolicy`](#restart-policy).
   Điều này ngăn các nỗ lực khởi động lại nhanh, liên tiếp làm quá tải hệ thống.
1. **Trạng thái CrashLoopBackOff**: Trạng thái này cho biết cơ chế trễ backoff hiện đang
   có hiệu lực đối với một container đang trong vòng lặp sự cố (crash loop), thất bại và
   khởi động lại liên tục.
1. **Đặt lại backoff (Backoff reset)**: Nếu một container chạy thành công trong một khoảng
   thời gian nhất định (ví dụ 10 phút), Kubernetes đặt lại độ trễ backoff, xem bất kỳ
   sự cố mới nào như là sự cố đầu tiên.

Trong thực tế, `CrashLoopBackOff` là một tình trạng hoặc sự kiện có thể thấy trong kết quả
của lệnh `kubectl`, khi mô tả (describe) hoặc liệt kê (list) các Pod, khi một container trong Pod
không khởi động được đúng cách rồi cứ liên tục thử và thất bại theo vòng lặp.

Nói cách khác, khi một container rơi vào vòng lặp sự cố, Kubernetes áp dụng độ trễ
backoff theo hàm mũ được đề cập trong [Chính sách khởi động lại container](#restart-policy).
Cơ chế này ngăn một container lỗi làm hệ thống quá tải bởi các nỗ lực khởi động
thất bại liên tiếp.

`CrashLoopBackOff` có thể do những nguyên nhân như sau:

* Lỗi ứng dụng khiến container thoát.
* Lỗi cấu hình, chẳng hạn như biến môi trường sai hoặc thiếu
  tệp cấu hình.
* Hạn chế tài nguyên, khi container có thể không có đủ bộ nhớ hoặc CPU
  để khởi động đúng cách.
* Health check thất bại nếu ứng dụng không bắt đầu phục vụ trong
  thời gian dự kiến.
* Liveness probe hoặc startup probe của container trả về kết quả `Failure`
  như đề cập trong [phần probe](#container-probes).

Để điều tra nguyên nhân gốc của sự cố `CrashLoopBackOff`, người dùng có thể:

1. **Kiểm tra log**: Dùng `kubectl logs <tên-pod>` để xem log của container.
   Đây thường là cách trực tiếp nhất để chẩn đoán vấn đề gây ra sự cố.
1. **Xem xét sự kiện (event)**: Dùng `kubectl describe pod <tên-pod>` để xem các sự kiện
   của Pod, có thể cung cấp gợi ý về các vấn đề cấu hình hoặc tài nguyên.
1. **Rà soát cấu hình**: Bảo đảm rằng cấu hình Pod, bao gồm biến môi trường
   và các volume được mount, là chính xác và mọi tài nguyên bên ngoài cần thiết
   đều khả dụng.
1. **Kiểm tra giới hạn tài nguyên**: Bảo đảm container được cấp đủ CPU
   và bộ nhớ. Đôi khi, tăng tài nguyên trong định nghĩa Pod
   có thể giải quyết vấn đề.
1. **Gỡ lỗi ứng dụng (Debug application)**: Có thể tồn tại lỗi hoặc cấu hình sai trong
   mã ứng dụng. Chạy container image này cục bộ hoặc trong môi trường phát triển
   có thể giúp chẩn đoán các vấn đề đặc thù của ứng dụng.

### Khởi động lại container (Container restarts) {#restart-policy}

Khi một container trong Pod của bạn dừng, hoặc gặp lỗi, Kubernetes có thể khởi động lại nó.
Khởi động lại không phải lúc nào cũng phù hợp; ví dụ,
các init container chỉ chạy một lần (nếu thành công) trong quá trình khởi động Pod.
Bạn có thể cấu hình việc khởi động lại như một chính sách áp dụng cho tất cả các Pod, hoặc
dùng cấu hình ở cấp container (ví dụ: khi bạn định nghĩa một
sidecar container) hoặc định nghĩa phần ghi đè (override) ở cấp container.

#### Khởi động lại container và tính chống chịu (Container restarts and resilience) {#container-restart-resilience}

Dự án Kubernetes khuyến nghị tuân theo các nguyên tắc cloud-native, bao gồm thiết kế
có tính chống chịu (resilient), tính đến các lần khởi động lại đột xuất hoặc tùy ý. Bạn có thể
đạt được điều này bằng cách để Pod thất bại và dựa vào cơ chế
[thay thế](62-controllers-index-vi.md) tự động, hoặc bạn có thể
thiết kế tính chống chịu ở cấp container.
Cả hai cách tiếp cận đều giúp bảo đảm workload tổng thể của bạn vẫn khả dụng dù có
lỗi cục bộ.

#### Chính sách khởi động lại container ở cấp Pod (Pod-level container restart policy) {#pod-level-container-restart-policy}

`spec` của Pod có trường `restartPolicy` với các giá trị khả dĩ là Always, OnFailure,
và Never. Giá trị mặc định là Always.

`restartPolicy` của một Pod áp dụng cho các app container
trong Pod và cho các [init container](50-init-containers-vi.md) thông thường.
[Sidecar container](51-sidecar-containers-vi.md)
bỏ qua trường `restartPolicy` cấp Pod: trong Kubernetes, một sidecar được định nghĩa là một
mục bên trong `initContainers` có trường `restartPolicy` ở cấp container được đặt là `Always`.
Với các init container thoát do lỗi, kubelet khởi động lại init container đó nếu
`restartPolicy` ở cấp Pod là `OnFailure` hoặc `Always`:

* `Always`: Tự động khởi động lại container sau bất kỳ lần kết thúc nào.
* `OnFailure`: Chỉ khởi động lại container nếu nó thoát do lỗi (mã thoát khác 0).
* `Never`: Không tự động khởi động lại container đã kết thúc.

##### So sánh hành vi khởi động lại (Restart behavior comparison) {#restart-behavior-comparison}

Bảng sau cho thấy container hành xử như thế nào với các chính sách khởi động lại và mã thoát khác nhau:

| Mã thoát | `restartPolicy: Always` | `restartPolicy: OnFailure` | `restartPolicy: Never` | Sidecar Container |
|-----------|-------------------------|---------------------------|------------------------|-------------------|
| 0 (Thành công) | Khởi động lại | Không khởi động lại | Không khởi động lại | Luôn khởi động lại |
| Khác 0 (Lỗi) | Khởi động lại | Khởi động lại | Không khởi động lại | Luôn khởi động lại |

> **Ghi chú:**
>
> Hành vi khởi động lại đặc biệt quan trọng khi lựa chọn giữa Deployment và Job:
> - **Deployment** thường dùng `restartPolicy: Always` (giá trị duy nhất được phép) để giữ cho ứng dụng chạy liên tục
> - **Job** thường dùng `restartPolicy: OnFailure` hoặc `restartPolicy: Never` để xử lý các tác vụ xử lý theo lô (batch) một cách phù hợp
> - **Sidecar container** là các init container luôn khởi động lại bất kể `restartPolicy` của Pod, vì chúng có `restartPolicy: Always` riêng ở cấp container

##### Các kịch bản ví dụ (Example scenarios) {#example-scenarios}

Dưới đây là các ví dụ cụ thể minh họa những hành vi khởi động lại khác nhau:

**Ví dụ 1: Web server với `restartPolicy: Always` (điển hình cho Deployment)**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-server
spec:
  restartPolicy: Always  # Container khởi động lại bất kể mã thoát
  containers:
  - name: nginx
    image: nginx:1.14.2
    # Nếu container này gặp sự cố hoặc thoát vì bất kỳ lý do gì, nó sẽ được khởi động lại
```

**Ví dụ 2: Batch job với `restartPolicy: OnFailure`**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processor
spec:
  template:
    spec:
      restartPolicy: OnFailure  # Chỉ khởi động lại khi mã thoát khác 0
      containers:
      - name: processor
        image: busybox:1.28
        command: ['sh', '-c', 'echo "Processing data..."; exit 0']
        # Mã thoát 0: Job hoàn tất thành công, không khởi động lại
        # Mã thoát 1 trở lên: Container khởi động lại để thử lại tác vụ
```

**Ví dụ 3: Tác vụ một lần với `restartPolicy: Never`**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: migration-task
spec:
  restartPolicy: Never  # Không bao giờ khởi động lại, bất kể mã thoát
  containers:
  - name: migrate
    image: busybox:1.28
    command: ['sh', '-c', 'echo "Running migration..."; exit 1']
    # Ngay cả với mã thoát 1 (lỗi), container sẽ không khởi động lại
    # Pod sẽ giữ nguyên trạng thái Failed
```

##### Sidecar container và chính sách khởi động lại (Sidecar containers and restart policies) {#sidecar-containers-and-restart-policies}

[Sidecar container](51-sidecar-containers-vi.md) có hành vi khởi động lại đặc biệt, khác với các app container thông thường:

- **Sidecar container bỏ qua `restartPolicy` cấp Pod**: Chúng dùng trường `restartPolicy` riêng ở cấp container, luôn được đặt là `Always`
- **Vòng đời độc lập**: Sidecar container có thể khởi động lại độc lập với container ứng dụng chính
- **Hoạt động bền bỉ**: Sidecar container tiếp tục chạy trong suốt thời gian sống của Pod để cung cấp các dịch vụ hỗ trợ

**Ví dụ: Pod với sidecar container**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  restartPolicy: OnFailure  # Chỉ áp dụng cho container chính
  initContainers:
  - name: logging-sidecar    # Đây là một sidecar container
    image: fluent/fluent-bit:1.8
    restartPolicy: Always    # Sidecar luôn khởi động lại bất kể mã thoát
    # Cung cấp dịch vụ ghi log trong suốt thời gian sống của Pod
  containers:
  - name: main-app          # Container này tuân theo restartPolicy cấp Pod
    image: nginx:1.14.2
    # Chỉ khởi động lại khi lỗi (mã thoát khác 0) do chính sách OnFailure của Pod
```

> **Ghi chú:**
>
> Trong khi container ứng dụng chính tuân theo `restartPolicy: OnFailure` của Pod, sidecar container sẽ khởi động lại bất kể mã thoát của nó, vì sidecar container luôn có `restartPolicy: Always` ở cấp container.

Khi kubelet xử lý việc khởi động lại container theo chính sách khởi động lại đã cấu hình,
điều đó chỉ áp dụng cho các lần khởi động lại tạo ra container thay thế bên trong
cùng một Pod và chạy trên cùng một node. Sau khi các container trong một Pod thoát, kubelet
khởi động lại chúng với độ trễ backoff theo hàm mũ (10s, 20s, 40s, …), với mức trần là
300 giây (5 phút). Một khi một container đã chạy 10 phút mà không gặp
vấn đề gì, kubelet đặt lại bộ đếm backoff khởi động lại cho container đó.
[Sidecar container và vòng đời Pod](51-sidecar-containers-vi.md#sidecar-containers-and-pod-lifecycle)
giải thích hành vi của `init containers` khi chỉ định trường `restartPolicy` trên chúng.

#### Chính sách và quy tắc khởi động lại cho từng container (Individual container restart policy and rules) {#container-restart-rules}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [beta]`

Nếu cluster của bạn bật feature gate `ContainerRestartRules`, bạn có thể chỉ định
`restartPolicy` và `restartPolicyRules` trên _từng container riêng lẻ_ để ghi đè chính sách
khởi động lại của Pod. Chính sách và quy tắc khởi động lại của container áp dụng cho các app container
trong Pod và cho các [init container](50-init-containers-vi.md) thông thường.

Một [sidecar container](51-sidecar-containers-vi.md)
gốc của Kubernetes có `restartPolicy` ở cấp container được đặt là `Always`.

Các lần khởi động lại container sẽ tuân theo cùng cơ chế backoff theo hàm mũ như chính sách khởi động lại của Pod đã mô tả ở trên.
Các chính sách khởi động lại container được hỗ trợ:

* `Always`: Tự động khởi động lại container sau bất kỳ lần kết thúc nào.
* `OnFailure`: Chỉ khởi động lại container nếu nó thoát do lỗi (mã thoát khác 0).
* `Never`: Không tự động khởi động lại container đã kết thúc.

Ngoài ra, _từng container riêng lẻ_ có thể chỉ định `restartPolicyRules`. Nếu trường
`restartPolicyRules` được chỉ định, thì `restartPolicy` của container **bắt buộc** cũng phải được
chỉ định. `restartPolicyRules` định nghĩa một danh sách quy tắc áp dụng khi container thoát.
Mỗi quy tắc gồm một điều kiện và một hành động. Điều kiện được hỗ trợ là `exitCodes`, so sánh
mã thoát của container với một danh sách giá trị cho trước. Hành động được hỗ trợ là `Restart`,
nghĩa là container sẽ được khởi động lại. Các quy tắc được đánh giá theo thứ tự. Khi khớp quy tắc
đầu tiên, hành động sẽ được áp dụng. Nếu không có điều kiện nào của các quy tắc khớp, Kubernetes
quay về `restartPolicy` đã cấu hình của container.

Ví dụ, một Pod với chính sách khởi động lại OnFailure có một container `try-once`. Điều này cho phép
Pod chỉ khởi động lại một số container nhất định:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: on-failure-pod
spec:
  restartPolicy: OnFailure
  containers:
  - name: try-once-container    # Container này chỉ chạy một lần vì restartPolicy là Never.
    image: registry.k8s.io/busybox:1.27.2
    command: ['sh', '-c', 'echo "Only running once" && sleep 10 && exit 1']
    restartPolicy: Never     
  - name: on-failure-container  # Container này sẽ được khởi động lại khi lỗi.
    image: registry.k8s.io/busybox:1.27.2
    command: ['sh', '-c', 'echo "Keep restarting" && sleep 1800 && exit 1']
```

Một Pod với chính sách khởi động lại `Always` có một init container chỉ thực thi một lần. Nếu init
container thất bại, Pod thất bại. Điều này cho phép Pod thất bại nếu quá trình khởi tạo thất bại,
nhưng vẫn tiếp tục chạy một khi khởi tạo thành công:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fail-pod-if-init-fails
spec:
  restartPolicy: Always
  initContainers:
  - name: init-once      # Init container này chỉ thử một lần. Nếu thất bại, Pod sẽ thất bại.
    image: registry.k8s.io/busybox:1.27.2
    command: ['sh', '-c', 'echo "Failing initialization" && sleep 10 && exit 1']
    restartPolicy: Never
  containers:
  - name: main-container # Container này sẽ luôn được khởi động lại một khi khởi tạo thành công.
    image: registry.k8s.io/busybox:1.27.2
    command: ['sh', '-c', 'sleep 1800 && exit 0']
```

Một Pod với chính sách khởi động lại Never có một container bỏ qua và khởi động lại theo các mã thoát cụ thể.
Điều này hữu ích để phân biệt giữa lỗi có thể khởi động lại và lỗi không thể khởi động lại:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restart-on-exit-codes
spec:
  restartPolicy: Never
  containers:
  - name: restart-on-exit-codes
    image: registry.k8s.io/busybox:1.27.2
    command: ['sh', '-c', 'sleep 60 && exit 0']
    restartPolicy: Never     # Phải chỉ định chính sách khởi động lại của container nếu có chỉ định quy tắc
    restartPolicyRules:      # Chỉ khởi động lại container nếu nó thoát với mã 42
    - action: Restart
      exitCodes:
        operator: In
        values: [42]
```

Các quy tắc khởi động lại có thể được dùng cho nhiều kịch bản quản lý vòng đời nâng cao hơn nữa.
Lưu ý, quy tắc khởi động lại chịu ảnh hưởng bởi cùng những bất nhất như chính sách khởi động lại
thông thường. Việc kubelet khởi động lại, thu gom rác (garbage collection) của container runtime,
sự cố kết nối gián đoạn với control plane có thể gây mất trạng thái và các container có thể bị
chạy lại ngay cả khi bạn kỳ vọng một container không bị khởi động lại.

#### Khởi động lại tất cả container (Restart All Containers) {#restart-all-containers}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [beta]`

Nếu cluster của bạn bật feature gate `RestartAllContainersOnContainerExits`, bạn có thể chỉ định
`RestartAllContainers` làm hành động trong `restartPolicyRules` ở cấp container. Khi việc thoát
của một container khớp với một quy tắc có hành động này, toàn bộ Pod bị kết thúc và khởi động lại
tại chỗ (in-place).

Kiểu khởi động lại "tại chỗ" này mang lại cách hiệu quả hơn để đặt lại trạng thái của Pod so với
việc xóa hoàn toàn rồi tạo lại. Điều này đặc biệt có giá trị với các workload mà việc lập lịch lại
tốn kém, chẳng hạn như batch job hoặc các tác vụ huấn luyện AI/ML.

##### Cách hoạt động của khởi động lại Pod tại chỗ (How in-place Pod restarts work) {#how-in-place-pod-restarts-work}

Khi hành động `RestartAllContainers` được kích hoạt, kubelet thực hiện các bước sau:

1. **Kết thúc nhanh (Fast Termination)**: Tất cả các container đang chạy trong Pod bị kết thúc.
   `terminationGracePeriodSeconds` đã cấu hình không được tôn trọng, và mọi hook `preStop` đã cấu hình
   sẽ không được thực thi. Điều này bảo đảm việc tắt diễn ra nhanh chóng.
1. **Bảo toàn tài nguyên của Pod (Preservation of Pod Resources)**: Các tài nguyên thiết yếu của Pod được giữ nguyên:

   * UID, địa chỉ IP và network namespace của Pod
   * Pod sandbox và mọi thiết bị được gắn kèm
   * Tất cả các volume, bao gồm `emptyDir` và các volume được mount

1. **Cập nhật trạng thái Pod (Pod Status Update)**: Trạng thái của Pod được cập nhật với condition
   `PodRestartInPlace` đặt là `True`. Điều này làm cho quá trình khởi động lại có thể quan sát được.
1. **Trình tự khởi động lại đầy đủ (Full Restart Sequence)**: Một khi tất cả các container đã kết thúc,
   condition `PodRestartInPlace` được đặt về `False`, và Pod bắt đầu quy trình khởi động tiêu chuẩn:

   * **Các init container được chạy lại** theo thứ tự.
   * Sidecar và các container thông thường được khởi động.

Một khía cạnh then chốt của tính năng này là **tất cả** các container đều được khởi động lại, kể cả
những container trước đó đã hoàn tất thành công hoặc thất bại. Hành động `RestartAllContainers` ghi đè
mọi `restartPolicy` đã cấu hình ở cấp container hoặc cấp Pod.

Cơ chế này hữu ích trong các kịch bản cần một khởi đầu sạch (clean slate) cho tất cả các container, chẳng hạn:

- Khi một init container thiết lập một môi trường có thể bị hỏng, tính năng này bảo đảm
  quy trình thiết lập được thực thi lại.
- Một sidecar container có thể giám sát sức khỏe của ứng dụng chính và kích hoạt khởi động lại
  toàn bộ Pod nếu ứng dụng rơi vào trạng thái không thể khôi phục.

Hãy xét một workload trong đó một sidecar giám sát (watcher) chịu trách nhiệm khởi động lại
ứng dụng chính từ một trạng thái tốt đã biết nếu nó gặp lỗi. Watcher có thể thoát với một mã cụ thể
để kích hoạt khởi động lại toàn bộ, tại chỗ, của Pod worker.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-worker
spec:
  restartPolicy: Never # Bản thân Pod không nên khởi động lại trừ khi được yêu cầu tường minh.
  initContainers:
  - name: setup-environment
    image: registry.k8s.io/busybox:1.27.2
    command: ['sh', '-c', 'echo "Setting up environment"']
    # Init container này chạy một lần để chuẩn bị môi trường.
    # Nó sẽ chạy lại sau một hành động RestartAllContainers.
  - name: watcher-sidecar
    image: registry.k8s.io/busybox:1.27.2
    # Trong kịch bản thực tế, đây sẽ là một image giám sát chuyên dụng.
    # Lệnh này mô phỏng việc watcher thoát với một mã đặc biệt.
    command: ['sh', '-c', 'sleep 60; exit 88']
    restartPolicy: Always
    restartPolicyRules:
    - action: RestartAllContainers
      exitCodes:
        # Mã thoát 88 kích hoạt khởi động lại toàn bộ Pod.
        operator: In
        values: [88]
  containers:
  - name: main-application
    image: registry.k8s.io/busybox:1.27.2
    command: ['sh', '-c', 'echo "Application is running"; sleep 3600']
```

Trong ví dụ này:

- `restartPolicy` tổng thể của Pod là `Never`.
- `watcher-sidecar` chạy một lệnh rồi thoát với mã `88`.
- Mã thoát khớp với quy tắc, kích hoạt hành động `RestartAllContainers`.
- Toàn bộ Pod, bao gồm init container `setup-environment` và container `main-application`,
  sau đó được khởi động lại tại chỗ. Pod giữ nguyên UID, sandbox, IP và các volume của nó.

### Giảm độ trễ khởi động lại container (Reduced container restart delay) {#reduced-container-restart-delay}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.33 [alpha]`

Khi bật feature gate alpha `ReduceDefaultCrashLoopBackOffDecay`,
các lần thử khởi động container trên toàn cluster của bạn sẽ được giảm xuống, bắt đầu ở 1s
(thay vì 10s) và tăng theo hàm mũ gấp 2 lần sau mỗi lần khởi động lại cho đến độ trễ
tối đa là 60s (thay vì 300s tức 5 phút).

Nếu bạn dùng tính năng này cùng với tính năng alpha
`KubeletCrashLoopBackOffMax` (mô tả bên dưới), các node riêng lẻ có thể có
độ trễ tối đa khác nhau.

### Cấu hình độ trễ khởi động lại container (Configurable container restart delay) {#configurable-container-restart-delay}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [beta]`

Khi bật feature gate `KubeletCrashLoopBackOffMax`, bạn có thể
cấu hình lại độ trễ tối đa giữa các lần thử khởi động container so với mặc định
là 300s (5 phút). Cấu hình này được đặt cho từng node bằng cấu hình kubelet.
Trong [cấu hình kubelet](224-kubelet-config-file-vi.md) của bạn,
dưới mục `crashLoopBackOff`, đặt trường `maxContainerRestartPeriod` trong khoảng từ `"1s"` đến
`"300s"`. Như mô tả ở trên trong [Chính sách khởi động lại container](#restart-policy),
độ trễ trên node đó vẫn bắt đầu ở 10s và tăng theo hàm mũ gấp 2 lần
sau mỗi lần khởi động lại, nhưng giờ sẽ bị giới hạn ở mức tối đa bạn đã cấu hình. Nếu
`maxContainerRestartPeriod` bạn cấu hình nhỏ hơn giá trị khởi đầu mặc định
là 10s, độ trễ ban đầu sẽ được đặt bằng mức tối đa đã cấu hình.

Xem các ví dụ cấu hình kubelet sau:

```yaml
# độ trễ khởi động lại container sẽ bắt đầu ở 10s, tăng
# gấp 2 lần mỗi lần khởi động lại, tối đa 100s
kind: KubeletConfiguration
crashLoopBackOff:
    maxContainerRestartPeriod: "100s"
```

```yaml
# độ trễ giữa các lần khởi động lại container sẽ luôn là 2s
kind: KubeletConfiguration
crashLoopBackOff:
    maxContainerRestartPeriod: "2s"
```

Nếu bạn dùng tính năng này cùng với tính năng alpha
`ReduceDefaultCrashLoopBackOffDecay` (mô tả ở trên), giá trị mặc định của cluster
cho backoff ban đầu và backoff tối đa sẽ không còn là 10s và 300s, mà là 1s
và 60s. Cấu hình theo từng node được ưu tiên hơn các giá trị mặc định do
`ReduceDefaultCrashLoopBackOffDecay` đặt ra, ngay cả khi điều này dẫn đến một node có
backoff tối đa dài hơn các node khác trong cluster.

## Pod condition (Pod conditions) {#pod-conditions}

Một Pod có PodStatus, trong đó có một mảng các
[PodCondition](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#podcondition-v1-core)
mà Pod đã hoặc chưa vượt qua. Kubelet quản lý các
PodCondition sau:

* `PodScheduled`: Pod đã được lập lịch lên một node.
* `PodReadyToStartContainers`: (tính năng beta; được bật [mặc định](#pod-ready-to-start-containers))
  Pod sandbox đã được tạo thành công, mạng đã được cấu hình, các volume lưu trữ đã được mount,
  và mọi tài nguyên động (nếu được yêu cầu) đã được cấp phát.
* `ContainersReady`: tất cả các container trong Pod đã sẵn sàng.
* `Initialized`: tất cả các [init container](50-init-containers-vi.md)
  đã hoàn tất thành công.
* `Ready`: Pod có khả năng phục vụ các yêu cầu và nên được thêm vào các nhóm
  cân bằng tải (load balancing pool) của tất cả các Service khớp với nó.
* `DisruptionTarget`: Pod sắp bị kết thúc do một sự gián đoạn (disruption) — chẳng hạn preemption, eviction hoặc thu gom rác (garbage-collection).
* `PodResizePending`: một yêu cầu thay đổi kích thước (resize) Pod đã được gửi nhưng không thể áp dụng. Xem [Trạng thái resize Pod](289-resize-container-resources-vi.md#pod-resize-status).
* `PodResizeInProgress`: Pod đang trong quá trình thay đổi kích thước. Xem
  [Trạng thái resize Pod](289-resize-container-resources-vi.md#pod-resize-status).

Tên trường           | Mô tả
:--------------------|:-----------
`type`               | Tên của Pod condition này.
`status`             | Cho biết condition đó có áp dụng hay không, với các giá trị khả dĩ "`True`", "`False`", hoặc "`Unknown`".
`lastProbeTime`      | Dấu thời gian của lần probe gần nhất đối với Pod condition này.
`lastTransitionTime` | Dấu thời gian của lần gần nhất Pod chuyển từ trạng thái này sang trạng thái khác.
`reason`             | Chuỗi văn bản dạng máy đọc được, viết theo UpperCamelCase, cho biết lý do của lần chuyển trạng thái gần nhất của condition.
`message`            | Thông điệp dạng người đọc được, cho biết chi tiết về lần chuyển trạng thái gần nhất.

### Độ sẵn sàng của Pod (Pod readiness) {#pod-readiness-gate}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.14 [stable]`

Ứng dụng của bạn có thể chèn phản hồi hoặc tín hiệu bổ sung vào PodStatus:
_Pod readiness_ (độ sẵn sàng của Pod). Để dùng tính năng này, hãy đặt `readinessGates` trong `spec` của Pod để
chỉ định một danh sách các condition bổ sung mà kubelet đánh giá cho độ sẵn sàng của Pod.

Readiness gate được xác định bởi trạng thái hiện tại của các trường `status.condition`
của Pod. Nếu Kubernetes không tìm thấy condition như vậy trong trường
`status.conditions` của một Pod, trạng thái của condition đó
được mặc định là "`False`".

Dưới đây là một ví dụ:

```yaml
kind: Pod
...
spec:
  readinessGates:
    - conditionType: "www.example.com/feature-1"
status:
  conditions:
    - type: Ready                              # một PodCondition có sẵn (built-in)
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
    - type: "www.example.com/feature-1"        # một PodCondition bổ sung
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
  containerStatuses:
    - containerID: docker://abcd...
      ready: true
...
```

Các Pod condition bạn thêm vào phải có tên tuân theo
[định dạng khóa label](18-labels-vi.md#syntax-and-character-set) của Kubernetes.

### Trạng thái cho độ sẵn sàng của Pod (Status for Pod readiness) {#pod-readiness-status}

Lệnh `kubectl patch` không hỗ trợ vá (patch) trạng thái (status) của đối tượng.
Để đặt các `status.conditions` này cho Pod, ứng dụng và các
operator nên dùng hành động `PATCH`.
Bạn có thể dùng một [thư viện client Kubernetes](https://kubernetes.io/docs/reference/using-api/client-libraries/) để
viết mã đặt các Pod condition tùy chỉnh cho độ sẵn sàng của Pod.

Với một Pod dùng các condition tùy chỉnh, Pod đó **chỉ** được đánh giá là sẵn sàng
khi cả hai điều sau cùng đúng:

* Tất cả các container trong Pod đều sẵn sàng.
* Tất cả các condition được chỉ định trong `readinessGates` đều là `True`.

Khi các container của một Pod đã Ready nhưng ít nhất một condition tùy chỉnh bị thiếu hoặc
là `False`, kubelet đặt [condition](#pod-conditions) của Pod thành `ContainersReady`.

### Độ sẵn sàng của Pod để bắt đầu chạy container (Pod readiness to start containers) {#pod-ready-to-start-containers}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.29 [beta]`

> **Ghi chú:**
>
> Trong giai đoạn phát triển ban đầu, condition này có tên là `PodHasNetwork`.

Sau khi một Pod được lập lịch lên một node, nó cần được kubelet chấp nhận (admit) và
cần có các volume lưu trữ cần thiết được mount. Một khi các giai đoạn này hoàn tất,
kubelet phối hợp với một container runtime (sử dụng CRI) để thiết lập một
runtime sandbox và cấu hình mạng cho Pod. Nếu Pod dùng
[Dynamic Resource Allocation](149-dynamic-resource-allocation-vi.md),
các tài nguyên đó cũng được cấp phát trong giai đoạn này.
Nếu [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`PodReadyToStartContainersCondition` được bật
(nó được bật mặc định cho Kubernetes v1.36), condition
`PodReadyToStartContainers` sẽ được thêm vào trường `status.conditions` của Pod.

Condition `PodReadyToStartContainers` được kubelet đặt là `False` khi nó phát hiện một
Pod không có runtime sandbox với mạng đã được cấu hình. Điều này xảy ra trong
các kịch bản sau:

- Ở giai đoạn đầu vòng đời của Pod, khi kubelet chưa bắt đầu thiết lập sandbox cho
  Pod bằng container runtime.
- Ở giai đoạn sau trong vòng đời của Pod, khi Pod sandbox đã bị hủy do một trong hai nguyên nhân:
  - node khởi động lại, mà Pod không bị trục xuất (evict)
  - với các container runtime dùng máy ảo để cách ly, máy ảo sandbox của Pod
    khởi động lại, đòi hỏi phải tạo một sandbox mới và
    cấu hình mạng container mới.

Sau khi việc tạo sandbox, cấu hình mạng, mount volume, và (nếu được yêu cầu) cấp phát tài nguyên
động hoàn tất, kubelet đặt condition `PodReadyToStartContainers` thành `True`.
Việc kéo image và tạo container diễn ra sau thời điểm này.

Với một Pod có init container, kubelet đặt condition `Initialized` thành
`True` sau khi các init container đã hoàn tất thành công (điều này xảy ra
sau khi runtime plugin tạo sandbox và cấu hình mạng thành công).
Với một Pod không có init container, kubelet đặt condition `Initialized`
thành `True` trước khi việc tạo sandbox và cấu hình mạng bắt đầu.

## Thay đổi kích thước Pod (Resizing Pods) {#pod-resize}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [stable]`

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [beta]`

Kubernetes hỗ trợ thay đổi tài nguyên CPU và bộ nhớ được cấp cho Pod
sau khi chúng đã được tạo. (Với các tài nguyên hạ tầng khác, bạn sẽ cần
dùng các kỹ thuật khác đặc thù cho những tài nguyên đó.) Có hai cách tiếp cận chính
để thay đổi kích thước CPU và bộ nhớ:

### Thay đổi kích thước Pod tại chỗ (In-place Pod resize) {#pod-resize-inplace}

Bạn có thể thay đổi tài nguyên CPU và bộ nhớ ở cấp container của một Pod mà không cần tạo lại Pod.
Cách này còn được gọi là _in-place Pod vertical scaling_ (co giãn dọc Pod tại chỗ). Nó cho phép bạn điều chỉnh
việc cấp phát tài nguyên cho các container đang chạy trong khi có khả năng tránh gián đoạn ứng dụng.

Nếu bạn đã chỉ định tài nguyên ở cấp Pod (pod-level), bạn cũng có thể thay đổi kích thước chúng tại chỗ.
Để biết thêm chi tiết, xem [Thay đổi tài nguyên CPU và bộ nhớ gán cho Pod](290-resize-pod-resources-vi.md).

Để thực hiện thay đổi kích thước tại chỗ, bạn cập nhật trạng thái mong muốn của Pod bằng
subresource `/resize`. Sau đó kubelet cố gắng áp dụng các giá trị tài nguyên mới cho các
container đang chạy. Các condition
`PodResizePending` và `PodResizeInProgress` của Pod (mô tả trong [Pod condition](#pod-conditions))
cho biết trạng thái của thao tác thay đổi kích thước. Để biết thêm chi tiết về trạng thái resize, xem
[Trạng thái resize container](https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources#container-resize-status).

Những điểm cần cân nhắc chính cho thay đổi kích thước tại chỗ:
- Chỉ có tài nguyên CPU và bộ nhớ mới có thể thay đổi kích thước tại chỗ.
- [Lớp Chất lượng Dịch vụ (QoS class)](54-pod-qos-vi.md) của Pod
  được xác định lúc tạo và không thể thay đổi bằng việc resize.
- Bạn có thể cấu hình việc có cần khởi động lại container khi resize hay không bằng
  `resizePolicy` trong đặc tả container.

Để có hướng dẫn chi tiết về cách thực hiện thay đổi kích thước tại chỗ, xem
[Thay đổi tài nguyên CPU và bộ nhớ gán cho Container](289-resize-container-resources-vi.md).

### Thay đổi kích thước bằng cách khởi chạy Pod thay thế (Resizing by launching replacement Pods) {#resizing-by-launching-replacement-pods}

Cách tiếp cận thuần cloud-native hơn để thay đổi tài nguyên của Pod là thông qua
workload resource quản lý nó (chẳng hạn Deployment hoặc StatefulSet).
Khi bạn cập nhật đặc tả tài nguyên trong Pod template,
controller của workload sẽ tạo các Pod mới với tài nguyên đã cập nhật và kết thúc
các Pod cũ theo chiến lược cập nhật (update strategy) của nó.

Cách tiếp cận này:
- Hoạt động với mọi phiên bản Kubernetes.
- Có thể thay đổi bất kỳ phần nào của đặc tả Pod, không chỉ tài nguyên.
- Dẫn đến việc thay thế Pod, do đó bạn nên thiết kế workload của mình để xử lý
  [các gián đoạn có kế hoạch](53-disruptions-vi.md). Cân nhắc dùng một
  [PodDisruptionBudget](339-configure-pdb-vi.md) để kiểm soát tính khả dụng.
- Yêu cầu các Pod của bạn được quản lý bởi một workload resource.

Bạn cũng có thể dùng
[VerticalPodAutoscaler](73-vertical-pod-autoscale-vi.md)
để tự động quản lý các khuyến nghị và cập nhật tài nguyên cho Pod.

## Container probe (Container probes) {#container-probes}

Kubernetes cho phép bạn định nghĩa các _probe_ để giám sát liên tục sức khỏe
của các container trong một Pod. Probe là một chẩn đoán được kubelet thực hiện
định kỳ trên một container.
Để thực hiện chẩn đoán, kubelet hoặc thực thi mã bên trong
container hoặc gửi một yêu cầu mạng.

Dựa trên kết quả probe, Kubernetes có thể khởi động lại các container không khỏe mạnh
hoặc ngừng gửi lưu lượng (traffic) đến các container chưa sẵn sàng.

Kubelet có thể tùy chọn thực hiện và phản ứng với ba loại probe trên các container
đang chạy, mỗi loại phục vụ một mục đích khác nhau. Về các cơ chế probe (`exec`,
`grpc`, `httpGet`, `tcpSocket`), các trường cấu hình, và hướng dẫn sử dụng chi tiết,
xem [Liveness, Readiness và Startup Probe](49-probes-vi.md).

### Startup probe {#startup-probe}

Startup probe xác minh liệu ứng dụng bên trong một container đã khởi động hay chưa.
Nếu một startup probe được cấu hình, Kubernetes không thực thi liveness probe hay
readiness probe cho đến khi startup probe thành công, cho ứng dụng
thời gian để hoàn tất quá trình khởi tạo.

Loại probe này chỉ được thực thi lúc khởi động, khác với liveness và readiness
probe vốn được chạy định kỳ.

Nếu startup probe thất bại, kubelet giết container đó, và container
sẽ chịu tác động của [chính sách khởi động lại](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy) của nó.

### Liveness probe {#liveness-probe}

Liveness probe xác định khi nào cần khởi động lại một container.
Ví dụ, liveness probe có thể phát hiện tình trạng khóa chết (deadlock), khi một ứng dụng
đang chạy nhưng không thể tiến triển. Khởi động lại container trong trạng thái như vậy
có thể giúp ứng dụng khả dụng hơn dù còn lỗi.

Nếu một container thất bại liveness probe nhiều lần hơn mức chịu đựng đã cấu hình,
kubelet khởi động lại container đó.
Liveness probe không chờ readiness probe thành công. Nếu bạn muốn
chờ trước khi thực thi liveness probe, bạn có thể định nghĩa
`initialDelaySeconds` hoặc dùng một startup probe.

### Readiness probe {#readiness-probe}

Readiness probe xác định khi nào một container sẵn sàng nhận lưu lượng.
Điều này hữu ích khi chờ một ứng dụng thực hiện các tác vụ khởi tạo tốn thời gian,
chẳng hạn như thiết lập kết nối mạng, tải tệp, và làm nóng
bộ đệm (cache).
Readiness probe cũng có thể hữu ích ở giai đoạn sau trong vòng đời của container,
ví dụ, khi khôi phục từ các lỗi tạm thời hoặc tình trạng quá tải.

Nếu readiness probe trả về trạng thái thất bại, controller
EndpointSlice sẽ loại bỏ địa chỉ IP của Pod khỏi các EndpointSlice của tất cả các Service
khớp với Pod đó.

Readiness probe chạy trên container trong suốt toàn bộ vòng đời của nó.

## Kết thúc Pod (Termination of Pods) {#pod-termination}

Vì Pod đại diện cho các tiến trình chạy trên các node trong cluster, điều quan trọng là
cho phép các tiến trình đó kết thúc một cách nhẹ nhàng (gracefully) khi không còn cần đến
(thay vì bị dừng đột ngột bằng tín hiệu `KILL` mà không có cơ hội dọn dẹp).

Mục tiêu thiết kế là để bạn có thể yêu cầu xóa và biết khi nào các tiến trình
kết thúc, nhưng đồng thời bảo đảm rằng việc xóa cuối cùng sẽ hoàn tất.
Khi bạn yêu cầu xóa một Pod, cluster ghi nhận và theo dõi khoảng thời gian ân hạn (grace period)
dự kiến trước khi Pod được phép bị giết cưỡng bức. Với cơ chế theo dõi việc tắt cưỡng bức đó,
kubelet cố gắng tắt một cách nhẹ nhàng.

Thông thường, với việc kết thúc nhẹ nhàng này của Pod, kubelet gửi yêu cầu tới container runtime
để cố gắng dừng các container trong Pod bằng cách trước tiên gửi tín hiệu TERM (còn gọi là SIGTERM),
kèm theo một thời gian chờ ân hạn, đến tiến trình chính trong mỗi container.
Các yêu cầu dừng container được container runtime xử lý bất đồng bộ.
Không có bảo đảm nào về thứ tự xử lý của các yêu cầu này.
Nhiều container runtime tôn trọng giá trị `STOPSIGNAL` được định nghĩa trong container image và,
nếu khác, sẽ gửi tín hiệu STOPSIGNAL được cấu hình trong container image thay vì TERM.
Một khi thời gian ân hạn hết, tín hiệu KILL được gửi đến mọi tiến trình
còn lại, và sau đó Pod bị xóa khỏi
API Server. Nếu kubelet hoặc dịch vụ quản lý của container runtime bị khởi động lại trong khi
đang chờ các tiến trình kết thúc, cluster sẽ thử lại từ đầu, bao gồm toàn bộ
thời gian ân hạn ban đầu.

### Tín hiệu dừng (Stop Signals) {#pod-termination-stop-signals}

Tín hiệu dừng dùng để giết container có thể được định nghĩa trong container image bằng chỉ thị `STOPSIGNAL`.
Nếu không có tín hiệu dừng nào được định nghĩa trong image, tín hiệu mặc định của container runtime
(SIGTERM với cả containerd lẫn CRI-O) sẽ được dùng để giết container.

### Định nghĩa tín hiệu dừng tùy chỉnh (Defining custom stop signals) {#defining-custom-stop-signals}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.33 [alpha]`

Nếu feature gate `ContainerStopSignals` được bật, bạn có thể cấu hình một tín hiệu dừng tùy chỉnh
cho các container của mình từ phần Lifecycle của container. Chúng tôi yêu cầu trường `spec.os.name`
của Pod phải hiện diện như một điều kiện để định nghĩa tín hiệu dừng trong lifecycle của container.
Danh sách các tín hiệu hợp lệ phụ thuộc vào hệ điều hành mà Pod được lập lịch lên.
Với các Pod được lập lịch lên node Windows, chỉ SIGTERM và SIGKILL được hỗ trợ làm tín hiệu hợp lệ.

Dưới đây là một ví dụ Pod spec định nghĩa một tín hiệu dừng tùy chỉnh:

```yaml
spec:
  os:
    name: linux
  containers:
    - name: my-container
      image: container-image:latest
      lifecycle:
        stopSignal: SIGUSR1
```

Nếu một tín hiệu dừng được định nghĩa trong lifecycle, nó sẽ ghi đè tín hiệu được định nghĩa trong container image.
Nếu không có tín hiệu dừng nào được định nghĩa trong spec của container, container sẽ quay về hành vi mặc định.

### Luồng kết thúc Pod (Pod Termination Flow) {#pod-termination-flow}

Luồng kết thúc Pod, minh họa bằng một ví dụ:

1. Bạn dùng công cụ `kubectl` để xóa thủ công một Pod cụ thể, với thời gian ân hạn mặc định
   (30 giây).

1. Pod trong API server được cập nhật với mốc thời gian mà quá thời điểm đó Pod bị coi là "chết",
   cùng với thời gian ân hạn.
   Nếu bạn dùng `kubectl describe` để kiểm tra Pod bạn đang xóa, Pod đó hiển thị là "Terminating".
   Trên node nơi Pod đang chạy: ngay khi kubelet thấy một Pod đã bị đánh dấu
   là đang kết thúc (một khoảng thời gian tắt nhẹ nhàng đã được đặt), kubelet bắt đầu quy trình
   tắt Pod cục bộ.

   1. Nếu một trong các container của Pod có định nghĩa một
      [hook](42-container-lifecycle-hooks-vi.md) `preStop` và `terminationGracePeriodSeconds`
      trong Pod spec không được đặt là 0, kubelet chạy hook đó bên trong container.
      Thiết lập `terminationGracePeriodSeconds` mặc định là 30 giây.

      Nếu hook `preStop` vẫn đang chạy sau khi thời gian ân hạn hết, kubelet yêu cầu
      một khoảng gia hạn ân hạn nhỏ, một lần duy nhất, là 2 giây.

   > **Ghi chú:**
   >
   > Nếu hook `preStop` cần nhiều thời gian hơn để hoàn tất so với mức thời gian ân hạn mặc định cho phép,
   > bạn phải điều chỉnh `terminationGracePeriodSeconds` cho phù hợp.

   1. Kubelet kích hoạt container runtime gửi tín hiệu TERM đến tiến trình 1 bên trong mỗi
      container.

      Có một [thứ tự đặc biệt](#termination-with-sidecars) nếu Pod có định nghĩa
      sidecar container.
      Nếu không, các container trong Pod nhận tín hiệu TERM vào những thời điểm khác nhau và theo
      thứ tự tùy ý. Nếu thứ tự tắt là quan trọng, hãy cân nhắc dùng hook `preStop`
      để đồng bộ hóa (hoặc chuyển sang dùng sidecar container).

1. Cùng lúc kubelet bắt đầu tắt Pod một cách nhẹ nhàng, control plane
   đánh giá xem có nên loại Pod đang tắt đó khỏi các đối tượng EndpointSlice hay không,
   trong đó các đối tượng này đại diện cho một Service
   có cấu hình selector.
   ReplicaSet và các workload resource khác
   không còn coi Pod đang tắt là một replica hợp lệ, đang phục vụ.

   Các Pod tắt chậm không nên tiếp tục phục vụ lưu lượng thông thường mà nên bắt đầu
   kết thúc và hoàn tất xử lý các kết nối đang mở. Một số ứng dụng cần đi xa hơn việc
   hoàn tất các kết nối đang mở và cần sự kết thúc nhẹ nhàng hơn nữa, ví dụ,
   rút phiên (session draining) và hoàn tất phiên.

   Các endpoint đại diện cho các Pod đang kết thúc không bị loại ngay lập tức khỏi
   EndpointSlice, và một trạng thái cho biết [trạng thái đang kết thúc](83-endpoint-slices-vi.md#conditions)
   được phơi bày từ EndpointSlice API.
   Các endpoint đang kết thúc luôn có trạng thái `ready` là `false` (để tương thích ngược
   với các phiên bản trước 1.26), vì vậy các bộ cân bằng tải sẽ không dùng chúng cho lưu lượng thông thường.

   Nếu cần rút lưu lượng (traffic draining) trên Pod đang kết thúc, độ sẵn sàng thực tế có thể được kiểm tra qua
   condition `serving`. Bạn có thể tìm thêm chi tiết về cách triển khai rút kết nối trong
   bài hướng dẫn [Luồng kết thúc Pod và Endpoint](https://kubernetes.io/docs/tutorials/services/pods-and-endpoint-termination-flow/)

   <a id="pod-termination-beyond-grace-period" />

1. Kubelet bảo đảm Pod được tắt và kết thúc

   1. Khi thời gian ân hạn hết, nếu vẫn còn bất kỳ container nào đang chạy trong Pod,
      kubelet kích hoạt việc tắt cưỡng bức.
      Container runtime gửi `SIGKILL` đến mọi tiến trình còn đang chạy trong bất kỳ container nào của Pod.
      Kubelet cũng dọn dẹp một container `pause` ẩn nếu container runtime đó có dùng.
   1. Kubelet chuyển Pod sang một phase kết thúc (`Failed` hoặc `Succeeded` tùy theo
      trạng thái cuối của các container của nó).
   1. Kubelet kích hoạt việc loại bỏ cưỡng bức đối tượng Pod khỏi API server, bằng cách đặt thời gian ân hạn
      về 0 (xóa ngay lập tức).
   1. API server xóa đối tượng API của Pod, từ đó Pod không còn hiển thị với bất kỳ client nào.

### Buộc kết thúc Pod (Forced Pod termination) {#pod-termination-forced}

> **Thận trọng:**
>
> Việc buộc xóa có khả năng gây gián đoạn cho một số workload và các Pod của chúng.

Theo mặc định, mọi thao tác xóa đều diễn ra nhẹ nhàng trong vòng 30 giây. Lệnh `kubectl delete` hỗ trợ
tùy chọn `--grace-period=<giây>` cho phép bạn ghi đè giá trị mặc định và chỉ định
giá trị của riêng mình.

Đặt thời gian ân hạn là `0` sẽ buộc xóa Pod ngay lập tức khỏi API
server. Nếu Pod vẫn đang chạy trên một node, việc buộc xóa đó kích hoạt kubelet
bắt đầu dọn dẹp ngay lập tức.

Khi dùng kubectl, bạn phải chỉ định thêm cờ `--force` cùng với `--grace-period=0`
để thực hiện việc buộc xóa.

Khi một thao tác buộc xóa được thực hiện, API server không chờ xác nhận
từ kubelet rằng Pod đã được kết thúc trên node mà nó đang chạy. Nó
loại bỏ Pod trong API ngay lập tức để một Pod mới có thể được tạo với cùng
tên. Trên node, các Pod được đặt để kết thúc ngay lập tức vẫn sẽ được cho
một khoảng ân hạn nhỏ trước khi bị giết cưỡng bức.

> **Thận trọng:**
>
> Xóa ngay lập tức không chờ xác nhận rằng tài nguyên đang chạy đã được kết thúc.
> Tài nguyên đó có thể tiếp tục chạy trên cluster vô thời hạn.

Nếu bạn cần buộc xóa các Pod thuộc một StatefulSet, hãy tham khảo tài liệu
tác vụ về
[xóa Pod khỏi một StatefulSet](341-force-delete-stateful-set-pod-vi.md).

### Tắt Pod và sidecar container (Pod shutdown and sidecar containers) {#termination-with-sidecars}

Nếu Pod của bạn bao gồm một hoặc nhiều
[sidecar container](51-sidecar-containers-vi.md)
(các init container với chính sách khởi động lại `Always`), kubelet sẽ trì hoãn việc gửi
tín hiệu TERM đến các sidecar container này cho đến khi container chính cuối cùng đã kết thúc hoàn toàn.
Các sidecar container sẽ được kết thúc theo thứ tự ngược với thứ tự chúng được định nghĩa trong Pod spec.
Điều này bảo đảm các sidecar container tiếp tục phục vụ các container khác trong Pod cho đến khi
chúng không còn cần thiết.

Điều này có nghĩa là việc kết thúc chậm của một container chính cũng sẽ trì hoãn việc kết thúc của các sidecar container.
Nếu thời gian ân hạn hết trước khi quá trình kết thúc hoàn tất, Pod có thể rơi vào [kết thúc cưỡng bức](#pod-termination-forced).
Trong trường hợp đó, tất cả các container còn lại trong Pod sẽ bị kết thúc đồng thời với một khoảng ân hạn ngắn.

Tương tự, nếu Pod có một hook `preStop` vượt quá thời gian ân hạn kết thúc, việc kết thúc khẩn cấp có thể xảy ra.
Nói chung, nếu trước đây bạn đã dùng các hook `preStop` để kiểm soát thứ tự kết thúc mà không có sidecar container, giờ bạn có thể
gỡ bỏ chúng và để kubelet tự động quản lý việc kết thúc sidecar.

### Thu gom rác cho Pod (Garbage collection of Pods) {#pod-garbage-collection}

Với các Pod đã thất bại, các đối tượng API vẫn còn trong API của cluster cho đến khi con người hoặc
một tiến trình controller xóa chúng một cách tường minh.

Bộ thu gom rác cho Pod (Pod garbage collector — PodGC), là một controller trong control plane, dọn dẹp
các Pod đã kết thúc (có phase `Succeeded` hoặc `Failed`), khi số lượng Pod vượt quá
ngưỡng đã cấu hình (được xác định bởi `terminated-pod-gc-threshold` trong kube-controller-manager).
Điều này tránh rò rỉ tài nguyên khi các Pod được tạo và kết thúc theo thời gian.

Ngoài ra, PodGC dọn dẹp mọi Pod thỏa mãn bất kỳ điều kiện nào sau đây:

1. là Pod mồ côi (orphan) — gắn với một node không còn tồn tại,
1. là Pod đang kết thúc mà chưa được lập lịch,
1. là Pod đang kết thúc, gắn với một node không sẵn sàng (non-ready) bị đánh taint
   [`node.kubernetes.io/out-of-service`](https://kubernetes.io/docs/reference/labels-annotations-taints/#node-kubernetes-io-out-of-service).

Cùng với việc dọn dẹp các Pod, PodGC cũng đánh dấu chúng là thất bại nếu chúng đang ở một
phase chưa kết thúc. Ngoài ra, PodGC thêm một Pod disruption condition khi dọn dẹp một Pod mồ côi.
Xem [Pod disruption condition](53-disruptions-vi.md#pod-disruption-conditions)
để biết thêm chi tiết.

## Hành vi của Pod khi kubelet khởi động lại (Pod behavior during kubelet restarts) {#kubelet-restarts}

Nếu bạn khởi động lại kubelet, các Pod (và các container của chúng) vẫn tiếp tục chạy
ngay cả trong lúc khởi động lại.
Khi có các Pod đang chạy trên một node, việc dừng hoặc khởi động lại kubelet
trên node đó **không** khiến kubelet dừng tất cả các Pod cục bộ
trước khi bản thân kubelet dừng.
Để dừng các Pod trên một node, bạn có thể dùng `kubectl drain`.

### Phát hiện kubelet khởi động lại (Detection of kubelet restarts) {#detection-of-kubelet-restarts}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [deprecated]`

Khi kubelet khởi động, nó kiểm tra xem liệu đã có một Node với các Pod đã được gắn hay chưa.
Nếu [condition `Ready`](https://kubernetes.io/docs/reference/node/node-status/#condition) của Node không thay đổi,
nói cách khác condition chưa từng chuyển từ true sang false, Kubernetes phát hiện đây là một _kubelet restart_ (kubelet khởi động lại).
(Có thể khởi động lại kubelet theo những cách khác, ví dụ để sửa một lỗi của node,
nhưng trong các trường hợp này, Kubernetes chọn phương án an toàn và xử lý như thể bạn
đã dừng kubelet rồi sau đó khởi động nó lại).

Khi kubelet khởi động lại, trạng thái các container được quản lý khác nhau tùy theo thiết lập feature gate:

* Theo mặc định, kubelet không thay đổi trạng thái container sau khi khởi động lại.
  Các container đang ở trạng thái `ready: true` vẫn giữ nguyên trạng thái sẵn sàng.

  Nếu bạn dừng kubelet đủ lâu để nó trượt một loạt các lần kiểm tra
  [nhịp tim của node (node heartbeat)](35-leases-vi.md#node-heart-beats),
  rồi bạn chờ một lúc trước khi khởi động kubelet trở lại, Kubernetes có thể bắt đầu trục xuất các Pod khỏi Node đó.
  Tuy nhiên, ngay cả khi việc trục xuất Pod bắt đầu diễn ra, Kubernetes không đánh dấu
  từng container trong các Pod đó là `ready: false`. Việc trục xuất ở cấp Pod
  xảy ra sau khi control plane đánh taint node là `node.kubernetes.io/not-ready` (do các lần heartbeat thất bại).

* Trong Kubernetes v1.36, bạn có thể chọn dùng (opt in) hành vi cũ (legacy) trong đó kubelet luôn thay đổi
  giá trị `ready` của các container thành false sau khi kubelet khởi động lại.

  Hành vi cũ này từng là mặc định trong một thời gian dài, nhưng gây ra vấn đề cho người dùng Kubernetes,
  đặc biệt trong các triển khai quy mô lớn. Mặc dù feature gate cho phép quay về hành vi cũ
  này một cách tạm thời, dự án Kubernetes khuyến nghị bạn gửi báo cáo lỗi (bug report) nếu gặp vấn đề.
  [Feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#ChangeContainerStatusOnKubeletRestart)
  `ChangeContainerStatusOnKubeletRestart`
  sẽ bị loại bỏ trong tương lai.

## Tiếp theo (What's next)

* Thực hành trực tiếp
  [gắn các handler vào sự kiện vòng đời của container](272-attach-handler-lifecycle-event-vi.md).

* Thực hành trực tiếp
  [cấu hình Liveness, Readiness và Startup Probe](274-configure-probes-vi.md).

* Tìm hiểu thêm về [container lifecycle hooks](42-container-lifecycle-hooks-vi.md).

* Tìm hiểu thêm về [sidecar container](51-sidecar-containers-vi.md).

* Để biết thông tin chi tiết về trạng thái của Pod và container trong API, xem
  tài liệu tham chiếu API về
  [`status`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#PodStatus) của Pod.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. `kubectl get pod` hiện `STATUS: CrashLoopBackOff`. Lúc đó `phase` của Pod là gì?
2. Một Pod đặt `restartPolicy: OnFailure` và có một sidecar. Sidecar thoát với mã 0. Nó có được
   khởi động lại không? Còn app container thoát với mã 0 thì sao?
3. Bạn `kubectl delete pod` một Pod đang chạy trên `lab-k8s-worker1`, và tiến trình trong container
   lờ hẳn tín hiệu TERM. Kể lại chuỗi việc kubelet làm, kèm mốc thời gian mặc định.
4. Hook `preStop` của bạn cần 45 giây, còn `terminationGracePeriodSeconds` để mặc định. Chuyện gì
   xảy ra, và bài bảo phải làm gì?
5. `lab-k8s-worker2` mất kết nối với phần còn lại của cluster. Control plane làm gì với các Pod trên
   node đó? Khi worker2 quay lại, chính Pod cũ có chạy tiếp không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **`Running`.** Đây là chỗ dễ nhầm nhất của bài: `CrashLoopBackOff` **không phải một phase**,
   nó là giá trị của **trường Status hiển thị của kubectl** cho người dùng dễ hình dung. Pod đã
   được gắn với một node và tất cả container đã được tạo, có container đang trong quá trình khởi
   động lại — đúng định nghĩa của phase `Running`. Phase là một phần tường minh của mô hình dữ
   liệu Kubernetes; trường Status của kubectl thì không.
2. **Sidecar vẫn được khởi động lại; app container thì không.** Sidecar **bỏ qua `restartPolicy`
   cấp Pod**: theo định nghĩa của Kubernetes, sidecar là một mục trong `initContainers` có
   `restartPolicy` **cấp container** đặt là `Always`, nên bảng so sánh trong bài ghi "Luôn khởi
   động lại" cho cả mã thoát 0 lẫn mã khác 0. App container thì tuân theo `OnFailure` của Pod:
   **mã thoát 0 là thành công nên không khởi động lại**.
3. Ngay khi kubelet thấy Pod đã bị đánh dấu đang kết thúc: (a) nếu container có hook `preStop` và
   `terminationGracePeriodSeconds` khác 0, **kubelet chạy `preStop` bên trong container**; (b)
   kubelet kích hoạt container runtime **gửi TERM tới tiến trình 1** của mỗi container — hoặc
   `STOPSIGNAL` nếu image định nghĩa; (c) tiến trình lờ TERM nên nó sống tới khi **hết thời gian
   ân hạn, mặc định 30 giây**; (d) kubelet kích hoạt tắt cưỡng bức, runtime gửi **`SIGKILL`** cho
   mọi tiến trình còn lại và kubelet dọn cả container `pause` ẩn; (e) kubelet chuyển Pod sang
   phase kết thúc (`Failed` hoặc `Succeeded`) rồi **buộc xóa đối tượng Pod khỏi API server** bằng
   cách đặt ân hạn về 0. Song song đó, control plane đã đánh giá loại Pod khỏi các EndpointSlice.
4. `preStop` chưa xong khi hết 30 giây ân hạn, và kubelet chỉ cho **một khoảng gia hạn nhỏ, một
   lần duy nhất, là 2 giây** — sau đó là tắt cưỡng bức, hook bị cắt ngang. Bài nói rõ: nếu
   `preStop` cần nhiều thời gian hơn mức ân hạn mặc định cho phép thì bạn **phải điều chỉnh
   `terminationGracePeriodSeconds`** cho phù hợp. Đừng trông vào 2 giây gia hạn đó.
5. Control plane **áp một chính sách đặt `phase` của tất cả Pod trên node bị mất thành `Failed`**,
   và các Pod đó được đánh dấu để xóa sau một khoảng chờ. Khi worker2 quay lại, **Pod cũ không
   chạy tiếp**: một Pod cho trước, được xác định bởi **UID**, không bao giờ được lập lịch lại sang
   chỗ khác hay hồi sinh; nó chỉ có thể được **thay bằng một Pod mới gần như giống hệt**, có thể
   trùng `.metadata.name` nhưng **khác `.metadata.uid`** — và Kubernetes không bảo đảm bản thay
   thế lên đúng node cũ.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

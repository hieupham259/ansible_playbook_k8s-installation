# Xóa cưỡng bức Pod của StatefulSet (Force Delete StatefulSet Pods)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/run-application/force-delete-stateful-set-pod/>
>
> Trang này chỉ cho bạn cách xóa các Pod thuộc một StatefulSet, và giải thích những điều cần
> cân nhắc khi thực hiện việc đó.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 4 — Workload controller](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller)
→ [4b. StatefulSet, DaemonSet, Job và autoscaling](00-ALO-TRINH-ADMIN.md#4b-statefulset-daemonset-job-và-autoscaling),
bài 1/7 của dòng **Thực hành** · Kiểm chứng ở
[Lab 4b — StatefulSet, DaemonSet và Job](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md),
**phần B3.2**.

Đây là bài **cảnh báo**, không phải bài hướng dẫn thao tác hằng ngày. Nó nói ngay từ đầu rằng
trong hoạt động bình thường **không bao giờ** có nhu cầu xóa cưỡng bức Pod của StatefulSet. Đọc
nó như một danh sách rủi ro cần biết trước khi gõ lệnh, và nhớ rằng lab chỉ chạy thao tác này
trên StatefulSet không có dữ liệu của riêng lab.

**Phải hiểu ở lần đọc này:**

- StatefulSet controller chịu trách nhiệm tạo, scale và xóa thành viên, và bảo đảm **ngữ nghĩa
  nhiều nhất một**: tại bất kỳ thời điểm nào chỉ có **một** Pod mang một định danh cho trước đang
  chạy trong cluster (mục *Những điều cần cân nhắc về StatefulSet*).
- Xóa êm thấm bằng `kubectl delete pods <pod>` là an toàn, với điều kiện Pod **không** đặt
  `pod.Spec.TerminationGracePeriodSeconds` bằng 0; kubelet chỉ gỡ tên Pod khỏi apiserver **sau
  khi** Pod đã tắt êm (mục *Xóa Pod*).
- Pod trên node không truy cập được **không tự bị xóa**: nó chuyển sang `Terminating` hoặc
  `Unknown`. Bài liệt kê đúng ba đường để nó rời khỏi apiserver — xóa object Node, kubelet phản
  hồi trở lại rồi tự kết thúc Pod, hoặc người dùng xóa cưỡng bức — và khuyến nghị hai cách đầu.
- Xóa cưỡng bức `--grace-period=0 --force` **không chờ kubelet xác nhận** và **giải phóng tên Pod
  ngay lập tức**. Controller lập tức tạo Pod thay thế mang đúng định danh đó, trong khi Pod cũ có
  thể vẫn đang chạy — đây chính là cách ngữ nghĩa nhiều nhất một bị phá vỡ, dẫn tới chia não
  (split brain) và mất dữ liệu ở hệ dựa trên quorum (mục *Xóa cưỡng bức*).
- Khi xóa cưỡng bức, bạn đang **khẳng định** rằng Pod đó sẽ không bao giờ liên lạc lại với các Pod
  khác trong StatefulSet. Nếu Pod vẫn kẹt `Unknown` sau lệnh đó, đường cuối là gỡ finalizer bằng
  `kubectl patch pod <pod> -p '{"metadata":{"finalizers":null}}'`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Cơ chế đứng sau trạng thái `Terminating`/`Unknown`: timeout của node condition và Node Controller | ở đây chỉ cần biết hệ quả — Pod không tự biến mất khi node mất liên lạc | bài [23](23-nodes-vi.md#node-controller) đã đọc ở [1a. Kiến trúc và mô hình điều khiển](00-ALO-TRINH-ADMIN.md#1a-kiến-trúc-và-mô-hình-điều-khiển) |
| Chỉ dẫn dành cho kubectl phiên bản `<= 1.4` (bỏ cờ `--force`) | không liên quan tới phiên bản đang khóa của cluster lab | [bảng A1.3 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) |
| Link *gỡ lỗi một StatefulSet* (bài [302](302-debug-statefulset-vi.md)) ở mục *Tiếp theo* | chẩn đoán workload hỏng là một mạch riêng, cần công cụ chưa học | [giai đoạn 24 — Xử lý sự cố](00-ALO-TRINH-ADMIN.md#giai-đoạn-24--xử-lý-sự-cố) |

---

## Trước khi bạn bắt đầu (Before you begin)

- Đây là một tác vụ khá nâng cao và có khả năng vi phạm một số thuộc tính vốn có của StatefulSet.
- Trước khi tiếp tục, hãy tự làm quen với những điều cần cân nhắc được liệt kê bên dưới.

## Những điều cần cân nhắc về StatefulSet (StatefulSet considerations)

Trong hoạt động bình thường của một StatefulSet, **không bao giờ** có nhu cầu xóa cưỡng bức
(force delete) một Pod của StatefulSet.
[StatefulSet controller](65-statefulset-vi.md)
chịu trách nhiệm tạo, scale và xóa các thành viên của StatefulSet. Nó cố gắng bảo đảm rằng số
lượng Pod được chỉ định, từ ordinal 0 đến N-1, luôn sống và sẵn sàng. StatefulSet bảo đảm rằng,
tại bất kỳ thời điểm nào, có nhiều nhất một Pod với một định danh (identity) cho trước đang chạy
trong cluster. Điều này được gọi là ngữ nghĩa *nhiều nhất một* (at most one) mà StatefulSet
cung cấp.

Việc xóa cưỡng bức thủ công cần được thực hiện một cách thận trọng, vì nó có khả năng vi phạm
ngữ nghĩa "nhiều nhất một" vốn có của StatefulSet. StatefulSet có thể được dùng để chạy các ứng
dụng phân tán và dạng cluster, vốn cần định danh mạng ổn định và lưu trữ ổn định. Những ứng dụng
này thường có cấu hình dựa trên một tập hợp (ensemble) gồm số lượng thành viên cố định với các
định danh cố định. Việc có nhiều thành viên mang cùng một định danh có thể gây hậu quả thảm khốc
và có thể dẫn đến mất dữ liệu (ví dụ: kịch bản chia não — split brain — trong các hệ thống dựa
trên quorum).

## Xóa Pod (Delete Pods)

Bạn có thể thực hiện xóa Pod một cách êm thấm (graceful) bằng lệnh sau:

```shell
kubectl delete pods <pod>
```

Để lệnh trên dẫn đến việc kết thúc êm thấm, Pod **không được** khai báo
`pod.Spec.TerminationGracePeriodSeconds` bằng 0. Việc đặt
`pod.Spec.TerminationGracePeriodSeconds` bằng 0 giây là không an toàn và hết sức không nên làm
đối với các Pod của StatefulSet. Xóa êm thấm là an toàn và sẽ bảo đảm Pod
[tắt một cách êm thấm](47-pod-lifecycle-vi.md#pod-termination)
trước khi kubelet xóa tên của Pod đó khỏi apiserver.

Một Pod không bị xóa tự động khi node trở nên không thể truy cập được (unreachable).
Các Pod đang chạy trên một Node không thể truy cập được sẽ chuyển sang trạng thái 'Terminating'
hoặc 'Unknown' sau một khoảng
[thời gian chờ (timeout)](https://kubernetes.io/docs/concepts/architecture/nodes#condition).
Pod cũng có thể rơi vào các trạng thái này khi người dùng cố gắng xóa êm thấm một Pod
trên một Node không thể truy cập được.
Các cách duy nhất để một Pod ở trạng thái như vậy bị gỡ khỏi apiserver là:

- Object Node bị xóa (bởi chính bạn, hoặc bởi
  [Node Controller](23-nodes-vi.md#node-controller)).
- kubelet trên Node không phản hồi bắt đầu phản hồi trở lại, kết thúc (kill) Pod và gỡ mục
  tương ứng khỏi apiserver.
- Người dùng xóa cưỡng bức Pod.

Thực hành tốt nhất được khuyến nghị là dùng cách thứ nhất hoặc thứ hai. Nếu một Node được xác
nhận là đã chết (ví dụ: bị ngắt kết nối vĩnh viễn khỏi mạng, đã tắt nguồn, v.v.), thì hãy xóa
object Node. Nếu Node đang gặp tình trạng phân mảnh mạng (network partition), hãy cố gắng khắc
phục hoặc chờ tình trạng đó tự khắc phục. Khi phân mảnh mạng được hồi phục, kubelet sẽ hoàn tất
việc xóa Pod và giải phóng tên của Pod đó trong apiserver.

Thông thường, hệ thống hoàn tất việc xóa khi Pod không còn chạy trên một Node nữa, hoặc khi
Node bị quản trị viên xóa. Bạn có thể ghi đè điều này bằng cách xóa cưỡng bức Pod.

### Xóa cưỡng bức (Force Deletion)

Xóa cưỡng bức **không** chờ xác nhận từ kubelet rằng Pod đã bị kết thúc.
Bất kể việc xóa cưỡng bức có thành công trong việc kết thúc Pod hay không, nó sẽ ngay lập tức
giải phóng tên của Pod khỏi apiserver. Điều này cho phép StatefulSet controller tạo một Pod
thay thế với chính định danh đó; việc này có thể dẫn đến sự tồn tại trùng lặp của một Pod vẫn
đang chạy, và nếu Pod đó vẫn có thể giao tiếp với các thành viên khác của StatefulSet, nó sẽ
vi phạm ngữ nghĩa "nhiều nhất một" mà StatefulSet được thiết kế để bảo đảm.

Khi bạn xóa cưỡng bức một Pod của StatefulSet, bạn đang khẳng định rằng Pod đó sẽ không bao giờ
liên lạc lại với các Pod khác trong StatefulSet nữa, và tên của nó có thể được giải phóng một
cách an toàn để tạo Pod thay thế.

Nếu bạn muốn xóa cưỡng bức một Pod bằng kubectl phiên bản >= 1.5, hãy chạy:

```shell
kubectl delete pods <pod> --grace-period=0 --force
```

Nếu bạn đang dùng bất kỳ phiên bản kubectl nào <= 1.4, bạn nên bỏ tùy chọn `--force` và dùng:

```shell
kubectl delete pods <pod> --grace-period=0
```

Nếu ngay cả sau các lệnh này Pod vẫn kẹt ở trạng thái `Unknown`, hãy dùng lệnh sau để gỡ Pod
khỏi cluster:

```shell
kubectl patch pod <pod> -p '{"metadata":{"finalizers":null}}'
```

Hãy luôn thực hiện việc xóa cưỡng bức Pod của StatefulSet một cách cẩn trọng và với hiểu biết
đầy đủ về những rủi ro liên quan.

## Tiếp theo (What's next)

Tìm hiểu thêm về
[gỡ lỗi một StatefulSet](302-debug-statefulset-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở nhóm 4b:

1. Ngữ nghĩa nhiều nhất một của StatefulSet phát biểu điều gì, và vì sao xóa cưỡng bức lại phá vỡ
   được nó? Bài nêu hậu quả cụ thể nào cho ứng dụng phân tán?
2. `lab-k8s-worker2` mất kết nối mạng, các Pod StatefulSet trên đó chuyển sang `Terminating` hoặc
   `Unknown`. Bài liệt kê ba đường để những Pod đó rời khỏi apiserver — ba đường nào, và bài
   khuyến nghị dùng đường nào?
3. **Câu bẫy.** Bài nói xóa êm thấm là an toàn. Vậy đặt `pod.Spec.TerminationGracePeriodSeconds`
   bằng 0 cho Pod StatefulSet để nó vừa êm vừa nhanh thì sao? Bài đánh giá cách đó thế nào?
4. Sau `kubectl delete pods <pod> --grace-period=0 --force`, Pod vẫn kẹt ở `Unknown`. Bài đưa ra
   lệnh cuối cùng nào, và lệnh đó gỡ cái gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Nó phát biểu rằng **tại bất kỳ thời điểm nào, nhiều nhất một Pod mang một định danh cho trước
   đang chạy trong cluster**. Xóa cưỡng bức phá vỡ nó vì lệnh **không chờ kubelet xác nhận Pod đã
   kết thúc** mà **giải phóng tên Pod khỏi apiserver ngay**; StatefulSet controller liền tạo một
   Pod thay thế **mang đúng định danh đó**, trong khi Pod cũ có thể vẫn đang chạy. Hậu quả bài nêu:
   ứng dụng phân tán cấu hình theo một tập thành viên cố định sẽ có **hai thành viên cùng định
   danh**, dẫn tới **chia não (split brain)** trong hệ dựa trên quorum và **mất dữ liệu**.
2. Ba đường: **object Node bị xóa** (bởi bạn hoặc bởi Node Controller); **kubelet trên node phản
   hồi trở lại**, tự kết thúc Pod và gỡ mục tương ứng khỏi apiserver; hoặc **người dùng xóa cưỡng
   bức Pod**. Bài khuyến nghị **hai cách đầu**: node chết hẳn thì xóa object Node, còn nếu chỉ là
   phân mảnh mạng thì khắc phục hoặc chờ nó tự hồi phục — khi mạng thông, kubelet sẽ hoàn tất việc
   xóa và giải phóng tên Pod.
3. **Rất không nên.** Đây là chỗ dễ nhầm vì lệnh vẫn là `kubectl delete pods` bình thường, trông
   như xóa êm thấm. Nhưng bài nói rõ: để lệnh đó **dẫn tới kết thúc êm thấm**, Pod **không được**
   đặt `TerminationGracePeriodSeconds` bằng 0; đặt bằng 0 giây là **không an toàn và hết sức không
   nên làm** với Pod của StatefulSet. Grace period bằng 0 biến việc xóa êm thành xóa cưỡng bức
   trên thực tế.
4. `kubectl patch pod <pod> -p '{"metadata":{"finalizers":null}}'`. Nó **gỡ finalizer** của Pod,
   để Pod được xóa khỏi cluster khi ngay cả lệnh xóa cưỡng bức cũng không đủ. Bài đặt lệnh này ở
   cuối cùng và nhắc lại rằng mọi thao tác xóa cưỡng bức Pod StatefulSet phải làm **cẩn trọng và
   với hiểu biết đầy đủ về rủi ro**.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

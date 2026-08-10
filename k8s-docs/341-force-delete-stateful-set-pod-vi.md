# Xóa cưỡng bức Pod của StatefulSet (Force Delete StatefulSet Pods)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/run-application/force-delete-stateful-set-pod/>
>
> Trang này chỉ cho bạn cách xóa các Pod thuộc một StatefulSet, và giải thích những điều cần
> cân nhắc khi thực hiện việc đó.

## Trước khi bạn bắt đầu (Before you begin)

- Đây là một tác vụ khá nâng cao và có khả năng vi phạm một số thuộc tính vốn có của StatefulSet.
- Trước khi tiếp tục, hãy tự làm quen với những điều cần cân nhắc được liệt kê bên dưới.

## Những điều cần cân nhắc về StatefulSet (StatefulSet considerations)

Trong hoạt động bình thường của một StatefulSet, **không bao giờ** có nhu cầu xóa cưỡng bức
(force delete) một Pod của StatefulSet.
[StatefulSet controller](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
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
[tắt một cách êm thấm](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination)
trước khi kubelet xóa tên của Pod đó khỏi apiserver.

Một Pod không bị xóa tự động khi node trở nên không thể truy cập được (unreachable).
Các Pod đang chạy trên một Node không thể truy cập được sẽ chuyển sang trạng thái 'Terminating'
hoặc 'Unknown' sau một khoảng
[thời gian chờ (timeout)](https://kubernetes.io/docs/concepts/architecture/nodes/#condition).
Pod cũng có thể rơi vào các trạng thái này khi người dùng cố gắng xóa êm thấm một Pod
trên một Node không thể truy cập được.
Các cách duy nhất để một Pod ở trạng thái như vậy bị gỡ khỏi apiserver là:

- Object Node bị xóa (bởi chính bạn, hoặc bởi
  [Node Controller](https://kubernetes.io/docs/concepts/architecture/nodes/#node-controller)).
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
[gỡ lỗi một StatefulSet](https://kubernetes.io/docs/tasks/debug/debug-application/debug-statefulset/).

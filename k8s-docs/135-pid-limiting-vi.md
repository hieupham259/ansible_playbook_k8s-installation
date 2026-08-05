# Giới hạn và dự trữ Process ID (Process ID Limits And Reservations)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/policy/pid-limiting/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.20 [stable]`

Kubernetes cho phép bạn giới hạn số lượng process ID (PID) mà một Pod có thể sử dụng.
Bạn cũng có thể dự trữ một số lượng PID có thể cấp phát (allocatable) cho mỗi node
để hệ điều hành và các daemon sử dụng (thay vì cho các Pod).

Process ID (PID) là một tài nguyên nền tảng trên các node. Rất dễ chạm đến
giới hạn số lượng task mà không hề chạm đến bất kỳ giới hạn tài nguyên nào khác,
và điều đó có thể gây mất ổn định cho máy chủ (host).

Quản trị viên cluster cần có các cơ chế để đảm bảo rằng các Pod chạy trong
cluster không thể gây cạn kiệt PID, khiến các daemon của host (chẳng hạn
kubelet hoặc kube-proxy, và có thể cả container runtime) không chạy được.
Ngoài ra, việc đảm bảo PID được giới hạn giữa các Pod cũng rất quan trọng
nhằm đảm bảo chúng chỉ có ảnh hưởng hạn chế đến các workload khác trên cùng node.

> **Ghi chú:**
> Trên một số bản cài đặt Linux, hệ điều hành đặt giới hạn PID ở một giá trị mặc định thấp,
> chẳng hạn `32768`. Hãy cân nhắc tăng giá trị của `/proc/sys/kernel/pid_max`.

Bạn có thể cấu hình kubelet để giới hạn số lượng PID mà một Pod nhất định được phép tiêu thụ.
Ví dụ, nếu hệ điều hành host của node được thiết lập dùng tối đa `262144` PID và
dự kiến chứa ít hơn `250` Pod, ta có thể cấp cho mỗi Pod một "ngân sách" `1000`
PID để tránh dùng cạn tổng số PID khả dụng của node đó. Nếu quản trị viên muốn
overcommit PID tương tự như với CPU hoặc bộ nhớ, họ cũng có thể làm vậy nhưng
kèm theo một số rủi ro bổ sung. Dù theo cách nào, một Pod đơn lẻ sẽ không thể
làm sập toàn bộ máy. Kiểu giới hạn tài nguyên này giúp ngăn các fork bomb đơn giản
ảnh hưởng đến hoạt động của cả cluster.

Giới hạn PID theo từng Pod cho phép quản trị viên bảo vệ Pod này khỏi Pod khác, nhưng
không đảm bảo rằng mọi Pod được lập lịch (schedule) lên host đó không thể tác động đến node nói chung.
Giới hạn theo từng Pod cũng không bảo vệ được chính các agent của node khỏi tình trạng cạn kiệt PID.

Bạn cũng có thể dự trữ một lượng PID cho phần chi phí vận hành (overhead) của node,
tách biệt với phần cấp phát cho các Pod. Điều này tương tự như cách bạn có thể dự trữ
CPU, bộ nhớ hoặc các tài nguyên khác cho hệ điều hành và các thành phần khác
nằm ngoài các Pod và các container của chúng.

Giới hạn PID là "người anh em" quan trọng của request và limit
[tài nguyên tính toán (compute resource)](./110-manage-resources-containers-vi.md).
Tuy nhiên, bạn chỉ định nó theo một cách khác: thay vì định nghĩa giới hạn tài nguyên
của Pod trong `.spec` của Pod, bạn cấu hình giới hạn này như một thiết lập trên kubelet.
Giới hạn PID định nghĩa ở cấp Pod hiện chưa được hỗ trợ.

> **Thận trọng:**
> Điều này có nghĩa là giới hạn áp dụng cho một Pod có thể khác nhau tùy theo
> nơi Pod được lập lịch. Để mọi thứ đơn giản, cách dễ nhất là tất cả các Node
> dùng cùng một mức giới hạn và dự trữ tài nguyên PID.

## Giới hạn PID của node (Node PID limits)

Kubernetes cho phép bạn dự trữ một số lượng process ID cho hệ thống sử dụng. Để
cấu hình mức dự trữ này, dùng tham số `pid=<number>` trong các tùy chọn
dòng lệnh `--system-reserved` và `--kube-reserved` của kubelet.
Giá trị bạn chỉ định khai báo rằng số lượng process ID đó sẽ được dự trữ
tương ứng cho toàn hệ thống và cho các daemon hệ thống của Kubernetes.

## Giới hạn PID của Pod (Pod PID limits)

Kubernetes cho phép bạn giới hạn số lượng process chạy trong một Pod. Bạn
chỉ định giới hạn này ở cấp node, thay vì cấu hình nó như một giới hạn tài nguyên
cho một Pod cụ thể. Mỗi Node có thể có một giới hạn PID khác nhau.
Để cấu hình giới hạn, bạn có thể chỉ định tham số dòng lệnh `--pod-max-pids`
cho kubelet, hoặc đặt `PodPidsLimit` trong
[file cấu hình](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/) của kubelet.

## Eviction dựa trên PID (PID based eviction)

Bạn có thể cấu hình kubelet để bắt đầu chấm dứt (terminate) một Pod khi Pod đó hoạt động
bất thường và tiêu thụ một lượng tài nguyên không bình thường.
Tính năng này được gọi là eviction (thu hồi). Bạn có thể
[Cấu hình xử lý khi hết tài nguyên (Configure Out of Resource Handling)](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)
cho nhiều tín hiệu eviction khác nhau.
Dùng tín hiệu eviction `pid.available` để cấu hình ngưỡng số lượng PID mà Pod sử dụng.
Bạn có thể đặt chính sách eviction mềm (soft) và cứng (hard).
Tuy nhiên, ngay cả với chính sách eviction cứng, nếu số lượng PID tăng rất nhanh,
node vẫn có thể rơi vào trạng thái mất ổn định do chạm đến giới hạn PID của node.
Giá trị của tín hiệu eviction được tính toán định kỳ và KHÔNG cưỡng chế giới hạn.

Giới hạn PID — theo từng Pod và theo từng Node — đặt ra giới hạn cứng.
Một khi chạm đến giới hạn, workload sẽ bắt đầu gặp lỗi khi cố lấy một PID mới.
Điều đó có thể dẫn đến hoặc không dẫn đến việc Pod bị lập lịch lại,
tùy vào cách workload phản ứng với các lỗi này và cách các probe liveness và readiness
của Pod được cấu hình. Tuy nhiên, nếu các giới hạn được đặt đúng,
bạn có thể đảm bảo rằng workload của các Pod khác và các process hệ thống
sẽ không bị cạn PID khi có một Pod hoạt động bất thường.

## Tiếp theo (What's next)

- Tham khảo [tài liệu enhancement về PID Limiting](https://github.com/kubernetes/enhancements/blob/097b4d8276bc9564e56adf72505d43ce9bc5e9e8/keps/sig-node/20190129-pid-limiting.md) để biết thêm thông tin.
- Về bối cảnh lịch sử, đọc
  [Process ID Limiting for Stability Improvements in Kubernetes 1.14](https://kubernetes.io/blog/2019/04/15/process-id-limiting-for-stability-improvements-in-kubernetes-1.14/).
- Đọc [Quản lý tài nguyên cho container (Managing Resources for Containers)](./110-manage-resources-containers-vi.md).
- Tìm hiểu cách [Cấu hình xử lý khi hết tài nguyên (Configure Out of Resource Handling)](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/).

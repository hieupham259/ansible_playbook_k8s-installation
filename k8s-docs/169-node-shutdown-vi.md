# Tắt node (Node Shutdowns)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/concepts/cluster-administration/node-shutdown/

Trong một cluster Kubernetes, một node có thể được tắt theo cách nhẹ nhàng có kế hoạch, hoặc bất ngờ vì những lý do như mất điện hay một nguyên nhân bên ngoài nào khác. Việc tắt node có thể dẫn đến hỏng workload nếu node không được drain trước khi tắt. Việc tắt node có thể là **nhẹ nhàng (graceful)** hoặc **không nhẹ nhàng (non-graceful)**.

> **Thận trọng:** Gói `unattended-upgrades` của Debian, trong cấu hình thông thường của nó, xung đột với tính năng tắt node nhẹ nhàng.
> Nếu bạn dùng cấu hình mặc định của `unattended-upgrades`, vốn tùy chỉnh thời gian gia hạn (grace period) khi tắt máy chủ, thì kubelet không lấy được lock cần thiết để xử lý các sự kiện tắt máy một cách đúng đắn.
>
> Điều này xảy ra nếu giá trị `shutdownGracePeriod` lớn hơn 30 giây.
> Để tránh vấn đề này, bạn có thể vô hiệu hóa một phần cấu hình của `unattended-upgrades`, bằng cách biến `/etc/systemd/logind.conf.d/unattended-upgrades-logind-maxdelay.conf` thành một liên kết tượng trưng (symbolic link) trỏ tới `/dev/null`.
>
> Để biết thêm chi tiết, tham khảo [tài liệu `logind.conf`](https://www.freedesktop.org/software/systemd/man/latest/logind.conf.html).

## Tắt node nhẹ nhàng (Graceful node shutdown) {#graceful-node-shutdown}

kubelet cố gắng phát hiện việc hệ thống của node tắt máy và chấm dứt (terminate) các pod đang chạy trên node đó.

Kubelet bảo đảm rằng các pod tuân theo [quy trình chấm dứt pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination) thông thường trong quá trình tắt node. Trong khi node đang tắt, kubelet không chấp nhận các Pod mới (ngay cả khi những Pod đó đã được gán (bind) vào node).

### Bật tính năng tắt node nhẹ nhàng (Enabling graceful node shutdown)

#### Linux

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.21 [beta]`

Trên Linux, tính năng tắt node nhẹ nhàng được điều khiển bằng [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) `GracefulNodeShutdown`, được bật mặc định từ phiên bản 1.21.

> **Ghi chú:** Tính năng tắt node nhẹ nhàng phụ thuộc vào systemd vì nó tận dụng [systemd inhibitor locks](https://www.freedesktop.org/wiki/Software/systemd/inhibit/) để trì hoãn việc tắt node trong một khoảng thời gian nhất định.

#### Windows

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.34 [beta]`

Trên Windows, tính năng tắt node nhẹ nhàng được điều khiển bằng [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) `WindowsGracefulNodeShutdown`, được giới thiệu trong phiên bản 1.32 dưới dạng tính năng alpha. Trong Kubernetes 1.34, tính năng này ở trạng thái Beta và được bật mặc định.

> **Ghi chú:** Tính năng tắt node nhẹ nhàng trên Windows phụ thuộc vào việc kubelet chạy như một Windows service; khi đó kubelet sẽ có một [service control handler](https://learn.microsoft.com/en-us/windows/win32/services/service-control-handler-function) đã được đăng ký để trì hoãn sự kiện preshutdown trong một khoảng thời gian nhất định.

Việc tắt node nhẹ nhàng trên Windows không thể bị hủy.

Nếu kubelet không chạy như một Windows service, nó sẽ không thể đặt và theo dõi sự kiện [Preshutdown](https://learn.microsoft.com/en-us/windows/win32/api/winsvc/ns-winsvc-service_preshutdown_info); khi đó node sẽ phải trải qua quy trình [Tắt node không nhẹ nhàng](#non-graceful-node-shutdown) đã đề cập ở trên.

Trong trường hợp tính năng tắt node nhẹ nhàng trên Windows được bật nhưng kubelet không chạy như một Windows service, kubelet sẽ tiếp tục chạy thay vì thất bại. Tuy nhiên, nó sẽ ghi một log lỗi cho biết rằng nó cần được chạy như một Windows service.

### Cấu hình tắt node nhẹ nhàng (Configuring graceful node shutdown)

Lưu ý rằng theo mặc định, cả hai tùy chọn cấu hình được mô tả bên dưới, `shutdownGracePeriod` và `shutdownGracePeriodCriticalPods`, đều được đặt bằng 0, do đó không kích hoạt chức năng tắt node nhẹ nhàng. Để kích hoạt tính năng này, cả hai tùy chọn phải được cấu hình một cách phù hợp và đặt giá trị khác 0.

Khi kubelet được thông báo về việc node sắp tắt, nó đặt condition `NotReady` trên Node, với `reason` được đặt là `"node is shutting down"`. kube-scheduler tôn trọng condition này và không lập lịch (schedule) Pod nào lên node bị ảnh hưởng; các scheduler bên thứ ba khác cũng được kỳ vọng tuân theo cùng logic đó. Điều này có nghĩa là các Pod mới sẽ không được lập lịch lên node đó và do đó sẽ không có Pod nào khởi động.

kubelet **cũng** từ chối các Pod trong giai đoạn `PodAdmission` nếu phát hiện quá trình tắt node đang diễn ra, vì vậy ngay cả các Pod có toleration cho `node.kubernetes.io/not-ready:NoSchedule` cũng không khởi động ở đó.

Khi kubelet đang đặt condition đó trên Node của nó thông qua API, kubelet cũng bắt đầu chấm dứt mọi Pod đang chạy cục bộ.

Trong quá trình tắt nhẹ nhàng, kubelet chấm dứt các pod theo hai giai đoạn:

1. Chấm dứt các pod thông thường đang chạy trên node.
1. Chấm dứt các [pod quan trọng (critical pod)](https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/#marking-pod-as-critical) đang chạy trên node.

Tính năng tắt node nhẹ nhàng được cấu hình bằng hai tùy chọn của [`KubeletConfiguration`](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/):

- `shutdownGracePeriod`:

  Chỉ định tổng khoảng thời gian mà node nên trì hoãn việc tắt máy. Đây là tổng thời gian gia hạn cho việc chấm dứt pod, tính cho cả pod thông thường lẫn [pod quan trọng](https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/#marking-pod-as-critical).

- `shutdownGracePeriodCriticalPods`:

  Chỉ định khoảng thời gian được dùng để chấm dứt các [pod quan trọng](https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/#marking-pod-as-critical) trong quá trình tắt node. Giá trị này nên nhỏ hơn `shutdownGracePeriod`.

> **Ghi chú:** Có những trường hợp việc chấm dứt Node bị hệ thống hủy bỏ (hoặc có thể được quản trị viên hủy thủ công). Trong cả hai tình huống đó, Node sẽ trở lại trạng thái `Ready`. Tuy nhiên, những Pod đã bắt đầu quá trình chấm dứt sẽ không được kubelet khôi phục và sẽ cần được lập lịch lại.

Ví dụ, nếu `shutdownGracePeriod=30s` và `shutdownGracePeriodCriticalPods=10s`, kubelet sẽ trì hoãn việc tắt node thêm 30 giây. Trong quá trình tắt, 20 giây đầu tiên (30-10) sẽ được dành để chấm dứt nhẹ nhàng các pod thông thường, và 10 giây cuối sẽ được dành để chấm dứt các [pod quan trọng](https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/#marking-pod-as-critical).

> **Ghi chú:** Khi các pod bị trục xuất (evict) trong quá trình tắt node nhẹ nhàng, chúng được đánh dấu là đã tắt (shutdown). Chạy `kubectl get pods` sẽ hiển thị trạng thái của các pod bị trục xuất là `Terminated`. Và `kubectl describe pod` cho biết pod đã bị trục xuất vì node tắt máy:
>
> ```
> Reason:         Terminated
> Message:        Pod was terminated in response to imminent node shutdown.
> ```

### Tắt node nhẹ nhàng dựa trên độ ưu tiên của Pod (Pod Priority based graceful node shutdown) {#pod-priority-graceful-node-shutdown}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.24 [beta]`

Để mang lại nhiều linh hoạt hơn cho việc sắp thứ tự các pod trong quá trình tắt node nhẹ nhàng, tính năng tắt node nhẹ nhàng tôn trọng PriorityClass của các Pod, với điều kiện bạn đã bật tính năng này trong cluster của mình. Tính năng cho phép quản trị viên cluster định nghĩa tường minh thứ tự các pod trong quá trình tắt node nhẹ nhàng dựa trên các [priority class](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#priorityclass).

Tính năng [Tắt node nhẹ nhàng](#graceful-node-shutdown), như mô tả ở trên, tắt các pod theo hai giai đoạn: các pod không quan trọng trước, sau đó đến các pod quan trọng. Nếu cần thêm sự linh hoạt để định nghĩa tường minh thứ tự các pod trong quá trình tắt máy một cách chi tiết hơn, có thể dùng tính năng tắt nhẹ nhàng dựa trên độ ưu tiên của pod.

Khi tính năng tắt node nhẹ nhàng tôn trọng độ ưu tiên của pod, điều này giúp việc tắt node nhẹ nhàng có thể được thực hiện theo nhiều giai đoạn, mỗi giai đoạn tắt một lớp độ ưu tiên (priority class) cụ thể của các pod. kubelet có thể được cấu hình với chính xác các giai đoạn và thời gian tắt cho từng giai đoạn.

Giả sử có các [priority class](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#priorityclass) tùy chỉnh sau cho pod trong một cluster,

| Tên priority class của Pod | Giá trị priority class của Pod |
| -------------------------- | ------------------------------ |
| `custom-class-a`           | 100000                         |
| `custom-class-b`           | 10000                          |
| `custom-class-c`           | 1000                           |
| `regular/unset`            | 0                              |

Trong [cấu hình kubelet](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/), các thiết lập cho `shutdownGracePeriodByPodPriority` có thể trông như sau:

| Giá trị priority class của Pod | Thời gian tắt |
| ------------------------------ | ------------- |
| 100000                         | 10 giây       |
| 10000                          | 180 giây      |
| 1000                           | 120 giây      |
| 0                              | 60 giây       |

Cấu hình YAML tương ứng trong kubelet config sẽ là:

```yaml
shutdownGracePeriodByPodPriority:
  - priority: 100000
    shutdownGracePeriodSeconds: 10
  - priority: 10000
    shutdownGracePeriodSeconds: 180
  - priority: 1000
    shutdownGracePeriodSeconds: 120
  - priority: 0
    shutdownGracePeriodSeconds: 60
```

Bảng trên ngụ ý rằng bất kỳ pod nào có giá trị `priority` >= 100000 sẽ chỉ có 10 giây để tắt, bất kỳ pod nào có giá trị >= 10000 và < 100000 sẽ có 180 giây để tắt, bất kỳ pod nào có giá trị >= 1000 và < 10000 sẽ có 120 giây để tắt. Cuối cùng, tất cả các pod còn lại sẽ có 60 giây để tắt.

Bạn không nhất thiết phải chỉ định giá trị tương ứng cho tất cả các lớp. Ví dụ, bạn có thể dùng các thiết lập sau thay thế:

| Giá trị priority class của Pod | Thời gian tắt |
| ------------------------------ | ------------- |
| 100000                         | 300 giây      |
| 1000                           | 120 giây      |
| 0                              | 60 giây       |

Trong trường hợp trên, các pod với `custom-class-b` sẽ rơi vào cùng nhóm với `custom-class-c` khi tắt.

Nếu không có pod nào trong một khoảng cụ thể, thì kubelet không chờ các pod trong khoảng độ ưu tiên đó. Thay vào đó, kubelet lập tức chuyển sang khoảng giá trị priority class kế tiếp.

Nếu tính năng này được bật nhưng không có cấu hình nào được cung cấp, thì sẽ không có hành động sắp thứ tự nào được thực hiện.

Việc sử dụng tính năng này yêu cầu bật [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) `GracefulNodeShutdownBasedOnPodPriority`, và đặt `ShutdownGracePeriodByPodPriority` trong [kubelet config](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/) thành cấu hình mong muốn chứa các giá trị priority class của pod và thời gian tắt tương ứng của chúng.

> **Ghi chú:** Khả năng tính đến độ ưu tiên của Pod trong quá trình tắt node nhẹ nhàng được giới thiệu dưới dạng tính năng Alpha trong Kubernetes v1.23. Trong Kubernetes v1.36, tính năng này ở trạng thái Beta và được bật mặc định.

Các metric `graceful_shutdown_start_time_seconds` và `graceful_shutdown_end_time_seconds` được phát ra dưới subsystem kubelet để giám sát các lần tắt node.

## Xử lý tắt node không nhẹ nhàng (Non-graceful node shutdown handling) {#non-graceful-node-shutdown}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.28 [stable]`

Một hành động tắt node có thể không được Node Shutdown Manager của kubelet phát hiện, hoặc vì lệnh tắt không kích hoạt cơ chế inhibitor locks mà kubelet sử dụng, hoặc vì lỗi của người dùng, tức là ShutdownGracePeriod và ShutdownGracePeriodCriticalPods không được cấu hình đúng. Vui lòng tham khảo phần [Tắt node nhẹ nhàng](#graceful-node-shutdown) ở trên để biết thêm chi tiết.

Khi một node bị tắt nhưng không được Node Shutdown Manager của kubelet phát hiện, các pod thuộc một StatefulSet sẽ bị kẹt ở trạng thái terminating trên node đã tắt và không thể chuyển sang một node mới đang chạy. Nguyên nhân là kubelet trên node đã tắt không còn khả dụng để xóa các pod, nên StatefulSet không thể tạo pod mới với cùng tên. Nếu các pod có sử dụng volume, các VolumeAttachment sẽ không bị xóa khỏi node đã tắt ban đầu, nên các volume mà những pod này sử dụng không thể được gắn (attach) vào một node mới đang chạy. Kết quả là, ứng dụng chạy trên StatefulSet không thể hoạt động đúng. Nếu node đã tắt ban đầu khởi động lại, các pod sẽ bị kubelet xóa và các pod mới sẽ được tạo trên một node khác đang chạy. Nếu node đã tắt ban đầu không khởi động lại, những pod này sẽ bị kẹt ở trạng thái terminating trên node đã tắt vĩnh viễn.

Để giảm nhẹ tình huống trên, người dùng có thể thêm thủ công taint `node.kubernetes.io/out-of-service` với effect `NoExecute` hoặc `NoSchedule` vào một Node để đánh dấu nó là hết khả năng phục vụ (out-of-service). Nếu một Node được đánh dấu out-of-service bằng taint này, các pod trên node đó sẽ bị xóa cưỡng bức nếu chúng không có toleration tương ứng, và các thao tác tách (detach) volume cho các pod đang chấm dứt trên node sẽ diễn ra ngay lập tức. Điều này cho phép các Pod trên node out-of-service phục hồi nhanh chóng trên một node khác.

Trong quá trình tắt không nhẹ nhàng, các Pod được chấm dứt theo hai giai đoạn:

1. Xóa cưỡng bức các Pod không có toleration `out-of-service` tương ứng.
1. Lập tức thực hiện thao tác tách volume cho các pod đó.

> **Ghi chú:**
>
> - Trước khi thêm taint `node.kubernetes.io/out-of-service`, cần xác minh rằng node đã thực sự ở trạng thái tắt máy hoặc mất điện (không phải đang trong quá trình khởi động lại).
> - Người dùng bắt buộc phải tự tay gỡ taint out-of-service sau khi các pod đã được chuyển sang node mới và người dùng đã kiểm tra rằng node bị tắt đã được phục hồi, vì chính người dùng là người ban đầu đã thêm taint đó.

### Buộc tách storage khi hết thời gian chờ (Forced storage detach on timeout) {#storage-force-detach-on-timeout}

Trong bất kỳ tình huống nào mà việc xóa pod không thành công trong 6 phút, kubernetes sẽ buộc tách (force detach) các volume đang được unmount nếu node không lành mạnh (unhealthy) tại thời điểm đó. Bất kỳ workload nào vẫn chạy trên node và sử dụng một volume bị buộc tách sẽ gây vi phạm [đặc tả CSI](https://github.com/container-storage-interface/spec/blob/master/spec.md#controllerunpublishvolume), vốn quy định rằng `ControllerUnpublishVolume` "**phải** được gọi sau khi tất cả các lời gọi `NodeUnstageVolume` và `NodeUnpublishVolume` trên volume đó đã được gọi và thành công". Trong những hoàn cảnh như vậy, các volume trên node liên quan có thể gặp hỏng dữ liệu (data corruption).

Hành vi buộc tách storage là tùy chọn; người dùng có thể chọn dùng tính năng "Tắt node không nhẹ nhàng" thay thế.

Có thể tắt tính năng buộc tách storage khi hết thời gian chờ bằng cách đặt trường cấu hình `disable-force-detach-on-timeout` trong `kube-controller-manager`. Việc tắt tính năng buộc tách khi hết thời gian chờ có nghĩa là một volume nằm trên node không lành mạnh trong hơn 6 phút sẽ không bị xóa [VolumeAttachment](https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/volume-attachment-v1/) liên kết với nó.

Sau khi thiết lập này được áp dụng, các pod không lành mạnh vẫn còn gắn với volume phải được phục hồi thông qua quy trình [Tắt node không nhẹ nhàng](#non-graceful-node-shutdown) đã đề cập ở trên.

> **Ghi chú:**
>
> - Phải cẩn trọng khi sử dụng quy trình [Tắt node không nhẹ nhàng](#non-graceful-node-shutdown).
> - Làm sai lệch khỏi các bước được ghi lại ở trên có thể dẫn đến hỏng dữ liệu.

## Tiếp theo (What's next)

Tìm hiểu thêm về các nội dung sau:

- Blog: [Non-Graceful Node Shutdown](https://kubernetes.io/blog/2023/08/16/kubernetes-1-28-non-graceful-node-shutdown-ga/).
- Kiến trúc cluster: [Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/).

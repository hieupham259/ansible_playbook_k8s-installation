# Tinh chỉnh hiệu năng bộ lập lịch (Scheduler Performance Tuning)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/scheduler-perf-tuning/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.14 [beta]`

[kube-scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/#kube-scheduler)
là bộ lập lịch (scheduler) mặc định của Kubernetes. Nó chịu trách nhiệm đặt các Pod
lên các Node trong một cluster.

Các Node trong cluster đáp ứng các yêu cầu lập lịch của một Pod được
gọi là các Node _khả thi_ (feasible) cho Pod đó. Bộ lập lịch tìm các Node khả thi
cho một Pod rồi chạy một tập các hàm để chấm điểm các Node khả thi này,
chọn ra Node có điểm cao nhất trong số các Node khả thi để chạy
Pod. Sau đó bộ lập lịch thông báo cho API server về quyết định này
trong một quá trình gọi là _Binding_.

Trang này giải thích các tối ưu hóa tinh chỉnh hiệu năng phù hợp với
các cluster Kubernetes lớn.

Trong các cluster lớn, bạn có thể tinh chỉnh hành vi của bộ lập lịch để cân bằng
kết quả lập lịch giữa độ trễ (latency — các Pod mới được đặt nhanh chóng) và
độ chính xác (accuracy — bộ lập lịch hiếm khi đưa ra quyết định đặt Pod kém).

Bạn cấu hình thiết lập tinh chỉnh này thông qua thiết lập `percentageOfNodesToScore`
của kube-scheduler. Thiết lập KubeSchedulerConfiguration này xác định
một ngưỡng (threshold) cho việc lập lịch các node trong cluster của bạn.

### Đặt ngưỡng (Setting the threshold)

Tùy chọn `percentageOfNodesToScore` chấp nhận các giá trị số nguyên từ 0
đến 100. Giá trị 0 là một số đặc biệt cho biết kube-scheduler
nên dùng giá trị mặc định được biên dịch sẵn của nó.
Nếu bạn đặt `percentageOfNodesToScore` lớn hơn 100, kube-scheduler hành xử như thể
bạn đã đặt giá trị 100.

Để thay đổi giá trị, hãy chỉnh sửa
[file cấu hình kube-scheduler](https://kubernetes.io/docs/reference/config-api/kube-scheduler-config.v1/)
rồi khởi động lại bộ lập lịch.
Trong nhiều trường hợp, file cấu hình có thể được tìm thấy tại `/etc/kubernetes/config/kube-scheduler.yaml`.

Sau khi thực hiện thay đổi này, bạn có thể chạy

```bash
kubectl get pods -n kube-system | grep kube-scheduler
```

để xác minh rằng thành phần kube-scheduler đang hoạt động bình thường (healthy).

## Ngưỡng chấm điểm node (Node scoring threshold) {#percentage-of-nodes-to-score}

Để cải thiện hiệu năng lập lịch, kube-scheduler có thể dừng việc tìm kiếm
các node khả thi một khi nó đã tìm thấy đủ số lượng. Trong các cluster lớn, điều này
tiết kiệm thời gian so với cách tiếp cận đơn giản là xem xét mọi node.

Bạn chỉ định một ngưỡng cho số lượng node được coi là đủ, dưới dạng tỷ lệ phần trăm
nguyên của tất cả các node trong cluster của bạn. kube-scheduler chuyển đổi giá trị này
thành một số nguyên các node. Trong quá trình lập lịch, nếu kube-scheduler đã xác định
đủ số node khả thi để vượt qua tỷ lệ phần trăm đã cấu hình, kube-scheduler
sẽ dừng tìm kiếm thêm các node khả thi và chuyển sang
[giai đoạn chấm điểm (scoring phase)](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/#kube-scheduler-implementation).

[Cách bộ lập lịch duyệt qua các Node](#how-the-scheduler-iterates-over-nodes)
mô tả quá trình này một cách chi tiết.

### Ngưỡng mặc định (Default threshold)

Nếu bạn không chỉ định ngưỡng, Kubernetes tính toán một con số bằng
công thức tuyến tính cho ra 50% đối với cluster 100 node và cho ra 10%
đối với cluster 5000 node. Cận dưới của giá trị tự động này là 5%.

Điều này có nghĩa là kube-scheduler luôn chấm điểm ít nhất 5% cluster của bạn
bất kể cluster lớn đến đâu, trừ khi bạn đã đặt tường minh
`percentageOfNodesToScore` nhỏ hơn 5.

Nếu bạn muốn bộ lập lịch chấm điểm tất cả các node trong cluster, hãy đặt
`percentageOfNodesToScore` bằng 100.

## Ví dụ (Example)

Dưới đây là một ví dụ cấu hình đặt `percentageOfNodesToScore` bằng 50%.

```yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
algorithmSource:
  provider: DefaultProvider

...

percentageOfNodesToScore: 50
```

## Tinh chỉnh percentageOfNodesToScore (Tuning percentageOfNodesToScore)

`percentageOfNodesToScore` phải là một giá trị từ 1 đến 100 với giá trị
mặc định được tính dựa trên kích thước cluster. Ngoài ra còn có một giá trị
tối thiểu 100 node được cố định trong mã nguồn (hardcoded).

> **Ghi chú:**
>
> Trong các cluster có ít hơn 100 node khả thi, bộ lập lịch vẫn
> kiểm tra tất cả các node vì không có đủ node khả thi để bộ lập lịch
> dừng việc tìm kiếm sớm.
>
> Trong một cluster nhỏ, nếu bạn đặt giá trị thấp cho `percentageOfNodesToScore`,
> thay đổi của bạn sẽ không có hoặc có rất ít tác dụng, vì lý do tương tự.
>
> Nếu cluster của bạn có vài trăm Node hoặc ít hơn, hãy để tùy chọn cấu hình này
> ở giá trị mặc định. Việc thay đổi khó có khả năng cải thiện đáng kể
> hiệu năng của bộ lập lịch.

Một chi tiết quan trọng cần cân nhắc khi đặt giá trị này là khi một số lượng
node nhỏ hơn trong cluster được kiểm tra tính khả thi, một số node sẽ không được
gửi đi chấm điểm cho một Pod nhất định. Kết quả là, một Node lẽ ra có thể
đạt điểm cao hơn cho việc chạy Pod đó thậm chí có thể không được chuyển đến
giai đoạn chấm điểm. Điều này dẫn đến vị trí đặt Pod kém lý tưởng hơn.

Bạn nên tránh đặt `percentageOfNodesToScore` quá thấp để kube-scheduler
không thường xuyên đưa ra các quyết định đặt Pod kém. Tránh đặt tỷ lệ
phần trăm xuống dưới 10%, trừ khi thông lượng (throughput) của bộ lập lịch là yếu tố
sống còn với ứng dụng của bạn và điểm số của các node không quan trọng. Nói cách khác, bạn
chấp nhận chạy Pod trên bất kỳ Node nào miễn là node đó khả thi.

## Cách bộ lập lịch duyệt qua các Node (How the scheduler iterates over Nodes) {#how-the-scheduler-iterates-over-nodes}

Phần này dành cho những ai muốn hiểu chi tiết nội bộ
của tính năng này.

Để mọi Node trong cluster đều có cơ hội công bằng được xem xét
cho việc chạy các Pod, bộ lập lịch duyệt qua các node theo kiểu
vòng tròn (round robin). Bạn có thể hình dung các Node nằm trong một mảng. Bộ lập lịch bắt đầu từ
đầu mảng và kiểm tra tính khả thi của các node cho đến khi tìm được đủ
số Node theo `percentageOfNodesToScore`. Đối với Pod tiếp theo,
bộ lập lịch tiếp tục từ vị trí trong mảng Node mà nó đã dừng lại
khi kiểm tra tính khả thi của các Node cho Pod trước đó.

Nếu các Node nằm ở nhiều zone, bộ lập lịch duyệt qua các Node ở nhiều
zone khác nhau để đảm bảo các Node từ các zone khác nhau đều được xem xét trong
các lần kiểm tra tính khả thi. Ví dụ, xét sáu node trong hai zone:

```
Zone 1: Node 1, Node 2, Node 3, Node 4
Zone 2: Node 5, Node 6
```

Bộ lập lịch đánh giá tính khả thi của các node theo thứ tự sau:

```
Node 1, Node 5, Node 2, Node 6, Node 3, Node 4
```

Sau khi duyệt qua tất cả các Node, nó quay lại Node 1.

## Bật Opportunistic Batching (Enabling Opportunistic Batching)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [beta]`

Khi lập lịch các workload lớn, định nghĩa của các pod thường giống hệt nhau và đòi hỏi bộ lập lịch
thực hiện đi thực hiện lại cùng những thao tác giống nhau. Tính năng [Opportunistic Batching](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#OpportunisticBatching)
cho phép bộ lập lịch tái sử dụng kết quả lọc (filtering) và chấm điểm (scoring) giữa các chu kỳ lập lịch,
giúp tăng tốc đáng kể quá trình lập lịch.

Về cơ bản, tính năng này hoạt động như sau:
1. Bộ lập lịch lập lịch pod-1 và lưu kết quả lập lịch vào cache.
1. Bộ lập lịch lập lịch pod-2, 3, ... bằng các kết quả đã lưu trong cache.
1. Cache hết hạn sau 0,5 giây. Bộ lập lịch lập lịch pod tiếp theo và xây dựng một cache mới.

Các pod có ràng buộc lập lịch tương đương phải đi vào chu kỳ lập lịch liền kề nhau. Khi bộ lập lịch lập lịch một pod có ràng buộc khác, cache không được sử dụng mà bị thay thế bằng một cache mới.

Chúng tôi áp dụng cơ chế lập lịch theo lô (batching) này cho các pod cụ thể mà:
1. Không có inter pod affinity/anti-affinity
1. Không có ràng buộc phân bố theo topology (topology spread constraints)
1. Không dùng DRA (tức là không có Resource Claim nào)
1. Không yêu cầu các tài nguyên mở rộng (extended resources) được hỗ trợ bởi DRA
1. Được lập lịch độc quyền trên các node (tức là việc đặt nhiều hơn một pod lên một node sẽ làm mất hiệu lực cache)

Ngoài ra, để bật tính năng này, cấu hình bộ lập lịch cần phải:
1. Tắt [phân bố topology mặc định (default topology spread)](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/#internal-default-constraints) (đặt rỗng)
1. Đặt `IgnorePreferredTermsOfExistingPods` của [InterPodAffinityArgs](https://kubernetes.io/docs/reference/config-api/kube-scheduler-config.v1/#kubescheduler-config-k8s-io-v1-InterPodAffinityArgs)
thành `true` để việc gom lô hiệu quả hơn

Lưu ý rằng bất cứ khi nào:
1. Các pod hiện có sử dụng ràng buộc pod affinity khớp với bất kỳ label nào của các pod được lập lịch, tính năng này có thể không mang lại lợi ích
1. Các plugin tùy chỉnh được sử dụng, chúng cần triển khai extension point Signature

Các hạn chế và điều kiện này dự kiến sẽ thay đổi trong các phiên bản tương lai.

## Tiếp theo (What's next)

* Xem [tài liệu tham khảo cấu hình kube-scheduler (v1)](https://kubernetes.io/docs/reference/config-api/kube-scheduler-config.v1/)

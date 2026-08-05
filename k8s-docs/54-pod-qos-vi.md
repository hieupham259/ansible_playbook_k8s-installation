# Các lớp chất lượng dịch vụ của Pod (Pod Quality of Service Classes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/>

Trang này giới thiệu về _các lớp chất lượng dịch vụ (Quality of Service — QoS class)_
trong Kubernetes, và giải thích cách Kubernetes gán một QoS class cho mỗi Pod như một
hệ quả của các ràng buộc tài nguyên mà bạn chỉ định cho các container trong Pod đó.
Kubernetes dựa vào sự phân loại này để đưa ra quyết định về việc trục xuất (evict)
những Pod nào khi không còn đủ tài nguyên khả dụng trên một node.

## Các lớp chất lượng dịch vụ (Quality of Service classes) {#quality-of-service-classes}

Kubernetes phân loại các Pod mà bạn chạy và xếp mỗi Pod vào một
_lớp chất lượng dịch vụ (QoS class)_ cụ thể. Kubernetes dùng sự phân loại đó để tác động
đến cách các pod khác nhau được xử lý. Kubernetes thực hiện việc phân loại này dựa trên
[yêu cầu tài nguyên (resource request)](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
của các Container trong Pod đó, cùng với
mối liên hệ giữa các request này và giới hạn tài nguyên (resource limit).
Đây được gọi là lớp Quality of Service
(QoS). Kubernetes gán cho mỗi Pod một QoS class dựa trên các yêu cầu và giới hạn
tài nguyên của các Container thành phần của nó. QoS class được Kubernetes dùng để quyết định
trục xuất những Pod nào khỏi một node đang chịu
[áp lực node (Node Pressure)](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/). Các
QoS class có thể có là `Guaranteed`, `Burstable`, và `BestEffort`. Khi một node cạn kiệt tài nguyên,
Kubernetes trước tiên sẽ trục xuất các Pod `BestEffort` đang chạy trên node đó, tiếp theo là các Pod `Burstable` và
cuối cùng là các Pod `Guaranteed`. Khi việc trục xuất này xảy ra do áp lực tài nguyên, chỉ những Pod
vượt quá yêu cầu tài nguyên mới là ứng viên bị trục xuất.

### Guaranteed

Các Pod thuộc lớp `Guaranteed` có giới hạn tài nguyên nghiêm ngặt nhất và ít có khả năng
bị trục xuất nhất. Chúng được đảm bảo không bị kill cho đến khi chúng vượt quá giới hạn
của mình, hoặc khi không còn Pod nào có độ ưu tiên thấp hơn có thể bị chiếm chỗ (preempt)
khỏi node. Chúng không được phép chiếm dụng tài nguyên vượt quá giới hạn đã chỉ định.
Các Pod này cũng có thể sử dụng các CPU độc quyền nhờ chính sách quản lý CPU
[`static`](https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/#static-policy-configuration).

#### Tiêu chí (Criteria)

Để một Pod được gán QoS class `Guaranteed`:

* Mọi Container trong Pod phải có giới hạn (limit) memory và yêu cầu (request) memory, cả hai đều lớn hơn 0.
* Với mọi Container trong Pod, giới hạn memory phải bằng yêu cầu memory.
* Mọi Container trong Pod phải có giới hạn CPU và yêu cầu CPU, cả hai đều lớn hơn 0.
* Với mọi Container trong Pod, giới hạn CPU phải bằng yêu cầu CPU.

Còn nếu Pod sử dụng [tài nguyên cấp Pod (Pod-level resources)](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#pod-level-resource-specification):

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.34 [beta]`

* Pod phải có giới hạn memory và yêu cầu memory ở cấp Pod, và hai giá trị này phải bằng nhau.
* Pod phải có giới hạn CPU và yêu cầu CPU ở cấp Pod, và hai giá trị này phải bằng nhau.

### Burstable

Các Pod thuộc lớp `Burstable` có một số đảm bảo tài nguyên ở mức sàn dựa trên request, nhưng
không đòi hỏi một limit cụ thể. Nếu limit không được chỉ định, nó mặc định là một
limit tương đương với năng lực (capacity) của node, điều này cho phép Pod linh hoạt tăng
lượng tài nguyên sử dụng nếu tài nguyên còn khả dụng. Trong trường hợp Pod bị trục xuất do
áp lực tài nguyên node, các Pod này chỉ bị trục xuất sau khi tất cả các Pod `BestEffort` đã bị trục xuất.
Vì một Pod `Burstable` có thể chứa Container không có giới hạn hay yêu cầu tài nguyên nào, nên một Pod
thuộc lớp `Burstable` có thể cố sử dụng bất kỳ lượng tài nguyên node nào.

#### Tiêu chí (Criteria)

Một Pod được gán QoS class `Burstable` nếu:

* Pod không đạt các tiêu chí cho QoS class `Guaranteed`.
* Ít nhất một Container trong Pod có yêu cầu hoặc giới hạn memory hoặc CPU,
  hoặc Pod có yêu cầu hoặc giới hạn memory hoặc CPU ở cấp Pod.

### BestEffort

Các Pod trong QoS class `BestEffort` có thể sử dụng những tài nguyên node chưa được gán
riêng cho các Pod thuộc các QoS class khác. Ví dụ, nếu bạn có một node với 16 lõi CPU
khả dụng cho kubelet, và bạn gán 4 lõi CPU cho một Pod `Guaranteed`, thì một Pod trong
QoS class `BestEffort` có thể cố sử dụng bất kỳ lượng nào trong 12 lõi CPU còn lại.

Kubelet ưu tiên trục xuất các Pod `BestEffort` nếu node rơi vào tình trạng áp lực tài nguyên.

#### Tiêu chí (Criteria)

Một Pod có QoS class `BestEffort` nếu nó không đạt tiêu chí của cả `Guaranteed`
lẫn `Burstable`. Nói cách khác, một Pod chỉ là `BestEffort` khi không Container nào trong Pod có
giới hạn memory hay yêu cầu memory, không Container nào trong Pod có
giới hạn CPU hay yêu cầu CPU, và Pod không có bất kỳ giới hạn hay yêu cầu memory hoặc CPU nào ở cấp Pod.
Các Container trong Pod vẫn có thể yêu cầu các tài nguyên khác (không phải CPU hay memory) mà vẫn được
phân loại là `BestEffort`.

## Memory QoS với cgroup v2 (Memory QoS with cgroup v2) {#memory-qos-with-cgroup-v2}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.22 [alpha]`

Memory QoS sử dụng memory controller của cgroup v2 để quản lý việc điều tiết (throttle)
và bảo vệ memory trong Kubernetes. Nó dùng QoS class của Pod để quyết định áp dụng các
thiết lập cgroup nào, nhưng đây là một tính năng opt-in riêng biệt. Việc tắt Memory QoS
không làm thay đổi cách các Pod được phân loại.

### Điều tiết memory (Memory throttling)

Với các pod Burstable, kubelet đặt `memory.high` để điều tiết việc cấp phát memory
trước khi workload chạm đến giới hạn cứng của nó (`memory.max`). Ngưỡng điều tiết
được tính như sau:

```
memory.high = requests + memoryThrottlingFactor * (limits - requests)
```

trong đó `memoryThrottlingFactor` mặc định là 0.9. Ví dụ, một container có
request 256 MiB và limit 1 GiB sẽ có `memory.high` được đặt xấp xỉ 947 MiB.
Nếu một container Burstable không có giới hạn memory, memory khả cấp (allocatable)
của node được dùng thay cho limit.

Các pod Guaranteed không được đặt `memory.high` vì request của chúng bằng
limit. Các pod BestEffort không được đặt `memory.high` vì chúng không có request
hay limit nào.

### Cấu hình memory reservation (Configuring memory reservation)

Việc dự trữ memory (memory reservation) được điều khiển qua trường cấu hình
`memoryReservationPolicy` của kubelet:

- `None` (mặc định): kubelet không đặt `memory.min` hay `memory.low` cho
  các container và pod. Không có phần memory nào bị kernel khóa cứng.
- `TieredReservation`: kubelet thiết lập mức bảo vệ memory phân tầng dựa trên
  QoS class của Pod:
  - Pod **Guaranteed**: `memory.min` được đặt bằng memory request. Kernel
    sẽ không thu hồi (reclaim) phần memory này trong bất kỳ hoàn cảnh nào.
  - Pod **Burstable**: `memory.low` được đặt bằng memory request. Kernel
    ưu tiên giữ lại phần memory này nhưng có thể thu hồi nó khi chịu áp lực cực lớn.
  - Pod **BestEffort**: không thiết lập cơ chế bảo vệ memory nào.

### Yêu cầu hệ thống (System requirements)

Memory QoS yêu cầu Linux với cgroup v2. Khuyến nghị dùng kernel 5.9 trở lên
vì việc điều tiết bằng `memory.high` trên các kernel cũ hơn có thể kích hoạt một
[lỗi livelock đã biết](https://lore.kernel.org/all/20200710012662.GA29679@chep.ntu.edu.tw/).
Nếu feature gate `MemoryQoS` được bật trên một kernel cũ hơn, kubelet sẽ ghi log
một cảnh báo khi khởi động.

## Một số hành vi không phụ thuộc vào QoS class (Some behavior is independent of QoS class) {#class-independent-behavior}

Một số hành vi nhất định không phụ thuộc vào QoS class mà Kubernetes gán. Ví dụ:

* Bất kỳ Container nào vượt quá giới hạn tài nguyên sẽ bị kubelet kill và khởi động lại
  mà không ảnh hưởng đến các Container khác trong Pod đó.

* Nếu một Container vượt quá yêu cầu tài nguyên của nó và node mà nó đang chạy gặp
  áp lực tài nguyên, thì Pod chứa nó trở thành ứng viên cho việc [trục xuất](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/).
  Nếu điều này xảy ra, tất cả các Container trong Pod sẽ bị kết thúc. Kubernetes có thể tạo một
  Pod thay thế, thường là trên một node khác.

* Yêu cầu tài nguyên của một Pod bằng tổng các yêu cầu tài nguyên của
  các Container thành phần của nó, và giới hạn tài nguyên của một Pod bằng tổng
  các giới hạn tài nguyên của các Container thành phần của nó.

* Kube-scheduler không xem xét QoS class khi chọn những Pod nào để
  [chiếm chỗ (preempt)](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#preemption).
  Việc chiếm chỗ có thể xảy ra khi một cluster không có đủ tài nguyên để chạy tất cả các Pod
  mà bạn đã định nghĩa.

* QoS class được xác định khi Pod được tạo và giữ nguyên không đổi trong suốt
  vòng đời của Pod. Nếu sau đó bạn thử một thao tác
  [thay đổi kích thước tại chỗ (in-place resize)](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-resize)
  mà sẽ dẫn đến một QoS class khác, thao tác resize đó sẽ bị tầng admission từ chối.

## Tiếp theo (What's next)

* Tìm hiểu về [quản lý tài nguyên cho Pod và Container](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/).
* Tìm hiểu về [trục xuất do áp lực node (Node-pressure eviction)](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/).
* Tìm hiểu về [độ ưu tiên và cơ chế chiếm chỗ của Pod (Pod priority and preemption)](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/).
* Tìm hiểu về [sự gián đoạn Pod (Pod disruptions)](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/).
* Tìm hiểu cách [gán tài nguyên memory cho container và pod](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/).
* Tìm hiểu cách [gán tài nguyên CPU cho container và pod](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/).
* Tìm hiểu cách [cấu hình Quality of Service cho Pod](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/).

# Đóng gói tài nguyên (Resource Bin Packing)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/resource-bin-packing/>

> **Ghi chú:**
>
> Bài viết này áp dụng cho việc đóng gói tài nguyên (resource bin packing) trong ngữ cảnh lập lịch một pod đơn lẻ. Đối với việc đóng gói khi lập lịch các nhóm pod (pod group), vui lòng đọc [bài viết về Lập lịch nhận biết topology (Topology-aware Scheduling)](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-aware-scheduling/).

Trong [plugin lập lịch](https://kubernetes.io/docs/reference/scheduling/config/#scheduling-plugins) `NodeResourcesFit` của kube-scheduler, có hai
chiến lược chấm điểm (scoring strategy) hỗ trợ đóng gói tài nguyên: `MostAllocated` và `RequestedToCapacityRatio`.

## Bật đóng gói tài nguyên bằng chiến lược MostAllocated (Enabling bin packing using MostAllocated strategy)

Chiến lược `MostAllocated` chấm điểm các node dựa trên mức sử dụng tài nguyên, ưu tiên các node có mức cấp phát (allocation) cao hơn.
Với mỗi loại tài nguyên, bạn có thể đặt một trọng số (weight) để điều chỉnh mức ảnh hưởng của nó trong điểm số của node.

Để đặt chiến lược `MostAllocated` cho plugin `NodeResourcesFit`, sử dụng một
[cấu hình bộ lập lịch (scheduler configuration)](https://kubernetes.io/docs/reference/scheduling/config) tương tự như sau:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- pluginConfig:
  - args:
      scoringStrategy:
        resources:
        - name: cpu
          weight: 1
        - name: memory
          weight: 1
        - name: intel.com/foo
          weight: 3
        - name: intel.com/bar
          weight: 3
        type: MostAllocated
    name: NodeResourcesFit
```

Với cấu hình này, các node được chấm điểm bằng trung bình có trọng số của mức sử dụng trên cả bốn
loại tài nguyên. Vì `intel.com/foo` và `intel.com/bar` mỗi loại mang trọng số `3` so với `1` của
CPU và memory, mức sử dụng của các tài nguyên mở rộng (extended resources) đó có ảnh hưởng gấp ba lần lên
điểm số cuối cùng của node. Bộ lập lịch chọn node có điểm cao nhất, nhằm lập lịch các pod lên
các node có mức sử dụng cao. Điều này giúp chuẩn bị cho việc thu hẹp (scale-down) các node có mức sử dụng thấp nhất.

Để tìm hiểu thêm về các tham số khác và cấu hình mặc định của chúng, xem tài liệu API của
[`NodeResourcesFitArgs`](https://kubernetes.io/docs/reference/config-api/kube-scheduler-config.v1/#kubescheduler-config-k8s-io-v1-NodeResourcesFitArgs).

## Bật đóng gói tài nguyên bằng RequestedToCapacityRatio (Enabling bin packing using RequestedToCapacityRatio)

Chiến lược `RequestedToCapacityRatio` cho phép người dùng chỉ định các tài nguyên cùng với trọng số cho
mỗi tài nguyên để chấm điểm các node dựa trên tỷ lệ giữa lượng yêu cầu (request) và dung lượng (capacity). Điều này
cho phép người dùng đóng gói các tài nguyên mở rộng bằng cách sử dụng các tham số phù hợp
để cải thiện mức sử dụng của các tài nguyên khan hiếm trong các cluster lớn. Nó ưu tiên các node theo một
hàm được cấu hình dựa trên lượng tài nguyên đã cấp phát. Hành vi của `RequestedToCapacityRatio` trong
hàm chấm điểm của `NodeResourcesFit` có thể được điều khiển thông qua trường
[scoringStrategy](https://kubernetes.io/docs/reference/config-api/kube-scheduler-config.v1/#kubescheduler-config-k8s-io-v1-ScoringStrategy).
Trong trường `scoringStrategy`, bạn có thể cấu hình hai tham số: `requestedToCapacityRatio` và
`resources`. Trường `shape` trong tham số `requestedToCapacityRatio`
cho phép người dùng tinh chỉnh hàm này theo hướng ưu tiên yêu cầu ít nhất (least requested) hoặc
yêu cầu nhiều nhất (most requested) dựa trên các giá trị `utilization` và `score`. Tham số `resources`
bao gồm cả `name` của tài nguyên được xem xét khi chấm điểm và
`weight` tương ứng, chỉ định trọng số của mỗi tài nguyên.

Dưới đây là một ví dụ cấu hình đặt
hành vi đóng gói tài nguyên cho các tài nguyên mở rộng `intel.com/foo` và `intel.com/bar`
bằng trường `requestedToCapacityRatio`.

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- pluginConfig:
  - args:
      scoringStrategy:
        resources:
        - name: intel.com/foo
          weight: 3
        - name: intel.com/bar
          weight: 3
        requestedToCapacityRatio:
          shape:
          - utilization: 0
            score: 0
          - utilization: 100
            score: 10
        type: RequestedToCapacityRatio
    name: NodeResourcesFit
```

Trong ví dụ này, chỉ các tài nguyên mở rộng `intel.com/foo` và `intel.com/bar` được liệt kê trong
`resources`. Do đó plugin `NodeResourcesFit` chấm điểm các node chỉ dựa trên mức sử dụng
của hai tài nguyên đó; CPU và memory không đóng góp vào điểm số từ plugin này. Vì
`shape` được cấu hình gán điểm cao hơn khi mức sử dụng tăng lên (`score: 0` tại `utilization: 0`
tăng đến `score: 10` tại `utilization: 100`), bộ lập lịch ưu tiên các node mà các
tài nguyên mở rộng này đã được sử dụng nhiều hơn, đóng gói các yêu cầu về chúng lên càng ít node càng tốt.

Để đưa CPU và memory vào chiến lược chấm điểm này, hãy thêm chúng vào danh sách `resources`. Lưu ý rằng tất cả
các tài nguyên trong danh sách dùng chung cùng một hàm `shape`, nên làm như vậy sẽ áp dụng cùng đường cong
đóng gói đó cho các tài nguyên này.

Việc tham chiếu file `KubeSchedulerConfiguration` bằng cờ
`--config=/path/to/config/file` của kube-scheduler sẽ truyền cấu hình này cho
bộ lập lịch.

Để tìm hiểu thêm về các tham số khác và cấu hình mặc định của chúng, xem tài liệu API của
[`NodeResourcesFitArgs`](https://kubernetes.io/docs/reference/config-api/kube-scheduler-config.v1/#kubescheduler-config-k8s-io-v1-NodeResourcesFitArgs).

### Tinh chỉnh hàm chấm điểm (Tuning the score function)

`shape` được dùng để chỉ định hành vi của hàm `RequestedToCapacityRatio`.

```yaml
shape:
  - utilization: 0
    score: 0
  - utilization: 100
    score: 10
```

Các đối số trên cho node một `score` bằng 0 nếu `utilization` là 0% và bằng 10 khi
`utilization` là 100%, nhờ đó bật hành vi đóng gói tài nguyên. Để bật chế độ ưu tiên
yêu cầu ít nhất (least requested), giá trị score phải được đảo ngược như sau.

```yaml
shape:
  - utilization: 0
    score: 10
  - utilization: 100
    score: 0
```

`resources` là một tham số tùy chọn với giá trị mặc định là:

```yaml
resources:
  - name: cpu
    weight: 1
  - name: memory
    weight: 1
```

Nó có thể được dùng để thêm các tài nguyên mở rộng như sau:

```yaml
resources:
  - name: intel.com/foo
    weight: 5
  - name: cpu
    weight: 3
  - name: memory
    weight: 1
```

Tham số `weight` là tùy chọn và được đặt bằng 1 nếu không được chỉ định. Ngoài ra,
`weight` không thể được đặt thành giá trị âm.

### Chấm điểm node cho việc phân bổ dung lượng (Node scoring for capacity allocation)

Phần này dành cho những ai muốn hiểu chi tiết nội bộ
của tính năng này.
Dưới đây là một ví dụ về cách điểm số của node được tính cho một tập giá trị cho trước.

Tài nguyên được yêu cầu (requested):

```
intel.com/foo : 2
memory: 256MB
cpu: 2
```

Trọng số tài nguyên:

```
intel.com/foo : 5
memory: 1
cpu: 3
```

FunctionShapePoint {{0, 0}, {100, 10}}

Thông số Node 1:

```
Available:
  intel.com/foo: 4
  memory: 1 GB
  cpu: 8

Used:
  intel.com/foo: 1
  memory: 256MB
  cpu: 1
```

Điểm số của node:

```
intel.com/foo  = resourceScoringFunction((2+1),4)
               = (100 - ((4-3)*100/4))
               = (100 - 25)
               = 75                       # requested + used = 75% * available
               = rawScoringFunction(75)
               = 7                        # floor(75/10)

memory         = resourceScoringFunction((256+256),1024)
               = (100 -((1024-512)*100/1024))
               = 50                       # requested + used = 50% * available
               = rawScoringFunction(50)
               = 5                        # floor(50/10)

cpu            = resourceScoringFunction((2+1),8)
               = (100 -((8-3)*100/8))
               = 37.5                     # requested + used = 37.5% * available
               = rawScoringFunction(37.5)
               = 3                        # floor(37.5/10)

NodeScore   =  ((7 * 5) + (5 * 1) + (3 * 3)) / (5 + 1 + 3)
            =  5
```

Thông số Node 2:

```
Available:
  intel.com/foo: 8
  memory: 1GB
  cpu: 8
Used:
  intel.com/foo: 2
  memory: 512MB
  cpu: 6
```

Điểm số của node:

```
intel.com/foo  = resourceScoringFunction((2+2),8)
               =  (100 - ((8-4)*100/8)
               =  (100 - 50)
               =  50
               =  rawScoringFunction(50)
               = 5

memory         = resourceScoringFunction((256+512),1024)
               = (100 -((1024-768)*100/1024))
               = 75
               = rawScoringFunction(75)
               = 7

cpu            = resourceScoringFunction((2+6),8)
               = (100 -((8-8)*100/8))
               = 100
               = rawScoringFunction(100)
               = 10

NodeScore   =  ((5 * 5) + (7 * 1) + (10 * 3)) / (5 + 1 + 3)
            =  7

```

## Tiếp theo (What's next)

- Đọc thêm về [scheduling framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)
- Đọc thêm về [cấu hình bộ lập lịch (scheduler configuration)](https://kubernetes.io/docs/reference/scheduling/config/)

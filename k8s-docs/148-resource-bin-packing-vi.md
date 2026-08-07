# Đóng gói tài nguyên (Resource Bin Packing)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/resource-bin-packing/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 7 → nhóm [7a](LO-TRINH-ADMIN.md#7a-scheduling-và-eviction), bài 13/13 ·
Kiểm chứng ở Lab 7a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài cuối của nhóm, và cũng là bài duy nhất **thay đổi cách chấm điểm mặc định** thay vì thêm
ràng buộc cho Pod. Mọi thứ ở đây là cấu hình cấp cluster trong `KubeSchedulerConfiguration`,
không phải trường trong Pod spec — nên bạn không thử được bằng một manifest.

Ví dụ trong bài dùng extended resource `intel.com/foo` và `intel.com/bar`; cluster lab không
có phần cứng như vậy, nhưng cú pháp và ý nghĩa `resources` + `weight` vẫn đọc như thường, chỉ
cần thay bằng `cpu` và `memory` khi hình dung.

**Phải hiểu ở lần đọc này:**

- Bin packing là **chiến lược chấm điểm** của plugin `NodeResourcesFit`, tức là nó chỉ tác động
  vào **bước chấm điểm** của bài [137](137-kube-scheduler-vi.md), không đụng vào bước lọc.
- `MostAllocated` chấm điểm cao cho node **đã cấp phát nhiều**, nhằm dồn Pod lên ít node nhất
  có thể — mục đích bài nêu là **chuẩn bị cho việc thu hẹp (scale-down) các node ít dùng**.
- `RequestedToCapacityRatio` cho bạn tự vẽ đường cong qua `shape`: điểm **tăng** theo
  `utilization` là bin packing; đảo ngược lại (`utilization: 0 → score: 10`) là chế độ **ưu
  tiên yêu cầu ít nhất**, tức trải Pod ra.
- Danh sách `resources` quyết định **tài nguyên nào được tính điểm**: tài nguyên không nằm
  trong danh sách **không đóng góp gì** vào điểm của plugin này. Mặc định là `cpu` và `memory`
  với `weight: 1`; `weight` không được âm, và **mọi tài nguyên trong danh sách dùng chung một
  `shape`**.
- Đây là cấu hình cấp cluster, nạp bằng cờ `--config=/path/to/config/file` của kube-scheduler.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Chấm điểm node cho việc phân bổ dung lượng* — phép tính điểm Node 1 và Node 2 | chính bài nói đây là chi tiết nội bộ | không cần |
| Ghi chú đầu bài về đóng gói khi lập lịch nhóm pod | thuộc lập lịch nâng cao | giai đoạn 13, bài [153](153-topology-aware-scheduling-vi.md) |
| Ý nghĩa vận hành của extended resource `intel.com/*` | cluster lab không có thiết bị chuyên dụng | giai đoạn 14, bài [184](184-device-plugins-vi.md) |

---

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

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 7:

1. Bin packing tác động vào bước nào của kube-scheduler? Bật `MostAllocated` có làm một node
   vốn không đủ tài nguyên trở nên nhận được Pod không?
2. Hai `shape` sau khác nhau ra sao về hành vi: `{utilization: 0 → score: 0, utilization: 100
   → score: 10}` và `{utilization: 0 → score: 10, utilization: 100 → score: 0}`?
3. Trong ví dụ `RequestedToCapacityRatio` của bài, `resources` chỉ liệt kê `intel.com/foo` và
   `intel.com/bar`. CPU và memory có ảnh hưởng tới điểm số từ plugin `NodeResourcesFit` không?
4. Trên cluster lab hai worker 2 vCPU / 6 GB RAM, giả sử bạn bật `MostAllocated` rồi tạo lần
   lượt nhiều Pod nhỏ. Chúng có xu hướng nằm ở đâu, và điều đó tốt hay xấu cho cluster này?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Chỉ tác động vào **bước chấm điểm** — cả `MostAllocated` lẫn `RequestedToCapacityRatio` đều
   là **chiến lược chấm điểm (scoring strategy)** của plugin `NodeResourcesFit`. **Không**:
   node không đủ tài nguyên đã bị loại từ bước lọc và không bao giờ đi tới bước chấm điểm.
   Bin packing chỉ đổi **thứ tự ưu tiên giữa các node vốn đã khả thi**.
2. Cái thứ nhất **bật đóng gói tài nguyên**: điểm tăng theo mức sử dụng, nên node càng đầy càng
   được ưu tiên, và Pod bị dồn lên càng ít node càng tốt. Cái thứ hai là bản **đảo ngược**, cho
   chế độ **ưu tiên yêu cầu ít nhất (least requested)**: node càng rỗng càng được ưu tiên, tức
   là trải Pod ra. Cùng một plugin, chỉ đổi đường cong là đảo ngược hoàn toàn ý đồ.
3. **Không.** Plugin chấm điểm các node **chỉ dựa trên mức sử dụng của hai tài nguyên được liệt
   kê**; CPU và memory không đóng góp vào điểm số từ plugin này. Muốn tính chúng thì phải thêm
   vào danh sách `resources` — và khi thêm, hãy nhớ **tất cả tài nguyên trong danh sách dùng
   chung cùng một `shape`**, nên cùng một đường cong đóng gói sẽ áp lên chúng. Đây là chỗ dễ
   nhầm: `resources` để trống mới lấy mặc định `cpu`/`memory` weight 1, chứ liệt kê tài nguyên
   khác **không** có nghĩa là "thêm vào bên cạnh mặc định".
4. Chúng **dồn về một worker** cho tới khi worker đó không còn khả thi, thay vì rải đều hai
   worker. Với cluster lab thì điều này **xấu**: lợi ích mà bài nêu cho `MostAllocated` là
   "chuẩn bị cho việc thu hẹp (scale-down) các node có mức sử dụng thấp nhất" — cluster lab
   không co giãn node nên bạn không thu được lợi ích đó, mà lại mất khả năng chịu lỗi vì gần
   như toàn bộ workload nằm trên một máy. Đây chính là lý do bin packing là lựa chọn cần cân
   nhắc theo bối cảnh, không phải mặc định tốt hơn.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng. Đây là bài cuối của nhóm 7a — khi
trả lời được hết cả 13 bài, bạn sẵn sàng vào Lab 7a (chưa viết, xem
[bản đồ lab](labs/README.md#4-bản-đồ-lab)).

# Taint và Toleration (Taints and Tolerations)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 7 → nhóm [7a](00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction), bài 4/13 ·
Kiểm chứng ở [Lab 7a](labs/LAB-7A-LAP-LICH-VA-EVICTION.md).

Đây là **mặt đối ngẫu** của bài [138](138-assign-pod-node-vi.md): affinity là thuộc tính của
Pod dùng để **thu hút** Pod về một tập node; taint là thuộc tính của node dùng để **đẩy** Pod
ra. Bạn đã gặp cơ chế này từ Lab 1a mà chưa biết tên: control plane node `lab-k8s-master` mang một
taint, và đó chính là lý do Pod thường của bạn luôn rơi xuống hai worker.

Nửa sau của bài (*Eviction dựa trên taint*, *Taint Node theo điều kiện*) không nói về việc bạn
đặt taint, mà về **các taint Kubernetes tự đặt** khi node có vấn đề. Đó là phần nối thẳng sang
bài [142](142-node-pressure-eviction-vi.md), đọc kỹ.

**Phải hiểu ở lần đọc này:**

- Taint nằm trên **node**, toleration nằm trên **Pod**. Toleration chỉ **cho phép** lập lịch
  lên node bị taint, **không đảm bảo** và cũng không kéo Pod về đó — bộ lập lịch vẫn đánh giá
  các tham số khác.
- Ba effect: `NoSchedule` chặn Pod mới nhưng **không đuổi** Pod đang chạy; `PreferNoSchedule`
  là bản mềm của nó; `NoExecute` **đuổi cả Pod đang chạy**, và `tolerationSeconds` quyết định
  Pod còn được ở lại bao lâu sau khi taint được thêm.
- Quy tắc khớp: cùng `key`, cùng `effect`, và `operator` là `Exists` hoặc là `Equal` với
  `value` bằng nhau. Nhiều taint được xử lý **như một bộ lọc**: bỏ qua các taint được dung
  thứ, những taint còn lại vẫn phát huy effect của chúng.
- `.spec.nodeName` bỏ qua bộ lập lịch nên Pod lên được cả node có taint `NoSchedule`; nhưng
  nếu node đó cũng có taint `NoExecute` thì **kubelet vẫn đẩy Pod ra**.
- Kubernetes tự taint node theo tình trạng: `node.kubernetes.io/not-ready`, `unreachable`,
  `memory-pressure`, `disk-pressure`, `pid-pressure`, `unschedulable`… Bộ lập lịch **nhìn
  taint chứ không nhìn node condition**. Với `not-ready` và `unreachable`, Kubernetes tự thêm
  toleration `tolerationSeconds=300` cho Pod, còn Pod của DaemonSet được thêm toleration
  `NoExecute` **không** kèm `tolerationSeconds` nên không bao giờ bị đuổi vì hai sự cố này.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Toán tử so sánh số* `Gt`/`Lt` và cảnh báo về feature gate `TaintTolerationComparisonOperators` | tính năng alpha, chỉ dùng với taint có giá trị số | không cần |
| *Các trường hợp sử dụng ví dụ* — node chuyên dụng, node có phần cứng đặc biệt | cần admission controller tùy chỉnh và extended resource | giai đoạn 9, bài [119](119-controlling-access-vi.md) |
| `ExtendedResourceToleration` | như trên | giai đoạn 9, bài [119](119-controlling-access-vi.md) |
| *Taint và toleration cho thiết bị* | thuộc cấp phát tài nguyên động | giai đoạn 13, bài [149](149-dynamic-resource-allocation-vi.md) |
| Ghi chú `taint-eviction-controller` tách khỏi node controller từ 1.29 | chi tiết triển khai nội bộ | không cần |

---

[_Node affinity_](138-assign-pod-node-vi.md#affinity-and-anti-affinity)
là một thuộc tính của Pod có tác dụng *thu hút* chúng đến
một tập các node (dưới dạng ưu tiên hoặc yêu cầu bắt buộc). _Taint_ thì ngược lại — chúng cho phép một node đẩy lùi (repel) một tập các pod.

_Toleration_ được áp dụng cho pod. Toleration cho phép bộ lập lịch (scheduler) lập lịch các pod có
taint khớp tương ứng. Toleration cho phép việc lập lịch nhưng không đảm bảo việc lập lịch: bộ lập lịch còn
[đánh giá các tham số khác](141-pod-priority-preemption-vi.md)
như một phần chức năng của nó.

Taint và toleration phối hợp với nhau để đảm bảo pod không bị lập lịch
lên các node không phù hợp. Một hoặc nhiều taint được áp dụng lên một node; điều này
đánh dấu rằng node đó không nên chấp nhận bất kỳ pod nào không dung thứ (tolerate) các taint đó.

## Khái niệm (Concepts) {#concepts}

Bạn thêm một taint vào node bằng [kubectl taint](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#taint).
Ví dụ,

```shell
kubectl taint nodes node1 key1=value1:NoSchedule
```

đặt một taint lên node `node1`. Taint này có key `key1`, value `value1`, và hiệu ứng (taint effect) `NoSchedule`.
Điều này có nghĩa là không pod nào có thể được lập lịch lên `node1` trừ khi nó có toleration khớp tương ứng.

Để gỡ bỏ taint đã được thêm bởi lệnh trên, bạn có thể chạy:

```shell
kubectl taint nodes node1 key1=value1:NoSchedule-
```

Bạn chỉ định toleration cho pod trong PodSpec. Cả hai toleration dưới đây đều "khớp" với
taint được tạo bởi lệnh `kubectl taint` ở trên, do đó một pod có một trong hai toleration này
sẽ có thể được lập lịch lên `node1`:

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
```

```yaml
tolerations:
- key: "key1"
  operator: "Exists"
  effect: "NoSchedule"
```

Bộ lập lịch mặc định của Kubernetes xét đến taint và toleration khi
chọn node để chạy một Pod cụ thể. Tuy nhiên, nếu bạn chỉ định thủ công
`.spec.nodeName` cho một Pod, hành động đó bỏ qua bộ lập lịch; Pod khi đó được
gắn (bound) vào node mà bạn đã gán, ngay cả khi node bạn chọn có các taint
`NoSchedule` trên đó.
Nếu điều này xảy ra và node đó cũng có taint `NoExecute`, kubelet sẽ
đẩy Pod ra khỏi node trừ khi có toleration phù hợp được thiết lập.

Dưới đây là ví dụ về một pod có định nghĩa một số toleration:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  tolerations:
  - key: "example-key"
    operator: "Exists"
    effect: "NoSchedule"
```

Giá trị mặc định cho `operator` là `Equal`.

Một toleration "khớp" với một taint nếu các key giống nhau và các effect giống nhau, và:

* `operator` là `Exists` (trường hợp này không được chỉ định `value`), hoặc
* `operator` là `Equal` và các value bằng nhau.

> **Ghi chú:**
>
> Có hai trường hợp đặc biệt:
>
> Nếu `key` rỗng, thì `operator` phải là `Exists`, khi đó sẽ khớp với mọi key và value.
> Lưu ý rằng `effect` vẫn cần phải khớp cùng lúc.
>
> Một `effect` rỗng khớp với mọi effect có key `key1`.

Ví dụ trên sử dụng `effect` là `NoSchedule`. Ngoài ra, bạn có thể dùng `effect` là `PreferNoSchedule`.

Các giá trị được phép cho trường `effect` là:

`NoExecute`
: Ảnh hưởng đến các pod đang chạy trên node như sau:

  * Các pod không dung thứ taint sẽ bị thu hồi (evict) ngay lập tức
  * Các pod dung thứ taint mà không chỉ định `tolerationSeconds` trong
    đặc tả toleration của chúng sẽ vẫn được gắn với node mãi mãi
  * Các pod dung thứ taint và có chỉ định `tolerationSeconds` sẽ vẫn
    được gắn trong khoảng thời gian được chỉ định. Sau khi hết thời gian đó,
    node lifecycle controller sẽ thu hồi các Pod khỏi node.

`NoSchedule`
: Không Pod mới nào được lập lịch lên node bị taint trừ khi chúng có toleration
  khớp tương ứng. Các Pod hiện đang chạy trên node **không** bị thu hồi.

`PreferNoSchedule`
: `PreferNoSchedule` là phiên bản "ưu tiên" hay "mềm" của `NoSchedule`.
  Control plane sẽ *cố gắng* tránh đặt một Pod không dung thứ taint
  lên node đó, nhưng điều này không được đảm bảo.

Bạn có thể đặt nhiều taint trên cùng một node và nhiều toleration trên cùng một pod.
Cách Kubernetes xử lý nhiều taint và toleration giống như một bộ lọc: bắt đầu
với tất cả các taint của node, sau đó bỏ qua những taint mà pod có toleration khớp tương ứng;
những taint còn lại không bị bỏ qua sẽ có các hiệu ứng tương ứng đối với pod. Cụ thể,

* nếu có ít nhất một taint không bị bỏ qua với effect `NoSchedule` thì Kubernetes sẽ không lập lịch
pod lên node đó
* nếu không có taint không bị bỏ qua nào với effect `NoSchedule` nhưng có ít nhất một taint không bị bỏ qua với
effect `PreferNoSchedule` thì Kubernetes sẽ *cố gắng* không lập lịch pod lên node đó
* nếu có ít nhất một taint không bị bỏ qua với effect `NoExecute` thì pod sẽ bị thu hồi khỏi
node (nếu nó đang chạy trên node đó), và sẽ không được
lập lịch lên node (nếu nó chưa chạy trên node đó).

Ví dụ, hãy tưởng tượng bạn taint một node như sau

```shell
kubectl taint nodes node1 key1=value1:NoSchedule
kubectl taint nodes node1 key1=value1:NoExecute
kubectl taint nodes node1 key2=value2:NoSchedule
```

Và một pod có hai toleration:

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
```

Trong trường hợp này, pod sẽ không thể được lập lịch lên node, vì không có
toleration nào khớp với taint thứ ba. Nhưng nó sẽ có thể tiếp tục chạy nếu nó
đã đang chạy trên node từ trước khi taint được thêm vào, vì taint thứ ba là taint
duy nhất trong ba taint không được pod dung thứ.

Thông thường, nếu một taint với effect `NoExecute` được thêm vào node, thì bất kỳ pod nào
không dung thứ taint đó sẽ bị thu hồi ngay lập tức, còn các pod dung thứ
taint sẽ không bao giờ bị thu hồi. Tuy nhiên, một toleration với effect `NoExecute` có thể chỉ định
trường tùy chọn `tolerationSeconds` quy định pod sẽ ở lại gắn với node
trong bao lâu sau khi taint được thêm vào. Ví dụ,

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600
```

có nghĩa là nếu pod này đang chạy và một taint khớp được thêm vào node, thì
pod sẽ vẫn gắn với node trong 3600 giây, và sau đó bị thu hồi. Nếu
taint được gỡ bỏ trước thời điểm đó, pod sẽ không bị thu hồi.

## Toán tử so sánh số (Numeric comparison operators) {#numeric-comparison-operators}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [alpha]`

Ngoài `Equal` và `Exists`, bạn có thể dùng các toán tử so sánh số
(`Gt` và `Lt`) để khớp các taint có giá trị số nguyên. Điều này hữu ích cho việc lập lịch
dựa trên ngưỡng (threshold), chẳng hạn khớp node theo mức độ tin cậy hoặc bậc SLA.

* `Gt` khớp khi giá trị taint lớn hơn giá trị toleration.
* `Lt` khớp khi giá trị taint nhỏ hơn giá trị toleration.

Với các toán tử số, cả giá trị toleration lẫn giá trị taint đều phải là số nguyên hợp lệ.
Nếu một trong hai giá trị không thể phân tích (parse) thành số nguyên, toleration không khớp.

> **Ghi chú:**
> Khi bạn tạo một Pod dùng toán tử toleration `Gt` hoặc `Lt`, API server xác thực rằng
> các giá trị toleration là số nguyên hợp lệ. Giá trị taint trên node không được xác thực tại
> thời điểm đăng ký node. Nếu một node có giá trị taint không phải là số (ví dụ,
> `servicelevel.organization.example/agreed-service-level=high:NoSchedule`),
> các pod dùng toán tử so sánh số sẽ không khớp với taint đó và không thể lập lịch lên node đó.

Ví dụ, nếu các node được taint với một giá trị biểu thị thỏa thuận mức dịch vụ (service level agreement — SLA):

```shell
kubectl taint nodes node1 servicelevel.organization.example/agreed-service-level=950:NoSchedule
```

Một pod có thể dung thứ các node có SLA lớn hơn 900:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-numeric-toleration
  labels:
    env: test
spec:
  containers:
    - name: nginx
      image: nginx
      imagePullPolicy: IfNotPresent
  tolerations:
    - key: "servicelevel.organization.example/agreed-service-level"
      operator: "Gt"
      value: "900"
      effect: "NoSchedule"
```

Toleration này khớp với taint trên `node1` vì `950 > 900` (giá trị taint
lớn hơn giá trị toleration đối với toán tử `Gt`).
Tương tự, bạn có thể dùng toán tử `Lt` để khớp các taint trong đó giá trị taint
nhỏ hơn giá trị toleration:

```yaml
tolerations:
- key: "servicelevel.organization.example/agreed-service-level"
  operator: "Lt"
  value: "1000"
  effect: "NoSchedule"
```

> **Ghi chú:**
> Khi dùng các toán tử so sánh số:
>
> * Cả giá trị toleration lẫn giá trị taint đều phải là số nguyên có dấu 64-bit hợp lệ
>   (không cho phép số có chữ số 0 ở đầu (ví dụ "0550")).
> * Nếu một giá trị không thể phân tích thành số nguyên, toleration không khớp.
> * Các toán tử số hoạt động với mọi effect của taint: `NoSchedule`, `PreferNoSchedule`, và `NoExecute`.
> * Với `PreferNoSchedule` cùng toán tử số: nếu toleration của pod không thỏa mãn phép so sánh số
>   (ví dụ, giá trị taint < giá trị toleration khi dùng `Gt`), bộ lập lịch sẽ cho node đó độ ưu tiên thấp hơn
>   nhưng vẫn có thể lập lịch lên đó nếu không có lựa chọn nào tốt hơn.

> **Cảnh báo:**
>
> Trước khi tắt feature gate `TaintTolerationComparisonOperators`:
>
> * Bạn nên xác định tất cả các workload đang dùng toán tử `Gt` hoặc `Lt` để tránh việc controller bị lặp liên tục (hot-loop).
> * Cập nhật tất cả các template của workload controller để dùng toán tử `Equal` hoặc `Exists` thay thế
> * Xóa mọi pod đang chờ (pending) có dùng toán tử `Gt` hoặc `Lt`
> * Theo dõi metric `apiserver_request_total` để phát hiện đột biến về lỗi xác thực

## Các trường hợp sử dụng ví dụ (Example Use Cases)

Taint và toleration là một cách linh hoạt để lái các pod *tránh xa* các node hoặc thu hồi
những pod không nên chạy. Một vài trường hợp sử dụng là

* **Node chuyên dụng (Dedicated Nodes)**: Nếu bạn muốn dành riêng một tập node cho một nhóm
người dùng cụ thể sử dụng độc quyền, bạn có thể thêm một taint vào các node đó (ví dụ,
`kubectl taint nodes nodename dedicated=groupName:NoSchedule`) rồi thêm toleration
tương ứng vào các pod của họ (cách dễ nhất là viết một
[admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) tùy chỉnh).
Các pod có toleration khi đó sẽ được phép sử dụng các node bị taint (node chuyên dụng)
cũng như bất kỳ node nào khác trong cluster. Nếu bạn muốn dành riêng các node cho họ *và*
đảm bảo họ *chỉ* dùng các node chuyên dụng đó, thì bạn nên thêm một label tương tự
taint vào cùng tập node đó (ví dụ `dedicated=groupName`), và admission
controller nên bổ sung thêm một node affinity để yêu cầu các pod chỉ có thể lập lịch
lên các node có label `dedicated=groupName`.

* **Node có phần cứng đặc biệt (Nodes with Special Hardware)**: Trong một cluster mà một nhóm nhỏ node có
phần cứng chuyên biệt (ví dụ GPU), ta muốn giữ những pod không cần phần cứng
chuyên biệt tránh khỏi các node đó, nhờ vậy chừa chỗ cho các pod đến sau thực sự cần
phần cứng chuyên biệt. Điều này có thể thực hiện bằng cách taint các node có phần cứng
chuyên biệt (ví dụ `kubectl taint nodes nodename special=true:NoSchedule` hoặc
`kubectl taint nodes nodename special=true:PreferNoSchedule`) và thêm toleration
tương ứng vào các pod sử dụng phần cứng đặc biệt. Giống như trường hợp node chuyên dụng,
cách dễ nhất có lẽ là áp dụng các toleration bằng một
[admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) tùy chỉnh.
Ví dụ, khuyến nghị dùng [Tài nguyên mở rộng (Extended
Resources)](110-manage-resources-containers-vi.md#extended-resources)
để biểu diễn phần cứng đặc biệt, taint các node có phần cứng đặc biệt bằng
tên extended resource và chạy admission controller
[ExtendedResourceToleration](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#extendedresourcetoleration).
Bây giờ, vì các node đã bị taint, không pod nào thiếu toleration
sẽ được lập lịch lên chúng. Nhưng khi bạn gửi một pod yêu cầu
extended resource, admission controller `ExtendedResourceToleration` sẽ
tự động thêm toleration đúng vào pod và pod đó sẽ được lập lịch
lên các node có phần cứng đặc biệt. Điều này đảm bảo các node phần cứng
đặc biệt này được dành riêng cho các pod yêu cầu phần cứng như vậy và bạn không phải
tự tay thêm toleration cho các pod của mình.

* **Eviction dựa trên taint (Taint based Evictions)**: Hành vi thu hồi (eviction) có thể cấu hình theo từng pod
khi node gặp sự cố, được mô tả ở phần tiếp theo.

## Eviction dựa trên taint (Taint based Evictions)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.18 [stable]`

Node controller tự động taint một Node khi một số điều kiện nhất định
xảy ra. Các taint sau được tích hợp sẵn:

 * `node.kubernetes.io/not-ready`: Node chưa sẵn sàng. Tương ứng với
   NodeCondition `Ready` có giá trị "`False`".
 * `node.kubernetes.io/unreachable`: Node không thể truy cập được từ node
   controller. Tương ứng với NodeCondition `Ready` có giá trị "`Unknown`".
 * `node.kubernetes.io/memory-pressure`: Node bị áp lực bộ nhớ (memory pressure).
 * `node.kubernetes.io/disk-pressure`: Node bị áp lực đĩa (disk pressure).
 * `node.kubernetes.io/pid-pressure`: Node bị áp lực PID (PID pressure).
 * `node.kubernetes.io/network-unavailable`: Mạng của node không khả dụng.
 * `node.kubernetes.io/unschedulable`: Node không thể lập lịch được.
 * `node.cloudprovider.kubernetes.io/uninitialized`: Khi kubelet được khởi động
    với cloud provider "external", taint này được đặt trên node để đánh dấu
    node chưa sử dụng được. Sau khi một controller từ cloud-controller-manager khởi tạo
    node này, kubelet sẽ gỡ bỏ taint đó.

Trong trường hợp một node cần được rút (drain), node controller hoặc kubelet sẽ thêm các taint liên quan
với effect `NoExecute`. Effect này được thêm mặc định cho các taint
`node.kubernetes.io/not-ready` và `node.kubernetes.io/unreachable`.
Nếu điều kiện lỗi trở lại bình thường, kubelet hoặc node
controller có thể gỡ bỏ (các) taint liên quan.

Trong một số trường hợp khi node không thể truy cập được, API server không thể liên lạc
với kubelet trên node đó. Quyết định xóa các pod không thể được truyền đạt tới
kubelet cho đến khi liên lạc với API server được thiết lập lại. Trong thời gian đó,
các pod đã được lên lịch xóa có thể tiếp tục chạy trên node bị chia cắt (partitioned).

> **Ghi chú:**
> Control plane giới hạn tốc độ thêm taint mới vào các node. Việc giới hạn tốc độ này
> quản lý số lượng eviction được kích hoạt khi nhiều node trở nên không thể truy cập
> cùng lúc (ví dụ: khi có sự cố mạng).

Bạn có thể chỉ định `tolerationSeconds` cho một Pod để xác định Pod đó gắn
với một Node bị lỗi hoặc không phản hồi trong bao lâu.

Ví dụ, bạn có thể muốn giữ một ứng dụng có nhiều trạng thái cục bộ (local state)
gắn với node trong thời gian dài khi xảy ra chia cắt mạng (network partition), với hy vọng
rằng sự chia cắt sẽ phục hồi và nhờ đó tránh được việc thu hồi pod.
Toleration bạn đặt cho Pod đó có thể trông như sau:

```yaml
tolerations:
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 6000
```

> **Ghi chú:**
> Kubernetes tự động thêm toleration cho
> `node.kubernetes.io/not-ready` và `node.kubernetes.io/unreachable`
> với `tolerationSeconds=300`,
> trừ khi bạn, hoặc một controller, đặt các toleration đó một cách tường minh.
>
> Các toleration được tự động thêm này có nghĩa là Pod vẫn gắn với
> Node trong 5 phút sau khi một trong các sự cố này được phát hiện.

Các pod của [DaemonSet](./66-daemonset-vi.md) được tạo với
toleration `NoExecute` cho các taint sau mà không có `tolerationSeconds`:

  * `node.kubernetes.io/unreachable`
  * `node.kubernetes.io/not-ready`

Điều này đảm bảo các pod DaemonSet không bao giờ bị thu hồi do các sự cố này.

> **Ghi chú:**
> Node controller từng chịu trách nhiệm thêm taint vào node và thu hồi pod. Nhưng sau phiên bản 1.29,
> phần triển khai eviction dựa trên taint đã được tách khỏi node controller thành một thành phần
> riêng biệt và độc lập gọi là taint-eviction-controller. Người dùng có thể tùy chọn tắt eviction
> dựa trên taint bằng cách đặt `--controllers=-taint-eviction-controller` trong kube-controller-manager.

## Taint Node theo điều kiện (Taint Nodes by Condition) {#taint-nodes-by-condition}

Control plane, thông qua node controller,
tự động tạo các taint với effect `NoSchedule` cho
[các điều kiện của node (node conditions)](142-node-pressure-eviction-vi.md#node-conditions).

Bộ lập lịch kiểm tra taint, chứ không phải điều kiện của node, khi đưa ra quyết định
lập lịch. Điều này đảm bảo các điều kiện của node không ảnh hưởng trực tiếp đến việc lập lịch.
Ví dụ, nếu điều kiện node `DiskPressure` đang hoạt động, control plane
thêm taint `node.kubernetes.io/disk-pressure` và không lập lịch pod mới
lên node bị ảnh hưởng. Nếu điều kiện node `MemoryPressure` đang hoạt động,
control plane thêm taint `node.kubernetes.io/memory-pressure`.

Bạn có thể bỏ qua các điều kiện của node đối với các pod mới tạo bằng cách thêm các toleration
tương ứng cho Pod. Control plane cũng thêm toleration `node.kubernetes.io/memory-pressure`
cho các pod có lớp QoS (QoS class)
khác `BestEffort`. Lý do là Kubernetes coi các pod trong lớp QoS `Guaranteed`
hoặc `Burstable` (kể cả pod không đặt memory request) như thể chúng
có khả năng chịu được áp lực bộ nhớ, trong khi các pod `BestEffort` mới không được lập lịch
lên node bị ảnh hưởng.

DaemonSet controller tự động thêm các toleration `NoSchedule`
sau vào tất cả các daemon, để tránh DaemonSet bị hỏng.

  * `node.kubernetes.io/memory-pressure`
  * `node.kubernetes.io/disk-pressure`
  * `node.kubernetes.io/pid-pressure` (1.14 trở lên)
  * `node.kubernetes.io/unschedulable` (1.10 trở lên)
  * `node.kubernetes.io/network-unavailable` (*chỉ với host network*)

Việc thêm các toleration này đảm bảo khả năng tương thích ngược. Bạn cũng có thể thêm
các toleration tùy ý vào DaemonSet.

## Taint và toleration cho thiết bị (Device taints and tolerations)

Thay vì taint toàn bộ node, quản trị viên cũng có thể [taint từng thiết bị riêng lẻ](149-dynamic-resource-allocation-vi.md#device-taints-and-tolerations)
khi cluster sử dụng [cấp phát tài nguyên động (dynamic resource allocation)](149-dynamic-resource-allocation-vi.md)
để quản lý phần cứng đặc biệt. Ưu điểm là việc taint có thể nhắm chính xác đến phần cứng
bị lỗi hoặc cần bảo trì. Toleration cũng được hỗ trợ và có thể được chỉ định khi yêu cầu
thiết bị. Giống như taint, chúng áp dụng cho tất cả các pod dùng chung cùng một thiết bị đã được cấp phát.

## Tiếp theo (What's next)

* Đọc về [Eviction do áp lực node (Node-pressure Eviction)](142-node-pressure-eviction-vi.md)
  và cách bạn có thể cấu hình nó
* Đọc về [Độ ưu tiên của Pod (Pod Priority)](141-pod-priority-preemption-vi.md)
* Đọc về [taint và toleration cho thiết bị](149-dynamic-resource-allocation-vi.md#device-taints-and-tolerations)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 7:

1. Taint đặt trên đối tượng nào, toleration đặt trên đối tượng nào? Thêm toleration vào một
   Pod có làm Pod đó **được đưa** lên node bị taint không?
2. Trên cluster lab, `lab-k8s-master` mang một taint effect `NoSchedule`. Vì sao Pod bạn tạo bằng
   `kubectl run` không bao giờ lên node đó, và bạn phải làm gì nếu muốn một Pod cụ thể chạy
   được ở đó?
3. Một node đang chạy 5 Pod. Bạn chạy `kubectl taint nodes <node> key1=value1:NoSchedule`.
   Năm Pod đó có bị đuổi không? Câu trả lời đổi thế nào nếu effect là `NoExecute`?
4. Bạn rút mạng `lab-k8s-worker2` để tạo sự cố. Node controller đặt taint gì, các Pod thường trên
   đó còn ở lại bao lâu trước khi bị thu hồi, và Pod của DaemonSet thì sao?
5. Một node có ba taint. Pod của bạn có toleration khớp hai trong ba taint đó, taint còn lại
   có effect `NoSchedule`. Pod có được lập lịch lên node không? Nếu Pod **đã đang chạy** trên
   node từ trước khi taint thứ ba được thêm thì sao?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Taint trên node, toleration trên Pod (trong PodSpec).** **Không.** Bài nói rõ: toleration
   *cho phép* bộ lập lịch lập lịch Pod có taint khớp, nhưng **không đảm bảo việc lập lịch** —
   bộ lập lịch còn đánh giá các tham số khác. Toleration chỉ gỡ rào, không phải lực hút; muốn
   ép Pod về đúng tập node đó thì phải thêm label và `nodeSelector`/node affinity của bài
   [138](138-assign-pod-node-vi.md).
2. Vì effect `NoSchedule` nghĩa là **không Pod mới nào được lập lịch lên node bị taint trừ khi
   chúng có toleration khớp tương ứng** — Pod mặc định không có toleration nào nên bị loại
   ngay ở bước lọc, và chỉ còn hai worker là ứng viên. Muốn một Pod lên được `lab-k8s-master`, bạn
   thêm vào PodSpec một toleration **khớp cả `key` lẫn `effect`** của taint đó (`operator:
   Exists`, hoặc `Equal` với `value` bằng đúng giá trị taint). Nhưng nhớ câu 1: khi đó Pod chỉ
   *được phép* lên, bộ lập lịch vẫn có thể chọn worker.
3. **Không bị đuổi.** `NoSchedule` chỉ chặn Pod mới; bài ghi rõ "Các Pod hiện đang chạy trên
   node **không** bị thu hồi". Với `NoExecute` thì ngược lại: **các Pod không dung thứ taint
   bị thu hồi ngay lập tức**; Pod có toleration mà không đặt `tolerationSeconds` thì ở lại mãi
   mãi; Pod có `tolerationSeconds` thì ở lại đúng khoảng đó rồi bị node lifecycle controller
   thu hồi.
4. Node controller đặt taint **`node.kubernetes.io/unreachable`** (ứng với NodeCondition
   `Ready` = `Unknown`) với effect **`NoExecute`**. Pod thường ở lại **5 phút**, vì Kubernetes
   **tự động thêm toleration `tolerationSeconds=300`** cho `not-ready` và `unreachable` trừ khi
   bạn hoặc controller đặt tường minh. Pod của **DaemonSet không bị thu hồi**, vì chúng được
   tạo với toleration `NoExecute` cho hai taint này mà **không có** `tolerationSeconds`. Lưu ý
   thêm: khi node không liên lạc được, quyết định xóa Pod **không truyền được tới kubelet**,
   nên Pod có thể vẫn tiếp tục chạy trên node bị chia cắt.
5. **Không được lập lịch** — vẫn còn một taint không bị bỏ qua với effect `NoSchedule`, và chỉ
   cần một taint như vậy là Kubernetes không lập lịch Pod lên node. Nhưng nếu Pod **đã chạy từ
   trước**, nó **tiếp tục chạy**: `NoSchedule` không thu hồi Pod đang chạy, và taint thứ ba là
   taint duy nhất không được dung thứ.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

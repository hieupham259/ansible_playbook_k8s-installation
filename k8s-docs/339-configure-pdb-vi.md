# Chỉ định Disruption Budget cho ứng dụng của bạn (Specifying a Disruption Budget for your Application)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/run-application/configure-pdb/

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.21 [stable]`

Trang này hướng dẫn cách giới hạn số lượng gián đoạn (disruption) đồng thời mà ứng dụng của bạn phải chịu, cho phép đạt tính sẵn sàng (availability) cao hơn trong khi vẫn cho phép quản trị viên cluster quản lý các node của cluster.

## Trước khi bạn bắt đầu (Before you begin)

Kubernetes server của bạn phải ở phiên bản v1.21 trở lên. Để kiểm tra phiên bản, hãy chạy `kubectl version`.

- Bạn là chủ sở hữu của một ứng dụng đang chạy trên Kubernetes cluster và ứng dụng đó yêu cầu tính sẵn sàng cao (high availability).
- Bạn nên biết cách triển khai [Ứng dụng Stateless được nhân bản (Replicated Stateless Applications)](https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/) và/hoặc [Ứng dụng Stateful được nhân bản (Replicated Stateful Applications)](https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/).
- Bạn nên đã đọc về [Gián đoạn Pod (Pod Disruptions)](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/).
- Bạn nên xác nhận với chủ sở hữu cluster hoặc nhà cung cấp dịch vụ rằng họ tôn trọng Pod Disruption Budget.

## Bảo vệ ứng dụng bằng PodDisruptionBudget (Protecting an Application with a PodDisruptionBudget)

1. Xác định ứng dụng nào bạn muốn bảo vệ bằng PodDisruptionBudget (PDB).
1. Suy nghĩ về cách ứng dụng của bạn phản ứng với các gián đoạn.
1. Tạo định nghĩa PDB dưới dạng file YAML.
1. Tạo đối tượng PDB từ file YAML đó.

## Xác định ứng dụng cần bảo vệ (Identify an Application to Protect)

Trường hợp sử dụng phổ biến nhất là khi bạn muốn bảo vệ một ứng dụng được khai báo bởi một trong các controller có sẵn của Kubernetes:

- Deployment
- ReplicationController
- ReplicaSet
- StatefulSet

Trong trường hợp này, hãy ghi lại `.spec.selector` của controller; selector đó cũng chính là selector được đưa vào `.spec.selector` của PDB.

Từ phiên bản 1.15, PDB hỗ trợ các controller tùy chỉnh (custom controller) khi [scale subresource](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#scale-subresource) được bật.

Bạn cũng có thể dùng PDB với các Pod không được quản lý bởi một trong các controller kể trên, hoặc với các nhóm Pod tùy ý, nhưng có một số hạn chế, được mô tả trong mục [Workload tùy ý và selector tùy ý](#arbitrary-controllers-and-selectors).

## Suy nghĩ về cách ứng dụng của bạn phản ứng với gián đoạn (Think about how your application reacts to disruptions)

Hãy quyết định xem bao nhiêu instance có thể ngừng hoạt động cùng lúc trong một khoảng thời gian ngắn do một gián đoạn tự nguyện (voluntary disruption).

- Frontend stateless:
  - Mối quan tâm: không giảm năng lực phục vụ quá 10%.
    - Giải pháp: ví dụ, dùng PDB với minAvailable 90%.
- Ứng dụng Stateful một instance:
  - Mối quan tâm: không được tắt ứng dụng này mà chưa trao đổi với tôi.
    - Giải pháp khả dĩ 1: Không dùng PDB và chấp nhận downtime thỉnh thoảng xảy ra.
    - Giải pháp khả dĩ 2: Đặt PDB với maxUnavailable=0. Có một thỏa thuận (ngoài phạm vi Kubernetes) rằng người vận hành cluster cần hỏi ý kiến bạn trước khi tắt ứng dụng. Khi người vận hành cluster liên hệ với bạn, hãy chuẩn bị cho downtime, rồi xóa PDB để báo hiệu rằng đã sẵn sàng cho gián đoạn. Sau đó tạo lại PDB.
- Ứng dụng Stateful nhiều instance như Consul, ZooKeeper hoặc etcd:
  - Mối quan tâm: Không giảm số instance xuống dưới quorum, nếu không các thao tác ghi (write) sẽ thất bại.
    - Giải pháp khả dĩ 1: đặt maxUnavailable là 1 (hoạt động được với mọi quy mô của ứng dụng).
    - Giải pháp khả dĩ 2: đặt minAvailable bằng kích thước quorum (ví dụ: 3 khi quy mô là 5). (Cho phép nhiều gián đoạn xảy ra cùng lúc hơn).
- Batch Job có thể khởi động lại được:
  - Mối quan tâm: Job cần hoàn thành ngay cả khi có gián đoạn tự nguyện.
    - Giải pháp khả dĩ: Không tạo PDB. Job controller sẽ tạo Pod thay thế.

### Logic làm tròn khi chỉ định phần trăm (Rounding logic when specifying percentages)

Giá trị của `minAvailable` hoặc `maxUnavailable` có thể được biểu diễn dưới dạng số nguyên hoặc phần trăm.

- Khi bạn chỉ định một số nguyên, nó biểu thị số lượng Pod. Ví dụ, nếu bạn đặt `minAvailable` là 10 thì 10 Pod phải luôn ở trạng thái khả dụng (available), ngay cả trong khi có gián đoạn.
- Khi bạn chỉ định phần trăm bằng cách đặt giá trị dưới dạng chuỗi biểu diễn phần trăm (ví dụ `"50%"`), nó biểu thị phần trăm trên tổng số Pod. Ví dụ, nếu bạn đặt `minAvailable` là `"50%"`, thì ít nhất 50% số Pod phải còn khả dụng trong khi có gián đoạn.

Khi bạn chỉ định giá trị dưới dạng phần trăm, nó có thể không ánh xạ ra một số Pod chính xác. Ví dụ, nếu bạn có 7 Pod và đặt `minAvailable` là `"50%"`, thì không thể thấy ngay điều đó nghĩa là 3 Pod hay 4 Pod phải khả dụng. Kubernetes làm tròn lên số nguyên gần nhất, nên trong trường hợp này, 4 Pod phải khả dụng. Khi bạn chỉ định giá trị `maxUnavailable` dưới dạng phần trăm, Kubernetes làm tròn lên số Pod có thể bị gián đoạn. Do đó, một gián đoạn có thể vượt quá tỷ lệ phần trăm `maxUnavailable` mà bạn đã định nghĩa. Bạn có thể xem [mã nguồn](https://github.com/kubernetes/kubernetes/blob/23be9587a0f8677eb8091464098881df939c44a9/pkg/controller/disruption/disruption.go#L539) điều khiển hành vi này.

## Chỉ định một PodDisruptionBudget (Specifying a PodDisruptionBudget)

Một `PodDisruptionBudget` có ba trường:

- Một label selector `.spec.selector` để chỉ định tập hợp Pod mà nó áp dụng. Trường này là bắt buộc.
- `.spec.minAvailable` mô tả số Pod trong tập hợp đó vẫn phải khả dụng sau khi eviction (trục xuất) diễn ra, kể cả khi không tính Pod đã bị trục xuất. `minAvailable` có thể là một số tuyệt đối hoặc một tỷ lệ phần trăm.
- `.spec.maxUnavailable` (có từ Kubernetes 1.7 trở lên) mô tả số Pod trong tập hợp đó có thể không khả dụng sau khi eviction diễn ra. Nó có thể là một số tuyệt đối hoặc một tỷ lệ phần trăm.

> **Ghi chú:**
> Hành vi đối với selector rỗng khác nhau giữa API policy/v1beta1 và policy/v1 của PodDisruptionBudget. Với policy/v1beta1, selector rỗng không khớp với Pod nào, trong khi với policy/v1, selector rỗng khớp với mọi Pod trong namespace.

Bạn chỉ có thể chỉ định một trong hai trường `maxUnavailable` và `minAvailable` trong một `PodDisruptionBudget`. `maxUnavailable` chỉ có thể được dùng để kiểm soát việc trục xuất các Pod mà tất cả đều có cùng một controller quản lý. Trong các ví dụ dưới đây, "số replica mong muốn" (desired replicas) là giá trị `scale` của controller quản lý các Pod được chọn bởi `PodDisruptionBudget`.

Ví dụ 1: Với `minAvailable` là 5, các eviction được phép miễn là chúng để lại từ 5 Pod [khỏe mạnh (healthy)](#healthiness-of-a-pod) trở lên trong số các Pod được chọn bởi `selector` của PodDisruptionBudget.

Ví dụ 2: Với `minAvailable` là 30%, các eviction được phép miễn là ít nhất 30% số replica mong muốn ở trạng thái khỏe mạnh.

Ví dụ 3: Với `maxUnavailable` là 5, các eviction được phép miễn là có tối đa 5 replica không khỏe mạnh trong tổng số replica mong muốn.

Ví dụ 4: Với `maxUnavailable` là 30%, các eviction được phép miễn là số replica không khỏe mạnh không vượt quá 30% tổng số replica mong muốn, được làm tròn lên số nguyên gần nhất. Nếu tổng số replica mong muốn chỉ là 1, thì replica duy nhất đó vẫn được phép bị gián đoạn, dẫn tới mức không khả dụng thực tế là 100%.

Trong cách sử dụng điển hình, một budget duy nhất sẽ được dùng cho một tập hợp Pod được quản lý bởi một controller — ví dụ, các Pod trong một ReplicaSet hoặc StatefulSet.

> **Ghi chú:**
> Một disruption budget không thực sự đảm bảo rằng số lượng/tỷ lệ phần trăm Pod đã chỉ định sẽ luôn hoạt động. Ví dụ, một node đang chạy một Pod trong tập hợp có thể gặp sự cố khi tập hợp đang ở kích thước tối thiểu được chỉ định trong budget, khiến số Pod khả dụng của tập hợp tụt xuống dưới mức đã chỉ định. Budget chỉ có thể bảo vệ trước các eviction tự nguyện, chứ không phải mọi nguyên nhân gây ra tình trạng không khả dụng.

Nếu bạn đặt `maxUnavailable` là 0% hoặc 0, hoặc đặt `minAvailable` là 100% hoặc bằng số replica, tức là bạn đang yêu cầu không cho phép bất kỳ eviction tự nguyện nào. Khi bạn đặt mức không cho phép eviction tự nguyện cho một đối tượng workload như ReplicaSet, thì bạn không thể drain thành công một Node đang chạy một trong các Pod đó. Nếu bạn cố drain một Node nơi một Pod không thể bị trục xuất đang chạy, thao tác drain sẽ không bao giờ hoàn thành. Điều này là được phép theo đúng ngữ nghĩa của `PodDisruptionBudget`.

Bạn có thể xem các ví dụ về pod disruption budget được định nghĩa bên dưới. Chúng khớp với các Pod có label `app: zookeeper`.

Ví dụ PDB dùng minAvailable:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: zookeeper
```

Ví dụ PDB dùng maxUnavailable:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: zookeeper
```

Ví dụ, nếu đối tượng `zk-pdb` ở trên chọn các Pod của một StatefulSet có kích thước 3, thì cả hai cách khai báo có ý nghĩa hoàn toàn giống nhau. Việc dùng `maxUnavailable` được khuyến nghị vì nó tự động phản ứng với những thay đổi về số replica của controller tương ứng.

## Tạo đối tượng PDB (Create the PDB object)

Bạn có thể tạo hoặc cập nhật đối tượng PDB bằng kubectl.

```shell
kubectl apply -f mypdb.yaml
```

## Kiểm tra trạng thái của PDB (Check the status of the PDB)

Dùng kubectl để kiểm tra rằng PDB của bạn đã được tạo.

Giả sử bạn thực sự không có Pod nào khớp với `app: zookeeper` trong namespace của mình, bạn sẽ thấy kết quả tương tự như sau:

```shell
kubectl get poddisruptionbudgets
```
```
NAME     MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
zk-pdb   2               N/A               0                     7s
```

Nếu có các Pod khớp (giả sử là 3), thì bạn sẽ thấy kết quả tương tự như sau:

```shell
kubectl get poddisruptionbudgets
```
```
NAME     MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
zk-pdb   2               N/A               1                     7s
```

Giá trị khác 0 của `ALLOWED DISRUPTIONS` có nghĩa là disruption controller đã nhìn thấy các Pod, đã đếm số Pod khớp, và đã cập nhật trạng thái của PDB.

Bạn có thể xem thêm thông tin về trạng thái của một PDB bằng lệnh sau:

```shell
kubectl get poddisruptionbudgets zk-pdb -o yaml
```
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  annotations:
…
  creationTimestamp: "2020-03-04T04:22:56Z"
  generation: 1
  name: zk-pdb
…
status:
  currentHealthy: 3
  desiredHealthy: 2
  disruptionsAllowed: 1
  expectedPods: 3
  observedGeneration: 1
```

### Tính khỏe mạnh của một Pod (Healthiness of a Pod) {#healthiness-of-a-pod}

Cách hiện thực hiện tại coi các Pod khỏe mạnh là các Pod có mục `.status.conditions` với `type="Ready"` và `status="True"`. Các Pod này được theo dõi thông qua trường `.status.currentHealthy` trong trạng thái của PDB.

## Chính sách trục xuất Pod không khỏe mạnh (Unhealthy Pod Eviction Policy)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.31 [stable]`

PodDisruptionBudget bảo vệ một ứng dụng bằng cách đảm bảo rằng số Pod trong `.status.currentHealthy` không tụt xuống dưới con số được chỉ định trong `.status.desiredHealthy`, thông qua việc không cho phép trục xuất các Pod khỏe mạnh. Bằng cách dùng `.spec.unhealthyPodEvictionPolicy`, bạn cũng có thể định nghĩa tiêu chí xác định khi nào các Pod không khỏe mạnh nên được xem xét trục xuất. Hành vi mặc định khi không chỉ định chính sách nào tương ứng với chính sách `IfHealthyBudget`.

Các chính sách:

`IfHealthyBudget`
: Các Pod đang chạy (`.status.phase="Running"`) nhưng chưa khỏe mạnh chỉ có thể bị trục xuất nếu ứng dụng được bảo vệ không đang bị gián đoạn (`.status.currentHealthy` ít nhất bằng `.status.desiredHealthy`).

: Chính sách này đảm bảo rằng các Pod đang chạy của một ứng dụng vốn đã bị gián đoạn có cơ hội tốt nhất để trở nên khỏe mạnh. Điều này có hệ quả tiêu cực đối với việc drain node, vì thao tác drain có thể bị chặn bởi các ứng dụng hoạt động sai được bảo vệ bởi một PDB. Cụ thể hơn là các ứng dụng có Pod ở trạng thái `CrashLoopBackOff` (do bug hoặc do cấu hình sai), hoặc các Pod đơn giản là không báo cáo được condition `Ready`.

`AlwaysAllow`
: Các Pod đang chạy (`.status.phase="Running"`) nhưng chưa khỏe mạnh được coi là đã bị gián đoạn và có thể bị trục xuất bất kể các tiêu chí trong PDB có được thỏa mãn hay không.

: Điều này có nghĩa là các Pod đang chạy của một ứng dụng đang bị gián đoạn có thể không có cơ hội để trở nên khỏe mạnh. Bằng cách dùng chính sách này, người quản lý cluster có thể dễ dàng trục xuất các ứng dụng hoạt động sai được bảo vệ bởi một PDB. Cụ thể hơn là các ứng dụng có Pod ở trạng thái `CrashLoopBackOff` (do bug hoặc do cấu hình sai), hoặc các Pod đơn giản là không báo cáo được condition `Ready`.

> **Ghi chú:**
> Các Pod ở phase `Pending`, `Succeeded` hoặc `Failed` luôn được xem xét trục xuất.

## Workload tùy ý và selector tùy ý (Arbitrary workloads and arbitrary selectors) {#arbitrary-controllers-and-selectors}

Bạn có thể bỏ qua mục này nếu bạn chỉ dùng PDB với các workload resource có sẵn (Deployment, ReplicaSet, StatefulSet và ReplicationController) hoặc với custom resources có hiện thực `scale` [subresource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#advanced-features-and-flexibility), và khi selector của PDB khớp chính xác với selector của resource sở hữu Pod.

Bạn có thể dùng PDB với các Pod được quản lý bởi một resource khác, bởi một "operator", hoặc với các Pod trần (bare pods), nhưng với các hạn chế sau:

- chỉ có thể dùng `.spec.minAvailable`, không dùng được `.spec.maxUnavailable`.
- chỉ có thể dùng giá trị số nguyên với `.spec.minAvailable`, không dùng được phần trăm.

Không thể dùng các cấu hình về tính sẵn sàng khác, vì Kubernetes không thể suy ra tổng số Pod khi không có một resource sở hữu được hỗ trợ.

Bạn có thể dùng một selector chọn một tập con hoặc tập cha của các Pod thuộc về một workload resource. Eviction API sẽ không cho phép trục xuất bất kỳ Pod nào được bao phủ bởi nhiều PDB, vì vậy hầu hết người dùng sẽ muốn tránh các selector chồng lấn nhau. Một cách sử dụng hợp lý của các PDB chồng lấn là khi các Pod đang được chuyển từ PDB này sang PDB khác.

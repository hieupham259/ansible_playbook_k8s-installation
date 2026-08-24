# ReplicationController

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/>
>
> API cũ (legacy) để quản lý các workload có thể scale theo chiều ngang.
> Đã được thay thế bởi các API Deployment và ReplicaSet.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 4](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller), bài 14/14 ·
Kiểm chứng ở Lab 4 (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Lộ trình đánh dấu bài này là **tài liệu lịch sử**: ReplicationController là tiền thân của
ReplicaSet và **không dùng cho hệ thống mới**. Đọc nó với đúng một mục đích — **nhận diện
khi gặp cluster cũ**: một `kubectl get rc` ra kết quả, một manifest `apiVersion: v1` với
`kind: ReplicationController`, một quy trình rolling update viết tay trong runbook của người
đi trước. Đừng tạo `rc` trên cluster lab, và đừng học các mục thiết kế cuối bài. Ghi chú
ngay dưới đây đã nói thay tất cả: Deployment cấu hình một ReplicaSet mới là cách được khuyến
nghị. Vì là bài đọc để nhận diện, phần tự kiểm tra chỉ có ba câu.

**Phải hiểu ở lần đọc này:**

- ReplicationController làm **đúng việc** mà ReplicaSet làm: giữ số pod khớp label selector
  luôn hoạt động, chấm dứt pod thừa, khởi động thêm khi thiếu, và thay thế pod bị lỗi, bị
  xóa hay mất theo node. Viết tắt là **`rc`** trong `kubectl`.
- Khác biệt kỹ thuật duy nhất đáng nhớ: **ReplicationController không hỗ trợ selector dựa
  trên tập hợp (set-based)**. `.spec.selector` của nó là một map phẳng và
  `.spec.template.metadata.labels` phải **bằng** nó (không đặt selector thì mặc định lấy theo
  labels) — trong khi [ReplicaSet](64-replicaset-vi.md) có `matchLabels` và `matchExpressions`.
- **Rolling update ở đây là quy trình thủ công**: tạo một `rc` mới với 1 replica, rồi scale
  cái mới +1 và cái cũ -1 từng bước, rồi xóa cái cũ khi nó về 0. So sánh với một lệnh
  `kubectl set image` của [Deployment](63-deployment-vi.md) là thấy Deployment thay thế cái
  gì.
- Vì sao chỉ đọc chứ không dùng: mục *Các lựa chọn thay thế* trong chính bài xếp
  **Deployment là khuyến nghị**, và nói ReplicaSet là "ReplicationController thế hệ kế tiếp".

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Chạy một ReplicationController mẫu* — các khối output `kubectl describe` | chỉ minh họa; đừng tạo `rc` trên cluster mới | không cần |
| *Nhiều track phát hành* và *Dùng ReplicationController với Service* | cần Service; bản hiện đại của mẫu canary nằm ở bài [61](61-management-vi.md) | giai đoạn 5 |
| *Viết chương trình cho việc nhân bản* và *Trách nhiệm của ReplicationController* | là ghi chép thiết kế thời kỳ đầu, dẫn các issue GitHub cũ | không cần |

---

> **Ghi chú:**
> Một [`Deployment`](./63-deployment-vi.md) cấu hình một [`ReplicaSet`](./64-replicaset-vi.md)
> hiện là cách được khuyến nghị để thiết lập việc nhân bản (replication).

Một _ReplicationController_ đảm bảo rằng một số lượng pod bản sao (replica) được chỉ định
luôn chạy tại bất kỳ thời điểm nào. Nói cách khác, một ReplicationController bảo đảm rằng
một pod hoặc một tập hợp các pod đồng nhất luôn hoạt động và sẵn sàng.

## Cách một ReplicationController hoạt động (How a ReplicationController works)

Nếu có quá nhiều pod, ReplicationController sẽ chấm dứt (terminate) các pod thừa. Nếu có
quá ít, ReplicationController sẽ khởi động thêm pod. Không giống các pod được tạo thủ công,
các pod do một ReplicationController duy trì sẽ tự động được thay thế nếu chúng gặp lỗi,
bị xóa, hoặc bị chấm dứt. Ví dụ, các pod của bạn được tạo lại trên một node sau khi có bảo
trì gây gián đoạn như nâng cấp kernel. Vì lý do này, bạn nên dùng một ReplicationController
ngay cả khi ứng dụng của bạn chỉ cần một pod duy nhất. Một ReplicationController tương tự
như một trình giám sát tiến trình (process supervisor), nhưng thay vì giám sát các tiến
trình riêng lẻ trên một node duy nhất, ReplicationController giám sát nhiều pod trên nhiều
node.

ReplicationController thường được viết tắt là "rc" trong thảo luận, và cũng là tên rút gọn
trong các lệnh kubectl.

Một trường hợp đơn giản là tạo một object ReplicationController để chạy một cách tin cậy
một instance của một Pod vô thời hạn. Một trường hợp phức tạp hơn là chạy nhiều bản sao
giống hệt nhau của một dịch vụ được nhân bản, chẳng hạn như các web server.

## Chạy một ReplicationController mẫu (Running an example ReplicationController)

Cấu hình ReplicationController mẫu này chạy ba bản sao của web server nginx.

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    app: nginx
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

Chạy job mẫu bằng cách tải file ví dụ về rồi chạy lệnh này:

```shell
kubectl apply -f https://k8s.io/examples/controllers/replication.yaml
```

Output tương tự như sau:

```
replicationcontroller/nginx created
```

Kiểm tra trạng thái của ReplicationController bằng lệnh này:

```shell
kubectl describe replicationcontrollers/nginx
```

Output tương tự như sau:

```
Name:        nginx
Namespace:   default
Selector:    app=nginx
Labels:      app=nginx
Annotations:    <none>
Replicas:    3 current / 3 desired
Pods Status: 0 Running / 3 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:       app=nginx
  Containers:
   nginx:
    Image:              nginx
    Port:               80/TCP
    Environment:        <none>
    Mounts:             <none>
  Volumes:              <none>
Events:
  FirstSeen       LastSeen     Count    From                        SubobjectPath    Type      Reason              Message
  ---------       --------     -----    ----                        -------------    ----      ------              -------
  20s             20s          1        {replication-controller }                    Normal    SuccessfulCreate    Created pod: nginx-qrm3m
  20s             20s          1        {replication-controller }                    Normal    SuccessfulCreate    Created pod: nginx-3ntk0
  20s             20s          1        {replication-controller }                    Normal    SuccessfulCreate    Created pod: nginx-4ok8v
```

Ở đây, ba pod đã được tạo, nhưng chưa có pod nào chạy, có thể vì image đang được kéo về
(pull). Một lúc sau, cùng lệnh đó có thể hiển thị:

```shell
Pods Status:    3 Running / 0 Waiting / 0 Succeeded / 0 Failed
```

Để liệt kê tất cả các pod thuộc về ReplicationController dưới dạng máy đọc được
(machine readable), bạn có thể dùng một lệnh như sau:

```shell
pods=$(kubectl get pods --selector=app=nginx --output=jsonpath={.items..metadata.name})
echo $pods
```

Output tương tự như sau:

```
nginx-3ntk0 nginx-4ok8v nginx-qrm3m
```

Ở đây, selector giống với selector của ReplicationController (thấy trong output của
`kubectl describe`), và ở một dạng khác trong `replication.yaml`. Tùy chọn
`--output=jsonpath` chỉ định một biểu thức lấy tên từ mỗi pod trong danh sách trả về.

## Viết một ReplicationController Manifest (Writing a ReplicationController Manifest)

Giống như mọi cấu hình Kubernetes khác, một ReplicationController cần các trường
`apiVersion`, `kind`, và `metadata`.

Khi control plane tạo các Pod mới cho một ReplicationController, `.metadata.name` của
ReplicationController là một phần cơ sở để đặt tên cho các Pod đó. Tên của một
ReplicationController phải là một giá trị
[DNS subdomain](./17-names-vi.md)
hợp lệ, nhưng điều này có thể tạo ra kết quả không mong muốn đối với hostname của Pod.
Để có độ tương thích tốt nhất, tên nên tuân theo các quy tắc chặt chẽ hơn của một
[DNS label](./17-names-vi.md#dns-label-names).

Để biết thông tin chung về cách làm việc với các file cấu hình, xem
[quản lý object](./27-object-management-vi.md).

Một ReplicationController cũng cần một
[phần `.spec`](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status).

### Pod Template

`.spec.template` là trường bắt buộc duy nhất trong `.spec`.

`.spec.template` là một [pod template](./46-pods-vi.md#pod-template). Nó có schema y hệt
như một Pod, ngoại trừ việc nó được lồng bên trong và không có `apiVersion` hay `kind`.

Ngoài các trường bắt buộc của một Pod, một pod template trong ReplicationController phải
chỉ định các label thích hợp và một chính sách khởi động lại (restart policy) thích hợp.
Với các label, hãy đảm bảo không trùng lặp với các controller khác. Xem
[pod selector](#pod-selector).

Chỉ cho phép [`.spec.template.spec.restartPolicy`](./47-pod-lifecycle-vi.md#restart-policy)
bằng `Always`, đây cũng là giá trị mặc định nếu không được chỉ định.

Với việc khởi động lại container cục bộ, ReplicationController ủy quyền cho một agent trên
node, ví dụ như [Kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/).

### Label trên ReplicationController (Labels on the ReplicationController)

Bản thân ReplicationController có thể có các label (`.metadata.labels`). Thông thường, bạn
sẽ đặt chúng giống với `.spec.template.metadata.labels`; nếu `.metadata.labels` không được
chỉ định thì nó mặc định bằng `.spec.template.metadata.labels`. Tuy nhiên, chúng được phép
khác nhau, và `.metadata.labels` không ảnh hưởng đến hành vi của ReplicationController.

### Pod Selector {#pod-selector}

Trường `.spec.selector` là một [label selector](./18-labels-vi.md#label-selectors). Một
ReplicationController quản lý tất cả các pod có label khớp với selector đó. Nó không phân
biệt giữa các pod do chính nó tạo hoặc xóa và các pod do người khác hay tiến trình khác tạo
hoặc xóa. Điều này cho phép thay thế ReplicationController mà không ảnh hưởng đến các pod
đang chạy.

Nếu được chỉ định, `.spec.template.metadata.labels` phải bằng với `.spec.selector`, nếu
không sẽ bị API từ chối. Nếu `.spec.selector` không được chỉ định, nó sẽ mặc định bằng
`.spec.template.metadata.labels`.

Ngoài ra, thông thường bạn không nên tạo bất kỳ pod nào có label khớp với selector này, dù
là tạo trực tiếp, bằng một ReplicationController khác, hay bằng một controller khác như
Job. Nếu bạn làm vậy, ReplicationController sẽ nghĩ rằng chính nó đã tạo ra các pod đó.
Kubernetes không ngăn bạn làm điều này.

Nếu rốt cuộc bạn có nhiều controller với các selector chồng lấn nhau, bạn sẽ phải tự mình
quản lý việc xóa (xem [bên dưới](#working-with-replicationcontrollers)).

### Nhiều replica (Multiple Replicas)

Bạn có thể chỉ định bao nhiêu pod nên chạy đồng thời bằng cách đặt `.spec.replicas` thành
số lượng pod bạn muốn chạy đồng thời. Số lượng đang chạy tại một thời điểm bất kỳ có thể
cao hơn hoặc thấp hơn, chẳng hạn khi số replica vừa được tăng hoặc giảm, hoặc khi một pod
đang được tắt một cách êm thấm (graceful shutdown) và pod thay thế khởi động sớm.

Nếu bạn không chỉ định `.spec.replicas`, thì nó mặc định là 1.

## Làm việc với ReplicationController (Working with ReplicationControllers) {#working-with-replicationcontrollers}

### Xóa một ReplicationController và các Pod của nó (Deleting a ReplicationController and its Pods)

Để xóa một ReplicationController và tất cả các pod của nó, dùng
[`kubectl delete`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#delete).
Kubectl sẽ scale ReplicationController về không và chờ nó xóa từng pod trước khi xóa chính
ReplicationController. Nếu lệnh kubectl này bị gián đoạn, nó có thể được chạy lại.

Khi dùng REST API hoặc [thư viện client](https://kubernetes.io/docs/reference/using-api/client-libraries),
bạn cần thực hiện các bước một cách tường minh (scale số replica về 0, chờ các pod bị xóa,
rồi xóa ReplicationController).

### Chỉ xóa ReplicationController (Deleting only a ReplicationController)

Bạn có thể xóa một ReplicationController mà không ảnh hưởng đến bất kỳ pod nào của nó.

Với kubectl, chỉ định tùy chọn `--cascade=orphan` cho
[`kubectl delete`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#delete).

Khi dùng REST API hoặc [thư viện client](https://kubernetes.io/docs/reference/using-api/client-libraries),
bạn có thể xóa object ReplicationController.

Sau khi bản gốc bị xóa, bạn có thể tạo một ReplicationController mới để thay thế. Miễn là
`.spec.selector` cũ và mới giống nhau, ReplicationController mới sẽ nhận nuôi (adopt) các
pod cũ. Tuy nhiên, nó sẽ không nỗ lực làm cho các pod hiện có khớp với một pod template
mới, khác đi. Để cập nhật các pod theo một spec mới một cách có kiểm soát, hãy dùng
[rolling update](#rolling-updates).

### Tách pod khỏi một ReplicationController (Isolating pods from a ReplicationController)

Các pod có thể bị loại khỏi tập đích (target set) của một ReplicationController bằng cách
thay đổi label của chúng. Kỹ thuật này có thể được dùng để rút các pod ra khỏi dịch vụ
nhằm mục đích gỡ lỗi (debug) và khôi phục dữ liệu. Các pod bị loại bỏ theo cách này sẽ
được tự động thay thế (giả định rằng số lượng replica không bị thay đổi theo).

## Các mẫu sử dụng phổ biến (Common usage patterns)

### Lập lịch lại (Rescheduling)

Như đã đề cập ở trên, cho dù bạn có 1 pod muốn duy trì hoạt động hay 1000 pod, một
ReplicationController sẽ đảm bảo số lượng pod được chỉ định luôn tồn tại, ngay cả khi có
sự cố node hoặc pod bị chấm dứt (ví dụ, do hành động của một tác nhân điều khiển khác).

### Co giãn (Scaling)

ReplicationController cho phép scale số lượng replica lên hoặc xuống, theo cách thủ công
hoặc bằng một tác nhân điều khiển tự động co giãn (auto-scaling), bằng cách cập nhật
trường `replicas`.

### Rolling update (Rolling updates) {#rolling-updates}

ReplicationController được thiết kế để hỗ trợ cập nhật cuốn chiếu (rolling update) cho một
dịch vụ bằng cách thay thế các pod từng cái một.

Như đã giải thích trong [#1353](https://issue.k8s.io/1353), cách tiếp cận được khuyến nghị
là tạo một ReplicationController mới với 1 replica, scale controller mới (+1) và cũ (-1)
từng bước một, rồi xóa controller cũ sau khi nó đạt 0 replica. Cách này cập nhật tập các
pod một cách dễ dự đoán bất kể có các lỗi không mong muốn.

Lý tưởng nhất, controller thực hiện rolling update sẽ tính đến độ sẵn sàng (readiness) của
ứng dụng, và đảm bảo rằng luôn có đủ số lượng pod đang phục vụ hiệu quả tại bất kỳ thời
điểm nào.

Hai ReplicationController cần tạo các pod với ít nhất một label khác biệt, chẳng hạn như
image tag của container chính trong pod, vì thông thường chính việc cập nhật image là lý
do thúc đẩy rolling update.

### Nhiều track phát hành (Multiple release tracks)

Ngoài việc chạy nhiều bản phát hành (release) của một ứng dụng trong khi rolling update
đang diễn ra, việc chạy nhiều bản phát hành trong một khoảng thời gian dài, hoặc thậm chí
liên tục, bằng nhiều track phát hành cũng rất phổ biến. Các track được phân biệt bằng
label.

Chẳng hạn, một service có thể nhắm đến tất cả các pod có `tier in (frontend), environment in (prod)`.
Giả sử bạn có 10 pod nhân bản tạo nên tier này. Nhưng bạn muốn có thể 'canary' (thử nghiệm
canary) một phiên bản mới của thành phần này. Bạn có thể thiết lập một ReplicationController
với `replicas` đặt là 9 cho phần lớn các replica, với các label
`tier=frontend, environment=prod, track=stable`, và một ReplicationController khác với
`replicas` đặt là 1 cho bản canary, với các label
`tier=frontend, environment=prod, track=canary`. Bây giờ service bao phủ cả các pod canary
và không phải canary. Nhưng bạn có thể thao tác riêng trên từng ReplicationController để
thử nghiệm, theo dõi kết quả, v.v.

### Dùng ReplicationController với Service (Using ReplicationControllers with Services)

Nhiều ReplicationController có thể đứng sau một service duy nhất, để chẳng hạn, một phần
lưu lượng đi đến phiên bản cũ, và một phần đi đến phiên bản mới.

Một ReplicationController sẽ không bao giờ tự chấm dứt, nhưng nó không được kỳ vọng tồn
tại lâu như các service. Các service có thể được cấu thành từ các pod do nhiều
ReplicationController điều khiển, và nhiều ReplicationController có thể được tạo ra và hủy
đi trong suốt vòng đời của một service (ví dụ, để thực hiện cập nhật các pod chạy service
đó). Cả bản thân các service lẫn các client của chúng đều không cần quan tâm đến các
ReplicationController đang duy trì các pod của service.

## Viết chương trình cho việc nhân bản (Writing programs for Replication)

Các pod do một ReplicationController tạo ra được thiết kế để có thể hoán đổi cho nhau
(fungible) và giống hệt nhau về mặt ngữ nghĩa, mặc dù cấu hình của chúng có thể trở nên
không đồng nhất theo thời gian. Điều này rõ ràng phù hợp với các server stateless được
nhân bản, nhưng ReplicationController cũng có thể được dùng để duy trì tính sẵn sàng của
các ứng dụng kiểu bầu chọn master (master-elected), phân mảnh (sharded), và worker-pool.
Những ứng dụng như vậy nên dùng các cơ chế phân công việc động, chẳng hạn như
[hàng đợi công việc RabbitMQ](https://www.rabbitmq.com/tutorials/tutorial-two-python.html),
thay vì tùy biến tĩnh/một lần cấu hình của từng pod — cách làm bị coi là một anti-pattern.
Mọi việc tùy biến pod, chẳng hạn như tự động điều chỉnh tài nguyên theo chiều dọc (ví dụ,
cpu hay memory), nên được thực hiện bởi một tiến trình controller trực tuyến khác, tương
tự như chính ReplicationController.

## Trách nhiệm của ReplicationController (Responsibilities of the ReplicationController)

ReplicationController đảm bảo rằng số lượng pod mong muốn khớp với label selector của nó
và các pod đó đang hoạt động. Hiện tại, chỉ các pod đã chấm dứt bị loại khỏi phép đếm của
nó. Trong tương lai, [readiness](https://issue.k8s.io/620) và các thông tin khác có sẵn từ
hệ thống có thể được tính đến, chúng tôi có thể thêm nhiều quyền kiểm soát hơn đối với
chính sách thay thế, và chúng tôi dự định phát ra các sự kiện (event) mà các client bên
ngoài có thể dùng để triển khai các chính sách thay thế và/hoặc scale-down phức tạp tùy ý.

ReplicationController mãi mãi bị giới hạn trong phạm vi trách nhiệm hẹp này. Bản thân nó
sẽ không thực hiện các phép thăm dò readiness hay liveness. Thay vì thực hiện auto-scaling,
nó được thiết kế để được điều khiển bởi một bộ auto-scaler bên ngoài (như thảo luận trong
[#492](https://issue.k8s.io/492)), bộ này sẽ thay đổi trường `replicas` của nó. Chúng tôi
sẽ không thêm các chính sách lập lịch (ví dụ,
[phân tán — spreading](https://issue.k8s.io/367#issuecomment-48428019)) vào
ReplicationController. Nó cũng không nên kiểm tra xem các pod được điều khiển có khớp với
template hiện được chỉ định hay không, vì điều đó sẽ cản trở việc tự động điều chỉnh kích
thước và các quy trình tự động khác. Tương tự, các thời hạn hoàn thành (completion
deadline), các phụ thuộc thứ tự, việc mở rộng cấu hình, và các tính năng khác thuộc về nơi
khác. Chúng tôi thậm chí dự định tách riêng cơ chế tạo pod hàng loạt
([#170](https://issue.k8s.io/170)).

ReplicationController được thiết kế để trở thành một khối dựng nguyên thủy có thể kết hợp
(composable building-block primitive). Chúng tôi kỳ vọng các API và/hoặc công cụ cấp cao
hơn sẽ được xây dựng trên nó và các thành phần nguyên thủy bổ trợ khác để thuận tiện cho
người dùng trong tương lai. Các thao tác "macro" hiện được kubectl hỗ trợ (run, scale) là
các ví dụ minh chứng cho khái niệm này. Chẳng hạn, chúng ta có thể hình dung một thứ như
[Asgard](https://netflixtechblog.com/asgard-web-based-cloud-management-and-deployment-2c9fc4e4d3a1)
quản lý các ReplicationController, auto-scaler, service, chính sách lập lịch, canary, v.v.

## API Object

Replication controller là một resource cấp cao nhất (top-level) trong Kubernetes REST API.
Thông tin chi tiết hơn về API object có thể xem tại:
[ReplicationController API object](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#replicationcontroller-v1-core).

## Các lựa chọn thay thế cho ReplicationController (Alternatives to ReplicationController)

### ReplicaSet

[`ReplicaSet`](./64-replicaset-vi.md) là ReplicationController thế hệ kế tiếp, hỗ trợ
[selector dựa trên tập hợp (set-based)](./18-labels-vi.md#set-based-requirement) mới.
Nó chủ yếu được [Deployment](./63-deployment-vi.md) dùng như một cơ chế để điều phối việc
tạo, xóa và cập nhật pod.
Lưu ý rằng chúng tôi khuyến nghị dùng Deployment thay vì trực tiếp dùng Replica Set, trừ
khi bạn cần cơ chế điều phối cập nhật tùy biến hoặc hoàn toàn không cần cập nhật.

### Deployment (Khuyến nghị) (Deployment (Recommended))

[`Deployment`](./63-deployment-vi.md) là một API object cấp cao hơn, thực hiện cập nhật
các Replica Set bên dưới nó và các Pod của chúng. Deployment được khuyến nghị nếu bạn muốn
chức năng rolling update, vì chúng có tính khai báo (declarative), hoạt động phía server,
và có thêm nhiều tính năng khác.

### Pod trần (Bare Pods)

Không giống trường hợp người dùng trực tiếp tạo pod, một ReplicationController thay thế
các pod bị xóa hoặc bị chấm dứt vì bất kỳ lý do gì, chẳng hạn trong trường hợp node gặp sự
cố hoặc node bị bảo trì gây gián đoạn, như nâng cấp kernel. Vì lý do này, chúng tôi khuyến
nghị bạn dùng một ReplicationController ngay cả khi ứng dụng của bạn chỉ cần một pod duy
nhất. Hãy hình dung nó tương tự như một trình giám sát tiến trình, chỉ khác là nó giám sát
nhiều pod trên nhiều node thay vì các tiến trình riêng lẻ trên một node duy nhất. Một
ReplicationController ủy quyền việc khởi động lại container cục bộ cho một agent nào đó
trên node, chẳng hạn như kubelet.

### Job

Dùng một [`Job`](67-job-vi.md) thay cho
ReplicationController đối với các pod được kỳ vọng sẽ tự chấm dứt (tức là các batch job).

### DaemonSet

Dùng một [`DaemonSet`](./66-daemonset-vi.md) thay cho ReplicationController đối với các
pod cung cấp chức năng ở cấp máy (machine-level), chẳng hạn như giám sát máy hoặc ghi log
của máy. Các pod này có vòng đời gắn liền với vòng đời của máy: pod cần chạy trên máy
trước khi các pod khác khởi động, và an toàn để chấm dứt khi máy đã sẵn sàng được khởi
động lại/tắt.

## Tiếp theo (What's next)

* Tìm hiểu về [Pod](./46-pods-vi.md).
* Tìm hiểu về [Deployment](./63-deployment-vi.md), thứ thay thế cho ReplicationController.
* `ReplicationController` là một phần của Kubernetes REST API.
  Đọc định nghĩa object
  [ReplicationController](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/replication-controller-v1/)
  để hiểu API dành cho replication controller.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 4. Bài này chỉ
đọc để nhận diện, nên chỉ có ba câu:

1. **Câu bẫy.** Bài [64](64-replicaset-vi.md) nói ReplicaSet là thế hệ kế nhiệm của
   ReplicationController. Khác biệt kỹ thuật giữa hai cái là gì — và điều gì **không** khác?
2. Trên cluster lab của bạn (Kubernetes v1.35.6, 2 worker), bạn tiếp quản một manifest
   `apiVersion: v1` / `kind: ReplicationController` đang chạy 3 pod và cần đổi container
   image. Bài này hướng dẫn làm thế nào, và bạn nên làm gì thay vào đó?
3. Nếu ReplicationController và ReplicaSet làm cùng một việc, vì sao lộ trình vẫn giữ bài này
   thay vì bỏ hẳn?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Khác biệt duy nhất: **ReplicationController không hỗ trợ các yêu cầu selector dựa trên tập
   hợp (set-based)**, thứ mà ReplicaSet có qua `matchExpressions`. Kèm theo đó là hệ quả về
   cú pháp: `.spec.selector` của `rc` là một map phẳng và `.spec.template.metadata.labels`
   phải **bằng** nó, còn ReplicaSet dùng `matchLabels`. Thứ **không** khác chính là phần lớn:
   bài 64 nói "cả hai phục vụ cùng một mục đích và hành xử tương tự nhau" — cùng giữ số pod
   khớp selector, cùng thay thế pod mất, cùng ủy quyền việc khởi động lại container cục bộ
   cho kubelet, cùng cho `--cascade=orphan` để giữ pod lại, cùng tách pod ra bằng cách đổi
   label. Trực giác "thế hệ kế nhiệm nghĩa là viết lại toàn bộ" là sai; khoảng cách thực sự
   lớn nằm ở tầng trên — Deployment — chứ không nằm giữa `rc` và `rs`.
2. Bài hướng dẫn quy trình **thủ công**: tạo một ReplicationController **mới** với 1 replica,
   scale controller mới **+1** và controller cũ **-1** từng bước một, rồi xóa controller cũ
   sau khi nó về 0 replica; hai `rc` phải tạo pod với **ít nhất một label khác biệt**, thường
   là image tag. Việc bạn nên làm thay vào đó: **chuyển sang Deployment** — chính bài nói
   Deployment "được khuyến nghị nếu bạn muốn chức năng rolling update, vì chúng có tính khai
   báo, hoạt động phía server, và có thêm nhiều tính năng khác". Sau đó một lệnh
   `kubectl set image` là đủ, kèm `kubectl rollout status` và `kubectl rollout undo`.
3. Vì bạn vẫn có thể **gặp** nó. API `v1` ReplicationController chưa bị gỡ, nên trên một
   cluster được kế thừa, `kubectl get rc` có thể ra kết quả và manifest cũ vẫn apply được.
   Mục tiêu của lần đọc này là **nhận ra nó và biết nó tương đương ReplicaSet**, để bạn không
   mất thời gian đoán nó là gì và biết ngay hướng xử lý là di trú sang Deployment. Ghi chú
   ngay đầu bài đã chốt: một Deployment cấu hình một ReplicaSet là cách được khuyến nghị hiện
   nay để thiết lập việc nhân bản.

</details>

Bạn đã đọc xong toàn bộ 14 bài của giai đoạn 4. Câu nào chưa trả lời được thì quay lại đúng
mục tương ứng, sau đó chuyển sang **Lab 4 — Workload controller** (chưa viết, xem
[bản đồ lab](labs/README.md#4-bản-đồ-lab)).

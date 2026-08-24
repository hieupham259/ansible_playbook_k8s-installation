# Deployment (Deployments)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>
>
> Deployment quản lý một tập các Pod để chạy một workload ứng dụng, thường là workload không cần lưu giữ trạng thái (state).

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 4](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller), bài 3/14 ·
Kiểm chứng ở Lab 4 (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Đây là **bài dài nhất bộ tài liệu**. Phần lớn độ dài đến từ các khối output mẫu của
`kubectl describe` và `kubectl get rs` — chúng là minh họa, không phải quy tắc. Đừng đọc
tuần tự từ đầu tới cuối như một cuốn sách: đọc lấy cơ chế theo năm gạch đầu dòng dưới đây,
còn output thì lướt để biết nhìn vào cột nào. Bạn sẽ tạo output thật của chính mình ở Lab 4.
Bài này giả định bạn đã nắm [ReplicaSet](64-replicaset-vi.md) — nếu chưa, quay lại bài đó.

**Phải hiểu ở lần đọc này:**

- Deployment **không** trực tiếp quản Pod: nó tạo và điều khiển các ReplicaSet, mỗi Pod
  template sinh một ReplicaSet riêng, phân biệt nhau bằng nhãn `pod-template-hash` băm từ
  chính template đó. Rollout chỉ là scale ReplicaSet mới lên và ReplicaSet cũ về 0.
- Rollout — và revision — **chỉ** được kích hoạt khi `.spec.template` thay đổi. Scale không
  tạo revision mới và không tạo ReplicaSet mới; rollback vì thế chỉ hoàn nguyên phần Pod
  template (ghi chú ở đầu mục *Cập nhật một Deployment* và *Rollback một Deployment*).
- Hai núm điều khiển của `RollingUpdate`: `maxUnavailable` (mặc định 25%, làm tròn xuống) và
  `maxSurge` (mặc định 25%, làm tròn lên) — chúng quy định khoảng dao động của tổng số Pod
  trong lúc cập nhật. `Recreate` thì kill toàn bộ Pod cũ trước khi tạo Pod mới.
- Lịch sử revision **được lưu trong chính các ReplicaSet cũ**; `.spec.revisionHistoryLimit`
  mặc định giữ 10 ReplicaSet cũ, xóa ReplicaSet nào là mất khả năng rollback về revision đó,
  và đặt bằng 0 là mất hẳn khả năng rollback.
- Ba trạng thái *Progressing* / *Complete* / *Failed* và `.spec.progressDeadlineSeconds`
  (mặc định 600): khi vượt deadline, Kubernetes **chỉ đặt condition** `ProgressDeadlineExceeded`
  và không làm gì thêm — nó **không tự rollback**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Các khối output `kubectl describe deployment` và `kubectl get rs` | là minh họa, không phải quy tắc | Lab 4 sinh output thật |
| *Cập nhật label selector* | selector bất biến sau khi tạo, tình huống hiếm và phá hủy | không cần |
| *Scale theo tỷ lệ* | chỉ xảy ra khi một autoscaler scale giữa chừng một rollout | bài [72](72-horizontal-pod-autoscale-vi.md) |
| *Deployment kiểu canary* | cần nhiều Deployment cùng đứng sau một Service | bài [61](61-management-vi.md) và giai đoạn 5 |
| *Pod đang kết thúc* và `.status.terminatingReplicas` | cần feature gate `DeploymentReplicaSetTerminatingReplicas` | không cần |
| Ví dụ `kubectl autoscale deployment/nginx-deployment` trong *Scale một Deployment* | cần metrics-server | bài [72](72-horizontal-pod-autoscale-vi.md), thực hành ở Lab 11b |

---

_Deployment_ cung cấp cơ chế cập nhật khai báo (declarative updates) cho Pod và ReplicaSet.

Bạn mô tả _trạng thái mong muốn_ (desired state) trong một Deployment, và controller của Deployment sẽ thay đổi trạng thái thực tế về trạng thái mong muốn theo một tốc độ có kiểm soát. Bạn có thể định nghĩa các Deployment để tạo ReplicaSet mới, hoặc để xóa các Deployment hiện có và tiếp nhận toàn bộ tài nguyên của chúng bằng các Deployment mới.

> **Ghi chú:**
> Không tự tay quản lý các ReplicaSet thuộc sở hữu của một Deployment. Hãy cân nhắc mở issue trong repository chính của Kubernetes nếu trường hợp sử dụng của bạn chưa được đề cập bên dưới.

## Trường hợp sử dụng (Use Case) {#use-case}

Sau đây là những trường hợp sử dụng điển hình của Deployment:

* [Tạo một Deployment để rollout một ReplicaSet](#creating-a-deployment). ReplicaSet tạo các Pod ở chế độ nền. Kiểm tra trạng thái của đợt triển khai (rollout) để xem nó thành công hay không.
* [Khai báo trạng thái mới của các Pod](#updating-a-deployment) bằng cách cập nhật PodTemplateSpec của Deployment. Một ReplicaSet mới được tạo, và Deployment từ từ scale up ReplicaSet mới đồng thời scale down ReplicaSet cũ, bảo đảm các Pod được thay thế theo tốc độ có kiểm soát. Mỗi ReplicaSet mới sẽ cập nhật revision (bản sửa đổi) của Deployment.
* [Rollback (quay lui) về một revision trước đó của Deployment](#rolling-back-a-deployment) nếu trạng thái hiện tại của Deployment không ổn định. Mỗi lần rollback cũng cập nhật revision của Deployment.
* [Scale up Deployment để đáp ứng tải lớn hơn](#scaling-a-deployment).
* [Tạm dừng rollout của một Deployment](#pausing-and-resuming-a-deployment) để áp dụng nhiều thay đổi vào PodTemplateSpec của nó, rồi tiếp tục (resume) để bắt đầu một rollout mới.
* [Dùng trạng thái của Deployment](#deployment-status) như một dấu hiệu cho biết một rollout đang bị kẹt.
* [Dọn dẹp các ReplicaSet cũ](#clean-up-policy) mà bạn không cần nữa.

## Tạo một Deployment (Creating a Deployment) {#creating-a-deployment}

Sau đây là một ví dụ về Deployment. Nó tạo một ReplicaSet để chạy lên ba Pod `nginx`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

Trong ví dụ này:

* Một Deployment tên là `nginx-deployment` được tạo, thể hiện qua trường
  `.metadata.name`. Tên này sẽ trở thành cơ sở đặt tên cho các ReplicaSet
  và Pod được tạo sau đó. Xem [Viết một Deployment Spec](#writing-a-deployment-spec)
  để biết thêm chi tiết.
* Deployment tạo một ReplicaSet, ReplicaSet này tạo ba Pod được nhân bản, thể hiện qua trường `.spec.replicas`.
* Trường `.spec.selector` định nghĩa cách ReplicaSet được tạo ra tìm những Pod cần quản lý.
  Trong trường hợp này, bạn chọn một nhãn (label) được định nghĩa trong Pod template (`app: nginx`).
  Tuy nhiên, các quy tắc chọn phức tạp hơn cũng khả thi,
  miễn là bản thân Pod template thỏa mãn quy tắc đó.

  > **Ghi chú:**
  > Trường `.spec.selector.matchLabels` là một map gồm các cặp {key,value}.
  > Một cặp {key,value} đơn lẻ trong map `matchLabels` tương đương với một phần tử của `matchExpressions`,
  > trong đó trường `key` là "key", `operator` là "In", và mảng `values` chỉ chứa "value".
  > Tất cả các yêu cầu, từ cả `matchLabels` lẫn `matchExpressions`, phải được thỏa mãn thì mới khớp.

* Trường `.spec.template` chứa các trường con sau:
  * Các Pod được gắn nhãn `app: nginx` thông qua trường `.metadata.labels`.
  * Đặc tả của Pod template, tức trường `.spec`, cho biết các Pod
    chạy một container, `nginx`, sử dụng image `nginx`
    trên [Docker Hub](https://hub.docker.com/) ở phiên bản 1.14.2.
  * Tạo một container và đặt tên là `nginx` thông qua trường `.spec.containers[0].name`.

Trước khi bắt đầu, hãy chắc chắn cluster Kubernetes của bạn đang chạy.
Làm theo các bước dưới đây để tạo Deployment trên:

1. Tạo Deployment bằng cách chạy lệnh sau:

   ```shell
   kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml
   ```

2. Chạy `kubectl get deployments` để kiểm tra Deployment đã được tạo hay chưa.

   Nếu Deployment vẫn đang trong quá trình tạo, kết quả sẽ tương tự như sau:
   ```
   NAME               READY   UP-TO-DATE   AVAILABLE   AGE
   nginx-deployment   0/3     0            0           1s
   ```
   Khi bạn xem các Deployment trong cluster, các trường sau được hiển thị:
   * `NAME` liệt kê tên các Deployment trong namespace.
   * `READY` hiển thị bao nhiêu bản sao (replica) của ứng dụng đang sẵn sàng phục vụ người dùng. Nó theo mẫu ready/desired (sẵn sàng/mong muốn).
   * `UP-TO-DATE` hiển thị số replica đã được cập nhật để đạt trạng thái mong muốn.
   * `AVAILABLE` hiển thị bao nhiêu replica của ứng dụng đang sẵn sàng phục vụ người dùng.
   * `AGE` hiển thị khoảng thời gian ứng dụng đã chạy.

   Lưu ý rằng số replica mong muốn là 3 theo trường `.spec.replicas`.

3. Để xem trạng thái rollout của Deployment, chạy `kubectl rollout status deployment/nginx-deployment`.

   Kết quả tương tự:
   ```
   Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
   deployment "nginx-deployment" successfully rolled out
   ```

4. Chạy lại `kubectl get deployments` sau đó vài giây.
   Kết quả tương tự như sau:
   ```
   NAME               READY   UP-TO-DATE   AVAILABLE   AGE
   nginx-deployment   3/3     3            3           18s
   ```
   Lưu ý rằng Deployment đã tạo đủ cả ba replica, và tất cả replica đều up-to-date (chứa Pod template mới nhất) và sẵn sàng (available).

5. Để xem ReplicaSet (`rs`) do Deployment tạo ra, chạy `kubectl get rs`. Kết quả tương tự như sau:
   ```
   NAME                          DESIRED   CURRENT   READY   AGE
   nginx-deployment-75675f5897   3         3         3       18s
   ```
   Output của ReplicaSet hiển thị các trường sau:

   * `NAME` liệt kê tên các ReplicaSet trong namespace.
   * `DESIRED` hiển thị số _replica_ mong muốn của ứng dụng, giá trị bạn định nghĩa khi tạo Deployment. Đây chính là _trạng thái mong muốn_ (desired state).
   * `CURRENT` hiển thị bao nhiêu replica hiện đang chạy.
   * `READY` hiển thị bao nhiêu replica của ứng dụng đang sẵn sàng phục vụ người dùng.
   * `AGE` hiển thị khoảng thời gian ứng dụng đã chạy.

   Chú ý rằng tên của ReplicaSet luôn có định dạng
   `[DEPLOYMENT-NAME]-[HASH]`. Tên này sẽ trở thành cơ sở đặt tên cho các Pod
   được tạo ra.

   Chuỗi `HASH` chính là giá trị của nhãn `pod-template-hash` trên ReplicaSet.

6. Để xem các nhãn được sinh tự động cho từng Pod, chạy `kubectl get pods --show-labels`.
   Kết quả tương tự:
   ```
   NAME                                READY     STATUS    RESTARTS   AGE       LABELS
   nginx-deployment-75675f5897-7ci7o   1/1       Running   0          18s       app=nginx,pod-template-hash=75675f5897
   nginx-deployment-75675f5897-kzszj   1/1       Running   0          18s       app=nginx,pod-template-hash=75675f5897
   nginx-deployment-75675f5897-qqcnn   1/1       Running   0          18s       app=nginx,pod-template-hash=75675f5897
   ```
   ReplicaSet được tạo ra bảo đảm luôn có ba Pod `nginx`.

> **Ghi chú:**
> Bạn phải chỉ định selector và nhãn Pod template phù hợp trong một Deployment
> (trong trường hợp này là `app: nginx`).
>
> Không để nhãn hoặc selector chồng lấn với các controller khác (bao gồm cả các Deployment và StatefulSet khác). Kubernetes không ngăn bạn tạo chồng lấn, và nếu nhiều controller có selector chồng lấn nhau thì các controller đó có thể xung đột và hành xử khó lường.

### Nhãn pod-template-hash (Pod-template-hash label) {#pod-template-hash-label}

> **Thận trọng:**
> Không thay đổi nhãn này.

Nhãn `pod-template-hash` được Deployment controller thêm vào mọi ReplicaSet mà một Deployment tạo ra hoặc tiếp nhận.

Nhãn này bảo đảm các ReplicaSet con của một Deployment không chồng lấn nhau. Nó được sinh ra bằng cách băm (hash) `PodTemplate` của ReplicaSet và dùng giá trị băm thu được làm giá trị nhãn, được thêm vào selector của ReplicaSet, nhãn của Pod template,
và vào bất kỳ Pod hiện có nào mà ReplicaSet đang quản lý.

## Cập nhật một Deployment (Updating a Deployment) {#updating-a-deployment}

> **Ghi chú:**
> Rollout của một Deployment chỉ được kích hoạt khi và chỉ khi Pod template của Deployment (tức `.spec.template`)
> bị thay đổi, ví dụ khi nhãn hoặc container image của template được cập nhật. Các cập nhật khác, chẳng hạn scale Deployment, không kích hoạt rollout.

Làm theo các bước dưới đây để cập nhật Deployment của bạn:

1. Hãy cập nhật các Pod nginx để dùng image `nginx:1.16.1` thay cho image `nginx:1.14.2`.

   ```shell
   kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1
   ```

   hoặc dùng lệnh sau:

   ```shell
   kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1
   ```
   trong đó `deployment/nginx-deployment` chỉ Deployment,
   `nginx` chỉ Container sẽ được cập nhật, và
   `nginx:1.16.1` chỉ image mới cùng tag của nó.

   Kết quả tương tự:

   ```
   deployment.apps/nginx-deployment image updated
   ```

   Ngoài ra, bạn có thể `edit` Deployment và đổi `.spec.template.spec.containers[0].image` từ `nginx:1.14.2` sang `nginx:1.16.1`:

   ```shell
   kubectl edit deployment/nginx-deployment
   ```

   Kết quả tương tự:

   ```
   deployment.apps/nginx-deployment edited
   ```

2. Để xem trạng thái rollout, chạy:

   ```shell
   kubectl rollout status deployment/nginx-deployment
   ```

   Kết quả tương tự như sau:

   ```
   Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
   ```

   hoặc

   ```
   deployment "nginx-deployment" successfully rolled out
   ```

Xem thêm chi tiết về Deployment đã cập nhật của bạn:

* Sau khi rollout thành công, bạn có thể xem Deployment bằng cách chạy `kubectl get deployments`.
  Kết quả tương tự như sau:

  ```
  NAME               READY   UP-TO-DATE   AVAILABLE   AGE
  nginx-deployment   3/3     3            3           36s
  ```

* Chạy `kubectl get rs` để thấy rằng Deployment đã cập nhật các Pod bằng cách tạo một ReplicaSet mới và scale nó
lên 3 replica, đồng thời scale ReplicaSet cũ xuống 0 replica.

  ```shell
  kubectl get rs
  ```

  Kết quả tương tự như sau:
  ```
  NAME                          DESIRED   CURRENT   READY   AGE
  nginx-deployment-1564180365   3         3         3       6s
  nginx-deployment-2035384211   0         0         0       36s
  ```

* Chạy `get pods` lúc này sẽ chỉ hiển thị các Pod mới:

  ```shell
  kubectl get pods
  ```

  Kết quả tương tự như sau:
  ```
  NAME                                READY     STATUS    RESTARTS   AGE
  nginx-deployment-1564180365-khku8   1/1       Running   0          14s
  nginx-deployment-1564180365-nacti   1/1       Running   0          14s
  nginx-deployment-1564180365-z9gth   1/1       Running   0          14s
  ```

  Lần tới khi muốn cập nhật các Pod này, bạn chỉ cần cập nhật lại Pod template của Deployment.

  Deployment bảo đảm rằng chỉ một số lượng nhất định Pod bị ngừng hoạt động trong khi chúng được cập nhật. Theo mặc định,
  nó bảo đảm rằng ít nhất 75% số Pod mong muốn đang chạy (tối đa 25% không sẵn sàng — 25% max unavailable).

  Deployment cũng bảo đảm rằng chỉ một số lượng nhất định Pod được tạo vượt trên số Pod mong muốn.
  Theo mặc định, nó bảo đảm rằng tối đa 125% số Pod mong muốn đang chạy (tối đa 25% vượt mức — 25% max surge).

  Ví dụ, nếu quan sát kỹ Deployment ở trên, bạn sẽ thấy rằng đầu tiên nó tạo một Pod mới,
  rồi mới xóa một Pod cũ, và tạo tiếp một Pod mới khác. Nó không kill các Pod cũ cho tới khi một số lượng đủ
  các Pod mới đã chạy lên, và không tạo các Pod mới cho tới khi một số lượng đủ các Pod cũ đã bị kill.
  Nó bảo đảm rằng ít nhất 3 Pod luôn sẵn sàng và tổng số Pod tối đa là 4. Trong trường hợp
  Deployment có 4 replica, số Pod sẽ nằm trong khoảng từ 3 đến 5.

* Xem chi tiết Deployment của bạn:
  ```shell
  kubectl describe deployments
  ```
  Kết quả tương tự như sau:
  ```
  Name:                   nginx-deployment
  Namespace:              default
  CreationTimestamp:      Thu, 30 Nov 2017 10:56:25 +0000
  Labels:                 app=nginx
  Annotations:            deployment.kubernetes.io/revision=2
  Selector:               app=nginx
  Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
  StrategyType:           RollingUpdate
  MinReadySeconds:        0
  RollingUpdateStrategy:  25% max unavailable, 25% max surge
  Pod Template:
    Labels:  app=nginx
     Containers:
      nginx:
        Image:        nginx:1.16.1
        Port:         80/TCP
        Environment:  <none>
        Mounts:       <none>
      Volumes:        <none>
    Conditions:
      Type           Status  Reason
      ----           ------  ------
      Available      True    MinimumReplicasAvailable
      Progressing    True    NewReplicaSetAvailable
    OldReplicaSets:  <none>
    NewReplicaSet:   nginx-deployment-1564180365 (3/3 replicas created)
    Events:
      Type    Reason             Age   From                   Message
      ----    ------             ----  ----                   -------
      Normal  ScalingReplicaSet  2m    deployment-controller  Scaled up replica set nginx-deployment-2035384211 to 3
      Normal  ScalingReplicaSet  24s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 1
      Normal  ScalingReplicaSet  22s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 2
      Normal  ScalingReplicaSet  22s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 2
      Normal  ScalingReplicaSet  19s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 1
      Normal  ScalingReplicaSet  19s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 3
      Normal  ScalingReplicaSet  14s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 0
  ```
  Ở đây bạn thấy rằng khi mới tạo Deployment, nó đã tạo một ReplicaSet (nginx-deployment-2035384211)
  và scale thẳng lên 3 replica. Khi bạn cập nhật Deployment, nó tạo một ReplicaSet mới
  (nginx-deployment-1564180365), scale ReplicaSet này lên 1 và chờ nó chạy lên. Sau đó nó scale ReplicaSet cũ
  xuống 2 và scale ReplicaSet mới lên 2, sao cho tại mọi thời điểm có ít nhất 3 Pod sẵn sàng và tối đa 4 Pod được tạo.
  Nó tiếp tục scale up và scale down ReplicaSet mới và ReplicaSet cũ theo cùng chiến lược rolling update như vậy.
  Cuối cùng, bạn sẽ có 3 replica sẵn sàng trong ReplicaSet mới, và ReplicaSet cũ được scale xuống 0.

> **Ghi chú:**
> Kubernetes không tính các Pod đang kết thúc (terminating) khi tính toán số `availableReplicas`, giá trị này phải nằm giữa
> `replicas - maxUnavailable` và `replicas + maxSurge`. Do đó, bạn có thể nhận thấy có nhiều Pod hơn
> dự kiến trong khi rollout, và tổng tài nguyên mà Deployment tiêu thụ nhiều hơn `replicas + maxSurge`
> cho tới khi `terminationGracePeriodSeconds` của các Pod đang kết thúc hết hạn.

### Rollover (còn gọi là nhiều bản cập nhật đang diễn ra đồng thời) {#rollover-aka-multiple-updates-in-flight}

Mỗi lần Deployment controller quan sát thấy một Deployment mới, một ReplicaSet được tạo để chạy lên
các Pod mong muốn. Nếu Deployment được cập nhật, ReplicaSet hiện có đang quản lý các Pod có nhãn
khớp với `.spec.selector` nhưng template không khớp với `.spec.template` sẽ bị scale down. Cuối cùng,
ReplicaSet mới được scale tới `.spec.replicas` và mọi ReplicaSet cũ được scale về 0.

Nếu bạn cập nhật một Deployment trong khi một rollout hiện có đang diễn ra, Deployment sẽ tạo một ReplicaSet mới
theo bản cập nhật đó và bắt đầu scale nó lên, đồng thời "rollover" ReplicaSet mà nó đang scale up trước đó
— nó sẽ thêm ReplicaSet này vào danh sách các ReplicaSet cũ và bắt đầu scale down.

Ví dụ, giả sử bạn tạo một Deployment để tạo 5 replica của `nginx:1.14.2`,
nhưng sau đó cập nhật Deployment để tạo 5 replica của `nginx:1.16.1`, khi mới chỉ có 3
replica của `nginx:1.14.2` được tạo. Trong trường hợp đó, Deployment lập tức bắt đầu
kill 3 Pod `nginx:1.14.2` đã tạo, và bắt đầu tạo các Pod
`nginx:1.16.1`. Nó không chờ cho đủ 5 replica của `nginx:1.14.2` được tạo
rồi mới đổi hướng.

### Cập nhật label selector (Label selector updates) {#label-selector-updates}

Nói chung, việc cập nhật label selector không được khuyến khích, và bạn nên hoạch định selector ngay từ đầu.
Label selector của một Deployment là **bất biến (immutable)** sau khi tạo;
nó không thể được cập nhật qua `kubectl patch`, `kubectl edit`, `kubectl apply`, hay các công cụ như `helm upgrade`.

Nếu buộc phải thay đổi selector, bạn phải xóa Deployment và tạo lại nó.
Theo mặc định, xóa Deployment cũng xóa luôn các Pod đang chạy của nó, gây gián đoạn dịch vụ (downtime); hãy dùng
`--cascade=orphan` nếu bạn cần các Pod đó tiếp tục chạy trong khi bạn tạo lại Deployment
(xem các hệ quả bên dưới).
Hãy hết sức thận trọng và bảo đảm bạn hiểu rõ các hệ quả sau:

* **Thêm mới (Additions):** Khi bạn tạo một Deployment mới với selector hẹp hơn, Deployment mới **bắt buộc** cũng phải có Pod template phù hợp.
  Nếu bạn có sẵn một manifest và bạn sửa manifest đó để thu hẹp selector, bạn cần sửa metadata của Pod template bên trong Deployment đó, thêm các
  nhãn mới
  sao cho khớp, nếu không API server sẽ trả về lỗi xác thực (validation error). Đây là một thay đổi _không chồng lấn_ (non-overlapping):
  Deployment mới sẽ không "thấy" các Pod cũ (vốn thiếu nhãn mới), khiến ReplicaSet
  cũ bị **mồ côi (orphaned)** và một ReplicaSet hoàn toàn mới được tạo ra.
* **Đổi giá trị (Value Updates):** Thay đổi giá trị hiện có trong một khóa của selector (ví dụ từ `v1` thành `v2`)
  dẫn đến hành vi giống như trường hợp thêm mới (mồ côi và tạo lại).
* **Xóa bớt (Removals):** Xóa một khóa hiện có khỏi selector của Deployment không đòi hỏi bất kỳ thay đổi nào
  trong nhãn của Pod template. Đây là một thay đổi _chồng lấn_ (overlapping): selector mới, rộng hơn, sẽ
  khớp với các Pod cũ. Các ReplicaSet hiện có không bị mồ côi, và không có ReplicaSet mới nào được tạo,
  nhưng lưu ý rằng nhãn đã bị xóa vẫn còn tồn tại trong mọi Pod và ReplicaSet hiện có.
  Bạn có thể dọn dẹp phần đó bằng cách kích hoạt một rollout cho Deployment.

## Rollback một Deployment (Rolling Back a Deployment) {#rolling-back-a-deployment}

Đôi khi bạn có thể muốn rollback một Deployment; ví dụ, khi Deployment không ổn định, chẳng hạn bị crash loop.
Theo mặc định, toàn bộ lịch sử rollout của Deployment được giữ lại trong hệ thống để bạn có thể rollback bất cứ lúc nào
(bạn có thể thay đổi điều đó bằng cách sửa giới hạn lịch sử revision).

> **Ghi chú:**
> Revision của một Deployment được tạo ra khi rollout của Deployment được kích hoạt. Điều này nghĩa là
> revision mới được tạo khi và chỉ khi Pod template của Deployment (`.spec.template`) bị thay đổi,
> ví dụ khi bạn cập nhật nhãn hoặc container image của template. Các cập nhật khác, như scale Deployment,
> không tạo revision mới của Deployment, nhờ vậy bạn có thể thực hiện scale thủ công hoặc scale tự động đồng thời một cách thuận tiện.
> Điều này đồng nghĩa rằng khi bạn rollback về một revision trước đó, chỉ phần Pod template của Deployment được
> rollback.

* Giả sử bạn gõ nhầm khi cập nhật Deployment, đặt tên image là `nginx:1.161` thay vì `nginx:1.16.1`:

  ```shell
  kubectl set image deployment/nginx-deployment nginx=nginx:1.161
  ```

  Kết quả tương tự như sau:
  ```
  deployment.apps/nginx-deployment image updated
  ```

* Rollout bị kẹt. Bạn có thể xác nhận điều đó bằng cách kiểm tra trạng thái rollout:

  ```shell
  kubectl rollout status deployment/nginx-deployment
  ```

  Kết quả tương tự như sau:
  ```
  Waiting for rollout to finish: 1 out of 3 new replicas have been updated...
  ```

* Nhấn Ctrl-C để dừng việc theo dõi trạng thái rollout ở trên. Để biết thêm thông tin về các rollout bị kẹt,
[đọc thêm tại đây](#deployment-status).

* Bạn thấy rằng số replica cũ (cộng số replica từ
  `nginx-deployment-1564180365` và `nginx-deployment-2035384211`) là 3, và số
  replica mới (từ `nginx-deployment-3066724191`) là 1.

  ```shell
  kubectl get rs
  ```

  Kết quả tương tự như sau:
  ```
  NAME                          DESIRED   CURRENT   READY   AGE
  nginx-deployment-1564180365   3         3         3       25s
  nginx-deployment-2035384211   0         0         0       36s
  nginx-deployment-3066724191   1         1         0       6s
  ```

* Nhìn vào các Pod đã tạo, bạn thấy 1 Pod do ReplicaSet mới tạo ra bị kẹt trong vòng lặp kéo image (image pull loop).

  ```shell
  kubectl get pods
  ```

  Kết quả tương tự như sau:
  ```
  NAME                                READY     STATUS             RESTARTS   AGE
  nginx-deployment-1564180365-70iae   1/1       Running            0          25s
  nginx-deployment-1564180365-jbqqo   1/1       Running            0          25s
  nginx-deployment-1564180365-hysrc   1/1       Running            0          25s
  nginx-deployment-3066724191-08mng   0/1       ImagePullBackOff   0          6s
  ```

  > **Ghi chú:**
  > Deployment controller tự động dừng đợt rollout hỏng, và ngừng scale up ReplicaSet mới. Điều này phụ thuộc vào các tham số rollingUpdate (cụ thể là `maxUnavailable`) mà bạn đã chỉ định. Kubernetes mặc định đặt giá trị này là 25%.

* Xem mô tả của Deployment:
  ```shell
  kubectl describe deployment
  ```

  Kết quả tương tự như sau:
  ```
  Name:           nginx-deployment
  Namespace:      default
  CreationTimestamp:  Tue, 15 Mar 2016 14:48:04 -0700
  Labels:         app=nginx
  Selector:       app=nginx
  Replicas:       3 desired | 1 updated | 4 total | 3 available | 1 unavailable
  StrategyType:       RollingUpdate
  MinReadySeconds:    0
  RollingUpdateStrategy:  25% max unavailable, 25% max surge
  Pod Template:
    Labels:  app=nginx
    Containers:
     nginx:
      Image:        nginx:1.161
      Port:         80/TCP
      Host Port:    0/TCP
      Environment:  <none>
      Mounts:       <none>
    Volumes:        <none>
  Conditions:
    Type           Status  Reason
    ----           ------  ------
    Available      True    MinimumReplicasAvailable
    Progressing    True    ReplicaSetUpdated
  OldReplicaSets:     nginx-deployment-1564180365 (3/3 replicas created)
  NewReplicaSet:      nginx-deployment-3066724191 (1/1 replicas created)
  Events:
    FirstSeen LastSeen    Count   From                    SubObjectPath   Type        Reason              Message
    --------- --------    -----   ----                    -------------   --------    ------              -------
    1m        1m          1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-2035384211 to 3
    22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 1
    22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 2
    22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 2
    21s       21s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 1
    21s       21s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 3
    13s       13s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 0
    13s       13s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-3066724191 to 1
  ```

  Để khắc phục, bạn cần rollback về một revision trước đó của Deployment đang ổn định.

### Kiểm tra lịch sử rollout của một Deployment (Checking Rollout History of a Deployment) {#checking-rollout-history-of-a-deployment}

Làm theo các bước dưới đây để kiểm tra lịch sử rollout:

1. Đầu tiên, kiểm tra các revision của Deployment này:
   ```shell
   kubectl rollout history deployment/nginx-deployment
   ```
   Kết quả tương tự như sau:
   ```
   deployments "nginx-deployment"
   REVISION    CHANGE-CAUSE
   1           <none>
   2           <none>
   3           <none>
   ```

   `CHANGE-CAUSE` được sao chép từ annotation `kubernetes.io/change-cause` của Deployment sang các revision của nó khi tạo. Bạn có thể chỉ định thông điệp `CHANGE-CAUSE` bằng cách:

   * Gắn annotation cho Deployment bằng lệnh `kubectl annotate deployment/nginx-deployment kubernetes.io/change-cause="image updated to 1.16.1"`
   * Sửa thủ công manifest của tài nguyên.
   * Dùng công cụ tự động đặt annotation này.

   > **Ghi chú:**
   > Trong các phiên bản Kubernetes cũ hơn, bạn có thể dùng cờ `--record` với các lệnh kubectl để tự động điền trường `CHANGE-CAUSE`. Cờ này đã bị loại bỏ dần (deprecated) và sẽ bị gỡ trong một bản phát hành tương lai.

2. Để xem chi tiết của từng revision, chạy:
   ```shell
   kubectl rollout history deployment/nginx-deployment --revision=2
   ```

   Kết quả tương tự như sau:
   ```
   deployments "nginx-deployment" revision 2
     Labels:       app=nginx
             pod-template-hash=1159050644
     Containers:
      nginx:
       Image:      nginx:1.16.1
       Port:       80/TCP
        QoS Tier:
           cpu:      BestEffort
           memory:   BestEffort
       Environment Variables:      <none>
     No volumes.
   ```

### Rollback về một revision trước đó (Rolling Back to a Previous Revision) {#rolling-back-to-a-previous-revision}

Làm theo các bước dưới đây để rollback Deployment từ phiên bản hiện tại về phiên bản trước, tức phiên bản 2.

1. Bây giờ bạn quyết định hoàn tác (undo) rollout hiện tại và rollback về revision trước đó:
   ```shell
   kubectl rollout undo deployment/nginx-deployment
   ```

   Kết quả tương tự như sau:
   ```
   deployment.apps/nginx-deployment rolled back
   ```
   Ngoài ra, bạn có thể rollback về một revision cụ thể bằng cách chỉ định nó với `--to-revision`:

   ```shell
   kubectl rollout undo deployment/nginx-deployment --to-revision=2
   ```

   Kết quả tương tự như sau:
   ```
   deployment.apps/nginx-deployment rolled back
   ```

   Để biết thêm chi tiết về các lệnh liên quan tới rollout, đọc [`kubectl rollout`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#rollout).

   Deployment giờ đã được rollback về revision ổn định trước đó. Như bạn thấy, một sự kiện `DeploymentRollback`
   cho việc rollback về revision 2 được Deployment controller sinh ra.

2. Kiểm tra xem việc rollback có thành công và Deployment có đang chạy như mong đợi hay không, chạy:
   ```shell
   kubectl get deployment nginx-deployment
   ```

   Kết quả tương tự như sau:
   ```
   NAME               READY   UP-TO-DATE   AVAILABLE   AGE
   nginx-deployment   3/3     3            3           30m
   ```
3. Xem mô tả của Deployment:
   ```shell
   kubectl describe deployment nginx-deployment
   ```
   Kết quả tương tự như sau:
   ```
   Name:                   nginx-deployment
   Namespace:              default
   CreationTimestamp:      Sun, 02 Sep 2018 18:17:55 -0500
   Labels:                 app=nginx
   Annotations:            deployment.kubernetes.io/revision=4
   Selector:               app=nginx
   Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
   StrategyType:           RollingUpdate
   MinReadySeconds:        0
   RollingUpdateStrategy:  25% max unavailable, 25% max surge
   Pod Template:
     Labels:  app=nginx
     Containers:
      nginx:
       Image:        nginx:1.16.1
       Port:         80/TCP
       Host Port:    0/TCP
       Environment:  <none>
       Mounts:       <none>
     Volumes:        <none>
   Conditions:
     Type           Status  Reason
     ----           ------  ------
     Available      True    MinimumReplicasAvailable
     Progressing    True    NewReplicaSetAvailable
   OldReplicaSets:  <none>
   NewReplicaSet:   nginx-deployment-c4747d96c (3/3 replicas created)
   Events:
     Type    Reason              Age   From                   Message
     ----    ------              ----  ----                   -------
     Normal  ScalingReplicaSet   12m   deployment-controller  Scaled up replica set nginx-deployment-75675f5897 to 3
     Normal  ScalingReplicaSet   11m   deployment-controller  Scaled up replica set nginx-deployment-c4747d96c to 1
     Normal  ScalingReplicaSet   11m   deployment-controller  Scaled down replica set nginx-deployment-75675f5897 to 2
     Normal  ScalingReplicaSet   11m   deployment-controller  Scaled up replica set nginx-deployment-c4747d96c to 2
     Normal  ScalingReplicaSet   11m   deployment-controller  Scaled down replica set nginx-deployment-75675f5897 to 1
     Normal  ScalingReplicaSet   11m   deployment-controller  Scaled up replica set nginx-deployment-c4747d96c to 3
     Normal  ScalingReplicaSet   11m   deployment-controller  Scaled down replica set nginx-deployment-75675f5897 to 0
     Normal  ScalingReplicaSet   11m   deployment-controller  Scaled up replica set nginx-deployment-595696685f to 1
     Normal  DeploymentRollback  15s   deployment-controller  Rolled back deployment "nginx-deployment" to revision 2
     Normal  ScalingReplicaSet   15s   deployment-controller  Scaled down replica set nginx-deployment-595696685f to 0
   ```

## Scale một Deployment (Scaling a Deployment) {#scaling-a-deployment}

Bạn có thể scale một Deployment bằng lệnh sau:

```shell
kubectl scale deployment/nginx-deployment --replicas=10
```
Kết quả tương tự như sau:
```
deployment.apps/nginx-deployment scaled
```

Giả sử [horizontal Pod autoscaling](72-horizontal-pod-autoscale-vi.md) đã được bật
trong cluster của bạn, bạn có thể thiết lập một autoscaler cho Deployment và chọn số Pod tối thiểu và tối đa
mà bạn muốn chạy dựa trên mức sử dụng CPU của các Pod hiện có.

```shell
kubectl autoscale deployment/nginx-deployment --min=10 --max=15 --cpu-percent=80%
```
Kết quả tương tự như sau:
```
deployment.apps/nginx-deployment scaled
```

### Scale theo tỷ lệ (Proportional scaling) {#proportional-scaling}

Deployment kiểu RollingUpdate hỗ trợ chạy nhiều phiên bản của một ứng dụng cùng một lúc. Khi bạn
hoặc một autoscaler scale một RollingUpdate Deployment đang ở giữa chừng một rollout (đang diễn ra
hoặc đang tạm dừng), Deployment controller sẽ cân bằng các replica bổ sung vào các ReplicaSet
đang hoạt động hiện có (các ReplicaSet có Pod) để giảm thiểu rủi ro. Cơ chế này được gọi là *scale theo tỷ lệ (proportional scaling)*.

Ví dụ, bạn đang chạy một Deployment với 10 replica, [maxSurge](#max-surge)=3, và [maxUnavailable](#max-unavailable)=2.

* Bảo đảm rằng 10 replica trong Deployment của bạn đang chạy.
  ```shell
  kubectl get deploy
  ```
  Kết quả tương tự như sau:

  ```
  NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
  nginx-deployment     10        10        10           10          50s
  ```

* Bạn cập nhật sang một image mới mà tình cờ không thể phân giải được từ bên trong cluster.
  ```shell
  kubectl set image deployment/nginx-deployment nginx=nginx:sometag
  ```

  Kết quả tương tự như sau:
  ```
  deployment.apps/nginx-deployment image updated
  ```

* Việc cập nhật image khởi động một rollout mới với ReplicaSet nginx-deployment-1989198191, nhưng nó bị chặn lại do
yêu cầu `maxUnavailable` mà bạn đã đề cập ở trên. Kiểm tra trạng thái rollout:
  ```shell
  kubectl get rs
  ```
  Kết quả tương tự như sau:
  ```
  NAME                          DESIRED   CURRENT   READY     AGE
  nginx-deployment-1989198191   5         5         0         9s
  nginx-deployment-618515232    8         8         8         1m
  ```

* Rồi một yêu cầu scale mới cho Deployment xuất hiện. Autoscaler tăng số replica của Deployment
lên 15. Deployment controller cần quyết định thêm 5 replica mới này vào đâu. Nếu không dùng
scale theo tỷ lệ, cả 5 sẽ được thêm vào ReplicaSet mới. Với scale theo tỷ lệ, bạn
phân bổ các replica bổ sung ra tất cả các ReplicaSet. Phần lớn hơn thuộc về các ReplicaSet có
nhiều replica nhất, phần nhỏ hơn thuộc về các ReplicaSet có ít replica hơn. Phần dư được thêm vào
ReplicaSet có nhiều replica nhất. Các ReplicaSet có 0 replica sẽ không được scale up.

Trong ví dụ trên, 3 replica được thêm vào ReplicaSet cũ và 2 replica được thêm vào
ReplicaSet mới. Quá trình rollout cuối cùng sẽ chuyển toàn bộ replica sang ReplicaSet mới, với điều kiện
các replica mới trở nên khỏe mạnh (healthy). Để xác nhận, chạy:

```shell
kubectl get deploy
```

Kết quả tương tự như sau:
```
NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment     15        18        7            8           7m
```
Trạng thái rollout xác nhận cách các replica được thêm vào từng ReplicaSet.
```shell
kubectl get rs
```

Kết quả tương tự như sau:
```
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-1989198191   7         7         0         7m
nginx-deployment-618515232    11        11        11        7m
```

## Tạm dừng và tiếp tục rollout của một Deployment (Pausing and Resuming a rollout of a Deployment) {#pausing-and-resuming-a-deployment}

Khi cập nhật một Deployment, hoặc có kế hoạch cập nhật, bạn có thể tạm dừng các rollout
cho Deployment đó trước khi kích hoạt một hoặc nhiều bản cập nhật. Khi
bạn sẵn sàng áp dụng những thay đổi đó, bạn tiếp tục (resume) các rollout cho
Deployment. Cách tiếp cận này cho phép bạn
áp dụng nhiều bản sửa trong khoảng giữa lúc tạm dừng và lúc tiếp tục mà không kích hoạt các rollout không cần thiết.

* Ví dụ, với một Deployment vừa được tạo:

  Lấy chi tiết Deployment:
  ```shell
  kubectl get deploy
  ```
  Kết quả tương tự như sau:
  ```
  NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
  nginx     3         3         3            3           1m
  ```
  Lấy trạng thái rollout:
  ```shell
  kubectl get rs
  ```
  Kết quả tương tự như sau:
  ```
  NAME               DESIRED   CURRENT   READY     AGE
  nginx-2142116321   3         3         3         1m
  ```

* Tạm dừng bằng cách chạy lệnh sau:
  ```shell
  kubectl rollout pause deployment/nginx-deployment
  ```

  Kết quả tương tự như sau:
  ```
  deployment.apps/nginx-deployment paused
  ```

* Sau đó cập nhật image của Deployment:
  ```shell
  kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1
  ```

  Kết quả tương tự như sau:
  ```
  deployment.apps/nginx-deployment image updated
  ```

* Lưu ý rằng không có rollout mới nào được bắt đầu:
  ```shell
  kubectl rollout history deployment/nginx-deployment
  ```

  Kết quả tương tự như sau:
  ```
  deployments "nginx"
  REVISION  CHANGE-CAUSE
  1   <none>
  ```
* Lấy trạng thái rollout để xác nhận rằng ReplicaSet hiện có không thay đổi:
  ```shell
  kubectl get rs
  ```

  Kết quả tương tự như sau:
  ```
  NAME               DESIRED   CURRENT   READY     AGE
  nginx-2142116321   3         3         3         2m
  ```

* Bạn có thể thực hiện bao nhiêu bản cập nhật tùy ý, ví dụ, cập nhật lượng tài nguyên sẽ được sử dụng:
  ```shell
  kubectl set resources deployment/nginx-deployment -c=nginx --limits=cpu=200m,memory=512Mi
  ```

  Kết quả tương tự như sau:
  ```
  deployment.apps/nginx-deployment resource requirements updated
  ```

  Trạng thái ban đầu của Deployment trước khi tạm dừng rollout vẫn tiếp tục hoạt động, nhưng các bản cập nhật mới vào
  Deployment sẽ không có bất kỳ hiệu lực nào chừng nào rollout của Deployment còn bị tạm dừng.

* Cuối cùng, tiếp tục rollout của Deployment và quan sát một ReplicaSet mới xuất hiện với tất cả các bản cập nhật mới:
  ```shell
  kubectl rollout resume deployment/nginx-deployment
  ```

  Kết quả tương tự như sau:
  ```
  deployment.apps/nginx-deployment resumed
  ```
* Theo dõi (watch) trạng thái của rollout cho đến khi nó hoàn tất.
  ```shell
  kubectl get rs --watch
  ```

  Kết quả tương tự như sau:
  ```
  NAME               DESIRED   CURRENT   READY     AGE
  nginx-2142116321   2         2         2         2m
  nginx-3926361531   2         2         0         6s
  nginx-3926361531   2         2         1         18s
  nginx-2142116321   1         2         2         2m
  nginx-2142116321   1         2         2         2m
  nginx-3926361531   3         2         1         18s
  nginx-3926361531   3         2         1         18s
  nginx-2142116321   1         1         1         2m
  nginx-3926361531   3         3         1         18s
  nginx-3926361531   3         3         2         19s
  nginx-2142116321   0         1         1         2m
  nginx-2142116321   0         1         1         2m
  nginx-2142116321   0         0         0         2m
  nginx-3926361531   3         3         3         20s
  ```
* Lấy trạng thái của rollout mới nhất:
  ```shell
  kubectl get rs
  ```

  Kết quả tương tự như sau:
  ```
  NAME               DESIRED   CURRENT   READY     AGE
  nginx-2142116321   0         0         0         2m
  nginx-3926361531   3         3         3         28s
  ```
> **Ghi chú:**
> Bạn không thể rollback một Deployment đang tạm dừng cho tới khi bạn tiếp tục (resume) nó.

## Trạng thái của Deployment (Deployment status) {#deployment-status}

Một Deployment trải qua nhiều trạng thái khác nhau trong vòng đời của nó. Nó có thể [đang tiến triển (progressing)](#progressing-deployment) trong khi
rollout một ReplicaSet mới, có thể [hoàn tất (complete)](#complete-deployment), hoặc có thể [thất bại trong việc tiến triển (fail to progress)](#failed-deployment).

### Deployment đang tiến triển (Progressing Deployment) {#progressing-deployment}

Kubernetes đánh dấu một Deployment là _progressing_ (đang tiến triển) khi một trong các tác vụ sau được thực hiện:

* Deployment tạo một ReplicaSet mới.
* Deployment đang scale up ReplicaSet mới nhất của nó.
* Deployment đang scale down (các) ReplicaSet cũ hơn của nó.
* Các Pod mới trở nên ready hoặc available (ready trong ít nhất [MinReadySeconds](#min-ready-seconds)).

Khi rollout trở thành "progressing", Deployment controller thêm một condition (điều kiện) với các
thuộc tính sau vào `.status.conditions` của Deployment:

* `type: Progressing`
* `status: "True"`
* `reason: NewReplicaSetCreated` | `reason: FoundNewReplicaSet` | `reason: ReplicaSetUpdated`

Bạn có thể theo dõi tiến trình của một Deployment bằng `kubectl rollout status`.

### Deployment hoàn tất (Complete Deployment) {#complete-deployment}

Kubernetes đánh dấu một Deployment là _complete_ (hoàn tất) khi nó có các đặc điểm sau:

* Tất cả replica gắn với Deployment đã được cập nhật lên phiên bản mới nhất mà bạn chỉ định, nghĩa là mọi
bản cập nhật bạn đã yêu cầu đều đã hoàn thành.
* Tất cả replica gắn với Deployment đều sẵn sàng (available).
* Không còn replica cũ nào của Deployment đang chạy.

Khi rollout trở thành "complete", Deployment controller đặt một condition với các
thuộc tính sau vào `.status.conditions` của Deployment:

* `type: Progressing`
* `status: "True"`
* `reason: NewReplicaSetAvailable`

Condition `Progressing` này sẽ giữ giá trị trạng thái `"True"` cho tới khi một rollout mới
được khởi động. Condition này được giữ nguyên ngay cả khi mức sẵn sàng của các replica thay đổi (điều đó
thay vào đó ảnh hưởng tới condition `Available`).

Bạn có thể kiểm tra một Deployment đã hoàn tất hay chưa bằng `kubectl rollout status`. Nếu rollout hoàn tất
thành công, `kubectl rollout status` trả về mã thoát (exit code) bằng 0.

```shell
kubectl rollout status deployment/nginx-deployment
```
Kết quả tương tự như sau:
```
Waiting for rollout to finish: 2 of 3 updated replicas are available...
deployment "nginx-deployment" successfully rolled out
```
và mã thoát từ `kubectl rollout` là 0 (thành công):
```shell
echo $?
```
```
0
```

### Deployment thất bại (Failed Deployment) {#failed-deployment}

Deployment của bạn có thể bị kẹt khi cố triển khai ReplicaSet mới nhất mà không bao giờ hoàn tất. Điều này có thể xảy ra
do một số yếu tố sau:

* Không đủ quota (hạn ngạch)
* Readiness probe thất bại
* Lỗi kéo image (image pull errors)
* Không đủ quyền
* Limit range
* Cấu hình sai môi trường chạy của ứng dụng

Một cách để phát hiện tình trạng này là chỉ định tham số deadline (hạn chót) trong spec của Deployment:
([`.spec.progressDeadlineSeconds`](#progress-deadline-seconds)). `.spec.progressDeadlineSeconds` biểu thị
số giây mà Deployment controller chờ trước khi thông báo (trong trạng thái của Deployment) rằng
tiến trình của Deployment đã đình trệ.

Lệnh `kubectl` sau đây đặt spec với `progressDeadlineSeconds` để controller báo cáo
việc thiếu tiến triển của một rollout cho Deployment sau 10 phút:

```shell
kubectl patch deployment/nginx-deployment -p '{"spec":{"progressDeadlineSeconds":600}}'
```
Kết quả tương tự như sau:
```
deployment.apps/nginx-deployment patched
```
Khi deadline bị vượt quá, Deployment controller thêm một DeploymentCondition với các
thuộc tính sau vào `.status.conditions` của Deployment:

* `type: Progressing`
* `status: "False"`
* `reason: ProgressDeadlineExceeded`

Condition này cũng có thể thất bại sớm hơn và khi đó được đặt giá trị trạng thái `"False"` với các lý do như `ReplicaSetCreateError`.
Ngoài ra, deadline không còn được tính đến nữa một khi rollout của Deployment đã hoàn tất.

Xem [quy ước API của Kubernetes](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#typical-status-properties) để biết thêm thông tin về các condition trạng thái.

> **Ghi chú:**
> Kubernetes không thực hiện hành động nào trên một Deployment bị đình trệ ngoài việc báo cáo một condition trạng thái với
> `reason: ProgressDeadlineExceeded`. Các bộ điều phối (orchestrator) cấp cao hơn có thể tận dụng điều này và hành động tương ứng, ví dụ
> rollback Deployment về phiên bản trước của nó.

> **Ghi chú:**
> Nếu bạn tạm dừng một rollout của Deployment, Kubernetes sẽ không kiểm tra tiến trình so với deadline mà bạn đã chỉ định.
> Bạn có thể yên tâm tạm dừng rollout của một Deployment giữa chừng và tiếp tục lại mà không kích hoạt
> condition vượt quá deadline.

Bạn có thể gặp các lỗi thoáng qua (transient) với các Deployment của mình, do timeout thấp mà bạn đã đặt hoặc
do bất kỳ loại lỗi nào khác có thể được xem là tạm thời. Ví dụ, giả sử bạn
không đủ quota. Nếu describe Deployment, bạn sẽ nhận thấy phần sau:

```shell
kubectl describe deployment nginx-deployment
```
Kết quả tương tự như sau:
```
<...>
Conditions:
  Type            Status  Reason
  ----            ------  ------
  Available       True    MinimumReplicasAvailable
  Progressing     True    ReplicaSetUpdated
  ReplicaFailure  True    FailedCreate
<...>
```

Nếu chạy `kubectl get deployment nginx-deployment -o yaml`, trạng thái Deployment tương tự như sau:

```
status:
  availableReplicas: 2
  conditions:
  - lastTransitionTime: 2016-10-04T12:25:39Z
    lastUpdateTime: 2016-10-04T12:25:39Z
    message: Replica set "nginx-deployment-4262182780" is progressing.
    reason: ReplicaSetUpdated
    status: "True"
    type: Progressing
  - lastTransitionTime: 2016-10-04T12:25:42Z
    lastUpdateTime: 2016-10-04T12:25:42Z
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: 2016-10-04T12:25:39Z
    lastUpdateTime: 2016-10-04T12:25:39Z
    message: 'Error creating: pods "nginx-deployment-4262182780-" is forbidden: exceeded quota:
      object-counts, requested: pods=1, used: pods=3, limited: pods=2'
    reason: FailedCreate
    status: "True"
    type: ReplicaFailure
  observedGeneration: 3
  replicas: 2
  unavailableReplicas: 2
```

Cuối cùng, khi deadline tiến trình của Deployment bị vượt quá, Kubernetes cập nhật trạng thái và
lý do của condition Progressing:

```
Conditions:
  Type            Status  Reason
  ----            ------  ------
  Available       True    MinimumReplicasAvailable
  Progressing     False   ProgressDeadlineExceeded
  ReplicaFailure  True    FailedCreate
```

Bạn có thể xử lý vấn đề thiếu quota bằng cách scale down Deployment, scale down các
controller khác mà bạn đang chạy, hoặc tăng quota trong namespace của bạn. Nếu bạn thỏa mãn các điều kiện
quota và Deployment controller sau đó hoàn tất rollout của Deployment, bạn sẽ thấy
trạng thái của Deployment được cập nhật với condition thành công (`status: "True"` và `reason: NewReplicaSetAvailable`).

```
Conditions:
  Type          Status  Reason
  ----          ------  ------
  Available     True    MinimumReplicasAvailable
  Progressing   True    NewReplicaSetAvailable
```

`type: Available` với `status: "True"` nghĩa là Deployment của bạn có mức sẵn sàng tối thiểu. Mức sẵn sàng tối thiểu được quy định
bởi các tham số chỉ định trong chiến lược triển khai (deployment strategy). `type: Progressing` với `status: "True"` nghĩa là Deployment của bạn
hoặc đang ở giữa chừng một rollout và đang tiến triển, hoặc đã hoàn tất tiến trình thành công và số
replica mới tối thiểu cần thiết đang sẵn sàng (xem Reason của condition để biết chi tiết — trong trường hợp của chúng ta
`reason: NewReplicaSetAvailable` nghĩa là Deployment đã hoàn tất).

Bạn có thể kiểm tra một Deployment có thất bại trong việc tiến triển hay không bằng `kubectl rollout status`. `kubectl rollout status`
trả về mã thoát khác 0 nếu Deployment đã vượt quá deadline tiến triển.

```shell
kubectl rollout status deployment/nginx-deployment
```
Kết quả tương tự như sau:
```
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
error: deployment "nginx" exceeded its progress deadline
```
và mã thoát từ `kubectl rollout` là 1 (biểu thị có lỗi):
```shell
echo $?
```
```
1
```

### Thao tác trên một deployment thất bại (Operating on a failed deployment) {#operating-on-a-failed-deployment}

Mọi hành động áp dụng được cho một Deployment hoàn tất cũng áp dụng được cho một Deployment thất bại. Bạn có thể scale up/down, rollback
về một revision trước đó, hoặc thậm chí tạm dừng nó nếu bạn cần áp dụng nhiều chỉnh sửa vào Pod template của Deployment.

## Chính sách dọn dẹp (Clean up Policy) {#clean-up-policy}

Bạn có thể đặt trường `.spec.revisionHistoryLimit` trong một Deployment để chỉ định bao nhiêu ReplicaSet cũ của
Deployment này mà bạn muốn giữ lại. Phần còn lại sẽ được thu gom rác (garbage-collected) ở chế độ nền. Theo mặc định,
giá trị này là 10.

> **Ghi chú:**
> Đặt trường này bằng 0 một cách tường minh sẽ dẫn đến việc dọn sạch toàn bộ lịch sử của Deployment,
> do đó Deployment sẽ không thể rollback được nữa.

Việc dọn dẹp chỉ bắt đầu **sau khi** Deployment đạt
[trạng thái hoàn tất](#complete-deployment).
Nếu bạn đặt `.spec.revisionHistoryLimit` bằng 0, bất kỳ rollout nào vẫn kích hoạt việc tạo một
ReplicaSet mới trước khi Kubernetes xóa ReplicaSet cũ.

Ngay cả với giới hạn lịch sử revision khác 0, bạn vẫn có thể có nhiều ReplicaSet hơn giới hạn
mà bạn cấu hình. Ví dụ, nếu các pod bị crash loop, và có nhiều sự kiện rolling update
được kích hoạt theo thời gian, bạn có thể có nhiều ReplicaSet hơn
`.spec.revisionHistoryLimit` vì Deployment không bao giờ đạt được trạng thái hoàn tất.

## Deployment kiểu canary (Canary Deployment) {#canary-deployment}

Nếu bạn muốn phát hành các bản phát hành (release) tới một nhóm nhỏ người dùng hoặc máy chủ bằng Deployment, bạn
có thể tạo nhiều Deployment, mỗi cái cho một bản phát hành, theo mẫu canary được mô tả trong
[quản lý tài nguyên](61-management-vi.md#canary-deployments).

## Viết một Deployment Spec (Writing a Deployment Spec) {#writing-a-deployment-spec}

Như mọi cấu hình Kubernetes khác, một Deployment cần các trường `.apiVersion`, `.kind`, và `.metadata`.
Để biết thông tin chung về cách làm việc với các file cấu hình, xem các tài liệu
[triển khai ứng dụng](345-run-stateless-application-vi.md),
cấu hình container, và [dùng kubectl để quản lý tài nguyên](./27-object-management-vi.md).

Khi control plane tạo các Pod mới cho một Deployment, `.metadata.name` của
Deployment là một phần cơ sở để đặt tên cho các Pod đó. Tên của một Deployment phải là một giá trị
[DNS subdomain](./17-names-vi.md) hợp lệ,
nhưng điều này có thể tạo ra kết quả bất ngờ cho hostname của các Pod. Để tương thích tốt nhất,
tên nên tuân theo các quy tắc chặt chẽ hơn của một
[DNS label](./17-names-vi.md#dns-label-names).

Một Deployment cũng cần một [phần `.spec`](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status).

### Pod Template {#pod-template}

`.spec.template` và `.spec.selector` là các trường bắt buộc duy nhất của `.spec`.

`.spec.template` là một [Pod template](./46-pods-vi.md#pod-template). Nó có schema y hệt như một Pod, ngoại trừ việc nó được lồng bên trong và không có `apiVersion` hay `kind`.

Bên cạnh các trường bắt buộc của một Pod, một Pod template trong Deployment phải chỉ định các nhãn phù hợp
và một chính sách khởi động lại (restart policy) phù hợp. Với các nhãn, hãy chắc chắn không chồng lấn với các controller khác. Xem [selector](#selector).

Chỉ cho phép [`.spec.template.spec.restartPolicy`](./47-pod-lifecycle-vi.md#restart-policy) bằng `Always`,
đây cũng là giá trị mặc định nếu không được chỉ định.

### Replicas {#replicas}

`.spec.replicas` là trường tùy chọn chỉ định số Pod mong muốn. Giá trị mặc định là 1.

Nếu bạn scale thủ công một Deployment, ví dụ qua `kubectl scale deployment
deployment --replicas=X`, rồi sau đó bạn cập nhật Deployment đó dựa trên một manifest
(ví dụ: bằng cách chạy `kubectl apply -f deployment.yaml`),
thì việc apply manifest đó sẽ ghi đè lên thao tác scale thủ công mà bạn đã làm trước đó.

Nếu một [HorizontalPodAutoscaler](72-horizontal-pod-autoscale-vi.md) (hoặc bất kỳ
API tương tự nào cho việc scale theo chiều ngang) đang quản lý việc scale cho một Deployment, đừng đặt `.spec.replicas`.

Thay vào đó, hãy để control plane của Kubernetes quản lý
trường `.spec.replicas` một cách tự động.

### Selector {#selector}

`.spec.selector` là trường bắt buộc chỉ định một [label selector](./18-labels-vi.md)
cho các Pod mà Deployment này nhắm tới.

`.spec.selector` phải khớp với `.spec.template.metadata.labels`, nếu không nó sẽ bị API từ chối.

Trong phiên bản API `apps/v1`, `.spec.selector` và `.metadata.labels` không mặc định lấy theo `.spec.template.metadata.labels` nếu không được đặt. Vì vậy chúng phải được đặt một cách tường minh. Cũng lưu ý rằng `.spec.selector` là bất biến sau khi tạo Deployment trong `apps/v1`.

Một Deployment có thể chấm dứt (terminate) các Pod có nhãn khớp với selector nếu template của chúng khác
với `.spec.template` hoặc nếu tổng số Pod như vậy vượt quá `.spec.replicas`. Nó tạo các Pod
mới với `.spec.template` nếu số Pod ít hơn số lượng mong muốn.

> **Ghi chú:**
> Bạn không nên tạo các Pod khác có nhãn khớp với selector này, dù là trực tiếp, bằng cách tạo
> một Deployment khác, hay bằng cách tạo một controller khác như ReplicaSet hoặc ReplicationController. Nếu
> làm vậy, Deployment thứ nhất sẽ nghĩ rằng chính nó đã tạo ra các Pod kia. Kubernetes không ngăn bạn làm điều này.

Nếu bạn có nhiều controller với selector chồng lấn nhau, các controller sẽ tranh chấp lẫn
nhau và không hoạt động đúng.

### Strategy {#strategy}

`.spec.strategy` chỉ định chiến lược được dùng để thay thế các Pod cũ bằng các Pod mới.
`.spec.strategy.type` có thể là "Recreate" hoặc "RollingUpdate". "RollingUpdate" là
giá trị mặc định.

#### Deployment kiểu Recreate (Recreate Deployment) {#recreate-deployment}

Tất cả Pod hiện có đều bị kill trước khi các Pod mới được tạo khi `.spec.strategy.type==Recreate`.

> **Ghi chú:**
> Điều này chỉ bảo đảm việc các Pod bị chấm dứt trước khi tạo mới đối với các lần nâng cấp. Nếu bạn nâng cấp một Deployment, tất cả Pod
> của revision cũ sẽ bị chấm dứt ngay lập tức. Việc gỡ bỏ thành công sẽ được chờ hoàn tất trước khi bất kỳ Pod nào của revision
> mới được tạo. Nếu bạn xóa thủ công một Pod, vòng đời của nó do ReplicaSet kiểm soát và
> Pod thay thế sẽ được tạo ngay lập tức (kể cả khi Pod cũ vẫn đang ở trạng thái Terminating). Nếu bạn cần
> sự bảo đảm "tối đa" (at most) cho số Pod của mình, bạn nên cân nhắc sử dụng
> [StatefulSet](65-statefulset-vi.md).

#### Deployment kiểu Rolling Update (Rolling Update Deployment) {#rolling-update-deployment}

Deployment cập nhật các Pod theo kiểu cập nhật cuốn chiếu (rolling update)
(từ từ scale down các ReplicaSet cũ và scale up ReplicaSet mới) khi `.spec.strategy.type==RollingUpdate`. Bạn có thể chỉ định `maxUnavailable` và `maxSurge` để kiểm soát
quá trình rolling update.

##### Max Unavailable {#max-unavailable}

`.spec.strategy.rollingUpdate.maxUnavailable` là trường tùy chọn chỉ định số Pod tối đa
có thể không sẵn sàng (unavailable) trong quá trình cập nhật. Giá trị có thể là một số tuyệt đối (ví dụ, 5)
hoặc tỷ lệ phần trăm của số Pod mong muốn (ví dụ, 10%). Số tuyệt đối được tính từ tỷ lệ phần trăm bằng cách
làm tròn xuống. Giá trị này không thể là 0 nếu `.spec.strategy.rollingUpdate.maxSurge` là 0. Giá trị mặc định là 25%.

Ví dụ, khi giá trị này được đặt là 30%, ReplicaSet cũ có thể được scale down ngay xuống 70% số Pod
mong muốn khi rolling update bắt đầu. Khi các Pod mới đã ready, ReplicaSet cũ có thể được scale
down thêm nữa, kèm theo việc scale up ReplicaSet mới, bảo đảm rằng tổng số Pod sẵn sàng
tại mọi thời điểm trong quá trình cập nhật là ít nhất 70% số Pod mong muốn.

##### Max Surge {#max-surge}

`.spec.strategy.rollingUpdate.maxSurge` là trường tùy chọn chỉ định số Pod tối đa
có thể được tạo vượt trên số Pod mong muốn. Giá trị có thể là một số tuyệt đối (ví dụ, 5) hoặc
tỷ lệ phần trăm của số Pod mong muốn (ví dụ, 10%). Giá trị này không thể là 0 nếu `maxUnavailable` là 0. Số tuyệt đối
được tính từ tỷ lệ phần trăm bằng cách làm tròn lên. Giá trị mặc định là 25%.

Ví dụ, khi giá trị này được đặt là 30%, ReplicaSet mới có thể được scale up ngay khi
rolling update bắt đầu, sao cho tổng số Pod cũ và mới không vượt quá 130% số Pod
mong muốn. Khi các Pod cũ đã bị kill, ReplicaSet mới có thể được scale up thêm nữa, bảo đảm rằng
tổng số Pod đang chạy tại bất kỳ thời điểm nào trong quá trình cập nhật tối đa là 130% số Pod mong muốn.

Dưới đây là một số ví dụ Rolling Update Deployment sử dụng `maxUnavailable` và `maxSurge`:

###### Max Unavailable

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
```

###### Max Surge

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
```

###### Kết hợp (Hybrid)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
```

### Progress Deadline Seconds {#progress-deadline-seconds}

`.spec.progressDeadlineSeconds` là trường tùy chọn chỉ định số giây bạn muốn
chờ Deployment của mình tiến triển trước khi hệ thống báo lại rằng Deployment đã
[thất bại trong việc tiến triển](#failed-deployment) — thể hiện qua một condition với `type: Progressing`, `status: "False"`
và `reason: ProgressDeadlineExceeded` trong trạng thái của tài nguyên. Deployment controller sẽ tiếp tục
thử lại Deployment. Giá trị mặc định là 600.

Nếu được chỉ định, trường này cần lớn hơn `.spec.minReadySeconds`.

### Min Ready Seconds {#min-ready-seconds}

`.spec.minReadySeconds` là trường tùy chọn chỉ định số giây tối thiểu mà một Pod
mới được tạo phải ở trạng thái ready mà không có container nào của nó bị crash, để Pod được coi là sẵn sàng (available).
Giá trị mặc định là 0 (Pod sẽ được coi là sẵn sàng ngay khi nó ready). Để tìm hiểu thêm về khi nào
một Pod được coi là ready, xem [Container Probes](./47-pod-lifecycle-vi.md#container-probes).

### Pod đang kết thúc (Terminating Pods) {#terminating-pods}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [beta]`

Bạn chỉ có thể thấy các pod đang kết thúc (terminating) nếu [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`DeploymentReplicaSetTerminatingReplicas` được bật
trên [API server](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)
và trên [kube-controller-manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)

Các Pod chuyển sang trạng thái kết thúc do bị xóa hoặc bị scale down có thể mất nhiều thời gian để chấm dứt, và có thể tiêu thụ
thêm tài nguyên trong khoảng thời gian đó. Do vậy, tổng số toàn bộ pod có thể tạm thời vượt quá
`.spec.replicas`. Các pod đang kết thúc có thể được theo dõi qua trường `.status.terminatingReplicas` của Deployment.

### Giới hạn lịch sử revision (Revision History Limit) {#revision-history-limit}

Lịch sử revision của một Deployment được lưu trong các ReplicaSet mà nó kiểm soát.

`.spec.revisionHistoryLimit` là trường tùy chọn chỉ định số ReplicaSet cũ được giữ lại
để cho phép rollback. Các ReplicaSet cũ này tiêu thụ tài nguyên trong `etcd` và làm rối output của `kubectl get rs`. Cấu hình của mỗi revision của Deployment được lưu trong các ReplicaSet của nó; do đó, một khi một ReplicaSet cũ bị xóa, bạn mất khả năng rollback về revision đó của Deployment. Theo mặc định, 10 ReplicaSet cũ sẽ được giữ lại, tuy nhiên giá trị lý tưởng phụ thuộc vào tần suất và độ ổn định của các Deployment mới.

Cụ thể hơn, đặt trường này bằng 0 nghĩa là tất cả ReplicaSet cũ có 0 replica sẽ bị dọn dẹp.
Trong trường hợp này, một rollout mới của Deployment không thể được hoàn tác, vì lịch sử revision của nó đã bị dọn sạch.

### Paused {#paused}

`.spec.paused` là trường boolean tùy chọn để tạm dừng và tiếp tục một Deployment. Khác biệt duy nhất giữa
một Deployment bị tạm dừng và một Deployment không bị tạm dừng là: mọi thay đổi vào PodTemplateSpec của Deployment
bị tạm dừng sẽ không kích hoạt các rollout mới chừng nào nó còn bị tạm dừng. Một Deployment không bị tạm dừng theo mặc định
khi được tạo.

## Tiếp theo (What's next)

* Tìm hiểu thêm về [Pod](./46-pods-vi.md).
* [Chạy một ứng dụng stateless bằng Deployment](345-run-stateless-application-vi.md).
* Đọc [tài liệu tham chiếu API `Deployment`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/deployment-v1/) để hiểu về API Deployment.
* Đọc về [PodDisruptionBudget](53-disruptions-vi.md) và cách
  bạn có thể dùng nó để quản lý mức sẵn sàng của ứng dụng trong các gián đoạn (disruption).
* Dùng kubectl để [tạo một Deployment](https://kubernetes.io/docs/tutorials/kubernetes-basics/deploy-app/deploy-intro/).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 4:

1. **Câu bẫy.** Bạn chạy `kubectl scale deployment/web --replicas=5`. Lệnh này có tạo một
   revision mới không? Có tạo một ReplicaSet mới không? Vì sao?
2. Một Deployment 4 replica để nguyên `maxUnavailable` và `maxSurge` mặc định. Trong lúc
   rolling update, tổng số Pod dao động trong khoảng nào, và số Pod sẵn sàng tối thiểu là bao
   nhiêu?
3. Bạn `kubectl set image` với một tag gõ nhầm, Pod mới kẹt `ImagePullBackOff`. Vì sao các
   Pod cũ vẫn phục vụ bình thường, trường nào quyết định điều đó, và sau khi vượt
   `progressDeadlineSeconds` thì Kubernetes có tự rollback không?
4. Trên cluster lab của bạn, một Deployment để `revisionHistoryLimit` mặc định và bạn đã đổi
   image 12 lần. Bạn còn `kubectl rollout undo --to-revision=1` được không? Vì sao?
5. Bạn đổi `.spec.strategy.type` sang `Recreate` cho một Deployment 3 replica trên hai worker.
   Điều gì đổi so với `RollingUpdate`, và bảo đảm đó có áp dụng khi bạn **xóa tay** một Pod
   không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không, cả hai đều không.** Bài nêu quy tắc hai lần: rollout của một Deployment được kích
   hoạt **khi và chỉ khi** Pod template (`.spec.template`) bị thay đổi — ví dụ đổi nhãn hoặc
   container image của template. Scale không đụng tới `.spec.template`, nên không sinh
   ReplicaSet mới (ReplicaSet được phân biệt bằng `pod-template-hash` băm từ template, mà
   template không đổi) và **không tạo revision mới**. Bài nói rõ đây là chủ ý: nhờ vậy bạn có
   thể scale thủ công hoặc tự động mà không làm rối lịch sử rollout.
2. Tổng số Pod nằm trong khoảng **3 đến 5**, và **ít nhất 3 Pod luôn sẵn sàng**. Mặc định
   `maxUnavailable` là 25% (làm tròn xuống → 1 Pod) nên ít nhất 75% số Pod mong muốn đang
   chạy, và `maxSurge` là 25% (làm tròn lên → 1 Pod) nên tối đa 125% số Pod mong muốn tồn
   tại. Bài lấy đúng ví dụ này: "Trong trường hợp Deployment có 4 replica, số Pod sẽ nằm
   trong khoảng từ 3 đến 5". Lưu ý ghi chú kèm theo: Pod đang kết thúc **không** được tính
   vào `availableReplicas`, nên trong thực tế bạn có thể thấy nhiều Pod hơn con số này cho
   tới khi `terminationGracePeriodSeconds` của chúng hết hạn.
3. Vì Deployment controller **tự động dừng đợt rollout hỏng và ngừng scale up ReplicaSet
   mới**; nó không kill Pod cũ chừng nào chưa đủ Pod mới chạy lên. Trường quyết định là
   **`maxUnavailable`** trong tham số `rollingUpdate` — Kubernetes mặc định 25%, nên với 3
   replica nó chỉ dám hạ ReplicaSet cũ xuống một mức nhất định rồi dừng. Sau khi vượt
   `progressDeadlineSeconds`, Kubernetes **không tự rollback**: bài nói rõ nó "không thực
   hiện hành động nào trên một Deployment bị đình trệ ngoài việc báo cáo một condition trạng
   thái với `reason: ProgressDeadlineExceeded`". Việc rollback do bạn hoặc một bộ điều phối
   cấp cao hơn quyết định, bằng `kubectl rollout undo`.
4. **Không.** `.spec.revisionHistoryLimit` mặc định là **10**, nghĩa là chỉ 10 ReplicaSet cũ
   được giữ lại; phần còn lại bị thu gom rác ở chế độ nền. Mà bài nói rõ "cấu hình của mỗi
   revision của Deployment được lưu trong các ReplicaSet của nó; do đó, một khi một ReplicaSet
   cũ bị xóa, bạn mất khả năng rollback về revision đó". Sau 12 lần đổi image, ReplicaSet của
   revision 1 đã bị dọn. Ngoại lệ cần nhớ: việc dọn dẹp **chỉ bắt đầu sau khi** Deployment
   đạt trạng thái hoàn tất, nên nếu Pod cứ crash loop bạn có thể thấy nhiều ReplicaSet hơn
   giới hạn.
5. Với `Recreate`, **tất cả Pod hiện có bị kill trước khi Pod mới được tạo** — nghĩa là có
   khoảng thời gian không Pod nào phục vụ, khác hẳn `RollingUpdate` vốn giữ tối thiểu 75% số
   Pod. Nhưng bảo đảm đó **chỉ áp dụng cho các lần nâng cấp**: ghi chú trong bài nói nếu bạn
   xóa tay một Pod thì vòng đời của nó do ReplicaSet kiểm soát và **Pod thay thế được tạo
   ngay lập tức**, kể cả khi Pod cũ vẫn đang `Terminating`. Muốn bảo đảm "tối đa bao nhiêu
   Pod" thật sự thì bài chỉ sang [StatefulSet](65-statefulset-vi.md).

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

# Chạy một ứng dụng Stateless bằng Deployment (Run a Stateless Application Using a Deployment)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 4 → nhóm [4a](00-ALO-TRINH-ADMIN.md#4a-replicaset-deployment-và-rollout), bài 5/7 ·
Kiểm chứng ở [Lab 4a](labs/LAB-4A-REPLICASET-DEPLOYMENT-VA-ROLLOUT.md) phần B2 và B9.

Đây là bài thực hành nhẹ nhất trong nhóm và là bài **được các bài khác dẫn ngược lại**: mục
*Trước khi bạn bắt đầu* của [260](260-use-cascading-deletion-vi.md) yêu cầu tạo Deployment mẫu theo
đúng mục *Tạo và khám phá một deployment nginx* dưới đây. Bài đi trọn một vòng đời: tạo → cập nhật
→ scale → xóa, và mọi bước đều làm bằng cùng một lệnh `kubectl apply -f`.

**Phải hiểu ở lần đọc này:**

- Mục *Tạo và khám phá một deployment nginx*: một Deployment tối thiểu chỉ cần `selector.matchLabels`,
  `replicas` và `template`; comment ngay trong file YAML nói rõ `replicas: 2` nghĩa là yêu cầu
  deployment chạy 2 pod **khớp với template**. Chính label `app: nginx` đó là thứ bài dùng ở mọi
  bước sau để liệt kê Pod bằng `kubectl get pods -l app=nginx`.
- Cũng ở mục đó, output của `kubectl describe deployment` cho ngay bốn thông tin quan trọng:
  `StrategyType: RollingUpdate`, `RollingUpdateStrategy: 1 max unavailable, 1 max surge`, dòng
  `Replicas: 2 desired | 2 updated | 2 total | 2 available | 0 unavailable`, và dòng `NewReplicaSet`
  — tức Deployment quản Pod **thông qua** một ReplicaSet, không quản trực tiếp.
- Mục *Cập nhật deployment*: cập nhật nghĩa là apply lại **cùng một file** với image mới
  (`nginx:1.14.2` → `nginx:1.16.1`). Bài mô tả kết quả bằng đúng một câu: deployment **tạo các pod
  với tên mới và xóa các pod cũ** — Pod cũ không được sửa image tại chỗ.
- Mục *Mở rộng quy mô ứng dụng bằng cách tăng số replica*: scale cũng chỉ là sửa `replicas` trong
  file rồi apply — cùng một đường khai báo, không cần lệnh riêng. Trong output kiểm chứng, hai Pod
  có `AGE` là `2m` còn hai Pod mới là `25s`: Pod cũ được giữ nguyên, chỉ thêm Pod mới.
- Mục *ReplicationControllers — cách làm cũ*: cách được khuyến nghị để chạy ứng dụng nhiều bản sao
  là **Deployment**, và Deployment **dùng ReplicaSet bên dưới**. ReplicationController là thứ có
  trước cả Deployment lẫn ReplicaSet.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| `containerPort: 80` trong manifest, và câu hỏi kéo theo là làm sao gọi được nginx đó từ bên ngoài | Pod có IP nhưng cách phơi ra một địa chỉ ổn định là việc của Service, chưa học | [giai đoạn 5](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng) |
| Câu dẫn cuối mục scale về quy trình chi tiết, gồm thu hẹp quy mô và đưa về không | là nội dung của bài kế tiếp trong chính nhóm này | bài [346](346-scale-deployment-vi.md), bài 6/7 của nhóm [4a](00-ALO-TRINH-ADMIN.md#4a-replicaset-deployment-và-rollout) |
| Ba lệnh `kubectl apply -f https://k8s.io/examples/application/...` kéo manifest thẳng từ internet | lab chép manifest thành file cục bộ rồi apply, để chạy được cả khi cluster không có egress | [Lab 4a](labs/LAB-4A-REPLICASET-DEPLOYMENT-VA-ROLLOUT.md) phần B2 |
| Mục *Trước khi bạn bắt đầu* dẫn minikube và các playground công cộng | lộ trình chạy trên cluster ba VM dựng ở Lab 00, không dùng cluster tạm hay cluster dùng chung | [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |

---

Trang này hướng dẫn cách chạy một ứng dụng bằng đối tượng Deployment của Kubernetes.

## Mục tiêu (Objectives)

- Tạo một deployment nginx.
- Dùng kubectl để liệt kê thông tin về deployment.
- Cập nhật deployment.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản v1.9 hoặc mới hơn. Để kiểm tra phiên bản,
hãy chạy `kubectl version`.

## Tạo và khám phá một deployment nginx (Creating and exploring an nginx deployment) {#creating-and-exploring-an-nginx-deployment}

Bạn có thể chạy một ứng dụng bằng cách tạo một đối tượng Deployment của Kubernetes, và bạn
có thể mô tả một Deployment trong file YAML. Ví dụ, file YAML sau mô tả một Deployment
chạy Docker image nginx:1.14.2:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # yêu cầu deployment chạy 2 pod khớp với template
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

1. Tạo một Deployment dựa trên file YAML:

   ```shell
   kubectl apply -f https://k8s.io/examples/application/deployment.yaml
   ```

1. Hiển thị thông tin về Deployment:

   ```shell
   kubectl describe deployment nginx-deployment
   ```

   Kết quả tương tự như sau:

   ```
   Name:     nginx-deployment
   Namespace:    default
   CreationTimestamp:  Tue, 30 Aug 2016 18:11:37 -0700
   Labels:     app=nginx
   Annotations:    deployment.kubernetes.io/revision=1
   Selector:   app=nginx
   Replicas:   2 desired | 2 updated | 2 total | 2 available | 0 unavailable
   StrategyType:   RollingUpdate
   MinReadySeconds:  0
   RollingUpdateStrategy:  1 max unavailable, 1 max surge
   Pod Template:
     Labels:       app=nginx
     Containers:
       nginx:
       Image:              nginx:1.14.2
       Port:               80/TCP
       Environment:        <none>
       Mounts:             <none>
     Volumes:              <none>
   Conditions:
     Type          Status  Reason
     ----          ------  ------
     Available     True    MinimumReplicasAvailable
     Progressing   True    NewReplicaSetAvailable
   OldReplicaSets:   <none>
   NewReplicaSet:    nginx-deployment-1771418926 (2/2 replicas created)
   No events.
   ```

1. Liệt kê các Pod do deployment tạo ra:

   ```shell
   kubectl get pods -l app=nginx
   ```

   Kết quả tương tự như sau:

   ```
   NAME                                READY     STATUS    RESTARTS   AGE
   nginx-deployment-1771418926-7o5ns   1/1       Running   0          16h
   nginx-deployment-1771418926-r18az   1/1       Running   0          16h
   ```

1. Hiển thị thông tin về một Pod:

   ```shell
   kubectl describe pod <pod-name>
   ```

   trong đó `<pod-name>` là tên của một trong các Pod của bạn.

## Cập nhật deployment (Updating the deployment)

Bạn có thể cập nhật deployment bằng cách apply một file YAML mới. File YAML sau
chỉ định rằng deployment cần được cập nhật để dùng nginx 1.16.1.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.16.1 # Cập nhật phiên bản nginx từ 1.14.2 lên 1.16.1
        ports:
        - containerPort: 80
```

1. Apply file YAML mới:

   ```shell
   kubectl apply -f https://k8s.io/examples/application/deployment-update.yaml
   ```

1. Quan sát deployment tạo các pod với tên mới và xóa các pod cũ:

   ```shell
   kubectl get pods -l app=nginx
   ```

## Mở rộng quy mô ứng dụng bằng cách tăng số replica (Scaling the application by increasing the replica count)

Bạn có thể tăng số lượng Pod trong Deployment của mình bằng cách apply một file YAML
mới. File YAML sau đặt `replicas` thành 4, tức là chỉ định Deployment cần có
bốn Pod:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 4 # Cập nhật replicas từ 2 lên 4
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.16.1
        ports:
        - containerPort: 80
```

1. Apply file YAML mới:

   ```shell
   kubectl apply -f https://k8s.io/examples/application/deployment-scale.yaml
   ```

1. Xác minh rằng Deployment có bốn Pod:

   ```shell
   kubectl get pods -l app=nginx
   ```

   Kết quả tương tự như sau:

   ```
   NAME                               READY     STATUS    RESTARTS   AGE
   nginx-deployment-148880595-4zdqq   1/1       Running   0          25s
   nginx-deployment-148880595-6zgi1   1/1       Running   0          25s
   nginx-deployment-148880595-fxcez   1/1       Running   0          2m
   nginx-deployment-148880595-rwovn   1/1       Running   0          2m
   ```

Để biết quy trình mở rộng quy mô chi tiết, bao gồm thu hẹp quy mô (scale down) và
đưa về không (scale to zero), hãy xem
[Scale một Deployment thủ công](346-scale-deployment-vi.md).

## Xóa một deployment (Deleting a deployment)

Xóa deployment theo tên:

```shell
kubectl delete deployment nginx-deployment
```

## ReplicationControllers -- cách làm cũ (ReplicationControllers -- the Old Way)

Cách được khuyến nghị để tạo một ứng dụng có nhiều bản sao (replicated application) là dùng
Deployment, và Deployment sẽ dùng ReplicaSet bên dưới. Trước khi Deployment và ReplicaSet
được thêm vào Kubernetes, các ứng dụng có nhiều bản sao được cấu hình bằng
[ReplicationController](70-replicationcontroller-vi.md).

## Tiếp theo (What's next)

- Tìm hiểu thêm về [đối tượng Deployment](63-deployment-vi.md).
- Tìm hiểu cách [cập nhật một Deployment mà không gây gián đoạn](348-update-deployment-rolling-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 4:

1. `kubectl describe deployment nginx-deployment` in ra dòng
   `NewReplicaSet: nginx-deployment-1771418926 (2/2 replicas created)`. Dòng này nói gì về quan hệ
   giữa Deployment và các Pod của nó?
2. **Câu bẫy.** Bạn apply file thứ hai, khác file đầu đúng một chỗ: `nginx:1.14.2` đổi thành
   `nginx:1.16.1`. Hai Pod đang chạy có được nâng image **tại chỗ** không? Bằng chứng nào trong bài
   cho thấy điều đó?
3. Trên `lab-k8s-master`, bạn muốn nâng Deployment từ 2 lên 4 replica theo đúng cách của bài này.
   Bạn làm gì, kiểm chứng bằng lệnh nào, và trong output kiểm chứng có chi tiết nào cho thấy hai Pod
   cũ **không** bị tạo lại?
4. Bài kết bằng mục *ReplicationControllers — cách làm cũ*. Theo bài, cách được khuyến nghị để chạy
   một ứng dụng có nhiều bản sao là gì, và nó dùng cái gì ở bên dưới?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Nó cho thấy **Deployment không quản Pod trực tiếp mà quản qua một ReplicaSet**. Tên ReplicaSet
   là tên Deployment cộng một hậu tố, và chính ReplicaSet đó mới là thứ đã tạo ra 2/2 replica. Cùng
   output còn có dòng `OldReplicaSets: <none>` — tức Deployment mới tạo nên chưa có ReplicaSet cũ
   nào. Tên Pod trong `kubectl get pods -l app=nginx` cũng mang đúng hậu tố của ReplicaSet đó.
2. **Không.** Bài mô tả thẳng ở bước sau lệnh apply: *quan sát deployment tạo các pod với tên mới và
   xóa các pod cũ*. Bằng chứng là **tên Pod đổi** — trước khi cập nhật là
   `nginx-deployment-1771418926-*`, sau khi cập nhật là một hậu tố khác. Pod là thứ bị thay thế, chứ
   không phải container bên trong Pod được nâng cấp tại chỗ. Chỗ dễ nhầm là nghĩ đổi `image` giống
   như đổi một giá trị cấu hình; thực ra nó tạo ra một pod template mới, và pod template mới thì
   sinh Pod mới.
3. Sửa `replicas` trong file cấu hình từ 2 thành **4** rồi `kubectl apply -f` lại file đó — không
   dùng lệnh riêng nào. Kiểm chứng bằng **`kubectl get pods -l app=nginx`**, phải thấy **bốn Pod**.
   Chi tiết cho thấy hai Pod cũ được giữ nguyên nằm ở cột `AGE`: hai Pod có tuổi `2m` (từ lần apply
   trước) còn hai Pod mới chỉ `25s`. Bốn Pod cùng mang một hậu tố ReplicaSet, vì `template` không đổi.
4. Cách được khuyến nghị là **Deployment**, và **Deployment sẽ dùng ReplicaSet bên dưới**.
   [ReplicationController](70-replicationcontroller-vi.md) là cách cấu hình ứng dụng nhiều bản sao
   **trước khi** Deployment và ReplicaSet được thêm vào Kubernetes — biết để nhận ra khi gặp cluster
   cũ, không dùng cho hệ thống mới.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

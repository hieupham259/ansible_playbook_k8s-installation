# Chạy ứng dụng có trạng thái đơn thực thể (Run a Single-Instance Stateful Application)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/run-application/run-single-instance-stateful-application/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 6 — Lưu trữ](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), dòng **Thực hành**, bài 4/4 ·
Kiểm chứng ở [Lab 6a — PV, PVC và StorageClass](labs/LAB-6A-PV-PVC-VA-STORAGECLASS.md), phần B2
(và B2.4 cho phần dữ liệu sống lâu hơn Pod).

Đây là **bài cuối của nhóm Thực hành giai đoạn 6**, và là mặt đối lập của bài
[343](343-run-replicated-stateful-application-vi.md) vừa đọc: cùng là MySQL, nhưng một thực thể,
chạy bằng Deployment thay vì StatefulSet. Đọc để thấy rõ khi nào Deployment là đủ và khi nào
không.

**Phải hiểu ở lần đọc này:**

- Ba tầng trong mục *Triển khai MySQL*: PV tĩnh ↔ PVC ↔ Pod. PV `mysql-pv-volume` và PVC
  `mysql-pv-claim` khớp nhau nhờ cùng khai `storageClassName: manual`, cùng `accessModes` và cùng
  dung lượng; Deployment **không nhắc tên PV** ở đâu cả, nó chỉ trỏ `claimName: mysql-pv-claim`.
- Mục *Cập nhật*, ràng buộc thứ nhất: **đừng scale ứng dụng** — bài nêu lý do là
  PersistentVolume bên dưới chỉ mount được vào **một** Pod; muốn ứng dụng có trạng thái dạng
  cluster thì phải dùng StatefulSet.
- Mục *Cập nhật*, ràng buộc thứ hai: bắt buộc `strategy: type: Recreate`. Rolling update không
  chạy được vì không thể có nhiều hơn một Pod cùng lúc; `Recreate` dừng Pod cũ **trước khi** tạo
  Pod mới.
- Mục *Truy cập thực thể MySQL*: `clusterIP: None` khiến tên DNS của Service phân giải **thẳng ra
  IP của Pod**. Bài nói rõ đây là cách tối ưu khi chỉ có một Pod đứng sau Service và bạn không
  định tăng số Pod.
- Mục *Xóa deployment*: PV **cấp phát thủ công** phải xóa thủ công, và tài nguyên bên dưới cũng
  phải tự giải phóng. Nếu dùng trình cấp phát động thì xóa PVC là PV tự đi theo.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| `hostPath: "/mnt/data"` trong `mysql-pv.yaml` | bài viết cho tình huống một đĩa cục bộ và không bàn gì về rủi ro của `hostPath` | bài [91](91-volumes-vi.md) và [92](92-persistent-volumes-vi.md) ở đầu [giai đoạn 6](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ); [Lab 6a](labs/LAB-6A-PV-PVC-VA-STORAGECLASS.md) B2.6 dựng đúng cái bẫy đó trên cluster nhiều node |
| Mật khẩu `MYSQL_ROOT_PASSWORD: password` đặt thẳng trong YAML | bài tự cảnh báo là không an toàn và chỉ sang Secret | bài [109](109-secret-vi.md) đã đọc ở [3b](00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod) |
| Các dòng `Conditions`, `OldReplicaSets`, `NewReplicaSet` trong output `kubectl describe deployment mysql` | thuộc cơ chế rollout của Deployment, không phải chuyện lưu trữ | bài [63](63-deployment-vi.md) đã đọc ở [4a](00-ALO-TRINH-ADMIN.md#4a-replicaset-deployment-và-rollout) |
| Nhánh "hoặc dùng trình cấp phát động với StorageClass mặc định" ở mục *Trước khi bạn bắt đầu* | ở lần đọc này bạn đi nhánh PV tĩnh, đúng như manifest của bài | [Lab 6a](labs/LAB-6A-PV-PVC-VA-STORAGECLASS.md) B3–B4 cài provisioner rồi làm lại bằng nhánh động |

---

Trang này hướng dẫn bạn cách chạy một ứng dụng có trạng thái (stateful) đơn thực thể (single-instance)
trong Kubernetes bằng cách sử dụng một PersistentVolume và một Deployment.
Ứng dụng này là MySQL.

## Mục tiêu (Objectives)

- Tạo một PersistentVolume tham chiếu đến một đĩa trong môi trường của bạn.
- Tạo một Deployment MySQL.
- Cung cấp MySQL cho các pod khác trong cluster tại một tên DNS đã biết.

## Trước khi bạn bắt đầu (Before you begin)

- Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình
  để giao tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất
  hai node không đóng vai trò máy chủ control plane. Nếu bạn chưa có cluster, bạn có thể
  tạo một cluster bằng
  [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
  hoặc sử dụng một trong các sân chơi (playground) Kubernetes sau:

  - [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
  - [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
  - [KodeKloud](https://kodekloud.com/public-playgrounds)

  Kubernetes server của bạn phải ở phiên bản v1.36 hoặc mới hơn.
  Để kiểm tra phiên bản, nhập `kubectl version`.

- Bạn cần có một [trình cấp phát PersistentVolume động (dynamic PersistentVolume provisioner)](98-dynamic-provisioning-vi.md)
  với một [StorageClass](96-storage-classes-vi.md) mặc định,
  hoặc tự [cấp phát tĩnh các PersistentVolume](92-persistent-volumes-vi.md#provisioning)
  để đáp ứng các [PersistentVolumeClaim](92-persistent-volumes-vi.md#persistentvolumeclaims)
  được dùng ở đây.

## Triển khai MySQL (Deploy MySQL)

Bạn có thể chạy một ứng dụng có trạng thái bằng cách tạo một Deployment Kubernetes
và kết nối nó với một PersistentVolume có sẵn thông qua một
PersistentVolumeClaim. Ví dụ, file YAML sau mô tả một
Deployment chạy MySQL và tham chiếu đến PersistentVolumeClaim. File này
định nghĩa một điểm mount volume cho /var/lib/mysql, rồi tạo một
PersistentVolumeClaim yêu cầu một volume 20G. Claim này được
đáp ứng bởi bất kỳ volume có sẵn nào thỏa mãn các yêu cầu,
hoặc bởi một trình cấp phát động (dynamic provisioner).

Lưu ý: Mật khẩu được định nghĩa ngay trong file yaml cấu hình, và điều này không an toàn. Xem
[Kubernetes Secrets](109-secret-vi.md)
để có một giải pháp an toàn.

```yaml
# application/mysql/mysql-deployment.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
  clusterIP: None
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:9
        name: mysql
        env:
          # Trong thực tế hãy dùng secret
        - name: MYSQL_ROOT_PASSWORD
          value: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```

```yaml
# application/mysql/mysql-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

1. Triển khai PV và PVC từ file YAML:

   ```shell
   kubectl apply -f https://k8s.io/examples/application/mysql/mysql-pv.yaml
   ```

1. Triển khai nội dung của file YAML:

   ```shell
   kubectl apply -f https://k8s.io/examples/application/mysql/mysql-deployment.yaml
   ```

1. Hiển thị thông tin về Deployment:

   ```shell
   kubectl describe deployment mysql
   ```

   Output tương tự như sau:

   ```
   Name:                 mysql
   Namespace:            default
   CreationTimestamp:    Tue, 01 Nov 2016 11:18:45 -0700
   Labels:               app=mysql
   Annotations:          deployment.kubernetes.io/revision=1
   Selector:             app=mysql
   Replicas:             1 desired | 1 updated | 1 total | 0 available | 1 unavailable
   StrategyType:         Recreate
   MinReadySeconds:      0
   Pod Template:
     Labels:       app=mysql
     Containers:
       mysql:
       Image:      mysql:9
       Port:       3306/TCP
       Environment:
         MYSQL_ROOT_PASSWORD:      password
       Mounts:
         /var/lib/mysql from mysql-persistent-storage (rw)
     Volumes:
       mysql-persistent-storage:
       Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
       ClaimName:  mysql-pv-claim
       ReadOnly:   false
   Conditions:
     Type          Status  Reason
     ----          ------  ------
     Available     False   MinimumReplicasUnavailable
     Progressing   True    ReplicaSetUpdated
   OldReplicaSets:       <none>
   NewReplicaSet:        mysql-63082529 (1/1 replicas created)
   Events:
     FirstSeen    LastSeen    Count    From                SubobjectPath    Type        Reason            Message
     ---------    --------    -----    ----                -------------    --------    ------            -------
     33s          33s         1        {deployment-controller }             Normal      ScalingReplicaSet Scaled up replica set mysql-63082529 to 1
   ```

1. Liệt kê các pod được Deployment tạo ra:

   ```shell
   kubectl get pods -l app=mysql
   ```

   Output tương tự như sau:

   ```
   NAME                   READY     STATUS    RESTARTS   AGE
   mysql-63082529-2z3ki   1/1       Running   0          3m
   ```

1. Kiểm tra PersistentVolumeClaim:

   ```shell
   kubectl describe pvc mysql-pv-claim
   ```

   Output tương tự như sau:

   ```
   Name:         mysql-pv-claim
   Namespace:    default
   StorageClass:
   Status:       Bound
   Volume:       mysql-pv-volume
   Labels:       <none>
   Annotations:    pv.kubernetes.io/bind-completed=yes
                   pv.kubernetes.io/bound-by-controller=yes
   Capacity:     20Gi
   Access Modes: RWO
   Events:       <none>
   ```

## Truy cập thực thể MySQL (Accessing the MySQL instance)

File YAML ở trên tạo ra một service cho phép
các Pod khác trong cluster truy cập cơ sở dữ liệu. Tùy chọn Service
`clusterIP: None` khiến tên DNS của Service phân giải trực tiếp thành
địa chỉ IP của Pod. Đây là cách tối ưu khi bạn chỉ có một Pod
đứng sau Service và bạn không có ý định tăng số lượng Pod.

Chạy một MySQL client để kết nối đến server:

```shell
kubectl run -it --rm --image=mysql:9 --restart=Never mysql-client -- mysql -h mysql -ppassword
```

Lệnh này tạo một Pod mới trong cluster chạy MySQL client
và kết nối nó đến server thông qua Service. Nếu kết nối thành công, bạn
biết rằng cơ sở dữ liệu MySQL có trạng thái của bạn đang chạy.

```
Waiting for pod default/mysql-client-274442439-zyp6i to be running, status is Pending, pod ready: false
If you don't see a command prompt, try pressing enter.

mysql>
```

## Cập nhật (Updating)

Image hoặc bất kỳ phần nào khác của Deployment có thể được cập nhật như thường lệ
bằng lệnh `kubectl apply`. Sau đây là một vài lưu ý phòng ngừa dành riêng
cho các ứng dụng có trạng thái:

- Đừng scale ứng dụng. Cách thiết lập này chỉ dành cho ứng dụng đơn thực thể.
  PersistentVolume bên dưới chỉ có thể được mount vào một
  Pod. Với các ứng dụng có trạng thái dạng cluster, xem
  [tài liệu về StatefulSet](65-statefulset-vi.md).
- Sử dụng `strategy:` `type: Recreate` trong file YAML cấu hình
  Deployment. Điều này chỉ thị Kubernetes _không_ sử dụng rolling
  update. Rolling update sẽ không hoạt động, vì bạn không thể có nhiều hơn
  một Pod chạy cùng lúc. Chiến lược `Recreate` sẽ dừng
  Pod thứ nhất trước khi tạo một Pod mới với cấu hình đã cập nhật.

## Xóa deployment (Deleting a deployment)

Xóa các object đã triển khai theo tên:

```shell
kubectl delete deployment,svc mysql
kubectl delete pvc mysql-pv-claim
kubectl delete pv mysql-pv-volume
```

Nếu bạn đã cấp phát PersistentVolume thủ công, bạn cũng cần xóa nó
thủ công, cũng như giải phóng tài nguyên bên dưới.
Nếu bạn dùng trình cấp phát động, nó sẽ tự động xóa
PersistentVolume khi thấy bạn đã xóa PersistentVolumeClaim.
Một số trình cấp phát động (chẳng hạn các trình cấp phát cho EBS và PD) cũng giải phóng
tài nguyên bên dưới khi xóa PersistentVolume.

## Tiếp theo (What's next)

- Tìm hiểu thêm về [các object Deployment](63-deployment-vi.md).

- Tìm hiểu thêm về [Triển khai ứng dụng](345-run-stateless-application-vi.md)

- [Tài liệu kubectl run](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#run)

- [Volumes](91-volumes-vi.md) và [Persistent Volumes](92-persistent-volumes-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 6:

1. Trong bài, thứ gì ghép PVC `mysql-pv-claim` với PV `mysql-pv-volume`? Deployment có chỗ nào
   nhắc tới tên PV không?
2. **Câu bẫy.** Bài bắt buộc `strategy: type: Recreate`. Nếu bạn bỏ dòng đó và để Deployment chạy
   chiến lược mặc định thì hỏng ở đâu?
3. Service của bài đặt `clusterIP: None`. Nó khác Service thường ở chỗ nào, và vì sao bài nói cách
   này "tối ưu" đúng cho trường hợp một Pod?
4. Trên cluster lab, `lab-k8s-master` bị taint nên workload nằm ở `lab-k8s-worker1` và
   `lab-k8s-worker2`. Bạn bỏ ngoài tai lời khuyên của bài và chạy
   `kubectl scale deployment mysql --replicas=2`. Theo lập luận của bài, Pod thứ hai gặp chuyện gì,
   và bài bảo dùng gì thay thế?
5. Mục *Xóa deployment* liệt kê bốn lệnh xóa. Vì bạn tự tay tạo PV bằng `mysql-pv.yaml`, lệnh nào
   trong đó là **bắt buộc thêm** so với khi dùng trình cấp phát động?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **`storageClassName: manual`, cùng với `accessModes: ReadWriteOnce` và dung lượng `20Gi` khớp
   nhau** — cả hai object cùng khai một chuỗi do bạn tự đặt, và claim được đáp ứng bởi "bất kỳ
   volume có sẵn nào thỏa mãn các yêu cầu". Deployment **không** nhắc tên PV ở bất cứ đâu; nó chỉ
   khai `persistentVolumeClaim.claimName: mysql-pv-claim`. Tên PV chỉ hiện ra ở output
   `kubectl describe pvc`, dòng `Volume: mysql-pv-volume`.
2. **Rollout kẹt.** Chiến lược mặc định là rolling update, tức tạo Pod mới **trước khi** bỏ Pod cũ
   — mà bài đã nói PersistentVolume bên dưới chỉ mount được vào một Pod, và "bạn không thể có
   nhiều hơn một Pod chạy cùng lúc". `Recreate` tồn tại chính để dừng Pod thứ nhất trước rồi mới
   tạo Pod mới với cấu hình đã cập nhật.
3. `clusterIP: None` khiến **tên DNS của Service phân giải trực tiếp thành địa chỉ IP của Pod**,
   thay vì thành một cluster IP ảo đứng trước nhóm Pod. Bài gọi đó là tối ưu **vì chỉ có một Pod**
   đứng sau Service và bạn không có ý định tăng số lượng Pod — không có gì để cân bằng tải, nên
   thêm một lớp IP ảo là thừa.
4. **Pod thứ hai không dùng được volume đó** — PersistentVolume bên dưới chỉ có thể được mount vào
   một Pod, và điều này không phụ thuộc vào việc nó rơi vào `lab-k8s-worker1` hay
   `lab-k8s-worker2`. Bài nói thẳng: cách thiết lập này chỉ dành cho ứng dụng **đơn thực thể**;
   với ứng dụng có trạng thái dạng cluster thì dùng [StatefulSet](65-statefulset-vi.md) — đúng
   thứ bài [343](343-run-replicated-stateful-application-vi.md) trình bày.
5. **`kubectl delete pv mysql-pv-volume`.** Bài nói rõ: PersistentVolume cấp phát thủ công thì
   phải xóa thủ công, và bạn còn phải tự giải phóng tài nguyên bên dưới. Với trình cấp phát động,
   PV tự bị xóa khi nó thấy bạn đã xóa PVC.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng. Trả lời trôi chảy được cả bốn bài thực
hành của nhóm rồi thì mở [Lab 6a — PV, PVC và StorageClass](labs/LAB-6A-PV-PVC-VA-STORAGECLASS.md):
B1 chạy lại bài [280](280-configure-volume-storage-vi.md), B2 chạy lại bài này, B6 trả nợ #2 của
bài [343](343-run-replicated-stateful-application-vi.md), và B7 chạy bài
[277](277-configure-projected-volume-vi.md).

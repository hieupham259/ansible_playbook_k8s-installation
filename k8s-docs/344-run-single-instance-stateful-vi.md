# Chạy ứng dụng có trạng thái đơn thực thể (Run a Single-Instance Stateful Application)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/run-application/run-single-instance-stateful-application/

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

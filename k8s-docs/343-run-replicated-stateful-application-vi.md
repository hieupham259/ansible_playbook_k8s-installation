# Chạy ứng dụng có trạng thái được nhân bản (Run a Replicated Stateful Application)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/

Trang này hướng dẫn cách chạy một ứng dụng có trạng thái (stateful) được nhân bản (replicated) bằng
StatefulSet.
Ứng dụng này là một cơ sở dữ liệu MySQL được nhân bản. Topology của ví dụ gồm một
máy chủ primary duy nhất và nhiều replica, sử dụng cơ chế replication bất đồng bộ
dựa trên hàng (asynchronous row-based replication).

> **Ghi chú:**
> **Đây không phải là cấu hình dành cho môi trường production**. Các thiết lập MySQL được giữ ở
> mặc định không an toàn để tập trung vào các mẫu hình (pattern) chung khi chạy ứng dụng có trạng thái
> trong Kubernetes.

## Trước khi bạn bắt đầu (Before you begin) {#before-you-begin}

- Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình
  để giao tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất
  hai node không đóng vai trò máy chủ control plane. Nếu bạn chưa có cluster, bạn có thể
  tạo một cluster bằng
  [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
  hoặc sử dụng một trong các sân chơi (playground) Kubernetes sau:

  - [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
  - [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
  - [KodeKloud](https://kodekloud.com/public-playgrounds)

- Bạn cần có một [trình cấp phát PersistentVolume động (dynamic PersistentVolume provisioner)](98-dynamic-provisioning-vi.md)
  với một [StorageClass](96-storage-classes-vi.md) mặc định,
  hoặc tự [cấp phát tĩnh các PersistentVolume](92-persistent-volumes-vi.md#provisioning)
  để đáp ứng các [PersistentVolumeClaim](92-persistent-volumes-vi.md#persistentvolumeclaims)
  được dùng ở đây.
- Hướng dẫn này giả định bạn đã quen thuộc với
  [PersistentVolumes](92-persistent-volumes-vi.md)
  và [StatefulSets](65-statefulset-vi.md),
  cũng như các khái niệm cốt lõi khác như [Pods](46-pods-vi.md),
  [Services](82-service-vi.md), và
  [ConfigMaps](275-configure-pod-configmap-vi.md).
- Có chút quen thuộc với MySQL sẽ hữu ích, nhưng hướng dẫn này hướng tới trình bày
  các mẫu hình chung mà bạn có thể áp dụng cho các hệ thống khác.
- Bạn đang dùng namespace default hoặc một namespace khác không chứa các object gây xung đột.
- Bạn cần có CPU tương thích AMD64.

## Mục tiêu (Objectives)

- Triển khai một topology MySQL được nhân bản bằng StatefulSet.
- Gửi lưu lượng từ MySQL client.
- Quan sát khả năng chống chịu khi có downtime.
- Scale StatefulSet lên và xuống.

## Triển khai MySQL (Deploy MySQL)

Ví dụ triển khai MySQL này gồm một ConfigMap, hai Service,
và một StatefulSet.

### Tạo ConfigMap (Create a ConfigMap) {#configmap}

Tạo ConfigMap từ file cấu hình YAML sau:

```yaml
# application/mysql/mysql-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql
  labels:
    app: mysql
    app.kubernetes.io/name: mysql
data:
  primary.cnf: |
    # Chỉ áp dụng cấu hình này trên máy chủ primary.
    [mysqld]
    log-bin
  replica.cnf: |
    # Chỉ áp dụng cấu hình này trên các replica.
    [mysqld]
    super-read-only
```

```shell
kubectl apply -f https://k8s.io/examples/application/mysql/mysql-configmap.yaml
```

ConfigMap này cung cấp các phần ghi đè cho `my.cnf`, cho phép bạn kiểm soát cấu hình
một cách độc lập trên máy chủ MySQL primary và các replica của nó.
Trong trường hợp này, bạn muốn máy chủ primary có thể phục vụ log replication cho các replica,
và bạn muốn các replica từ chối mọi thao tác ghi không đến qua đường replication.

Bản thân ConfigMap không có gì đặc biệt khiến các phần khác nhau
được áp dụng cho các Pod khác nhau.
Mỗi Pod tự quyết định phần nào cần dùng khi nó khởi tạo,
dựa trên thông tin do StatefulSet controller cung cấp.

### Tạo các Service (Create Services) {#services}

Tạo các Service từ file cấu hình YAML sau:

```yaml
# application/mysql/mysql-services.yaml
# Headless service cho các bản ghi DNS ổn định của các thành viên StatefulSet.
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
    app.kubernetes.io/name: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  clusterIP: None
  selector:
    app: mysql
---
# Service cho client kết nối tới bất kỳ instance MySQL nào để đọc.
# Để ghi, bạn phải kết nối tới primary: mysql-0.mysql.
apiVersion: v1
kind: Service
metadata:
  name: mysql-read
  labels:
    app: mysql
    app.kubernetes.io/name: mysql
    readonly: "true"
spec:
  ports:
  - name: mysql
    port: 3306
  selector:
    app: mysql
```

```shell
kubectl apply -f https://k8s.io/examples/application/mysql/mysql-services.yaml
```

Headless Service cung cấp nơi lưu trú cho các bản ghi DNS mà các controller của StatefulSet
tạo ra cho từng Pod thuộc tập hợp đó.
Vì headless Service có tên là `mysql`, các Pod có thể được truy cập bằng cách
phân giải `<pod-name>.mysql` từ bên trong bất kỳ Pod nào khác trong cùng cluster Kubernetes
và cùng namespace.

Service cho client, có tên `mysql-read`, là một Service thông thường với
cluster IP riêng, phân phối các kết nối đến tất cả các Pod MySQL đang báo cáo
trạng thái Ready. Tập các endpoint tiềm năng bao gồm máy chủ MySQL primary và tất cả
các replica.

Lưu ý rằng chỉ các truy vấn đọc mới có thể dùng Service client được cân bằng tải này.
Vì chỉ có một máy chủ MySQL primary duy nhất, các client nên kết nối trực tiếp đến
Pod MySQL primary (thông qua bản ghi DNS của nó bên trong headless Service) để thực hiện
các thao tác ghi.

### Tạo StatefulSet (Create the StatefulSet) {#statefulset}

Cuối cùng, tạo StatefulSet từ file cấu hình YAML sau:

```yaml
# application/mysql/mysql-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
      app.kubernetes.io/name: mysql
  serviceName: mysql
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
        app.kubernetes.io/name: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:5.7
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Sinh server-id của mysql từ chỉ số thứ tự (ordinal index) của Pod.
          [[ $HOSTNAME =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          # Cộng thêm một khoảng bù (offset) để tránh giá trị dành riêng server-id=0.
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          # Sao chép các file conf.d thích hợp từ config-map sang emptyDir.
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/config-map/primary.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/replica.cnf /mnt/conf.d/
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      - name: clone-mysql
        image: gcr.io/google-samples/xtrabackup:1.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Bỏ qua bước clone nếu dữ liệu đã tồn tại.
          [[ -d /var/lib/mysql/mysql ]] && exit 0
          # Bỏ qua bước clone trên primary (ordinal index 0).
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          [[ $ordinal -eq 0 ]] && exit 0
          # Clone dữ liệu từ Pod liền trước.
          ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql
          # Chuẩn bị bản backup.
          xtrabackup --prepare --target-dir=/var/lib/mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ALLOW_EMPTY_PASSWORD
          value: "1"
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            # Kiểm tra rằng ta có thể thực thi truy vấn qua TCP (skip-networking đang tắt).
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 2
          timeoutSeconds: 1
      - name: xtrabackup
        image: gcr.io/google-samples/xtrabackup:1.0
        ports:
        - name: xtrabackup
          containerPort: 3307
        command:
        - bash
        - "-c"
        - |
          set -ex
          cd /var/lib/mysql

          # Xác định vị trí binlog của dữ liệu đã clone, nếu có.
          if [[ -f xtrabackup_slave_info && "x$(<xtrabackup_slave_info)" != "x" ]]; then
            # XtraBackup đã sinh sẵn một phần truy vấn "CHANGE MASTER TO"
            # vì chúng ta clone từ một replica có sẵn. (Cần bỏ dấu chấm phẩy ở cuối!)
            cat xtrabackup_slave_info | sed -E 's/;$//g' > change_master_to.sql.in
            # Bỏ qua xtrabackup_binlog_info trong trường hợp này (nó vô dụng).
            rm -f xtrabackup_slave_info xtrabackup_binlog_info
          elif [[ -f xtrabackup_binlog_info ]]; then
            # Chúng ta clone trực tiếp từ primary. Phân tích vị trí binlog.
            [[ `cat xtrabackup_binlog_info` =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
            rm -f xtrabackup_binlog_info xtrabackup_slave_info
            echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                  MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
          fi

          # Kiểm tra xem có cần hoàn tất bước clone bằng cách khởi động replication hay không.
          if [[ -f change_master_to.sql.in ]]; then
            echo "Waiting for mysqld to be ready (accepting connections)"
            until mysql -h 127.0.0.1 -e "SELECT 1"; do sleep 1; done

            echo "Initializing replication from clone position"
            mysql -h 127.0.0.1 \
                  -e "$(<change_master_to.sql.in), \
                          MASTER_HOST='mysql-0.mysql', \
                          MASTER_USER='root', \
                          MASTER_PASSWORD='', \
                          MASTER_CONNECT_RETRY=10; \
                        START SLAVE;" || exit 1
            # Trong trường hợp container khởi động lại, chỉ thử thao tác này tối đa một lần.
            mv change_master_to.sql.in change_master_to.sql.orig
          fi

          # Khởi động một server để gửi bản backup khi được các Pod khác yêu cầu.
          exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
            "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

```shell
kubectl apply -f https://k8s.io/examples/application/mysql/mysql-statefulset.yaml
```

Bạn có thể theo dõi tiến trình khởi động bằng cách chạy:

```shell
kubectl get pods -l app=mysql --watch
```

Sau một lúc, bạn sẽ thấy cả 3 Pod chuyển sang trạng thái `Running`:

```
NAME      READY     STATUS    RESTARTS   AGE
mysql-0   2/2       Running   0          2m
mysql-1   2/2       Running   0          1m
mysql-2   2/2       Running   0          1m
```

Nhấn **Ctrl+C** để dừng lệnh watch.

> **Ghi chú:**
> Nếu bạn không thấy tiến triển nào, hãy đảm bảo bạn đã bật trình cấp phát PersistentVolume
> động, như đã đề cập trong phần [điều kiện tiên quyết](#before-you-begin).

Manifest này sử dụng nhiều kỹ thuật để quản lý các Pod có trạng thái như một phần của
StatefulSet. Mục tiếp theo sẽ nêu bật một số kỹ thuật đó để giải thích
điều gì xảy ra khi StatefulSet tạo các Pod.

## Tìm hiểu quá trình khởi tạo Pod có trạng thái (Understanding stateful Pod initialization)

StatefulSet controller khởi động các Pod từng cái một, theo thứ tự
chỉ số thứ tự (ordinal index) của chúng.
Nó đợi cho đến khi mỗi Pod báo cáo trạng thái Ready rồi mới khởi động Pod tiếp theo.

Ngoài ra, controller gán cho mỗi Pod một cái tên duy nhất, ổn định có dạng
`<statefulset-name>-<ordinal-index>`, kết quả là các Pod có tên `mysql-0`,
`mysql-1`, và `mysql-2`.

Pod template trong manifest StatefulSet ở trên tận dụng các thuộc tính này
để thực hiện việc khởi động replication MySQL một cách có trật tự.

### Sinh cấu hình (Generating configuration)

Trước khi khởi động bất kỳ container nào trong Pod spec, Pod trước tiên chạy các
[init container](50-init-containers-vi.md)
theo thứ tự đã định nghĩa.

Init container đầu tiên, tên là `init-mysql`, sinh ra các file cấu hình MySQL
đặc biệt dựa trên chỉ số thứ tự.

Script tự xác định chỉ số thứ tự của chính nó bằng cách trích xuất nó từ cuối
tên Pod, được trả về bởi lệnh `hostname`.
Sau đó nó lưu chỉ số thứ tự đó (kèm một khoảng bù số học để tránh các giá trị dành riêng)
vào một file tên là `server-id.cnf` trong thư mục `conf.d` của MySQL.
Việc này chuyển định danh duy nhất, ổn định do StatefulSet cung cấp
sang miền định danh server ID của MySQL, vốn cũng yêu cầu các thuộc tính tương tự.

Script trong container `init-mysql` cũng áp dụng `primary.cnf` hoặc
`replica.cnf` từ ConfigMap bằng cách sao chép nội dung vào `conf.d`.
Vì topology của ví dụ gồm một máy chủ MySQL primary duy nhất và một số lượng bất kỳ
replica, script gán chỉ số thứ tự `0` làm máy chủ primary, và tất cả
các Pod còn lại làm replica.
Kết hợp với [đảm bảo thứ tự triển khai](65-statefulset-vi.md#deployment-and-scaling-guarantees)
của StatefulSet controller,
điều này đảm bảo máy chủ MySQL primary đã Ready trước khi tạo các replica, để chúng có thể bắt đầu
replication.

### Sao chép dữ liệu có sẵn (Cloning existing data)

Nhìn chung, khi một Pod mới gia nhập tập hợp với vai trò replica, nó phải giả định rằng máy chủ MySQL
primary có thể đã có dữ liệu. Nó cũng phải giả định rằng các log replication
có thể không truy ngược được về tận thời điểm ban đầu.
Những giả định thận trọng này là chìa khóa cho phép một StatefulSet đang chạy
scale lên và xuống theo thời gian, thay vì bị cố định ở kích thước ban đầu.

Init container thứ hai, tên là `clone-mysql`, thực hiện thao tác clone trên
một Pod replica ở lần đầu tiên nó khởi động với một PersistentVolume trống.
Nghĩa là nó sao chép toàn bộ dữ liệu hiện có từ một Pod khác đang chạy,
để trạng thái cục bộ của nó đủ nhất quán để bắt đầu replication từ máy chủ primary.

Bản thân MySQL không cung cấp cơ chế để làm việc này, nên ví dụ sử dụng một
công cụ mã nguồn mở phổ biến tên là Percona XtraBackup.
Trong quá trình clone, máy chủ MySQL nguồn có thể bị giảm hiệu năng.
Để giảm thiểu ảnh hưởng đến máy chủ MySQL primary, script chỉ thị mỗi Pod clone
từ Pod có chỉ số thứ tự nhỏ hơn một đơn vị.
Cách này hoạt động được vì StatefulSet controller luôn đảm bảo Pod `N`
đã Ready trước khi khởi động Pod `N+1`.

### Bắt đầu replication (Starting replication)

Sau khi các init container hoàn thành thành công, các container thông thường sẽ chạy.
Các Pod MySQL gồm một container `mysql` chạy server `mysqld`
thực sự, và một container `xtrabackup` đóng vai trò
[sidecar](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns).

Sidecar `xtrabackup` xem xét các file dữ liệu đã clone và xác định xem
có cần khởi tạo replication MySQL trên replica hay không.
Nếu cần, nó đợi `mysqld` sẵn sàng rồi thực thi các lệnh
`CHANGE MASTER TO` và `START SLAVE` với các tham số replication
được trích xuất từ các file clone của XtraBackup.

Một khi replica bắt đầu replication, nó ghi nhớ máy chủ MySQL primary của mình và
tự động kết nối lại nếu server khởi động lại hoặc kết nối bị đứt.
Ngoài ra, vì các replica tìm máy chủ primary qua tên DNS ổn định của nó
(`mysql-0.mysql`), chúng tự động tìm thấy máy chủ primary ngay cả khi nó nhận
Pod IP mới do bị lập lịch lại (rescheduled).

Cuối cùng, sau khi bắt đầu replication, container `xtrabackup` lắng nghe
các kết nối từ những Pod khác yêu cầu clone dữ liệu.
Server này duy trì hoạt động vô thời hạn phòng trường hợp StatefulSet scale lên, hoặc
trường hợp Pod tiếp theo mất PersistentVolumeClaim và cần thực hiện lại việc clone.

## Gửi lưu lượng từ client (Sending client traffic)

Bạn có thể gửi các truy vấn thử nghiệm đến máy chủ MySQL primary (hostname `mysql-0.mysql`)
bằng cách chạy một container tạm thời với image `mysql:5.7` và chạy
binary client `mysql`.

```shell
kubectl run mysql-client --image=mysql:5.7 -i --rm --restart=Never --\
  mysql -h mysql-0.mysql <<EOF
CREATE DATABASE test;
CREATE TABLE test.messages (message VARCHAR(250));
INSERT INTO test.messages VALUES ('hello');
EOF
```

Sử dụng hostname `mysql-read` để gửi truy vấn thử nghiệm đến bất kỳ server nào đang báo cáo
trạng thái Ready:

```shell
kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  mysql -h mysql-read -e "SELECT * FROM test.messages"
```

Bạn sẽ nhận được output như sau:

```
Waiting for pod default/mysql-client to be running, status is Pending, pod ready: false
+---------+
| message |
+---------+
| hello   |
+---------+
pod "mysql-client" deleted
```

Để chứng minh rằng Service `mysql-read` phân phối kết nối giữa các
server, bạn có thể chạy `SELECT @@server_id` trong một vòng lặp:

```shell
kubectl run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
  bash -ic "while sleep 1; do mysql -h mysql-read -e 'SELECT @@server_id,NOW()'; done"
```

Bạn sẽ thấy giá trị `@@server_id` được báo cáo thay đổi ngẫu nhiên, vì một
endpoint khác nhau có thể được chọn ở mỗi lần thử kết nối:

```
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         100 | 2006-01-02 15:04:05 |
+-------------+---------------------+
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         102 | 2006-01-02 15:04:06 |
+-------------+---------------------+
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         101 | 2006-01-02 15:04:07 |
+-------------+---------------------+
```

Bạn có thể nhấn **Ctrl+C** khi muốn dừng vòng lặp, nhưng nên giữ nó
chạy trong một cửa sổ khác để bạn có thể quan sát hiệu ứng của các bước tiếp theo.

## Mô phỏng sự cố Pod và Node (Simulate Pod and Node failure) {#simulate-pod-and-node-downtime}

Để minh họa tính sẵn sàng (availability) tăng lên khi đọc từ nhóm các replica
thay vì từ một server duy nhất, hãy giữ vòng lặp `SELECT @@server_id` ở trên
tiếp tục chạy trong khi bạn buộc một Pod rời khỏi trạng thái Ready.

### Phá vỡ Readiness probe (Break the Readiness probe)

[Readiness probe](274-configure-probes-vi.md#define-readiness-probes)
của container `mysql` chạy lệnh `mysql -h 127.0.0.1 -e 'SELECT 1'`
để chắc chắn rằng server đang hoạt động và có thể thực thi truy vấn.

Một cách để buộc readiness probe này thất bại là phá vỡ lệnh đó:

```shell
kubectl exec mysql-2 -c mysql -- mv /usr/bin/mysql /usr/bin/mysql.off
```

Lệnh này can thiệp vào hệ thống file của container thực tế trong Pod `mysql-2` và
đổi tên lệnh `mysql` để readiness probe không thể tìm thấy nó.
Sau vài giây, Pod sẽ báo cáo một trong các container của nó không Ready,
điều mà bạn có thể kiểm tra bằng cách chạy:

```shell
kubectl get pod mysql-2
```

Hãy quan sát giá trị `1/2` trong cột `READY`:

```
NAME      READY     STATUS    RESTARTS   AGE
mysql-2   1/2       Running   0          3m
```

Lúc này, bạn sẽ thấy vòng lặp `SELECT @@server_id` vẫn tiếp tục chạy,
nhưng không bao giờ báo cáo `102` nữa.
Nhớ lại rằng script `init-mysql` định nghĩa `server-id` là `100 + $ordinal`,
nên server ID `102` tương ứng với Pod `mysql-2`.

Bây giờ hãy sửa lại Pod và nó sẽ xuất hiện trở lại trong output của vòng lặp
sau vài giây:

```shell
kubectl exec mysql-2 -c mysql -- mv /usr/bin/mysql.off /usr/bin/mysql
```

### Xóa các Pod (Delete Pods)

StatefulSet cũng tạo lại các Pod nếu chúng bị xóa, tương tự như cách một
ReplicaSet làm với các Pod phi trạng thái (stateless).

```shell
kubectl delete pod mysql-2
```

StatefulSet controller nhận thấy không còn Pod `mysql-2` nào tồn tại nữa,
và tạo một Pod mới có cùng tên và liên kết với cùng
PersistentVolumeClaim.
Bạn sẽ thấy server ID `102` biến mất khỏi output của vòng lặp trong một khoảng thời gian
rồi tự quay trở lại.

### Drain một Node (Drain a Node)

Nếu cluster Kubernetes của bạn có nhiều Node, bạn có thể mô phỏng downtime của Node
(chẳng hạn như khi các Node được nâng cấp) bằng cách thực hiện lệnh
[drain](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#drain).

Trước tiên hãy xác định một trong các Pod MySQL đang nằm trên Node nào:

```shell
kubectl get pod mysql-2 -o wide
```

Tên Node sẽ hiển thị ở cột cuối cùng:

```
NAME      READY     STATUS    RESTARTS   AGE       IP            NODE
mysql-2   2/2       Running   0          15m       10.244.5.27   kubernetes-node-9l2t
```

Sau đó, drain Node bằng cách chạy lệnh sau, lệnh này cordon (phong tỏa) Node để
không có Pod mới nào được lập lịch lên đó, rồi trục xuất (evict) mọi Pod hiện có.
Thay `<node-name>` bằng tên của Node bạn tìm được ở bước trước.

> **Thận trọng:**
> Việc drain một Node có thể ảnh hưởng đến các workload và ứng dụng khác
> đang chạy trên cùng node đó. Chỉ thực hiện bước sau đây trên một cluster
> thử nghiệm.

```shell
# Xem khuyến cáo ở trên về ảnh hưởng tới các workload khác
kubectl drain <node-name> --force --delete-emptydir-data --ignore-daemonsets
```

Bây giờ bạn có thể quan sát Pod được lập lịch lại trên một Node khác:

```shell
kubectl get pod mysql-2 -o wide --watch
```

Kết quả sẽ trông giống như sau:

```
NAME      READY   STATUS          RESTARTS   AGE       IP            NODE
mysql-2   2/2     Terminating     0          15m       10.244.1.56   kubernetes-node-9l2t
[...]
mysql-2   0/2     Pending         0          0s        <none>        kubernetes-node-fjlm
mysql-2   0/2     Init:0/2        0          0s        <none>        kubernetes-node-fjlm
mysql-2   0/2     Init:1/2        0          20s       10.244.5.32   kubernetes-node-fjlm
mysql-2   0/2     PodInitializing 0          21s       10.244.5.32   kubernetes-node-fjlm
mysql-2   1/2     Running         0          22s       10.244.5.32   kubernetes-node-fjlm
mysql-2   2/2     Running         0          30s       10.244.5.32   kubernetes-node-fjlm
```

Và một lần nữa, bạn sẽ thấy server ID `102` biến mất khỏi output của vòng lặp
`SELECT @@server_id` trong một khoảng thời gian rồi quay trở lại.

Bây giờ hãy uncordon Node để đưa nó trở về trạng thái bình thường:

```shell
kubectl uncordon <node-name>
```

## Scale số lượng replica (Scaling the number of replicas)

Khi bạn sử dụng MySQL replication, bạn có thể scale năng lực xử lý truy vấn đọc bằng cách
thêm các replica.
Với một StatefulSet, bạn có thể làm điều này chỉ với một lệnh duy nhất:

```shell
kubectl scale statefulset mysql  --replicas=5
```

Theo dõi các Pod mới được khởi động bằng cách chạy:

```shell
kubectl get pods -l app=mysql --watch
```

Khi chúng đã chạy, bạn sẽ thấy các server ID `103` và `104` bắt đầu xuất hiện trong
output của vòng lặp `SELECT @@server_id`.

Bạn cũng có thể xác minh rằng các server mới này có dữ liệu bạn đã thêm vào từ trước khi chúng
tồn tại:

```shell
kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  mysql -h mysql-3.mysql -e "SELECT * FROM test.messages"
```

```
Waiting for pod default/mysql-client to be running, status is Pending, pod ready: false
+---------+
| message |
+---------+
| hello   |
+---------+
pod "mysql-client" deleted
```

Việc scale ngược xuống cũng diễn ra liền mạch:

```shell
kubectl scale statefulset mysql --replicas=3
```

> **Ghi chú:**
> Mặc dù việc scale lên sẽ tự động tạo các PersistentVolumeClaim mới,
> việc scale xuống không tự động xóa các PVC này.
>
> Điều này cho bạn quyền lựa chọn: giữ lại các PVC đã được khởi tạo để việc
> scale lên trở lại nhanh hơn, hoặc trích xuất dữ liệu trước khi xóa chúng.

Bạn có thể thấy điều này bằng cách chạy:

```shell
kubectl get pvc -l app=mysql
```

Kết quả cho thấy cả 5 PVC vẫn tồn tại, mặc dù đã scale
StatefulSet xuống còn 3:

```
NAME           STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
data-mysql-0   Bound     pvc-8acbf5dc-b103-11e6-93fa-42010a800002   10Gi       RWO           20m
data-mysql-1   Bound     pvc-8ad39820-b103-11e6-93fa-42010a800002   10Gi       RWO           20m
data-mysql-2   Bound     pvc-8ad69a6d-b103-11e6-93fa-42010a800002   10Gi       RWO           20m
data-mysql-3   Bound     pvc-50043c45-b1c5-11e6-93fa-42010a800002   10Gi       RWO           2m
data-mysql-4   Bound     pvc-500a9957-b1c5-11e6-93fa-42010a800002   10Gi       RWO           2m
```

Nếu bạn không có ý định tái sử dụng các PVC dư thừa, bạn có thể xóa chúng:

```shell
kubectl delete pvc data-mysql-3
kubectl delete pvc data-mysql-4
```

## Dọn dẹp (Cleaning up)

1. Hủy vòng lặp `SELECT @@server_id` bằng cách nhấn **Ctrl+C** trong terminal của nó,
   hoặc chạy lệnh sau từ một terminal khác:

   ```shell
   kubectl delete pod mysql-client-loop --now
   ```

1. Xóa StatefulSet. Việc này cũng bắt đầu quá trình terminate các Pod.

   ```shell
   kubectl delete statefulset mysql
   ```

1. Xác minh rằng các Pod đã biến mất.
   Chúng có thể cần một khoảng thời gian để hoàn tất việc terminate.

   ```shell
   kubectl get pods -l app=mysql
   ```

   Bạn sẽ biết các Pod đã terminate xong khi lệnh trên trả về:

   ```
   No resources found.
   ```

1. Xóa ConfigMap, các Service, và các PersistentVolumeClaim.

   ```shell
   kubectl delete configmap,service,pvc -l app=mysql
   ```

1. Nếu bạn đã cấp phát các PersistentVolume thủ công, bạn cũng cần xóa chúng
   thủ công, cũng như giải phóng các tài nguyên bên dưới.
   Nếu bạn dùng trình cấp phát động, nó sẽ tự động xóa các
   PersistentVolume khi thấy bạn đã xóa các PersistentVolumeClaim.
   Một số trình cấp phát động (chẳng hạn các trình cấp phát cho EBS và PD) cũng giải phóng
   tài nguyên bên dưới khi xóa các PersistentVolume.

## Tiếp theo (What's next)

- Tìm hiểu thêm về [scale một StatefulSet](347-scale-stateful-set-vi.md).
- Tìm hiểu thêm về [debug một StatefulSet](302-debug-statefulset-vi.md).
- Tìm hiểu thêm về [xóa một StatefulSet](340-delete-stateful-set-vi.md).
- Tìm hiểu thêm về [buộc xóa các Pod của StatefulSet](341-force-delete-stateful-set-pod-vi.md).
- Xem [kho Helm Charts](https://artifacthub.io/)
  để tìm các ví dụ khác về ứng dụng có trạng thái.

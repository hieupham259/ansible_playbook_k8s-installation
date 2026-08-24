# Cấu hình nhiều Scheduler (Configure Multiple Schedulers)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/

Kubernetes đi kèm một scheduler mặc định, được mô tả
[tại đây](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/).
Nếu scheduler mặc định không đáp ứng nhu cầu của bạn, bạn có thể tự hiện thực một scheduler
riêng. Hơn nữa, bạn thậm chí có thể chạy đồng thời nhiều scheduler song song với scheduler mặc
định, rồi chỉ dẫn cho Kubernetes biết mỗi Pod của bạn cần dùng scheduler nào. Hãy cùng tìm hiểu
cách chạy nhiều scheduler trong Kubernetes qua một ví dụ.

Mô tả chi tiết cách hiện thực một scheduler nằm ngoài phạm vi của tài liệu này. Vui lòng tham
khảo phần hiện thực kube-scheduler trong
[pkg/scheduler](https://github.com/kubernetes/kubernetes/tree/master/pkg/scheduler)
thuộc thư mục mã nguồn Kubernetes để có một ví dụ chuẩn mực.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
hoặc dùng một trong các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, hãy chạy `kubectl version`.

## Đóng gói scheduler (Package the scheduler)

Hãy đóng gói binary scheduler của bạn thành một container image. Với mục đích của ví dụ này,
bạn có thể dùng chính scheduler mặc định (kube-scheduler) làm scheduler thứ hai.
Clone [mã nguồn Kubernetes từ GitHub](https://github.com/kubernetes/kubernetes) rồi build mã
nguồn đó.

```shell
git clone https://github.com/kubernetes/kubernetes.git
cd kubernetes
make
```

Tạo một container image chứa binary kube-scheduler. Dưới đây là `Dockerfile` để build image:

```docker
FROM busybox
ADD ./_output/local/bin/linux/amd64/kube-scheduler /usr/local/bin/kube-scheduler
```

Lưu file với tên `Dockerfile`, build image rồi push nó lên một registry. Ví dụ này push image lên
[Google Container Registry (GCR)](https://cloud.google.com/container-registry/).
Để biết thêm chi tiết, vui lòng đọc
[tài liệu](https://cloud.google.com/container-registry/docs/) của GCR. Ngoài ra, bạn cũng có thể
dùng [docker hub](https://hub.docker.com/search?q=). Để biết thêm chi tiết, hãy tham khảo
[tài liệu](https://docs.docker.com/docker-hub/repos/create/#create-a-repository) của docker hub.

```shell
docker build -t gcr.io/my-gcp-project/my-kube-scheduler:1.0 .     # Tên image và repository
gcloud docker -- push gcr.io/my-gcp-project/my-kube-scheduler:1.0 # dùng ở đây chỉ là ví dụ
```

## Định nghĩa một Kubernetes Deployment cho scheduler (Define a Kubernetes Deployment for the scheduler)

Giờ đây bạn đã có scheduler nằm trong một container image, hãy tạo một cấu hình Pod cho nó và
chạy nó trong cluster Kubernetes của bạn. Nhưng thay vì tạo trực tiếp một Pod trong cluster, ở ví
dụ này bạn có thể dùng một [Deployment](63-deployment-vi.md). Một
[Deployment](63-deployment-vi.md) quản lý một [Replica Set](64-replicaset-vi.md), và đến lượt
mình Replica Set quản lý các Pod, nhờ vậy scheduler có khả năng chống chịu sự cố. Dưới đây là
cấu hình deployment. Hãy lưu nó với tên `my-scheduler.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-scheduler
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: my-scheduler-as-kube-scheduler
subjects:
- kind: ServiceAccount
  name: my-scheduler
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: system:kube-scheduler
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: my-scheduler-as-volume-scheduler
subjects:
- kind: ServiceAccount
  name: my-scheduler
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: system:volume-scheduler
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-scheduler-extension-apiserver-authentication-reader
  namespace: kube-system
roleRef:
  kind: Role
  name: extension-apiserver-authentication-reader
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: my-scheduler
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-scheduler-config
  namespace: kube-system
data:
  my-scheduler-config.yaml: |
    apiVersion: kubescheduler.config.k8s.io/v1
    kind: KubeSchedulerConfiguration
    profiles:
      - schedulerName: my-scheduler
    leaderElection:
      leaderElect: false
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    component: scheduler
    tier: control-plane
  name: my-scheduler
  namespace: kube-system
spec:
  selector:
    matchLabels:
      component: scheduler
      tier: control-plane
  replicas: 1
  template:
    metadata:
      labels:
        component: scheduler
        tier: control-plane
        version: second
    spec:
      serviceAccountName: my-scheduler
      containers:
      - command:
        - /usr/local/bin/kube-scheduler
        - --config=/etc/kubernetes/my-scheduler/my-scheduler-config.yaml
        image: gcr.io/my-gcp-project/my-kube-scheduler:1.0
        livenessProbe:
          httpGet:
            path: /healthz
            port: 10259
            scheme: HTTPS
          initialDelaySeconds: 15
        name: kube-second-scheduler
        readinessProbe:
          httpGet:
            path: /healthz
            port: 10259
            scheme: HTTPS
        resources:
          requests:
            cpu: '0.1'
        securityContext:
          privileged: false
        volumeMounts:
          - name: config-volume
            mountPath: /etc/kubernetes/my-scheduler
      hostNetwork: false
      hostPID: false
      volumes:
        - name: config-volume
          configMap:
            name: my-scheduler-config
```

Trong manifest ở trên, bạn dùng một
[KubeSchedulerConfiguration](https://kubernetes.io/docs/reference/scheduling/config/)
để tùy chỉnh hành vi cho phần hiện thực scheduler của mình. Cấu hình này được truyền cho
`kube-scheduler` trong quá trình khởi tạo thông qua tùy chọn `--config`. ConfigMap
`my-scheduler-config` lưu file cấu hình đó. Pod của Deployment `my-scheduler` mount ConfigMap
`my-scheduler-config` dưới dạng một volume.

Trong Scheduler Configuration nói trên, phần hiện thực scheduler của bạn được biểu diễn qua một
[KubeSchedulerProfile](https://kubernetes.io/docs/reference/config-api/kube-scheduler-config.v1/#kubescheduler-config-k8s-io-v1-KubeSchedulerProfile).

> **Ghi chú:** Để xác định một scheduler có chịu trách nhiệm lập lịch cho một Pod cụ thể hay
> không, trường `spec.schedulerName` trong PodTemplate hoặc trong manifest của Pod phải khớp với
> trường `schedulerName` của `KubeSchedulerProfile`. Mọi scheduler đang chạy trong cluster đều
> phải có tên duy nhất.

Ngoài ra, hãy lưu ý rằng bạn tạo một service account riêng là `my-scheduler` và gán ClusterRole
`system:kube-scheduler` cho nó, để nó có được cùng những đặc quyền như `kube-scheduler`.

Vui lòng xem
[tài liệu kube-scheduler](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)
để có mô tả chi tiết về các tham số dòng lệnh khác, và
[tài liệu tham khảo Scheduler Configuration](https://kubernetes.io/docs/reference/config-api/kube-scheduler-config.v1/)
để có mô tả chi tiết về các cấu hình `kube-scheduler` khác mà bạn có thể tùy chỉnh.

## Chạy scheduler thứ hai trong cluster (Run the second scheduler in the cluster)

Để chạy scheduler của bạn trong một cluster Kubernetes, hãy tạo deployment đã được đặc tả trong
cấu hình ở trên vào cluster Kubernetes:

```shell
kubectl create -f my-scheduler.yaml
```

Xác minh rằng Pod của scheduler đang chạy:

```shell
kubectl get pods --namespace=kube-system
```

```
NAME                                           READY     STATUS    RESTARTS   AGE
....
my-scheduler-lnf4s-4744f                       1/1       Running   0          2m
...
```

Bạn sẽ thấy một Pod my-scheduler ở trạng thái "Running", bên cạnh Pod kube-scheduler mặc định
trong danh sách này.

### Bật leader election (Enable leader election)

Để chạy nhiều scheduler với leader election được bật, bạn phải làm những việc sau:

Cập nhật các trường sau đây cho KubeSchedulerConfiguration nằm trong ConfigMap
`my-scheduler-config` trong file YAML của bạn:

* `leaderElection.leaderElect` thành `true`
* `leaderElection.resourceNamespace` thành `<lock-object-namespace>`
* `leaderElection.resourceName` thành `<lock-object-name>`

> **Ghi chú:** Control plane sẽ tạo các lock object giúp bạn, nhưng namespace thì phải tồn tại
> sẵn. Bạn có thể dùng namespace `kube-system`.

Nếu RBAC được bật trên cluster của bạn, bạn phải cập nhật cluster role `system:kube-scheduler`.
Hãy thêm tên scheduler của bạn vào phần resourceNames của rule áp dụng cho các resource
`endpoints` và `leases`, như trong ví dụ sau:

```shell
kubectl edit clusterrole system:kube-scheduler
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-scheduler
rules:
  - apiGroups:
      - coordination.k8s.io
    resources:
      - leases
    verbs:
      - create
  - apiGroups:
      - coordination.k8s.io
    resourceNames:
      - kube-scheduler
      - my-scheduler
    resources:
      - leases
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resourceNames:
      - kube-scheduler
      - my-scheduler
    resources:
      - endpoints
    verbs:
      - delete
      - get
      - patch
      - update
```

## Chỉ định scheduler cho các Pod (Specify schedulers for pods)

Giờ đây scheduler thứ hai của bạn đã chạy, hãy tạo vài Pod và chỉ định chúng được lập lịch bởi
scheduler mặc định hoặc bởi scheduler mà bạn vừa triển khai. Để lập lịch một Pod nhất định bằng
một scheduler cụ thể, hãy chỉ định tên của scheduler đó trong pod spec. Hãy cùng xem ba ví dụ.

- Pod spec không có tên scheduler nào

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: no-annotation
    labels:
      name: multischeduler-example
  spec:
    containers:
    - name: pod-with-no-annotation-container
      image: registry.k8s.io/pause:3.8
  ```

  Khi không có tên scheduler nào được cung cấp, Pod sẽ tự động được lập lịch bằng
  default-scheduler.

  Lưu file này với tên `pod1.yaml` rồi gửi nó lên cluster Kubernetes.

  ```shell
  kubectl create -f pod1.yaml
  ```

- Pod spec với `default-scheduler`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: annotation-default-scheduler
    labels:
      name: multischeduler-example
  spec:
    schedulerName: default-scheduler
    containers:
    - name: pod-with-default-annotation-container
      image: registry.k8s.io/pause:3.8
  ```

  Một scheduler được chỉ định bằng cách cung cấp tên scheduler làm giá trị cho
  `spec.schedulerName`. Trong trường hợp này, chúng ta cung cấp tên của scheduler mặc định, đó là
  `default-scheduler`.

  Lưu file này với tên `pod2.yaml` rồi gửi nó lên cluster Kubernetes.

  ```shell
  kubectl create -f pod2.yaml
  ```

- Pod spec với `my-scheduler`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: annotation-second-scheduler
    labels:
      name: multischeduler-example
  spec:
    schedulerName: my-scheduler
    containers:
    - name: pod-with-second-annotation-container
      image: registry.k8s.io/pause:3.8
  ```

  Trong trường hợp này, chúng ta chỉ định rằng Pod này phải được lập lịch bằng scheduler mà
  chúng ta đã triển khai — `my-scheduler`. Hãy lưu ý rằng giá trị của `spec.schedulerName` phải
  khớp với tên đã đặt cho scheduler ở trường `schedulerName` của `KubeSchedulerProfile` tương ứng.

  Lưu file này với tên `pod3.yaml` rồi gửi nó lên cluster Kubernetes.

  ```shell
  kubectl create -f pod3.yaml
  ```

  Xác minh rằng cả ba Pod đều đang chạy.

  ```shell
  kubectl get pods
  ```

### Xác minh các Pod đã được lập lịch bằng đúng scheduler mong muốn (Verifying that the pods were scheduled using the desired schedulers)

Để việc thực hành các ví dụ trên dễ dàng hơn, chúng ta đã không xác minh rằng các Pod thực sự
được lập lịch bằng đúng scheduler mong muốn. Chúng ta có thể xác minh điều đó bằng cách đổi thứ
tự gửi các cấu hình Pod và deployment ở trên. Nếu chúng ta gửi tất cả các cấu hình Pod lên cluster
Kubernetes trước khi gửi cấu hình deployment của scheduler, chúng ta sẽ thấy Pod
`annotation-second-scheduler` mãi mãi ở trạng thái "Pending", trong khi hai Pod còn lại thì được
lập lịch. Ngay khi chúng ta gửi cấu hình deployment của scheduler và scheduler mới của chúng ta
bắt đầu chạy, Pod `annotation-second-scheduler` cũng sẽ được lập lịch.

Ngoài ra, bạn có thể xem các mục "Scheduled" trong event log để xác minh rằng các Pod đã được lập
lịch bởi đúng những scheduler mong muốn.

```shell
kubectl get events
```

Bạn cũng có thể dùng một
[cấu hình scheduler tùy chỉnh](https://kubernetes.io/docs/reference/scheduling/config/#multiple-profiles)
hoặc một container image tùy chỉnh cho scheduler chính của cluster, bằng cách sửa static pod
manifest của nó trên các control plane node liên quan.

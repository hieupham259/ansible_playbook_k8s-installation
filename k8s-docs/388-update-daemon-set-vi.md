# Thực hiện rolling update trên một DaemonSet (Perform a Rolling Update on a DaemonSet)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/manage-daemon/update-daemon-set/>
>
> Trang này hướng dẫn cách thực hiện một rolling update trên một DaemonSet.

Trang này hướng dẫn cách thực hiện một rolling update trên một DaemonSet.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Chiến lược cập nhật của DaemonSet (DaemonSet Update Strategy)

DaemonSet có hai kiểu chiến lược cập nhật (update strategy):

* `OnDelete`: Với chiến lược cập nhật `OnDelete`, sau khi bạn cập nhật template của một
  DaemonSet, các pod mới của DaemonSet *chỉ* được tạo ra khi bạn xóa thủ công các pod cũ của
  DaemonSet. Đây cũng chính là hành vi của DaemonSet trong Kubernetes phiên bản 1.5 trở về
  trước.
* `RollingUpdate`: Đây là chiến lược cập nhật mặc định.
  Với chiến lược cập nhật `RollingUpdate`, sau khi bạn cập nhật template của một DaemonSet,
  các pod cũ của DaemonSet sẽ bị hủy, và các pod mới của DaemonSet sẽ được tạo ra tự động,
  theo một cách có kiểm soát. Trong suốt quá trình cập nhật, trên mỗi node sẽ có nhiều nhất
  một pod của DaemonSet đang chạy.

## Thực hiện một rolling update (Performing a Rolling Update)

Để bật tính năng rolling update của một DaemonSet, bạn phải đặt `.spec.updateStrategy.type`
của nó thành `RollingUpdate`.

Bạn có thể cũng muốn đặt thêm
[`.spec.updateStrategy.rollingUpdate.maxUnavailable`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/daemon-set-v1/#DaemonSetSpec)
(mặc định là 1),
[`.spec.minReadySeconds`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/daemon-set-v1/#DaemonSetSpec)
(mặc định là 0) và
[`.spec.updateStrategy.rollingUpdate.maxSurge`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/daemon-set-v1/#DaemonSetSpec)
(mặc định là 0).

### Tạo một DaemonSet với chiến lược cập nhật `RollingUpdate` (Creating a DaemonSet with `RollingUpdate` update strategy)

File YAML sau đây khai báo một DaemonSet có chiến lược cập nhật là 'RollingUpdate'

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # các toleration này để daemonset có thể chạy được trên các node control plane
      # hãy xóa chúng nếu các node control plane của bạn không nên chạy pod
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v5.0.1
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

Sau khi đã kiểm tra chiến lược cập nhật trong manifest của DaemonSet, hãy tạo DaemonSet:

```shell
kubectl create -f https://k8s.io/examples/controllers/fluentd-daemonset.yaml
```

Ngoài ra, bạn có thể dùng `kubectl apply` để tạo cùng DaemonSet đó nếu bạn dự định sẽ cập nhật
DaemonSet bằng `kubectl apply`.

```shell
kubectl apply -f https://k8s.io/examples/controllers/fluentd-daemonset.yaml
```

### Kiểm tra chiến lược cập nhật `RollingUpdate` của DaemonSet (Checking DaemonSet `RollingUpdate` update strategy)

Hãy kiểm tra chiến lược cập nhật của DaemonSet, và đảm bảo rằng nó được đặt thành
`RollingUpdate`:

```shell
kubectl get ds/fluentd-elasticsearch -o go-template='{{.spec.updateStrategy.type}}{{"\n"}}' -n kube-system
```

Nếu bạn chưa tạo DaemonSet trong hệ thống, hãy dùng lệnh sau để kiểm tra manifest của
DaemonSet thay thế:

```shell
kubectl apply -f https://k8s.io/examples/controllers/fluentd-daemonset.yaml --dry-run=client -o go-template='{{.spec.updateStrategy.type}}{{"\n"}}'
```

Kết quả của cả hai lệnh đều phải là:

```
RollingUpdate
```

Nếu kết quả không phải là `RollingUpdate`, hãy quay lại và sửa đối tượng DaemonSet hoặc
manifest cho phù hợp.


### Cập nhật template của một DaemonSet (Updating a DaemonSet template)

Bất kỳ thay đổi nào đối với `.spec.template` của một DaemonSet dùng `RollingUpdate` đều sẽ
kích hoạt một rolling update. Hãy cập nhật DaemonSet bằng cách apply một file YAML mới. Việc
này có thể thực hiện bằng vài lệnh `kubectl` khác nhau.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # các toleration này để daemonset có thể chạy được trên các node control plane
      # hãy xóa chúng nếu các node control plane của bạn không nên chạy pod
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v5.0.1
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

#### Lệnh khai báo (Declarative commands)

Nếu bạn cập nhật DaemonSet bằng [file cấu hình](319-declarative-config-vi.md), hãy dùng
`kubectl apply`:

```shell
kubectl apply -f https://k8s.io/examples/controllers/fluentd-daemonset-update.yaml
```

#### Lệnh mệnh lệnh (Imperative commands)

Nếu bạn cập nhật DaemonSet bằng
[lệnh mệnh lệnh (imperative commands)](320-imperative-command-vi.md), hãy dùng `kubectl edit`:

```shell
kubectl edit ds/fluentd-elasticsearch -n kube-system
```

##### Chỉ cập nhật container image (Updating only the container image)

Nếu bạn chỉ cần cập nhật container image trong template của DaemonSet, tức là
`.spec.template.spec.containers[*].image`, hãy dùng `kubectl set image`:

```shell
kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=quay.io/fluentd_elasticsearch/fluentd:v2.6.0 -n kube-system
```

### Theo dõi trạng thái của rolling update (Watching the rolling update status)

Cuối cùng, hãy theo dõi trạng thái rollout của lần rolling update DaemonSet mới nhất:

```shell
kubectl rollout status ds/fluentd-elasticsearch -n kube-system
```

Khi rollout hoàn tất, kết quả sẽ tương tự như sau:

```shell
daemonset "fluentd-elasticsearch" successfully rolled out
```

## Khắc phục sự cố (Troubleshooting)

### Rolling update của DaemonSet bị kẹt (DaemonSet rolling update is stuck)

Đôi khi, một rolling update của DaemonSet có thể bị kẹt. Dưới đây là một số nguyên nhân có
thể xảy ra:

#### Một số node cạn kiệt tài nguyên (Some nodes run out of resources)

Rollout bị kẹt vì các pod mới của DaemonSet không thể được lập lịch (schedule) trên ít nhất
một node. Điều này có thể xảy ra khi node đang
[cạn kiệt tài nguyên](142-node-pressure-eviction-vi.md).

Khi điều này xảy ra, hãy tìm ra các node chưa có pod của DaemonSet được lập lịch lên bằng
cách so sánh kết quả của `kubectl get nodes` với kết quả của lệnh:

```shell
kubectl get pods -l name=fluentd-elasticsearch -o wide -n kube-system
```

Khi đã tìm ra các node đó, hãy xóa bớt một số pod không thuộc DaemonSet khỏi node để tạo chỗ
trống cho các pod mới của DaemonSet.

> **Ghi chú:** Việc này sẽ gây gián đoạn dịch vụ khi các pod bị xóa không được điều khiển bởi
> controller nào hoặc các pod đó không được nhân bản (replicate). Nó cũng không tôn trọng
> [PodDisruptionBudget](339-configure-pdb-vi.md).

#### Rollout hỏng (Broken rollout)

Nếu bản cập nhật template DaemonSet gần đây bị lỗi, ví dụ container rơi vào vòng lặp crash
(crash looping), hoặc container image không tồn tại (thường là do gõ sai), rollout của
DaemonSet sẽ không tiến triển.

Để khắc phục, hãy cập nhật lại template của DaemonSet một lần nữa. Rollout mới sẽ không bị
chặn bởi các rollout không lành mạnh trước đó.

#### Lệch đồng hồ (Clock skew)

Nếu `.spec.minReadySeconds` được chỉ định trong DaemonSet, việc lệch đồng hồ (clock skew)
giữa master và các node sẽ khiến DaemonSet không thể phát hiện đúng tiến độ rollout.

## Dọn dẹp (Clean up)

Xóa DaemonSet khỏi một namespace:

```shell
kubectl delete ds fluentd-elasticsearch -n kube-system
```

## Tiếp theo (What's next)

* Xem [Thực hiện rollback trên một DaemonSet (Performing a rollback on a DaemonSet)](387-rollback-daemon-set-vi.md)
* Xem [Tạo một DaemonSet để nhận (adopt) các pod DaemonSet đã có](66-daemonset-vi.md)

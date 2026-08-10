# Gỡ lỗi Pod đang chạy (Debug Running Pods)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/>

Trang này giải thích cách gỡ lỗi (debug) các Pod đang chạy (hoặc đang bị crash) trên một Node.

## Trước khi bạn bắt đầu (Before you begin)

* Pod của bạn cần đã được lập lịch (schedule) và đang chạy. Nếu Pod của bạn chưa chạy, hãy bắt
  đầu với [Gỡ lỗi Pod](https://kubernetes.io/docs/tasks/debug/debug-application/).
* Với một số bước gỡ lỗi nâng cao, bạn cần biết Pod đang chạy trên Node nào và có quyền truy
  cập shell để chạy lệnh trên Node đó. Bạn không cần quyền truy cập này để thực hiện các bước
  gỡ lỗi tiêu chuẩn sử dụng `kubectl`.

## Dùng `kubectl describe pod` để lấy thông tin chi tiết về pod (Using `kubectl describe pod` to fetch details about pods)

Trong ví dụ này, chúng ta sẽ dùng một Deployment để tạo hai pod, tương tự ví dụ trước đó.

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
        image: nginx
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 80
```

Tạo deployment bằng cách chạy lệnh sau:

```shell
kubectl apply -f https://k8s.io/examples/application/nginx-with-request.yaml
```

```none
deployment.apps/nginx-deployment created
```

Kiểm tra trạng thái pod bằng lệnh sau:

```shell
kubectl get pods
```

```none
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-67d4bdd6f5-cx2nz   1/1     Running   0          13s
nginx-deployment-67d4bdd6f5-w6kd7   1/1     Running   0          13s
```

Chúng ta có thể lấy nhiều thông tin hơn về mỗi pod này bằng `kubectl describe pod`. Ví dụ:

```shell
kubectl describe pod nginx-deployment-67d4bdd6f5-w6kd7
```

```none
Name:         nginx-deployment-67d4bdd6f5-w6kd7
Namespace:    default
Priority:     0
Node:         kube-worker-1/192.168.0.113
Start Time:   Thu, 17 Feb 2022 16:51:01 -0500
Labels:       app=nginx
              pod-template-hash=67d4bdd6f5
Annotations:  <none>
Status:       Running
IP:           10.88.0.3
IPs:
  IP:           10.88.0.3
  IP:           2001:db8::1
Controlled By:  ReplicaSet/nginx-deployment-67d4bdd6f5
Containers:
  nginx:
    Container ID:   containerd://5403af59a2b46ee5a23fb0ae4b1e077f7ca5c5fb7af16e1ab21c00e0e616462a
    Image:          nginx
    Image ID:       docker.io/library/nginx@sha256:2834dc507516af02784808c5f48b7cbe38b8ed5d0f4837f16e78d00deb7e7767
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Thu, 17 Feb 2022 16:51:05 -0500
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     500m
      memory:  128Mi
    Requests:
      cpu:        500m
      memory:     128Mi
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-bgsgp (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-bgsgp:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Guaranteed
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  34s   default-scheduler  Successfully assigned default/nginx-deployment-67d4bdd6f5-w6kd7 to kube-worker-1
  Normal  Pulling    31s   kubelet            Pulling image "nginx"
  Normal  Pulled     30s   kubelet            Successfully pulled image "nginx" in 1.146417389s
  Normal  Created    30s   kubelet            Created container nginx
  Normal  Started    30s   kubelet            Started container nginx
```

Ở đây bạn có thể thấy thông tin cấu hình về (các) container và Pod (label, yêu cầu tài nguyên,
v.v.), cũng như thông tin trạng thái về (các) container và Pod (trạng thái, mức độ sẵn sàng
(readiness), số lần khởi động lại, sự kiện, v.v.).

Trạng thái của container là một trong các giá trị Waiting, Running hoặc Terminated. Tùy theo
trạng thái, thông tin bổ sung sẽ được cung cấp — ở đây bạn có thể thấy rằng với một container
đang ở trạng thái Running, hệ thống cho bạn biết container đã khởi động lúc nào.

Ready cho bạn biết container có vượt qua lần readiness probe gần nhất hay không. (Trong trường
hợp này, container không được cấu hình readiness probe; container được coi là sẵn sàng nếu
không có readiness probe nào được cấu hình.)

Restart Count cho bạn biết container đã bị khởi động lại bao nhiêu lần; thông tin này hữu ích
để phát hiện vòng lặp crash (crash loop) ở các container được cấu hình restart policy là
`Always`.

Hiện tại, Condition duy nhất gắn với một Pod là condition Ready dạng nhị phân, cho biết pod có
khả năng phục vụ các request và nên được thêm vào các pool cân bằng tải của mọi service khớp
với nó.

Cuối cùng, bạn thấy một nhật ký các sự kiện gần đây liên quan đến Pod của bạn. "From" cho biết
thành phần nào ghi nhận sự kiện. "Reason" và "Message" cho bạn biết chuyện gì đã xảy ra.


## Ví dụ: gỡ lỗi Pod ở trạng thái Pending (Example: debugging Pending Pods)

Một tình huống phổ biến mà bạn có thể phát hiện qua các sự kiện là khi bạn tạo một Pod không
vừa với bất kỳ node nào. Ví dụ, Pod có thể yêu cầu nhiều tài nguyên hơn lượng còn trống trên
mọi node, hoặc nó có thể chỉ định một label selector không khớp với node nào. Giả sử chúng ta
tạo Deployment ở trên với 5 bản sao (thay vì 2) và yêu cầu 600 millicore thay vì 500, trên một
cluster bốn node trong đó mỗi máy (ảo) có 1 CPU. Khi đó một trong các Pod sẽ không thể được
lập lịch. (Lưu ý rằng do có các pod addon của cluster như fluentd, skydns, v.v. chạy trên mỗi
node, nếu chúng ta yêu cầu 1000 millicore thì sẽ không Pod nào được lập lịch cả.)

```shell
kubectl get pods
```

```none
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-1006230814-6winp   1/1       Running   0          7m
nginx-deployment-1006230814-fmgu3   1/1       Running   0          7m
nginx-deployment-1370807587-6ekbw   1/1       Running   0          1m
nginx-deployment-1370807587-fg172   0/1       Pending   0          1m
nginx-deployment-1370807587-fz9sd   0/1       Pending   0          1m
```

Để tìm hiểu vì sao pod nginx-deployment-1370807587-fz9sd không chạy, chúng ta có thể dùng
`kubectl describe pod` trên Pod đang pending và xem các sự kiện của nó:

```shell
kubectl describe pod nginx-deployment-1370807587-fz9sd
```

```none
  Name:		nginx-deployment-1370807587-fz9sd
  Namespace:	default
  Node:		/
  Labels:		app=nginx,pod-template-hash=1370807587
  Status:		Pending
  IP:
  Controllers:	ReplicaSet/nginx-deployment-1370807587
  Containers:
    nginx:
      Image:	nginx
      Port:	80/TCP
      QoS Tier:
        memory:	Guaranteed
        cpu:	Guaranteed
      Limits:
        cpu:	1
        memory:	128Mi
      Requests:
        cpu:	1
        memory:	128Mi
      Environment Variables:
  Volumes:
    default-token-4bcbi:
      Type:	Secret (a volume populated by a Secret)
      SecretName:	default-token-4bcbi
  Events:
    FirstSeen	LastSeen	Count	From			        SubobjectPath	Type		Reason			    Message
    ---------	--------	-----	----			        -------------	--------	------			    -------
    1m		    48s		    7	    {default-scheduler }			        Warning		FailedScheduling	pod (nginx-deployment-1370807587-fz9sd) failed to fit in any node
  fit failure on node (kubernetes-node-6ta5): Node didn't have enough resource: CPU, requested: 1000, used: 1420, capacity: 2000
  fit failure on node (kubernetes-node-wul5): Node didn't have enough resource: CPU, requested: 1000, used: 1100, capacity: 2000
```

Ở đây bạn có thể thấy sự kiện do scheduler sinh ra, cho biết Pod không lập lịch được với lý do
`FailedScheduling` (và có thể còn các lý do khác). Thông điệp cho chúng ta biết rằng không có
đủ tài nguyên cho Pod trên bất kỳ node nào.

Để khắc phục tình huống này, bạn có thể dùng `kubectl scale` để cập nhật Deployment của mình
xuống bốn bản sao hoặc ít hơn. (Hoặc bạn có thể cứ để một Pod ở trạng thái pending — điều này
vô hại.)

Các sự kiện như những gì bạn thấy ở cuối output của `kubectl describe pod` được lưu bền vững
trong etcd và cung cấp thông tin mức cao về những gì đang diễn ra trong cluster. Để liệt kê
toàn bộ sự kiện, bạn có thể dùng

```shell
kubectl get events
```

nhưng bạn phải nhớ rằng các sự kiện thuộc về namespace. Điều này có nghĩa là nếu bạn quan tâm
đến sự kiện của một đối tượng thuộc namespace nào đó (ví dụ chuyện gì đã xảy ra với các Pod
trong namespace `my-namespace`), bạn cần chỉ định namespace một cách tường minh trong lệnh:

```shell
kubectl get events --namespace=my-namespace
```

Để xem sự kiện từ tất cả các namespace, bạn có thể dùng tham số `--all-namespaces`.

Ngoài `kubectl describe pod`, một cách khác để lấy thêm thông tin về một pod (ngoài những gì
`kubectl get pod` cung cấp) là truyền flag định dạng output `-o yaml` cho `kubectl get pod`.
Cách này sẽ cho bạn, ở định dạng YAML, thậm chí nhiều thông tin hơn cả `kubectl describe pod`
— về cơ bản là toàn bộ thông tin mà hệ thống có về Pod đó. Ở đây bạn sẽ thấy những thứ như
annotation (là metadata dạng key-value không chịu các ràng buộc của label, được các thành phần
hệ thống Kubernetes sử dụng nội bộ), restart policy, các port, và các volume.

```shell
kubectl get pod nginx-deployment-1006230814-6winp -o yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2022-02-17T21:51:01Z"
  generateName: nginx-deployment-67d4bdd6f5-
  labels:
    app: nginx
    pod-template-hash: 67d4bdd6f5
  name: nginx-deployment-67d4bdd6f5-w6kd7
  namespace: default
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: nginx-deployment-67d4bdd6f5
    uid: 7d41dfd4-84c0-4be4-88ab-cedbe626ad82
  resourceVersion: "1364"
  uid: a6501da1-0447-4262-98eb-c03d4002222e
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: nginx
    ports:
    - containerPort: 80
      protocol: TCP
    resources:
      limits:
        cpu: 500m
        memory: 128Mi
      requests:
        cpu: 500m
        memory: 128Mi
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-bgsgp
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: kube-worker-1
  preemptionPolicy: PreemptLowerPriority
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: kube-api-access-bgsgp
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2022-02-17T21:51:01Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2022-02-17T21:51:06Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2022-02-17T21:51:06Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2022-02-17T21:51:01Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: containerd://5403af59a2b46ee5a23fb0ae4b1e077f7ca5c5fb7af16e1ab21c00e0e616462a
    image: docker.io/library/nginx:latest
    imageID: docker.io/library/nginx@sha256:2834dc507516af02784808c5f48b7cbe38b8ed5d0f4837f16e78d00deb7e7767
    lastState: {}
    name: nginx
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2022-02-17T21:51:05Z"
  hostIP: 192.168.0.113
  phase: Running
  podIP: 10.88.0.3
  podIPs:
  - ip: 10.88.0.3
  - ip: 2001:db8::1
  qosClass: Guaranteed
  startTime: "2022-02-17T21:51:01Z"
```

## Xem log của pod (Examining pod logs) {#examine-pod-logs}

Trước tiên, hãy xem log của container gặp vấn đề:

```shell
kubectl logs ${POD_NAME} -c ${CONTAINER_NAME}
```

Nếu container của bạn đã từng bị crash trước đó, bạn có thể truy cập log crash của lần chạy
container trước bằng:

```shell
kubectl logs ${POD_NAME} -c ${CONTAINER_NAME} --previous
```

## Gỡ lỗi bằng container exec (Debugging with container exec) {#container-exec}

Nếu image của container có sẵn các tiện ích gỡ lỗi, như trường hợp các image được build từ
image nền của hệ điều hành Linux và Windows, bạn có thể chạy lệnh bên trong một container cụ
thể bằng `kubectl exec`:

```shell
kubectl exec ${POD_NAME} -c ${CONTAINER_NAME} -- ${CMD} ${ARG1} ${ARG2} ... ${ARGN}
```

> **Ghi chú:**
> `-c ${CONTAINER_NAME}` là tùy chọn. Bạn có thể bỏ qua nó với các Pod chỉ chứa một container
> duy nhất.

Ví dụ, để xem log từ một pod Cassandra đang chạy, bạn có thể chạy

```shell
kubectl exec cassandra -- cat /var/log/cassandra/system.log
```

Bạn có thể chạy một shell được kết nối với terminal của mình bằng các tham số `-i` và `-t`
của `kubectl exec`, ví dụ:

```shell
kubectl exec -it cassandra -- sh
```

Để biết thêm chi tiết, xem
[Mở Shell vào một Container đang chạy](https://kubernetes.io/docs/tasks/debug/debug-application/get-shell-running-container/).

## Gỡ lỗi bằng ephemeral debug container (Debugging with an ephemeral debug container) {#ephemeral-container}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.25 [stable]`

Ephemeral container (container tạm thời) hữu ích cho việc xử lý sự cố tương tác khi
`kubectl exec` không đủ dùng, vì container đã bị crash hoặc image của container không có sẵn
các tiện ích gỡ lỗi, chẳng hạn với các
[distroless image](https://github.com/GoogleContainerTools/distroless).

### Ví dụ gỡ lỗi bằng ephemeral container (Example debugging using ephemeral containers) {#ephemeral-container-example}

Bạn có thể dùng lệnh `kubectl debug` để thêm ephemeral container vào một Pod đang chạy.
Trước tiên, tạo một pod cho ví dụ:

```shell
kubectl run ephemeral-demo --image=registry.k8s.io/pause:3.1 --restart=Never
```

Các ví dụ trong mục này dùng image container `pause` vì nó không chứa các tiện ích gỡ lỗi,
nhưng phương pháp này hoạt động với mọi image container.

Nếu bạn thử dùng `kubectl exec` để tạo một shell, bạn sẽ gặp lỗi vì không có shell nào trong
image container này.

```shell
kubectl exec -it ephemeral-demo -- sh
```

```
OCI runtime exec failed: exec failed: container_linux.go:346: starting container process caused "exec: \"sh\": executable file not found in $PATH": unknown
```

Thay vào đó, bạn có thể thêm một container gỡ lỗi bằng `kubectl debug`. Nếu bạn chỉ định tham
số `-i`/`--interactive`, `kubectl` sẽ tự động attach vào console của Ephemeral Container.

```shell
kubectl debug -it ephemeral-demo --image=busybox:1.28 --target=ephemeral-demo
```

```
Defaulting debug container name to debugger-8xzrl.
If you don't see a command prompt, try pressing enter.
/ #
```

Lệnh này thêm một container busybox mới và attach vào nó. Tham số `--target` nhắm tới process
namespace của một container khác. Nó cần thiết ở đây vì `kubectl run` không bật
[chia sẻ process namespace](https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/)
trong pod mà nó tạo ra.

> **Ghi chú:**
> Tham số `--target` phải được container runtime hỗ trợ. Khi không được hỗ trợ, Ephemeral
> Container có thể không khởi động được, hoặc nó có thể khởi động với một process namespace
> cô lập khiến `ps` không hiển thị các tiến trình trong các container khác.

Bạn có thể xem trạng thái của ephemeral container vừa tạo bằng `kubectl describe`:

```shell
kubectl describe pod ephemeral-demo
```

```
...
Ephemeral Containers:
  debugger-8xzrl:
    Container ID:   docker://b888f9adfd15bd5739fefaa39e1df4dd3c617b9902082b1cfdc29c4028ffb2eb
    Image:          busybox
    Image ID:       docker-pullable://busybox@sha256:1828edd60c5efd34b2bf5dd3282ec0cc04d47b2ff9caa0b6d4f07a21d1c08084
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Wed, 12 Feb 2020 14:25:42 +0100
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:         <none>
...
```

Dùng `kubectl delete` để xóa Pod khi bạn hoàn tất:

```shell
kubectl delete pod ephemeral-demo
```

## Gỡ lỗi bằng một bản sao của Pod (Debugging using a copy of the Pod)

Đôi khi các tùy chọn cấu hình của Pod khiến việc xử lý sự cố trong một số tình huống trở nên
khó khăn. Ví dụ, bạn không thể chạy `kubectl exec` để xử lý sự cố container nếu image container
không có shell hoặc nếu ứng dụng của bạn bị crash ngay khi khởi động. Trong những tình huống
này, bạn có thể dùng `kubectl debug` để tạo một bản sao của Pod với các giá trị cấu hình được
thay đổi nhằm hỗ trợ việc gỡ lỗi.

### Sao chép Pod đồng thời thêm một container mới (Copying a Pod while adding a new container)

Thêm một container mới sẽ hữu ích khi ứng dụng của bạn đang chạy nhưng không hoạt động như bạn
mong đợi và bạn muốn thêm các tiện ích xử lý sự cố vào Pod.

Ví dụ, có thể các image container của ứng dụng bạn được build trên nền `busybox` nhưng bạn cần
các tiện ích gỡ lỗi không có trong `busybox`. Bạn có thể mô phỏng tình huống này bằng
`kubectl run`:

```shell
kubectl run myapp --image=busybox:1.28 --restart=Never -- sleep 1d
```

Chạy lệnh sau để tạo một bản sao của `myapp` tên là `myapp-debug`, có thêm một container
Ubuntu mới để gỡ lỗi:

```shell
kubectl debug myapp -it --image=ubuntu --share-processes --copy-to=myapp-debug
```

```
Defaulting debug container name to debugger-w7xmf.
If you don't see a command prompt, try pressing enter.
root@myapp-debug:/#
```

> **Ghi chú:**
> * `kubectl debug` tự động sinh tên container nếu bạn không tự chọn tên bằng flag
>   `--container`.
> * Flag `-i` khiến `kubectl debug` mặc định attach vào container mới. Bạn có thể ngăn điều
>   này bằng cách chỉ định `--attach=false`. Nếu phiên làm việc của bạn bị ngắt kết nối, bạn
>   có thể attach lại bằng `kubectl attach`.
> * `--share-processes` cho phép các container trong Pod này nhìn thấy tiến trình của các
>   container khác trong Pod. Để biết thêm về cách cơ chế này hoạt động, xem
>   [Chia sẻ Process Namespace giữa các Container trong một Pod](https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/).

Đừng quên dọn dẹp Pod gỡ lỗi khi bạn dùng xong:

```shell
kubectl delete pod myapp myapp-debug
```

### Sao chép Pod đồng thời thay đổi lệnh của nó (Copying a Pod while changing its command)

Đôi khi việc thay đổi lệnh (command) của một container là hữu ích, ví dụ để thêm một flag gỡ
lỗi hoặc vì ứng dụng đang bị crash.

Để mô phỏng một ứng dụng bị crash, dùng `kubectl run` để tạo một container thoát ra ngay lập
tức:

```
kubectl run --image=busybox:1.28 myapp -- false
```

Bạn có thể thấy container này đang bị crash bằng `kubectl describe pod myapp`:

```
Containers:
  myapp:
    Image:         busybox
    ...
    Args:
      false
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
```

Bạn có thể dùng `kubectl debug` để tạo một bản sao của Pod này với lệnh được đổi thành một
shell tương tác:

```
kubectl debug myapp -it --copy-to=myapp-debug --container=myapp -- sh
```

```
If you don't see a command prompt, try pressing enter.
/ #
```

Bây giờ bạn có một shell tương tác mà bạn có thể dùng để thực hiện các việc như kiểm tra đường
dẫn trong hệ thống file hoặc chạy thủ công lệnh của container.

> **Ghi chú:**
> * Để thay đổi lệnh của một container cụ thể, bạn phải chỉ định tên của nó bằng
>   `--container`, nếu không `kubectl debug` sẽ tạo một container mới để chạy lệnh mà bạn đã
>   chỉ định.
> * Flag `-i` khiến `kubectl debug` mặc định attach vào container. Bạn có thể ngăn điều này
>   bằng cách chỉ định `--attach=false`. Nếu phiên làm việc của bạn bị ngắt kết nối, bạn có
>   thể attach lại bằng `kubectl attach`.

Đừng quên dọn dẹp Pod gỡ lỗi khi bạn dùng xong:

```shell
kubectl delete pod myapp myapp-debug
```

### Sao chép Pod đồng thời thay đổi image của container (Copying a Pod while changing container images)

Trong một số tình huống, bạn có thể muốn chuyển một Pod đang hoạt động bất thường từ các image
production bình thường sang một image chứa bản build phục vụ gỡ lỗi hoặc chứa các tiện ích bổ
sung.

Ví dụ, tạo một Pod bằng `kubectl run`:

```
kubectl run myapp --image=busybox:1.28 --restart=Never -- sleep 1d
```

Bây giờ dùng `kubectl debug` để tạo một bản sao và đổi image container của nó thành `ubuntu`:

```
kubectl debug myapp --copy-to=myapp-debug --set-image=*=ubuntu
```

Cú pháp của `--set-image` dùng cùng cú pháp `container_name=image` như `kubectl set image`.
`*=ubuntu` nghĩa là đổi image của tất cả các container thành `ubuntu`.

Đừng quên dọn dẹp Pod gỡ lỗi khi bạn dùng xong:

```shell
kubectl delete pod myapp myapp-debug
```

## Gỡ lỗi qua một shell trên node (Debugging via a shell on the node) {#node-shell-session}

Nếu không có cách nào ở trên hiệu quả, bạn có thể tìm Node mà Pod đang chạy trên đó và tạo một
Pod chạy trên Node này. Để tạo một shell tương tác trên một Node bằng `kubectl debug`, chạy:

```shell
kubectl debug node/mynode -it --image=ubuntu
```

```
Creating debugging pod node-debugger-mynode-pdx84 with container debugger on node mynode.
If you don't see a command prompt, try pressing enter.
root@ek8s:/#
```

Khi tạo phiên gỡ lỗi trên một node, hãy lưu ý rằng:

* `kubectl debug` tự động sinh tên cho Pod mới dựa trên tên của Node.
* Hệ thống file gốc (root filesystem) của Node sẽ được mount tại `/host`.
* Container chạy trong các namespace IPC, Network và PID của host, mặc dù pod này không có
  đặc quyền (privileged), vì vậy việc đọc một số thông tin tiến trình có thể thất bại, và
  `chroot /host` có thể thất bại.
* Nếu bạn cần một pod có đặc quyền, hãy tạo thủ công hoặc dùng flag `--profile=sysadmin`.

Đừng quên dọn dẹp Pod gỡ lỗi khi bạn dùng xong:

```shell
kubectl delete pod node-debugger-mynode-pdx84
```

### Bắt và phân tích lưu lượng mạng của Node/Pod (Capturing and analyzing Node/Pod traffic)

Khi gỡ lỗi các sự cố mạng, việc bắt (capture) và phân tích lưu lượng mạng từ Node/Pod có thể
mang lại những hiểu biết quý giá về các vấn đề kết nối, lỗi phân giải DNS, hoặc hành vi mạng
bất thường.

Bạn có thể dùng `kubectl debug` với flag `--profile=sysadmin` để chạy các công cụ bắt lưu
lượng mạng trên một node. Trước tiên, tạo một phiên gỡ lỗi trên node nơi Pod của bạn đang
chạy:

```shell
kubectl debug --profile=sysadmin node/${NODE_NAME} -it --image=ubuntu:latest
```

Khi đã vào bên trong debug container, cài đặt tcpdump và bắt lưu lượng trên các network
interface của node:

```shell
apt-get update && apt-get install -y tcpdump
tcpdump -i any -n
```

> **Ghi chú:**
> Đừng quên dọn dẹp Pod gỡ lỗi khi bạn dùng xong:
>
> ```shell
> kubectl delete pod node-debugger-mynode-pdx84
> ```

Bạn cũng có thể bắt lưu lượng từ một Pod cụ thể:

```shell
kubectl debug --profile=sysadmin pod/${POD_NAME} -n ${NAMESPACE} -it --image=ubuntu:latest
```

Rồi thực hiện cùng lệnh `tcpdump` bên trong debug container để bắt lưu lượng từ network
namespace của Pod.

## Gỡ lỗi Pod hoặc Node có áp dụng profile (Debugging a Pod or Node while applying a profile) {#debugging-profiles}

Khi dùng `kubectl debug` để gỡ lỗi một node thông qua một Pod gỡ lỗi, một Pod thông qua một
ephemeral container, hoặc một Pod được sao chép, bạn có thể áp dụng một profile cho chúng.
Bằng cách áp dụng profile, các thuộc tính cụ thể như
[securityContext](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
sẽ được thiết lập, cho phép thích ứng với nhiều tình huống khác nhau.
Có hai loại profile: profile tĩnh (static profile) và profile tùy chỉnh (custom profile).

### Áp dụng Static Profile (Applying a Static Profile) {#static-profile}

Static profile là một tập các thuộc tính được định nghĩa sẵn, và bạn có thể áp dụng chúng bằng
flag `--profile`. Các profile khả dụng như sau:

| Profile      | Mô tả                                                           |
| ------------ | --------------------------------------------------------------- |
| legacy       | Tập thuộc tính tương thích ngược với hành vi của phiên bản 1.22 |
| general      | Tập thuộc tính chung hợp lý cho mỗi phiên gỡ lỗi |
| baseline     | Tập thuộc tính tương thích với [chính sách baseline của PodSecurityStandard](https://kubernetes.io/docs/concepts/security/pod-security-standards/#baseline) |
| restricted   | Tập thuộc tính tương thích với [chính sách restricted của PodSecurityStandard](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted) |
| netadmin     | Tập thuộc tính bao gồm các đặc quyền của quản trị viên mạng (Network Administrator) |
| sysadmin     | Tập thuộc tính bao gồm các đặc quyền của quản trị viên hệ thống (System Administrator, tức root) |


> **Ghi chú:**
> Nếu bạn không chỉ định `--profile`, profile `legacy` sẽ được dùng mặc định, nhưng profile
> này dự kiến sẽ bị loại bỏ (deprecated) trong tương lai gần. Vì vậy, khuyến nghị dùng các
> profile khác như `general`.


Giả sử bạn tạo một Pod và gỡ lỗi nó.
Trước tiên, tạo một Pod tên `myapp` làm ví dụ:

```shell
kubectl run myapp --image=busybox:1.28 --restart=Never -- sleep 1d
```

Sau đó, gỡ lỗi Pod bằng một ephemeral container.
Nếu ephemeral container cần có đặc quyền, bạn có thể dùng profile `sysadmin`:

```shell
kubectl debug -it myapp --image=busybox:1.28 --target=myapp --profile=sysadmin
```

```
Targeting container "myapp". If you don't see processes from this container it may be because the container runtime doesn't support this feature.
Defaulting debug container name to debugger-6kg4x.
If you don't see a command prompt, try pressing enter.
/ #
```

Kiểm tra các capability của tiến trình trong ephemeral container bằng cách chạy lệnh sau bên
trong container:

```shell
/ # grep Cap /proc/$$/status
```

```
...
CapPrm:	000001ffffffffff
CapEff:	000001ffffffffff
...
```

Điều này có nghĩa là tiến trình của container được cấp đầy đủ capability như một privileged
container nhờ áp dụng profile `sysadmin`. Xem thêm chi tiết về
[capabilities](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-capabilities-for-a-container).

Bạn cũng có thể kiểm tra rằng ephemeral container đã được tạo như một privileged container:

```shell
kubectl get pod myapp -o jsonpath='{.spec.ephemeralContainers[0].securityContext}'
```

```
{"privileged":true}
```

Dọn dẹp Pod khi bạn dùng xong:

```shell
kubectl delete pod myapp
```

### Áp dụng Custom Profile (Applying Custom Profile) {#custom-profile}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.32 [stable]`

Bạn có thể định nghĩa một phần spec của container (partial container spec) phục vụ gỡ lỗi dưới
dạng custom profile ở định dạng YAML hoặc JSON, và áp dụng nó bằng flag `--custom`.

> **Ghi chú:**
> Custom profile chỉ hỗ trợ sửa đổi spec của container, nhưng không cho phép sửa đổi các
> trường `name`, `image`, `command`, `lifecycle` và `volumeDevices` của spec container.
> Nó không hỗ trợ sửa đổi spec của Pod.

Tạo một Pod tên myapp làm ví dụ:

```shell
kubectl run myapp --image=busybox:1.28 --restart=Never -- sleep 1d
```

Tạo một custom profile ở định dạng YAML hoặc JSON.
Ở đây, tạo một file định dạng YAML tên là `custom-profile.yaml`:

```yaml
env:
- name: ENV_VAR_1
  value: value_1
- name: ENV_VAR_2
  value: value_2
securityContext:
  capabilities:
    add:
    - NET_ADMIN
    - SYS_TIME

```

Chạy lệnh sau để gỡ lỗi Pod bằng một ephemeral container với custom profile trên:

```shell
kubectl debug -it myapp --image=busybox:1.28 --target=myapp --profile=general --custom=custom-profile.yaml
```

Bạn có thể kiểm tra rằng ephemeral container đã được thêm vào Pod đích với custom profile được
áp dụng:

```shell
kubectl get pod myapp -o jsonpath='{.spec.ephemeralContainers[0].env}'
```

```
[{"name":"ENV_VAR_1","value":"value_1"},{"name":"ENV_VAR_2","value":"value_2"}]
```

```shell
kubectl get pod myapp -o jsonpath='{.spec.ephemeralContainers[0].securityContext}'
```

```
{"capabilities":{"add":["NET_ADMIN","SYS_TIME"]}}
```

Dọn dẹp Pod khi bạn dùng xong:

```shell
kubectl delete pod myapp
```

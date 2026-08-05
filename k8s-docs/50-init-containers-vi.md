# Container khởi tạo (Init Containers)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/pods/init-containers/>

Trang này cung cấp cái nhìn tổng quan về container khởi tạo (init container): các
container chuyên biệt chạy trước các container ứng dụng (app container) trong một
Pod. Init container có thể chứa các tiện ích hoặc script cài đặt không có trong
image của ứng dụng.

Bạn có thể chỉ định init container trong đặc tả Pod (Pod specification) bên cạnh
mảng `containers` (mảng mô tả các app container).

Trong Kubernetes, một [sidecar container](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)
là container khởi động trước container ứng dụng chính và _tiếp tục chạy_. Tài liệu
này nói về init container: các container chạy đến khi hoàn thành (run to completion)
trong quá trình khởi tạo Pod.

## Hiểu về init container (Understanding init containers)

Một Pod có thể có nhiều container chạy các ứng dụng bên trong nó, nhưng nó cũng có
thể có một hoặc nhiều init container, được chạy trước khi các app container khởi
động.

Init container giống hệt các container thông thường, ngoại trừ:

* Init container luôn chạy đến khi hoàn thành.
* Mỗi init container phải hoàn thành thành công trước khi container kế tiếp bắt đầu.

Nếu init container của một Pod thất bại, kubelet sẽ liên tục khởi động lại init
container đó cho đến khi nó thành công. Tuy nhiên, nếu Pod có `restartPolicy` là
Never, và một init container thất bại trong quá trình khởi động của Pod đó,
Kubernetes coi toàn bộ Pod là thất bại.

Để chỉ định init container cho một Pod, thêm trường `initContainers` vào
[đặc tả Pod](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#PodSpec),
dưới dạng một mảng các mục `container` (tương tự trường `containers` của ứng dụng
và nội dung của nó).
Xem [Container](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Container)
trong tài liệu tham khảo API để biết thêm chi tiết.

Trạng thái của các init container được trả về trong trường
`.status.initContainerStatuses` dưới dạng một mảng trạng thái container (tương tự
trường `.status.containerStatuses`).

### Khác biệt so với container thông thường (Differences from regular containers)

Init container hỗ trợ tất cả các trường và tính năng của app container, bao gồm
giới hạn tài nguyên (resource limits), [volume](https://kubernetes.io/docs/concepts/storage/volumes/),
và các thiết lập bảo mật. Tuy nhiên, resource request và limit của init container
được xử lý khác đi, như được mô tả trong
[Chia sẻ tài nguyên giữa các container](#resource-sharing-within-containers).

Các init container thông thường (nói cách khác: không tính sidecar container) không
hỗ trợ các trường `lifecycle`, `livenessProbe`, `readinessProbe`, hay
`startupProbe`. Init container phải chạy đến khi hoàn thành trước khi Pod có thể
sẵn sàng; sidecar container tiếp tục chạy trong suốt vòng đời của Pod, và _có_ hỗ
trợ một số probe. Xem [sidecar container](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)
để biết thêm chi tiết về sidecar container.

Nếu bạn chỉ định nhiều init container cho một Pod, kubelet chạy từng init container
tuần tự. Mỗi init container phải thành công trước khi container kế tiếp có thể
chạy. Khi tất cả init container đã chạy đến khi hoàn thành, kubelet khởi tạo các
container ứng dụng cho Pod và chạy chúng như bình thường.

### Khác biệt so với sidecar container (Differences from sidecar containers)

Init container chạy và hoàn thành các tác vụ của chúng trước khi container ứng dụng
chính khởi động. Khác với [sidecar container](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers),
init container không chạy liên tục song song với các container chính.

Init container chạy đến khi hoàn thành một cách tuần tự, và container chính không
khởi động cho đến khi tất cả init container đã hoàn thành thành công.

Init container không hỗ trợ `lifecycle`, `livenessProbe`, `readinessProbe`, hay
`startupProbe`, trong khi sidecar container hỗ trợ tất cả các
[probe](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#types-of-probe)
này để kiểm soát vòng đời của chúng.

Init container chia sẻ cùng tài nguyên (CPU, bộ nhớ, mạng) với các container ứng
dụng chính nhưng không tương tác trực tiếp với chúng. Tuy nhiên, chúng có thể dùng
các volume dùng chung để trao đổi dữ liệu.

## Sử dụng init container (Using init containers)

Vì init container có image riêng biệt so với app container, chúng mang lại một số
lợi thế cho mã liên quan đến khởi động:

* Init container có thể chứa các tiện ích hoặc mã tùy chỉnh phục vụ việc cài đặt
  mà không có trong image của ứng dụng. Ví dụ, không cần phải tạo một image `FROM`
  một image khác chỉ để dùng một công cụ như `sed`, `awk`, `python`, hay `dig`
  trong quá trình cài đặt.
* Vai trò người xây dựng image ứng dụng và người triển khai (deployer) có thể làm
  việc độc lập mà không cần cùng nhau xây dựng một image ứng dụng duy nhất.
* Init container có thể chạy với góc nhìn hệ thống file khác với app container
  trong cùng Pod. Do đó, chúng có thể được cấp quyền truy cập vào các Secret mà
  app container không thể truy cập.
* Vì init container chạy đến khi hoàn thành trước khi bất kỳ app container nào
  khởi động, init container cung cấp một cơ chế để chặn hoặc trì hoãn việc khởi
  động app container cho đến khi một tập các điều kiện tiên quyết được thỏa mãn.
  Một khi các điều kiện tiên quyết được thỏa mãn, tất cả app container trong Pod
  có thể khởi động song song.
* Init container có thể chạy một cách an toàn các tiện ích hoặc mã tùy chỉnh mà
  nếu đưa vào image của app container sẽ làm image đó kém an toàn hơn. Bằng cách
  tách riêng các công cụ không cần thiết, bạn có thể hạn chế bề mặt tấn công
  (attack surface) của image app container.

### Các ví dụ (Examples)

Dưới đây là một số ý tưởng về cách sử dụng init container:

* Chờ một Service được tạo, dùng một lệnh shell một dòng như:
  ```shell
  for i in {1..100}; do sleep 1; if nslookup myservice; then exit 0; fi; done; exit 1
  ```

* Đăng ký Pod này với một server ở xa từ downward API bằng một lệnh như:
  ```shell
  curl -X POST http://$MANAGEMENT_SERVICE_HOST:$MANAGEMENT_SERVICE_PORT/register -d 'instance=$(<POD_NAME>)&ip=$(<POD_IP>)'
  ```

* Chờ một khoảng thời gian trước khi khởi động app container bằng một lệnh như
  ```shell
  sleep 60
  ```

* Clone một Git repository vào một Volume

* Đưa các giá trị vào một file cấu hình và chạy một công cụ template để sinh file
  cấu hình cho container ứng dụng chính một cách động. Ví dụ, đặt giá trị `POD_IP`
  vào một cấu hình và sinh file cấu hình cho ứng dụng chính bằng Jinja.

#### Init container trong thực tế (Init containers in use)

Ví dụ này định nghĩa một Pod đơn giản có hai init container.
Container thứ nhất chờ `myservice`, và container thứ hai chờ `mydb`. Khi cả hai
init container hoàn thành, Pod sẽ chạy app container từ phần `spec` của nó.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app.kubernetes.io/name: MyApp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done"]
```

Bạn có thể khởi động Pod này bằng cách chạy:

```shell
kubectl apply -f myapp.yaml
```
Kết quả xuất ra tương tự như sau:
```
pod/myapp-pod created
```

Và kiểm tra trạng thái của nó bằng:
```shell
kubectl get -f myapp.yaml
```
Kết quả xuất ra tương tự như sau:
```
NAME        READY     STATUS     RESTARTS   AGE
myapp-pod   0/1       Init:0/2   0          6m
```

hoặc để xem chi tiết hơn:
```shell
kubectl describe -f myapp.yaml
```
Kết quả xuất ra tương tự như sau:
```
Name:          myapp-pod
Namespace:     default
[...]
Labels:        app.kubernetes.io/name=MyApp
Status:        Pending
[...]
Init Containers:
  init-myservice:
[...]
    State:         Running
[...]
  init-mydb:
[...]
    State:         Waiting
      Reason:      PodInitializing
    Ready:         False
[...]
Containers:
  myapp-container:
[...]
    State:         Waiting
      Reason:      PodInitializing
    Ready:         False
[...]
Events:
  FirstSeen    LastSeen    Count    From                      SubObjectPath                           Type          Reason        Message
  ---------    --------    -----    ----                      -------------                           --------      ------        -------
  16s          16s         1        {default-scheduler }                                              Normal        Scheduled     Successfully assigned myapp-pod to 172.17.4.201
  16s          16s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Pulling       pulling image "busybox"
  13s          13s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Pulled        Successfully pulled image "busybox"
  13s          13s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Created       Created container init-myservice
  13s          13s         1        {kubelet 172.17.4.201}    spec.initContainers{init-myservice}     Normal        Started       Started container init-myservice
```

Để xem log của các init container trong Pod này, chạy:
```shell
kubectl logs myapp-pod -c init-myservice # Xem init container thứ nhất
kubectl logs myapp-pod -c init-mydb      # Xem init container thứ hai
```

Tại thời điểm này, các init container đó sẽ đang chờ để phát hiện các Service có
tên `mydb` và `myservice`.

Đây là cấu hình bạn có thể dùng để làm cho các Service đó xuất hiện:

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: myservice
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
---
apiVersion: v1
kind: Service
metadata:
  name: mydb
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9377
```

Để tạo các service `mydb` và `myservice`:

```shell
kubectl apply -f services.yaml
```
Kết quả xuất ra tương tự như sau:
```
service/myservice created
service/mydb created
```

Sau đó bạn sẽ thấy các init container đó hoàn thành, và Pod `myapp-pod` chuyển sang
trạng thái Running:

```shell
kubectl get -f myapp.yaml
```
Kết quả xuất ra tương tự như sau:
```
NAME        READY     STATUS    RESTARTS   AGE
myapp-pod   1/1       Running   0          9m
```

Ví dụ đơn giản này hẳn sẽ mang lại chút cảm hứng để bạn tự tạo các init container
của riêng mình. [Tiếp theo](#what-s-next) chứa liên kết đến một ví dụ chi tiết hơn.

## Hành vi chi tiết (Detailed behavior)

Trong quá trình khởi động Pod, kubelet trì hoãn việc chạy init container cho đến
khi mạng và lưu trữ sẵn sàng. Sau đó kubelet chạy các init container của Pod theo
thứ tự chúng xuất hiện trong spec của Pod.

Mỗi init container phải thoát thành công trước khi container kế tiếp khởi động.
Nếu một container không khởi động được do runtime hoặc thoát với trạng thái thất
bại, nó sẽ được thử lại theo `restartPolicy` của Pod. Tuy nhiên, nếu
`restartPolicy` của Pod được đặt là Always, các init container sử dụng
`restartPolicy` OnFailure.

Một Pod không thể ở trạng thái `Ready` cho đến khi tất cả init container đã thành
công. Các port trên init container không được gom (aggregate) dưới một Service.
Một Pod đang trong quá trình khởi tạo ở trạng thái `Pending` nhưng sẽ có một
condition `Initialized` được đặt thành false.

Nếu Pod [khởi động lại](#pod-restart-reasons), hoặc bị khởi động lại, tất cả init
container phải được thực thi lại.

Các thay đổi đối với spec của init container bị giới hạn ở trường container image.
Việc thay đổi trực tiếp trường `image` của một init container _không_ khởi động
lại Pod hay kích hoạt việc tạo lại nó. Nếu Pod còn chưa khởi động, thay đổi đó có
thể ảnh hưởng đến cách Pod khởi động.

Với một [pod template](https://kubernetes.io/docs/concepts/workloads/pods/#pod-templates),
thông thường bạn có thể thay đổi bất kỳ trường nào của init container; tác động của
thay đổi đó phụ thuộc vào nơi pod template được sử dụng.

Vì init container có thể bị khởi động lại, thử lại, hoặc thực thi lại, mã của init
container nên có tính bất biến kết quả (idempotent). Cụ thể, mã ghi vào bất kỳ
volume `emptyDir` nào nên được chuẩn bị cho khả năng file đầu ra đã tồn tại sẵn.

Init container có tất cả các trường của một app container. Tuy nhiên, Kubernetes
cấm sử dụng `readinessProbe` vì init container không thể định nghĩa trạng thái sẵn
sàng (readiness) tách biệt với việc hoàn thành. Điều này được thực thi trong quá
trình kiểm tra hợp lệ (validation).

Dùng `activeDeadlineSeconds` trên Pod để ngăn init container thất bại mãi mãi.
Deadline này bao gồm cả các init container.
Tuy nhiên, khuyến nghị chỉ dùng `activeDeadlineSeconds` nếu các nhóm triển khai
ứng dụng của họ dưới dạng Job, vì `activeDeadlineSeconds` vẫn có hiệu lực ngay cả
sau khi initContainer đã hoàn thành.
Pod đang chạy đúng sẽ bị kill bởi `activeDeadlineSeconds` nếu bạn đặt giá trị này.

Tên của mỗi app container và init container trong một Pod phải là duy nhất; một
lỗi kiểm tra hợp lệ sẽ được ném ra cho bất kỳ container nào trùng tên với container
khác.

### Chia sẻ tài nguyên giữa các container (Resource sharing within containers) {#resource-sharing-within-containers}

Với thứ tự thực thi của init container, sidecar container và app container, các
quy tắc sau về sử dụng tài nguyên được áp dụng:

* Giá trị cao nhất của bất kỳ resource request hay limit cụ thể nào được định
  nghĩa trên tất cả init container là *request/limit khởi tạo hiệu dụng*
  (effective init request/limit). Nếu bất kỳ tài nguyên nào không có resource
  limit được chỉ định, giá trị này được coi là limit cao nhất.
* *Request/limit hiệu dụng* của Pod đối với một tài nguyên là giá trị cao hơn giữa:
  * tổng của tất cả request/limit của các app container đối với một tài nguyên
  * request/limit khởi tạo hiệu dụng đối với một tài nguyên
* Việc lập lịch (scheduling) được thực hiện dựa trên request/limit hiệu dụng,
  nghĩa là init container có thể chiếm trước (reserve) tài nguyên cho quá trình
  khởi tạo mà không được sử dụng trong suốt vòng đời của Pod.
* Hạng QoS (chất lượng dịch vụ) của *hạng QoS hiệu dụng* của Pod là hạng QoS chung
  cho cả init container lẫn app container.

Quota và limit được áp dụng dựa trên request và limit hiệu dụng của Pod.

### Init container và cgroup trên Linux (Init containers and Linux cgroups) {#cgroups}

Trên Linux, việc cấp phát tài nguyên cho các control group (cgroup) cấp Pod dựa
trên request và limit hiệu dụng của Pod, giống như scheduler.

### Các lý do khởi động lại Pod (Pod restart reasons) {#pod-restart-reasons}

Một Pod có thể khởi động lại, dẫn đến việc thực thi lại các init container, vì các
lý do sau:

* Container hạ tầng (infrastructure container) của Pod bị khởi động lại. Điều này
  hiếm gặp và phải được thực hiện bởi ai đó có quyền root trên các node.
* Tất cả container trong Pod bị kết thúc trong khi `restartPolicy` được đặt là
  Always, buộc phải khởi động lại, và bản ghi hoàn thành của init container đã bị
  mất do thu gom rác (garbage collection).

Pod sẽ không bị khởi động lại khi image của init container bị thay đổi, hoặc khi
bản ghi hoàn thành của init container bị mất do thu gom rác. Điều này áp dụng cho
Kubernetes v1.20 trở lên. Nếu bạn đang dùng phiên bản Kubernetes cũ hơn, hãy tham
khảo tài liệu của phiên bản bạn đang sử dụng.

## Tiếp theo (What's next)

Tìm hiểu thêm về các nội dung sau:
* [Tạo một Pod có init container](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-initialization/#create-a-pod-that-has-an-init-container).
* [Debug init container](https://kubernetes.io/docs/tasks/debug/debug-application/debug-init-containers/).
* Tổng quan về [kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/) và [kubectl](https://kubernetes.io/docs/reference/kubectl/).
* [Các loại probe](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#types-of-probe): liveness, readiness, startup probe.
* [Sidecar container](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers).

# Cấu hình các probe Liveness, Readiness và Startup (Configure Liveness, Readiness and Startup Probes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/>

Trang này hướng dẫn cách cấu hình các probe liveness, readiness và startup cho container.

Để biết thêm thông tin về probe, xem
[Các probe Liveness, Readiness và Startup](./49-probes-vi.md).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Định nghĩa một lệnh liveness (Define a liveness command)

Nhiều ứng dụng chạy trong thời gian dài cuối cùng sẽ chuyển sang trạng thái hỏng (broken state)
và không thể phục hồi ngoại trừ việc được khởi động lại. Kubernetes cung cấp các liveness probe
để phát hiện và khắc phục những tình huống như vậy.

Trong bài thực hành này, bạn tạo một Pod chạy một container dựa trên image
`registry.k8s.io/busybox:1.27.2`. Đây là file cấu hình của Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: registry.k8s.io/busybox:1.27.2
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

Trong file cấu hình, bạn có thể thấy Pod có một `Container` duy nhất.
Field `periodSeconds` chỉ định rằng kubelet cần thực hiện liveness probe mỗi 5 giây. Field
`initialDelaySeconds` cho kubelet biết rằng nó cần đợi 5 giây trước khi thực hiện probe đầu
tiên. Để thực hiện một probe, kubelet thực thi lệnh `cat /tmp/healthy` trong container đích.
Nếu lệnh thành công, nó trả về 0, và kubelet coi container là còn sống (alive) và khỏe mạnh
(healthy). Nếu lệnh trả về giá trị khác 0, kubelet sẽ kill container và khởi động lại nó.

Khi container khởi động, nó thực thi lệnh này:

```shell
/bin/sh -c "touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600"
```

Trong 30 giây đầu tiên của vòng đời container, có một file `/tmp/healthy` tồn tại.
Vì vậy trong 30 giây đầu tiên, lệnh `cat /tmp/healthy` trả về mã thành công. Sau 30 giây,
`cat /tmp/healthy` trả về mã thất bại.

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/probe/exec-liveness.yaml
```

Trong vòng 30 giây, xem các sự kiện (event) của Pod:

```shell
kubectl describe pod liveness-exec
```

Kết quả cho thấy chưa có liveness probe nào thất bại:

```none
Type    Reason     Age   From               Message
----    ------     ----  ----               -------
Normal  Scheduled  11s   default-scheduler  Successfully assigned default/liveness-exec to node01
Normal  Pulling    9s    kubelet, node01    Pulling image "registry.k8s.io/busybox:1.27.2"
Normal  Pulled     7s    kubelet, node01    Successfully pulled image "registry.k8s.io/busybox:1.27.2"
Normal  Created    7s    kubelet, node01    Created container liveness
Normal  Started    7s    kubelet, node01    Started container liveness
```

Sau 35 giây, xem lại các sự kiện của Pod:

```shell
kubectl describe pod liveness-exec
```

Ở cuối phần kết quả, có các thông báo cho biết các liveness probe đã thất bại, và các container
thất bại đã bị kill và được tạo lại.

```none
Type     Reason     Age                From               Message
----     ------     ----               ----               -------
Normal   Scheduled  57s                default-scheduler  Successfully assigned default/liveness-exec to node01
Normal   Pulling    55s                kubelet, node01    Pulling image "registry.k8s.io/busybox:1.27.2"
Normal   Pulled     53s                kubelet, node01    Successfully pulled image "registry.k8s.io/busybox:1.27.2"
Normal   Created    53s                kubelet, node01    Created container liveness
Normal   Started    53s                kubelet, node01    Started container liveness
Warning  Unhealthy  10s (x3 over 20s)  kubelet, node01    Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
Normal   Killing    10s                kubelet, node01    Container liveness failed liveness probe, will be restarted
```

Đợi thêm 30 giây nữa, và xác minh rằng container đã được khởi động lại:

```shell
kubectl get pod liveness-exec
```

Kết quả cho thấy `RESTARTS` đã tăng lên. Lưu ý rằng bộ đếm `RESTARTS` tăng ngay khi container
thất bại quay trở lại trạng thái running:

```none
NAME            READY     STATUS    RESTARTS   AGE
liveness-exec   1/1       Running   1          1m
```

## Định nghĩa một liveness HTTP request (Define a liveness HTTP request)

Một loại liveness probe khác sử dụng HTTP GET request. Đây là file cấu hình cho một Pod chạy
một container dựa trên image `registry.k8s.io/e2e-test-images/agnhost`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - name: liveness
    image: registry.k8s.io/e2e-test-images/agnhost:2.40
    args:
    - liveness
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: Custom-Header
          value: Awesome
      initialDelaySeconds: 3
      periodSeconds: 3
```

Trong file cấu hình, bạn có thể thấy Pod có một container duy nhất.
Field `periodSeconds` chỉ định rằng kubelet cần thực hiện liveness probe mỗi 3 giây. Field
`initialDelaySeconds` cho kubelet biết rằng nó cần đợi 3 giây trước khi thực hiện probe đầu
tiên. Để thực hiện một probe, kubelet gửi một HTTP GET request đến server đang chạy trong
container và lắng nghe (listen) trên port 8080. Nếu handler cho path `/healthz` của server trả
về mã thành công, kubelet coi container là còn sống và khỏe mạnh. Nếu handler trả về mã thất
bại, kubelet sẽ kill container và khởi động lại nó.

Bất kỳ mã nào lớn hơn hoặc bằng 200 và nhỏ hơn 400 đều biểu thị thành công. Bất kỳ mã nào khác
đều biểu thị thất bại.

Bạn có thể xem mã nguồn của server tại
[server.go](https://github.com/kubernetes/kubernetes/blob/master/test/images/agnhost/liveness/server.go).

Trong 10 giây đầu tiên container còn sống, handler `/healthz` trả về trạng thái 200. Sau đó,
handler trả về trạng thái 500.

```go
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    duration := time.Now().Sub(started)
    if duration.Seconds() > 10 {
        w.WriteHeader(500)
        w.Write([]byte(fmt.Sprintf("error: %v", duration.Seconds())))
    } else {
        w.WriteHeader(200)
        w.Write([]byte("ok"))
    }
})
```

Kubelet bắt đầu thực hiện các kiểm tra sức khỏe (health check) 3 giây sau khi container khởi
động. Vì vậy vài lần kiểm tra đầu tiên sẽ thành công. Nhưng sau 10 giây, các lần kiểm tra sẽ
thất bại, và kubelet sẽ kill và khởi động lại container.

Để thử kiểm tra liveness HTTP, hãy tạo một Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/probe/http-liveness.yaml
```

Sau 10 giây, xem các sự kiện của Pod để xác minh rằng các liveness probe đã thất bại và
container đã được khởi động lại:

```shell
kubectl describe pod liveness-http
```

Trong các bản phát hành sau v1.13, các thiết lập biến môi trường HTTP proxy cục bộ không ảnh
hưởng đến HTTP liveness probe.

## Định nghĩa một TCP liveness probe (Define a TCP liveness probe)

Loại liveness probe thứ ba sử dụng TCP socket. Với cấu hình này, kubelet sẽ cố gắng mở một
socket đến container của bạn trên port được chỉ định. Nếu nó có thể thiết lập kết nối,
container được coi là khỏe mạnh; nếu không thể, kết quả được coi là thất bại.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: goproxy
  labels:
    app: goproxy
spec:
  containers:
  - name: goproxy
    image: registry.k8s.io/goproxy:0.1
    ports:
    - containerPort: 8080
    readinessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 10
```

Như bạn thấy, cấu hình cho kiểm tra TCP khá giống với kiểm tra HTTP.
Ví dụ này sử dụng cả readiness probe lẫn liveness probe. Kubelet sẽ chạy liveness probe đầu
tiên 15 giây sau khi container khởi động. Probe này sẽ cố gắng kết nối đến container `goproxy`
trên port 8080. Nếu liveness probe thất bại, container sẽ được khởi động lại. Kubelet sẽ tiếp
tục chạy kiểm tra này mỗi 10 giây.

Ngoài liveness probe, cấu hình này còn bao gồm một readiness probe. Kubelet sẽ chạy readiness
probe đầu tiên 15 giây sau khi container khởi động. Tương tự như liveness probe, probe này sẽ
cố gắng kết nối đến container `goproxy` trên port 8080. Nếu probe thành công, Pod sẽ được đánh
dấu là sẵn sàng (ready) và sẽ nhận lưu lượng (traffic) từ các Service. Nếu readiness probe
thất bại, Pod sẽ bị đánh dấu là chưa sẵn sàng (unready) và sẽ không nhận lưu lượng từ bất kỳ
Service nào.

Để thử kiểm tra liveness TCP, hãy tạo một Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/probe/tcp-liveness-readiness.yaml
```

Sau 15 giây, xem các sự kiện của Pod để xác minh các liveness probe:

```shell
kubectl describe pod goproxy
```

## Định nghĩa một gRPC liveness probe (Define a gRPC liveness probe)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.27 [stable]`

Nếu ứng dụng của bạn hiện thực
[gRPC Health Checking Protocol](https://github.com/grpc/grpc/blob/master/doc/health-checking.md),
ví dụ này hướng dẫn cách cấu hình Kubernetes để sử dụng nó cho các kiểm tra liveness của ứng
dụng. Tương tự, bạn có thể cấu hình readiness probe và startup probe.

Đây là một manifest ví dụ:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: etcd-with-grpc
spec:
  containers:
  - name: etcd
    image: registry.k8s.io/etcd:3.5.1-0
    command: [ "/usr/local/bin/etcd", "--data-dir",  "/var/lib/etcd", "--listen-client-urls", "http://0.0.0.0:2379", "--advertise-client-urls", "http://127.0.0.1:2379", "--log-level", "debug"]
    ports:
    - containerPort: 2379
    livenessProbe:
      grpc:
        port: 2379
      initialDelaySeconds: 10
```

Để thử kiểm tra liveness gRPC, hãy tạo một Pod bằng lệnh dưới đây.
Trong ví dụ dưới đây, Pod etcd được cấu hình để sử dụng gRPC liveness probe.

```shell
kubectl apply -f https://k8s.io/examples/pods/probe/grpc-liveness.yaml
```

Sau 15 giây, xem các sự kiện của Pod để xác minh rằng kiểm tra liveness chưa thất bại:

```shell
kubectl describe pod etcd-with-grpc
```

Khi sử dụng gRPC probe, có một số chi tiết kỹ thuật cần lưu ý:

- Các probe chạy dựa trên địa chỉ IP của Pod hoặc hostname của nó.
  Hãy chắc chắn cấu hình gRPC endpoint của bạn để lắng nghe trên địa chỉ IP của Pod.
- Các probe không hỗ trợ bất kỳ tham số xác thực nào (như `-tls`).
- Không có mã lỗi cho các probe tích hợp sẵn (built-in). Mọi lỗi đều được coi là probe thất bại.
- Nếu feature gate `ExecProbeTimeout` được đặt là `false`, grpc-health-probe **không** tôn
  trọng thiết lập `timeoutSeconds` (mặc định là 1s), trong khi probe tích hợp sẵn sẽ thất bại
  khi hết thời gian chờ (timeout).

## Sử dụng port có tên (Use a named port)

Bạn có thể sử dụng một
[`port`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#ports)
có tên (named port) cho các probe HTTP và TCP. Các gRPC probe không hỗ trợ port có tên.

Ví dụ:

```yaml
ports:
- name: liveness-port
  containerPort: 8080

livenessProbe:
  httpGet:
    path: /healthz
    port: liveness-port
```

## Bảo vệ các container khởi động chậm bằng startup probe (Protect slow starting containers with startup probes) {#define-startup-probes}

Đôi khi, bạn phải xử lý các ứng dụng cần thêm thời gian khởi động trong lần khởi tạo đầu tiên.
Trong những trường hợp như vậy, việc thiết lập các tham số liveness probe có thể trở nên khó
khăn nếu không muốn đánh đổi khả năng phản ứng nhanh với deadlock — vốn là lý do tồn tại của
probe này. Giải pháp là thiết lập một startup probe với cùng lệnh, kiểm tra HTTP hoặc TCP, với
`failureThreshold * periodSeconds` đủ dài để bao phủ thời gian khởi động trong trường hợp xấu
nhất.

Như vậy, ví dụ trước sẽ trở thành:

```yaml
ports:
- name: liveness-port
  containerPort: 8080

livenessProbe:
  httpGet:
    path: /healthz
    port: liveness-port
  failureThreshold: 1
  periodSeconds: 10

startupProbe:
  httpGet:
    path: /healthz
    port: liveness-port
  failureThreshold: 30
  periodSeconds: 10
```

Nhờ startup probe, ứng dụng sẽ có tối đa 5 phút (30 * 10 = 300s) để hoàn tất quá trình khởi
động. Một khi startup probe đã thành công một lần, liveness probe sẽ tiếp quản để cung cấp
phản ứng nhanh với các deadlock của container.
Nếu startup probe không bao giờ thành công, container sẽ bị kill sau 300s và chịu sự chi phối
của `restartPolicy` của Pod.

## Định nghĩa các readiness probe (Define readiness probes) {#define-readiness-probes}

Đôi khi, các ứng dụng tạm thời không thể phục vụ lưu lượng.
Ví dụ, một ứng dụng có thể cần nạp dữ liệu lớn hoặc các file cấu hình trong quá trình khởi
động, hoặc phụ thuộc vào các dịch vụ bên ngoài sau khi khởi động. Trong những trường hợp như
vậy, bạn không muốn kill ứng dụng, nhưng bạn cũng không muốn gửi request đến nó. Kubernetes
cung cấp các readiness probe để phát hiện và giảm thiểu những tình huống này. Một Pod có các
container báo cáo rằng chúng chưa sẵn sàng sẽ không nhận lưu lượng thông qua các Service của
Kubernetes.

> **Ghi chú:**
> Các readiness probe chạy trên container trong suốt toàn bộ vòng đời của nó.

> **Thận trọng:**
> Các readiness probe và liveness probe không phụ thuộc vào nhau để thành công.
> Nếu bạn muốn đợi trước khi thực thi một readiness probe, bạn nên dùng
> `initialDelaySeconds` hoặc một `startupProbe`.

Các readiness probe được cấu hình tương tự như liveness probe. Điểm khác biệt duy nhất là bạn
dùng field `readinessProbe` thay vì field `livenessProbe`.

```yaml
readinessProbe:
  exec:
    command:
    - /bin/cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

Cấu hình cho các readiness probe HTTP và TCP cũng giống hệt như liveness probe.

Các readiness probe và liveness probe có thể được dùng song song cho cùng một container.
Việc dùng cả hai có thể đảm bảo rằng lưu lượng không đến được container chưa sẵn sàng nhận nó,
và các container được khởi động lại khi chúng gặp lỗi.

## Tiếp theo (What's next)

* Tìm hiểu thêm về
  [Các probe Liveness, Readiness và Startup](./49-probes-vi.md).
* Để xem đặc tả đầy đủ của các field liên quan đến probe, xem tài liệu tham khảo API:
  [Pod](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/),
  [Container](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Container),
  [Probe](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Probe)

# Cấu hình các probe Liveness, Readiness và Startup (Configure Liveness, Readiness and Startup Probes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 3 — Pod và cấu hình](00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình)
→ nhóm [3a. Pod và vòng đời](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời), bài thực hành 4/11 ·
Kiểm chứng ở [Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md) phần B4.1, B4.2 và B4.3.

Đây là bài **dài nhất** trong khối thực hành 3a và là bài thực hành quan trọng nhất của nhóm.
Nó là mặt thi công của bài khái niệm [49](49-probes-vi.md): cùng ba loại probe, nhưng ở đây bạn
nhìn thấy field cụ thể và hậu quả cụ thể. Bài đi theo **kiểu handler** (`exec`, `httpGet`,
`tcpSocket`, `grpc`) rồi mới quay lại **loại probe** (startup, readiness), nên đừng lấy thứ tự
mục làm thứ tự quan trọng.

**Phải hiểu ở lần đọc này:**

- Bốn kiểu handler dùng chung một bộ field cấu hình, chỉ khác cách quyết định thành công: `exec`
  thành công khi lệnh trả về 0; `httpGet` thành công khi mã trả về **≥ 200 và < 400**;
  `tcpSocket` thành công khi **mở được kết nối**; `grpc` theo gRPC Health Checking Protocol.
- Hai field nhịp độ có mặt ở mọi ví dụ: `initialDelaySeconds` là thời gian kubelet đợi trước lần
  probe **đầu tiên**, `periodSeconds` là khoảng lặp giữa các lần probe.
- Hậu quả của liveness thất bại, đọc thẳng trên ví dụ `liveness-exec`: kubelet **kill container
  rồi khởi động lại nó**; `kubectl describe` hiện `Unhealthy` và `Killing`, còn `kubectl get pod`
  hiện cột **`RESTARTS` tăng** — bộ đếm tăng ngay khi container thất bại quay lại trạng thái
  running.
- Readiness cấu hình **giống hệt** liveness, khác đúng tên field (`readinessProbe` thay
  `livenessProbe`), nhưng hậu quả khác hẳn: Pod bị đánh dấu **chưa sẵn sàng** và không nhận lưu
  lượng — container **không** bị kill. Hai loại probe này **không phụ thuộc nhau để thành công**;
  muốn hoãn readiness thì dùng `initialDelaySeconds` hoặc một `startupProbe`, và readiness probe
  chạy suốt **toàn bộ vòng đời** container.
- Startup probe là ngân sách khởi động: `failureThreshold * periodSeconds` (ví dụ 30 × 10 = 300s)
  che cho liveness trong lúc ứng dụng khởi động chậm. Thành công **một lần** thì liveness tiếp
  quản; **không bao giờ thành công** thì container bị kill sau 300s và chịu sự chi phối của
  `restartPolicy` của Pod.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Định nghĩa một gRPC liveness probe* và các chi tiết kỹ thuật kèm theo (không hỗ trợ `-tls`, không có mã lỗi, `ExecProbeTimeout`) | lộ trình không có ứng dụng gRPC nào; lab dùng probe `exec` | ranh giới bốn kiểu handler đã nằm ở bài [49](49-probes-vi.md) của chính nhóm 3a |
| Câu "Pod sẽ nhận lưu lượng từ các Service" ở ví dụ TCP và ở mục readiness | Service chưa học; ở đây chỉ dừng được tới condition `Ready` của Pod | [giai đoạn 5](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng), thực hành ở Lab 5a |
| Mục *Sử dụng port có tên* | cần khai báo `ports` của container và một Service trỏ vào nó mới thấy được ích lợi | [giai đoạn 5](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng) |
| Mã Go của server `agnhost` và ghi chú về HTTP proxy cục bộ | là chi tiết của image ví dụ, không phải cơ chế probe | không cần cho lộ trình; cơ chế nằm ở bài [49](49-probes-vi.md) |

---

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

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Bốn kiểu handler của probe quyết định "thành công" bằng tiêu chí nào? Riêng `httpGet`, dải mã
   trả về nào được coi là thành công?
2. Bạn ghim Pod `liveness-exec` vào `lab-k8s-worker2` với `initialDelaySeconds: 5` và
   `periodSeconds: 5`, container chạy `touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep
   600`. Vì sao probe chỉ bắt đầu thất bại sau khoảng 30 giây, kubelet làm gì khi đó, và bạn nhìn
   vào đâu để chứng minh container đã bị khởi động lại?
3. **Câu bẫy.** Đổi `livenessProbe` thành `readinessProbe` mà giữ nguyên toàn bộ field bên dưới —
   manifest gần như y hệt. Hai probe đó khác nhau ở đâu, và cái nào **không** giết container?
4. Ứng dụng của bạn cần tới 4 phút để khởi động. Vì sao bài khuyên dùng `startupProbe` thay vì
   nới `initialDelaySeconds` của liveness, và cặp field nào quyết định ngân sách khởi động? Nếu
   startup probe không bao giờ thành công thì sao?
5. Bài có một mục *Thận trọng* về quan hệ giữa readiness và liveness. Nó cảnh báo điều gì, và
   bảo dùng gì nếu bạn muốn hoãn readiness probe?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. `exec`: kubelet thực thi lệnh trong container đích, **trả về 0 là thành công**, khác 0 là thất
   bại. `httpGet`: kubelet gửi HTTP GET tới container; **mã ≥ 200 và < 400 là thành công**, mọi
   mã khác là thất bại. `tcpSocket`: kubelet **cố mở một socket** tới port chỉ định, thiết lập
   được kết nối là khỏe mạnh, không được là thất bại. `grpc`: theo gRPC Health Checking Protocol,
   và **mọi lỗi đều được coi là probe thất bại** vì probe tích hợp sẵn không có mã lỗi riêng.
2. Vì trong **30 giây đầu** file `/tmp/healthy` còn tồn tại nên `cat /tmp/healthy` trả về mã thành
   công; sau 30 giây file bị xóa và lệnh trả về mã thất bại. Khi đó kubelet **kill container và
   khởi động lại nó**. Bằng chứng: `kubectl describe pod` hiện `Warning Unhealthy ... Liveness
   probe failed` và `Normal Killing ... will be restarted`, còn `kubectl get pod` hiện **cột
   `RESTARTS` tăng lên** — bộ đếm tăng ngay khi container thất bại quay trở lại trạng thái
   running.
3. Khác nhau ở **hậu quả**, không ở cấu hình — bài nói thẳng "điểm khác biệt duy nhất là bạn dùng
   field `readinessProbe` thay vì `livenessProbe`", và cấu hình HTTP/TCP của hai loại **giống hệt
   nhau**. Liveness thất bại → **container bị kill và khởi động lại**. Readiness thất bại → Pod bị
   **đánh dấu chưa sẵn sàng (unready)** và không nhận lưu lượng, **container vẫn chạy nguyên**.
   Đây là chỗ dễ sai nhất: nhìn manifest gần như không phân biệt được, phải nhìn tên field.
4. Vì nới `initialDelaySeconds` của liveness là đánh đổi: liveness đặt thưa để chịu được khởi
   động chậm thì mất luôn **khả năng phản ứng nhanh với deadlock** — vốn là lý do tồn tại của
   liveness probe. `startupProbe` tách hai nhu cầu đó ra. Ngân sách khởi động là
   **`failureThreshold * periodSeconds`** — trong ví dụ 30 × 10 = **300 giây**. Startup thành công
   **một lần** thì liveness tiếp quản. Nếu **không bao giờ thành công**, container **bị kill sau
   300s** và chịu sự chi phối của `restartPolicy` của Pod.
5. Cảnh báo rằng **readiness probe và liveness probe không phụ thuộc vào nhau để thành công** —
   đừng trông chờ cái này chờ cái kia. Muốn đợi trước khi thực thi một readiness probe thì dùng
   **`initialDelaySeconds`** hoặc một **`startupProbe`**. Kèm theo là ghi chú: readiness probe
   chạy trên container **trong suốt toàn bộ vòng đời** của nó, không phải chỉ lúc khởi động.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

# Các container sidecar (Sidecar Containers)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.33 [stable]`

Sidecar container là các container phụ chạy cùng với container ứng dụng chính bên trong
cùng một Pod. Các container này được dùng để tăng cường hoặc mở rộng chức năng của
_container ứng dụng_ (app container) chính bằng cách cung cấp các dịch vụ bổ sung, hoặc
các chức năng như ghi log (logging), giám sát (monitoring), bảo mật (security), hay đồng
bộ dữ liệu (data synchronization), mà không cần thay đổi trực tiếp mã nguồn của ứng dụng
chính.

Thông thường, bạn chỉ có một container ứng dụng trong một Pod. Ví dụ, nếu bạn có một
ứng dụng web cần một webserver cục bộ, thì webserver cục bộ đó là sidecar còn bản thân
ứng dụng web là container ứng dụng.

## Sidecar container trong Kubernetes (Sidecar containers in Kubernetes) {#pod-sidecar-containers}

Kubernetes triển khai sidecar container như một trường hợp đặc biệt của
[init container](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/);
sidecar container vẫn tiếp tục chạy sau khi Pod đã khởi động. Tài liệu này dùng thuật ngữ
_init container thông thường_ (regular init container) để chỉ rõ các container chỉ chạy
trong quá trình khởi động Pod.

Với điều kiện cluster của bạn đã bật
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`SidecarContainers` (tính năng này được kích hoạt mặc định kể từ Kubernetes v1.29), bạn có
thể chỉ định `restartPolicy` cho các container được liệt kê trong trường `initContainers`
của Pod. Các container _sidecar_ có thể khởi động lại (restartable) này độc lập với các
init container khác và với (các) container ứng dụng chính trong cùng một pod. Chúng có thể
được khởi động, dừng, hoặc khởi động lại mà không ảnh hưởng đến container ứng dụng chính
và các init container khác.

Bạn cũng có thể chạy một Pod với nhiều container không được đánh dấu là init container hay
sidecar container. Cách này phù hợp khi các container trong Pod đều cần thiết để Pod hoạt
động nói chung, nhưng bạn không cần kiểm soát container nào phải khởi động hay dừng trước.
Bạn cũng có thể làm như vậy nếu cần hỗ trợ các phiên bản Kubernetes cũ không hỗ trợ trường
`restartPolicy` ở cấp container.

### Ứng dụng ví dụ (Example application) {#sidecar-example}

Dưới đây là ví dụ về một Deployment có hai container, trong đó một container là sidecar:

> **Ghi chú:**
> Trong ví dụ này, sidecar container được định nghĩa một cách có chủ đích dưới
> `initContainers` với `restartPolicy: Always`. Kubernetes coi những container như vậy là
> sidecar và chúng tiếp tục chạy trong suốt vòng đời của Pod.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: alpine:latest
          command: ['sh', '-c', 'while true; do echo "logging" >> /opt/logs.txt; sleep 1; done']
          volumeMounts:
            - name: data
              mountPath: /opt
      initContainers:
        - name: logshipper
          image: alpine:latest
          # Đặt restartPolicy: Always biến container này thành một sidecar container.
          restartPolicy: Always
          command: ['sh', '-c', 'tail -F /opt/logs.txt']
          volumeMounts:
            - name: data
              mountPath: /opt
      volumes:
        - name: data
          emptyDir: {}
```

## Sidecar container và vòng đời của Pod (Sidecar containers and Pod lifecycle)

Nếu một init container được tạo với `restartPolicy` đặt là `Always`, nó sẽ khởi động và
tiếp tục chạy trong suốt vòng đời của Pod. Điều này hữu ích khi cần chạy các dịch vụ hỗ
trợ tách biệt khỏi các container ứng dụng chính.

Nếu một `readinessProbe` được chỉ định cho init container này, kết quả của nó sẽ được
dùng để xác định trạng thái `ready` của Pod.

Vì các container này được định nghĩa là init container, chúng được hưởng cùng các đảm bảo
về thứ tự và tính tuần tự như các init container thông thường, cho phép bạn kết hợp
sidecar container với các init container thông thường trong những luồng khởi tạo Pod
phức tạp.

So với init container thông thường, các sidecar được định nghĩa trong `initContainers`
tiếp tục chạy sau khi đã khởi động. Điều này quan trọng khi có nhiều hơn một mục trong
`.spec.initContainers` của một Pod. Sau khi một init container kiểu sidecar đang chạy
(kubelet đã đặt trạng thái `started` của init container đó thành true), kubelet mới khởi
động init container tiếp theo trong danh sách `.spec.initContainers` theo thứ tự. Trạng
thái đó trở thành true hoặc là vì có một tiến trình (process) đang chạy trong container
và không có startup probe nào được định nghĩa, hoặc là do `startupProbe` của nó thành công.

Khi Pod [kết thúc (termination)](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#termination-with-sidecars),
kubelet trì hoãn việc kết thúc các sidecar container cho đến khi container ứng dụng chính
đã dừng hoàn toàn. Các sidecar container sau đó được tắt theo thứ tự ngược lại với thứ tự
xuất hiện của chúng trong đặc tả (specification) của Pod. Cách tiếp cận này đảm bảo các
sidecar vẫn tiếp tục hoạt động, hỗ trợ các container khác trong Pod, cho đến khi dịch vụ
của chúng không còn cần thiết nữa.

### Job với sidecar container (Jobs with sidecar containers)

Nếu bạn định nghĩa một Job dùng sidecar theo kiểu init container của Kubernetes, sidecar
container trong mỗi Pod sẽ không ngăn Job hoàn thành (complete) sau khi container chính
đã kết thúc.

Dưới đây là ví dụ về một Job có hai container, trong đó một container là sidecar:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: myjob
spec:
  template:
    spec:
      containers:
        - name: myjob
          image: alpine:latest
          command: ['sh', '-c', 'echo "logging" > /opt/logs.txt']
          volumeMounts:
            - name: data
              mountPath: /opt
      initContainers:
        - name: logshipper
          image: alpine:latest
          # Đặt restartPolicy: Always biến container này thành một sidecar container.
          restartPolicy: Always
          command: ['sh', '-c', 'tail -F /opt/logs.txt']
          volumeMounts:
            - name: data
              mountPath: /opt
      restartPolicy: Never
      volumes:
        - name: data
          emptyDir: {}
```

## Khác biệt so với container ứng dụng (Differences from application containers)

Sidecar container chạy song song với các _container ứng dụng_ (app container) trong cùng
một pod. Tuy nhiên, chúng không thực thi logic ứng dụng chính; thay vào đó, chúng cung
cấp chức năng hỗ trợ cho ứng dụng chính.

Sidecar container có vòng đời độc lập của riêng mình. Chúng có thể được khởi động, dừng
và khởi động lại một cách độc lập với các container ứng dụng. Điều này có nghĩa là bạn có
thể cập nhật, mở rộng quy mô (scale) hoặc bảo trì các sidecar container mà không ảnh
hưởng đến ứng dụng chính.

Sidecar container chia sẻ cùng namespace về mạng và lưu trữ với container chính. Việc đặt
cạnh nhau (co-location) này cho phép chúng tương tác chặt chẽ và chia sẻ tài nguyên với
nhau.

Từ góc nhìn của Kubernetes, việc kết thúc êm (graceful termination) của sidecar container
ít quan trọng hơn. Khi các container khác dùng hết toàn bộ thời gian kết thúc êm được
cấp, các sidecar container sẽ nhận tín hiệu `SIGTERM`, theo sau là tín hiệu `SIGKILL`,
trước khi chúng kịp kết thúc một cách êm thấm. Vì vậy, việc sidecar container có mã thoát
(exit code) khác `0` (`0` biểu thị thoát thành công) khi Pod kết thúc là điều bình thường
và nhìn chung nên được các công cụ bên ngoài bỏ qua.

## Khác biệt so với init container (Differences from init containers)

Sidecar container làm việc song hành cùng container chính, mở rộng chức năng của nó và
cung cấp các dịch vụ bổ sung.

Sidecar container chạy đồng thời với container ứng dụng chính. Chúng hoạt động trong suốt
vòng đời của pod và có thể được khởi động, dừng độc lập với container chính. Không giống
[init container](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/),
sidecar container hỗ trợ các
[probe](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#types-of-probe)
để kiểm soát vòng đời của chúng.

Sidecar container có thể tương tác trực tiếp với các container ứng dụng chính, bởi vì
giống như init container, chúng luôn chia sẻ cùng một mạng, và tùy chọn cũng có thể chia
sẻ các volume (hệ thống tệp).

Init container dừng trước khi các container chính khởi động, do đó init container không
thể trao đổi thông điệp với container ứng dụng trong Pod. Mọi việc truyền dữ liệu đều là
một chiều (ví dụ, một init container có thể đặt thông tin vào trong một volume
`emptyDir`).

Việc thay đổi image của một sidecar container sẽ không làm Pod khởi động lại, nhưng sẽ
kích hoạt việc khởi động lại container đó.

## Chia sẻ tài nguyên giữa các container (Resource sharing within containers)

Với thứ tự thực thi của các init, sidecar và app container như trên, các quy tắc sau về
việc sử dụng tài nguyên được áp dụng:

* Giá trị cao nhất của một request hoặc limit tài nguyên cụ thể được định nghĩa trên tất
  cả các init container là *request/limit init hiệu dụng* (effective init request/limit).
  Nếu có tài nguyên nào không được chỉ định resource limit thì giá trị này được coi là
  limit cao nhất.
* *Request/limit hiệu dụng* của Pod đối với một tài nguyên là tổng của
  [pod overhead](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-overhead/)
  và giá trị lớn hơn giữa:
  * tổng request/limit của tất cả các container không phải init (app container và sidecar
    container) đối với tài nguyên đó
  * request/limit init hiệu dụng đối với tài nguyên đó
* Việc lập lịch (scheduling) được thực hiện dựa trên các request/limit hiệu dụng, nghĩa
  là init container có thể giữ trước (reserve) tài nguyên cho quá trình khởi tạo dù tài
  nguyên đó không được sử dụng trong suốt vòng đời của Pod.
* Hạng QoS (chất lượng dịch vụ — quality of service) trong *hạng QoS hiệu dụng* của Pod
  là hạng QoS chung cho tất cả các init, sidecar và app container.

Quota và limit được áp dụng dựa trên request và limit hiệu dụng của Pod.

### Sidecar container và cgroup trên Linux (Sidecar containers and Linux cgroups) {#cgroups}

Trên Linux, việc phân bổ tài nguyên cho các control group (cgroup) ở cấp Pod dựa trên
request và limit hiệu dụng của Pod, giống như scheduler.

## Tiếp theo (What's next)

* Tìm hiểu cách [áp dụng sidecar container](https://kubernetes.io/docs/tutorials/configuration/pod-sidecar-containers/).
* Đọc bài blog về [native sidecar container](https://kubernetes.io/blog/2023/08/25/native-sidecar-containers/).
* Đọc về [tạo một Pod có init container](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-initialization/#create-a-pod-that-has-an-init-container).
* Tìm hiểu về [các loại probe](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#types-of-probe): liveness probe, readiness probe, startup probe.
* Tìm hiểu về [pod overhead](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-overhead/).

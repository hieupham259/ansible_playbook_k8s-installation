# Các probe Liveness, Readiness và Startup (Liveness, Readiness, and Startup Probes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/pods/probes/>

Kubernetes cho phép bạn định nghĩa các _probe_ (đầu dò) để liên tục giám sát sức khỏe
của các container trong một Pod. Probe là một phép chẩn đoán được kubelet thực hiện
định kỳ trên một container. Để thực hiện phép chẩn đoán, kubelet hoặc thực thi mã bên
trong container, hoặc tạo một request qua mạng.

Dựa trên kết quả của các probe, Kubernetes có thể khởi động lại các container không
khỏe mạnh hoặc ngừng gửi lưu lượng (traffic) đến các container chưa sẵn sàng.

## Các loại probe (Types of probe) {#types-of-probe}

kubelet có thể (tùy chọn) thực hiện và phản ứng với ba loại probe trên các container
đang chạy, mỗi loại phục vụ một mục đích khác nhau:

- [Startup probe](#startup-probe)
- [Liveness probe](#liveness-probe)
- [Readiness probe](#readiness-probe)

### Startup probe {#startup-probe}

Startup probe xác minh xem ứng dụng bên trong container đã khởi động hay chưa.
Nếu một startup probe được cấu hình, Kubernetes sẽ không thực thi liveness probe hay
readiness probe cho đến khi startup probe thành công, nhờ đó ứng dụng có thời gian
hoàn tất quá trình khởi tạo của nó.

Loại probe này chỉ được thực thi lúc khởi động, khác với liveness probe và readiness
probe vốn chạy định kỳ.
Nếu startup probe thất bại, kubelet sẽ kill container, và container chịu sự chi phối
của [chính sách khởi động lại (restart policy)](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy) của nó.

### Liveness probe {#liveness-probe}

Liveness probe xác định khi nào cần khởi động lại một container.
Ví dụ, liveness probe có thể bắt được tình huống deadlock, khi ứng dụng vẫn đang chạy
nhưng không thể tiến triển. Khởi động lại container trong trạng thái như vậy có thể
giúp ứng dụng khả dụng hơn dù vẫn còn lỗi.

Nếu một container thất bại liveness probe nhiều lần hơn mức chịu đựng đã cấu hình,
kubelet sẽ khởi động lại container đó.
Liveness probe không chờ readiness probe thành công. Nếu bạn muốn chờ trước khi thực
thi liveness probe, bạn có thể định nghĩa `initialDelaySeconds` hoặc dùng
[startup probe](#startup-probe).

> **Thận trọng:**
> Liveness probe có thể là một cách mạnh mẽ để khôi phục sau lỗi ứng dụng,
> nhưng cần dùng chúng một cách thận trọng.
> Liveness probe phải được cấu hình cẩn thận để đảm bảo rằng chúng thật sự chỉ ra
> lỗi ứng dụng không thể khôi phục, ví dụ như deadlock.
>
> Triển khai liveness probe không đúng có thể dẫn đến lỗi dây chuyền (cascading
> failures). Hậu quả là container bị khởi động lại khi tải cao; các request của
> client thất bại do ứng dụng của bạn giảm khả năng mở rộng; và khối lượng công việc
> tăng lên trên các pod còn lại do một số pod bị lỗi. Hãy hiểu rõ sự khác biệt giữa
> liveness probe và readiness probe cũng như khi nào nên áp dụng chúng cho ứng dụng
> của bạn.

### Readiness probe {#readiness-probe}

Readiness probe xác định khi nào một container sẵn sàng nhận lưu lượng.
Điều này hữu ích khi cần chờ ứng dụng thực hiện các tác vụ ban đầu tốn thời gian,
chẳng hạn thiết lập kết nối mạng, nạp file, và làm nóng (warm) cache.
Readiness probe cũng có thể hữu ích ở giai đoạn sau trong vòng đời của container,
ví dụ khi khôi phục từ các sự cố tạm thời hoặc tình trạng quá tải.

Nếu readiness probe trả về trạng thái thất bại, controller EndpointSlice sẽ loại bỏ
địa chỉ IP của Pod khỏi các EndpointSlice của tất cả các Service khớp với Pod đó.

Readiness probe chạy trên container trong suốt toàn bộ vòng đời của nó.

> **Ghi chú:**
> Nếu bạn muốn có khả năng thoát dần (drain) các request khi Pod bị xóa, bạn không
> nhất thiết cần readiness probe; khi Pod bị xóa, endpoint tương ứng trong
> EndpointSlice sẽ cập nhật [các condition](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/#conditions)
> của nó: condition ready của endpoint sẽ được đặt thành false, do đó các bộ cân bằng
> tải (load balancer) sẽ không dùng Pod này cho lưu lượng thông thường. Xem
> [Pod termination](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination)
> để biết thêm thông tin về cách kubelet xử lý việc xóa Pod.

## Khi nào dùng từng loại probe (When to use each probe) {#when-to-use-each-probe}

### Khi nào bạn nên dùng startup probe? (When should you use a startup probe?) {#when-should-you-use-a-startup-probe}

Startup probe hữu ích cho các Pod có container cần nhiều thời gian mới bắt đầu phục
vụ được. Thay vì đặt một khoảng liveness dài, bạn có thể cấu hình riêng cho việc
probe container khi nó khởi động, cho phép thời gian dài hơn mức mà khoảng liveness
cho phép.

Nếu container của bạn thường khởi động lâu hơn
`initialDelaySeconds + failureThreshold × periodSeconds`, bạn nên chỉ định một
startup probe kiểm tra cùng endpoint với liveness probe. Giá trị mặc định của
`periodSeconds` là 10 giây. Khi đó bạn nên đặt `failureThreshold` của nó đủ cao để
container kịp khởi động, mà không phải thay đổi các giá trị mặc định của liveness
probe. Điều này giúp bảo vệ khỏi deadlock.

### Khi nào bạn nên dùng liveness probe? (When should you use a liveness probe?) {#when-should-you-use-a-liveness-probe}

Nếu tiến trình trong container của bạn có khả năng tự crash mỗi khi gặp sự cố hoặc
trở nên không khỏe mạnh, bạn không nhất thiết cần liveness probe; kubelet sẽ tự động
thực hiện hành động đúng theo `restartPolicy` của Pod.

Nếu bạn muốn container bị kill và khởi động lại khi một probe thất bại, hãy chỉ định
liveness probe, và chỉ định `restartPolicy` là `Always` hoặc `OnFailure`.

Một mẫu (pattern) phổ biến cho liveness probe là dùng cùng một HTTP endpoint chi phí
thấp như của readiness probe, nhưng với `failureThreshold` cao hơn. Điều này đảm bảo
pod được quan sát là chưa sẵn sàng (not-ready) trong một khoảng thời gian trước khi
bị kill cứng.

### Khi nào bạn nên dùng readiness probe? (When should you use a readiness probe?) {#when-should-you-use-a-readiness-probe}

Để chỉ bắt đầu gửi lưu lượng đến một Pod khi probe thành công, hãy chỉ định readiness
probe. Readiness probe có thể giống hệt liveness probe, nhưng sự hiện diện của
readiness probe trong spec có nghĩa là Pod sẽ khởi động mà không nhận bất kỳ lưu
lượng nào và chỉ bắt đầu nhận lưu lượng sau khi probe bắt đầu thành công.

Bạn cũng có thể dùng readiness probe để cho phép container tự đưa mình ra khỏi hoạt
động nhằm bảo trì, bằng cách kiểm tra một endpoint dành riêng cho readiness, khác
với endpoint của liveness probe.

Khi ứng dụng của bạn phụ thuộc chặt chẽ vào các dịch vụ backend, bạn có thể triển
khai cả liveness probe lẫn readiness probe. Liveness probe đạt khi bản thân ứng dụng
khỏe mạnh, còn readiness probe kiểm tra thêm rằng từng dịch vụ backend cần thiết đều
khả dụng. Điều này giúp bạn tránh chuyển lưu lượng đến các Pod chỉ có thể trả lời
bằng thông báo lỗi.

Với các container cần xử lý việc nạp dữ liệu lớn, file cấu hình, hoặc migration trong
lúc khởi động, hãy cân nhắc dùng [startup probe](#startup-probe). Tuy nhiên, nếu bạn
muốn phân biệt giữa một ứng dụng đã thất bại và một ứng dụng vẫn đang xử lý dữ liệu
khởi động của nó, có lẽ bạn sẽ ưu tiên readiness probe hơn.

## Các cơ chế kiểm tra (Check mechanisms) {#check-mechanisms}

Có bốn cách khác nhau để kiểm tra một container bằng probe. Mỗi probe phải định nghĩa
đúng một trong bốn cơ chế sau:

`exec`
: Thực thi một lệnh được chỉ định bên trong container. Phép chẩn đoán được coi là
  thành công nếu lệnh thoát với status code bằng 0.

`grpc`
: Thực hiện một lời gọi thủ tục từ xa (remote procedure call) bằng [gRPC](https://grpc.io/).
  Đích cần triển khai [gRPC health checks](https://grpc.io/grpc/core/md_doc_health-checking.html).
  Phép chẩn đoán được coi là thành công nếu `status` của response là `SERVING`.
  Xem chi tiết tại [gRPC probes](#grpc-probes).

`httpGet`
: Thực hiện một request HTTP `GET` tới địa chỉ IP của Pod trên một port và path được
  chỉ định. Phép chẩn đoán được coi là thành công nếu response có status code lớn hơn
  hoặc bằng 200 và nhỏ hơn 400.
  Xem chi tiết tại [HTTP probes](#http-probes).

`tcpSocket`
: Thực hiện một phép kiểm tra TCP tới địa chỉ IP của Pod trên một port được chỉ định.
  Phép chẩn đoán được coi là thành công nếu port đang mở. Nếu hệ thống ở xa
  (container) đóng kết nối ngay sau khi nó được mở, điều này vẫn được tính là khỏe
  mạnh.
  Xem chi tiết tại [TCP probes](#tcp-probes).

> **Thận trọng:**
> Khác với các cơ chế còn lại, phần triển khai của probe `exec` liên quan đến việc
> tạo/fork nhiều tiến trình mỗi lần thực thi. Do đó, với các cluster có mật độ pod
> cao hơn và các khoảng `initialDelaySeconds`, `periodSeconds` thấp hơn, việc cấu
> hình bất kỳ probe nào dùng cơ chế exec có thể gây thêm chi phí (overhead) lên mức
> sử dụng CPU của node. Trong các tình huống như vậy, hãy cân nhắc dùng các cơ chế
> probe thay thế để tránh chi phí này.

## Kết quả của probe (Probe results) {#probe-results}

kubelet đánh giá kết quả của mỗi lần thực thi probe và hành động tương ứng. Mỗi
probe có một trong ba kết quả:

`Success`
: Container vượt qua phép chẩn đoán.

`Failure`
: Container thất bại phép chẩn đoán. Với liveness probe và startup probe, kubelet
  sẽ kill container, và container chịu sự chi phối của
  [chính sách khởi động lại (restart policy)](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy) của nó.
  Với readiness probe, kubelet đánh dấu container là chưa sẵn sàng, và Pod ngừng
  nhận lưu lượng từ các Service khớp với nó.

`Unknown`
: Phép chẩn đoán thất bại (không có hành động nào được thực hiện, và kubelet sẽ
  tiếp tục kiểm tra thêm).

Nếu một container không cung cấp một probe cụ thể, kubelet luôn coi kết quả là
`Success`. Riêng với readiness probe, kết quả được coi là `Failure` trước khoảng
trễ ban đầu (initial delay).

## Các trường cấu hình (Configuration fields) {#configure-probes}

[Probe](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#probe-v1-core)
có một số trường mà bạn có thể dùng để kiểm soát chính xác hơn hành vi của các phép
kiểm tra startup, liveness và readiness. Ví dụ:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-example
spec:
  containers:
  - name: app
    image: registry.k8s.io/e2e-test-images/agnhost:2.40
    ports:
    - containerPort: 8080
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
```

`initialDelaySeconds`
: Số giây sau khi container đã khởi động trước khi các probe startup, liveness hoặc
  readiness được bắt đầu. Nếu một startup probe được định nghĩa, khoảng trễ của
  liveness probe và readiness probe không bắt đầu cho đến khi startup probe thành
  công. Trong một số phiên bản Kubernetes cũ hơn, initialDelaySeconds có thể bị bỏ
  qua nếu periodSeconds được đặt thành giá trị cao hơn initialDelaySeconds. Tuy
  nhiên, ở các phiên bản hiện tại, initialDelaySeconds luôn được tôn trọng và probe
  sẽ không bắt đầu cho đến sau khoảng trễ ban đầu này. Mặc định là 0 giây. Giá trị
  tối thiểu là 0.

`periodSeconds`
: Tần suất thực hiện probe (tính bằng giây). Mặc định là 10 giây. Giá trị tối thiểu
  là 1. Khi một container chưa Ready, readiness probe có thể được thực thi vào những
  thời điểm khác với khoảng `periodSeconds` đã cấu hình. Điều này nhằm giúp Pod trở
  nên sẵn sàng nhanh hơn.

`timeoutSeconds`
: Số giây sau đó probe bị hết thời gian chờ (timeout). Mặc định là 1 giây. Giá trị
  tối thiểu là 1.

`successThreshold`
: Số lần thành công liên tiếp tối thiểu để probe được coi là thành công sau khi đã
  thất bại. Mặc định là 1. Phải là 1 đối với liveness probe và startup probe. Giá
  trị tối thiểu là 1.

`failureThreshold`
: Sau khi một probe thất bại `failureThreshold` lần liên tiếp, Kubernetes coi rằng
  phép kiểm tra tổng thể đã thất bại: container _không_ ready/khỏe mạnh/sống. Mặc
  định là 3. Giá trị tối thiểu là 1. Với trường hợp startup probe hoặc liveness
  probe, nếu ít nhất `failureThreshold` lần probe đã thất bại, Kubernetes coi
  container là không khỏe mạnh và kích hoạt khởi động lại cho riêng container đó.
  kubelet tôn trọng thiết lập `terminationGracePeriodSeconds` của container đó. Với
  readiness probe thất bại, kubelet tiếp tục chạy container không vượt qua các phép
  kiểm tra, đồng thời cũng tiếp tục chạy thêm các probe; vì phép kiểm tra thất bại,
  kubelet đặt [condition](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-conditions)
  `Ready` của Pod thành `false`.

`terminationGracePeriodSeconds`
: Cấu hình khoảng thời gian ân hạn (grace period) để kubelet chờ giữa lúc kích hoạt
  việc tắt container bị lỗi và lúc buộc container runtime dừng container đó. Mặc
  định là kế thừa giá trị `terminationGracePeriodSeconds` ở cấp Pod (30 giây nếu
  không chỉ định), và giá trị tối thiểu là 1. Xem
  [`terminationGracePeriodSeconds` cấp probe](#probe-level-terminationgraceperiodseconds)
  để biết thêm chi tiết.

> **Thận trọng:**
> Triển khai readiness probe không đúng có thể dẫn đến số lượng tiến trình trong
> container tăng lên không ngừng, và cạn kiệt tài nguyên (resource starvation) nếu
> tình trạng này không được kiểm soát.

### `terminationGracePeriodSeconds` cấp probe (Probe-level `terminationGracePeriodSeconds`) {#probe-level-terminationgraceperiodseconds}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.28 [stable]`

Từ phiên bản 1.25 trở lên, người dùng có thể chỉ định `terminationGracePeriodSeconds`
ở cấp probe như một phần của đặc tả probe. Khi cả `terminationGracePeriodSeconds`
cấp pod lẫn cấp probe đều được đặt, kubelet sẽ dùng giá trị cấp probe.

Khi đặt `terminationGracePeriodSeconds`, hãy lưu ý các điểm sau:

* kubelet luôn tôn trọng trường `terminationGracePeriodSeconds` cấp probe nếu nó
  có mặt trên một Pod.
* Nếu bạn có các Pod hiện hữu với trường `terminationGracePeriodSeconds` đã được
  đặt và bạn không còn muốn dùng thời gian ân hạn kết thúc theo từng probe nữa, bạn
  phải xóa các Pod hiện hữu đó.

Ví dụ:

```yaml
spec:
  terminationGracePeriodSeconds: 3600  # cấp pod
  containers:
  - name: test
    image: ...

    ports:
    - name: liveness-port
      containerPort: 8080

    livenessProbe:
      httpGet:
        path: /healthz
        port: liveness-port
      failureThreshold: 1
      periodSeconds: 60
      # Ghi đè terminationGracePeriodSeconds cấp pod
      terminationGracePeriodSeconds: 60
```

`terminationGracePeriodSeconds` cấp probe **không thể** được đặt cho readiness
probe. Nó sẽ bị API server từ chối.

## Chi tiết các cơ chế probe (Probe mechanism details) {#probe-mechanism-details}

### HTTP probes {#http-probes}

[HTTP probe](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#httpgetaction-v1-core)
có thêm các trường có thể được đặt trên `httpGet`:

* `host`: Tên host để kết nối tới, mặc định là IP của pod. Có lẽ bạn nên đặt "Host"
  trong `httpHeaders` thay vì dùng trường này.
* `scheme`: Scheme dùng để kết nối tới host (HTTP hoặc HTTPS). Mặc định là "HTTP".
* `path`: Path truy cập trên HTTP server. Mặc định là "/".
* `httpHeaders`: Các header tùy chỉnh đặt trong request. HTTP cho phép các header
  lặp lại.
* `port`: Tên hoặc số của port truy cập trên container. Số phải nằm trong khoảng
  từ 1 đến 65535.

Với một HTTP probe, kubelet gửi một HTTP request tới port và path được chỉ định để
thực hiện phép kiểm tra. kubelet gửi probe tới địa chỉ IP của Pod, trừ khi địa chỉ
bị ghi đè bởi trường tùy chọn `host` trong `httpGet`. Nếu trường `scheme` được đặt
là `HTTPS`, kubelet gửi một request HTTPS và bỏ qua bước xác minh certificate.
Trong hầu hết các tình huống, bạn không nên đặt trường `host`. Đây là một tình
huống mà bạn sẽ đặt nó: giả sử container lắng nghe trên 127.0.0.1 và trường
`hostNetwork` của Pod là true. Khi đó `host`, bên dưới `httpGet`, nên được đặt là
127.0.0.1. Nếu pod của bạn dựa vào virtual host, vốn có lẽ là trường hợp phổ biến
hơn, bạn không nên dùng `host` mà nên đặt header `Host` trong `httpHeaders`.

Với một HTTP probe, kubelet gửi hai request header ngoài header `Host` bắt buộc:

- `User-Agent`, mặc định là `kube-probe/v1.36` trong đó `v1.36` là phiên bản của
  kubelet.
- `Accept`, mặc định là `*/*`.

Bạn có thể ghi đè các header này bằng cách định nghĩa `httpHeaders` cho probe.
Ví dụ:

```yaml
livenessProbe:
  httpGet:
    httpHeaders:
      - name: Accept
        value: application/json

startupProbe:
  httpGet:
    httpHeaders:
      - name: User-Agent
        value: MyUserAgent
```

Bạn cũng có thể loại bỏ hai header này bằng cách định nghĩa chúng với giá trị rỗng.

```yaml
livenessProbe:
  httpGet:
    httpHeaders:
      - name: Accept
        value: ""

startupProbe:
  httpGet:
    httpHeaders:
      - name: User-Agent
        value: ""
```

#### Xử lý redirect (Redirect handling) {#http-probes-redirects}

Khi kubelet probe một container bằng HTTP, nó chỉ đi theo (follow) redirect nếu
redirect trỏ tới cùng host. Điều này bao gồm cả các redirect đổi giao thức từ HTTP
sang HTTPS, ngay cả khi probe được cấu hình với `scheme: HTTP`.

Nếu redirect trỏ tới một hostname khác, kubelet không đi theo nó. Thay vào đó,
kubelet coi probe là thành công và ghi lại một event `ProbeWarning`.

Nếu kubelet đi theo redirect và nhận tổng cộng từ 11 redirect trở lên, probe được
coi là thành công và một event `ProbeWarning` được ghi lại. Ví dụ:

```none
Events:
  Type     Reason        Age                     From               Message
  ----     ------        ----                    ----               -------
  Normal   Scheduled     29m                     default-scheduler  Successfully assigned default/httpbin-7b8bc9cb85-bjzwn to daocloud
  Normal   Pulling       29m                     kubelet            Pulling image "docker.io/kennethreitz/httpbin"
  Normal   Pulled        24m                     kubelet            Successfully pulled image "docker.io/kennethreitz/httpbin" in 5m12.402735213s
  Normal   Created       24m                     kubelet            Created container httpbin
  Normal   Started       24m                     kubelet            Started container httpbin
 Warning  ProbeWarning  4m11s (x1197 over 24m)  kubelet            Readiness probe warning: Probe terminated redirects
```

> **Thận trọng:**
> Khi xử lý một probe `httpGet`, kubelet ngừng đọc phần thân response (response
> body) sau 10KiB. Sự thành công của probe được xác định hoàn toàn bởi status code
> của response, vốn nằm trong các response header.
>
> Nếu bạn probe một endpoint trả về response body lớn hơn **10KiB**, kubelet vẫn
> sẽ đánh dấu probe là thành công dựa trên status code, nhưng nó sẽ đóng kết nối
> sau khi chạm giới hạn 10KiB. Việc đóng đột ngột này có thể khiến các lỗi
> **connection reset by peer** hoặc **broken pipe** xuất hiện trong log của ứng
> dụng, vốn khó phân biệt với các sự cố mạng thật sự.
>
> Để có các probe `httpGet` đáng tin cậy, khuyến nghị mạnh mẽ là dùng các endpoint
> health check chuyên dụng trả về response body tối thiểu. Nếu bạn buộc phải dùng
> một endpoint có sẵn với payload lớn, hãy cân nhắc dùng probe `exec` để thực hiện
> request HEAD thay thế.

### TCP probes {#tcp-probes}

Với một TCP probe, kubelet tạo kết nối probe tại node, không phải bên trong Pod,
nghĩa là bạn không thể dùng tên service trong tham số `host` vì kubelet không thể
phân giải nó.

### gRPC probes {#grpc-probes}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.27 [stable]`

Nếu ứng dụng của bạn triển khai
[gRPC Health Checking Protocol](https://github.com/grpc/grpc/blob/master/doc/health-checking.md),
bạn có thể cấu hình Kubernetes dùng giao thức này cho các phép kiểm tra startup,
liveness hoặc readiness của ứng dụng.

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

Để dùng gRPC probe, `port` phải được cấu hình. Nếu bạn muốn phân biệt các probe
thuộc các loại khác nhau và các probe cho các tính năng khác nhau, bạn có thể dùng
trường `service`. Bạn có thể đặt `service` thành giá trị `liveness` và làm cho
endpoint gRPC Health Checking của bạn phản hồi request này khác với khi bạn đặt
`service` thành `readiness`. Điều này cho phép bạn dùng cùng một endpoint cho các
loại kiểm tra sức khỏe container khác nhau thay vì phải lắng nghe trên hai port
khác nhau. Nếu bạn muốn chỉ định tên service tùy chỉnh của riêng mình và đồng thời
chỉ định loại probe, dự án Kubernetes khuyến nghị bạn dùng một tên nối hai phần đó
lại. Ví dụ: `myservice-liveness` (dùng `-` làm ký tự phân cách).

> **Ghi chú:**
> Khác với HTTP probe hay TCP probe, bạn không thể chỉ định port health check theo
> tên, và bạn không thể cấu hình hostname tùy chỉnh.

Các vấn đề về cấu hình (ví dụ: port hoặc service không đúng, giao thức health
checking chưa được triển khai) được coi là probe thất bại, tương tự như HTTP probe
và TCP probe.

## Tiếp theo (What's next)

* Tìm hiểu cách
  [Cấu hình Liveness, Readiness và Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).
* Để xem đặc tả đầy đủ của các trường liên quan đến probe, xem tài liệu tham khảo API:
  [Pod](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/),
  [Container](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Container),
  [Probe](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Probe)

# Gỡ lỗi Pod (Debug Pods)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/>

Hướng dẫn này giúp người dùng gỡ lỗi (debug) các ứng dụng được triển khai vào Kubernetes nhưng
không hoạt động đúng như mong đợi. Đây *không phải* là hướng dẫn dành cho những người muốn gỡ lỗi
cluster của mình. Cho việc đó, bạn nên xem
[hướng dẫn này](https://kubernetes.io/docs/tasks/debug/debug-cluster).

## Chẩn đoán vấn đề (Diagnosing the problem)

Bước đầu tiên trong xử lý sự cố là phân loại (triage). Vấn đề nằm ở đâu?
Ở Pod, ở Replication Controller hay ở Service của bạn?

   * [Gỡ lỗi Pod](#debugging-pods)
   * [Gỡ lỗi Replication Controller](#debugging-replication-controllers)
   * [Gỡ lỗi Service](#debugging-services)

### Gỡ lỗi Pod (Debugging Pods) {#debugging-pods}

Bước đầu tiên khi gỡ lỗi một Pod là quan sát nó. Kiểm tra trạng thái hiện tại của Pod và các sự
kiện (event) gần đây bằng lệnh sau:

```shell
kubectl describe pods ${POD_NAME}
```

Hãy nhìn vào trạng thái của các container trong pod. Tất cả chúng có đang ở trạng thái `Running`
không? Gần đây có lần khởi động lại (restart) nào không?

Tiếp tục gỡ lỗi tùy theo trạng thái của các pod.

#### Pod của tôi cứ ở trạng thái pending (My pod stays pending)

Nếu một Pod bị kẹt ở trạng thái `Pending`, điều đó có nghĩa là nó không thể được lập lịch
(schedule) lên một node. Thông thường nguyên nhân là do thiếu hụt một loại tài nguyên nào đó
khiến việc lập lịch không thể diễn ra. Hãy xem output của lệnh `kubectl describe ...` ở trên.
Ở đó sẽ có các thông báo từ scheduler giải thích vì sao nó không thể lập lịch cho pod của bạn.
Các lý do bao gồm:

* **Bạn không có đủ tài nguyên**: Có thể bạn đã dùng cạn nguồn cung CPU hoặc Memory trong
  cluster; trong trường hợp này bạn cần xóa bớt Pod, điều chỉnh yêu cầu tài nguyên (resource
  request), hoặc thêm node mới vào cluster. Xem
  [tài liệu Tài nguyên tính toán](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
  để biết thêm thông tin.

* **Bạn đang dùng `hostPort`**: Khi bạn gắn một Pod vào một `hostPort`, số vị trí mà pod đó có
  thể được lập lịch là có hạn. Trong hầu hết các trường hợp, `hostPort` là không cần thiết; hãy
  thử dùng một đối tượng Service để expose Pod của bạn. Nếu bạn thực sự cần `hostPort` thì bạn
  chỉ có thể lập lịch số Pod tối đa bằng đúng số node trong cluster Kubernetes của mình.


#### Pod của tôi cứ ở trạng thái waiting (My pod stays waiting)

Nếu một Pod bị kẹt ở trạng thái `Waiting`, nghĩa là nó đã được lập lịch lên một worker node,
nhưng không thể chạy trên máy đó. Một lần nữa, thông tin từ `kubectl describe ...` sẽ rất hữu
ích. Nguyên nhân phổ biến nhất của các pod ở trạng thái `Waiting` là lỗi kéo (pull) image.
Có ba điều cần kiểm tra:

* Đảm bảo rằng bạn viết đúng tên image.
* Bạn đã đẩy (push) image lên registry chưa?
* Thử kéo image thủ công để xem image có thể kéo về được không. Ví dụ, nếu bạn dùng Docker trên
  PC của mình, hãy chạy `docker pull <image>`.


#### Pod của tôi cứ ở trạng thái terminating (My pod stays terminating)

Nếu một Pod bị kẹt ở trạng thái `Terminating`, nghĩa là một yêu cầu xóa đã được đưa ra cho Pod,
nhưng control plane không thể xóa đối tượng Pod đó.

Điều này thường xảy ra khi Pod có một
[finalizer](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/)
và trong cluster có cài một
[admission webhook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
ngăn không cho control plane gỡ bỏ finalizer đó.

Để nhận diện tình huống này, hãy kiểm tra xem cluster của bạn có
ValidatingWebhookConfiguration hoặc MutatingWebhookConfiguration nào nhắm vào các thao tác
`UPDATE` đối với tài nguyên `pods` hay không.

Nếu webhook do bên thứ ba cung cấp:
- Đảm bảo bạn đang dùng phiên bản mới nhất.
- Tắt webhook đối với các thao tác `UPDATE`.
- Báo cáo sự cố (issue) với nhà cung cấp tương ứng.

Nếu bạn là tác giả của webhook:
- Với mutating webhook, đảm bảo nó không bao giờ thay đổi các trường bất biến (immutable field)
  trong các thao tác `UPDATE`. Ví dụ, các thay đổi đối với containers thường không được phép.
- Với validating webhook, đảm bảo các chính sách kiểm tra (validation policy) của bạn chỉ áp
  dụng cho các thay đổi mới. Nói cách khác, bạn nên cho phép các Pod đang có sẵn vi phạm được
  vượt qua bước kiểm tra. Điều này cho phép các Pod đã được tạo trước khi validating webhook
  được cài đặt tiếp tục chạy.

#### Pod của tôi bị crash hoặc không khỏe mạnh (My pod is crashing or otherwise unhealthy)

Một khi pod của bạn đã được lập lịch, các phương pháp được mô tả trong
[Gỡ lỗi Pod đang chạy](300-debug-running-pod-vi.md)
sẽ sẵn sàng cho việc gỡ lỗi.

#### Pod của tôi đang chạy nhưng không làm điều tôi yêu cầu (My pod is running but not doing what I told it to do)

Nếu pod của bạn không hoạt động như mong đợi, có thể đã có lỗi trong phần mô tả pod (ví dụ file
`mypod.yaml` trên máy cục bộ của bạn), và lỗi đó đã bị bỏ qua một cách âm thầm khi bạn tạo pod.
Thường thì một phần của mô tả pod bị lồng (nested) sai chỗ, hoặc tên một key bị gõ nhầm, và vì
vậy key đó bị bỏ qua. Ví dụ, nếu bạn gõ nhầm `command` thành `commnd` thì pod vẫn sẽ được tạo
nhưng sẽ không dùng dòng lệnh mà bạn dự định.

Việc đầu tiên cần làm là xóa pod của bạn và thử tạo lại nó với tùy chọn `--validate`.
Ví dụ, chạy `kubectl apply --validate -f mypod.yaml`.
Nếu bạn gõ nhầm `command` thành `commnd` thì sẽ nhận được lỗi như sau:

```shell
I0805 10:43:25.129850   46757 schema.go:126] unknown field: commnd
I0805 10:43:25.129973   46757 schema.go:129] this may be a false alarm, see https://github.com/kubernetes/kubernetes/issues/6842
pods/mypod
```

Điều tiếp theo cần kiểm tra là liệu pod trên apiserver có khớp với pod bạn định tạo hay không
(ví dụ so với file yaml trên máy cục bộ của bạn).
Ví dụ, chạy `kubectl get pods/mypod -o yaml > mypod-on-apiserver.yaml` rồi so sánh thủ công
phần mô tả pod gốc, `mypod.yaml`, với bản bạn nhận lại từ apiserver,
`mypod-on-apiserver.yaml`. Thường sẽ có một số dòng trên phiên bản "apiserver" mà không có
trong phiên bản gốc. Điều này là bình thường. Tuy nhiên, nếu có những dòng trong bản gốc mà
không có trong phiên bản trên apiserver, thì đó có thể là dấu hiệu của vấn đề trong spec của
pod.

### Gỡ lỗi Replication Controller (Debugging Replication Controllers) {#debugging-replication-controllers}

Replication controller khá đơn giản: hoặc chúng tạo được Pod, hoặc không. Nếu chúng không tạo
được pod, hãy tham khảo [hướng dẫn ở trên](#debugging-pods) để gỡ lỗi các pod của bạn.

Bạn cũng có thể dùng `kubectl describe rc ${CONTROLLER_NAME}` để xem xét các sự kiện liên quan
đến replication controller.

### Gỡ lỗi Service (Debugging Services) {#debugging-services}

Service cung cấp cân bằng tải (load balancing) trên một tập các pod. Có một vài vấn đề phổ biến
có thể khiến Service không hoạt động đúng. Các hướng dẫn sau đây sẽ giúp gỡ lỗi các vấn đề của
Service.

Trước tiên, hãy xác minh rằng service có endpoint. Với mỗi đối tượng Service, apiserver cung cấp
một hoặc nhiều tài nguyên `EndpointSlice`.

Bạn có thể xem các tài nguyên này bằng lệnh:

```shell
kubectl get endpointslices -l kubernetes.io/service-name=${SERVICE_NAME}
```

Đảm bảo rằng các endpoint trong các EndpointSlice khớp với số pod mà bạn kỳ vọng là thành viên
của service. Ví dụ, nếu Service của bạn dành cho một container nginx với 3 bản sao (replica),
bạn sẽ kỳ vọng thấy ba địa chỉ IP khác nhau trong các endpoint slice của Service.

#### Service của tôi thiếu endpoint (My service is missing endpoints)

Nếu bạn bị thiếu endpoint, hãy thử liệt kê các pod bằng chính các label mà Service đang dùng.
Giả sử bạn có một Service với các label như sau:

```yaml
...
spec:
  - selector:
     name: nginx
     type: frontend
```

Bạn có thể dùng:

```shell
kubectl get pods --selector=name=nginx,type=frontend
```

để liệt kê các pod khớp với selector này. Xác minh rằng danh sách đó khớp với các Pod mà bạn kỳ
vọng sẽ phục vụ cho Service của mình.
Xác minh rằng `containerPort` của pod khớp với `targetPort` của Service.

#### Lưu lượng mạng không được chuyển tiếp (Network traffic is not forwarded)

Vui lòng xem [gỡ lỗi service](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
để biết thêm thông tin.

## Tiếp theo (What's next)

Nếu không có cách nào ở trên giải quyết được vấn đề của bạn, hãy làm theo hướng dẫn trong
[tài liệu Gỡ lỗi Service](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
để đảm bảo rằng `Service` của bạn đang chạy, có `Endpoints`, và các `Pods` của bạn thực sự
đang phục vụ; rằng DNS hoạt động, các quy tắc iptables đã được cài đặt, và kube-proxy không có
biểu hiện bất thường.

Bạn cũng có thể xem [tài liệu xử lý sự cố](https://kubernetes.io/docs/tasks/debug/) để biết
thêm thông tin.

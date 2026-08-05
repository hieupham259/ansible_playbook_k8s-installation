# Mức sẵn sàng lập lịch của Pod (Pod Scheduling Readiness)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/pod-scheduling-readiness/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.30 [stable]`

Trước đây, các Pod được coi là sẵn sàng để lập lịch ngay khi được tạo. Kubernetes scheduler
luôn cần mẫn tìm các node để đặt tất cả các Pod đang pending. Tuy nhiên, trong
thực tế, một số Pod có thể ở trạng thái "thiếu tài nguyên thiết yếu" (miss-essential-resources) trong một thời gian dài.
Những Pod này thực chất gây xáo trộn (churn) cho scheduler (và cả các thành phần tích hợp hạ nguồn như Cluster AutoScaler)
một cách không cần thiết.

Bằng cách chỉ định/gỡ bỏ trường `.spec.schedulingGates` của một Pod, bạn có thể kiểm soát
thời điểm một Pod sẵn sàng được xem xét để lập lịch.

## Cấu hình schedulingGates của Pod (Configuring Pod schedulingGates)

Trường `schedulingGates` chứa một danh sách các chuỗi, và mỗi chuỗi được hiểu là một
tiêu chí mà Pod phải thỏa mãn trước khi được coi là có thể lập lịch. Trường này chỉ có thể
được khởi tạo khi Pod được tạo (bởi client, hoặc được biến đổi (mutate) trong quá trình admission).
Sau khi tạo, mỗi schedulingGate có thể được gỡ bỏ theo thứ tự tùy ý, nhưng không được phép thêm scheduling gate mới.

![pod-scheduling-gates-diagram](https://kubernetes.io/docs/images/podSchedulingGates.svg)

*Hình. SchedulingGates của Pod ([xem sơ đồ](https://mermaid.live/edit#pako:eNplkktTwyAUhf8KgzuHWpukaYszutGlK3caFxQuCVMCGSDVTKf_XfKyPlhxz4HDB9wT5lYAptgHFuBRsdKxenFMClMYFIdfUdRYgbiD6ItJTEbR8wpEq5UpUfnDTf-5cbPoJjcbXdcaE61RVJIiqJvQ_Y30D-OCt-t3tFjcR5wZayiVnIGmkv4NiEfX9jijKTmmRH5jf0sRugOP0HyHUc1m6KGMFP27cM28fwSJDluPpNKaXqVJzmFNfHD2APRKSjnNFx9KhIpmzSfhVls3eHdTRrwG8QnxKfEZUUNeYTDBNbiaKRF_5dSfX-BQQQ0FpnEqQLJWhwIX5hyXsjbYl85wTINrgeC2EZd_xFQy7b_VJ6GCdd-itkxALE84dE3fAqXyIUZya6Qqe711OspVCI2ny2Vv35QqVO3-htt66ZWomAvVcZcv8yTfsiSFfJOydZoKvl_ttjLJVlJsblcJw-czwQ0zr9ZeqGDgeR77b2jD8xdtjtDn))*

## Ví dụ sử dụng (Usage example)

Để đánh dấu một Pod là chưa sẵn sàng để lập lịch, bạn có thể tạo nó với một hoặc nhiều scheduling gate như sau:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  schedulingGates:
  - name: example.com/foo
  - name: example.com/bar
  containers:
  - name: pause
    image: registry.k8s.io/pause:3.6
```

Sau khi Pod được tạo, bạn có thể kiểm tra trạng thái của nó bằng:

```bash
kubectl get pod test-pod
```

Kết quả cho thấy Pod đang ở trạng thái `SchedulingGated`:

```none
NAME       READY   STATUS            RESTARTS   AGE
test-pod   0/1     SchedulingGated   0          7s
```

Bạn cũng có thể kiểm tra trường `schedulingGates` của nó bằng cách chạy:

```bash
kubectl get pod test-pod -o jsonpath='{.spec.schedulingGates}'
```

Kết quả là:

```none
[{"name":"example.com/foo"},{"name":"example.com/bar"}]
```

Để báo cho scheduler biết rằng Pod này đã sẵn sàng để lập lịch, bạn có thể gỡ bỏ hoàn toàn
`schedulingGates` của nó bằng cách áp dụng lại một manifest đã chỉnh sửa:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: pause
    image: registry.k8s.io/pause:3.6
```

Bạn có thể kiểm tra xem `schedulingGates` đã được xóa hết chưa bằng cách chạy:

```bash
kubectl get pod test-pod -o jsonpath='{.spec.schedulingGates}'
```

Kết quả được kỳ vọng là rỗng. Và bạn có thể kiểm tra trạng thái mới nhất của Pod bằng cách chạy:

```bash
kubectl get pod test-pod -o wide
```

Vì test-pod không yêu cầu bất kỳ tài nguyên CPU/bộ nhớ nào, trạng thái của Pod này được kỳ vọng
chuyển từ `SchedulingGated` trước đó sang `Running`:

```none
NAME       READY   STATUS    RESTARTS   AGE   IP         NODE
test-pod   1/1     Running   0          15s   10.0.0.4   node-2
```

## Khả năng quan sát (Observability)

Metric `scheduler_pending_pods` đi kèm một nhãn mới `"gated"` để phân biệt giữa một Pod
đã được thử lập lịch nhưng bị kết luận là không thể lập lịch (unschedulable), với một Pod được
đánh dấu tường minh là chưa sẵn sàng để lập lịch. Bạn có thể dùng
`scheduler_pending_pods{queue="gated"}` để xem kết quả của metric này.

## Các chỉ thị lập lịch có thể thay đổi của Pod (Mutable Pod scheduling directives)

Bạn có thể thay đổi các chỉ thị lập lịch (scheduling directive) của Pod trong khi chúng vẫn còn scheduling gate, với một số ràng buộc nhất định.
Ở mức tổng quát, bạn chỉ có thể siết chặt các chỉ thị lập lịch của một Pod. Nói cách khác, các chỉ thị
sau khi cập nhật sẽ khiến các Pod chỉ có thể được lập lịch trên một tập con của những node
mà trước đó nó có thể khớp. Cụ thể hơn, các quy tắc cập nhật chỉ thị lập lịch của một Pod như sau:

1. Với `.spec.nodeSelector`, chỉ cho phép thêm mới. Nếu trường này chưa có, sẽ được phép đặt nó.

2. Với `spec.affinity.nodeAffinity`, nếu đang là nil, thì được phép đặt bất kỳ giá trị nào.

3. Nếu `NodeSelectorTerms` đang rỗng, sẽ được phép đặt.
   Nếu không rỗng, thì chỉ cho phép thêm các `NodeSelectorRequirements` vào `matchExpressions`
   hoặc `fieldExpressions`, và không cho phép bất kỳ thay đổi nào đối với các `matchExpressions`
   và `fieldExpressions` hiện có. Lý do là các term trong
   `.requiredDuringSchedulingIgnoredDuringExecution.NodeSelectorTerms` được OR với nhau,
   trong khi các biểu thức trong `nodeSelectorTerms[].matchExpressions` và
   `nodeSelectorTerms[].fieldExpressions` được AND với nhau.

4. Với `.preferredDuringSchedulingIgnoredDuringExecution`, mọi cập nhật đều được cho phép.
   Lý do là các term ở mức preferred không mang tính ràng buộc bắt buộc (authoritative), nên các policy controller
   không kiểm định (validate) các term này.

## Tiếp theo (What's next)

* Đọc [KEP PodSchedulingReadiness](https://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/3521-pod-scheduling-readiness) để biết thêm chi tiết

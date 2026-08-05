# Eviction khởi phát qua API (API-initiated Eviction)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/api-eviction/>

Eviction khởi phát qua API (API-initiated eviction) là quá trình bạn sử dụng
[Eviction API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#create-eviction-pod-v1-core)
để tạo một đối tượng `Eviction` nhằm kích hoạt việc chấm dứt Pod một cách êm thấm (graceful pod termination).

Bạn có thể yêu cầu eviction bằng cách gọi trực tiếp Eviction API, hoặc theo cách
lập trình bằng một client của API server, chẳng hạn lệnh `kubectl drain`. Thao tác này
tạo ra một đối tượng `Eviction`, khiến API server chấm dứt (terminate) Pod.

Các eviction khởi phát qua API tôn trọng các
[`PodDisruptionBudgets`](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
và [`terminationGracePeriodSeconds`](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#pod-termination)
mà bạn đã cấu hình.

Việc dùng API để tạo một đối tượng Eviction cho một Pod giống như thực hiện một
[thao tác `DELETE`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#delete-delete-a-pod)
có kiểm soát theo chính sách (policy-controlled) trên Pod đó.

## Gọi Eviction API (Calling the Eviction API)

Bạn có thể dùng một [client theo ngôn ngữ lập trình của Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/access-cluster-api/#programmatic-access-to-the-api)
để truy cập Kubernetes API và tạo một đối tượng `Eviction`. Để làm điều đó, bạn
POST thao tác muốn thực hiện, tương tự ví dụ sau:

#### policy/v1

> **Ghi chú:** Eviction `policy/v1` khả dụng từ v1.22 trở đi. Dùng `policy/v1beta1` với các bản phát hành trước đó.

```json
{
  "apiVersion": "policy/v1",
  "kind": "Eviction",
  "metadata": {
    "name": "quux",
    "namespace": "default"
  }
}
```

#### policy/v1beta1

> **Ghi chú:** Đã ngưng dùng (deprecated) từ v1.22, được thay thế bởi `policy/v1`.

```json
{
  "apiVersion": "policy/v1beta1",
  "kind": "Eviction",
  "metadata": {
    "name": "quux",
    "namespace": "default"
  }
}
```

Ngoài ra, bạn có thể thử một thao tác eviction bằng cách truy cập API qua
`curl` hoặc `wget`, tương tự ví dụ sau:

```bash
curl -v -H 'Content-type: application/json' https://your-cluster-api-endpoint.example/api/v1/namespaces/default/pods/quux/eviction -d @eviction.json
```

## Cách hoạt động của eviction khởi phát qua API (How API-initiated eviction works)

Khi bạn yêu cầu một eviction qua API, API server thực hiện các kiểm tra
admission và phản hồi theo một trong các cách sau:

* `200 OK`: eviction được cho phép, subresource `Eviction` được tạo, và
  Pod bị xóa, tương tự như việc gửi một request `DELETE` đến URL của Pod.
* `429 Too Many Requests`: eviction hiện không được phép do
  PodDisruptionBudget đã cấu hình.
  Bạn có thể thử lại eviction sau đó. Bạn cũng có thể thấy
  phản hồi này do cơ chế giới hạn tốc độ (rate limiting) của API.
* `500 Internal Server Error`: eviction không được phép vì có
  cấu hình sai, ví dụ như khi nhiều PodDisruptionBudget cùng tham chiếu đến một Pod.

Nếu Pod mà bạn muốn evict không thuộc một workload có
PodDisruptionBudget, API server luôn trả về `200 OK` và cho phép
eviction.

Nếu API server cho phép eviction, Pod bị xóa theo trình tự sau:

1. Tài nguyên `Pod` trong API server được cập nhật với một dấu thời gian xóa
   (deletion timestamp), sau thời điểm đó API server coi tài nguyên `Pod` là đã chấm dứt.
   Tài nguyên `Pod` cũng được đánh dấu với grace period (thời gian ân hạn) đã cấu hình.
1. kubelet trên node nơi Pod cục bộ đang chạy nhận thấy tài nguyên `Pod`
   được đánh dấu để chấm dứt và bắt đầu tắt Pod cục bộ
   một cách êm thấm (gracefully).
1. Trong khi kubelet đang tắt Pod, control plane gỡ Pod
   khỏi các đối tượng EndpointSlice.
   Kết quả là các controller không còn coi Pod là một đối tượng hợp lệ nữa.
1. Sau khi grace period của Pod hết hạn, kubelet cưỡng bức chấm dứt
   (forcefully terminate) Pod cục bộ.
1. kubelet báo cho API server gỡ bỏ tài nguyên `Pod`.
1. API server xóa tài nguyên `Pod`.

## Xử lý sự cố eviction bị kẹt (Troubleshooting stuck evictions)

Trong một số trường hợp, các ứng dụng của bạn có thể rơi vào trạng thái hỏng,
khi đó Eviction API sẽ chỉ trả về các phản hồi `429` hoặc `500` cho đến khi bạn
can thiệp. Điều này có thể xảy ra nếu, ví dụ, một ReplicaSet tạo các pod cho
ứng dụng của bạn nhưng các pod mới không đạt được trạng thái `Ready`. Bạn cũng có thể
nhận thấy hành vi này trong các trường hợp Pod bị evict gần nhất có termination grace period dài.

Nếu bạn nhận thấy các eviction bị kẹt, hãy thử một trong các giải pháp sau:

* Hủy bỏ hoặc tạm dừng thao tác tự động đang gây ra sự cố. Điều tra
  ứng dụng bị kẹt trước khi bạn khởi động lại thao tác đó.
* Chờ một lúc, sau đó xóa trực tiếp Pod khỏi control plane của cluster
  thay vì dùng Eviction API.

## Tiếp theo (What's next)

* Tìm hiểu cách bảo vệ ứng dụng của bạn với một [Pod Disruption Budget](https://kubernetes.io/docs/tasks/run-application/configure-pdb/).
* Tìm hiểu về [Eviction do áp lực node (Node-pressure Eviction)](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/).
* Tìm hiểu về [Độ ưu tiên và Preemption của Pod (Pod Priority and Preemption)](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/).

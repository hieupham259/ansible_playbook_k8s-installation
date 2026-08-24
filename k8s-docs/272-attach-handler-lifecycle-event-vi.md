# Gắn handler vào các sự kiện vòng đời của Container (Attach Handlers to Container Lifecycle Events)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/attach-handler-lifecycle-event/>

Trang này hướng dẫn cách gắn handler vào các sự kiện vòng đời (lifecycle events) của Container.
Kubernetes hỗ trợ hai sự kiện postStart và preStop. Kubernetes gửi sự kiện postStart ngay sau
khi một Container được khởi động, và gửi sự kiện preStop ngay trước khi Container bị chấm dứt
(terminated). Mỗi Container có thể chỉ định một handler cho mỗi sự kiện.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, hãy nhập `kubectl version`.

## Định nghĩa handler postStart và preStop (Define postStart and preStop handlers)

Trong bài thực hành này, bạn sẽ tạo một Pod có một Container. Container này có handler cho
các sự kiện postStart và preStop.

Đây là file cấu hình cho Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/bin/sh","-c","nginx -s quit; while killall -0 nginx; do sleep 1; done"]
```

Trong file cấu hình, bạn có thể thấy lệnh postStart ghi một file `message` vào thư mục
`/usr/share` của Container. Lệnh preStop tắt nginx một cách êm thấm (gracefully). Điều này
hữu ích khi Container bị chấm dứt do một sự cố.

Tạo Pod:

    kubectl apply -f https://k8s.io/examples/pods/lifecycle-events.yaml

Xác nhận rằng Container trong Pod đang chạy:

    kubectl get pod lifecycle-demo

Mở một shell vào Container đang chạy trong Pod của bạn:

    kubectl exec -it lifecycle-demo -- /bin/bash

Trong shell, xác nhận rằng handler `postStart` đã tạo file `message`:

    root@lifecycle-demo:/# cat /usr/share/message

Kết quả hiển thị đoạn văn bản do handler postStart ghi ra:

    Hello from the postStart handler

## Thảo luận (Discussion)

Kubernetes gửi sự kiện postStart ngay sau khi Container được tạo. Tuy nhiên, không có gì đảm
bảo rằng handler postStart được gọi trước khi entrypoint của Container được gọi. Handler
postStart chạy bất đồng bộ (asynchronously) so với mã của Container, nhưng việc quản lý
container của Kubernetes sẽ bị chặn (block) cho đến khi handler postStart hoàn thành. Trạng
thái của Container không được đặt thành RUNNING cho đến khi handler postStart hoàn thành.

Kubernetes gửi sự kiện preStop ngay trước khi Container bị chấm dứt. Việc quản lý Container
của Kubernetes sẽ bị chặn cho đến khi handler preStop hoàn thành, trừ khi khoảng thời gian ân
hạn (grace period) của Pod hết hạn. Để biết thêm chi tiết, xem
[Vòng đời của Pod](47-pod-lifecycle-vi.md).

> **Ghi chú:**
>
> Kubernetes chỉ gửi sự kiện preStop khi một Pod hoặc một container trong Pod bị *chấm dứt*
> (terminated). Điều này có nghĩa là hook preStop không được gọi khi Pod *hoàn thành*
> (completed). Về hạn chế này, vui lòng xem chi tiết tại
> [Container hooks](42-container-lifecycle-hooks-vi.md#container-hooks).

## Tiếp theo (What's next)

* Tìm hiểu thêm về [các hook vòng đời của Container](42-container-lifecycle-hooks-vi.md).
* Tìm hiểu thêm về [vòng đời của một Pod](47-pod-lifecycle-vi.md).

### Tham khảo (Reference)

* [Lifecycle](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#lifecycle-v1-core)
* [Container](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#container-v1-core)
* Xem `terminationGracePeriodSeconds` trong [PodSpec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#podspec-v1-core)

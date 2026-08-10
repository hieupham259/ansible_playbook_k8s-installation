# Gỡ lỗi Init Container (Debug Init Containers)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/debug-application/debug-init-containers/>

Trang này chỉ ra cách điều tra các vấn đề liên quan đến việc thực thi Init Container. Các dòng
lệnh ví dụ dưới đây gọi Pod là `<pod-name>` và các Init Container là `<init-container-1>` và
`<init-container-2>`.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, nhập `kubectl version`.

* Bạn nên nắm được những điều cơ bản về [Init Container](./50-init-containers-vi.md).
* Bạn nên đã [Cấu hình một Init Container](./276-configure-pod-initialization-vi.md#tạo-một-pod-có-init-container-create-a-pod-that-has-an-init-container).

## Kiểm tra trạng thái của Init Container (Checking the status of Init Containers)

Hiển thị trạng thái của Pod của bạn:

```shell
kubectl get pod <pod-name>
```

Ví dụ, trạng thái `Init:1/2` cho biết một trong hai Init Container đã hoàn thành thành công:

```
NAME         READY     STATUS     RESTARTS   AGE
<pod-name>   0/1       Init:1/2   0          7s
```

Xem [Hiểu trạng thái Pod](#hiểu-trạng-thái-pod-understanding-pod-status) để có thêm ví dụ về
các giá trị trạng thái và ý nghĩa của chúng.

## Xem chi tiết về Init Container (Getting details about Init Containers)

Xem thông tin chi tiết hơn về việc thực thi Init Container:

```shell
kubectl describe pod <pod-name>
```

Ví dụ, một Pod có hai Init Container có thể hiển thị như sau:

```
Init Containers:
  <init-container-1>:
    Container ID:    ...
    ...
    State:           Terminated
      Reason:        Completed
      Exit Code:     0
      Started:       ...
      Finished:      ...
    Ready:           True
    Restart Count:   0
    ...
  <init-container-2>:
    Container ID:    ...
    ...
    State:           Waiting
      Reason:        CrashLoopBackOff
    Last State:      Terminated
      Reason:        Error
      Exit Code:     1
      Started:       ...
      Finished:      ...
    Ready:           False
    Restart Count:   3
    ...
```

Bạn cũng có thể truy cập trạng thái của các Init Container theo cách lập trình bằng cách đọc
trường `status.initContainerStatuses` trong Pod Spec:

```shell
kubectl get pod <pod-name> --template '{{.status.initContainerStatuses}}'
```

Lệnh này sẽ trả về cùng thông tin như trên, được định dạng bằng một
[Go template](https://pkg.go.dev/text/template).

## Truy cập log của Init Container (Accessing logs from Init Containers)

Truyền tên Init Container cùng với tên Pod để truy cập log của nó.

```shell
kubectl logs <pod-name> -c <init-container-2>
```

Các Init Container chạy shell script sẽ in ra các lệnh khi chúng được thực thi. Ví dụ, bạn có
thể làm điều này trong Bash bằng cách chạy `set -x` ở đầu script.

## Hiểu trạng thái Pod (Understanding Pod status)

Một trạng thái Pod bắt đầu bằng `Init:` tóm tắt trạng thái thực thi của các Init Container.
Bảng dưới đây mô tả một số giá trị trạng thái ví dụ mà bạn có thể gặp khi gỡ lỗi Init Container.

Trạng thái | Ý nghĩa
------ | -------
`Init:N/M` | Pod có `M` Init Container, và cho tới lúc này `N` container đã hoàn thành.
`Init:Error` | Một Init Container đã thực thi thất bại.
`Init:CrashLoopBackOff` | Một Init Container đã thất bại lặp đi lặp lại.
`Pending` | Pod chưa bắt đầu thực thi các Init Container.
`PodInitializing` hoặc `Running` | Pod đã thực thi xong các Init Container.

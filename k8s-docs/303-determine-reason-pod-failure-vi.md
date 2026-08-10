# Xác định nguyên nhân Pod bị lỗi (Determine the Reason for Pod Failure)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/debug-application/determine-reason-pod-failure/>

Trang này hướng dẫn cách ghi và đọc thông điệp kết thúc (termination message) của một Container.

Thông điệp kết thúc cung cấp một cách để container ghi thông tin về các sự kiện nghiêm trọng
(fatal event) vào một vị trí mà các công cụ như dashboard và phần mềm giám sát (monitoring) có
thể dễ dàng truy xuất và hiển thị. Trong hầu hết các trường hợp, thông tin bạn đưa vào thông
điệp kết thúc cũng nên được ghi vào
[log chung của Kubernetes](https://kubernetes.io/docs/concepts/cluster-administration/logging/).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Ghi và đọc thông điệp kết thúc (Writing and reading a termination message)

Trong bài thực hành này, bạn tạo một Pod chạy một container. Manifest của Pod đó chỉ định một
lệnh sẽ chạy khi container khởi động:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: termination-demo
spec:
  containers:
  - name: termination-demo-container
    image: debian
    command: ["/bin/sh"]
    args: ["-c", "sleep 10 && echo Sleep expired > /dev/termination-log"]
```

1. Tạo Pod dựa trên file cấu hình YAML:

    ```shell
    kubectl apply -f https://k8s.io/examples/debug/termination.yaml
    ```

    Trong file YAML, ở các trường `command` và `args`, bạn có thể thấy container ngủ (sleep)
    trong 10 giây rồi ghi "Sleep expired" vào file `/dev/termination-log`. Sau khi container
    ghi xong thông điệp "Sleep expired", nó kết thúc.

1. Hiển thị thông tin về Pod:

    ```shell
    kubectl get pod termination-demo
    ```

    Lặp lại lệnh trên cho đến khi Pod không còn chạy nữa.

1. Hiển thị thông tin chi tiết về Pod:

    ```shell
    kubectl get pod termination-demo --output=yaml
    ```

    Kết quả đầu ra bao gồm thông điệp "Sleep expired":

    ```yaml
    apiVersion: v1
    kind: Pod
    ...
        lastState:
          terminated:
            containerID: ...
            exitCode: 0
            finishedAt: ...
            message: |
              Sleep expired
            ...
    ```

1. Dùng một Go template để lọc kết quả đầu ra sao cho chỉ còn thông điệp kết thúc:

    ```shell
    kubectl get pod termination-demo -o go-template="{{range .status.containerStatuses}}{{.lastState.terminated.message}}{{end}}"
    ```

Nếu bạn đang chạy một Pod có nhiều container, bạn có thể dùng Go template để kèm theo tên của
container. Bằng cách đó, bạn có thể phát hiện container nào đang bị lỗi:

```shell
kubectl get pod multi-container-pod -o go-template='{{range .status.containerStatuses}}{{printf "%s:\n%s\n\n" .name .lastState.terminated.message}}{{end}}'
```

## Tùy chỉnh thông điệp kết thúc (Customizing the termination message)

Kubernetes truy xuất thông điệp kết thúc từ file thông điệp kết thúc được chỉ định trong trường
`terminationMessagePath` của một Container, với giá trị mặc định là `/dev/termination-log`.
Bằng cách tùy chỉnh trường này, bạn có thể yêu cầu Kubernetes dùng một file khác. Kubernetes
dùng nội dung của file được chỉ định để điền vào thông điệp trạng thái (status message) của
Container trong cả trường hợp thành công lẫn thất bại.

Thông điệp kết thúc được thiết kế để chứa trạng thái cuối cùng ngắn gọn, ví dụ như một thông
báo lỗi assertion. Kubelet cắt bớt (truncate) các thông điệp dài hơn 4096 byte.

Tổng độ dài thông điệp của tất cả các container bị giới hạn ở 12KiB, chia đều cho mỗi
container. Ví dụ, nếu có 12 container (`initContainers` hoặc `containers`), mỗi container có
1024 byte không gian dành cho thông điệp kết thúc.

Đường dẫn thông điệp kết thúc mặc định là `/dev/termination-log`. Bạn không thể thay đổi đường
dẫn thông điệp kết thúc sau khi Pod đã được khởi chạy.

Trong ví dụ sau, container ghi thông điệp kết thúc vào `/tmp/my-log` để Kubernetes truy xuất:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: msg-path-demo
spec:
  containers:
  - name: msg-path-demo-container
    image: debian
    terminationMessagePath: "/tmp/my-log"
```

Hơn nữa, người dùng có thể thiết lập trường `terminationMessagePolicy` của một Container để
tùy chỉnh sâu hơn. Trường này mặc định là "`File`", nghĩa là thông điệp kết thúc chỉ được truy
xuất từ file thông điệp kết thúc. Bằng cách đặt `terminationMessagePolicy` thành
"`FallbackToLogsOnError`", bạn có thể yêu cầu Kubernetes dùng đoạn cuối cùng của log container
nếu file thông điệp kết thúc rỗng và container thoát với lỗi. Phần log được lấy bị giới hạn ở
2048 byte hoặc 80 dòng, tùy theo giá trị nào nhỏ hơn.

## Tiếp theo (What's next)

* Xem trường `terminationMessagePath` trong
  [Container](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#container-v1-core).
* Xem [ImagePullBackOff](https://kubernetes.io/docs/concepts/containers/images/#imagepullbackoff)
  trong [Images](https://kubernetes.io/docs/concepts/containers/images/).
* Tìm hiểu về [truy xuất log](https://kubernetes.io/docs/concepts/cluster-administration/logging/).
* Tìm hiểu về [Go templates](https://pkg.go.dev/text/template).
* Tìm hiểu về [trạng thái Pod](https://kubernetes.io/docs/tasks/debug/debug-application/debug-init-containers/#understanding-pod-status)
  và [Pod phase](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-phase).
* Tìm hiểu về [trạng thái container](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-states).

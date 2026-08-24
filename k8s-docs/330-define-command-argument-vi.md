# Định nghĩa command và argument cho container (Define a Command and Arguments for a Container)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/>

Trang này hướng dẫn cách định nghĩa command (lệnh) và argument (đối số) khi bạn chạy một container
trong một Pod.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình
để giao tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít
nhất hai node không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có
thể tạo một cluster bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
hoặc dùng một trong các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, hãy chạy `kubectl version`.

## Định nghĩa command và argument khi bạn tạo Pod (Define a command and arguments when you create a Pod)

Khi tạo một Pod, bạn có thể định nghĩa command và argument cho các container
chạy trong Pod đó. Để định nghĩa một command, hãy đưa trường `command`
vào file cấu hình. Để định nghĩa argument cho command, hãy đưa
trường `args` vào file cấu hình. Command và argument mà bạn
định nghĩa không thể thay đổi sau khi Pod đã được tạo.

Command và argument mà bạn định nghĩa trong file cấu hình
sẽ ghi đè command và argument mặc định do container image cung cấp.
Nếu bạn định nghĩa args nhưng không định nghĩa command, command mặc định sẽ được dùng
cùng với các argument mới của bạn.

> **Ghi chú:** Trường `command` tương ứng với `ENTRYPOINT`, và trường `args` tương ứng với `CMD` trong một số container runtime.

Trong bài thực hành này, bạn tạo một Pod chạy một container. File cấu hình
của Pod định nghĩa một command và hai argument:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: command-demo
  labels:
    purpose: demonstrate-command
spec:
  containers:
  - name: command-demo-container
    image: debian
    command: ["printenv"]
    args: ["HOSTNAME", "KUBERNETES_PORT"]
  restartPolicy: OnFailure
```

1. Tạo một Pod dựa trên file cấu hình YAML:

   ```shell
   kubectl apply -f https://k8s.io/examples/pods/commands.yaml
   ```

1. Liệt kê các Pod đang chạy:

   ```shell
   kubectl get pods
   ```

   Kết quả xuất ra cho thấy container đã chạy trong Pod command-demo
   đã hoàn thành.

1. Để xem kết quả của command đã chạy trong container, hãy xem log
của Pod:

   ```shell
   kubectl logs command-demo
   ```

   Kết quả xuất ra hiển thị giá trị của các biến môi trường HOSTNAME và
   KUBERNETES_PORT:

   ```
   command-demo
   tcp://10.3.240.1:443
   ```

## Dùng biến môi trường để định nghĩa argument (Use environment variables to define arguments)

Trong ví dụ trước, bạn đã định nghĩa các argument một cách trực tiếp bằng cách
cung cấp các chuỗi ký tự. Thay vì cung cấp chuỗi trực tiếp,
bạn có thể định nghĩa argument bằng cách dùng các biến môi trường:

```yaml
env:
- name: MESSAGE
  value: "hello world"
command: ["/bin/echo"]
args: ["$(MESSAGE)"]
```

Điều này có nghĩa là bạn có thể định nghĩa một argument cho Pod bằng bất kỳ
kỹ thuật nào dùng để định nghĩa biến môi trường, bao gồm
[ConfigMap](275-configure-pod-configmap-vi.md)
và
[Secret](109-secret-vi.md).

> **Ghi chú:** Biến môi trường xuất hiện trong cặp ngoặc đơn, `"$(VAR)"`. Cách viết này
> là bắt buộc để biến được mở rộng (expand) trong trường `command` hoặc `args`.

## Chạy command trong shell (Run a command in a shell)

Trong một số trường hợp, bạn cần command của mình chạy trong một shell. Ví dụ,
command của bạn có thể gồm nhiều lệnh nối với nhau qua pipe, hoặc nó có thể là một
shell script. Để chạy command trong một shell, hãy bọc nó lại như sau:

```shell
command: ["/bin/sh"]
args: ["-c", "while true; do echo hello; sleep 10;done"]
```

## Tiếp theo (What's next)

* Tìm hiểu thêm về [cấu hình Pod và container](367-tasks-index-vi.md).
* Tìm hiểu thêm về [chạy lệnh trong container](304-get-shell-running-container-vi.md).
* Xem [Container](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#container-v1-core).

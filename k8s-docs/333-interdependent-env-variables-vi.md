# Định nghĩa các biến môi trường phụ thuộc (Define Dependent Environment Variables)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/inject-data-application/define-interdependent-environment-variables/

Trang này chỉ cách định nghĩa các biến môi trường phụ thuộc (dependent environment variable)
cho một container trong một Pod của Kubernetes.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Định nghĩa một biến môi trường phụ thuộc cho một container (Define an environment dependent variable for a container)

Khi bạn tạo một Pod, bạn có thể thiết lập các biến môi trường phụ thuộc cho các container chạy
trong Pod đó. Để thiết lập biến môi trường phụ thuộc, bạn có thể dùng cú pháp $(VAR_NAME)
trong `value` của `env` trong file cấu hình.

Trong bài thực hành này, bạn tạo một Pod chạy một container. File cấu hình của Pod định nghĩa
một biến môi trường phụ thuộc theo cách dùng phổ biến. Dưới đây là manifest cấu hình của Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dependent-envars-demo
spec:
  containers:
    - name: dependent-envars-demo
      args:
        - while true; do echo -en '\n'; printf UNCHANGED_REFERENCE=$UNCHANGED_REFERENCE'\n'; printf SERVICE_ADDRESS=$SERVICE_ADDRESS'\n';printf ESCAPED_REFERENCE=$ESCAPED_REFERENCE'\n'; sleep 30; done;
      command:
        - sh
        - -c
      image: busybox:1.28
      env:
        - name: SERVICE_PORT
          value: "80"
        - name: SERVICE_IP
          value: "172.17.0.1"
        - name: UNCHANGED_REFERENCE
          value: "$(PROTOCOL)://$(SERVICE_IP):$(SERVICE_PORT)"
        - name: PROTOCOL
          value: "https"
        - name: SERVICE_ADDRESS
          value: "$(PROTOCOL)://$(SERVICE_IP):$(SERVICE_PORT)"
        - name: ESCAPED_REFERENCE
          value: "$$(PROTOCOL)://$(SERVICE_IP):$(SERVICE_PORT)"
```

1. Tạo một Pod dựa trên manifest đó:

   ```shell
   kubectl apply -f https://k8s.io/examples/pods/inject/dependent-envars.yaml
   ```
   ```
   pod/dependent-envars-demo created
   ```

2. Liệt kê các Pod đang chạy:

   ```shell
   kubectl get pods dependent-envars-demo
   ```
   ```
   NAME                      READY     STATUS    RESTARTS   AGE
   dependent-envars-demo     1/1       Running   0          9s
   ```

3. Kiểm tra log của container đang chạy trong Pod của bạn:

   ```shell
   kubectl logs pod/dependent-envars-demo
   ```
   ```

   UNCHANGED_REFERENCE=$(PROTOCOL)://172.17.0.1:80
   SERVICE_ADDRESS=https://172.17.0.1:80
   ESCAPED_REFERENCE=$(PROTOCOL)://172.17.0.1:80
   ```

Như trên, bạn đã định nghĩa một tham chiếu phụ thuộc đúng ở `SERVICE_ADDRESS`, một tham chiếu
phụ thuộc sai ở `UNCHANGED_REFERENCE`, và bỏ qua tham chiếu phụ thuộc ở `ESCAPED_REFERENCE`.

Khi một biến môi trường đã được định nghĩa từ trước lúc nó được tham chiếu, tham chiếu đó sẽ
được phân giải (resolve) đúng, như trong trường hợp `SERVICE_ADDRESS`.

Lưu ý rằng thứ tự trong danh sách `env` rất quan trọng. Một biến môi trường không được coi là
"đã định nghĩa" nếu nó nằm phía dưới trong danh sách. Đó là lý do `UNCHANGED_REFERENCE` không
phân giải được `$(PROTOCOL)` trong ví dụ trên.

Khi biến môi trường chưa được định nghĩa hoặc chỉ có một phần các biến được định nghĩa, biến
môi trường chưa định nghĩa sẽ được xử lý như một chuỗi bình thường, như trường hợp
`UNCHANGED_REFERENCE`. Lưu ý rằng nói chung, các biến môi trường được phân tích cú pháp sai sẽ
không ngăn container khởi động.

Cú pháp `$(VAR_NAME)` có thể được thoát (escape) bằng hai dấu `$`, tức là: `$$(VAR_NAME)`.
Tham chiếu đã thoát không bao giờ được khai triển (expand), bất kể biến được tham chiếu có
được định nghĩa hay không. Điều này thể hiện qua trường hợp `ESCAPED_REFERENCE` ở trên.

## Tiếp theo (What's next)

* Tìm hiểu thêm về [biến môi trường](336-env-variable-expose-pod-info-vi.md).
* Xem [EnvVarSource](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#envvarsource-v1-core).

# Định nghĩa biến môi trường cho một Container (Define Environment Variables for a Container)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/

Trang này chỉ cách định nghĩa các biến môi trường (environment variable) cho một container
trong một Pod của Kubernetes.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Định nghĩa một biến môi trường cho một container (Define an environment variable for a container)

Khi bạn tạo một Pod, bạn có thể thiết lập các biến môi trường cho các container chạy trong
Pod đó. Để thiết lập biến môi trường, hãy đưa field `env` hoặc `envFrom` vào file cấu hình.

Hai field `env` và `envFrom` có tác dụng khác nhau.

`env`
: cho phép bạn thiết lập các biến môi trường cho một container, chỉ định trực tiếp giá trị cho từng biến mà bạn đặt tên.

`envFrom`
: cho phép bạn thiết lập các biến môi trường cho một container bằng cách tham chiếu tới một ConfigMap hoặc một Secret.
 Khi bạn dùng `envFrom`, tất cả các cặp key-value trong ConfigMap hoặc Secret được tham chiếu
 sẽ được thiết lập thành biến môi trường cho container.
 Bạn cũng có thể chỉ định một chuỗi tiền tố (prefix) chung.

Bạn có thể đọc thêm về [ConfigMap](275-configure-pod-configmap-vi.md#configure-all-key-value-pairs-in-a-configmap-as-container-environment-variables)
và [Secret](334-distribute-credentials-secure-vi.md#configure-all-key-value-pairs-in-a-secret-as-container-environment-variables).

Trang này giải thích cách dùng `env`.

Trong bài thực hành này, bạn tạo một Pod chạy một container. File cấu hình của Pod định nghĩa
một biến môi trường có tên `DEMO_GREETING` với giá trị `"Hello from the environment"`. Dưới đây
là manifest cấu hình của Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: envar-demo
  labels:
    purpose: demonstrate-envars
spec:
  containers:
  - name: envar-demo-container
    image: gcr.io/google-samples/hello-app:2.0
    env:
    - name: DEMO_GREETING
      value: "Hello from the environment"
    - name: DEMO_FAREWELL
      value: "Such a sweet sorrow"
```

1. Tạo một Pod dựa trên manifest đó:

   ```shell
   kubectl apply -f https://k8s.io/examples/pods/inject/envars.yaml
   ```

1. Liệt kê các Pod đang chạy:

   ```shell
   kubectl get pods -l purpose=demonstrate-envars
   ```

   Kết quả tương tự như sau:

   ```
   NAME            READY     STATUS    RESTARTS   AGE
   envar-demo      1/1       Running   0          9s
   ```

1. Liệt kê các biến môi trường của container trong Pod:

   ```shell
   kubectl exec envar-demo -- printenv
   ```

   Kết quả tương tự như sau:

   ```
   NODE_VERSION=4.4.2
   EXAMPLE_SERVICE_PORT_8080_TCP_ADDR=10.3.245.237
   HOSTNAME=envar-demo
   ...
   DEMO_GREETING=Hello from the environment
   DEMO_FAREWELL=Such a sweet sorrow
   ```

> **Ghi chú:** Các biến môi trường được thiết lập bằng field `env` hoặc `envFrom`
> sẽ ghi đè mọi biến môi trường được chỉ định trong container image.

> **Ghi chú:** Các biến môi trường có thể tham chiếu lẫn nhau, tuy nhiên thứ tự rất quan trọng.
> Biến nào sử dụng các biến khác được định nghĩa trong cùng ngữ cảnh thì phải đứng sau trong
> danh sách. Tương tự, hãy tránh tham chiếu vòng (circular reference).

## Sử dụng biến môi trường bên trong cấu hình của bạn (Using environment variables inside of your config) {#using-environment-variables-inside-of-your-config}

Các biến môi trường mà bạn định nghĩa trong cấu hình của một Pod tại
`.spec.containers[*].env[*]` có thể được dùng ở những chỗ khác trong cấu hình, ví dụ trong
các command và argument mà bạn thiết lập cho các container của Pod.
Trong cấu hình ví dụ dưới đây, các biến môi trường `GREETING`, `HONORIFIC` và `NAME` lần lượt
được gán các giá trị `Warm greetings to`, `The Most Honorable` và `Kubernetes`. Biến môi trường
`MESSAGE` gộp tất cả các biến môi trường này lại rồi dùng nó làm argument CLI truyền vào
container `env-print-demo`.

Tên biến môi trường có thể gồm bất kỳ [ký tự ASCII in được (printable ASCII characters)](https://www.ascii-code.com/characters/printable-characters) nào ngoại trừ '='.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: print-greeting
spec:
  containers:
  - name: env-print-demo
    image: bash
    env:
    - name: GREETING
      value: "Warm greetings to"
    - name: HONORIFIC
      value: "The Most Honorable"
    - name: NAME
      value: "Kubernetes"
    - name: MESSAGE
      value: "$(GREETING) $(HONORIFIC) $(NAME)"
    command: ["echo"]
    args: ["$(MESSAGE)"]
```

Khi Pod được tạo, lệnh `echo Warm greetings to The Most Honorable Kubernetes` sẽ được chạy trong container.

## Tiếp theo (What's next)

* Tìm hiểu thêm về [biến môi trường](336-env-variable-expose-pod-info-vi.md).
* Tìm hiểu về [sử dụng Secret làm biến môi trường](109-secret-vi.md#using-secrets-as-environment-variables).
* Xem [EnvVarSource](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#envvarsource-v1-core).

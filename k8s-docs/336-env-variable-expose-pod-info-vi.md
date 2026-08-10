# Expose thông tin Pod cho container thông qua biến môi trường (Expose Pod Information to Containers Through Environment Variables)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/>
>
> Trang này hướng dẫn cách một Pod có thể dùng biến môi trường để expose thông tin về chính nó cho các container đang chạy trong Pod, thông qua downward API.

Trang này hướng dẫn cách một Pod có thể dùng biến môi trường để expose thông tin
về chính nó cho các container đang chạy trong Pod, bằng cách sử dụng _downward API_.
Bạn có thể dùng biến môi trường để expose các field của Pod, các field của container, hoặc cả hai.

Trong Kubernetes, có hai cách để expose các field của Pod và container cho một container đang chạy:

* _Biến môi trường_, như được trình bày trong tác vụ này
* [File trong volume](335-downward-api-volume-vi.md)

Gộp chung lại, hai cách expose các field của Pod và container này được gọi là
downward API.

Vì Service là phương thức giao tiếp chính giữa các ứng dụng chạy trong container do Kubernetes quản lý,
việc có thể khám phá (discover) chúng lúc chạy (runtime) là rất hữu ích.

Đọc thêm về cách truy cập Service [tại đây](https://kubernetes.io/docs/tutorials/services/connect-applications-service/#accessing-the-service).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Dùng field của Pod làm giá trị cho biến môi trường (Use Pod fields as values for environment variables)

Trong phần này của bài thực hành, bạn tạo một Pod có một container, và bạn
chiếu (project) các field ở cấp Pod vào container đang chạy dưới dạng các biến môi trường.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-envars-fieldref
spec:
  containers:
    - name: test-container
      image: registry.k8s.io/busybox:1.27.2
      command: [ "sh", "-c"]
      args:
      - while true; do
          echo -en '\n';
          printenv MY_NODE_NAME MY_POD_NAME MY_POD_NAMESPACE;
          printenv MY_POD_IP MY_POD_SERVICE_ACCOUNT;
          sleep 10;
        done;
      env:
        - name: MY_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: MY_POD_SERVICE_ACCOUNT
          valueFrom:
            fieldRef:
              fieldPath: spec.serviceAccountName
  restartPolicy: Never
```

Trong manifest đó, bạn có thể thấy năm biến môi trường. Field `env`
là một mảng các định nghĩa biến môi trường.
Phần tử đầu tiên trong mảng chỉ định rằng biến môi trường `MY_NODE_NAME`
lấy giá trị từ field `spec.nodeName` của Pod. Tương tự, các
biến môi trường còn lại lấy giá trị từ các field của Pod.

> **Ghi chú:**
> Các field trong ví dụ này là các field của Pod. Chúng không phải là field của
> container trong Pod.

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/inject/dapi-envars-pod.yaml
```

Xác minh rằng container trong Pod đang chạy:

```shell
# Nếu Pod mới chưa healthy, hãy chạy lại lệnh này vài lần.
kubectl get pods
```

Xem log của container:

```shell
kubectl logs dapi-envars-fieldref
```

Kết quả hiển thị giá trị của các biến môi trường được chọn:

```
minikube
dapi-envars-fieldref
default
172.17.0.4
default
```

Để hiểu vì sao các giá trị này xuất hiện trong log, hãy xem các field `command` và `args`
trong file cấu hình. Khi container khởi động, nó ghi giá trị của
năm biến môi trường ra stdout. Nó lặp lại việc này mỗi mười giây.

Tiếp theo, mở một shell vào container đang chạy trong Pod của bạn:

```shell
kubectl exec -it dapi-envars-fieldref -- sh
```

Trong shell của bạn, xem các biến môi trường:

```shell
# Chạy lệnh này trong shell bên trong container
printenv
```

Kết quả cho thấy một số biến môi trường đã được gán
giá trị của các field của Pod:

```
MY_POD_SERVICE_ACCOUNT=default
...
MY_POD_NAMESPACE=default
MY_POD_IP=172.17.0.4
...
MY_NODE_NAME=minikube
...
MY_POD_NAME=dapi-envars-fieldref
```

## Dùng field của container làm giá trị cho biến môi trường (Use container fields as values for environment variables)

Ở bài thực hành trước, bạn đã dùng thông tin từ các field ở cấp Pod làm giá trị
cho biến môi trường.
Trong bài thực hành tiếp theo này, bạn sẽ truyền các field vốn là một phần của định nghĩa
Pod, nhưng được lấy từ một
[container](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Container)
cụ thể thay vì từ Pod nói chung.

Dưới đây là manifest cho một Pod khác cũng chỉ có một container:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-envars-resourcefieldref
spec:
  containers:
    - name: test-container
      image: registry.k8s.io/busybox:1.27.2
      command: [ "sh", "-c"]
      args:
      - while true; do
          echo -en '\n';
          printenv MY_CPU_REQUEST MY_CPU_LIMIT;
          printenv MY_MEM_REQUEST MY_MEM_LIMIT;
          sleep 10;
        done;
      resources:
        requests:
          memory: "32Mi"
          cpu: "125m"
        limits:
          memory: "64Mi"
          cpu: "250m"
      env:
        - name: MY_CPU_REQUEST
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              resource: requests.cpu
        - name: MY_CPU_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              resource: limits.cpu
        - name: MY_MEM_REQUEST
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              resource: requests.memory
        - name: MY_MEM_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              resource: limits.memory
  restartPolicy: Never
```

Trong manifest này, bạn có thể thấy bốn biến môi trường. Field `env`
là một mảng các định nghĩa biến môi trường.
Phần tử đầu tiên trong mảng chỉ định rằng biến môi trường `MY_CPU_REQUEST`
lấy giá trị từ field `requests.cpu` của container có tên
`test-container`. Tương tự, các biến môi trường còn lại lấy giá trị
từ các field riêng của container này.

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/inject/dapi-envars-container.yaml
```

Xác minh rằng container trong Pod đang chạy:

```shell
# Nếu Pod mới chưa healthy, hãy chạy lại lệnh này vài lần.
kubectl get pods
```

Xem log của container:

```shell
kubectl logs dapi-envars-resourcefieldref
```

Kết quả hiển thị giá trị của các biến môi trường được chọn:

```
1
1
33554432
67108864
```

## Tiếp theo (What's next)

* Đọc [Định nghĩa biến môi trường cho container](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/)
* Đọc định nghĩa API [`spec`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#PodSpec)
  của Pod. Định nghĩa này bao gồm cả định nghĩa của Container (một phần của Pod).
* Đọc danh sách [các field khả dụng](https://kubernetes.io/docs/concepts/workloads/pods/downward-api/#available-fields) mà bạn
  có thể expose bằng downward API.

Đọc về Pod, container và biến môi trường trong tài liệu tham khảo API cũ (legacy):

* [PodSpec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#podspec-v1-core)
* [Container](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#container-v1-core)
* [EnvVar](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#envvar-v1-core)
* [EnvVarSource](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#envvarsource-v1-core)
* [ObjectFieldSelector](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#objectfieldselector-v1-core)
* [ResourceFieldSelector](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#resourcefieldselector-v1-core)

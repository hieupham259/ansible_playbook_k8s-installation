# Expose thông tin Pod cho container thông qua file (Expose Pod Information to Containers Through Files)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/>
>
> Trang này hướng dẫn cách một Pod có thể dùng volume `downwardAPI` để expose thông tin về chính nó cho các container đang chạy trong Pod.

Trang này hướng dẫn cách một Pod có thể dùng
[volume `downwardAPI`](91-volumes-vi.md#downwardapi)
để expose thông tin về chính nó cho các container đang chạy trong Pod.
Một volume `downwardAPI` có thể expose các field của Pod và các field của container.

Trong Kubernetes, có hai cách để expose các field của Pod và container cho một container đang chạy:

* [Biến môi trường](336-env-variable-expose-pod-info-vi.md)
* File trong volume, như được trình bày trong tác vụ này

Gộp chung lại, hai cách expose các field của Pod và container này được gọi là
_downward API_.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Lưu trữ các field của Pod (Store Pod fields)

Trong phần này của bài thực hành, bạn tạo một Pod có một container, và bạn
chiếu (project) các field ở cấp Pod vào container đang chạy dưới dạng các file.
Dưới đây là manifest cho Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubernetes-downwardapi-volume-example
  labels:
    zone: us-est-coast
    cluster: test-cluster1
    rack: rack-22
  annotations:
    build: two
    builder: john-doe
spec:
  containers:
    - name: client-container
      image: registry.k8s.io/busybox:1.27.2
      command: ["sh", "-c"]
      args:
      - while true; do
          if [[ -e /etc/podinfo/labels ]]; then
            echo -en '\n\n'; cat /etc/podinfo/labels; fi;
          if [[ -e /etc/podinfo/annotations ]]; then
            echo -en '\n\n'; cat /etc/podinfo/annotations; fi;
          sleep 5;
        done;
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
  volumes:
    - name: podinfo
      downwardAPI:
        items:
          - path: "labels"
            fieldRef:
              fieldPath: metadata.labels
          - path: "annotations"
            fieldRef:
              fieldPath: metadata.annotations
```

Trong manifest, bạn có thể thấy Pod có một Volume `downwardAPI`,
và container mount volume đó tại `/etc/podinfo`.

Hãy nhìn vào mảng `items` bên dưới `downwardAPI`. Mỗi phần tử của mảng
định nghĩa một volume `downwardAPI`.
Phần tử đầu tiên chỉ định rằng giá trị của field
`metadata.labels` của Pod sẽ được lưu vào một file có tên `labels`.
Phần tử thứ hai chỉ định rằng giá trị của field `annotations`
của Pod sẽ được lưu vào một file có tên `annotations`.

> **Ghi chú:**
> Các field trong ví dụ này là các field của Pod. Chúng không phải là
> field của container trong Pod.

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/inject/dapi-volume.yaml
```

Xác minh rằng container trong Pod đang chạy:

```shell
kubectl get pods
```

Xem log của container:

```shell
kubectl logs kubernetes-downwardapi-volume-example
```

Kết quả hiển thị nội dung của file `labels` và file `annotations`:

```
cluster="test-cluster1"
rack="rack-22"
zone="us-est-coast"

build="two"
builder="john-doe"
```

Mở một shell vào container đang chạy trong Pod của bạn:

```shell
kubectl exec -it kubernetes-downwardapi-volume-example -- sh
```

Trong shell của bạn, xem file `labels`:

```shell
/# cat /etc/podinfo/labels
```

Kết quả cho thấy tất cả các label của Pod đã được ghi
vào file `labels`:

```shell
cluster="test-cluster1"
rack="rack-22"
zone="us-est-coast"
```

Tương tự, xem file `annotations`:

```shell
/# cat /etc/podinfo/annotations
```

Xem các file trong thư mục `/etc/podinfo`:

```shell
/# ls -laR /etc/podinfo
```

Trong kết quả, bạn có thể thấy các file `labels` và `annotations`
nằm trong một thư mục con tạm thời: trong ví dụ này là
`..2982_06_02_21_47_53.299460680`. Trong thư mục `/etc/podinfo`, `..data` là
một symbolic link trỏ tới thư mục con tạm thời đó. Cũng trong thư mục `/etc/podinfo`,
`labels` và `annotations` là các symbolic link.

```
drwxr-xr-x  ... Feb 6 21:47 ..2982_06_02_21_47_53.299460680
lrwxrwxrwx  ... Feb 6 21:47 ..data -> ..2982_06_02_21_47_53.299460680
lrwxrwxrwx  ... Feb 6 21:47 annotations -> ..data/annotations
lrwxrwxrwx  ... Feb 6 21:47 labels -> ..data/labels

/etc/..2982_06_02_21_47_53.299460680:
total 8
-rw-r--r--  ... Feb  6 21:47 annotations
-rw-r--r--  ... Feb  6 21:47 labels
```

Việc dùng symbolic link cho phép làm mới (refresh) metadata một cách động và nguyên tử (atomic); các bản cập nhật được
ghi vào một thư mục tạm mới, và symlink `..data` được cập nhật
một cách nguyên tử bằng [rename(2)](http://man7.org/linux/man-pages/man2/rename.2.html).

> **Ghi chú:**
> Một container dùng Downward API dưới dạng volume mount kiểu
> [subPath](91-volumes-vi.md#using-subpath) sẽ không
> nhận được các bản cập nhật của Downward API.

Thoát khỏi shell:

```shell
/# exit
```

## Lưu trữ các field của container (Store container fields)

Ở bài thực hành trước, bạn đã làm cho các field ở cấp Pod truy cập được bằng
downward API.
Trong bài thực hành tiếp theo này, bạn sẽ truyền các field vốn là một phần của định nghĩa
Pod, nhưng được lấy từ một
[container](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#Container)
cụ thể thay vì từ Pod nói chung. Dưới đây là manifest cho một Pod cũng chỉ có
một container:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubernetes-downwardapi-volume-example-2
spec:
  containers:
    - name: client-container
      image: registry.k8s.io/busybox:1.27.2
      command: ["sh", "-c"]
      args:
      - while true; do
          echo -en '\n';
          if [[ -e /etc/podinfo/cpu_limit ]]; then
            echo -en '\n'; cat /etc/podinfo/cpu_limit; fi;
          if [[ -e /etc/podinfo/cpu_request ]]; then
            echo -en '\n'; cat /etc/podinfo/cpu_request; fi;
          if [[ -e /etc/podinfo/mem_limit ]]; then
            echo -en '\n'; cat /etc/podinfo/mem_limit; fi;
          if [[ -e /etc/podinfo/mem_request ]]; then
            echo -en '\n'; cat /etc/podinfo/mem_request; fi;
          sleep 5;
        done;
      resources:
        requests:
          memory: "32Mi"
          cpu: "125m"
        limits:
          memory: "64Mi"
          cpu: "250m"
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
  volumes:
    - name: podinfo
      downwardAPI:
        items:
          - path: "cpu_limit"
            resourceFieldRef:
              containerName: client-container
              resource: limits.cpu
              divisor: 1m
          - path: "cpu_request"
            resourceFieldRef:
              containerName: client-container
              resource: requests.cpu
              divisor: 1m
          - path: "mem_limit"
            resourceFieldRef:
              containerName: client-container
              resource: limits.memory
              divisor: 1Mi
          - path: "mem_request"
            resourceFieldRef:
              containerName: client-container
              resource: requests.memory
              divisor: 1Mi
```

Trong manifest, bạn có thể thấy Pod có một
[volume `downwardAPI`](91-volumes-vi.md#downwardapi),
và container duy nhất trong Pod đó mount volume này tại `/etc/podinfo`.

Hãy nhìn vào mảng `items` bên dưới `downwardAPI`. Mỗi phần tử của mảng
định nghĩa một file trong downward API volume.

Phần tử đầu tiên chỉ định rằng trong container có tên `client-container`,
giá trị của field `limits.cpu` theo định dạng được chỉ định bởi `1m` sẽ được
xuất ra dưới dạng một file có tên `cpu_limit`. Field `divisor` là tùy chọn và có
giá trị mặc định là `1`. Divisor bằng 1 nghĩa là đơn vị core cho tài nguyên `cpu`, hoặc
byte cho tài nguyên `memory`.

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/inject/dapi-volume-resources.yaml
```

Mở một shell vào container đang chạy trong Pod của bạn:

```shell
kubectl exec -it kubernetes-downwardapi-volume-example-2 -- sh
```

Trong shell của bạn, xem file `cpu_limit`:

```shell
# Chạy lệnh này trong shell bên trong container
cat /etc/podinfo/cpu_limit
```

Bạn có thể dùng các lệnh tương tự để xem các file `cpu_request`, `mem_limit` và
`mem_request`.

## Chiếu key vào những đường dẫn và quyền file cụ thể (Project keys to specific paths and file permissions)

Bạn có thể chiếu (project) các key vào những đường dẫn cụ thể và với những quyền cụ thể trên cơ sở
từng file. Để biết thêm thông tin, xem
[Secret](109-secret-vi.md).

## Tiếp theo (What's next)

* Đọc định nghĩa API [`spec`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#PodSpec)
  của Pod. Định nghĩa này bao gồm cả định nghĩa của Container (một phần của Pod).
* Đọc danh sách [các field khả dụng](56-downward-api-vi.md#available-fields) mà bạn
  có thể expose bằng downward API.

Đọc về volume trong tài liệu tham khảo API cũ (legacy):
* Xem định nghĩa API [`Volume`](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#volume-v1-core),
  định nghĩa một volume tổng quát trong Pod để các container truy cập.
* Xem định nghĩa API [`DownwardAPIVolumeSource`](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#downwardapivolumesource-v1-core),
  định nghĩa một volume chứa thông tin Downward API.
* Xem định nghĩa API [`DownwardAPIVolumeFile`](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#downwardapivolumefile-v1-core),
  chứa các tham chiếu tới field của đối tượng hoặc field tài nguyên để
  điền dữ liệu vào một file trong Downward API volume.
* Xem định nghĩa API [`ResourceFieldSelector`](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#resourcefieldselector-v1-core),
  chỉ định các tài nguyên của container và định dạng đầu ra của chúng.

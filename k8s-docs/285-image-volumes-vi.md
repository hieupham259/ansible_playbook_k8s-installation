# Sử dụng Image Volume với một Pod (Use an Image Volume With a Pod)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/image-volumes/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [stable]`

Trang này hướng dẫn cách cấu hình một pod sử dụng image volume. Tính năng này cho phép bạn
mount nội dung từ các OCI registry vào bên trong container.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản v1.31 hoặc mới hơn. Để kiểm tra phiên bản, hãy
nhập `kubectl version`.

- Container runtime cần hỗ trợ tính năng image volume
- Bạn cần có khả năng thực thi (exec) lệnh trên host
- Bạn cần có khả năng exec vào trong các pod

## Chạy một Pod sử dụng image volume (Run a Pod that uses an image volume) {#create-pod}

Image volume cho một pod được bật bằng cách đặt trường `volumes[*].image` của `.spec` thành
một tham chiếu (reference) hợp lệ và sử dụng nó trong `volumeMounts` của container. Ví dụ:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: image-volume
spec:
  containers:
  - name: shell
    command: ["sleep", "infinity"]
    image: debian
    volumeMounts:
    - name: volume
      mountPath: /volume
  volumes:
  - name: volume
    image:
      reference: quay.io/crio/artifact:v2
      pullPolicy: IfNotPresent
```

1. Tạo pod trên cluster của bạn:

   ```shell
   kubectl apply -f https://k8s.io/examples/pods/image-volumes.yaml
   ```

1. Attach vào container:

   ```shell
   kubectl exec image-volume -it -- bash
   ```

1. Kiểm tra nội dung của một file trong volume:

   ```shell
   cat /volume/dir/file
   ```

   Đầu ra tương tự như:

   ```none
   1
   ```

   Bạn cũng có thể kiểm tra một file khác ở một đường dẫn khác:

   ```shell
   cat /volume/file
   ```

   Đầu ra tương tự như:

   ```none
   2
   ```

## Sử dụng `subPath` (hoặc `subPathExpr`) (Use `subPath` (or `subPathExpr`))

Từ Kubernetes v1.33, bạn có thể sử dụng
[`subPath`](./91-volumes-vi.md#using-subpath) hoặc
[`subPathExpr`](./91-volumes-vi.md#using-subpath-expanded-environment)
khi dùng tính năng image volume.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: image-volume
spec:
  containers:
  - name: shell
    command: ["sleep", "infinity"]
    image: debian
    volumeMounts:
    - name: volume
      mountPath: /volume
      subPath: dir
  volumes:
  - name: volume
    image:
      reference: quay.io/crio/artifact:v2
      pullPolicy: IfNotPresent
```

1. Tạo pod trên cluster của bạn:

   ```shell
   kubectl apply -f https://k8s.io/examples/pods/image-volumes-subpath.yaml
   ```

1. Attach vào container:

   ```shell
   kubectl exec image-volume -it -- bash
   ```

1. Kiểm tra nội dung của file từ sub path `dir` trong volume:

   ```shell
   cat /volume/file
   ```

   Đầu ra tương tự như:

   ```none
   1
   ```

## Đọc thêm (Further reading)

- [Volume kiểu `image`](./91-volumes-vi.md#image)

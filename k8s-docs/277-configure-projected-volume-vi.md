# Cấu hình Pod sử dụng projected Volume cho lưu trữ (Configure a Pod to Use a Projected Volume for Storage)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/configure-projected-volume-storage/>

Trang này hướng dẫn cách sử dụng Volume kiểu
[`projected`](91-volumes-vi.md#projected) để mount nhiều
nguồn volume có sẵn vào cùng một thư mục. Hiện tại, các volume `secret`, `configMap`,
`downwardAPI` và `serviceAccountToken` có thể được chiếu (project).

> **Ghi chú:**
>
> `serviceAccountToken` không phải là một loại volume.

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

## Cấu hình một projected volume cho Pod (Configure a projected volume for a pod)

Trong bài thực hành này, bạn tạo các Secret chứa tên người dùng (username) và mật khẩu
(password) từ các file cục bộ. Sau đó bạn tạo một Pod chạy một container, sử dụng Volume kiểu
[`projected`](91-volumes-vi.md#projected) để mount các
Secret vào cùng một thư mục dùng chung.

Đây là file cấu hình của Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-projected-volume
spec:
  containers:
  - name: test-projected-volume
    image: busybox:1.28
    args:
    - sleep
    - "86400"
    volumeMounts:
    - name: all-in-one
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: user
      - secret:
          name: pass
```

1. Tạo các Secret:

    ```shell
    # Tạo các file chứa username và password:
    echo -n "admin" > ./username.txt
    echo -n "1f2d1e2e67df" > ./password.txt

    # Đóng gói các file này thành các secret:
    kubectl create secret generic user --from-file=./username.txt
    kubectl create secret generic pass --from-file=./password.txt
    ```

1. Tạo Pod:

    ```shell
    kubectl apply -f https://k8s.io/examples/pods/storage/projected.yaml
    ```

1. Xác nhận rằng container của Pod đang chạy, sau đó theo dõi (watch) các thay đổi của Pod:

    ```shell
    kubectl get --watch pod test-projected-volume
    ```

    Kết quả trông như sau:

    ```
    NAME                    READY     STATUS    RESTARTS   AGE
    test-projected-volume   1/1       Running   0          14s
    ```

1. Trong một terminal khác, mở một shell tới container đang chạy:

    ```shell
    kubectl exec -it test-projected-volume -- /bin/sh
    ```

1. Trong shell của bạn, xác nhận rằng thư mục `projected-volume` chứa các nguồn đã được chiếu:

    ```shell
    ls /projected-volume/
    ```

## Dọn dẹp (Clean up)

Xóa Pod và các Secret:

```shell
kubectl delete pod test-projected-volume
kubectl delete secret user pass
```

## Tiếp theo (What's next)

* Tìm hiểu thêm về volume
  [`projected`](91-volumes-vi.md#projected).
* Đọc tài liệu thiết kế
  [all-in-one volume](https://git.k8s.io/design-proposals-archive/node/all-in-one-volume.md).

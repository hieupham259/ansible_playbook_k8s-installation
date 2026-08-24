# Giao tiếp giữa các Container trong cùng Pod bằng Volume dùng chung (Communicate Between Containers in the Same Pod Using a Shared Volume)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/access-application-cluster/communicate-containers-same-pod-shared-volume/>
>
> Trang này hướng dẫn cách dùng một Volume để giao tiếp giữa hai Container chạy trong cùng một Pod.

Trang này hướng dẫn cách dùng một Volume để giao tiếp giữa hai Container chạy
trong cùng một Pod. Xem thêm cách cho phép các tiến trình giao tiếp bằng việc
[chia sẻ process namespace](292-share-process-namespace-vi.md)
giữa các container.

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

## Tạo một Pod chạy hai Container (Creating a Pod that runs two Containers)

Trong bài thực hành này, bạn tạo một Pod chạy hai Container. Hai container
chia sẻ một Volume mà chúng có thể dùng để giao tiếp với nhau. Đây là file cấu hình
cho Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:

  restartPolicy: Never

  volumes:
  - name: shared-data
    emptyDir: {}

  containers:

  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html

  - name: debian-container
    image: debian
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]
```

Trong file cấu hình, bạn có thể thấy Pod có một Volume tên là
`shared-data`.

Container đầu tiên được liệt kê trong file cấu hình chạy một nginx server.
Đường dẫn mount (mount path) cho Volume dùng chung là `/usr/share/nginx/html`.
Container thứ hai dựa trên image debian, và có đường dẫn mount là
`/pod-data`. Container thứ hai chạy lệnh sau rồi kết thúc.

    echo Hello from the debian container > /pod-data/index.html

Lưu ý rằng container thứ hai ghi file `index.html` vào thư mục gốc
của nginx server.

Tạo Pod cùng hai Container:

    kubectl apply -f https://k8s.io/examples/pods/two-container-pod.yaml

Xem thông tin về Pod và các Container:

    kubectl get pod two-containers --output=yaml

Đây là một phần của kết quả:

    apiVersion: v1
    kind: Pod
    metadata:
      ...
      name: two-containers
      namespace: default
      ...
    spec:
      ...
      containerStatuses:

      - containerID: docker://c1d8abd1 ...
        image: debian
        ...
        lastState:
          terminated:
            ...
        name: debian-container
        ...

      - containerID: docker://96c1ff2c5bb ...
        image: nginx
        ...
        name: nginx-container
        ...
        state:
          running:
        ...

Bạn có thể thấy Container debian đã kết thúc (terminated), còn Container nginx
vẫn đang chạy.

Mở một shell vào Container nginx:

    kubectl exec -it two-containers -c nginx-container -- /bin/bash

Trong shell của bạn, xác minh rằng nginx đang chạy:

    root@two-containers:/# apt-get update
    root@two-containers:/# apt-get install curl procps
    root@two-containers:/# ps aux

Kết quả tương tự như sau:

    USER       PID  ...  STAT START   TIME COMMAND
    root         1  ...  Ss   21:12   0:00 nginx: master process nginx -g daemon off;

Hãy nhớ rằng Container debian đã tạo file `index.html` trong thư mục gốc
của nginx. Dùng `curl` để gửi một request GET tới nginx server:

```
root@two-containers:/# curl localhost
```

Kết quả cho thấy nginx phục vụ trang web do container debian viết ra:

```
Hello from the debian container
```

## Thảo luận (Discussion)

Lý do chính khiến Pod có thể có nhiều container là để hỗ trợ
các ứng dụng phụ trợ (helper application) trợ giúp cho một ứng dụng chính. Các ví dụ điển hình của
ứng dụng phụ trợ là bộ kéo dữ liệu (data puller), bộ đẩy dữ liệu (data pusher), và proxy.
Ứng dụng phụ trợ và ứng dụng chính thường cần giao tiếp với nhau.
Thông thường, việc này được thực hiện qua một hệ thống file dùng chung, như minh họa
trong bài thực hành này, hoặc qua giao diện mạng loopback, localhost. Một ví dụ của mẫu hình này là
một web server đi cùng một chương trình phụ trợ định kỳ kiểm tra một Git repository để lấy các cập nhật mới.

Volume trong bài thực hành này cung cấp một cách để các Container giao tiếp trong
suốt vòng đời của Pod. Nếu Pod bị xóa và được tạo lại, mọi dữ liệu lưu trong
Volume dùng chung sẽ bị mất.

## Tiếp theo (What's next)

* Tìm hiểu thêm về [các mẫu hình cho composite container](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns/).

* Tìm hiểu về [composite container cho kiến trúc dạng module](https://www.slideshare.net/Docker/slideshare-burns).

* Xem [Cấu hình một Pod dùng Volume để lưu trữ](280-configure-volume-storage-vi.md).

* Xem [Cấu hình một Pod chia sẻ process namespace giữa các container trong Pod](292-share-process-namespace-vi.md)

* Xem [Volume](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#volume-v1-core).

* Xem [Pod](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#pod-v1-core).

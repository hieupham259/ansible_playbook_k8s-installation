# Cấu hình khởi tạo Pod (Configure Pod Initialization)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-initialization/>

Trang này hướng dẫn cách sử dụng một Init Container để khởi tạo một Pod trước khi container
ứng dụng chạy.

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

## Tạo một Pod có Init Container (Create a Pod that has an Init Container) {#create-a-pod-that-has-an-init-container}

Trong bài thực hành này, bạn tạo một Pod có một container ứng dụng và một Init Container.
Init container chạy đến khi hoàn tất trước khi container ứng dụng khởi động.

Đây là file cấu hình của Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: workdir
      mountPath: /usr/share/nginx/html
  # Các container này được chạy trong quá trình khởi tạo Pod
  initContainers:
  - name: install
    image: busybox:1.28
    command:
    - wget
    - "-O"
    - "/work-dir/index.html"
    - http://info.cern.ch
    volumeMounts:
    - name: workdir
      mountPath: "/work-dir"
  dnsPolicy: Default
  volumes:
  - name: workdir
    emptyDir: {}
```

Trong file cấu hình, bạn có thể thấy Pod có một Volume mà init container và container ứng dụng
cùng chia sẻ.

Init container mount Volume dùng chung này tại `/work-dir`, còn container ứng dụng mount Volume
dùng chung này tại `/usr/share/nginx/html`. Init container chạy lệnh sau rồi kết thúc:

```shell
wget -O /work-dir/index.html http://info.cern.ch
```

Lưu ý rằng init container ghi file `index.html` vào thư mục gốc của nginx server.

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/init-containers.yaml
```

Xác minh rằng container nginx đang chạy:

```shell
kubectl get pod init-demo
```

Kết quả cho thấy container nginx đang chạy:

```
NAME        READY     STATUS    RESTARTS   AGE
init-demo   1/1       Running   0          1m
```

Mở một shell vào container nginx đang chạy trong Pod init-demo:

```shell
kubectl exec -it init-demo -- /bin/bash
```

Trong shell của bạn, gửi một GET request đến nginx server:

```
root@nginx:~# apt-get update
root@nginx:~# apt-get install curl
root@nginx:~# curl localhost
```

Kết quả cho thấy nginx đang phục vụ trang web đã được init container ghi vào:

```html
<html><head></head><body><header>
<title>http://info.cern.ch</title>
</header>

<h1>http://info.cern.ch - home of the first website</h1>
  ...
  <li><a href="http://info.cern.ch/hypertext/WWW/TheProject.html">Browse the first website</a></li>
  ...
```

## Tiếp theo (What's next)

* Tìm hiểu thêm về
  [giao tiếp giữa các container chạy trong cùng một Pod](./360-containers-shared-volume-vi.md).
* Tìm hiểu thêm về [Init Container](./50-init-containers-vi.md).
* Tìm hiểu thêm về [Volume](./91-volumes-vi.md).
* Tìm hiểu thêm về [Gỡ lỗi Init Container](./298-debug-init-containers-vi.md)

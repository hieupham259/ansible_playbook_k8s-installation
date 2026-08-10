# Truy cập shell của một container đang chạy (Get a Shell to a Running Container)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/debug/debug-application/get-shell-running-container/>

Trang này hướng dẫn cách dùng `kubectl exec` để truy cập shell của một container đang chạy.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Truy cập shell của một container (Getting a shell to a container)

Trong bài thực hành này, bạn tạo một Pod có một container. Container chạy image nginx. Đây là
file cấu hình của Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shell-demo
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  hostNetwork: true
  dnsPolicy: Default
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/application/shell-demo.yaml
```

Xác nhận rằng container đang chạy:

```shell
kubectl get pod shell-demo
```

Truy cập shell của container đang chạy:

```shell
kubectl exec --stdin --tty shell-demo -- /bin/bash
```

> **Ghi chú:**
> Dấu gạch ngang kép (`--`) phân tách các đối số bạn muốn truyền cho lệnh khỏi các đối số của
> kubectl.

Trong shell của bạn, liệt kê thư mục gốc:

```shell
# Chạy lệnh này bên trong container
ls /
```

Trong shell của bạn, hãy thử nghiệm với các lệnh khác. Dưới đây là một vài ví dụ:

```shell
# Bạn có thể chạy các lệnh ví dụ này bên trong container
ls /
cat /proc/mounts
cat /proc/1/maps
apt-get update
apt-get install -y tcpdump
tcpdump
apt-get install -y lsof
lsof
apt-get install -y procps
ps aux
ps aux | grep nginx
```

## Ghi trang gốc cho nginx (Writing the root page for nginx)

Hãy xem lại file cấu hình của Pod. Pod có một volume `emptyDir`, và container mount volume này
tại `/usr/share/nginx/html`.

Trong shell của bạn, tạo một file `index.html` trong thư mục `/usr/share/nginx/html`:

```shell
# Chạy lệnh này bên trong container
echo 'Hello shell demo' > /usr/share/nginx/html/index.html
```

Trong shell của bạn, gửi một request GET đến nginx server:

```shell
# Chạy lệnh này trong shell bên trong container của bạn
apt-get update
apt-get install curl
curl http://localhost/
```

Kết quả đầu ra hiển thị đoạn văn bản mà bạn đã ghi vào file `index.html`:

```
Hello shell demo
```

Khi bạn dùng xong shell, hãy nhập `exit`.

```shell
exit # Để thoát khỏi shell trong container
```

## Chạy từng lệnh riêng lẻ trong container (Running individual commands in a container)

Trong một cửa sổ lệnh thông thường (không phải shell của bạn trong container), liệt kê các
biến môi trường trong container đang chạy:

```shell
kubectl exec shell-demo -- env
```

Hãy thử nghiệm chạy các lệnh khác. Dưới đây là một vài ví dụ:

```shell
kubectl exec shell-demo -- ps aux
kubectl exec shell-demo -- ls /
kubectl exec shell-demo -- cat /proc/1/mounts
```

## Mở shell khi Pod có nhiều hơn một container (Opening a shell when a Pod has more than one container)

Nếu một Pod có nhiều hơn một container, hãy dùng `--container` hoặc `-c` để chỉ định container
trong lệnh `kubectl exec`. Ví dụ, giả sử bạn có một Pod tên là my-pod, và Pod này có hai
container tên là _main-app_ và _helper-app_. Lệnh sau sẽ mở một shell tới container _main-app_.

```shell
kubectl exec -i -t my-pod --container main-app -- /bin/bash
```

> **Ghi chú:**
> Các tùy chọn ngắn `-i` và `-t` tương đương với các tùy chọn dài `--stdin` và `--tty`.

## Tiếp theo (What's next)

* Đọc về [kubectl exec](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#exec)

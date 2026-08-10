# Tạo static Pod (Create static Pods)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/>

Trang này hướng dẫn bạn cách tạo _static Pod_ trên một node. Để có cái nhìn tổng quan về static
Pod là gì và khi nào nên dùng chúng, xem [Static Pod](./58-static-pods-vi.md).

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

Trang này giả định rằng bạn đang dùng CRI-O để chạy Pod, và các node của bạn chạy hệ điều hành
Fedora. Hướng dẫn cho các bản phân phối khác hoặc các cách cài đặt Kubernetes khác có thể
khác đi.

## Tạo một static Pod (Create a static pod) {#static-pod-creation}

Bạn có thể cấu hình một static Pod bằng [file cấu hình đặt trên hệ thống file](#configuration-files)
hoặc [file cấu hình đặt trên web](#pods-created-via-http).

### Manifest static Pod đặt trên hệ thống file (Filesystem-hosted static Pod manifest) {#configuration-files}

Manifest là các định nghĩa Pod tiêu chuẩn ở định dạng JSON hoặc YAML nằm trong một thư mục cụ
thể. Hãy dùng trường `staticPodPath: <thư mục>` trong
[file cấu hình kubelet](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/);
kubelet sẽ định kỳ quét thư mục này và tạo/xóa static Pod khi các file YAML/JSON xuất hiện/biến
mất trong đó. Lưu ý rằng kubelet sẽ bỏ qua các file bắt đầu bằng dấu chấm khi quét thư mục
được chỉ định.

> **Thận trọng:** Kubelet xử lý **tất cả các file không bắt đầu bằng dấu chấm** trong thư mục
> static Pod — không hề có việc lọc theo phần mở rộng của file. Ví dụ, nếu bạn tạo một bản sao
> lưu (backup) của manifest bằng cách chạy `cp kube-apiserver.yaml kube-apiserver.yaml.backup`,
> kubelet sẽ đọc **cả hai** file và cố gắng tạo một static Pod từ mỗi file. Khi hai file định
> nghĩa một Pod có cùng tên, hành vi thu được là không xác định (undefined) và có thể khiến
> spec lỗi thời của bản backup âm thầm có hiệu lực thay cho manifest hiện tại. Nếu bạn có tạo
> bản backup, hãy lưu nó **bên ngoài** thư mục static Pod (ví dụ, trong
> `/etc/kubernetes/backup/`).

Ví dụ, đây là cách khởi động một web server đơn giản dưới dạng static Pod:

1. Chọn một node mà bạn muốn chạy static Pod. Trong ví dụ này, đó là `my-node1`.

   ```shell
   ssh my-node1
   ```

1. Chọn một thư mục, chẳng hạn `/etc/kubernetes/manifests`, và đặt một định nghĩa Pod web
   server vào đó, ví dụ `/etc/kubernetes/manifests/static-web.yaml`:

   ```shell
   # Chạy lệnh này trên node nơi kubelet đang chạy
   mkdir -p /etc/kubernetes/manifests/
   cat <<EOF >/etc/kubernetes/manifests/static-web.yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: static-web
     labels:
       role: myrole
   spec:
     containers:
       - name: web
         image: nginx
         ports:
           - name: web
             containerPort: 80
             protocol: TCP
   EOF
   ```

1. Cấu hình kubelet trên node đó để đặt giá trị `staticPodPath` trong
   [file cấu hình kubelet](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/).
   Xem [Thiết lập tham số kubelet qua file cấu hình](./224-kubelet-config-file-vi.md)
   để biết thêm thông tin.

   Một cách thay thế đã bị loại bỏ dần (deprecated) là cấu hình kubelet trên node đó để tìm
   manifest static Pod cục bộ bằng một đối số dòng lệnh. Để dùng cách tiếp cận đã deprecated
   này, hãy khởi động kubelet với đối số `--pod-manifest-path=/etc/kubernetes/manifests/`.

1. Khởi động lại kubelet. Trên Fedora, bạn sẽ chạy:

   ```shell
   # Chạy lệnh này trên node nơi kubelet đang chạy
   systemctl restart kubelet
   ```

### Manifest static Pod đặt trên web (Web-hosted static pod manifest) {#pods-created-via-http}

Kubelet định kỳ tải xuống file được chỉ định bởi đối số `--manifest-url=<URL>` và diễn giải nó
như một file JSON/YAML chứa các định nghĩa Pod. Tương tự cách
[manifest đặt trên hệ thống file](#configuration-files) hoạt động, kubelet tải lại manifest
theo lịch định kỳ. Nếu có thay đổi trong danh sách static Pod, kubelet sẽ áp dụng chúng.

Để dùng cách tiếp cận này:

1. Tạo một file YAML và lưu nó trên một web server sao cho bạn có thể truyền URL của file đó
   cho kubelet.

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: static-web
     labels:
       role: myrole
   spec:
     containers:
       - name: web
         image: nginx
         ports:
           - name: web
             containerPort: 80
             protocol: TCP
   ```

1. Cấu hình kubelet trên node bạn đã chọn để dùng manifest trên web này bằng cách cập nhật
   file cấu hình kubelet để bao gồm trường `staticPodURL`:

   ```yaml
   apiVersion: kubelet.config.k8s.io/v1beta1
   kind: KubeletConfiguration
   staticPodURL: "<manifest-url>"
   ```

1. Khởi động lại kubelet. Trên Fedora, bạn sẽ chạy:

   ```shell
   # Chạy lệnh này trên node nơi kubelet đang chạy
   systemctl restart kubelet
   ```

## Quan sát hành vi của static Pod (Observe static pod behavior) {#behavior-of-static-pods}

Khi kubelet khởi động, nó tự động khởi động tất cả các static Pod đã được định nghĩa. Vì bạn
đã định nghĩa một static Pod và khởi động lại kubelet, static Pod mới hẳn đã đang chạy rồi.

Bạn có thể xem các container đang chạy (bao gồm cả static Pod) bằng cách chạy (trên node):

```shell
# Chạy lệnh này trên node nơi kubelet đang chạy
crictl ps
```

Output có thể trông giống như sau:

```console
CONTAINER       IMAGE                                 CREATED           STATE      NAME    ATTEMPT    POD ID
129fd7d382018   docker.io/library/nginx@sha256:...    11 minutes ago    Running    web     0          34533c6729106
```

> **Ghi chú:** `crictl` in ra URI của image và checksum SHA-256. `NAME` sẽ trông giống như:
> `docker.io/library/nginx@sha256:0d17b565c37bcbd895e9d92315a05c1c3c9a29f762b011a10c54a66cd53c9b31`.

Bạn có thể thấy mirror Pod trên API server:

```shell
kubectl get pods
```
```console
NAME                  READY   STATUS    RESTARTS        AGE
static-web-my-node1   1/1     Running   0               2m
```

> **Ghi chú:** Hãy đảm bảo kubelet có quyền tạo mirror Pod trong API server. Nếu không, yêu
> cầu tạo sẽ bị API server từ chối.

Các Label từ static Pod được lan truyền (propagate) sang mirror Pod. Bạn có thể dùng các label
đó một cách bình thường thông qua các selector, v.v.

Nếu bạn thử dùng `kubectl` để xóa mirror Pod khỏi API server, kubelet sẽ _không_ xóa static
Pod:

```shell
kubectl delete pod static-web-my-node1
```
```console
pod "static-web-my-node1" deleted
```

Bạn có thể thấy rằng Pod vẫn đang chạy:

```shell
kubectl get pods
```
```console
NAME                  READY   STATUS    RESTARTS   AGE
static-web-my-node1   1/1     Running   0          4s
```

Quay lại node nơi kubelet đang chạy, bạn có thể thử dừng container theo cách thủ công. Bạn sẽ
thấy rằng, sau một khoảng thời gian, kubelet sẽ nhận ra và tự động khởi động lại Pod:

```shell
# Chạy các lệnh này trên node nơi kubelet đang chạy
crictl stop 129fd7d382018 # thay bằng ID container của bạn
sleep 20
crictl ps
```

```console
CONTAINER       IMAGE                                 CREATED           STATE      NAME    ATTEMPT    POD ID
89db4553e1eeb   docker.io/library/nginx@sha256:...    19 seconds ago    Running    web     1          34533c6729106
```

Khi bạn đã xác định được đúng container, bạn có thể lấy log của container đó bằng `crictl`:

```shell
# Chạy các lệnh này trên node nơi container đang chạy
crictl logs <container_id>
```

```console
10.240.0.48 - - [16/Nov/2022:12:45:49 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.47.0" "-"
10.240.0.48 - - [16/Nov/2022:12:45:50 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.47.0" "-"
10.240.0.48 - - [16/Nove/2022:12:45:51 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.47.0" "-"
```

Để tìm hiểu thêm về cách debug bằng `crictl`, hãy xem
[_Debug các node Kubernetes bằng crictl_](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/).

## Thêm và xóa static Pod một cách động (Dynamic addition and removal of static pods)

Kubelet đang chạy sẽ định kỳ quét thư mục đã cấu hình (`/etc/kubernetes/manifests` trong ví dụ
của chúng ta) để tìm các thay đổi, và thêm/xóa Pod khi các file xuất hiện/biến mất trong thư
mục này.

```shell
# Phần này giả định bạn đang dùng cấu hình static Pod đặt trên hệ thống file
# Chạy các lệnh này trên node nơi container đang chạy
mv /etc/kubernetes/manifests/static-web.yaml /tmp
sleep 20
crictl ps
# Bạn thấy rằng không có container nginx nào đang chạy
mv /tmp/static-web.yaml  /etc/kubernetes/manifests/
sleep 20
crictl ps
```
```console
CONTAINER       IMAGE                                 CREATED           STATE      NAME    ATTEMPT    POD ID
f427638871c35   docker.io/library/nginx@sha256:...    19 seconds ago    Running    web     1          34533c6729106
```

## Tiếp theo (What's next)

* [Static Pod](./58-static-pods-vi.md)
* [Sinh manifest static Pod cho các thành phần control plane](https://kubernetes.io/docs/reference/setup-tools/kubeadm/implementation-details/#generate-static-pod-manifests-for-control-plane-components)
* [Sinh manifest static Pod cho etcd cục bộ](https://kubernetes.io/docs/reference/setup-tools/kubeadm/implementation-details/#generate-static-pod-manifest-for-local-etcd)
* [Debug các node Kubernetes bằng `crictl`](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/)
* [Tìm hiểu thêm về `crictl`](https://github.com/kubernetes-sigs/cri-tools)
* [Thiết lập các instance etcd dưới dạng static Pod do kubelet quản lý](./07-setup-ha-etcd-with-kubeadm-vi.md)

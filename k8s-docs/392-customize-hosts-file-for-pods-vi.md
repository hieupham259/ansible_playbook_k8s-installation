# Thêm entry vào file /etc/hosts của Pod bằng HostAliases (Adding entries to Pod /etc/hosts with HostAliases)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/network/customize-hosts-file-for-pods/>
>
> Thêm entry vào file `/etc/hosts` của một Pod cho phép ghi đè việc phân giải hostname ở cấp Pod,
> dùng khi DNS và các phương án khác không áp dụng được.

Việc thêm entry vào file `/etc/hosts` của một Pod cho phép ghi đè việc phân giải hostname
(hostname resolution) ở cấp Pod, dùng cho trường hợp DNS và các lựa chọn khác không phù hợp.
Bạn có thể thêm các entry tùy chỉnh này bằng trường HostAliases trong PodSpec.

Dự án Kubernetes khuyến nghị bạn thay đổi cấu hình DNS thông qua trường `hostAliases`
(một phần của `.spec` của Pod), chứ không dùng init container hay cách nào khác để sửa trực tiếp
`/etc/hosts`.
Thay đổi thực hiện theo những cách khác có thể bị kubelet ghi đè trong quá trình tạo hoặc khởi
động lại Pod.

## Nội dung mặc định của file hosts (Default hosts file content)

Khởi chạy một Pod Nginx được cấp một Pod IP:

```shell
kubectl run nginx --image nginx
```

```
pod/nginx created
```

Kiểm tra Pod IP:

```shell
kubectl get pods --output=wide
```

```
NAME     READY     STATUS    RESTARTS   AGE    IP           NODE
nginx    1/1       Running   0          13s    10.200.0.4   worker0
```

Nội dung file hosts sẽ trông như sau:

```shell
kubectl exec nginx -- cat /etc/hosts
```

```
# Kubernetes-managed hosts file.
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
fe00::0	ip6-mcastprefix
fe00::1	ip6-allnodes
fe00::2	ip6-allrouters
10.200.0.4	nginx
```

Theo mặc định, file `hosts` chỉ chứa các dòng khuôn mẫu (boilerplate) cho IPv4 và IPv6 như
`localhost` cùng với hostname của chính Pod đó.

## Thêm entry bổ sung bằng hostAliases (Adding additional entries with hostAliases)

Ngoài phần khuôn mẫu mặc định, bạn có thể thêm các entry khác vào file `hosts`.
Ví dụ: để phân giải `foo.local`, `bar.local` thành `127.0.0.1` và `foo.remote`,
`bar.remote` thành `10.1.2.3`, bạn có thể cấu hình HostAliases cho một Pod tại
`.spec.hostAliases`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostaliases-pod
spec:
  restartPolicy: Never
  hostAliases:
  - ip: "127.0.0.1"
    hostnames:
    - "foo.local"
    - "bar.local"
  - ip: "10.1.2.3"
    hostnames:
    - "foo.remote"
    - "bar.remote"
  containers:
  - name: cat-hosts
    image: busybox:1.28
    command:
    - cat
    args:
    - "/etc/hosts"
```

Bạn có thể khởi chạy một Pod với cấu hình đó bằng cách chạy:

```shell
kubectl apply -f https://k8s.io/examples/service/networking/hostaliases-pod.yaml
```

```
pod/hostaliases-pod created
```

Xem chi tiết của Pod để biết địa chỉ IPv4 và trạng thái của nó:

```shell
kubectl get pod --output=wide
```

```
NAME                           READY     STATUS      RESTARTS   AGE       IP              NODE
hostaliases-pod                0/1       Completed   0          6s        10.200.0.5      worker0
```

Nội dung file `hosts` trông như sau:

```shell
kubectl logs hostaliases-pod
```

```
# Kubernetes-managed hosts file.
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
fe00::0	ip6-mcastprefix
fe00::1	ip6-allnodes
fe00::2	ip6-allrouters
10.200.0.5	hostaliases-pod

# Entries added by HostAliases.
127.0.0.1	foo.local	bar.local
10.1.2.3	foo.remote	bar.remote
```

với các entry bổ sung được ghi ở cuối file.

## Vì sao kubelet quản lý file hosts? (Why does the kubelet manage the hosts file?) {#why-does-kubelet-manage-the-hosts-file}

kubelet quản lý file `hosts` cho từng container của Pod nhằm ngăn container runtime sửa file này
sau khi các container đã khởi động.
Về mặt lịch sử, Kubernetes luôn dùng Docker Engine làm container runtime, và Docker Engine sẽ
sửa file `/etc/hosts` sau khi mỗi container khởi động.

Kubernetes hiện nay có thể dùng nhiều container runtime khác nhau; dù vậy, kubelet vẫn quản lý
file hosts bên trong từng container để kết quả luôn đúng như mong đợi, bất kể bạn dùng container
runtime nào.

> **Thận trọng:** Tránh sửa thủ công file hosts bên trong một container.
>
> Nếu bạn sửa thủ công file hosts, những thay đổi đó sẽ mất khi container thoát.

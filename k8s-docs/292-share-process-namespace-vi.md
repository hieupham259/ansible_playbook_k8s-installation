# Chia sẻ Process Namespace giữa các Container trong một Pod (Share Process Namespace between Containers in a Pod)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/

Trang này hướng dẫn cách cấu hình chia sẻ process namespace cho một pod. Khi tính năng chia sẻ
process namespace được bật, các process (tiến trình) trong một container sẽ hiển thị với tất cả
các container khác trong cùng pod.

Bạn có thể dùng tính năng này để cấu hình các container phối hợp với nhau, chẳng hạn một
sidecar container xử lý log, hoặc để khắc phục sự cố (troubleshoot) các container image không
kèm theo các tiện ích gỡ lỗi như shell.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Cấu hình một Pod (Configure a Pod)

Tính năng chia sẻ process namespace được bật bằng field `shareProcessNamespace` trong `.spec`
của một Pod. Ví dụ:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  shareProcessNamespace: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox:1.28
    command: ["sleep", "3600"]
    securityContext:
      capabilities:
        add:
        - SYS_PTRACE
    stdin: true
    tty: true
```

1. Tạo pod `nginx` trên cluster của bạn:

   ```shell
   kubectl apply -f https://k8s.io/examples/pods/share-process-namespace.yaml
   ```

1. Attach vào container `shell` và chạy `ps`:

   ```shell
   kubectl exec -it nginx -c shell -- /bin/sh
   ```

   Nếu bạn không thấy dấu nhắc lệnh, hãy thử nhấn Enter. Bên trong shell của container:

   ```shell
   # chạy lệnh này bên trong container "shell"
   ps ax
   ```

   Kết quả xuất ra tương tự như sau:

   ```none
   PID   USER     TIME  COMMAND
       1 root      0:00 /pause
       8 root      0:00 nginx: master process nginx -g daemon off;
      14 101       0:00 nginx: worker process
      15 root      0:00 sh
      21 root      0:00 ps ax
   ```

Bạn có thể gửi tín hiệu (signal) tới các process trong container khác. Ví dụ, gửi `SIGHUP` tới
`nginx` để khởi động lại worker process. Việc này yêu cầu capability `SYS_PTRACE`.

```shell
# chạy lệnh này bên trong container "shell"
kill -HUP 8   # đổi "8" cho khớp với PID của process nginx chính, nếu cần
ps ax
```

Kết quả xuất ra tương tự như sau:

```none
PID   USER     TIME  COMMAND
    1 root      0:00 /pause
    8 root      0:00 nginx: master process nginx -g daemon off;
   15 root      0:00 sh
   22 101       0:00 nginx: worker process
   23 root      0:00 ps ax
```

Thậm chí bạn còn có thể truy cập hệ thống tập tin (file system) của một container khác thông
qua liên kết `/proc/$pid/root`.

```shell
# chạy lệnh này bên trong container "shell"
# đổi "8" thành PID của process Nginx, nếu cần
head /proc/8/root/etc/nginx/nginx.conf
```

Kết quả xuất ra tương tự như sau:

```none
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
```

## Hiểu về chia sẻ process namespace (Understanding process namespace sharing)

Các Pod vốn đã chia sẻ nhiều tài nguyên với nhau, nên việc chúng cũng chia sẻ một process
namespace là điều hợp lý. Tuy nhiên, một số container có thể kỳ vọng được cô lập khỏi các
container khác, vì vậy điều quan trọng là phải hiểu những khác biệt sau:

1. **Process của container không còn mang PID 1 nữa.** Một số container từ chối khởi động nếu
   không có PID 1 (ví dụ: các container dùng `systemd`) hoặc chạy các lệnh như `kill -HUP 1`
   để gửi tín hiệu tới process của container. Trong các pod có process namespace được chia sẻ,
   `kill -HUP 1` sẽ gửi tín hiệu tới pod sandbox (`/pause` trong ví dụ ở trên).

1. **Các process hiển thị với các container khác trong pod.** Điều này bao gồm toàn bộ thông
   tin nhìn thấy được trong `/proc`, chẳng hạn các mật khẩu được truyền dưới dạng đối số
   (argument) hoặc biến môi trường. Những thông tin này chỉ được bảo vệ bởi các quyền
   (permission) Unix thông thường.

1. **Hệ thống tập tin của container hiển thị với các container khác trong pod thông qua liên
   kết `/proc/$pid/root`.** Điều này giúp việc gỡ lỗi dễ dàng hơn, nhưng cũng có nghĩa là các
   bí mật (secret) trên hệ thống tập tin chỉ được bảo vệ bởi các quyền của hệ thống tập tin.

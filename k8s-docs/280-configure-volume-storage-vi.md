# Cấu hình Pod sử dụng Volume để lưu trữ (Configure a Pod to Use a Volume for Storage)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/configure-volume-storage/>

Trang này hướng dẫn cách cấu hình một Pod sử dụng Volume để lưu trữ.

Hệ thống file (filesystem) của một Container chỉ tồn tại chừng nào Container còn tồn tại. Vì vậy,
khi một Container kết thúc và khởi động lại, các thay đổi trên filesystem sẽ bị mất. Để có nơi
lưu trữ nhất quán hơn và độc lập với Container, bạn có thể dùng một
[Volume](./91-volumes-vi.md). Điều này đặc biệt quan trọng đối với các ứng dụng có trạng thái
(stateful), chẳng hạn như các kho lưu trữ key-value (ví dụ Redis) và các cơ sở dữ liệu.

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

## Cấu hình một volume cho Pod (Configure a volume for a Pod)

Trong bài thực hành này, bạn tạo một Pod chạy một Container. Pod này có một Volume kiểu
[emptyDir](./91-volumes-vi.md#emptydir) tồn tại trong suốt vòng đời của Pod, ngay cả khi
Container kết thúc và khởi động lại. Đây là file cấu hình cho Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis
    volumeMounts:
    - name: redis-storage
      mountPath: /data/redis
  volumes:
  - name: redis-storage
    emptyDir: {}
```

1. Tạo Pod:

   ```shell
   kubectl apply -f https://k8s.io/examples/pods/storage/redis.yaml
   ```

1. Xác minh rằng Container của Pod đang chạy, sau đó theo dõi (watch) các thay đổi của Pod:

   ```shell
   kubectl get pod redis --watch
   ```

   Output trông như sau:

   ```console
   NAME      READY     STATUS    RESTARTS   AGE
   redis     1/1       Running   0          13s
   ```

1. Trong một terminal khác, mở một shell vào Container đang chạy:

   ```shell
   kubectl exec -it redis -- /bin/bash
   ```

1. Trong shell của bạn, đi tới `/data/redis`, sau đó tạo một file:

   ```shell
   root@redis:/data# cd /data/redis/
   root@redis:/data/redis# echo Hello > test-file
   ```

1. Trong shell của bạn, liệt kê các tiến trình (process) đang chạy:

   ```shell
   root@redis:/data/redis# apt-get update
   root@redis:/data/redis# apt-get install procps
   root@redis:/data/redis# ps aux
   ```

   Output tương tự như sau:

   ```console
   USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
   redis        1  0.1  0.1  33308  3828 ?        Ssl  00:46   0:00 redis-server *:6379
   root        12  0.0  0.0  20228  3020 ?        Ss   00:47   0:00 /bin/bash
   root        15  0.0  0.0  17500  2072 ?        R+   00:48   0:00 ps aux
   ```

1. Trong shell của bạn, kill tiến trình Redis:

   ```shell
   root@redis:/data/redis# kill <pid>
   ```

   trong đó `<pid>` là ID tiến trình (PID) của Redis.

1. Trong terminal ban đầu của bạn, theo dõi các thay đổi của Pod Redis. Cuối cùng, bạn sẽ thấy
   kết quả tương tự như sau:

   ```console
   NAME      READY     STATUS     RESTARTS   AGE
   redis     1/1       Running    0          13s
   redis     0/1       Completed  0         6m
   redis     1/1       Running    1         6m
   ```

Đến thời điểm này, Container đã kết thúc và khởi động lại. Đó là vì Pod Redis có
[restartPolicy](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#podspec-v1-core)
là `Always`.

1. Mở một shell vào Container đã được khởi động lại:

   ```shell
   kubectl exec -it redis -- /bin/bash
   ```

1. Trong shell của bạn, đi tới `/data/redis` và xác minh rằng `test-file` vẫn còn ở đó.

   ```shell
   root@redis:/data/redis# cd /data/redis/
   root@redis:/data/redis# ls
   test-file
   ```

1. Xóa Pod mà bạn đã tạo cho bài thực hành này:

   ```shell
   kubectl delete pod redis
   ```

## Tiếp theo (What's next)

- Xem [Volume](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#volume-v1-core).

- Xem [Pod](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#pod-v1-core).

- Ngoài kho lưu trữ trên đĩa cục bộ do `emptyDir` cung cấp, Kubernetes còn hỗ trợ nhiều giải pháp
  lưu trữ gắn qua mạng (network-attached storage) khác nhau, bao gồm PD trên GCE và EBS trên EC2;
  các giải pháp này được ưu tiên cho dữ liệu quan trọng và sẽ xử lý các chi tiết như mount và
  unmount thiết bị trên các node. Xem [Volumes](./91-volumes-vi.md) để biết thêm chi tiết.

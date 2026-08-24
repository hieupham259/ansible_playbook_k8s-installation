# Job với giao tiếp Pod-đến-Pod (Job with Pod-to-Pod Communication)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/job/job-with-pod-to-pod-communication/

Trong ví dụ này, bạn sẽ chạy một Job ở [chế độ hoàn thành Indexed (Indexed completion mode)](https://kubernetes.io/blog/2021/04/19/introducing-indexed-jobs/)
được cấu hình sao cho các Pod do Job tạo ra có thể giao tiếp với nhau bằng hostname của Pod
thay vì địa chỉ IP của Pod.

Các Pod trong một Job có thể cần giao tiếp với nhau. Workload của người dùng chạy trong mỗi Pod
có thể truy vấn Kubernetes API server để biết IP của các Pod khác, nhưng sẽ đơn giản hơn nhiều
nếu dựa vào cơ chế phân giải DNS có sẵn của Kubernetes.

Job ở chế độ hoàn thành Indexed tự động đặt hostname của các Pod theo định dạng
`${jobName}-${completionIndex}`. Bạn có thể dùng định dạng này để xây dựng hostname
của Pod một cách xác định (deterministic) và cho phép các Pod giao tiếp với nhau *mà không*
cần tạo kết nối client tới control plane của Kubernetes để lấy hostname/IP của Pod
thông qua các yêu cầu API.

Cấu hình này hữu ích cho các trường hợp cần kết nối mạng giữa các Pod nhưng bạn không muốn
phụ thuộc vào kết nối mạng với Kubernetes API server.

## Trước khi bạn bắt đầu (Before you begin)

Bạn nên đã quen với cách sử dụng cơ bản của [Job](67-job-vi.md).

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được
cấu hình để giao tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một
cluster có ít nhất hai node không đóng vai trò máy chủ control plane. Nếu bạn
chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
hoặc dùng một trong các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản v1.21 hoặc mới hơn. Để kiểm tra phiên bản,
hãy chạy `kubectl version`.

> **Ghi chú:**
> Nếu bạn đang dùng minikube hoặc một công cụ tương tự, bạn có thể cần thực hiện
> [các bước bổ sung](https://minikube.sigs.k8s.io/docs/handbook/addons/ingress-dns/)
> để đảm bảo bạn có DNS.

## Khởi chạy một Job với giao tiếp Pod-đến-Pod (Starting a Job with pod-to-pod communication)

Để cho phép các Pod trong một Job giao tiếp với nhau bằng hostname của Pod, bạn phải làm những việc sau:

1. Thiết lập một [headless Service](82-service-vi.md#headless-services)
   với một label selector hợp lệ khớp với các Pod do Job của bạn tạo ra. Headless Service phải
   nằm cùng namespace với Job. Một cách dễ dàng để làm điều này là dùng selector
   `job-name: <tên-job-của-bạn>`, vì label `job-name` sẽ được Kubernetes tự động thêm vào.
   Cấu hình này sẽ kích hoạt hệ thống DNS tạo các bản ghi hostname
   cho các Pod đang chạy Job của bạn.

1. Cấu hình headless Service làm subdomain service cho các Pod của Job bằng cách thêm
   giá trị sau vào template spec của Job:

   ```yaml
   subdomain: <tên-headless-svc>
   ```

### Ví dụ (Example)

Dưới đây là một ví dụ hoạt động được của một Job có bật giao tiếp Pod-đến-Pod thông qua hostname của Pod.
Job chỉ hoàn thành sau khi tất cả các Pod ping thành công lẫn nhau bằng hostname.

> **Ghi chú:**
> Trong Bash script chạy trên mỗi Pod ở ví dụ dưới đây, hostname của Pod cũng có thể được
> thêm tiền tố namespace nếu Pod cần được truy cập từ bên ngoài namespace đó.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-svc
spec:
  clusterIP: None # clusterIP phải là None để tạo headless service
  selector:
    job-name: example-job # phải khớp với tên Job
---
apiVersion: batch/v1
kind: Job
metadata:
  name: example-job
spec:
  completions: 3
  parallelism: 3
  completionMode: Indexed
  template:
    spec:
      subdomain: headless-svc # phải khớp với tên Service
      restartPolicy: Never
      containers:
      - name: example-workload
        image: bash:latest
        command:
        - bash
        - -c
        - |
          for i in 0 1 2
          do
            gotStatus="-1"
            wantStatus="0"             
            while [ $gotStatus -ne $wantStatus ]
            do                                       
              ping -c 1 example-job-${i}.headless-svc > /dev/null 2>&1
              gotStatus=$?                
              if [ $gotStatus -ne $wantStatus ]; then
                echo "Failed to ping pod example-job-${i}.headless-svc, retrying in 1 second..."
                sleep 1
              fi
            done                                                         
            echo "Successfully pinged pod: example-job-${i}.headless-svc"
          done
```

Sau khi apply ví dụ trên, các Pod truy cập lẫn nhau qua mạng bằng tên dạng:
`<pod-hostname>.<tên-headless-service>`. Bạn sẽ thấy output tương tự như sau:

```shell
kubectl logs example-job-0-qws42
```

```
Failed to ping pod example-job-0.headless-svc, retrying in 1 second...
Successfully pinged pod: example-job-0.headless-svc
Successfully pinged pod: example-job-1.headless-svc
Successfully pinged pod: example-job-2.headless-svc
```

> **Ghi chú:**
> Lưu ý rằng định dạng tên `<pod-hostname>.<tên-headless-service>` dùng trong
> ví dụ này sẽ không hoạt động nếu DNS policy được đặt là `None` hoặc `Default`.
> Tham khảo [DNS Policy của Pod](10-dns-pod-service-vi.md#pod-s-dns-policy).

# Chạy các tác vụ tự động với CronJob (Running Automated Tasks with a CronJob)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/>

Trang này hướng dẫn cách chạy các tác vụ tự động bằng đối tượng CronJob của Kubernetes.

## Trước khi bạn bắt đầu (Before you begin)

* Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
  tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
  không đóng vai trò là máy chủ control plane. Nếu bạn chưa có cluster, bạn có thể tạo một
  cluster bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
  hoặc dùng một trong các sân chơi (playground) Kubernetes sau:

  * [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
  * [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
  * [KodeKloud](https://kodekloud.com/public-playgrounds)

## Tạo một CronJob (Creating a CronJob) {#creating-a-cron-job}

Cron job cần một file cấu hình.
Dưới đây là manifest cho một CronJob chạy một tác vụ minh họa đơn giản mỗi phút một lần:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.28
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

Chạy CronJob ví dụ bằng lệnh sau:

```shell
kubectl create -f https://k8s.io/examples/application/job/cronjob.yaml
```

Kết quả tương tự như sau:

```
cronjob.batch/hello created
```

Sau khi tạo cron job, lấy trạng thái của nó bằng lệnh sau:

```shell
kubectl get cronjob hello
```

Kết quả tương tự như sau:

```
NAME    SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
hello   */1 * * * *   False     0        <none>          10s
```

Như bạn thấy từ kết quả của lệnh trên, cron job này chưa lập lịch (schedule) hay chạy bất kỳ
job nào. Theo dõi (watch) job được tạo ra trong khoảng một phút:

```shell
kubectl get jobs --watch
```

Kết quả tương tự như sau:

```
NAME               COMPLETIONS   DURATION   AGE
hello-4111706356   0/1                      0s
hello-4111706356   0/1           0s         0s
hello-4111706356   1/1           5s         5s
```

Giờ bạn đã thấy một job đang chạy được lập lịch bởi cron job "hello".
Bạn có thể dừng theo dõi job và xem lại cron job để thấy rằng nó đã lập lịch cho job đó:

```shell
kubectl get cronjob hello
```

Kết quả tương tự như sau:

```
NAME    SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
hello   */1 * * * *   False     0        50s             75s
```

Bạn sẽ thấy cron job `hello` đã lập lịch thành công một job tại thời điểm được ghi trong
`LAST SCHEDULE`. Hiện tại có 0 job đang hoạt động (active), nghĩa là job đã hoàn thành hoặc
thất bại.

Bây giờ, hãy tìm các Pod mà job được lập lịch gần nhất đã tạo ra và xem output chuẩn
(standard output) của một trong các Pod đó.

> **Ghi chú:**
> Tên của job khác với tên của Pod.

```shell
# Thay "hello-4111706356" bằng tên job trong hệ thống của bạn
pods=$(kubectl get pods --selector=job-name=hello-4111706356 --output=jsonpath={.items[*].metadata.name})
```

Hiển thị log của Pod:

```shell
kubectl logs $pods
```

Kết quả tương tự như sau:

```
Fri Feb 22 11:02:09 UTC 2019
Hello from the Kubernetes cluster
```

## Xóa một CronJob (Deleting a CronJob) {#deleting-a-cron-job}

Khi bạn không cần một cron job nữa, hãy xóa nó bằng `kubectl delete cronjob <tên cronjob>`:

```shell
kubectl delete cronjob hello
```

Việc xóa cron job sẽ xóa tất cả các job và Pod mà nó đã tạo ra, đồng thời ngăn nó tạo thêm
các job mới. Bạn có thể đọc thêm về việc xóa job trong
[thu gom rác (garbage collection)](./36-garbage-collection-vi.md).

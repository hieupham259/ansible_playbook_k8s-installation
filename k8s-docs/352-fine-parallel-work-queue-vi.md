# Xử lý song song mịn sử dụng hàng đợi công việc (Fine Parallel Processing Using a Work Queue)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/job/fine-parallel-processing-work-queue/

Trong ví dụ này, bạn sẽ chạy một Job Kubernetes thực hiện nhiều tác vụ song song
dưới dạng các tiến trình worker, mỗi tiến trình chạy trong một Pod riêng.

Trong ví dụ này, khi mỗi pod được tạo ra, nó lấy một đơn vị công việc
từ một hàng đợi tác vụ (task queue), xử lý nó, và lặp lại cho đến khi hết hàng đợi.

Dưới đây là tổng quan các bước trong ví dụ này:

1. **Khởi động một dịch vụ lưu trữ để chứa hàng đợi công việc.** Trong ví dụ này, bạn sẽ dùng Redis để lưu
   các phần tử công việc. Trong [ví dụ trước](https://kubernetes.io/docs/tasks/job/coarse-parallel-processing-work-queue),
   bạn đã dùng RabbitMQ. Trong ví dụ này, bạn sẽ dùng Redis và một thư viện client hàng đợi công việc tự viết;
   lý do là AMQP không cung cấp một cách tốt để client
   phát hiện khi nào một hàng đợi công việc có độ dài hữu hạn đã rỗng. Trong thực tế, bạn sẽ dựng một kho lưu trữ
   như Redis một lần và tái sử dụng nó cho hàng đợi công việc của nhiều job, cũng như cho các mục đích khác.
1. **Tạo một hàng đợi và nạp thông điệp vào đó.** Mỗi thông điệp đại diện cho một tác vụ cần thực hiện.
   Trong ví dụ này, một thông điệp là một số nguyên mà chúng ta sẽ thực hiện một phép tính tốn thời gian trên đó.
1. **Khởi động một Job xử lý các tác vụ từ hàng đợi.** Job khởi động vài pod. Mỗi pod lấy
   một tác vụ từ hàng đợi thông điệp, xử lý nó, và lặp lại cho đến khi hết hàng đợi.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes và công cụ dòng lệnh kubectl phải được cấu hình
để giao tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất
hai node không đóng vai trò host của control plane. Nếu bạn chưa có cluster, bạn có thể
tạo một cluster bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
hoặc dùng một trong các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Bạn sẽ cần một container image registry, nơi bạn có thể tải image lên để chạy trong cluster của mình.
Ví dụ này dùng [Docker Hub](https://hub.docker.com/), nhưng bạn có thể điều chỉnh để dùng một
container image registry khác.

Ví dụ này cũng giả định rằng bạn đã cài Docker trên máy cục bộ. Bạn dùng Docker để
build các container image.

Hãy làm quen với cách dùng cơ bản, không song song, của
[Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/).

## Khởi động Redis (Starting Redis)

Với ví dụ này, để đơn giản, bạn sẽ khởi động một instance Redis duy nhất.
Xem [ví dụ Redis](https://github.com/kubernetes/examples/tree/master/web/guestbook/) để có
một ví dụ về việc triển khai Redis có khả năng mở rộng (scalable) và dự phòng (redundant).

Bạn cũng có thể tải trực tiếp các file sau:

- [`redis-pod.yaml`](https://kubernetes.io/examples/application/job/redis/redis-pod.yaml)
- [`redis-service.yaml`](https://kubernetes.io/examples/application/job/redis/redis-service.yaml)
- [`Dockerfile`](https://kubernetes.io/examples/application/job/redis/Dockerfile)
- [`job.yaml`](https://kubernetes.io/examples/application/job/redis/job.yaml)
- [`rediswq.py`](https://kubernetes.io/examples/application/job/redis/rediswq.py)
- [`worker.py`](https://kubernetes.io/examples/application/job/redis/worker.py)

Để khởi động một instance Redis duy nhất, bạn cần tạo pod redis và service redis:

```shell
kubectl apply -f https://k8s.io/examples/application/job/redis/redis-pod.yaml
kubectl apply -f https://k8s.io/examples/application/job/redis/redis-service.yaml
```

## Nạp tác vụ vào hàng đợi (Filling the queue with tasks)

Bây giờ, hãy nạp vào hàng đợi một số "tác vụ". Trong ví dụ này, các tác vụ là các chuỗi
cần được in ra.

Khởi động một pod tương tác tạm thời để chạy Redis CLI.

```shell
kubectl run -i --tty temp --image redis --command "/bin/sh"
```
```
Waiting for pod default/redis2-c7h78 to be running, status is Pending, pod ready: false
Hit enter for command prompt
```

Bây giờ nhấn enter, khởi động Redis CLI, và tạo một danh sách (list) chứa một số phần tử công việc.

```shell
redis-cli -h redis
```
```console
redis:6379> rpush job2 "apple"
(integer) 1
redis:6379> rpush job2 "banana"
(integer) 2
redis:6379> rpush job2 "cherry"
(integer) 3
redis:6379> rpush job2 "date"
(integer) 4
redis:6379> rpush job2 "fig"
(integer) 5
redis:6379> rpush job2 "grape"
(integer) 6
redis:6379> rpush job2 "lemon"
(integer) 7
redis:6379> rpush job2 "melon"
(integer) 8
redis:6379> rpush job2 "orange"
(integer) 9
redis:6379> lrange job2 0 -1
1) "apple"
2) "banana"
3) "cherry"
4) "date"
5) "fig"
6) "grape"
7) "lemon"
8) "melon"
9) "orange"
```

Như vậy, danh sách với khóa `job2` sẽ là hàng đợi công việc.

Lưu ý: nếu Kube DNS của bạn chưa được thiết lập đúng, bạn có thể cần đổi
bước đầu tiên của khối lệnh trên thành `redis-cli -h $REDIS_SERVICE_HOST`.

## Tạo container image (Create a container image) {#create-an-image}

Bây giờ bạn đã sẵn sàng tạo một image sẽ xử lý công việc trong hàng đợi đó.

Bạn sẽ dùng một chương trình worker viết bằng Python với một client Redis để đọc
các thông điệp từ hàng đợi thông điệp.

Một thư viện client hàng đợi công việc Redis đơn giản được cung cấp sẵn,
tên là `rediswq.py` ([Tải về](https://kubernetes.io/examples/application/job/redis/rediswq.py)).

Chương trình "worker" trong mỗi Pod của Job dùng thư viện client hàng đợi công việc
để lấy công việc. Nội dung của nó như sau:

```python
#!/usr/bin/env python

import time
import rediswq

host="redis"
# Bỏ comment hai dòng tiếp theo nếu Kube-DNS của bạn không hoạt động.
# import os
# host = os.getenv("REDIS_SERVICE_HOST")

q = rediswq.RedisWQ(name="job2", host=host)
print("Worker with sessionID: " +  q.sessionID())
print("Initial queue state: empty=" + str(q.empty()))
while not q.empty():
  item = q.lease(lease_secs=10, block=True, timeout=2) 
  if item is not None:
    itemstr = item.decode("utf-8")
    print("Working on " + itemstr)
    time.sleep(10) # Đặt công việc thực sự của bạn ở đây thay cho sleep.
    q.complete(item)
  else:
    print("Waiting for work")
print("Queue empty, exiting")
```

Bạn cũng có thể tải các file [`worker.py`](https://kubernetes.io/examples/application/job/redis/worker.py),
[`rediswq.py`](https://kubernetes.io/examples/application/job/redis/rediswq.py) và
[`Dockerfile`](https://kubernetes.io/examples/application/job/redis/Dockerfile), rồi build
container image. Dưới đây là một ví dụ dùng Docker để build image:

```shell
docker build -t job-wq-2 .
```

### Đẩy image lên registry (Push the image)

Với [Docker Hub](https://hub.docker.com/), gắn tag cho image ứng dụng của bạn với
username của bạn và đẩy (push) lên Hub bằng các lệnh dưới đây. Thay
`<username>` bằng username Hub của bạn.

```shell
docker tag job-wq-2 <username>/job-wq-2
docker push <username>/job-wq-2
```

Bạn cần đẩy lên một repository công khai hoặc [cấu hình cluster của bạn để có thể truy cập
repository riêng tư của bạn](https://kubernetes.io/docs/concepts/containers/images/).

## Định nghĩa một Job (Defining a Job)

Dưới đây là manifest cho Job mà bạn sẽ tạo:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-wq-2
spec:
  parallelism: 2
  template:
    metadata:
      name: job-wq-2
    spec:
      containers:
      - name: c
        image: gcr.io/myproject/job-wq-2
      restartPolicy: OnFailure
```

> **Ghi chú:**
> Hãy nhớ sửa manifest để
> đổi `gcr.io/myproject` thành đường dẫn của riêng bạn.

Trong ví dụ này, mỗi pod xử lý vài phần tử từ hàng đợi rồi thoát khi không còn phần tử nào nữa.
Vì chính các worker tự phát hiện khi nào hàng đợi công việc rỗng, và Job controller không
biết gì về hàng đợi công việc, nên nó dựa vào việc các worker phát tín hiệu khi chúng làm xong việc.
Các worker phát tín hiệu rằng hàng đợi đã rỗng bằng cách thoát với trạng thái thành công. Vì vậy, ngay khi
**bất kỳ** worker nào thoát với trạng thái thành công, controller biết rằng công việc đã xong, và các Pod sẽ sớm thoát.
Do đó, bạn cần để trống (không đặt) số lần hoàn thành (completion count) của Job. Job controller sẽ chờ
các pod còn lại hoàn thành nốt.

## Chạy Job (Running the Job)

Bây giờ, hãy chạy Job:

```shell
# lệnh này giả định bạn đã tải manifest về và đã chỉnh sửa nó
kubectl apply -f ./job.yaml
```

Bây giờ chờ một chút, rồi kiểm tra Job:

```shell
kubectl describe jobs/job-wq-2
```
```
Name:             job-wq-2
Namespace:        default
Selector:         controller-uid=b1c7e4e3-92e1-11e7-b85e-fa163ee3c11f
Labels:           controller-uid=b1c7e4e3-92e1-11e7-b85e-fa163ee3c11f
                  job-name=job-wq-2
Annotations:      <none>
Parallelism:      2
Completions:      <unset>
Start Time:       Mon, 11 Jan 2022 17:07:59 +0000
Pods Statuses:    1 Running / 0 Succeeded / 0 Failed
Pod Template:
  Labels:       controller-uid=b1c7e4e3-92e1-11e7-b85e-fa163ee3c11f
                job-name=job-wq-2
  Containers:
   c:
    Image:              container-registry.example/exampleproject/job-wq-2
    Port:
    Environment:        <none>
    Mounts:             <none>
  Volumes:              <none>
Events:
  FirstSeen    LastSeen    Count    From            SubobjectPath    Type        Reason            Message
  ---------    --------    -----    ----            -------------    --------    ------            -------
  33s          33s         1        {job-controller }                Normal      SuccessfulCreate  Created pod: job-wq-2-lglf8
```

Bạn có thể chờ Job thành công, với một khoảng thời gian chờ tối đa (timeout):
```shell
# Việc kiểm tra tên condition không phân biệt chữ hoa chữ thường
kubectl wait --for=condition=complete --timeout=300s job/job-wq-2
```

```shell
kubectl logs pods/job-wq-2-7r7b2
```
```
Worker with sessionID: bbd72d0a-9e5c-4dd6-abf6-416cc267991f
Initial queue state: empty=False
Working on banana
Working on date
Working on lemon
```

Như bạn thấy, một trong các pod của Job này đã xử lý nhiều đơn vị công việc.

## Các cách thay thế (Alternatives)

Nếu việc chạy một dịch vụ hàng đợi hoặc sửa container của bạn để dùng hàng đợi công việc là bất tiện,
bạn có thể cân nhắc một trong các
[mẫu job (job patterns)](https://kubernetes.io/docs/concepts/workloads/controllers/job/#job-patterns) khác.

Nếu bạn có một luồng công việc xử lý nền (background processing) liên tục cần chạy,
hãy cân nhắc chạy các worker nền của bạn bằng một ReplicaSet thay thế,
và cân nhắc dùng một thư viện xử lý nền chẳng hạn như
[https://github.com/resque/resque](https://github.com/resque/resque).

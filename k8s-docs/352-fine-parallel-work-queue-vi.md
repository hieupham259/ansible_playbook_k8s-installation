# Xử lý song song mịn sử dụng hàng đợi công việc (Fine Parallel Processing Using a Work Queue)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/job/fine-parallel-processing-work-queue/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 5 — Mạng nền tảng](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng), dòng **Thực
hành**, bài 5/10 · **Không có mục thực hành trong lab**:
[Lab 4b](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) xếp bài này vào bảng *không kiểm chứng được
trên cluster baseline* vì nó bắt bạn tự build một image worker Python rồi đẩy lên registry; phần
B6.4 của lab đó chỉ thực hành **hình dạng** của Job hàng đợi công việc. Bảng ánh xạ của
[Lab 5a](labs/LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md) và
[Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) cũng không có bài này.

Bài về Job nhưng nằm ở giai đoạn 5 vì hàng đợi sống sau một **Service**: worker tìm Redis bằng tên
`redis`, và bài đưa sẵn đường lui bằng biến môi trường khi DNS không hoạt động — đúng hai cách khám
phá Service mà giai đoạn 5 dạy. Đọc bài này như một **mẫu thiết kế**, không phải một bài chạy được
ngay: phần image là thứ bạn không dựng ở lab.

**Phải hiểu ở lần đọc này:**

- Ba bước của mẫu, liệt kê ngay đầu bài: dựng dịch vụ lưu trữ hàng đợi (một Pod Redis và một Service
  Redis), nạp các phần tử công việc vào một list Redis khóa `job2`, rồi chạy Job mà mỗi Pod lấy việc
  ra xử lý tới khi hết. Bài chọn Redis thay RabbitMQ của [351](351-coarse-parallel-work-queue-vi.md)
  vì "AMQP không cung cấp một cách tốt để client phát hiện khi nào một hàng đợi công việc có độ dài
  hữu hạn đã rỗng".
- Worker tìm hàng đợi **bằng tên Service**: `worker.py` đặt `host="redis"`, và bài để sẵn hai dòng
  comment `os.getenv("REDIS_SERVICE_HOST")` cho trường hợp DNS hỏng — cùng ghi chú tương ứng cho
  `redis-cli -h $REDIS_SERVICE_HOST` ở mục *Nạp tác vụ vào hàng đợi*.
- Điểm quyết định của manifest ở mục *Định nghĩa một Job*: chỉ đặt `parallelism: 2` và **để trống
  `completions`**. Lý do bài nêu: Job controller không biết gì về hàng đợi, nên nó dựa vào tín hiệu
  của worker — ngay khi **bất kỳ** worker nào thoát với trạng thái thành công, controller coi là
  công việc đã xong và chờ các Pod còn lại hoàn thành nốt. `restartPolicy` là `OnFailure`.
- Vòng đời một phần tử công việc trong `worker.py`: `q.lease(...)` mượn việc, xử lý, rồi
  `q.complete(item)`; vòng lặp dừng khi `q.empty()` và in `Queue empty, exiting`. Hệ quả nhìn thấy ở
  mục *Chạy Job*: một Pod xử lý **nhiều** phần tử, log mẫu cho thấy `banana`, `date`, `lemon` trong
  cùng một Pod.
- Mục *Các cách thay thế*: nếu chạy một dịch vụ hàng đợi là bất tiện thì chọn
  [mẫu job khác](67-job-vi.md#job-patterns); còn nếu là luồng xử lý nền **chạy liên tục** thì dùng
  ReplicaSet chứ không phải Job.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Tạo container image* và *Đẩy image lên registry* — `docker build -t job-wq-2 .`, `docker tag`, `docker push` | cluster lab không build và không push image; đây chính là lý do bài không có bước thực hành | [Lab 4b](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) phần B6.4 thực hành **hình dạng** Job hàng đợi công việc, và bảng *không kiểm chứng được* của lab đó ghi rõ lý do |
| Nội dung và cơ chế lease/session của thư viện `rediswq.py` | đây là thư viện ví dụ của trang gốc, không phải API Kubernetes | không có bài nào trong lộ trình dạy nó; đoạn `worker.py` in ngay trong bài là đủ để thấy vòng lặp |
| Link tới ví dụ triển khai Redis "có khả năng mở rộng và dự phòng" ở mục *Khởi động Redis* | bài chỉ trỏ ra ngoài; với ví dụ này một instance Redis duy nhất là đủ | bài [343 — Chạy ứng dụng có trạng thái được nhân bản](343-run-replicated-stateful-application-vi.md), [giai đoạn 6](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ) |

---

Trong ví dụ này, bạn sẽ chạy một Job Kubernetes thực hiện nhiều tác vụ song song
dưới dạng các tiến trình worker, mỗi tiến trình chạy trong một Pod riêng.

Trong ví dụ này, khi mỗi pod được tạo ra, nó lấy một đơn vị công việc
từ một hàng đợi tác vụ (task queue), xử lý nó, và lặp lại cho đến khi hết hàng đợi.

Dưới đây là tổng quan các bước trong ví dụ này:

1. **Khởi động một dịch vụ lưu trữ để chứa hàng đợi công việc.** Trong ví dụ này, bạn sẽ dùng Redis để lưu
   các phần tử công việc. Trong [ví dụ trước](351-coarse-parallel-work-queue-vi.md),
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
[Job](67-job-vi.md).

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
repository riêng tư của bạn](40-images-vi.md).

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
[mẫu job (job patterns)](67-job-vi.md#job-patterns) khác.

Nếu bạn có một luồng công việc xử lý nền (background processing) liên tục cần chạy,
hãy cân nhắc chạy các worker nền của bạn bằng một ReplicaSet thay thế,
và cân nhắc dùng một thư viện xử lý nền chẳng hạn như
[https://github.com/resque/resque](https://github.com/resque/resque).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 5:

1. Manifest Job trong bài chỉ có `parallelism: 2` và **không** có `completions`. Vì sao phải để
   trống trường đó, và Job controller dựa vào tín hiệu nào để biết công việc đã xong?
2. **Câu bẫy.** Hàng đợi có chín phần tử còn Job chỉ chạy hai Pod. Vậy Job có phải chạy đủ chín Pod
   mới xong không, và mỗi Pod xử lý đúng một phần tử phải không? Log mẫu ở mục *Chạy Job* nói gì?
3. Hai Pod worker của bạn nằm trên `lab-k8s-worker1` và `lab-k8s-worker2`, còn Redis chạy ở một Pod
   khác trong cluster. Cả hai worker đều viết cứng `host="redis"`. Nhờ đâu chúng tìm được đúng Redis,
   và bài cho đường lui nào khi cách đó không hoạt động?
4. Vì sao bài này dùng Redis trong khi [351](351-coarse-parallel-work-queue-vi.md) dùng RabbitMQ? Câu
   trả lời nằm ở tính chất nào của hàng đợi, chứ không phải ở việc phần mềm nào tốt hơn.

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Vì **Job controller không biết gì về hàng đợi công việc**: nó không đếm được còn bao nhiêu phần
   tử, nên không có con số `completions` nào đúng. Thay vào đó chính các worker tự phát hiện hàng đợi
   rỗng và **phát tín hiệu bằng cách thoát với trạng thái thành công**. Bài nói rõ: ngay khi **bất
   kỳ** worker nào thoát thành công, controller biết công việc đã xong, và nó chờ các Pod còn lại
   hoàn thành nốt.
2. **Không, và không.** Số Pod là `parallelism`, không phải số phần tử — chỉ hai Pod chạy, và mỗi Pod
   **lặp**: lấy một phần tử, xử lý, lấy tiếp, cho tới khi hàng đợi rỗng. Log mẫu của một Pod duy nhất
   in `Working on banana`, `Working on date`, `Working on lemon`, và bài kết luận thẳng: "một trong
   các pod của Job này đã xử lý nhiều đơn vị công việc". Trực giác "một phần tử một Pod" là mẫu khác
   — mẫu khai triển template, không phải mẫu hàng đợi.
3. Nhờ **tên Service**: `redis` là tên của Service Redis mà bước đầu đã tạo, và Pod phân giải tên đó
   qua DNS của cluster. Đường lui bài đưa sẵn là **biến môi trường** `REDIS_SERVICE_HOST` — hai dòng
   comment `import os` / `host = os.getenv("REDIS_SERVICE_HOST")` trong `worker.py`, và ghi chú
   tương ứng ở mục nạp hàng đợi: nếu Kube DNS chưa được thiết lập đúng thì đổi thành
   `redis-cli -h $REDIS_SERVICE_HOST`.
4. Vì **client cần biết khi nào hàng đợi đã rỗng để tự thoát**, mà theo bài, "AMQP không cung cấp một
   cách tốt để client phát hiện khi nào một hàng đợi công việc có độ dài hữu hạn đã rỗng". Toàn bộ
   thiết kế của bài — `completions` để trống, worker thoát thành công làm tín hiệu — đứng được là nhờ
   worker phát hiện được trạng thái rỗng, nên bài đổi sang Redis cùng một thư viện client tự viết.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

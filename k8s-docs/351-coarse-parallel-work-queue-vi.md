# Xử lý song song thô sử dụng hàng đợi công việc (Coarse Parallel Processing Using a Work Queue)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/job/coarse-parallel-processing-work-queue/

Trong ví dụ này, bạn sẽ chạy một Job Kubernetes với nhiều tiến trình worker song song.

Trong ví dụ này, khi mỗi pod được tạo ra, nó lấy một đơn vị công việc
từ một hàng đợi tác vụ (task queue), hoàn thành công việc đó, xóa nó khỏi hàng đợi rồi thoát.

Dưới đây là tổng quan các bước trong ví dụ này:

1. **Khởi động một dịch vụ hàng đợi thông điệp (message queue).** Trong ví dụ này, bạn dùng RabbitMQ,
   nhưng bạn có thể dùng dịch vụ khác. Trong thực tế, bạn sẽ dựng dịch vụ hàng đợi thông điệp một lần
   và tái sử dụng nó cho nhiều job.
1. **Tạo một hàng đợi và nạp thông điệp vào đó.** Mỗi thông điệp đại diện cho một tác vụ cần thực hiện.
   Trong ví dụ này, một thông điệp là một số nguyên mà chúng ta sẽ thực hiện một phép tính tốn thời gian trên đó.
1. **Khởi động một Job xử lý các tác vụ từ hàng đợi.** Job khởi động vài pod. Mỗi pod lấy
   một tác vụ từ hàng đợi thông điệp, xử lý nó rồi thoát.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần đã quen với cách dùng cơ bản, không song song, của
[Job](67-job-vi.md).

Bạn cần có một cluster Kubernetes và công cụ dòng lệnh kubectl phải được cấu hình
để giao tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất
hai node không đóng vai trò host của control plane. Nếu bạn chưa có cluster, bạn có thể
tạo một cluster bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
hoặc dùng một trong các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Bạn sẽ cần một container image registry, nơi bạn có thể tải image lên để chạy trong cluster của mình.

Ví dụ này cũng giả định rằng bạn đã cài Docker trên máy cục bộ.

## Khởi động dịch vụ hàng đợi thông điệp (Starting a message queue service)

Ví dụ này dùng RabbitMQ, tuy nhiên bạn có thể điều chỉnh ví dụ để dùng một dịch vụ
thông điệp kiểu AMQP khác.

Trong thực tế, bạn có thể dựng dịch vụ hàng đợi thông điệp một lần trong cluster
và tái sử dụng nó cho nhiều job, cũng như cho các dịch vụ chạy dài hạn.

Khởi động RabbitMQ như sau:

```shell
# tạo một Service cho StatefulSet sử dụng
kubectl create -f https://kubernetes.io/examples/application/job/rabbitmq/rabbitmq-service.yaml
```
```
service "rabbitmq-service" created
```

```shell
kubectl create -f https://kubernetes.io/examples/application/job/rabbitmq/rabbitmq-statefulset.yaml
```
```
statefulset "rabbitmq" created
```

## Kiểm tra dịch vụ hàng đợi thông điệp (Testing the message queue service)

Bây giờ, chúng ta có thể thử nghiệm việc truy cập hàng đợi thông điệp. Chúng ta sẽ
tạo một pod tương tác tạm thời, cài một số công cụ lên nó,
và thử nghiệm với các hàng đợi.

Đầu tiên, tạo một Pod tương tác tạm thời.

```shell
# Tạo một container tương tác tạm thời
kubectl run -i --tty temp --image ubuntu:22.04
```
```
Waiting for pod default/temp-loe07 to be running, status is Pending, pod ready: false
... [ previous line repeats several times .. hit return when it stops ] ...
```

Lưu ý rằng tên pod và dấu nhắc lệnh của bạn sẽ khác.

Tiếp theo, cài `amqp-tools` để bạn có thể làm việc với các hàng đợi thông điệp.
Các lệnh tiếp theo cho thấy những gì bạn cần chạy bên trong shell tương tác của Pod đó:

```shell
apt-get update && apt-get install -y curl ca-certificates amqp-tools python3 dnsutils
```

Sau này, bạn sẽ tạo một container image bao gồm các gói này.

Tiếp theo, bạn sẽ kiểm tra rằng bạn có thể phát hiện (discover) Service của RabbitMQ:

```
# Chạy các lệnh này bên trong Pod
# Lưu ý rằng rabbitmq-service có một tên DNS, do Kubernetes cung cấp:
nslookup rabbitmq-service
```
```
Server:        10.0.0.10
Address:    10.0.0.10#53

Name:    rabbitmq-service.default.svc.cluster.local
Address: 10.0.147.152
```
(các địa chỉ IP sẽ khác nhau)

Nếu addon kube-dns không được thiết lập đúng, bước trước đó có thể không hoạt động với bạn.
Bạn cũng có thể tìm địa chỉ IP của Service đó trong một biến môi trường:

```shell
# chạy kiểm tra này bên trong Pod
env | grep RABBITMQ_SERVICE | grep HOST
```
```
RABBITMQ_SERVICE_SERVICE_HOST=10.0.147.152
```
(địa chỉ IP sẽ khác nhau)

Tiếp theo, bạn sẽ xác minh rằng bạn có thể tạo một hàng đợi, cũng như phát hành (publish)
và tiêu thụ (consume) thông điệp.

```shell
# Chạy các lệnh này bên trong Pod
# Ở dòng tiếp theo, rabbitmq-service là hostname mà qua đó có thể truy cập
# rabbitmq-service. 5672 là port chuẩn của rabbitmq.
export BROKER_URL=amqp://guest:guest@rabbitmq-service:5672
# Nếu bạn không phân giải được "rabbitmq-service" ở bước trước,
# thì dùng lệnh này thay thế:
BROKER_URL=amqp://guest:guest@$RABBITMQ_SERVICE_SERVICE_HOST:5672

# Bây giờ tạo một hàng đợi:

/usr/bin/amqp-declare-queue --url=$BROKER_URL -q foo -d
```
```
foo
```

Phát hành một thông điệp vào hàng đợi:
```shell
/usr/bin/amqp-publish --url=$BROKER_URL -r foo -p -b Hello

# Và lấy nó về.

/usr/bin/amqp-consume --url=$BROKER_URL -q foo -c 1 cat && echo 1>&2
```
```
Hello
```

Ở lệnh cuối cùng, công cụ `amqp-consume` lấy một thông điệp (`-c 1`)
từ hàng đợi, và truyền thông điệp đó vào đầu vào chuẩn (standard input) của một lệnh tùy ý.
Trong trường hợp này, chương trình `cat` in ra các ký tự đọc được từ đầu vào chuẩn, và
lệnh echo thêm một ký tự xuống dòng để ví dụ dễ đọc.

## Nạp tác vụ vào hàng đợi (Fill the queue with tasks)

Bây giờ, hãy nạp vào hàng đợi một số tác vụ mô phỏng. Trong ví dụ này, các tác vụ là các chuỗi
cần được in ra.

Trong thực tế, nội dung của các thông điệp có thể là:

- tên các file cần được xử lý
- các cờ (flag) bổ sung cho chương trình
- các khoảng khóa (range of keys) trong một bảng cơ sở dữ liệu
- các tham số cấu hình cho một mô phỏng
- số thứ tự các khung hình (frame) của một cảnh cần render

Nếu có dữ liệu lớn mà tất cả các pod của Job cần đọc ở chế độ chỉ đọc (read-only),
thông thường bạn sẽ đặt dữ liệu đó trong một hệ thống file chia sẻ như NFS và mount
nó ở chế độ chỉ đọc trên tất cả các pod, hoặc viết chương trình trong pod sao cho nó có thể
đọc dữ liệu trực tiếp từ một hệ thống file cluster (ví dụ: HDFS).

Với ví dụ này, bạn sẽ tạo hàng đợi và nạp nó bằng các công cụ dòng lệnh AMQP.
Trong thực tế, bạn có thể viết một chương trình để nạp hàng đợi bằng một thư viện client AMQP.

```shell
# Chạy lệnh này trên máy của bạn, không phải trong Pod
/usr/bin/amqp-declare-queue --url=$BROKER_URL -q job1  -d
```
```
job1
```
Thêm các phần tử vào hàng đợi:
```shell
for f in apple banana cherry date fig grape lemon melon
do
  /usr/bin/amqp-publish --url=$BROKER_URL -r job1 -p -b $f
done
```

Bạn đã thêm 8 thông điệp vào hàng đợi.

## Tạo container image (Create a container image)

Bây giờ bạn đã sẵn sàng tạo một image mà bạn sẽ chạy dưới dạng một Job.

Job sẽ dùng tiện ích `amqp-consume` để đọc thông điệp
từ hàng đợi và chạy công việc thực sự. Dưới đây là một chương trình
ví dụ rất đơn giản:

```python
#!/usr/bin/env python

# Chỉ in ra đầu ra chuẩn và ngủ 10 giây.
import sys
import time
print("Processing " + sys.stdin.readlines()[0])
time.sleep(10)
```

Cấp quyền thực thi cho script:

```shell
chmod +x worker.py
```

Bây giờ, build một image. Tạo một thư mục tạm, chuyển vào đó,
tải [Dockerfile](https://kubernetes.io/examples/application/job/rabbitmq/Dockerfile)
và [worker.py](https://kubernetes.io/examples/application/job/rabbitmq/worker.py). Trong cả hai trường hợp,
build image bằng lệnh này:

```shell
docker build -t job-wq-1 .
```

Với [Docker Hub](https://hub.docker.com/), gắn tag cho image ứng dụng của bạn với
username của bạn và đẩy (push) lên Hub bằng các lệnh dưới đây. Thay
`<username>` bằng username Hub của bạn.

```shell
docker tag job-wq-1 <username>/job-wq-1
docker push <username>/job-wq-1
```

Nếu bạn đang dùng một container image registry khác, hãy gắn tag cho
image và đẩy nó lên đó thay thế.

## Định nghĩa một Job (Defining a Job)

Dưới đây là manifest cho một Job. Bạn sẽ cần tạo một bản sao của manifest Job này
(đặt tên là `./job.yaml`),
và sửa tên của container image cho khớp với tên bạn đã dùng.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-wq-1
spec:
  completions: 8
  parallelism: 2
  template:
    metadata:
      name: job-wq-1
    spec:
      containers:
      - name: c
        image: gcr.io/<project>/job-wq-1
        env:
        - name: BROKER_URL
          value: amqp://guest:guest@rabbitmq-service:5672
        - name: QUEUE
          value: job1
      restartPolicy: OnFailure
```

Trong ví dụ này, mỗi pod xử lý một phần tử từ hàng đợi rồi thoát.
Vì vậy, số lần hoàn thành (completion count) của Job tương ứng với số phần tử công việc
đã được thực hiện. Đó là lý do vì sao manifest ví dụ đặt `.spec.completions` là `8`.

## Chạy Job (Running the Job)

Bây giờ, chạy Job:

```shell
# lệnh này giả định bạn đã tải manifest về và đã chỉnh sửa nó
kubectl apply -f ./job.yaml
```

Bạn có thể chờ Job thành công, với một khoảng thời gian chờ tối đa (timeout):
```shell
# Việc kiểm tra tên condition không phân biệt chữ hoa chữ thường
kubectl wait --for=condition=complete --timeout=300s job/job-wq-1
```

Tiếp theo, kiểm tra Job:

```shell
kubectl describe jobs/job-wq-1
```
```
Name:             job-wq-1
Namespace:        default
Selector:         controller-uid=41d75705-92df-11e7-b85e-fa163ee3c11f
Labels:           controller-uid=41d75705-92df-11e7-b85e-fa163ee3c11f
                  job-name=job-wq-1
Annotations:      <none>
Parallelism:      2
Completions:      8
Start Time:       Wed, 06 Sep 2022 16:42:02 +0000
Pods Statuses:    0 Running / 8 Succeeded / 0 Failed
Pod Template:
  Labels:       controller-uid=41d75705-92df-11e7-b85e-fa163ee3c11f
                job-name=job-wq-1
  Containers:
   c:
    Image:      container-registry.example/causal-jigsaw-637/job-wq-1
    Port:
    Environment:
      BROKER_URL:       amqp://guest:guest@rabbitmq-service:5672
      QUEUE:            job1
    Mounts:             <none>
  Volumes:              <none>
Events:
  FirstSeen  LastSeen   Count    From    SubobjectPath    Type      Reason              Message
  ─────────  ────────   ─────    ────    ─────────────    ──────    ──────              ───────
  27s        27s        1        {job }                   Normal    SuccessfulCreate    Created pod: job-wq-1-hcobb
  27s        27s        1        {job }                   Normal    SuccessfulCreate    Created pod: job-wq-1-weytj
  27s        27s        1        {job }                   Normal    SuccessfulCreate    Created pod: job-wq-1-qaam5
  27s        27s        1        {job }                   Normal    SuccessfulCreate    Created pod: job-wq-1-b67sr
  26s        26s        1        {job }                   Normal    SuccessfulCreate    Created pod: job-wq-1-xe5hj
  15s        15s        1        {job }                   Normal    SuccessfulCreate    Created pod: job-wq-1-w2zqe
  14s        14s        1        {job }                   Normal    SuccessfulCreate    Created pod: job-wq-1-d6ppa
  14s        14s        1        {job }                   Normal    SuccessfulCreate    Created pod: job-wq-1-p17e0
```

Tất cả các pod của Job đó đã thành công! Bạn đã hoàn tất.

## Các cách thay thế (Alternatives)

Cách tiếp cận này có ưu điểm là bạn không cần sửa chương trình "worker" của mình để nó
biết về sự tồn tại của hàng đợi công việc. Bạn có thể đưa chương trình worker vào container
image mà không cần chỉnh sửa gì.

Việc dùng cách tiếp cận này đòi hỏi bạn phải chạy một dịch vụ hàng đợi thông điệp.
Nếu việc chạy một dịch vụ hàng đợi là bất tiện, bạn có thể
cân nhắc một trong các [mẫu job (job patterns)](67-job-vi.md#job-patterns) khác.

Cách tiếp cận này tạo một pod cho mỗi phần tử công việc. Tuy nhiên, nếu các phần tử công việc
của bạn chỉ mất vài giây, việc tạo một Pod cho mỗi phần tử công việc có thể tạo ra nhiều chi phí phụ trội (overhead).
Hãy cân nhắc một thiết kế khác, chẳng hạn như trong [ví dụ hàng đợi công việc song song mịn](352-fine-parallel-work-queue-vi.md),
trong đó mỗi Pod thực thi nhiều phần tử công việc.

Trong ví dụ này, bạn đã dùng tiện ích `amqp-consume` để đọc thông điệp
từ hàng đợi và chạy chương trình thực sự. Cách này có ưu điểm là bạn
không cần sửa chương trình của mình để nó biết về hàng đợi.
[Ví dụ hàng đợi công việc song song mịn](352-fine-parallel-work-queue-vi.md)
trình bày cách giao tiếp với hàng đợi công việc bằng một thư viện client.

## Lưu ý (Caveats)

Nếu số lần hoàn thành (completions) được đặt nhỏ hơn số phần tử trong hàng đợi, thì
không phải tất cả các phần tử sẽ được xử lý.

Nếu số lần hoàn thành được đặt lớn hơn số phần tử trong hàng đợi,
thì Job sẽ không có vẻ như đã hoàn thành, mặc dù tất cả các phần tử trong hàng đợi
đã được xử lý. Nó sẽ khởi động thêm các pod, và các pod này sẽ bị chặn (block) chờ
một thông điệp.
Bạn sẽ cần tự xây dựng cơ chế riêng để phát hiện khi nào có công việc
cần làm và đo kích thước của hàng đợi, rồi đặt số lần hoàn thành cho khớp.

Có một tình huống tranh chấp (race) hiếm gặp với mẫu này. Nếu container bị kill trong khoảng
thời gian giữa lúc thông điệp được xác nhận (acknowledge) bởi lệnh `amqp-consume` và lúc container
thoát với trạng thái thành công, hoặc nếu node gặp sự cố trước khi kubelet kịp báo cáo trạng thái thành công của pod
về API server, thì Job sẽ không có vẻ như đã hoàn thành, mặc dù tất cả các phần tử
trong hàng đợi đã được xử lý.

# Indexed Job để xử lý song song với phân công việc tĩnh (Indexed Job for Parallel Processing with Static Work Assignment)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/job/indexed-parallel-processing-static/

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.24 [stable]`

Trong ví dụ này, bạn sẽ chạy một Job Kubernetes sử dụng nhiều tiến trình worker song song.
Mỗi worker là một container khác nhau chạy trong Pod riêng của nó. Các Pod có một
_chỉ số (index number)_ được control plane tự động gán, cho phép mỗi Pod
xác định phần nào của tổng thể công việc mà nó cần xử lý.

Chỉ số của Pod có sẵn trong annotation
`batch.kubernetes.io/job-completion-index` dưới dạng một chuỗi biểu diễn
giá trị thập phân của nó. Để tiến trình tác vụ trong container lấy được chỉ số này,
bạn có thể công bố giá trị của annotation bằng cơ chế
[downward API](56-downward-api-vi.md).
Để thuận tiện, control plane tự động thiết lập downward API để
đưa chỉ số này vào biến môi trường `JOB_COMPLETION_INDEX`.

Dưới đây là tổng quan các bước trong ví dụ này:

1. **Định nghĩa một manifest Job sử dụng chế độ hoàn thành theo chỉ số (indexed completion)**.
   Downward API cho phép bạn truyền annotation chứa chỉ số của Pod dưới dạng
   biến môi trường hoặc file vào container.
2. **Khởi chạy một Job kiểu `Indexed` dựa trên manifest đó**.

## Trước khi bạn bắt đầu (Before you begin)

Bạn nên đã quen với cách sử dụng cơ bản, không song song, của
[Job](67-job-vi.md).

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

## Chọn cách tiếp cận (Choose an approach)

Để chương trình worker truy cập được mục công việc (work item), bạn có một vài lựa chọn:

1. Đọc biến môi trường `JOB_COMPLETION_INDEX`. Controller
   của Job tự động liên kết biến này với annotation chứa chỉ số hoàn thành
   (completion index).
1. Đọc một file chứa chỉ số hoàn thành.
1. Giả sử bạn không thể sửa đổi chương trình, bạn có thể bọc nó bằng một script
   đọc chỉ số theo một trong các cách trên và chuyển đổi nó thành thứ mà
   chương trình có thể dùng làm đầu vào.

Với ví dụ này, hãy hình dung rằng bạn chọn phương án 3 và bạn muốn chạy
tiện ích [rev](https://man7.org/linux/man-pages/man1/rev.1.html). Chương trình
này nhận một file làm đối số và in nội dung của file theo thứ tự đảo ngược.

```shell
rev data.txt
```

Bạn sẽ dùng công cụ `rev` từ container image
[`busybox`](https://hub.docker.com/_/busybox).

Vì đây chỉ là ví dụ, mỗi Pod chỉ làm một phần việc rất nhỏ (đảo ngược một chuỗi
ngắn). Trong một workload thực tế, chẳng hạn bạn có thể tạo một Job đại diện cho
tác vụ tạo ra 60 giây video dựa trên dữ liệu cảnh quay.
Mỗi mục công việc trong Job render video đó sẽ là render một khung hình (frame)
cụ thể của đoạn video. Chế độ hoàn thành theo chỉ số nghĩa là mỗi Pod trong
Job biết cần render và xuất bản khung hình nào, bằng cách đếm số khung hình từ
đầu đoạn video.

## Định nghĩa một Indexed Job (Define an Indexed Job)

Dưới đây là một manifest Job mẫu sử dụng chế độ hoàn thành `Indexed`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: 'indexed-job'
spec:
  completions: 5
  parallelism: 3
  completionMode: Indexed
  template:
    spec:
      restartPolicy: Never
      initContainers:
      - name: 'input'
        image: 'docker.io/library/bash'
        command:
        - "bash"
        - "-c"
        - |
          items=(foo bar baz qux xyz)
          echo ${items[$JOB_COMPLETION_INDEX]} > /input/data.txt
        volumeMounts:
        - mountPath: /input
          name: input
      containers:
      - name: 'worker'
        image: 'docker.io/library/busybox'
        command:
        - "rev"
        - "/input/data.txt"
        volumeMounts:
        - mountPath: /input
          name: input
      volumes:
      - name: input
        emptyDir: {}
```

Trong ví dụ trên, bạn dùng biến môi trường có sẵn `JOB_COMPLETION_INDEX`
được Job controller thiết lập cho mọi container. Một [init container](50-init-containers-vi.md)
ánh xạ chỉ số này sang một giá trị tĩnh và ghi nó vào một file được chia sẻ với
container chạy worker thông qua một [volume emptyDir](91-volumes-vi.md#emptydir).
Tùy chọn, bạn có thể [tự định nghĩa biến môi trường của riêng mình thông qua
downward API](336-env-variable-expose-pod-info-vi.md)
để công bố chỉ số cho các container. Bạn cũng có thể chọn nạp một danh sách giá trị
từ [ConfigMap dưới dạng biến môi trường hoặc file](275-configure-pod-configmap-vi.md).

Cách khác, bạn có thể trực tiếp [dùng downward API để truyền giá trị annotation
dưới dạng file trong volume](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information#store-pod-fields),
như trong ví dụ sau:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: 'indexed-job'
spec:
  completions: 5
  parallelism: 3
  completionMode: Indexed
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: 'worker'
        image: 'docker.io/library/busybox'
        command:
        - "rev"
        - "/input/data.txt"
        volumeMounts:
        - mountPath: /input
          name: input
      volumes:
      - name: input
        downwardAPI:
          items:
          - path: "data.txt"
            fieldRef:
              fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
```

## Chạy Job (Running the Job)

Bây giờ hãy chạy Job:

```shell
# Lệnh này dùng cách tiếp cận thứ nhất (dựa vào $JOB_COMPLETION_INDEX)
kubectl apply -f https://kubernetes.io/examples/application/job/indexed-job.yaml
```

Khi bạn tạo Job này, control plane tạo ra một loạt Pod, mỗi Pod cho một chỉ số mà bạn đã chỉ định. Giá trị của `.spec.parallelism` quyết định số Pod có thể chạy cùng lúc, trong khi `.spec.completions` quyết định tổng số Pod mà Job tạo ra.

Vì `.spec.parallelism` nhỏ hơn `.spec.completions`, control plane sẽ đợi một số Pod đầu tiên hoàn thành trước khi khởi chạy thêm các Pod khác.

Bạn có thể đợi Job thành công, với một khoảng thời gian chờ (timeout):

```shell
# Việc kiểm tra tên condition không phân biệt chữ hoa chữ thường
kubectl wait --for=condition=complete --timeout=300s job/indexed-job
```

Bây giờ, hãy describe Job và kiểm tra rằng nó đã thành công.

```shell
kubectl describe jobs/indexed-job
```

Kết quả tương tự như sau:

```
Name:              indexed-job
Namespace:         default
Selector:          controller-uid=bf865e04-0b67-483b-9a90-74cfc4c3e756
Labels:            controller-uid=bf865e04-0b67-483b-9a90-74cfc4c3e756
                   job-name=indexed-job
Annotations:       <none>
Parallelism:       3
Completions:       5
Start Time:        Thu, 11 Mar 2021 15:47:34 +0000
Pods Statuses:     2 Running / 3 Succeeded / 0 Failed
Completed Indexes: 0-2
Pod Template:
  Labels:  controller-uid=bf865e04-0b67-483b-9a90-74cfc4c3e756
           job-name=indexed-job
  Init Containers:
   input:
    Image:      docker.io/library/bash
    Port:       <none>
    Host Port:  <none>
    Command:
      bash
      -c
      items=(foo bar baz qux xyz)
      echo ${items[$JOB_COMPLETION_INDEX]} > /input/data.txt

    Environment:  <none>
    Mounts:
      /input from input (rw)
  Containers:
   worker:
    Image:      docker.io/library/busybox
    Port:       <none>
    Host Port:  <none>
    Command:
      rev
      /input/data.txt
    Environment:  <none>
    Mounts:
      /input from input (rw)
  Volumes:
   input:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:  <unset>
Events:
  Type    Reason            Age   From            Message
  ----    ------            ----  ----            -------
  Normal  SuccessfulCreate  4s    job-controller  Created pod: indexed-job-njkjj
  Normal  SuccessfulCreate  4s    job-controller  Created pod: indexed-job-9kd4h
  Normal  SuccessfulCreate  4s    job-controller  Created pod: indexed-job-qjwsz
  Normal  SuccessfulCreate  1s    job-controller  Created pod: indexed-job-fdhq5
  Normal  SuccessfulCreate  1s    job-controller  Created pod: indexed-job-ncslj
```

Trong ví dụ này, bạn chạy Job với các giá trị tùy biến cho từng chỉ số. Bạn có thể
xem output của một trong các Pod:

```shell
kubectl logs indexed-job-fdhq5 # Thay tên này bằng tên của một Pod thuộc Job đó
```

Kết quả tương tự như sau:

```
xuq
```

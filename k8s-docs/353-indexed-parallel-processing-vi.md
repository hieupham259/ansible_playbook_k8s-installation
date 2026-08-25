# Indexed Job để xử lý song song với phân công việc tĩnh (Indexed Job for Parallel Processing with Static Work Assignment)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/job/indexed-parallel-processing-static/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 4 — Workload controller](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller)
→ [4b. StatefulSet, DaemonSet, Job và autoscaling](00-ALO-TRINH-ADMIN.md#4b-statefulset-daemonset-job-và-autoscaling),
bài 6/7 của dòng **Thực hành** · Kiểm chứng ở
[Lab 4b — StatefulSet, DaemonSet và Job](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md),
**phần B6.3**.

Đây là mẫu song song thứ hai của nhóm 4b, và là mẫu **duy nhất chạy trọn vẹn được** trên cluster
lab: nó không cần hàng đợi, không cần Service, không cần image tự build. Đọc nó cạnh bài
[351](351-coarse-parallel-work-queue-vi.md) để thấy ranh giới giữa hai cách phân công việc.

**Phải hiểu ở lần đọc này:**

- Ý tưởng của mẫu: control plane **tự gán cho mỗi Pod một chỉ số**, và Pod dựa vào chỉ số đó để
  biết phần nào của tổng thể công việc là của mình. Đây là **phân công việc tĩnh** — không có hàng
  đợi nào để lấy việc.
- Chỉ số nằm ở annotation `batch.kubernetes.io/job-completion-index` dưới dạng chuỗi. Control plane
  **tự thiết lập downward API** để đưa nó vào biến môi trường `JOB_COMPLETION_INDEX` cho mọi
  container, nên bạn không phải cấu hình gì để dùng biến này.
- Mục *Chọn cách tiếp cận* nêu ba đường để chương trình worker lấy được chỉ số: đọc biến môi
  trường `JOB_COMPLETION_INDEX`, đọc một file chứa chỉ số, hoặc **bọc chương trình bằng một
  script** khi bạn không sửa được chương trình. Ví dụ trong bài chọn đường thứ ba.
- Ba trường quyết định hình dạng Job: `completionMode: Indexed` bật chế độ này, `.spec.completions`
  quyết định **tổng số Pod** Job tạo ra, `.spec.parallelism` quyết định **số Pod chạy cùng lúc**.
  Vì `parallelism` nhỏ hơn `completions`, control plane **đợi một số Pod đầu hoàn thành rồi mới
  khởi chạy thêm** (mục *Chạy Job*).
- Cách hiện thực trong manifest đầu: một init container đọc `$JOB_COMPLETION_INDEX`, ánh xạ sang
  một giá trị tĩnh và ghi vào file trên volume `emptyDir` chia sẻ với container worker. Kết quả đọc
  bằng `kubectl describe jobs/indexed-job`, ở dòng `Completed Indexes`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Manifest thứ hai dùng volume `downwardAPI` với `fieldRef` trỏ thẳng vào annotation, cùng các link sang bài [336](336-env-variable-expose-pod-info-vi.md) và [275](275-configure-pod-configmap-vi.md) | là các cơ chế nạp dữ liệu vào Pod, không phải cơ chế của Job; một cách hiện thực là đủ cho lần đọc này | bài [336](336-env-variable-expose-pod-info-vi.md) ở [3a. Pod và vòng đời](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời) và bài [275](275-configure-pod-configmap-vi.md) ở [3b. Cấu hình ứng dụng](00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod) |
| Nhãn trạng thái tính năng `Kubernetes v1.24 [stable]` và yêu cầu server từ v1.21 trở lên | phiên bản khóa của cluster lab mới hơn nhiều nên điều kiện luôn thỏa | [bảng A1.3 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) |
| Mục *Trước khi bạn bắt đầu* — minikube và các playground | cluster lab ba VM đã dựng sẵn từ trước | [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |

---

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

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở nhóm 4b:

1. So với mẫu hàng đợi công việc ở bài [351](351-coarse-parallel-work-queue-vi.md), mẫu này khác ở
   chỗ nào về cách một Pod biết phần việc của mình? Bài gọi tên cách phân công đó là gì?
2. Chỉ số của Pod được lưu ở đâu trên object, và bằng đường nào nó tới được tiến trình bên trong
   container mà bạn không phải tự cấu hình gì?
3. **Câu bẫy.** Job có `completions: 5` và `parallelism: 3`. Job tạo ra tất cả bao nhiêu Pod, bao
   nhiêu Pod chạy cùng lúc, và vì sao control plane không tạo cả 5 Pod ngay từ đầu?
4. Bạn chạy Indexed Job trên cluster lab và muốn biết những chỉ số nào đã hoàn thành. Lệnh nào, và
   dòng nào trong output cho bạn biết điều đó?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Ở mẫu hàng đợi, Pod **lấy việc từ hàng đợi** lúc chạy; ở đây **control plane gán sẵn cho mỗi Pod
   một chỉ số**, và Pod suy ra phần việc của mình từ chỉ số đó. Bài gọi đây là **phân công việc
   tĩnh (static work assignment)**, và hệ quả là **không cần dịch vụ hàng đợi nào cả**.
2. Chỉ số nằm ở **annotation `batch.kubernetes.io/job-completion-index`** của Pod, dưới dạng chuỗi
   biểu diễn giá trị thập phân. Đường tới container: **control plane tự động thiết lập downward
   API** để công bố giá trị đó thành **biến môi trường `JOB_COMPLETION_INDEX`** cho mọi container —
   bài nói rõ đây là việc control plane làm sẵn cho thuận tiện, bạn không phải khai báo gì.
3. **5 Pod tất cả, 3 Pod chạy cùng lúc.** Đây là chỗ dễ nhầm hai trường với nhau:
   **`.spec.completions` quyết định tổng số Pod mà Job tạo ra**, còn **`.spec.parallelism` quyết
   định số Pod có thể chạy cùng lúc**. Vì `parallelism` **nhỏ hơn** `completions`, control plane
   **đợi một số Pod đầu tiên hoàn thành trước rồi mới khởi chạy thêm** — không phải vì cluster
   thiếu chỗ.
4. **`kubectl describe jobs/indexed-job`**, và dòng cần đọc là **`Completed Indexes`** — trong ví
   dụ của bài nó hiện `0-2` khi mới có ba chỉ số đầu xong. Muốn xem kết quả thật thì
   `kubectl logs <tên Pod>` của một Pod thuộc Job đó.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

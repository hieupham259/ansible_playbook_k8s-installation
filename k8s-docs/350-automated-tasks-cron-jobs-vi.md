# Chạy các tác vụ tự động với CronJob (Running Automated Tasks with a CronJob)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 4 — Workload controller](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller)
→ [4b. StatefulSet, DaemonSet, Job và autoscaling](00-ALO-TRINH-ADMIN.md#4b-statefulset-daemonset-job-và-autoscaling),
bài 4/7 của dòng **Thực hành** · Kiểm chứng ở
[Lab 4b — StatefulSet, DaemonSet và Job](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md),
**phần B9.1** (tạo CronJob, đọc Job và Pod nó sinh ra) và **B9.5** (xóa CronJob, quan sát Job
biến mất theo).

Bài này chỉ đi qua vòng đời cơ bản của một CronJob. Các núm điều khiển thật sự khó —
`concurrencyPolicy`, `startingDeadlineSeconds`, `suspend` — nằm ở bài khái niệm
[69](69-cron-jobs-vi.md) của cùng nhóm, không ở đây.

**Phải hiểu ở lần đọc này:**

- Một CronJob cần manifest với hai phần: `schedule` viết theo cú pháp cron, và `jobTemplate` mô tả
  Job sẽ được tạo. Manifest ví dụ chạy mỗi phút một lần và đặt `restartPolicy: OnFailure`
  (mục *Tạo một CronJob*).
- Chuỗi ba tầng **CronJob → Job → Pod**: CronJob chỉ lập lịch và tạo Job; Job mới là thứ tạo Pod.
  Bạn thấy tầng giữa bằng `kubectl get jobs --watch`.
- Đọc trạng thái lịch bằng `kubectl get cronjob <tên>`: cột `LAST SCHEDULE` là thời điểm lập lịch
  gần nhất, `ACTIVE` bằng 0 nghĩa là Job đã hoàn thành hoặc đã thất bại, và `SUSPEND` cho biết
  lịch có đang bị treo không.
- Bài nhắc rõ **tên Job khác tên Pod**, nên muốn xem output một lần chạy thì phải đi vòng: lấy Pod
  theo selector `--selector=job-name=<tên job>` rồi `kubectl logs`.
- Xóa CronJob bằng `kubectl delete cronjob <tên>` **xóa luôn mọi Job và Pod nó đã tạo**, đồng thời
  ngăn nó tạo Job mới (mục *Xóa một CronJob*).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Trước khi bạn bắt đầu* — minikube và các playground | cluster lab ba VM đã dựng sẵn từ trước | [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |
| Link *thu gom rác* — cơ chế đứng sau việc xóa CronJob kéo theo Job và Pod | ở đây chỉ cần biết kết quả, chưa cần quan hệ owner và dependent | bài [36](36-garbage-collection-vi.md) đã đọc ở [1c. Vòng đời và cơ chế nền của object](00-ALO-TRINH-ADMIN.md#1c-vòng-đời-và-cơ-chế-nền-của-object) |
| Việc lấy manifest ví dụ thẳng từ `k8s.io/examples` bằng `kubectl create -f <URL>` | lab tự viết manifest vào `~/lab-work/4b/` để sửa và đọc lại được | [Lab 4b](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) phần B9.1 |

---

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

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở nhóm 4b:

1. Một CronJob đang chạy sinh ra những object nào, và ai tạo ai trong chuỗi đó? Lệnh nào cho bạn
   nhìn thấy tầng ở giữa?
2. **Câu bẫy.** `kubectl get cronjob hello` báo `ACTIVE 0` nhưng `LAST SCHEDULE 50s`. CronJob của
   bạn có đang hỏng không? Hai cột đó nói lên điều gì?
3. Trên `lab-k8s-master`, bạn muốn đọc output của lần chạy gần nhất. Bài nhắc điều gì về tên
   object, và bạn đi từ tên Job tới log của Pod bằng đường nào?
4. Bạn chạy `kubectl delete cronjob hello`. Các Job đã chạy xong và Pod của chúng còn lại không,
   và CronJob còn tạo Job mới nữa không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Chuỗi ba tầng: **CronJob tạo Job theo lịch, và Job mới là thứ tạo Pod**. CronJob không tạo Pod
   trực tiếp. Tầng ở giữa nhìn thấy bằng **`kubectl get jobs --watch`** — bài cho theo dõi khoảng
   một phút và bạn thấy một Job mới xuất hiện rồi chuyển từ `0/1` sang `1/1`.
2. **Không hỏng — đó là trạng thái bình thường.** Đây là chỗ dễ đọc nhầm thành CronJob không chạy
   gì. `LAST SCHEDULE` cho biết **CronJob đã lập lịch thành công một Job tại thời điểm đó**, còn
   `ACTIVE 0` chỉ có nghĩa là **hiện không có Job nào đang hoạt động**, tức Job đó **đã hoàn thành
   hoặc đã thất bại**. Muốn biết là hoàn thành hay thất bại thì phải nhìn sang Job.
3. Bài ghi rõ **tên của job khác với tên của Pod**, nên không thể đoán tên Pod từ tên Job. Đường
   đi: lấy tên Job từ `kubectl get jobs`, rồi lấy Pod bằng selector
   **`kubectl get pods --selector=job-name=<tên job>`** (bài lấy tên ra biến `pods` bằng
   `--output=jsonpath`), sau đó **`kubectl logs $pods`**.
4. **Không còn gì cả.** Bài nói thẳng: xóa cron job **xóa tất cả các job và Pod mà nó đã tạo ra**,
   đồng thời **ngăn nó tạo thêm các job mới**. Cơ chế đứng sau là thu gom rác.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

# Xử lý các lần Pod thất bại có thể thử lại và không thể thử lại bằng Pod failure policy (Handling retriable and non-retriable pod failures with Pod failure policy)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/job/pod-failure-policy/>
>
> Trang này hướng dẫn bạn dùng Pod failure policy, kết hợp với Pod backoff failure policy mặc
> định, để kiểm soát tốt hơn cách xử lý lỗi ở mức container hoặc mức Pod bên trong một Job.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 29 — DaemonSet, Job nâng cao và thiết bị chuyên dụng](00-ALO-TRINH-ADMIN.md#giai-đoạn-29--daemonset-job-nâng-cao-và-thiết-bị-chuyên-dụng),
bài 6/8 · Kiểm chứng trực tiếp trên cluster lab: chạy kịch bản 1 (`FailJob` theo exit code) — đây là
**phần thứ hai của Checkpoint giai đoạn 29**: chứng minh Job dừng theo exit code đã khai báo thay vì
thử lại tới `backoffLimit`.

Bài dài nhất của giai đoạn 29 và là bài duy nhất bạn **bắt buộc phải chạy tay** để qua Checkpoint.
Nó nối tiếp bài [67](67-job-vi.md) và phần `backoffLimit` bạn đã thực hành ở
[Lab 4b](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) phần B7 — đọc bài này với đúng một câu hỏi
trong đầu: **`backoffLimit` chỉ đếm số lần thất bại, còn `podFailurePolicy` thì nhìn vào *lý do*
thất bại.** Bốn kịch bản trong bài chỉ là bốn cách khai thác sự khác biệt đó.

Bốn kịch bản không cần chạy hết. Kịch bản 1 là bắt buộc. Kịch bản 2 chạy được nhưng nó **drain một
node thật** bằng đúng bộ lệnh bạn đã học ở bài [255](255-safely-drain-node-vi.md): giữ gián đoạn
trong phạm vi `lab-k8s-worker2`, kiểm `nodeName` của Pod trước khi drain, và đừng quên `uncordon`.
Kịch bản 3 và 4 đọc để hiểu cơ chế là đủ.

**Phải hiểu ở lần đọc này:**

- Một rule của `podFailurePolicy` gồm một `action` cộng **một trong hai bộ so khớp**: `onExitCodes`
  (có `containerName`, `operator`, `values`) hoặc `onPodConditions` (có `type`). Rule được đánh chỉ
  số, và `message` trong status Job chỉ đích danh rule nào khớp — ví dụ `matching FailJob rule at
  index 0`.
- Ba `action` khác nhau ở **phạm vi tác động**: `FailJob` chấm dứt cả Job ngay (kịch bản 1);
  `Ignore` khiến lần thất bại đó **không được tính** vào bộ đếm `.spec.backoffLimit` (kịch bản 2);
  `FailIndex` chỉ đánh hỏng một index và không thử lại index đó — cần đi kèm `completionMode:
  Indexed` và `backoffLimitPerIndex` (kịch bản 4).
- Vì sao chỉ có `backoffLimit` là không đủ, theo mục *Các lựa chọn thay thế* và hai đoạn "để so
  sánh": đặt thấp thì gián đoạn Pod giết Job (kịch bản 2 có `backoffLimit: 0`), đặt cao thì lỗi phần
  mềm chắc chắn hỏng vẫn bị thử lại — kịch bản 1 nếu tắt policy sẽ thử tới 6 lần với backoff theo
  cấp số nhân và mất ít nhất 9 phút.
- Đọc bằng chứng ở đâu: trong `status` của Job có condition `FailureTarget` — Job controller thêm
  **ngay khi Job bị coi là thất bại** — rồi condition `Failed` với cùng `reason`/`message`, thêm
  **sau khi mọi Pod của Job đã bị chấm dứt**. Cả hai mang `reason: PodFailurePolicy`.
- Vì sao kịch bản 3 phải đi đường vòng qua một Pod condition tùy chỉnh: Pod kẹt ở
  `ImagePullBackOff` **vẫn ở phase `Pending`** và chưa hề có exit code nào để `onExitCodes` bắt, nên
  bài phải tự vá condition `ConfigIssue` vào `status` của Pod rồi xóa Pod để nó chuyển sang phase
  kết thúc.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Trước khi bạn bắt đầu* — minikube, ba môi trường thử nghiệm, yêu cầu "v1.25 hoặc mới hơn" | Lộ trình cấm minikube và kind; điều kiện phiên bản thì baseline lab vượt xa | Bỏ hẳn — ba VM và bảng phiên bản khóa ở [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |
| Ghi chú "bước 3 và bước 4 nên được tự động hóa bằng một controller do người dùng cung cấp" ở kịch bản 3 | Tự viết controller là một chủ đề riêng, không phải nội dung của Job | [Giai đoạn 28 — Mở rộng Kubernetes](00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes) và [Lab 14 — CRD và Operator](labs/LAB-14-CRD-VA-OPERATOR.md) |
| Cơ chế Indexed Job và biến `JOB_COMPLETION_INDEX` trong script Python của kịch bản 4 | Ở đây nó chỉ là phương tiện để mỗi index thoát một mã khác nhau | Đã học ở bài [353](353-indexed-parallel-processing-vi.md) và thực hành ở [Lab 4b](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) phần B6.3 |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.31 [stable]`

Tài liệu này hướng dẫn bạn cách dùng
[Pod failure policy](67-job-vi.md#pod-failure-policy),
kết hợp với
[Pod backoff failure policy](67-job-vi.md#pod-backoff-failure-policy)
mặc định, để kiểm soát tốt hơn việc xử lý lỗi ở mức container hoặc mức Pod bên trong một Job.

Việc định nghĩa Pod failure policy có thể giúp bạn:

* tận dụng tài nguyên tính toán tốt hơn nhờ tránh được các lần thử lại Pod không cần thiết.
* tránh việc Job thất bại do các gián đoạn Pod (Pod disruption) — chẳng hạn preemption,
  API-initiated eviction (eviction khởi tạo qua API) hoặc eviction dựa trên taint.

## Trước khi bạn bắt đầu (Before you begin)

Bạn nên đã quen với cách sử dụng cơ bản của [Job](67-job-vi.md).

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Khuyến nghị chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò máy chủ control plane. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các môi trường thử nghiệm Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Máy chủ Kubernetes của bạn phải ở phiên bản bằng hoặc mới hơn v1.25. Để kiểm tra phiên bản,
hãy chạy `kubectl version`.

## Các tình huống sử dụng (Usage scenarios)

Hãy cân nhắc các tình huống sử dụng sau đây cho những Job có định nghĩa Pod failure policy:

- [Tránh các lần thử lại Pod không cần thiết](#pod-failure-policy-failjob)
- [Bỏ qua các gián đoạn Pod](#pod-failure-policy-ignore)
- [Tránh các lần thử lại Pod không cần thiết dựa trên Pod condition tùy chỉnh](#pod-failure-policy-config-issue)
- [Tránh các lần thử lại Pod không cần thiết theo từng index](#backoff-limit-per-index-failindex)

### Dùng Pod failure policy để tránh các lần thử lại Pod không cần thiết (Using Pod failure policy to avoid unnecessary Pod retries) {#pod-failure-policy-failjob}

Qua ví dụ sau, bạn sẽ học cách dùng Pod failure policy để tránh việc khởi động lại Pod một
cách không cần thiết khi lỗi của Pod cho thấy đó là một lỗi phần mềm không thể thử lại được.

1. Hãy xem manifest sau:

   ```yaml
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: job-pod-failure-policy-failjob
   spec:
     completions: 8
     parallelism: 2
     template:
       spec:
         restartPolicy: Never
         containers:
         - name: main
           image: docker.io/library/bash:5
           command: ["bash"]
           args:
           - -c
           - echo "Hello world! I'm going to exit with 42 to simulate a software bug." && sleep 30 && exit 42
     backoffLimit: 6
     podFailurePolicy:
       rules:
       - action: FailJob
         onExitCodes:
           containerName: main
           operator: In
           values: [42]
   ```

1. Áp dụng manifest:

   ```sh
   kubectl create -f https://k8s.io/examples/controllers/job-pod-failure-policy-failjob.yaml
   ```

1. Sau khoảng 30 giây, toàn bộ Job sẽ bị chấm dứt. Hãy kiểm tra trạng thái của Job bằng cách
   chạy:

   ```sh
   kubectl get jobs -l job-name=job-pod-failure-policy-failjob -o yaml
   ```

   Trong status của Job, các condition sau sẽ hiển thị:
   - Condition `FailureTarget`: có trường `reason` được đặt là `PodFailurePolicy` và trường
     `message` chứa thêm thông tin về việc chấm dứt, ví dụ
     `Container main for pod default/job-pod-failure-policy-failjob-8ckj8 failed with exit code 42 matching FailJob rule at index 0`.
     Job controller thêm condition này ngay khi Job được coi là thất bại.
     Để biết chi tiết, xem [Chấm dứt các Pod của Job](67-job-vi.md#termination-of-job-pods).
   - Condition `Failed`: có cùng `reason` và `message` với condition `FailureTarget`. Job
     controller thêm condition này sau khi tất cả các Pod của Job đã bị chấm dứt.

   Để so sánh, nếu Pod failure policy bị tắt, Job sẽ thử lại cho đến khi đạt `backoffLimit`
   (6 lần thất bại). Vì các lần thử lại dùng backoff theo cấp số nhân (exponential backoff)
   và với `parallelism: 2` thì các lần thất bại xảy ra theo cặp, nên độ trễ giữa các lần thử
   tăng dần sau mỗi lần thử lại. Kết quả là ví dụ này sẽ mất ít nhất 9 phút trước khi Job
   thất bại.

#### Dọn dẹp (Clean up)

Xóa Job mà bạn đã tạo:

```sh
kubectl delete jobs/job-pod-failure-policy-failjob
```

Cluster sẽ tự động dọn dẹp các Pod.

### Dùng Pod failure policy để bỏ qua các gián đoạn Pod (Using Pod failure policy to ignore Pod disruptions) {#pod-failure-policy-ignore}

Qua ví dụ sau, bạn sẽ học cách dùng Pod failure policy để các gián đoạn Pod không làm tăng bộ
đếm số lần thử lại Pod tính vào giới hạn `.spec.backoffLimit`.

> **Thận trọng:** Thời điểm thao tác rất quan trọng trong ví dụ này, vì vậy bạn nên đọc hết
> các bước trước khi thực hiện. Để kích hoạt một gián đoạn Pod, điều quan trọng là bạn phải
> drain node trong lúc Pod đang chạy trên node đó (trong vòng 90 giây kể từ khi Pod được lập
> lịch).

1. Hãy xem manifest sau:

   ```yaml
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: job-pod-failure-policy-ignore
   spec:
     completions: 4
     parallelism: 2
     template:
       spec:
         restartPolicy: Never
         containers:
         - name: main
           image: docker.io/library/bash:5
           command: ["bash"]
           args:
           - -c
           - echo "Hello world! I'm going to exit with 0 (success)." && sleep 90 && exit 0
     backoffLimit: 0
     podFailurePolicy:
       rules:
       - action: Ignore
         onPodConditions:
         - type: DisruptionTarget
   ```

1. Áp dụng manifest:

   ```sh
   kubectl create -f https://k8s.io/examples/controllers/job-pod-failure-policy-ignore.yaml
   ```

1. Chạy lệnh sau để kiểm tra `nodeName` mà Pod được lập lịch tới:

   ```sh
   nodeName=$(kubectl get pods -l job-name=job-pod-failure-policy-ignore -o jsonpath='{.items[0].spec.nodeName}')
   ```

1. Drain node để evict Pod trước khi nó hoàn tất (trong vòng 90 giây):

   ```sh
   kubectl drain nodes/$nodeName --ignore-daemonsets --grace-period=0
   ```

1. Kiểm tra `.status.failed` để xác nhận bộ đếm của Job không tăng:

   ```sh
   kubectl get jobs -l job-name=job-pod-failure-policy-ignore -o yaml
   ```

1. Uncordon node:

   ```sh
   kubectl uncordon nodes/$nodeName
   ```

Job tiếp tục chạy và thành công.

Để so sánh, nếu Pod failure policy bị tắt thì gián đoạn Pod sẽ dẫn tới việc chấm dứt toàn bộ
Job (vì `.spec.backoffLimit` được đặt bằng 0).

#### Dọn dẹp (Cleaning up)

Xóa Job mà bạn đã tạo:

```sh
kubectl delete jobs/job-pod-failure-policy-ignore
```

Cluster sẽ tự động dọn dẹp các Pod.

### Dùng Pod failure policy để tránh các lần thử lại Pod không cần thiết dựa trên Pod condition tùy chỉnh (Using Pod failure policy to avoid unnecessary Pod retries based on custom Pod Conditions) {#pod-failure-policy-config-issue}

Qua ví dụ sau, bạn sẽ học cách dùng Pod failure policy để tránh việc khởi động lại Pod một
cách không cần thiết dựa trên các Pod condition tùy chỉnh.

> **Ghi chú:** Ví dụ bên dưới hoạt động từ phiên bản 1.27 trở đi vì nó dựa trên việc các pod
> đã bị xóa, đang ở phase `Pending`, được chuyển sang một phase kết thúc (terminal phase)
> (xem: [Phase của Pod](47-pod-lifecycle-vi.md#pod-phase)).

1. Hãy xem manifest sau:

   ```yaml
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: job-pod-failure-policy-config-issue
   spec:
     completions: 8
     parallelism: 2
     template:
       spec:
         restartPolicy: Never
         containers:
         - name: main
           image: "non-existing-repo/non-existing-image:example"
     backoffLimit: 6
     podFailurePolicy:
       rules:
       - action: FailJob
         onPodConditions:
         - type: ConfigIssue
   ```

1. Áp dụng manifest:

   ```sh
   kubectl create -f https://k8s.io/examples/controllers/job-pod-failure-policy-config-issue.yaml
   ```

   Lưu ý rằng image bị cấu hình sai, vì nó không tồn tại.

1. Kiểm tra trạng thái các Pod của job bằng cách chạy:

   ```sh
   kubectl get pods -l job-name=job-pod-failure-policy-config-issue -o yaml
   ```

   Bạn sẽ thấy kết quả tương tự như sau:

   ```yaml
   containerStatuses:
   - image: non-existing-repo/non-existing-image:example
      ...
      state:
      waiting:
         message: Back-off pulling image "non-existing-repo/non-existing-image:example"
         reason: ImagePullBackOff
         ...
   phase: Pending
   ```

   Lưu ý rằng pod vẫn ở phase `Pending` vì nó không kéo được image bị cấu hình sai. Về nguyên
   tắc, đây có thể là một sự cố tạm thời và image có thể được kéo về thành công. Tuy nhiên,
   trong trường hợp này image không tồn tại, nên chúng ta biểu thị sự thật đó bằng một
   condition tùy chỉnh.

1. Thêm condition tùy chỉnh. Trước tiên, hãy chuẩn bị bản vá (patch) bằng cách chạy:

   ```sh
   cat <<EOF > patch.yaml
   status:
     conditions:
     - type: ConfigIssue
       status: "True"
       reason: "NonExistingImage"
       lastTransitionTime: "$(date -u +"%Y-%m-%dT%H:%M:%SZ")"
   EOF
   ```

   Tiếp theo, chọn một trong các pod do job tạo ra bằng cách chạy:

   ```
   podName=$(kubectl get pods -l job-name=job-pod-failure-policy-config-issue -o jsonpath='{.items[0].metadata.name}')
   ```

   Sau đó, áp dụng bản vá lên một trong các pod bằng cách chạy lệnh sau:

   ```sh
   kubectl patch pod $podName --subresource=status --patch-file=patch.yaml
   ```

   Nếu áp dụng thành công, bạn sẽ nhận được thông báo như sau:

   ```sh
   pod/job-pod-failure-policy-config-issue-k6pvp patched
   ```

1. Xóa pod để chuyển nó sang phase `Failed`, bằng cách chạy lệnh:

   ```sh
   kubectl delete pods/$podName
   ```

1. Kiểm tra trạng thái của Job bằng cách chạy:

   ```sh
   kubectl get jobs -l job-name=job-pod-failure-policy-config-issue -o yaml
   ```

   Trong status của Job, bạn sẽ thấy condition `Failed` của job với trường `reason` bằng
   `PodFailurePolicy`. Ngoài ra, trường `message` chứa thông tin chi tiết hơn về việc chấm dứt
   Job, chẳng hạn như:
   `Pod default/job-pod-failure-policy-config-issue-k6pvp has condition ConfigIssue matching FailJob rule at index 0`.

> **Ghi chú:** Trong môi trường production, bước 3 và bước 4 nên được tự động hóa bằng một
> controller do người dùng cung cấp.

#### Dọn dẹp (Cleaning up)

Xóa Job mà bạn đã tạo:

```sh
kubectl delete jobs/job-pod-failure-policy-config-issue
```

Cluster sẽ tự động dọn dẹp các Pod.

### Dùng Pod failure policy để tránh các lần thử lại Pod không cần thiết theo từng index (Using Pod Failure Policy to avoid unnecessary Pod retries per index) {#backoff-limit-per-index-failindex}

Để tránh việc khởi động lại Pod không cần thiết theo từng index, bạn có thể dùng kết hợp hai
tính năng *Pod failure policy* và *giới hạn backoff theo từng index* (backoff limit per
index). Mục này của trang sẽ hướng dẫn cách dùng hai tính năng đó cùng nhau.

1. Hãy xem manifest sau:

   ```yaml
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: job-backoff-limit-per-index-failindex
   spec:
     completions: 4
     parallelism: 2
     completionMode: Indexed
     backoffLimitPerIndex: 1
     template:
       spec:
         restartPolicy: Never
         containers:
         - name: main
           image: docker.io/library/python:3
           command:
             # Script này:
             # - làm Pod với index 0 thất bại với exit code 1, dẫn tới một lần thử lại;
             # - làm Pod với index 1 thất bại với exit code 42, dẫn tới việc index đó thất bại
             #   mà không thử lại.
             # - làm các Pod với index khác thành công.
             - python3
             - -c
             - |
               import os, sys
               index = int(os.environ.get("JOB_COMPLETION_INDEX"))
               if index == 0:
                 sys.exit(1)
               elif index == 1:
                 sys.exit(42)
               else:
                 sys.exit(0)
     backoffLimit: 6
     podFailurePolicy:
       rules:
       - action: FailIndex
         onExitCodes:
           containerName: main
           operator: In
           values: [42]
   ```

1. Áp dụng manifest:

   ```sh
   kubectl create -f https://k8s.io/examples/controllers/job-backoff-limit-per-index-failindex.yaml
   ```

1. Sau khoảng 15 giây, hãy kiểm tra trạng thái các Pod của Job. Bạn có thể làm điều đó bằng
   cách chạy:

   ```shell
   kubectl get pods -l job-name=job-backoff-limit-per-index-failindex -o yaml
   ```

   Bạn sẽ thấy kết quả tương tự như sau:

   ```none
   NAME                                            READY   STATUS      RESTARTS   AGE
   job-backoff-limit-per-index-failindex-0-4g4cm   0/1     Error       0          4s
   job-backoff-limit-per-index-failindex-0-fkdzq   0/1     Error       0          15s
   job-backoff-limit-per-index-failindex-1-2bgdj   0/1     Error       0          15s
   job-backoff-limit-per-index-failindex-2-vs6lt   0/1     Completed   0          11s
   job-backoff-limit-per-index-failindex-3-s7s47   0/1     Completed   0          6s
   ```

   Lưu ý rằng kết quả trên cho thấy:

   * Có hai Pod mang index 0, vì giới hạn backoff cho phép index đó được thử lại một lần.
   * Chỉ có một Pod mang index 1, vì exit code của Pod thất bại đó khớp với Pod failure policy
   có hành động `FailIndex`.

1. Kiểm tra trạng thái của Job bằng cách chạy:

   ```sh
   kubectl get jobs -l job-name=job-backoff-limit-per-index-failindex -o yaml
   ```

   Trong status của Job, bạn sẽ thấy trường `failedIndexes` hiển thị "0,1", vì cả hai index
   đều thất bại. Vì index 1 không được thử lại nên số Pod thất bại, được biểu thị bởi trường
   status "failed", bằng 3.

#### Dọn dẹp (Cleaning up)

Xóa Job mà bạn đã tạo:

```sh
kubectl delete jobs/job-backoff-limit-per-index-failindex
```

Cluster sẽ tự động dọn dẹp các Pod.

## Các lựa chọn thay thế (Alternatives)

Bạn có thể chỉ dựa vào
[Pod backoff failure policy](67-job-vi.md#pod-backoff-failure-policy),
bằng cách chỉ định trường `.spec.backoffLimit` của Job. Tuy nhiên, trong nhiều tình huống,
việc tìm ra điểm cân bằng là rất khó: đặt `.spec.backoffLimit` đủ thấp để tránh các lần thử
lại Pod không cần thiết, nhưng vẫn phải đủ cao để đảm bảo Job không bị chấm dứt bởi các gián
đoạn Pod.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 29:

1. Bạn chạy `job-pod-failure-policy-failjob` trên cluster lab: `completions: 8`, `parallelism: 2`,
   `backoffLimit: 6`, rule `FailJob` bắt exit code `42`. Hai Pod đầu — mỗi worker một Pod — cùng
   thoát với mã `42`. Job kết thúc sau bao lâu và sau bao nhiêu lần thử? Nếu gỡ `podFailurePolicy`
   ra khỏi manifest thì kết cục khác thế nào?
2. **Câu bẫy.** `job-pod-failure-policy-ignore` đặt `backoffLimit: 0` — tức "một lần thất bại là
   chết". Bạn drain `lab-k8s-worker2` trong lúc Pod của Job đang chạy ở đó. Job có thất bại không?
   Trường nào của status là bằng chứng, và vì sao `backoffLimit: 0` không giết Job ở đây?
3. Kịch bản 3 dùng một image không tồn tại. Vì sao **không thể** dùng `onExitCodes` để bắt tình
   huống này, và bài phải làm hai thao tác nào để rule `FailJob` khớp được?
4. Trong kịch bản 4, `kubectl get pods` cho **hai** Pod mang index 0 nhưng chỉ **một** Pod mang
   index 1. Giải thích cả hai con số. Cuối cùng `failedIndexes` là `"0,1"` còn `failed` bằng 3 —
   con số 3 đến từ đâu?
5. Status của Job có hai condition `FailureTarget` và `Failed` với cùng `reason` và `message`. Vì
   sao cần tới hai cái, và Job controller thêm mỗi cái vào lúc nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Sau khoảng 30 giây và sau đúng lần thất bại đầu tiên** — script `sleep 30` rồi `exit 42`, mã
   `42` khớp rule `FailJob` ở index 0, nên Job controller chấm dứt toàn bộ Job ngay, không đợi
   `completions: 8` và không dùng tới `backoffLimit`. Nếu gỡ `podFailurePolicy` đi thì chỉ còn Pod
   backoff failure policy: Job **thử lại tới 6 lần thất bại**, các lần thử dùng **backoff theo cấp
   số nhân** và với `parallelism: 2` thì thất bại xảy ra theo cặp, nên bài ước lượng **ít nhất 9
   phút** mới tới lúc Job thất bại. Cùng một kết cục, khác nhau ở chỗ lãng phí gần 9 phút tài nguyên
   tính toán cho một lỗi phần mềm chắc chắn không sửa được bằng cách thử lại.
2. **Không** — Job tiếp tục chạy và về sau thành công. Bằng chứng là **`.status.failed` không tăng**.
   Lý do: drain sinh ra một gián đoạn Pod, Pod mang condition `DisruptionTarget`, và rule
   `action: Ignore` với `onPodConditions: [DisruptionTarget]` nói rằng lần thất bại đó **không được
   tính vào bộ đếm** của `.spec.backoffLimit`. Chỗ dễ nhầm là đọc `backoffLimit: 0` thành "cấm mọi
   thất bại": nó chỉ giới hạn **số lần thất bại được đếm**, mà `Ignore` thì làm cho lần này không
   được đếm. Nếu tắt policy đi, đúng như bài nói, gián đoạn Pod sẽ chấm dứt cả Job.
3. Vì Pod **không bao giờ có exit code để mà bắt**: nó kẹt ở `ImagePullBackOff` và **vẫn ở phase
   `Pending`** — container còn chưa chạy. `onExitCodes` chỉ so khớp được với mã thoát của một
   container đã kết thúc. Bài phải làm hai việc: **vá một condition tùy chỉnh `ConfigIssue` vào
   `status` của Pod** bằng `kubectl patch --subresource=status`, rồi **xóa Pod** để nó chuyển sang
   phase kết thúc — lúc đó rule `FailJob` với `onPodConditions: [ConfigIssue]` mới khớp và Job dừng.
4. **Hai Pod cho index 0** vì index 0 thoát với mã `1` — mã này **không khớp** rule `FailIndex`
   (chỉ bắt `42`), nên nó rơi về `backoffLimitPerIndex: 1` và được thử lại đúng một lần.
   **Một Pod cho index 1** vì index 1 thoát với mã `42`, khớp rule `FailIndex`, nên index đó hỏng
   luôn **không thử lại**. `failedIndexes: "0,1"` vì rốt cuộc cả hai index đều hỏng. `failed: 3` là
   **tổng số Pod thất bại**: hai Pod của index 0 cộng một Pod của index 1 — nó đếm Pod, không đếm
   index.
5. Vì hai cái đánh dấu **hai thời điểm khác nhau**: `FailureTarget` được thêm **ngay khi Job bị coi
   là thất bại**, còn `Failed` chỉ được thêm **sau khi toàn bộ Pod của Job đã bị chấm dứt**. Giữa
   hai mốc đó Job vẫn đang dọn Pod. Cả hai mang cùng `reason: PodFailurePolicy` và cùng `message`
   chỉ ra Pod nào, exit code nào, khớp rule ở index nào.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

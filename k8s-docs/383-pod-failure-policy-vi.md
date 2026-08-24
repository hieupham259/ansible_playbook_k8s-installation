# Xử lý các lần Pod thất bại có thể thử lại và không thể thử lại bằng Pod failure policy (Handling retriable and non-retriable pod failures with Pod failure policy)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/job/pod-failure-policy/>
>
> Trang này hướng dẫn bạn dùng Pod failure policy, kết hợp với Pod backoff failure policy mặc
> định, để kiểm soát tốt hơn cách xử lý lỗi ở mức container hoặc mức Pod bên trong một Job.

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

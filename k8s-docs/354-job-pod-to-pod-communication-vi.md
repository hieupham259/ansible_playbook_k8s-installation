# Job với giao tiếp Pod-đến-Pod (Job with Pod-to-Pod Communication)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/job/job-with-pod-to-pod-communication/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 5 — Mạng nền tảng](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng), dòng **Thực
hành**, bài 6/10 · Kiểm chứng ở [Lab 5a](labs/LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md) phần B11.3 —
headless Service với selector `job-name`, `completionMode: Indexed` và `subdomain`; Job chỉ
`Complete` khi mọi Pod gọi được nhau bằng hostname.

Bài về Job nhưng nằm ở giai đoạn 5 vì thứ làm nó chạy được là **headless Service** và **DNS cấp
Pod** — hai khái niệm của giai đoạn này. Job chỉ đóng vai người dùng: nó cung cấp hostname xác định
trước, còn Service mới là thứ khiến hostname đó phân giải được.

**Phải hiểu ở lần đọc này:**

- Vấn đề bài giải và lý do chọn cách này: Pod của Job cần gọi nhau; chúng **có thể** hỏi API server
  để lấy IP của các Pod khác, nhưng bài chọn dựa vào DNS có sẵn — cấu hình này hữu ích khi bạn cần
  mạng giữa các Pod mà **không muốn phụ thuộc vào kết nối tới API server**.
- `completionMode: Indexed` tự đặt hostname Pod theo `${jobName}-${completionIndex}`. Đó là điều
  kiện làm cho tên **tính trước được một cách xác định** — script trong container dựng thẳng
  `example-job-${i}` mà không cần gọi API nào.
- Hai việc **bắt buộc** ở mục *Khởi chạy một Job với giao tiếp Pod-đến-Pod*: (1) một headless Service
  **cùng namespace** với Job, có label selector khớp Pod của Job — cách dễ nhất là dùng
  `job-name: <tên-job>` vì label `job-name` được Kubernetes tự thêm; (2) khai
  `subdomain: <tên-headless-svc>` trong template spec của Job. Chính cấu hình (1) là thứ "kích hoạt
  hệ thống DNS tạo các bản ghi hostname cho các Pod đang chạy Job của bạn".
- Dạng tên dùng để gọi là `<pod-hostname>.<tên-headless-service>`; ghi chú của bài nói hostname còn
  có thể được thêm tiền tố namespace nếu Pod cần được truy cập từ **ngoài** namespace đó.
- Ranh giới ở ghi chú cuối bài: dạng tên này **không hoạt động** nếu DNS policy được đặt là `None`
  hoặc `Default`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Link *chế độ hoàn thành Indexed* trỏ ra bài blog kubernetes.io năm 2021 | bài blog là phần giới thiệu lịch sử; bản thân `completionMode: Indexed` bạn đã học và đã chạy rồi | bài [67 — Jobs](67-job-vi.md) và [353](353-indexed-parallel-processing-vi.md) ở [giai đoạn 4](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller), thực hành ở [Lab 4b](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) phần B6.3 |
| Script bash trong container ví dụ — vòng `while` `ping` lại tới khi `gotStatus` bằng `wantStatus` | đây là cách bài **chứng minh** mọi Pod gọi được nhau, không phải một cơ chế của Kubernetes; điều cần nhớ là Job chỉ hoàn thành sau khi mọi Pod ping thành công lẫn nhau | [Lab 5a](labs/LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md) phần B11.3 chạy đúng ý tưởng này trên cluster lab |

---

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

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 5:

1. Job `example-job` có `completions: 3` và `completionMode: Indexed`. Hostname của ba Pod là gì, và
   vì sao script trong container tính được ba cái tên đó **trước khi** Pod nào kịp chạy?
2. **Câu bẫy.** Bạn tạo đúng headless Service `headless-svc` với selector `job-name: example-job`,
   nhưng quên khai `subdomain: headless-svc` trong template của Job. Job có được tạo không, và các
   Pod có gọi được nhau bằng `example-job-0.headless-svc` không?
3. Ba Pod của Job nằm rải trên `lab-k8s-worker1` và `lab-k8s-worker2`. Chúng gọi nhau bằng tên
   `example-job-0.headless-svc` mà không hỏi API server lần nào. Ai đã tạo ra bản ghi DNS đó, và
   điều kiện nào để nó tồn tại?
4. Cùng ba Pod đó, nhưng một Pod ở namespace khác cần gọi tới. Tên gọi phải đổi thế nào, và có cấu
   hình nào của Pod làm cả cách gọi này hỏng hẳn không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. `example-job-0`, `example-job-1`, `example-job-2`. Vì ở chế độ hoàn thành Indexed, Kubernetes
   **tự đặt hostname theo `${jobName}-${completionIndex}`** — tên Job bạn biết trước, các index là
   `0..completions-1`, nên script chỉ cần lặp `for i in 0 1 2` và ghép chuỗi. Đó chính là điều bài
   gọi là dựng hostname "một cách xác định (deterministic)", và là lý do Pod **không** phải mở kết
   nối client tới control plane để hỏi hostname/IP.
2. **Job vẫn được tạo và Pod vẫn chạy, nhưng chúng không gọi được nhau.** Bài liệt kê hai việc bạn
   *phải* làm, và chúng bổ sung cho nhau: headless Service làm hệ thống DNS sinh bản ghi, còn
   `subdomain` là thứ gắn Pod vào đúng miền tên đó — ví dụ ghi rõ `subdomain: headless-svc # phải
   khớp với tên Service`. Trực giác sai ở chỗ tưởng chỉ cần selector khớp là đủ. Hệ quả nhìn thấy
   được: script `ping` lặp mãi mà không bao giờ thành công, mà theo bài "Job chỉ hoàn thành sau khi
   tất cả các Pod ping thành công lẫn nhau bằng hostname" — nên Job **không bao giờ hoàn thành**.
3. **Hệ thống DNS của cluster tạo ra**, chứ không phải Job và cũng không phải API server trả lời trực
   tiếp: bài nói việc dựng headless Service với selector khớp "sẽ kích hoạt hệ thống DNS tạo các bản
   ghi hostname cho các Pod đang chạy Job của bạn". Điều kiện gồm ba phần — headless Service **cùng
   namespace** với Job, **selector khớp** Pod của Job (dễ nhất là `job-name: <tên-job>` vì label này
   được Kubernetes tự thêm), và Pod khai `subdomain` đúng bằng tên Service đó.
4. Phải **thêm tiền tố namespace** vào hostname: ghi chú của bài nói hostname của Pod "cũng có thể
   được thêm tiền tố namespace nếu Pod cần được truy cập từ bên ngoài namespace đó". Và có: ghi chú
   cuối bài nói dạng tên `<pod-hostname>.<tên-headless-service>` **sẽ không hoạt động nếu DNS policy
   được đặt là `None` hoặc `Default`**.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

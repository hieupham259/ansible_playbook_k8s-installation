# CronJob

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/>
>
> CronJob khởi chạy các Job chạy-một-lần (one-time Job) theo một lịch lặp lại.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 4](LO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller), bài 8/14 ·
Kiểm chứng ở Lab 4 (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

CronJob chỉ là một tầng mỏng đặt trên [Job](67-job-vi.md): nó tạo Job theo lịch, còn Job mới
là thứ quản lý Pod. Vì vậy nếu bài 67 chưa vững thì quay lại đó trước. Cú pháp cron thì tra
khi cần, không cần thuộc; cái phải hiểu là **hai trường quyết định hành vi khi mọi thứ chạy
chậm hoặc control plane gián đoạn**: `concurrencyPolicy` và `startingDeadlineSeconds`.

**Phải hiểu ở lần đọc này:**

- Phân tầng trách nhiệm: CronJob **chỉ chịu trách nhiệm tạo các Job khớp với lịch của nó**,
  còn Job chịu trách nhiệm quản lý Pod. Hai trường bắt buộc là `.spec.schedule` và
  `.spec.jobTemplate`.
- `.spec.concurrencyPolicy`: `Allow` (mặc định, cho chạy chồng), `Forbid` (bỏ qua lần mới nếu
  lần trước chưa xong), `Replace` (thay lần đang chạy bằng lần mới). Chính sách này **chỉ áp
  dụng trong phạm vi cùng một CronJob**.
- `.spec.startingDeadlineSeconds`: nếu lỡ mốc lịch quá số giây này thì lần chạy đó bị bỏ qua
  và **được tính là Job thất bại**; không đặt thì các lần chạy không có thời hạn. Đặt nhỏ hơn
  10 giây thì CronJob có thể **không được lập lịch**, vì controller chỉ kiểm tra mỗi 10 giây.
- Cửa sổ đếm số lịch bị lỡ **đổi theo trường trên**: không đặt thì đếm từ lần lập lịch cuối
  tới hiện tại, và **quá 100 lịch bị lỡ** thì controller ngừng khởi động Job và ghi log lỗi;
  có đặt thì chỉ đếm trong khoảng `startingDeadlineSeconds` gần nhất. Đây chính là ví dụ
  `08:29:00`–`10:21:00` trong bài.
- Việc tạo Job chỉ là **xấp xỉ** một lần cho mỗi mốc lịch — có thể hai Job, có thể không Job
  nào — nên Job bạn viết phải **idempotent**. Và sửa một CronJob **không** đụng tới các Job
  đã khởi động, kể cả khi chúng đang chạy.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Cú pháp schedule* — bước nhảy kiểu Vixie cron, dấu `?`, bảng macro `@daily` và các dòng khác | chỉ là cú pháp, tra khi cần | không cần |
| *Múi giờ* và *Chỉ định TimeZone không được hỗ trợ* | chỉ cần nhớ: mặc định theo múi giờ của kube-controller-manager, và `CRON_TZ`/`TZ` trong `schedule` bị API từ chối | không cần |
| *Giới hạn lịch sử Job* | chỉ là hai con số mặc định: giữ 3 Job thành công và 1 Job thất bại | không cần |
| *Tạm ngưng lập lịch* (`.spec.suspend`) | thao tác vận hành, chỉ cần khi đã quen CronJob | không cần |
| Annotation `batch.kubernetes.io/cronjob-scheduled-timestamp` | dùng để truy vết, không đổi hành vi | không cần |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.21 [stable]`

Một _CronJob_ tạo các Job theo một lịch lặp lại.

CronJob được thiết kế để thực hiện các hành động định kỳ theo lịch như sao lưu (backup),
tạo báo cáo, v.v. Một object CronJob giống như một dòng trong file _crontab_ (cron table)
trên hệ thống Unix. Nó chạy một Job theo chu kỳ với một lịch cho trước, được viết theo
định dạng [Cron](https://en.wikipedia.org/wiki/Cron).

CronJob có những hạn chế và đặc thù riêng.
Ví dụ, trong một số trường hợp nhất định, một CronJob duy nhất có thể tạo ra nhiều Job chạy
đồng thời. Xem phần [các hạn chế](#cron-job-limitations) bên dưới.

Khi control plane tạo các Job mới và (một cách gián tiếp) các Pod cho một CronJob,
`.metadata.name` của CronJob là một phần cơ sở để đặt tên cho các Pod đó. Tên của một
CronJob phải là một giá trị
[DNS subdomain](./17-names-vi.md)
hợp lệ, nhưng điều này có thể tạo ra kết quả không mong muốn đối với hostname của Pod.
Để có độ tương thích tốt nhất, tên nên tuân theo các quy tắc chặt chẽ hơn của một
[DNS label](./17-names-vi.md#dns-label-names).
Ngay cả khi tên là một DNS subdomain, tên cũng không được dài quá 52 ký tự. Lý do là
CronJob controller sẽ tự động thêm 11 ký tự vào tên bạn cung cấp, và có một ràng buộc rằng
độ dài tên của một Job không được vượt quá 63 ký tự.

## Ví dụ (Example)

Manifest CronJob mẫu này in ra thời gian hiện tại và một thông điệp chào mỗi phút một lần:

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

([Chạy các tác vụ tự động với CronJob](https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/)
hướng dẫn bạn đi qua ví dụ này chi tiết hơn).

## Viết một CronJob spec (Writing a CronJob spec)

### Cú pháp schedule (Schedule syntax)

Trường `.spec.schedule` là bắt buộc. Giá trị của trường này tuân theo cú pháp
[Cron](https://en.wikipedia.org/wiki/Cron):

```
# ┌───────────── phút (0 - 59)
# │ ┌───────────── giờ (0 - 23)
# │ │ ┌───────────── ngày trong tháng (1 - 31)
# │ │ │ ┌───────────── tháng (1 - 12)
# │ │ │ │ ┌───────────── thứ trong tuần (0 - 6) (Chủ nhật đến Thứ bảy)
# │ │ │ │ │                                   HOẶC sun, mon, tue, wed, thu, fri, sat
# │ │ │ │ │
# │ │ │ │ │
# * * * * *
```

Ví dụ, `0 3 * * 1` có nghĩa là tác vụ này được lập lịch chạy hằng tuần vào thứ Hai lúc 3 giờ sáng.

Định dạng này cũng bao gồm giá trị bước nhảy (step value) mở rộng kiểu "Vixie cron".
Như giải thích trong [tài liệu FreeBSD](https://www.freebsd.org/cgi/man.cgi?crontab%285%29):

> Giá trị bước nhảy có thể được dùng kết hợp với khoảng giá trị (range). Viết `/<số>`
> sau một khoảng nghĩa là bỏ qua với bước nhảy bằng giá trị của số đó trong khoảng.
> Ví dụ, `0-23/2` có thể được dùng trong trường giờ để chỉ định thực thi lệnh cách một
> giờ một lần (cách viết thay thế theo chuẩn V7 là
> `0,2,4,6,8,10,12,14,16,18,20,22`). Bước nhảy cũng được phép đứng sau dấu hoa thị,
> vì vậy nếu bạn muốn nói "hai giờ một lần", chỉ cần dùng `*/2`.

> **Ghi chú:**
> Dấu chấm hỏi (`?`) trong lịch có cùng ý nghĩa với dấu hoa thị `*`, tức là nó đại diện
> cho bất kỳ giá trị khả dụng nào của trường tương ứng.

Ngoài cú pháp tiêu chuẩn, bạn cũng có thể dùng một số macro như `@monthly`:

| Mục nhập | Mô tả | Tương đương với |
| ------------- | ------------- |------------- |
| @yearly (hoặc @annually) | Chạy một lần mỗi năm vào nửa đêm ngày 1 tháng 1 | 0 0 1 1 * |
| @monthly | Chạy một lần mỗi tháng vào nửa đêm ngày đầu tiên của tháng | 0 0 1 * * |
| @weekly | Chạy một lần mỗi tuần vào nửa đêm sáng Chủ nhật | 0 0 * * 0 |
| @daily (hoặc @midnight) | Chạy một lần mỗi ngày vào nửa đêm | 0 0 * * * |
| @hourly | Chạy một lần mỗi giờ vào đầu giờ | 0 * * * * |

Để tạo các biểu thức lịch CronJob, bạn cũng có thể dùng các công cụ web như
[crontab.guru](https://crontab.guru/).

### Job template

`.spec.jobTemplate` định nghĩa một template cho các Job mà CronJob tạo ra, và nó là bắt buộc.
Nó có schema y hệt như một [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/),
ngoại trừ việc nó được lồng bên trong và không có `apiVersion` hay `kind`.
Bạn có thể chỉ định metadata chung cho các Job được tạo từ template, chẳng hạn như các
[label](./18-labels-vi.md) hoặc các [annotation](./20-annotations-vi.md).
Để biết cách viết `.spec` cho một Job, xem
[Viết một Job Spec](https://kubernetes.io/docs/concepts/workloads/controllers/job/#writing-a-job-spec).

### Thời hạn cho Job khởi động trễ (Deadline for delayed Job start) {#starting-deadline}

Trường `.spec.startingDeadlineSeconds` là tùy chọn.
Trường này định nghĩa một thời hạn (deadline, tính bằng số giây nguyên) cho việc khởi động
Job, nếu Job đó lỡ mất thời điểm được lập lịch vì bất kỳ lý do gì.

Sau khi đã lỡ thời hạn, CronJob sẽ bỏ qua lần chạy đó của Job (các lần chạy trong tương lai
vẫn được lập lịch). Ví dụ, nếu bạn có một Job sao lưu chạy hai lần mỗi ngày, bạn có thể cho
phép nó khởi động trễ tối đa 8 giờ, nhưng không muộn hơn, vì một bản sao lưu được thực hiện
muộn hơn nữa sẽ không còn hữu ích: thay vào đó bạn muốn chờ lần chạy được lập lịch kế tiếp.

Với các Job lỡ thời hạn đã cấu hình, Kubernetes coi chúng là các Job thất bại.
Nếu bạn không chỉ định `startingDeadlineSeconds` cho một CronJob, các lần chạy Job sẽ không
có thời hạn.

Nếu trường `.spec.startingDeadlineSeconds` được đặt (khác null), CronJob controller sẽ đo
khoảng thời gian giữa thời điểm Job được kỳ vọng sẽ được tạo và thời điểm hiện tại. Nếu
khoảng chênh lệch lớn hơn giới hạn đó, controller sẽ bỏ qua lần thực thi này.

Ví dụ, nếu trường này được đặt là `200`, nó cho phép Job được tạo trong tối đa 200 giây
sau thời điểm theo lịch thực tế.

### Chính sách chạy đồng thời (Concurrency policy)

Trường `.spec.concurrencyPolicy` cũng là tùy chọn.
Nó chỉ định cách xử lý các lần thực thi đồng thời của Job được tạo bởi CronJob này.
Spec chỉ có thể chỉ định một trong các chính sách đồng thời sau:

* `Allow` (mặc định): CronJob cho phép các Job chạy đồng thời
* `Forbid`: CronJob không cho phép chạy đồng thời; nếu đã đến thời điểm chạy Job mới mà
  lần chạy Job trước đó chưa kết thúc, CronJob sẽ bỏ qua lần chạy Job mới. Cũng lưu ý rằng
  khi lần chạy Job trước đó kết thúc, `.spec.startingDeadlineSeconds` vẫn được tính đến và
  có thể dẫn tới một lần chạy Job mới.
* `Replace`: Nếu đã đến thời điểm chạy Job mới mà lần chạy Job trước đó chưa kết thúc,
  CronJob sẽ thay thế lần chạy Job hiện tại bằng một lần chạy Job mới

Lưu ý rằng chính sách đồng thời chỉ áp dụng cho các Job được tạo bởi cùng một CronJob.
Nếu có nhiều CronJob, các Job tương ứng của chúng luôn được phép chạy đồng thời.

### Tạm ngưng lập lịch (Schedule suspension)

Bạn có thể tạm ngưng việc thực thi các Job của một CronJob bằng cách đặt trường tùy chọn
`.spec.suspend` thành true. Trường này mặc định là false.

Thiết lập này _không_ ảnh hưởng đến các Job mà CronJob đã khởi động trước đó.

Nếu bạn đặt trường này thành true, tất cả các lần thực thi tiếp theo sẽ bị tạm ngưng
(chúng vẫn được lập lịch, nhưng CronJob controller không khởi động các Job để chạy tác vụ)
cho đến khi bạn bỏ tạm ngưng CronJob.

> **Thận trọng:**
> Các lần thực thi bị tạm ngưng trong thời gian đã được lập lịch của chúng được tính là
> các Job bị lỡ (missed). Khi `.spec.suspend` thay đổi từ `true` thành `false` trên một
> CronJob hiện có mà không có [thời hạn khởi động](#starting-deadline), các Job bị lỡ sẽ
> được lập lịch ngay lập tức.

### Giới hạn lịch sử Job (Jobs history limits)

Các trường `.spec.successfulJobsHistoryLimit` và `.spec.failedJobsHistoryLimit` chỉ định
số lượng Job đã hoàn thành và Job thất bại cần được giữ lại. Cả hai trường đều là tùy chọn.

* `.spec.successfulJobsHistoryLimit`: Trường này chỉ định số lượng Job kết thúc thành công
cần giữ lại. Giá trị mặc định là `3`. Đặt trường này thành `0` sẽ không giữ lại bất kỳ
Job thành công nào.

* `.spec.failedJobsHistoryLimit`: Trường này chỉ định số lượng Job kết thúc thất bại cần
giữ lại. Giá trị mặc định là `1`. Đặt trường này thành `0` sẽ không giữ lại bất kỳ Job
thất bại nào.

Để biết một cách khác dọn dẹp Job tự động, xem
[Tự động dọn dẹp các Job đã hoàn thành](https://kubernetes.io/docs/concepts/workloads/controllers/job/#clean-up-finished-jobs-automatically).

### Múi giờ (Time zones)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.27 [stable]`

Với các CronJob không chỉ định múi giờ, kube-controller-manager
diễn giải lịch theo múi giờ cục bộ (local time zone) của nó.

Bạn có thể chỉ định múi giờ cho một CronJob bằng cách đặt `.spec.timeZone` thành tên của
một [múi giờ](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) hợp lệ.
Ví dụ, đặt `.spec.timeZone: "Etc/UTC"` sẽ chỉ dẫn Kubernetes diễn giải lịch theo Giờ Phối
hợp Quốc tế (Coordinated Universal Time).

Một cơ sở dữ liệu múi giờ từ thư viện chuẩn của Go được tích hợp sẵn trong các file nhị
phân và được dùng làm phương án dự phòng trong trường hợp cơ sở dữ liệu bên ngoài không
khả dụng trên hệ thống.

## Các hạn chế của CronJob (CronJob limitations) {#cron-job-limitations}

### Chỉ định TimeZone không được hỗ trợ (Unsupported TimeZone specification)

Việc chỉ định múi giờ bằng biến `CRON_TZ` hoặc `TZ` bên trong `.spec.schedule` là
**không được hỗ trợ chính thức** (và chưa bao giờ được hỗ trợ). Nếu bạn cố đặt một lịch
có chứa chỉ định múi giờ `TZ` hoặc `CRON_TZ`, Kubernetes sẽ không thể tạo hoặc cập nhật
resource và trả về lỗi kiểm tra hợp lệ (validation error). Thay vào đó, bạn nên chỉ định
múi giờ bằng [trường time zone](#time-zones).

### Sửa đổi một CronJob (Modifying a CronJob)

Theo thiết kế, một CronJob chứa một template cho các Job _mới_.
Nếu bạn sửa đổi một CronJob hiện có, các thay đổi của bạn sẽ áp dụng cho các Job mới bắt
đầu chạy sau khi việc sửa đổi hoàn tất. Các Job (và các Pod của chúng) đã khởi động trước
đó vẫn tiếp tục chạy mà không có thay đổi nào.
Nghĩa là, CronJob _không_ cập nhật các Job hiện có, ngay cả khi các Job đó vẫn đang chạy.

### Việc tạo Job (Job creation)

Một CronJob tạo một object Job xấp xỉ một lần cho mỗi thời điểm thực thi trong lịch của nó.
Việc lập lịch chỉ là xấp xỉ vì có một số trường hợp nhất định mà hai Job có thể được tạo ra,
hoặc không Job nào được tạo cả. Kubernetes cố gắng tránh những tình huống đó, nhưng không
ngăn chặn được chúng hoàn toàn. Do đó, các Job mà bạn định nghĩa nên có tính _idempotent_
(chạy lại nhiều lần vẫn cho cùng kết quả).

Bắt đầu từ Kubernetes v1.32, CronJob gắn một annotation
`batch.kubernetes.io/cronjob-scheduled-timestamp` vào các Job mà nó tạo ra. Annotation này
cho biết thời điểm tạo được lập lịch ban đầu của Job và được định dạng theo RFC3339.

Nếu `startingDeadlineSeconds` được đặt thành một giá trị lớn hoặc không được đặt (mặc định)
và nếu `concurrencyPolicy` được đặt thành `Allow`, các Job sẽ luôn chạy ít nhất một lần.

> **Thận trọng:**
> Nếu `startingDeadlineSeconds` được đặt thành một giá trị nhỏ hơn 10 giây, CronJob có thể
> không được lập lịch. Lý do là CronJob controller kiểm tra mọi thứ mỗi 10 giây một lần.

Với mỗi CronJob, CronJob controller kiểm tra xem nó đã lỡ bao nhiêu lịch trong khoảng thời
gian từ lần được lập lịch cuối cùng đến hiện tại. Nếu có hơn 100 lịch bị lỡ, controller sẽ
không khởi động Job và ghi log lỗi.

```
too many missed start times. Set or decrease .spec.startingDeadlineSeconds or check clock skew
```

Hành vi này áp dụng cho việc lập lịch bù (catch-up scheduling) và không có nghĩa là CronJob
sẽ ngừng chạy.

Ví dụ, khi dùng `concurrencyPolicy: Forbid`, các Job chạy lâu có thể khiến các thời điểm
theo lịch bị bỏ qua, nhưng một Job mới có thể được tạo ngay khi Job trước đó hoàn thành.

Điều quan trọng cần lưu ý là nếu trường `startingDeadlineSeconds` được đặt (khác `nil`),
controller sẽ đếm số Job bị lỡ trong khoảng từ giá trị của `startingDeadlineSeconds` đến
hiện tại, thay vì từ lần được lập lịch cuối cùng đến hiện tại. Ví dụ, nếu
`startingDeadlineSeconds` là `200`, controller sẽ đếm số Job bị lỡ trong 200 giây gần nhất.

Một CronJob được tính là bị lỡ nếu nó không được tạo thành công vào thời điểm đã lập lịch.
Ví dụ, nếu `concurrencyPolicy` được đặt thành `Forbid` và một CronJob được thử lập lịch
trong khi một lần chạy theo lịch trước đó vẫn đang chạy, thì lần đó sẽ được tính là bị lỡ.

Ví dụ, giả sử một CronJob được thiết lập để lập lịch một Job mới mỗi phút một lần bắt đầu
từ `08:30:00`, và trường `startingDeadlineSeconds` của nó không được đặt. Nếu CronJob
controller tình cờ bị ngừng hoạt động từ `08:29:00` đến `10:21:00`, Job sẽ không được khởi
động vì số lượng Job bị lỡ lịch đã vượt quá 100.

Để minh họa khái niệm này rõ hơn, giả sử một CronJob được thiết lập để lập lịch một Job mới
mỗi phút một lần bắt đầu từ `08:30:00`, và `startingDeadlineSeconds` của nó được đặt là
200 giây. Nếu CronJob controller tình cờ bị ngừng hoạt động trong cùng khoảng thời gian như
ví dụ trước (`08:29:00` đến `10:21:00`), Job vẫn sẽ được khởi động lúc 10:22:00. Điều này
xảy ra vì lúc này controller kiểm tra xem có bao nhiêu lịch bị lỡ trong 200 giây gần nhất
(tức là 3 lịch bị lỡ), thay vì tính từ lần được lập lịch cuối cùng đến hiện tại.

CronJob chỉ chịu trách nhiệm tạo các Job khớp với lịch của nó, còn Job đến lượt mình chịu
trách nhiệm quản lý các Pod mà nó đại diện.

## Tiếp theo (What's next)

* Tìm hiểu về [Pod](./46-pods-vi.md) và
  [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/), hai khái niệm
  mà CronJob dựa trên.
* Đọc chi tiết về [định dạng](https://pkg.go.dev/github.com/robfig/cron/v3#hdr-CRON_Expression_Format)
  của trường `.spec.schedule` trong CronJob.
* Để biết hướng dẫn tạo và làm việc với CronJob, và một ví dụ về manifest CronJob,
  xem [Chạy các tác vụ tự động với CronJob](https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/).
* `CronJob` là một phần của Kubernetes REST API.
  Đọc tài liệu tham khảo API
  [CronJob](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/cron-job-v1/)
  để biết thêm chi tiết.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 4:

1. Một CronJob `schedule: "* * * * *"` với `concurrencyPolicy: Forbid`, mỗi Job chạy mất 3
   phút. Trong ba phút đó có mấy Job được tạo, và các mốc lịch bị bỏ qua được tính là gì?
2. **Câu bẫy.** Hai CronJob khác nhau, cả hai cùng đặt `concurrencyPolicy: Forbid`. Job của
   CronJob A có bị chặn vì Job của CronJob B đang chạy không?
3. `k8s-master` — nơi kube-controller-manager chạy — ngừng hoạt động từ `08:29:00` tới
   `10:21:00`. Bạn có một CronJob chạy mỗi phút. Khi control plane trở lại, Job có được khởi
   động không? Câu trả lời đổi thế nào nếu `startingDeadlineSeconds: 200`?
4. Vì sao bài cảnh báo không đặt `startingDeadlineSeconds` nhỏ hơn 10 giây?
5. Vì sao bài yêu cầu Job do CronJob tạo phải idempotent?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Chỉ **một Job** — Job đang chạy. Với `Forbid`, "nếu đã đến thời điểm chạy Job mới mà lần
   chạy Job trước đó chưa kết thúc, CronJob sẽ bỏ qua lần chạy Job mới". Các mốc bị bỏ qua
   được **tính là bị lỡ (missed)**: bài nói "nếu `concurrencyPolicy` được đặt thành `Forbid`
   và một CronJob được thử lập lịch trong khi một lần chạy theo lịch trước đó vẫn đang chạy,
   thì lần đó sẽ được tính là bị lỡ". Lưu ý thêm: khi Job trước kết thúc,
   `startingDeadlineSeconds` vẫn được tính đến và có thể dẫn tới một lần chạy mới ngay.
2. **Không.** Bài nói rõ: "chính sách đồng thời chỉ áp dụng cho các Job được tạo bởi **cùng
   một CronJob**. Nếu có nhiều CronJob, các Job tương ứng của chúng **luôn được phép chạy đồng
   thời**." Trực giác sai ở chỗ hiểu `Forbid` như một khóa toàn cluster; nó chỉ là khóa trong
   phạm vi một object CronJob. Muốn hai CronJob không giẫm chân nhau thì phải tự xử lý ở tầng
   ứng dụng.
3. **Không đặt `startingDeadlineSeconds`: Job sẽ không được khởi động.** Controller đếm số
   lịch bị lỡ từ lần được lập lịch cuối tới hiện tại, ở đây là hơn 100 lịch, và bài nói khi
   quá 100 lịch bị lỡ thì controller không khởi động Job mà ghi log
   `too many missed start times...`. **Với `startingDeadlineSeconds: 200`: Job vẫn khởi động
   lúc 10:22:00**, vì lúc này controller chỉ đếm số lịch bị lỡ **trong 200 giây gần nhất** —
   tức 3 lịch — thay vì từ lần lập lịch cuối. Hành vi này chỉ liên quan tới việc lập lịch bù,
   không có nghĩa CronJob ngừng chạy hẳn.
4. Vì **CronJob controller kiểm tra mọi thứ mỗi 10 giây một lần**. Đặt thời hạn ngắn hơn chu
   kỳ kiểm tra thì mốc lịch có thể đã quá hạn trước khi controller kịp nhìn tới, và CronJob
   **có thể không được lập lịch** chút nào.
5. Vì **việc lập lịch chỉ là xấp xỉ**: bài nói một CronJob tạo một object Job "xấp xỉ một lần
   cho mỗi thời điểm thực thi trong lịch của nó", và có những trường hợp **hai Job được tạo,
   hoặc không Job nào được tạo cả**. Kubernetes cố tránh nhưng không ngăn được hoàn toàn. Do
   đó Job phải chạy lại nhiều lần vẫn cho cùng kết quả. Ngoài ra, nếu `startingDeadlineSeconds`
   lớn hoặc không đặt và `concurrencyPolicy` là `Allow`, các Job sẽ luôn chạy **ít nhất một
   lần** — "ít nhất", không phải "đúng một lần".

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

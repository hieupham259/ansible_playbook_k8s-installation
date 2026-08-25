# Giới hạn và dự trữ Process ID (Process ID Limits And Reservations)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/policy/pid-limiting/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 7 → nhóm [7b](00-ALO-TRINH-ADMIN.md#7b-chính-sách-giới-hạn-tài-nguyên),
bài 4/6 · Kiểm chứng ở [Lab 7b](labs/LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md).

Bài này **đổi mặt phẳng cấu hình**. Hai bài trước là đối tượng API trong namespace; từ đây tới
hết nhóm 7b là chính sách nằm trên kubelet của từng node — đúng nhánh thứ hai mà bài
[132](132-policies-vi.md) đã vạch ra. Bài ngắn, nhưng đọc kỹ chỗ nó nói vì sao PID không khai
được trong `.spec` của Pod: đó là điểm khác biệt lớn nhất so với `requests`/`limits` bạn đã
quen từ bài [110](110-manage-resources-containers-vi.md).

**Phải hiểu ở lần đọc này:**

- PID là **tài nguyên nền tảng của node**: rất dễ chạm giới hạn số task mà chưa chạm bất kỳ
  giới hạn tài nguyên nào khác, và điều đó gây mất ổn định cho host.
- Nơi khai giới hạn: **không** nằm trong `.spec` của Pod mà là **thiết lập trên kubelet** —
  tham số `--pod-max-pids` hoặc `PodPidsLimit` trong file cấu hình kubelet. Giới hạn PID định
  nghĩa ở cấp Pod hiện **chưa được hỗ trợ**. Hệ quả bài cảnh báo: giới hạn áp cho một Pod có
  thể khác nhau tùy nơi Pod được lập lịch, nên cách dễ nhất là để mọi Node dùng cùng một mức.
- Hai lớp bảo vệ khác nhau, đừng nhầm: **dự trữ PID cho node** — tham số `pid=<number>` trong
  `--system-reserved` và `--kube-reserved` — giữ PID cho hệ điều hành và các daemon hệ thống
  của Kubernetes; **giới hạn PID của Pod** chỉ bảo vệ Pod này khỏi Pod khác. Bài nói thẳng:
  giới hạn theo từng Pod **không** đảm bảo mọi Pod trên host không tác động tới node nói chung,
  và **không** bảo vệ được chính các agent của node khỏi cạn PID.
- Cách nghĩ về "ngân sách": ví dụ của bài là node có `262144` PID và dự kiến dưới `250` Pod thì
  cấp mỗi Pod `1000` PID; overcommit được như với CPU hay bộ nhớ nhưng kèm rủi ro bổ sung. Dù
  theo cách nào, một Pod đơn lẻ cũng không làm sập được cả máy.
- Ranh giới giữa giới hạn cứng và eviction: giới hạn PID theo Pod và theo Node là **giới hạn
  cứng** — chạm tới là workload bắt đầu lỗi khi xin PID mới, và việc Pod có bị lập lịch lại hay
  không còn tùy cách workload phản ứng cùng cấu hình probe. Còn tín hiệu eviction
  `pid.available` được **tính toán định kỳ và KHÔNG cưỡng chế giới hạn**, nên kể cả đặt eviction
  cứng, node vẫn có thể mất ổn định nếu số PID tăng rất nhanh.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| `/proc/sys/kernel/pid_max` và giới hạn PID mặc định thấp của một số bản Linux | là cấu hình hệ điều hành, nằm ngoài Kubernetes | không cần |
| Cách thực sự áp `--pod-max-pids` / `PodPidsLimit` lên từng node | bạn chưa dựng cluster nên chưa sửa được kubelet | giai đoạn 8, bài [04](04-kubelet-integration-vi.md) |
| Cách đặt cụ thể `--system-reserved` và `--kube-reserved`, kèm phần dự trữ CPU/bộ nhớ đi cùng | là thao tác cấu hình trên node thật, không phải khái niệm | nhóm task giai đoạn 20 ở cuối lộ trình (*Reserve Compute Resources for System Daemons*) |
| Cách đặt ngưỡng eviction mềm và cứng cho `pid.available` | ở đây chỉ cần biết eviction **không** thay thế được giới hạn cứng | nhóm task giai đoạn 20 ở cuối lộ trình |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.20 [stable]`

Kubernetes cho phép bạn giới hạn số lượng process ID (PID) mà một Pod có thể sử dụng.
Bạn cũng có thể dự trữ một số lượng PID có thể cấp phát (allocatable) cho mỗi node
để hệ điều hành và các daemon sử dụng (thay vì cho các Pod).

Process ID (PID) là một tài nguyên nền tảng trên các node. Rất dễ chạm đến
giới hạn số lượng task mà không hề chạm đến bất kỳ giới hạn tài nguyên nào khác,
và điều đó có thể gây mất ổn định cho máy chủ (host).

Quản trị viên cluster cần có các cơ chế để đảm bảo rằng các Pod chạy trong
cluster không thể gây cạn kiệt PID, khiến các daemon của host (chẳng hạn
kubelet hoặc kube-proxy, và có thể cả container runtime) không chạy được.
Ngoài ra, việc đảm bảo PID được giới hạn giữa các Pod cũng rất quan trọng
nhằm đảm bảo chúng chỉ có ảnh hưởng hạn chế đến các workload khác trên cùng node.

> **Ghi chú:**
> Trên một số bản cài đặt Linux, hệ điều hành đặt giới hạn PID ở một giá trị mặc định thấp,
> chẳng hạn `32768`. Hãy cân nhắc tăng giá trị của `/proc/sys/kernel/pid_max`.

Bạn có thể cấu hình kubelet để giới hạn số lượng PID mà một Pod nhất định được phép tiêu thụ.
Ví dụ, nếu hệ điều hành host của node được thiết lập dùng tối đa `262144` PID và
dự kiến chứa ít hơn `250` Pod, ta có thể cấp cho mỗi Pod một "ngân sách" `1000`
PID để tránh dùng cạn tổng số PID khả dụng của node đó. Nếu quản trị viên muốn
overcommit PID tương tự như với CPU hoặc bộ nhớ, họ cũng có thể làm vậy nhưng
kèm theo một số rủi ro bổ sung. Dù theo cách nào, một Pod đơn lẻ sẽ không thể
làm sập toàn bộ máy. Kiểu giới hạn tài nguyên này giúp ngăn các fork bomb đơn giản
ảnh hưởng đến hoạt động của cả cluster.

Giới hạn PID theo từng Pod cho phép quản trị viên bảo vệ Pod này khỏi Pod khác, nhưng
không đảm bảo rằng mọi Pod được lập lịch (schedule) lên host đó không thể tác động đến node nói chung.
Giới hạn theo từng Pod cũng không bảo vệ được chính các agent của node khỏi tình trạng cạn kiệt PID.

Bạn cũng có thể dự trữ một lượng PID cho phần chi phí vận hành (overhead) của node,
tách biệt với phần cấp phát cho các Pod. Điều này tương tự như cách bạn có thể dự trữ
CPU, bộ nhớ hoặc các tài nguyên khác cho hệ điều hành và các thành phần khác
nằm ngoài các Pod và các container của chúng.

Giới hạn PID là "người anh em" quan trọng của request và limit
[tài nguyên tính toán (compute resource)](./110-manage-resources-containers-vi.md).
Tuy nhiên, bạn chỉ định nó theo một cách khác: thay vì định nghĩa giới hạn tài nguyên
của Pod trong `.spec` của Pod, bạn cấu hình giới hạn này như một thiết lập trên kubelet.
Giới hạn PID định nghĩa ở cấp Pod hiện chưa được hỗ trợ.

> **Thận trọng:**
> Điều này có nghĩa là giới hạn áp dụng cho một Pod có thể khác nhau tùy theo
> nơi Pod được lập lịch. Để mọi thứ đơn giản, cách dễ nhất là tất cả các Node
> dùng cùng một mức giới hạn và dự trữ tài nguyên PID.

## Giới hạn PID của node (Node PID limits)

Kubernetes cho phép bạn dự trữ một số lượng process ID cho hệ thống sử dụng. Để
cấu hình mức dự trữ này, dùng tham số `pid=<number>` trong các tùy chọn
dòng lệnh `--system-reserved` và `--kube-reserved` của kubelet.
Giá trị bạn chỉ định khai báo rằng số lượng process ID đó sẽ được dự trữ
tương ứng cho toàn hệ thống và cho các daemon hệ thống của Kubernetes.

## Giới hạn PID của Pod (Pod PID limits)

Kubernetes cho phép bạn giới hạn số lượng process chạy trong một Pod. Bạn
chỉ định giới hạn này ở cấp node, thay vì cấu hình nó như một giới hạn tài nguyên
cho một Pod cụ thể. Mỗi Node có thể có một giới hạn PID khác nhau.
Để cấu hình giới hạn, bạn có thể chỉ định tham số dòng lệnh `--pod-max-pids`
cho kubelet, hoặc đặt `PodPidsLimit` trong
[file cấu hình](224-kubelet-config-file-vi.md) của kubelet.

## Eviction dựa trên PID (PID based eviction)

Bạn có thể cấu hình kubelet để bắt đầu chấm dứt (terminate) một Pod khi Pod đó hoạt động
bất thường và tiêu thụ một lượng tài nguyên không bình thường.
Tính năng này được gọi là eviction (thu hồi). Bạn có thể
[Cấu hình xử lý khi hết tài nguyên (Configure Out of Resource Handling)](142-node-pressure-eviction-vi.md)
cho nhiều tín hiệu eviction khác nhau.
Dùng tín hiệu eviction `pid.available` để cấu hình ngưỡng số lượng PID mà Pod sử dụng.
Bạn có thể đặt chính sách eviction mềm (soft) và cứng (hard).
Tuy nhiên, ngay cả với chính sách eviction cứng, nếu số lượng PID tăng rất nhanh,
node vẫn có thể rơi vào trạng thái mất ổn định do chạm đến giới hạn PID của node.
Giá trị của tín hiệu eviction được tính toán định kỳ và KHÔNG cưỡng chế giới hạn.

Giới hạn PID — theo từng Pod và theo từng Node — đặt ra giới hạn cứng.
Một khi chạm đến giới hạn, workload sẽ bắt đầu gặp lỗi khi cố lấy một PID mới.
Điều đó có thể dẫn đến hoặc không dẫn đến việc Pod bị lập lịch lại,
tùy vào cách workload phản ứng với các lỗi này và cách các probe liveness và readiness
của Pod được cấu hình. Tuy nhiên, nếu các giới hạn được đặt đúng,
bạn có thể đảm bảo rằng workload của các Pod khác và các process hệ thống
sẽ không bị cạn PID khi có một Pod hoạt động bất thường.

## Tiếp theo (What's next)

- Tham khảo [tài liệu enhancement về PID Limiting](https://github.com/kubernetes/enhancements/blob/097b4d8276bc9564e56adf72505d43ce9bc5e9e8/keps/sig-node/20190129-pid-limiting.md) để biết thêm thông tin.
- Về bối cảnh lịch sử, đọc
  [Process ID Limiting for Stability Improvements in Kubernetes 1.14](https://kubernetes.io/blog/2019/04/15/process-id-limiting-for-stability-improvements-in-kubernetes-1.14/).
- Đọc [Quản lý tài nguyên cho container (Managing Resources for Containers)](./110-manage-resources-containers-vi.md).
- Tìm hiểu cách [Cấu hình xử lý khi hết tài nguyên (Configure Out of Resource Handling)](142-node-pressure-eviction-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 7:

1. Giới hạn PID của một Pod được khai ở đâu? Vì sao hai bản sao giống hệt nhau của cùng một
   Pod trong cluster của bạn lại có thể chịu hai mức giới hạn PID khác nhau?
2. Cluster lab của bạn có hai worker, mỗi máy 2 vCPU / 6 GB RAM. Bạn chỉ đặt `--pod-max-pids`
   trên `lab-k8s-worker2`. Một Pod chạy loạn được lập lịch lên `lab-k8s-worker1` — chuyện gì xảy ra, và
   bài khuyên làm gì để khỏi rơi vào tình huống này?
3. Giới hạn PID theo từng Pod có đủ để kubelet và kube-proxy trên node không bị cạn PID không?
   Nếu không thì cơ chế nào lo việc đó, và nó khai ở đâu?
4. Tín hiệu eviction `pid.available` có phải là cơ chế cưỡng chế giới hạn PID không? Nếu đặt
   eviction cứng thì node có chắc chắn an toàn không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Khai trên kubelet của node, không phải trong `.spec` của Pod.** Cụ thể là tham số dòng
   lệnh `--pod-max-pids` hoặc trường `PodPidsLimit` trong file cấu hình kubelet; bài nói rõ giới
   hạn PID định nghĩa ở cấp Pod **hiện chưa được hỗ trợ**. Vì mỗi Node có thể có một giới hạn
   PID khác nhau, **giới hạn thực tế áp lên một Pod phụ thuộc vào nơi nó được lập lịch tới** —
   hai bản sao giống hệt nhau rơi lên hai node cấu hình khác nhau sẽ nhận hai mức khác nhau. Đây
   là chỗ PID lệch hẳn khỏi `requests`/`limits`, vốn đi theo Pod.
2. **Pod đó không bị chặn gì cả trên `lab-k8s-worker1`** — không có giới hạn thì không có gì để
   cưỡng chế, và nó có thể ngốn PID tới mức làm mất ổn định node. Bài đưa ra đúng lời khuyên cho
   trường hợp này trong ghi chú thận trọng: vì giới hạn áp cho một Pod thay đổi theo nơi Pod
   được lập lịch, **cách dễ nhất là để tất cả các Node dùng cùng một mức giới hạn và dự trữ tài
   nguyên PID**. Nghĩa là cấu hình phải đặt trên cả `lab-k8s-worker1` lẫn `lab-k8s-worker2`, không phải
   chỉ trên node bạn đang thử nghiệm.
3. **Không đủ.** Bài nói thẳng: giới hạn PID theo từng Pod cho phép quản trị viên bảo vệ Pod này
   khỏi Pod khác, nhưng **không đảm bảo mọi Pod được lập lịch lên host đó không tác động đến
   node nói chung**, và **cũng không bảo vệ được chính các agent của node** khỏi cạn PID. Việc
   đó thuộc về **dự trữ PID cho node**: tham số `pid=<number>` trong các tùy chọn
   `--system-reserved` và `--kube-reserved` của kubelet, dành riêng một lượng PID cho toàn hệ
   thống và cho các daemon hệ thống của Kubernetes. Hai cơ chế bổ sung nhau, không thay thế nhau.
4. **Không, `pid.available` không cưỡng chế gì cả.** Trực giác "đặt hard eviction là xong" sai vì
   nó lẫn hai loại cơ chế: bài ghi rõ giá trị của tín hiệu eviction được **tính toán định kỳ và
   KHÔNG cưỡng chế giới hạn** — nó chỉ quan sát rồi chấm dứt Pod khi vượt ngưỡng. Vì việc quan
   sát là định kỳ, **nếu số PID tăng rất nhanh thì node vẫn có thể rơi vào trạng thái mất ổn
   định do chạm giới hạn PID của node**, kể cả với chính sách eviction cứng. Thứ đặt ra giới hạn
   cứng thật sự là **giới hạn PID theo từng Pod và theo từng Node**: chạm tới là workload lập
   tức gặp lỗi khi cố xin một PID mới.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

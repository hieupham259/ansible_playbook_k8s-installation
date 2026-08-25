# Quản lý tài nguyên cho các node Windows (Resource Management for Windows nodes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/configuration/windows-resource-management/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 15](00-ALO-TRINH-ADMIN.md#giai-đoạn-15--windows-nếu-môi-trường-có-node-windows),
bài 6/7 · Kiểm chứng ở [Lab 15](labs/LAB-15-NODE-WINDOWS.md).

**Lộ trình ghi rõ: bỏ qua hoàn toàn giai đoạn 15 nếu cluster của bạn chỉ có Linux.** Cluster lab
ba VM Ubuntu của bạn không có node Windows.

Bài ngắn nhưng là bài **quan trọng nhất về mặt vận hành** trong giai đoạn: nó nói vì sao những
phản xạ bạn xây dựng ở giai đoạn 3 và 7 — requests bảo đảm tài nguyên, limits được kernel ép,
OOM kill bảo vệ node — **không còn đúng** trên node Windows. Đọc bài này ngay sau bài
[110](110-manage-resources-containers-vi.md) trong đầu để thấy rõ độ lệch.

**Phải hiểu ở lần đọc này:**

- Ranh giới tài nguyên khác nhau: Linux dùng **cgroup làm ranh giới của pod**; Windows dùng
  **một job object cho mỗi container** cộng bộ lọc namespace hệ thống. **Job object của Windows
  không liên quan gì tới Kubernetes Job.**
- **Không thể chạy container Windows mà không có bộ lọc namespace này**, nên đặc quyền hệ thống
  không thực thi được trong ngữ cảnh host và **container đặc quyền không khả dụng trên Windows**;
  container cũng không mượn được danh tính host vì **SAM tách biệt**.
- Bộ nhớ: **Windows không có cơ chế kết thúc tiến trình khi hết bộ nhớ như Linux**. Mọi cấp phát
  user-mode là bộ nhớ ảo, **pagefile là bắt buộc**, và node **không overcommit** bộ nhớ. Hệ quả:
  hết RAM thì tiến trình **chuyển trang xuống đĩa và chậm đi**, chứ không bị kill.
- CPU: Windows **giới hạn được** thời gian CPU nhưng **không bảo đảm được mức tối thiểu**. Flag
  `--windows-priorityclass` đặt mức ưu tiên cho tiến trình kubelet; nên đặt từ
  `ABOVE_NORMAL_PRIORITY_CLASS` trở lên để Pod không chiếm hết chu kỳ CPU của kubelet.
- Dành riêng tài nguyên: trên Windows, `--kube-reserved` và `--system-reserved` **chỉ được dùng
  để tính `NodeAllocatable`** — chúng không giữ chỗ thật. Vì vậy bài **cảnh báo** phải đặt limit
  cho container; Pod không có limit có thể làm node Windows bị cấp phát quá mức và **trở nên
  không lành mạnh**. Khuyến nghị dành riêng ít nhất **2GiB** bộ nhớ.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Danh sách giá trị hợp lệ của Windows Priority Class | tra khi cấu hình kubelet trên node Windows | khi môi trường thực sự có node Windows |
| Cách chọn con số CPU dành riêng theo mật độ pod tối đa | phải đo trên node thật mới có số | khi môi trường thực sự có node Windows |
| Khái niệm `requests`/`limits` và `NodeAllocatable` nói chung | đã học kỹ rồi | giai đoạn 3 — bài [110](110-manage-resources-containers-vi.md) |
| Nền tảng cgroup v2 mà bài lấy làm mốc so sánh | đã học ở phần container | giai đoạn 2 — bài [33](33-cgroups-vi.md) |

---

Trang này trình bày những khác biệt trong cách quản lý tài nguyên giữa Linux và Windows.

Trên các node Linux, cgroups được dùng
làm ranh giới của pod để kiểm soát tài nguyên. Các container được tạo bên trong ranh giới đó
nhằm cô lập về mạng, tiến trình và hệ thống file. Các API cgroup của Linux có thể được dùng để
thu thập số liệu thống kê về mức sử dụng CPU, I/O và bộ nhớ.

Ngược lại, Windows dùng một [_job object_](https://docs.microsoft.com/windows/win32/procthread/job-objects) cho mỗi container, kết hợp với một bộ lọc namespace hệ thống (system namespace filter)
để chứa toàn bộ các tiến trình trong một container và cung cấp sự cô lập logic khỏi
host.
(Job object là một cơ chế cô lập tiến trình của Windows và khác với
khái niệm mà Kubernetes gọi là Job).

Không có cách nào để chạy một container Windows mà không có cơ chế lọc namespace này.
Điều đó có nghĩa là các đặc quyền hệ thống (system privilege) không thể được thực thi trong ngữ cảnh của
host, và do đó các container đặc quyền (privileged container) không khả dụng trên Windows.
Container không thể mượn danh tính từ host vì Security Account Manager
(SAM) là tách biệt.

## Quản lý bộ nhớ (Memory management) {#resource-management-memory}

Windows không có cơ chế kết thúc tiến trình khi hết bộ nhớ (out-of-memory process killer) như Linux. Windows luôn
coi mọi cấp phát bộ nhớ ở chế độ người dùng (user-mode) là bộ nhớ ảo, và pagefile là bắt buộc.

Các node Windows không cấp phát vượt mức (overcommit) bộ nhớ cho các tiến trình. Hệ quả
là Windows sẽ không rơi vào tình trạng hết bộ nhớ theo cách mà Linux
gặp phải, và các tiến trình sẽ chuyển trang (page) xuống đĩa thay vì bị kết thúc do hết bộ nhớ (OOM
termination). Nếu bộ nhớ bị cấp phát quá mức (over-provisioned) và toàn bộ bộ nhớ vật lý bị dùng cạn,
thì việc chuyển trang có thể làm chậm hiệu năng.

## Quản lý CPU (CPU management) {#resource-management-cpu}

Windows có thể giới hạn lượng thời gian CPU cấp cho các tiến trình khác nhau nhưng không thể
bảo đảm một lượng thời gian CPU tối thiểu.

Trên Windows, kubelet hỗ trợ một flag dòng lệnh để đặt
[mức ưu tiên lập lịch (scheduling priority)](https://docs.microsoft.com/windows/win32/procthread/scheduling-priorities) cho
tiến trình kubelet: `--windows-priorityclass`. Flag này cho phép tiến trình kubelet nhận được
nhiều lát thời gian CPU (CPU time slice) hơn so với các tiến trình khác đang chạy trên host Windows.
Thông tin chi tiết về các giá trị hợp lệ và ý nghĩa của chúng có tại
[Windows Priority Classes](https://docs.microsoft.com/en-us/windows/win32/procthread/scheduling-priorities#priority-class).
Để bảo đảm các Pod đang chạy không chiếm hết chu kỳ CPU của kubelet, hãy đặt flag này ở mức `ABOVE_NORMAL_PRIORITY_CLASS` trở lên.

## Dành riêng tài nguyên (Resource reservation) {#resource-reservation}

Để tính đến lượng bộ nhớ và CPU mà hệ điều hành, container runtime và
các tiến trình host của Kubernetes như kubelet sử dụng, bạn có thể (và nên) dành riêng (reserve)
tài nguyên bộ nhớ và CPU bằng các flag `--kube-reserved` và/hoặc `--system-reserved` của kubelet.
Trên Windows, các giá trị này chỉ được dùng để tính toán tài nguyên
[có thể cấp phát (allocatable)](253-reserve-compute-resources-vi.md#node-allocatable) của node.

> **Thận trọng:**
> Khi triển khai workload, hãy đặt giới hạn (limit) tài nguyên bộ nhớ và CPU cho các container.
> Việc này cũng trừ vào `NodeAllocatable` và giúp scheduler toàn cluster xác định nên đặt pod nào lên node nào.
>
> Lập lịch các pod không có limit có thể khiến các node Windows bị cấp phát quá mức và trong những trường hợp
> cực đoan có thể làm cho node trở nên không lành mạnh (unhealthy).

Trên Windows, một thực hành tốt là dành riêng ít nhất 2GiB bộ nhớ.

Để xác định lượng CPU cần dành riêng,
hãy xác định mật độ pod tối đa cho mỗi node và theo dõi mức sử dụng CPU của
các dịch vụ hệ thống đang chạy trên đó, sau đó chọn một giá trị đáp ứng được nhu cầu workload của bạn.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 15:

1. Câu bẫy: trên node Ubuntu, ranh giới kiểm soát tài nguyên là cgroup của **pod**. Trên Windows
   ranh giới là gì và ở mức nào? Và "job object" có liên quan gì tới Kubernetes Job không?
2. Trên node Ubuntu, container vượt memory limit bị kernel OOM kill. Trên node Windows điều gì
   xảy ra thay vào đó, và hậu quả với node là gì?
3. `requests.cpu` trên Linux bảo đảm phần CPU tối thiểu cho container qua cgroup. Bài nói gì về
   khả năng bảo đảm CPU trên Windows, và Kubernetes bù lại bằng cơ chế nào cho riêng kubelet?
4. Vì sao bài cảnh báo mạnh về việc lập lịch Pod **không có limit** lên node Windows, trong khi
   bạn đã đặt `--kube-reserved` và `--system-reserved` rồi?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Windows dùng **một job object cho mỗi container**, kết hợp với **bộ lọc namespace hệ thống**,
   để chứa toàn bộ tiến trình trong container và cô lập logic khỏi host. Hai khác biệt so với
   Linux: ranh giới ở mức **container** chứ không phải pod, và cơ chế là job object chứ không
   phải cgroup. **Không liên quan gì tới Kubernetes Job** — bài phải ghi chú riêng điều này:
   "Job object là một cơ chế cô lập tiến trình của Windows và **khác với khái niệm mà Kubernetes
   gọi là Job**." Trùng tên là bẫy thuần túy. Hệ quả kèm theo: vì không thể chạy container Windows
   mà không có bộ lọc namespace, **container đặc quyền không khả dụng trên Windows**.
2. **Không có gì bị kill.** "Windows không có cơ chế kết thúc tiến trình khi hết bộ nhớ
   (out-of-memory process killer) như Linux": mọi cấp phát user-mode được coi là bộ nhớ ảo và
   **pagefile là bắt buộc**, node **không overcommit** bộ nhớ. Vì vậy Windows "sẽ không rơi vào
   tình trạng hết bộ nhớ theo cách mà Linux gặp phải, và các tiến trình sẽ **chuyển trang xuống
   đĩa** thay vì bị kết thúc". Hậu quả không phải Pod chết mà là **hiệu năng tụt** khi bộ nhớ bị
   cấp phát quá mức và RAM vật lý dùng cạn — kiểu hỏng âm thầm, khó phát hiện hơn OOM kill.
3. Bài viết: Windows "**có thể giới hạn** lượng thời gian CPU cấp cho các tiến trình khác nhau
   nhưng **không thể bảo đảm** một lượng thời gian CPU tối thiểu". Tức nửa `limits` thì có, nửa
   `requests` theo nghĩa bảo đảm thì không. Riêng cho kubelet, Kubernetes bù bằng flag
   **`--windows-priorityclass`**, đặt mức ưu tiên lập lịch cho tiến trình kubelet; bài khuyên đặt
   **`ABOVE_NORMAL_PRIORITY_CLASS` trở lên** để Pod đang chạy không chiếm hết chu kỳ CPU của
   kubelet.
4. Vì trên Windows hai flag đó **chỉ được dùng để tính toán `NodeAllocatable`**, chúng không thực
   sự giữ chỗ tài nguyên cho hệ điều hành hay kubelet. Thứ duy nhất còn tác dụng thật là **limit
   trên container**: nó cũng trừ vào `NodeAllocatable` và giúp scheduler biết đặt pod nào lên node
   nào. Bài cảnh báo: "Lập lịch các pod không có limit có thể khiến các node Windows bị cấp phát
   quá mức và trong những trường hợp cực đoan có thể làm cho node **trở nên không lành mạnh**."
   Ghép với câu 2 thì thấy rõ vì sao nguy hiểm hơn Linux: không có OOM killer nào dọn dẹp giúp
   bạn, và bài [175](175-windows-intro-vi.md) cũng nói kubelet trên Windows không làm OOM
   eviction.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

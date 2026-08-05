# Quản lý tài nguyên cho các node Windows (Resource Management for Windows nodes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/configuration/windows-resource-management/>

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
[có thể cấp phát (allocatable)](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#node-allocatable) của node.

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

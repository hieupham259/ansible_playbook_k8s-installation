# Giới thiệu về cgroup v2 (About cgroup v2)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/architecture/cgroups/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 2](00-ALO-TRINH-ADMIN.md#giai-đoạn-2--container-và-runtime), bài 6/8 ·
Kiểm chứng ở Lab 2 (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Lộ trình gọi đây là **nền tảng của mọi giới hạn tài nguyên học ở giai đoạn 3**. Bạn cũng đã
chạy `stat -fc %T /sys/fs/cgroup` ở [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) như một gate mà chưa
biết nó kiểm cái gì.

**Phải hiểu ở lần đọc này:**

- cgroup là cơ chế của **Linux kernel** giới hạn tài nguyên cho tiến trình. Kubelet và runtime
  đều phải nói chuyện với nó để thực thi `requests` và `limits`.
- Lệnh kiểm tra phiên bản: `stat -fc %T /sys/fs/cgroup/` → **`cgroup2fs`** là v2, **`tmpfs`**
  là v1.
- **cgroup driver phải khớp** giữa kubelet và container runtime — cả hai cùng `systemd`. Đây
  là yêu cầu được nêu trong mục *Yêu cầu*, và là lỗi cấu hình kinh điển nhất khi tự dựng
  cluster.
- Kubelet **tự phát hiện** cgroup v2, không cần cấu hình thêm.
- cgroup v1 đã **deprecated** từ v1.35: mặc định kubelet **không khởi động** trên node dùng
  cgroup v1.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Danh sách cải tiến kỹ thuật của v2 (PSI, sub-tree delegation) | chi tiết kernel | không cần |
| MemoryQoS | cần hiểu QoS class trước | giai đoạn 3, bài [54](54-pod-qos-vi.md) |
| Danh sách phiên bản Java, Node.js, cAdvisor | là danh sách tra cứu khi di chuyển | tra khi cần |
| Cách bật cgroup v2 bằng GRUB | Ubuntu 24.04 đã bật sẵn | không cần cho lab này |

Điều quan trọng nhất mang sang giai đoạn 3: **cgroup chính là thứ biến `limits` trong YAML
thành ràng buộc thật trên node.**

---

Trên Linux, các nhóm điều khiển (control group) giới hạn tài nguyên được cấp phát
cho các tiến trình (process).

Kubelet và container runtime bên dưới cần giao tiếp với cgroup để thực thi
[quản lý tài nguyên cho pod và container](110-manage-resources-containers-vi.md),
bao gồm request và limit về cpu/memory cho các workload chạy trong container.

Có hai phiên bản cgroup trên Linux: cgroup v1 và cgroup v2. cgroup v2 là
thế hệ mới của API `cgroup`.

## cgroup v2 là gì? (What is cgroup v2?) {#cgroup-v2}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.25 [stable]`

cgroup v2 là phiên bản kế tiếp của API `cgroup` trên Linux. cgroup v2 cung cấp
một hệ thống điều khiển hợp nhất với các khả năng quản lý tài nguyên được nâng cao.

cgroup v2 mang lại một số cải tiến so với cgroup v1, chẳng hạn như:

- Thiết kế phân cấp hợp nhất duy nhất trong API
- Ủy quyền cây con (sub-tree delegation) cho container an toàn hơn
- Các tính năng mới hơn như [Pressure Stall Information](https://www.kernel.org/doc/html/latest/accounting/psi.html)
- Quản lý cấp phát tài nguyên và cách ly được nâng cao trên nhiều loại tài nguyên
  - Thống kê (accounting) hợp nhất cho các kiểu cấp phát bộ nhớ khác nhau (bộ nhớ mạng, bộ nhớ kernel, v.v.)
  - Thống kê cho các thay đổi tài nguyên không tức thời, chẳng hạn như ghi ngược page cache (page cache write back)

Một số tính năng của Kubernetes chỉ dùng cgroup v2 để tăng cường quản lý và
cách ly tài nguyên. Ví dụ, tính năng
[MemoryQoS](54-pod-qos-vi.md#memory-qos-with-cgroup-v2)
cải thiện QoS về bộ nhớ và dựa trên các thành phần nguyên thủy (primitive) của cgroup v2.

## Sử dụng cgroup v2 (Using cgroup v2) {#using-cgroupv2}

Cách được khuyến nghị để dùng cgroup v2 là dùng một bản phân phối Linux
(Linux distribution) bật và dùng cgroup v2 theo mặc định.

Để kiểm tra bản phân phối của bạn có dùng cgroup v2 hay không, xem
[Xác định phiên bản cgroup trên các node Linux](#check-cgroup-version).

### Yêu cầu (Requirements) {#requirements}

cgroup v2 có các yêu cầu sau:

* Bản phân phối hệ điều hành bật cgroup v2
* Phiên bản Linux Kernel là 5.8 trở lên
* Container runtime hỗ trợ cgroup v2. Ví dụ:
  * [containerd](https://containerd.io/) v1.4 trở lên
  * [cri-o](https://cri-o.io/) v1.20 trở lên
* Kubelet và container runtime được cấu hình để dùng [systemd cgroup driver](00-container-runtimes-vi.md#systemd-cgroup-driver)

### Hỗ trợ cgroup v2 của các bản phân phối Linux (Linux Distribution cgroup v2 support)

Để xem danh sách các bản phân phối Linux dùng cgroup v2, tham khảo
[tài liệu cgroup v2](https://github.com/opencontainers/runc/blob/main/docs/cgroup-v2.md)

* Container Optimized OS (từ M97)
* Ubuntu (từ 21.10, khuyến nghị 22.04+)
* Debian GNU/Linux (từ Debian 11 bullseye)
* Fedora (từ 31)
* Arch Linux (từ tháng 4 năm 2021)
* RHEL và các bản phân phối tương tự RHEL (từ 9)

Để kiểm tra bản phân phối của bạn có đang dùng cgroup v2 hay không, hãy tham khảo
tài liệu của bản phân phối đó hoặc làm theo hướng dẫn trong
[Xác định phiên bản cgroup trên các node Linux](#check-cgroup-version).

Bạn cũng có thể bật cgroup v2 thủ công trên bản phân phối Linux của mình bằng cách
sửa các tham số khởi động dòng lệnh của kernel (kernel cmdline boot arguments).
Nếu bản phân phối của bạn dùng GRUB, cần thêm `systemd.unified_cgroup_hierarchy=1`
vào `GRUB_CMDLINE_LINUX` trong `/etc/default/grub`, sau đó chạy `sudo update-grub`.
Tuy nhiên, cách tiếp cận được khuyến nghị vẫn là dùng một bản phân phối đã bật sẵn
cgroup v2 theo mặc định.

### Di chuyển sang cgroup v2 (Migrating to cgroup v2) {#migrating-cgroupv2}

Để di chuyển sang cgroup v2, hãy đảm bảo bạn đáp ứng các [yêu cầu](#requirements),
sau đó nâng cấp lên một phiên bản kernel bật cgroup v2 theo mặc định.

Kubelet tự động phát hiện hệ điều hành đang chạy trên cgroup v2 và hoạt động
tương ứng mà không cần cấu hình gì thêm.

Sẽ không có khác biệt đáng chú ý nào trong trải nghiệm người dùng khi chuyển sang
cgroup v2, trừ khi người dùng truy cập trực tiếp vào hệ thống file cgroup,
dù là trên node hay từ bên trong các container.

cgroup v2 dùng một API khác với cgroup v1, do đó nếu có ứng dụng nào truy cập
trực tiếp vào hệ thống file cgroup, chúng cần được cập nhật lên các phiên bản
mới hơn có hỗ trợ cgroup v2. Ví dụ:

* Một số agent giám sát (monitoring) và bảo mật của bên thứ ba có thể phụ thuộc vào
  hệ thống file cgroup. Hãy cập nhật các agent này lên phiên bản hỗ trợ cgroup v2.
* Nếu bạn chạy [cAdvisor](https://github.com/google/cadvisor) như một DaemonSet
  độc lập để giám sát pod và container, hãy cập nhật nó lên v0.43.0 trở lên.
* Nếu bạn triển khai các ứng dụng Java, ưu tiên dùng các phiên bản hỗ trợ đầy đủ cgroup v2:
    * [OpenJDK / HotSpot](https://bugs.openjdk.org/browse/JDK-8230305): jdk8u372, 11.0.16, 15 trở lên
    * [IBM Semeru Runtimes](https://www.ibm.com/support/pages/apar/IJ46681): 8.0.382.0, 11.0.20.0, 17.0.8.0 trở lên
    * [IBM Java](https://www.ibm.com/support/pages/apar/IJ46681): 8.0.8.6 trở lên
* Nếu bạn dùng package [uber-go/automaxprocs](https://github.com/uber-go/automaxprocs),
  hãy đảm bảo phiên bản bạn dùng là v1.5.1 trở lên.
* Nếu bạn triển khai các ứng dụng [Node.js](https://nodejs.org/), ưu tiên dùng
  các phiên bản phát hiện được giới hạn bộ nhớ của cgroup v2. Node.js đọc giới hạn
  bộ nhớ của cgroup v2 (thông qua [libuv](https://libuv.org/)) kể từ Node.js v20.3.0.
  Dòng phát hành v18 không phát hiện giới hạn bộ nhớ cgroup v2 một cách đáng tin cậy.
  Các phiên bản thiếu hỗ trợ này có thể đọc tổng bộ nhớ của host thay vì
  giới hạn được áp cho pod, điều này có thể dẫn tới kích thước heap không chính xác
  và bị chấm dứt do hết bộ nhớ (out-of-memory - OOM). Trên các phiên bản bị ảnh hưởng,
  hãy đặt kích thước heap một cách tường minh, ví dụ với cờ `--max-old-space-size`.

## Xác định phiên bản cgroup trên các node Linux (Identify the cgroup version on Linux Nodes) {#check-cgroup-version}

Phiên bản cgroup phụ thuộc vào bản phân phối Linux đang được dùng và phiên bản
cgroup mặc định được cấu hình trên hệ điều hành. Để kiểm tra bản phân phối của bạn
dùng phiên bản cgroup nào, chạy lệnh `stat -fc %T /sys/fs/cgroup/` trên node:

```shell
stat -fc %T /sys/fs/cgroup/
```

Với cgroup v2, output là `cgroup2fs`.

Với cgroup v1, output là `tmpfs.`

## Loại bỏ dần cgroup v1 (Deprecation of cgroup v1) {#deprecation-of-cgroup-v1}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [deprecated]`

Kubernetes đã loại bỏ dần (deprecate) cgroup v1.
Việc gỡ bỏ sẽ tuân theo [chính sách loại bỏ của Kubernetes](https://kubernetes.io/docs/reference/using-api/deprecation-policy/).

Theo mặc định, kubelet sẽ không còn khởi động trên một node dùng cgroup v1 nữa.
Để tắt thiết lập này, người quản trị cluster nên đặt `failCgroupV1` thành false
trong [file cấu hình kubelet](224-kubelet-config-file-vi.md).

## Tiếp theo (What's next)

- Tìm hiểu thêm về [cgroups](https://man7.org/linux/man-pages/man7/cgroups.7.html)
- Tìm hiểu thêm về [container runtime](https://kubernetes.io/docs/concepts/architecture/cri)
- Tìm hiểu thêm về [cgroup driver](00-container-runtimes-vi.md#cgroup-drivers)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

1. Lệnh nào cho biết node đang dùng cgroup phiên bản mấy, và hai giá trị output tương ứng với
   phiên bản nào?
2. Ở Lab 00 bạn đặt `SystemdCgroup = true` cho containerd. Nếu kubelet dùng `systemd` còn
   runtime dùng `cgroupfs` thì vấn đề là gì?
3. cgroup nằm ở tầng nào — Kubernetes, container runtime, hay Linux kernel? Nó liên quan thế
   nào tới `limits` bạn viết trong manifest?
4. Bạn nâng một node lên Kubernetes v1.35 nhưng node đó vẫn dùng cgroup v1. Chuyện gì xảy ra
   với kubelet, và có cách nào tạm thời đi tiếp không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. `stat -fc %T /sys/fs/cgroup/`. Output **`cgroup2fs`** là cgroup **v2**, output **`tmpfs`**
   là cgroup **v1**.
2. Hai bên **quản lý hai cây cgroup khác nhau** cho cùng những tiến trình đó. Hệ quả là việc
   thống kê và áp giới hạn tài nguyên trở nên không nhất quán, node dễ rơi vào trạng thái bất
   ổn dưới tải. Bài liệt kê "kubelet và container runtime được cấu hình để dùng systemd cgroup
   driver" là một **yêu cầu**, không phải khuyến nghị.
3. Ở tầng **Linux kernel**. `limits` trong manifest là con số bạn khai báo; kubelet và runtime
   dịch nó thành các thiết lập cgroup trên node, và **chính kernel** mới là thứ thực sự chặn
   tiến trình vượt ngưỡng. Không có cgroup thì `limits` chỉ là chữ trong YAML.
4. **Kubelet mặc định không khởi động.** cgroup v1 đã deprecated từ v1.35. Cách đi tiếp tạm
   thời là đặt `failCgroupV1: false` trong file cấu hình kubelet — nhưng đó là hoãn binh, việc
   đúng là chuyển node sang cgroup v2.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

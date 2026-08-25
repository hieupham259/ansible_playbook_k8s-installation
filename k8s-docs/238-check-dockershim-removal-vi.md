# Kiểm tra xem việc loại bỏ dockershim có ảnh hưởng đến bạn không (Check whether dockershim removal affects you)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/check-if-dockershim-removal-affects-you/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 27 — Di chuyển khỏi dockershim (cluster cũ)](00-ALO-TRINH-ADMIN.md#giai-đoạn-27--di-chuyển-khỏi-dockershim-cluster-cũ),
bài 3/6 · Phần II không có lab riêng: tự chấm bằng **Checkpoint** ghi ở cuối mục giai đoạn 27 trong
lộ trình. Bài này **không kiểm chứng được trên cluster lab** — nó là một danh sách rà soát dành cho
cluster đang chạy Docker Engine, mà ba node lab đều chạy containerd nên không có gì để rà.

**Đây là bài trả lời cho Checkpoint, không phải bài gõ lệnh.** Lộ trình cho phép *bỏ qua toàn bộ
giai đoạn 27 nếu cluster của bạn đã dùng containerd*, và cluster lab thuộc nhóm đó. Nhưng chính bài
này chứa hai trong ba câu mà Checkpoint giai đoạn 27 bắt bạn nói được: **dockershim là gì** và
**phải kiểm những gì trước khi chuyển sang containerd**. Đọc nó như đọc một checklist bàn giao: mai
này nhận một cluster đời cũ, đây là thứ bạn mang theo. Đừng cố dựng một cluster dockershim để tập —
không đáng, và lộ trình không yêu cầu.

**Phải hiểu ở lần đọc này:**

- Ranh giới mà bài đặt ngay câu đầu: dùng Docker để **build** container **không** tính là phụ thuộc
  vào Docker với vai trò container runtime. Image build ra vẫn chạy được trên bất kỳ runtime nào.
  Thứ bị ảnh hưởng là việc **thực thi lệnh Docker khi runtime đã đổi**.
- Danh sách năm điểm rà soát của mục *Tìm hiểu xem ứng dụng của bạn có phụ thuộc vào Docker không*:
  Pod đặc quyền chạy lệnh Docker hoặc sửa `/etc/docker/daemon.json`; thiết lập private registry và
  image mirror nằm trong file cấu hình Docker; **script và agent chạy ngoài hạ tầng Kubernetes trên
  node**; công cụ bên thứ ba; và phụ thuộc gián tiếp vào hành vi của dockershim.
- Vì sao dockershim từng tồn tại (mục *Giải thích về sự phụ thuộc vào Docker*): kubelet dùng **CRI
  làm lớp trừu tượng** để chạy được nhiều runtime, nhưng **Docker có trước đặc tả CRI**, nên dự án
  Kubernetes viết adapter `dockershim` để kubelet nói chuyện với Docker như thể Docker tương thích
  CRI.
- Hệ quả cụ thể của việc bỏ khâu trung gian: container được lập lịch **thẳng** với runtime nên
  **không còn hiển thị với Docker** — mất `docker ps`, `docker inspect`, và vì không liệt kê được
  container thì cũng mất luôn đường lấy log, dừng container và `docker exec`. Image build hoặc pull
  bằng Docker **không hiển thị** với runtime và Kubernetes; muốn dùng thì phải push lên registry.
- Lời khuyên vận hành đi kèm, đúng cho **mọi** runtime chứ không riêng Docker: cách tốt nhất để dừng
  một container đang chạy dưới Kubernetes là qua **Kubernetes API**, không phải qua container
  runtime.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Điểm 4 của danh sách — công cụ bên thứ ba làm thao tác đặc quyền | bài này chỉ nêu tên rồi trỏ đi, chi tiết nằm ở bài kế | bài [240](240-migrating-telemetry-agents-vi.md) của chính giai đoạn 27 |
| Mục *Các vấn đề đã biết* — đổi định dạng tên metric và danh sách `container_fs_*` bị thiếu | chỉ áp dụng khi tiếp quản cluster dockershim đang có collector bám vào `/metrics/cadvisor` theo định dạng cũ; cluster lab sinh ra đã ở định dạng của containerd nên không có bước "đổi" nào | nền tảng metric là bài [160](160-system-metrics-vi.md), endpoint đã gọi tay ở [Lab 11a phần B2.3](labs/LAB-11A-OBSERVABILITY.md#b23-bốn-endpoint-của-kubelet) |
| Mục *Giải pháp tạm thời* — chạy cAdvisor như một daemonset độc lập | là bản vá cho cluster đang di chuyển, và là add-on ngoài baseline của Lab 00 | chỉ áp dụng khi tiếp quản cluster dockershim có collector bị hụt metric |

---

Thành phần `dockershim` của Kubernetes cho phép sử dụng Docker làm container runtime của Kubernetes.
Thành phần `dockershim` tích hợp sẵn của Kubernetes đã bị loại bỏ trong bản phát hành v1.24.

Trang này giải thích cluster của bạn có thể đang dùng Docker làm container runtime như thế nào,
cung cấp chi tiết về vai trò của `dockershim` khi được sử dụng, và trình bày các bước
bạn có thể thực hiện để kiểm tra xem có workload nào bị ảnh hưởng bởi việc loại bỏ `dockershim` hay không.

## Tìm hiểu xem ứng dụng của bạn có phụ thuộc vào Docker không (Finding if your app has a dependencies on Docker) {#find-docker-dependencies}

Nếu bạn đang dùng Docker để build các container cho ứng dụng của mình, bạn vẫn có thể
chạy những container này trên bất kỳ container runtime nào. Cách sử dụng Docker này không được tính
là phụ thuộc (dependency) vào Docker với vai trò container runtime.

Khi một container runtime thay thế được sử dụng, việc thực thi các lệnh Docker có thể
không hoạt động hoặc cho ra kết quả không mong muốn. Đây là cách bạn có thể tìm ra liệu mình có
phụ thuộc vào Docker hay không:

1. Đảm bảo không có Pod đặc quyền (privileged) nào thực thi lệnh Docker (như `docker ps`),
   khởi động lại dịch vụ Docker (các lệnh như `systemctl restart docker.service`),
   hoặc chỉnh sửa các file dành riêng cho Docker như `/etc/docker/daemon.json`.
1. Kiểm tra các thiết lập private registry hoặc image mirror trong file cấu hình
   Docker (như `/etc/docker/daemon.json`). Những thiết lập này thường cần được
   cấu hình lại cho container runtime khác.
1. Kiểm tra rằng các script và ứng dụng chạy trên các node bên ngoài hạ tầng
   Kubernetes của bạn không thực thi lệnh Docker. Đó có thể là:
   - SSH vào node để xử lý sự cố;
   - Script khởi động của node;
   - Agent giám sát (monitoring) và bảo mật được cài trực tiếp trên node.
1. Các công cụ bên thứ ba thực hiện các thao tác đặc quyền nêu trên. Xem
   [Di chuyển agent telemetry và bảo mật khỏi dockershim](240-migrating-telemetry-agents-vi.md)
   để biết thêm thông tin.
1. Đảm bảo không có sự phụ thuộc gián tiếp nào vào hành vi của dockershim.
   Đây là trường hợp hiếm và khó có khả năng ảnh hưởng đến ứng dụng của bạn. Một số công cụ có thể được cấu hình
   để phản ứng với các hành vi đặc thù của Docker, ví dụ: phát cảnh báo dựa trên một số metric cụ thể, hoặc tìm kiếm
   một thông điệp log cụ thể như một phần của quy trình xử lý sự cố.
   Nếu bạn có công cụ được cấu hình như vậy, hãy kiểm thử hành vi đó trên một cluster
   thử nghiệm trước khi di chuyển.

## Giải thích về sự phụ thuộc vào Docker (Dependency on Docker explained) {#role-of-dockershim}

[Container runtime](39-containers-vi.md#container-runtimes) là phần mềm có thể
thực thi các container tạo nên một Pod của Kubernetes. Kubernetes chịu trách nhiệm điều phối (orchestration)
và lập lịch (scheduling) các Pod; trên mỗi node, kubelet
sử dụng giao diện container runtime (Container Runtime Interface - CRI) như một lớp trừu tượng để bạn có thể dùng bất kỳ
container runtime nào tương thích.

Trong các bản phát hành đầu tiên, Kubernetes chỉ tương thích với một container runtime duy nhất: Docker.
Về sau trong lịch sử của dự án Kubernetes, những người vận hành cluster muốn dùng thêm các container runtime khác.
CRI được thiết kế để cho phép sự linh hoạt này - và kubelet bắt đầu hỗ trợ CRI. Tuy nhiên,
vì Docker tồn tại trước khi đặc tả CRI ra đời, dự án Kubernetes đã tạo ra một
thành phần chuyển đổi (adapter), `dockershim`. Adapter dockershim cho phép kubelet tương tác với Docker như thể
Docker là một runtime tương thích CRI.

Bạn có thể đọc thêm trong bài blog [Kubernetes Containerd integration goes GA](https://kubernetes.io/blog/2018/05/24/kubernetes-containerd-integration-goes-ga/).

![Dockershim so với CRI với Containerd](https://kubernetes.io/images/blog/2018-05-24-kubernetes-containerd-integration-goes-ga/cri-containerd.png)

Chuyển sang Containerd làm container runtime giúp loại bỏ khâu trung gian. Tất cả
các container như trước đây vẫn có thể được chạy bởi những container runtime như Containerd. Nhưng
giờ đây, vì container được lập lịch trực tiếp với container runtime, chúng không còn hiển thị với Docker.
Vì vậy, bất kỳ công cụ Docker hoặc giao diện đồ họa nào bạn từng dùng
trước đây để kiểm tra các container này đều không còn khả dụng.

Bạn không thể lấy thông tin container bằng các lệnh `docker ps` hay `docker inspect`.
Vì không thể liệt kê container, bạn cũng không thể lấy log, dừng container,
hay thực thi lệnh bên trong container bằng `docker exec`.

> **Ghi chú:**
>
> Nếu bạn đang chạy workload thông qua Kubernetes, cách tốt nhất để dừng một container là thông qua
> Kubernetes API thay vì trực tiếp qua container runtime (lời khuyên này áp dụng
> cho mọi container runtime, không riêng gì Docker).

Bạn vẫn có thể pull image hoặc build image bằng lệnh `docker build`. Nhưng những image
được build hoặc pull bởi Docker sẽ không hiển thị với container runtime và
Kubernetes. Chúng cần được đẩy (push) lên một registry nào đó để Kubernetes
có thể sử dụng.

## Các vấn đề đã biết (Known issues)

### Một số metric filesystem bị thiếu và định dạng metric khác đi (Some filesystem metrics are missing and the metrics format is different)

Endpoint `/metrics/cadvisor` của kubelet cung cấp các metric Prometheus,
như được mô tả trong [Metrics cho các thành phần hệ thống Kubernetes](160-system-metrics-vi.md).
Nếu bạn cài một trình thu thập metric phụ thuộc vào endpoint đó, bạn có thể gặp các vấn đề sau:

- Định dạng metric trên node dùng Docker là `k8s_<container-name>_<pod-name>_<namespace>_<pod-uid>_<restart-count>`
  nhưng định dạng trên runtime khác thì khác. Ví dụ, trên node dùng containerd, nó là `<container-id>`.
- Một số metric filesystem bị thiếu, cụ thể như sau:
  ```
  container_fs_inodes_free
  container_fs_inodes_total
  container_fs_io_current
  container_fs_io_time_seconds_total
  container_fs_io_time_weighted_seconds_total
  container_fs_limit_bytes
  container_fs_read_seconds_total
  container_fs_reads_merged_total
  container_fs_sector_reads_total
  container_fs_sector_writes_total
  container_fs_usage_bytes
  container_fs_write_seconds_total
  container_fs_writes_merged_total
  ```

#### Giải pháp tạm thời (Workaround)

Bạn có thể giảm thiểu vấn đề này bằng cách dùng [cAdvisor](https://github.com/google/cadvisor) như một daemonset độc lập.

1. Tìm [bản phát hành cAdvisor](https://github.com/google/cadvisor/releases) mới nhất
   có tên theo mẫu `vX.Y.Z-containerd-cri` (ví dụ, `v0.42.0-containerd-cri`).
2. Làm theo các bước trong [cAdvisor Kubernetes Daemonset](https://github.com/google/cadvisor/tree/master/deploy/kubernetes) để tạo daemonset.
3. Trỏ trình thu thập metric đã cài sang endpoint `/metrics` của cAdvisor,
   nơi cung cấp đầy đủ bộ
   [metric container Prometheus](https://github.com/google/cadvisor/blob/master/docs/storage/prometheus.md).

Các phương án thay thế:

- Dùng giải pháp thu thập metric của bên thứ ba khác.
- Thu thập metric từ Kubelet summary API được phục vụ tại `/stats/summary`.

## Tiếp theo (What's next)

- Đọc [Di chuyển khỏi dockershim](236-migrating-from-dockershim-vi.md) để hiểu các bước tiếp theo của bạn
- Đọc bài viết [dockershim deprecation FAQ](https://kubernetes.io/blog/2020/12/02/dockershim-faq/) để biết thêm thông tin.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 27. Cả năm câu trả lời
được bằng lập luận, không cần cluster dockershim:

1. **Câu bẫy.** Đội của bạn build mọi image bằng `docker build`, Dockerfile nằm ở mọi repo, CI cũng
   chạy Docker. Theo bài, cluster đó có "phụ thuộc vào Docker" theo nghĩa bị ảnh hưởng bởi việc
   loại bỏ dockershim không?
2. Ba node lab của bạn chạy containerd. Giả sử có người cài Docker CLI lên `lab-k8s-worker2` rồi gõ
   `docker ps` để tìm container của một Pod: theo lập luận của bài, kết quả ra sao và vì sao? Bài
   khuyên cách đúng để **dừng** một container đang chạy dưới Kubernetes là gì?
3. Vì sao dockershim từng phải tồn tại? Trả lời bằng thứ tự thời gian giữa Docker và đặc tả CRI, và
   nói rõ dockershim đứng ở đâu trong đường từ kubelet tới container.
4. Trên một node đang di chuyển, bạn `docker pull` một image về node rồi tạo Pod dùng đúng image
   đó. Vì sao runtime mới vẫn không dùng được image ấy, và bài bảo phải làm gì?
5. Trong năm điểm rà soát, có một điểm nói tới những thứ chạy **ngoài** hạ tầng Kubernetes. Đó là
   những thứ gì, và vì sao chỉ rà soát bên trong cluster là chưa đủ?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không.** Bài tách bạch ngay câu đầu: dùng Docker để **build** container **không được tính là
   phụ thuộc vào Docker với vai trò container runtime** — image build ra vẫn chạy được trên bất kỳ
   container runtime nào. Thứ bị ảnh hưởng là việc **thực thi lệnh Docker sau khi runtime đã đổi**,
   tức những chỗ mà một tiến trình đang chạy trông đợi `dockerd` có mặt. Chỗ dễ nhầm ở đây là lẫn
   giữa *công cụ đóng gói* với *runtime chạy container trên node*: hai vai trò khác nhau.
2. **`docker ps` không thấy container nào của Pod.** Khi chuyển sang containerd, container được
   **lập lịch trực tiếp với container runtime**, nên chúng **không còn hiển thị với Docker**; và vì
   không liệt kê được container thì cũng không lấy được log, không dừng được, không `docker exec`
   được. Về cách dừng container, bài nói thẳng và nói cho **mọi** runtime chứ không riêng Docker:
   **dừng qua Kubernetes API**, không phải trực tiếp qua container runtime.
3. Vì **Docker tồn tại trước khi đặc tả CRI ra đời**. Kubernetes bản đầu chỉ tương thích một
   runtime duy nhất là Docker; về sau người vận hành muốn dùng runtime khác nên CRI được thiết kế
   làm **lớp trừu tượng** giữa kubelet và runtime. Docker thì không nói CRI, nên dự án Kubernetes
   viết **`dockershim` làm thành phần chuyển đổi (adapter)**, nằm **giữa kubelet và Docker**, để
   kubelet tương tác với Docker **như thể** Docker là một runtime tương thích CRI. Chuyển sang
   containerd chính là **bỏ khâu trung gian đó**.
4. Vì image được **build hoặc pull bởi Docker không hiển thị với container runtime và Kubernetes** —
   chúng nằm trong kho riêng của Docker, không phải kho mà runtime mới đọc. Bài chỉ đúng một lối
   ra: **push image lên một registry** rồi để Kubernetes kéo về qua runtime mới. Lưu ý phần này
   không mâu thuẫn với câu 1: `docker build` vẫn dùng được, chỉ là **kết quả phải đi qua registry**.
5. Đó là **script và ứng dụng chạy trên node bên ngoài hạ tầng Kubernetes**, bài liệt kê ba dạng:
   **SSH vào node để xử lý sự cố**, **script khởi động của node**, và **agent giám sát cùng agent
   bảo mật được cài trực tiếp lên node**. Rà soát bên trong cluster là chưa đủ vì những thứ này
   **không phải Pod** — chúng không xuất hiện trong bất kỳ danh sách object nào của API server, nên
   không có lệnh `kubectl` nào lôi chúng ra được; phải rà từ phía node và từ phía quy trình vận
   hành.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

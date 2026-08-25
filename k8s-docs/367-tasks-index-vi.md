# Tác vụ (Tasks)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/>
>
> Phần này của tài liệu Kubernetes chứa các trang hướng dẫn thực hiện từng tác vụ riêng lẻ.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster) — đọc
một lượt **trước khi vào [giai đoạn 16](00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node)**.
Trang này **không mang số `bài N/M` nào**: nó là trang mục gốc của **cả nhánh** `/docs/tasks/`,
nên đứng ngoài danh sách đọc của mọi giai đoạn — đoạn dẫn đầu Phần II gọi tên nó đúng với vai trò
đó. Phần II không có lab: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm bằng **Checkpoint** của từng giai đoạn.

Đây là **bản đồ**, không phải bài học. Đọc trong vài phút để biết nhánh này có những mục gì và tra
nó ở đâu, rồi quay lại lộ trình.

**Phải hiểu ở lần đọc này:**

- Bản chất của cả nhánh, nêu ở đoạn đầu: một trang tác vụ **chỉ hướng dẫn làm một việc duy nhất**,
  thường trình bày thành **một chuỗi bước ngắn gọn**. Đừng tìm phần giải thích khái niệm ở đây.
- Mục *Danh sách các mục trong phần này* liệt kê **17 mục con** theo đúng thứ tự hiển thị trên
  trang web. Thứ tự đó là thứ tự của website, **không phải thứ tự học** — thứ tự học nằm ở
  [lộ trình](00-ALO-TRINH-ADMIN.md), và lộ trình rải 17 mục này ra nhiều giai đoạn khác nhau.
- Đây mới là **tầng một** của bản đồ: nhiều mục trong danh sách lại là một trang mục lục của riêng
  nó — ví dụ [Mạng](391-network-index-vi.md), [TLS](396-tls-index-vi.md),
  [Mở rộng Kubernetes](373-extend-kubernetes-index-vi.md),
  [Quản trị một Cluster](189-administer-cluster-vi.md). Tra một trang tác vụ cụ thể thường phải đi
  qua hai tầng mục lục.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Link *Tạo một Pull Request cho tài liệu* | dành cho người đóng góp tài liệu Kubernetes, không phải người vận hành cluster | không có trong lộ trình |
| Nội dung bên trong từng mục trong số 17 mục | trang này chỉ là bản đồ tầng một | các giai đoạn tương ứng của lộ trình — ví dụ [TLS](396-tls-index-vi.md) ở [giai đoạn 18](00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ), [Mạng](391-network-index-vi.md) ở [giai đoạn 21](00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy), [Mở rộng Kubernetes](373-extend-kubernetes-index-vi.md) ở [giai đoạn 28](00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes) |

---

Phần này của tài liệu Kubernetes chứa các trang hướng dẫn cách thực hiện từng tác vụ riêng lẻ.
Một trang tác vụ chỉ hướng dẫn làm một việc duy nhất, thường được trình bày dưới dạng một chuỗi
các bước ngắn gọn.

Nếu bạn muốn viết một trang tác vụ, hãy xem
[Tạo một Pull Request cho tài liệu (Creating a Documentation Pull Request)](https://kubernetes.io/docs/contribute/new-content/open-a-pr/).

## Danh sách các mục trong phần này (Pages in this section)

Đây là trang mục gốc của toàn nhánh *Tasks*. Danh sách dưới đây liệt kê 17 mục con theo đúng thứ
tự hiển thị trên trang web.

- **[Cài đặt công cụ (Install Tools)](185-tools-vi.md)** — Thiết lập các công cụ Kubernetes
  trên máy tính của bạn.
- **[Quản trị một Cluster (Administer a Cluster)](189-administer-cluster-vi.md)** — Tìm hiểu các
  tác vụ thường gặp khi quản trị một cluster.
- **[Cấu hình Pod và Container (Configure Pods and Containers)](262-configure-pod-container-vi.md)**
  — Thực hiện các tác vụ cấu hình thường gặp cho Pod và container.
- **[Giám sát, ghi log và gỡ lỗi (Monitoring, Logging, and Debugging)](296-debug-vi.md)** — Thiết
  lập giám sát (monitoring) và ghi log để chẩn đoán sự cố của cluster, hoặc gỡ lỗi một ứng dụng
  chạy trong container.
- **[Quản lý các đối tượng Kubernetes (Manage Kubernetes Objects)](318-manage-kubernetes-objects-vi.md)**
  — Các mô hình khai báo (declarative) và mệnh lệnh (imperative) để tương tác với Kubernetes API.
- **[Quản lý Secret (Managing Secrets)](325-configmap-secret-vi.md)** — Quản lý dữ liệu cấu hình
  nhạy cảm bằng Secret.
- **[Đưa dữ liệu vào ứng dụng (Inject Data Into Applications)](329-inject-data-application-vi.md)**
  — Chỉ định cấu hình và các dữ liệu khác cho những Pod đang chạy workload của bạn.
- **[Chạy ứng dụng (Run Applications)](337-run-application-vi.md)** — Chạy và quản lý cả ứng dụng
  stateless lẫn stateful.
- **[Chạy Job (Run Jobs)](349-job-tasks-vi.md)** — Chạy Job bằng xử lý song song (parallel
  processing).
- **[Truy cập ứng dụng trong một cluster (Access Applications in a Cluster)](368-access-application-cluster-index-vi.md)**
  — Cấu hình cân bằng tải (load balancing), chuyển tiếp port (port forwarding), hoặc thiết lập
  firewall và DNS để truy cập các ứng dụng trong một cluster.
- **[Mở rộng Kubernetes (Extend Kubernetes)](373-extend-kubernetes-index-vi.md)** — Tìm hiểu những
  cách nâng cao để điều chỉnh cluster Kubernetes của bạn cho phù hợp với nhu cầu của môi trường
  làm việc.
- **[TLS](396-tls-index-vi.md)** — Tìm hiểu cách bảo vệ lưu lượng bên trong cluster của bạn bằng
  Transport Layer Security (TLS).
- **[Quản lý các daemon của cluster (Manage Cluster Daemons)](384-manage-daemon-index-vi.md)** —
  Thực hiện các tác vụ thường gặp khi quản lý một DaemonSet, chẳng hạn như tiến hành một rolling
  update.
- **[Mạng (Networking)](391-network-index-vi.md)** — Tìm hiểu cách cấu hình mạng (networking) cho
  cluster của bạn.
- **[Mở rộng kubectl bằng plugin (Extend kubectl with plugins)](372-kubectl-plugins-vi.md)** — Mở
  rộng kubectl bằng cách tạo và cài đặt các plugin cho kubectl.
- **[Quản lý HugePages (Manage HugePages)](390-scheduling-hugepages-vi.md)** — Cấu hình và quản lý
  huge page như một tài nguyên có thể lập lịch trong cluster.
- **[Lập lịch GPU (Schedule GPUs)](389-scheduling-gpus-vi.md)** — Cấu hình và lập lịch GPU để các
  node trong cluster sử dụng như một tài nguyên.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc trước khi vào giai đoạn 16:

1. Trang liệt kê 17 mục con theo thứ tự nào, và thứ tự đó có phải thứ tự bạn nên học không? Nếu
   không thì thứ tự học lấy ở đâu?
2. **Câu bẫy.** Đây là trang mục gốc của **toàn** nhánh *Tasks*, nên dễ nghĩ mở nó ra là thấy hết
   mọi trang tác vụ. Thực tế bạn thấy gì ở tầng này, và muốn tới một trang tác vụ cụ thể thì phải
   đi qua mấy tầng mục lục?
3. Bạn đang ngồi trước `lab-k8s-master` và cần làm một việc cụ thể trên cluster. Theo đúng mô tả
   của trang, một trang tác vụ cho bạn cái gì — và **không** cho bạn cái gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Theo **đúng thứ tự hiển thị trên trang web** của kubernetes.io. Đó **không** phải thứ tự học:
   trang này không nói gì về việc nên đọc mục nào trước. Thứ tự học nằm ở
   [lộ trình](00-ALO-TRINH-ADMIN.md), và lộ trình **rải 17 mục này ra nhiều giai đoạn khác nhau**
   thay vì đi tuần tự theo danh sách.
2. Ở tầng này bạn chỉ thấy **17 mục cấp một cùng một câu mô tả mỗi mục**, không thấy trang tác vụ
   nào. Nhiều mục trong đó lại là **một trang mục lục của riêng nó** —
   [Mạng](391-network-index-vi.md), [TLS](396-tls-index-vi.md),
   [Mở rộng Kubernetes](373-extend-kubernetes-index-vi.md),
   [Quản trị một Cluster](189-administer-cluster-vi.md) — nên tới được một trang tác vụ cụ thể
   thường phải qua **hai tầng** mục lục. Nghĩ rằng một tầng là đủ sẽ khiến bạn kết luận nhầm rằng
   tài liệu không có trang cho việc mình cần.
3. Nó cho bạn **hướng dẫn làm đúng một việc**, dạng **chuỗi bước ngắn gọn** — đúng thứ cần khi đã
   biết mình phải làm gì. Nó **không** cho bạn phần giải thích khái niệm, không cho bạn bối cảnh,
   và không nói việc đó nằm ở đâu trong bức tranh chung. Vì vậy cách dùng đúng là: tra nhánh này
   **khi đã biết tên việc cần làm**, còn muốn hiểu vì sao thì quay về bài khái niệm mà lộ trình
   chỉ tới.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.

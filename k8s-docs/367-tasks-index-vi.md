# Tác vụ (Tasks)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/>
>
> Phần này của tài liệu Kubernetes chứa các trang hướng dẫn thực hiện từng tác vụ riêng lẻ.

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

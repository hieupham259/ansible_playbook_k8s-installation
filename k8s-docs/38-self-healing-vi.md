# Khả năng tự phục hồi của Kubernetes (Kubernetes Self-Healing)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/architecture/self-healing/>

Kubernetes được thiết kế với các khả năng tự phục hồi (self-healing) giúp duy trì sức
khỏe và tính khả dụng của các workload. Nó tự động thay thế các container bị lỗi, lên
lịch lại (reschedule) các workload khi node trở nên không khả dụng, và đảm bảo trạng
thái mong muốn của hệ thống luôn được duy trì.

## Các khả năng tự phục hồi (Self-Healing capabilities) {#self-healing-capabilities}

- **Khởi động lại ở cấp container (Container-level restarts):** Nếu một container bên
  trong Pod bị lỗi, Kubernetes sẽ khởi động lại nó dựa trên
  [`restartPolicy`](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy).

- **Thay thế replica (Replica replacement):** Nếu một Pod trong
  [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) hoặc
  [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
  bị lỗi, Kubernetes sẽ tạo một Pod thay thế để duy trì số lượng replica đã chỉ định.
  Nếu một Pod thuộc [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
  bị lỗi, control plane sẽ tạo một Pod thay thế để chạy trên chính node đó.

- **Khôi phục lưu trữ bền vững (Persistent storage recovery):** Nếu một node đang chạy
  Pod có gắn PersistentVolume (PV) và node đó gặp sự cố, Kubernetes có thể gắn lại
  (reattach) volume vào một Pod mới trên một node khác.

- **Cân bằng tải cho Service (Load balancing for Services):** Nếu một Pod đứng sau một
  [Service](https://kubernetes.io/docs/concepts/services-networking/service/) bị lỗi,
  Kubernetes sẽ tự động loại bỏ nó khỏi các endpoint của Service để chỉ định tuyến lưu
  lượng (traffic) tới các Pod khỏe mạnh.

Dưới đây là một số thành phần chính cung cấp khả năng tự phục hồi của Kubernetes:

- **[kubelet](./22-architecture-vi.md#kubelet):** Đảm bảo các container đang chạy, và
  khởi động lại những container bị lỗi.

- **Các controller Deployment (thông qua ReplicaSet), ReplicaSet, StatefulSet và
  DaemonSet:** Duy trì số lượng Pod replica mong muốn.

- **PersistentVolume controller:** Quản lý việc gắn (attach) và tháo (detach) volume
  cho các workload có trạng thái (stateful).

## Những điểm cần cân nhắc (Considerations) {#considerations}

- **Lỗi lưu trữ (Storage Failures):** Nếu một persistent volume trở nên không khả dụng,
  có thể cần thực hiện các bước khôi phục.

- **Lỗi ứng dụng (Application Errors):** Kubernetes có thể khởi động lại container,
  nhưng các vấn đề nằm ở bản thân ứng dụng phải được xử lý riêng.

## Tiếp theo (What's next)

- Đọc thêm về [Pod](https://kubernetes.io/docs/concepts/workloads/pods/)
- Tìm hiểu về [các Controller của Kubernetes](./25-controllers-vi.md)
- Khám phá [PersistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- Đọc về [tự động co giãn node (node autoscaling)](https://kubernetes.io/docs/concepts/cluster-administration/node-autoscaling/).
  Tự động co giãn node cũng cung cấp khả năng tự phục hồi nếu hoặc khi các node trong
  cluster của bạn gặp sự cố.

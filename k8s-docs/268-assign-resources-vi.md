# Gán thiết bị cho Pod và Container (Assign Devices to Pods and Containers)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/
>
> Gán tài nguyên hạ tầng cho các workload Kubernetes của bạn.

Đây là trang mục lục (section index). Trang gốc chỉ gồm tiêu đề, phần mô tả ở trên và danh
sách các trang con dưới đây; nội dung chi tiết nằm trong từng trang con:

* [Thiết lập DRA trong một cluster (Set Up DRA in a Cluster)](https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/set-up-dra-cluster/) —
  Trang này hướng dẫn cách cấu hình cấp phát tài nguyên động (dynamic resource allocation — DRA)
  trong một cluster Kubernetes bằng cách bật các API group và cấu hình các lớp thiết bị
  (classes of devices). Các hướng dẫn này dành cho quản trị viên cluster.
* [Cấp phát thiết bị cho workload bằng DRA (Allocate Devices to Workloads with DRA)](https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/allocate-devices-dra/) —
  Trang này hướng dẫn cách cấp phát thiết bị cho các Pod của bạn bằng cấp phát tài nguyên động
  (DRA). Các hướng dẫn này dành cho người vận hành workload.
* [Truy cập metadata thiết bị DRA (Access DRA Device Metadata)](https://kubernetes.io/docs/tasks/configure-pod-container/assign-resources/access-dra-device-metadata/) —
  Trang này hướng dẫn cách truy cập metadata của thiết bị từ các container sử dụng cấp phát
  tài nguyên động (DRA), bằng cách đọc các file JSON tại những đường dẫn quy ước bên trong
  container.

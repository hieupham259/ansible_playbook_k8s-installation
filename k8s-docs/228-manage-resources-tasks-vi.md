# Quản lý tài nguyên Memory, CPU và API (Manage Memory, CPU, and API Resources)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/>

Đây là trang mục lục của mục tác vụ (task) *Manage Memory, CPU, and API Resources* trong tài
liệu Kubernetes: các tác vụ dành cho người quản trị cluster để đặt giá trị mặc định, ràng buộc
và quota cho tài nguyên memory, CPU trong từng namespace.

---

Các trang trong mục này:

* [Cấu hình memory request và limit mặc định cho một Namespace (Configure Default Memory Requests and Limits for a Namespace)](232-memory-default-namespace-vi.md)

  Định nghĩa memory resource limit mặc định cho một namespace, để mọi Pod mới trong namespace
  đó được cấu hình sẵn memory resource limit.

* [Cấu hình CPU request và limit mặc định cho một Namespace (Configure Default CPU Requests and Limits for a Namespace)](230-cpu-default-namespace-vi.md)

  Định nghĩa CPU resource limit mặc định cho một namespace, để mọi Pod mới trong namespace đó
  được cấu hình sẵn CPU resource limit.

* [Cấu hình ràng buộc memory tối thiểu và tối đa cho một Namespace (Configure Minimum and Maximum Memory Constraints for a Namespace)](231-memory-constraint-namespace-vi.md)

  Định nghĩa một khoảng giá trị memory resource limit hợp lệ cho một namespace, để mọi Pod
  mới trong namespace đó nằm trong khoảng mà bạn cấu hình.

* [Cấu hình ràng buộc CPU tối thiểu và tối đa cho một Namespace (Configure Minimum and Maximum CPU Constraints for a Namespace)](./229-cpu-constraint-namespace-vi.md)

  Định nghĩa một khoảng giá trị CPU resource limit hợp lệ cho một namespace, để mọi Pod mới
  trong namespace đó nằm trong khoảng mà bạn cấu hình.

* [Cấu hình quota memory và CPU cho một Namespace (Configure Memory and CPU Quotas for a Namespace)](233-quota-memory-cpu-namespace-vi.md)

  Định nghĩa giới hạn tổng thể về tài nguyên memory và CPU cho một namespace.

* [Cấu hình quota Pod cho một Namespace (Configure a Pod Quota for a Namespace)](234-quota-pod-namespace-vi.md)

  Hạn chế số lượng Pod bạn có thể tạo trong một namespace.

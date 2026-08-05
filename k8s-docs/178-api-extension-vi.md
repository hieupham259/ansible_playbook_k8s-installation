# Mở rộng Kubernetes API (Extending the Kubernetes API)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/>

Custom resource là các phần mở rộng của Kubernetes API. Kubernetes cung cấp hai cách để thêm custom resource vào cluster của bạn:

- Cơ chế [CustomResourceDefinition](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
  (CRD) cho phép bạn định nghĩa một custom API mới theo cách khai báo (declarative) với API group, kind và
  schema do bạn chỉ định.
  Control plane của Kubernetes phục vụ và đảm nhận việc lưu trữ custom resource của bạn. CRD cho phép bạn
  tạo các loại resource mới cho cluster mà không cần viết và vận hành một API server tùy chỉnh.
- [Tầng tổng hợp (aggregation layer)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/)
  nằm phía sau API server chính, và API server chính đóng vai trò như một proxy.
  Cách bố trí này được gọi là API Aggregation (AA), cho phép bạn cung cấp
  các hiện thực chuyên biệt cho các custom resource của mình bằng cách viết và
  triển khai API server của riêng bạn.
  API server chính ủy quyền (delegate) các request tới API server của bạn đối với những custom API mà bạn chỉ định,
  giúp chúng khả dụng cho tất cả các client của nó.

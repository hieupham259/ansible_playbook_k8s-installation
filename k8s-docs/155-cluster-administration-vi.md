# Quản trị cluster (Cluster Administration)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/cluster-administration/>
>
> Các chi tiết ở mức thấp hơn liên quan đến việc tạo hoặc quản trị một cluster Kubernetes.

Trang tổng quan về quản trị cluster dành cho bất kỳ ai đang tạo hoặc quản trị một cluster
Kubernetes. Trang này giả định bạn đã có một số hiểu biết về các
[khái niệm](https://kubernetes.io/docs/concepts/) cốt lõi của Kubernetes.

## Lập kế hoạch cho cluster (Planning a cluster)

Xem các hướng dẫn trong mục [Setup](https://kubernetes.io/docs/setup/) để có các ví dụ về
cách lập kế hoạch, dựng và cấu hình cluster Kubernetes. Các giải pháp được liệt kê trong
bài viết này được gọi là *distro*.

> **Ghi chú:**
> Không phải distro nào cũng được duy trì (maintain) một cách tích cực. Hãy chọn những
> distro đã được kiểm thử với một phiên bản Kubernetes gần đây.

Trước khi chọn một hướng dẫn, dưới đây là một số điểm cần cân nhắc:

- Bạn muốn thử nghiệm Kubernetes trên máy tính của mình, hay muốn xây dựng một cluster
  nhiều node có tính sẵn sàng cao (high availability)? Hãy chọn distro phù hợp nhất với
  nhu cầu của bạn.
- Bạn sẽ sử dụng **một cluster Kubernetes được host sẵn (hosted)**, chẳng hạn như
  [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/), hay **tự host
  cluster của riêng mình**?
- Cluster của bạn sẽ đặt **tại chỗ (on-premises)** hay **trên cloud (IaaS)**? Kubernetes
  không hỗ trợ trực tiếp cluster lai (hybrid). Thay vào đó, bạn có thể dựng nhiều cluster.
- **Nếu bạn đang cấu hình Kubernetes on-premises**, hãy cân nhắc xem
  [mô hình mạng (networking model)](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
  nào phù hợp nhất.
- Bạn sẽ chạy Kubernetes trên **phần cứng "bare metal"** hay trên **máy ảo (VM)**?
- Bạn **muốn vận hành một cluster**, hay dự định **tham gia phát triển mã nguồn của dự án
  Kubernetes**? Nếu là trường hợp sau, hãy chọn một distro đang được phát triển tích cực.
  Một số distro chỉ dùng bản phát hành nhị phân (binary release), nhưng lại cung cấp nhiều
  lựa chọn đa dạng hơn.
- Làm quen với các [thành phần (components)](https://kubernetes.io/docs/concepts/overview/components/)
  cần thiết để vận hành một cluster.

## Quản lý cluster (Managing a cluster)

* Tìm hiểu cách [quản lý node](https://kubernetes.io/docs/concepts/architecture/nodes/).
  * Đọc về [tự động mở rộng node (Node autoscaling)](https://kubernetes.io/docs/concepts/cluster-administration/node-autoscaling/).

* Tìm hiểu cách thiết lập và quản lý [resource quota](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
  cho các cluster dùng chung.

## Bảo mật cluster (Securing a cluster)

* [Generate Certificates](https://kubernetes.io/docs/tasks/administer-cluster/certificates/)
  mô tả các bước tạo certificate bằng những bộ công cụ (tool chain) khác nhau.

* [Kubernetes Container Environment](https://kubernetes.io/docs/concepts/containers/container-environment/)
  mô tả môi trường dành cho các container do Kubelet quản lý trên một node Kubernetes.

* [Controlling Access to the Kubernetes API](https://kubernetes.io/docs/concepts/security/controlling-access)
  mô tả cách Kubernetes triển khai kiểm soát truy cập (access control) cho chính API của nó.

* [Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
  giải thích việc xác thực (authentication) trong Kubernetes, bao gồm các tùy chọn xác thực
  khác nhau.

* [Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
  là bước phân quyền (authorization), tách biệt với xác thực, và kiểm soát cách các lời gọi
  HTTP được xử lý.

* [Using Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
  giải thích các plug-in chặn (intercept) các request tới Kubernetes API server sau bước
  xác thực và phân quyền.

* [Admission Webhook Good Practices](https://kubernetes.io/docs/concepts/cluster-administration/admission-webhooks-good-practices/)
  cung cấp các thực hành tốt và những điểm cần cân nhắc khi thiết kế mutating admission
  webhook và validating admission webhook.

* [Using Sysctls in a Kubernetes Cluster](https://kubernetes.io/docs/tasks/administer-cluster/sysctl-cluster/)
  mô tả cho quản trị viên cách sử dụng công cụ dòng lệnh `sysctl` để thiết lập các tham số
  kernel.

* [Auditing](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/) mô tả cách làm
  việc với log kiểm toán (audit log) của Kubernetes.

### Bảo mật kubelet (Securing the kubelet)

* [Giao tiếp giữa Control Plane và Node](./24-control-plane-node-communication-vi.md)
* [TLS bootstrapping](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/)
* [Xác thực/phân quyền cho kubelet](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/)

## Các dịch vụ cluster tùy chọn (Optional Cluster Services)

* [DNS Integration](./10-dns-pod-service-vi.md) mô tả cách phân giải một tên DNS trực tiếp
  thành một service của Kubernetes.

* [Logging and Monitoring Cluster Activity](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
  giải thích cách hoạt động của logging trong Kubernetes và cách triển khai nó.
